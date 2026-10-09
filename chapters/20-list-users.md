# 第 20 章 ListUsers：从对象出发找主体

> 本章目标：理解 ListUsers 的语义（用户过滤器、userset 结果、通配符结果）；读懂 OpenFGA 一次遍历完成集合运算的实现，特别是它对通配符和“被排除的用户”的处理；理解 mini 选择的“候选 + Check 确认”设计，以及为了和 OpenFGA 的错误语义一致而做的两次修正。
> 前置：第 14、16、18 章。对应代码：mini-openfga **v0.4** `pkg/listusers/listusers.go`（v1.0 修正了 subtract 一侧的条件处理）。

## 20.1 语义

“谁能看 design.doc？”请求：

```json
{"object": {"type": "document", "id": "design"}, "relation": "can_view",
 "user_filters": [{"type": "user"}]}
```

`user_filters` 指定要哪一类主体（目前只允许一个）：

- `{"type": "user"}`：具体用户，以及 `user:*`（如果对所有用户公开）；
- `{"type": "group", "relation": "member"}`：返回 userset，例如“eng 组的成员整体上可以看”。

响应里每个用户是三种形态之一：

```json
{"users": [
  {"object": {"type": "user", "id": "alice"}},
  {"userset": {"type": "group", "id": "eng", "relation": "member"}},
  {"wildcard": {"type": "user"}}
]}
```

用第 3 章的云盘数据比较 OpenFGA 和 mini（结果排序后完全相同）：

| ListUsers | 结果 |
|---|---|
| `document:design#viewer`，过滤 `user` | alice, bob, carol, ceo, dave |
| `document:design#can_view`，过滤 `user` | alice, bob, carol, ceo（dave 被拉黑） |
| `document:design#can_share`，过滤 `user` | bob |
| `folder:eng#editor`，过滤 `user` | alice |
| `document:design#viewer`，过滤 `folder#viewer` | folder:eng#viewer, folder:eng-docs#viewer, folder:root#viewer |

最后一行说明 userset 结果的含义：三个文件夹的 viewer 整体上都能看这份文档（通过 `viewer from parent`）。

## 20.2 OpenFGA：正向遍历 + 集合代数

ListUsers 的方向和 Check 一样，从对象出发沿重写规则展开，但每个节点产出的不是布尔值，而是一个**用户集合**。`pkg/server/commands/listusers/list_users_rpc.go`：

- `ListUsers`（:198）：如果用户过滤器和目标不是同一种 userset，先用第 18 章的关系图问一句 `doesHavePossibleEdges`（:323）：从这种用户到这个关系根本没有路径，就直接返回空；
- `expand`（:352）：深度检查、环检查（`enteredCycle`，:996，以 `object#relation` 为键），如果当前 `object#relation` 本身匹配一个 userset 过滤器就把它作为结果发出，然后按重写规则分派；
- `expandDirect`（:453）：`Read(object#relation)`，过滤不合法的元组，**求值条件**；user 是对象且类型匹配过滤器就发出；user 是 userset 就递归展开它；
- `expandTTU`（:905）：读 tupleset，对每个父对象递归展开被计算的关系；
- `expandUnion`（:683）、`expandIntersection`（:569）、`expandExclusion`（:771）：并发展开子节点，收集各自的结果，做集合运算。

集合运算的难点在于**通配符**和**被排除的用户**。看 `expandIntersection` 的计数：

```go
for _, foundUsersChan := range intersectionFoundUsersChans {
	go func(foundUsersChan chan foundUser) {
		defer wg.Done()
		foundUsersMap := make(map[string]uint32, 0)
		for foundUser := range foundUsersChan {
			...
			foundUsersMap[key]++
		}
		_, wildcardExists := foundUsersMap[wildcardKey]
		if wildcardExists {
			wildcardCount.Add(1)
		}
		for userKey := range foundUsersMap {
			mu.Lock()
			foundUsersCountMap[userKey]++
			if wildcardExists {
				foundUsersCountMap[userKey]--
			}
			mu.Unlock()
		}
	}(foundUsersChan)
}
...
for key, count := range foundUsersCountMap {
	if _, excluded := excludedUsersMap[key]; excluded {
		continue
	}
	if (count + wildcardCount.Load()) == uint32(len(childOperands)) {
		... // 发出这个用户
	}
}
```

一个用户在交集里，当且仅当它出现在**每一个**操作数里，**或者**那个操作数里有 `user:*`。所以“出现次数 + 含通配符的操作数个数 == 操作数个数”。`foundUsersCountMap[userKey]--` 那一行防止一个操作数里同时有 alice 和 `user:*` 时 alice 被重复计数。

`expandExclusion`（:771）更复杂：base 里有 `user:*`、subtract 里有具体的 bob 时，结果是“`user:*`，但 bob 除外”。API 的用户列表表达不了“除外”，所以内部用一个 `relationshipStatus`（`HasRelationship` / `NoRelationship`）和 `excludedUsers` 列表在节点之间传递这种信息，再由上层的 union、intersection 继续合并。最终返回的列表里会有 `user:*`，而 bob 不会单独出现。

`expandExclusion` 对 subtract 一侧的环有一个简单粗暴的处理（:811）：只要 subtract 一侧遇到环，整个排除返回空集。第 31 章的差分测试发现，这会让 ListUsers 和 OpenFGA **自己的** Check 不一致：g0、g2 两个组互相包含、没有成员，`blocked` 包含 g2，Check 认为 owner u0 能看（第 12 章 12.3.1 节），ListObjects 也返回这份文档，ListUsers 却返回空列表。mini 的 ListUsers 建立在 Check 之上，三个 API 互相一致，在这一点上与 OpenFGA 的 ListUsers 不同。

**错误语义**也和 Check 不同。`expandDirect` 遇到条件求值失败时：

```go
if err != nil {
	errs = errors.Join(errs, err)
	if !errors.Is(err, condition.ErrEvaluationFailed) {
		break LoopOnIterator
	}
}
if !condMet {
	continue
}
```

记下错误，继续处理其他元组，最后把所有错误一起返回。和 Check 的区别是：Check 的 union 里只要有一条路为真，其他路上的错误就被忽略；**ListUsers 返回的是一个集合，集合里有一个成员无法确定，整个答案就不可靠**，所以只要遇到条件求值错误，请求就失败。`expandExclusion` 还会把 subtract 一侧的错误也合并进来。

## 20.3 mini-openfga：候选 + Check 确认

mini 没有移植这套集合代数，而是分两步：

1. **收集候选**：沿重写规则正向遍历，把所有**可能**有这个关系的、匹配过滤器的主体收集起来，忽略 `and`、`but not`、条件可能去掉的那些（得到一个超集）；
2. **逐个确认**：对每个候选调用 Check。Check 已经正确实现了 `and`、`but not`、通配符、条件、环。

`pkg/listusers/listusers.go` 的收集部分：

```go
func (c *collector) rewrite(ctx context.Context, object, relation string, rw *model.Rewrite, depth int) error {
	switch rw.Kind {
	case model.This:
		keys, err := c.read(ctx, object, relation)
		...
		for _, k := range keys {
			if c.matches(k.User) {
				c.add(k.User) // user:anne, user:*, group:eng#member
			}
			if obj, rel := tuple.SplitObjectRelation(k.User); rel != "" {
				errs = errors.Join(errs, c.collect(ctx, obj, rel, depth+1))
			}
		}
		return errs
	case model.Computed:
		return c.collect(ctx, object, rw.Relation, depth)
	case model.TTU:
		... // 读 tupleset，对每个父对象 collect
	case model.Difference:
		// Only the base can grant the relation; the subtract only removes,
		// so its users are not candidates. It is still walked: OpenFGA's
		// expandExclusion evaluates both sides, so a condition on the
		// subtract side that cannot be evaluated fails the request.
		baseErr := c.rewrite(ctx, object, relation, rw.Children[0], depth)
		probe := &collector{...}
		return errors.Join(baseErr, probe.rewrite(ctx, object, relation, rw.Children[1], depth))
	}
	// union and intersection: every operand. For `and` one operand would
	// do, except that a wildcard in one operand (user:*) hides concrete
	// users that only appear in the others.
	...
}
```

注意交集为什么要收集**所有**操作数：`[user:*] and allowed`，第一个操作数只产出 `user:*`，具体的 alice 只出现在第二个操作数里。只收集第一个操作数的话，候选里只有 `user:*`，它在 Check 里会是 false（`allowed@user:*` 不存在），alice 就漏掉了。

确认阶段对每个候选做一次 Check（有界并发），为 true 的留下。`user:*` 本身也是候选：Check `document:1#viewer@user:*` 为 true，说明公开，结果里就有 `{"wildcard": {"type": "user"}}`；被排除的 bob 没有被收集为候选（subtract 一侧不产出候选），所以不会出现在结果里。这与 OpenFGA 的输出一致。

代价是每个候选一次 Check，换来的是“集合代数”只在 Check 里写一遍。

### 20.3.1 两次修正：错误语义

第一次运行矩阵测试时，ListUsers 是 726/730。失败的 4 条来自同一个测试 `relation_through_ttu_with_condition`：文档有两个父文件夹，分别带条件 `str_cond(s)` 和 `xcond(x)`，请求只提供了 `s`。Check 经过 `str_cond` 那个父文件夹得到 true，另一个父文件夹的缺参数错误被 union 忽略了，所以“确认”阶段认为 jon 有权限；而 OpenFGA 的 ListUsers 在遍历时求值了**每一条**元组的条件，`xcond` 缺参数，整个请求失败（2000）。修正：收集阶段也求值条件，遇到错误就记下并让请求失败（`collector.read`）。之后是 730/730。

第二次修正在 v1.0。第 31 章的差分测试（随机数据 + 条件 + 不带 context）发现，`document:d0#can_view`（`viewer but not blocked`）在 OpenFGA 上报错，在 mini 上返回空列表。原因：OpenFGA 的 `expandExclusion` 也展开了 subtract 一侧，`blocked` 元组的条件缺参数；mini 只遍历 base。修正：对 subtract 一侧用一个“探针”收集器走一遍，只收集错误、不产出候选（上面代码里的 `probe`）。

## 20.4 运行与验证

```bash
cd code/mini-openfga && git checkout v0.4
go run ./cmd/minifga list-users -model examples/drive/model.fga -tuples examples/drive/tuples.txt document:design#can_view user
```

```json
{
  "users": [
    "user:alice",
    "user:bob",
    "user:carol",
    "user:ceo"
  ]
}
```

```bash
go run ./cmd/minifga matrix -api ListUsers
```

```text
ListUsers    assertions: 730 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
```

## 20.5 小结

- ListUsers 从对象出发正向遍历，结果是一个主体集合，可以包含具体用户、userset、通配符；
- OpenFGA 在遍历中直接做集合代数，用计数处理交集、用 `excludedUsers` 和关系状态处理“通配符但某人除外”；
- 它的错误语义比 Check 严格：集合里任何一个成员因条件无法确定，整个请求失败，包括 subtract 一侧；
- mini 用“收集候选的超集 + 逐个 Check 确认”实现，代码少、复用 Check 的语义，但错误语义需要单独对齐。

## 20.6 练习

1. （巩固）模型 `define viewer: [user:*] but not blocked`，数据 `doc:1#viewer@user:*`、`doc:1#blocked@user:bob`。ListUsers(doc:1#viewer, user) 返回什么？`bob` 会出现吗？
2. （扩展）mini 的确认阶段对每个候选都做一次完整的 Check。如果一份文档授权给了一个有 10 万成员的组，会发生什么？设计一个优化：哪些候选不需要确认？（提示：路径上没有经过 `and`、`but not` 和条件的候选。）
3. （读源码）`expandExclusion`（:771）里有一个分支 `if subtractHasCycle { return expandResponse{err: nil} }`：subtract 一侧遇到环时，整个排除返回空。用第 12 章的规则（subtract 遇到环当作 true）解释它。

## 20.7 延伸阅读

- OpenFGA 文档 *ListUsers*：<https://openfga.dev/docs/getting-started/perform-list-users>
- `pkg/server/commands/listusers/list_users_rpc.go` 全文（约 1000 行）
- 矩阵测试中名字包含 `wildcard`、`exclusion` 的 listUsersAssertions
