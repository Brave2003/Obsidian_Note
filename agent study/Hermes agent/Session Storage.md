# Session Storage（Session 存储）

> Hermes Agent 使用 SQLite 数据库（`~/.hermes/state.db`）持久化 session 元数据、完整消息历史和模型配置。替代了早期按 session 的 JSONL 文件方案。

**核心文件**：`hermes_state.py`

---

## 架构总览

```
~/.hermes/state.db (SQLite, WAL mode)
├── sessions              — Session 元数据、token 计数、计费
├── messages              — 每个 session 的完整消息历史
├── messages_fts          — FTS5 虚拟表（content + tool_name + tool_calls）
├── messages_fts_trigram  — FTS5 虚拟表（trigram tokenizer，CJK/子串搜索）
├── state_meta            — Key/value 元数据表
└── schema_version        — 单行表，追踪迁移状态
```

### 关键设计决策

| 决策 | 原因 |
|------|------|
| WAL 模式 | 并发读 + 单写者（Gateway 多平台场景） |
| FTS5 全文搜索 | 跨所有 session 消息的快速文本搜索 |
| Session 血缘 | `parent_session_id` 链（压缩触发的分裂） |
| Source 标记 | `cli`、`telegram`、`discord` 等，支持平台过滤 |
| Batch/RL 独立 | Batch runner 和 RL 轨迹不存此处（独立系统） |

---

## SQLite Schema

### Sessions 表

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,
    user_id TEXT,
    model TEXT,
    model_config TEXT,
    system_prompt TEXT,
    parent_session_id TEXT,
    started_at REAL NOT NULL,
    ended_at REAL,
    end_reason TEXT,
    message_count INTEGER DEFAULT 0,
    tool_call_count INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,
    reasoning_tokens INTEGER DEFAULT 0,
    billing_provider TEXT,
    billing_base_url TEXT,
    billing_mode TEXT,
    estimated_cost_usd REAL,
    actual_cost_usd REAL,
    cost_status TEXT,
    cost_source TEXT,
    pricing_version TEXT,
    title TEXT,
    api_call_count INTEGER DEFAULT 0,
    FOREIGN KEY (parent_session_id) REFERENCES sessions(id)
);

CREATE INDEX IF NOT EXISTS idx_sessions_source ON sessions(source);
CREATE INDEX IF NOT EXISTS idx_sessions_parent ON sessions(parent_session_id);
CREATE INDEX IF NOT EXISTS idx_sessions_started ON sessions(started_at DESC);
CREATE UNIQUE INDEX IF NOT EXISTS idx_sessions_title_unique
    ON sessions(title) WHERE title IS NOT NULL;
```

### Messages 表

```sql
CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,          -- JSON 字符串
    tool_name TEXT,
    timestamp REAL NOT NULL,   -- Unix epoch float
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_content TEXT,
    reasoning_details TEXT,    -- JSON 字符串
    codex_reasoning_items TEXT,-- JSON 字符串
    codex_message_items TEXT   -- JSON 字符串
);

CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, timestamp);
```

### FTS5 全文搜索

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
);
```

通过三个触发器保持同步（INSERT/UPDATE/DELETE messages → 自动更新 FTS5）。

**设计原因 — 外部内容模式**：`content=messages` 让 FTS5 索引引用 messages 表而非复制内容，避免重复存储。代价是删除操作较慢（需要引用计数），但对于写少读多的 session 存储是合理的权衡。

---

## Schema 版本与迁移

当前版本：**11**

| 版本 | 变更 |
|------|------|
| 1 | 初始 schema（sessions、messages、FTS5） |
| 2 | 添加 `finish_reason` 列 |
| 3 | 添加 `title` 列 |
| 4 | 添加 title 唯一索引（NULL 允许，非 NULL 必须唯一） |
| 5 | 添加计费列：`cache_read_tokens`、`billing_*`、`*_cost_usd` 等 |
| 6 | 添加推理列：`reasoning`、`reasoning_details`、`codex_reasoning_items` |
| 7 | 添加 `reasoning_content` 列 |
| 8 | 添加 `api_call_count` 列 |
| 9 | 添加 `codex_message_items` 列 |
| 10 | 添加 `messages_fts_trigram` 虚拟表（trigram tokenizer） |
| 11 | 重索引 FTS 覆盖 `tool_name`+`tool_calls`，切换到内联模式 |

### 迁移策略

- **声明式列添加**：`_reconcile_columns()` 比对当前列与 `SCHEMA_SQL`，用 `ALTER TABLE ADD COLUMN` 补充缺失列（try/except 处理已存在的情况 — 幂等）
- **版本门控链**：数据迁移和索引/FTS 变更使用版本号链
- 每次成功迁移后递增版本号

---

## 写入竞争处理

多个 Hermes 进程（Gateway + CLI sessions + worktree agents）共享一个 `state.db`。`SessionDB` 通过以下机制处理写入竞争：

| 机制 | 配置 | 原因 |
|------|------|------|
| 短 SQLite 超时 | **1 秒**（而非默认 30s） | 避免长时间阻塞 |
| 应用级抖动重试 | 20-150ms 随机抖动，最多 **15 次** | 打破 convoy effect |
| BEGIN IMMEDIATE 事务 | 事务开始时暴露锁竞争 | 避免延迟到 COMMIT 才失败 |
| 周期性 WAL checkpoint | 每 **50** 次成功写入（PASSIVE 模式） | 控制 WAL 文件增长 |

```text
_WRITE_MAX_RETRIES = 15
_WRITE_RETRY_MIN_S = 0.020   # 20ms
_WRITE_RETRY_MAX_S = 0.150   # 150ms
_CHECKPOINT_EVERY_N_WRITES = 50
```

### 设计原因：为什么不用 SQLite 默认的 30 秒超时

SQLite 的默认 30 秒超时在低冲突场景下是合理的，但在多进程并发写入时会导致**车队效应**（convoy effect）：所有等待者按相同的确定性回退间隔重试，导致持续的锁竞争。短超时 + 应用级随机抖动打破了这种同步，让不同写入者错开重试时间。

---

## 常用操作

### 初始化

```python
from hermes_state import SessionDB
db = SessionDB()                               # 默认: ~/.hermes/state.db
db = SessionDB(db_path=Path("/tmp/test.db"))   # 自定义路径
```

### 创建和管理 Session

```python
# 创建新 session
db.create_session(
    session_id="sess_abc123",
    source="cli",
    model="anthropic/claude-sonnet-4.6",
    user_id="user_1",
    parent_session_id=None,  # 或前一个 session ID 形成血缘
)

# 结束 session
db.end_session("sess_abc123", end_reason="user_exit")

# 重新打开 session（清除 ended_at/end_reason）
db.reopen_session("sess_abc123")
```

### 存储消息

```python
msg_id = db.append_message(
    session_id="sess_abc123",
    role="assistant",
    content="Here's the answer...",
    tool_calls=[{"id": "call_1", "function": {"name": "terminal", "arguments": "{}"}}],
    token_count=150,
    finish_reason="stop",
    reasoning="Let me think about this...",
)
```

### 检索消息

```python
# 原始消息（含所有元数据）
messages = db.get_messages("sess_abc123")

# OpenAI 对话格式（用于 API 回放）
conversation = db.get_messages_as_conversation("sess_abc123")
# 返回: [{"role": "user", "content": "..."}, {"role": "assistant", ...}]
```

### Session Titles

```python
# 设置标题（非 NULL 标题必须唯一）
db.set_session_title("sess_abc123", "Fix Docker Build")

# 按标题解析（返回血缘中最近的）
session_id = db.resolve_session_by_title("Fix Docker Build")

# 自动生成血缘中的下一个标题
next_title = db.get_next_title_in_lineage("Fix Docker Build")
# 返回: "Fix Docker Build #2"
```

### 设计原因：标题唯一索引

非 NULL 标题必须唯一，这是为了支持 `/resume "Fix Docker Build"` 的精确匹配。同时允许 NULL（大多数 session 没有标题），唯一索引用 `WHERE title IS NOT NULL` 实现。

---

## 全文搜索

`search_messages()` 支持 FTS5 查询语法，自动净化用户输入。

### 基本搜索

```python
results = db.search_messages("docker deployment")
```

### FTS5 查询语法

| 语法 | 示例 | 含义 |
|------|------|------|
| 关键词 | `docker deployment` | 两个词（隐式 AND） |
| 引号短语 | `"exact phrase"` | 精确短语匹配 |
| 布尔 OR | `docker OR kubernetes` | 任一词 |
| 布尔 NOT | `python NOT java` | 排除词 |
| 前缀 | `deploy*` | 前缀匹配 |

### 过滤搜索

```python
# 仅搜索 CLI session
results = db.search_messages("error", source_filter=["cli"])

# 排除 gateway session
results = db.search_messages("bug", exclude_sources=["telegram", "discord"])

# 仅搜索 user 消息
results = db.search_messages("help", role_filter=["user"])
```

### 搜索结果格式

每条结果包含：
- `id`、`session_id`、`role`、`timestamp`
- `snippet` — FTS5 生成的片段，含 `>>>match<<<` 标记
- `context` — 匹配前后各 1 条消息（内容截断至 200 字符）
- `source`、`model`、`session_started` — 来自父 session

### `_sanitize_fts5_query()` 处理边界情况

- 移除未匹配的引号和特殊字符
- 连字符词加引号包裹（`chat-send` → `"chat-send"`）
- 移除悬空的布尔操作符（`hello AND` → `hello`）

---

## Session 血缘

Session 通过 `parent_session_id` 形成链。在 Gateway 中上下文压缩触发 session 分裂时发生。

### 查询：查找 Session 血缘

```sql
-- 查找所有祖先
WITH RECURSIVE lineage AS (
    SELECT * FROM sessions WHERE id = ?
    UNION ALL
    SELECT s.* FROM sessions s
    JOIN lineage l ON s.id = l.parent_session_id
)
SELECT id, title, started_at, parent_session_id FROM lineage;

-- 查找所有后代
WITH RECURSIVE descendants AS (
    SELECT * FROM sessions WHERE id = ?
    UNION ALL
    SELECT s.* FROM sessions s
    JOIN descendants d ON s.parent_session_id = d.id
)
SELECT id, title, started_at FROM descendants;
```

### 查询：Token 使用统计

```sql
-- 按模型统计 token
SELECT model,
       COUNT(*) as session_count,
       SUM(input_tokens) as total_input,
       SUM(output_tokens) as total_output,
       SUM(estimated_cost_usd) as total_cost
FROM sessions
WHERE model IS NOT NULL
GROUP BY model
ORDER BY total_cost DESC;

-- Token 使用最高的 session
SELECT id, title, model,
       input_tokens + output_tokens AS total_tokens,
       estimated_cost_usd
FROM sessions
ORDER BY total_tokens DESC LIMIT 10;
```

---

## 导出与清理

```python
# 导出单个 session（含消息）
data = db.export_session("sess_abc123")

# 导出所有 session（含消息）
all_data = db.export_all(source="cli")

# 删除旧 session（仅已结束的）
deleted_count = db.prune_sessions(older_than_days=90)
deleted_count = db.prune_sessions(older_than_days=30, source="telegram")

# 清除消息但保留 session 记录
db.clear_messages("sess_abc123")

# 删除 session 和所有消息
db.delete_session("sess_abc123")
```

---

## 数据库位置

默认路径：`~/.hermes/state.db`

通过 `hermes_constants.get_hermes_home()` 解析，默认为 `~/.hermes/` 或 `HERMES_HOME` 环境变量值。

数据库文件、WAL 文件（`state.db-wal`）和共享内存文件（`state.db-shm`）都在同一目录。
