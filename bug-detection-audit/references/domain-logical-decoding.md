# 业务领域扩展：逻辑解码（Logical Decoding）

适用：OpenGauss / PostgreSQL 逻辑解码与逻辑复制模块。

## D5 扩展：所有权转移函数

调用后对象归 ReorderBuffer 管理，原调用者不得再释放：

| 转移函数 | 错误模式 |
|---------|---------|
| `ReorderBufferQueueChange(rb, txn, change)` | QueueChange 后又 ReturnChange |
| 赋值给 `change->data.tp.newtuple` 等字段 | tuple 既被 change 持有，又被局部变量释放 |

## D8 扩展：对等文件对

| 文件A | 文件B |
|-------|-------|
| `reorderbuffer.cpp` | `parallel_reorderbuffer.cpp` |
| `decode.cpp` | `logical_parse.cpp` |
| `logical.cpp` | `parallel_decode.cpp` |

## 配对函数（D1-D4 参考）

以下每个获取必须在所有路径上有对应释放：

| 获取 | 释放 |
|-----|-----|
| `palloc`/`palloc0` | `pfree` |
| `OpenTransientFile` | `CloseTransientFile` |
| `logicalrep_rel_open` | `logicalrep_rel_close` |
| `heap_open` | `heap_close` |
| `RelationIdGetRelation` | `RelationClose` |
| `ReorderBufferGetChange` | `ReorderBufferReturnChange` 或 `ReorderBufferQueueChange` |
| `MakeSingleTupleTableSlot` | `ExecDropSingleTupleTableSlot` |
| `ResourceOwnerCreate` | `ResourceOwnerRelease` |
| `walrcv_exec` | `walrcv_clear_result` |

以下对象不得直接 `pfree()`，必须用专用函数释放：

| 对象 | 正确释放函数 |
|------|------------|
| `ReorderBufferChange` | `ReorderBufferReturnChange(rb, change)` |
| `ReorderBufferTupleBuf` | `ReorderBufferReturnTupleBuf(rb, tuple)` |

## 模块文件速查

| 文件 | 职责 |
|------|------|
| `decode.cpp` | WAL 记录解析入口 |
| `reorderbuffer.cpp` | 事务重排序、大事务溢出磁盘 |
| `snapbuild.cpp` | 从 WAL 重建 MVCC 快照 |
| `logical.cpp` | LogicalDecodingContext 生命周期 |
| `parallel_decode.cpp` | 并行解码主控线程 |
| `parallel_decode_worker.cpp` | 并行解码工作线程 |
| `logical_queue.cpp` | 主控与 worker 间无锁队列 |
| `origin.cpp` | 复制源管理 |
| `tablesync.cpp` | 初始表同步 |
| `launcher.cpp` | 逻辑复制 worker 生命周期 |
| `worker.cpp` | apply worker 主逻辑 |
