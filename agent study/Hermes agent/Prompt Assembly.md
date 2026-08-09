# Prompt Assembly（Prompt 组装）

> Hermes 刻意分离了**缓存的 system prompt 状态**和**API 调用时的临时附加内容**。这是整个项目最重要的设计决策之一，因为它直接影响 token 用量、prompt 缓存效率、session 连续性和记忆正确性。

**核心文件**：`run_agent.py`、`agent/prompt_builder.py`、`tools/memory_tool.py`

---

## 核心设计：缓存 vs 临时

```
┌─────────────────────────────────────────────────────┐
│               Cached System Prompt                    │
│  (session 内不变，可被 Anthropic prefix cache 命中)    │
│                                                       │
│  stable  →  context  →  volatile                      │
│  (身份)    (项目规则)   (记忆/画像/时间戳)               │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│          API-Call-Time Ephemeral Layers               │
│  (每次 API 调用时注入，不破坏缓存)                      │
│                                                       │
│  ephemeral_system_prompt / prefill messages            │
│  gateway session overlays / plugin pre_llm_call       │
└─────────────────────────────────────────────────────┘
```

### 设计原因

如果将临时内容（预算警告、上下文压力提示、Gateway 会话上下文）写入缓存的 system prompt，会导致：

1. **Prompt cache 频繁失效**：Anthropic 的 prefix cache 要求前缀完全匹配。任何 system prompt 的变化都会导致整个缓存失效，重新计费。
2. **Session 连续性受损**：恢复 session 时会加载缓存过的 system prompt，但临时内容（如"你还有 5 轮迭代"）在恢复时已无意义。
3. **记忆语义混乱**：持久化记忆和临时提示混在一起，难以确定什么该保存、什么该丢弃。

**分离策略**：缓存 stable 部分（prefix cache 收益），临时内容在 API-call-time 额外注入（不影响缓存）。

---

## 缓存 System Prompt 的三层结构

缓存的 system prompt 按三个有序层级组装（`agent/system_prompt.py`）：

```
┌──────────────────────────────────────────┐
│  stable (稳定层)                          │
│  ├── SOUL.md 身份定义（或默认回退）        │
│  ├── 工具/模型行为指导                     │
│  ├── Skills 提示词                        │
│  ├── 环境提示（platform hints）            │
│  └── 平台提示（CLI/Gateway/ACP）          │
├──────────────────────────────────────────┤
│  context (上下文层)                       │
│  ├── 调用方传入的 system_message          │
│  └── 项目上下文文件（.hermes.md 等）       │
├──────────────────────────────────────────┤
│  volatile (易变层)                        │
│  ├── MEMORY.md 快照                       │
│  ├── USER.md 快照                         │
│  ├── 外部 Memory Provider 数据            │
│  └── 时间戳 / session / model / provider  │
└──────────────────────────────────────────┘
```

最终拼接顺序：**stable → context → volatile**

### 为什么顺序重要

- **stable 在最前**：身份和行为指导首先被模型读取，影响后续所有内容的解释方式
- **context 在中间**：项目规则在身份之后、记忆之前，让项目规则可以覆盖通用行为但不覆盖身份
- **volatile 在最后**：记忆和时间戳是最常变化的内容，放在末尾确保它们被模型最新读取（recency bias），同时不影响前面的缓存结构

### 层级归属

- Skills 属于 **stable** 层 — skill 指令是稳定的操作流程
- 记忆/画像快照属于 **volatile** 层 — 虽然也在缓存的 system prompt 中，但概念上是"可变的快照"
- 两者都**在**缓存的 system prompt 中（不是中途注入的临时层）

### skip_context_files 的特殊情况

当 `skip_context_files=True`（如子 agent 委托场景），`SOUL.md` 不加载，改用硬编码的 `DEFAULT_AGENT_IDENTITY`：

```text
You are Hermes Agent, an intelligent AI assistant created by Nous Research.
You are helpful, knowledgeable, and direct. You assist users with a wide
range of tasks including answering questions, writing and editing code,
analyzing information, creative work, and executing actions via your tools.
You communicate clearly, admit uncertainty when appropriate, and prioritize
being genuinely useful over being verbose unless otherwise directed below.
Be targeted and efficient in your exploration and investigations.
```

### 设计原因：为什么子 agent 用默认身份

子 agent 是执行具体任务的工人角色，不需要父 agent 的完整人格。跳过 `SOUL.md` 和上下文文件可以：
- 减少子 agent 的 system prompt 大小（每次委托都节省 token）
- 避免项目规则在子 agent 中被重复解释
- 让子 agent 更专注于单一任务

---

## 组装后的 System Prompt 实例

以下是所有层都加载后，最终 system prompt 的简化视图：

```text
# Layer 1: Agent 身份（来自 ~/.hermes/SOUL.md）
You are Hermes, an AI assistant created by Nous Research.
You are an expert software engineer and researcher.
You value correctness, clarity, and efficiency.
...

# Layer 2: 工具感知行为指导
You have persistent memory across sessions. Save durable facts using
the memory tool: user preferences, environment details, tool quirks,
and stable conventions. Memory is injected into every turn, so keep
it compact and focused on facts that will still matter later.
...
When the user references something from a past conversation or you
suspect relevant cross-session context exists, use session_search
to recall it before asking them to repeat themselves.

# Layer 3: Honcho 静态块（当激活时）
[Honcho personality/context data]

# Layer 4: 可选系统消息（来自配置或 API）
[User-configured system message override]

# Layer 5: 冻结的 MEMORY 快照
## Persistent Memory
- User prefers Python 3.12, uses pyproject.toml
- Default editor is nvim
- Working on project "atlas" in ~/code/atlas
- Timezone: US/Pacific

# Layer 6: 冻结的 USER 画像快照
## User Profile
- Name: Alice
- GitHub: alice-dev

# Layer 7: Skills 索引
## Skills (mandatory)
Before replying, scan the skills below. If one clearly matches
your task, load it with skill_view(name) and follow its instructions.
...
<available_skills>
  software-development:
    - code-review: Structured code review workflow
    - test-driven-development: TDD methodology
  research:
    - arxiv: Search and summarize arXiv papers
</available_skills>

# Layer 8: 上下文文件（来自项目目录）
# Project Context
The following project context files have been loaded
and should be followed:

## AGENTS.md
This is the atlas project. Use pytest for testing. The main
entry point is src/atlas/main.py. Always run `make lint` before
committing.

# Layer 9: 时间戳 + Session
Current time: 2026-03-30T14:30:00-07:00
Session: abc123

# Layer 10: 平台提示
You are a CLI AI Agent. Try not to use markdown but simple text
renderable inside a terminal.
```

---

## SOUL.md 机制

`SOUL.md` 位于 `~/.hermes/SOUL.md`，是 agent 的身份定义 — system prompt 的**第一个部分**。

### 加载逻辑

```python
# agent/prompt_builder.py (简化)
def load_soul_md() -> Optional[str]:
    soul_path = get_hermes_home() / "SOUL.md"
    if not soul_path.exists():
        return None
    content = soul_path.read_text(encoding="utf-8").strip()
    content = _scan_context_content(content, "SOUL.md")  # 安全扫描
    content = _truncate_content(content, "SOUL.md")       # 上限 20k 字符
    return content
```

### 处理流程

| 步骤 | 操作 | 原因 |
|------|------|------|
| 1. 安全扫描 | 检测 prompt injection 模式（不可见 Unicode、"ignore previous instructions"、凭证泄露等） | 防止用户文件中的恶意内容注入 agent |
| 2. 截断 | 上限 20,000 字符，使用 70/20 头尾比例 | 防止过大的 SOUL.md 挤占上下文窗口 |
| 3. 替换身份 | 如果 SOUL.md 存在，替代硬编码的 `DEFAULT_AGENT_IDENTITY` | 用户自定义优先于默认 |
| 4. 去重保护 | `build_context_files_prompt(skip_soul=True)` 防止 SOUL.md 出现两次 | 一次作为身份，一次作为上下文文件会导致重复 |

### 设计原因：为什么 SOUL.md 是文件而不是配置项

将身份定义为文件（而非 YAML/JSON 配置字段）有几个好处：
- **自由格式**：用户可以用任意自然语言描述 agent 人格，不受配置 schema 限制
- **可版本控制**：SOUL.md 可以放入 dotfiles 仓库，跨机器同步
- **熟悉的编辑体验**：用任何文本编辑器修改变得自然，不需要学习配置语法
- **与其他 context 文件一致**：`.hermes.md`、`AGENTS.md`、`CLAUDE.md` 都是 Markdown 文件，用户已有心智模型

---

## 上下文文件的注入

`build_context_files_prompt()` 使用优先级系统 — **只加载一种**项目上下文类型（先匹配者胜）：

```python
# agent/prompt_builder.py (简化)
def build_context_files_prompt(cwd=None, skip_soul=False):
    cwd_path = Path(cwd).resolve()
    # 优先级：先匹配者胜 — 只加载一种项目上下文
    project_context = (
        _load_hermes_md(cwd_path)       # 1. .hermes.md / HERMES.md
        or _load_agents_md(cwd_path)    # 2. AGENTS.md
        or _load_claude_md(cwd_path)    # 3. CLAUDE.md
        or _load_cursorrules(cwd_path)  # 4. .cursorrules / .cursor/rules/*.mdc
    )
    sections = []
    if project_context:
        sections.append(project_context)
    # SOUL.md 从 HERMES_HOME 独立加载
    if not skip_soul:
        soul_content = load_soul_md()
        if soul_content:
            sections.append(soul_content)
    if not sections:
        return ""
    return (
        "# Project Context\n\n"
        "The following project context files have been loaded "
        "and should be followed:\n\n"
        + "\n".join(sections)
    )
```

### 上下文文件发现细节

| 优先级 | 文件 | 搜索范围 | 说明 |
|--------|------|---------|------|
| 1 | `.hermes.md`、`HERMES.md` | CWD 向上到 git root | Hermes 原生项目配置 |
| 2 | `AGENTS.md` | CWD 仅当前目录 | 通用 agent 指令文件 |
| 3 | `CLAUDE.md` | CWD 仅当前目录 | Claude Code 兼容 |
| 4 | `.cursorrules`、`.cursor/rules/*.mdc` | CWD 仅当前目录 | Cursor 兼容 |

### 设计原因

**只加载一种而非全部**：如果同时加载了 `.hermes.md`、`AGENTS.md` 和 `CLAUDE.md`，三者可能包含重复或冲突的指令。优先级系统避免了信息冗余和指令冲突。如果需要多个上下文来源，用户可以通过 `SOUL.md` 的 identity 块统一管理。

**`.hermes.md` 向上搜索到 git root**：这是唯一向上搜索的上下文文件。原因是在 monorepo 场景下，用户可能在子目录中运行 Hermes，但项目规则应该来自仓库根目录。git root 是自然的项目边界。

**其他文件仅搜索 CWD**：`AGENTS.md`、`CLAUDE.md` 通常放在项目根目录。如果也向上搜索，可能会意外加载不相关的上下文文件。

### 所有上下文文件的安全处理

| 处理 | 操作 | 原因 |
|------|------|------|
| 安全扫描 | 检测 prompt injection 模式 | 不可见 Unicode 字符、"ignore previous instructions"、凭证泄露 |
| 截断 | 上限 20,000 字符，70/20 头尾比 | 防止超大文件挤占上下文 |
| YAML frontmatter 剥离 | `.hermes.md` 的 frontmatter 被移除 | 保留给未来的配置覆盖功能 |

**70/20 头尾比截断**：保留前 70% 和后 20% 的内容（中间 10% 被截断标记替代）。原因是文件的开头通常包含最重要的规则和约定，结尾可能包含最近的更新。中间部分往往是详细的示例和说明，丢失后影响最小。

---

## API-Call-Time 临时层

以下内容**故意不持久化**到缓存的 system prompt 中：

| 临时层 | 注入时机 | 内容 |
|--------|---------|------|
| `ephemeral_system_prompt` | API 调用前 | 预算警告、上下文压力提示 |
| prefill messages | API 调用前 | 预填充的 assistant 消息 |
| Gateway session overlays | API 调用前 | 会话上下文（用户信息、频道规则） |
| Honcho 外部召回 | 当前 turn user message | 跨 session 记忆召回 |
| `pre_llm_call` plugin context | 当前 turn user message | 插件提供的临时上下文 |

当多个插件返回 context 时，Hermes **拼接**这些上下文块（参见 Hooks → pre_llm_call）。

### 设计原因

**不在缓存中的内容这些内容的共同特征**：每个 turn 都可能不同。

- 预算警告的数字在变化（"剩余 27 轮" → "剩余 26 轮"）
- Gateway session overlay 依赖当前会话的元数据
- Plugin context 可能包含实时数据

如果这些内容写入缓存，每次变化都会导致 prompt cache 失效。将它们作为 API-call-time 额外层注入，让稳定的核心 prompt 保持可缓存。

**Plugin context 追加到 user message 而非 system prompt**：`pre_llm_call` 的上下文追加到当前 turn 的 user message 中，而非 system prompt。这样做的原因是：
- user message 本身就是不缓存的（每次不同），加上 plugin context 不增加新的缓存破坏
- plugin context 在语义上是对"用户意图"的补充，而非对"agent 身份"的修改

---

## 记忆快照

本地记忆（MEMORY.md）和用户画像（USER.md）数据捕获在 system prompt 的 **volatile** 层。

### 关键行为

- Session 开始（或 prompt 重建时）：读取磁盘上的 `MEMORY.md` / `USER.md`，生成快照注入 system prompt
- 会话中写入记忆：更新磁盘文件，但**不更新**已构建的缓存 system prompt
- 直到下一个重建路径（新 session、压缩触发重建），新记忆才会出现在 system prompt 中

### 设计原因：为什么不在写入后立即更新 prompt

如果每次 `memory` 工具写入后立刻更新 system prompt：
1. Prompt cache 会立即失效（system prompt 变了）
2. 后续 API 调用需要全价 token，失去缓存收益
3. 在频繁写入记忆的 session 中，cache 命中率为 0

**延迟更新**的策略意味着：当前 session 中写入的记忆在**下一个** session 才可见。这是一个有意的权衡 — 用当前 session 内记忆不可见的代价换取稳定的 prompt cache。对于大多数用例来说，当前 session 中的信息已经在对话历史里，不需要记忆来重复。

---

## Skills 索引

Skills 系统在有 skills 工具可用时，向 prompt 注入一个紧凑的 skills 索引：

```text
## Skills (mandatory)
Before replying, scan the skills below. If one clearly matches
your task, load it with skill_view(name) and follow its instructions.
...
<available_skills>
  software-development:
    - code-review: Structured code review workflow
    - test-driven-development: TDD methodology
  research:
    - arxiv: Search and summarize arXiv papers
</available_skills>
```

Skills 本身是**懒加载**的 — prompt 只包含索引（名称 + 简短描述），完整的 skill 指令通过 `skill_view` 工具在需要时加载。这样保持了 system prompt 的精简。

---

## 支持的 Prompt 定制面

> 大多数用户应将 `agent/prompt_builder.py` 视为实现代码，而非配置面。支持的定制路径是**修改 Hermes 已经加载的 prompt 输入**，而非直接编辑 Python 模板。

### 首选定制面

| 定制面 | 用途 | 路径 |
|--------|------|------|
| `SOUL.md` | 替换默认 agent 身份和持续行为 | `~/.hermes/SOUL.md` |
| `MEMORY.md` / `USER.md` | 提供跨 session 的持久事实和用户画像 | `~/.hermes/MEMORY.md`、`~/.hermes/USER.md` |
| 项目上下文文件 | 注入仓库特定的工作规则 | `.hermes.md`、`AGENTS.md`、`CLAUDE.md`、`.cursorrules` |
| Skills | 打包可重用的工作流程 | `skills/` 目录 |
| 可选 system prompt 配置/API 覆盖 | 添加部署特定的指令文本 | 配置或 API 参数 |
| 临时覆盖 | 添加 turn 范围指导 | `HERMES_EPHEMERAL_SYSTEM_PROMPT` 或 prefill messages |

### 何时编辑代码

只在以下情况编辑 `agent/prompt_builder.py`：
- 你在维护一个 fork
- 你在贡献上游的行为变更

该文件组装了所有 session 的 prompt 管道、缓存边界和注入顺序。直接编辑是**全局产品变更**，而非按用户定制。

| 目标 | 正确做法 |
|------|---------|
| 不同的 assistant 身份 | 编辑 `SOUL.md` |
| 不同的仓库规则 | 编辑项目上下文文件 |
| 可重用的操作流程 | 添加或修改 Skills |
| 改变 Hermes 为所有人组装 prompt 的方式 | 修改 Python 代码（视为代码贡献） |

---

## Prompt 组装架构总结

```
                         ┌─────────────────────┐
                         │   prompt_builder.py  │
                         │   组装引擎            │
                         └──────────┬──────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────────┐
│  stable 层     │         │  context 层    │         │  volatile 层      │
│               │         │               │         │                   │
│ SOUL.md       │         │ .hermes.md    │         │ MEMORY.md 快照    │
│ 工具行为指导   │    +    │ AGENTS.md     │    +    │ USER.md 快照      │
│ Skills 索引   │         │ CLAUDE.md     │         │ Memory Provider   │
│ 平台提示      │         │ .cursorrules  │         │ 时间戳/Session    │
└───────┬───────┘         └───────┬───────┘         └────────┬──────────┘
        │                         │                          │
        └─────────────────────────┼──────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │  Cached System Prompt    │
                    │  (session 内不变)         │
                    │  → 可被 prefix cache 命中 │
                    └────────────┬────────────┘
                                 │
                                 │  API-call-time
                                 ▼
                    ┌─────────────────────────┐
                    │  Ephemeral Layers        │
                    │  ├── 预算/压力警告        │
                    │  ├── prefill messages     │
                    │  ├── Gateway overlays     │
                    │  └── Plugin context       │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │  最终 API 请求            │
                    │  messages = [system]     │
                    │           + [user_msg]   │
                    │           + [history]    │
                    └─────────────────────────┘
```

---

## 关键源文件

| 文件 | 用途 |
|------|------|
| `agent/prompt_builder.py` | System prompt 组装引擎 — 三层拼接、安全检查、截断 |
| `agent/system_prompt.py` | System prompt 模板和默认内容 |
| `run_agent.py` | `_build_system_prompt()` — prompt 缓存与重建触发 |
| `tools/memory_tool.py` | Memory 工具 — 写入触发磁盘更新但不刷新 prompt |
| `agent/context_compressor.py` | 压缩触发 prompt 重建的路径之一 |
