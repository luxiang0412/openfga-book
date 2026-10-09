# 第 29 章 认证、API 授权与多租户

> 本章目标：分清三件事：谁在调用 API（认证）、调用方能对哪个 store 做什么（API 授权）、怎样用 store 隔离租户；读懂 OpenFGA 的预共享密钥与 OIDC 认证，以及“用 FGA 保护 FGA”的访问控制实验特性；修复 mini 预共享密钥比较中一个可以测量的时序泄露。
> 前置：第 22、24 章。对应代码：mini-openfga **v1.0** `pkg/httpapi/middleware.go`。

## 29.1 三件事

OpenFGA 是授权服务，但它自己的 API 也需要被保护：

1. **认证**：这个请求来自谁？OpenFGA 支持三种方式：`none`（默认）、`preshared`（预共享密钥）、`oidc`（JWT）；
2. **API 授权**：这个调用方能不能对 store X 调用 Write？默认没有这一层，认证通过就什么都能做；
3. **多租户**：不同租户的数据怎样隔离？OpenFGA 的隔离单元是 store（第 22 章）。

认证在 gRPC 拦截器里完成（第 24 章：HTTP 请求也经过同一条拦截器链），结果作为 `authclaims.AuthClaims` 放进请求上下文，后面的 API 授权从这里取调用方身份。健康检查服务实现了 `AuthFuncOverride`（`pkg/server/health/health.go:27`），不需要认证；指标在单独的端口（默认 `0.0.0.0:2112`，`config.go:941`）。

## 29.2 预共享密钥

最简单的方式：启动时配置一组密钥（`--authn-method preshared --authn-preshared-keys k1,k2`），请求带 `Authorization: Bearer <key>`。`internal/authn/presharedkey/presharedkey.go` 只有几十行，但值得逐行读：

```go
type PresharedKeyAuthenticator struct {
	// validKeyHashes holds SHA-256 digests of each configured key. Storing
	// fixed-length hashes lets Authenticate use subtle.ConstantTimeCompare
	// without the length-dependent early-return that would otherwise leak
	// configured key lengths through timing.
	validKeyHashes [][sha256.Size]byte
}
...
	// Hash the presented token and compare against every configured key
	// hash without early-return. All inputs to ConstantTimeCompare are
	// fixed-length, so total time depends only on the number of configured
	// keys — not on which (if any) matched, nor on configured key lengths.
	tokenHash := sha256.Sum256([]byte(authHeader))
	var matched int
	for _, kh := range pka.validKeyHashes {
		matched |= subtle.ConstantTimeCompare(tokenHash[:], kh[:])
	}
	if matched == 1 {
		...
	}
	return nil, authn.ErrUnauthenticated
```

两个细节：

- `subtle.ConstantTimeCompare` 只在**长度相同**时是常数时间，长度不同时立即返回 0。所以先把密钥和请求里的令牌都哈希成 32 字节，再比较；
- 和**每一个**密钥比较，用 `|=` 合并结果，不在匹配时提前返回，否则耗时会暴露“匹配的是第几个密钥”。

### 29.2.1 踩坑：mini 的比较泄露了密钥长度

对照这段代码，mini v1.0 原来的实现是：

```go
for _, k := range keys {
	if subtle.ConstantTimeCompare([]byte(token), []byte(k)) == 1 {
		next.ServeHTTP(w, r)
		return
	}
}
```

正好是上面注释说要避免的两件事。为了确认这不只是理论问题，写了一个微基准：攻击者发送一个固定的 32 字节令牌，**变化的是服务端密钥的长度**：

```text
BenchmarkRawKey32-24       	20000000	        12.74 ns/op
BenchmarkRawKey32-24       	20000000	        11.48 ns/op
BenchmarkRawKey32-24       	20000000	        11.50 ns/op
BenchmarkRawKey48-24       	20000000	         1.572 ns/op
BenchmarkRawKey48-24       	20000000	         0.7832 ns/op
BenchmarkRawKey48-24       	20000000	         1.756 ns/op
BenchmarkHashedKey32-24    	20000000	        56.18 ns/op
BenchmarkHashedKey32-24    	20000000	        56.24 ns/op
BenchmarkHashedKey32-24    	20000000	        56.29 ns/op
BenchmarkHashedKey48-24    	20000000	        56.19 ns/op
BenchmarkHashedKey48-24    	20000000	        56.29 ns/op
BenchmarkHashedKey48-24    	20000000	        56.23 ns/op
```

直接比较时，令牌长度和密钥长度相同要约 11.5ns，不同只要约 1.5ns：攻击者逐个尝试不同长度的令牌，就能找出密钥的长度。哈希后比较，两种密钥长度都是 56.2ns。

> 我第一次写这个基准时，变化的是**令牌**的长度，结果哈希版本也有差别（63ns 对 83ns），一度以为哈希也不能解决问题。其实那是因为 64 字节的输入需要多算一个 SHA-256 块：耗时取决于攻击者自己发送的令牌，攻击者本来就知道它的长度，不泄露任何秘密。要测的是“耗时是否依赖**秘密**”，所以重做时固定令牌、改变密钥。

几纳秒的差别隔着网络很难利用，但这类问题的修复成本几乎为零，而且是纵深防御的一部分。mini 的修复与 OpenFGA 相同（`PresharedKeyAuth`），测试覆盖了两个有效密钥、前缀、多一个字符、缺少 `Bearer`、`Basic` 方案等情况。

## 29.3 OIDC

`internal/authn/oidc/oidc.go`：调用方先从身份提供方（Auth0、Keycloak……）拿到 JWT，作为 Bearer 令牌发送。`RemoteOidcAuthenticator.Authenticate`（:92）：

- 用签发方发布的公钥集（JWKS，`fetchJWK`，:182，定期刷新）验证签名；
- 验证 `iss`（签发方，允许配置别名）、`aud`（受众）；可选地限制 `sub`（主题）；
- 从配置的声明（`clientIDClaims`，默认依次尝试 `azp`、`client_id`）里取出**客户端 ID**，放进 `AuthClaims`。

客户端 ID 是下一节 API 授权的依据。mini 没有实现 OIDC：它需要一个身份提供方才能演示，而对理解授权引擎帮助不大。

## 29.4 用 FGA 保护 FGA

“调用方 A 只能写 store X，调用方 B 只能读 store Y”，这本身就是一个授权问题。OpenFGA 的实验特性 `enable-access-control`（`internal/authz/authz.go`）的做法是：**用一个专门的 OpenFGA store 来描述谁能调用哪个 API**，每个请求先在这个 store 上做一次 Check。

- 启动参数 `--access-control-enabled --access-control-store-id ... --access-control-model-id ...`，三者必须一起给（`run.go:160`）；只能和 OIDC 一起用，否则启动报错（`server.go:1117`）：“Access control is only supported with OIDC authentication”；
- 每个 API 方法对应一个关系（`getRelation`，:147）：Check 和 BatchCheck → `can_call_check`，Write → `can_call_write`，CreateStore → `can_call_create_stores`……；
- `Authorize`（:193）以 OIDC 的客户端 ID 为用户（`application:<client id>`），在访问控制 store 上 Check `store:<store id>#<关系>`，并带一条上下文元组 `store:<id>#system@system:fga`（`getSystemAccessTuple`，:474），让模型可以表达“系统管理员对所有 store 有权限”；
- 更细的粒度：**模块**。DSL 的模块化模型里，类型可以属于某个模块（比如 `issue-tracker`），Write 时根据写入元组的类型算出涉及哪些模块（`GetModulesForWriteRequest`，:322），调用方即使没有整个 store 的写权限，只要有某个模块的写权限，也能写这个模块的类型。一次请求最多涉及 1 个模块（`MaxModulesInRequest`）；
- ListStores 只返回调用方有权限的 store（`ListAuthorizedStores`，:283，内部是一次 ListObjects）。

这是一个很漂亮的自举：授权引擎的所有能力（usersets、TTU、上下文元组）都可以用来表达“谁能管理授权数据”。

## 29.5 多租户

在 OpenFGA 上做多租户，常见两种方式：

| 方式 | 做法 | 优点 | 缺点 |
|---|---|---|---|
| 每个租户一个 store | 租户 ID ↔ store ID | 数据完全隔离；模型可以各自演进；删除租户就是删 store | store 多了之后管理成本高；不同租户共享的数据（例如平台管理员）需要复制 |
| 共享 store，租户是一个类型 | `type tenant`，所有对象通过 `tenant` 关系挂到租户上 | 一个模型、一处管理；跨租户的关系可以表达 | 隔离靠模型的正确性；一个写错的规则可能跨租户泄露 |

无论哪种，**API 层的隔离**都需要上一节的访问控制或者调用方自己的网关：预共享密钥是全局的，拿到密钥就能操作所有 store。

mini 只实现了预共享密钥，和 OpenFGA 的 `preshared` 模式一样是全有或全无。几个和 OpenFGA 不同的地方：

- `/healthz` 和 `/metrics` 不需要认证；OpenFGA 的指标在单独的端口上，mini 为了部署简单和 API 共用一个端口。指标里没有元组数据，但有路由和延迟信息，生产环境应该用网络策略限制访问；
- `/playground/` 的静态页面不需要认证，页面里的 API 请求需要（第 25 章：遇到 401 时弹窗要密钥）；而 OpenFGA 的本地 Playground 只支持 `none` 认证（第 25 章）。

## 29.6 运行与验证

```bash
cd code/mini-openfga && git checkout v1.0
go test -run TestPresharedKey -v ./pkg/httpapi/
MINIFGA_AUTHN_PRESHARED_KEYS=k1,k2 ./dist/minifga serve -addr 127.0.0.1:8080 &
curl -s -o /dev/null -w "%{http_code}\n" 127.0.0.1:8080/stores                          # 401
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer k2" 127.0.0.1:8080/stores   # 200
./dist/minifga matrix -http http://127.0.0.1:8080 -token k1
```

Lab 7 的 Docker 端到端脚本也覆盖了这两种情况。

## 29.7 小结

- 认证、API 授权、租户隔离是三个层次；OpenFGA 默认只有第一层（而且默认是 `none`）；
- 预共享密钥的比较要对哈希做、和所有密钥比较、不提前返回；mini 原来的实现泄露了密钥长度，已按 OpenFGA 的方式修复，修复前后的耗时可以测出来；
- OIDC 提供调用方身份（客户端 ID），访问控制特性用一个 OpenFGA store 来授权 OpenFGA 的 API，粒度可以到模块；
- 多租户可以按 store 隔离，也可以在一个 store 里用类型建模，前者隔离更强。

## 29.8 练习

1. （巩固）为什么 `matched |= ...` 之后判断 `matched == 1` 是安全的？如果两个配置的密钥相同会怎样？
2. （扩展）给 mini 实现“每个 store 一个密钥”：密钥配置成 `storeID:key` 的形式，中间件从路径里取出 store ID 再比较。ListStores、CreateStore 这类不针对某个 store 的 API 怎么办？
3. （读源码）`Authorize`（`authz.go:193`）先做 store 级别的 Check，失败后才检查模块。如果一个 Write 同时写了两个模块的类型，而调用方只有其中一个模块的权限，会得到什么错误？

## 29.9 延伸阅读

- OpenFGA 文档 *Configure OpenFGA → Authentication*
- `docs/security-self-assessment.md`：OpenFGA 的安全自评
- OpenFGA RFC / 文档 *Access Control*（实验特性）
