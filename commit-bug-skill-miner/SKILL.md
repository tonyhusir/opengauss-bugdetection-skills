---
name: commit-bug-skill-miner
description: 根据用户输入的 bug-fix commit id 和可选 bug 类型，基于具体代码修改分析该 commit 修复的一个或多个 bug 并归类，优先查找已有同类 BUG detector skill 并验证能否在父提交中挖出原始 bug；若无合适 skill 或验证失败，再按 bug 类型归档生成新的泛化 detector skill 并迭代验证，最多迭代优化 3 次。适用于用户给出 commit hash 并要求提炼 bug 模式、复用或生成检测 skill、验证检测能力的场景。
---

# Commit Bug Skill Miner

## 目标

以用户输入的 bug-fix commit id 作为 `COMMIT_ID`，可选 bug 类型作为 `BUG_TYPE`，完成：

1. 基于 diff、修复前代码、修复后代码和调用上下文识别该 commit 修复的一个或多个 bug；若 `BUG_TYPE` 非空，只处理语义匹配的 bug。
2. 为每个目标 bug 先查找已有同类 BUG detector skill，并在父提交上做干净验证。
3. 若没有合适已有 skill，或已有 skill 验证失败，再生成泛化 detector skill 并验证；禁止把原始答案直接写进 skill。

## 硬约束

- commit message 只能作为线索，不能作为结论来源；必须从代码路径证明不变量破坏。
- 先理解修复，再抽象规则；不要把补丁细节当作检测规则。
- 新生成 detector skill 不能包含原始 commit id、目标文件名、目标函数名、精确变量名、行号、commit message 原文或补丁字面量。
- 新 detector skill 必须按 bug 类型或不变量类型归档；类型相同的 detector skill 放在同一个类型目录下，`name`、目录名和文件内容都不能包含 commit id。
- 验证必须由上下文干净、prompt 干净的新 agent 完成；当前上下文只能准备 worktree、准备 detector skill、汇总验证结果和做语义映射判断，不能替代验证 agent 做最终命中判断。
- 如果运行环境无法启动新 agent，必须停止在验证前并报告“无法完成干净上下文验证”；不得用当前上下文验证替代，不得宣称已有或新生成 skill 验证成功。
- 不要在用户当前活动 worktree 执行破坏性 reset。验证前版本必须优先通过 `git worktree` 创建；若子 agent 有独立 forked workspace，只能在该 fork 中 reset。
- 本轮创建的 skill、验证 worktree、临时日志和中间文件必须位于当前 agent 工作路径下，并在最终答复前清理不需要保留的临时资源。

## 路径约定

读取输入后先绑定当前 agent 工作路径：

```bash
pwd
```

将输出绑定为 `AGENT_WORKDIR`。后续默认路径为：

- 生成 skill 目录：`${AGENT_WORKDIR}/.commit-bug-skill-miner/generated-skills/`
- 验证 worktree 目录：`${AGENT_WORKDIR}/.commit-bug-skill-miner/validation/<bug-type>-<short-hash>`
- 临时日志和中间文件目录：`${AGENT_WORKDIR}/.commit-bug-skill-miner/`

不得使用 `/tmp` 作为默认创建目录或验证目录。若当前路径不是期望的 agent 工作路径，先切换到当前 agent 的工作路径后再绑定。

## 固化提示词

本 skill 的固定提示词模板位于同目录 `prompts/` 下，执行到对应阶段时读取并渲染：

- `prompts/existing_skill_validation.md`：验证已有 detector skill 是否能在修复前代码中独立发现同类 bug。
- `prompts/skill_generation.md`：根据 bug dossier 生成泛化 detector skill。
- `prompts/generated_skill_review.md`：由专门检视 agent 审查新生成 skill 的可检测性和泛化性。
- `prompts/generated_skill_validation.md`：可选，用于将新生成 skill 的检测验证和泛化审查拆开执行。

验证类提示词只能注入 `WORKTREE_PATH`、`DETECTOR_SKILL_PATH` 和 `OUTPUT_SCHEMA_PATH` 等执行必要变量。不得注入 commit id、diff、bug dossier、目标文件、目标函数、目标行号、预期答案或修复形态。每轮实际发送给验证 agent 的渲染后 prompt 必须保存到 `${AGENT_WORKDIR}/.commit-bug-skill-miner/runs/<run-id>/prompts/` 以便审计。

## 工作流程

### 0. 读取输入

- 从用户请求中读取 bug-fix commit id，绑定为 `COMMIT_ID`。如果没有明确 commit id，先要求用户提供；不要自行猜测或从日志中挑选 commit。
- 从用户请求中读取可选 bug 类型，绑定为 `BUG_TYPE`。`BUG_TYPE` 可以是“memory-leak”“double-free”“死锁”“状态机错误”等自然语言或短横线命名类型；未提供则为空，表示分析全部可识别 bug。
- 后续命令中的 `${COMMIT_ID}` 都必须替换为本次输入的 commit id。

### 1. 确认 commit 和父提交

执行：

```bash
git rev-parse --verify "${COMMIT_ID}^{commit}"
git rev-parse "${COMMIT_ID}^"
git show --stat --find-renames "${COMMIT_ID}"
git show --name-status --find-renames --find-copies "${COMMIT_ID}"
git show --find-renames --find-copies --format=fuller "${COMMIT_ID}"
```

如果是 merge commit，先判断使用哪个父提交作为修复前版本；无法确定时询问用户。

### 2. 分析 bug dossier

必须阅读 diff 和修复前后代码。必要时用：

```bash
git show "${COMMIT_ID}^:<path>"
git show "${COMMIT_ID}:<path>"
git diff "${COMMIT_ID}^" "${COMMIT_ID}" -- <path>
rg -n "<changed-symbol>|<callee>|<state-field>|<resource-name>" <relevant-paths>
```

按独立 bug 或独立不变量分组。若一个 commit 同时修复多个根因、触发路径或修复机制不同的问题，必须拆成多个 bug dossier。

分析每个候选 bug 时回答：

- 修改新增、删除或移动了哪些资源释放、锁操作、状态转移、边界检查、错误处理或持久化步骤。
- 修复前代码在哪些路径没有执行这些动作，或执行顺序/条件为什么错误。
- 修复后代码通过什么控制流、数据流或生命周期变化恢复不变量。
- 该变化是 bug 修复、重构、防御性增强还是测试适配；只有能从代码路径证明存在不变量破坏的项才进入 bug dossier。

每个 bug dossier 必须包含：

- **Bug 内容**：一句话说明问题本质。
- **根因**：违反了什么资源生命周期、锁顺序、状态机、边界条件或数据一致性不变量。
- **触发路径**：输入、状态、分支、调用顺序或错误路径。
- **影响**：例如内存泄露、死锁、崩溃、数据损坏、错误结果、安全绕过。
- **修复机制**：commit 如何恢复不变量。
- **分类**：给出主分类和可选副分类，并说明理由。
- **泛化线索**：提炼可用于同类 bug 检测的抽象信号。

如果 `BUG_TYPE` 非空，先列出所有候选 bug 及初步分类，只保留主分类或副分类与 `BUG_TYPE` 语义匹配的 dossier；对未匹配项简要说明忽略原因。若没有任何候选 bug 匹配，停止并说明该 commit 的具体代码修改中未证明存在该类型 bug。

分类参考：资源生命周期、并发同步、控制流/状态机、数据正确性、边界与输入、权限与隔离。

### 3. 查找已有 detector skill

生成新 skill 前，必须先为每个保留的 bug dossier 分别查找已有可复用 detector skill；不要让一个 bug 的候选 skill、验证输入或结论污染另一个 bug。检索范围：

- 当前已安装 skills：`~/.codex/skills/`
- 本轮或历史生成的 skills：`${AGENT_WORKDIR}/.commit-bug-skill-miner/generated-skills/`
- 用户明确指定的其他 skill 目录。

检索应使用 bug 分类、影响和泛化线索，不能使用 commit id、目标文件名、目标函数名或补丁字面量。可用：

```bash
mkdir -p "${AGENT_WORKDIR}/.commit-bug-skill-miner/generated-skills"
find ~/.codex/skills "${AGENT_WORKDIR}/.commit-bug-skill-miner/generated-skills" -name SKILL.md -print 2>/dev/null
rg -n "<bug-type>|<invariant-keyword>|<impact-keyword>" ~/.codex/skills "${AGENT_WORKDIR}/.commit-bug-skill-miner/generated-skills"
```

如果找到候选 skill，阅读其 `name`、`description` 和核心检测规则。只要目标、适用信号、审计清单或确认标准与目标 bug 的不变量有语义交集，即使覆盖不够精确，也必须选择最接近的一个先进入干净验证；不得仅凭当前上下文判断跳过验证并直接生成新 skill。

如果有多个候选，优先验证最接近目标不变量、触发路径和资源生命周期的 skill；必要时可验证多个。如果没有任何可能覆盖的已有 skill，进入第 5 步生成新 skill。只有在没有任何语义可能覆盖的已有 detector，或最接近的已有 detector 已完成干净验证且不能语义映射回第 2 步 bug，才允许生成新 skill。

### 4. 验证 detector skill

无论验证已有 skill 还是新生成 skill，都使用本节协议。将要验证的 detector skill 路径绑定为 `DETECTOR_SKILL_PATH`，验证用代码必须是 `${COMMIT_ID}^`。

创建隔离 worktree，并记录到本轮临时资源列表：

```bash
mkdir -p "${AGENT_WORKDIR}/.commit-bug-skill-miner/validation"
git worktree add "${AGENT_WORKDIR}/.commit-bug-skill-miner/validation/<bug-type>-<short-hash>" "${COMMIT_ID}^"
```

验证 agent 必须同时满足：

- **上下文干净**：不能继承当前对话、commit 分析、diff 阅读记录或 bug dossier；如果使用 agent 工具，必须使用不 fork 当前上下文的新 agent。
- **prompt 干净**：首轮和后续提示词都不能包含目标答案线索。

只给验证 agent 以下信息：

- 修复前 worktree 路径。
- `DETECTOR_SKILL_PATH`。
- 通用任务：“使用这个 skill 审计该仓库是否存在此类 bug，输出具体文件/行号和推理依据。”

禁止给验证 agent：

- 原始 commit id。
- 修复 diff。
- 第 2 步 bug dossier。
- 预期文件、函数、行号。
- 目标触发路径、目标不变量破坏、补丁后的修复形态。
- “原始 bug 一定存在”的提示。
- 任何为了帮助命中而从当前上下文改写出来的提示、搜索关键词或候选范围。

如果验证过程中需要追问或补充信息，后续 prompt 只能补充通用执行约束、日志格式或工具使用要求，不能追加目标答案线索。

如果无法启动新 agent，停止流程并在最终答复中说明验证阻塞；不要继续生成或报告“验证通过”的结论。

### 5. 生成新的 detector skill

生成新 skill 前，必须确认第 3 步没有任何可能覆盖的已有 detector，或第 4 步已用最接近的已有 detector 做过干净验证且验证失败。若存在可能覆盖但尚未验证的已有 detector，必须回到第 4 步先验证。

生成路径：

```text
${AGENT_WORKDIR}/.commit-bug-skill-miner/generated-skills/<bug-class>/<detector-name>/SKILL.md
```

要求：

- `<bug-class>` 是稳定类型目录，例如 `memory-leak`、`double-free`、`deadlock`、`state-machine`、`data-corruption`、`boundary-input`。
- `<detector-name>` 来自不变量类型或通用触发模式，例如 `error-path-cleanup-detector`、`lock-order-inversion-detector`、`state-cleanup-symmetry-detector`。
- 创建前检查同类型目录下是否已有 detector；若已有 detector 可通过小幅泛化覆盖当前 bug，优先扩展该 detector；否则再创建新子目录。
- 生成内容使用中文，至少包含 frontmatter、目标、适用信号、预扫描、审计清单、确认标准、误报排除、输出格式。
- 该 skill 必须指导 agent 发现”同类问题”，而不是定位”这个 commit 曾修过的问题”；内容必须来自第 2 步的代码级不变量分析，而不是 commit message 摘要。
- **去具体化要求**：skill 中所有搜索命令、示例、适用信号必须使用通用模式（如 `->\w+\s*=` 而非 `->user_name=`）和语义角色描述（如”执行身份标识”而非 `user_name`），不能包含 bug dossier 中的精确标识符。
- **生成后自检**：skill 写完后，从 bug dossier 中提取所有具体标识符（变量名、结构体名、函数名），在 skill 文件中逐个搜索。命中任何一条必须改写为通用描述后才能进入验证。

新 skill 生成后，必须按第 4 步用干净上下文和干净 prompt 验证。

### 6. 判断成功与迭代

只有上下文干净、prompt 干净的新验证 agent 的独立发现能语义映射回第 2 步 bug，才算成功：

- 命中相同脆弱文件/函数，或等价调用路径。
- 命中相同不变量破坏。
- 命中相同触发条件或错误路径。
- 修复建议与原 commit 的修复形态语义一致。

泛泛风险、无关 bug、必须看补丁才知道含义的发现，都不能算成功。当前上下文可以做映射判断，但输入必须来自验证 agent 的独立发现；如果没有验证 agent 结果，或验证 agent 上下文/prompt 不干净，本轮验证状态只能是“未完成”或“无效”。

如果已有 skill 验证成功，复用该 skill 并结束当前 bug dossier 的处理，不再为该 bug 生成新 skill。如果已有 skill 验证失败，不要直接改写已有 skill；在完成最接近已有 detector 的干净验证后，再生成新的 detector skill 并验证。

如果新 skill 验证失败，最多优化并重试 3 轮。每轮优化只能补充泛化内容：更准确的语义线索、更好的搜索关键词、更完整的审计清单、更严格的确认标准或更有效的误报排除规则。不得加入原始答案信息；失败后的优化信息只能写进 detector skill 本身，不能通过验证 prompt 泄露目标答案。每轮验证都必须使用新的干净 agent/context 和从空白上下文构造的干净 prompt。

### 7. 清理临时资源

最终答复前清理本轮创建的临时资源：

1. 等待所有验证 agent 结束，避免删除仍在使用的 worktree。
2. 对本轮通过 `git worktree add` 创建的路径执行：

```bash
git worktree remove "${AGENT_WORKDIR}/.commit-bug-skill-miner/validation/<bug-type>-<short-hash>"
git worktree prune
```

3. 如果 worktree 内存在验证 agent 生成的未跟踪文件导致无法删除，先判断这些文件是否确为本轮临时产物；若是，优先使用 `git worktree remove --force <path>` 清理。不要删除用户已有目录或非本轮创建的路径。
4. 删除本轮创建且不需要保留的临时日志、中间输出和草稿文件。
5. `${AGENT_WORKDIR}/.commit-bug-skill-miner/generated-skills/` 下的新 detector skill 默认作为本轮产物保留，便于后续复用；只有在验证失败且确认无复用价值时才删除。无论保留还是删除，都要在最终答复中说明。
6. 如果由于权限、仍有进程占用或用户要求保留而无法清理，最终答复必须列出未清理路径和原因。

## 最终输出

最终答复包含：

- 分析的 commit 和使用的父提交。
- 本轮绑定的 `AGENT_WORKDIR`，以及实际使用的 skill 创建目录和验证目录。
- 本轮 `BUG_TYPE` 输入值；若为空，说明已分析全部可识别 bug；若非空，说明保留和忽略了哪些候选 bug。
- 每个目标 bug dossier 的 bug 内容、根因、触发路径、影响、修复机制、分类。
- 是否找到并尝试已有 detector skill；列出路径和匹配理由。
- 若生成了新 detector skill，列出按类型归档后的路径。
- 每轮干净上下文验证 agent 的结果和优化说明；若未能启动验证 agent，明确标记为验证未完成/阻塞。
- 每轮验证是否满足上下文干净和 prompt 干净要求；若任一要求未满足，必须标记该轮验证无效。
- 是否成功检测出目标 bug dossier 对应的问题，以及成功使用的是已有 skill 还是新 skill。
- 临时资源清理结果：已删除的 worktree/临时文件、保留的新 detector skill，以及未能清理的路径和原因。
- 剩余风险或限制。
