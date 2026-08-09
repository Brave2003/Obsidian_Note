# CodeGraph

> 一个本地优先的代码智能工具，将任何代码库转化为可查询的知识图谱，供 AI 编程代理使用。

安装: `npx @colbymchenry/codegraph`

## 核心定位

- **本地优先** — 数据存储在本地 SQLite，不依赖外部服务
- **MCP 服务器** — 通过 MCP 协议向 Claude Code、Cursor、Codex、opencode、Hermes、Gemini、Antigravity、Kiro 等工具暴露图谱
- **影响分析** — 在修改代码前追踪任何符号的调用者、被调用者和完整的影响范围
- **Tree-sitter 解析** — 基于真实 AST 的快速增量解析，覆盖 20+ 种语言，精确提取符号和边

---

## 工作原理

CodeGraph 通过四个阶段将源代码转化为可查询的图谱：

```
源文件 → 提取（tree-sitter）→ 数据库（节点/边/文件）
                        ↓
                  解析（import、名称匹配、框架模式）
                        ↓
                  图谱查询（调用者、被调用者、影响范围）
                        ↓
                  上下文构建（Markdown / JSON，供 AI 消费）
```

### 1. 提取（Extraction）

- **tree-sitter** 将源代码解析为 AST（抽象语法树）
- 语言特定的查询提取**节点**（函数、类、方法、类型等）和**边**（调用、导入、继承、实现等）
- 繁重的解析工作在**主线程之外**执行，不阻塞 UI

### 2. 存储（Storage）

- 所有数据存入本地 **SQLite** 数据库（`.codegraph/codegraph.db`）
- 启用 **FTS5** 全文搜索
- 使用 Node.js 内置的 `node:sqlite`，运行在 **WAL 模式**（Write-Ahead Logging），从打包的运行时中执行

### 3. 解析（Resolution）

提取完成后，进行引用解析：

- **函数调用 → 定义**：确定每次调用对应的具体函数定义
- **import → 源文件**：追踪导入语句指向的实际文件
- **类继承关系**：建立完整的继承链
- **框架特定模式**：通过**合成器（synthesizers）**桥接动态分发边界，包括：
  - 回调函数（callbacks）
  - 观察者模式（observers）
  - React 重渲染（re-render）
  - JSX 子组件（children）

> 合成器使得数据流能够端到端连接，即使跨越了动态分发的边界。详见「解析与框架」章节。

### 4. 自动同步（Auto-sync）

- MCP 服务器通过**原生操作系统文件事件**监听项目变化：
  - macOS: FSEvents
  - Linux: inotify
  - Windows: ReadDirectoryChangesW
- 文件变更经过**防抖（debounce）**处理，仅过滤源文件
- **增量同步** — 图谱随编码实时保持最新，**无需任何配置**

---

## 核心概念总结

| 概念 | 说明 |
|------|------|
| **节点（Nodes）** | 函数、类、方法、类型等代码符号 |
| **边（Edges）** | 调用、导入、继承、实现等关系 |
| **图谱查询** | 查询调用者、被调用者、影响范围 |
| **上下文构建** | 将查询结果转为 Markdown/JSON 供 AI 消费 |
| **合成器（Synthesizers）** | 桥接框架动态分发边界的组件 |
| **自动同步** | 零配置的文件监听 + 增量更新 |

---

## 参考

- 官方文档: https://colbymchenry.github.io/codegraph/
- 工作原理: https://colbymchenry.github.io/codegraph/core-concepts/how-it-works/
