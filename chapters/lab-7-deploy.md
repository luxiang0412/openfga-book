# Lab 7：mini-openfga v1.0：一条命令部署，一个脚本验收

> 目标：用 Docker Compose 部署 v1.0（SQLite 持久化、预共享密钥、缓存与缓存控制器、求值优化），然后用一个端到端脚本完成验收：认证、全部矩阵测试、指标、Playground、重启后数据仍在。
> 前置：第 21–31 章。对应代码：mini-openfga **v1.0** `Dockerfile`、`docker-compose.yaml`、`scripts/e2e.sh`。

## 1. 镜像

`Dockerfile` 分三个阶段：

```dockerfile
# Stage 1: build the Playground (web/dist is also committed, so this stage
# only matters when the UI changed; it keeps the image reproducible).
FROM node:22-alpine AS web
...
RUN npm run build

# Stage 2: build a static Go binary with the UI embedded.
FROM golang:1.26-bookworm AS build
...
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/minifga ./cmd/minifga

# Stage 3: minimal runtime image, non-root, like OpenFGA's chainguard/static base.
FROM alpine:3.20
RUN adduser -D -u 10001 fga && mkdir /data && chown fga /data
...
HEALTHCHECK --interval=5s --timeout=2s --retries=10 CMD wget -qO- http://127.0.0.1:8080/healthz || exit 1
ENTRYPOINT ["minifga", "serve"]
CMD ["-addr", ":8080", "-datastore", "sqlite", "-sqlite-path", "/data/fga.db", "-check-cache", "10s", "-cache-controller", "1s", "-check-optimizations"]
```

- 纯 Go 的 SQLite（`modernc.org/sqlite`）让 `CGO_ENABLED=0` 成为可能，二进制是静态的；
- 以非 root 用户运行，数据在卷 `/data` 上；
- 默认参数打开了第 15、23、27 章的全部功能：Check 缓存 10 秒、缓存控制器 1 秒、优化。

构建出的镜像是 29.6MB。

> 第一个阶段的注释说“web/dist 也提交在仓库里”，在第 25 章的修复之前这句话并不成立，而正是这个 Node 阶段掩盖了问题：Docker 构建总会重新生成前端，所以部署一直是好的，只有在干净的克隆里直接 `go build` 才会失败。

`docker-compose.yaml` 只有一个服务和一个卷，密钥从环境变量 `MINIFGA_AUTHN_PRESHARED_KEYS` 读取，为空时不开启认证：

```bash
MINIFGA_AUTHN_PRESHARED_KEYS=secret docker compose up -d --build
```

## 2. 验收脚本

`scripts/e2e.sh` 依次做：

1. `docker compose up -d --build`，等 `/healthz` 返回；
2. **认证**：不带密钥请求 `/stores` 应为 401，带上应为 200；
3. **全部矩阵测试**：`minifga matrix -http http://127.0.0.1:8080 -token ...`，即第 24 章那个同时在真实 OpenFGA 上验证过的 HTTP 客户端，现在打容器里的 mini（SQLite、缓存、优化全开）；
4. 建一个 store、写模型和一条元组；
5. **指标**：读取 `/metrics` 里缓存、规划器和 Check 路由的计数；
6. **Playground**：`/playground/` 返回页面；
7. **重启**：`docker compose restart`，再 Check 刚才那条元组，应为 true；
8. 退出时 `docker compose down -v` 清理容器和卷。

## 3. 运行

```bash
cd code/mini-openfga && git checkout v1.0
bash scripts/e2e.sh
```

在 v1.0 最终版本上的输出（略去 Docker 构建过程）：

```text
== authentication
GET /stores without key -> 401
GET /stores with key    -> 200
== OpenFGA matrix tests over HTTP
Check        assertions: 939 passed, 0 failed, 0 skipped (100.0%); tests: 298/298 fully passed
ListObjects  assertions: 641 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
ListUsers    assertions: 730 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
== write a store that must survive a restart
== metrics
minifga_check_cache_hits_total 801
minifga_check_cache_misses_total 1763
minifga_planner_selections_total{strategy="default"} 36
minifga_planner_selections_total{strategy="weight2"} 516
minifga_request_duration_seconds_count{code="200",route="POST /stores/{store_id}/check"} 855
== playground
<title>mini-openfga Playground</title>
== restart
{"allowed":true,"resolution":"","resolution_metadata":{"dispatch_count":0,"datastore_query_count":1}}

E2E OK
```

全程约 20 秒。几个数字的含义：

- **缓存**：801 次命中、1763 次未命中。矩阵测试的每个用例都建一个新 store，缓存键又包含 store ID，所以命中只能来自同一个 store 内部重复出现的子问题（同一阶段的多条断言、断言之间共享的子问题）；具体比例本书没有逐一拆分。带环标志或 `PathDependent` 的结果不进缓存（第 12、15 章）；
- **规划器**：weight2 被选了 516 次，default 36 次。和第 28 章的模拟一致：weight2 的先验偏向它，矩阵测试的数据量小，它的两次查询总是很快，default 只在探索时偶尔被选中；
- **路由标签**：`route="POST /stores/{store_id}/check"` 这一行在第 30 章修复之前是不会出现的（所有请求都是 `route="/"`）。脚本只是打印这些行，不会因为缺少某一行而失败，这是它的一个弱点（练习 2）；
- **重启后**：Check 只读了 1 次存储，没有派发，数据来自 SQLite 卷。

## 4. 动手改

1. **不开优化**：把 `CMD` 里的 `-check-optimizations` 去掉重新跑。规划器的计数会变成什么？矩阵测试还都通过吗？
2. **让脚本更严格**：第 3 步的指标检查只是 `grep` 后打印。改成断言：路由标签那一行必须存在，`weight2` 的计数必须大于 0。然后临时撤销第 30 章的修复，确认脚本会失败。
3. **备份**：容器运行时，用 `docker compose exec` 在容器里执行 SQLite 的检查点，再把 `/data/fga.db` 复制出来（Lab 5 的练习 3）。alpine 镜像里没有 `sqlite3` 命令，你会怎么做？

## 5. 本版小结

v1.0 在 v0.6 的基础上加入了：带权图与 PathExists、weight2 策略与 Thompson 采样规划器（第 27、28 章）、预共享密钥认证（第 29 章）、请求日志与 Prometheus 指标（第 30 章）、对着真实 OpenFGA 的差分测试（第 31 章），以及一条命令的 Docker 部署。

写后面几章时，对照 OpenFGA 又修了一批问题，都带着回归测试进了 v1.0：SQLite 查询计划（第 21 章）、store 名称校验（第 22 章）、模型 ID 响应头（第 24 章）、Playground 的分页、语义校验、版本下拉框和解析树（第 25、26 章，Lab 6）、`web/dist` 未提交与 CRLF（第 25 章）、预共享密钥的时序（第 29 章）、路由标签回归（第 30 章），以及自递归关系里环的语义（第 12、31 章）。

全书到这里结束。附录 A 是 OpenFGA 源码地图，附录 B 是术语表，附录 C 把 OpenFGA 和 SpiceDB、Ory Keto、Casbin、OPA 放在一起比较，附录 D 是练习的参考答案。
