---
name: bug-detection-audit
description: C/C++代码深度Bug审计。当前支持：二次释放（double-free）、内存泄露（memory-leak）。支持按业务领域加载专项扩展。当用户要求审计C/C++代码、排查double-free、memory-leak或检查代码bug时自动触发。
argument-hint: <代码目录> [doublefree|memoryleak|all] [domain=logical-decoding]
---

# C/C++ 代码深度 Bug 审计

## 参数

- `$ARG_0` 代码目录路径
- `$ARG_1` 审计类型（默认 `all`）：`doublefree` / `memoryleak` / `all`
- `domain=<业务名>` 业务扩展（可选）：当前支持 `logical-decoding`

## 知识库加载（必须执行）

**Step 1** 始终加载通用前置知识：
```!
cat references/general-reference.md
```

**Step 2** 按 `$ARG_1` 加载专项清单：

| 类型 | 文件 | 清单 |
|------|------|------|
| `doublefree` / `all` | `references/double-free-audit.md` | D1-D8 |
| `memoryleak` / `all` | `references/memory-leak-audit.md` | L1-L10 |

**Step 3** 若指定 `domain=<业务名>`，将 `<业务名>` 替换为实际参数值，执行：

- `domain=logical-decoding` →
```!
cat references/domain-logical-decoding.md
```

## 审计流程

**预扫描**：grep 生成可疑点索引（文件 + 行号）

| 清单 | 扫描关键词 |
|------|-----------|
| D1-D8 | 分配：`palloc/palloc0/palloc_extended/repalloc/MemoryContextAlloc/malloc/calloc`<br>释放：`pfree/pfree_ext/free/CloseTransientFile/fclose`<br>控制流：`return/goto/break/PG_TRY/PG_CATCH/ereport` |
| L1-L10 | 分配：`palloc/palloc0/palloc_extended/palloc0_noexcept/repalloc/MemoryContextAlloc/MemoryContextStrdup/pstrdup/pnstrdup/TextDatumGetCString/AllocSetContextCreate/malloc/calloc/strdup/new`<br>释放：`pfree/pfree_ext/MemoryContextReset/MemoryContextDelete/MemoryContextDeleteChildren/free/delete`<br>控制流：`return/goto/break/continue/PG_TRY/PG_CATCH/PG_RE_THROW/ereport/elog/ThrowErrorData/CHECK_FOR_INTERRUPTS` |

**逐文件审计**：完整读取每个文件，对照已加载清单逐项检查，记录可疑点。

**交叉验证**：读取可疑点上下文（前后 20 行），追踪完整调用链，标注已确认 / 疑似 / 排除。

## 输出格式

```
### BUG-D{序号} [高|中|低] {标题}
- **文件**: xxx.cpp:{行号}
- **描述**: {问题本质}
- **触发条件**: {触发场景}
- **影响**: {崩溃风险等}
- **修复建议**: {具体方案}
- **对比依据**: {参照实现，无则省略}
```

## 审计纪律

1. 预扫描覆盖所有文件，不得跳过
2. 每个函数的错误路径必须单独分析
3. 发现可疑点时，找同模块相似的正确实现对比
4. 清单每个步骤都要执行，不得跳过

## 扩展方法

- 新增 Bug 类型：添加 `references/{type}-audit.md`，在 Step 2 表格注册一行
- 新增业务领域：添加 `references/domain-{业务名}.md`，在参数说明注册一行
