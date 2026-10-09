# 第 27 章 带权图与求值策略

> 本章目标：理解“权重”这个模型层面的量，以及它支撑的两类优化：不读存储就判定“不可能”（PathExists），和为一个子问题选择更便宜的求值方式（weight2、recursive）；对照 mini 的实现，并记录两个让 ListUsers 出错的权重 bug。
> 前置：第 9–12 章。对应代码：mini-openfga **v1.0** `pkg/typesystem/weights.go`、`pkg/check/strategies.go`。

## 27.1 问题：通用算法的扇出

第 11 章的 userset 求值是“每个组派发一次”：

```text
define editor: [user, team#member]
document:d#editor@team:t0#member ... document:d#editor@team:t49#member   （50 条）
team:t49#member@user:anne
```

Check `document:d#editor@user:anne`：读出 50 个团队，对每个团队派发一次 `team:tN#member@user:anne`，49 次白跑。mini 的 `TestWeight2SavesDispatches` 量出来是 **50 次派发、100 次存储读取**。

换个方向想：“anne 直接属于哪些团队”只要一次反向查询（`ReadStartingWithUser`），“文档授权给了哪些团队”一次正向查询，两个集合求交集，非空即为 true。同一个测试，用这种方式是 **0 次派发、4 次读取**。

但这个技巧只在 `team#member` 是**一跳**能到用户的关系时成立。如果团队可以嵌套（`define member: [user, team#member]`），“anne 直接属于的团队”就不是“anne 属于的所有团队”了。要知道什么时候能用，需要一个描述模型结构的量：**权重**。

## 27.2 权重

`weight(type#relation, U)`：从类型为 U 的用户出发，最多经过几条元组能到达 `type#relation`。

```text
define viewer: [user]                    weight(viewer, user) = 1
define viewer: [group#member]            weight(viewer, user) = weight(group#member, user) + 1 = 2
define viewer: viewer from parent        weight = weight(parent 类型#viewer) + 1
define member: [user, group#member]      weight(member, user) = ∞（可以无限嵌套）
```

没有条目表示**根本没有路径**。

OpenFGA 用 `openfga/language` 包的 `WeightedAuthorizationModelGraphBuilder` 构造一张完整的带权图：类型#关系是节点，union/intersection/exclusion 是运算节点，边有 direct、rewrite、TTU 等类型，每条边、每个节点都带一个“按终端类型的权重表”。TypeSystem 构造时建好，`GetWeightedGraph`（`pkg/typesystem/typesystem.go:187`）取用。图上还标记了**递归关系**（一个关系通过自身的 TTU 或 userset 递归）和**元组环**（不同关系之间通过元组形成的环）。

mini 不建图，用不动点迭代直接算出每个“类型#关系”的权重表（`computeWeights`，`weights.go:30`）：所有权重从“未知”开始，反复用重写规则更新，直到不再变化；权重只增不减，超过节点总数的值只可能来自环，截断为 ∞。

用 Lab 4 的公司模型对照两边的结果：

| 节点 | OpenFGA | mini |
|---|---|---|
| `document#blocked` | user=1 | user=1 |
| `document#owner` | user=1 | user=1 |
| `document#parent` | folder=1 | folder=1 |
| `document#viewer` | user=∞ | user=∞，team#member=∞ |
| `document#can_view` | user=∞ | user=∞，team#member=∞ |
| `folder#parent` | folder=1 | folder=1 |
| `folder#viewer` | user=∞ | user=∞，team#member=∞ |
| `team#member` | user=∞ | user=∞，team#member=∞ |

终端类型 `user` 的权重完全一致。mini 多了 userset 作为键的条目（`team#member`），用于下一节判断“userset 作为用户”的路径。

## 27.3 PathExists：不读存储就说“不可能”

`LocalChecker.ResolveCheck` 在进入重写规则之前先问一句（`internal/graph/check.go:455`）：

```go
hasPath, err := typesys.PathExists(tupleKey.GetUser(), relation, objectType)
if err != nil {
	return nil, err
}
if !hasPath {
	return &ResolveCheckResponse{
		Allowed: false,
	}, nil
}
```

`PathExists`（`typesystem.go:751`）在模型图上找从用户类型（或 userset）到 `objectType#relation` 的路径；具体用户还要再试一次通配符 `user:*` 出发的路径。没有路径就直接返回 false，**一次存储都不读**。比如在云盘模型里问 `folder:eng#viewer@document:design`：document 不可能成为文件夹的 viewer。注意 OpenFGA 这里用的是 language 包里另一张**不带权重**的图（`authorizationModelGraph`），带权图用于策略选择。

mini 用权重表实现：`PathExists` 查 `weight(objectType#relation, 用户的键)` 是否存在，用户的键是类型（`user`）、通配符对应的类型，或 userset（`group#member`）。在云盘数据上问 `folder:eng#viewer@document:design`：

```text
开启优化:  {"allowed":false,...,"resolution_metadata":{"dispatch_count":0,"datastore_query_count":0}}
关闭优化:  {"allowed":false,...,"resolution_metadata":{"dispatch_count":2,"datastore_query_count":5}}
```

不开优化时，引擎老老实实沿 `viewer from parent` 和 `editor` 走了一遍才得出 false。

### 27.3.1 踩坑：两个让结果变错的权重 bug

这两个问题都是开启优化后跑矩阵测试发现的，都是**优化把本该为 true 的结果剪成了 false**。这正是优化最危险的地方：它不会报错，只会悄悄给出错误答案。

**userset 到自己的路径**。开启优化后，ListUsers 有 42 条失败，例如 `tuple_to_userset_and_union`：

```text
list-users document:1#viewer filters=[folder#writer]: want [folder:X#writer], got []
```

`document:1#viewer` 通过 `viewer from parent` 到达 `folder:X#viewer`，再通过 `writer` 到达 `folder:X#writer`。而用户 `folder:X#writer` 本身就“是” `folder:X#writer`，路径存在。最初的权重表只有在模型里显式写了 `[folder#writer]` 时才有 `folder#writer` 这个键，`PathExists` 说没有路径。修复：每个节点到自身的权重是 0（`w[n] = map[string]int{n: 0}`，上面 `computeWeights` 里的注释）。

**通配符的键**。修复上一个问题后，又有 6 条 ListObjects/ListUsers 失败，都涉及“`and` 下面的通配符”（`relation_with_wildcard_involving_intersection` 等）。原因是 mini 把 `[user:*]` 记成键 `user:*`，`[user]` 记成键 `user`，交集只保留两边都有的键，`[user:*] and allowed` 的权重表就成了空的。OpenFGA 的带权图把通配符记在终端类型 `user` 下。改成一样之后，全部通过。

## 27.4 求值策略

权重告诉我们一个子问题的“形状”，据此可以选择不同的求值方式。OpenFGA v1 的 LocalChecker 有三种（`internal/graph/`）：

| 策略 | 适用条件 | 做法 |
|---|---|---|
| default | 总是可用 | 第 11 章：读出每个 userset/父对象，逐个派发 |
| weight2 | userset 或 TTU 目标关系对该用户类型的权重为 1，且不在元组环里 | 两个集合求交：对象这一侧的 userset/父对象 × 用户直接所在的 userset/父对象 |
| recursive | 关系通过自身递归、权重为 ∞，其余分支权重为 1 | 沿递归边逐层展开，用集合代替逐个派发 |

条件判断都在 TypeSystem 上：`UsersetUseWeight2Resolver`（`typesystem.go:474`）要求 userset 节点对用户类型的权重**恰好为 1**，而且当前节点不在元组环里、也不是递归关系：

```go
if node.IsPartOfTupleCycle() || len(node.GetRecursiveRelation()) > 0 {
	// if there is a tuple cycle, we have to go through default resolver (or recursive one)
	return false
}
...
// the node itself has to be weight 1 (not 2, because its the userset node that we are verifying at this point). the edge pointing to it would be weight 2.
weight, ok := usersetNode.GetWeight(userType)
...
return weight == 1
```

`TTUUseWeight2Resolver`（:524）、`TTUUseRecursiveResolver`（:596）、`UsersetUseRecursiveResolver`（:679）是类似的判断。weight2 的实现在 `weight_two_resolver.go`，注释说得很清楚（:103）：

> weight2 attempts to find the intersection across 2 producers (channels) of ObjectIDs.
> In the case of a TTU: Right channel is the result set of the Read of ObjectID/Relation that yields the User's ObjectID. Left channel is the result set of ReadStartingWithUser of User/Relation that yields Object's ObjectID.

`checkDirectUsersetTuples`（`check.go:665` 起）把可选的策略放进一个表，交给规划器（第 28 章）选：

```go
possibleStrategies := map[string]*planner.PlanConfig{
	defaultResolver: defaultPlan,
}
...
if typesys.UsersetUseRecursiveResolver(objectType, relation, userType) {
	...
	possibleStrategies[defaultResolver] = defaultRecursivePlan
	possibleStrategies[recursiveResolver] = recursivePlan
	...
	keyPlan := c.planner.GetPlanSelector(key)
	plan := keyPlan.Select(possibleStrategies)
```

这些快速路径**默认开启**。第 2 章提到，`enable-check-optimizations` 实验开关自 v1.9.3 起已经名存实亡：`LocalChecker.optimizationsEnabled` 只被赋值，不再被读取。

在云盘模型上问 OpenFGA 的 TypeSystem（一个调用 `TTUUseRecursiveResolver`/`TTUUseWeight2Resolver` 的小程序），对终端类型 `user`：

```text
folder#editor  editor from parent: recursive=true weight2=false
folder#viewer  viewer from parent: recursive=false weight2=false
document#editor  editor from parent: recursive=false weight2=false
document#viewer  viewer from parent: recursive=false weight2=false
```

只有 `folder#editor` 满足 recursive 的条件；`folder#viewer` 不满足，因为它的另一个分支 `or editor` 权重是 ∞，而条件要求其余分支权重为 1。

## 27.5 mini-openfga 的实现

mini 实现了 default 和 weight2 两种，没有 recursive。但 recursive 策略有一个**可观察的语义**：它用集合运算求成员，自递归关系里的环只意味着“没有新成员”，不会产生环标志。mini v1.0 用 `typesystem.RecursiveRelation`（判断条件对照 OpenFGA 的两个函数，测试里固定了 OpenFGA 对四个模型所有关系的答案）在环检测处复现了这个语义，第 12 章 12.3.1 节和第 31 章有完整经过。`pkg/check/strategies.go`：

```go
for _, ref := range usersets {
	if !ts.UsersetUseWeight2(objectType, k.Relation, userType, ref) {
		remaining = append(remaining, ref)
		continue
	}
	key := "USERSET|" + req.StoreID + "|" + req.ModelID + "|" + objectType + "#" + k.Relation + "|" + userType + "|" + ref.String()
	handlers = append(handlers, c.planned(key,
		c.defaultUsersets(req, []model.RelationRef{ref}),
		c.weight2Userset(req, ref)))
}
if len(remaining) > 0 {
	handlers = append(handlers, c.defaultUsersets(req, remaining))
}
return union(ctx, c.concurrency, handlers...)
```

和 OpenFGA 的结构一致：每个可以用 weight2 的 userset 类型单独成为一个处理器，由规划器在 default 和 weight2 之间选；其余的合在一起走 default。`weight2Userset` 的两侧：

```go
//	right = groups assigned to the object  (document:1#viewer@group:x#member)
//	left  = groups the user is directly in (group:x#member@user:anne)
//
// and the result is "left ∩ right ≠ ∅".
```

用户这一侧还要考虑 `group:*`、条件和上下文元组（`intersectWithUserObjects`）。TTU 的 weight2 是同样的思路：对象的父对象 × 用户有该关系的父对象。

和 OpenFGA 的一个区别是开关：mini 的优化默认**关闭**，`minifga serve -check-optimizations` 打开（第 15 章的 `WithCheckOptimizations`）。这样读者可以对比两种行为，矩阵测试也分别跑两遍。

## 27.6 运行与验证

**三种方式结果相同**。`TestStrategiesAgree` 随机生成 40 轮数据（团队、文件夹、文档、通配符），每个查询分别用强制 default、强制 weight2、规划器选择三种方式求值：

```text
strategies_test.go:109: 4320 checks across 3 strategies, 0 mismatches; 480 editor checks with zero dispatches under weight2
```

**省下的工作量**：

```text
strategies_test.go:133: default: allowed=true dispatches=50 queries=100; weight2: allowed=true dispatches=0 queries=4
```

**开启优化跑全部矩阵**：

```text
matrix_test.go:61: Check assertions: 939 passed, 0 failed, 0 skipped (100.0%); tests: 298/298 fully passed
matrix_test.go:61: ListObjects assertions: 641 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
matrix_test.go:61: ListUsers assertions: 730 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
```

**和 OpenFGA 比工作量**。第 3 章云盘例子 `document:design#can_view@user:alice`，OpenFGA 的日志显示 4 次派发。mini 同样的查询各跑 5 次（分派/读取）：

```text
开启优化:  5/19 6/24 5/19 8/32 7/23
关闭优化:  7/26 6/22 5/20 6/27 7/29
```

两边都不确定（第 10 章：并发分支的取消时机）。在这个模型上 weight2 没有用武之地（上面 OpenFGA 自己的判断也是 `weight2=false`），开不开优化差别不大。OpenFGA 的 `folder#editor` 可以走 recursive 策略而 mini 没有，这是一个已知的差别；但 alice 的授权路径还经过 `folder#viewer`，它不满足 recursive 的条件，所以这个差别**最多解释一部分**分派次数的差距，本书没有进一步拆分。

## 27.7 小结

- 权重是模型层面的量：最多经过几条元组能从某类用户到达某个关系，∞ 表示递归；
- PathExists 用模型结构在读存储之前排除不可能的查询；
- weight2 把“逐个派发”变成“两个集合求交”，前提是权重为 1、没有元组环；recursive 处理自递归的关系；
- 这些优化一旦出错，就是悄悄的错误答案。mini 的两个权重 bug 都是矩阵测试抓到的，策略之间的一致性靠随机差分测试保证。

## 27.8 练习

1. （巩固）模型 `define member: [user, team#member]`，`define viewer: [team#member]`。`weight(viewer, user)` 是多少？weight2 能用于 `viewer` 吗？为什么？
2. （扩展）给 mini 实现 recursive 策略：对 `define member: [user, group#member]`，从用户出发反向找到直接所在的组，再逐层找包含这些组的组（每层一次 `ReadStartingWithUser`），直到和对象一侧的组相交或不再有新组。用 `TestStrategiesAgree` 的方法验证。
3. （读源码）`TTUUseRecursiveResolver`（`typesystem.go:596`）的注释列了 6 个条件。逐条说明为什么需要它，尤其是“不能是元组环的一部分”。

## 27.9 延伸阅读

- `openfga/language` 仓库：`pkg/go/graph/weighted_graph_builder.go`
- `internal/graph/weight_two_resolver.go`、`internal/graph/recursive_resolver.go`
- OpenFGA 博客与 CHANGELOG 中关于 “Check optimizations” 的条目
