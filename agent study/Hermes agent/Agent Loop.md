# Agent Loop 内部机制

> 核心编排引擎是 `run_agent.py` 中的 `AIAgent` 类 — 一个处理从 prompt 组装到工具分发再到 provider 故障转移所有逻辑的大型文件。

---

## 核心职责

`AIAgent` 负责以下全部工作：

| 职责 | 说明 |
|------|------|
| 系统 prompt 组装 | 通过 `prompt_builder.py` 组装有效的系统 prompt 和工具 schema |
| Provider/API 模式选择 | 选择正确的 API 模式（chat_completions / codex_responses / anthropic_messages） |
| 可中断模型调用 | 发起支持取消操作的模型调用 |
| 工具执行 | 顺序或通过线程池并发执行工具调用 |
| 对话历史维护 | 以 OpenAI 消息格式维护对话历史 |
| 压缩/重试/回退 | 处理上下文压缩、API 重试和 fallback 模型切换 |
| 迭代预算追踪 | 跨父 agent 和子 agent 追踪迭代预算 |
| 内存刷写 | 在上下文丢失前将持久化内存刷写到磁盘 |

### 设计原因

将所有这些职责集中在一个类中，而非分散到多个模块，是因为这些职责之间存在**深度耦合**：
- 压缩决定会影响 prompt 缓存策略
- 工具执行结果的大小会影响是否需要触发压缩
- Fallback 切换会影响后续迭代的 API 模式
- 子 agent 的预算消耗会影响父 agent 的压力预警

如果将这些职责拆分到独立模块，需要大量的跨模块协调和状态共享，带来的间接性成本超过模块化的收益。

---

## 两个入口

```python
# 简单接口 — 返回最终响应字符串
response = agent.chat("Fix the bug in main.py")

# 完整接口 — 返回 dict，包含 messages、metadata、usage stats
result = agent.run_conversation(
    user_message="Fix the bug in main.py",
    system_message=None,           # 省略则自动构建
    conversation_history=None,      # 省略则从 session 自动加载
    task_id="task_abc123"
)
```

| 方法 | 返回值 | 适用场景 |
|------|--------|---------|
| `chat()` | `str` | 简单调用、batch 处理、脚本集成 |
| `run_conversation()` | `dict`（含 final_response + messages + usage） | 需要完整元数据、Gateway、ACP、多轮会话 |

`chat()` 是 `run_conversation()` 的**薄封装**，仅从结果 dict 中提取 `final_response`。

### 设计原因

**两个入口而非一个**：`chat()` 的存在是为了降低简单场景的心智负担。调用方不需要知道返回 dict 的结构，不需要从中提取字段 — 如果只需要一段文本回复，`chat()` 就是正确的抽象层级。`run_conversation()` 暴露完整信息给需要追踪 token 用量、消息历史、session 状态的入口（Gateway、ACP）。

---

## API 模式

Hermes 支持三种 API 执行模式：

| API 模式 | 用途 | 客户端类型 | 内部消息格式 |
|----------|------|-----------|-------------|
| `chat_completions` | OpenAI 兼容端点（OpenRouter、自定义、大多数 provider） | `openai.OpenAI` | OpenAI 格式原生 |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI` + Responses 格式 | 转换为 OpenAI 格式 |
| `anthropic_messages` | 原生 Anthropic Messages API | `anthropic.Anthropic` 适配器 | 通过 adapter 转换为 OpenAI 格式 |

### 模式解析优先级

```
1. 显式 api_mode 构造函数参数       ← 最高优先级
2. Provider 特定检测                 例: "anthropic" provider → anthropic_messages
3. Base URL 启发式                   例: "api.anthropic.com" → anthropic_messages
4. 默认: chat_completions            ← 最低优先级
```

### 设计原因：为什么内部收敛到 OpenAI 格式

这是一个关键的架构决策。三种 API 的原生消息格式差异很大：

- **OpenAI**: `role`/`content`/`tool_calls` dict
- **Anthropic**: `role`/`content` 块数组，tool_use 是独立 content block
- **Codex Responses**: 完全不同的输入/输出结构

如果在整个系统中维护三种格式，每个操作（消息清洗、历史追加、上下文压缩、session 持久化）都需要三套实现。

**收敛策略**：
- API 调用前：Anthropic/Codex 格式通过 adapter 转换为内部 OpenAI 格式
- API 调用后：内部 OpenAI 格式通过 adapter 转换为各 API 的原生格式
- 中间所有逻辑只操作一种格式

代价是多了两次转换，收益是压缩、持久化、消息清洗等所有子系统只需一套实现。

---

## 单轮迭代生命周期

Agent loop 的每次迭代（一个 turn）按以下顺序执行：

```text
run_conversation()
  1. 若未提供则生成 task_id
  2. 将用户消息追加到对话历史
  3. 构建或复用已缓存的系统 prompt（prompt_builder.py）
  4. 检查是否需要预检压缩（上下文超过 50%）
  5. 从对话历史构建 API 消息
     - chat_completions: OpenAI 格式原样
     - codex_responses: 转换为 Responses API 输入项
     - anthropic_messages: 通过 anthropic_adapter.py 转换
  6. 注入临时 prompt 层（预算警告、上下文压力提示）
  7. 若使用 Anthropic，应用 prompt 缓存标记
  8. 发起可中断的 API 调用（_interruptible_api_call）
  9. 解析响应：
     - 若有 tool_calls → 执行工具，追加结果，回到步骤 5
     - 若为文本响应 → 持久化 session，按需刷写内存，返回
```

### 各步骤设计原因

**步骤 3 — 系统 prompt 缓存**：
System prompt 在一个 session 内缓存，只有压缩等重大事件后才重建。原因是 system prompt 通常有几千 token，每次重建不仅浪费 CPU，更重要的是会**破坏 Anthropic prompt cache 的 prefix 命中**。缓存保证了连续多个 turn 只需要为新消息付费。

**步骤 4 — 预检压缩先于 API 调用**：
压缩在 API 调用之前执行（而非之后），是为了**防止上下文溢出导致的 API 错误**。如果等 API 返回 "context too long" 再压缩，已经浪费了一次 API 调用的时间和费用。

**步骤 6 — 临时 prompt 注入不在缓存中**：
预算警告和上下文压力提示作为 `ephemeral_system_prompt` 注入，不进入 `_build_system_prompt()` 的缓存输出。原因是这些内容每个 turn 都不同（预算数字在变化），如果放在缓存中会破坏 cache 命中。将它们作为 API-call-time 的额外层注入，让稳定的核心 prompt 保持可缓存。

**步骤 9 — 工具循环回到步骤 5 而非步骤 3**：
工具执行后回到"构建 API 消息"步骤，跳过 system prompt 重建和压缩检查。原因是工具的几轮调用通常在几秒内完成，上下文不会快速增长到需要压缩的程度，system prompt 也不会变化。跳过这些步骤减少了延迟。

---

## 消息格式与交替规则

### 内部消息格式

所有消息使用 OpenAI 兼容格式：

```python
{"role": "system", "content": "..."}
{"role": "user", "content": "..."}
{"role": "assistant", "content": "...", "tool_calls": [...]}
{"role": "tool", "tool_call_id": "...", "content": "..."}
```

推理内容（支持 extended thinking 的模型）存储在 `assistant_msg["reasoning"]` 中，可选通过 `reasoning_callback` 展示。

### 消息角色交替规则

Agent loop 强制执行严格的消息角色交替：

- 系统消息之后：`User → Assistant → User → Assistant → ...`
- 工具调用期间：`Assistant (含 tool_calls) → Tool → Tool → ... → Assistant`
- ❌ 不允许连续两条 assistant 消息
- ❌ 不允许连续两条 user 消息
- ✅ 只有 `tool` 角色可以连续出现（并行工具结果）

### 设计原因

这些规则不是 Hermes 发明的 — 是 **LLM provider API 的强制要求**。OpenAI 和 Anthropic 的 API 都会拒绝格式错误的消息序列。Hermes 在 `_sanitize_api_messages()` 中主动修复常见格式问题（孤立的 tool 结果、重复的 assistant 消息等），但严格的交替规则是最基本的正确性保证。

**为什么 tool 角色可以连续出现**：这是并行工具执行的必然结果。一次 assistant 消息可以包含多个 `tool_calls`，每个调用产生一个 tool 结果消息。这些结果在逻辑上是"平行的" — 它们属于同一个 assistant 决策，需要连续排列才能被模型正确解析。

---

## 可中断的 API 调用

API 请求封装在 `_interruptible_api_call()` 中：

```
┌────────────────────────────────────────────────────┐
│  Main thread                  API thread           │
│                                                    │
│   wait on:                     HTTP POST           │
│    - response ready     ───▶   to provider         │
│    - interrupt event                               │
│    - timeout                                       │
└────────────────────────────────────────────────────┘
```

### 中断触发

- 用户发送新消息（在上一轮仍在执行时）
- `/stop` 命令
- 操作系统信号（SIGINT）

### 中断行为

- API 线程被**放弃**（响应丢弃，不等待完成）
- Agent 可以处理新输入或干净关闭
- **不会将部分流式响应注入对话历史**

### 设计原因

**不保存部分响应**是关键的完整性保证。如果流式响应被中断时已经收到了部分 token，将这些不完整的文本注入对话历史会导致模型在下一轮看到一段被截断的"自己说的话"，产生难以调试的行为异常。丢弃整个响应保持了对话历史的原子性 — 一个 turn 要么完整，要么不存在。

**为什么用线程而不是协程**：AIAgent 整体是同步的（见下文），使用 `threading.Event` + 后台线程比引入 asyncio 更简单。后台线程执行阻塞的 HTTP 调用，主线程通过 event 通信。这个模式让中断逻辑与 API 调用逻辑完全分离。

---

## 工具执行

### 顺序 vs 并发

模型返回多个 tool_calls 时的执行策略：

| 场景 | 策略 |
|------|------|
| 多个独立工具调用 | `ThreadPoolExecutor` **并发执行** |
| 标记为交互式的工具（如 `clarify`） | 强制**顺序执行** |
| 结果插入顺序 | 按**原始工具调用顺序**，而非完成顺序 |

### 执行流程

```text
for each tool_call in response.tool_calls:
    1. 从 tools/registry.py 解析处理器
    2. 触发 pre_tool_call 插件 hook
    3. 检查是否为危险命令（tools/approval.py）
       - 若危险：调用 approval_callback，等待用户确认
    4. 使用参数 + task_id 执行处理器
    5. 触发 post_tool_call 插件 hook
    6. 将 {"role": "tool", "content": result} 追加到历史
```

### Agent 级工具（被拦截，不经过 registry）

| 工具 | 拦截原因 | 拦截后的行为 |
|------|---------|-------------|
| `todo` | 读写 agent 本地任务状态 | 直接修改 `self.todos`，返回合成结果 |
| `memory` | 向持久化内存写入（有字符限制） | 直接写入 MemoryManager，返回合成结果 |
| `session_search` | 查询 session 历史 | 直接查询 SessionDB，返回合成结果 |
| `delegate_task` | 生成子 agent | 创建新 AIAgent 实例，运行子对话 |

### 设计原因

**并发执行**：大多数工具调用之间没有依赖关系（如同时 read_file 和 web_search），并发执行可以显著降低多工具调用的延迟。默认使用 `ThreadPoolExecutor` 而非 asyncio，因为工具处理器大多是同步的（文件 I/O、subprocess、HTTP 请求等）。

**结果按原始顺序插入**：模型在生成 tool_calls 时假设了特定的语义顺序。即使工具实际并发完成，结果也按模型指定的顺序排列，保持模型预期的上下文结构。

**Agent 级工具被拦截**：`todo`、`memory`、`session_search`、`delegate_task` 这些工具修改的是 agent 自身的状态（任务列表、持久化内存、session 数据库、子 agent），而非外部世界。将它们与普通工具在同一层分发会导致：
- 状态修改需要跨层传递（registry 不持有 agent 引用）
- 子 agent 的生命周期管理需要访问 session、budget、memory 等内部状态
- 合成结果可以更高效（不需要真实的工具注册/参数验证开销）

在 `run_agent.py` 内部拦截它们，让状态修改保持在同一层，也避免了循环依赖（registry → agent state → registry）。

---

## 回调接口

`AIAgent` 支持平台特定的回调，用于在 CLI、Gateway 和 ACP 集成中实现实时进度展示：

| 回调 | 触发时机 | 使用方 |
|------|---------|--------|
| `tool_progress_callback` | 每次工具执行前后 | CLI spinner、gateway 进度消息 |
| `thinking_callback` | 模型开始/停止思考时 | CLI "thinking..." 指示器 |
| `reasoning_callback` | 模型返回推理内容时 | CLI 推理展示、gateway 推理块 |
| `clarify_callback` | `clarify` 工具被调用时 | CLI 输入提示、gateway 交互消息 |
| `step_callback` | 每次完整 agent turn 结束后 | Gateway 步骤追踪、ACP 进度 |
| `stream_delta_callback` | 每个流式 token（启用时） | CLI 流式展示 |
| `tool_gen_callback` | 从流中解析出工具调用时 | CLI spinner 中的工具预览 |
| `status_callback` | 状态变更时（思考、执行等） | ACP 状态更新 |

### 设计原因

**回调而非事件总线**：使用简单的 callable 属性而非复杂的事件系统。每个回调是 AIAgent 构造函数的一个可选参数，默认 `None`。如果入口不提供某个回调，编排层就跳过该通知 — 不会报错，也不需要注册/注销。

**编排层决定"何时"，入口层决定"如何"**：这是一个单向依赖设计：
- `run_agent.py` 只负责在正确的时机调用回调，不关心回调做什么
- CLI/Gateway/ACP 各自传入自己的实现，负责渲染展示
- 同一个编排逻辑零修改适配所有入口

例如：`thinking_callback(True)` → CLI 显示 spinner，Gateway 发送 "typing..." 状态，ACP 更新编辑器状态栏。三者的展示方式完全不同，但 `run_agent.py` 不需要知道这些差异。

**为什么回调在构造函数注册而非每个方法调用传入**：回调的生命周期与 AIAgent 实例相同。一个 agent 实例在整个 `run_conversation()` 过程中使用同一组回调，不需要每次 turn 重新指定。构造时注入意味着回调是 agent "身份"的一部分（"我是一个 CLI agent" vs "我是一个 Gateway agent"）。

---

## 预算与回退

### IterationBudget

```python
class IterationBudget:
    """线程安全的迭代计数器，跨父子 agent 共享"""
    def __init__(self, max_iterations: int = 90):
        self._remaining = max_iterations
        self._lock = threading.Lock()

    def consume(self) -> bool:
        """消耗一次迭代，返回是否还有剩余"""
```

| 规则 | 值 | 原因 |
|------|-----|------|
| 默认迭代数 | **90** | 足够完成复杂多步任务，同时防止无限循环 |
| 子 agent 预算上限 | **50**（`delegation.max_iterations`） | 防止单个子任务消耗全部预算 |
| 父+子总迭代 | **可以超过**父 agent 上限 | 否则委托会大幅削减父 agent 的可用迭代 |
| `execute_code` 轮次 | **不计入**预算 | 代码执行的 trial-and-error 不应消耗对话预算 |
| 预算耗尽时 | 停止并返回已完成工作的**摘要** | 优雅降级，而非突然中断 |

### 预算压力预警

压力警告**注入到工具结果 JSON 中**，而非作为独立系统消息：

- **70%**：温和提醒 — "你已使用了 63/90 轮迭代，请开始收尾"
- **90%**：强烈警告 — "你只有 9 轮迭代剩余，立即总结并完成"

### Fallback Provider 链

```
主模型失败 → 检查 fallback_providers 列表 → 按序尝试
  ├── 429 (限流) → 切换到下一个 provider
  ├── 5xx (服务端错误) → 切换到下一个 provider
  └── 401/403 (鉴权错误) → 先尝试刷新凭证 → 仍失败则切换
成功 → 使用新 provider 继续对话
```

辅助任务各自拥有独立的回退链，通过 `auxiliary.*` 配置段控制。

### 设计原因

**压力警告注入工具结果 JSON 而非独立消息**：如果作为独立 system 消息注入，会破坏 prompt cache 的 prefix 命中（system prompt 变了）。注入到工具结果的 JSON 中意味着 model 在处理工具返回时能看到警告，但不会影响 system prompt 的缓存稳定性。

**子 agent 有独立预算但受上限约束**：完全独立预算会让 `delegate_task` 成为预算黑洞（一个委托消耗 90 次迭代）。完全共享预算又会让委托不够灵活。独立预算 + 上限（50）是平衡点 — 子 agent 有足够的自主空间，但单个委托不会耗尽系统资源。

**execute_code 不计入预算**：代码执行是典型的 trial-and-error 过程。如果每次 `execute_code` 后的修正都消耗预算，agent 会为了避免消耗而不敢验证自己的代码。免计费让 agent 有充分的空间进行自我验证。

---

## 上下文压缩与持久化

### 压缩触发条件

| 触发场景 | 阈值 | 时机 |
|----------|------|------|
| 预检压缩 | 对话超过上下文窗口的 **50%** | API 调用前 |
| Gateway 自动压缩 | 对话超过上下文窗口的 **85%** | 轮次之间 |

> 50% 和 85% 的差异是故意的：预检压缩给 API 调用留足余量（消息格式化会膨胀），Gateway 压缩更激进是因为长轮对话的上下文增长更快。

### 压缩流程

```text
1. 先将内存刷写到磁盘（防止数据丢失）
2. 将中间对话轮次摘要为紧凑的摘要内容
3. 保留最后 N 条消息完整不变（compression.protect_last_n，默认 20）
4. 工具调用/结果消息对保持完整（不拆分）
5. 生成新的 session 血缘 ID（压缩创建"子" session）
```

### Session 持久化

每个 turn 结束后：

- 消息保存到 SQLite session 存储（`hermes_state.py`）
- SQLite 使用 **WAL 模式** + **FTS5 全文搜索** + 抖动重试 + 被动 checkpoint
- 内存变更刷写到 `MEMORY.md` / `USER.md`
- 可通过 `/resume` 或 `hermes chat --resume` 恢复
- 压缩创建子 session，通过 `parent_session_id` 形成血缘链

### 设计原因

**先刷写内存再压缩**：压缩会丢失信息（中间轮次被摘要替代）。在丢失之前必须确保内存中的用户偏好、决策、学习内容已经被持久化。顺序是 `flush_memories() → compress → persist_session()`。

**工具调用/结果对不拆分**：工具调用和其结果是一个不可分割的原子对。如果把 `assistant(tool_calls=[read_file])` 压缩了但保留 `tool(result=...)`，你会得到一个没有调用者的结果。反过来也一样。保持配对完整是消息序列语义正确性的基本要求。

**Session 分裂（parent_session_id）而非原地覆盖**：压缩后的 session 是一个新的 session ID，通过 `parent_session_id` 链接到原始 session。这保留了完整的历史审计轨迹 — 你可以遍历血缘链回溯到最初的对话。如果原地覆盖旧 session，压缩前的对话历史就永久丢失了。

**WAL 模式 + 抖动重试**：多个 agent 可能同时写入同一个 session（Gateway 场景）。WAL 模式提供更好的并发写入性能。抖动重试处理临时的 SQLITE_BUSY 错误，避免多个写入者同时竞争时的一致性问题。

---

## 关键源文件

| 文件 | 用途 |
|------|------|
| `run_agent.py` | **AIAgent 类** — 完整的 agent loop |
| `agent/prompt_builder.py` | 从 memory、skills、context files、personality 组装系统 prompt |
| `agent/context_engine.py` | ContextEngine ABC — 可插拔的上下文管理 |
| `agent/context_compressor.py` | 默认引擎 — 有损摘要压缩算法 |
| `agent/prompt_caching.py` | Anthropic prompt 缓存标记和缓存命中率统计 |
| `agent/auxiliary_client.py` | 辅助 LLM 客户端，处理视觉、摘要等辅助任务 |
| `model_tools.py` | 工具 schema 收集、`handle_function_call()` 分发 |

---

## 设计原则总结

从 Agent Loop 的设计中可以提炼出以下贯穿始终的原则：

1. **缓存优先（Cache-first thinking）**：System prompt 的稳定性、ephemeral injection 的分离、压力警告注入工具结果而非系统消息 — 这些设计都服务于最大化 prompt cache 命中率。在 LLM API 按 token 计费的模型下，cache 命中直接转化为成本节约和延迟降低。

2. **优雅降级（Graceful degradation）**：预算耗尽时不崩溃而是返回摘要，API 失败时不中断而是 fallback，压缩时不丢数据而是先生成摘要。每个异常路径都有设计好的降级行为。

3. **原子性（Atomicity）**：一个 turn 要么完整要么不存在（中断丢弃全部响应），工具调用/结果对不可分割（压缩不拆分），session 通过血缘链保持可追溯（不原地覆盖）。这些约束保证了状态的正确性和可调试性。

4. **入口无关（Entry-agnostic）**：CLI、Gateway、ACP、Cron — 都使用同一个 AIAgent 类。平台差异通过回调机制隔离，核心编排逻辑只写一次。

5. **同步核心 + 异步边界**：AIAgent 本身是同步的（简化了状态管理、错误处理和调试），但工具执行（ThreadPoolExecutor）和 API 调用（后台线程 + 中断）在边界处使用并发。这是一种务实的混合模式 — 核心逻辑的简单性 + I/O 密集型操作的并行性。
