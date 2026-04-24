# bug-detection-audit

C/C++ 代码深度 Bug 审计技能，用于 Claude Code / Codex 等 AI 编程助手。

## 功能

- **二次释放（Double-Free）检测**：系统性扫描 C/C++ 代码中的二次释放漏洞
- **业务领域扩展**：支持按业务领域加载专项审计规则，当前支持 `logical-decoding`（逻辑解码）

## 使用方式

将本目录放置到 AI 助手的 skills 目录下（如 `~/.claude/skills/bug-detection-audit/`），即可通过技能名称调用。

### 参数

```
bug-detection-audit <代码目录> [doublefree|all] [domain=logical-decoding]
```

- `代码目录路径`：要审计的代码目录
- `审计类型`（默认 `all`）：`doublefree` / `all`
- `domain=<业务名>`（可选）：当前支持 `logical-decoding`

## 目录结构

```
bug-detection-audit/
├── SKILL.md                          # 技能定义文件
├── references/
│   ├── general-reference.md          # 通用前置知识
│   ├── double-free-audit.md          # 二次释放审计清单 (D1-D8)
│   └── domain-logical-decoding.md    # 逻辑解码领域专项扩展
```

## 扩展方法

- **新增 Bug 类型**：添加 `references/{type}-audit.md`，在 SKILL.md 的 Step 2 表格注册一行
- **新增业务领域**：添加 `references/domain-{业务名}.md`，在 SKILL.md 的参数说明注册一行

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

## License

MIT
