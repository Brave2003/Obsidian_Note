# Provider Runtime Resolution（Provider 运行时解析）

> Hermes 有一个共享的 provider 运行时解析器，被 CLI、Gateway、Cron、ACP 和 auxiliary 模型调用共同使用。

**核心文件**：`hermes_cli/runtime_provider.py`、`hermes_cli/auth.py`、`hermes_cli/model_switch.py`、`agent/auxiliary_client.py`、`providers/`、`plugins/model-providers/`

---

## 解析优先级

```text
1. 显式 CLI/Runtime 请求         ← 最高优先级
2. config.yaml 模型/Provider 配置
3. 环境变量
4. Provider 特定默认值或自动解析   ← 最低优先级
```

**设计原因**：Hermes 将保存的模型/provider 选择作为正常运行的真值源（source of truth）。这防止了**过期的 shell 环境变量**静默覆盖用户在 `hermes model` 中最后选择的端点。

---

## Provider 体系

当前 provider 家族（参见 `plugins/model-providers/` 获取完整列表）：

- OpenRouter
- Nous Portal
- OpenAI Codex
- Copilot / Copilot ACP
- Anthropic（原生）
- Google / Gemini
- Alibaba / DashScope
- DeepSeek
- Z.AI
- Kimi / Moonshot
- MiniMax
- Kilo Code
- Hugging Face
- OpenCode Zen / OpenCode Go
- AWS Bedrock
- Azure Foundry
- NVIDIA NIM
- xAI (Grok)
- Arcee
- GMI Cloud
- StepFun
- Qwen OAuth
- Xiaomi
- Ollama Cloud
- LM Studio
- Tencent TokenHub
- Custom（一等公民，任何 OpenAI 兼容端点）
- Named custom providers（`custom_providers` 列表）

### Provider Plugin 机制

`get_provider_profile()` 在 `providers/` 中返回给定 provider id 的 `ProviderProfile`。`runtime_provider.py` 在解析时调用它获取规范的 `base_url`、`env_vars` 优先级列表、`api_mode` 和 `fallback_models`。

添加新 provider 只需在 `plugins/model-providers/<name>/`（或 `$HERMES_HOME/plugins/model-providers/<name>/`）下创建插件，调用 `register_provider()` 即可 — **不需要在解析器本身添加分支**。用户插件可以覆盖同名的内置插件。

---

## 运行时解析输出

解析器返回：

- `provider` — provider 标识
- `api_mode` — chat_completions / codex_responses / anthropic_messages
- `base_url` — API 端点
- `api_key` — 凭证
- `source` — 凭证来源
- provider 特定元数据（过期/刷新信息等）

---

## OpenRouter 与自定义 OpenAI 兼容端点

Hermes 包含逻辑避免在多个 provider key 存在时将错误的 API key 泄露给自定义端点：

- `OPENROUTER_API_KEY` → 只发送到 `openrouter.ai` 端点
- `OPENAI_API_KEY` → 用于自定义端点和作为回退

Hermes 区分：
- 用户选择的**真实自定义端点**
- 未配置自定义端点时的 OpenRouter 回退路径

这对本地模型服务器、非 OpenRouter 的 OpenAI 兼容 API、无需重运行 setup 切换 provider 等场景尤为重要。

---

## 原生 Anthropic 路径

当 provider 解析选择 `anthropic` 时：

- `api_mode = anthropic_messages`
- 使用原生 Anthropic Messages API
- `agent/anthropic_adapter.py` 进行格式转换

凭证解析偏好：
1. **可刷新的 Claude Code 凭证**优先于复制的 env token
2. 手动 `ANTHROPIC_TOKEN` / `CLAUDE_CODE_OAUTH_TOKEN` 仍可作为显式覆盖
3. 原生 Messages API 调用前预刷新 Anthropic 凭证
4. 401 后重建 Anthropic client 重试一次作为回退

---

## OpenAI Codex 路径

- `api_mode = codex_responses`
- 专用凭证解析和 auth store 支持
- 使用独立的 Responses API 路径

---

## Auxiliary 模型路由

Auxiliary 任务可使用自己的 provider/model 路由：

| 任务 | 说明 |
|------|------|
| Vision | 图像理解 |
| Web extraction | 网页提取摘要 |
| Context compression | 上下文压缩摘要 |
| Skills hub | Skills 操作 |
| MCP helper | MCP 辅助操作 |
| Memory flushes | 记忆刷写 |

当 auxiliary 任务配置为 `provider: main` 时，Hermes 通过相同的共享运行时路径解析。这意味着 env 驱动的自定义端点仍然有效，通过 `hermes model`/`config.yaml` 保存的自定义端点也有效。

---

## Fallback Models

支持配置的回退 provider 链 — 一组按序尝试的 `(provider, model)` 条目。

### 内部工作原理

**存储**：`AIAgent.__init__` 存储 `fallback_model` dict，设置 `_fallback_activated = False`

**触发点**（`run_agent.py` 主重试循环中的三处）：

1. 无效 API 响应达到最大重试（None choices、missing content）
2. 不可重试的客户端错误（HTTP 401、403、404）
3. 临时错误达到最大重试（HTTP 429、500、502、503）

**激活流程**（`_try_activate_fallback`）：

- 已激活或未配置 → 直接返回 `False`
- 调用 `resolve_provider_client()` 构建新 client
- 确定 `api_mode`：openai-codex → `codex_responses`，anthropic → `anthropic_messages`，其余 → `chat_completions`
- 原地交换：`self.model`、`self.provider`、`self.base_url`、`self.api_mode`、`self.client`、`self._client_kwargs`
- Anthropic fallback：构建原生 Anthropic client 而非 OpenAI 兼容
- 重新评估 prompt caching
- 设置 `_fallback_activated = True`，重置重试计数

**配置流**：
- CLI：`cli.py` 读取 `CLI_CONFIG["fallback_model"]` → `AIAgent(fallback_model=...)`
- Gateway：`gateway/run.py._load_fallback_model()` 读取 `config.yaml` → `AIAgent`
- 验证：`provider` 和 `model` 键必须都非空，否则 fallback 被禁用

### 不支持 Fallback 的场景

| 场景 | 原因 |
|------|------|
| 子 agent 委托 | 子 agent 继承父 provider 但不继承 fallback 配置 |
| Auxiliary 任务 | 使用自己独立的 provider 自动检测链 |

> Cron jobs **支持** fallback：`run_job()` 从 `config.yaml` 读取 `fallback_providers` 并传递给 `AIAgent`。

### 测试覆盖

- `tests/run_agent/test_fallback_credential_isolation.py` — 主备凭证隔离
- `tests/hermes_cli/test_fallback_cmd.py` — `/fallback` CLI 命令
- `tests/gateway/test_fallback_eviction.py` — Gateway 淘汰失败 provider

---

## 关键源文件

| 文件 | 用途 |
|------|------|
| `hermes_cli/runtime_provider.py` | 凭证解析、`_resolve_custom_runtime()` |
| `hermes_cli/auth.py` | Provider 注册表、`resolve_provider()` |
| `hermes_cli/model_switch.py` | 共享 `/model` 切换管道（CLI + Gateway） |
| `agent/auxiliary_client.py` | Auxiliary 模型路由 |
| `providers/` | ABC + 注册表入口点 |
| `plugins/model-providers/` | 按 provider 的插件 |
