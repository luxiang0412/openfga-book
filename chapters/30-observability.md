# 第 30 章 可观测性：日志、指标、追踪

> 本章目标：知道 OpenFGA 在三种信号里分别记录了什么，特别是它把“分派次数”和“存储查询次数”做成延迟指标标签的设计；实现 mini 的请求日志和 Prometheus 指标；记录一个我在前面章节引入、在这里才发现的回归。
> 前置：第 3、15、24、28 章。对应代码：mini-openfga **v1.0** `pkg/httpapi/middleware.go`、`cmd/minifga/main.go`。

## 30.1 一次 Check 的“工作量”

第 3 章就看到，OpenFGA 的每个查询都会算出两个数：`dispatch_count`（派发了几次子问题）和 `datastore_query_count`（读了几次存储）。它们是理解授权服务性能的关键：Check 的延迟主要由**数据形状**决定（组有多深、文件夹有多少层），同一个 API 的两次调用可以相差几个数量级。只看“Check 的 p99 延迟”无法区分“数据库变慢了”和“有人建了一个 30 层的组嵌套”。

## 30.2 日志

OpenFGA 每个请求结束时记一行结构化日志（zap，JSON）。下面是本书对照用的 OpenFGA 服务上的一条 Check 日志（差分测试的一个 store，字段顺序为原样）：

```json
{
    "level": "info",
    "msg": "grpc_req_complete",
    "grpc_service": "openfga.v1.OpenFGAService",
    "grpc_method": "Check",
    "raw_request": {
        "store_id": "01M4F42Z2EH9QE36J61V69WMP7",
        "tuple_key": {"user": "user:bob", "relation": "viewer", "object": "document:3"},
        "contextual_tuples": {"tuple_keys": []},
        "authorization_model_id": "01M4F42Z2F46HR7DR2TKQKTS1J",
        "trace": false,
        "context": null,
        "consistency": "UNSPECIFIED"
    },
    "raw_response": {"allowed": false, "resolution": ""},
    "query_duration_ms": "0",
    "peer.address": "/tmp/openfga-3378712-371027730/grpc.sock",
    "request_id": "d0864ffa-9bdd-4d24-9b1b-abe34d609399",
    "authorization_model_id": "01M4F42Z2F46HR7DR2TKQKTS1J",
    "dispatch_count": 0,
    "datastore_item_count": 3,
    "request.dispatch_throttled": false,
    "store_id": "01M4F42Z2EH9QE36J61V69WMP7",
    "datastore_query_count": 3,
    "request.datastore_throttled": false,
    "grpc_code": 0
}
```

（略去了时间戳和构建信息。）几点值得注意：

- **完整的请求和响应**都在日志里，包括上下文元组和条件参数 `context`。条件参数可能含有 IP、用户属性等数据，日志的保存和访问需要相应的控制；
- 工作量（`dispatch_count`、`datastore_query_count`、`datastore_item_count`）和是否被限流（第 15 章）都在同一行；
- `peer.address` 是一个 UDS 路径：HTTP 网关在进程内转发（第 24 章）；
- `request_id` 同时作为响应头 `X-Request-Id` 返回（实测），便于把客户端的报错和服务端的日志对上；
- `raw_request` 里的 `"trace": false` 让我去查了这个字段，结论见第 26 章：服务端从不读取它。

## 30.3 指标

OpenFGA 的指标在单独的端口（默认 2112）上，命名空间 `openfga_`。在本书使用的 v1.22 服务上，`/metrics` 里有 23 个 `openfga_` 开头的指标，按用途分：

| 类别 | 指标 |
|---|---|
| 请求 | `request_duration_ms`、`check_duration_ms`、`write_duration_ms`、`check_result_count` |
| 工作量 | `dispatch_count`、`datastore_query_count`、`datastore_item_count`、`datastore_bounded_read_delay_ms` |
| 缓存 | `check_cache_hit_count`、`check_cache_invalid_hit_count`、`check_cache_total_count`、`cache_item_count`、`cachecontroller_*`（3 个） |
| 条件 | `condition_compilation_duration_ms`、`condition_evaluation_duration_ms`、`condition_evaluation_cost` |
| ListObjects | `list_objects_further_eval_required_count`、`list_objects_no_further_eval_required_count`、`list_objects_optimization_count` |
| 迭代器 | `shared_iterator_count`、`shared_iterator_clone_count` |

最有意思的是 `request_duration_ms` 的标签（`pkg/server/server.go:97` 起）：

```go
Help: "The request duration (in ms) labeled by method and buckets of datastore query counts and number of dispatches. This allows for reporting percentiles based on the number of datastore queries and number of dispatches required to resolve the request.",
...
}, []string{"grpc_service", "grpc_method", "datastore_query_count", "dispatch_count", "consistency"})
```

查询次数和派发次数被**分桶**后作为标签（`utils.Bucketize`，`internal/utils/bucket.go:10`：取第一个不小于该值的桶边界，超出则为 `+Inf`）。这样就能回答 30.1 的问题：“需要 ≤50 次查询的 Check，p99 是多少？”如果这个数在变大，是存储变慢了；如果只是“+Inf 桶”的请求变多了，是数据形状变了。分桶是为了控制标签的基数，原始计数直接做标签会让时间序列数量失控。

> 读指标时顺便发现：`list_objects_no_further_eval_required_count` 的帮助文本（`pkg/server/commands/list_objects.go:55`）和它的反义指标 `list_objects_further_eval_required_count`（:49）**一字不差**，都是 “Number of objects in a ListObjects call that needed to issue a Check call to determine a final result”。显然是复制后忘了改。只影响文档，不影响数值。

## 30.4 追踪

OpenFGA 用 OpenTelemetry：`--trace-enabled`，采样器和采样率可配置（`config.go:601` 起校验）。LocalChecker 的每次解析都开一个 span（`ResolveCheck`，`internal/graph/check.go:403`），直接读、userset 读等也各有 span，所以一次 Check 在 Jaeger 里是一棵和第 26 章解析树类似的调用树，但是并发、带取消、带时间的那一棵。

mini 没有实现追踪：它会引入 OpenTelemetry 的一整套依赖，而第 26 章的解析树已经覆盖了“理解一次 Check”的需求。mini 只做了请求 ID：读取或生成 `X-Request-Id`，写回响应头，记在日志里。

## 30.5 mini-openfga 的实现

`Observe` 中间件（`middleware.go:94`）记录每个请求：

```go
rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
start := time.Now()
next.ServeHTTP(rec, r)
elapsed := time.Since(start)
route := r.Pattern
...
m.Requests.WithLabelValues(route, strconv.Itoa(rec.status)).Observe(elapsed.Seconds())
...
log.Info("request", "request_id", id, "method", r.Method, "path", r.URL.Path,
	"route", route, "status", rec.status, "duration_ms", float64(elapsed.Microseconds())/1000)
```

指标用路由模式（`POST /stores/{store_id}/check`）而不是路径做标签，同样是为了控制基数：路径里有 store ID。另外三个指标：每次 Check 的派发次数和查询次数直方图（`minifga_dispatch_count`、`minifga_datastore_query_count`），以及规划器的选择次数（`minifga_planner_selections_total`，第 28 章）。和 OpenFGA 不同，mini 没有把工作量做成延迟的标签，而是单独的直方图，也没有缓存的指标（练习 2）。

在云盘数据上做 5 次 Check 和 1 次 ListObjects 之后（开启优化）：

```text
minifga_datastore_query_count_sum{method="check"} 84
minifga_datastore_query_count_count{method="check"} 5
minifga_dispatch_count_sum{method="check"} 21
minifga_dispatch_count_count{method="check"} 5
minifga_planner_selections_total{strategy="default"} 0
minifga_planner_selections_total{strategy="weight2"} 0
minifga_request_duration_seconds_count{code="200",route="POST /stores/{store_id}/check"} 5
minifga_request_duration_seconds_count{code="200",route="POST /stores/{store_id}/list-objects"} 1
minifga_request_duration_seconds_count{code="200",route="POST /stores/{store_id}/write"} 1
minifga_request_duration_seconds_count{code="201",route="POST /stores"} 1
minifga_request_duration_seconds_count{code="201",route="POST /stores/{store_id}/authorization-models"} 1
```

规划器一次都没有被调用，和第 27 章的结论一致：云盘模型里没有能用 weight2 的关系。

### 30.5.1 踩坑：我自己引入的回归

写本章时看了一眼服务日志，发现所有 API 请求的 `route` 都是 `"/"`：

```json
{"level":"INFO","msg":"request","method":"POST","path":"/stores/01M4F5455R75TDC2TP5KB443JX/check","route":"/","status":200,"duration_ms":0.117}
```

`/metrics` 里也只有 `route="/"` 一个标签值，Check、Write、ListObjects 的延迟全部混在一起，这个指标等于没有。

原因要从 Go 的 `ServeMux` 说起：它把匹配到的模式写进**它收到的那个** `*http.Request` 的 `Pattern` 字段。`minifga serve` 的结构是外层 mux 把 API 挂在 `/` 上，`Observe` 包在最外面，读的是外层请求的 `Pattern`。这原本没问题，因为同一个请求对象一路传下去，内层 API mux 写的模式外层也能看到。第 24 章为了模型 ID 响应头加的 `withModelIDHeader` 用 `r.WithContext(ctx)` 把请求**复制**了一份传给 API mux，模式写进了副本，外层只剩下 `/`。

为了确认，把那次修改之前的提交编译出来跑一遍，标签是 `route="POST /stores/{store_id}/check"`。修复只有一行，把副本的 `Pattern` 拷回来：

```go
inner := r.WithContext(ctx)
next.ServeHTTP(&modelIDWriter{ResponseWriter: w, rec: rec}, inner)
// The mux records the matched route in the request it was given, a
// copy here; hand it back so that Observe labels metrics and logs by
// route (without this every API call was "/", book ch. 30).
r.Pattern = inner.Pattern
```

并加了 `TestObserveRouteLabel`：把 API 挂在外层 mux 的 `/` 上（和 `serve` 一样），检查指标里有 `POST /stores` 和 `POST /stores/{store_id}/check`。去掉修复后这个测试失败（`got map[/:true]`），加上后通过。

教训：第 24 章的修改有针对响应头的测试，却没有任何测试覆盖它**可能破坏的东西**。中间件改变了请求对象，就应该检查下游依赖请求对象的部分。

## 30.6 小结

- OpenFGA 每个请求一行结构化日志，包含完整请求、响应和工作量；
- 它的延迟指标把分桶后的查询次数、派发次数作为标签，能区分“存储变慢”和“数据形状变化”；
- 追踪用 OpenTelemetry，每次解析一个 span；mini 只做了请求 ID；
- mini 的路由标签曾因为中间件复制了请求而失效，已修复并加了回归测试。

## 30.7 练习

1. （巩固）为什么不直接用请求路径（`/stores/01M4.../check`）做指标标签？如果这么做，一个有 1 万个 store 的服务会有多少条时间序列？
2. （扩展）给 mini 加上 Check 缓存的命中、未命中、失效命中三个计数器（对照 OpenFGA 的 `check_cache_*`），并在第 23 章的 `TestCacheInvalidation` 场景下验证它们的值。
3. （读源码）`Bucketize` 用的桶边界来自哪里？默认值是多少？（提示：`requestDurationByQueryHistogramBuckets`，在 `pkg/server/server.go` 和配置里找。）

## 30.8 延伸阅读

- OpenFGA 文档 *Monitoring*：<https://openfga.dev/docs/getting-started/setup-openfga/configure-openfga#telemetry>
- Prometheus 文档 *Metric and label naming*（关于标签基数）
- OpenTelemetry Go SDK 文档
