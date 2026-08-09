# AI SDK Harnesses

> AI SDK Harnesses 提供统一的 API 层，通过单一接口运行成熟的 Agent 运行环境（Claude Code、Codex、Pi 等）。所有 Agent 在沙箱中运行，保护宿主机环境安全。

- **官网**: https://ai-sdk.dev/docs/ai-sdk-harnesses
- **状态**: 实验性（可能包含破坏性变更）
- **安装**: `npm install @ai-sdk/harness/agent @ai-sdk/harness-claude-code @ai-sdk/sandbox-vercel`

---

## 1. 概述 (Overview)

### 核心定位

AI SDK Harnesses 与 Provider/Model 抽象**相互独立**，解决的是不同层面的问题：

| 抽象 | 暴露内容 | 使用场景 |
|------|------|------|
| **Provider** | 暴露模型给 `generateText` / `streamText` | 直接控制模型调用和工具循环 |
| **Harnesses** | 暴露 Agent 运行环境给 `HarnessAgent` | 使用现有 Agent 运行时的完整能力 |

Provider 让你直接与 LLM 交互，自己管理工具循环、对话历史和状态。Harnesses 则将整个 Agent 运行时（如 Claude Code）作为一个黑盒来调用——它自带工具集、权限模型、对话管理和沙箱执行能力。`HarnessAgent` 的输出会映射为 AI SDK 的流和响应类型，可以无缝传递给 `toUIMessageStream` 并用 `useChat` 渲染。

### 何时使用 Harness

**使用 Harness 的场景：**
- 需要可检查和修改沙箱代码的编程 Agent（如自动修 bug、生成项目文件）
- Agent 运行环境自带内置工具（文件读写、Shell 命令、网络搜索）和权限模型，不需要自己实现
- 多轮会话，运行时自行管理对话历史（不需要手动拼接消息数组）
- 需要保留原生 Harness 的完整行为（如 Claude Code 的权限提示、Codex 的交互模式）

**使用 Provider/Model 的场景：**
- 需要对模型调用和工具循环有完全的控制权
- 需要结构化输出（`Output.object()` / `Output.array()`）或自定义 Agent 架构
- 不需要沙箱，只需要文本生成能力

### 四大核心组件

| 组件 | 说明 |
|------|------|
| **HarnessAgent** | 应用代码中的 AI SDK Agent 实现，提供 `generate()` 和 `stream()` 方法 |
| **Harness Adapter** | 连接特定运行时的包（如 `@ai-sdk/harness-claude-code`），标准化会话、流事件、工具、用量和配置 |
| **Sandbox Provider** | 隔离的文件系统和进程环境（`@ai-sdk/sandbox-vercel` 或 `@ai-sdk/sandbox-just-bash`） |
| **Session** | 实时会话和状态管理，持有活跃的运行时连接 |

---

## 2. HarnessAgent

`HarnessAgent` 是 AI SDK 中 `Agent` 接口的实现，由 Harness Adapter 提供底层能力。它提供 `generate()` 和 `stream()` 两个方法，返回与 AI SDK 兼容的结果，而实际工作由预配置的 Harness 运行时完成。

### 创建 Agent

Agent 在**模块作用域**构建，仅持有配置（不持有活跃会话）。活跃状态归 `HarnessAgentSession` 管理。这种设计让 Agent 可以复用，每次请求创建独立的会话。

```ts
import { HarnessAgent } from '@ai-sdk/harness/agent';
import { claudeCode } from '@ai-sdk/harness-claude-code';
import { createVercelSandbox } from '@ai-sdk/sandbox-vercel';

export const agent = new HarnessAgent({
  harness: claudeCode,
  sandbox: createVercelSandbox({ runtime: 'node24', ports: [4000] }),
  instructions: '你是一位谨慎的编程助手。优先小改动，解释权衡。',
});
```

桥接型 Harness（Claude Code、Codex、Deep Agents、OpenCode）需要真实的网络沙箱（`@ai-sdk/sandbox-vercel`），因为它们需要通过沙箱暴露的端口与运行时通信。主机运行时 Harness（Pi）可以也使用 `@ai-sdk/sandbox-just-bash`，因为不需要沙箱端口。

### 运行轮次

**`generate()` — 等待完整结果：** 消耗整个轮次，返回 `GenerateTextResult`，包含 `result.text`、`result.steps`、`result.usage`、`result.responseMessages` 等完整信息。

```ts
const session = await agent.createSession();
try {
  const result = await agent.generate({
    session,
    prompt: '为这个仓库创建一个简短的 TODO.md。',
  });
  console.log(result.text);
} finally {
  await session.destroy();
}
```

**`stream()` — 流式增量输出：** 返回 `StreamTextResult`，可以通过 `for await...of` 逐块消费输出。每个 part 可以是 `text-delta`（文本增量）、工具调用、工具结果等类型。

```ts
const session = await agent.createSession();
try {
  const result = await agent.stream({
    session,
    prompt: '为这个仓库创建一个简短的 TODO.md。',
  });
  for await (const part of result.stream) {
    if (part.type === 'text-delta') {
      process.stdout.write(part.text);
    }
  }
} finally {
  await session.destroy();
}
```

### 返回结果兼容性

`generate()` 返回 `GenerateTextResult`，`stream()` 返回 `StreamTextResult`。可以消费的属性包括：

- `result.text` — 完整文本输出
- `result.stream` — 流式数据
- `result.steps` — 多步骤执行记录（含每步的工具调用和结果）
- `result.usage` — token 使用量统计
- `result.responseMessages` — 模型响应消息列表

Harness 特有事件（文件变更、上下文压缩）会以动态 `provider-executed` tool parts 的形式呈现，便于在 UI 中统一渲染。

### 消息和历史

Harness 会话拥有**原生对话历史**，这是与模型调用的关键区别。当你传入 `messages` 或消息数组的 `prompt` 时，`HarnessAgent` 只提取**最新的用户消息**作为本轮输入，不会把完整的对话历史回放给 Harness。Harness 运行时自己维护对话上下文。

这意味着在聊天路由中，应该持久化和恢复 Harness 会话，而不是依赖消息回放来重建对话状态。

### 会话生命周期

每个会话必须显式结束。四种生命周期方法：

| 方法 | 说明 |
|------|------|
| `session.destroy()` | 停止运行时，**丢弃**可恢复性。适用于一次性脚本和测试 |
| `session.detach()` | 挂起运行时，返回恢复状态，**保持沙箱热**。适用于需要多轮连续性的 HTTP 路由 |
| `session.stop()` | 保存恢复状态，然后停止运行时和沙箱。下一请求从持久化状态恢复 |
| `session.suspendTurn()` | 跨进程边界挂起活跃轮次（高级用途） |

**多轮 HTTP 路由示例：**

```ts
const session = await agent.createSession({ sessionId: chatId });
try {
  const result = await agent.stream({ session, messages });
  for await (const part of result.stream) {
    if (part.type === 'text-delta') process.stdout.write(part.text);
  }
  const resumeState = await session.detach();
  await persistResumeState({ chatId, resumeState });
} catch (error) {
  await session.destroy();
  throw error;
}
```

### 恢复会话

持久化不透明的恢复状态，下次请求时用原始 `sessionId` 传回。`HarnessAgent` 会验证恢复状态由同一 Harness Adapter 生成后再交给运行时。

```ts
const resumeState = await loadResumeState({ chatId });
const session = await agent.createSession(
  resumeState
    ? { sessionId: chatId, resumeFrom: resumeState }
    : { sessionId: chatId },
);
```

### 暂停和继续轮次

对于需要跨进程边界移交活跃轮次的高级工作流，先挂起轮次并持久化继续状态：

```ts
// 挂起
const continuationState = await session.suspendTurn();
await persistContinuationState({ chatId, continuationState });

// 恢复并继续
const session = await agent.createSession({
  sessionId: chatId,
  continueFrom: continuationState,
});
const result = await agent.continueStream({ session });
```

### 沙箱配置

`HarnessAgent` 通过 `sandboxConfig` 在 Harness 启动前准备沙箱环境：

| 配置项 | 说明 |
|------|------|
| `workDir` | 可选，相对路径，作为会话工作目录 |
| `bootstrapHash` | 当 bootstrap 输出变更时使快照失效，强制重建模板 |
| `onBootstrap` | 沙箱模板创建时运行一次，结果可被快照复用（适合昂贵的安装操作） |
| `onSession` | 每个沙箱会话后运行（包括恢复的会话），适合每个会话独立的文件配置 |

```ts
const agent = new HarnessAgent({
  harness: claudeCode,
  sandbox: createVercelSandbox({ runtime: 'node24', ports: [4000] }),
  sandboxConfig: {
    workDir: 'repo',
    bootstrapHash: 'ripgrep-v1',
    onBootstrap: async ({ session, abortSignal }) => {
      const result = await session.run({
        command: 'command -v rg >/dev/null || (apt-get update && apt-get install -y ripgrep)',
        abortSignal,
      });
      if (result.exitCode !== 0) throw new Error(`安装失败: ${result.stderr}`);
    },
    onSession: async ({ session, sessionWorkDir, abortSignal }) => {
      await session.writeTextFile({
        path: `${sessionWorkDir}/README.md`,
        content: 'Session notes.',
        abortSignal,
      });
    },
  },
});
```

### 预准备沙箱模板

当需要提前创建或刷新可复用的沙箱模板时（例如在部署时预热），使用 `prepareHarnessSandboxTemplate()`：

```ts
import { prepareHarnessSandboxTemplate } from '@ai-sdk/harness/agent';

await prepareHarnessSandboxTemplate({
  harness: claudeCode,
  sandboxProvider: createVercelSandbox({ runtime: 'node24', ports: [4000] }),
  sandboxConfig: { /* ... */ },
});
```

`prepareSandboxForHarness()` 适用于自行管理原生沙箱生命周期时——它应用 Harness bootstrap 配方和 `onBootstrap`，但不停止或快照沙箱。

### Agent 设置汇总

| 设置 | 说明 |
|------|------|
| `harness` | Adapter 实例，指定底层运行时 |
| `sandbox` | `HarnessV1SandboxProvider`，提供隔离环境 |
| `id` | 可选稳定 Agent 标识 |
| `instructions` | 仅对新鲜会话应用一次的系统提示 |
| `tools` | AI SDK 主机执行工具（在应用进程中执行） |
| `skills` | 可复用指令包（按需加载） |
| `permissionMode` | 内置工具权限模式（`allow-all` / `allow-edits` / `allow-reads`） |
| `toolApproval` | 主机工具审批状态 |
| `sandboxConfig` | 沙箱工作目录和生命周期钩子 |
| `telemetry`、`debug`、`onLog` | 可观测性配置 |

---

## 3. 工具 (Tools)

Harness 有两层工具面：内置工具（由 Harness 运行时执行）和主机执行工具（在应用进程中执行）。`HarnessAgent` 将两者合并为统一的工具集，通过 `agent.tools` 暴露。

### 内置工具 (Built-in Tools)

每个 Adapter 声明其运行时原生可调用的工具。内置工具调用由 **Harness 运行时**执行（`providerExecuted: true`），不在应用进程中执行。流 part 中会标记 `providerExecuted: true` 表示运行时已经完成了调用。

```ts
agent.tools.bash;    // Shell 命令执行
agent.tools.read;    // 文件读取
agent.tools.write;   // 文件写入
agent.tools.edit;    // 文件编辑
agent.tools.grep;    // 内容搜索
agent.tools.glob;    // 文件名匹配
agent.tools.webSearch; // 网络搜索
```

Adapter 尽可能使用通用名称。某些运行时还会暴露不具备跨 Harness 通用名称的原生工具，这些以原生名称出现。

### 主机执行工具 (Host-Executed Tools)

主机工具在应用进程中执行，与 AI SDK 普通工具的定义方式完全相同。当 Harness 调用主机工具时，`HarnessAgent` 在宿主进程中执行并返回结果。

```ts
const weather = tool({
  description: '获取城市当前温度。',
  inputSchema: z.object({ city: z.string() }),
  execute: async ({ city }) => {
    const temperatures = { Paris: 12, Tokyo: 18, Reykjavik: 3 };
    return { city, celsius: temperatures[city] ?? 20 };
  },
});

const agent = new HarnessAgent({
  harness: claudeCode,
  sandbox: createVercelSandbox({ runtime: 'node24', ports: [4000] }),
  tools: { weather },
});
```

### 工具执行中访问沙箱

主机工具通过 `experimental_sandbox` 参数访问会话沙箱。该 sandbox 是受限的——工具可以读写文件和运行命令，但不能停止网络沙箱或更改其网络策略。

```ts
const inspectFile = tool({
  description: '从工作空间读取文件。',
  inputSchema: z.object({ path: z.string() }),
  execute: async ({ path }, { experimental_sandbox }) => {
    return { content: await experimental_sandbox?.readTextFile({ path }) };
  },
});
```

### 文件变更和压缩事件

某些 Harness 事件不是普通的工具调用。为了 UI 兼容性，`HarnessAgent` 将它们投射为动态的 provider-executed tool parts：

- **fileChange** — 工作空间文件变更（不透明文件变更事件）
- **compaction** — 运行时上下文压缩（当运行时压缩对话上下文时发出）

在 UI 中处理 tool parts 时，应首先检查 `part.dynamic` 标记。如果为 true，说明这是 Harness 特有事件而非类型化工具调用。

### 工具审批

**内置工具的权限控制**使用 `permissionMode`：

| 值 | 说明 |
|------|------|
| `allow-all` | 允许读取、编辑和 Shell 命令（默认） |
| `allow-edits` | 允许读取和编辑，Shell 命令需审批 |
| `allow-reads` | 只允许读取，编辑和 Shell 需审批 |

**主机工具的审批控制**使用 `toolApproval`：

```ts
toolApproval: { weather: 'user-approval' }
```

值：`not-applicable`（不适用）、`approved`（已批准）、`user-approval`（需用户审批）、`denied`（已拒绝）

当审批被要求时，流会在 `tool-approval-request` 之后暂停。在 UI 流程中，`useChat` 会在添加审批结果后自动发送审批响应消息。在直接 Agent 代码中，需要将审批响应作为 `messages` 传递给下一次 `stream()` 或 `generate()` 调用。

大多数 Adapter 支持暂停内置工具调用等待审批。不支持的 Adapter 会在指定了不支持的模式时报错。主机工具的审批由 `HarnessAgent` 处理，因此跨 Adapter 均可使用。

---

## 4. 技能 (Skills)

Skills 是可复用的指令包，适用于项目约定、工作流指南、领域特定流程或任何应由底层 Harness 运行时按需发现的指令。

### Skills vs Instructions

- **Skills**：按需可用，模型在需要时发现并加载，不会一直占用上下文窗口
- **Instructions**：通用 Agent 行为和当前会话优先级，始终在上下文中

### 定义 Skills

每个 Skill 包含以下字段：

| 字段 | 说明 |
|------|------|
| `name` | 稳定标识符 |
| `description` | 简短的模型可见摘要，用于判断何时加载该 Skill |
| `content` | 完整指令内容 |
| `files` | 可选附加文本文件（POSIX 相对路径），从 `content` 中引用 |

```ts
const agent = new HarnessAgent({
  harness: claudeCode,
  sandbox: createVercelSandbox({ runtime: 'node24', ports: [4000] }),
  skills: [
    {
      name: 'careful-refactors',
      description: '进行小规模、低风险的代码变更。',
      content: '优先最小 diff。保留公共 API。编辑前先读取 references/checklist.md。',
      files: [
        {
          path: 'references/checklist.md',
          content: '# 重构清单\n\n- 确定最小的有用变更。\n- 保留公共 API。\n- 运行最相关的测试。',
        },
      ],
    },
  ],
});
```

附加文件使用 Skill 相对路径，如 `reference.md`、`references/codes.md`、`templates/config.json`。在 `content` 中引用这些路径，Agent 就知道何时读取它们。

---

## 5. Harness Adapters

Harness Adapter 连接 `HarnessAgent` 到特定的 Agent 运行时。它们是 Harness 世界中相当于 AI SDK Model Provider 的存在：每个 Adapter 包装一个运行时，将其会话、流事件、工具、用量、生命周期状态和配置标准化为 Harness 契约。

### 可用 Adapter

| Adapter | 包 | 运行位置 | 自定义工具 | 自定义 Skills | 内置工具审批 |
|------|------|------|------|------|------|
| Claude Code | `@ai-sdk/harness-claude-code` | Sandbox bridge | ✓ | ✓ | ✓ |
| Codex | `@ai-sdk/harness-codex` | Sandbox bridge | ✓ | ✓ | ✓ |
| Deep Agents | `@ai-sdk/harness-deepagents` | Sandbox bridge | ✓ | ✓ | ✓ |
| OpenCode | `@ai-sdk/harness-opencode` | Sandbox bridge | ✓ | ✓ | ✓ |
| Pi | `@ai-sdk/harness-pi` | Host process | ✓ | ✓ | ✓ |

**即将推出：** Amp (`@ai-sdk/harness-amp`)、Goose (`@ai-sdk/harness-goose`)、Mastra (`@ai-sdk/harness-mastra`)

### 运行位置说明

- **Sandbox bridge**：Adapter 在沙箱内部运行桥接进程，通过网络端口与外部 HarnessAgent 通信（Claude Code、Codex、Deep Agents、OpenCode 均属此类）
- **Host process**：Adapter 直接在宿主机进程中运行，不需要网络沙箱端口（Pi 属此类）

---

## 6. 工作流工具 (Workflow Utilities)

`@ai-sdk/workflow-harness` 提供在 Workflow DevKit 工作流中运行 `HarnessAgent` 轮次的辅助工具。它将 Harness 轮次分解为可序列化的状态机和切片运行器。

### 核心流程

1. 定义 Agent（与普通 HarnessAgent 相同）
2. 创建切片步骤（`runHarnessAgentSlice`）——运行一个时间盒限制的 Harness 轮次片段
3. 创建工作流函数——循环调用切片步骤直到轮次完成
4. 从 HTTP 端点启动工作流

```ts
// 切片步骤
import { runHarnessAgentSlice, type HarnessWorkflowState } from '@ai-sdk/workflow-harness';

export async function runSlice(state: HarnessWorkflowState): Promise<HarnessWorkflowState> {
  'use step';
  const { agent } = await import('./agent');
  return runHarnessAgentSlice({ agent, state });
}
```

```ts
// 工作流
import { createHarnessWorkflowState, finalizeHarnessWorkflow } from '@ai-sdk/workflow-harness';

export async function codingWorkflow(input) {
  'use workflow';
  let state = createHarnessWorkflowState(input);
  while (state.status === 'running' || state.status === 'timed_out') {
    state = await runSlice(state);
  }
  return finalizeHarnessWorkflow(state);
}
```

工作流方式的价值在于：长时间运行的 Agent 任务可以被持久化和恢复，即使进程重启也不会丢失进度。适用于需要数小时甚至数天的编程任务。

---

## 7. UI 集成

Harness 流与 AI SDK UI 消息流兼容。可以在客户端使用 `useChat()`，在服务端路由中流式输出 `HarnessAgent` 结果。

### 与模型聊天路由的关键区别

Harness **拥有自己的对话状态**。服务端路由应该为每个 `chatId` 恢复或创建 `HarnessAgentSession`，而不是把完整的 UI 消息历史回放给模型。

### 客户端

客户端代码与普通 `useChat` 用法完全相同：

```tsx
const { messages, sendMessage, status } = useChat({
  id: 'example-chat',
  transport: new DefaultChatTransport({ api: '/api/chat' }),
});
```

### 会话存储

只需持久化 `session.detach()` 返回的不透明恢复状态。如果轮次因审批暂停或被中断，恢复状态内部包含继续状态。

```ts
const states: Record<string, HarnessAgentResumeSessionState | undefined> = {};

export async function resumeOrCreateSession({ agent, chatId }) {
  const resumeFrom = states[chatId];
  return agent.createSession(
    resumeFrom ? { sessionId: chatId, resumeFrom } : { sessionId: chatId },
  );
}

export async function detachAndPersist({ chatId, session }) {
  states[chatId] = await session.detach();
}
```

### 服务端路由

转换 UI 消息为模型消息，运行 Harness 轮次，将结果流转换回 UI 消息流：

```ts
export async function POST(request: Request) {
  const { id, messages } = await request.json();
  const chatId = id;
  const modelMessages = await convertToModelMessages(messages);
  const session = await resumeOrCreateSession({ agent, chatId });
  const result = await agent.stream({ session, messages: modelMessages });

  return createUIMessageStreamResponse({
    stream: toUIMessageStream({
      stream: result.stream,
      onEnd: async () => { await detachAndPersist({ chatId, session }); },
    }),
  });
}
```

### Detach vs Stop

- **`session.detach()`**：挂起运行时，保持沙箱热，桥接型 Adapter 通常能高效重新挂接或重放。适用于需要低延迟的连续请求。
- **`session.stop()`**：在每轮后保存恢复状态并停止运行时和沙箱。下一请求从持久化状态恢复，在接受新提示前继续未完成的轮次。

### 渲染 Harness Parts

Harness 输出包含与 AI SDK 模型流相同的 UI 消息 part 形状：
- `text` 和 `reasoning` parts — 生成内容
- 类型化工具 parts — `tool-bash`、`tool-read`、`tool-weather` 等
- `dynamic-tool` parts — 动态事件如 `fileChange` 和 `compaction`

### 类型安全的工具 Parts

在 HarnessAgent 会话选项成为基础 `Agent` 调用参数的一部分之前，可从 `agent.tools` 推断 UI 工具类型：

```ts
import type { InferUITools, UIMessage } from 'ai';
export type HarnessMessage = UIMessage<unknown, never, InferUITools<typeof agent.tools>>;
```

---

## 8. 终端 UI (Terminal UI)

`@ai-sdk/tui` 可以在终端中渲染 Harness 流、工具调用、推理部分和审批提示。因为 `HarnessAgent` 每次调用都需要 `session`，需要用一个小型 `AgentTUIAgent` 适配器包装它，为终端 UI 的生命周期注入一个会话。

```ts
import { runAgentTUI, type AgentTUIAgent } from '@ai-sdk/tui';

function createTUIAgent({ agent, session }): AgentTUIAgent {
  return {
    version: 'agent-v1',
    id: agent.id,
    tools: agent.tools,
    generate(request) {
      return agent.generate({ ...request, session });
    },
    stream(request) {
      return agent.stream({ ...request, session });
    },
  } as AgentTUIAgent;
}

const session = await agent.createSession();
try {
  await runAgentTUI({
    title: 'Codex',
    agent: createTUIAgent({ agent, session }),
    tools: 'auto-collapsed',    // 自动折叠工具调用
    reasoning: 'collapsed',     // 折叠推理过程
  });
} finally {
  await session.destroy();
}
```

终端 UI 运行直到用户按 `Esc` 或 `Ctrl+C` 退出。每个终端运行使用一个会话。对于长期运行的终端工具，如果以后需要恢复，可以持久化 `session.detach()` 或 `session.stop()` 的状态。

---

## 常用 API 引用速览

| API | 类别 | 说明 |
|------|------|------|
| `HarnessAgent` | Agent | Harness Agent 核心类 |
| `HarnessAgent.createSession()` | 会话 | 创建新会话或恢复已有会话 |
| `HarnessAgent.generate()` | 执行 | 执行轮次并等待完整结果 |
| `HarnessAgent.stream()` | 执行 | 执行轮次并流式返回增量输出 |
| `session.destroy()` | 生命周期 | 停止运行时，丢弃可恢复性 |
| `session.detach()` | 生命周期 | 挂起运行时，返回恢复状态 |
| `session.stop()` | 生命周期 | 保存恢复状态后停止运行时 |
| `session.suspendTurn()` | 生命周期 | 跨进程边界挂起活跃轮次 |
| `prepareHarnessSandboxTemplate()` | 沙箱 | 预创建或刷新可复用沙箱模板 |
| `prepareSandboxForHarness()` | 沙箱 | 准备沙箱但不停止或快照 |
| `runAgentTUI()` | TUI | 在终端中运行 Agent 交互界面 |
| `runHarnessAgentSlice()` | 工作流 | 运行一个工作流时间片 |
| `claudeCode` | Adapter | Claude Code Adapter 实例 |
| `codex` | Adapter | Codex Adapter 实例 |
| `pi` | Adapter | Pi Adapter 实例 |
