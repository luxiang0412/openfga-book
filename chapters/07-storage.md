# 第 7 章 存储接口：引擎只需要四种读法

> 本章目标：理解 OpenFGA 的存储接口为什么是现在这个样子：查询引擎只用到四种访问模式；理解迭代器和“读出来再过滤”的设计；实现 mini-openfga 的存储接口与内存实现。
> 前置：第 6 章。对应代码：mini-openfga **v0.1** `pkg/storage/storage.go`、`pkg/storage/memory/memory.go`（v0.5 扩展成完整的 `OpenFGADatastore`，见第 21 章）。

## 7.1 从查询反推接口

存储接口不是凭空设计的，而是从引擎的需要反推出来的。回顾第 3 章，求值 `document:design#viewer@user:ceo` 时引擎问了存储这样几类问题：

| 引擎的问题 | 对应的重写 | 读法 |
|---|---|---|
| `document:design#viewer@user:ceo` 这条元组存在吗？ | `[user]` | 按完整的键精确查找 |
| `document:design#viewer` 上有哪些 userset 或通配符？ | `[group#member, user:*]` | 按 object + relation 查，user 限定为 userset/通配符 |
| `document:design#parent` 指向哪些对象？ | `viewer from parent` | 按 object + relation 查 |
| `user:anne` 出现在哪些 `document#viewer` 元组里？ | ListObjects（第 18 章） | 按 user 反查 |

四种问题，正好对应 OpenFGA `RelationshipTupleReader`（`pkg/storage/storage.go:152`）的方法：

```go
type RelationshipTupleReader interface {
	Read(ctx context.Context, store string, filter ReadFilter, options ReadOptions) (TupleIterator, error)
	ReadPage(ctx context.Context, store string, filter ReadFilter, options ReadPageOptions) ([]*openfgav1.Tuple, string, error)
	ReadUserTuple(ctx context.Context, store string, filter ReadUserTupleFilter, options ReadUserTupleOptions) (*openfgav1.Tuple, error)
	ReadUsersetTuples(ctx context.Context, store string, filter ReadUsersetTuplesFilter, options ReadUsersetTuplesOptions) (TupleIterator, error)
	ReadStartingWithUser(ctx context.Context, store string, filter ReadStartingWithUserFilter, options ReadStartingWithUserOptions) (TupleIterator, error)
}
```

（`ReadPage` 是 `Read` 的分页版本，只给 Read API 用。）源码里的注释把每个方法的语义写得很清楚，例如 `ReadUsersetTuples`：

```go
// ReadUsersetTuples returns all userset tuples for a specified object and relation.
// For example, given the following relationship tuples:
//	document:doc1, viewer, user:*
//	document:doc1, viewer, group:eng#member
// and the filter
//	object=document:1, relation=viewer, allowedTypesForUser=[group#member]
// this method would return the tuple (document:doc1, viewer, group:eng#member)
// If allowedTypesForUser is empty, both tuples would be returned.
```

为什么不是一个通用的 `Query(filter)`？因为每种读法在 SQL 里对应**不同的索引**：正向查找用主键 `(store, object_type, object_id, relation, user...)`，反查用 `(store, object_type, relation, user...)`（第 21 章）。把读法写进接口，就等于把“需要哪些索引”写进了存储的契约里。

每个方法都带一个 `options` 参数，里面最重要的是 `Consistency`：`HIGHER_CONSISTENCY` 要求读最新数据，存储包装器据此绕过迭代器缓存（第 23 章）。

## 7.2 迭代器：为什么不直接返回切片

除了 `ReadUserTuple` 只返回一条，其他读法都返回迭代器（`pkg/storage/tuple_iterators.go:21`）：

```go
type Iterator[T any] interface {
	// Next will return the next available item or ErrIteratorDone.
	Next(ctx context.Context) (T, error)
	// Stop terminates iteration. Any subsequent calls to Next must return ErrIteratorDone.
	Stop()
	// Head will return the first item or ErrIteratorDone
	Head(ctx context.Context) (T, error)
}
```

原因有两个：

1. **结果可能很大**。一个热门文件夹的 viewer 可能有上万个 userset，全部读进内存再处理既浪费又慢。SQL 实现按批次（默认 50 行一页）从数据库取，消费者处理一条再取一条。
2. **可以提前结束**。union 找到一条为真的路径后要取消其他分支（第 10 章），这时迭代器调用 `Stop()` 释放数据库连接，剩下的行根本不会被读出来。

在迭代器上可以套迭代器：`NewFilteredTupleKeyIterator`（:319）丢掉不合法的元组（第 6 章的 `FilterInvalidTuples`），`NewConditionsFilteredTupleKeyIterator`（:436）丢掉条件不满足的元组，`NewCombinedIterator`（:126）把上下文元组和存储里的元组拼起来（第 13 章）。引擎看到的始终是一个迭代器。

## 7.3 一个细节：条件求值失败了怎么办

`ConditionsFilteredTupleKeyIterator` 的 `Next`（:348）值得逐行读：

```go
func (f *ConditionsFilteredTupleKeyIterator) Next(ctx context.Context) (*openfgav1.TupleKey, error) {
	for {
		tuple, err := f.iter.Next(ctx)
		if err != nil {
			if errors.Is(err, ErrIteratorDone) {
				if f.onceValid || f.lastError == nil {
					return nil, ErrIteratorDone
				}
				lastError := f.lastError
				f.lastError = nil
				return nil, lastError
			}
			return nil, err
		}

		valid, err := f.filter(tuple)
		if err != nil {
			var fatal *interrors.FatalError
			if errors.As(err, &fatal) {
				return nil, err // propagate immediately; keep wrapper intact
			}
			f.lastError = err
			continue
		}
		if !valid {
			continue
		}
		f.onceValid = true
		return tuple, nil
	}
}
```

如果某条元组的条件无法求值（比如请求没带条件需要的参数），这个迭代器**不会立刻报错**，而是记下错误、跳过这条、继续往后读。到迭代结束时：

- 如果读的过程中**至少有一条元组是可用的**（`onceValid`），刚才的错误就被丢弃，迭代正常结束；
- 只有一条可用的都没有时，才把最后一个错误报出去；
- 内联表达式（`$expression`）的错误被包成 `FatalError`，立刻报出（第 14 章）。

这个策略的含义是：同一次读取里，一条因缺少参数而无法判断的元组，只要旁边还有能用的元组，就被**当作不存在**。

> **踩坑记录（写本章时发现）**：mini-openfga 的 `FilteredIterator` 从 v0.2 起就有类似的“记下错误、继续读、最后再报”的逻辑，注释里还写着“与 OpenFGA 的 ConditionsFilteredTupleKeyIterator 相同”。但它漏掉了 `onceValid` 这个条件：只要有错误，结束时一定报出。2310 条矩阵断言没有一条覆盖这个差别，是写这一章逐行对照源码时才看出来的。
>
> 用一个最小的例子在真实的 OpenFGA 上验证：`document:1#viewer` 有两个 userset，`group:a#member`（无条件）和 `group:b#member`（带条件 `xcond(x)`），anne 不在任何组里，请求不带 context：
>
> ```text
> OpenFGA:  {"allowed":false,"resolution":""}
> mini:     {"code":"validation_error","message":"'xcond' - tuple is missing context parameters '[x]'"}
> ```
>
> 修复（mini-openfga v1.0 的最后一个提交）：`FilteredIterator` 增加 `onceValid`，新增 `storage.FatalError`，内联表达式的错误包成 fatal；Explain 接口按同样的规则处理；回归测试 `TestConditionErrorIgnoredWhenAnotherTupleIsUsable`。为了防止类似的问题，第 31 章的差分测试又加了一个“带条件和不同 context”的场景，它随后又找出了 ListUsers 的一个差异。

## 7.4 写入

`RelationshipTupleWriter.Write`（:273 之后）的契约写在注释里：

```go
// Write updates data in the tuple backend, performing all delete operations in
// `deletes` before adding new values in `writes`.
// It must also write to the changelog.
// If two concurrent requests attempt to write the same tuple at the same time, it must return ErrTransactionalWriteFailed.
// If the tuple to be written already existed or the tuple to be deleted didn't exist, it must return InvalidWriteInputError.
Write(ctx context.Context, store string, d Deletes, w Writes, opts ...TupleWriteOption) error
```

- **先删后写**，整体是一个事务，要么全部成功要么全不生效；
- **同时写 changelog**（第 23 章的 ReadChanges 和缓存失效都依赖它）；
- 写一条已存在的元组、删一条不存在的元组都是错误。新版本通过 `TupleWriteOptions` 允许改成忽略（`OnDuplicateInsertIgnore`、`OnMissingDeleteIgnore`），对应 Write API 的 `on_duplicate`、`on_missing` 参数。

## 7.5 OpenFGA 的内存实现

`pkg/storage/memory/memory.go` 是开发和测试用的实现，结构很朴素（:129）：

```go
type MemoryBackend struct {
	// map: store => set of tuples
	tuples      map[string][]*storage.TupleRecord // GUARDED_BY(mutexTuples).
	mutexTuples sync.RWMutex
	// map: store => set of changes
	changes map[string][]*tupleChangeRec // GUARDED_BY(mutexTuples).
	// map: store = > map: type definition id => type definition
	authorizationModels map[string]map[string]*AuthorizationModelEntry
	...
}
```

每个 store 一个切片，所有读法都是线性扫描。它的价值不在性能，而在**语义简单**：它是其他实现的参照。OpenFGA 的 `pkg/storage/test/storage.go` 是一套对所有实现都要跑的测试，内存实现首先通过它们。

## 7.6 mini-openfga 的实现

### 7.6.1 接口

`pkg/storage/storage.go` 只保留引擎需要的四种读法和一个写法：

```go
// TupleReader is everything the query engine needs from storage.
type TupleReader interface {
	// Read: TTU (`viewer from parent` reads document:1#parent@*) and the Read API.
	Read(ctx context.Context, store string, f ReadFilter) (Iterator, error)
	// ReadUserTuple: is exactly `object#relation@user` stored? (direct check)
	ReadUserTuple(ctx context.Context, store, object, relation, user string) (*tuple.Key, error)
	// ReadUsersetTuples: usersets/wildcards assigned to object#relation.
	ReadUsersetTuples(ctx context.Context, store string, f ReadUsersetTuplesFilter) (Iterator, error)
	// ReadStartingWithUser: reverse lookup for ListObjects.
	ReadStartingWithUser(ctx context.Context, store string, f ReadStartingWithUserFilter) (Iterator, error)
}

// TupleWriter applies deletes then writes atomically.
type TupleWriter interface {
	Write(ctx context.Context, store string, deletes, writes []*tuple.Key) error
}
```

去掉了 `options` 参数：mini 的内存实现和 SQLite 实现都只有一份数据，“一致性偏好”只影响上层缓存，不需要传到存储。迭代器也简化成两个方法 `Next` 和 `Stop`。

`ReadFilter.Object` 可以只写类型（`document:`），这是 Read API 的约定：“列出 anne 在所有 document 上的元组”。

两个共享的匹配函数 `MatchRead` 和 `MatchUsersetRef` 放在接口包里，内存实现和组合读取器（第 13 章）都用它们，保证语义一致。

### 7.6.2 内存实现

和 OpenFGA 的线性扫描不同，mini 按 `object#relation` 建了一层索引，因为 Check 最常用的三种读法（精确查找、读 userset、读 tupleset）都指定了 object 和 relation：

```go
type storeData struct {
	// byObjectRelation indexes tuples by `object#relation`: the access path of
	// both direct checks and TTU reads.
	byObjectRelation map[string][]*storedTuple
	changes          []*storage.Change
	...
}
```

读的时候在读锁下把匹配的元组**复制**出来（`snapshot`），再包成一个切片迭代器返回。复制保证迭代器不会看到之后并发写入造成的变化，也不会和写操作争用锁。

写入严格遵守“全部检查完再修改”：

```go
// validate everything before mutating, so a failed write changes nothing
for _, k := range deletes {
	if find(k) < 0 {
		return storage.ErrTupleNotExists
	}
}
for _, k := range writes {
	deleted := slices.ContainsFunc(deletes, func(del *tuple.Key) bool { return del.String() == k.String() })
	if find(k) >= 0 && !deleted {
		return storage.ErrTupleExists
	}
}
// ... 然后才删除、追加、记 changelog
```

“同一次请求里先删后写同一条元组”是允许的，用来修改一条元组的条件。

### 7.6.3 过滤迭代器

`FilteredIterator` 是 7.3 节那个策略的移植（v1.0 修复后的版本）：

```go
func (f *FilteredIterator) Next(ctx context.Context) (*tuple.Key, error) {
	for {
		k, err := f.inner.Next(ctx)
		if err != nil {
			if errors.Is(err, ErrIteratorDone) {
				if f.onceValid || f.lastErr == nil {
					return nil, ErrIteratorDone
				}
				e := f.lastErr
				f.lastErr = nil
				return nil, e
			}
			return nil, err
		}
		ok, err := f.keep(k)
		if err != nil {
			var fatal *FatalError
			if errors.As(err, &fatal) {
				return nil, err
			}
			f.lastErr = err
			continue
		}
		if ok {
			f.onceValid = true
			return k, nil
		}
	}
}
```

OpenFGA 用两个迭代器分别做“合法性过滤”和“条件过滤”，mini 合成一个：`keep` 函数里先 `IsValidForRead`，再求条件。

## 7.7 运行与验证

v0.1 时存储只有内存实现，测试通过 `pkg/check` 的测试间接覆盖。v0.5 引入了一致性测试套件 `pkg/storage/storagetest`（第 21 章），内存实现和 SQLite 实现跑同一套测试：

```bash
cd code/mini-openfga
go test ./pkg/storage/...
```

```text
ok  	github.com/luxiang0412/mini-openfga/pkg/storage/memory	0.002s
ok  	github.com/luxiang0412/mini-openfga/pkg/storage/sqlite	0.047s
```

7.3 节的回归测试：

```bash
go test ./pkg/server/ -run TestConditionErrorIgnoredWhenAnotherTupleIsUsable -v
```

## 7.8 小结

- 存储接口由查询的需要决定：精确查找、读 userset、按 object+relation 读、按 user 反查；每种读法对应一个索引；
- 迭代器让大结果集可以流式处理、提前结束；过滤以迭代器套迭代器的方式叠加；
- 条件无法求值的元组，在同一次读取里有其他可用元组时被当作不存在；这条规则是读源码时发现、再在真实 OpenFGA 上确认的；
- 写入是“先删后写”的事务，并同步写 changelog。

## 7.9 练习

1. （巩固）为下面的每次读取写出 mini 的 `ReadFilter` / `ReadUsersetTuplesFilter`：求值 `document:1#viewer@user:anne`，模型 `define viewer: [user, group#member] or viewer from parent`。
2. （扩展）mini 的内存实现在 `snapshot` 里复制元组。如果不复制、直接返回内部切片的元素，会出现什么问题？写一个 `-race` 能检测到的测试。
3. （读源码）OpenFGA 的 `staticIterator`（`pkg/storage/memory/memory.go:27`）为什么还需要实现 `Head` 方法？在 `internal/graph` 里找一个用到 `Head` 的地方。

## 7.10 延伸阅读

- `pkg/storage/storage.go` 中各接口的完整注释
- `pkg/storage/test/storage.go`：OpenFGA 的存储一致性测试
- `pkg/storage/storagewrappers/`：读取路径上的各层包装器
