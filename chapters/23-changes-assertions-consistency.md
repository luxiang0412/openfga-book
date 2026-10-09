# 第 23 章 变更日志、断言与一致性

> 本章目标：理解 ReadChanges 作为“授权数据的变更流”怎样被下游轮询消费；理解断言作为“模型的单元测试”；理解开启 Check 缓存后写入什么时候能被看到、缓存控制器怎样借助变更日志让缓存失效，以及 `HIGHER_CONSISTENCY` 的作用。
> 前置：第 15、21、22 章。对应代码：mini-openfga **v0.5** `pkg/server/cachecontroller.go`、`assertions.go`、`pkg/storage/datastore.go`（`ReadChanges`）。

## 23.1 ReadChanges：一条可续读的变更流

每次 Write 都在同一个事务里往 `changelog` 表追加记录（第 21 章）：操作（WRITE/DELETE）、元组、时间戳、ULID。`GET /stores/{id}/changes` 按 ULID 升序返回这些记录，并附带一个续读令牌。

`ReadChangesQuery.Execute`（`pkg/server/commands/read_changes.go`）做的事：

- 有令牌：解码出上次读到的 ULID 和对象类型，类型必须与本次请求一致；
- 没有令牌但有 `start_time`：用这个时间构造一个 ULID 作为起点（ULID 的前缀就是时间，第 22 章）；
- 读存储；**没有新变更时，原样返回请求里的令牌**：

```go
changes, contUlid, err := q.backend.ReadChanges(ctx, req.GetStoreId(), filter, opts)
if err != nil {
	if errors.Is(err, storage.ErrNotFound) {
		return &openfgav1.ReadChangesResponse{
			ContinuationToken: req.GetContinuationToken(),
		}, nil
	}
	return nil, serverErrors.HandleError("", err)
}
```

这一点决定了它的用法：消费方保存令牌，定期带着它轮询；没有变化时拿回同一个令牌，有变化时拿到新变更和新令牌。这就是一个简单的“订阅”。典型用途：同步到搜索索引（只索引用户有权看的文档）、审计、让下游缓存失效。

`ChangelogHorizonOffset`（默认 0 分钟）可以让最近 N 分钟内的变更暂不返回：多副本数据库下最新的事务可能还没对所有读者可见，跳过“太新”的变更，保证令牌之前的记录不会再冒出新的。

### 23.1.1 对照验证

同一个脚本对两个服务执行：写两条、删一条，读全部；带令牌再读（无变化）；再写一条，带同一个令牌读。

```text
=== OpenFGA
1) read all:
    WRITE doc:1#viewer@user:anne
    WRITE doc:2#viewer@user:bob
    DELETE doc:1#viewer@user:anne
    token: MDFNNEYzVjgwNlhONk41V0Ra...
2) poll with token, nothing new:
    token: MDFNNEYzVjgwNlhONk41V0Ra...
3) poll again after a write:
    WRITE doc:3#viewer@user:carol
    token: MDFNNEYzVjgxUUZNSzlOTjNC...
=== mini
1) read all:
    WRITE doc:1#viewer@user:anne
    WRITE doc:2#viewer@user:bob
    DELETE doc:1#viewer@user:anne
    token: MDFNNEYzVjgzQlZaVDNTSFZa...
2) poll with token, nothing new:
    token: MDFNNEYzVjgzQlZaVDNTSFZa...
3) poll again after a write:
    WRITE doc:3#viewer@user:carol
    token: MDFNNEYzVjg0WDdFUFoxN0Yw...
```

两边行为一致。令牌都是 base64 编码的 ULID，前缀相同（`MDFNNEYz…` 解码后是 `01M4F3…`），因为它们是同一分钟生成的。

## 23.2 断言：模型的单元测试

断言（assertion）是挂在**某个模型版本**上的一组期望：`(元组, 期望结果)`，可以带上下文元组和条件参数。`WriteAssertionsCommand.Execute`（`pkg/server/commands/write_assertions.go:52`）校验每条断言的 user/object/relation 是否符合模型，上下文元组按写入规则校验，总大小有上限，最多 100 条（protobuf 规则）。

注意 OpenFGA **只存储断言，不执行它们**。执行在客户端：FGA CLI 的 `fga model test`、VS Code 插件、Playground 的 “Run assertions”。断言跟模型 ID 绑定，所以每次写入新模型，需要把断言一并写到新版本上。

mini 多了一个 `POST /stores/{id}/assertions/{model_id}/run`（`pkg/server/assertions.go` 的 `RunAssertions`），在服务端逐条执行 Check，第 26 章的 Playground 用它显示通过/失败。示例（第四条断言故意写错）：

```json
{"results":[
 {"assertion":{"tuple_key":{"object":"doc:1","relation":"viewer","user":"user:anne"},"expectation":true},"got":true,"pass":true},
 {"assertion":{"tuple_key":{"object":"doc:1","relation":"viewer","user":"user:bob"},"expectation":false},"got":false,"pass":true},
 {"assertion":{"tuple_key":{"object":"doc:1","relation":"viewer","user":"user:bob"},"expectation":true,
   "contextual_tuples":[{"object":"doc:1","relation":"owner","user":"user:bob"}]},"got":true,"pass":true},
 {"assertion":{"tuple_key":{"object":"doc:1","relation":"owner","user":"user:anne"},"expectation":false},"got":true,"pass":false}]}
```

> 写这个例子时，我先把 `contextual_tuples` 写成了 Check 请求里的形式 `{"tuple_keys": [...]}`，mini 拒绝了。以为是 mini 的兼容问题，拿同样的请求去试 OpenFGA，它也拒绝（`syntax error ... unexpected token {`）：断言里的 `contextual_tuples` 在 protobuf 里是 `repeated TupleKey`，JSON 中是一个**直接的数组**，和 Check 请求里包了一层的 `ContextualTupleKeys` 不同。两边行为一致，错在我的请求。

## 23.3 一致性：缓存什么时候过期

第 15 章的 `CachedCheckResolver` 把子问题的结果缓存起来（`--check-query-cache-ttl`，默认 10 秒，`pkg/server/config/config.go:44`）。问题是：写入一条元组后，已缓存的 `false` 还会被用多久？

OpenFGA 的答案分三层：

**第一层：什么都不做**。默认只有 TTL：写入后，最多 10 秒内可能还读到旧结果。对很多场景这可以接受（Zanzibar 论文里的 *new enemy problem* 讨论的就是这个窗口）。

**第二层：缓存控制器**（`--cache-controller-enabled`，默认关闭，`--cache-controller-ttl` 默认 10 秒；`internal/cachecontroller/cache_controller.go`）。每次 Check 前，`check_command.go:111` 向它询问这个 store 的“最后失效时间”：

```go
// DetermineInvalidationTime（:150），略去日志和指标
entry, _ := cacheResp.(*storage.ChangelogCacheEntry)
if entry == nil {
	c.InvalidateIfNeeded(ctx, storeID) // async
	return time.Time{}
}
if time.Since(entry.LastChecked) > c.minInvalidationInterval {
	c.InvalidateIfNeeded(ctx, storeID) // async
}
return entry.LastModified
```

它记着每个 store 最近一次变更的时间；超过 TTL（默认 10 秒）没看过 changelog，就**异步**读一次最新的变更（`findChangesAndInvalidateIfNecessary`，:241）。然后缓存的结果只有在它的写入时间晚于这个时间时才有效（`cached_resolver.go`）：

```go
tryCache := req.Consistency != openfgav1.ConsistencyPreference_HIGHER_CONSISTENCY
if tryCache {
	if cachedResp := c.cache.Get(cacheKey); cachedResp != nil {
		res := cachedResp.(*CheckResponseCacheEntry)
		isValid := res.LastModified.After(req.LastCacheInvalidationTime)
		if isValid {
			return res.CheckResponse.clone(), nil
		}
	}
}
```

它不是逐条删除缓存，而是“这个时间点之前写入的缓存条目一律作废”，非常便宜。同一个函数还会让存储层的迭代器缓存失效：如果所有新变更都晚于上次失效，整个 store 的迭代器缓存作废；否则只作废受影响的 `object#relation` 和 `user + 对象类型`。

由于刷新是异步的，而且每个 store 每个 TTL 最多一次，写入后依然有一个窗口，只是这个窗口不再由 Check 缓存的 TTL 决定，而由控制器的 TTL 和请求节奏决定；它还让 Check 缓存的 TTL 可以放心设得更长。

**第三层：`HIGHER_CONSISTENCY`**。请求里带 `"consistency": "HIGHER_CONSISTENCY"`，就完全跳过缓存（上面的 `tryCache`），直接读存储。需要“写后立即读”（刚把用户加进组，下一步就要用这个权限）时用它，代价是没有缓存。

### 23.3.1 mini 的实现与测试

mini 的 `CacheController`（`pkg/server/cachecontroller.go`）是同样的思路，只保留了 Check 缓存的部分（mini 没有迭代器缓存）：

```go
func (c *CacheController) DetermineInvalidationTime(ctx context.Context, store string) time.Time {
	c.mu.Lock()
	e, ok := c.entries[store]
	c.mu.Unlock()
	if !ok || time.Since(e.lastChecked) > c.ttl {
		c.refreshAsync(store)
	}
	return e.lastModified
}
```

`refreshAsync` 对每个 store 最多有一个进行中的刷新，用 `ReadChanges(..., Page{Size: 1}, desc=true)` 只读最新一条。

`TestCacheInvalidation`（`pkg/server/persist_test.go`）把三层行为都写成了断言：开启缓存（TTL 1 小时）和控制器（TTL 50 毫秒）；先 Check 得到 false 并被缓存；写入 owner 元组；

1. 立即 Check：仍然是 **false**（控制器还没看 changelog，这是预期的陈旧读）；
2. 带 `HIGHER_CONSISTENCY` Check：**true**；
3. 等待超过 50 毫秒，触发一次刷新并等它完成，再 Check：**true**，旧的缓存条目被作废。

```bash
cd code/mini-openfga && git checkout v0.5
go test -run TestCacheInvalidation -v ./pkg/server/
```

```text
--- PASS: TestCacheInvalidation (0.06s)
PASS
```

第一条断言值得强调：它证明了陈旧读**确实存在**，而不是被测试碰巧掩盖。理解了这一点，才能正确选择什么时候用 `HIGHER_CONSISTENCY`。

## 23.4 小结

- changelog 与元组同事务写入，ReadChanges 以 ULID 令牌续读，没有变化时返回原令牌，适合轮询订阅；
- 断言是挂在模型版本上的期望，OpenFGA 只存不跑；mini 加了一个服务端执行的接口；
- 开启 Check 缓存后，写入的可见性取决于缓存 TTL；缓存控制器借助 changelog 用“失效时间点”批量作废；`HIGHER_CONSISTENCY` 完全绕过缓存。

## 23.5 练习

1. （巩固）一个消费方把 ReadChanges 的令牌存在 Redis 里，进程崩溃后从令牌处继续。它会漏掉变更吗？会重复处理吗？怎样让处理逻辑对重复变更是安全的？
2. （扩展）给 mini 的 ReadChanges 实现 `start_time` 参数（用 `ulid.Timestamp` 构造起始 ULID），并写一个测试。
3. （读源码）`findChangesAndInvalidateIfNecessary`（:241）在“部分失效”时对每个变更调用了两个函数：`invalidateIteratorCacheByObjectRelation` 和 `invalidateIteratorCacheByUserAndObjectType`。分别对应第 7 章的哪种读法？

## 23.6 延伸阅读

- OpenFGA 文档 *Read Changes*、*Query Consistency*：<https://openfga.dev/docs/interacting/consistency>
- Zanzibar 论文第 2.2 节：*Consistency Model*（zookie 与 new enemy problem）
- `internal/cachecontroller/cache_controller.go` 全文
