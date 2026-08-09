# Programmatic Integration（程序化集成）

> Hermes 提供三种协议用于从外部程序驱动 agent — IDE 插件、自定义 UI、CI 管道、嵌入式子 agent。

**所有三种协议驱动同一个 `AIAgent` 核心**，区别仅在于传输格式和暴露的功能集。

---

## 三种协议对比

| 协议 | 传输 | 最佳场景 | 定义位置 |
|------|------|---------|---------|
| **ACP** | JSON-RPC over stdio | IDE 客户端（VS Code、Zed、JetBrains） | `acp_adapter/` |
| **TUI Gateway** | JSON-RPC over stdio（或 WebSocket） | 自定义主机，需要精细控制 sessions 和 streaming | `tui_gateway/server.py` |
| **API Server** | HTTP + Server-Sent Events | OpenAI 兼容前端、语言无关 HTTP 客户端 | `gateway/platforms/api_server.py` |

---

## ACP（Agent Client Protocol）

`hermes acp` 启动一个 stdio JSON-RPC 服务器，使用 ACP 协议。用于 VS Code（Zed Industries 的 ACP 扩展）、Zed 和任何有 ACP 插件的 JetBrains IDE。

暴露的能力：session 创建、prompt 提交、流式 agent 消息块、工具调用事件、权限请求、session fork、取消和认证。工具输出渲染为 IDE 理解的 ACP `Diff`/`ToolCall` 内容块。

```bash
hermes acp                  # stdio 上提供 ACP
hermes acp --bootstrap      # 打印 ACP IDE 的安装片段
```

---

## TUI Gateway JSON-RPC

`tui_gateway/server.py` 是 Ink TUI（`hermes --tui`）和嵌入式仪表盘 PTY bridge 使用的协议。任何外部主机都可以通过 stdio（或 WebSocket，via `tui_gateway/ws.py`）使用相同协议。

### 方法目录（精选）

```text
prompt.submit           prompt.background       session.steer
session.create          session.list            session.active_list
session.activate        session.close           session.interrupt
session.history         session.compress        session.branch
session.title           session.usage           session.status
clarify.respond         sudo.respond            secret.respond
approval.respond        config.set / config.get commands.catalog
command.resolve         command.dispatch        cli.exec
reload.mcp              reload.env              process.stop
delegation.status       subagent.interrupt      spawn_tree.save / list / load
terminal.resize         clipboard.paste         image.attach
```

### 流式事件

`message.delta`、`message.complete`、`tool.start`、`tool.progress`、`tool.complete`、`approval.request`、`clarify.request`、`sudo.request`、`secret.request`、`gateway.ready`，以及 session 生命周期和错误事件。

### Pi 风格 RPC 映射

每个 Pi-mono RPC spec 命令都有对应的 TUI Gateway 等价方法：

| Pi 命令 | Hermes 等价 |
|---------|------------|
| `prompt` | `prompt.submit` |
| `steer` | `session.steer` |
| `follow_up` | `prompt.submit`（当前 turn 后排队） |
| `abort` | `session.interrupt` |
| `set_model` | `command.dispatch` for `/model` |
| `compact` | `session.compress` |
| `get_state` | `session.status` |
| `get_messages` | `session.history` |
| `switch_session` | `session.resume` |
| `fork` | `session.branch` |
| `ui_request`/`ui_response` | `clarify.respond`/`sudo.respond`/`secret.respond`/`approval.respond` |

---

## OpenAI 兼容 API Server

`gateway/platforms/api_server.py` 通过 HTTP 暴露 Hermes。适用于 Web 前端、curl 驱动的 CI runner 或非 Python 消费者。

### 端点

```text
POST /v1/chat/completions        OpenAI Chat Completions（SSE 流式）
POST /v1/responses               OpenAI Responses API（有状态）
POST /v1/runs                    启动运行，返回 run_id（202）
GET  /v1/runs/{id}               运行状态
GET  /v1/runs/{id}/events        SSE 生命周期事件流
POST /v1/runs/{id}/approval      解决待处理审批
POST /v1/runs/{id}/stop          中断运行
GET  /v1/capabilities            机器可读的功能标志
GET  /v1/models                  列出模型
GET  /health, /health/detailed   健康检查
```

Header：`X-Hermes-Session-Id`、`X-Hermes-Session-Key`、`X-Hermes-Model`

---

## 选择指南

| 你的场景 | 推荐协议 | 原因 |
|---------|---------|------|
| IDE 插件（IDE 已支持 ACP） | **ACP** | IDE 端零协议工作 |
| 自定义桌面/Web/TUI 主机 | **TUI Gateway JSON-RPC** | 完整的 Hermes 功能（斜杠命令、approval、clarify、多 agent、session 分支） |
| OpenAI 兼容前端或语言无关 HTTP | **API Server** | 任何能发 HTTP 的客户端 |
| Python 进程内嵌入（无子进程） | 直接 `import run_agent.AIAgent` | 参见 Agent Loop 文档 |

---

## Model 热切换

会话中的模型切换在所有界面上都可用：

- CLI / TUI: `/model claude-sonnet-4` 或 `/model openrouter:anthropic/claude-sonnet-4.6`
- TUI Gateway RPC: `command.dispatch` with `{"command": "/model claude-sonnet-4"}`
- ACP: IDE 以 prompt 形式发送斜杠命令，agent 分发它
- API Server: 请求体中包含 `model` 字段或设置 `X-Hermes-Model` header

Provider 感知解析内置（同一模型名在你的 provider 上选择正确的格式）。参见 `hermes_cli/model_switch.py`。

---

## 关于 --mode rpc

Hermes **没有** `--mode rpc` 标志。以上三种协议已覆盖所有用例：
- ACP → IDE 协议客户端
- TUI Gateway → stdio JSON-RPC 主机
- API Server → HTTP 客户端

如果发现这三种协议都无法填补的实际差距，请在 issue 中描述你要构建的具体消费者。
