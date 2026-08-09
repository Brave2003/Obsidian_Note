# Context Compression & Prompt Caching（上下文压缩与缓存）

> Hermes Agent 使用**双重压缩系统**和 **Anthropic prompt caching** 来高效管理长对话的上下文窗口使用。

**核心文件**：`agent/context_engine.py`（ABC）、`agent/context_compressor.py`（默认引擎）、`agent/prompt_caching.py`、`gateway/run.py`（session hygiene）、`run_agent.py`（`_compress_context`）

---

## 可插拔的 Context Engine

上下文管理建立在 `ContextEngine` ABC（`agent/context_engine.py`）之上。内置的 `ContextCompressor` 是默认实现，但插件可以替换为替代引擎（如 LCM: Lossless Context Management）。

```yaml
context:
  engine: "compressor"    # 默认 — 内置有损摘要
  engine: "lcm"           # 示例 — 插件提供的无损上下文
```

### 引擎职责

- 决定何时应触发压缩（`should_compress()`）
- 执行压缩（`compress()`）
- 可选地暴露 agent 可调用的工具（如 `lcm_grep`）
- 追踪 API 响应中的 token 使用

### 解析顺序

```
1. 检查 plugins/context_engine/<name>/ 目录
2. 检查通用插件系统（register_context_engine()）
3. 回退到内置 ContextCompressor
```

插件引擎**不会**自动激活 — 用户必须显式设置 `context.engine`。默认的 `"compressor"` 始终使用内置引擎。

---

## 双重压缩系统

Hermes 有两个**独立运行的压缩层**：

```
                     ┌──────────────────────────┐
  Incoming message   │   Gateway Session Hygiene │  在上下文的 85% 触发
  ─────────────────► │   (pre-agent, 粗略估算)    │  大规模 session 的安全网
                     └─────────────┬────────────┘
                                   │
                                   ▼
                     ┌──────────────────────────┐
                     │   Agent ContextCompressor │  在上下文的 50% 触发（默认）
                     │   (in-loop, 精确 token)   │  正常的上下文管理
                     └──────────────────────────┘
```

### 1. Gateway Session Hygiene（85% 阈值）

位于 `gateway/run.py`。在 agent 处理消息**之前**运行的安全网。防止 session 在 turn 之间增长过大（例如 Telegram/Discord 中过夜积累）：

- **阈值**：固定为模型上下文长度的 **85%**
- **Token 来源**：优先使用上一轮 API 报告的实际 token；回退到基于字符的粗略估算
- **触发条件**：仅当 `len(history) >= 4` 且压缩启用
- **目的**：捕获逃脱 agent 自身压缩器的 session

**设计原因**：设置为 50%（与 agent 相同）会导致长 Gateway session 中每个 turn 都触发过早压缩。85% 是安全网 — 正常情况下 agent 的 compressor 在 50% 时压缩，hygiene 只在 compressor 失败或跳过时才介入。

### 2. Agent ContextCompressor（50% 阈值，可配置）

位于 `agent/context_compressor.py`。在 agent 的 tool loop 内部运行的**主要压缩系统**，能访问精确的 API 报告 token 计数。

---

## 配置

```yaml
compression:
  enabled: true              # 默认: true
  threshold: 0.50            # 上下文窗口比例（默认: 0.50 = 50%）
  target_ratio: 0.20         # 保留为 tail 的阈值 token 比例（默认: 0.20）
  protect_last_n: 20         # 最少保护的尾部消息数（默认: 20）

auxiliary:
  compression:
    model: null              # 覆盖摘要模型（默认: 自动检测）
    provider: auto
    base_url: null
```

### 参数详解

| 参数 | 默认值 | 范围 | 说明 |
|------|--------|------|------|
| `threshold` | `0.50` | 0.0-1.0 | prompt tokens ≥ `threshold × context_length` 时触发 |
| `target_ratio` | `0.20` | 0.10-0.80 | tail 保护的 token 预算：`threshold_tokens × target_ratio` |
| `protect_last_n` | `20` | ≥1 | 始终保留的最近消息最小数量 |
| `protect_first_n` | `3` | 硬编码 | System prompt + 第一轮交换始终保留 |

**设计原因 — protect_first_n 硬编码为 3**：System prompt、第一条 user 消息和第一条 assistant 回复定义了 agent 的初始上下文和行为方向。如果被摘要掉，后续压缩摘要会失去语义锚点。

### 计算示例（200K 上下文模型，默认参数）

```text
context_length       = 200,000
threshold_tokens     = 200,000 × 0.50  = 100,000  ← 触发压缩的阈值
tail_token_budget    = 100,000 × 0.20  =  20,000  ← 保护尾部的 token 预算
max_summary_tokens   = min(200,000 × 0.05, 12,000) = 10,000
```

> 阈值基于**主 agent 模型**的上下文窗口计算，非辅助/摘要模型。

---

## 压缩算法

`ContextCompressor.compress()` 遵循四阶段算法：

### Phase 1：清理旧工具结果（廉价，无需 LLM）

protected tail 之外的旧工具结果（>200 字符）被替换为：

```text
[Old tool output cleared to save context space]
```

**设计原因**：旧工具输出通常是最冗长但价值最低的内容。删掉它们可以显著减少需要 LLM 处理的摘要量，同时不丢失重要的对话语义。

### Phase 2：确定边界

```
┌─────────────────────────────────────────────────────────────┐
│  Message list                                               │
│  [0..2]  ← protect_first_n (system + 第一轮交换)             │
│  [3..N]  ← 中间 turn → 被摘要                                │
│  [N..end] ← tail (按 token 预算 或 protect_last_n)           │
└─────────────────────────────────────────────────────────────┘
```

Tail 保护是**基于 token 预算**的：从末尾向后遍历，累积 token 直到预算耗尽。如果预算保护的消息少于 `protect_last_n`，回退到固定数量。

边界会**对齐**以避免拆散 tool_call/tool_result 组。`_align_boundary_backward()` 向后遍历连续的 tool 结果找到父 assistant 消息。

**设计原因**：工具调用和其结果是不可分割的原子对。拆散会导致孤立的 assistant 消息或 tool 结果，被 provider API 拒绝。

### Phase 3：生成结构化摘要

### ⚠️ 摘要模型上下文长度警告

摘要模型必须具有**至少与主 agent 模型相同**的上下文窗口。整个中间部分在单次 `call_llm(task="compression")` 中发送。如果摘要模型上下文更小，API 返回上下文长度错误 → 压缩器**丢弃中间 turn 而不生成摘要**，静默丢失对话上下文。**这是压缩质量下降最常见的原因。**

**设计原因 — 为什么不用分块摘要**：分块会导致摘要的摘要（信息逐层丢失）、跨块引用断裂、多次 LLM 调用增加延迟和费用。单次调用保证摘要质量和语义连贯性。

### 结构化摘要模板

```text
## Goal
[用户要完成什么]

## Constraints & Preferences
[用户偏好、编码风格、约束、重要决策]

## Progress
### Done — [已完成的具体工作]
### In Progress — [正在进行的工作]
### Blocked — [阻塞或问题]

## Key Decisions
[重要技术决策及原因]

## Relevant Files
[读取、修改或创建的文件 — 每个附简要说明]

## Next Steps
[接下来需要做什么]

## Critical Context
[具体的值、错误消息、配置细节]
```

摘要预算随被压缩内容缩放：`content_tokens × 0.20`，最小 2000，最大 `min(context_length × 0.05, 12000)`。

**设计原因 — 结构化模板**：普通自由文本摘要在多次压缩后会丢失结构化信息。模型可能记住"做了什么"但忘记"为什么"、"哪个文件"、"具体配置"。八字段模板确保每个关键维度都被覆盖。

### Phase 4：组装压缩后的消息

1. **Head 消息**（首次压缩时在 system prompt 追加注释）
2. **摘要消息**（角色选择避免违反连续同角色规则）
3. **Tail 消息**（未修改）

孤立的 tool_call/tool_result 对被 `_sanitize_tool_pairs()` 清理。

### 迭代重压缩

后续压缩时，先前的摘要传递给 LLM，指令是**更新**而非从头摘要。保留多次压缩之间的信息 — 项目从 "In Progress" 移到 "Done"，过时信息被移除。

**设计原因**：每次从头摘要会导致"摘要漂移" — 每次丢失 5-10% 信息，5 次压缩后只剩原始信息的 60%。更新模式大幅减少信息损失。

---

## 压缩前后对比

### 压缩前（45 条消息，~95K tokens）

```text
[0] system:    "You are a helpful assistant..."
[1] user:      "Help me set up a FastAPI project"
[2] assistant: <tool_call> terminal: mkdir project </tool_call>
[3] tool:      "directory created"
    ... 30+ turns of file editing, testing ...
[44] user:      "Great, also add error handling"
```

### 压缩后（25 条消息，~45K tokens）

```text
[0] system:    "You are a helpful assistant...
               [Note: Some earlier conversation turns have been compacted...]"
[1] user:      "Help me set up a FastAPI project"
[2] assistant: "[CONTEXT COMPACTION]
               ## Goal
               Set up a FastAPI project with tests and error handling
               ## Progress
               ### Done
               - Created project structure, 5 API endpoints, 10 test cases
               ### In Progress
               - Fixing 2 failing tests
               ## Next Steps
               - Fix failing test fixtures, add error handling"
[3] user:      "Fix the failing tests"
[4] assistant: <tool_call> read_file: tests/test_api.py </tool_call>
    ...
```

---

## Prompt Caching（Anthropic）

通过缓存对话前缀，将多轮对话的输入 token 成本降低约 **~75%**。

### 策略：system_and_3

Anthropic 允许每请求最多 **4 个** `cache_control` breakpoint：

```
Breakpoint 1: System prompt              (跨所有 turn 稳定)
Breakpoint 2: 倒数第 3 条非系统消息  ─┐
Breakpoint 3: 倒数第 2 条非系统消息    ├─ 滚动窗口
Breakpoint 4: 倒数第 1 条非系统消息   ─┘
```

### 缓存感知设计模式

1. **System prompt 稳定**：避免中途修改（压缩仅在首次时追加注释）
2. **消息顺序重要**：缓存命中需要前缀匹配，中间增删消息导致之后所有内容缓存失效
3. **压缩与缓存交互**：压缩后压缩区域缓存失效，但 system prompt 缓存存活。滚动 3 消息窗口在 1-2 个 turn 内重建
4. **TTL 选择**：默认 `5m`，长会话用 `1h`

### 启用条件

- 模型是 Anthropic Claude（通过名称检测）
- Provider 支持 `cache_control`（原生 Anthropic API 或 OpenRouter）

---

## 上下文压力警告（已移除）

> 中间压力警告已被移除。注释说明："No intermediate pressure warnings — they caused models to 'give up' prematurely on complex tasks"

**设计原因**：早期版本在 70%/90% 时发送警告，但导致模型过早放弃任务、跳过验证步骤、"节省 token"。直接压缩让模型专注完成任务。

---

## 关键源文件

| 文件 | 用途 |
|------|------|
| `agent/context_engine.py` | ContextEngine ABC — 可插拔接口 |
| `agent/context_compressor.py` | 默认引擎 — 四阶段有损摘要算法 |
| `agent/prompt_caching.py` | Anthropic prompt 缓存策略和标记注入 |
| `gateway/run.py` | Gateway Session Hygiene（85% 安全网） |
| `run_agent.py` | `_compress_context()` — 触发压缩的编排逻辑 |
