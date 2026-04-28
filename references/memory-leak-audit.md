# 内存泄露审计清单 L1-L10 (必须逐条仔细检查)

本文基于 openGauss 源码中的内存管理模型编写。审计时先判断资源由谁拥有：当前 `MemoryContext`、显式 `MemoryContext`、结构体字段、链表/哈希表、全局长期对象、调用者，还是 C/C++ 运行库。只有确认生命周期没有被 context/reset/delete 或所有权转移覆盖后，才能判定为泄露。

## openGauss 内存模型要点

- `palloc`、`palloc0`、`pstrdup`、`pnstrdup` 默认分配到 `CurrentMemoryContext`；`MemoryContextAlloc`、`MemoryContextStrdup` 分配到显式 context。
- `palloc_extended`、`palloc0_noexcept` 可能返回 `NULL`；普通 `palloc` 类接口失败时通常 `ereport(ERROR)` 跳转，不会按 C 返回路径继续执行。
- `MemoryContextSwitchTo` 只切换当前 context，不释放任何对象；每个切换点都要检查所有 `return`、`goto`、`ereport(ERROR)`、`PG_CATCH` 是否恢复旧 context 或删除临时 context。
- `AllocSetContextCreate` 创建的 context 是资源本身；必须能到达 `MemoryContextDelete`、`MemoryContextReset`、父 context 生命周期结束，或明确挂到长期父 context。
- `pfree` 只释放单个 chunk；分配在专用 context 内的一组对象通常应通过 `MemoryContextDelete/Reset` 释放，不能只看局部变量没有 `pfree` 就判泄露。
- `ereport(ERROR)`、`elog(ERROR)`、`PG_RE_THROW()` 会 `siglongjmp`；错误点之后的清理语句不执行。PG_TRY 内分配的非自动清理资源必须在 PG_CATCH 或上层 owner 中释放。

## L1 错误路径绕过显式释放

同一函数中先分配资源，再经过 `ereport(ERROR)`、`elog(ERROR)`、`PG_RE_THROW()`、`CHECK_FOR_INTERRUPTS()` 或会抛 ERROR 的子函数，若错误点之前的资源不属于会自动清理的 context/resource owner，且 `PG_CATCH` 或上层 cleanup 没有释放，即为泄露。

重点检查：
- `malloc/calloc/strdup/new` 后调用 openGauss 可能 ERROR 的函数。
- `AllocSetContextCreate` 后，在 `MemoryContextDelete` 前出现 ERROR。
- `MemoryContextSwitchTo(long-lived context)` 后分配临时对象，再在释放前 ERROR。

## L2 提前返回遗漏清理

函数存在多个 `return`、`break`、`continue`、`goto` 分支时，逐条路径核对已分配资源。只在尾部释放不够，任何早退路径都必须释放，或跳转到统一 cleanup 标签。

正确模式通常是：
- 分配后所有失败分支 `goto cleanup`。
- cleanup 中按所有权释放 `pfree/free/MemoryContextDelete`。
- 如果资源转移给调用者或长期结构，早退路径要明确不再负责释放。

## L3 临时 MemoryContext 未删除或未重置

`AllocSetContextCreate` 创建临时 context 后，必须检查所有路径是否执行 `MemoryContextDelete` 或 `MemoryContextReset`。临时 context 常用于批处理、函数调用、循环迭代和逻辑复制 launcher。

参考 openGauss 正确模式：
- `src/gausskernel/storage/replication/logical/launcher.cpp` 中创建 `subctx`，切换到该 context 构建订阅列表，随后切回旧 context 并 `MemoryContextDelete(subctx)`。
- `src/gausskernel/storage/replication/logical/parallel_decode.cpp` 的白名单过滤路径切回旧 context 并 `MemoryContextReset(ctx)`。

风险模式：
- 创建临时 context 后直接 `return`。
- `MemoryContextSwitchTo(subctx)` 后调用可能 ERROR 的函数，但没有 PG_CATCH 删除 `subctx`。
- 循环中创建 context，只在正常循环尾部删除。

## L4 MemoryContextSwitchTo 后未恢复旧 context

切换到短期或长期 context 后，必须追踪每个出口是否恢复旧 context。未恢复会导致后续无关分配落入错误 context：落入长期 context 会形成累积泄露，落入短期 context 会导致对象生命周期过短或被提前 reset。

审计步骤：
1. 找到 `old = MemoryContextSwitchTo(ctx)`。
2. 枚举后续所有 `return/goto/break/continue/ereport`。
3. 确认每条路径都有 `MemoryContextSwitchTo(old)`，或由封装函数负责恢复。

参考 openGauss 风险敏感模式：
- `src/gausskernel/storage/replication/logical/parallel_reorderbuffer.cpp` 使用 `oldCtxPtr` 在批发送前后反复保存和恢复 context。

## L5 长期 context 中的临时分配累积

分配到 `TopMemoryContext`、`t_thrd.top_mem_cxt`、`g_instance.instance_context`、dispatcher/worker 长生命周期 context、slot/global context 的对象，如果只是临时使用，必须显式 `pfree` 或在批次结束 `MemoryContextReset`。否则单次泄露不明显，但循环、后台线程、逻辑复制 worker 会长期累积。

重点检查：
- `MemoryContextSwitchTo(g_instance.../TopMemoryContext/t_thrd.top_mem_cxt)` 后的 `palloc/pstrdup/list_make/lappend`。
- 后台常驻循环、launcher、worker、dispatcher 中的 list、StringInfo、tuple、option 字符串。
- 白名单过滤、批发送、异常中断后是否 reset 对应 batch context。

## L6 所有权转移不完整或无人接管

资源写入结构体字段、链表、哈希表、队列、全局数组、resource owner 后，必须确认接收方有销毁函数，并在对象生命周期结束时调用。若只赋值但没有任何释放路径，即为泄露。

重点追踪：
- `struct->field = palloc...`
- `list = lappend(list, ptr)`
- `hash_search(..., HASH_ENTER, ...)` 后填入的指针字段
- dispatcher、worker、txn、snapshot、reorder buffer、logical log 等长期对象

判定要求：
- 找到 `FreeXXX/CleanupXXX/ResetXXX/DeleteXXX` 释放字段。
- 若字段分配在对象专属 context 中，确认该 context 随对象删除。
- 若接收方仅保存借用指针，原 owner 仍需释放。

## L7 repalloc 失败、覆盖旧指针或派生指针失效

`repalloc` 成功会返回新地址并释放旧块；失败通常 ERROR。审计时重点看旧指针是否被提前覆盖，以及派生指针是否在 repalloc 后继续使用。

风险模式：
- `ptr = repalloc(ptr, size)` 前后存在错误路径，且外层 cleanup 依赖旧指针但状态已不一致。
- `field = repalloc(field, size)` 后，其他结构仍保存旧地址或子指针。
- 在 `palloc_extended` 风格无 ERROR 分配中，返回 `NULL` 后覆盖了唯一可释放指针。

## L8 palloc_extended / noexcept 返回 NULL 后已分配资源未清理

普通 `palloc` 失败会 ERROR，但 `palloc_extended`、`palloc0_noexcept` 可能返回 `NULL`。如果函数在检测到 NULL 后直接返回，必须释放此前已分配的手工资源或删除临时 context。

审计步骤：
1. 找到 `palloc_extended`、`palloc0_noexcept`。
2. 检查 NULL 分支。
3. 回溯同函数此前的 `malloc/calloc/AllocSetContextCreate/MemoryContextSwitchTo(long-lived)` 分配。
4. 确认 NULL 分支释放或转交所有权。

## L9 malloc/free 与 palloc/pfree 家族混用

openGauss 代码中既有 MemoryContext 分配，也有底层 allocator 使用 `malloc/free`。必须按分配族配对释放：
- `malloc/calloc/strdup` -> `free`
- `new` -> `delete`
- `new[]` -> `delete[]`
- `palloc/palloc0/pstrdup/pnstrdup/MemoryContextAlloc` -> `pfree` 或所属 `MemoryContextDelete/Reset`

混用不仅可能崩溃，也会让审计误判所有权。底层内存上下文实现文件（如 `src/common/backend/utils/mmgr/aset.cpp`、`mcxt.cpp`）可以直接调用 malloc/free；业务代码优先检查是否错误地把 C 分配交给 `pfree`，或把 palloc 内存交给 `free`。

## L10 对等函数和生命周期闭环对比

找到同模块中功能相似的创建/释放函数、init/fini 函数、正常/异常路径、串行/并行实现，比较分配与释放闭环。数量或生命周期明显不对称时，优先作为疑似泄露。

对比维度：
- `CreateXXX/FreeXXX`、`AllocateXXX/FreeXXX`、`InitXXX/CleanupXXX` 是否一一覆盖字段。
- 串行逻辑解码与并行逻辑解码是否都 reset batch context。
- 正常路径释放的资源，PG_CATCH 是否也释放或交给上层 owner。
- openGauss 示例：`AllocateSnapshotBuilder` 创建专属 context 并把 `builder->context` 挂入对象，`FreeSnapshotBuilder` 删除整个 context；这是“局部未逐个 pfree，但生命周期闭环完整”的非泄露模式。

## 误报排除规则

- 对象分配在短生命周期 context，且该 context 在函数/事务/批次结束必定 `MemoryContextReset/Delete`，不是泄露。
- 返回给调用者的 palloc 对象不是本函数泄露；必须继续追踪调用者是否释放或接管。
- 存入全局 cache、session cache、resource owner、memory context tree 的对象不一定泄露；必须确认是否存在失效、回收、重置机制。
- `PG_CATCH` 只 `PG_RE_THROW()` 不一定泄露；若资源由当前事务/portal/resource owner 自动清理，可排除；若是 `malloc` 或长期 context 临时对象，不能排除。
- 注释中出现 “leak” 不能直接判定；需要用执行路径和生命周期闭环验证。

## 输出编号

内存泄露问题使用 `BUG-L{序号}` 编号，例如：

```
### BUG-L1 [高|中|低] {标题}
- **文件**: xxx.cpp:{行号}
- **描述**: {哪类资源在哪条路径失去释放/重置}
- **触发条件**: {错误路径、早退路径、循环累积、长期 worker 等}
- **影响**: {单次泄露、长期累积、OOM、后台线程内存膨胀等}
- **修复建议**: {统一 cleanup、PG_CATCH 清理、删除/重置 context、明确所有权转移等}
- **对比依据**: {同模块正确生命周期闭环，无则省略}
```
