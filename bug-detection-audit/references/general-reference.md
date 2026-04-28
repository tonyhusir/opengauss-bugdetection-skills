# 通用知识库 - Bug审计前置参考

所有类型Bug审计前必须先载入此文档。

---

## PG_TRY / PG_CATCH 执行机制（与C++ try/catch相同）

- PG_TRY块正常完成 → PG_CATCH块**不执行**
- PG_TRY块中途ERROR → 从ERROR点跳转到PG_CATCH，ERROR之后的代码不执行

**二次释放触发条件**：

```
PG_TRY块:
    释放资源A          ← 已执行
    [此处发生ERROR]     ← 跳转到PG_CATCH
    释放资源B          ← 不执行

PG_CATCH块:
    释放资源A          ← 二次释放！
    释放资源B          ← 正确释放
```

**关键**: PG_TRY中ERROR之前已执行的操作与PG_CATCH中的操作**不互斥**。

**防御**: 释放后立即置空指针，PG_CATCH中检查指针是否非NULL。

---

## ereport(ERROR) 行为

触发siglongjmp，当前函数后续代码不执行。

---