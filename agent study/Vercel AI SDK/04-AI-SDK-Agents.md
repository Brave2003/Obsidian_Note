# AI SDK Agents

> AI SDK Agents 提供了构建 AI Agent 的完整框架——从简单的工具循环到持久化的生产级 Agent。它封装了模型调用、工具执行、循环控制、记忆管理等能力，让你用最少的代码搭建最强大的 Agent。

- **官网**: https://ai-sdk.dev/docs/agents
- **核心类**: `ToolLoopAgent`、`WorkflowAgent`
- **配套工具**: `runAgentTUI` (终端界面)、DevTools (调试工具)

---

## 1. 快速开始：在编码 Agent 中使用 AI SDK

这一节针对**已经在用编码 Agent**（如 Claude Code、Codex、OpenCode、Cursor 等）的开发者，教你如何让编码 Agent 更好地使用 AI SDK。

### 安装 AI SDK Skill

最快让编码 Agent 深度理解 AI SDK 的方法是安装官方 Skill。Skill 是轻量级 Markdown 文件，在需要时按需加载专业指令到 Agent 上下文中——Agent 知道如何正确使用 SDK，不需要你手动解释。

```bash
npx skills add vercel/ai
```

安装后，Skill 放入 Agent 的特定 skills 目录（如 `.claude/skills`、`.codex/skills`）。如果选择了多个 Agent，CLI 会创建符号链接让每个 Agent 都能发现。用 `-a` 指定 Agent（如 `-a amp` 安装到通用 `.agents/skills` 目录），用 `-y` 跳过交互确认。

**渐进式披露机制**：Agent 启动时只加载 Skill 的名称和描述。完整指令只在任务真正需要时才拉入上下文，保持 Agent 快速和专注。

### 利用 `node_modules` 中的文档和源码

安装 `ai` 包后，完整的 AI SDK 文档和源码已经在本地 `node_modules` 中了：

```
node_modules/ai/src/     # 按模块组织的完整源码
node_modules/ai/docs/    # 官方文档和示例
```

编码 Agent 可以直接读取这些文件——不需要联网。Agent 从已安装的包中查找准确的 API 签名、实现和用法示例，确保使用的是项目实际安装的 SDK 版本。

### 安装 DevTools

AI SDK DevTools 让你在开发过程中完全可视化 AI SDK 调用。它捕获 LLM 请求、响应、工具调用、token 用量和多步交互，在本地 Web UI 中展示（仅本地开发，不要用于生产）。

```bash
npm install @ai-sdk/devtools
```

注册全局遥测：

```ts
import { registerTelemetry } from 'ai';
import { DevToolsTelemetry } from '@ai-sdk/devtools';
registerTelemetry(DevToolsTelemetry());
```

启动查看器：

```bash
npx @ai-sdk/devtools
```

打开 `http://localhost:4983` 实时检查 AI SDK 交互。DevTools 将每次调用分组为 **runs**（完整交互）和 **steps**（交互中的每次 LLM 调用），追踪 Agent 做了什么以及为什么这样做。数据存储在 `.devtools/generations.json`，自动添加到 `.gitignore`。

也可以在代码中直接打印工具结果快速调试：

```ts
const result = streamText({
  model,
  prompt: "What's the weather in New York?",
  tools: { weather: weatherTool },
  stopWhen: isStepCount(5),
  onStepEnd: async ({ toolResults }) => {
    if (toolResults.length) {
      console.log(JSON.stringify(toolResults, null, 2));
    }
  },
});
```

---

## 2. 构建 Agent (Building Agents)

`ToolLoopAgent` 提供了一种结构化的方式来封装 LLM 配置、工具和行为到可复用组件中。它自动处理 Agent 循环——LLM 可以多次调用工具来完成复杂任务，你只需定义 Agent 一次，在应用各处使用。

### 为什么用 ToolLoopAgent？

构建 AI 应用时，你经常需要：
- **复用配置**：相同的模型设置、工具和提示词在应用的不同部分使用
- **保持一致**：确保整个代码库中行为一致
- **简化 API 路由**：减少端点中的样板代码
- **类型安全**：获得工具的完整 TypeScript 支持

`ToolLoopAgent` 提供了一个集中的地方定义 Agent 的行为。

### 创建 Agent

```ts
import { ToolLoopAgent, tool } from 'ai';
import { z } from 'zod';

const myAgent = new ToolLoopAgent({
  model: __MODEL__,
  instructions: 'You are a helpful assistant.',
  tools: {
    runCode: tool({
      description: 'Execute Python code',
      inputSchema: z.object({ code: z.string() }),
      execute: async ({ code }) => ({ output: 'Code executed successfully' }),
    }),
  },
});
```

`ToolLoopAgent` 接受 `generateText` 和 `streamText` 的所有设置。可配置模型、系统指令、工具、循环控制等。

### 上下文和 Agent 状态

`runtimeContext` 是 Agent 的共享运行时状态——它贯穿整个 Agent 循环，在 `prepareStep`、生命周期回调和最终结果中都可用。`toolsContext` 用于传递服务端值（如凭据、权限、默认设置），需要通过工具的 `contextSchema` 声明：

```ts
const agent = new ToolLoopAgent({
  model: __MODEL__,
  tools: {
    searchTickets: tool({
      description: 'Search support tickets',
      inputSchema: z.object({ query: z.string() }),
      contextSchema: z.object({ apiKey: z.string(), accountId: z.string() }),
      execute: async ({ query }, { context }) =>
        searchTickets(query, context.accountId, context.apiKey),
    }),
  },
  prepareStep: async ({ runtimeContext }) => {
    if (runtimeContext.escalated) return { temperature: 0.1 };
    return {};
  },
});

const result = await agent.generate({
  prompt: 'Find open billing tickets.',
  runtimeContext: { requestId: 'req_abc', escalated: false },
  toolsContext: { searchTickets: { apiKey: 'xxx', accountId: 'acct_123' } },
});
```

### 工具审批

有 `execute` 函数的工具默认自动运行。通过 `toolApproval` 在 `ToolLoopAgent` 上配置审批：

```ts
const agent = new ToolLoopAgent({
  model: __MODEL__,
  tools: { runCode: runCodeTool },
  toolApproval: { runCode: 'user-approval' },
});
```

---

## 3. 循环控制 (Loop Control)

Agent 循环会持续执行直到满足停止条件。默认最大 20 步（`isStepCount(20)`）——这是防止失控循环导致过度 API 调用的安全措施。

### 内置停止条件

| 条件 | 说明 |
|------|------|
| `isStepCount(n)` | 达到 n 步后停止 |
| `hasToolCall(...names)` | 调用了指定工具后停止 |
| `isLoopFinished()` | 永不触发，让 LLM 自然结束 |

```ts
import { ToolLoopAgent, isStepCount, hasToolCall } from 'ai';

const agent = new ToolLoopAgent({
  model: __MODEL__,
  tools: { /* ... */ },
  // 多条件组合：任一满足即停止
  stopWhen: [
    isStepCount(20),
    hasToolCall('someTool', 'done'),
  ],
});
```

### 自定义停止条件

基于步数历史数据构建自己的条件：

```ts
const budgetExceeded: StopCondition<typeof tools> = ({ steps }) => {
  const totalUsage = steps.reduce(
    (acc, step) => ({
      inputTokens: acc.inputTokens + (step.usage?.inputTokens ?? 0),
      outputTokens: acc.outputTokens + (step.usage?.outputTokens ?? 0),
    }),
    { inputTokens: 0, outputTokens: 0 },
  );
  const costEstimate = (totalUsage.inputTokens * 0.01 + totalUsage.outputTokens * 0.03) / 1000;
  return costEstimate > 0.5; // 超过 $0.50 停止
};
```

### Prepare Step — 动态调整每一步

`prepareStep` 在循环的每一步之前运行。可以动态切换模型、管理上下文、或基于执行历史调整行为：

```ts
const agent = new ToolLoopAgent({
  model: 'openai/gpt-4o-mini',
  tools: { /* ... */ },
  prepareStep: async ({ stepNumber, messages }) => {
    if (stepNumber > 2 && messages.length > 10) {
      return { model: 'openai/o1-mini' }; // 复杂步骤换更强模型
    }
    return {};
  },
});
```

`prepareStep` 还可以用于上下文压缩——当消息太多时，返回一个修剪过的 `messages` 数组来减少 token 消耗。

---

## 4. 配置调用选项 (Configuring Call Options)

Call Options 让你在每次调用 Agent 时传入类型安全的动态参数，实现"一个 Agent，多种行为"。

### 经典三步模式

1. 用 `callOptionsSchema` 定义接受什么参数
2. 在 `prepareCall` 中用这些参数修改 Agent 设置
3. 调用 `generate()` 或 `stream()` 时传入 `options`

```ts
const supportAgent = new ToolLoopAgent({
  model: __MODEL__,
  callOptionsSchema: z.object({
    userId: z.string(),
    accountType: z.enum(['free', 'pro', 'enterprise']),
  }),
  instructions: 'You are a helpful customer support agent.',
  prepareCall: ({ options, ...settings }) => ({
    ...settings,
    instructions: settings.instructions +
      `\nUser: ${options.userId}, Account: ${options.accountType}`,
  }),
});

await supportAgent.generate({
  prompt: 'How do I upgrade?',
  options: { userId: 'user_123', accountType: 'free' },
});
```

`options` 参数在定义 schema 后变为**必填且类型检查**——少传或类型错误，TypeScript 会报错。

### 典型应用：动态模型选择

```ts
prepareCall: ({ options, ...settings }) => ({
  ...settings,
  model: options.complexity === 'simple' ? 'openai/gpt-4o-mini' : 'openai/o1-mini',
}),
```

### 典型应用：RAG 注入

```ts
prepareCall: async ({ options, ...settings }) => {
  const documents = await vectorSearch(options.query);
  return {
    ...settings,
    instructions: `Answer using: ${documents.map(d => d.content).join('\n')}`,
  };
},
```

`prepareCall` 支持 async，可以在配置 Agent 前获取数据。

---

## 5. 记忆 (Memory)

记忆让 Agent 保存信息并在之后回忆。没有记忆，每次对话从零开始。有记忆，Agent 能构建跨时间的上下文。

### 三种实现方式

| 方式 | 工作量 | 灵活性 | 供应商锁定 |
|------|--------|--------|------------|
| 供应商定义工具 | 低 | 中 | 是 |
| 记忆服务商 | 低 | 低 | 取决于服务商 |
| 自定义工具 | 高 | 高 | 否 |

### Anthropic Memory Tool

Anthropic 提供了结构化的记忆工具，Claude 会管理一个 `/memories` 目录——在开始任务前读取记忆，工作中创建和更新文件，在未来的对话中引用它们。收到结构化命令：`view`、`create`、`str_replace`、`insert`、`delete`、`rename`，路径限定在 `/memories` 内。你的 `execute` 函数映射这些命令到存储后端。

```ts
const memory = anthropic.tools.memory_20250818({
  execute: async action => {
    // 处理 view, create, str_replace, insert, delete, rename 命令
  },
});
const agent = new ToolLoopAgent({
  model: 'anthropic/claude-haiku-4.5',
  tools: { memory },
});
```

优点：实现工作量最小，模型专门训练过使用此工具。缺点：仅限 Claude。

### 记忆服务商

**Letta**：持久化长期记忆平台。在 Letta 创建 Agent 并配置记忆，通过 AI SDK provider 交互。Letta 运行时代管记忆管理（核心记忆、归档记忆、回忆）。

**Mem0**：在任何 LLM provider 上叠加记忆层。自动从对话提取记忆、存储、在后续提示中检索相关内容。支持 OpenAI、Anthropic、Google、Groq、Cohere 等多个 provider。

```ts
const mem0 = createMem0({ provider: 'openai', mem0ApiKey: 'xxx', apiKey: 'xxx' });
const agent = new ToolLoopAgent({ model: mem0('gpt-4.1', { user_id: 'user-123' }) });
```

**Supermemory**：持久化自增长记忆平台，通过语义搜索自动保存和检索记忆。提供 `addMemory` 和 `searchMemories` 操作，与任何 AI SDK provider 配合。

**Hindsight**：五个工具实现持久记忆——`retain`、`recall`、`reflect`、`getMentalModel`、`getDocument`。支持 Docker 自托管或云服务。`bankId` 标识记忆存储（通常是用户 ID）。

**MongoDB**：`@mongodb-developer/vercel-ai-memory` 提供 MongoDB Atlas 持久记忆，五个结构化层级：Session、Semantic、Procedural、Episodic、Scratchpad。使用 Atlas Vector Search + 任意嵌入模型进行检索，自动创建索引、支持按类型设置保留策略。

---

## 6. 子 Agent (Subagents)

子 Agent 是被父 Agent 调用的独立 Agent。父 Agent 通过工具委派任务，子 Agent 自主执行后返回结果。

### 执行机制

1. 定义子 Agent，配有自己的模型、指令和工具
2. 创建工具让主 Agent 调用
3. 子 Agent 在自己独立的上下文窗口中运行
4. 返回结果（可选：流式显示进度）
5. 通过 `toModelOutput` 控制主 Agent 看到的内容

### 何时使用

| 该用子 Agent | 不用子 Agent |
|------|------|
| 任务需要探索大量 token | 任务简单且专注 |
| 需要并行化独立研究 | 顺序处理足够 |
| 上下文会超出模型限制 | 上下文保持可控 |
| 想按能力隔离工具访问 | 所有工具可安全共存 |

### 核心价值：卸载上下文重任务

某些任务需要读取大量信息——文件、代码库搜索、研究话题。在主 Agent 中运行这些会快速消耗上下文，让 Agent 变得越来越不连贯。子 Agent 可以：消耗数十万 token 做深入研究 → 只返回约 1000 token 的精简摘要 → 让主 Agent 的上下文保持干净。

### 基础用法（无流式）

```ts
const researchSubagent = new ToolLoopAgent({
  model: __MODEL__,
  instructions: 'You are a research agent. Summarize findings.',
  tools: { read: readFileTool, search: searchTool },
});

const researchTool = tool({
  description: 'Research a topic in depth.',
  inputSchema: z.object({ task: z.string() }),
  execute: async ({ task }, { abortSignal }) => {
    const result = await researchSubagent.generate({ prompt: task, abortSignal });
    return result.text;
  },
});

const mainAgent = new ToolLoopAgent({
  model: __MODEL__,
  tools: { research: researchTool },
});
```

务必传递 `abortSignal` 给子 Agent——用户取消请求时子 Agent 会同步停止。

### 流式子 Agent 进度

要让 UI 显示子 Agent 的渐进进度，将 `execute` 从普通函数改为 **async generator**（`async function*`）。每次 `yield` 向 UI 发送一个 preliminary result。使用 `readUIMessageStream` 工具累积子 Agent 的流式响应，构建逐渐增长的完整消息。

```ts
const researchTool = tool({
  description: 'Research a topic in depth.',
  inputSchema: z.object({ task: z.string() }),
  execute: async function* ({ task }, { abortSignal }) {
    const subStream = await researchSubagent.stream({ prompt: task, abortSignal });
    const uiStream = toUIMessageStream({ stream: subStream.stream });

    for await (const message of readUIMessageStream({ stream: uiStream })) {
      yield { content: message.parts }; // 每个 yield 替换上一次输出
    }
  },
});
```

---

## 7. 工具审批 (Tool Approvals)

默认情况下，有 `execute` 函数的工具自动运行。`toolApproval` 让你在工具执行前审查、批准或拒绝。

### 四种审批状态

| 状态 | 含义 |
|------|------|
| `not-applicable` | 正常执行，无审批元数据（默认） |
| `approved` | 记录自动批准，然后执行 |
| `denied` | 记录自动拒绝，返回拒绝输出 |
| `user-approval` | 发出审批请求，等待显式响应 |

自动批准/拒绝可附带原因：`{ type: 'denied', reason: '此工作区禁用文件删除' }`

### 按工具配置

```ts
toolApproval: {
  runCommand: 'user-approval',  // 总是需要用户确认
}
```

### 基于输入决策

```ts
toolApproval: {
  processPayment: async ({ amount }, { runtimeContext }) => {
    if (runtimeContext.role !== 'admin')
      return { type: 'denied', reason: '仅管理员可付款' };
    return amount > 1000 ? 'user-approval' : undefined; // 大额需审批
  },
},
```

### 通用审批函数（所有工具）

```ts
toolApproval: ({ toolCall }) => {
  if (toolCall.dynamic) return 'user-approval';
  if (toolCall.toolName === 'deleteFile') return 'user-approval';
  return undefined;
},
```

### 按请求配置

因为 `toolApproval` 是 Agent 设置，可以从 `prepareCall` 中返回，实现基于调用选项的动态审批策略：

```ts
prepareCall: ({ options, ...settings }) => ({
  ...settings,
  toolApproval: {
    runCommand: options.canRunCommands ? 'user-approval'
      : { type: 'denied', reason: '命令访问已禁用' },
  },
}),
```

### 人工审批流程

需要两次调用：第一次发出审批请求 → 收集用户决策 → 添加 `tool-approval-response` 到消息 → 第二次调用继续。在 UI 中 `useChat` 会自动处理这个流程。

---

## 8. WorkflowAgent — 持久化 Agent

`WorkflowAgent`（`@ai-sdk/workflow`）在 Workflow 内部运行，提供与 `ToolLoopAgent` 相同的 Agent 循环，但增加了自动状态持久化、工具 schema 序列化和跨步骤存活的审批流程。

### 为什么需要持久化？

标准 `ToolLoopAgent` 全在内存中运行——进程崩溃，所有进度丢失：

- **有状态**：长时间 Agent 循环需要跨进程边界持久化状态
- **可恢复**：步骤失败时从上次检查点重试，而非从头开始
- **人机协作**：需要用户审批的工具要能暂停并在之后恢复
- **可观测**：每次工具调用作为独立的工作流步骤，在仪表板中可见

### ToolLoopAgent vs WorkflowAgent

| 维度 | ToolLoopAgent | WorkflowAgent |
|------|------|------|
| 包 | `ai` | `@ai-sdk/workflow` |
| 运行时 | 内存中 | Workflow |
| 持久性 | 崩溃丢失 | 重启存活 |
| 工具重试 | 手动 | 自动（工作流步骤） |
| 人工审批 | 内置 | 内置 + 暂停后存活 |
| `generate()` | 可用 | 不可用 |
| `stream()` | 可用 | 主要 API |
| 流输出 | `streamText` 返回值 | `writable` 参数 + `ModelCallStreamPart` |

### 创建工作流 Agent

```ts
import { WorkflowAgent, type ModelCallStreamPart } from '@ai-sdk/workflow';
import { convertToModelMessages, tool, type UIMessage } from 'ai';
import { getWritable } from 'workflow';

export async function chat(messages: UIMessage[]) {
  'use workflow';
  const modelMessages = await convertToModelMessages(messages);
  const agent = new WorkflowAgent({
    model: 'anthropic/claude-sonnet-4-6',
    instructions: 'You are a flight booking assistant.',
    tools: {
      searchFlights: tool({ /* ... */ }),
      bookFlight: tool({ /* ... */ }),
    },
  });

  const result = await agent.stream({
    messages: modelMessages,
    writable: getWritable<ModelCallStreamPart>(),
  });
  return { messages: result.messages };
}
```

关键集成点：
1. 函数标记 `'use workflow'`
2. `getWritable()` 传入 `stream()` 方法
3. 从 API 路由用 `start()` 启动工作流

API 路由侧使用 `createModelCallToUIChunkTransform()` 将原始 `ModelCallStreamPart` 转换为 `UIMessageChunk`：

```ts
import { createModelCallToUIChunkTransform } from '@ai-sdk/workflow';
import { createUIMessageStreamResponse } from 'ai';
import { start } from 'workflow/api';

export async function POST(request: Request) {
  const { messages } = await request.json();
  const run = await start(chat, [messages]);
  return createUIMessageStreamResponse({
    stream: run.readable.pipeThrough(createModelCallToUIChunkTransform()),
    headers: { 'x-workflow-run-id': run.runId },
  });
}
```

### 可恢复流与 WorkflowChatTransport

Workflow 函数可能超时或被网络中断。`WorkflowChatTransport` 自动检测流在无 `finish` 事件时结束，并重连从断点恢复：

```tsx
const transport = new WorkflowChatTransport({
  api: '/api/chat',
  maxConsecutiveErrors: 5,
  initialStartIndex: -50, // 刷新时获取最近 50 个 chunk
});
const { messages, sendMessage } = useChat({ transport });
```

---

## 9. 终端 UI (Terminal UI)

`@ai-sdk/tui` 让你在交互式终端中运行 `ToolLoopAgent`。适用于本地开发、演示和内部工具——不想构建自定义 UI 时终端体验就够了。

### 安装与运行

```bash
pnpm add @ai-sdk/tui ai @ai-sdk/openai
```

```ts
import { runAgentTUI } from '@ai-sdk/tui';
import { ToolLoopAgent, tool } from 'ai';

const agent = new ToolLoopAgent({
  model: openai('gpt-5'),
  instructions: 'You are a helpful terminal assistant.',
  tools: { weather: weatherTool },
});

await runAgentTUI({
  title: 'Weather Agent',
  agent,
  tools: 'auto-collapsed',
  reasoning: 'collapsed',
  responseStatistics: 'outputTokensPerSecond',
  contextSize: 200_000,
});
```

### 显示选项

| 设置 | 可选值 | 默认值 | 说明 |
|------|--------|--------|------|
| `tools` | full / collapsed / auto-collapsed / hidden | auto-collapsed | 工具调用渲染方式 |
| `reasoning` | full / collapsed / auto-collapsed / hidden | auto-collapsed | 推理部分渲染方式 |
| `responseStatistics` | outputTokensPerSecond / outputTokenCount | outputTokensPerSecond | 响应统计类型 |
| `contextSize` | 数字 | - | 模型上下文窗口大小（用于显示用量百分比） |

### 沙箱支持

当 Agent 工具需要执行环境时，传入沙箱会话：

```ts
const sandboxSession = await createJustBashSandbox({ cwd: '/home/user' }).createSession();
await runAgentTUI({ title: 'Sandbox Agent', agent, sandbox: sandboxSession.restricted() });
```

TUI 会将沙箱作为 `experimental_sandbox` 传递给每次 Agent 调用。

### 键盘控制

- `Enter` — 提交提示词
- `y`/`n` — 批准或拒绝工具调用
- `Up`/`Down` — 滚动对话记录
- `PageUp`/`PageDown` — 整页滚动
- `Ctrl+L` — 重绘屏幕
- `Esc`/`Ctrl+C` — 退出

`runAgentTUI` 支持 `ToolLoopAgent` 的工具审批流程——需要审批时会提示用户决定。

### 兼容性

TUI 适用于可直接从终端用户输入运行的 Agent。Agent 不能要求每次调用的 options，也不能使用结构化输出——因为 TUI 无法从自由格式提示词推断这些值。如需要固定提示词、call options、结构化输出或自定义流处理，直接使用 `agent.generate()` 或 `agent.stream()`。

---

## 常用 API 引用速览

| API | 类别 | 说明 |
|------|------|------|
| `ToolLoopAgent` | Agent 类 | 标准 Agent 循环实现 |
| `WorkflowAgent` | Agent 类 | 持久化、可恢复的 Workflow Agent |
| `isStepCount(n)` | 循环控制 | n 步后停止 |
| `hasToolCall(...names)` | 循环控制 | 调用指定工具后停止 |
| `isLoopFinished()` | 循环控制 | 永不触发，让 LLM 自然结束 |
| `StopCondition` | 类型 | 自定义停止条件类型 |
| `runAgentTUI()` | TUI | 在终端中运行 Agent 交互界面 |
| `prepareStep` | 回调 | 每步前动态调整设置 |
| `prepareCall` | 回调 | 调用前根据 options 调整设置 |
| `callOptionsSchema` | 配置 | 定义调用时传入参数的类型 |
| `toolApproval` | 审批 | 工具执行前的审批策略 |
| `runtimeContext` | 上下文 | Agent 循环共享状态 |
| `toolsContext` | 上下文 | 按工具索引的上下文值 |
| `contextSchema` | Schema | 工具声明需要的 context |
| `experimental_sandbox` | 沙箱 | 工具执行时的沙箱环境 |
| `readUIMessageStream` | 流工具 | 读取 UI 消息流（子 Agent 流式所用） |
| `toUIMessageStream` | 流工具 | 模型流转换到 UI 消息流 |
| `convertToModelMessages` | 转换 | UI 消息转模型消息 |
| `createModelCallToUIChunkTransform` | 转换 | 原始模型流转换到 UI chunk |
| `WorkflowChatTransport` | 传输层 | Workflow 专用传输（自动重连） |
| `registerTelemetry` | 遥测 | 注册全局遥测 |
| `DevToolsTelemetry` | 遥测 | DevTools 遥测集成 |
