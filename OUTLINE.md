# 《从 OpenFGA 到 Mini-OpenFGA：从零构建 Zanzibar 式授权引擎》大纲

> 参照仓库：`openfga/openfga` @ `526995eb`（v1.22.0）；DSL 解析器来自外部包 `openfga/language/pkg/go` v0.3.2
> 读者目标：读完后，能从零写出一个前后端都能运行的 ReBAC 授权引擎：DSL 建模 → 元组存储 → Check / ListObjects / ListUsers → HTTP API → Playground UI。
> 写作方式：每章先讲概念，再读 OpenFGA 源码（`路径:行号`），再在 mini-openfga 里实现，最后贴真实运行输出。
> 正确性验收：mini-openfga 直接跑原仓库 `assets/tests/*.yaml` 的矩阵测试（挑能支持的子集），通过率写进书里。

## 贯穿全书的项目：mini-openfga

| 版本 | 能力 | 章节 |
|---|---|---|
| v0.1 | 手写 DSL 词法/语法解析器、TypeSystem 与模型校验、Tuple、内存存储、递归 Check（direct / computed / TTU / union / intersection / but not）、CLI | Part II（4–8 章、Lab 1） |
| v0.2 | 并发 Check：短路 union/intersection、exclusion、userset `group#member`、通配符 `user:*`、环检测、深度/广度限制、上下文元组 | Part III（9–13 章、Lab 2） |
| v0.3 | 条件（CEL）、Resolver 链（缓存 + 限流）、Expand、BatchCheck | Part IV（14–17 章、Lab 3） |
| v0.4 | ListObjects（反向展开、流式）、ListUsers | Part V（18–20 章、Lab 4） |
| v0.5 | SQLite 存储、Store / 模型版本（ULID）、Write 校验、Read / ReadChanges、Assertions、一致性与缓存失效、HTTP API | Part VI（21–24 章、Lab 5） |
| v0.6 | Vite + React Playground：模型编辑、元组表、查询面板、解析树可视化 | Part VII（25–26 章、Lab 6） |
| v1.0 | Weighted Graph 与策略选择（v2 check）、Thompson Sampling Planner、API 鉴权（预共享密钥）、指标/追踪、Docker 部署、跑原仓库矩阵测试 | Part VIII（27–31 章、Lab 7） |

默认技术栈：Go 1.26 + `net/http`（不用 gRPC/protobuf）、`modernc.org/sqlite`（纯 Go）、`google/cel-go`（和原仓库一致）、Vite + React + TypeScript。

## Part 0 导读
- 第 0 章 本书怎么读：环境、目录、如何读 OpenFGA 源码（含外部 language 包）

## Part I 全景（浅）
- 第 1 章 授权的几种形态：ACL → RBAC → ABAC → ReBAC，Zanzibar 论文要点
- 第 2 章 OpenFGA 架构鸟瞰：目录地图、Server/Command/Graph/Storage 分层、运行时拓扑
- 第 3 章 一次 Check 的完整生命周期 ★：从 `POST /check` 到 `allowed: true`

## Part II 模型与数据 → v0.1
- 第 4 章 授权模型 DSL：type / relation / 重写规则，ANTLR 解析器 vs 手写递归下降
- 第 5 章 TypeSystem：模型校验、可直接赋值类型、TTU 约束 ★
- 第 6 章 关系元组：object#relation@user、userset、通配符、Tuple 校验
- 第 7 章 存储接口：`OpenFGADatastore` 与内存实现、迭代器
- 第 8 章 最朴素的 Check：把重写规则翻译成递归 ★
- Lab 1：mini-openfga v0.1 —— 用 CLI 给 Google Drive 式模型做权限判断

## Part III Check 引擎 ★★ → v0.2
- 第 9 章 LocalChecker 解剖：ResolveCheck、dispatch 与环形委托
- 第 10 章 集合运算的并发求值：union / intersection / exclusion 的短路 ★★
- 第 11 章 Userset 与 TTU：间接关系的展开与流式读取
- 第 12 章 环、深度与广度：防止授权图把服务拖垮
- 第 13 章 上下文元组与 CombinedTupleReader
- Lab 2：并发 Check + 环形模型 + 上下文元组

## Part IV 条件与 Resolver 链 → v0.3
- 第 14 章 条件元组：CEL 表达式、参数类型、缺参处理
- 第 15 章 Resolver 链：CachedCheckResolver 与 DispatchThrottling ★
- 第 16 章 Expand：把授权树展开给人看
- 第 17 章 BatchCheck：去重、并发与 correlation id
- Lab 3：带时间条件的临时授权 + 缓存命中观测

## Part V 反向查询 → v0.4
- 第 18 章 ListObjects：反向展开（reverse expand）的思路 ★★
- 第 19 章 交集/排除下的 ListObjects：候选 + Check 复核，pipeline 架构
- 第 20 章 ListUsers：从对象出发找主体
- Lab 4：“我能看哪些文档” / “谁能看这份文档”

## Part VI 服务端 → v0.5
- 第 21 章 持久化：SQL 表结构、索引、SQLite 实现
- 第 22 章 Store、模型版本与 Write：ULID、幂等、事务、写入校验
- 第 23 章 ReadChanges、Assertions 与一致性偏好（MINIMIZE_LATENCY / HIGHER_CONSISTENCY）
- 第 24 章 HTTP API：路由、错误映射、请求校验
- Lab 5：curl 走通完整 API

## Part VII Playground → v0.6
- 第 25 章 模型编辑器与元组管理
- 第 26 章 查询面板与解析树可视化
- Lab 6：浏览器里建模并调试授权

## Part VIII 平台化（深）→ v1.0
- 第 27 章 Weighted Graph：给模型边加权，选择 Check 策略（default / weight2 / recursive）★
- 第 28 章 Planner：用 Thompson Sampling 在线挑策略
- 第 29 章 API 鉴权与多租户：preshared key、OIDC、Store 隔离
- 第 30 章 可观测性：OTel 追踪、Prometheus 指标、dispatch/datastore 计数
- 第 31 章 正确性工程：矩阵测试、Shadow Resolver、v2 回退
- Lab 7：Docker 部署 mini-openfga v1.0，并跑原仓库矩阵测试

## 附录
- A 源码阅读地图（概念 → OpenFGA 路径 → mini-openfga 路径）
- B 术语表
- C 横向对比：OpenFGA / SpiceDB / Ory Keto / Casbin / OPA
- D 练习参考答案
