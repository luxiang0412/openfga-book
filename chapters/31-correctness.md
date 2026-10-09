# 第 31 章 正确性：怎样知道答案是对的

> 本章目标：OpenFGA 的第一优先级是正确性。本章梳理它用来保证正确性的三种手段：共享的矩阵测试、生产中的影子比较，以及本书为 mini 做的、对着真实 OpenFGA 的差分测试；并完整记录差分测试在本书最后阶段发现的一个**语义差异**、它的修复，以及顺带发现的 OpenFGA 自身 ListUsers 与 Check 的不一致。
> 前置：全书，尤其是第 10–12、20、27 章。对应代码：mini-openfga **v1.0** `internal/matrix/`、`cmd/difftest/`、`pkg/check/recursion_test.go`。

## 31.1 三层保障

授权服务的错误很少表现为崩溃，更常见的是**悄悄给出错误的 true 或 false**。错误的 false 让用户打不开自己的文档，错误的 true 是安全事故。所以光有“测试通过”不够，还要问：测试的期望从哪里来？

| 层次 | 期望从哪里来 | 能发现什么 |
|---|---|---|
| 单元测试 | 写测试的人 | 写测试的人想到的情况 |
| 矩阵测试 | 一份共享的、人工审核过的 YAML | 规范里列出的情况 |
| 差分 / 影子比较 | **另一个实现**的答案 | 两个实现不一致的任何情况，包括没人想到的 |

## 31.2 矩阵测试

`AGENTS.md` 说得很直接：“OpenFGA 服务端算法正确性的主要测试是 tests 目录下的各种矩阵测试。”`assets/tests/` 下的两个 YAML（第 0 章介绍过格式）里共有 171 个测试（`consolidated_1_1_tests.yaml` 138 个，`abac_tests.yaml` 33 个），每个测试分若干阶段，每阶段一个模型、一组元组和若干 Check / ListObjects / ListUsers 断言。运行器在 `tests/check/check.go`、`tests/listobjects/`、`tests/listusers/`，同一组断言会在内存、Postgres、MySQL、SQLite 上分别跑。

mini 从 v0.1 开始就用同一份 YAML（`testdata/openfga/`，保留了 Apache-2.0 声明），运行器 `internal/matrix` 支持三类 API，并且客户端是可替换的。到 v1.0，同一份断言在这些配置下全部通过（2310 条：Check 939、ListObjects 641、ListUsers 730）：

| 测试 | 配置 |
|---|---|
| `TestMatrix` | 进程内，内存存储 |
| `TestMatrixCheckWithCacheAndThrottling` | 开启 Check 缓存和派发限流 |
| `TestMatrixCheckOptimizations` | 开启 PathExists 和 weight2 |
| `TestMatrixHTTPSQLite` | 走 HTTP，SQLite 存储 |
| `TestMatrixExplain` | 用第 26 章的 Explainer 代替 Check |
| `minifga matrix -http <OpenFGA>` | 同一个 HTTP 客户端打真实的 OpenFGA（验证客户端本身，第 24 章） |

矩阵测试在本书里抓到过不少问题：v0.1 运行器跳过了没有断言的阶段（第 8 章）、ListUsers 的条件错误语义（第 20 章）、权重的两个 bug（第 27 章）。但下面会看到，**矩阵 100% 通过之后**，还有一系列差异是它覆盖不到的。

## 31.3 影子比较：OpenFGA 在生产中怎么做

新算法在 OpenFGA 里先放在实验开关后面，然后用**影子模式**对照（`shadow_check`，`internal/graph/shadow_resolver.go`）。`ShadowResolver.ResolveCheck`（:47）：

```go
res, err := s.main.ResolveCheck(ctx, req)
...
reqClone := req.clone()
reqClone.VisitedPaths = nil // reset completely for evaluation
s.wg.Add(1)
go func() {
	...
	ctx, cancel := context.WithTimeout(ctxClone, s.shadowTimeout)
	defer cancel()
	shadowRes, err := s.shadow.ResolveCheck(ctx, reqClone)
	...
	if shadowRes.GetAllowed() != resClone.GetAllowed() {
		s.logger.InfoWithContext(ctx, "shadow check difference", ...)
	}
}()
return res, nil
```

主实现的结果立即返回给调用方，影子实现在后台用克隆的请求再算一遍（有超时，默认 1 秒，`config.go:61`），只比较 `Allowed`，不同就记一条日志，包括双方的 `cycle` 标志和延迟。真实流量就是最好的测试数据，而且不影响线上结果。代价是双倍的计算，所以通常只在一部分 store 上开启。

## 31.4 差分测试：mini 对 OpenFGA

mini 没有生产流量，但有一个更强的对照对象：**真实的 OpenFGA**。`cmd/difftest` 的做法：

1. 几个场景（模型 + 随机元组生成器）：嵌套的组和文件夹（会随机生成环）、通配符、`and`、`but not`，以及带条件的元组；
2. 每轮在两个服务上建相同的 store，写入相同的 15 条随机元组；
3. 对一个小宇宙（几个对象 × 所有关系 × 几个用户 × 几种请求上下文）的**每一个** Check、ListObjects、ListUsers 都在两边各问一次，比较结果（包括“是否报错”）；
4. 不同就打印出来，带着两边的 store ID，可以直接复现。

“已知差异”必须是经过分析、在书里有记录的类别，而且分类条件要尽可能窄，否则它会把真正的问题藏起来。到 v1.0 有三类：

| 类别 | 内容 | 章节 |
|---|---|---|
| 流水线 ListObjects 求值 subtract 一侧的条件 | 不带参数时 OpenFGA 报错、mini（和经典算法一样）不报 | 第 19 章 |
| 缺参数报错还是 false 取决于策略 | 两边都取决于规划器选了哪个策略 | 第 14 章 |
| OpenFGA 的 ListUsers 漏掉了它自己的 Check 允许的用户 | 见下文 31.5.2；**每一例都用 OpenFGA 的 Check 逐个确认** | 本章 |

### 31.4.1 它一路发现了什么

按时间顺序，差分测试在矩阵测试 100% 通过之后发现的问题：

| 发现 | 结果 |
|---|---|
| OpenFGA 拒绝写入隐式元组 `group:g2#member@group:g2#member`，mini 接受 | mini 修复（第 6、22 章） |
| Write 的校验错误 OpenFGA 一律报 2000，mini 有时报 2027 | mini 修复（第 22 章） |
| 条件错误的“只要有一条可用就忽略”规则 | mini 修复（第 7 章），这一条是读源码发现、再用 OpenFGA 确认的 |
| ListUsers 也要走 subtract 一侧并求值条件 | mini 修复（第 20 章） |
| 流水线 ListObjects 的 subtract 条件 | 已知差异（第 19 章） |
| 缺参数时报错与否取决于策略 | 已知差异（第 14 章） |
| **自递归关系里的环** | mini 修复（下文） |
| **OpenFGA ListUsers 与自己的 Check 不一致** | 已知差异（下文） |

## 31.5 最后一个发现：环到底是什么意思

### 31.5.1 差异

写本章时用新的随机种子重跑差分测试。种子 21 没有差异，种子 22 报告了 4 个：

```text
DIFF [groups-folders round 23, ...] check document:d1#can_view@user:u0
   a=false <nil>
   b=true <nil>
DIFF [groups-folders round 23, ...] check document:d1#can_edit@user:u0
   a=false <nil>
   b=true <nil>
DIFF [groups-folders round 23, ...] list-objects document#can_view@user:u0
   a=[]
   b=[document:d1]
DIFF [groups-folders round 23, ...] list-objects document#can_edit@user:u0
   a=[]
   b=[document:d1]
```

这是 true/false 的差异，没有条件参与，是最严重的一类。关掉 mini 的优化再跑，差异不变，说明和 weight2、PathExists 无关。把 store 导出来（模型的相关部分和全部 15 条元组中的 5 条）：

```text
define member: [user, user:*, group#member]           （group）
define blocked: [user, group#member]                  （document）
define viewer: [user, user:*, group#member] or editor or viewer from parent
define editor: [user, group#member] or owner
define can_view: viewer but not blocked

group:g2#member@group:g0#member
group:g0#member@group:g2#member
document:d1#blocked@group:g1#member
document:d1#blocked@group:g2#member
document:d1#owner@user:u0
```

u0 是 owner，所以是 editor、是 viewer；`blocked` 来自 g1（没有成员）和 g2，而 g2 和 g0 互相包含、没有任何用户。mini 的解析树（第 26 章）说得很清楚：

```text
    computed  blocked  => cycle
      relation  document:d1#blocked@user:u0  => cycle
        direct    [user, group#member]  => cycle
          userset   document:d1#blocked@group:g1#member  => false
          userset   document:d1#blocked@group:g2#member  => cycle
            relation  group:g2#member@user:u0  => cycle
              ...
                      userset   group:g0#member@group:g2#member  => cycle
                        relation  group:g2#member@user:u0  => cycle
```

subtract 一侧是“环”，按第 12 章的规则，`but not` 拒绝。

可 OpenFGA 的 `exclusion`（`internal/graph/check.go:295`）也是这条规则：

```go
// Short-circuit: If subtract is true, the whole expression is false.
if res.resp.GetCycleDetected() || res.resp.GetAllowed() {
	return &ResolveCheckResponse{Allowed: false, ...}, nil
}
```

那它的 subtract 一侧一定**没有**报告环。向 OpenFGA 的 TypeSystem 求证：

```text
UsersetUseRecursiveResolver(group, member, user) = true
UsersetUseRecursiveResolver(document, blocked, user) = false
```

`group#member` 满足 recursive 策略的条件（第 27 章）。这个策略用集合运算求成员，g0、g2 互相包含只是“没有新成员”，根本不会产生环标志；对同一个 Check 连续问 OpenFGA 100 次，100 次都是 true。

从语义上说，OpenFGA 是对的：只由 union 组成、只通过自身递归的关系是一个最小不动点，g2 的成员是空集，u0 不是成员。“环 = 不确定”只适用于环穿过 `and`/`but not` 的情况（第 12 章的 `true_butnot_cycle_return_false`，那里它的确无法判定）。

### 31.5.2 修复，以及修复里的陷阱

mini 没有 recursive 策略，但可以复现它的**语义**：

1. `typesystem.RecursiveRelation(objectType, relation, userType)`：权重为 ∞，唯一的无限路径是对自身的 userset 或 TTU，其余分支一跳可达。判断条件对照 OpenFGA 的两个函数写成，`TestRecursiveRelationMatchesOpenFGA` 把 OpenFGA 对四个模型**每一个关系**的回答固定下来（云盘：只有 `folder#editor`；公司：只有 `team#member`；本例：只有 `group#member`；Lab 6：没有）；
2. 环检测处：如果回到的节点属于这样的关系，返回 false 而不是环（`LocalChecker`、`Reference`、`Explainer` 三处一起改，后两者分别用于第 8 章的参照实现和第 26 章的解析树）。

第 2 步有一个陷阱，我在动手前想到了，并为它先写了测试。深度优先遍历时把“回到路径上已有的节点”当作 false，对**进入递归的那个节点**是精确的，但递归内部的节点可能被低估：

```text
group:a#member@group:b#member
group:a#member@group:c#member
group:b#member@group:a#member
group:c#member@user:u
```

查 a：a 并发地查 b 和 c；b 回到 a，被当作 false，b 也没有直接成员，于是 b 的结果是 false。a 经 c 得到 true，答案正确。但 b 自己的那个 false 是错的（b 包含 a，a 包含 c，u 是 b 的成员）。如果第 15 章的缓存记住了它，之后单独查 b 就会读到 false。

所以这种 false 带一个新的标志 `PathDependent`：语义上就是 false，exclusion 和 intersection 照常使用，但缓存永远不存它，标志沿着 reducer 向上传播（exclusion 的 true 依赖 subtract 的 false，也要继承它）。`TestRecursionResultsAreNotCached` 在开启缓存的解析链上先查 a、再查 b，重复 50 次。为了确认这个测试真能抓住问题，把缓存条件临时改回“只跳过 CycleDetected”：

```text
--- FAIL: TestRecursionResultsAreNotCached (0.00s)
    recursion_test.go:90: run 1: b answered false from the cache; b contains a, which contains c, which contains u
```

改回来后通过。第 12 章的一个旧测试期望 `document:3#viewer@user:alice` 返回 `cycle=true`（a、b 两组成环），按新语义改成了不带环标志的 false，第 12 章也补了 12.3.1 节。

### 31.5.3 OpenFGA 自己的不一致

修复后重跑种子 22，Check 和 ListObjects 的差异消失了，剩下 2 个 ListUsers 差异：

```text
DIFF [...] list-users document:d1#can_view user
   a=[user:u0]
   b=[]
```

这次 mini 说 u0 能看，OpenFGA 的 ListUsers 说没有人能看。直接在 OpenFGA 的同一个 store 上问三个 API：

```text
OpenFGA Check   document:d1#can_view@user:u0 -> {"allowed":true,"resolution":""}
OpenFGA ListUsers document:d1#can_view (user)  -> {"users":[]}
OpenFGA ListObjects user:u0 can_view document  -> {"objects":["document:d1"]}
```

**OpenFGA 的 ListUsers 和它自己的 Check、ListObjects 矛盾。**原因在第 20 章提到的 `expandExclusion`（`pkg/server/commands/listusers/list_users_rpc.go:811`）：subtract 一侧只要遇到环，整个排除就返回空集，它没有 Check 那种对自递归关系的特殊处理。

mini 的 ListUsers 建立在 Check 之上（第 20 章），三个 API 互相一致，所以这里选择**不**模仿 OpenFGA 的 ListUsers。差分测试把它作为第三个已知类别，但条件很严格：只有当 OpenFGA 的列表是 mini 列表的真子集，并且 OpenFGA 自己的 Check 对**每一个**被漏掉的用户都回答 true 时，才算已知；否则仍然报告为差异。这是一个值得向 OpenFGA 报告的问题。

### 31.5.4 最终结果

种子 21–30，每个种子 40 轮、9080 个查询，分别对开启和关闭优化的 mini：

```text
seed 22 port 8096: 9080 queries compared, 0 differences|  13 x pipeline evaluates subtract-side conditions|  2 x OpenFGA ListUsers omits users its own Check allows (cycle on the subtract side)|
seed 26 port 8097: 9080 queries compared, 0 differences|  2 x OpenFGA ListUsers omits users its own Check allows (cycle on the subtract side)|  9 x pipeline evaluates subtract-side conditions|
...
TOTAL: 181600 queries, 0 unexplained differences
```

（只摘了两行。）全部 20 次运行中，第三类已知差异共出现 10 次（种子 22、23、26，开启和关闭优化各一次），每一次都经过了上面的确认。

## 31.6 回顾：全书发现的问题

把本书各章“踩坑”里**实际观察到**的问题按发现手段归类：

| 手段 | 发现的问题（章节） |
|---|---|
| 矩阵测试 | 运行器跳过阶段（8）、ListUsers 条件错误（20）、权重 userset 自身路径与通配符键（27） |
| 竞态检测 | `NormalizeTuples` 原地修改共享数据（14） |
| 差分测试 | 隐式元组、Write 错误码（22）、ListUsers 的 subtract 条件（20）、自递归关系的环（12、31）、OpenFGA ListUsers 的不一致（31） |
| 对照 OpenFGA 的源码或实际行为 | 条件错误的 once-valid 规则（7）、store 名称规则（22）、模型 ID 响应头（24）、预共享密钥的时序（29） |
| 查询计划 / 基准 | SQLite 的 `ORDER BY` 让读取慢 200 倍（21） |
| 浏览器里真正用一遍 | Explain 路由没注册（26）、分页（25）、语义校验（25）、版本下拉框、解析树展开状态（Lab 6） |
| 干净的克隆 | `web/dist` 从未提交、CRLF 让脚本无法运行（25） |
| 看日志 | 我自己引入的路由标签回归（30） |

没有哪一种手段能单独发现所有问题。最有效的组合是：**一份共享的规范（矩阵测试）+ 一个独立的对照实现（差分测试）+ 像用户一样真正使用它**。

## 31.7 运行与验证

```bash
cd code/mini-openfga && git checkout v1.0
go test ./...                                   # 包括全部矩阵配置和本章的回归测试
go test -run 'TestRecursive|TestRecursion' -v ./pkg/check/ ./pkg/typesystem/

# 差分测试需要一个真实的 OpenFGA（第 0 章）
openfga run --playground-enabled=false --http-addr 127.0.0.1:18080 --grpc-addr 127.0.0.1:18081 \
  --experimentals inline_expressions &
./dist/minifga serve -addr 127.0.0.1:8080 &
go run ./cmd/difftest -a http://127.0.0.1:8080 -b http://127.0.0.1:18080 -rounds 40 -seed 22
```

## 31.8 小结

- 授权错误往往是静默的，测试的期望必须有独立的来源；
- OpenFGA 用共享的矩阵测试定义行为，用影子模式在真实流量上比较新旧实现；
- 对一个“重新实现”来说，最强的测试是对着原实现做随机差分；已知差异要窄、要可证明；
- 最后一次差分测试发现了一个真正的语义差异：只通过自身递归的关系里，环意味着“没有成员”而不是“不确定”。修复它要小心缓存，修复之后又暴露出 OpenFGA 自己 ListUsers 与 Check 的不一致。

## 31.9 练习

1. （巩固）为什么 `PathDependent` 的 false 可以被 exclusion 当作普通的 false 使用，却不能被缓存？用 31.5.2 的 a、b、c 例子说明“对进入递归的节点精确”是什么意思。
2. （扩展）给 `cmd/difftest` 增加一个场景：TTU 的自递归（`define viewer: [user] or viewer from parent`，文件夹随机成环），再加上 `but not`。预测会不会出现第三类差异，然后跑一下。
3. （读源码）OpenFGA 的 `ShadowResolver` 只比较 `Allowed`，不比较错误。举一个“主实现返回 false、影子实现返回错误”的情况，说明为什么它选择只记一条 warn 日志。

## 31.10 延伸阅读

- `AGENTS.md` 的 Testing 部分；`tests/` 目录
- McKeeman, *Differential Testing for Software*（1998）
- Zanzibar 论文第 3.2 节：生产中的验证
