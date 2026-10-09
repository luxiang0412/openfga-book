# 第 9 章 LocalChecker 解剖：从“返回结果”到“返回待执行的函数”

> 本章目标：理解 OpenFGA 的 Check 引擎为什么要重新组织成 `CheckResolver` 接口 + 处理函数（handler）闭包；读懂 `ResolveCheckRequest` 的克隆语义；把 mini-openfga 的朴素递归改写成同样的结构。
> 前置：第 3、8 章。对应代码：mini-openfga **v0.2** `pkg/check/interface.go`、`pkg/check/local_checker.go`。

## 9.1 朴素版本的三个结构性问题

第 8 章的 `Checker` 是一组互相调用的函数，每个函数直接返回 `(bool, error)`。要在上面加功能，会遇到三个结构性的障碍：

1. **没法并发，也没法提前结束**。`rewrite` 一调用子节点就得等它算完。union 想“几个分支同时跑、谁先 true 就停”，需要先拿到“还没开始算的分支”，再交给一个调度器。
2. **没法插入横切逻辑**。缓存、限流这些功能要作用在**每一个子问题**上（`folder:eng#viewer@user:ceo` 的结果可以缓存，第 15 章），朴素版本的递归直接调用自己，中间没有地方可插。
3. **状态散落在参数里**。深度、已访问的路径、计数器……每加一种就要改所有函数的签名。

OpenFGA 的 `LocalChecker` 对这三个问题各有一个设计：

| 问题 | 设计 |
|---|---|
| 并发与提前结束 | 每个重写节点返回一个 `CheckHandlerFunc`（未执行的闭包），由集合运算决定怎么执行 |
| 横切逻辑 | 所有跨对象的子问题都通过 `CheckResolver` 接口“派发”（dispatch），接口可以层层包装 |
| 状态 | 一个请求对象 `ResolveCheckRequest`，带有克隆语义 |

## 9.2 CheckResolver：一个可以串起来的接口

`internal/graph/interface.go`：

```go
type CheckResolver interface {
	ResolveCheck(ctx context.Context, req *ResolveCheckRequest) (*ResolveCheckResponse, error)
	Close()
	SetDelegate(delegate CheckResolver)
	GetDelegate() CheckResolver
}
```

`ResolveCheck` 回答一个子问题；`SetDelegate` / `GetDelegate` 让每个 resolver 知道“下一个”是谁。缓存层的 `ResolveCheck` 查不到缓存时调用 `delegate.ResolveCheck`；`LocalChecker` 遇到需要跳到别的对象的子问题时，也调用 `delegate.ResolveCheck`。把最后一个 resolver 的 delegate 指回第一个，就构成了一个环（第 15 章）。只有 `LocalChecker` 一个 resolver 时，它的 delegate 就是它自己：

```go
func NewLocalChecker(opts ...LocalCheckerOption) *LocalChecker {
	checker := &LocalChecker{
		concurrencyLimit:   serverconfig.DefaultResolveNodeBreadthLimit, // 10
		maxResolutionDepth: serverconfig.DefaultResolveNodeLimit,        // 25
		upstreamTimeout:    serverconfig.DefaultRequestTimeout,          // 3s
		logger:             logger.NewNoopLogger(),
		planner:            planner.NewNoopPlanner(),
	}
	checker.delegate = checker
	for _, opt := range opts {
		opt(checker)
	}
	return checker
}
```

（`internal/graph/check.go:105`，默认值来自 `pkg/server/config/config.go:25`。）`concurrencyLimit` 叫 *resolve node breadth limit*：同一个节点下最多同时求值几个子问题；`maxResolutionDepth` 叫 *resolve node limit*：一条路径最多跳几次。

## 9.3 ResolveCheckRequest：哪些状态共享，哪些各走各的

一次 Check 会展开成一棵子问题树，每个子问题都有自己的请求对象，由父问题的请求**克隆**而来（`internal/graph/resolve_check_request.go:119`）：

```go
func (r *ResolveCheckRequest) clone() *ResolveCheckRequest {
	var requestMetadata *ResolveCheckRequestMetadata
	origRequestMetadata := r.GetRequestMetadata()
	if origRequestMetadata != nil {
		requestMetadata = &ResolveCheckRequestMetadata{
			DispatchCounter:    origRequestMetadata.DispatchCounter,
			Depth:              origRequestMetadata.Depth,
			DispatchThrottled:  origRequestMetadata.DispatchThrottled,
			DatastoreThrottled: origRequestMetadata.DatastoreThrottled,
		}
	}
	...
	cloned := &ResolveCheckRequest{
		StoreID:          r.GetStoreID(),
		TupleKey:         tupleKey,
		ContextualTuples: r.GetContextualTuples(),
		Context:          r.GetContext(),
		RequestMetadata:  requestMetadata,
		VisitedPaths:     maps.Clone(r.GetVisitedPaths()),
		...
	}
	return cloned
}
```

仔细看哪些字段是**复制值**，哪些是**复制指针**：

| 字段 | 克隆方式 | 含义 |
|---|---|---|
| `Depth` | 复制值 | 每条路径各自计深度：兄弟分支互不影响 |
| `VisitedPaths` | `maps.Clone` | 每条路径各自记录“我从哪里来”，用于检测环（第 12 章） |
| `DispatchCounter` | 复制指针（`*atomic.Uint32`） | 整棵树共享一个计数器，最后报告总调度次数 |
| `DispatchThrottled` 等 | 复制指针 | 整棵树共享的标志 |
| `ContextualTuples`、`Context` | 共享（只读） | 整个请求的输入 |

这个区分很关键。如果 `VisitedPaths` 是共享的，两个并发的兄弟分支恰好都经过 `group:a#member`，后一个会误以为“遇到了环”。如果 `DispatchCounter` 是复制的，最后就数不出总数。

## 9.4 处理函数：先构造，后执行

`CheckHandlerFunc`（`check.go:135`）就是一个闭包类型：

```go
type CheckHandlerFunc func(ctx context.Context) (*ResolveCheckResponse, error)
```

`CheckRewrite`（第 3 章看过）对每种重写返回一个这样的闭包，而不是结果。以集合运算为例（`checkSetOperation`，:987）：

```go
for _, child := range children {
	handlers = append(handlers, c.CheckRewrite(ctx, req, child))
}
...
return func(ctx context.Context) (*ResolveCheckResponse, error) {
	...
	resp, err = reducer(ctx, c.concurrencyLimit, handlers...)
	return resp, err
}
```

子节点的闭包先全部构造好（便宜，只是捕获变量），再交给 `reducer`（`union`、`intersection` 或 `exclusion`）。reducer 拿到的是一组“还没开始的工作”，可以自由决定并发度、执行顺序和何时取消，第 10 章细讲。

构造闭包时有一个容易出错的地方，源码里专门写了注释。`checkDirect`（:799）：

```go
func (c *LocalChecker) checkDirect(parentctx context.Context, req *ResolveCheckRequest) CheckHandlerFunc {
	return func(ctx context.Context) (*ResolveCheckResponse, error) {
		...
		typesys, _ := typesystem.TypesystemFromContext(parentctx) // note: use of 'parentctx' not 'ctx' - this is important
```

闭包被执行时收到的 `ctx` 是 reducer 派生出来的（加了取消），TypeSystem 和存储是放在**构造时**的 `parentctx` 里的。reducer 派生的 ctx 会继承父 ctx 的值，通常两者都能取到；但构造和执行之间的 ctx 来源不同，写错一个就可能在某些路径上取不到。mini 里的写法是在闭包执行时从 `ctx` 取，因为 mini 的 reducer 一律用 `context.WithCancel` 从父 ctx 派生，值一定能继承。

## 9.5 dispatch：跳到别的对象

TTU 和 userset 产生的子问题在另一个对象上，它们不直接调用 `c.ResolveCheck`，而是调用 `dispatch`（:381）：

```go
func (c *LocalChecker) dispatch(_ context.Context, req *ResolveCheckRequest) CheckHandlerFunc {
	return func(ctx context.Context) (*ResolveCheckResponse, error) {
		req.GetRequestMetadata().Depth++
		resp, err := c.delegate.ResolveCheck(ctx, req)
		if err != nil {
			return nil, err
		}
		return resp, nil
	}
}
```

两件事：深度加一，交给 **delegate**。如果链上有缓存层，这个子问题就会先经过缓存。computed 不调 `dispatch`，而是直接 `c.ResolveCheck(ctx, childRequest)`（`checkComputedUserset`，:840）：同一个对象上换个关系，不增加深度，也不经过缓存。

调度计数（`DispatchCounter`）在 OpenFGA 里是在 `processDispatches` 主循环里、真正启动子任务前加一的（`default_resolver.go`，第 11 章）。

## 9.6 处理函数里的 panic

所有 handler 都通过 `runHandler`（:144）执行：

```go
func runHandler(ctx context.Context, handler CheckHandlerFunc) checkOutcome {
	var res *ResolveCheckResponse
	var err error
	recoveredErr := panics.Try(func() {
		res, err = handler(ctx)
	})
	if recoveredErr != nil {
		err = fmt.Errorf("%w: %w", ErrPanic, recoveredErr.AsError())
	}
	return checkOutcome{res, err}
}
```

handler 在 goroutine 里执行，goroutine 里的 panic 不会被 gRPC 拦截器的 recovery 捕获，会直接让整个进程崩溃。所以每个 handler 都要自己兜底，把 panic 转成一个普通错误。一个授权服务因为某个奇怪模型触发的 bug 整个挂掉，比返回一次错误糟糕得多。

## 9.7 mini-openfga v0.2 的实现

### 9.7.1 接口与请求

`pkg/check/interface.go`：

```go
type CheckResolver interface {
	ResolveCheck(ctx context.Context, req *Request) (*Response, error)
	SetDelegate(d CheckResolver)
	GetDelegate() CheckResolver
	Close()
}

// Metadata is shared by a request and all its sub-requests.
type Metadata struct {
	Depth int // hops so far on this path (copied, not shared)
	// counters shared across the whole request tree
	DispatchCount       *atomic.Uint32
	DatastoreQueryCount *atomic.Uint32
	DispatchThrottled   *atomic.Bool // v0.3
}

type Request struct {
	StoreID          string
	ModelID          string
	TupleKey         *tuple.Key
	ContextualTuples []*tuple.Key
	Context          map[string]any
	VisitedPaths     map[string]struct{}
	Metadata         *Metadata
	...
}

// clone copies the request for a child evaluation. VisitedPaths is copied so
// that sibling branches do not see each other's paths; counters are shared.
func (r *Request) clone() *Request {
	c := *r
	c.VisitedPaths = maps.Clone(r.VisitedPaths)
	md := *r.Metadata
	c.Metadata = &md
	return &c
}
```

`md := *r.Metadata` 复制了 `Metadata` 结构体：`Depth` 是值，各自独立；三个计数器字段是指针，复制后仍指向同一个计数器。和 OpenFGA 的克隆语义完全一致，用 Go 的值语义一行就表达出来了。

mini 多数了一个 `DatastoreQueryCount`。OpenFGA 的查询计数在存储包装器里（第 2 章），mini 用一个 `CountingReader` 包装存储，计数器指针放在请求的 Metadata 里，`Server.Check` 返回时一并报告。

### 9.7.2 LocalChecker

```go
type LocalChecker struct {
	delegate    CheckResolver
	concurrency int
	maxDepth    int
}

func NewLocalChecker(opts ...LocalCheckerOption) *LocalChecker {
	c := &LocalChecker{concurrency: DefaultConcurrency, maxDepth: DefaultMaxDepth}
	c.delegate = c
	for _, o := range opts {
		o(c)
	}
	return c
}

// dispatch sends a sub-problem through the resolver chain, one hop deeper.
func (c *LocalChecker) dispatch(req *Request) HandlerFunc {
	return func(ctx context.Context) (*Response, error) {
		req.Metadata.Depth++
		req.Metadata.DispatchCount.Add(1)
		return c.delegate.ResolveCheck(ctx, req)
	}
}
```

`ResolveCheck` 的检查顺序和 OpenFGA 相同：取消 → 深度 → 环 → 自定义（`IsSelfDefining`）→ 取 TypeSystem 和存储 → 查关系 → `CheckRewrite`。`CheckRewrite` 也是同样的 `switch`，返回 `HandlerFunc`。

和 OpenFGA 的差别：

| | OpenFGA | mini v0.2 |
|---|---|---|
| 调度计数的位置 | `processDispatches` 启动子任务前 | `dispatch` 闭包执行时 |
| 查询计数 | 存储包装器 | `CountingReader`，计数器挂在请求上 |
| 追踪（OpenTelemetry span） | 每个 handler 一个 span | 无（第 30 章用指标代替） |
| 策略选择（planner） | 有 | v1.0 才有（第 27、28 章） |
| PathExists 剪枝 | 默认开启 | v1.0 才有，且需要开关 |

### 9.7.3 Server 怎么调用它

```go
r := check.NewRequest(req.StoreID, ts.ModelID(), req.TupleKey)
r.ContextualTuples = contextual
r.Context = condition.Normalize(req.Context)
reader := &storage.CountingReader{
	TupleReader: storage.NewCombinedTupleReader(s.ds, contextual),
	Count:       r.Metadata.DatastoreQueryCount,
}
ctx = typesystem.ContextWith(ctx, ts)
ctx = storage.ContextWithReader(ctx, reader)

resp, err := s.resolver.ResolveCheck(ctx, r)
```

和 `CheckQuery.Execute` 一样：组装请求、包装存储、放进 context、交给 resolver 链的链头。

## 9.8 运行与验证

`pkg/check/local_checker_test.go` 的 `TestMetadataCounts` 验证计数器：document 在 folder:a 里，folder:a 在 folder:b 里，viewer 写在 folder:b 上，应该恰好 dispatch 2 次。

```bash
cd code/mini-openfga && git checkout v0.2
go test ./pkg/check/ -run TestMetadataCounts -v
```

v0.2 的矩阵结果（包括“上下文元组”变体，第 13 章）：

```text
Check assertions: 720 passed, 219 failed, 0 skipped (76.7%); tests: 235/298 fully passed
```

剩下的 219 个失败全部来自条件（第 14 章）：

```bash
go run ./cmd/minifga matrix -v | grep FAIL | grep -vc "conditions are not supported"
```

```text
0
```

## 9.9 小结

- `CheckResolver` 接口 + delegate，让缓存、限流可以插在每一个子问题前面；
- 重写节点返回未执行的 `CheckHandlerFunc`，把“怎么执行”交给集合运算；
- 克隆请求时，深度和已访问路径各走各的，计数器和标志全树共享；
- 跨对象的子问题走 `dispatch`（深度 +1、经过整条链），同对象的 computed 直接递归；
- handler 在 goroutine 里执行，必须自己兜住 panic。

## 9.10 练习

1. （巩固）画出 `document:design#viewer@user:ceo`（第 3 章）的子问题树，标出每个节点的 `Depth` 和 `VisitedPaths`。
2. （扩展）mini 的 `dispatch` 在闭包执行时才给 `DispatchCount` 加一。如果 union 在这个闭包开始之前就被取消了，计数会怎样？和 OpenFGA 的计数位置比，哪个更能反映“实际做了多少工作”？
3. （读源码）`internal/graph/builder.go` 的 `LocalCheckResolver` 函数（:104 附近）在一个 resolver 链里找 `LocalChecker`。它为什么需要沿着 `GetDelegate()` 找，而不是直接持有一个引用？

## 9.11 延伸阅读

- `internal/graph/interface.go`、`internal/graph/resolve_check_request.go`
- `github.com/sourcegraph/conc/panics`：`panics.Try` 的实现
- *Go Concurrency Patterns: Context*（Go 博客）
