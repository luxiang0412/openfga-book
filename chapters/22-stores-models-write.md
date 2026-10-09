# 第 22 章 Store、模型版本与 Write

> 本章目标：理解 store 作为隔离单元、模型不可变且按 ULID 排序的版本管理；读懂 Write 的校验顺序、限制与事务语义；实现 mini 的对应部分，并和 OpenFGA 逐条对照错误码。
> 前置：第 5、6、21 章。对应代码：mini-openfga **v0.5** `pkg/server/stores.go`、`models.go`、`write.go`。

## 22.1 Store：隔离单元

Store 是 OpenFGA 里最外层的容器：一组模型版本 + 一组元组 + 断言 + 变更日志。不同的 store 之间完全隔离，常见用法：

- 每个环境一个（dev、staging、prod）；
- 多租户 SaaS 里每个租户一个；
- 测试里每个用例一个（矩阵测试就是这样做的）。

`CreateStoreCommand.Execute`（`pkg/server/commands/create_store.go:43`）给新 store 生成一个 ULID 作为 ID。名字由 protobuf 规则约束：3 到 64 个字符，只能是字母、数字、空白和 `.-/^_&@`。删除是**软删除**（SQLite 表里的 `deleted_at`），数据不会立刻消失。

> mini 在 v1.0 之前只要求名字非空。本章对照 OpenFGA 的错误码时发现了这个差别（OpenFGA 拒绝了名字为 `ab` 的 store，mini 接受了），于是补上了同样的正则（`pkg/server/stores.go` 的 `storeNameRe`）。

## 22.2 ULID

OpenFGA 的 store ID、模型 ID、元组和变更日志的排序键都是 ULID：26 个字符，前 10 个字符编码毫秒时间戳，后 16 个是随机数。

```text
01M4F1B2ZHGRA681WX6X1DSJ5W
└─ 时间 ──┘└──── 随机 ────┘
```

它有两个 UUID 没有的性质：**按字符串排序就是按时间排序**；同一毫秒内用单调熵生成时仍然递增。于是“最新的模型”就是 ID 最大的那个，`ReadAuthorizationModels` 按 ID 倒序就是从新到旧，changelog 按 ULID 升序就是提交顺序，不需要额外的时间戳列。

mini 的内存后端和 SQLite 后端都用 `ulid.Monotonic` 生成单调递增的 ID（`newULID`），否则同一毫秒内写入的两条元组顺序可能是乱的，分页和 changelog 就会出错。

## 22.3 模型：不可修改，只能追加

`WriteAuthorizationModelCommand.Execute`（`pkg/server/commands/write_authzmodel.go:56`，第 5 章读过）：检查类型数量（≤100）和序列化大小（≤256 KB），`typesystem.NewAndValidate` 校验，生成 ULID，写入。**没有“更新模型”这个 API**，每次写入都产生一个新版本，旧版本永远保留。

这个设计的好处：

- 已经缓存的 TypeSystem（按模型 ID）永远不会过期（第 3 章）；
- 可以随时回到某个版本，或者让新旧两个版本的服务同时运行；
- 元组不属于任何一个版本，模型变化后，不再合法的旧元组在读取时被过滤掉（第 6 章）。

Check、ListObjects 等 API 都可以带 `authorization_model_id`，不带就用最新版本。**生产环境建议总是带上**：查找“最新模型”需要一次额外的存储查询；更重要的是，有人写入了一个新模型后，所有不带 ID 的请求会立刻换成新模型的语义。OpenFGA 把实际使用的模型 ID 放在响应头 `Openfga-Authorization-Model-Id` 里（`pkg/server/server.go:49`），调用方可以记下来。

## 22.4 Write

`WriteCommand.Execute`（`pkg/server/commands/write.go:81`）分三步：校验、解析选项、调用存储。

**校验**（`validateWriteRequest`，:117）：

1. 写入和删除都为空 → `invalid_write_input`；
2. 有写入时，读出模型（ID 由上层解析），对每条写入的元组：
   - `ValidateTupleForWrite`（第 6 章），出错一律报 `validation_error`；
   - 不能是隐式元组（`validateNotImplicit`，:218，第 6 章）；
   - 条件 context 的大小不能超过限制；
3. 删除只检查 user 的格式，不检查是否符合当前模型：模型变化后，旧的、不再合法的元组也必须能被删除；
4. `validateNoDuplicatesAndCorrectSize`（:189）：同一个请求里不能有重复的元组（`cannot_allow_duplicate_tuples_in_one_request`），写入 + 删除总数不能超过 100（`exceeded_entity_limit`）。

**选项**（:58）：`writes.on_duplicate` 和 `deletes.on_missing` 可以是 `error`（默认）或 `ignore`，让写入变成幂等的：重复写入已存在的元组、删除不存在的元组时不报错。在需要重试的场景（消息队列重放、同步任务）很有用。

**存储**：一次 `Write` 调用、一个事务，先删后写，同时写 changelog（第 21 章）。“已存在”“不存在”的错误被翻译成 `write_failed_due_to_invalid_input`；并发写同一条元组导致事务冲突时返回 gRPC 的 `Aborted`，调用方可以重试。

## 22.5 mini-openfga 的实现

v0.5 把模型和 store 都移进了 `OpenFGADatastore`，Server 不再自己保存状态，重启后（SQLite）一切都还在：

```go
// typesystem resolves a model id; "" means the latest model of the store
// (OpenFGA: typesystem.MemoizedTypesystemResolverFunc).
func (s *Server) typesystem(ctx context.Context, storeID, modelID string) (*typesystem.TypeSystem, error) {
	if err := s.storeExists(ctx, storeID); err != nil {
		return nil, err
	}
	if modelID != "" {
		if ts, ok := s.typesystems.Get(storeID + "/" + modelID); ok {
			return ts, nil
		}
	}
	var m *model.Model
	if modelID == "" {
		m, err = s.ds.FindLatestAuthorizationModel(ctx, storeID)
		...
	} else {
		m, err = s.ds.ReadAuthorizationModel(ctx, storeID, modelID)
		...
	}
	if ts, ok := s.typesystems.Get(storeID + "/" + m.ID); ok {
		return ts, nil
	}
	ts, err := compile(m)
	...
	s.typesystems.Set(storeID+"/"+m.ID, ts, 24*time.Hour)
	return ts, nil
}
```

TypeSystem 按 `store/模型ID` 缓存在一个 LRU 里，从存储读出的模型要重新编译（索引、校验、编译条件），只在第一次用到时做。

`Write` 的校验顺序与 OpenFGA 一致：空请求、数量上限、请求内重复、逐条 `ValidateTupleForWrite` 和隐式元组检查（全部报 2000）、删除只查格式，然后一次 `ds.Write`。mini 没有实现 `on_duplicate`/`on_missing` 选项，也没有条件 context 的大小限制。

## 22.6 运行与验证：错误码逐条对照

同一组写入请求分别发给 OpenFGA 和 mini-openfga v1.0：

| 场景 | OpenFGA | mini |
|---|---|---|
| 正常写入 | `{}` | `{}` |
| 写入已存在的元组 | `write_failed_due_to_invalid_input` | 相同 |
| 删除不存在的元组 | `write_failed_due_to_invalid_input` | 相同 |
| 同一请求里重复 | `cannot_allow_duplicate_tuples_in_one_request` | 相同 |
| user 的类型不在模型里 | `validation_error`：`Invalid tuple 'doc:4#viewer@team:x'. Reason: type 'team' not found` | 相同（信息一字不差） |
| 空请求 | `invalid_write_input` | 相同 |
| 101 条 | `exceeded_entity_limit`：`The number of write operations exceeds the allowed limit of 100` | 相同 |
| store 名为 `ab` | `validation_error` | v1.0 之前：创建成功；v1.0：`validation_error` |

SQLite 后端的“重启后数据还在”：

```bash
cd code/mini-openfga && git checkout v0.5
go test ./pkg/server/ -run TestSQLiteSurvivesRestart -v
```

测试用一个 Server 写入模型和元组，关闭；再用同一个数据库文件打开一个新的 Server，Check 仍然为 true。

## 22.7 小结

- Store 是隔离单元；ID 是 ULID，按字符串排序即按时间排序；
- 模型只能追加不能修改，“最新模型”就是 ID 最大的；生产环境的请求应该带上模型 ID；
- Write 的校验分三层：请求级（空、数量、重复）、逐条（模型规则、隐式元组）、存储级（已存在、不存在、事务冲突）；
- 删除只检查格式，保证模型变化后旧元组仍能被清理；`on_duplicate`/`on_missing` 让写入可以幂等。

## 22.8 练习

1. （巩固）为什么删除元组时不按当前模型校验？构造一个“如果校验了就删不掉”的场景。
2. （扩展）给 mini 实现 `on_duplicate: "ignore"` 和 `on_missing: "ignore"`，需要改存储接口吗？内存后端和 SQLite 后端分别怎么实现？
3. （读源码）OpenFGA 的 `ReadAuthorizationModels` 返回的模型带分页令牌。`pkg/server/commands/read_authzmodels.go:47` 是怎样编码这个令牌的？它和 ListStores 的令牌有什么不同？

## 22.9 延伸阅读

- ULID 规范：<https://github.com/ulid/spec>
- OpenFGA 文档 *Production best practices*：关于固定模型 ID 的建议
- `pkg/server/commands/write.go`、`write_test.go`
