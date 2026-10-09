# 第 19 章 交集与排除下的 ListObjects：候选复核与流水线

> 本章目标：理解 `and`/`but not` 为什么让反向展开变难；量一量“候选 + Check 复核”的代价；了解 OpenFGA v1.22 默认使用的新实现，基于带权图的流水线，以及它和经典实现在一个边界情况下的差异。
> 前置：第 18 章。对应代码：mini-openfga **v0.4** `pkg/listobjects/listobjects.go`（mini 只实现经典算法）。

## 19.1 为什么交集和排除难

反向展开沿着边“往上走”，每一步都是在扩大集合：从 anne 到她的组，再到组授权的文件夹，再到文件夹下的文档。union 天然适合这种方式：`a or b` 的结果就是从 a 走到的加上从 b 走到的。

`a and b`、`a but not b` 不是这样。`document#can_view = viewer but not blocked`：从 anne 出发沿 `viewer` 能走到一组文档，但其中哪些被拉黑了，**从 anne 出发沿 `blocked` 走**只能告诉你“anne 在哪些文档上被拉黑”，要把两个集合相减，就得把两边都完整算出来。经典算法选择了另一条路：只沿第一个操作数 / base 走，把得到的对象都当作**候选**，再逐个用 Check 确认（第 18 章的“剪枝”）。

OpenFGA 的 `docs/list_objects/example_with_intersection_or_exclusion/example.md` 用图演示了这个过程。

## 19.2 代价：每个候选一次 Check

用 mini-openfga 量一下（为了只数次数，把 Check 换成了一个直接给答案的桩函数）：anne 是 `folder:big` 的 viewer，文件夹里有 n 份文档，每 10 份里有 1 份把 anne 拉黑了：

```text
n=100    viewer    objects=100    checks=0
n=100    can_view  objects=90     checks=100
n=1000   viewer    objects=1000   checks=0
n=1000   can_view  objects=900    checks=1000
n=10000  viewer    objects=10000  checks=0
n=10000  can_view  objects=9000   checks=10000
```

`viewer` 一条复核都不需要；`can_view` 对**每一个**候选都要复核，哪怕只有 10% 会被排除。真实的 Check 每次至少要读几次存储（`viewer` 那条 TTU 链、`blocked` 的精确查找），复核的代价随候选数线性增长。在 3 秒的截止时间内，ListObjects 可能只来得及确认一部分，返回不完整的结果（第 18 章）。

这就是 OpenFGA 要重新设计 ListObjects 的原因。

## 19.3 流水线：把模型变成一张数据流图

`internal/listobjects/pipeline/`（约 9700 行，含测试）是 v1.22 默认的 ListObjects 实现，用于“具体用户”的查询（`list_objects.go:576` 的条件，第 18 章）。`doc.go` 里的示意图：

```text
//	Builder.Build(ctx, graph, spec)
//	       │
//	       ▼
//	  DFS walk ─── builds worker graph from authorization model
//	       │
//	       ▼
//	  ┌─────────┐     ┌─────────┐     ┌─────────┐
//	  │ Worker  │◄────│ Worker  │◄────│ Worker  │
//	  │ (root)  │     │         │     │ (leaf)  │
//	  └─────────┘     └─────────┘     └─────────┘
//	       │
//	       ▼
//	  Pipeline.Recv() ─── results streamed to caller
```

思路：

- 遍历带权图（第 27 章），为模型里的**每个节点**建一个 worker，节点之间的边变成 worker 之间的有界管道（`Medium`）；
- 叶子端从用户开始，每个 worker 收到一批对象 ID，通过一个 `Interpreter`（通常是一次存储查询）变换，再广播给下游；
- 根节点的输出就是结果，调用方用 `Recv` 一个一个地取（流式）；
- 管道有容量上限，下游慢时上游自然被阻塞（背压），内存不会因结果集变大而失控。

集合运算由不同类型的 worker 实现（`internal/worker/doc.go`）：

> - `Basic` — union/passthrough with cross-sender deduplication.
> - `Intersection` — outputs only items common to all senders.
> - `Difference` — outputs base items not present in the subtract set.

关键在后两个：它们**直接做集合运算**。`Intersection.Execute`（`internal/worker/intersection.go:58`）为每个输入收集一个“袋子”，所有输入都结束后，从最小的袋子出发求交集：

```go
for index := range len(w.senders) {
	wg.Go(func() {
		...
		w.ProcessSender(ctx, index, w)
		if w.bags[index].Len() == 0 && w.stats[index].SumErrors == 0 {
			cancel()
		}
	})
}
wg.Wait()
...
objMin := len(w.bags[0].Unwrap())
indexMin := 0
for i := 1; i < len(w.bags); i++ {
	bag := w.bags[i].Unwrap()
	if len(bag) < objMin {
		inputs = append(inputs, w.bags[indexMin].Unwrap())
		indexMin = i
		objMin = len(bag)
	} else {
		inputs = append(inputs, bag)
	}
}
output := w.bags[indexMin].Unwrap()
```

注意那个 `cancel()`：任何一个输入为空，交集一定为空，其他输入不用再算了，这是 Check 里 intersection 短路的集合版本。`Difference` 类似：把 subtract 一侧的集合算出来，再从 base 的输出里减掉。不再需要逐个候选去 Check。

数据里的环由 `CycleGroup`（`internal/worker/cycle.go`）处理：环上的 worker 对输入去重，并协调“何时所有消息都处理完了”（quiescence），再按顺序关闭。

## 19.4 一个行为差异：subtract 一侧的条件

“直接算 subtract 的集合”有一个副作用。考虑：

```text
type document
  relations
    define viewer: [user]
    define blocked: [user with small]
    define can_view: viewer but not blocked
condition small(x: int) { x < 10 }

document:d1#blocked@user:u1 with small      （没有任何 viewer 元组）
```

`ListObjects(user:u1, can_view, document)`，请求**不带** context：

- **经典算法**：沿 base（`viewer`）找候选，u1 什么都看不到，没有候选，结果为空。`blocked` 一侧根本没被读到；
- **流水线**：要算出 subtract 的集合，就要读 `blocked` 元组、求值它的条件，而条件缺参数 `x`，于是整个请求失败。

用一个直接调用 OpenFGA `server.ListObjects` 的小程序验证（分别关闭和打开 `pipeline_list_objects`）：

```text
experimentals=[]
  objects=[]
  err=<nil>
experimentals=[pipeline_list_objects]
  objects=[]
  err=rpc error: code = Code(2000) desc = rpc error: code = Code(4000) desc = Internal Server Error
```

两点观察：

1. 两种实现对“缺参数”的处理不同：经典算法只在走到的路径上求值条件，流水线会求值 subtract 一侧的条件；
2. 流水线把条件错误交给 `HandleError("", err)` 处理（`list_objects.go` 流水线分支末尾），被包装成一个内部错误，HTTP 上看到的是 `{"code":"validation_error","message":"Internal Server Error"}`：错误码是 2000，但原始信息丢了。

这个差异是第 31 章的差分测试发现的（它在“带条件、不带 context”的场景里一直报告 ListObjects 的差异），然后才用上面的程序确认原因。mini 实现的是经典算法，行为和 OpenFGA 关闭流水线时一致；差分测试把这一类差异标为“已知”，单独统计。

和第 14 章的结论一样，对调用方的建议也一样：**把条件需要的参数都带上**，这些边界情况就不会出现。

## 19.5 mini-openfga 的选择

mini 只实现了经典算法（第 18 章），没有实现流水线，理由：

- 经典算法在结构上和 Check 是对称的（正向的 `CheckRewrite` 对应反向的边），复核又复用 Check，两三百行代码就能做对，并通过全部 641 条矩阵断言；
- 流水线是一个独立的数据流引擎，worker 类型、背压、环的静默检测、按批处理……它的价值在于大规模数据下的性能，教学上可以作为读源码的材料。

## 19.6 小结

- `and`/`but not` 无法沿着一条边单调扩张，经典算法用“候选 + Check 复核”解决，正确但代价随候选数线性增长；
- 流水线把模型编译成一张由 worker 和有界管道组成的数据流图，用集合运算代替逐个复核，流式输出、天然背压；
- 两种实现在“subtract 一侧的条件缺参数”时行为不同，流水线还会把错误信息变成 “Internal Server Error”。

## 19.7 练习

1. （巩固）对 `define can_edit: editor and can_view`，经典算法沿哪个操作数找候选？如果把定义写成 `can_view and editor`，候选集会变吗？哪种写法更好？
2. （扩展）给 mini 的 ListObjects 加一个优化：当关系是 `X but not Y`、且 Y 是一个只有直接赋值的关系（`[user]`）时，先用一次 `ReadStartingWithUser` 读出用户被 Y 排除的所有对象，再从候选里减掉，不再逐个 Check。这和流水线的 `difference_direct_subtract.go` 有什么相似之处？
3. （读源码）`internal/listobjects/pipeline/pipeline.go` 的 `Build`（:270）怎样决定一个模型节点用哪种 worker？找到 `createWorker`（:155）。

## 19.8 延伸阅读

- `docs/list_objects/example_with_intersection_or_exclusion/example.md`
- `internal/listobjects/pipeline/doc.go`、`internal/listobjects/pipeline/internal/worker/doc.go`
- OpenFGA CHANGELOG 中与 `pipeline_list_objects` 相关的条目
