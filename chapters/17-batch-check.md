# 第 17 章 BatchCheck：一次请求，多个问题

> 本章目标：理解 BatchCheck 为什么存在；读懂它的三个关键点：关联 ID、去重、有界并发；实现 mini 的 BatchCheck，并和 OpenFGA 的响应对照。
> 前置：第 3、15 章。对应代码：mini-openfga **v0.3** `pkg/server/batch_check.go`。

## 17.1 一个页面要问几十个问题

渲染一个文件列表页时，前端要知道当前用户对每个文件能不能看、能不能编辑、能不能分享：50 个文件 × 3 个权限 = 150 次 Check。一次一次地发，网络往返和每次请求的固定开销（鉴权、解析模型、组装存储包装器）会累积起来。

BatchCheck 把它们放进一个请求：

```json
{"checks": [
  {"tuple_key": {"user": "user:ceo", "relation": "viewer", "object": "document:design"}, "correlation_id": "a"},
  {"tuple_key": {"user": "user:ceo", "relation": "viewer", "object": "document:design"}, "correlation_id": "b"},
  {"tuple_key": {"user": "user:dave", "relation": "can_view", "object": "document:design"}, "correlation_id": "c"},
  {"tuple_key": {"user": "user:dave", "relation": "nope", "object": "document:design"}, "correlation_id": "d"}
]}
```

每一项可以有自己的上下文元组和 context，所有项共用一个 store、一个模型和一个一致性设置。真实 OpenFGA 的响应（第 3 章的云盘数据）：

```json
{"result": {
  "a": {"allowed": true},
  "b": {"allowed": true},
  "c": {"allowed": false},
  "d": {"error": {"input_error": "validation_error", "message": "invalid relation: relation 'document#nope' not found"}}
}}
```

两点值得注意：

- 结果是一个以 `correlation_id` 为键的 map，而不是按顺序的数组。关联 ID 由调用方生成，必须非空且在请求内唯一；
- **单项出错不影响其他项**：`d` 的关系不存在，只有 `d` 带着错误返回，整个请求仍然成功。只有请求级的问题（项数超限、关联 ID 重复）才让整个请求失败。

## 17.2 OpenFGA 的实现

`pkg/server/commands/batch_check_command.go` 的 `Execute`：

**1. 请求级校验**：项数不能超过 `maxChecksAllowed`（默认 50，`--max-checks-per-batch-check`），不能为 0；`validateCorrelationIDs`（:209）检查关联 ID 非空、不重复。

**2. 去重**。用和子问题缓存（第 15 章）相同的键给每一项算一个键（`generateCacheKeyFromCheck`，:232）：

```go
func generateCacheKeyFromCheck(check *openfgav1.BatchCheckItem, storeID, authModelID string) keys.Key {
	checkTupleKey := check.GetTupleKey()
	key := storage.CheckCacheKey(
		storeID,
		checkTupleKey.GetObject(),
		checkTupleKey.GetRelation(),
		checkTupleKey.GetUser(),
		storage.InvariantCacheKey(storeID, authModelID, check.GetContext(), check.GetContextualTuples().GetTupleKeys()...),
	)
	return key
}
```

键相同的项只算一次，结果分发给所有对应的关联 ID。上面例子中的 `a` 和 `b` 只求值一次。

**3. 有界并发**。去重后的每一项在一个最多 `maxConcurrentChecks`（默认 50）个 goroutine 的 pool 里执行，每一项走完整的 `CheckQuery.Execute`（第 3 章）。结果写进一个 `sync.Map`，所有任务结束后，按去重表把结果展开回每个关联 ID：

```go
for key, item := range cacheKeyMap {
	check := item.Check
	pool.Go(func(ctx context.Context) error {
		...
		res, err := bq.checker.Execute(ctx, checkParams)
		...
		resultMap.Store(key, &BatchCheckOutcome{Allowed: res.Allowed, ..., Err: err})
		...
	})
}
_ = pool.Wait()
results := map[CorrelationID]*BatchCheckOutcome{}
for cacheKey, checkItem := range cacheKeyMap {
	res, _ := resultMap.Load(cacheKey)
	outcome := res.(*BatchCheckOutcome)
	for _, id := range checkItem.CorrelationIDs {
		results[id] = outcome
	}
}
```

元数据（总调度次数、总查询次数、重复项数）被汇总后写进日志和指标。

BatchCheck 的另一个收益在第 15 章的缓存里：同一批请求的各项往往共享子问题（同一个用户、同一个文件夹下的多个文件），开了子问题缓存时，后执行的项可以直接用先执行的项算出的中间结果。

## 17.3 mini-openfga 的实现

`pkg/server/batch_check.go` 约 110 行，结构相同：

```go
// group identical checks: same key -> evaluated once
groups := map[string]*group{}
seenIDs := map[string]bool{}
for _, c := range req.Checks {
	if c.CorrelationID == "" || seenIDs[c.CorrelationID] {
		return nil, fgaerr.New(fgaerr.ValidationError, "correlation_id must be unique and non-empty: %q", c.CorrelationID)
	}
	seenIDs[c.CorrelationID] = true
	r := check.NewRequest(req.StoreID, ts.ModelID(), c.TupleKey)
	r.ContextualTuples, r.Context = c.ContextualTuples, c.Context
	key := check.CacheKey(r)
	if g, ok := groups[key]; ok {
		g.ids = append(g.ids, c.CorrelationID)
	} else {
		groups[key] = &group{item: c, ids: []string{c.CorrelationID}}
	}
}
```

去重键直接复用 `check.CacheKey`，保证“BatchCheck 认为相同的两项”和“缓存认为相同的两个子问题”是同一个概念。执行部分用一个信号量通道控制并发，每一项调用 `s.Check`，错误放进该项的结果里。测试 `TestBatchCheck` 断言 `res["a"] == res["b"]`：两个重复项拿到的是同一个结果对象，证明只算了一次。

同一个请求发给 mini-openfga v1.0，结果与 OpenFGA 相同（只有 `d` 的错误信息措辞不同）：

```json
"d": {"error": {"input_error": "validation_error", "message": "relation 'document#nope' not found"}}
```

关联 ID 重复时，两边都拒绝整个请求：

```text
OpenFGA: {"code":"validation_error","message":"received duplicate correlation id: x"}
mini:    {"code":"validation_error","message":"correlation_id must be unique and non-empty: \"x\""}
```

## 17.4 小结

- BatchCheck 把多个 Check 合成一个请求，用关联 ID 对应结果，单项错误不影响其他项；
- 先用子问题缓存的键去重，再在有界并发中执行，最后把结果展开回关联 ID；
- 和子问题缓存一起使用时，同一批里的项还会共享中间结果。

## 17.5 练习

1. （巩固）一个 BatchCheck 里有 3 项，键相同但关联 ID 不同，其中一项带了不同的 `context`。它们会被合并吗？
2. （扩展）OpenFGA 的 BatchCheck 对每一项都单独走 `CheckQuery.Execute`，包括单独组装存储包装器。如果多项共享同一组上下文元组，能不能共享一个 `CombinedTupleReader`？会有什么风险？
3. （读源码）在 `pkg/server/batch_check.go` 里找到 `BatchCheckValidationError` 的处理（:143 附近），请求级错误和单项错误分别是怎样转成 API 响应的？

## 17.6 延伸阅读

- OpenFGA 文档 *Perform a Batch Check*
- `pkg/server/commands/batch_check_command.go`、`batch_check_command_test.go`
