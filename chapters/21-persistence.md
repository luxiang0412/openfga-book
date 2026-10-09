# 第 21 章 持久化：表结构、索引与一个慢查询

> 本章目标：读懂 OpenFGA 的 SQL 表结构，理解为什么元组要拆成 object 和 user 两半、各配一个索引；看 SQLite 实现的写入事务和繁忙重试；实现 mini 的 `OpenFGADatastore` 接口、SQLite 后端和一致性测试套件；用 `EXPLAIN QUERY PLAN` 找出并修复一个让读取变慢 200 倍的问题。
> 前置：第 7 章。对应代码：mini-openfga **v0.5** `pkg/storage/datastore.go`、`pkg/storage/sqlite/sqlite.go`、`pkg/storage/storagetest/`（v1.0 修复了查询计划）。

## 21.1 从四种读法到一个完整的存储接口

第 7 章的 `TupleReader` 只够引擎用。一个服务还要管理 store、模型、断言、变更日志。OpenFGA 的 `OpenFGADatastore`（`pkg/storage/storage.go:409`）由几个小接口组成：

```go
type OpenFGADatastore interface {
	TupleBackend                 // 读写元组
	AuthorizationModelBackend    // 写模型、按 ID 读、列出、找最新
	StoresBackend                // 建、查、列、删 store
	AssertionsBackend            // 读写断言（第 23 章）
	ChangelogBackend             // 读变更日志（第 23 章）
	IsReady(ctx context.Context) (ReadinessStatus, error)
	Close()
}
```

mini v0.5 定义了对应的接口（`pkg/storage/datastore.go`），把分页统一成一个 `Page{Size, Token}`：

```go
type OpenFGADatastore interface {
	TupleReader
	TupleWriter
	ReadPage(ctx context.Context, store string, f ReadFilter, p Page) ([]*Tuple, string, error)
	ReadChanges(ctx context.Context, store string, f ReadChangesFilter, p Page, desc bool) ([]*Change, string, error)
	CreateStore(ctx context.Context, s *Store) error
	GetStore(ctx context.Context, id string) (*Store, error)
	ListStores(ctx context.Context, p Page) ([]*Store, string, error)
	DeleteStore(ctx context.Context, id string) error
	WriteAuthorizationModel(ctx context.Context, store string, m *model.Model) error
	ReadAuthorizationModel(ctx context.Context, store, id string) (*model.Model, error)
	ReadAuthorizationModels(ctx context.Context, store string, p Page) ([]*model.Model, string, error)
	FindLatestAuthorizationModel(ctx context.Context, store string) (*model.Model, error)
	WriteAssertions(ctx context.Context, store, modelID string, a []*Assertion) error
	ReadAssertions(ctx context.Context, store, modelID string) ([]*Assertion, error)
	Close()
}
```

分页令牌（continuation token）对调用方是不透明的：mini 内部存的是一个偏移量或最后一个 ULID，对外做 base64 编码。

## 21.2 表结构

OpenFGA 的 SQLite 迁移（`assets/migrations/sqlite/005_initialize_schema.sql`）建了五张表，核心是 `tuple`：

```sql
CREATE TABLE tuple (
    store CHAR(26) NOT NULL,
    object_type VARCHAR(128) NOT NULL,
    object_id VARCHAR(128) NOT NULL,
    relation VARCHAR(50) NOT NULL,
    user_object_type VARCHAR(128) NOT NULL,
    user_object_id VARCHAR(128) NOT NULL,
    user_relation VARCHAR(50) NOT NULL,
    user_type VARCHAR(7) NOT NULL,
    ulid CHAR(26) NOT NULL,
    inserted_at TIMESTAMP NOT NULL,
    condition_name VARCHAR(256),
    condition_context LONGBLOB,
    PRIMARY KEY (store, object_type, object_id, relation, user_object_type, user_object_id, user_relation)
);
CREATE UNIQUE INDEX idx_tuple_ulid ON tuple (ulid);
CREATE INDEX idx_reverse_lookup_user ON tuple (store, object_type, relation, user_object_type, user_object_id, user_relation);
```

几个设计点：

- **元组拆成两半**：`object` 拆成 `object_type` + `object_id`，`user` 拆成 `user_object_type` + `user_object_id` + `user_relation`。这样才能按类型查（“所有 document”）、按 userset 的形态查（“类型是 group、关系是 member 的 user”）；
- **两个方向两个索引**。主键的列顺序是 `object → relation → user`，服务正向查询（Check 的三种读法，`Read(object#relation)`）；`idx_reverse_lookup_user` 的列顺序是 `object_type → relation → user`，服务反向查询（ListObjects 的 `ReadStartingWithUser`）。第 7 章说“读法写进接口，就等于把需要的索引写进了契约”，指的就是这里；
- **`user_type`**：`user` 或 `userset`，配合两个部分索引加速“只读具体用户”和“只读 userset”；
- **`ulid`**：每条元组一个全局唯一、按时间递增的 ID，用于分页和变更日志的排序；
- **条件**：`condition_name` 和序列化后的 `condition_context`；
- 另外四张表：`store`（软删除，`deleted_at`）、`authorization_model`（protobuf 序列化后的模型）、`assertion`、`changelog`（结构和 tuple 相同，外加 `operation`）。

Postgres 和 MySQL 的表结构相同，只是类型和索引语法不同。

## 21.3 SQLite 实现

`pkg/storage/sqlite/sqlite.go`。打开数据库时（`PrepareDSN`，:61）设置三个参数：

```go
if !foundJournalMode {
	query.Add("_pragma", "journal_mode(WAL)")
}
if !foundBusyTimeout {
	query.Add("_pragma", "busy_timeout(100)")
}
if !query.Has("_txlock") {
	query.Set("_txlock", "immediate")
}
```

- WAL 模式：读不阻塞写、写不阻塞读；
- `busy_timeout`：数据库被锁时等待的毫秒数；
- `_txlock=immediate`：事务一开始就拿写锁，避免两个事务都先读后写、到写时才发现冲突。

即使这样，SQLite 仍可能返回 `SQLITE_BUSY`，所以写入用 `busyRetry`（:1369）包了一层，最多重试 10 次。

写入（`write`，:394）在一个事务里：先按“锁键”排序并分批 `SELECT` 出已存在的行（排序保证多个并发事务以相同的顺序加锁，避免死锁），然后检查“写已存在的元组”“删不存在的元组”（可以按选项忽略），删除、插入、写 changelog，最后提交。

读取用 Squirrel 构造 SQL，结果通过一个按页取数的迭代器（`SQLTupleIterator`）流式返回。反向查询（`ReadStartingWithUser`，:820）：

```go
var targetUsersArg sq.Or
for _, u := range filter.UserFilter {
	userObjectType, userObjectID, userRelation := tupleUtils.ToUserPartsFromObjectRelation(u)
	targetUser := sq.Eq{"user_object_type": userObjectType, "user_object_id": userObjectID}
	if userRelation != "" {
		targetUser["user_relation"] = userRelation
	}
	targetUsersArg = append(targetUsersArg, targetUser)
}
builder := s.stbl.Select(...).From("tuple").
	Where(sq.Eq{"store": store, "object_type": filter.ObjectType, "relation": filter.Relation}).
	Where(targetUsersArg).OrderBy("object_id")
```

## 21.4 mini-openfga 的 SQLite 后端

`pkg/storage/sqlite/sqlite.go` 用纯 Go 的 `modernc.org/sqlite`（和 OpenFGA 一样，不需要 cgo），表结构照搬 OpenFGA，模型存成 JSON 而不是 protobuf。几个取舍：

- **单连接**：`db.SetMaxOpenConns(1)`。SQLite 同一时刻只能有一个写者，一个连接让 mini 自己的 goroutine 之间不会互相 `SQLITE_BUSY`，代价是读也被串行化。对教学和单机场景足够；
- **读取一次取完**，不流式；
- **写入依赖唯一约束**：直接 `INSERT`，遇到 `UNIQUE constraint failed` 就返回 `ErrTupleExists`；删除检查 `RowsAffected`；
- **changelog 和元组在同一个事务里写入**，保证读 changelog 的人不会错过任何已提交的变更（第 23 章依赖这一点）。

### 21.4.1 一致性测试套件

两个后端必须有完全相同的语义。`pkg/storage/storagetest` 是一套对所有后端都要跑的测试（对应 OpenFGA 的 `pkg/storage/test/storage.go`），9 组：元组的读写与原子性、userset 读取、反向查询、分页读、changelog、store、模型、断言、条件的往返：

```go
func TestMemory(t *testing.T) {
	storagetest.RunAll(t, func(*testing.T) storage.OpenFGADatastore { return New() })
}

func TestSQLite(t *testing.T) {
	storagetest.RunAll(t, func(t *testing.T) storage.OpenFGADatastore {
		ds, err := New(filepath.Join(t.TempDir(), "fga.db"))
		...
		return ds
	})
}
```

其中原子性的测试：

```go
// a failed write is atomic: the valid delete in the same call is not applied
if err := ds.Write(ctx, "s", []*tuple.Key{k("document:1#viewer@user:bob"), k("document:9#viewer@user:nobody")}, nil); !errors.Is(err, storage.ErrTupleNotExists) {
	t.Fatalf("delete missing: want ErrTupleNotExists, got %v", err)
}
if _, err := ds.ReadUserTuple(ctx, "s", "document:1", "viewer", "user:bob"); err != nil {
	t.Fatal("bob must survive the failed write")
}
```

两个后端第一次运行就都通过了。更强的检验是把整个矩阵测试放在 SQLite 上跑，而且走 HTTP（第 24 章）：`TestMatrixHTTPSQLite`，2310 条断言全部通过。

## 21.5 踩坑：一个让读取慢 200 倍的 `ORDER BY`

> 这是写本章时，为了展示“两个方向用两个索引”而去查询计划时发现的。功能测试全部通过，问题只在于慢。

最初的实现在每个读取查询后面加了 `ORDER BY ulid`，想让结果顺序稳定。用 `EXPLAIN QUERY PLAN` 看三种读取：

```text
== forward (Check: ReadUserTuple)
   SEARCH tuple USING INDEX sqlite_autoindex_tuple_1 (store=? AND object_type=? AND object_id=? AND relation=? AND user_object_type=? AND user_object_id=? AND user_relation=?)
== forward (TTU: Read object#relation)
   SEARCH tuple USING INDEX idx_store_ulid (store=?)
== reverse (ListObjects: ReadStartingWithUser)
   SEARCH tuple USING INDEX idx_store_ulid (store=?)
```

精确查找没有 `ORDER BY`，用了主键。另外两个查询因为要按 `ulid` 排序，SQLite 选了 `(store, ulid)` 这个索引：它能直接给出排好序的结果，但只能按 `store` 过滤，等于**扫描整个 store** 再逐行判断其他条件。小数据量下看不出来，SQLite 也没有统计信息（没有运行 `ANALYZE`）来做出更好的选择。

去掉 `ORDER BY` 之后：

```text
... object_id = '1' AND relation = 'parent'
   [SEARCH tuple USING INDEX sqlite_autoindex_tuple_1 (store=? AND object_type=? AND object_id=? AND relation=?)]
... object_id = '*' AND user_relation = ''))
   [SEARCH tuple USING INDEX idx_reverse_lookup_user (store=? AND object_type=? AND relation=? AND user_object_type=?)]
```

正向读取用上了主键；反向查询用上了 `idx_reverse_lookup_user`，但只用到前四列，`user_object_id` 和 `user_relation` 没用上：原因是多个用户之间的 `OR`，SQLite 没能把它变成多次索引查找，于是对 `user_object_type = 'user'` 的所有行逐行判断。把条件改写成**行值 IN**：

```go
// A row-value IN lets SQLite seek idx_reverse_lookup_user on all of its
// columns for each user; with an OR of ANDs it only used the first four
// and scanned every tuple of that user type (book ch. 21).
q += " AND (user_object_type, user_object_id, user_relation) IN (VALUES " + strings.Join(rows, ",") + ")"
```

在一个有 2 万份文档、4 万条元组的 store 上测（`pkg/storage/sqlite/bench_test.go`，每个 200 次）：

| 读取 | 修复前 | 去掉 ORDER BY | 再改成行值 IN |
|---|---|---|---|
| `Read(object#relation)`（TTU 每一跳） | 5,220,052 ns/op | 24,897 ns/op | 22,926 ns/op |
| `ReadStartingWithUser`（ListObjects 每一跳） | 7,102,506 ns/op | 2,732,045 ns/op | 206,637 ns/op |

正向读取快了约 200 倍，反向查询快了约 34 倍。存储接口本来就不保证顺序（第 7 章），去掉排序不影响任何语义，只有分页的 `ReadPage` 需要按 `ulid` 排序，它保留了。

经验：

- **功能测试不会发现查询计划的问题**，要么写基准测试，要么直接看 `EXPLAIN`；
- 给查询加 `ORDER BY` 不是“免费的整洁”，它可能改变优化器的索引选择；
- OpenFGA 的 SQLite 实现对反向查询用的是 `ORDER BY object_id`（按对象排序是有意义的，第 13 章的有序合并迭代器依赖它），和主键的第二列一致。

## 21.6 运行与验证

```bash
cd code/mini-openfga && git checkout v1.0
go test ./pkg/storage/...
go test -run xxx -bench . -benchtime 200x ./pkg/storage/sqlite/
go test ./internal/matrix/ -run TestMatrixHTTPSQLite -v
```

## 21.7 小结

- 存储接口由几个后端接口组成：元组、模型、store、断言、变更日志；
- 元组表把 object 和 user 拆开存，主键服务正向查询，反向索引服务 ListObjects；
- SQLite 用 WAL、繁忙等待、立即锁事务，并对 `SQLITE_BUSY` 重试；
- 同一套一致性测试跑在所有后端上，再用矩阵测试从上层验证；
- 查询计划要亲眼看：一个 `ORDER BY` 让读取慢了 200 倍。

## 21.8 练习

1. （巩固）为什么 changelog 必须和元组写在同一个事务里？如果先提交元组、再单独写 changelog，第 23 章的缓存失效会出现什么问题？
2. （扩展）给 mini 加一个 Postgres 后端：哪些 SQL 需要改？`storagetest.RunAll` 能不能原样复用？
3. （读源码）OpenFGA SQLite 的 `write`（:394）在查询已存在的行时按 `storage.DefaultMaxTuplesPerWrite` 分批，并且先对锁键排序（`makeTupleLockKeys`，:270）。分别说明两者的目的。

## 21.9 延伸阅读

- `assets/migrations/{postgres,mysql,sqlite}/`：三种数据库的迁移历史
- `pkg/storage/sqlcommon/`：三种 SQL 后端共用的代码
- SQLite 文档：*Query Planning*、*Write-Ahead Logging*、*Row Values*
