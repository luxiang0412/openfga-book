# 第 16 章 Expand：把授权规则展开一层给人看

> 本章目标：理解 Expand 返回的“用户集树”是什么、为什么只展开一层；读懂 OpenFGA 的实现；实现 mini 的 Expand，并让它的 JSON 与 OpenFGA 逐字节相同。
> 前置：第 4、11 章。对应代码：mini-openfga **v0.3** `pkg/expand/expand.go`；HTTP 形态见 v0.5 `pkg/httpapi`。

## 16.1 Check 只说“是或否”，Expand 说“为什么可能”

排查权限问题时，“anne 为什么能看这份文档”比“能不能看”更有用。Zanzibar 论文定义了一个 Expand API：给定 `object#relation`，返回它的**用户集树**（userset tree）。OpenFGA 照搬了这个 API。

对第 3 章云盘里的 `document:design#viewer`，真实 OpenFGA 返回：

```json
{
  "tree": {
    "root": {
      "name": "document:design#viewer",
      "union": {
        "nodes": [
          {"name": "document:design#viewer", "leaf": {"users": {"users": ["user:carol"]}}},
          {"name": "document:design#viewer", "leaf": {"computed": {"userset": "document:design#editor"}}},
          {"name": "document:design#viewer", "leaf": {"tupleToUserset": {
              "tupleset": "document:design#parent",
              "computed": [{"userset": "folder:eng-docs#viewer"}]}}}
        ]
      }
    }
  }
}
```

读法：viewer 是三部分的并集：

1. 直接写在上面的用户：carol；
2. `document:design#editor` 的所有成员（要知道是谁，再对它调用一次 Expand）；
3. 父对象 `folder:eng-docs` 的 viewer（同样要再展开）。

**它只展开一层**。叶子节点要么是一组具体的用户（可能包含 `group:eng#member` 这样的 userset 和 `user:*`），要么是一个指向别的 `object#relation` 的指针。为什么不一直展开到底？

- 完全展开可能非常大（一个公开文件夹下的所有人）甚至无限（有环的组）；
- 调用方可以按需逐层展开，比如在 UI 上点开一个节点再请求下一层；
- 它的目的是**解释规则的结构**，不是列出所有用户（那是第 20 章 ListUsers 的工作）。

## 16.2 OpenFGA 的实现

`pkg/server/commands/expand.go`。`Execute`（:50）校验上下文元组、object、relation，用 `CombinedTupleReader` 包装存储，然后从关系的重写规则开始递归（`resolveUserset`，:116）：

```go
switch us := userset.GetUserset().(type) {
case nil, *openfgav1.Userset_This:
	return q.resolveThis(ctx, store, tk, typesys, consistency)
case *openfgav1.Userset_ComputedUserset:
	return q.resolveComputedUserset(ctx, us.ComputedUserset, tk)
case *openfgav1.Userset_TupleToUserset:
	return q.resolveTupleToUserset(ctx, store, us.TupleToUserset, tk, typesys, consistency)
case *openfgav1.Userset_Union:
	return q.resolveUnionUserset(ctx, store, us.Union, tk, typesys, consistency)
case *openfgav1.Userset_Difference:
	return q.resolveDifferenceUserset(ctx, store, us.Difference, tk, typesys, consistency)
case *openfgav1.Userset_Intersection:
	return q.resolveIntersectionUserset(ctx, store, us.Intersection, tk, typesys, consistency)
```

和 Check 的 `CheckRewrite` 一模一样的 `switch`，但每个分支产生的是树节点而不是布尔值：

- `resolveThis`（:146）：`Read` 出 `object#relation` 上所有元组，过滤掉不合法的，去重、排序，得到一个 `users` 叶子；
- `resolveComputedUserset`（:208）：**不读存储**，直接返回一个指向 `object#computedRelation` 的 `computed` 叶子；
- `resolveTupleToUserset`（:240）：读 tupleset（`document:design#parent`），对每个父对象生成一个 `父对象#computedRelation`，去重，得到一个 `tupleToUserset` 叶子；
- 集合运算：对子节点递归（`resolveUsersets`，:426，用 `errgroup` 并发），组成 `union`/`intersection` 节点，或 `difference{base, subtract}`。

注意两个细节：

1. Expand **不求值条件**。带条件的元组照样出现在 `users` 里，它只描述结构；
2. 集合运算的子节点全部展开，没有提前结束：这里不是要一个结论，而是要完整的一层结构。

## 16.3 mini-openfga 的实现

`pkg/expand/expand.go` 约 120 行：

```go
// Node is a node of the userset tree. Exactly one field besides Name is set.
type Node struct {
	Name         string        `json:"name"`
	Users        []string      `json:"users,omitempty"`    // leaf: direct users
	Computed     string        `json:"computed,omitempty"` // leaf: object#relation to expand next
	TTU          *TTULeaf      `json:"tupleToUserset,omitempty"`
	Union        []*Node       `json:"union,omitempty"`
	Intersection []*Node       `json:"intersection,omitempty"`
	Difference   *DifferenceOp `json:"difference,omitempty"`
}

func expand(ctx context.Context, ts *typesystem.TypeSystem, ds storage.TupleReader, store, object, relation string, rw *model.Rewrite) (*Node, error) {
	name := tuple.ToObjectRelation(object, relation)
	switch rw.Kind {
	case model.This:
		keys, err := read(ctx, ts, ds, store, object, relation)
		...
		users := []string{}
		for _, k := range keys {
			if !slices.Contains(users, k.User) {
				users = append(users, k.User)
			}
		}
		slices.Sort(users)
		return &Node{Name: name, Users: users}, nil
	case model.Computed:
		return &Node{Name: name, Computed: tuple.ToObjectRelation(object, rw.Relation)}, nil
	case model.TTU:
		...
	}
	... // 集合运算：递归子节点
}
```

内部用一个扁平的 `Node` 结构；HTTP 层（第 24 章）的 `expandNode` 把它转换成 OpenFGA 的 JSON 形态：`leaf.users.users`、`leaf.computed.userset`、`leaf.tupleToUserset{tupleset, computed[{userset}]}`、`union.nodes`、`difference{base, subtract}`。mini 的子节点顺序求值，没有用并发。

## 16.4 运行与验证

用同一份数据（第 3 章的云盘）分别请求真实 OpenFGA 和 mini-openfga v1.0 的 `POST /stores/{id}/expand`，比较两份 JSON：

```bash
curl -s -X POST $OPENFGA/stores/$S1/expand -d '{"tuple_key":{"object":"document:design","relation":"viewer"}}' > ofga.json
curl -s -X POST $MINI/stores/$S2/expand    -d '{"tuple_key":{"object":"document:design","relation":"viewer"}}' > mini.json
python3 -c 'import json; print("identical:", json.load(open("ofga.json")) == json.load(open("mini.json")))'
```

```text
identical: True
```

命令行（v0.4 起）也能看展开结果，输出的是 mini 内部的结构：

```bash
go run ./cmd/minifga expand -model examples/drive/model.fga -tuples examples/drive/tuples.txt document:design#viewer
```

单元测试 `pkg/server/server_test.go` 的 `TestExpand` 检查三个子节点的形态。

## 16.5 小结

- Expand 返回 `object#relation` 的用户集树，只展开一层：叶子是用户列表或指向下一层的指针；
- 实现与 Check 共用“按重写类型分派”的结构，但不求值条件、不提前结束；
- 它用于解释和排查；列出“所有能访问的用户”是 ListUsers 的事。

## 16.6 练习

1. （巩固）对 `document:design#can_view` 调用 Expand，结果的根节点是什么类型？两个子节点分别是什么？
2. （扩展）写一个客户端函数，反复调用 Expand 把 `document:design#viewer` 完全展开成用户集合，并处理环（同一个 `object#relation` 只展开一次）。和 ListUsers 的结果比较。
3. （读源码）Zanzibar 论文说 Expand 返回的树“follows the userset rewrite rules”，OpenFGA 的 `UsersetTree` 里还有一个 `Leaf.users` 的元素可能是 `group:eng#member`。客户端拿到它之后，应该对哪个 `object#relation` 再次调用 Expand？

## 16.7 延伸阅读

- Zanzibar 论文 2.4.5 节（Expand）
- `pkg/server/commands/expand.go`、`expand_test.go`
- OpenFGA 文档 *Expand API*
