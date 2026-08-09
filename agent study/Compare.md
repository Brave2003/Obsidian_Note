# AI Agent 框架架构对比：Hermes / Claw-code / OpenClaw / Claude Code

## 一、Hermes Agent：run_agent.py 的同步巨人

### 核心架构

`run_conversation()` 是一个 3700+ 行的同步方法（run_agent.py:10980-14712），是整个 Hermes 的心脏。它的核心循环极其朴素：

```python
while (api_call_count < self.max_iterations
       and self.iteration_budget.remaining > 0) \
      or self._budget_grace_call:
    # 1. 构建 API 消息（注入记忆、上下文、缓存控制）
    # 2. 调用 LLM API（带重试、降级链）
    # 3. 标准化响应（处理各 provider 的差异格式）
    # 4. 提取 tool_calls
    # 5. 并行或串行执行工具
    # 6. 检查结果 → 追加到消息历史
    # 7. 上下文压缩检查
```

### 关键设计决策

| 设计 | 实现 | 理由 |
|------|------|------|
| 同步循环 | while + 阻塞调用 | Python async 生态碎片化，大量第三方库（Playwright、Firecrawl）是同步的 |
| 线程池并行工具 | ThreadPoolExecutor | 在同步循环内实现工具并行，同时保持与同步库的兼容 |
| 硬上限 90 轮 | max_iterations=90 | 保守策略，防止失控循环消耗过多 token |
| 共享迭代预算 | IterationBudget | 父 Agent 和子 Agent（delegate）共享预算，防止子任务耗尽资源 |
| 多 Provider 适配 | `_create_openai_client()` 分支 | 支持 OpenAI、Anthropic、Bedrock、Codex Responses、OpenRouter 等，运行时切换 |
| 上下文压缩 | 可插拔 ContextCompressor | 触发时总结历史、重建会话、保留 todo 状态 |
| reasoning 多格式处理 | `_extract_reasoning()` | 处理 DeepSeek、Qwen、Moonshot、OpenRouter 等 4+ 种 reasoning 格式 |

### 代码规模的含义

`AIAgent.__init__` 有 ~65 个参数，`run_conversation` 有 3700+ 行。这说明了什么？

- **高集成度**：所有逻辑（重试、降级、压缩、工具执行、记忆、审批）都在一个类里
- **紧耦合**：修改任何子系统都可能影响整个循环
- **防御性编程**：大量边缘 case 处理（空响应、无效 JSON、reasoning 预算耗尽、incomplete scratchpad）

这就像一个"瑞士军刀式"的 monolith —— 功能极其丰富，但维护成本高。

---

## 二、Claw-code：Rust 的同步精简主义

### 核心架构

`ConversationRuntime::run_turn()`（rust/crates/runtime/src/conversation.rs:318-519）也是一个同步循环，但比 Hermes 精简得多：

```rust
pub fn run_turn(&mut self, user_input: impl Into<String>, ...) -> Result<TurnSummary, RuntimeError> {
    loop {
        iterations += 1;
        if iterations > self.max_iterations { /* error */ }

        // 阻塞调用 API
        let events = self.api_client.stream(request)?;

        // 提取 tool_use
        let pending_tool_uses = extract_tools(&assistant_message);

        if pending_tool_uses.is_empty() { break; }

        // 串行执行工具（for 循环）
        for (tool_use_id, tool_name, input) in pending_tool_uses {
            let result = self.tool_executor.execute(&tool_name, &input)?;
            self.session.push_message(result)?;
        }
    }
}
```

### 与 Hermes 的关键差异

| 方面 | Hermes | Claw-code |
|------|--------|-----------|
| 工具执行 | ThreadPoolExecutor 并行 | for 循环串行 |
| 流式处理 | 流式读取 + 实时回调 | 收集到 Vec<AssistantEvent> 后处理 |
| MCP 桥接 | 异步 stdio 进程 | sync→async 通过 thread-per-call + Tokio runtime |
| 会话持久化 | SQLite JSON | JSONL 追加文件 |
| Provider 支持 | 运行时切换多 provider | Provider chain（fallback） |
| 权限系统 | 审批回调 + 配置 | PermissionEnforcer trait + 分类 |
| 代码规模 | 3700+ 行循环 | ~200 行循环 |

### Rust 的设计哲学

Claw-code 的架构更"Rust 化"：
- **分层清晰**：api（Provider 客户端）→ tools（工具规格+执行）→ runtime（会话编排）
- **类型安全**：`ContentBlock::ToolUse { id, name, input }` 是 enum variant，编译时检查
- **错误处理**：`Result<T, E>` 贯穿始终，没有异常
- **unsafe 禁止**：workspace 级别 `unsafe_code = "forbid"`

> **核心洞察**：Claw-code 的 MCP 桥接设计特别有意思：为了在一个同步的 conversation loop 中调用异步的 MCP 工具，它采用了一个极端但可靠的方案——每个 MCP 调用 spawn 一个独立线程，在线程内创建一个全新的 Tokio runtime，阻塞执行异步调用。这避免了在同步代码中引入 async/await 的传染性，代价是每个 MCP 调用都有线程创建开销。这是一种"隔离而非集成"的策略。

---

## 三、OpenClaw：TypeScript 的全异步 Actor 模型

### 核心架构

OpenClaw 的架构与前两者完全不同。它使用全异步架构 + Actor 队列 + 流式事件消费：

```typescript
// AcpSessionManager.runTurn() (manager.core.ts:708)
async runTurn(input: AcpRunTurnInput): Promise<void> {
    await this.withSessionActor(sessionKey, async () => {
        // 1. 确保 runtime handle（缓存复用）
        const { runtime, handle, meta } = await this.ensureRuntimeHandle(...);
        
        // 2. 设置 session 状态为 running
        await this.setSessionState({ state: "running" });
        
        // 3. 消费事件流（for await）
        const turnOutcome = await this.awaitTurnWithTimeout({
            turnPromise: consumeAcpTurnStream({
                runtime,
                turn: { handle, text, mode, requestId, signal },
                onOutputEvent: (event) => { /* 流式输出 */ },
                onEvent: input.onEvent,
            }),
            timeoutMs: ...,
            onTimeout: async () => { /* 超时清理 */ },
        });
        
        // 4. 自动重试（1 次 fresh handle retry）
        if (!turnOutcome.sawTerminalEvent && attempt < 2) {
            retryFreshHandle = true;
            continue;
        }
        
        // 5. 设置 session 状态为 idle/error
        await this.setSessionState({ state: "idle" });
    });
}
```

### 关键设计模式

| 模式 | 实现 | 目的 |
|------|------|------|
| Actor 队列 | SessionActorQueue（KeyedAsyncQueue） | 保证每个 session 的 turn 串行执行，避免竞态 |
| Runtime 缓存 | RuntimeCache | 复用 provider runtime handle，减少连接开销 |
| 事件流消费 | `for await...of runtime.runTurn()` | 纯流式，支持 text_delta、tool_call、done、error |
| 自动重试 | `prepareFreshHandleRetry()` | turn 失败时自动重建 handle 重试 1 次 |
| 超时控制 | `Promise.race + AbortController` | 防止 runaway turn 阻塞 session |
| 持久化状态 | SessionEntry（磁盘/存储） | session 状态（idle/running/error）持久化，支持崩溃恢复 |
| 后台任务 | BackgroundTaskContext | 跟踪 turn 进度，支持外部查询 |

### 与 Hermes 的本质差异

- **Hermes**：一个进程 = 一个 Agent = 一个循环，同步循环内处理一切
- **OpenClaw**：Gateway 进程 = 多 Session = Actor 队列，每个 session 的 turn 串行，但多 session 并发，异步事件流贯穿始终

OpenClaw 不是"一个 Agent 在循环"，而是"一个 Gateway 管理多个 Agent Session，每个 Session 通过 Actor 队列调度 turn"。这更符合服务端架构——同时服务多个用户/频道。

---

## 四、Claude Code：AsyncGenerator + 动态压缩

Claude Code（Anthropic 官方实现）的架构：

```typescript
// 伪代码，基于公开架构文档
async function* runAgentLoop(userInput: string): AsyncGenerator<StreamEvent> {
    while (true) {
        const response = await streamLLM(messages);
        
        for await (const chunk of response) {
            yield { type: "text_delta", text: chunk.text };
        }
        
        if (response.tool_calls) {
            for (const tool of response.tool_calls) {
                yield { type: "tool_start", name: tool.name };
                const result = await executeTool(tool);
                yield { type: "tool_result", result };
                messages.push(toolResultMessage(result));
            }
        } else {
            yield { type: "done" };
            return;
        }
        
        // 动态压缩：如果上下文接近上限，自动触发
        if (estimateTokens(messages) > threshold) {
            messages = await compressContext(messages);
        }
    }
}
```

### 核心特点

- **AsyncGenerator + yield**：流式输出天然支持，UI 可以实时显示每个 token
- **无限循环 + 动态压缩**：没有硬上限，依靠压缩来延长对话生命周期
- **前端-后端协同**：UI 进程（React/Node）和后端通过流式 RPC 通信

---

## 五、四框架对比矩阵

| 维度 | Hermes Agent | Claw-code | OpenClaw | Claude Code |
|------|-------------|-----------|----------|-------------|
| 语言 | Python | Rust | TypeScript | TypeScript |
| 循环模式 | 同步 while | 同步 loop | 异步 for await | AsyncGenerator yield |
| 并发模型 | ThreadPoolExecutor（工具并行） | 串行 | Actor 队列 + 多 session 并发 | 单 session 串行 |
| 流式 | 支持，回调式 | 收集后处理 | 原生事件流 | 原生 AsyncGenerator |
| 迭代上限 | 硬上限 90 | usize::MAX | 超时控制 | 无上限（动态压缩） |
| 上下文压缩 | 可插拔引擎，多触发点 | turn 后自动 | turn 后 CLI compaction | 动态、自动 |
| 多 Provider | 运行时切换 | Provider chain fallback | Runtime 抽象层 | Anthropic 为主 |
| MCP | 异步 stdio 进程 | sync→async thread bridge | 原生集成 | 原生集成 |
| 会话持久化 | SQLite | JSONL 追加 | 结构化存储 | 云端/本地 |
| 目标场景 | 多平台消息网关 | CLI 本地执行 | 多用户消息网关 | 开发者 CLI |
| 架构复杂度 | 高（monolith） | 中（分层清晰） | 高（分布式状态） | 中 |
| 核心文件行数 | run_conversation: 3700+ | run_turn: ~200 | runTurn: ~250 | ~500 |

---

## 六、Trade-off 分析：为什么每个框架做了不同的选择？

### 1. 同步 vs 异步：生态系统的囚徒

**Hermes 选择同步：**
- Python async 生态碎片化
- Playwright（同步 API 更成熟）
- Firecrawl、各种 SDK 多为同步
- 需要兼容大量第三方工具的同步调用
- 用 ThreadPoolExecutor 在同步框架内"打补丁"实现并行

**Claw-code 选择同步：**
- Rust async = 传染性（一旦用 async，整个调用链都要 async）
- CLI 场景不需要高并发（单用户、单 session）
- 简化的错误处理（Result 而非 async Result）
- MCP 用 thread bridge 隔离 async 边界

**OpenClaw 选择异步：**
- Node.js 的 async 是原生一致的
- Gateway 需要同时服务多个 session
- 流式事件是 TypeScript 的强项（AsyncIterator）
- Actor 队列在 async 环境下实现简单

**Claude Code 选择 AsyncGenerator：**
- 流式 UX 是核心卖点
- 前端（React）天然消费 Generator
- TypeScript 对 async generator 支持优秀
- 单用户场景不需要复杂的并发控制

### 2. 硬上限 vs 动态压缩：风险偏好

| 策略 | 代表 | 优点 | 缺点 |
|------|------|------|------|
| 硬上限 | Hermes (90) | 防止 token 爆炸，成本可控 | 复杂任务可能中途被截断 |
| 自动压缩 | Claude Code | 理论上无限对话 | 压缩可能丢失信息，增加延迟 |
| 混合 | OpenClaw | turn 级超时 + CLI compaction | 需要外部触发 compaction |

Hermes 的 90 轮上限反映了它的使用场景：运行在 Telegram/Discord/Slack 等消息平台上，用户可能不在线，失控的循环会产生巨额账单。Claude Code 面对的是坐在终端前的开发者，可以实时观察并手动干预。

### 3. 工具并行 vs 串行：正确性 vs 效率

- **Hermes 并行**：ThreadPoolExecutor 同时执行多个独立工具，减少总等待时间
- **Claw-code 串行**：for 循环逐个执行，简化状态管理，避免工具间的副作用竞态
- **OpenClaw 由 runtime 决定**：底层的 Codex/Claude runtime 自己决定工具调度

### 4. 架构复杂度：功能范围 vs 可维护性

**Hermes**：单体 ~15k 行 run_agent.py
- 所有功能在一个文件：工具、记忆、压缩、审批、子代理
- 修改任何功能都需要理解整个循环
- 好处：没有跨模块协调的 overhead

**Claw-code**：分层 ~30+ crates
- api → tools → runtime → plugins → cli
- 每个 crate <500 行（约定）
- 修改工具不影响 runtime 逻辑
- 好处：编译时类型安全保证接口一致性

**OpenClaw**：网关 + 插件
- Core 保持扩展无关
- 插件通过 SDK 边界接入
- 状态持久化在 gateway 层
- 好处：第三方插件生态，多用户隔离

---

## 七、总结：每个框架的"灵魂"

| 框架 | 一句话描述 | 核心取舍 |
|------|----------|---------|
| Hermes Agent | "一个用同步 Python 写的全能瑞士军刀" | 牺牲架构优雅换取生态系统兼容性 |
| Claw-code | "用 Rust 重写的精简版 Claude Code" | 牺牲高级功能换取类型安全和可维护性 |
| OpenClaw | "面向多用户的多通道 AI 网关" | 牺牲简单性换取扩展性和服务端能力 |
| Claude Code | "为开发者优化的流式 AI 助手" | 牺牲通用性换取极致的开发者体验 |

> **核心洞察**：这四个框架代表了 AI Agent 架构的四种"极端"：
>
> 1. **Hermes = 兼容性优先**：Python + 同步 + 单体，为了兼容碎片化生态牺牲了架构 purity
> 2. **Claw-code = 正确性优先**：Rust + 同步 + 分层，编译时保证正确性，运行时牺牲并发
> 3. **OpenClaw = 扩展性优先**：TypeScript + 异步 + Actor 模型，为了插件生态和多用户做了最复杂的设计
> 4. **Claude Code = 体验优先**：TypeScript + AsyncGenerator，为了流式开发者体验做了最专注的设计
>
> 有趣的是，同步 vs 异步的选择与语言强相关：Python 和 Rust 项目都选了同步（因为各自的 async 生态痛点），而 TypeScript 项目都选了异步（因为 JS 的 async 是原生一致的）。这不是巧合，而是生态系统演化的结果。
