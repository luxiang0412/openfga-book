# Lab 1：mini-openfga v0.1：用命令行给“云盘”做权限判断

> 目标：把 Part II 的四个组件（DSL 解析器、TypeSystem、存储、朴素 Check）串起来，用命令行回答第 3 章那个云盘场景里的问题；动手改模型，看校验器和 Check 的反应。
> 前置：第 4–8 章。对应代码：mini-openfga **v0.1**。

## 1. 准备

```bash
cd code/mini-openfga
git checkout v0.1
go test ./...
```

```text
ok  	github.com/luxiang0412/mini-openfga/internal/matrix
ok  	github.com/luxiang0412/mini-openfga/pkg/check
ok  	github.com/luxiang0412/mini-openfga/pkg/dsl
ok  	github.com/luxiang0412/mini-openfga/pkg/typesystem
```

v0.1 的目录：

```text
cmd/minifga/        命令行：check、fmt、matrix
pkg/model/          Model、TypeDefinition、Relation、Rewrite（第 4 章）
pkg/dsl/            词法器 + 递归下降解析器（第 4 章）
pkg/typesystem/     索引 + NewAndValidate（第 5 章）
pkg/tuple/          元组格式（第 6 章）
pkg/validation/     三层校验（第 6 章）
pkg/storage/        TupleReader/TupleWriter + memory（第 7 章）
pkg/check/          朴素递归 Check（第 8 章）
pkg/server/         把以上组合成 API（Store、模型、Write、Check）
internal/matrix/    OpenFGA 矩阵测试运行器
examples/drive/     本 Lab 的模型和数据
```

`pkg/server` 是一层很薄的组合：`CreateStore`、`WriteAuthorizationModelDSL`（解析 → 校验 → 保存）、`Write`（校验每条元组 → 写入）、`Check`（校验请求 → 交给 Checker）。命令行和矩阵测试都通过它工作，第 24 章的 HTTP 层也是。

## 2. 第一次运行

`minifga check` 读入模型文件和元组文件（每行一条 `object#relation@user`），在内存里建一个 store，然后逐个回答命令行上的问题：

```bash
go run ./cmd/minifga check -model examples/drive/model.fga -tuples examples/drive/tuples.txt \
  document:design#viewer@user:carol document:design#viewer@user:bob \
  document:design#editor@user:alice document:design#viewer@user:ceo \
  document:design#editor@user:ceo document:design#can_view@user:dave \
  document:design#can_share@user:bob document:design#can_share@user:alice
```

```text
document:design#viewer@user:carol                  true
document:design#viewer@user:bob                    true
document:design#editor@user:alice                  true
document:design#viewer@user:ceo                    true
document:design#editor@user:ceo                    false
document:design#can_view@user:dave                 false
document:design#can_share@user:bob                 true
document:design#can_share@user:alice               false
```

逐条解释：

| 问题 | 结果 | 路径 |
|---|---|---|
| carol 能看吗 | true | 直接元组 `document:design#viewer@user:carol` |
| bob 能看吗 | true | bob 是 owner → owner 是 editor → editor 是 viewer（两次 computed，不算跳数） |
| alice 能编辑吗 | true | alice 是 `folder:eng` 的 editor；design 在 eng-docs 里，eng-docs 在 eng 里：两次 `editor from parent` |
| ceo 能看吗 | true | 第 3 章那条三层的 `viewer from parent` |
| ceo 能编辑吗 | false | 能看不等于能编辑：viewer 包含 editor，反之不成立 |
| dave 能 can_view 吗 | false | dave 通过 root 是 viewer，但他在 `blocked` 里：`viewer but not blocked` |
| bob 能分享吗 | true | `owner and can_view`：bob 是 owner，也能看 |
| alice 能分享吗 | false | alice 能看（editor），但不是 owner |

## 3. 格式化

`minifga fmt` 解析再打印，可以用来规范缩进，也能检验 round trip：

```bash
go run ./cmd/minifga fmt examples/drive/model.fga
```

```text
model
  schema 1.1

type user

type folder
  relations
    define owner: [user]
    define parent: [folder]
    define editor: [user] or owner or editor from parent
    define viewer: [user] or editor or viewer from parent
...
```

## 4. 故意犯错

**模型里引用了不存在的关系。** 把 `can_view` 里的 `blocked` 拼错成 `blockd`：

```text
error: undefined relation: 'document#blockd'
```

这是第 5 章规则 1 的检查，在写入模型时就被拒绝，不会等到 Check。

**元组的类型不在模型里：**

```text
error: Invalid tuple 'document:design#viewer@team:x'. Reason: type 'team' not found
```

第 6 章的 `ValidateTupleForWrite`。

**查询一个不存在的关系**（`viewr`）：

```text
document:design#viewr@user:ceo                     error: relation 'document#viewr' not found
```

查询只做第一层校验（存在性），关系不存在是错误；但如果关系存在、只是不可能成立（比如问 `document:design#parent@user:ceo`），答案是 false 而不是错误。

## 5. 动手改

1. **加一个评论权限**：在 `document` 下加 `define commenter: [user] or editor`，再把 `viewer` 改成 `[user] or commenter or viewer from parent`。写一条 `document:design#commenter@user:erin`，验证 erin 能看但不能编辑。
2. **限制分享**：要求“只有 owner 并且所在文件夹的 owner 也同意”才能分享。提示：`define can_share: owner and owner from parent`。用一条 `folder:eng-docs#owner@user:bob` 让 bob 重新获得分享权限。
3. **制造深度错误**：写一个脚本生成 30 层文件夹链，观察 `resolution depth exceeded`；再改成 20 层，确认能正常回答。

## 6. 跑一次矩阵测试

```bash
go run ./cmd/minifga matrix
```

```text
Check assertions: 325 passed, 191 failed, 0 skipped (63.0%); tests: 78/157 fully passed
```

加 `-v` 可以看到每个失败的断言。按第 8 章的分类，大部分失败来自 userset 和通配符：这是 Part III 的第一个目标。

## 7. 本版小结

| 组件 | 行数（非测试） | 测试 |
|---|---|---|
| `pkg/dsl` | 465 | round trip、条件体、4 种语法错误 |
| `pkg/typesystem` | 479 | 11 个模型，结论与真实 OpenFGA 一致 |
| `pkg/check` | 174 | drive 模型 10 个问题、深度限制 |
| 合计（v0.1 全部非测试 Go 代码） | 2767 | 274 行测试 |

v0.1 是一个**正确但不完整**的授权引擎：它支持的语言子集里的答案都是对的。下一步（Part III）把它变成一个并发的、支持 userset、通配符、环和上下文元组的引擎，目标是让矩阵测试里除了“条件”以外的用例全部通过。
