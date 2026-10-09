# 附录 A 源码阅读地图

左列是概念，中间是 OpenFGA（`openfga/openfga` @ `526995eb`，v1.22.0）里的位置，右列是 mini-openfga v1.0 里对应的位置，最后一列是本书讲解它的章节。路径都经过脚本核对（见本附录末尾）。

## 模型与类型系统

| 概念 | OpenFGA | mini-openfga | 章 |
|---|---|---|---|
| DSL 解析 | `openfga/language` 模块的 `pkg/go/transformer/dsltojson.go` | `pkg/dsl/lexer.go`、`pkg/dsl/parser.go` | 4 |
| 模型数据结构 | `openfga/api` 模块的 `openfga/v1/authzmodel.pb.go` | `pkg/model/model.go`、`pkg/model/openfgajson.go` | 4 |
| 写模型 | `pkg/server/commands/write_authzmodel.go` | `pkg/server/models.go` | 5、22 |
| 类型系统与模型校验 | `pkg/typesystem/typesystem.go` | `pkg/typesystem/typesystem.go`、`pkg/typesystem/validate.go` | 5 |
| 带权图 | `pkg/typesystem/weighted_graph.go`（图本身由 language 模块构造） | `pkg/typesystem/weights.go` | 27 |
| 递归策略的判定 | `pkg/typesystem/typesystem.go`（`UsersetUseRecursiveResolver`、`TTUUseRecursiveResolver`） | `pkg/typesystem/weights.go`（`RecursiveRelation`） | 27、31 |

## 元组与存储

| 概念 | OpenFGA | mini-openfga | 章 |
|---|---|---|---|
| 元组工具 | `pkg/tuple/tuple.go` | `pkg/tuple/tuple.go` | 6 |
| 元组校验 | `internal/validation/validation.go` | `pkg/validation/validation.go` | 6 |
| 存储接口 | `pkg/storage/storage.go` | `pkg/storage/storage.go`、`pkg/storage/datastore.go` | 7、21 |
| 条件过滤与 once-valid | `pkg/storage/tuple_iterators.go` | `pkg/storage/storage.go`（`FilteredIterator`） | 7 |
| 内存实现 | `pkg/storage/memory/memory.go` | `pkg/storage/memory/` | 7 |
| SQLite 实现 | `pkg/storage/sqlite/sqlite.go` | `pkg/storage/sqlite/sqlite.go` | 21 |
| 表结构 | `assets/migrations/sqlite/` | `pkg/storage/sqlite/sqlite.go`（建表语句） | 21 |
| 一致性测试套件 | `pkg/storage/test/storage.go` | `pkg/storage/storagetest/` | 21 |
| 上下文元组合并 | `pkg/storage/storagewrappers/combinedtuplereader.go` | `pkg/storage/wrappers.go` | 13 |
| 有界并发读 | `pkg/storage/storagewrappers/request.go` | `pkg/check/local_checker.go`（并发上限） | 12 |

## Check

| 概念 | OpenFGA | mini-openfga | 章 |
|---|---|---|---|
| API 入口 | `pkg/server/check.go` | `pkg/server/server.go`（`Check`） | 3 |
| 命令层 | `pkg/server/commands/check_command.go` | `pkg/server/server.go` | 3 |
| 解析器接口 | `internal/graph/interface.go` | `pkg/check/interface.go` | 9 |
| LocalChecker | `internal/graph/check.go` | `pkg/check/local_checker.go` | 9–12 |
| 并发与集合运算 | `internal/graph/check.go`（`union`、`intersection`、`exclusion`） | `pkg/check/reducers.go` | 10 |
| userset 与 TTU | `internal/graph/default_resolver.go` | `pkg/check/local_checker.go` | 11 |
| 朴素参照实现 | — | `pkg/check/reference.go` | 8 |
| 解析链构建 | `internal/graph/builder.go` | `pkg/check/builder.go` | 15 |
| Check 缓存 | `internal/graph/cached_resolver.go` | `pkg/check/cached_resolver.go` | 15 |
| 派发限流 | `internal/graph/dispatch_throttling_check_resolver.go` | `pkg/check/throttling_resolver.go` | 15 |
| weight2 / recursive 策略 | `internal/graph/weight_two_resolver.go`、`internal/graph/recursive_resolver.go` | `pkg/check/strategies.go`（只有 weight2） | 27 |
| 规划器 | `internal/planner/` | `pkg/planner/planner.go` | 28 |
| 影子比较 | `internal/graph/shadow_resolver.go` | — | 31 |
| 条件（CEL） | `internal/condition/` | `pkg/condition/` | 14 |
| 解释一次 Check | — | `pkg/check/explain.go` | 26 |
| BatchCheck | `pkg/server/batch_check.go`、`pkg/server/commands/batch_check_command.go` | `pkg/server/batch_check.go` | 17 |
| Expand | `pkg/server/commands/expand.go` | `pkg/expand/` | 16 |

## ListObjects 与 ListUsers

| 概念 | OpenFGA | mini-openfga | 章 |
|---|---|---|---|
| ListObjects 命令 | `pkg/server/commands/list_objects.go` | `pkg/listobjects/listobjects.go` | 18 |
| 关系图（反向边） | `internal/graph/graph.go` | `pkg/listobjects/edges.go` | 18 |
| 经典反向展开 | `pkg/server/commands/reverseexpand/reverse_expand.go` | `pkg/listobjects/reverse_expand.go` | 18 |
| 流水线 | `internal/listobjects/pipeline/` | — | 19 |
| ListUsers | `pkg/server/commands/listusers/list_users_rpc.go` | `pkg/listusers/listusers.go` | 20 |

## 服务

| 概念 | OpenFGA | mini-openfga | 章 |
|---|---|---|---|
| 启动与配置 | `cmd/run/run.go`、`pkg/server/config/config.go` | `cmd/minifga/main.go` | 2、24 |
| Server 与选项 | `pkg/server/server.go` | `pkg/server/server.go` | 2 |
| Store | `pkg/server/commands/create_store.go` 等 | `pkg/server/stores.go` | 22 |
| Write | `pkg/server/commands/write.go` | `pkg/server/write.go` | 22 |
| ReadChanges | `pkg/server/commands/read_changes.go` | `pkg/storage/datastore.go`、`pkg/server/server.go` | 23 |
| 断言 | `pkg/server/commands/write_assertions.go` | `pkg/server/assertions.go` | 23 |
| 缓存控制器 | `internal/cachecontroller/cache_controller.go` | `pkg/server/cachecontroller.go` | 23 |
| HTTP 网关与错误码 | `cmd/run/run.go`（`runHTTPServer`）、`pkg/server/errors/encoded_errors.go` | `pkg/httpapi/httpapi.go` | 24 |
| 模型 ID 响应头 | `pkg/server/server.go`（`resolveTypesystem`） | `pkg/server/modelid.go` | 24 |
| 认证 | `internal/authn/` | `pkg/httpapi/middleware.go`（`PresharedKeyAuth`） | 29 |
| API 访问控制 | `internal/authz/authz.go` | — | 29 |
| 指标与日志 | `pkg/server/server.go`（`request_duration_ms`）、`internal/utils/bucket.go` | `pkg/httpapi/middleware.go`（`Observe`、`Metrics`） | 30 |
| Playground | `assets/playground/index.html`（iframe） | `web/` | 25、26 |

## 测试

| 概念 | OpenFGA | mini-openfga | 章 |
|---|---|---|---|
| 矩阵测试数据 | `assets/tests/consolidated_1_1_tests.yaml`、`assets/tests/abac_tests.yaml` | `testdata/openfga/`（复制） | 0、31 |
| 矩阵测试运行器 | `tests/check/check.go`、`tests/listobjects/listobjects.go`、`tests/listusers/listusers.go` | `internal/matrix/` | 8、31 |
| 差分测试 | — | `cmd/difftest/main.go` | 31 |
| 端到端部署测试 | — | `scripts/e2e.sh` | Lab 7 |

## 核对

本附录里的路径用下面的命令核对过（OpenFGA 仓库里的路径相对于仓库根目录，`openfga/language`、`openfga/api` 模块里的两项除外）：

```bash
grep -o '`[a-z][a-zA-Z0-9_./-]*`' appendix/a-source-map.md | tr -d '`' | sort -u | while read p; do
  [ -e "$OPENFGA/$p" ] || [ -e "code/mini-openfga/$p" ] || echo "missing: $p"
done
```
