# Lab 2：mini-openfga v0.2：并发 Check、成环的组与上下文元组

> 目标：在 v0.2 上跑通嵌套组、成环的组、公开文档、拉黑和上下文元组；用竞态检测和差分测试确认并发引擎的正确性；读懂调度次数和查询次数。
> 前置：第 9–13 章。对应代码：mini-openfga **v0.2**（本 Lab 的示例程序在 v1.0 的 `examples/lab2/main.go`，在 v0.2 上可以照抄创建）。

## 1. 看看这一版改了什么

```bash
cd code/mini-openfga
git diff --stat v0.1 v0.2
```

主要变化：

| 文件 | 内容 |
|---|---|
| `pkg/check/interface.go`（新） | `CheckResolver`、`Request`/`Metadata`/`clone`、`Response` |
| `pkg/check/reducers.go`（新） | `fanOutStream`、`union`、`intersection`、`exclusion` |
| `pkg/check/local_checker.go`（新） | `LocalChecker`：`ResolveCheck`、`CheckRewrite`、三路直接赋值、userset、TTU、`consumeDispatches` |
| `pkg/check/check.go` → `reference.go` | 朴素版本保留为参照实现，补上 userset、通配符、环 |
| `pkg/storage/wrappers.go`（新） | `CombinedTupleReader`、`Concat`、`CountingReader` |
| `pkg/server/server.go` | Check 改用 resolver；上下文元组校验；返回调度/查询计数 |
| `internal/matrix/matrix.go` | `_ctxTuples` 变体；修复“没有断言的阶段被跳过”的 bug |

## 2. 示例程序

模型：

```text
type user
type group
  relations
    define member: [user, user:*, group#member]
type document
  relations
    define viewer: [user, user:*, group#member]
    define blocked: [user]
    define can_view: viewer but not blocked
```

数据：eng 组包含 anne，all 组包含 eng 组；a、b 两个组**互相包含**；roadmap 授权给 all 组，secret 授权给 a 组，handbook 公开但拉黑了 mallory。程序对 7 个问题调用 `Server.Check`，并打印 `ResolutionMetadata`：

```bash
go run ./examples/lab2
```

```text
document:roadmap#viewer@user:anne             true  dispatches=2 queries=9
document:roadmap#viewer@group:eng#member      true  dispatches=2 queries=4
document:secret#viewer@user:anne              false dispatches=3 queries=9
document:secret#viewer@user:anne              true  dispatches=2 queries=9  (contextual: [group:b#member@user:anne])
document:handbook#can_view@user:zoe           true  dispatches=0 queries=4
document:handbook#can_view@user:mallory       false dispatches=0 queries=4
document:secret#viewer@user:anne              false dispatches=3 queries=9
```

`true/false` 是确定的；`dispatches` 和 `queries` 可能和你的运行结果略有出入：并发分支被取消的时机不同，做掉的工作量就不同（第 10 章）。例如在 v1.0 上运行同一个程序，第 2 行打印的是 `dispatches=1`。

逐行看：

1. **嵌套组**：roadmap → all 组（dispatch 1）→ eng 组（dispatch 2）→ anne 的直接元组。
2. **userset 作为 user**：“eng 组的成员能看 roadmap 吗？”同样两次 dispatch，最后在 `group:eng#member@group:eng#member` 处由 `IsSelfDefining` 结束，查询更少。
3. **环**：secret → a 组 → b 组 → a 组（环），没有任何组包含 anne，结果 false。dispatch 3 次：a、b、再回到 a 时被 `hasCycle` 截住。
4. **上下文元组**：同一个问题，随请求带上 `group:b#member@user:anne`，结果变成 true：a 组包含 b 组，b 组（在这次请求里）包含 anne。
5. **通配符**：handbook 对 `user:*` 公开，任何 user 都能看，不需要 dispatch。
6. **拉黑**：mallory 能看（通配符），但在 blocked 里，`but not` 给出 false。
7. **上下文元组没有落库**：再问一次第 3 个问题，又回到 false。

## 3. 用 OpenFGA 的矩阵测试检验

```bash
go run ./cmd/minifga matrix                 # 包含 _ctxTuples 变体
go run ./cmd/minifga matrix -contextual=false
```

```text
Check assertions: 720 passed, 219 failed, 0 skipped (76.7%); tests: 235/298 fully passed
Check assertions: 402 passed, 114 failed, 0 skipped (77.9%); tests: 124/157 fully passed
```

所有失败都和条件有关（v0.2 遇到带条件的元组会返回 `conditions are not supported yet`）：

```bash
go run ./cmd/minifga matrix -v | grep FAIL | grep -vc "conditions are not supported"
```

```text
0
```

## 4. 用竞态检测和差分测试检验并发

```bash
go test -race -count=1 ./pkg/check/ -v 2>&1 | grep -E "^(---|ok)|mismatches"
```

```text
--- PASS: TestDriveExample (0.00s)
--- PASS: TestDepthLimit (0.00s)
--- PASS: TestLocalCheckerUsersetsWildcardsCycles (0.00s)
--- PASS: TestContextualTuples (0.00s)
--- PASS: TestMetadataCounts (0.00s)
--- PASS: TestUnionShortCircuits (0.00s)
--- PASS: TestDepthExceeded (0.00s)
    local_checker_test.go:284: 3000 checks compared, 0 mismatches
--- PASS: TestDifferential (1.18s)
ok  	github.com/luxiang0412/mini-openfga/pkg/check	2.211s
```

差分测试用固定的随机种子（42）生成 40 组数据，每组 25 条元组，覆盖组的嵌套和环、文件夹的嵌套和环、通配符、拉黑，然后对 75 个问题比较 `LocalChecker` 和 `Reference` 的答案。

## 5. 动手改

1. **换种子**：把 `TestDifferential` 里的 `rand.NewSource(42)` 改成别的数，或者改成按时间取种子并打印出来，多跑几次。如果哪天出现了不一致，打印出的种子就能复现它。
2. **故意弄坏 exclusion**：在 `reducers.go` 的 `exclusion` 里把“subtract 遇到环当作 true”改成“当作 false”，跑矩阵测试，看哪个用例失败（第 12 章的 `true_butnot_cycle_return_false`）。差分测试能发现吗？为什么？（提示：`Reference` 的实现是不是也要一起改，才会“一起错”？）
3. **观察取消**：在 `consumeDispatches` 的生产者里加一行日志，打印每次读到的元组。构造一个有 100 个组的文档，anne 在第一个组里，看生产者在取消前读了多少条。

## 6. 本版小结

v0.2 的引擎结构已经和 OpenFGA 的 v1 引擎一致：resolver 接口、handler 闭包、三个并发 reducer、生产者-消费者式的派发、路径级的环检测、深度限制、上下文元组。代码量从 2767 行增加到 3618 行（非测试）。

下一步（Part IV）补上最后一块语义：条件。之后再在 resolver 链上插入缓存和限流。
