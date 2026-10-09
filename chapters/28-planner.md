# 第 28 章 规划器：用 Thompson 采样选策略

> 本章目标：理解为什么“哪种策略更快”不能只看模型；读懂 OpenFGA 规划器的 Thompson 采样实现：正态-伽马先验、贝叶斯更新、按键隔离；用模拟实验观察它的三种行为：快速收敛、偶尔犹豫、对变化反应迟缓，并从更新公式解释最后一种。
> 前置：第 27 章。对应代码：mini-openfga **v1.0** `pkg/planner/planner.go`。

## 28.1 为什么需要规划器

第 27 章的 weight2 并不总是更快。它的代价取决于**数据**：

- 对象授权给了 50 个团队、用户只在 1 个团队里：weight2 两次查询，default 50 次派发，weight2 完胜；
- 对象只授权给了 1 个团队、用户却在 5 万个团队里：weight2 要读出 5 万个团队再求交集，default 只派发一次。

模型一样、查询的形状一样，哪个快取决于具体的数据分布，而且会随数据变化。OpenFGA 的做法是不做静态判断，而是**边用边学**：对每一类子问题（一个“键”）记录各策略的实际耗时，按一个在探索与利用之间平衡的规则选择。

键由子问题的形状组成，例如 userset 的情况（`internal/graph/check.go` 中 `checkDirectUsersetTuples`）：

```go
builder.EncodeString("USERSET")
builder.EncodeString(req.GetStoreID())
builder.EncodeString(req.GetAuthorizationModelID())
builder.EncodeString(objectType)
builder.EncodeString(relation)
builder.EncodeString(userType)
```

同一个 store、同一个模型下，“document#viewer 上的 group#member，用户类型是 user”这一类子问题共享一份统计，和具体的文档、用户无关。

## 28.2 Thompson 采样

多臂老虎机问题：几台老虎机（策略），每台的回报（耗时）分布未知，要在尽量少浪费的前提下找到最好的那台。Thompson 采样的规则很简单：

1. 对每个策略，维护一个关于“它的平均耗时是多少”的**信念分布**；
2. 每次选择时，从每个策略的信念分布里**随机抽一个值**，选抽到的值最小的那个；
3. 执行后，用实际耗时更新这个策略的信念。

不确定性大的策略，抽样结果分散，偶尔会抽到很小的值而被选中，这就是“探索”；信念已经很集中的策略，抽样结果稳定，这就是“利用”。不需要一个单独的“探索率”参数。

OpenFGA 用**正态-伽马**分布表示信念（`internal/planner/thompson.go`）：耗时服从均值 μ、精度 τ 的正态分布，参数 `(μ, λ, α, β)`：μ 是均值的估计，λ 是对它的信心（相当于“已经见过几次”），α、β 描述对波动大小的信念。抽样（`Sample`，:28）先从 Gamma(α, β) 抽精度 τ，再从 Normal(μ, 1/(λτ)) 抽均值：

```go
tau := ts.fastGammaSample(r, params.alpha, params.beta)
variance := 1.0 / (params.lambda * tau)
...
mean := params.mu + stdNormal*math.Sqrt(variance)
```

更新（`Update`，:83）是这个共轭先验的标准公式，每观察到一个耗时 x（毫秒）：

```go
newLambda := currentParams.lambda + 1
newMu := (currentParams.lambda*currentParams.mu + x) / newLambda
newAlpha := currentParams.alpha + 0.5
diff := x - currentParams.mu
newBeta := currentParams.beta + (currentParams.lambda*diff*diff)/(2*newLambda)
```

参数存在一个不可变的结构体里，用原子指针做比较并交换（CAS），读路径不加锁。伽马分布的抽样用 Marsaglia–Tsang 方法（α ≥ 1 时），注释说这是为了避开通用库的开销。

**先验**体现了设计者对每个策略的预期（`config.go` 里有大段注释解释每个参数）：

| 策略 | InitialGuess | λ | α | β | 含义 |
|---|---|---|---|---|---|
| default | 50ms | 1 | 0.5 | 0.5 | 不太确定，波动可能很大 |
| weight2 | 20ms | 10 | 20 | 2 | 相当有把握：快，而且稳定 |
| recursive | 150ms | 3 | 3 | 2 | 中等把握，偶有尖峰 |
| default（递归场景） | 500ms | 1 | … | … | 递归场景下 default 预计很慢 |

`Select`（`plan.go:47`）对候选策略各抽一次，取最小。规划器按键保存这些统计（`sync.Map`），可以配置淘汰长时间不用的键（`EvictionThreshold`、`CleanupInterval`），**默认都是 0，即不淘汰**（`config.go:101`、`:104`）。

## 28.3 mini-openfga 的实现

`pkg/planner/planner.go`（171 行）是同样的算法：同样的先验（`DefaultPlan`、`Weight2Plan`）、同样的更新公式、同样的伽马抽样方法（α < 1 时 OpenFGA 退回 gonum 库，mini 用 `Gamma(a) = Gamma(a+1)·U^(1/a)` 的变换，避免引入依赖）；用一个互斥锁代替 CAS（简单，对教学够用），另外统计每个策略被选中的次数，导出为 Prometheus 指标 `minifga_planner_selections_total`（第 30 章）。

`LocalChecker.planned`（第 27 章）把它接进求值过程：

```go
kp := c.planner.For(key)
plan := kp.Select(planner.DefaultPlan, planner.Weight2Plan)
...
start := time.Now()
resp, err := h(ctx)
if err == nil {
	kp.Update(plan, time.Since(start))
}
```

只有成功的执行才更新统计，和 OpenFGA 的 `profiledCheckHandler`（`check.go:632`）一致。

## 28.4 实验：规划器怎样选

不跑真实的查询，直接给规划器喂合成的耗时，每 50 轮统计一次 weight2 被选中的次数。每个场景跑 5 次（规划器的随机数种子来自时间，每次不同）。

**场景 A：weight2 明显更快**（default 30ms，weight2 2ms），200 轮：

```text
run 1: 48 50 50 50
run 2: 50 50 48 50
run 3: 50 50 49 50
run 4: 50 50 50 49
run 5: 49 50 48 50
```

从一开始就几乎总选 weight2。先验本来就偏向它，观察结果又印证了先验。偶尔选一次 default 是探索：default 的信念很宽，偶尔会抽到一个很小的值。

**场景 B：weight2 反而更慢**（default 2ms，weight2 30ms），400 轮：

```text
run 1: 24  0  0  0  0  0  0  0
run 2: 50 50 35  0  0  0  0  0
run 3:  5  0  0  0  0  0  0  0
run 4: 32  0  0  0  0  0  0  0
run 5: 32  0  0  0  0  0  0  0
```

多数情况下几十轮内就改选 default，但第 2 次花了约 135 轮。原因在于 default 的先验均值是 50ms：在它被选中、观察到 2ms 之前，它的抽样大多比 weight2 的 30ms 左右更大。规划器只有在某次“运气好”抽到 default 时才会发现它其实很快，这一次来得早晚是随机的。

**场景 C：情况变了**（default 30ms；weight2 前 200 轮 2ms，之后变成 60ms），600 轮：

```text
run 1: 50 50 50 48 50 50 47 44 22  8  3  1
run 2: 49 50 48 50 50 50 48 43 21  3  4  0
run 3: 50 49 49 50 50 48 48 40 24  6  6  3
run 4: 50 50 50 50 50 50 50 50 48  6  0  0
run 5: 48 50 50 50 50 48 49 44 21  5  3  1
```

第 200 轮之后 weight2 已经比 default 慢一倍，规划器却又坚持了大约 **200 轮**（第 5–8 个桶几乎全是 weight2），到第 400 轮附近才转向。

这可以直接从更新公式算出来。前 200 轮之后，weight2 的 λ = 10 + 200 = 210，μ ≈ (10×20 + 200×2)/210 ≈ 2.9ms。之后每次观察 60ms，均值变为 (210×2.9 + 60n)/(210 + n)，要超过 default 的 30ms，需要 n > 约 190。λ 只增不减，**每一个历史观察的权重永远相同**，旧证据越多，新证据推动均值越慢。

这不是 mini 的问题，OpenFGA 用的是同一个公式；它的键淘汰只针对“长时间没被使用”的键，而且默认关闭，一个持续被使用的键永远不会重置。实际影响取决于数据变化有多突然：如果某个组突然被加入了几万个成员，对应的键可能要再慢上一段时间才会换策略。常见的改进是给旧观察加衰减（滑动窗口或指数遗忘），代价是更多的探索。

## 28.5 在真实负载下

Lab 7 的 Docker 端到端脚本（`scripts/e2e.sh`）在开启优化的 mini 上跑完全部矩阵测试后，读 `/metrics` 里规划器的选择次数。规划器在所有键上的选择大多是 weight2：先验偏向它，矩阵测试的数据量又小，weight2 的两次查询总是很快。具体数字见 Lab 7。

## 28.6 小结

- “哪个策略快”取决于数据，所以 OpenFGA 用在线学习而不是静态规则：按子问题的形状分键，每个键独立学习；
- Thompson 采样：从信念分布中抽样、选最小、用观察更新；不确定性本身驱动探索；
- 先验编码了设计者的预期：weight2 被认为快而稳定，default 不确定；
- 正态-伽马的更新没有遗忘，长期稳定的键对突变反应迟缓（实验中约 200 轮，与公式推算一致）。

## 28.7 练习

1. （巩固）场景 B 中，如果把 default 的先验 InitialGuess 从 50ms 改成 10ms，你预期结果会怎样变？改 mini 的 `DefaultPlan` 跑一下验证。
2. （扩展）给 mini 的 `Stats.Update` 加一个上限：λ 最大为 50（超过后，旧观察的权重按比例缩小）。重跑场景 C，转向需要多少轮？场景 A 会不会因此变差？
3. （读源码）OpenFGA 的 `keyPlan.Select`（`plan.go:47`）对 `map[string]*PlanConfig` 做 `range`。Go 的 map 遍历顺序是随机的，这会影响结果吗？在什么情况下会？

## 28.8 延伸阅读

- Russo et al., *A Tutorial on Thompson Sampling*（2018）
- 正态-伽马共轭先验：Murphy, *Conjugate Bayesian analysis of the Gaussian distribution*
- `internal/planner/` 全部文件，以及 `benchmark_planner_test.go`
