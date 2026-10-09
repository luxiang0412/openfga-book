# 第 13 章 上下文元组：只在这一次请求里成立的关系

> 本章目标：理解上下文元组解决什么问题；读懂 OpenFGA 怎样用一个包装器把它们“混入”每一次存储读取；实现 mini 的 `CombinedTupleReader`；理解矩阵测试的 `_ctxTuples` 变体为什么能把测试用例数量翻倍。
> 前置：第 7、11 章。对应代码：mini-openfga **v0.2** `pkg/storage/wrappers.go`、`pkg/server/server.go`（Check 的校验部分）。

## 13.1 有些关系不该存进数据库

几个场景：

- 用户登录时，身份提供方（IdP）在 JWT 里给出了他所在的组：`groups: ["eng", "oncall"]`。这份信息以 IdP 为准、随时会变，每次登录都同步一遍到 OpenFGA 既麻烦又容易过期；
- “当前请求来自公司内网”“用户现在是值班状态”：这是请求时刻的事实，本来就不是持久的关系；
- 在写入之前**预演**一下：“如果我把 bob 加进 eng 组，他能看到哪些文档？”

OpenFGA 的答案是**上下文元组**（contextual tuples）：随请求一起发送的元组，只在这次请求中、就像已经写入了一样参与求值，请求结束即丢弃，不落库。

```bash
curl -X POST $F/stores/$STORE/check -d '{
  "tuple_key": {"user": "user:anne", "relation": "viewer", "object": "document:1"},
  "contextual_tuples": {"tuple_keys": [
    {"user": "user:anne", "relation": "member", "object": "group:eng"}
  ]}
}'
```

Check、BatchCheck、Expand、ListObjects、ListUsers 都支持上下文元组。每次请求最多 100 条（`openfga/api` 的 `ContextualTupleKeys.Validate`，`len(m.GetTupleKeys()) > 100` 时拒绝）。

## 13.2 实现方式：包一层读取器

引擎不需要知道上下文元组的存在。第 2 章看过，`NewRequestStorageWrapperWithCache` 在存储包装链的最外层套了一个 `CombinedTupleReader`（`pkg/storage/storagewrappers/combinedtuplereader.go:16`）：

```go
func NewCombinedTupleReader(ds storage.RelationshipTupleReader, contextualTuples []*openfgav1.TupleKey) *CombinedTupleReader {
	ctr := &CombinedTupleReader{RelationshipTupleReader: ds}
	cu := make([]*openfgav1.TupleKey, len(contextualTuples))
	for i, t := range contextualTuples {
		cu[i] = tuple.NewTupleKeyWithCondition(t.GetObject(), t.GetRelation(), t.GetUser(), t.GetCondition().GetName(), t.GetCondition().GetContext())
	}
	slices.SortFunc(cu, func(a *openfgav1.TupleKey, b *openfgav1.TupleKey) int {
		return strings.Compare(a.GetObject(), b.GetObject())
	})
	ctr.contextualTuplesOrderedByObjectID = cu
	return ctr
}
```

它嵌入了被包装的读取器，然后逐个改写读方法。以 `Read` 为例：

```go
func (c *CombinedTupleReader) Read(ctx context.Context, storeID string, filter storage.ReadFilter, options storage.ReadOptions) (storage.TupleIterator, error) {
	filteredTuples := filterTuples(c.contextualTuplesOrderedByObjectID, filter.Object, filter.Relation, []string{})
	iter1 := storage.NewStaticTupleIterator(filteredTuples)

	iter2, err := c.RelationshipTupleReader.Read(ctx, storeID, filter, options)
	if err != nil {
		return nil, err
	}
	return storage.NewCombinedIterator(iter1, iter2), nil
}
```

先返回匹配的上下文元组，再返回存储里的元组。`ReadUserTuple` 先在上下文元组里找，找到就直接返回；`ReadUsersetTuples` 用 `tupleMatchesAllowedUserTypeRestrictions` 筛出符合类型限制的上下文 userset；`ReadStartingWithUser` 同理。`ReadPage` 不混入上下文元组，因为 Read API 不接受上下文元组。

两个值得注意的地方：

1. **上下文元组和存储里的同一条元组可以同时存在**，读取时两条都会出现。对“有没有”的判断不影响结果，但如果两者的条件不同，就相当于这条关系有两种成立的方式。
2. **校验**。上下文元组相当于写入，所以要像写入一样严格校验（`ValidateTupleForWrite`，第 6 章）：类型限制、tupleset 规则、条件参数都要检查。在 Check 中，任何一条上下文元组不合法，返回 `invalid_tuple`（2027）。

## 13.3 一个特性，翻倍的测试

OpenFGA 的矩阵测试运行器有一个很巧妙的做法（`tests/check/check.go` 的 `runTest`）：每个测试跑两遍。第二遍名字后加 `_ctxTuples`，**不写入**阶段里的元组，而是把它们全部作为上下文元组随每个断言发送：

```go
for _, test := range allTestCases {
	runTest(t, test, params, false)
	runTest(t, test, params, true)
}
...
ctxTuples := testutils.Shuffle(assertion.ContextualTuples)
if contextTupleTest {
	ctxTuples = append(ctxTuples, stage.Tuples...)
}
```

两遍的期望结果完全相同。这条规则本身就是一个很强的正确性约束：**一条关系元组，写进存储还是随请求发送，对查询结果必须没有区别**。它同时测试了 `CombinedTupleReader` 的每一种读法和上下文元组的校验。（多阶段的测试和元组超过 100 条的阶段跳过第二遍，前者因为后面的阶段依赖前面写入的数据，后者因为超过了上下文元组的数量限制。）

mini 的矩阵运行器照搬了这个做法（`Options.Contextual`），v0.2 开始默认开启。v0.2 的 Check 断言总数从 516 变成 939，`tests: 235/298` 里的 298 就是 157 个可运行的测试加上 141 个 `_ctxTuples` 变体。

## 13.4 mini-openfga 的实现

`pkg/storage/wrappers.go`：

```go
// CombinedTupleReader overlays the contextual tuples of one request on top
// of a datastore: every read returns the matching contextual tuples first,
// then the stored ones. Contextual tuples are never persisted; they model
// facts known only at request time ("user is on the corporate network").
type CombinedTupleReader struct {
	TupleReader
	contextual []*tuple.Key
}

// NewCombinedTupleReader wraps ds; with no contextual tuples it returns ds.
func NewCombinedTupleReader(ds TupleReader, contextual []*tuple.Key) TupleReader {
	if len(contextual) == 0 {
		return ds
	}
	return &CombinedTupleReader{TupleReader: ds, contextual: contextual}
}

func (c *CombinedTupleReader) Read(ctx context.Context, store string, f ReadFilter) (Iterator, error) {
	inner, err := c.TupleReader.Read(ctx, store, f)
	if err != nil {
		return nil, err
	}
	return Concat(NewSliceIterator(c.filter(func(k *tuple.Key) bool { return MatchRead(k, f) })), inner), nil
}
```

没有上下文元组时直接返回原读取器，省一层包装。匹配逻辑复用第 7 章的 `MatchRead`、`MatchUsersetRef`，保证“上下文里的元组”和“存储里的元组”被同样的规则筛选。`Concat` 是一个串联迭代器。

`Server.Check` 里对上下文元组的校验与 OpenFGA 一致：

```go
// The queried tuple is validated loosely...
if err := validation.ValidateUserObjectRelation(ts, req.TupleKey); err != nil {
	return nil, fgaerr.Wrap(fgaerr.ValidationError, err)
}
// ...but contextual tuples as strictly as a Write.
contextual := condition.NormalizeTuples(req.ContextualTuples)
for _, k := range contextual {
	if err := validation.ValidateTupleForWrite(ts, k); err != nil {
		return nil, fgaerr.Wrap(fgaerr.InvalidTuple, err)
	}
}
```

（`NormalizeTuples` 是 v0.3 加的，用来把条件参数统一成 JSON 的数据形态，第 14 章。它最初的版本会就地修改调用方传入的元组，被 `-race` 抓到了一个数据竞争，也在第 14 章讲。）

## 13.5 运行与验证

```go
func TestContextualTuples(t *testing.T) {
	e := newEnv(t, groupsModel, keys("document:1#viewer@group:eng#member"))
	k := tuple.New("document:1", "viewer", "user:alice")
	resp, _, _ := e.check(k)
	if resp.Allowed {
		t.Fatal("alice is not in eng yet")
	}
	resp, _, _ = e.check(k, keys("group:eng#member@user:alice")...)
	if !resp.Allowed {
		t.Fatal("contextual tuple should make alice a member")
	}
	resp, _, _ = e.check(k)
	if resp.Allowed {
		t.Fatal("contextual tuples must not be persisted")
	}
}
```

三步：没有上下文元组时为假；带上 alice 的组成员关系后为真；再查一次又为假，说明没有落库。

```bash
cd code/mini-openfga && git checkout v0.2
go test ./pkg/check/ -run TestContextualTuples -v
go run ./cmd/minifga matrix                # 默认包含 _ctxTuples 变体
go run ./cmd/minifga matrix -contextual=false
```

```text
Check assertions: 720 passed, 219 failed, 0 skipped (76.7%); tests: 235/298 fully passed
Check assertions: 402 passed, 114 failed, 0 skipped (77.9%); tests: 124/157 fully passed
```

## 13.6 小结

- 上下文元组是随请求发送、只在本次求值中成立的元组，适合 IdP 给出的组、请求时刻的事实、写入前的预演；
- 实现方式是在存储读取器外面包一层，把匹配的上下文元组混进每一种读法的结果里，引擎无需改动；
- 它们按写入的标准校验，在 Check 中不合法时报 2027；
- “写入还是随请求发送，结果必须一样”是一条很强的测试原则，矩阵测试用它把用例数翻了一倍。

## 13.7 练习

1. （巩固）用上下文元组实现“值班工程师可以查看生产日志”：模型和请求各是什么？
2. （扩展）第 15 章的检查缓存以子问题为键。如果两个请求的上下文元组不同，同一个子问题的结果可能不同。缓存键里应该包含什么？去 OpenFGA 的 `pkg/storage/cache.go` 找 `InvariantCacheKey` 验证你的答案。
3. （读源码）`CombinedTupleReader` 的构造函数把上下文元组按 object 排序，但 `filterTuples` 是线性扫描的，排序有什么用？（提示：看 `ReadStartingWithUser` 的实现和它的选项 `WithResultsSortedAscending`。）

## 13.8 延伸阅读

- OpenFGA 文档 *Contextual and Time-Based Authorization*
- `tests/check/check.go` 的 `runTest`
- `pkg/storage/storagewrappers/combinedtuplereader_test.go`
