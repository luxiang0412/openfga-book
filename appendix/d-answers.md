# 附录 D 练习参考答案

说明：

- **巩固题**给出答案。能运行的都实际运行过（用 mini-openfga v1.0 或本书使用的 OpenFGA v1.22 服务），标注为“（已验证）”；
- **读源码题**给出出处，引用的行号都在 `526995eb` 上核对过；
- **扩展题**给出思路和要点，除非特别注明，**没有**给出可运行的实现，也没有实测。

---

## 第 0 章

1. 运行器输出 `Check assertions: 325 passed, 191 failed, 0 skipped (63.0%); tests: 78/157 fully passed`（已验证，`notes/outputs/v0.1-matrix.txt`）。注意第 8 章的踩坑记录：这个数字是在一个有 bug 的运行器上得到的。
2. 不含 `web/dist` 和 `package-lock.json` 的统计（已验证）：v0.1→v0.2 +1329/−192，v0.2→v0.3 +1760/−38，v0.3→v0.4 +1300/−76，**v0.4→v0.5 +3359/−167（最大，存储接口、SQLite、HTTP API 一起加入）**，v0.5→v0.6 +1313/−3，v0.6→v1.0 +2823/−68。
3. `assets/assets.go:24` 用 `//go:embed tests/*` 把测试目录嵌进二进制；使用它们的是 `tests/check/check.go`、`tests/listobjects/listobjects.go`、`tests/listusers/listusers.go`（`AGENTS.md` 的 Testing 部分）。

## 第 1 章

1. `team:platform#member@team:infra#member`；模型 `define member: [user, team#member]`，类型限制里必须有 `team#member`，嵌套才能写入并生效（Lab 4 的公司模型就是这样，已验证）。
2. 加一个类型 `system`，`define admin: [user]`；`document` 加 `define system: [system]`、`define viewer: [user] or admin from system`，每份文档写一条 `document:x#system@system:global`（已验证：`system:global#admin@user:root` 让 root 能看两份文档，bob 不能）。不能写 `document:*`：通配符只能出现在 user 一侧，表示“该类型的所有主体”，不能表示“所有对象”。
3. `Tupleset`（对应 `from parent` 的 `parent`）和 `ComputedUserset`（对应 `viewer`），`authzmodel.pb.go:1046–1047`。

## 第 2 章

1. `cmd/run/run.go:799` 用 `rs/cors` 包装 HTTP 处理器；默认 `CORSAllowedOrigins` 是 `["*"]`（`pkg/server/config/config.go:902`），即允许任何来源。
2. 日志拦截器的注释写着 “needed to log invalid requests”（`run.go` 中 `logging.NewLoggingInterceptor` 那一行）。校验拦截器放到它前面，校验失败的请求在到达日志拦截器之前就返回了，这些请求不会被记录。
3. `CheckCommand.maxConcurrentReads`（`pkg/server/commands/check_command.go:37`）被放进 `storagewrappers.Operation.Concurrency`（:134），在 `NewRequestStorageWrapper` 里交给 `NewBoundedTupleReader`（`pkg/storage/storagewrappers/request.go:47`），它用一个容量为该值的信号量限制同时进行的存储读取。

## 第 3 章

1. dave 经 design → eng-docs → eng → root 的 `viewer from parent` 链得到 viewer；`blocked` 直接包含 dave，`but not` 的结果为 false（已验证：`viewer@dave` true，`blocked@dave` true，`can_view@dave` false）。两侧并发执行，blocked 先返回时 base 一侧被取消，读取次数会变少，所以**会**影响 `datastore_query_count`（Lab 5 观察到同一个 Check 读取次数不同）。
2. OpenFGA 返回 `{"allowed":true}`（已验证，对 `can_view` 和 `viewer` 都是）。路径是 design → `viewer from parent` → eng-docs → eng，到达 `folder:eng#viewer@folder:eng#viewer`，`ResolveCheck` 里的 `tuple.IsSelfDefining`（`internal/graph/check.go:434`）认为一个 userset 总是它自己的成员。
3. 深度只在 `dispatch` 里加一（`check.go:383`，recursive 策略里另有一处 `recursive_resolver.go:278`），每次加 1，从 0 开始，不会跳过 25，所以 `==` 足够；`>=` 只是更稳妥。

## 第 4 章

1. 根是 `Difference`：base 是 `Union(This[user], Computed a)`，subtract 是 `Intersection(Computed b, Computed c)`。
2. 已验证的差异：`define a: [user]<制表符># 注释`，**mini 接受，OpenFGA 的解析器拒绝**（`no viable alternative at input '\t'`）。两者对 ` # 注释`（空格）都接受，对 `[user]# 注释`（无空白）都拒绝。
3. `ExitRelationDeclaration`（`transformer/dsltojson.go:270`）在把关系放进当前类型之前检查 `l.currentTypeDef.GetRelations()[relationName] != nil`，存在就报 `'%s' is already defined in '%s'`。mini 在解析器里检查：`pkg/dsl/parser.go:151`，`duplicate relation '%s' in type '%s'`。

## 第 5 章

1. **不能通过**。`a` 的唯一入口在 `[user] and b`，`b` 又只是 `a`：交集要求 b 也有入口，而 b 的入口只能来自 a，形成“潜在循环”。两边都拒绝，信息相同（已验证）：`the definition of relation 'a' in object type 'doc' is invalid: potential loop`。
2. 放在服务层 `pkg/server/models.go`（`MaxTypesPerModel = 100`，超过时报 `exceeded_entity_limit`）。这是一个**写入时的资源限制**，不是模型的语义规则：OpenFGA 同样放在 `WriteAuthorizationModelCommand` 里；TypeSystem 还会被用来加载已存储的模型，在那里做限制会让历史模型无法读取。
3. `typesystem.go:1455`：类型限制里的 `RelationReference` 设置了 `oneof` 但内容为空（`Relation{Relation: ""}` 或 `Wildcard{}` 为 nil），源码注释就是这样描述的。DSL 的语法里 `#` 后面必须有关系名、`:*` 是固定写法，写不出空引用；JSON（protobuf）可以构造任意结构。

## 第 6 章

1. 已验证（mini 和 OpenFGA 结果、信息完全相同）：`user:*` 拒绝（类型限制里没有 `user:*`）；`group:a#owner` 拒绝（`group#owner` 不在限制里）；`group:a` 拒绝（`group` 不是允许的直接类型）；`group:a#member` 接受。
2. tupleset 的含义是“对象的父对象是谁”，父必须是一个具体对象。如果允许 `document:1#parent@folder:x#viewer`，`viewer from parent` 就要先把“folder:x 的所有 viewer”展开成对象集合，再对其中每个对象求 viewer，语义上变成了“父是一组用户”，既无法和 `from` 的含义对应，又会让求值成本不可控。
3. `HasTypeInfo`（`typesystem.go:827`）在 schema 版本不受支持（1.0）或关系没有类型信息（`GetTypeInfo() == nil`）时返回 false；schema 1.0 的模型没有类型限制。

## 第 7 章

1. `[user]`：`ReadUserTuple(store, "document:1", "viewer", "user:anne")`，以及通配符 `ReadUserTuple(..., "user:*")`（如果允许）；`[group#member]`：`ReadUsersetTuples(ReadUsersetTuplesFilter{Object: "document:1", Relation: "viewer", AllowedTypes: [group#member]})`；`viewer from parent`：`Read(ReadFilter{Object: "document:1", Relation: "parent"})`。之后对每个组、每个父对象派发子问题，各自产生新的读取。
2. 调用方（比如条件求值时的 `Normalize`）可能修改返回的元组，与其他并发请求读同一份数据形成数据竞争。测试：两个 goroutine 读同一条元组，一个修改它的 condition context，`go test -race` 会报告（第 14 章遇到的就是这一类问题）。（思路，未实现。）
3. weight2 和 recursive 策略要对两个有序流做“归并求交”，需要先看一眼当前元素而不消费它：`internal/graph/weight_two_resolver.go:334`、`:412` 的 `stream.Head(ctx)`。

## 第 8 章

1. `can_share = owner and can_view`；alice 是 eng 文件夹的 editor，不是 design 的 owner，`owner` 为 false，交集为 false（v0.1 和 v1.0 都已验证）。
2. 失败的会是“union 中一条路有条件错误、另一条路为 true”的用例，比如第 20 章提到的 `relation_through_ttu_with_condition`（一个父文件夹的条件缺参数，另一个满足）。期望 true，改坏后得到错误。（思路，未实测。）
3. 由公式 `eval(b) ∧ ¬eval(s)`：把“s 不确定”当作 s 为真，结果是 false，即**拒绝**；当作假则可能错误地放行。授权系统里错误的拒绝比错误的放行更安全，所以这是更安全的选择。（第 12 章 12.3.1 节补充了一个例外：只通过自身递归的关系里，环不是“不确定”。）

## 第 9 章

1. 根 `document:design#viewer@user:ceo`（Depth 0，Visited={根}）→ union：直接赋值（不派发）、`editor`（同对象，计算关系，不加深度）、`viewer from parent` → 派发 `folder:eng-docs#viewer@user:ceo`（Depth 1，Visited 加入它）→ … → `folder:eng#viewer`（Depth 2）→ `folder:root#viewer`（Depth 3，直接元组 `folder:root#viewer@user:ceo` 为 true）。`editor` 一侧也沿 `editor from parent` 逐层派发，每个分支有自己的 Visited 副本。
2. 被取消前没有执行的闭包不会计数，所以 mini 的计数只包含真正开始的派发，比在构造闭包时计数更接近“实际做了多少工作”。OpenFGA 在 `dispatch` 闭包里加深度、在 `ResolveCheck` 里统计派发，同样是在执行时计数。
3. 解析链是一个环形链表，LocalChecker 可能被缓存、限流、影子解析器包在里面，调用方拿到的是链头。`LocalCheckResolver`（`internal/graph/builder.go:116`）沿 `GetDelegate()` 找，遇到 `ShadowResolver` 取它的 `main`，并防止“委托就是自己”造成的无限循环。

## 第 10 章

1. 容量为 0 时，生产者在发送结果时阻塞，直到有人接收。如果 union 已经因为某个 true 短路返回，其余的生产者就永远阻塞，goroutine 泄漏。场景：`a or b`，a 很快为 true、b 较慢；union 返回后 b 的 goroutine 卡在发送上。
2. subtract 便宜、base 昂贵、而且 subtract 经常为 true 时，先算 subtract 更快；subtract 很少为 true 时，串行化让延迟变成两者之和，比并发更慢。并发执行的好处是延迟取两者中“决定结果”的那一个。（思路。）
3. 测试把一个生产者人为阻塞 300ms（`blockedProducerDelay`），用 `runtime/metrics` 读取进程的用户态 CPU 时间（`/cpu/classes/user:cpu-seconds`），要求这段时间里消耗的 CPU 少于 100ms（`spinCPUBudget`）。如果在已关闭的通道上反复 select，就会在等待期间空转烧 CPU。

## 第 11 章

1. 已验证都为 true。`user:bob`：命中 `doc:1#viewer@user:*`（通配符分支，`checkPublicAssignable`）；`group:a#member`：作为 userset 用户，命中元组 `doc:1#viewer@group:a#member`（直接匹配）。
2. 思路：对象上有大量 userset 元组时，`ReadUsersetTuples` 返回很多行，而 `ReadUserTuple(user:*)` 是一次主键查找。OpenFGA 选择前者是因为它要同时处理带条件的通配符元组和多个通配符类型；基准测试应该能看到两者随 userset 元组数量的差距。（未实测。）
3. `default_resolver.go:269`：`dispatchParams` 非空但 `parentReq` 或 `tk` 为空，只可能是程序错误（构造消息时漏了字段）。返回 `ErrPanic` 表示“不该发生的内部错误”，比静默跳过安全。

## 第 12 章

1. 已验证为 true。子问题树：`a#member@x` → userset 元组 `a#member@b#member` → 派发 `b#member@x` → 直接元组 `b#member@user:x` 为 true（同时 `b#member@a#member` → 派发 `a#member@x`，在路径上已出现，检测到环）。union 中有 true，环不影响结果。
2. 例子：`doc#viewer: [group#member]`，`doc:1#viewer@group:a#member`、`doc:1#viewer@group:b#member`，`group:a#member@group:c#member`、`group:b#member@group:c#member`、`group:c#member@user:x`。两个并发分支都会经过 `c#member@x`；如果 Visited 共享，后到的分支会把它当成环，如果先到的分支恰好被取消，答案就可能变成错误的 false。
3. `cycle_and_cycle_return_false`（`consolidated_1_1_tests.yaml:8163` 起）：`viewer: [user, document#editor] and editor`，`editor: [user, document#viewer]`，两条元组让 viewer 和 editor 互相引用。intersection 的规则：任一操作数为 false 或环，结果为 false。期望 false。

## 第 13 章

1. 模型：`type incident`；`type logs`，`define reader: [user] or oncall from incident`、`define incident: [incident]`；`incident` 上 `define oncall: [user]`。请求：Check `logs:prod#reader@user:anne`，上下文元组 `incident:current#oncall@user:anne`（当前值班表由调用方的排班系统提供，不写入 OpenFGA）。
2. 缓存键必须包含上下文元组和请求的 `context`。`pkg/storage/cache.go:332` 的 `InvariantCacheKey(storeID, modelID, ctx, contextualTuples...)` 正是这样：先复制并排序上下文元组，再和 context 一起哈希。mini 的 `CacheKey` 用 `invariantKey` 做同样的事。
3. `ReadStartingWithUser` 可以要求按对象 ID 升序返回（`WithResultsSortedAscending`），`CombinedTupleReader` 此时用 `NewOrderedCombinedIterator` 把上下文元组和存储结果归并（`combinedtuplereader.go:189`），归并要求两边都有序，所以上下文元组预先排好序。

## 第 14 章

1. 条件 `condition in_allowed_countries(country: string, allowed: list<string>) { country in allowed }`；类型限制 `define viewer: [user with in_allowed_countries]`；元组 `doc:1#viewer@user:anne with in_allowed_countries {"allowed": ["DE", "FR"]}`；请求 `context: {"country": "DE"}`（参数一部分来自元组、一部分来自请求，第 14 章的合并规则）。已验证：`country=DE` 为 true，`country=US` 为 false。
2. 把密级做成对象之间的关系：`type clearance`，用户 `user:x` 是 `clearance:secret#holder`，文档写 `document:1#required@clearance:secret`；“级别 ≥ 密级”用层级关系表达：`clearance:top_secret#implies@clearance:secret` 之类，`define can_read: holder from required`，配合 `holder: [user] or holder from implies_from`。条件适合处理请求时才知道的值（时间、IP），对象属性本身更适合建模成元组。（思路。）
3. 注释（`internal/condition/eval/eval.go` 中 `EvaluateInlineExpression`）解释得很清楚：请求的 context 是整个请求共享的，可能包含给**其他**元组用的字段；没有声明参数的表达式如果收到这些字段，会报“没有定义参数”之类的错误。

## 第 15 章

1. 链的组成依赖按 store 查询的特性开关，例如 `getCheckResolverBuilder(storeID)`（`pkg/server/check.go:455`）里的 `WithOptimizations(featureFlagClient.Boolean(..., storeID))` 和 `WithShadowResolverEnabled(...)`。不同 store 可能有不同的链，所以每个请求构建一次。另外，派发限流器也是在这里按请求创建的（第 15 章的更正）。
2. 键：store ID、模型 ID（或模型无关时省略）、读法和参数（object、relation、允许的类型、用户）、上下文元组和条件 context 的不变键。失效：缓存条目记录写入时间，读取时与 `LastCacheInvalidationTime` 比较，早于它的作废；OpenFGA 的缓存控制器对迭代器缓存还按 `object#relation`、`user + 对象类型` 做部分失效（第 23 章）。（思路。）
3. `ResolveCheckResponse` 包含 `ResolutionMetadata`，调用方会修改它（例如 union 把 `CycleDetected` 置为 true，`check.go:203`、`:277`）。不克隆的话，缓存里的对象会被并发的请求修改，既有数据竞争，又会让后来的命中读到被改过的元数据。

## 第 16 章

1. 已验证（OpenFGA）：根节点是 `difference`，base 是叶子 `{"computed": {"userset": "document:design#viewer"}}`，subtract 是叶子 `{"computed": {"userset": "document:design#blocked"}}`。
2. 思路：BFS，队列里放 `object#relation`，已展开集合去重；对叶子里的 `users` 中形如 `x#y` 的元素和 `computed`、`tupleToUserset` 节点继续调用 Expand；最后得到的具体用户集合应与 ListUsers 一致，但 `and`/`but not` 需要客户端自己做集合运算，这正是 ListUsers 存在的原因。（未实现。）
3. 对 `group:eng#member` 本身调用 Expand（object `group:eng`，relation `member`）。

## 第 17 章

1. 不会全部合并：两项键相同且 context 相同的会合并成一次求值，带不同 context 的那一项单独求值。OpenFGA 的去重键包含 context 和上下文元组（`batch_check_command.go:232` 的 `generateCacheKeyFromCheck` 调用 `InvariantCacheKey(storeID, authModelID, check.GetContext(), check.GetContextualTuples()...)`）；mini 的 `BatchCheck` 也用包含它们的 `CacheKey`（已核对代码）。
2. 只读的话可以共享，节省重复构造；风险在于共享对象内部的可变状态（例如迭代器缓存、读取计数）会在各项之间串扰，单项的 `datastore_query_count` 会变得不准确。（思路。）
3. `pkg/server/batch_check.go` 中，`errors.As(err, &BatchCheckValidationError)` 为真时转成 `serverErrors.ValidationError`（整个请求 400）；单项求值的错误不会让整个请求失败，而是写进该项结果的 `error` 字段。

## 第 18 章

1. 从 `user:alice` 出发：读 `ReadStartingWithUser(object_type=folder, relation=editor, users=[user:alice])` 得到 `folder:eng`；沿 `editor from parent` 反向：`ReadStartingWithUser(object_type=folder, relation=parent, users=[folder:eng])` 得到 `folder:eng-docs`，`ReadStartingWithUser(object_type=document, relation=parent, users=[folder:eng])`（无）；再从 `folder:eng-docs`：`ReadStartingWithUser(object_type=document, relation=parent, users=[folder:eng-docs])` 得到 `document:design`。结果 `[document:design]`（结果已用 CLI 验证；读取参数是推导；文档上 `editor: [user]` 也会查 `ReadStartingWithUser(document, editor, [user:alice])`，无结果。）
2. 思路：把“读完再遍历”改成边读边处理，用计数器在达到 `MaxResults` 时取消上下文、关闭迭代器。大文件夹下，首个结果的延迟和内存占用都会明显下降。（未实测。）
3. `GetPrunedRelationshipEdges` 对交集只取第一个操作数、对排除只取 base，并标记 `RequiresFurtherEval`；`GetRelationshipEdges` 返回全部边。非测试代码只用到剪枝版本（`list_users_rpc.go:331`、`reverse_expand.go:432`），非剪枝版本只在测试里用到。

## 第 19 章

1. 剪枝时交集取**第一个**操作数（`internal/graph/graph.go:298`，`t.Intersection.GetChild()[0]`）。`editor and can_view` 沿 `editor` 找候选；写成 `can_view and editor` 就沿 `can_view` 找，候选集变成所有能看的对象，通常大得多，而且 `can_view` 本身含 `but not`，每个候选都要复核。把**更有选择性、更简单**的操作数放在前面更好。
2. 思路：`Y` 为直接关系时，一次 `ReadStartingWithUser(type, Y, [user, user:*])` 得到被排除的对象集合，从候选里减掉即可，不再逐个 Check。流水线的 `DifferenceDirectSubtract` 也是在 subtract 一侧能直接查到（通配符直连）时，用集合操作代替逐个求值。（未实现。）
3. `createWorker`（`pipeline.go:155`）按节点类型选择：具体类型、通配符、类型#关系节点用基础 worker；运算节点按标签选 `Intersection`、`Difference`（满足条件且开启 `FlagOptimizeSubtract` 时用 `DifferenceDirectSubtract`）、`Union`。顺便一提：判断写成 `b.config.Flags&FlagOptimizeSubtract == 1`，因为 `FlagOptimizeSubtract` 恰好是 `1 << 0` 才成立；如果将来在它前面加一个标志，这个判断会悄悄失效，写成 `!= 0` 更稳妥。

## 第 20 章

1. 已验证（mini 和 OpenFGA 相同）：`{"users":[{"wildcard":{"type":"user"}}]}`，bob 不出现；单独 Check bob 为 false。
2. 10 万个候选、每个一次完整的 Check，成本和延迟都很高。优化：收集候选时记录路径上是否经过 `and`、`but not` 或带条件的元组，没有经过的候选直接确认为结果，只对其余的做 Check。（思路，未实现。）
3. 与第 12 章的规则一致：subtract 一侧有环被当作“不确定”，排除的结果为 false，即“没有人满足”，于是整个排除返回空集。但第 31 章发现这和 OpenFGA 自己的 Check 不一致：只通过自身递归的关系里，Check 不认为那是环。

## 第 21 章

1. 缓存控制器根据 changelog 里最新变更的时间判断缓存是否过期。如果元组先提交、changelog 后写，中间这段时间里读到的“最新变更时间”不包含这次写入，新写入的数据可能被旧缓存遮住；更糟的是，如果 changelog 写入失败，这次变更永远不会触发失效。
2. 要改：占位符（`?` → `$1`）、`INSERT ... ON CONFLICT` 的写法、行值 `IN` 的写法（Postgres 支持）、类型（`BYTEA` 等）、连接池配置。`storagetest.RunAll` 只依赖 `OpenFGADatastore` 接口，可以原样复用。（思路。）
3. 排序：`makeTupleLockKeys` 的注释是 “flattens deletes+writes into a deduped, sorted slice to ensure stable lock order”，所有事务以相同顺序加锁，避免两个事务互相等待对方的锁（死锁）。分批：限制单条 `SELECT` 里的参数数量，避免超过数据库对单条语句参数个数的限制，也让每条语句的规模可控。

## 第 22 章

1. 模型变化后，旧的元组可能不再符合新模型（例如删掉了 `[group#member]` 这个限制）。如果删除也按当前模型校验，这些已经存在、已经无效的元组就再也删不掉了。场景：先写 `doc:1#viewer@group:a#member`，再写一个 `viewer: [user]` 的新模型，然后删除那条元组。
2. 要扩展 `Write` 的选项（OpenFGA 是 `storage.WithOnDuplicateInsert`、`WithOnMissingDelete`）。内存后端：已存在时跳过、不存在时跳过；SQLite：`INSERT ... ON CONFLICT DO NOTHING`、删除不检查 `RowsAffected`。注意 changelog 只应记录实际发生的变更。（思路。）
3. 两者都是用 base64 编码器（`encoder.NewBase64Encoder()`）包装后端返回的令牌；SQLite 后端返回的都是“下一页第一行的 ID”（多查一行判断是否有下一页）：模型按 ID 倒序（从新到旧），store 按 ID 正序。

## 第 23 章

1. 不会漏：令牌之后的变更都会被读到（前提是 changelog 和元组同事务写入，第 21 章）。会重复：处理完一批、还没保存新令牌就崩溃，重启后会再处理一次。让处理幂等：按“元组 + 操作”的结果写入（写入已存在的忽略、删除不存在的忽略），或者记录已处理的变更 ULID。
2. 思路：`start_time` 非空且没有令牌时，用 `ulid.New(ulid.Timestamp(t), nil)` 生成一个只有时间部分的 ULID 作为起点（OpenFGA 的 `ReadChangesQuery.Execute` 就是这样）。测试：在两次写入之间记一个时间，带这个时间读，只应读到第二次写入。（未实现。）
3. `invalidateIteratorCacheByObjectRelation` 对应正向的读法（`Read(object#relation)`、`ReadUsersetTuples`、`ReadUserTuple`，按对象和关系定位）；`invalidateIteratorCacheByUserAndObjectType` 对应反向的 `ReadStartingWithUser`（按用户和对象类型定位）。见 `cache_controller.go:370`、`:376`。

## 第 24 章

1. 客户端自己对协议的理解可能有误（JSON 字段名、包装层级、错误码），如果只在 mini 上跑，两边可能“一起错”而测试照样通过。在真实 OpenFGA 上通过，证明客户端是对的，再拿它测 mini 才有意义。第 23 章断言里的 `contextual_tuples` 就是这类问题的例子。
2. 思路：处理函数设置 `Content-Type`，ListObjects 通过回调或通道每得到一个确认的对象就写一行 JSON 并 `Flush()`；第 18 章的实现需要把“收集到切片再返回”改成“产出一个、发送一个”，并在客户端断开时取消上下文。（未实现。）
3. `network, address := tcp, config.GRPC.Addr`（`run.go` 中 `config.HTTP.Enabled` 分支的开头），回退时 `goto DIAL_SERVER` 跳过 UDS，直接用这个 TCP 地址；`dialLocalGrpc`（:681）会把未指定的主机（`0.0.0.0` 或空）换成 `localhost`。

## 第 25 章

1. 模型以结构化的 JSON（protobuf）存储，DSL 是从结构重新生成的，注释、空行、缩进都不在结构里。要保留，得把 DSL 原文和模型一起存；OpenFGA 的 `WriteAuthorizationModel` 只接受结构化模型，做不到（这也是 FGA CLI 和 VS Code 插件把 `.fga` 文件放在代码仓库里管理的原因）。
2. 思路：错误类型里带上类型名和关系名（mini 的校验错误信息里已经有，例如 `undefined relation: 'document#banned'`），前端在 DSL 文本里定位 `type <类型>` 之后第一个 `define <关系>` 的行号。（未实现。）
3. 默认 `false`（`config.go:931`，已验证）。`net.Dial` 有两个作用：确认 HTTP 服务已经起来（带指数退避，最多 3 秒）；用 `conn.RemoteAddr().String()` 得到实际的地址填进模板里的 `fga_api_host`。

## 第 26 章

1. 已验证：根节点变为 false，`viewer@dave` 也变为 false。base 已经是 false，`but not` 的结果与 subtract 无关，`blocked` 子树不再重要（它仍然是 true）。
2. 思路：用一个 map 记录已求值的 `object#relation@user` → 结果节点；再次遇到时插入一个引用节点。结果为 `cycle` 的节点**不能**复用，它取决于到达它的路径；第 31 章的 `PathDependent` 结果同理。（未实现。）
3. 追踪多了时间（每个 span 的耗时）、并发关系和存储查询的细节；少了被短路取消的分支（它们只有开始没有结果，或根本没有开始），也就看不到“所有可能的路径”。（思路。）

## 第 27 章

1. ∞（OpenFGA 的带权图：`doc#viewer user=inf`、`team#member user=inf`，已验证）。weight2 不能用：它要求 userset 节点对用户类型的权重恰好为 1，`team#member` 是 ∞（团队可以嵌套，“用户直接所在的团队”不等于“用户所在的所有团队”）。
2. 思路见题目；注意每层的去重（已访问的组）、深度上限，以及和对象一侧组集合的提前相交。（未实现。）
3. 要点：权重 ∞ 说明确实递归；递归关系必须是它自己，说明只有一种递归方式；不能是元组环的一部分，因为跨关系的环里集合运算的“自身展开”不再成立；只有一条边指向 OR 节点，保证结构是并集；TTU 边计算的关系相同，保证是“同一个关系沿父链递归”；其他边权重为 1，保证非递归部分一跳就能判定。

## 第 28 章

1. 已验证（mini 的规划器，5 次运行，每 50 轮统计 weight2 被选次数）：场景 B 中 default 的先验改成 10ms 后，**weight2 一次都没有被选中**（`0 0 0 0 0 0 0 0`）；顺带测了场景 A（weight2 真的更快），仍然很快收敛到 weight2（`47 50 50 50` 之类，只是前 50 轮多选了几次 default）。先验低到和实际接近时，探索代价几乎为零。
2. 思路：λ 封顶相当于只记住最近约 50 个观察，场景 C 的转向预计在约 50 轮内完成；场景 A 不受影响，因为一直观察到的就是同一个值。（未实测。）
3. 每个策略各抽一个样本，选最小值，与遍历顺序无关；只有当两个抽样值**完全相等**时，先遍历到的那个获胜，此时选择取决于随机的遍历顺序。连续分布下这几乎不会发生。

## 第 29 章

1. `ConstantTimeCompare` 返回 0 或 1，`|=` 之后只要有一个匹配就是 1，否则是 0，结果只能是 0 或 1。两个相同的密钥都匹配时，`1|1` 仍是 1，不影响结果。
2. 思路：中间件从 `r.PathValue("store_id")` 或路径前缀取 store ID；`ListStores` 只返回该密钥对应的 store（需要服务层配合过滤）；`CreateStore` 需要一个单独的管理密钥。这实际上是在用配置文件实现第 29 章的访问控制。（未实现。）
3. 两个模块超过了 `MaxModulesInRequest = 1`，`Authorize` 返回内部错误 `the principal cannot write tuples of more than 1 module(s) in a single request (modules in request: 2)`（`authz.go:230`）；但服务层把所有授权失败都转成统一的 `ErrUnauthorizedResponse`（`server.go:1140`、`:1218`），调用方看到的是 `the principal is not authorized to perform the action`（forbidden）。

## 第 30 章

1. 每个不同的标签值组合都是一条时间序列。1 万个 store × 约 10 个 API × 若干状态码 × 直方图的十几个桶，就是上百万条序列，Prometheus 会被拖垮；而且路径里的 store ID 对聚合毫无意义。
2. 思路：在 `CachedCheckResolver` 上已有 `Hits`/`Misses` 计数，再加一个“命中但已失效”的计数，注册成 `CounterFunc`；在 `TestCacheInvalidation` 中断言：第一次 Check 后 misses=1，陈旧读 hits=1，控制器刷新后 invalid hits=1。（未实现；v1.0 的 Docker 端到端脚本已经导出了 `minifga_check_cache_hits_total` 和 `misses_total`。）
3. 配置项 `RequestDurationDatastoreQueryCountBuckets` 和 `RequestDurationDispatchCountBuckets`，默认都是 `["50", "200"]`（`config.go:879–880`；`server.go:917` 的默认值 `[]uint{50, 200}`），所以标签值只有 `50`、`200`、`+Inf` 三种。

## 第 31 章

1. 递归的语义是最小不动点。从进入递归的节点出发做深度优先遍历，把“回到路径上已有的节点”当作 false：任何真正的成员都能通过一条**不回头**的路径到达，这条路径会被其他分支找到，所以入口节点的结果是精确的，exclusion 拿到的正是这个结果。但递归内部的节点（a、b、c 例子中的 b）在求值时“回到 a”被截断，它自己的 false 只是低估；单独问 b 时答案是 true。缓存按子问题存储，存下 b 的 false 就会在之后的单独查询中返回错误答案。
2. 预测：`viewer: [user] or viewer from parent` 满足 `TTUUseRecursiveResolver` 的条件，文件夹成环、subtract 一侧经过这种环时，会出现第三类差异（OpenFGA 的 ListUsers 返回空，而它的 Check 为 true）。（未实测。）
3. 例：主实现走 weight2，一条带条件的元组缺参数被“once-valid”规则忽略，返回 false；影子实现（例如另一种策略）读到那条元组并报缺参数错误（第 14 章：缺参数报错与否取决于策略）。这种差异不是“结果不同”而是“影子出错”，只记 warn 日志，避免把已知的策略差异当成告警，同时不影响主路径。
