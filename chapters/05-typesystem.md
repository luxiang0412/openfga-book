# 第 5 章 TypeSystem：把模型变成可查询的索引，并拒绝坏模型

> 本章目标：理解 TypeSystem 提供的几类查询；读懂 OpenFGA 写入模型时的五条校验规则（尤其是“入口点”和“计算关系成环”）；在 mini-openfga 里实现它们，并和真实 OpenFGA 的校验结果逐条对照。
> 前置：第 4 章。对应代码：mini-openfga **v0.1** `pkg/typesystem/typesystem.go`、`pkg/typesystem/validate.go`。

## 5.1 为什么需要 TypeSystem

解析得到的模型是一组类型定义，每个关系一棵重写树。引擎在求值时要不停地问模型问题：

- `document` 类型有没有 `viewer` 关系？它的重写树是什么？
- `document#viewer` 允许直接写入 `group#member` 吗？允许 `user:*` 吗？
- `document#parent` 是不是某个 `X from parent` 的 tupleset？
- 名为 `non_expired` 的条件是什么？

每次都去遍历类型列表太慢，所以 OpenFGA 在模型之上建了一个只读的索引结构 **TypeSystem**，模型的每个版本对应一个 TypeSystem，构建一次、缓存起来、到处共享（第 3 章的 `resolveTypesystem`）。

TypeSystem 的另一项职责是**守门**：写入模型时检查它是否“有意义”。语法正确但语义错误的模型，比如 `define viewer: editor` 而 `editor` 不存在，必须在写入时拒绝，而不是等到 Check 时才出错。

## 5.2 OpenFGA 的 TypeSystem：索引

构造函数 `newWithoutGraphs`（`pkg/typesystem/typesystem.go:195`）建了三张表：

```go
for _, td := range model.GetTypeDefinitions() {
	typeName := td.GetType()
	tds[typeName] = td
	...
	for relation, rewrite := range td.GetRelations() {
		r := &openfgav1.Relation{Name: relation, Rewrite: rewrite, TypeInfo: &openfgav1.RelationTypeInfo{}}
		if metadata, ok := td.GetMetadata().GetRelations()[relation]; ok {
			r.TypeInfo.DirectlyRelatedUserTypes = metadata.GetDirectlyRelatedUserTypes()
		}
		tdRelations[relation] = r
		ttuRelations[typeName][relation] = flattenUserset(rewrite)
	}
	relations[typeName] = tdRelations
}
```

- `typeDefinitions[type]`：类型定义；
- `relations[type][relation]`：把重写树和 `metadata` 里的类型限制合成一个 `Relation`；
- `ttuRelations[type][relation]`：这个关系的重写树里出现的所有 TTU，用于快速回答“某关系是不是 tupleset”。

条件也在这里包装成 `EvaluableCondition`，并设置了求值成本上限（第 14 章）。模型校验通过之后，`attachGraphs` 还会再建两张图：普通的关系图和带权图（第 27 章）。

常用的查询方法：

| 方法 | 行 | 回答的问题 |
|---|---|---|
| `GetRelation(type, rel)` | :374 | 关系定义；区分“类型不存在”和“关系不存在”两种错误 |
| `GetDirectlyRelatedUserTypes` | :416 | `[...]` 里的类型限制列表 |
| `DirectlyRelatedUsersets` | :426 | 类型限制中的 userset（`group#member`） |
| `IsDirectlyRelated(target, source)` | :460 | 某种用户（`user`、`group#member`）能否直接写在这个关系上 |
| `IsPubliclyAssignable(target, type)` | :798 | 能否写 `type:*` |
| `IsTuplesetRelation(type, rel)` | :1574 | 这个关系是不是某个 TTU 的 tupleset |
| `PathExists(user, rel, type)` | :751 | 从这种用户到这个关系，模型里有没有路径（第 27 章） |

`IsDirectlyRelated` 的比较规则由 `RelationEquals`（:441）定义：类型相同，并且“都没有关系也不是通配符”或“都是通配符”或“关系名相同”。也就是说 `user`、`user:*`、`user#x` 是三种不同的限制。

## 5.3 OpenFGA 的 TypeSystem：校验

`NewAndValidate`（:1170）的注释列出了规则，代码按顺序执行：

```go
func NewAndValidate(ctx context.Context, model *openfgav1.AuthorizationModel) (*TypeSystem, error) {
	t := newWithoutGraphs(model)
	schemaVersion := t.GetSchemaVersion()
	if !IsSchemaVersionSupported(schemaVersion) {
		return nil, ErrInvalidSchemaVersion
	}
	if containsDuplicateType(model) {
		return nil, ErrDuplicateTypes
	}
	if err := t.validateNames(); err != nil {
		return nil, err
	}
	...
	sort.Strings(typeNames)
	for _, typeName := range typeNames {
		...
		sort.Strings(relationNames)
		for _, relationName := range relationNames {
			err := t.validateRelation(typeName, relationName, relationMap)
			...
		}
	}
	if err := t.validateConditions(); err != nil {
		return nil, err
	}
	// All validation passed; now safe to build the graphs.
	if err := t.attachGraphs(model); err != nil {
		return nil, err
	}
	return t, nil
}
```

注意两处 `sort.Strings`：Go 的 map 遍历顺序是随机的，排序保证同一个坏模型每次报**同一个**错误。

每个关系的检查在 `validateRelation`（:1238）里，分四步：

```go
func (t *TypeSystem) validateRelation(typeName, relationName string, relationMap map[string]*openfgav1.Userset) error {
	rewrite := relationMap[relationName]
	err := t.isUsersetRewriteValid(typeName, relationName, rewrite)
	...
	err = t.validateTypeRestrictions(typeName, relationName)
	...
	hasEntrypoints, loop, err := hasEntrypoints(t.relations, typeName, relationName, rewrite, visitedRelations)
	...
	if !hasEntrypoints {
		cause := ErrNoEntrypoints
		if loop {
			cause = ErrNoEntryPointsLoop
		}
		return &InvalidRelationError{ObjectType: typeName, Relation: relationName, Cause: cause}
	}
	hasCycle, err := t.HasCycle(typeName, relationName)
	...
}
```

### 规则 1：重写引用的关系必须存在（`isUsersetRewriteValid`，:1327）

- computed：`define viewer: viewer` 不允许（自引用）；`editor` 必须是同一类型上的关系；
- TTU：`X from parent` 里，`parent` 必须存在，**而且必须是纯直接关系**（重写树就是一个 `This`）。`X` 必须在 `parent` 允许的**至少一个**类型上存在；
- 集合运算：递归检查每个子节点。

为什么 tupleset 必须是纯直接关系？因为 TTU 求值时是**读元组**来找父对象的（第 11 章）。如果允许 `define parent: [folder] or other_parent`，“document:1 的 parent 有哪些”就不再是一次读取能回答的问题。

### 规则 2：类型限制必须合法（`validateTypeRestrictions`，:1409）

- 有 `[...]` 的关系必须至少有一个类型，没有 `[...]` 的关系不能有类型限制；
- 引用的类型必须存在；`group#member` 里的 `member` 必须存在；
- **tupleset 关系不能允许 userset 或通配符**：`define parent: [folder#viewer]` 不合法。父对象必须是一个具体对象；
- 引用的条件必须在模型里定义（`$expression` 除外）。

### 规则 3：每个关系都要有入口点（`hasEntrypoints`，:981）

这是最不直观的一条。考虑：

```text
type group
  relations
    define member: [group#member]
```

语法和引用都没问题，但想一想：要让 `group:a#member@user:x` 成立，需要一条 `group:a#member@group:b#member`，再要求 `group:b#member@user:x` 成立……这条链永远到不了一个具体的用户。**没有任何元组能让这个关系为真**。这样的关系没有“入口点”。

`hasEntrypoints` 递归地问：这条重写规则有没有一条路径能落到一个具体类型上？

- `This`：类型限制里只要有一个具体类型（`user`）或通配符（`user:*`）就有入口；如果是 `group#member` 这种 userset，就递归去问 `group#member` 有没有入口；
- computed：问被引用的关系；
- TTU：问父类型上的那个关系，任意一个父类型有入口就行；
- union：任意一个子节点有入口；
- intersection、difference：**每个**子节点都要有入口（交集里有一项永远为空，整个交集就永远为空）。

为了不在环上无限递归，它带着一个 `visitedRelations` 表，记录“正在检查的关系”。如果顺着 computed 关系转了一圈回到正在检查的关系，就返回 `loop=true`，报错信息会从 `no entrypoints defined` 变成 `potential loop`：

```text
define a: b
define b: a
```

### 规则 4：计算关系不能成环（`hasCycle`，:1510）

`define a: b`、`define b: a` 已经被规则 3 拒绝了，但规则 3 只关心“有没有入口”。`define a: [user] or b`、`define b: a` 有入口（`[user]`），却依然构成一个只靠计算关系转圈的环：求值 `a` 要求值 `b`，求值 `b` 又要求值 `a`，和数据无关，一定死循环。`hasCycle` 只沿 **computed** 边查环（遇到 `This` 和 TTU 就停），因为经过元组的环（`member: [user, group#member]`）是合法的，它们取决于数据，在查询时处理（第 12 章）。

### 规则 5：条件必须能编译（`validateConditions`）

每个条件用 CEL 编译一次，表达式必须返回布尔值，参数类型必须合法（第 14 章）。

### 写入模型时的其他限制

校验由 `WriteAuthorizationModelCommand.Execute`（`pkg/server/commands/write_authzmodel.go:56`）调用，在此之前还有两条资源限制：最多 100 个类型（`MaxTypesPerAuthorizationModel`），序列化后不超过 256 KB（`DefaultMaxAuthorizationModelSizeInBytes`）。校验失败返回错误码 `invalid_authorization_model`（2056）。通过后才生成 ULID 作为模型 ID 并写入存储。

## 5.4 mini-openfga 的实现

### 5.4.1 索引

`pkg/typesystem/typesystem.go` 建了两张表：类型名到类型定义，以及每个类型里的所有 TTU。关系定义直接存在 `TypeDefinition.Relations` 里（第 4 章解析时已经把类型限制放进了 `Relation`），不需要再合并：

```go
func New(m *model.Model) *TypeSystem {
	ts := &TypeSystem{model: m, types: map[string]*model.TypeDefinition{}, ttus: map[string][]*model.Rewrite{}}
	for _, td := range m.Types {
		ts.types[td.Name] = td
		for _, name := range td.Order {
			td.Relations[name].Rewrite.Walk(func(r *model.Rewrite) {
				if r.Kind == model.TTU {
					ts.ttus[td.Name] = append(ts.ttus[td.Name], r)
				}
			})
		}
	}
	return ts
}
```

查询方法和 OpenFGA 同名或几乎同名：`Relation`、`DirectTypes`、`DirectlyRelatedUsersets`、`IsDirectlyRelated`、`IsPubliclyAssignable`、`IsTuplesetRelation`。`IsDirectlyRelated` 的参数从 protobuf 的 `RelationReference` 换成了字符串：

```go
// IsDirectlyRelated reports whether a user of shape (userType, userRelation)
// may be stored directly on objectType#relation. userRelation is "" for a
// concrete object; wildcards are handled by IsPubliclyAssignable.
func (t *TypeSystem) IsDirectlyRelated(objectType, relation, userType, userRelation string) bool {
	for _, ref := range t.DirectTypes(objectType, relation) {
		if ref.Type == userType && !ref.Wildcard && ref.Relation == userRelation {
			return true
		}
	}
	return false
}
```

和 OpenFGA 一样，TypeSystem 通过 `context` 传递：`typesystem.ContextWith(ctx, ts)` / `typesystem.FromContext(ctx)`。

### 5.4.2 校验

`pkg/typesystem/validate.go` 的 `NewAndValidate` 是对 OpenFGA 同名函数的逐条移植，同样对类型名和关系名排序后遍历。`hasEntrypoints` 连一个容易忽略的细节也照搬了：OpenFGA 用 `maps.Clone` 复制 `visited`，而这是**浅拷贝**，内层的 `map[string]bool` 在分支之间是共享的：

```go
// It is a direct port of OpenFGA's hasEntrypoints, including the shallow clone
// of the visited map (inner maps are shared between branches).
func (t *TypeSystem) hasEntrypoints(typeName, relName string, rw *model.Rewrite, visited map[string]map[string]bool) (bool, bool, error) {
	v := maps.Clone(visited)
	if inner, ok := v[typeName]; ok {
		inner[relName] = false
	} else {
		v[typeName] = map[string]bool{relName: false}
	}
	...
	switch rw.Kind {
	case model.This:
		for _, ref := range rel.DirectTypes {
			if ref.Relation == "" { // `user` or `user:*` is a concrete entrypoint
				v[typeName][relName] = true
				return true, false, nil
			}
			...
			if _, seen := v[ref.Type][ref.Relation]; seen {
				continue
			}
			ok, _, err := t.hasEntrypoints(ref.Type, ref.Relation, target.Rewrite, v)
			...
```

如果“改进”成深拷贝，某些模型的校验结果就会和 OpenFGA 不同。对一个授权引擎来说，**接受的模型集合和参照实现一致**比代码优雅更重要。

条件的编译通过一个函数参数注入：

```go
// ConditionCompiler compiles a condition; it is injected so that this package
// does not depend on CEL (conditions arrive in v0.3).
type ConditionCompiler func(*model.Condition) error

func NewAndValidate(m *model.Model, compile ConditionCompiler) (*TypeSystem, error)
```

v0.1 传 `nil`，v0.3 由 Server 传入 CEL 编译函数。

## 5.5 运行与验证：与真实 OpenFGA 逐条对照

`pkg/typesystem/validate_test.go` 列了 11 个模型。为了确认 mini 的判断和 OpenFGA 一致，在临时目录里写一个小程序，用 `replace` 指向本地的 OpenFGA 仓库，对同样的模型调用 `typesystem.NewAndValidate`：

```go
m := testutils.MustTransformDSLToProtoWithID("model\n  schema 1.1\n" + c.src)
_, err := typesystem.NewAndValidate(context.Background(), m)
fmt.Printf("%-28s %v\n", c.name, err)
```

OpenFGA 的输出：

```text
ok                           <nil>
self reference               the definition of relation 'viewer' in object type 'doc' is invalid: invalid userset rewrite definition
computed cycle               the definition of relation 'a' in object type 'doc' is invalid: potential loop
tupleset no userset          the relation type 'folder#viewer' on 'parent' in object type 'doc' is not valid
ttu relation missing         undefined relation: viewer does not appear as a relation in any of the directly related user types [type:"folder"]
no entrypoint userset loop   the definition of relation 'member' in object type 'group' is invalid: no entrypoints defined
recursive userset ok         <nil>
```

mini 对这 7 个模型的判断（接受或拒绝、拒绝的原因类别）与之完全一致。

> **踩坑记录**：写测试时，我先给 `define a: b` + `define b: a` 写的预期是 `no entrypoints`，测试失败，mini 实际报的是 `potential loop`。对照 OpenFGA 的输出后确认 `potential loop` 才是正确的，于是修改了测试的预期。这类“我以为”和“参照实现实际行为”的差异，只有对照运行才能发现。

```bash
cd code/mini-openfga    # v0.1 及之后
go test ./pkg/typesystem/ -run TestValidate -v
```

## 5.6 小结

- TypeSystem = 模型的只读索引 + 写入时的守门员，一个模型版本对应一个、可以长期缓存；
- 五条校验规则：引用存在且 tupleset 是纯直接关系；类型限制合法；每个关系有入口点；计算关系不成环；条件能编译；
- “入口点”保证每个关系都**有可能**为真；“计算关系不成环”保证求值不会在模型层面死循环；经过元组的环是合法的，留给查询时处理；
- 移植校验逻辑时要连细节（排序、浅拷贝）一起移植，并和参照实现对照运行。

## 5.7 练习

1. （巩固）判断下面的模型能否通过校验，并说明理由：
   ```text
   type user
   type doc
     relations
       define a: [user] and b
       define b: a
   ```
2. （扩展）mini 的 `NewAndValidate` 没有实现“最多 100 个类型”的限制，它在 v0.5 被放到了哪一层？为什么放在那里而不是 TypeSystem 里？
3. （读源码）OpenFGA 的 `validateTypeRestrictions` 里有一个分支返回 `EmptyRelationReferenceError`（:1409 之后），什么样的输入会触发它？为什么 DSL 写不出这样的模型，而 JSON 可以？

## 5.8 延伸阅读

- `pkg/typesystem/typesystem_test.go`：大量合法与非法模型的例子
- OpenFGA 文档 *Model validation*：入口点和环的官方说明
- `cmd/validatemodels`：批量校验存储里已有模型的工具（校验规则升级时用）
