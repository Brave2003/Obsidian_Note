> [!NOTE] 链路
> 从模型到 Agent 到部署到自我改进

---

## 官方系统总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Entry Points                                  │
│                                                                      │
│  CLI (cli.py)    Gateway (gateway/run.py)    ACP (acp_adapter/)     │
│  Batch Runner    API Server                  Python Library          │
└──────────┬──────────────┬───────────────────────┬───────────────────┘
           │              │                       │
           ▼              ▼                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     AIAgent (run_agent.py)                          │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Prompt       │  │ Provider     │  │ Tool         │               │
│  │ Builder      │  │ Resolution   │  │ Dispatch     │               │
│  │ (prompt_     │  │ (runtime_    │  │ (model_      │               │
│  │  builder.py) │  │  provider.py)│  │  tools.py)   │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                 │                       │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐               │
│  │ Compression  │  │ 3 API Modes  │  │ Tool Registry│               │
│  │ & Caching    │  │ chat_compl.  │  │ (registry.py)│               │
│  │              │  │ codex_resp.  │  │ 70+ tools    │               │
│  │              │  │ anthropic    │  │ 28 toolsets  │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
└─────────┴─────────────────┴─────────────────┴───────────────────────┘
           │                                    │
           ▼                                    ▼
┌───────────────────┐              ┌──────────────────────┐
│ Session Storage   │              │ Tool Backends         │
│ (SQLite + FTS5)   │              │ Terminal (6 backends) │
│ hermes_state.py   │              │ Browser (5 backends)  │
│ gateway/session.py│              │ Web (4 backends)      │
└───────────────────┘              │ MCP (dynamic)         │
                                   │ File, Vision, etc.    │
                                   └──────────────────────┘
```

---

## 目录结构

```
hermes-agent/
├── run_agent.py              # AIAgent — 核心对话循环（大文件）
├── cli.py                    # HermesCLI — 交互式终端 UI（大文件）
├── model_tools.py            # 工具发现、schema 收集、分发
├── toolsets.py               # 工具分组与平台预设
├── hermes_state.py           # SQLite session/state 数据库（FTS5）
├── hermes_constants.py       # HERMES_HOME、profile 感知路径
├── batch_runner.py           # 批量轨迹生成
│
├── agent/                    # Agent 内部子系统
│   ├── prompt_builder.py     # 系统 prompt 组装
│   ├── context_engine.py     # ContextEngine ABC（可插拔）
│   ├── context_compressor.py # 默认引擎 — 有损摘要压缩
│   ├── prompt_caching.py     # Anthropic prompt 缓存
│   ├── auxiliary_client.py   # 辅助 LLM 客户端（视觉、摘要等）
│   ├── model_metadata.py     # 模型上下文长度、token 估算
│   ├── models_dev.py         # models.dev 注册表集成
│   ├── anthropic_adapter.py  # Anthropic Messages API 格式转换
│   ├── display.py            # KawaiiSpinner、工具预览格式化
│   ├── skill_commands.py     # Skill 斜杠命令
│   ├── memory_manager.py     # Memory Manager 编排
│   ├── memory_provider.py    # Memory Provider ABC
│   └── trajectory.py         # 轨迹保存辅助
│
├── hermes_cli/               # CLI 子命令与设置
│   ├── main.py               # 入口 — 所有 `hermes` 子命令
│   ├── config.py             # DEFAULT_CONFIG、OPTIONAL_ENV_VARS、迁移
│   ├── commands.py           # COMMAND_REGISTRY — 斜杠命令定义
│   ├── auth.py               # PROVIDER_REGISTRY、凭证解析
│   ├── runtime_provider.py   # Provider → api_mode + 凭证
│   ├── models.py             # 模型目录、provider 模型列表
│   ├── model_switch.py       # /model 命令逻辑（CLI + Gateway 共享）
│   ├── setup.py              # 交互式设置向导
│   ├── skin_engine.py        # CLI 主题引擎
│   ├── skills_config.py      # hermes skills — 按平台启用/禁用
│   ├── skills_hub.py         # /skills 斜杠命令
│   ├── tools_config.py       # hermes tools — 按平台启用/禁用
│   ├── plugins.py            # PluginManager — 发现、加载、hooks
│   ├── callbacks.py          # 终端回调（clarify、sudo、approval）
│   └── gateway.py            # hermes gateway start/stop
│
├── tools/                    # 工具实现（每个工具一个文件）
│   ├── registry.py           # 中央工具注册表
│   ├── approval.py           # 危险命令检测
│   ├── terminal_tool.py      # 终端编排
│   ├── process_registry.py   # 后台进程管理
│   ├── file_tools.py         # read_file、write_file、patch、search_files
│   ├── web_tools.py          # web_search、web_extract
│   ├── browser_tool.py       # 10 个浏览器自动化工具
│   ├── code_execution_tool.py # execute_code 沙箱
│   ├── delegate_tool.py      # 子 Agent 委托
│   ├── mcp_tool.py           # MCP 客户端（大文件）
│   ├── credential_files.py   # 基于文件的凭证透传
│   ├── env_passthrough.py    # 沙箱环境变量透传
│   ├── ansi_strip.py         # ANSI 转义剥离
│   └── environments/         # 终端后端（local、docker、ssh、modal、daytona、singularity）
│
├── gateway/                  # 消息平台网关
│   ├── run.py                # GatewayRunner — 消息分发（大文件）
│   ├── session.py            # SessionStore — 对话持久化
│   ├── delivery.py           # 出站消息投递
│   ├── pairing.py            # DM 配对授权
│   ├── hooks.py              # Hook 发现与生命周期事件
│   ├── mirror.py             # 跨 session 消息镜像
│   ├── status.py             # Token 锁、profile 作用域进程追踪
│   ├── builtin_hooks/        # 始终注册的 hooks 扩展点
│   └── platforms/            # 20 个适配器：
│                             #   telegram, discord, slack, whatsapp,
│                             #   signal, matrix, mattermost, email, sms,
│                             #   dingtalk, feishu, wecom, wecom_callback, weixin,
│                             #   bluebubbles, qqbot, homeassistant, webhook,
│                             #   api_server, yuanbao
│
├── acp_adapter/              # ACP 服务器（VS Code / Zed / JetBrains）
├── cron/                     # 调度器（jobs.py、scheduler.py）
├── plugins/memory/           # Memory Provider 插件
├── plugins/context_engine/   # Context Engine 插件
├── skills/                   # 内置 skills（始终可用）
├── optional-skills/          # 官方可选 skills（需显式安装）
├── website/                  # Docusaurus 文档站点
└── tests/                    # Pytest 测试套件（~25,000 测试 / ~1,250 文件）
```

---

## 数据流

### CLI Session

```
User input → HermesCLI.process_input()
  → AIAgent.run_conversation()
    → prompt_builder.build_system_prompt()
    → runtime_provider.resolve_runtime_provider()
    → API call (chat_completions / codex_responses / anthropic_messages)
    → tool_calls? → model_tools.handle_function_call() → loop
    → final response → display → save to SessionDB
```

### Gateway Message

```
Platform event → Adapter.on_message() → MessageEvent
  → GatewayRunner._handle_message()
    → authorize user
    → resolve session key
    → create AIAgent with session history
    → AIAgent.run_conversation()
    → deliver response back through adapter
```

### Cron Job

```
Scheduler tick → load due jobs from jobs.json
  → create fresh AIAgent (no history)
  → inject attached skills as context
  → run job prompt
  → deliver response to target platform
  → update job state and next_run
```

---

## run_agent.py 内部结构全景

`run_agent.py` 是 Hermes 的核心文件（~9431 行），围绕主循环形成**辐射状架构**，分为六大区域：

```
辅助类 (1-415 行)
├── _SafeWriter          — broken pipe 防护
├── IterationBudget      — 线程安全迭代计数器
└── _should_parallelize  — 并行安全判断

AIAgent.__init__ (416-1140 行)
├── 阶段1-3: 配置 + API模式 + 回调
├── 阶段4:   LLM 客户端构造 (Anthropic / OpenAI / Codex)
├── 阶段5-6: 工具加载 + 记忆初始化
└── 阶段7:   压缩器初始化

内部方法 (1140-6800 行)
├── _build_system_prompt       — 系统提示词拼装
├── _sanitize_api_messages     — 消息清洗与修复
├── _interruptible_streaming_api_call — 流式 API + 健康检测
├── _build_api_kwargs          — API 请求构建
├── _compress_context          — 上下文压缩
├── _persist_session           — Session 持久化
├── _save_trajectory           — 轨迹保存
├── _spawn_background_review   — 后台记忆/技能审查
└── flush_memories             — 记忆刷盘

工具执行 (5930-6590 行)
├── _execute_tool_calls           — 入口：并行/串行判断
├── _invoke_tool                  — 四路分发
├── _execute_tool_calls_concurrent — ThreadPoolExecutor
└── _execute_tool_calls_sequential — 逐个执行 + 显示

run_conversation 主循环 (6800-9431 行)
├── 准备阶段: SafeWriter + 清洗 + prompt + 预飞行压缩 + hooks + memory prefetch
├── 主循环:   while budget > 0: 组装消息 → API调用 → 解析响应 → 工具执行
└── 退出收尾: persist + sync + hooks + background review
```

---

## AIAgent 初始化 — 七个阶段

`AIAgent.__init__()` 有 **45+ 个参数**，初始化代码超过 700 行。每个参数和每段初始化代码都对应一个具体的工程问题：

| 阶段 | 内容 | 解决的问题 |
|------|------|-----------|
| 阶段1 | 配置注入与默认值合并 | 多入口（CLI/Gateway/Cron/ACP）如何统一配置 |
| 阶段2 | API 模式检测与路由 | 自动识别 chat_completions / codex_responses / anthropic_messages |
| 阶段3 | 回调注册（8 个回调） | 编排层与入口层解耦 |
| 阶段4 | LLM 客户端构造 | 根据 provider + api_mode 构造对应的 SDK client |
| 阶段5 | 工具加载与过滤 | enabled_toolsets / disabled_toolsets 控制工具集合 |
| 阶段6 | 记忆系统初始化 | MemoryManager 选择后端并预加载 |
| 阶段7 | 上下文压缩器初始化 | 选择压缩策略，配置 protect_last_n |

> **设计原则：初始化即决策** — 50+ 个参数在构造时就确定了 API 模式、prompt caching 策略、工具集合、记忆后端等关键决策。这些决策在整个 `run_conversation()` 生命周期中不会改变（除非 fallback 触发），让编排逻辑可以安全地做出假设。

---

## 三种 API 执行模式

Hermes 支持三种 API 模式，通过 **显式参数 → provider 检测 → base URL 启发式 → 默认值** 四级优先级确定：

| API 模式 | 用途 | 客户端类型 | 内部消息格式 |
|----------|------|-----------|-------------|
| `chat_completions` | OpenAI 兼容端点（OpenRouter、自定义 provider） | `openai.OpenAI` | OpenAI 格式原生 |
| `codex_responses` | OpenAI Codex / Responses API | `openai.OpenAI` + Responses 格式 | 转换为 OpenAI 格式 |
| `anthropic_messages` | 原生 Anthropic Messages API | `anthropic.Anthropic` 适配器 | 通过 adapter 转换为 OpenAI 格式 |

> **核心设计**：三种模式在 API 调用前后**都收敛到相同的内部消息格式**（OpenAI 风格的 `role`/`content`/`tool_calls` dict），上层编排逻辑无需关心底层 provider 差异。

---

## 单轮迭代生命周期

Agent loop 的每次迭代按以下顺序执行（`run_conversation()`）：

```text
1. 若未提供则生成 task_id
2. 将用户消息追加到对话历史
3. 构建或复用已缓存的系统 prompt（prompt_builder.py）
4. 检查是否需要预检压缩（上下文超过 50%）
5. 从对话历史构建 API 消息
   - chat_completions：直接使用 OpenAI 格式
   - codex_responses：转换为 Responses API 输入项
   - anthropic_messages：通过 anthropic_adapter.py 转换
6. 注入临时 prompt 层（预算警告、上下文压力提示）
7. 若使用 Anthropic，应用 prompt 缓存标记
8. 发起可中断的 API 调用（_interruptible_api_call）
9. 解析响应：
   - 若有 tool_calls → 执行工具，追加结果，回到步骤 5
   - 若为文本响应 → 持久化 session，按需刷写内存，返回
```

### 消息角色交替规则

循环强制执行严格的消息角色交替：

- 系统消息之后：`User → Assistant → User → Assistant → ...`
- 工具调用期间：`Assistant (含 tool_calls) → Tool → Tool → ... → Assistant`
- ❌ 不允许连续两条 assistant 消息
- ❌ 不允许连续两条 user 消息
- ✅ 只有 `tool` 角色可以连续出现（并行工具结果）

---

## 可中断的 API 调用

API 请求封装在 `_interruptible_api_call()` 中，在后台线程执行 HTTP 调用，同时监听中断事件：

- 用户发送新消息、`/stop` 命令或信号 → API 线程被**放弃**（响应丢弃）
- Agent 可以处理新输入或干净关闭
- **不会将部分响应注入对话历史**（保证状态一致性）

---

## 工具系统

### 注册机制

工具通过 **自注册** 模式加入系统。每个 `tools/*.py` 文件在 import 时调用 `registry.register()`，无需手动维护导入列表。

```
tools/registry.py  (零依赖 — 被所有工具文件导入)
      ↑
tools/*.py  (每个文件在 import 时调用 registry.register())
      ↑
model_tools.py  (导入 tools/registry + 触发工具发现)
      ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### 规模

- **70+** 注册工具
- **28** 个 toolsets
- **6** 种终端后端（local、Docker、SSH、Daytona、Modal、Singularity）
- **5** 种浏览器后端
- **4** 种 Web 后端
- MCP 工具动态注册

### 顺序执行 vs 并发执行

- 多个工具调用 → 通过 `ThreadPoolExecutor` **并发执行**
- 例外：标记为交互式的工具（如 `clarify`）强制**顺序执行**
- 结果按**原始工具调用顺序**重新插入，而非完成顺序

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

### Agent 级工具（被 run_agent.py 拦截，不经过 registry）

| 工具 | 拦截原因 |
|------|---------|
| `todo` | 读写 agent 本地任务状态 |
| `memory` | 向持久化内存文件写入内容（有字符限制） |
| `session_search` | 通过 agent 的 session DB 查询 session 历史 |
| `delegate_task` | 以隔离上下文生成子 agent |

---

## 回调系统

回调的设计原则：**编排层决定"什么时候"调用，入口层决定"怎么处理"**。同一个编排逻辑零修改适配所有入口。

| 回调 | 触发时机 | 使用方 |
|------|---------|--------|
| `tool_progress_callback` | 每次工具执行前后 | CLI spinner、gateway 进度消息 |
| `thinking_callback` | 模型开始/停止思考时 | CLI "thinking..." 指示器 |
| `reasoning_callback` | 模型返回推理内容时 | CLI 推理展示、gateway 推理块 |
| `clarify_callback` | `clarify` 工具被调用时 | CLI 输入提示、gateway 交互消息 |
| `step_callback` | 每次完整 agent 轮次结束后 | Gateway 步骤追踪、ACP 进度 |
| `stream_delta_callback` | 每个流式 token（启用时） | CLI 流式展示 |
| `tool_gen_callback` | 从流中解析出工具调用时 | CLI spinner 中的工具预览 |
| `status_callback` | 状态变更时（思考、执行等） | ACP 状态更新 |

---

## 预算与回退行为

### IterationBudget（防止 Agent 失控）

- 默认 **90 次迭代**（可通过 `agent.max_turns` 配置）
- 每个 agent 拥有独立预算
- 子 agent 预算上限为 `delegation.max_iterations`（默认 50）
- 父 agent + 子 agent 总迭代次数**可以超过**父 agent 上限
- `execute_code` 的轮次**不计入**预算
- 达到 100% 时，agent 停止并返回已完成工作的摘要

### 预算压力预警

当 agent 接近 `max_iterations` 时，压力警告**注入到工具结果 JSON 中**（而非作为独立消息），推动模型尽快收尾：
- **70%**: 温和提醒
- **90%**: 强烈警告

### Fallback Provider 链

当主模型失败（429 限流、5xx 错误、401/403 鉴权错误）：

1. 检查配置中的 `fallback_providers` 列表
2. 按顺序尝试每个回退 provider
3. 成功后，使用新 provider 继续对话
4. 401/403 时，先尝试刷新凭据再故障转移

辅助任务（视觉、压缩、网页提取）也各自拥有独立的回退链，通过 `auxiliary.*` 配置。

---

## Prompt 系统

System prompt 按 **三层架构** 组装（`system_prompt.py` + `prompt_builder.py`）：

| 层 | 内容 | 稳定性 |
|----|------|--------|
| `stable` | 身份定义、工具指导、skills | session 内不变，可缓存 |
| `context` | 上下文文件（SOUL.md、AGENTS.md 等） | session 内不变 |
| `volatile` | 记忆、用户画像、时间戳 | session 内不变 |

**关键细节**：
- System prompt 在一个 session 内**缓存**，只有压缩等事件后才重建，目的是最大化 **prefix cache 命中**
- `ephemeral_system_prompt`（预算警告、上下文压力）**不在** `_build_system_prompt()` 里拼，只在 API-call time 注入，不进入缓存和持久化
- 技术模式：**session-scoped immutable system prompt snapshot + API-call-time ephemeral injection**

---

## 上下文压缩与会话持久化

### 压缩触发条件

- **预检压缩**（API 调用前）：对话超过模型上下文窗口的 **50%**
- **Gateway 自动压缩**：对话超过 **85%**（更激进，在轮次之间运行）

### 压缩流程

```text
1. 先将内存刷写到磁盘（防止数据丢失）
2. 将中间对话轮次摘要为紧凑的摘要内容
3. 保留最后 N 条消息完整不变（compression.protect_last_n，默认 20）
4. 工具调用/结果消息对保持完整（不拆分）
5. 生成新的 session 血缘 ID（压缩创建"子" session）
```

### Session 持久化

- 消息保存到 session 存储（SQLite via `hermes_state.py`）
- SQLite 使用 **WAL 模式** + **FTS5 全文搜索** + 抖动重试 + 被动 checkpoint
- 内存变更刷写到 `MEMORY.md` / `USER.md`
- 可通过 `/resume` 或 `hermes chat --resume` 恢复 session
- 压缩通过 `parent_session_id` 链式关联，形成 **session 血缘链**

---

## 插件系统

三种发现来源：

| 来源 | 路径 | 作用域 |
|------|------|--------|
| 用户级 | `~/.hermes/plugins/` | 全局 |
| 项目级 | `.hermes/plugins/` | 当前项目 |
| pip entry points | `hermes_plugins` 入口点 | 系统级 |

插件通过 context API 注册 **tools、hooks、CLI commands**。

两种专用插件类型：
- **Memory Provider**（`plugins/memory/`）— 单例选择，一次只能激活一个
- **Context Engine**（`plugins/context_engine/`）— 单例选择，一次只能激活一个

通过 `hermes plugins` 或 `config.yaml` 配置。

---

## 多入口收敛架构

```
CLI (cli.py) ────┐
Gateway (20平台) ─┤
Cron ─────────────┼──→ AIAgent.run_conversation()  ← 共享主循环
ACP ──────────────┤
API Server ───────┤
Batch Runner ─────┤
Delegation ───────┘
```

> **核心设计**：多入口收敛到一个 Agent loop。所有入口最终都把输入整理成消息、配置、工具集合和会话状态，交给同一个主循环执行。这让所有入口共享能力 — 同样的 provider fallback、工具执行、memory、compression 和持久化语义。

**其他入口**：
- **ACP**（Agent Communication Protocol）：通过 stdio/JSON-RPC 将 Hermes 暴露为编辑器原生 agent（VS Code、Zed、JetBrains）
- **Cron**：一等公民的 agent 任务（非 shell 任务），支持多种调度格式、可附加 skills 和脚本、可投递到任意平台
- **Trajectories**：生成 ShareGPT 格式的训练数据轨迹

---

## 设计原则

| 原则 | 实践含义 |
|------|---------|
| **Prompt 稳定性** | System prompt 在会话中间不变。除显式用户操作（`/model`）外，不会发生破坏缓存的变更。 |
| **可观察执行** | 每个工具调用都通过回调对用户可见。CLI（spinner）和 Gateway（聊天消息）均有进度更新。 |
| **可中断** | API 调用和工具执行可以通过用户输入或信号中途取消。 |
| **平台无关核心** | 一个 AIAgent 类服务 CLI、Gateway、ACP、Batch 和 API Server。平台差异存在于入口点，而非 agent 内部。 |
| **松耦合** | 可选子系统（MCP、插件、Memory Provider、RL 环境）使用注册表模式和 check_fn 门控，非硬依赖。 |
| **Profile 隔离** | 每个 profile（`hermes -p <name>`）拥有独立的 HERMES_HOME、配置、记忆、会话和 Gateway PID。多个 profile 可同时运行。 |

---

## 设计权衡 — 为什么 run_agent.py 有 9431 行不拆分

1. **初始化即决策**：构造时确定 API 模式、prompt caching、工具集合、记忆后端，整个生命周期不改变，编排逻辑可以安全假设
2. **回调是适配层的接口**：8 个回调让编排层和入口层保持单向依赖 — 编排层不关心自己运行在 CLI 还是 Telegram 还是 Cron
3. **Budget 是安全网**：IterationBudget + 压力预警 + background review 让 agent 既有自主性（90次迭代），又有收敛机制（70%/90% 预警），还有自我改进能力
4. **有状态内聚优于无状态拆分**：9431 行的 `run_agent.py` 不是设计缺陷而是权衡结果 — 当 80% 的方法需要深度访问实例状态时，强行模块化带来的不是解耦而是间接性。策略是"能拆的拆出去（`agent/` 下 25 个模块），剩下的保持内聚"

---

## 关键源文件速查

| 文件 | 用途 |
|------|------|
| `run_agent.py` | **AIAgent 类** — 完整的 agent loop，~9431 行 |
| `agent/prompt_builder.py` | 系统 prompt 组装：从 memory、skills、context files、personality 拼接 |
| `agent/context_engine.py` | ContextEngine ABC — 可插拔的上下文管理 |
| `agent/context_compressor.py` | 默认引擎 — 有损摘要压缩算法 |
| `agent/prompt_caching.py` | Anthropic prompt 缓存标记和缓存命中率统计 |
| `agent/auxiliary_client.py` | 辅助 LLM 客户端，处理视觉、摘要等辅助任务 |
| `agent/anthropic_adapter.py` | Anthropic Messages API → 内部 OpenAI 格式转换 |
| `agent/memory_manager.py` | Memory Manager 编排 — 插件式后端管理 |
| `model_tools.py` | 工具 schema 收集、`handle_function_call()` 分发 |
| `tools/registry.py` | ToolRegistry 单例 — 所有工具的中央注册表 |
| `hermes_state.py` | SessionDB — SQLite + FTS5 全文搜索 session 存储 |
| `hermes_cli/runtime_provider.py` | Provider → api_mode + 凭证解析 |
| `cli.py` | HermesCLI 类 — 交互式 CLI 编排器 |
| `gateway/run.py` | GatewayRunner — 20 平台消息网关主循环 |
| `gateway/session.py` | SessionStore — 跨平台的用户会话持久化 |

### 推荐阅读顺序

1. 本页 — 全局定位
2. Agent Loop Internals — AIAgent 如何工作
3. Prompt Assembly — 系统 prompt 构建
4. Provider Runtime Resolution — provider 选择机制
5. Adding Providers — 添加新 provider 实践指南
6. Tools Runtime — 工具注册表、分发、环境
7. Session Storage — SQLite schema、FTS5、session 血缘
8. Gateway Internals — 消息平台网关
9. Context Compression & Prompt Caching — 压缩与缓存
10. ACP Internals — IDE 集成

### 按目标阅读

| 想理解什么 | 先读 | 再读 |
|-----------|------|------|
| CLI 如何启动 | `hermes_cli/main.py:cmd_chat()` | `cli.py:HermesCLI.chat()` |
| 主循环 | `run_agent.py:class AIAgent` | `_build_system_prompt()`, `run_conversation()`, `_execute_tool_calls*()` |
| 工具系统 | `tools/registry.py` | `model_tools.py`, `tools/*.py` |
| 记忆系统 | `agent/memory_manager.py` | `tools/memory_tool.py` |
| 上下文压缩 | `agent/context_compressor.py` | `run_agent.py:_compress_context()` |
| Gateway | `gateway/run.py:GatewayRunner` | `gateway/session.py`, `gateway/platforms/` |
| Provider 系统 | `hermes_cli/runtime_provider.py` | `hermes_cli/auth.py`, `agent/anthropic_adapter.py` |
