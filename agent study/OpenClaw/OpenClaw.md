[文档中心 | OpenClaw 中文文档](https://openclaw-docs.dx3n.cn/tutorials/getting-started/hubs)

> [!Tip] 总流程
> `CLI` 发命令 -> `Gateway` 启动和编排 -> `Channel` 收发消息 -> `Routing` 选 agent/session -> `Auto-Reply` 调模型生成回复 -> `Outbound` 发回通道 -> `Config/Session/Media` 持久化状态。

**词解释**

| 词          | 解释                         |
| ---------- | -------------------------- |
| Gateway    | 总服务台，所有消息、控制UI、节点和插件都走这里开始 |
| Control UI | 浏览器中的控制面板                  |
| Channel    | 第三方聊天入口                    |
| Agent      | 负责思考和回复的AI助手               |
| Provider   | AI模型                       |
| Tool       | Agent可以进行的动作，比如读文件、跑命令、查网页 |
| Node       | 接入Gateway的外部设备或远程机器        |
| Session    | 一段独立聊天，防止不同人的上下文混在一起       |

--- 

## 架构总览

OpenClaw 是一个**多通道 AI 助手运行时**，核心设计目标：

- **自托管 Gateway**：一个长期运行的 Gateway 统一管理会话、通道、节点、控制 UI 和事件流。
- **多通道统一接入**：Telegram、WhatsApp、Discord、Slack、Signal、iMessage、WebChat 等使用同一套控制面。
- **插件化扩展**：通道、工具、Provider、语音、媒体、搜索、后台服务都可以通过插件能力接入。
- **节点能力扩展**：iOS、Android、macOS、远程机器都可以作为 Node 连接 Gateway，提供 Canvas、相机、语音或远程执行能力。
- **智能体内核稳定运行**：Lane 队列、上下文守护、模型回退、工具审批，保证长期可靠执行。

如果只用一句话概括最新版架构：

```
Gateway 是总服务台；Control UI、CLI、聊天通道和节点都连到它；Agent 在它背后调用模型、工具和
```

**整体分层**

``` 
┌─────────────────────────────────────────────────────┐
│                   CLI 层                            │
│        entry.ts → run-main.ts → command-registry    │
├─────────────────────────────────────────────────────┤
│                  Gateway 层（控制平面）               │
│  WebSocket · HTTP/Control UI · 通道管理 · 节点管理     │
├──────────────┬──────────────┬────────────────────────┤
│  Channel 层  │   Routing 层 │    Plugin 层            │
│  多通道适配  │  路由 + 会话键│  manifest + 能力注册     │
├──────────────┴──────────────┴────────────────────────┤
│               Auto-Reply / Agent 执行层               │
│   dispatch → get-reply → agent-runner → embedded PI  │
├─────────────────────────────────────────────────────┤
│                AI Provider 层                        │
│    Anthropic · OpenAI · Ollama · Bedrock · ...       │
├─────────────────────────────────────────────────────┤
│        节点 / 媒体 / 持久化 / 基础设施层                │
│ Nodes · Canvas · Config · Sessions · Security · Cron  │
└─────────────────────────────────────────────────────┘
```

