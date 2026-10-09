# Lab 5：mini-openfga v0.5：一个能重启的授权服务

> 目标：用 SQLite 启动 mini-openfga 服务，通过 HTTP 建 store、写模型和元组、做 Check、读变更；重启后确认数据仍在；用 HTTP 客户端在 SQLite 服务上跑全部矩阵测试。
> 前置：第 21–24 章。对应代码：mini-openfga **v0.5**（加载脚本 `examples/lab5/load.sh` 在 v1.0 里，复制到 v0.5 即可使用）。

## 1. 启动

```bash
cd code/mini-openfga && git checkout v0.5
go build -o minifga ./cmd/minifga
./minifga serve -addr 127.0.0.1:8095 -datastore sqlite -sqlite-path lab5.db
```

```text
2026/10/09 09:24:59 mini-openfga listening on 127.0.0.1:8095 (datastore=sqlite)
```

`serve` 的参数（v0.5）：`-datastore memory|sqlite`、`-sqlite-path`、`-check-cache 10s`（开启 Check 缓存）、`-cache-controller 1s`（开启缓存控制器，第 23 章）。

## 2. 加载云盘数据

`examples/lab5/load.sh` 依次调用三个 API：`POST /stores`、`POST /stores/{id}/authorization-models`、`POST /stores/{id}/write`，最后输出 store ID。模型用的是 mini 独有的 `{"dsl": "..."}` 字段；真实的 OpenFGA 只接受 JSON 形式的模型（可以用 `fga model transform` 转换）。

```bash
STORE=$(examples/lab5/load.sh http://127.0.0.1:8095); echo $STORE
```

```text
01M4F46Y5TX889WG59MYR977VJ
```

## 3. Check

```bash
U=http://127.0.0.1:8095
curl -s -X POST $U/stores/$STORE/check \
  -d '{"tuple_key":{"object":"document:design","relation":"can_view","user":"user:alice"}}'
```

| 查询 | 响应 |
|---|---|
| alice can_view design | `{"allowed":true,...,"resolution_metadata":{"dispatch_count":5,"datastore_query_count":25}}` |
| dave can_view design | `{"allowed":false,...,"resolution_metadata":{"dispatch_count":0,"datastore_query_count":6}}` |
| bob can_share design | `{"allowed":true,...,"resolution_metadata":{"dispatch_count":2,"datastore_query_count":15}}` |

`resolution_metadata` 是 mini 加的调试信息。dave 那条没有任何分派、只读了 6 次：`but not blocked` 的两侧并发执行，subtract 一侧是一次精确查找，结果为 true，排除可以直接短路为 false，base 一侧还没来得及分派就被取消了（第 10 章）。这是对计数的解释，不是每次都一样，下一节会看到。

## 4. 读变更

```bash
curl -s "$U/stores/$STORE/changes?page_size=3"
```

```text
TUPLE_OPERATION_WRITE folder:eng#parent@folder:root
TUPLE_OPERATION_WRITE folder:eng-docs#parent@folder:eng
TUPLE_OPERATION_WRITE document:design#parent@folder:eng-docs
token: MDFNNEY0Nlk3OTdN...
```

（输出经过整理。）一共写了 10 条元组，带着 `continuation_token` 继续读就能拿到其余的；读到末尾之后再读，会拿回同一个令牌（第 23 章）。

## 5. 重启

停掉服务，用同一个文件再启动：

```bash
./minifga serve -addr 127.0.0.1:8095 -datastore sqlite -sqlite-path lab5.db
curl -s $U/stores
curl -s -X POST $U/stores/$STORE/check \
  -d '{"tuple_key":{"object":"document:design","relation":"can_view","user":"user:alice"}}'
```

```text
store 01M4F46Y5TX889WG59MYR977VJ drive
{"allowed":true,"resolution":"","resolution_metadata":{"dispatch_count":7,"datastore_query_count":33}}
```

store、模型、元组都还在。两个值得注意的细节：

- 同一个 Check，重启前是 5 次分派、25 次读取，重启后是 7 次、33 次。结果相同，工作量不同：union 的各分支并发执行，谁先返回 true 取决于调度（第 10 章）。重启后的第一次 Check 也要重新编译模型（第 22 章的 TypeSystem 缓存），但这不影响分派次数；
- 看一眼数据库文件：

```text
4096    lab5.db
32768   lab5.db-shm
119512  lab5.db-wal
```

主文件只有 4 KB，数据都在 `-wal` 文件里：WAL 模式下，提交的事务先追加到 WAL，到检查点时才写回主文件（第 21 章）。所以**复制数据库时要连同 `-wal` 一起复制**，或者先做检查点。

## 6. 在 SQLite 服务上跑矩阵测试

```bash
./minifga matrix -http http://127.0.0.1:8095
```

```text
Check        assertions: 939 passed, 0 failed, 0 skipped (100.0%); tests: 298/298 fully passed
ListObjects  assertions: 641 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
ListUsers    assertions: 730 passed, 0 failed, 0 skipped (100.0%); tests: 255/255 fully passed
```

每个测试用例建一个新的 store，所有请求都走 HTTP 和 SQLite。耗时约 6.9 秒；同样的客户端打真实的 OpenFGA（内存存储）是 3.3 秒（第 24 章）。

> 这里我还想顺便验证第 21 章的索引修复，于是在 v1.0 上又跑了两次：7.10 秒和 6.87 秒，v0.5 是 6.84 秒和 6.88 秒，**没有差别**。原因是矩阵测试的每个 store 只有几条到几十条元组，全表扫描和索引查找一样快，时间花在 HTTP 往返和写事务上。索引问题只有在第 21 章那样的大 store 上才看得出来，这也说明了为什么需要专门的基准测试。

## 7. 动手改

1. **缓存与陈旧读**：用 `-check-cache 1m` 启动（不开缓存控制器），先 Check `dave can_view design`（false），再删除 `document:design#blocked@user:dave`，立即再 Check。结果是什么？加上 `"consistency":"HIGHER_CONSISTENCY"` 呢？再用 `-cache-controller 1s` 重复一次。
2. **模型版本**：写入第二个模型（例如去掉 `blocked`），不带 `authorization_model_id` Check dave，再带上第一个模型的 ID Check dave。对照第 22 章的说法。
3. **备份**：服务运行时直接 `cp lab5.db backup.db`（不复制 WAL），用 `backup.db` 启动一个新服务，数据还在吗？再试试 `sqlite3 lab5.db "PRAGMA wal_checkpoint(TRUNCATE)"` 之后复制。

## 8. 本版小结

v0.5 把引擎变成了服务：存储接口和 SQLite 后端（第 21 章）、store/模型/写入（第 22 章）、变更日志、断言、缓存控制器（第 23 章）、OpenFGA 兼容的 HTTP API（第 24 章）。同一个 HTTP 客户端在真实 OpenFGA 和 mini 上都通过了全部 2310 条矩阵断言。

Part VII 给它加上一个浏览器界面。
