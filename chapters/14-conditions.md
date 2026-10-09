# 第 14 章 条件元组：把 ABAC 嵌进 ReBAC

> 本章目标：理解条件（condition）的语法与语义；读懂 OpenFGA 如何用 CEL 编译和求值条件，参数怎样从 JSON 转成 CEL 的类型，“缺少参数”怎样被识别；实现 mini 的 `condition` 包，让 Check 矩阵测试全部通过。
> 前置：第 6、7、11 章。对应代码：mini-openfga **v0.3** `pkg/condition/`。

## 14.1 问题：关系有时只在某些条件下成立

第 1 章说过，ReBAC 擅长“谁和什么有关系”，不擅长“在什么情况下”。OpenFGA 在类型限制上加了一个 `with`：

```text
type document
  relations
    define viewer: [user with non_expired, user with in_office]

condition non_expired(current_time: timestamp, expires_at: timestamp) {
  current_time < expires_at
}
condition in_office(user_ip: ipaddress, cidr: string) {
  user_ip.in_cidr(cidr)
}
```

写元组时带上条件名和一部分参数：

```json
{"user": "user:contractor", "relation": "viewer", "object": "document:1",
 "condition": {"name": "non_expired", "context": {"expires_at": "2026-12-31T00:00:00Z"}}}
```

Check 时由请求补上另一部分参数：

```json
{"tuple_key": {"user": "user:contractor", "relation": "viewer", "object": "document:1"},
 "context": {"current_time": "2026-06-01T00:00:00Z"}}
```

这条元组只有在条件表达式为真时才“存在”。参数分两处提供是这个设计的关键：**元组上的参数是授权方定下的**（合同到期日、允许的网段），**请求里的参数是调用方报告的**（现在几点、客户端 IP）。

把同一组请求分别发给真实的 OpenFGA 和 mini-openfga（脚本见本章 14.7 节），得到同样的结论：

| 请求 | OpenFGA | mini |
|---|---|---|
| contractor，`current_time` = 2026-06-01 | true | true |
| contractor，`current_time` = 2027-01-01 | false | false |
| contractor，不带 context | 错误：缺少参数 `current_time` | 错误：缺少参数 `current_time` |
| anne，`user_ip` = 10.1.2.3 | true | true |
| anne，`user_ip` = 192.168.0.1 | false | false |
| anne，`user_ip` = 192.168.0.1，请求里另给 `cidr` = 192.168.0.0/16 | **false** | **false** |
| anne，`user_ip` = not-an-ip | 错误：IP 格式不对 | 错误：IP 格式不对 |

第 6 行值得注意：调用方在请求里提供了一个宽松的 `cidr`，想绕过限制，但无效。原因在 14.3 节。

## 14.2 CEL：编译一次，求值多次

条件表达式用 Google 的 [CEL](https://github.com/google/cel-spec)（Common Expression Language）书写。CEL 是一种非图灵完备、有类型、求值有成本上限的表达式语言，Kubernetes、Envoy、Firebase 都用它。它适合这里的原因：没有循环和递归，不会死循环；有静态类型检查，写模型时就能发现 `x + "a"` 这种错误；可以限制每次求值的“成本”。

OpenFGA 在 `internal/condition/condition.go` 里先建一个共享的基础环境（:28），注册自定义类型 `ipaddress` 和它的方法 `in_cidr`：

```go
func init() {
	var envOpts []cel.EnvOption
	for _, customTypeOpts := range types.CustomParamTypes {
		envOpts = append(envOpts, customTypeOpts...)
	}
	envOpts = append(envOpts, types.IPAddressEnvOption(), cel.EagerlyValidateDeclarations(true))
	env, err := cel.NewEnv(envOpts...)
	...
	celBaseEnv = env
}
```

每个条件编译一次（`compile`，:81）：

```go
for paramName, paramType := range conditionParamTypes {
	envOpts = append(envOpts, cel.Variable(paramName, paramType.CelType()))
}
env, err := celBaseEnv.Extend(envOpts...)
...
ast, issues := env.CompileSource(source)
...
e.celProgramOpts = append(e.celProgramOpts, cel.EvalOptions(cel.OptPartialEval))
prg, err := env.Program(ast, e.celProgramOpts...)
...
if !reflect.DeepEqual(ast.OutputType(), cel.BoolType) {
	return &CompilationError{..., Cause: fmt.Errorf("expected a bool condition expression output, but got '%s'", ast.OutputType())}
}
```

1. 在基础环境上为每个参数声明一个有类型的变量；
2. 编译表达式，类型检查失败（引用了没声明的变量、类型不匹配）就报错；
3. 要求结果类型是 `bool`；
4. 开启 **partial evaluation**（部分求值），下一节解释为什么。

TypeSystem 创建条件对象时还加了成本限制（第 5 章看过）：`WithMaxEvaluationCost(config.MaxConditionEvaluationCost())`，默认 100。一个过于复杂的表达式求值到一半会因超出成本被中止，避免一条元组上的条件拖慢整个服务。

## 14.3 求值：合并上下文、识别缺失参数

一条元组的条件求值在 `eval.EvaluateTupleCondition`（`internal/condition/eval/eval.go:90`）：

```go
contextFields := []map[string]*structpb.Value{{}}
if context != nil {
	contextFields = []map[string]*structpb.Value{context.GetFields()}   // 请求的 context
}
tupleContext := tupleKey.GetCondition().GetContext()
if tupleContext != nil {
	contextFields = append(contextFields, tupleContext.GetFields())    // 元组的 context
}
conditionResult, err := evaluableCondition.Evaluate(ctx, contextFields...)
...
if len(conditionResult.MissingParameters) > 0 {
	return false, condition.NewEvaluationError(
		tupleKey.GetCondition().GetName(),
		fmt.Errorf("tuple '%s' is missing context parameters '%v'", ...),
	)
}
```

`Evaluate`（`condition.go:202`）按顺序合并这些 map，**后面的覆盖前面的**。请求的 context 在前、元组的 context 在后，所以同名参数以**元组上的值为准**。这就是 14.1 节第 6 行的答案：元组上写死了 `cidr: 10.0.0.0/8`，请求传来的 `cidr` 被覆盖，绕不过去。

合并之后，`CastContextToTypedParameters`（:153）把 JSON 值转换成参数声明的类型，然后：

```go
activation, err := e.celEnv.PartialVars(typedParams)
...
var missingParameters []string
for key := range e.GetParameters() {
	if _, ok := activation.ResolveName(key); ok {
		continue
	}
	missingParameters = append(missingParameters, key)
}
...
out, details, err := e.celProgram.ContextEval(ctx, activation)
...
if celtypes.IsUnknown(out) {
	return EvaluationResult{ConditionMet: false, MissingParameters: missingParameters, ...}, nil
}
```

`PartialVars` 允许部分变量“未知”。如果表达式的结果依赖未知变量，CEL 返回一个 Unknown 值，而不是报错或者瞎猜一个 false。这样就能区分：

- 条件**不满足**：参数齐全，表达式为 false → 这条元组不存在；
- 条件**无法判断**：缺参数 → 一个错误，告诉调用方“你少传了 `current_time`”。

把缺参数当成 false 很危险：在 `viewer but not blocked` 里，如果 `blocked` 的条件因缺参数被当成 false，用户就会被放行。

即使表达式因短路求出了结果（`x < 10 || y > 0`，`x = 5`，没给 `y`），只要有参数缺失，OpenFGA 仍然报错。这条规则让行为不依赖于表达式内部的求值顺序。

第 7 章讲过另一半规则：在**同一次读取**里，一条元组的条件无法求值时，如果还有别的可用元组，这个错误会被忽略，那条元组被当作不存在。内联表达式（`$expression`）例外，它的错误总是立刻失败。

### 14.3.1 参数类型与 JSON

请求和元组上的 context 都是 protobuf 的 `Struct`，也就是 JSON。JSON 只有 number、string、bool、null、array、object，而条件参数有 `int`、`uint`、`double`、`timestamp`、`duration`、`ipaddress`、`map<T>`、`list<T>`、`any`。转换规则在 `internal/condition/types/converters.go`：

| 参数类型 | 接受的 JSON 值 | 说明 |
|---|---|---|
| `int` / `uint` | number 或数字字符串 | JSON 的数字是 float64，必须是整数：`1.5` 拒绝；`uint` 不能是负数（`numericTypeConverterFunc`，:19） |
| `double` | number 或数字字符串 | |
| `timestamp` | RFC 3339 字符串 | `timestampTypeConverterFunc`（:92） |
| `duration` | Go 的 duration 字符串，如 `"1h30m"` | |
| `ipaddress` | IP 字符串 | `ipaddressTypeConverterFunc`（:106），IPv4 映射的 IPv6 地址会被还原成 IPv4 |
| `map<T>` / `list<T>` | object / array | 元素逐个按 `T` 转换 |

`ipaddress` 是 OpenFGA 自己注册的 CEL 类型（`types/ipaddress.go:41`），`in_cidr` 方法的实现（`ipaddressCELBinaryBinding`，:135）解析 CIDR 后调用 `netip.Prefix.Contains`。

### 14.3.2 写入时的校验

写入带条件的元组时（第 6 章的 `validateCondition`），要检查：条件名存在且出现在对应的类型限制上（包括 facet 匹配）；元组上携带的 context **只能包含已声明的参数**，并且能转换成声明的类型。这样，写入成功的元组在求值时就不会因为“元组自己的参数类型错”而失败。

### 14.3.3 内联表达式（实验特性）

v1.22 新增了 `$expression`：元组自带表达式，不需要在模型里声明条件。

```text
define editor: [user with $expression]
```

```json
{"condition": {"name": "$expression",
  "context": {"expression": "count < limit", "parameters": {"count": "int", "limit": "int"}}}}
```

没在 `parameters` 里声明的标识符默认为 `string` 类型（`compileInlineExpression`，`inline.go:103`，用 `extractIdents` 找出表达式里的自由变量，跳过推导式绑定的变量）。它需要实验开关 `inline_expressions`：第 24 章用真实 OpenFGA 跑矩阵测试时，没开这个开关，所有内联表达式的用例都在写模型时失败，报错 `$expression requires the "inline_expressions" experimental feature flag`。

## 14.4 mini-openfga 的实现

`pkg/condition` 约 730 行（含注释），结构和 OpenFGA 对应：

| 文件 | 内容 |
|---|---|
| `types.go` | `ParamType`：名字、CEL 类型、JSON→Go 的转换函数；`ParseParamType("map<string>")` |
| `ipaddress.go` | `IPAddress` 实现 CEL 的 `ref.Val`，`ipaddress()` 构造函数和 `in_cidr` 方法 |
| `condition.go` | `Compile`（类型检查 + 要求 bool + partial eval + 成本上限 100）、`Evaluate`、`EvaluateTuple`、按条件对象缓存编译结果 |
| `inline.go` | `$expression`：解析、推断自由变量、按“表达式 + 参数”缓存 |
| `normalize.go` | 把 context 统一成 JSON 的形态 |

`EvaluateTuple` 的合并顺序与 OpenFGA 相同：

```go
// request context first, tuple context second: values stored with the
// tuple cannot be overridden by the caller.
res, err := e.Evaluate(ctx, reqCtx, tupleCtx)
if err != nil {
	return false, err
}
if len(res.MissingParameters) > 0 {
	return false, &EvaluationError{name, fmt.Errorf("tuple is missing context parameters '%v'", res.MissingParameters)}
}
return res.ConditionMet, nil
```

`EvaluationError` 的 `Unwrap` 返回哨兵错误 `ErrEvaluationFailed`，Server 用 `errors.Is` 把它翻译成 `validation_error`（2000）。

接入点只有三处：

1. 写模型时，`typesystem.NewAndValidate` 通过注入的 `ConditionCompiler` 编译每个条件（第 5 章留下的接口）；
2. 校验元组时，`validation.validateCondition` 检查 context 的参数和类型；
3. Check 读出元组后，`check.conditionMet` 求值（内联表达式的错误包成 `storage.FatalError`，第 7 章）。

### 14.4.1 踩坑一：YAML 里的整数不是 JSON 的数字

OpenFGA 收到的 context 来自 protobuf `Struct`，数字一律是 `float64`，转换函数也是按这个写的。但矩阵测试用 YAML 描述 context，`yaml.v3` 把 `x: 10` 解析成 `int`，把未加引号的时间解析成 `time.Time`。直接把这些值交给转换函数，`int` 参数会报“期望 int64，得到 int”。

解决方法是在入口把 context 走一遍 JSON：

```go
// Normalize gives a context map the shape it would have after travelling as
// JSON (OpenFGA receives it as a protobuf Struct): numbers become float64,
// timestamps strings, nested maps map[string]any.
func Normalize(m map[string]any) map[string]any {
	b, err := json.Marshal(m)
	...
	var out map[string]any
	if err := json.Unmarshal(b, &out); err != nil {
		return m
	}
	return out
}
```

这样，无论调用方是 HTTP（本来就是 JSON）、矩阵测试（YAML）还是 Go 代码，引擎看到的数据形态都和 OpenFGA 一样。

### 14.4.2 踩坑二：只有 `-race` 才看得到的 bug

`NormalizeTuples` 最初是**就地**修改调用方传入的元组的：

```go
func NormalizeTuples(keys []*tuple.Key) {
	for _, k := range keys {
		if k.Condition != nil {
			k.Condition.Context = Normalize(k.Condition.Context)
		}
	}
}
```

所有普通测试都通过了，`go test -race` 却报告了数据竞争：

```text
WARNING: DATA RACE
Write at 0x00c000484058 by goroutine 8:
  github.com/luxiang0412/mini-openfga/pkg/condition.NormalizeTuples()
      .../pkg/condition/normalize.go:32
  github.com/luxiang0412/mini-openfga/pkg/server.(*Server).Check()
      .../pkg/server/server.go:248
  github.com/luxiang0412/mini-openfga/internal/matrix.Local.Check()
  ...
```

原因：矩阵测试的 `_ctxTuples` 变体对同一个阶段的多个断言**复用同一组元组对象**作为上下文元组。第一个断言的 Check 提前返回（union 找到了 true）时，被取消的分支还在后台读这些元组；第二个断言的 Check 已经开始就地改写它们。第 10 章说过：**提前返回之后，被取消的 worker 可能还在运行**。

修复：不修改输入，返回副本。

```go
// NormalizeTuples returns copies of keys with normalized condition
// contexts. It never mutates its input: the caller's tuples may still be
// read by goroutines of an earlier, short-circuited query.
func NormalizeTuples(keys []*tuple.Key) []*tuple.Key {
	...
	out := make([]*tuple.Key, len(keys))
	for i, k := range keys {
		c := *k
		if k.Condition != nil {
			c.Condition = &tuple.Condition{Name: k.Condition.Name, Context: Normalize(k.Condition.Context)}
		}
		out[i] = &c
	}
	return out
}
```

## 14.5 一个设计上的注意点：缺参数时，结果取决于策略

第 31 章的差分测试在“带条件、不带 context”的场景里发现，同一份数据、同一个请求，OpenFGA 有时报“缺少参数”，有时返回 false，取决于 Check 选择的求值策略。原因是两种策略**读的元组不同**：

- 默认策略对文档上的每个组派发子问题，读到那个带条件、缺参数的成员元组，报错；
- weight2 策略（第 27 章）一次读出用户所在的**所有**组，其中有一条可用，按第 7 章的规则，另一条缺参数的就被忽略了，于是返回 false。

两种情况下 `true/false` 都不会给错：缺参数的元组要么导致报错，要么被当作不存在，都不会凭空授予权限。但调用方看到的“报错还是拒绝”不是确定的。结论很实际：**调用 Check 时，把模型里所有条件需要的请求参数都带上**。

## 14.6 运行与验证

```bash
cd code/mini-openfga && git checkout v0.3
go test ./pkg/condition/ -v
go run ./cmd/minifga matrix
```

```text
Check assertions: 939 passed, 0 failed, 0 skipped (100.0%); tests: 298/298 fully passed
```

v0.2 的 219 个条件相关的失败全部消失。Check 的矩阵测试从这一版起全部通过。

## 14.7 对照脚本

14.1 节的表格来自一个脚本：用 mini 的 `/dsl/parse` 接口把 DSL 转成 OpenFGA 的 JSON 模型，然后对任意一个 OpenFGA 兼容的服务执行同样的写入和 7 次 Check。在 mini-openfga v1.0 和本地编译的 OpenFGA 上各跑一次即可对照。（脚本依赖第 24 章的 HTTP API。）

## 14.8 小结

- 条件 = 模型里声明的 CEL 表达式 + 元组上的一部分参数 + 请求里的另一部分参数；同名参数以元组为准；
- 条件在写模型时编译（类型检查、必须返回 bool），求值有成本上限；
- 部分求值区分“不满足”和“缺参数”，缺参数是错误而不是 false；
- 参数以 JSON 的形态到达，转换规则决定了 `1.5` 不是 int、时间必须是 RFC 3339；
- 并发代码里，不要修改调用方传入的数据。

## 14.9 练习

1. （巩固）写一个条件 `in_allowed_countries(country: string, allowed: list<string>) { country in allowed }`，元组上存 `allowed`，请求里传 `country`。
2. （扩展）OpenFGA 的条件不能读取对象或用户的属性（比如“文档的密级”）。如何用元组 + 条件 + 上下文元组组合出“用户的级别 ≥ 文档密级”？
3. （读源码）`EvaluateInlineExpression`（`eval.go:26`）只有在表达式**声明了参数**时才把请求的 context 传进去（`if reqCtx != nil && len(ieCond.GetParameters()) > 0`）。为什么？如果表达式不用任何参数，传进去会有什么问题？

## 14.10 延伸阅读

- OpenFGA 文档 *Conditions*：<https://openfga.dev/docs/modeling/conditions>
- CEL 规范：<https://github.com/google/cel-spec>，尤其是 *Partial evaluation* 一节
- `assets/tests/abac_tests.yaml`：33 个条件相关的矩阵测试
