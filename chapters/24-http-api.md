# 第 24 章 HTTP API：协议兼容是可以测出来的

> 本章目标：理解 OpenFGA 的传输层，即 gRPC 服务加一个经由 Unix 域套接字转发的 HTTP 网关，以及错误码到 HTTP 状态码的映射；用 Go 1.22 的 `net/http` 路由实现一个线上格式兼容的 HTTP API；用“同一个客户端、两个服务端”的方法验证兼容性，并修复对照中发现的差异。
> 前置：第 3、22、23 章。对应代码：mini-openfga **v0.5** `pkg/httpapi/httpapi.go`、`internal/matrix/http.go`（v1.0 补上了模型 ID 响应头）。

## 24.1 OpenFGA 的传输层

第 3 章跟踪一次 Check 时已经看到：OpenFGA 的业务实现只有一份，是 gRPC 服务 `OpenFGAServiceServer`（`pkg/server/server.go`）。HTTP API 由 **grpc-gateway** 从 protobuf 注解生成：`openfga/api` 仓库里每个 RPC 都标注了 HTTP 路径，例如 `POST /stores/{store_id}/check`，生成的 `openfga_service.pb.gw.go` 把 JSON 请求解码成 protobuf 消息，再作为 gRPC 客户端调用服务端。

`cmd/run/run.go` 里把两者接起来：

- gRPC 服务在 TCP 上监听（默认 `:8081`），供外部 gRPC 客户端使用；
- 启动 HTTP 时，**同一个 gRPC 服务**再在一个 Unix 域套接字上监听（`createUDS`，:923），注释说明了用意：“HTTP 网关的内部客户端用 UDS，外部 gRPC 客户端继续用 TCP”；
- `dialLocalGrpc`（:681）建立到这个套接字的客户端连接，`runHTTPServer`（:761）用它创建网关 `ServeMux`、注册 `RegisterOpenFGAServiceHandler`（:785）。

第 3 章真实日志里 Check 请求的 `peer.address` 是一个 UDS 路径，就是这个原因：HTTP 请求在进程内又“绕”了一次 gRPC。代价是一次序列化和本地套接字往返，换来的是 gRPC 的拦截器链（认证、日志、指标、校验、超时）对两种协议**完全一致**地生效。

### 24.1.1 错误编码

网关的错误处理器（`run.go:764`）把 gRPC 状态转换成 OpenFGA 的数字错误码，再由 `NewEncodedError`（`pkg/server/errors/encoded_errors.go:71`）按区间映射成 HTTP 状态码：

| 错误码区间 | 类别 | HTTP | gRPC |
|---|---|---|---|
| 1000–1999 | 认证 | 401 | Unauthenticated |
| 2000–3499 | 校验（`validation_error`、`type_not_found`……） | 400 | InvalidArgument |
| 3500–3999 | 限流 | 422 | ResourceExhausted |
| 4000–4999 | 内部错误 | 500 | Internal |
| 5000 及以上 | 找不到（`store_id_not_found`、`undefined_endpoint`） | 404 | NotFound |

另外，`codes.Aborted`（并发写冲突，第 22 章）映射为 409。响应体统一是 `{"code": "<名字>", "message": "..."}`，内部错误的信息会被替换成 `Internal Server Error`，不把内部细节暴露给调用方。

### 24.1.2 模型 ID 响应头

`resolveTypesystem`（`server.go` 末尾附近，:1105）解析出模型后调用：

```go
s.transport.SetHeader(ctx, AuthorizationModelIDHeader, resolvedModelID)
```

于是每个需要模型的请求，响应里都有 `Openfga-Authorization-Model-Id`，调用方可以知道“这次是按哪个模型算的”（第 22 章建议固定模型 ID 的另一面）。

## 24.2 mini-openfga：直接写 HTTP

mini 没有 gRPC，`pkg/httpapi/httpapi.go` 直接用 Go 1.22 起支持方法和路径参数的 `http.ServeMux`：

```go
mux.HandleFunc("POST /stores", h.createStore)
mux.HandleFunc("GET /stores/{store_id}", h.getStore)
mux.HandleFunc("POST /stores/{store_id}/authorization-models", h.writeModel)
mux.HandleFunc("POST /stores/{store_id}/write", h.write)
mux.HandleFunc("GET /stores/{store_id}/changes", h.readChanges)
mux.HandleFunc("POST /stores/{store_id}/check", h.check)
mux.HandleFunc("POST /stores/{store_id}/batch-check", h.batchCheck)
mux.HandleFunc("POST /stores/{store_id}/list-objects", h.listObjects)
mux.HandleFunc("POST /stores/{store_id}/list-users", h.listUsers)
...
mux.HandleFunc("POST /stores/{store_id}/explain", h.explain)  // mini-openfga only (UI)
mux.HandleFunc("POST /dsl/parse", h.parseDSL)                // mini-openfga only (UI)
mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	writeError(w, fgaerr.New(fgaerr.UndefinedEndpoint, "undefined endpoint %s %s", r.Method, r.URL.Path))
})
```

共 22 条路由：OpenFGA 19 个 HTTP 端点中的 17 个（没有实现 UpdateStore 和 StreamedListObjects），加上 `/healthz`、3 个 mini 自己的（`/explain`、`/dsl/parse`、`/assertions/{id}/run`）和兜底的 404。每个处理函数做三件事：把 OpenFGA 的 JSON 形状解码成 `server` 包的请求结构，调用 `server.Server`，把结果编码回 OpenFGA 的 JSON 形状。JSON 形状的细节（`tuple_key` vs `tuple_keys`、ListUsers 的 `{object: {type, id}}`、断言里**不包一层**的 `contextual_tuples`，第 23 章）都在这一层。

错误映射是一个函数，区间和 OpenFGA 一致（mini 没有限流错误）：

```go
func statusOf(code int) int {
	switch {
	case code == fgaerr.Unauthenticated:
		return http.StatusUnauthorized
	case code == fgaerr.DeadlineExceeded:
		return http.StatusGatewayTimeout
	case code >= 2000 && code < 3000:
		return http.StatusBadRequest
	case code >= 5000 && code < 6000:
		return http.StatusNotFound
	}
	return http.StatusInternalServerError
}
```

请求体用 `http.MaxBytesReader` 限制在 4 MB，空请求体当作 `{}`，JSON 语法错误报 `validation_error`。

## 24.3 兼容性怎么测：同一个客户端，两个服务端

“兼容 OpenFGA 的 HTTP API”这句话，只有拿真实的 OpenFGA 对照才有意义。mini 的做法是把矩阵测试的客户端也做成可替换的（`internal/matrix`）：

- `Local`：进程内直接调用 `server.Server`；
- `HTTP`（`internal/matrix/http.go`）：说 OpenFGA HTTP 协议的客户端，模型以 **OpenFGA JSON** 发送（不用 mini 独有的 DSL 字段），因此既能指向 mini，也能指向真实的 OpenFGA。

```go
// HTTP is a Client speaking OpenFGA's HTTP API. It sends models as
// OpenFGA JSON (not DSL), so it works against mini-openfga and against a
// real OpenFGA server alike.
type HTTP struct {
	BaseURL string
	Token   string // optional preshared key
	Client  *http.Client
}
```

先把这个客户端指向**真实的 OpenFGA**，跑全部矩阵测试。这一步验证的不是 mini，而是**客户端本身**：如果客户端对协议的理解有误，在真实服务上就会失败。

```bash
openfga run --playground-enabled=false --http-addr 127.0.0.1:18080 --grpc-addr 127.0.0.1:18081 \
  --experimentals inline_expressions
./dist/minifga matrix -http http://127.0.0.1:18080
```

```text
Check        assertions: 939 passed, 0 failed, 0 skipped (100.0%); tests: 298/298 fully passed
ListObjects  assertions: 641 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
ListUsers    assertions: 730 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
```

（3.3 秒。）第一次跑的时候没加 `--experimentals inline_expressions`，结果是 897/939、633/641、722/730，所有失败都是写模型时的 `$expression requires the "inline_expressions" experimental feature flag`，见第 14 章。

然后把同一个客户端指向 mini，并且让 mini 用 SQLite 存储（`TestMatrixHTTPSQLite`），同样是 2310 条全部通过。**同一个客户端在两个服务端上得到相同的结果**，这才是“协议兼容”的证据。第 31 章的差分测试把这个思路推广到随机生成的数据上。

## 24.4 逐项对照：发现两处差异

矩阵测试只覆盖“正常路径”和少数错误码。写本章时，又用一组脚本对比了两边的 HTTP 状态码、错误名和响应头：

| 请求 | OpenFGA | mini（修复前） |
|---|---|---|
| Check 正常 | 200 | 200 |
| Check 未知关系 | 400 `validation_error` | 相同 |
| Check 未知模型 ID | 400 `authorization_model_not_found` | 相同 |
| JSON 语法错误 | 400 `validation_error` | 相同 |
| 读不存在的 store | 404 `store_id_not_found` | 相同 |
| 未知路由 | 404 `undefined_endpoint` | 相同 |
| **Check 一个不存在的 store** | 400 `latest_authorization_model_not_found` | **404 `store_id_not_found`** |
| **`Openfga-Authorization-Model-Id` 响应头** | 有 | **没有** |

**响应头**：这是 mini 的遗漏。继续确认 OpenFGA 在哪些端点上设置它：write、check、batch-check、expand、list-objects、list-users、读断言都有，**包括模型解析之后才失败的 400**；read 和列出模型没有（它们不解析模型）。mini 用同样的方式实现：服务端解析出模型时，把 ID 记到请求 context 里的一个记录器（`pkg/server/modelid.go`）；HTTP 层包一层 `ResponseWriter`，在写状态行之前把它放进响应头：

```go
func (w *modelIDWriter) WriteHeader(code int) {
	if !w.wroteHeader {
		w.wroteHeader = true
		if id := w.rec.ModelID(); id != "" {
			w.Header().Set(server.AuthorizationModelIDHeader, id)
		}
	}
	w.ResponseWriter.WriteHeader(code)
}
```

`TestModelIDHeader` 覆盖了上面的每一种情况，包括 400 和“不应该有”的两个端点。

**不存在的 store**：进一步对比发现，OpenFGA **根本不检查 store 是否存在**：

| 对不存在的 store | OpenFGA | mini |
|---|---|---|
| Write | 400 `latest_authorization_model_not_found` | 404 `store_id_not_found` |
| 读模型列表、读变更、Read | 200，空结果 | 404 `store_id_not_found` |
| **写模型** | **201，成功** | 404 `store_id_not_found` |

store ID 在 OpenFGA 里只是数据的分区键，任何 API 都不会先查 `store` 表。最后一行意味着，往一个从未创建过的 store ID 里写模型是可以成功的。mini 选择保留更严格的行为，把它作为**有意的差异**记录在这里：对学习者来说，“store 不存在”比“这个 store 没有模型”更容易理解，而矩阵测试和差分测试都不依赖这个行为。

## 24.5 运行与验证

```bash
cd code/mini-openfga && git checkout v1.0
go test ./pkg/httpapi/ ./internal/matrix/ -v -run 'TestEndToEnd|TestModelIDHeader|TestMatrixHTTPSQLite'
./dist/minifga serve -addr 127.0.0.1:8082 &
./dist/minifga matrix -http http://127.0.0.1:8082
```

## 24.6 小结

- OpenFGA 只有一份 gRPC 实现，HTTP 由 grpc-gateway 生成，经 UDS 在进程内转发，所有拦截器对两种协议一致生效；
- 错误码按区间映射到 HTTP 状态码：1xxx→401、2xxx→400、3500+→422、4xxx→500、5xxx→404；
- mini 用 `net/http` 直接实现同样的 JSON 形状和错误映射；
- 兼容性靠“同一个客户端、两个服务端”来证明：先在真实 OpenFGA 上验证客户端，再拿它测 mini；
- 逐项对照找到一个遗漏（模型 ID 响应头，已修复）和一个有意保留的差异（store 存在性检查）。

## 24.7 练习

1. （巩固）为什么要先把矩阵客户端指向真实的 OpenFGA？如果直接只在 mini 上跑，可能漏掉哪一类问题？
2. （扩展）给 mini 加一个 `POST /stores/{store_id}/streamed-list-objects`：OpenFGA 的流式响应是每行一个 JSON 对象（`{"result": {"object": "..."}}`）。用 `http.Flusher` 实现，并说明第 18 章的 ListObjects 需要怎样改才能边算边发。
3. （读源码）`run.go` 在非 Windows 系统上用 UDS，创建失败时回退到 TCP（`goto DIAL_SERVER`）。找到这段代码，说明回退时网关连的是哪个地址。

## 24.8 延伸阅读

- grpc-gateway 文档：<https://grpc-ecosystem.github.io/grpc-gateway/>
- `openfga/api` 仓库的 `openfga/v1/openfga_service.proto`：每个 RPC 的 HTTP 注解
- OpenFGA API 参考：<https://openfga.dev/api/service>
