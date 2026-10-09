# 从 OpenFGA 到 Mini-OpenFGA

> 从零构建 Zanzibar 式 ReBAC 授权引擎
> 参照仓库：`openfga/openfga` @ `526995eb`（v1.22.0）
> 配套代码：[github.com/luxiang0412/mini-openfga](https://github.com/luxiang0412/mini-openfga)（本书目录下的 `code/mini-openfga`）

读完这本书，你能从零写出一个和 OpenFGA 协议兼容、通过 OpenFGA 全部矩阵测试的授权引擎：模型 DSL 与类型系统、并发 Check 引擎、条件与缓存、ListObjects/ListUsers、SQLite 持久化、HTTP API、浏览器 Playground，以及带权图、规划器、认证、指标和 Docker 部署。每一部分都先读 OpenFGA 的源码，再自己实现一遍，最后用真实的输出验证。

## 目录

**Part 0 导读**

- [第 0 章 本书怎么读](chapters/00-how-to-read.md)

**Part I 全景**

- [第 1 章 授权的几种形态：从 ACL 到 ReBAC](chapters/01-authz-models.md)
- [第 2 章 OpenFGA 架构鸟瞰](chapters/02-architecture.md)
- [第 3 章 一次 Check 的完整生命周期 ★](chapters/03-check-lifecycle.md)

**Part II 模型与数据 → v0.1**

- [第 4 章 授权模型 DSL：从文本到重写树](chapters/04-dsl.md)
- [第 5 章 TypeSystem：把模型变成可查询的索引，并拒绝坏模型](chapters/05-typesystem.md)
- [第 6 章 关系元组：数据的形状与三层校验](chapters/06-tuples.md)
- [第 7 章 存储接口：引擎只需要四种读法](chapters/07-storage.md)
- [第 8 章 最朴素的 Check：把重写规则翻译成递归](chapters/08-naive-check.md)
- [Lab 1：用命令行给“云盘”做权限判断](chapters/lab-1-drive-cli.md)

**Part III Check 引擎 → v0.2**

- [第 9 章 LocalChecker 解剖](chapters/09-local-checker.md)
- [第 10 章 集合运算的并发求值 ★★](chapters/10-set-operations.md)
- [第 11 章 Userset、通配符与 TTU](chapters/11-usersets-ttu.md)
- [第 12 章 环、深度与广度](chapters/12-cycles-depth.md)
- [第 13 章 上下文元组](chapters/13-contextual-tuples.md)
- [Lab 2：并发 Check、成环的组与上下文元组](chapters/lab-2-concurrent-check.md)

**Part IV 条件与 Resolver 链 → v0.3**

- [第 14 章 条件元组：把 ABAC 嵌进 ReBAC](chapters/14-conditions.md)
- [第 15 章 Resolver 链：缓存与限流 ★](chapters/15-resolver-chain.md)
- [第 16 章 Expand](chapters/16-expand.md)
- [第 17 章 BatchCheck](chapters/17-batch-check.md)
- [Lab 3：临时授权、子问题缓存、BatchCheck 与 Expand](chapters/lab-3-conditions-cache.md)

**Part V 反向查询 → v0.4**

- [第 18 章 ListObjects：沿着授权图反着走 ★★](chapters/18-list-objects.md)
- [第 19 章 交集与排除下的 ListObjects](chapters/19-list-objects-advanced.md)
- [第 20 章 ListUsers：从对象出发找主体](chapters/20-list-users.md)
- [Lab 4：“我能看哪些文档”与“谁能看这份文档”](chapters/lab-4-list.md)

**Part VI 服务端 → v0.5**

- [第 21 章 持久化：表结构、索引与一个慢查询](chapters/21-persistence.md)
- [第 22 章 Store、模型版本与 Write](chapters/22-stores-models-write.md)
- [第 23 章 变更日志、断言与一致性](chapters/23-changes-assertions-consistency.md)
- [第 24 章 HTTP API：协议兼容是可以测出来的](chapters/24-http-api.md)
- [Lab 5：一个能重启的授权服务](chapters/lab-5-service.md)

**Part VII Playground → v0.6**

- [第 25 章 Playground（上）：模型编辑器与元组管理](chapters/25-playground-model-tuples.md)
- [第 26 章 Playground（下）：查询面板与解析树](chapters/26-playground-query-trace.md)
- [Lab 6：在浏览器里找出模型的错误](chapters/lab-6-playground.md)

**Part VIII 平台化 → v1.0**

- [第 27 章 带权图与求值策略](chapters/27-weighted-graph.md)
- [第 28 章 规划器：用 Thompson 采样选策略](chapters/28-planner.md)
- [第 29 章 认证、API 授权与多租户](chapters/29-authn-tenancy.md)
- [第 30 章 可观测性：日志、指标、追踪](chapters/30-observability.md)
- [第 31 章 正确性：怎样知道答案是对的](chapters/31-correctness.md)
- [Lab 7：一条命令部署，一个脚本验收](chapters/lab-7-deploy.md)

**附录**

- [附录 A 源码阅读地图](appendix/a-source-map.md)
- [附录 B 术语表](appendix/b-glossary.md)
- [附录 C 横向对比：OpenFGA / SpiceDB / Ory Keto / Casbin / OPA](appendix/c-comparison.md)
- [附录 D 练习参考答案](appendix/d-answers.md)

## 快速开始

需要 Go 1.26。Node 只有在修改前端时才需要；Docker 只用于 Lab 7。

```bash
cd code/mini-openfga
git checkout v1.0                     # 每个 Part 一个标签：v0.1 … v0.6、v1.0
go test ./...                         # 包括 OpenFGA 的全部矩阵测试
go run ./cmd/minifga check -model examples/drive/model.fga -tuples examples/drive/tuples.txt \
  document:design#can_view@user:alice
go build -o dist/minifga ./cmd/minifga
./dist/minifga serve -addr 127.0.0.1:8080   # API + http://127.0.0.1:8080/playground/
```

看相邻两个版本之间加了什么：`git diff v0.4 v0.5 --stat`。

## 本书是怎样验证的

- **矩阵测试**：OpenFGA 的 `assets/tests/*.yaml`（Check 939 条、ListObjects 641 条、ListUsers 730 条断言）在 mini v1.0 上全部通过，分别在进程内、开启缓存与限流、开启优化、HTTP + SQLite、用解析树代替 Check 这几种配置下运行；
- **客户端本身**：同一个 HTTP 客户端在真实的 OpenFGA v1.22 上也通过全部断言（第 24 章）；
- **差分测试**：mini 与真实 OpenFGA 在随机数据上逐一比较 Check、ListObjects、ListUsers 的答案。最终一轮：10 个随机种子 × 开/关优化，共 181,600 个查询，没有无法解释的差异，剩下的差异都属于三个有记录、可证明的类别（第 31 章）；
- **像用户一样使用**：Playground 用 Playwright 驱动无头 Chrome 操作，书中截图都来自这些运行；Docker 部署用 `scripts/e2e.sh` 验收（认证、矩阵测试、指标、重启后数据仍在，Lab 7）；
- **干净的克隆**：v0.6 和 v1.0 都在新克隆的仓库里 `go build` 和 `go test` 通过（第 25 章记录了为什么要专门做这一步）；
- **输出都是真实的**：书中的命令输出、数字和截图都来自实际运行；推导或未运行的内容会注明。每章的“踩坑记录”只收录实际遇到的问题，原始记录在 `notes/pitfalls.md`。

写作过程中对照 OpenFGA 发现并修复了 mini 的一批问题（每个都带回归测试），也发现了 OpenFGA 本身的几处值得注意的地方：内置 Playground 认证分支的死代码（第 25 章）、`CheckRequest.trace` 字段从未被读取（第 26 章）、一个指标的帮助文本复制错误（第 30 章），以及 ListUsers 与 Check 在自递归关系的环上答案不一致（第 31 章）。

## 配套代码概况（v1.0）

| | |
|---|---|
| Go 代码 | 10,854 行（不含测试），测试 2,037 行（另有 OpenFGA 的 YAML 矩阵测试数据） |
| 前端 | 902 行（Vite + React + TypeScript），构建产物嵌入二进制 |
| 依赖 | `google/cel-go`、`modernc.org/sqlite`、`oklog/ulid`、`prometheus/client_golang`、`gopkg.in/yaml.v3` |
| 不包含 | gRPC、Postgres/MySQL、OIDC、OpenTelemetry 追踪、ListObjects 流水线、recursive 策略（只复现了它的语义）、访问控制 |

## 许可

`code/mini-openfga/testdata/openfga/` 中的测试数据来自 OpenFGA（Apache-2.0），见该目录的 README。
