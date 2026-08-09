# Gateway Internals（消息网关内部机制）

> 消息网关是连接 Hermes 到 20+ 外部消息平台的**长期运行进程**，通过统一架构处理所有平台的会话。

**核心文件**：`gateway/run.py`、`gateway/session.py`、`gateway/delivery.py`、`gateway/pairing.py`、`gateway/hooks.py`、`gateway/platforms/`

---

## 架构总览

```
┌─────────────────────────────────────────────────┐
│                  GatewayRunner                  │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Telegram │  │ Discord  │  │  Slack   │       │
│  │ Adapter  │  │ Adapter  │  │ Adapter  │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │             │             │             │
│       └─────────────┼─────────────┘             │
│                     ▼                           │
│              _handle_message()                  │
│                     │                           │
│         ┌───────────┼───────────┐               │
│         ▼           ▼           ▼               │
│  Slash command   AIAgent    Queue/BG            │
│    dispatch      creation   sessions            │
│                     │                           │
│                     ▼                           │
│                 SessionStore                    │
│              (SQLite persistence)               │
└───────┴─────────────┴─────────────┴─────────────┘
```

---

## 消息流

### 步骤 1：基础适配器检查活跃 session 守卫

- Agent 正在运行 → **队列消息**，设置中断事件
- `/approve`、`/deny`、`/stop` → **绕过守卫**（内联分发）

### 步骤 2：GatewayRunner._handle_message() 接收事件

- 通过 `_session_key_for_source()` 解析 session key
- 检查授权
- 检查是否为斜杠命令 → 分发到命令处理器
- 检查 agent 是否已在运行 → 拦截 `/stop`、`/status` 等
- 否则 → 创建 `AIAgent` 实例并运行对话

### 步骤 3：响应通过平台适配器发回

---

## Session Key 格式

```text
agent:main:{platform}:{chat_type}:{chat_id}
```

示例：`agent:main:telegram:private:123456789`

线程感知平台（Telegram forum topics、Discord threads、Slack threads）可能在 chat_id 中包含 thread ID。

> 永远不要手动构造 session key — 始终使用 `gateway/session.py` 的 `build_session_key()`。

**设计原因**：结构化 key 天然支持平台隔离、多用户并发、线程独立管理，且可反向解析出平台和聊天 ID 用于投递。

---

## 两级消息守卫

### Level 1 — 基础适配器（`gateway/platforms/base.py`）

检查 `_active_sessions`。如果活跃 → 队列化 `_pending_messages` + 设置中断事件。**在到达 GatewayRunner 之前**捕获。

### Level 2 — GatewayRunner（`gateway/run.py`）

检查 `_running_agents`。拦截特定命令（`/stop`、`/new`、`/queue`、`/status`、`/approve`、`/deny`），其他触发 `running_agent.interrupt()`。

**设计原因**：Level 1 提前阻止避免竞态条件。Level 2 是安全检查 — 确保不会错误地为同一 session 创建第二个 AIAgent。需要绕过守卫的命令（如 `/approve`）通过 `await self._message_handler(event)` 内联分发。

---

## 授权

多层授权检查，**按顺序**评估：

| 优先级 | 检查 | 说明 |
|--------|------|------|
| 1 | 按平台 allow-all | `TELEGRAM_ALLOW_ALL_USERS` — 该平台所有用户授权 |
| 2 | 平台 allowlist | `TELEGRAM_ALLOWED_USERS` — 逗号分隔用户 ID |
| 3 | DM 配对 | 已认证用户通过配对码配对新人 |
| 4 | 全局 allow-all | `GATEWAY_ALLOW_ALL_USERS` |
| 5 | 默认拒绝 | 未授权用户被拒绝 |

### DM 配对流程

```text
Admin: /pair
Gateway: "Pairing code: ABC123. Share with the user."
New user: ABC123
Gateway: "Paired! You're now authorized."
```

配对状态持久化，**重启后保留**。

---

## 斜杠命令分发

所有 Gateway 斜杠命令经过同一管道：

1. `hermes_cli/commands.py` 的 `resolve_command()` → 规范名称（处理别名、前缀匹配）
2. 对照 `GATEWAY_KNOWN_COMMANDS` 检查
3. 根据规范名称分发
4. 某些命令受 `gateway_config_gate` 门控

### Running-Agent Guard

```python
if _quick_key in self._running_agents:
    if canonical == "model":
        return "⏳ Agent running — wait or /stop first."
```

**绕过命令**：`/stop`、`/new`、`/approve`、`/deny`、`/queue`、`/status`

---

## 配置来源

| 来源 | 提供 |
|------|------|
| `~/.hermes/.env` | API keys、bot tokens、平台凭证 |
| `~/.hermes/config.yaml` | 模型设置、工具配置、显示选项 |
| 环境变量 | 覆盖以上任何内容 |

> CLI 使用 `load_cli_config()`（含硬编码默认值），Gateway 直接读 YAML。CLI 默认值中存在但用户 config 中不存在的键可能在两者间行为不同。

---

## 平台适配器（21 个）

```text
gateway/platforms/
├── base.py              # BaseAdapter — 所有平台共享逻辑
├── telegram.py          # Telegram Bot API
├── discord.py           # Discord bot
├── slack.py             # Slack Socket Mode
├── whatsapp.py          # WhatsApp Business Cloud API
├── signal.py            # Signal via signal-cli REST API
├── matrix.py            # Matrix via mautrix (可选 E2EE)
├── mattermost.py        # Mattermost WebSocket API
├── email.py             # Email via IMAP/SMTP
├── sms.py               # SMS via Twilio
├── dingtalk.py          # DingTalk WebSocket
├── feishu.py            # Feishu/Lark WebSocket 或 webhook
├── wecom.py             # WeCom (企业微信) callback
├── weixin.py            # 微信 via iLink Bot API
├── bluebubbles.py       # Apple iMessage via BlueBubbles
├── qqbot/               # QQ Bot via Official API v2 (子包)
├── yuanbao.py           # 元宝 DM/group 适配器
├── feishu_comment.py    # Feishu 文档/云盘评论回复
├── msgraph_webhook.py   # Microsoft Graph 变更通知 webhook
├── webhook.py           # 入站/出站 webhook
├── api_server.py        # REST API server
└── homeassistant.py     # Home Assistant 对话集成
```

### 统一接口

- `connect()` / `disconnect()` — 生命周期管理
- `send_message()` — 出站消息投递
- `on_message()` — 入站消息规范化 → `MessageEvent`

### Token Locks

使用唯一凭证的适配器调用 `acquire_scoped_lock()` / `release_scoped_lock()`，**防止两个 profile 同时使用同一 bot token**。

---

## 投递路径

| 投递方式 | 说明 |
|---------|------|
| 直接回复 | 响应发回发起聊天的来源 |
| Home channel | cron 任务输出路由到配置的 home channel |
| 显式目标 | `send_message` 工具指定 `telegram:-1001234567890` |
| 跨平台 | 投递到与发起消息不同的平台 |

> Cron 任务投递**不会**镜像到 Gateway session 历史 — 只存在于自己的 cron session。这是防止消息交替规则违规的有意设计。

---

## Hooks

Gateway hooks 是响应生命周期事件的 Python 模块：

| 事件 | 触发时机 |
|------|---------|
| `gateway:startup` | Gateway 进程启动 |
| `session:start` | 新对话 session 开始 |
| `session:end` | Session 完成或超时 |
| `session:reset` | 用户 `/new` 重置 session |
| `agent:start` | Agent 开始处理消息 |
| `agent:step` | Agent 完成一个工具调用迭代 |
| `agent:end` | Agent 完成并返回响应 |
| `command:*` | 任意斜杠命令被执行 |

发现来源：`gateway/builtin_hooks/`（扩展点）+ `~/.hermes/hooks/`（用户安装）。每个 hook 包含 `HOOK.yaml` manifest + `handler.py`。

---

## Memory Provider 集成

```
AIAgent._invoke_tool()
  → self._memory_manager.handle_tool_call(name, args)
    → provider.handle_tool_call(name, args)
```

Session 结束/重置时：内置记忆刷写 → `on_session_end()` hook → 临时 AIAgent 运行记忆对话 → 上下文丢弃或归档。

---

## 后台维护

| 任务 | 说明 |
|------|------|
| Cron ticking | 检查任务调度并触发到期任务 |
| Session 过期 | 超时后清理被遗弃的 session |
| Memory flush | 在 session 过期前主动刷写记忆 |
| Cache refresh | 刷新模型列表和 provider 状态 |

---

## 进程管理

- `hermes gateway start` / `hermes gateway stop` — 手动控制
- Profile 作用域 PID 文件：`~/.hermes/gateway.pid`
- `hermes gateway stop --all` — 全局 `ps aux` 扫描杀死所有 gateway 进程

---

## 设计原则总结

1. **消息原子性**：接收→授权→Agent→投递在一个统一管道中完成
2. **两级守卫**：适配器级 + Runner 级双重检查确保并发安全
3. **Session 隔离**：结构化 session key 保证平台、用户、线程级别隔离
4. **安全默认**：多层授权以"默认拒绝"兜底
5. **适配器统一接口**：20+ 平台共享同一 connect/send/on_message 契约
