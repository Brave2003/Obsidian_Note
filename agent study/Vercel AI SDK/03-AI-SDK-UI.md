# AI SDK UI

> AI SDK UI 是框架无关的工具包，用于构建交互式聊天、补全和 AI 助手应用。它简化了前端聊天流管理和 UI 更新的复杂性，提供实时流式消息、状态管理和工具调用渲染能力。

- **官网**: https://ai-sdk.dev/docs/ai-sdk-ui
- **支持框架**: React (`@ai-sdk/react`)、Vue.js (`@ai-sdk/vue`)、Svelte (`@ai-sdk/svelte`)、Angular (`@ai-sdk/angular`)、SolidJS (社区)

---

## 1. 概述 (Overview)

### 三大核心 Hook

AI SDK UI 提供三个主要 Hook，覆盖最常见的 AI 交互模式：

| Hook | 用途 | 关键返回值 |
|------|------|------|
| `useChat` | 实时流式聊天消息，管理完整对话状态 | `messages`, `sendMessage`, `status`, `stop`, `regenerate`, `addToolOutput` |
| `useCompletion` | 文本补全，单次提示词获取补全结果 | `completion`, `complete`, `status`, `stop` |
| `useObject` | 消费流式 JSON 对象 | `object`, `submit`, `status`, `stop` |

### 框架支持矩阵

| 功能 | React | Vue.js | Svelte | Angular | SolidJS |
|------|-------|--------|--------|---------|---------|
| `useChat` | ✓ | ✓ | Chat | Chat | 社区 |
| `useCompletion` | ✓ | ✓ | Completion | Completion | 社区 |
| `useObject` | ✓ | ✓ | StructuredObject | StructuredObject | 社区 |
| MCP Apps | ✓ | ✓ | - | - | 社区 |

### 核心设计理念

AI SDK UI 的核心设计围绕 **UI Message Stream** 协议展开。服务端通过 Server-Sent Events (SSE) 或 HTTP Streaming 将结构化的消息事件流式传输到前端，前端 Hook 自动管理状态更新、消息累积和 UI 渲染。消息使用 `parts` 数组而非简单的 `content` 字符串，每种 part 类型（文本、工具调用、工具结果、推理等）都有对应的处理方式。

---

## 2. 聊天机器人 (Chatbot)

`useChat` 是实现聊天 UI 的核心 Hook，提供以下关键能力：

- **消息流式传输**：AI 回复实时流式显示，用户无需等待完整响应
- **状态自动管理**：管理输入、消息列表、加载状态、错误状态等
- **无缝集成**：通过 `transport` 配置轻松接入任何后端

### 基础示例

**前端页面：**

`useChat` 通过 `DefaultChatTransport` 配置 API 端点。消息使用 `parts` 属性渲染，而非 `content` 属性——`parts` 支持文本、工具调用、工具结果等多种消息类型，允许更灵活的聊天 UI。

```tsx
'use client';
import { useChat } from '@ai-sdk/react';
import { DefaultChatTransport } from 'ai';
import { useState } from 'react';

export default function Page() {
  const { messages, sendMessage, status } = useChat({
    transport: new DefaultChatTransport({ api: '/api/chat' }),
  });
  const [input, setInput] = useState('');

  return (
    <>
      {messages.map(message => (
        <div key={message.id}>
          {message.role === 'user' ? 'User: ' : 'AI: '}
          {message.parts.map((part, index) =>
            part.type === 'text' ? <span key={index}>{part.text}</span> : null,
          )}
        </div>
      ))}
      <form onSubmit={e => {
        e.preventDefault();
        if (input.trim()) { sendMessage({ text: input }); setInput(''); }
      }}>
        <input value={input} onChange={e => setInput(e.target.value)}
          disabled={status !== 'ready'} placeholder="输入内容..." />
        <button type="submit" disabled={status !== 'ready'}>发送</button>
      </form>
    </>
  );
}
```

**服务端 API 路由：**

服务端使用 `streamText` 生成响应，通过 `convertToModelMessages` 将 UI 消息格式转换为模型消息格式，最后用 `toUIMessageStream` 和 `createUIMessageStreamResponse` 包装为标准的 UI Message Stream HTTP 响应。

```ts
import { convertToModelMessages, createUIMessageStreamResponse, streamText, toUIMessageStream } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();
  const result = streamText({
    model: __MODEL__,
    instructions: '你是一个乐于助人的助手。',
    messages: await convertToModelMessages(messages),
  });
  return createUIMessageStreamResponse({
    stream: toUIMessageStream({ stream: result.stream }),
  });
}
```

### Status 状态机

`status` 反映聊天请求的四个阶段：

| 状态 | 含义 | UI 使用建议 |
|------|------|------|
| `submitted` | 消息已发送，等待响应流开始 | 显示加载指示器 |
| `streaming` | 响应正在流式返回 | 显示"停止"按钮 + 实时内容 |
| `ready` | 完整响应已接收，可发送新消息 | 启用输入框和发送按钮 |
| `error` | API 请求出错 | 显示错误信息和重试按钮 |

```tsx
{(status === 'submitted' || status === 'streaming') && (
  <div>
    {status === 'submitted' && <Spinner />}
    <button onClick={() => stop()}>停止</button>
  </div>
)}
```

### 错误处理

`error` 状态反映 fetch 请求中抛出的错误对象。建议向用户显示通用错误消息（如"出了点问题"），避免泄露服务端信息。可通过 `regenerate()` 重试最后一条消息。

```tsx
const { error, regenerate } = useChat({ /* ... */ });
{error && <>
  <div>发生错误。</div>
  <button onClick={() => regenerate()}>重试</button>
</>}
```

也可以自定义提交处理：当存在错误时，先移除最后一条消息再重新发送。

### 消息操作

- **删除消息：** `setMessages(messages.filter(m => m.id !== id))` — 直接从消息列表中移除
- **停止响应：** `stop()` — 中止正在进行的流式响应，避免不必要的资源消耗
- **重新生成：** `regenerate()` — 重新生成最后一条助手消息

### 节流 UI 更新（仅 React）

流式响应的默认行为是每个 chunk 触发一次渲染。在大流量的场景下，可以使用 `experimental_throttle` 减少渲染频率：

```tsx
const { messages } = useChat({
  experimental_throttle: 50, // 50ms 更新间隔
});
```

### 事件回调

`useChat` 提供多个生命周期回调，用于日志记录、分析或自定义 UI 更新：

```tsx
const { /* ... */ } = useChat({
  onFinish: ({ message, messages, isAbort, isDisconnect, isError }) => {
    // 响应完成时调用。isAbort: 是否用户主动中止
    // isDisconnect: 是否连接断开; isError: 是否发生错误
  },
  onError: error => {
    // 发生错误时调用，可用于日志上报
  },
  onData: data => {
    // 收到自定义数据部分时调用
  },
});
```

> 在 `onData` 中抛出错误可中止处理，触发 `onError` 回调。

---

## 3. 请求配置

`useChat` 通过 Transport 层提供灵活的请求配置能力。

### Hook 级别配置（应用于所有请求）

Hook 构造时的配置会影响所有的消息发送：

```tsx
const { messages, sendMessage } = useChat({
  transport: new DefaultChatTransport({
    api: '/api/custom-chat',
    headers: { Authorization: 'your_token' },
    body: { user_id: '123' },
    credentials: 'same-origin',
  }),
});
```

### 动态 Hook 级别配置

当配置值需要在运行时确定时（如刷新 token），可使用函数形式：

```tsx
headers: () => ({ Authorization: `Bearer ${getAuthToken()}` }),
body: () => ({ sessionId: getCurrentSessionId() }),
```

### 请求级别配置（单次请求，推荐方式）

每次调用 `sendMessage` 时可以覆盖配置，优先级高于 Hook 级别：

```tsx
sendMessage(
  { text: input },
  {
    headers: { Authorization: 'Bearer token123' },
    body: { temperature: 0.7, max_tokens: 100 },
    metadata: { userId: 'user123', sessionId: 'session456' },
  },
);
```

### 请求转换 (Request Transformation)

通过 `prepareSendMessagesRequest` 可以在请求发送前完全控制请求体，如限制发送的消息数量：

```tsx
transport: new DefaultChatTransport({
  prepareSendMessagesRequest: ({ id, messages, trigger, messageId }) => {
    return {
      headers: { 'X-Session-ID': id },
      body: { messages: messages.slice(-10) }, // 只发送最近10条
    };
  },
}),
```

---

## 4. 消息元数据 (Message Metadata)

消息元数据允许在消息级别附加自定义信息，如时间戳、模型信息、token 用量等。与 data parts（消息内容的一部分）不同，元数据是关于消息本身的信息。

### 定义元数据类型

```tsx
import { UIMessage } from 'ai';
import { z } from 'zod';

export const messageMetadataSchema = z.object({
  createdAt: z.number().optional(),
  model: z.string().optional(),
  totalTokens: z.number().optional(),
});

export type MyUIMessage = UIMessage<z.infer<typeof messageMetadataSchema>>;
```

### 服务端发送元数据

使用 `toUIMessageStream` 的 `messageMetadata` 回调在不同阶段发送不同的元数据：

```ts
return createUIMessageStreamResponse({
  stream: toUIMessageStream({
    stream: result.stream,
    originalMessages: messages,
    messageMetadata: ({ part }) => {
      if (part.type === 'start') {
        return { createdAt: Date.now(), model: 'gpt-5-mini' };
      }
      if (part.type === 'finish') {
        return { totalTokens: part.totalUsage.totalTokens };
      }
    },
  }),
});
```

### 客户端访问元数据

通过类型参数 `<MyUIMessage>` 获得类型安全的元数据访问：

```tsx
const { messages } = useChat<MyUIMessage>({ /* ... */ });
// message.metadata?.createdAt — 类型安全的时间戳
// message.metadata?.totalTokens — 类型安全的 token 计数
```

常见用途：时间戳、模型信息、token 用量追踪、用户上下文、性能指标等。

---

## 5. 聊天工具使用 (Chatbot Tool Usage)

`useChat` + `streamText` 支持三种工具执行模式，适用于不同的交互场景：

1. **服务端自动执行** — `execute` 在服务端运行，结果透明返回
2. **客户端自动执行** — 通过 `onToolCall` 回调在客户端处理
3. **需用户交互** — UI 展示确认对话框，用户决定是否执行

### 完整执行流程

```
用户消息 → API路由 → streamText
    → 模型生成工具调用 → 转发到客户端
    → 服务端工具：执行 execute → 结果返回客户端
    → 客户端自动工具：onToolCall → addToolOutput
    → 交互工具：UI展示状态 → 用户操作 → addToolOutput
    → sendAutomaticallyWhen 检测 → 自动提交下一轮
```

### 工具 Part 状态机

每个工具 part 经历以下生命周期：

| 状态 | 含义 |
|------|------|
| `input-streaming` | 工具输入（参数）正在流式生成 |
| `input-available` | 工具输入完整，等待执行 |
| `output-available` | 工具执行完成，结果可用 |
| `output-error` | 工具执行过程中出错 |
| `approval-requested` | 等待用户批准（审批模式） |

### 服务端配置

有 `execute` 函数的工具在服务端自动执行。没有 `execute` 的是客户端工具——它们的调用会被转发到客户端，由 `onToolCall` 处理或通过 UI 交互。

```ts
const result = streamText({
  model: __MODEL__,
  messages: await convertToModelMessages(messages),
  tools: {
    getWeatherInformation: {
      description: '显示某个城市的天气',
      inputSchema: z.object({ city: z.string() }),
      execute: async ({ city }) => { /* 服务端执行 */ },
    },
    askForConfirmation: {
      description: '请求用户确认。',
      inputSchema: z.object({ message: z.string() }),
      // 无 execute = 客户端交互工具
    },
    getLocation: {
      description: '获取用户位置。',
      inputSchema: z.object({}),
      // 无 execute = 客户端自动工具
    },
  },
});
```

### 客户端处理

客户端的核心模式是：`onToolCall` 处理自动工具 → `addToolOutput` 提交结果 → `sendAutomaticallyWhen` 自动推进。

```tsx
const { messages, sendMessage, addToolOutput } = useChat({
  transport: new DefaultChatTransport({ api: '/api/chat' }),

  // 当所有工具结果就绪时自动提交
  sendAutomaticallyWhen: lastAssistantMessageIsCompleteWithToolCalls,

  async onToolCall({ toolCall }) {
    if (toolCall.dynamic) return; // 先检查动态工具（Harness 特有事件）

    if (toolCall.toolName === 'getLocation') {
      const cities = ['New York', 'Los Angeles', 'Chicago', 'San Francisco'];
      addToolOutput({
        tool: 'getLocation',
        toolCallId: toolCall.toolCallId,
        output: cities[Math.floor(Math.random() * cities.length)],
      }); // 不 await — 避免死锁
    }
  },
});
```

### 渲染工具 Part

每个工具类型在 UI 中需要根据其状态渲染不同内容。交互工具需要处理多个状态：

```tsx
{message.parts.map(part => {
  if (part.type === 'tool-askForConfirmation') {
    switch (part.state) {
      case 'input-streaming':
        return <div>加载确认请求中...</div>;
      case 'input-available':
        return (
          <div key={part.toolCallId}>
            {part.input.message}
            <button onClick={() => addToolOutput({
              tool: 'askForConfirmation',
              toolCallId: part.toolCallId,
              output: 'Yes, confirmed.',
            })}>是</button>
            <button onClick={() => addToolOutput({
              tool: 'askForConfirmation',
              toolCallId: part.toolCallId,
              output: 'No, denied',
            })}>否</button>
          </div>
        );
      case 'output-available':
        return <div>结果: {part.output}</div>;
      case 'output-error':
        return <div>错误: {part.errorText}</div>;
    }
  }
})}
```

### 客户端工具错误处理

在 `onToolCall` 中使用 try-catch 处理工具执行错误，通过 `state: 'output-error'` 标记：

```tsx
onToolCall({ toolCall }) {
  if (toolCall.toolName === 'getWeatherInformation') {
    try {
      const weather = await getWeatherInformation(toolCall.input);
      addToolOutput({ tool: 'getWeatherInformation', toolCallId: toolCall.toolCallId, output: weather });
    } catch (err) {
      addToolOutput({
        tool: 'getWeatherInformation',
        toolCallId: toolCall.toolCallId,
        state: 'output-error',
        errorText: '无法获取天气信息',
      });
    }
  }
}
```

### 工具执行审批

服务端通过 `toolApproval` 配置需要人工审批的工具：

```ts
streamText({
  tools: { getWeather: tool({ /* ... */ }) },
  toolApproval: { getWeather: 'user-approval' },
});
```

客户端处理 `approval-requested` 状态的工具 part，通过 `addToolApprovalResponse` 批准或拒绝：

```tsx
case 'approval-requested': {
  if (part.approval.isAutomatic) {
    return <div>自动批准中...</div>;
  }
  return (
    <div>
      批准执行 {part.input.city} 的天气查询？
      <button onClick={() => addToolApprovalResponse({
        toolCallId: part.toolCallId, approved: true
      })}>批准</button>
      <button onClick={() => addToolApprovalResponse({
        toolCallId: part.toolCallId, approved: false
      })}>拒绝</button>
    </div>
  );
}
```

---

## 6. 生成式用户界面 (Generative UI)

生成式 UI 让 LLM 超越纯文本，直接"生成 UI"。核心思路是将 LLM 的工具调用结果连接到 React 组件，创建动态、自适应的用户界面。

### 工作原理

1. 向模型提供提示词和工具集
2. 模型根据对话上下文决定调用工具
3. 工具执行并返回结构化数据
4. 数据传递给对应的 React 组件渲染

### 创建工具

工具定义在 `ai/tools.ts` 中，供服务端和客户端共享：

```ts
import { tool as createTool } from 'ai';
import { z } from 'zod';

export const weatherTool = createTool({
  description: '显示某地天气',
  inputSchema: z.object({
    location: z.string().describe('要获取天气的地点'),
  }),
  execute: async function ({ location }) {
    await new Promise(resolve => setTimeout(resolve, 2000));
    return { weather: '晴天', temperature: 25, location };
  },
});

export const tools = { displayWeather: weatherTool };
```

### 创建 UI 组件

组件的 props 接口需要与工具返回值对齐：

```tsx
// components/weather.tsx
export const Weather = ({ temperature, weather, location }: {
  temperature: number; weather: string; location: string;
}) => (
  <div>
    <h2>{location} 当前天气</h2>
    <p>状况: {weather}</p>
    <p>温度: {temperature}°C</p>
  </div>
);
```

### 在聊天中渲染

根据 tool part 的类型和状态，渲染对应的 React 组件：

```tsx
{message.parts.map((part, index) => {
  if (part.type === 'text') {
    return <span key={index}>{part.text}</span>;
  }
  if (part.type === 'tool-displayWeather') {
    switch (part.state) {
      case 'input-available':
        return <div key={index}>正在加载天气...</div>;
      case 'output-available':
        return <div key={index}><Weather {...part.output} /></div>;
      case 'output-error':
        return <div key={index}>错误: {part.errorText}</div>;
    }
  }
  return null;
})}
```

要添加更多工具，只需遵循相同的模式：定义工具 → 创建组件 → 在 `parts` 中添加 `tool-xxx` 处理分支。

---

## 7. MCP Apps（聊天中渲染交互式 MCP 工具 UI）

`experimental_MCPAppRenderer` 是一个专门的 React 组件，用于在聊天界面中渲染 MCP App 工具的交互式 UI。它处理沙箱隔离、资源加载和事件处理。

```tsx
import { experimental_MCPAppRenderer as MCPAppRenderer, isToolUIPart } from '@ai-sdk/react';

if (isToolUIPart(part)) {
  return (
    <MCPAppRenderer
      part={part}
      loadResource={loadResource}
      handlers={handlers}
      sandbox={sandbox}
      fallback={<div>Loading MCP App...</div>}
    />
  );
}
```

`isToolUIPart` 用于判断一个 part 是否为 MCP 工具 UI part，确保类型安全。

---

## 8. 文本补全 (Completion)

`useCompletion` 是 `@ai-sdk/react` 的一部分，用于创建文本补全界面。相比 `useChat`（管理完整对话），`useCompletion` 更适用于单次提示词、一次生成一个补全结果的场景。

```tsx
'use client';
import { useCompletion } from '@ai-sdk/react';

export default function Page() {
  const { completion, input, handleInputChange, handleSubmit, isLoading, stop } = useCompletion({
    api: '/api/completion',
  });

  return (
    <form onSubmit={handleSubmit}>
      <input name="prompt" value={input} onChange={handleInputChange} />
      <button type="submit">提交</button>
      {isLoading && <button onClick={stop}>停止</button>}
      <div>{completion}</div>
    </form>
  );
}
```

服务端使用 `streamText` + `prompt`（而非 `messages`）生成补全：

```ts
export async function POST(req: Request) {
  const { prompt } = await req.json();
  const result = streamText({ model: __MODEL__, prompt });
  return createUIMessageStreamResponse({
    stream: toUIMessageStream({ stream: result.stream }),
  });
}
```

`useCompletion` 提供受控和非受控两种输入管理模式。`handleSubmit`/`handleInputChange` 适合标准表单场景，`setInput`/`complete` 适合自定义组件。也支持 `experimental_throttle` 节流和 `onFinish`/`onError` 事件回调。

---

## 9. 对象生成 (Object Generation)

`useObject`（实验性功能，支持 React、Svelte、Vue）允许消费流式 JSON 对象。对象以增量方式生成和渲染，用户可以看到数据的逐步构建。

### 基础用法

Schema 定义在共享文件中，客户端和服务端使用同一份：

```ts
// schema.ts
import { z } from 'zod';
export const notificationSchema = z.object({
  notifications: z.array(z.object({
    name: z.string().describe('虚构人名'),
    message: z.string().describe('消息内容'),
  })),
});
```

**客户端：** `useObject` 流式消费部分结果，`object` 初始为 `undefined`：

```tsx
const { object, submit, isLoading } = useObject({
  api: '/api/notifications',
  schema: notificationSchema,
});
// object?.notifications?.map(...) — 注意处理 undefined
```

**服务端：** 使用 `streamText` + `Output.object()` 生成结构化对象：

```ts
const result = streamText({
  model: __MODEL__,
  output: Output.object({ schema: notificationSchema }),
  prompt: '生成3条通知消息',
});
return createTextStreamResponse({
  stream: toTextStream({ stream: result.stream }),
});
```

### Enum 输出模式

对于需要分类的场景，`useObject` 支持 enum 输出模式。Schema 必须有 `enum` 作为键：

```tsx
// 客户端
const { object, submit } = useObject({
  api: '/api/classify',
  schema: z.object({ enum: z.enum(['true', 'false']) }),
});

// 服务端
const result = streamText({
  output: Output.choice({ options: ['true', 'false'] }),
  prompt: `Classify: ${context}`,
});
```

`useObject` 也支持 `isLoading`、`stop`、`error` 状态管理和 `onFinish`/`onError` 事件回调。

---

## 10. 流式自定义数据 (Streaming Custom Data)

通过 `onData` 回调，服务端可以在流式响应中发送自定义的数据部分。自定义数据的 type 使用 `data-*` 命名模式，前端可以在 `onData` 回调中按类型处理。

---

## 11. 读取 UI Message 流 (Reading UI Message Streams)

`readUIMessageStream` 用于在传统聊天场景之外消费 UI Message 流——终端 UI、自定义客户端、服务端组件等。它将 `UIMessageChunk` 对象流转换为 `UIMessage` 对象的 `AsyncIterableStream`。

### 基础用法

```ts
import { readUIMessageStream, streamText, toUIMessageStream } from 'ai';

const result = streamText({
  model: __MODEL__,
  prompt: 'Write a short story about a robot.',
});

for await (const uiMessage of readUIMessageStream({
  stream: toUIMessageStream({ stream: result.stream }),
})) {
  console.log('Current message state:', uiMessage);
}
```

### 工具调用集成

当流中包含工具调用时，可以按 part 类型分别处理：

```ts
for await (const uiMessage of readUIMessageStream({
  stream: toUIMessageStream({ stream: result.stream }),
})) {
  uiMessage.parts.forEach(part => {
    switch (part.type) {
      case 'text': console.log('Text:', part.text); break;
      case 'tool-call': console.log('Tool called:', part.toolName); break;
      case 'tool-result': console.log('Tool result:', part.result); break;
    }
  });
}
```

### 恢复对话

支持从已有的 `UIMessage` 状态继续流式接收：

```ts
for await (const uiMessage of readUIMessageStream({
  stream: toUIMessageStream({ stream: result.stream }),
  message: lastMessage, // 从这个消息状态恢复
})) { /* ... */ }
```

---

## 12. 聊天消息持久化 (Chatbot Message Persistence)

`useChat` 通过 `DefaultChatTransport` 原生支持消息的存储和加载。

### 设计思路

- 消息以 UI Message 格式存储（而非模型消息格式），因为它包含了 `id`、`createdAt` 等前端展示字段
- 通过 `id` 参数关联聊天会话
- 通过 `messages` 参数传入初始消息（从数据库加载）

### 服务端验证

从数据库加载的消息（尤其是包含工具调用、自定义元数据或 data parts 的消息）应在发给模型前使用 `validateUIMessages` 验证：

```ts
const validatedMessages = await validateUIMessages({
  messages: [...previousMessages, message],
  tools,              // 确保工具调用匹配当前 schema
  dataPartsSchema,    // 可选：验证 data parts
  metadataSchema,     // 可选：验证元数据
});
```

如果数据库中的消息与当前 schema 不匹配，会抛出 `TypeValidationError`。建议捕获此错误并优雅降级（如清空历史重新开始）。

### 存储消息

利用 `toUIMessageStream` 的 `onEnd` 回调在流完成时自动保存：

```ts
return createUIMessageStreamResponse({
  stream: toUIMessageStream({
    stream: result.stream,
    originalMessages: messages,
    onEnd: ({ messages }) => { saveChat({ chatId: id, messages }); },
  }),
});
```

---

## 13. 聊天流恢复 (Chatbot Resume Streams)

`useChat` 支持页面刷新后恢复正在进行的流。适用于长时间生成（如大型代码文件、长文本），在网络中断或用户刷新页面后从断点继续接收。

### 工作原理

1. 服务端将 SSE 流持久化到 Redis（通过 `resumable-stream` 包）
2. 数据库记录每个聊天当前活跃的 stream ID
3. 客户端重新连接时，`useChat` 自动发 GET 请求检查活跃流
4. 发现活跃流后自动重连并从断点继续接收

### 关键区别

在可恢复流设置中，客户端的 abort（关闭标签页、刷新页面、调用 `stop()`）被视为**断开连接**而非取消生成。如需让用户真正停止生成，应添加专用的停止端点。

### 客户端启用

```tsx
const { messages, sendMessage } = useChat({
  id: chatData.id,
  messages: chatData.messages,
  resume: true,  // 启用自动流恢复
  transport: new DefaultChatTransport({
    prepareSendMessagesRequest: ({ id, messages }) => ({
      body: { id, message: messages[messages.length - 1] },
    }),
  }),
});
```

### 自定义恢复端点

通过 `prepareReconnectToStreamRequest` 自定义重连时的 API 路径、请求头和凭据：

```tsx
transport: new DefaultChatTransport({
  prepareReconnectToStreamRequest: ({ id }) => ({
    api: `/api/chat/${id}/stream`,
    credentials: 'include',
    headers: { Authorization: 'Bearer token' },
  }),
}),
```

---

## 14. 错误处理 (Error Handling)

### 警告系统

AI SDK 在浏览器控制台显示警告（以 "AI SDK Warning:" 开头），帮助在问题引发错误之前发现它们。警告场景包括：使用了模型不支持的设置、兼容模式下功能行为不同、模型报告的其他问题。

**关闭所有警告：**
```ts
globalThis.AI_SDK_LOG_WARNINGS = false;
```

**自定义警告处理：**
```ts
globalThis.AI_SDK_LOG_WARNINGS = ({ warnings, provider, model }) => {
  // 自定义处理逻辑（如发送到监控系统）
};
```

### 客户端错误处理

每个 UI Hook 返回 `error` 对象用于渲染错误 UI。建议显示通用消息而非服务端错误详情。

```tsx
const { error, regenerate } = useChat({
  onError: (error) => console.error(error),  // 回调方式
});
{error && <div>错误: {error.message}</div>}
```

也可以自定义提交处理函数，在错误存在时先移除最后一条消息再重新发送。

### 服务端错误处理

```ts
export async function POST(req: Request) {
  try {
    const result = streamText({ /* ... */ });
    return createUIMessageStreamResponse({
      stream: toUIMessageStream({ stream: result.stream }),
    });
  } catch (error) {
    return new Response('内部服务器错误', { status: 500 });
  }
}
```

### 测试用错误注入

在路由处理函数中直接抛出错误即可模拟错误场景：

```ts
export async function POST(req: Request) {
  throw new Error('This is a test error');
}
```

---

## 15. Transport（传输层）

Transport 层是 `useChat` 的网络通信抽象，提供对请求发送和响应处理的精细控制。

### DefaultChatTransport

默认 HTTP 传输实现。未指定 transport 时 `useChat` 自动使用它，默认 POST 到 `/api/chat`。

```tsx
const { messages, sendMessage } = useChat({
  transport: new DefaultChatTransport({
    api: '/api/chat',
    headers: { Authorization: 'Bearer token' },
    credentials: 'include',
  }),
});
```

### DirectChatTransport — 无 HTTP 直接通信

`DirectChatTransport` 绕过 HTTP 直接调用 Agent 的 `stream()` 方法，用途包括：

- **服务端渲染**：无需 API 端点直接在服务端运行 Agent
- **测试**：无需网络请求就能测试聊天功能
- **单进程应用**：桌面或 CLI 应用中客户端和 Agent 同进程运行

```tsx
import { DirectChatTransport, ToolLoopAgent } from 'ai';

const agent = new ToolLoopAgent({
  model: __MODEL__,
  instructions: 'You are a helpful assistant.',
  tools: { weather: weatherTool },
});

const { messages, sendMessage } = useChat({
  transport: new DirectChatTransport({ agent }),
});
```

工作流程：验证 UI 消息 → 转换为模型消息 → 调用 agent.stream() → 转换为 UI Message Stream。可配置 `sendReasoning`、`sendSources` 等选项。注意 `DirectChatTransport` 不支持流重连，因为没有持久化服务端流。

### WorkflowChatTransport

`@ai-sdk/workflow` 提供的 WorkflowChatTransport 专为 Workflow SDK 设计，自动处理流中断重连和页面刷新恢复：

```tsx
const transport = new WorkflowChatTransport({
  api: '/api/chat',
  maxConsecutiveErrors: 5,
  initialStartIndex: -50,  // 刷新时只获取最近50个chunk
  onChatEnd: ({ chatId, chunkIndex }) => { /* ... */ },
});
```

### 构建自定义 Transport

参考 `DefaultChatTransport`、`HttpChatTransport` 和 `ChatTransport` 接口源码，可以实现完全自定义的通信层（如 WebSocket transport）。

---

## 16. 流协议 (Stream Protocol)

流协议定义了数据从后端传输到 AI SDK UI 前端的方式。`useChat` 和 `useCompletion` 支持两种流协议。

### Text Stream Protocol（文本流协议）

纯文本流，每个 chunk 追加到之前的内容形成完整响应。仅支持基础文本数据，不支持工具调用。适用于 `useChat`、`useCompletion`、`useObject`。

```ts
// 服务端
return createTextStreamResponse({
  stream: toTextStream({ stream: result.stream }),
});

// 客户端
const { messages } = useChat({
  transport: new TextStreamChatTransport({ api: '/api/chat' }),
});
```

### UI Message Stream Protocol（UI 消息流协议）

富消息事件流，使用 Server-Sent Events (SSE) 格式。响应头标记 `x-vercel-ai-ui-message-stream: v1`。

**全部支持的 Part 类型：**

| Part 类型 | 说明 | 核心字段 |
|------|------|------|
| `start` | 新消息开始 | `messageId` |
| `text-start` | 文本块开始 | `id` |
| `text-delta` | 文本增量内容 | `id`, `delta` |
| `text-end` | 文本块结束 | `id` |
| `reasoning-start` | 推理块开始 | `id` |
| `reasoning-delta` | 推理增量 | `id`, `delta` |
| `reasoning-end` | 推理块结束 | `id` |
| `reasoning-file` | 推理中生成的文件 | `url`, `mediaType` |
| `source-url` | 外部 URL 引用 | `sourceId`, `url` |
| `source-document` | 文件引用 | `sourceId`, `mediaType`, `title` |
| `file` | 文件引用 | `url`, `mediaType` |
| `custom` | 提供商特定内容 | `kind` (格式: `{provider}.{provider-type}`) |
| `data-*` | 自定义结构化数据 | `data` |
| `error` | 错误信息 | `errorText` |
| `tool-input-start` | 工具输入开始 | `toolCallId`, `toolName` |
| `tool-input-delta` | 工具输入增量 | `toolCallId`, `inputTextDelta` |
| `tool-input-available` | 工具输入完整 | `toolCallId`, `toolName` |
| `tool-output-available` | 工具输出可用 | `toolCallId`, `output` |
| `tool-output-error` | 工具输出错误 | `toolCallId`, `errorText` |
| `finish` | 消息流结束 | `finishReason`, `totalUsage` |

SSE 格式示例：
```
data: {"type":"start","messageId":"msg-123"}
data: {"type":"text-start","id":"text-1"}
data: {"type":"text-delta","id":"text-1","delta":"Hello"}
data: {"type":"text-end","id":"text-1"}
data: {"type":"finish"}
data: [DONE]
```

自定义数据部分使用 `data-{customType}` 命名模式，前端可在 `onData` 回调中按类型处理。自定义部分使用 `kind` 字段标识特定的提供商内容类型（如 `openai.compaction`）。

---

## 常用 API 引用速览

| API | 类别 | 说明 |
|------|------|------|
| `useChat` | UI Hook | 聊天界面核心 Hook |
| `useCompletion` | UI Hook | 文本补全 Hook |
| `experimental_useObject` | UI Hook | 流式 JSON 对象 Hook |
| `experimental_useRealtime` | UI Hook | 实时语音对话 Hook |
| `convertToModelMessages` | 工具函数 | UI 消息 → 模型消息格式转换 |
| `pruneMessages` | 工具函数 | 修剪消息列表（适配上下文窗口） |
| `validateUIMessages` | 工具函数 | 验证消息中的工具调用、元数据和 data parts |
| `createUIMessageStreamResponse` | 流工具 | 创建 UI Message Stream HTTP 响应 |
| `createUIMessageStream` | 流工具 | 创建 UI Message Stream |
| `createTextStreamResponse` | 流工具 | 创建 Text Stream HTTP 响应 |
| `toUIMessageStream` | 流工具 | 模型流 → UI Message Stream 转换 |
| `toTextStream` | 流工具 | 模型流 → Text Stream 转换 |
| `pipeUIMessageStreamToResponse` | 流工具 | 写入 UI Message Stream 到 Node.js 响应 |
| `readUIMessageStream` | 流工具 | 以 AsyncIterable 方式读取 UI Message Stream |
| `InferUITools` | 类型工具 | 从工具集推断 UI 工具类型 |
| `InferUITool` | 类型工具 | 推断单个 UI 工具类型 |
| `DefaultChatTransport` | 传输层 | 默认 HTTP 聊天传输实现 |
| `TextStreamChatTransport` | 传输层 | 纯文本流传输实现 |
| `DirectChatTransport` | 传输层 | 直接进程内 Agent 通信（无 HTTP） |
| `WorkflowChatTransport` | 传输层 | Workflow SDK 专用传输（自动重连） |
| `experimental_MCPAppRenderer` | React 组件 | MCP App 交互式 UI 渲染器 |
| `isToolUIPart` | 辅助函数 | 判断 part 是否为 MCP 工具 UI part |
| `lastAssistantMessageIsCompleteWithToolCalls` | 辅助函数 | 检查最后一条助手消息工具调用是否完成 |
| `addToolOutput` | 工具操作 | 提交工具执行结果 |
| `addToolApprovalResponse` | 工具操作 | 提交工具审批响应 |
