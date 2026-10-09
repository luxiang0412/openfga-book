# Lab 3：mini-openfga v0.3：临时授权、子问题缓存、BatchCheck 与 Expand

> 目标：用条件实现“合同到期自动失效”的授权；观察子问题缓存怎样让第二、三个文档的检查少读存储；体验 BatchCheck 的去重和 Expand 的单层展开；确认 Check 矩阵测试全部通过。
> 前置：第 14–17 章。对应代码：mini-openfga **v0.3**（本 Lab 的程序在 v1.0 的 `examples/lab3/main.go`，在 v0.3 上照抄即可运行）。

## 1. 本版改了什么

```bash
cd code/mini-openfga
git diff --stat v0.2 v0.3
```

| 新增 | 内容 |
|---|---|
| `pkg/condition/` | CEL 编译与求值、参数类型、ipaddress、`$expression`、JSON 规整（第 14 章） |
| `pkg/cache/lru.go` | 泛型 LRU + TTL |
| `pkg/check/cached_resolver.go`、`throttling_resolver.go`、`builder.go` | resolver 链（第 15 章） |
| `pkg/expand/` | Expand（第 16 章） |
| `pkg/server/batch_check.go` | BatchCheck（第 17 章） |
| `pkg/server/server.go` | `Config`、`WithCheckCache`、`WithDispatchThrottling`；Check 支持 context 与 consistency |

## 2. 模型与数据

```text
type user
type folder
  relations
    define viewer: [user, user with non_expired]
type document
  relations
    define parent: [folder]
    define viewer: [user] or viewer from parent
condition non_expired(current_time: timestamp, expires_at: timestamp) {
  current_time < expires_at
}
```

`folder:specs` 下有三份文档 spec1..spec3；anne 是 specs 的 viewer；外包人员 contractor 也是 viewer，但带条件 `non_expired`，元组上写着 `expires_at = 2026-12-31`。

## 3. 运行

```bash
go run ./examples/lab3
```

```text
== 1. time-limited access
current_time="2026-06-01T00:00:00Z" allowed=true
current_time="2027-01-01T00:00:00Z" allowed=false
current_time=""                     error: 'non_expired' - tuple is missing context parameters '[current_time]'
== 2. the check cache
document:spec1#viewer@user:anne allowed=true queries=3 cache hits +0
document:spec2#viewer@user:anne allowed=true queries=2 cache hits +1
document:spec3#viewer@user:anne allowed=true queries=2 cache hits +1
again spec1:                        queries=0
spec1 with HIGHER_CONSISTENCY:       queries=3
== 3. batch check
anne-1        allowed=true
anne-1-again  allowed=true
contractor-2  allowed=true
bob-3         allowed=false
== 4. expand
{"name":"document:spec1#viewer","union":[{"name":"document:spec1#viewer"},{"name":"document:spec1#viewer","tupleToUserset":{"tupleset":"document:spec1#parent","computed":["folder:specs#viewer"]}}]}
```

### 3.1 临时授权

合同期内为 true，过期后为 false，**不需要任何人去删除元组**。不传 `current_time` 时是一个错误，而不是 false：引擎不知道现在几点，就不做判断（第 14 章的“缺少参数”）。

在实际系统里，`current_time` 由调用方的服务端填写（不能让客户端自己报时间）。`expires_at` 写在元组上，调用方无法覆盖：即使请求里也带了一个 `expires_at`，求值时元组上的值优先。

### 3.2 子问题缓存

- spec1：缓存是空的，读了 3 次存储（文档自己的 viewer、文档的 parent、folder:specs 的 viewer）；
- spec2：只读了 2 次。`folder:specs#viewer@user:anne` 这个子问题在检查 spec1 时已经被缓存，所以**命中 1 次**，省掉一次读取；
- spec3：同上；
- 再查 spec1：整个问题本身就在缓存里，**0 次读取**；
- 带 `HIGHER_CONSISTENCY`：绕过缓存，又是 3 次读取。

这就是第 15 章说的“环形链让每个子问题都经过缓存”的实际效果：同一个文件夹下的文档越多，省得越多。

### 3.3 BatchCheck

`anne-1` 和 `anne-1-again` 是同一个问题，只求值一次。`contractor-2` 带自己的 context。bob 什么关系都没有，false。四项共用一个请求。

### 3.4 Expand

`document:spec1#viewer` 是一个 union，两个子节点：直接用户（这份文档没有直接的 viewer，所以是空的；mini 内部的 JSON 用了 `omitempty`，空列表被省略了，HTTP 接口会输出 OpenFGA 格式的 `{"leaf":{"users":{"users":[]}}}`），以及指向 `folder:specs#viewer` 的 TTU 叶子。

## 4. 矩阵测试

```bash
go run ./cmd/minifga matrix
```

```text
Check assertions: 939 passed, 0 failed, 0 skipped (100.0%); tests: 298/298 fully passed
```

缓存和限流不改变任何答案：

```bash
go test ./internal/matrix/ -run TestMatrixCheckWithCacheAndThrottling -v 2>&1 | grep assertions
```

## 5. 动手改

1. **公司网络限制**：给 folder 再加一种 `user with in_office`（第 14 章的 ipaddress 条件），让 anne 只有在 `10.0.0.0/8` 内才能看。分别用 `user_ip` = `10.1.2.3` 和 `192.168.0.1` 检查。
2. **缓存的代价**：开着缓存，先查一次 spec1（anne 为 true），然后删除 `folder:specs#viewer@user:anne`，立刻再查。结果是什么？为什么？有哪两种办法让它立刻变成 false？（答案在第 23 章。）
3. **限流**：用 `server.WithDispatchThrottling(1, 50*time.Millisecond)` 创建 Server，构造一个需要 5 次 dispatch 的检查，计时并观察 `ResolutionMetadata.DispatchThrottled`。

## 6. 本版小结

v0.3 补齐了 Check 的全部语义（条件），并在 resolver 链上加入了缓存和限流，同时提供了 Expand 和 BatchCheck。Check 的矩阵测试在这一版全部通过。代码量 5020 行（非测试）。

下一步（Part V）是另一个方向的问题：不再问“能不能”，而是问“能对哪些”“谁能”。
