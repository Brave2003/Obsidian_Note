# AI SDK Core

> AI SDK Core 是 Vercel AI SDK 的核心模块。大规模语言模型（LLM）是能够大规模理解、创造和参与人类语言的高级程序，它们在海量文本材料上训练，能够识别语言模式并预测给定文本中可能出现的内容。AI SDK Core 通过提供标准化接口简化了与 LLM 的集成，让你专注于为用户构建出色的 AI 应用，而非浪费时间处理技术细节。

- **官网**: https://ai-sdk.dev/docs/ai-sdk-core
- **包**: `ai` (npm)
- **支持环境**: Node.js / Edge Runtime / 浏览器（部分功能）

---

## 1. 概述 (Overview)

### 核心函数

AI SDK Core 有多个函数，分别设计用于文本生成、结构化数据生成和工具使用。这些函数采用标准化的方式设置提示词和配置，让跨不同模型的工作更加容易。

| 函数 | 用途 | 适用场景 |
|------|------|----------|
| `generateText` | 生成文本和工具调用 | 非交互场景（自动写邮件、摘要网页）和需要工具的 Agent |
| `streamText` | 流式输出文本和工具调用 | 交互场景（聊天机器人、内容实时流式输出） |

两个函数都支持通过 `output` 属性生成结构化数据（`Output.object()`、`Output.array()`），允许生成类型化、schema 验证的数据，用于信息提取、合成数据生成、分类任务和生成式 UI 流式输出。

---

## 2. 生成和流式输出文本 (Generating Text)

### `generateText`

`generateText` 用于生成文本。该函数适用于非交互式场景，例如需要撰写文本的自动化任务（如起草邮件或总结网页），以及使用工具的 Agent。

```ts
import { generateText } from 'ai';

const { text } = await generateText({
  model: __MODEL__,
  prompt: '写一份四人份的素食千层面食谱。',
});
```

你可以使用更高级的提示词，让模型处理更复杂的指令和内容。例如通过 `instructions` 设定角色：

```ts
const { text } = await generateText({
  model: __MODEL__,
  instructions: '你是一位专业作家。写出简洁、清晰、精炼的内容。',
  prompt: `用3-5句话总结以下文章: ${article}`,
});
```

#### 返回结果对象详解

`generateText` 的结果对象包含生成的输出和元数据：

| 属性 | 类型 | 说明 |
|------|------|------|
| `result.text` | `string` | 最后一步生成的文本 |
| `result.content` | `ContentPart[]` | 所有步骤中生成的内容数组 |
| `result.files` | `GeneratedFile[]` | 所有步骤中模型生成的文件 |
| `result.sources` | `Source[]` | 引用来源（仅 Perplexity、Google 等部分模型支持） |
| `result.toolCalls` | `ToolCall[]` | 所有步骤中进行的工具调用 |
| `result.toolResults` | `ToolResult[]` | 所有步骤中工具调用的结果 |
| `result.finishReason` | `string` | 模型结束生成的原因（如 `'stop'`、`'length'`、`'tool-calls'`） |
| `result.rawFinishReason` | `string` | 提供商返回的原始结束原因 |
| `result.usage` | `TokenUsage` | 所有步骤的总 token 用量（跨多步生成） |
| `result.warnings` | `Warning[]` | 所有步骤中模型的警告（如不支持的设置） |
| `result.steps` | `StepResult[]` | 每步详细信息，含 `performance` 性能数据 |
| `result.finalStep` | `StepResult` | 最后一步详细信息，含 `performance` 和 `response` |
| `result.output` | `OUTPUT` | 使用 `output` 规范生成的结构化输出 |

#### 性能指标详解 (Performance)

每步包含 `performance` 对象提供计时和吞吐量信息。对于 `generateText`（非流式），`outputTokensPerSecond`、`inputTokensPerSecond` 和 `timeToFirstOutputMs` 为 `undefined`。

| 指标 | 说明 |
|------|------|
| `effectiveOutputTokensPerSecond` | 有效输出 tokens/秒 = `outputTokens / requestSeconds` |
| `outputTokensPerSecond` | 首个输出块之后的 tokens/秒（仅流式） |
| `inputTokensPerSecond` | 首个输出块之前的输入 tokens/秒（仅流式） |
| `effectiveTotalTokensPerSecond` | 有效总 tokens/秒 = `(inputTokens + outputTokens) / requestSeconds` |
| `stepTimeMs` | 步骤总耗时（模型响应时间 + 工具执行时间），毫秒 |
| `responseTimeMs` | 等待模型响应的耗时，毫秒 |
| `toolExecutionMs` | 该步骤中每个客户端工具执行的耗时，以 toolCallId 为 key |
| `timeToFirstOutputMs` | 首个输出块的延迟（仅流式），毫秒 |
| `timeBetweenOutputChunksMs` | 输出块之间的间隔统计，仅流式且至少两个输出块时有值，包含 `min`、`p10`、`median`、`avg`、`p90`、`max` |

#### 访问原始响应

有时你需要访问模型提供商的完整响应，例如获取提供商特有的响应头或响应体内容：

```ts
const result = await generateText({ /* ... */ });
console.log(JSON.stringify(result.finalStep.response.headers, null, 2));
console.log(JSON.stringify(result.finalStep.response.body, null, 2));
```

#### `onEnd` 回调

当使用 `generateText` 时，可提供 `onEnd` 回调，在最后一步完成后触发。回调参数包含 `text`、`finishReason`、`usage`、`responseMessages`、`steps`、`totalUsage` 等。

#### 生命周期回调 (Lifecycle Callbacks)（实验性 — 可能随版本变化）

`generateText` 提供了多个实验性生命周期回调，让你可以挂接到生成过程的不同阶段。这些回调适用于日志记录、可观测性、调试和自定义遥测。回调内部抛出的错误会被静默捕获，不会中断生成流程。

| 回调 | 触发时机 |
|------|------|
| `onStart` | `generateText` 操作开始时调用一次，在任何 LLM 调用之前。接收 model、messages、settings 和 `runtimeContext` |
| `onStepStart` | 每个步骤（LLM 调用）开始前。接收步数、模型、消息、工具和之前步骤 |
| `onLanguageModelCallStart` | 即将进行提供商模型调用前 |
| `onLanguageModelCallEnd` | 模型响应已归一化和解析后，但在客户端工具执行之前 |
| `onToolExecutionStart` | 工具的 `execute` 函数运行前。接收 toolCall、messages 和 `toolContext` |
| `onToolExecutionEnd` | 工具的 `execute` 函数完成或出错后。接收 toolCall、`toolExecutionMs` 和 `toolOutput`（区分联合类型：`type: 'tool-result'` 含 `output`，或 `type: 'tool-error'` 含 `error`） |
| `onStepEnd` | 每个步骤完成后。含 `stepNumber`（从 0 开始的索引）、`finishReason`、`usage`、`performance` |

---

### `streamText`

根据不同模型和提示词，大语言模型可能需要长达一分钟才能完成响应。对于聊天机器人或实时应用等交互场景，这种延迟是不可接受的。

`streamText` 简化了从 LLM 流式传输文本的过程。它会立即开始流式输出并抑制错误，防止服务端崩溃（使用 `onError` 回调记录错误）。

```ts
import { streamText } from 'ai';

const result = streamText({
  model: __MODEL__,
  prompt: '发明一个新节日并描述其传统。',
});

// textStream 既是 ReadableStream 也是 AsyncIterable
for await (const textPart of result.textStream) {
  console.log(textPart);
}
```

`streamText` 使用**背压（backpressure）**机制，仅在被消费时才生成 tokens。你需要消费流才能使其完成。

#### 可 await 的 Promise 属性

`streamText` 返回的对象上提供多个 Promise，解析后可获取最终结果：

| 属性 | 说明 |
|------|------|
| `result.text` | 最终生成的文本 |
| `result.content` | 所有步骤生成的内容 |
| `result.finalStep` | 最后步骤详情（含 `performance`） |
| `result.files` | 模型生成的文件 |
| `result.sources` | 引用来源 |
| `result.toolCalls` | 已执行的工具调用 |
| `result.toolResults` | 工具调用结果 |
| `result.finishReason` | 结束原因 |
| `result.rawFinishReason` | 提供商原始结束原因 |
| `result.usage` | 总用量（跨多步） |
| `result.totalUsage` | 已弃用，使用 `result.usage` |
| `result.warnings` | 模型警告 |
| `result.steps` | 所有步骤详情（含 `performance`） |

#### `onError` 回调

`streamText` 立即开始流式传输，使错误成为流的一部分而不会导致服务端崩溃。使用 `onError` 记录错误：

```ts
const result = streamText({
  model: __MODEL__,
  prompt: '...',
  onError({ error }) {
    console.error(error);
  },
});
```

#### `onChunk` 回调

为流中的每个块触发。接收以下所有流块类型（20+ 种）：

`start`、`start-step`、`text-start`、`text-delta`、`text-end`、`reasoning-start`、`reasoning-delta`、`reasoning-end`、`custom`、`source`、`file`、`reasoning-file`、`tool-call`、`tool-input-start`、`tool-input-delta`、`tool-input-end`、`tool-result`、`tool-error`、`tool-output-denied`、`tool-approval-request`、`tool-approval-response`、`finish-step`、`finish`、`abort`、`error`、`raw`

```ts
const result = streamText({
  model: __MODEL__,
  prompt: '...',
  onChunk({ chunk }) {
    if (chunk.type === 'text-delta') {
      console.log(chunk.text);
    }
  },
});
```

#### `stream` 属性 — 原始流处理

你可以通过 `stream` 属性读取包含所有事件的原始流，这对于自定义 UI 或以不同方式处理流非常有用：

```ts
for await (const part of result.stream) {
  switch (part.type) {
    case 'start':            // 流开始
    case 'start-step':       // 步骤开始
    case 'text-start':       // 文本开始
    case 'text-delta':       // 文本增量
    case 'text-end':         // 文本结束
    case 'reasoning-start':  // 推理开始
    case 'reasoning-delta':  // 推理增量
    case 'reasoning-end':    // 推理结束
    case 'source':           // 来源
    case 'file':             // 文件
    case 'tool-call':        // 工具调用
    case 'tool-input-start': // 工具输入开始
    case 'tool-input-delta': // 工具输入增量
    case 'tool-input-end':   // 工具输入结束
    case 'tool-result':      // 工具结果
    case 'tool-error':       // 工具错误
    case 'tool-approval-request':  // 审批请求
    case 'tool-approval-response': // 审批响应
    case 'finish-step':      // 步骤结束
    case 'finish':           // 流结束
    case 'abort':            // 流中止
    case 'error':            // 错误
    case 'raw':              // 原始值
  }
}
```

#### 流转换 (Stream Transformation)

使用 `experimental_transform` 选项转换流，用于过滤、修改或平滑文本流。转换在回调调用和 Promise 解析之前应用。

**平滑流 (`smoothStream`)：**

`simulateReadableStream` 从 `ai` 导入，`smoothStream` 用于平滑文本和推理流式输出：

```ts
import { smoothStream, streamText } from 'ai';

const result = streamText({
  model, prompt,
  experimental_transform: smoothStream(),
});
```

**自定义转换：**

转换函数接收模型可用的工具，返回用于转换流的函数。以下是全部转为大写的示例：

```ts
const upperCaseTransform =
  <TOOLS extends ToolSet>() =>
  (options: { tools: TOOLS; stopStream: () => void }) =>
    new TransformStream<TextStreamPart<TOOLS>, TextStreamPart<TOOLS>>({
      transform(chunk, controller) {
        controller.enqueue(
          chunk.type === 'text-delta'
            ? { ...chunk, text: chunk.text.toUpperCase() }
            : chunk,
        );
      },
    });
```

**停止流：**

你可以通过 `stopStream` 函数在转换中停止流。例如当模型护栏被违反时（生成了不当内容）。调用 `stopStream` 时，需要模拟 `finish-step` 和 `finish` 事件以保证返回格式良好的流，且所有回调都会被调用。

**多个转换链：**

```ts
experimental_transform: [firstTransform, secondTransform]
```

#### `onAbort` 回调

当流被中止时（通过 AbortSignal），`onAbort` 被调用但 `onEnd` 不会被调用。接收已完成步骤的数组：

```ts
const { textStream } = streamText({
  model: __MODEL__,
  prompt: '...',
  onAbort: ({ steps }) => {
    console.log('流被中止，已完成步骤:', steps.length);
  },
  onEnd: ({ steps, totalUsage }) => {
    console.log('流正常完成');
  },
});
```

#### 流协议集成

你可以将 `streamText` 单独使用，也可以与 AI SDK UI 和 AI SDK RSC 结合使用。`result.stream` 可传递给以下独立辅助函数：

| 辅助函数 | 用途 |
|------|------|
| `createUIMessageStreamResponse({ stream: toUIMessageStream({ stream: result.stream }) })` | 创建 UI Message 流 HTTP 响应，用于 Next.js App Router API 路由 |
| `pipeUIMessageStreamToResponse({ stream: toUIMessageStream({ stream: result.stream }), response })` | 写入 UI Message 流到 Node.js 响应对象 |
| `createTextStreamResponse({ stream: toTextStream({ stream: result.stream }) })` | 创建纯文本流 HTTP 响应 |
| `pipeTextStreamToResponse({ stream: toTextStream({ stream: result.stream }), response })` | 写入纯文本流到 Node.js 响应对象 |

---

### 来源引用 (Sources)

部分提供商（如 Perplexity 和 Google）在响应中包含来源引用。目前来源仅限于为响应提供依据的网页。每个 `url` 来源包含：

- `id`: 来源 ID
- `url`: 来源 URL
- `title`: 可选标题
- `providerMetadata`: 提供商元数据

```ts
const result = await generateText({
  model: 'google/gemini-2.5-flash',
  tools: { google_search: google.tools.googleSearch({}) },
  prompt: '列出过去一周旧金山的5大新闻。',
});

for (const source of result.sources) {
  if (source.sourceType === 'url') {
    console.log('ID:', source.id, '标题:', source.title, 'URL:', source.url);
  }
}
```

`streamText` 也可通过 `result.sources` Promise 或在 `onChunk` 中捕获 `source` 块类型来获取来源。

---

## 3. 生成结构化数据 (Generating Structured Data)

虽然文本生成很有用，但实际用例通常需要生成结构化数据——例如从文本中提取信息、对数据进行分类或生成合成数据。

许多语言模型能够生成结构化数据，通常定义为"JSON 模式"或"工具"。然而，你需要手动提供 schema 并验证生成的数据，因为 LLM 可能产生错误或不完整的结构化数据。

AI SDK 通过 `generateText` 和 `streamText` 的 `output` 属性标准化了跨模型提供商的结构化对象生成。你可以使用 Zod schema、Valibot 或 JSON schema 来指定期望的数据形状，AI 模型将生成符合该结构的数据。

结构化输出是 `generateText` 和 `streamText` 流程的一部分，意味着你可以在同一请求中将其与工具调用结合。但结构化输出生成计为一个步骤，需要在使用工具时在 `stopWhen` 配置中考虑这一点。

### `Output.text()`

默认行为，生成纯文本，不对结果进行任何 schema 约束：

```ts
import { generateText, Output } from 'ai';
const { output } = await generateText({
  output: Output.text(),
  prompt: '给我讲个笑话。',
});
// output: string
```

### `Output.object()`

使用 `Output.object({ schema })` 基于 schema 生成结构化对象。输出经过类型验证以确保结果匹配 schema：

```ts
import { generateText, Output } from 'ai';
import { z } from 'zod';

const { output } = await generateText({
  model: __MODEL__,
  output: Output.object({
    schema: z.object({
      name: z.string(),
      age: z.number().nullable(),
      labels: z.array(z.string()),
    }),
  }),
  prompt: '生成一个测试用户的信息。',
});
```

> 通过 `streamText` 流式传输的部分输出无法被验证，因为不完整的数据可能还不符合预期结构。

### `Output.array()`

使用 `Output.array({ element })` 指定期望从模型获得一个类型化对象数组，每个元素应符合 `element` 中定义的 schema：

```ts
const { output } = await generateText({
  output: Output.array({
    element: z.object({
      location: z.string(), temperature: z.number(), condition: z.string(),
    }),
  }),
  prompt: '列出 San Francisco 和 Paris 的天气。',
});
// output: [{ location: 'San Francisco', temperature: 70, condition: 'Sunny' }, ...]
```

流式数组时使用 `elementStream`，接收每个在生成时已完成且已验证的元素（不同于 `partialOutputStream`，后者流式传输整个部分数组含不完整元素）：

```ts
const { elementStream } = streamText({
  output: Output.array({ element: schema }),
  prompt: '生成3个英雄描述。',
});
for await (const hero of elementStream) {
  console.log(hero); // 每个英雄都是完整且已验证的
}
```

### `Output.choice()`

当期望模型从特定字符串选项中选择时使用，例如分类或固定枚举值：

```ts
const { output } = await generateText({
  output: Output.choice({ options: ['sunny', 'rainy', 'snowy'] }),
  prompt: '今天天气是晴天、下雨还是下雪？',
});
// output: 'sunny' | 'rainy' | 'snowy'
```

AI SDK 会验证结果匹配选项之一，如果模型返回无效值则抛出错误。这对于分类式生成或强制 API 兼容的有效值特别有用。

### `Output.json()`

生成并解析非结构化 JSON，不强制执行特定 schema。适用于需要任意对象、灵活结构的场景：

```ts
const { output } = await generateText({
  output: Output.json(),
  prompt: '以 JSON 格式返回每个城市的温度和天气条件。',
});
```

`Output.json` 仅检查响应是否为有效 JSON，不验证结构或类型。如需 schema 验证，使用 `.object` 或 `.array`。

### 属性描述 (Property Descriptions)

使用 `.describe("...")` 为单个 schema 属性添加描述，帮助模型理解每个属性的用途，提高生成数据的质量和准确性：

```ts
z.string().describe('食谱名称'),
z.string().describe('配料用量（克或毫升）'),
z.array(z.string()).describe('分步骤的烹饪说明'),
```

属性描述尤其适用于：澄清模糊的属性名、指定期望格式或约定、为复杂嵌套结构提供上下文。

### 输出命名 (Name & Description)

可选的 `name` 和 `description` 用于额外 LLM 指导（某些提供商使用它们，例如通过工具或 schema 名称）：

```ts
output: Output.object({
  name: 'Recipe',
  description: 'A recipe for a dish.',
  schema: { /* ... */ },
})
```

适用于所有支持结构化生成的输出类型。

### 流式结构化输出

由于返回结构化数据更复杂，模型响应时间可能对交互场景不可接受。使用 `streamText` + `output` 流式获取结构化响应：

```ts
const { partialOutputStream } = streamText({
  model: __MODEL__,
  output: Output.object({ schema }),
  prompt: '生成一份千层面食谱。',
});
for await (const partialObject of partialOutputStream) {
  console.log(partialObject);
}
```

客户端可使用 `useObject` hook 消费流式结构化输出。

当流式输出中发生错误时，它们会成为流的一部分而非抛出异常。提供 `onError` 回调处理错误。

### 结合工具调用

```ts
const result = await generateText({
  model: __MODEL__,
  tools: {
    weather: tool({
      description: '获取某地天气',
      inputSchema: z.object({ location: z.string() }),
      execute: async ({ location }) => ({ temperature: 72, condition: 'sunny' }),
    }),
  },
  output: Output.object({
    schema: z.object({ summary: z.string(), recommendation: z.string() }),
  }),
  stopWhen: isStepCount(5),
  prompt: '今天在旧金山我该穿什么？',
});
```

### 访问推理过程

通过 `result.reasoningText` 访问模型的思考过程（需要推理模型）。

### 错误处理

当 `generateText` 无法生成有效对象时，抛出 `AI_NoObjectGeneratedError`。错误保留以下信息：
- `text`: 模型生成的文本（原始文本或工具调用文本）
- `response`: 语言模型响应的元数据
- `usage`: 请求 token 用量
- `cause`: 错误原因（如 JSON 解析错误）

```ts
import { NoObjectGeneratedError } from 'ai';
try { /* ... */ } catch (error) {
  if (NoObjectGeneratedError.isInstance(error)) {
    console.log('原因:', error.cause);
    console.log('文本:', error.text);
  }
}
```

---

## 4. 工具调用 (Tool Calling)

工具是模型可以调用以执行特定任务的对象，包含以下核心元素：

- `description`: 可选，影响模型何时选择该工具。可以是字符串或基于 context/experimental_sandbox 的函数
- `inputSchema`: 定义输入参数的 Zod schema 或 JSON schema。被 LLM 消费，也用于验证 LLM 的工具调用
- `execute`: 可选异步函数。不提供则表示工具需要转发到客户端或队列
- `strict`: 可选布尔值，启用严格工具调用模式（需要提供商支持）
- `inputExamples`: 可选，指定示例输入以指导模型

使用 `tool` 辅助函数来推断 `execute` 参数的类型。`generateText` 和 `streamText` 的 `tools` 参数是一个以工具名为 key、工具定义为 value 的对象。

当模型使用工具时称为"工具调用"(tool call)，工具的输出称为"工具结果"(tool result)。工具调用不限于文本生成，也可用于渲染用户界面 (Generative UI)。

### 动态描述 (Dynamic Descriptions)

工具描述可以是固定字符串或函数。当发送给模型的描述应取决于当前工具上下文或 experimental_sandbox 时使用函数（如租户、项目、环境或工作空间）。描述函数在每个生成步骤的工具定义发送到模型前解析。

```ts
const shell = tool({
  contextSchema: z.object({ projectName: z.string() }),
  description: ({ context, experimental_sandbox }) => [
    `为 ${context.projectName} 项目运行 shell 命令。`,
    experimental_sandbox ? `沙箱: ${experimental_sandbox.description}` : undefined,
  ].filter(Boolean).join('\n'),
  inputSchema: z.object({ command: z.string() }),
  execute: async ({ command }, { experimental_sandbox }) => {
    if (!experimental_sandbox) throw new Error('沙箱不可用');
    return experimental_sandbox.run({ command });
  },
});
```

### 严格模式 (Strict Mode)

启用后，支持严格工具调用的提供商将只生成根据你定义的 `inputSchema` 有效的工具调用，提高了工具调用的可靠性。但并非所有 schema 都支持严格模式，支持情况取决于具体提供商。默认禁用，可按工具设置：

```ts
tool({ /* ... */, strict: true })
```

### 输入示例 (Input Examples)

为工具指定示例输入可帮助模型理解输入数据应如何结构化。当 JSON schema 本身不能完全说明预期用途或存在可选值时，示例输入很有帮助。目前只有 Anthropic 提供商原生支持工具输入示例，其他提供商会忽略此设置。

```ts
tool({
  /* ... */
  inputExamples: [
    { input: { location: 'San Francisco' } },
    { input: { location: 'London' } },
  ],
})
```

### 工具执行审批 (Tool Execution Approval)

默认情况下，有 `execute` 函数的工具在模型调用时自动运行。通过 `generateText`、`streamText` 或 `ToolLoopAgent` 上的 `toolApproval` 配置审批（旧的 `needsApproval` 属性已弃用）。

`toolApproval` 可以是：
- 一个 `GenericToolApprovalFunction`（处理所有工具调用）
- 一个按工具的状态和/或 `SingleToolApprovalFunction` 回调映射

**四种状态：**

| 状态 | 含义 |
|------|------|
| `not-applicable` | 不触发审批流程，正常执行（默认） |
| `approved` | 自动批准，发出审批请求/响应部分记录决策 |
| `denied` | 自动拒绝，发出审批请求/响应并返回拒绝输出，可含原因 |
| `user-approval` | 发出审批请求，等待手动审批响应 |

**手动审批流程：**
1. 调用 `generateText`/`streamText` 并附带 `toolApproval`
2. 模型生成工具调用
3. 返回 `tool-approval-request` 内容部分
4. 应用请求审批并收集用户决策
5. 创建 `tool-approval-response` 添加到消息数组
6. 用更新后的消息再次调用
7. 如批准，工具运行并返回结果；如拒绝，模型知晓拒绝并做出相应回应

**通用审批函数：**
```ts
toolApproval: ({ toolCall, tools, toolsContext, messages, runtimeContext }) => {
  if (toolCall.toolName === 'runCommand' && !toolCall.dynamic) {
    return 'user-approval';
  }
  return undefined; // 相当于 'not-applicable'
}
```

**动态审批（基于工具输入）：**
```ts
toolApproval: {
  processPayment: async ({ amount }) =>
    amount > 1000 ? 'user-approval' : undefined,
}
```

**自动审批/拒绝带原因：**
```ts
toolApproval: {
  runCommand: { type: 'denied', reason: 'blocked by policy' },
}
```

**与 useChat 配合：** 审批流通过 UI 状态处理，使用 `addToolApprovalResponse`。

### 多步调用 (Multi-Step Calls)

使用 `stopWhen` 启用多步调用。当设置 `stopWhen` 且模型生成工具调用时，AI SDK 会触发一次新的生成，传入工具结果，直到没有更多工具调用或满足停止条件为止。

**内置停止条件：**

| 条件 | 说明 |
|------|------|
| `isStepCount(count)` | 指定步数后停止（默认 20） |
| `hasToolCall(...toolNames)` | 当指定工具被调用时停止 |
| `isLoopFinished()` | 永不触发，让循环自然结束 |

可组合多个条件为数组，或创建自定义条件。`stopWhen` 条件仅在上一步包含工具结果时才被评估。

---

## 5. MCP 应用 (MCP Apps)

MCP Apps 扩展了 Model Context Protocol (MCP) 工具，增加了交互式 UI 资源。模型仍然调用普通的 MCP 工具，但工具可以指向包含 HTML 的 `ui://` 资源，你的应用在沙箱 iframe 中渲染这些 HTML。

**AI SDK 提供两部分：**
- `@ai-sdk/mcp` 辅助函数：用于声明 MCP Apps 支持、分离模型可见和应用可见工具、读取 `ui://` 资源
- `@ai-sdk/react` 组件：用于渲染应用 iframe 和桥接 MCP Apps JSON-RPC 消息

**宿主流程：**
1. 使用 MCP Apps 客户端能力连接到 MCP 服务器
2. 列出工具并按 MCP Apps 可见性分离
3. 仅将模型可见工具传递给 `streamText`/`generateText`
4. 当工具部分包含 MCP App 元数据时，读取应用的 `ui://` 资源
5. 在沙箱 iframe 中渲染 HTML 资源
6. 代理允许的 iframe 请求（如应用可见的 `tools/call`）回 MCP 服务器

**关键 API：**
- `mcpAppClientCapabilities` — 声明宿主可渲染 `text/html;profile=mcp-app` 资源
- `splitMCPAppTools(definitions)` — 分离 `modelVisible` 和 `appVisible` 工具
- `readMCPAppResource({ client, uri })` — 读取并规范化应用资源
- `experimental_MCPAppRenderer` — React 端渲染组件（实验性）

**最佳实践：**
- 将 MCP App HTML 视为不受信任内容，在沙箱 iframe 中渲染
- 使用 `splitMCPAppTools`，仅暴露 `modelVisible` 工具
- 在服务端验证每个 iframe 请求后再调用 `client.callTool`
- 按 `resourceUri` 缓存应用资源
- 保持工具 `content` 和 `structuredContent` 在无 UI 时也能用
- 响应完成后关闭短期 MCP 客户端

---

## 6. 提示词工程 (Prompt Engineering)

### 工具提示词技巧

当创建包含工具的提示词时，随着工具数量和复杂度的增加，获得良好结果可能会很棘手。以下建议可帮助获得最佳结果：

1. 使用工具调用能力强的模型（如 `gpt-5` 或 `gpt-4.1`）
2. 保持工具数量在 5 个或更少
3. 保持工具参数复杂度低。深层嵌套、可选元素和联合类型对模型来说难以处理
4. 为工具、参数、参数属性使用语义化的名称
5. 使用 `.describe("...")` 为 Zod schema 属性提供提示
6. 当工具输出对模型可能不清晰且工具间存在依赖时，使用工具的 `description` 字段提供工具执行输出的信息
7. 在提示词中包含工具调用的示例输入/输出（JSON 格式）

总的目标是：以清晰的方式给模型所有需要的信息。

### 工具和结构化数据 Schema

**Zod Dates：** Zod 期望 JavaScript Date 对象，但模型将日期返回为字符串。使用 `z.string().datetime()` 或 `z.string().date()`，然后用 Zod 转换器将字符串转为 Date 对象。

**可选参数：** 使用 OpenAI 严格模式时，可选参数应使用 `.nullable()` 替代 `.optional()` 以获得最大兼容性。

**温度设置：** 对于工具调用和对象生成，推荐使用 `temperature: 0` 以确保确定性和一致性。较低的温度值减少了模型输出的随机性，当模型需要生成特定格式的结构化数据、使用正确参数进行精确工具调用或严格遵循 schema 时尤为重要。

### 调试 (Debugging)

**检查警告：** 并非所有提供商都支持 AI SDK 的全部功能。提供商在不支持功能时会抛出异常或返回警告。
```ts
console.log(result.warnings);
```

**检查请求消息：** 通过设置 `include.requestMessages: true` 查看发送到模型的输入消息。
```ts
console.log(result.finalStep.request.messages);
```

**检查 HTTP 请求体：** 对于暴露请求体的模型（如 OpenAI），可检查发送到模型提供商的精确负载。
```ts
console.log(result.finalStep.request.body);
```

---

## 7. 设置 (Settings)

### 语言模型调用选项 (LanguageModelCallOptions)

所有 AI SDK 函数支持以下通用设置，外加模型、提示词和提供商特有设置。某些提供商可能不支持全部通用设置，使用不支持的设置时会生成警告。

| 设置 | 说明 |
|------|------|
| `maxOutputTokens` | 最大生成 token 数 |
| `temperature` | 温度参数。值传递给提供商，范围取决于提供商和模型。对大多数提供商，`0` 表示近乎确定性，更高值表示更多随机性。**AI SDK 5.0 开始不再默认为 0。** 推荐只设置 `temperature` 或 `topP`，不要同时设置 |
| `topP` | 核采样。对大多数提供商是 0 到 1 之间的数，如 0.1 表示仅考虑前 10% 概率质量的 token。推荐只设置 `temperature` 或 `topP` |
| `topK` | 仅从前 K 个选项中采样。推荐仅高级用例，通常只用 `temperature` 即可 |
| `presencePenalty` | 存在惩罚，影响模型重复提示中已有信息的可能性。`0` 表示无惩罚 |
| `frequencyPenalty` | 频率惩罚，影响模型反复使用相同词或短语的可能性。`0` 表示无惩罚 |
| `stopSequences` | 停止序列。模型生成到任一序列时停止 |
| `seed` | 随机采样的种子（整数）。如果模型支持，相同种子产生确定性结果 |
| `reasoning` | 推理级别控制。详见推理章节 |

### 请求选项 (RequestOptions)

| 选项 | 说明 |
|------|------|
| `maxRetries` | 最大重试次数，设为 `0` 禁用重试。默认 `2` |
| `abortSignal` | 取消信号，如 `AbortSignal.timeout(5000)` |
| `timeout` | 超时（毫秒），也可为对象 `{ totalMs, stepMs, chunkMs, toolMs, tools: { xxxMs } }` |
| `headers` | 自定义 HTTP 头，如 `{'Prompt-Id': 'my-id'}` |

#### 超时详解

超时可指定为数字（毫秒）或对象：

| 超时字段 | 说明 |
|------|------|
| `totalMs` | 整个调用（含所有步骤）的总超时 |
| `stepMs` | 每个单独步骤（LLM 调用）的超时。多步生成中独立限制每步时间 |
| `chunkMs` | 流块间超时（仅流式）。若在此期间未收到新块则中止，用于检测停滞流 |
| `toolMs` | 所有工具执行的默认超时。若工具超时则中止并返回 tool-error 供模型响应或重试 |
| `tools.{toolName}Ms` | 按工具超时覆盖，优先级高于 `toolMs` |

---

## 8. 推理 (Reasoning)

许多语言模型支持在生成响应前的内部"推理"阶段（有时也称"思考"）。AI SDK 提供顶层 `reasoning` 参数，使用单一可移植设置跨提供商控制此行为。

### 基本用法

```ts
const { text, reasoning, reasoningText } = await generateText({
  model: 'anthropic/claude-sonnet-4.6',
  reasoning: 'medium',
  prompt: '2040年世界人口有多少？',
});
```

### 推理级别

| 值 | 行为 |
|------|------|
| `provider-default` | 使用提供商默认推理行为（省略时的默认值） |
| `none` | 禁用推理 |
| `minimal` | 最少推理 |
| `low` | 快速、简洁推理 |
| `medium` | 平衡推理 |
| `high` | 深入推理 |
| `xhigh` | 最大推理 |

### 优先级规则

顶层 `reasoning` 参数和 `providerOptions` 中的推理设置**永不合并**。如果 `providerOptions` 包含了推理相关设置，它们完全优先，`reasoning` 参数被忽略。这种设计让你默认使用可移植的 `reasoning` 参数，仅在需要提供商特有功能（如精确 token 预算）时才使用 `providerOptions`。

### 支持的提供商

支持：OpenAI、Anthropic、Google、xAI、Groq、DeepSeek、Fireworks、Amazon Bedrock。每个提供商将值转换为其原生推理 API。某些提供商原生支持全部 6 级，其他则强制转换到较少级别（强制转换时发出警告）。某些提供商使用数值 token 预算而非枚举控制推理，此时顶层 `reasoning` 值映射为模型最大输出 tokens 百分比计算的预算。

不支持推理的提供商（如 Mistral、Perplexity、Cohere）发出 `unsupported` 警告并忽略参数。

### 从 `providerOptions` 迁移

如果当前通过 `providerOptions` 控制推理，可迁移到顶层 `reasoning` 以跨提供商移植。

```ts
// Before (Anthropic)
providerOptions: { anthropic: { thinking: { type: 'adaptive', effort: 'high' } } }

// After (Anthropic)
reasoning: 'high'

// 需要精确 token 预算时保留 providerOptions（如 budgetTokens: 12000）
```

---

## 9. 嵌入 (Embeddings)

嵌入是将单词、短语或图像表示为高维空间向量的一种方式。在此空间中，相似词彼此靠近，词之间的距离可用于度量它们的相似性。

### 嵌入单个值 (`embed`)

```ts
import { embed } from 'ai';
const { embedding } = await embed({
  model: 'openai/text-embedding-3-small',
  value: 'sunny day at the beach',
});
// embedding: number[]
```

### 批量嵌入 (`embedMany`)

加载数据时（如准备 RAG 数据存储），通常需要批量嵌入多个值：

```ts
const { embeddings } = await embedMany({
  model: 'openai/text-embedding-3-small',
  values: ['sunny day at the beach', 'rainy afternoon in the city', 'snowy night in the mountains'],
});
// embeddings: number[][]，与输入顺序相同
```

### 相似度计算

```ts
import { cosineSimilarity } from 'ai';
console.log(`余弦相似度: ${cosineSimilarity(embeddings[0], embeddings[1])}`);
```

### 设置选项

- `maxParallelCalls` — 并行请求数限制
- `maxRetries` — 重试次数（默认 2）
- `abortSignal` — 取消信号，如 `AbortSignal.timeout(1000)`
- `headers` — 自定义 HTTP 头
- `providerOptions` — 提供商特定参数，如 OpenAI 的 `dimensions: 512` 可减少嵌入维度

### 响应信息

`embed` 和 `embedMany` 都返回 `usage`（token 用量）和 `response`（原始提供商响应）。

### 嵌入中间件

通过 `wrapEmbeddingModel` 和 `EmbeddingModelMiddleware` 增强嵌入模型。内置 `defaultEmbeddingSettingsMiddleware` 可设置默认提供商选项。

---

## 10. 重排序 (Reranking)

重排序是一种用于提高搜索相关性的技术，通过根据文档与查询的相关性对一组文档重新排序。与基于嵌入的相似度搜索不同，重排序模型经过专门训练以理解查询和文档之间的关系，通常产生更准确的相关性分数。

### 基本用法

```ts
import { rerank } from 'ai';
import { cohere } from '@ai-sdk/cohere';

const documents = ['sunny day at the beach', 'rainy afternoon in the city', 'snowy night in the mountains'];

const { ranking } = await rerank({
  model: cohere.reranking('rerank-v3.5'),
  documents,
  query: 'talk about rain',
  topN: 2,
});
// ranking: [{ originalIndex: 1, score: 0.9, document: '...' }, ...]
```

### 结构化文档

也支持 JSON 对象文档，适合搜索数据库、邮件等结构化内容：

```ts
const documents = [
  { from: 'Paul Doe', subject: 'Follow-up', text: '...' },
  { from: 'John McGill', subject: 'Missing Info', text: '...' },
];
const { ranking, rerankedDocuments } = await rerank({ model, documents, query, topN: 1 });
```

### 结果说明

| 属性 | 说明 |
|------|------|
| `ranking` | 排序后的 `{ originalIndex, score, document }` 数组 |
| `rerankedDocuments` | 按相关性排序的文档（便捷属性） |
| `originalDocuments` | 原始文档数组 |
| `response` | 原始提供商响应 |

评分通常为 0-1，越高越相关。支持 `maxRetries`、`abortSignal`、`headers`、`providerOptions`。

### 支持的模型

| 提供商 | 模型 |
|------|------|
| Cohere | `rerank-v3.5`, `rerank-english-v3.0`, `rerank-multilingual-v3.0` |
| Amazon Bedrock | `amazon.rerank-v1:0`, `cohere.rerank-v3-5:0` |
| Together.ai | `Salesforce/Llama-Rank-v1`, `mixedbread-ai/Mxbai-Rerank-Large-V2` |

---

## 11. 图像生成 (Image Generation)

使用 `generateImage` 函数基于提示词生成图像：

```ts
import { generateImage } from 'ai';
const { image } = await generateImage({
  model: __IMAGE_MODEL__,
  prompt: 'Santa Claus driving a Cadillac',
});
const base64 = image.base64;      // base64 图像
const uint8Array = image.uint8Array; // Uint8Array 图像
```

### 设置选项

- **尺寸** (`size`)：`'1024x1024'` 等，支持尺寸因模型而异
- **宽高比** (`aspectRatio`)：`'16:9'` 等
- **多张图像** (`n`)：`n: 4` 生成 4 张。SDK 自动并行调用并管理批处理。每个模型有内部每调用限制（如 DALL-E 3 每次仅 1 张，DALL-E 2 支持最多 10 张），可通过 `maxImagesPerCall` 覆盖
- **种子** (`seed`)：确定性生成
- **提供商选项** (`providerOptions`)：如 OpenAI 的 `style: 'vivid'`, `quality: 'hd'`
- **abortSignal**、**headers**、**warnings**

### 通过语言模型生成图像

某些语言模型（如 Google `gemini-2.5-flash-image`）支持多模态输出含图像：

```ts
const result = await generateText({
  model: google('gemini-2.5-flash-image'),
  prompt: '生成一张漫画猫的图片',
});
for (const file of result.files) {
  if (file.mediaType.startsWith('image/')) {
    // file.base64, file.uint8Array, file.mediaType
  }
}
```

### 图像中间件

通过 `wrapImageModel` 和 `ImageModelV4Middleware` 增强图像模型，如设置默认尺寸。

### 错误处理

当无法生成有效图像时抛出 `AI_NoImageGeneratedError`。错误保留 `responses` 和 `cause`。

---

## 12. 实时 (Realtime)（实验性功能）

AI SDK 提供实时模型功能，通过 WebSocket 进行双向音频和文本对话。实时会话在浏览器中运行，使用服务端创建的短期 token 直接连接到提供商。

### 典型流程

1. 浏览器调用设置端点
2. 服务端使用 `experimental_realtime.getToken()` 创建短期 token
3. 浏览器打开到提供商或 AI Gateway 的 WebSocket 连接
4. 模型流式返回音频、文本和工具调用
5. 工具调用由应用通过 `onToolCall` 处理

### 服务端设置端点

`experimental_getRealtimeToolDefinitions` 将 AI SDK 工具转为提供商工具定义格式。

### 客户端会话 (`experimental_useRealtime`)

```tsx
import { experimental_useRealtime } from '@ai-sdk/react';

const realtime = experimental_useRealtime({
  model: openai.experimental_realtime('gpt-realtime'),
  api: { token: '/api/realtime/setup' },
  sessionConfig: {
    instructions: '你是一个乐于助人的助手。请简洁。',
    inputAudioTranscription: {},
    voice: 'alloy',
    turnDetection: { type: 'server-vad' },
  },
});
// realtime.connect(), realtime.disconnect(), realtime.messages
```

### 工具调用

客户端驱动的工具执行。对于立即可用的结果，从 `onToolCall` 返回。对于需要服务端支持的工具，调用应用特定的 API 端点。也可通过 `addToolOutput` 手动提交。

### AI Gateway

通过 AI Gateway 跨提供商工作，Gateway 在服务端标准化实时事件：

```ts
const token = await gateway.experimental_realtime.getToken({
  model: 'openai/gpt-realtime-2',
});
```

### 支持的提供商

OpenAI (`gpt-realtime`)、Google (`gemini-3.1-flash-live-preview`)、xAI (`grok-voice-latest`)，以及通过 AI Gateway 路由。

---

## 13. 转录 (Transcription)

使用 `transcribe` 函数将音频转为文字：

```ts
import { transcribe } from 'ai';
const transcript = await transcribe({
  model: openai.transcription('whisper-1'),
  audio: await readFile('audio.mp3'),
});
const text = transcript.text;                    // 转录文本
const segments = transcript.segments;             // 分段（含起止时间，如果可用）
const language = transcript.language;             // 语言（如 "en"）
const durationInSeconds = transcript.durationInSeconds; // 时长（秒）
```

`audio` 可以是 `Uint8Array`、`ArrayBuffer`、`Buffer`、base64 字符串或 URL。

### 设置

- `providerOptions`：提供商特定设置（如 `timestampGranularities: ['word']`）
- `download`：URL 下载时的大小限制（默认 2 GiB），可通过 `createDownload({ maxBytes })` 自定义
- `abortSignal`、`headers`、`warnings`

### 错误处理

抛出 `AI_NoTranscriptGeneratedError`（原因可能是模型未生成响应或响应无法解析），使用 `NoTranscriptGeneratedError.isInstance(error)` 判断。错误保留 `responses` 和 `cause`。

---

## 14. 语音 (Speech)

使用 `generateSpeech` 从文本生成语音：

```ts
import { generateSpeech } from 'ai';
const audio = await generateSpeech({
  model: openai.speech('tts-1'),
  text: 'Hello, world!',
  voice: 'alloy',
});
```

语言设置（提供商支持不同）：`language: 'es'`。语音数据通过 `result.audio.uint8Array` 或 `result.audio.base64` 访问。

### 设置：`providerOptions`、`abortSignal`、`headers`、`warnings`

### 错误处理

抛出 `AI_NoSpeechGeneratedError`，使用 `NoSpeechGeneratedError.isInstance(error)`。

---

## 15. 视频生成 (Video Generation)（实验性）

使用 `experimental_generateVideo` 生成视频：

```ts
import { experimental_generateVideo as generateVideo } from 'ai';
const { video } = await generateVideo({
  model: __VIDEO_MODEL__,
  prompt: 'A cat walking on a treadmill',
});
```

### 设置选项

- **宽高比** (`aspectRatio`)：`'16:9'`
- **分辨率** (`resolution`)：`'1280x720'`
- **时长** (`duration`)：秒数
- **帧率** (`fps`)：如 `24`
- **音频** (`generateAudio`)：`true`/`false`
- **多视频** (`n`)：一次生成多个。大多数模型每次调用仅支持 1 个，可通过 `maxVideosPerCall` 覆盖
- **图生视频**：提供输入图像，`prompt: { image: 'url', text: '描述' }`
- **种子** (`seed`)：确定性生成
- **提供商选项**：如 FAL 的 `loop: true, motionStrength: 0.8`
- **轮询超时** (`pollTimeoutMs`)：视频生成是异步的，可能需要数分钟。默认约 5 分钟，建议生产环境设为至少 10 分钟（600000ms）

### 错误处理

抛出 `AI_NoVideoGeneratedError`，使用 `NoVideoGeneratedError.isInstance(error)`。

---

## 16. 文件上传 (File Uploads)

使用 `uploadFile` 上传文件到提供商，获取可在后续 API 调用中使用的 `ProviderReference`。

`ProviderReference` 是一个 `Record<string, string>`，将提供商名称映射到提供商特定的文件标识符（如 `{ openai: 'file-abc123' }`）。

```ts
import { uploadFile, generateText } from 'ai';
const { providerReference } = await uploadFile({
  api: openai.files(), // 或简写 openai
  data: fs.readFileSync('./photo.png'),
  filename: 'photo.png',
});
```

媒体类型自动检测。支持提供商特定选项（如 OpenAI 的 `purpose: 'assistants'`）。

### 多提供商场景

切换提供商时，需上传到两者并合并引用：

```ts
const mergedReference = { ...openaiResult.providerReference, ...anthropicResult.providerReference };
```

支持的提供商：Anthropic、Google、OpenAI、xAI。

---

## 17. 语言模型中间件 (Middleware)

语言模型中间件是一种以模型无关的方式拦截和修改语言模型调用的方法，用于添加护栏、RAG、缓存和日志等功能。

通过 `wrapLanguageModel` 使用。多个中间件按提供顺序应用：`wrapLanguageModel({ model, middleware: [first, second] })`。

### 内置中间件

| 中间件 | 功能 |
|------|------|
| `extractReasoningMiddleware` | 从生成文本中提取推理信息（如 `<think>` 标签），暴露为 `reasoning` 属性。支持 `tagName` 和 `startWithReasoning` 选项 |
| `extractJsonMiddleware` | 去除 markdown 代码围栏提取 JSON，使 `Output.object()` 兼容包裹 JSON 的模型 |
| `simulateStreamingMiddleware` | 为非流式模型模拟流式行为，保持一致流式接口 |
| `defaultSettingsMiddleware` | 应用默认设置（`temperature`、`maxOutputTokens`、`providerOptions`） |
| `addToolInputExamplesMiddleware` | 为不原生支持 `inputExamples` 的提供商，将示例附加到工具描述文本 |

### 社区中间件

`@ai-sdk-tool/parser` 为不原生支持 OpenAI 风格 `tools` 参数的自托管和第三方模型扩展工具调用能力：
- `createToolMiddleware`：灵活创建自定义工具中间件
- `hermesToolMiddleware`：Hermes & Qwen 格式
- `gemmaToolMiddleware`：Gemma 3 格式

通过将函数 schema 转为提示词指令，然后分析生成文本提取函数调用尝试来工作。

### 自定义中间件

实现 `transformParams`、`wrapGenerate`、`wrapStream` 中的一个或多个。`transformParams` 在传递给模型前转换参数。`wrapGenerate`/`wrapStream` 包装 `doGenerate`/`doStream` 方法。

以下日志中间件示例展示了基本模式：记录参数和生成文本。

---

## 18. Skill 上传 (Skill Uploads)

使用 `uploadSkill` 上传自定义技能到提供商。技能是文件的捆绑包（如包含技能行为的 `SKILL.md`），提供商可以在沙箱容器环境中加载。

```ts
import { uploadSkill, generateText } from 'ai';
const { providerReference } = await uploadSkill({
  api: anthropic.skills(),
  files: [{ path: 'my-skill/SKILL.md', content: readFileSync('./SKILL.md') }],
  displayTitle: 'My Skill',
});
```

文件内容可以是 `Uint8Array` 或 base64 字符串。

### 上传结果

| 字段 | 说明 |
|------|------|
| `providerReference` | 将提供商名称映射到提供商特定技能 ID |
| `displayTitle` | 人类可读标题（如支持且提供） |
| `name` | 提供商从技能文件推断的名称 |
| `description` | 提供商从技能文件推断的描述 |
| `latestVersion` | 提供商分配的最新版本标识符 |
| `providerMetadata` | 额外提供商元数据 |
| `warnings` | 不支持的选项警告 |

### 多提供商

与文件上传类似，上传到多个提供商并合并引用。

### 在推理调用中使用

**Anthropic：** 通过 `providerOptions.anthropic.container.skills` 传入。

**OpenAI：** 通过 `tools.shell.environment.skills` 传入（`openai.tools.shell()`）。

支持的提供商：Anthropic、OpenAI。

---

## 19. 提供商与模型管理 (Provider & Model Management)

处理多个提供商和模型时，通常需要在中心位置管理它们，并通过简单的字符串 ID 访问模型。

### 自定义提供商 (`customProvider`)

允许预配置模型设置、提供模型名称别名和限制可用模型：

```ts
const openai = customProvider({
  languageModels: {
    'gpt-5.1': wrapLanguageModel({
      model: gateway('openai/gpt-5.1'),
      middleware: defaultSettingsMiddleware({ settings: { /* ... */ } }),
    }),
    'gpt-5.1-high-reasoning': /* 带预配置的别名 */,
  },
  fallbackProvider: gateway,
});
```

也可附加 `files` 和 `skills` 接口。若无 `fallbackProvider`，则限制为仅注册的模型。

### 提供商注册表 (`createProviderRegistry`)

```ts
const registry = createProviderRegistry({ gateway, anthropic, openai });
// 支持自定义分隔符: createProviderRegistry({...}, { separator: ' > ' })
```

使用：
- `registry.languageModel('openai:gpt-5.1')`
- `registry.embeddingModel('openai:text-embedding-3-small')`
- `registry.imageModel('openai:dall-e-3')`
- `registry.videoModel('fal:luma-dream-machine/ray-2')`
- `registry.files('openai')`
- `registry.skills('anthropic')`

---

## 20. 运行时与工具上下文 (Runtime and Tool Context)

上下文让你在不将状态放入提示词的情况下传递服务端状态。AI SDK 将共享运行时状态与按工具的执���状态分离。

### 上下文类型

| 概念 | 定义位置 | 读取位置 | 用途 |
|------|------|------|------|
| `runtimeContext` | 生成或 Agent 调用 | `prepareStep`、生命周期回调、步骤结果、遥测 | 共享生成或 Agent 状态 |
| `toolsContext` | 生成或 Agent 调用 | `prepareStep`、审批回调、工具上下文解析、工具描述函数 | 按工具名称索引的上下文值映射 |
| tool `context` | 工具的 `contextSchema` 验证 | 工具描述函数、`execute`、审批、生命周期回调 | 单个工具需要的值 |
| `telemetry.includeRuntimeContext` | 生成或 Agent 调用 | 遥测过滤 | 包含在遥测中的 `runtimeContext` 属性 |
| `telemetry.includeToolsContext` | 生成或 Agent 调用 | 遥测过滤 | 包含在遥测中的工具上下文属性 |

**选择正确上下文的指南：**
- 使用 `runtimeContext` 存放整个生成或 Agent 循环共享的状态（请求元数据、租户设置、功能标志）
- 使用 `toolsContext` + `contextSchema` 存放特定工具需要的值（API 密钥、作用域客户端、权限）
- 使用提示消息存放模型应推理或在回答中提及的信息
- 使用 `telemetry.includeRuntimeContext` / `telemetry.includeToolsContext` 减少遥测暴露

---

## 21. 错误处理 (Error Handling)

### 常规错误

使用 `try/catch` 处理：

```ts
try { const { text } = await generateText({ model, prompt: '...' }); }
catch (error) { /* 处理 */ }
```

### 流式错误

`streamText` 的 `stream` 结果支持 `error` 部分：

```ts
for await (const part of stream) {
  switch (part.type) {
    case 'error': { /* part.error */ break; }
    case 'abort': { /* 流中止 */ break; }
    case 'tool-error': { /* part.error */ break; }
  }
}
```

同时建议外层使用 `try/catch` 处理流外错误。

### 流中止

使用 `onAbort` 回调执行清理操作（`onEnd` 在流中止时不会被调用）。

### 错误类型

AI SDK 定义了丰富的错误类型体系，包括 `AI_APICallError`、`AI_NoObjectGeneratedError`、`AI_NoImageGeneratedError`、`AI_NoTranscriptGeneratedError`、`AI_NoSpeechGeneratedError`、`AI_NoVideoGeneratedError`、`AI_InvalidToolInputError`、`AI_TypeValidationError` 等 30+ 种错误类型，每个都有 `.isInstance()` 静态方法。

---

## 22. 测试 (Testing)

测试语言模型具有挑战性（非确定性、慢、昂贵）。AI SDK 从 `ai/test` 导入模拟提供商和测试辅助工具：

| 导出 | 说明 |
|------|------|
| `MockLanguageModelV4` | 模拟语言模型 v4 |
| `MockEmbeddingModelV4` | 模拟嵌入模型 v4 |
| `mockId` | 自增整数 ID |
| `mockValues` | 遍历数组值，耗尽后返回最后一个 |

`simulateReadableStream`（从 `ai` 导入）模拟带延迟的可读流。

支持测试场景：`generateText`、`streamText`、`generateText` with Output、`streamText` with Output、模拟 UI Message Stream 响应。

---

## 23. 遥测 (Telemetry)

AI SDK 使用 OpenTelemetry 收集遥测数据。安装 `@ai-sdk/otel` 包并注册：

```ts
import { registerTelemetry } from 'ai';
import { OpenTelemetry } from '@ai-sdk/otel';
registerTelemetry(new OpenTelemetry());
```

Next.js 在 `instrumentation.ts` 中注册。默认为启用（需注册集成），可通过 `telemetry: { isEnabled: false }` 按调用退出。

### 遥测元数据

- `functionId`：标识函数
- `runtimeContext`：额外信息（通过 `telemetry.includeRuntimeContext` 过滤）
- `includeToolsContext`：按工具过滤上下文

### 自定义集成

实现 `Telemetry` 接口（所有方法可选）：`onStart`、`onStepStart`、`onLanguageModelCallStart`、`onLanguageModelCallEnd`、`onToolExecutionStart`、`onToolExecutionEnd`、`onStepEnd`、`onEnd`、`onAbort`、`onEmbedEnd`、`onRerankEnd`。

### OpenTelemetry Span

`OpenTelemetry` 集成发出遵循 GenAI 语义约定的 span：
- `invoke_agent {modelId}`（根 span，INTERNAL）：覆盖完整操作
- `chat {modelId}`（步骤 span，CLIENT）：每个 LLM 提供商调用

### Tracing Channel（Node.js）

AI SDK 遥测事件还在 Node.js 的 `ai:telemetry` tracing channel 上追踪，允许可观测性提供商无需单独注册集成即可订阅。

---

## 24. DevTools & 生命周期回调

AI SDK 提供开发调试工具和完整的生命周期回调体系，包括 `onStart`、`onStepStart`、`onLanguageModelCallStart`、`onLanguageModelCallEnd`、`onToolExecutionStart`、`onToolExecutionEnd`、`onStepEnd` 等。
