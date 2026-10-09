# 第 8 章 最朴素的 Check：把重写规则翻译成递归

> 本章目标：不考虑并发、不考虑性能，写出一个**能用**的 Check：每种重写规则对应一段递归代码；想清楚三种集合运算遇到错误时该怎么办；用 OpenFGA 的矩阵测试量出它离“正确”还差多远。
> 前置：第 4–7 章。对应代码：mini-openfga **v0.1** `pkg/check/check.go`（v0.2 起保留为 `pkg/check/reference.go`，作为对照用的参考实现）。

## 8.1 Check 的定义

Check 回答：`user` 是否属于 `object#relation` 这个用户集。按第 1 章的说法，`object#relation` 的成员由它的重写规则定义。于是 Check 的定义是递归的：

```text
check(object#relation@user) = eval(rewrite(object.type, relation), object, user)

eval(This)              = 存储里有 object#relation@user 这条元组
eval(Computed r)        = check(object#r@user)
eval(TTU r from ts)     = ∃ 元组 object#ts@p ： check(p#r@user)
eval(Union c1..cn)      = eval(c1) ∨ … ∨ eval(cn)
eval(Intersection c1..) = eval(c1) ∧ … ∧ eval(cn)
eval(Difference b, s)   = eval(b) ∧ ¬eval(s)
```

六行公式，对应 Zanzibar 的六种原语。本章把这六行原样翻译成 Go，然后看它在哪些地方不够。

## 8.2 OpenFGA 怎么定义这些语义

OpenFGA 仓库里的 `docs/check/README.md` 用例子说明了每种重写的含义，代码则在 `internal/graph/check.go` 的 `CheckRewrite`（:1041）和几个处理函数里。现在只需要注意三件和语义有关的事，实现细节留给第 9 章：

1. **computed 不算一跳**。`checkComputedUserset`（:840）直接调用 `c.ResolveCheck`，不经过 `dispatch`，深度不加一（同一个对象上换个关系，不会无限加深）。TTU 和 userset 才 `dispatch`、深度加一。
2. **TTU 只认定义了目标关系的父类型**。`viewer from parent` 中，如果某个父对象的类型没有 `viewer` 关系，就跳过它（`produceTTUDispatches`，`default_resolver.go:158`，`ErrRelationUndefined` 时 `continue`）。模型校验只要求“至少一个父类型有这个关系”（第 5 章），不要求全部都有。
3. **错误不一定导致失败**。三个集合运算对错误的处理各不相同（`union` :160、`intersection` :222、`exclusion` :295）：

| 运算 | 规则 | 理由 |
|---|---|---|
| union | 有任一分支为 true → true（忽略其他分支的错误）；否则有错误 → 错误；否则 false | 只要找到一条授权路径就够了，其他路径出不出错无所谓 |
| intersection | 有任一分支为 false → false（忽略错误）；否则有错误 → 错误；否则 true | 只要有一项不满足，交集就为空 |
| exclusion | base 为 false → false；subtract 为 true → false；否则有错误 → 错误；否则 true | 两边都能“决定”结果 |

这张表很重要：它说明一个正确的 Check 不是“遇到错误就返回错误”。如果把错误当作 false，就可能把不该给的权限给出去（exclusion 的 subtract 出错时当成 false，结果变成 true）；如果遇到错误就失败，就会让一条不相关的坏数据挡住所有请求。

## 8.3 实现

`pkg/check/check.go`（v0.1）的入口很短：

```go
// resolve: look up the relation definition and evaluate its rewrite.
func (c *Checker) resolve(ctx context.Context, store string, k *tuple.Key, depth int) (bool, error) {
	if depth >= c.MaxDepth {
		return false, ErrResolutionDepthExceeded
	}
	rel, err := c.ts.Relation(tuple.GetType(k.Object), k.Relation)
	if err != nil {
		return false, err
	}
	return c.rewrite(ctx, store, k, rel.Rewrite, depth)
}
```

`rewrite` 是一个 `switch`，computed 和三个集合运算如下（错误处理完全按 8.2 节的表格）：

```go
case model.Computed:
	// `define viewer: editor` -> check document:1#editor@user. Same object,
	// so it does not count as a hop (OpenFGA does not dispatch here either).
	return c.resolve(ctx, store, tuple.New(k.Object, rw.Relation, k.User), depth)

case model.Union:
	var firstErr error
	for _, child := range rw.Children {
		ok, err := c.rewrite(ctx, store, k, child, depth)
		if err != nil {
			firstErr = err // another branch may still say yes
			continue
		}
		if ok {
			return true, nil
		}
	}
	return false, firstErr

case model.Intersection:
	var firstErr error
	for _, child := range rw.Children {
		ok, err := c.rewrite(ctx, store, k, child, depth)
		if err != nil {
			if firstErr == nil {
				firstErr = err // a later "false" makes the error irrelevant
			}
			continue
		}
		if !ok {
			return false, nil
		}
	}
	return firstErr == nil, firstErr

case model.Difference:
	base, baseErr := c.rewrite(ctx, store, k, rw.Children[0], depth)
	if baseErr == nil && !base {
		return false, nil
	}
	sub, subErr := c.rewrite(ctx, store, k, rw.Children[1], depth)
	if subErr == nil && sub {
		return false, nil
	}
	if baseErr != nil {
		return false, baseErr
	}
	if subErr != nil {
		return false, subErr
	}
	return true, nil
```

直接赋值只看“这个用户本人有没有元组”，并先用 TypeSystem 判断这种用户能不能直接出现在这个关系上，不能就不读存储：

```go
// direct: is exactly object#relation@user stored, and allowed by the model?
func (c *Checker) direct(ctx context.Context, store string, k *tuple.Key) (bool, error) {
	userType, userRel := typesystem.UserShape(k.User)
	if !c.ts.IsDirectlyRelated(tuple.GetType(k.Object), k.Relation, userType, userRel) {
		return false, nil // the model forbids this user type here: no need to read
	}
	t, err := c.ds.ReadUserTuple(ctx, store, k.Object, k.Relation, k.User)
	if errors.Is(err, storage.ErrNotFound) {
		return false, nil
	}
	if err != nil {
		return false, err
	}
	// The model may have changed since the tuple was written.
	return validation.IsValidForRead(c.ts, t), nil
}
```

TTU 读出所有父对象，逐个递归，深度加一：

```go
// ttu: `define viewer: viewer from parent` on document:1 reads every
// document:1#parent@folder:x and checks folder:x#viewer@user.
func (c *Checker) ttu(ctx context.Context, store string, k *tuple.Key, rw *model.Rewrite, depth int) (bool, error) {
	it, err := c.ds.Read(ctx, store, storage.ReadFilter{Object: k.Object, Relation: rw.Tupleset})
	...
	for _, p := range parents {
		if !validation.IsValidForRead(c.ts, p) {
			continue
		}
		parentType := tuple.GetType(p.User)
		if _, err := c.ts.Relation(parentType, rw.Relation); err != nil {
			continue // `viewer from parent` only applies to parent types that define viewer
		}
		ok, err := c.resolve(ctx, store, tuple.New(p.User, rw.Relation, k.User), depth+1)
		...
		if ok {
			return true, nil
		}
	}
	return false, firstErr
}
```

深度上限取 OpenFGA 的默认值 25（`resolveNodeLimit`）。一条 30 层的文件夹链会返回 `ErrResolutionDepthExceeded`，在 API 上是错误码 2002（`authorization_model_resolution_too_complex`），20 层可以正常回答。`check_test.go` 的 `TestDepthLimit` 测的就是这两种情况。

整个 Check 不到 200 行。它是**正确的**，前提是数据里没有 userset、没有通配符、没有环、没有条件。

## 8.4 量一量：OpenFGA 的矩阵测试

v0.1 就把 OpenFGA 的两个 YAML 测试文件放进了 `testdata/openfga/`，并写了一个运行器 `internal/matrix`。它模仿 `tests/check/check.go` 的流程：每个测试一个 store；每个阶段（stage）写入模型和元组；逐条执行断言，比较 `allowed` 或错误码。

```bash
cd code/mini-openfga && git checkout v0.1
go run ./cmd/minifga matrix
```

```text
Check assertions: 325 passed, 191 failed, 0 skipped (63.0%); tests: 78/157 fully passed
```

63%。按失败原因归类：

| 失败 | 数量 | 原因 |
|---|---|---|
| want true, got false | 127 | 没有实现 userset（`group#member`）和通配符（`user:*`）：经过组或公开授权的访问全部判为 false |
| want false, got true | 26 | 忽略了条件，条件不满足的元组也被当成有效；以及 `but not` 的 subtract 一侧需要 userset 才能判为 true |
| want error 2000 / 2027 | 37 | 没有校验上下文元组和条件参数 |
| want error 2002 | 1 | `resolution_too_complex_throws_error`：一条 27 层的 userset 链（`a2: [resource#a1]` … `a27`），期望超过深度上限报错；v0.1 不展开 userset，直接得到 false |

每一类都对应后面的一章：userset 和通配符（第 11 章）、环（第 12 章）、上下文元组（第 13 章）、条件（第 14 章）。这就是矩阵测试的用法：**它不只告诉你“错了”，还按测试名告诉你错在哪一类功能上**。比如 127 个 `want true` 里出现的测试名：`simple_userset_child_wildcard`、`ttu_to_userset`、`userset_as_user`……

> **踩坑记录**：v0.1 的矩阵运行器有一个 bug，没有断言的阶段被整个跳过（连模型和元组都没写），后面依赖这些数据的阶段就查了一个空 store。v0.2 跑出一个奇怪的失败（`ttu_orphan_public_wildcard_parent`：想要 true，得到 false）才发现。所以 v0.1 的 325 这个数字是在有 bug 的运行器上得到的，修复后（v0.2 的运行器）同样的代码会多过几条。运行器本身也是需要测试的代码。

## 8.5 v0.1 缺了什么

| 缺的 | 后果 | 在哪一章补 |
|---|---|---|
| userset 元组的展开 | 经过组、团队的权限全部丢失 | 11 |
| 通配符 | 公开文档对所有人都不可见 | 11 |
| 环检测 | `group:a` 包含 `group:b`、`group:b` 又包含 `group:a` 时，递归到深度上限才停 | 12 |
| 并发与提前结束 | 每个分支都要算完；union 的第一个分支很慢时，第二个分支即使能立即返回 true 也要等 | 9、10 |
| 上下文元组、条件 | 见上表 | 13、14 |

v0.2 用 `LocalChecker` 重写了引擎，而这个朴素版本没有被删除：它补上 userset、通配符和环检测后改名为 `Reference`（`pkg/check/reference.go`），作为**参照实现**。第 10 章的差分测试会拿并发引擎的每一个答案和它比较。一个简单到一眼就能看懂的实现，是检验复杂实现最好的工具。

## 8.6 小结

- Check 的定义是六行递归公式，直接翻译就能得到一个正确但不完整的实现；
- 集合运算的错误处理有讲究：union 里 true 优先，intersection 里 false 优先，exclusion 两边都能决定结果；
- 深度限制是最基本的保护，computed 不计深度，TTU 和 userset 计；
- 矩阵测试按功能告诉你差了什么：v0.1 通过 63%，缺的是 userset、通配符、环、上下文元组和条件。

## 8.7 练习

1. （巩固）在 v0.1 上，对第 3 章的 drive 模型执行 `document:design#can_share@user:alice`，手工推导为什么是 false。
2. （扩展）把 union 的实现改成“遇到错误立即返回错误”，矩阵测试会多失败哪些用例？（提示：搜索名字里带 `condition` 的测试。）
3. （读源码）OpenFGA 的 `exclusion`（`internal/graph/check.go:295`）在 subtract 一侧遇到 `CycleDetected` 时返回 false。用 8.1 节的公式解释这个选择：把“遇到环”当作 subtract 为真，是更安全还是更危险？

## 8.8 延伸阅读

- `docs/check/README.md`：OpenFGA 官方对每种重写的语义说明和例子
- Zanzibar 论文 2.3.1 节：userset rewrite 的定义
- `assets/tests/consolidated_1_1_tests.yaml` 中的 `this`、`computed_userset`、`tuple_to_userset` 三个最简单的用例
