# 附录 B 术语表

按拼音/字母顺序。括号里是本书主要讲解它的章节。

| 术语 | 英文 / 代码中的名字 | 含义 |
|---|---|---|
| 授权模型 | authorization model | 一组类型定义和条件，描述有哪些对象类型、每种类型有哪些关系、关系怎样由元组和其他关系推出。不可修改，每次写入产生一个新版本（4、5、22） |
| 版本（模型） | authorization model ID | 模型的 ULID；“最新模型”就是 ID 最大的那个（22） |
| 元组 | relationship tuple, `object#relation@user` | 一条事实：某个用户（或用户集）与某个对象有某种关系（6） |
| 对象 | object, `type:id` | 被授权的东西，例如 `document:design`（1、6） |
| 用户 / 主体 | user, subject | 元组的 user 一侧：具体对象（`user:anne`）、通配符（`user:*`）或用户集（`group:eng#member`）（6） |
| 用户集 | userset, `object#relation` | “与某对象有某关系的所有主体”，例如 `group:eng#member`（1、11） |
| 通配符 | typed wildcard, `user:*` | 某类型的所有主体，只能出现在 user 一侧（6、11） |
| 隐式元组 | implicit tuple | `group:a#member@group:a#member` 这种“自己包含自己”的元组，总是成立，不允许写入（6、22） |
| 类型限制 | type restriction, directly related user types | `define viewer: [user, group#member]` 方括号里的内容，规定可以直接写入哪些形态的用户（5、6） |
| 重写规则 | userset rewrite | 关系的定义：直接赋值（this）、计算关系（computed）、元组到用户集（TTU）、并、交、差（1、4） |
| 计算关系 | computed userset | `define viewer: editor`：同一个对象上的另一个关系（4） |
| 元组到用户集 | tuple to userset, TTU, `X from parent` | 先读 tupleset 关系找到父对象，再在父对象上求 X（4、11） |
| tupleset | tupleset | TTU 里 `from` 后面的那个关系，必须是直接关系、不能是 userset（5、6） |
| 条件 | condition | 用 CEL 写的布尔表达式，挂在类型限制和元组上，参数来自元组或请求（14） |
| 内联表达式 | `$expression` | 直接写在元组上的条件，实验特性 `inline_expressions`（14） |
| 上下文元组 | contextual tuples | 随请求发送、只对这次请求有效的元组（13） |
| store | store | 隔离单元：一组模型版本、元组、断言和变更日志（22、29） |
| Check | Check | “用户 U 与对象 O 是否有关系 R”（3、8–12） |
| BatchCheck | BatchCheck | 一次请求里的多个 Check，相同的会被去重（17） |
| Expand | Expand | 展开一个 `object#relation` 的一层用户集合树（16） |
| ListObjects | ListObjects | “用户 U 对哪些 T 类型的对象有关系 R”（18、19） |
| ListUsers | ListUsers | “哪些主体（按过滤器）对对象 O 有关系 R”（20） |
| 断言 | assertion | 挂在模型版本上的期望结果，相当于模型的单元测试（23） |
| 变更日志 | changelog, ReadChanges | 元组写入与删除的有序记录，用续读令牌消费（23） |
| 续读令牌 | continuation token | 分页 API 返回的不透明字符串，带着它继续读下一页（21–23） |
| 一致性偏好 | consistency, `HIGHER_CONSISTENCY` | 要求跳过缓存、直接读存储（23） |
| 解析器 | CheckResolver | Check 的处理单元接口；多个解析器组成一个环形链（9、15） |
| LocalChecker | LocalChecker | 真正按重写规则求值的解析器（9） |
| 派发 | dispatch | 把一个子问题交给解析链（可能带缓存、限流）再求一次，例如对每个组求成员（9） |
| 深度 | resolution depth | 一条路径上派发的次数，默认上限 25，超过报 `authorization_model_resolution_too_complex`（12） |
| 环检测 | cycle detection, `VisitedPaths` | 同一条路径上第二次遇到同一个子问题（12） |
| 环标志 | `CycleDetected` | 结果来自环、而不是来自数据；交集和排除对它特殊处理（10、12） |
| 路径相关 | `PathDependent`（mini） | 自递归关系里截断环得到的 false：语义上是 false，但不能缓存（12、31） |
| 短路 | short-circuit | union 遇到 true、intersection 遇到 false、exclusion 遇到 base false 或 subtract true 时立即返回并取消其他分支（10） |
| 不变缓存键 | invariant cache key | 缓存键里与子问题无关、但影响结果的部分：上下文元组和 context 的哈希（13、15） |
| 缓存控制器 | cache controller | 根据变更日志判断缓存何时过期（23） |
| 派发限流 | dispatch throttling | 派发次数超过阈值后放慢派发（15） |
| 权重 | weight | 从某类用户到某个关系最多经过几条元组，∞ 表示递归（27） |
| 带权图 | weighted authorization model graph | 由 `openfga/language` 构造的模型图，节点和边都带权重（27） |
| PathExists | PathExists | 模型里是否存在从这类用户到这个关系的路径；不存在就直接返回 false（27） |
| 求值策略 | strategy: default / weight2 / recursive | 同一个子问题的不同算法：逐个派发、两个集合求交、递归集合计算（27） |
| 规划器 | planner | 用 Thompson 采样按子问题形状选择策略（28） |
| 反向展开 | reverse expand | ListObjects 的经典算法：从用户出发沿关系图反向走（18） |
| 关系图的边 | relationship edge | 反向展开用的边：direct、TTU、computed（18） |
| 需要复核 | `RequiresFurtherEval` | 反向展开经过 `and`/`but not` 得到的候选，需要用 Check 确认（18、19） |
| 流水线 | pipeline ListObjects | v1.22 默认的 ListObjects 实现：模型编译成 worker 数据流图（19） |
| 矩阵测试 | matrix tests | `assets/tests/*.yaml` 中的共享测试，在所有存储上运行（0、31） |
| 影子模式 | shadow check | 生产中用主实现返回结果、后台用新实现再算一遍并比较（31） |
| 差分测试 | differential testing, `cmd/difftest` | 随机数据上比较两个实现的所有答案（31） |
| 预共享密钥 | preshared key | 最简单的 API 认证方式（29） |
| 访问控制（实验） | `enable-access-control` | 用一个 OpenFGA store 授权 OpenFGA 自己的 API（29） |
| 实验特性 | `--experimentals` | 默认关闭（或默认开启但可配置）的新功能开关（2） |
| ULID | ULID | 26 字符、按字符串排序即按时间排序的 ID（22） |
| WAL | write-ahead logging | SQLite 的日志模式：读写互不阻塞（21） |
| UDS | Unix domain socket | OpenFGA 的 HTTP 网关通过它在进程内调用 gRPC 服务（24） |
| grpc-gateway | grpc-gateway | 从 protobuf 注解生成 HTTP/JSON 到 gRPC 的转换层（24） |
| 解析树（mini） | explanation tree, `/explain` | 不短路地记录一次 Check 的每一步，用于 Playground（26） |
