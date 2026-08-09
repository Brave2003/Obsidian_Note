# OpenSpec — AI 原生规范驱动开发系统

> 一个轻量级的规范驱动开发（Spec-Driven Development, SDD）框架，为 AI 编码助手提供规范层，让开发过程更可预测、更高效。

- **GitHub**: https://github.com/Fission-AI/OpenSpec
- **官网**: https://openspec.dev/
- **Stars**: 56k+
- **协议**: MIT
- **语言**: TypeScript (99.2%)
- **安装**: `npm install -g @fission-ai/openspec@latest`
- **环境要求**: Node.js 20.19.0+

---

## 设计哲学

```
→ 流动而非僵化 (fluid not rigid)
→ 迭代而非瀑布 (iterative not waterfall)
→ 简单而非复杂 (easy not complex)
→ 存量项目优先 (built for brownfield not just greenfield)
→ 从个人项目扩展到企业级 (scalable from personal projects to enterprises)
```

---

## 核心定位

AI 编码助手功能强大，但当需求只存在于聊天记录中时，结果往往难以预测。OpenSpec 在编写第一行代码之前，先与 AI 对齐需求，让开发过程更可预测、更高效。

### 解决的核心问题

| 问题 | OpenSpec 的解决方式 |
|------|---------------------|
| 需求只在聊天记录中 | 需求以结构化文件夹持久化到项目中 |
| AI "失忆"（上下文丢失） | Spec 作为项目的单一事实来源（Single Source of Truth） |
| 无法追溯变更历史 | 每个变更独立文件夹 + 归档机制 |
| 人与 AI 理解偏差 | 提案 → 审查 → 对齐 → 实施，先对齐再编码 |
| 工具锁定 | 支持 25+ AI 编码助手 |

---

## 快速开始

### 环境要求

- Node.js 20.19.0 或更高版本
- 支持的包管理器：npm、pnpm、yarn、bun、nix

### 安装与初始化

```bash
# 第一步：全局安装
npm install -g @fission-ai/openspec@latest

# 第二步：在项目中初始化
cd your-project
openspec init
```

初始化后生成：

```
your-project/
└── openspec/
    ├── specs/        # 当前系统规范（Source of Truth）
    ├── changes/      # 变更提案目录
    │   └── archive/  # 已归档的历史变更
    └── project.md    # 项目上下文说明
```

### 第一个五分钟：完整流程

```text
终端    $ npm install -g @fission-ai/openspec@latest
终端    $ cd your-project && openspec init
AI 聊天   /opsx:explore                    (可选：先探索思考)
AI 聊天   /opsx:propose add-dark-mode      (AI 起草计划；你审查)
AI 聊天   /opsx:apply                      (AI 实现)
AI 聊天   /opsx:archive                    (规范更新，变更归档)
```

两个终端步骤完成设置，之后全部在 AI 聊天中操作。

> **重要区分** — `openspec ...` 命令在**终端**中运行，`/opsx:...` 命令在 **AI 助手聊天窗口**中输入。

---

## 三步核心流程

```
/opsx:propose（提案）→ /opsx:apply（实施）→ /opsx:archive（归档）
```

### 阶段一：Propose（提案）

在写任何代码之前，先与 AI 就"要做什么"达成一致，生成结构化文档。

```
/opsx:propose <你的需求>
```

**产出物：**

```
openspec/changes/<change-id>/
├── proposal.md    # 为什么做、做什么
├── specs/         # 规范增量（Spec Delta）— 需求与验收场景
├── design.md      # 技术实现方案
└── tasks.md       # 实现任务清单
```

**各文件说明：**

- **proposal.md** — 描述变更动机、范围和预期影响（为什么做、做什么、影响范围）
- **specs/** — 受影响的规范模块，每个模块一个 spec.md，使用 RFC 2119 关键词（MUST/SHALL/SHOULD/MAY）+ Given/When/Then 验收场景格式。通过 +（新增）-（删除）符号展示差异
- **design.md** — 技术选型理由、架构变更、关键决策、潜在风险
- **tasks.md** — 分解后的可执行任务清单，按阶段组织

### 阶段二：Apply（实施）

AI 严格按照提案和任务清单执行代码实现。

```
/opsx:apply
```

- AI 读取 proposal.md、specs/、design.md、tasks.md
- 按照 tasks.md 中的清单逐项实现
- 严格遵循规范，不随意发挥
- 可中断可恢复：中断后重新 `/opsx:apply` 继续

### 阶段三：Archive（归档）

变更完成后，将 Spec Delta 合并回主规范，变更文件夹移至归档目录。

```
/opsx:archive
```

归档后的目录结构：

```
openspec/
├── specs/                          # 已更新（合并了变更）
│   ├── auth-session/spec.md
│   └── theme/spec.md
├── changes/
│   └── archive/
│       └── 2026-03-10-add-dark-mode/  # 已归档
└── project.md
```

### 演示

```text
You: /opsx:propose add-dark-mode
AI:  Created openspec/changes/add-dark-mode/
     ✓ proposal.md — why we're doing this, what's changing
     ✓ specs/       — requirements and scenarios
     ✓ design.md    — technical approach
     ✓ tasks.md     — implementation checklist
     Ready for implementation!

You: /opsx:apply
AI:  Implementing tasks...
     ✓ 1.1 Add theme context provider
     ✓ 1.2 Create toggle component
     ✓ 2.1 Add CSS variables
     ✓ 2.2 Wire up localStorage
     All tasks complete!

You: /opsx:archive
AI:  Archived to openspec/changes/archive/2025-01-23-add-dark-mode/
     Specs updated. Ready for the next feature.
```

---

## 扩展命令

除了三段式核心流程，还提供扩展命令：

| 命令 | 作用 |
|------|------|
| `/opsx:explore` | 探索想法，分析代码库可行性，不生成文件 — 思考伙伴 |
| `/opsx:new` | 创建新的变更工作单元（扩展工作流） |
| `/opsx:continue` | 从中断处继续当前变更的实施 |
| `/opsx:ff` | 快速跟进，跳过某些审查步骤 |
| `/opsx:verify` | 验证实现是否与规范一致 |
| `/opsx:bulk-archive` | 批量归档多个已完成的变更 |
| `/opsx:onboard` | 为新成员或 AI 会话快速建立项目上下文 |

### 扩展工作流切换

```bash
openspec config profile    # 交互式选择
openspec update            # 应用配置
```

---

## 升级与维护

```bash
# 升级包
npm install -g @fission-ai/openspec@latest

# 刷新 AI 指令（在每个项目内运行）
openspec update
```

---

## 推荐工作习惯

1. **从 `/opsx:explore` 开始** — 如果不确定要构建什么，先探索。它会阅读代码库、权衡选项、将模糊想法打磨成具体计划
2. **模型选择** — 推荐高推理能力模型：Codex 5.5、Claude Opus 4.7
3. **上下文卫生** — 开始实现前清理上下文窗口，OpenSpec 在干净上下文中效果更好

---

## 核心概念

### 整体架构

```
┌──────────────────────────────────────────┐
│              AI 编码助手层                 │
│  Claude Code / Cursor / Codex / ...      │
│         通过 Slash 命令交互               │
├──────────────────────────────────────────┤
│             OpenSpec 规范层               │
│  specs/ ←→ changes/ (Spec Delta)         │
│  CLI (openspec init/update)              │
├──────────────────────────────────────────┤
│              源代码层                      │
│  你的实际项目代码                          │
└──────────────────────────────────────────┘
```

### 1. Specification（规范）

按能力（Capability）组织：

```
openspec/specs/
├── auth-login/spec.md       # 登录功能规范
├── auth-session/spec.md     # 会话管理规范
├── checkout-cart/spec.md    # 购物车规范
└── checkout-payment/spec.md # 支付流程规范
```

每个 `spec.md` 包含：

| 部分 | 说明 |
|------|------|
| **Purpose** | 该能力的职责描述 |
| **Requirements** | 功能性需求列表（使用 RFC 2119: SHALL/MUST/SHOULD/MAY） |
| **Scenarios** | 验收场景（Given/When/Then 格式） |

示例：

```markdown
# auth-session Specification

## Purpose
Manage user session lifecycle including creation, validation, and expiration.

## Requirements

### Requirement: Session expiration
The system SHALL support configurable session expiration periods.

#### Scenario: Default session timeout
- GIVEN a user has authenticated
- WHEN 24 hours pass without "Remember me"
- THEN invalidate the session token

#### Scenario: Extended session with remember me
- GIVEN user checks "Remember me" at login
- WHEN 30 days have passed
- THEN invalidate the session token
- AND clear the persistent cookie
```

### 2. Change（变更）

每次变更拥有独立的文件夹，包含四个产物：

| 产物 | 作用 | 受众 |
|------|------|------|
| **proposal.md** | 变更动机和范围 | 所有人 |
| **specs/** | 需求变更的增量（diff） | 产品/QA |
| **design.md** | 技术实现方案 | 开发者 |
| **tasks.md** | 可执行的任务清单 | AI/开发者 |

### 3. Spec Delta（规范增量）

变更是对规范产生的差异，使用 +（新增）-（修改）符号展示：

```
原规范:  The system SHALL expire sessions after 24h.

Spec Delta:
- The system SHALL expire sessions after a configured duration.
+ The system SHALL support configurable session expiration periods.

- WHEN 24 hours pass without activity
+ WHEN 24 hours pass without "Remember me"

+ #### Scenario: Extended session with remember me
+ - GIVEN user checks "Remember me" at login
+ - WHEN 30 days have passed
+ - THEN invalidate the session token
+ - AND clear the persistent cookie
```

Spec Delta 的价值：
- **变更可审查** — 清晰看到需求的变化
- **避免冲突** — 多个并行变更互不干扰
- **安全归档** — 归档时执行合并，减少出错

### 4. 双文件夹模型

```
openspec/
├── specs/       ← 系统当前状态（已达成共识的规范）
└── changes/     ← 待定更新（进行中的提案）
    └── archive/ ← 已完成的历史变更
```

| 文件夹 | 稳定性 | 用途 |
|--------|--------|------|
| `specs/` | 稳定，已共识 | 系统的权威描述 |
| `changes/` | 流动，进行中 | 新功能的试验场 |
| `archive/` | 历史记录 | 追溯已完成的变更 |

分离的好处：支持并行开发，多个团队成员可同时在不同变更上工作，互不干扰。

### 5. Living Spec（活文档）

规范不是一次性的规划产物，而是随代码持续演进的**活文档**：

- **创建** — 实现新功能时创建规范
- **审查** — PR 中同时审查规范和代码
- **更新** — 功能变更时随之更新规范
- **归档** — 完成后规范合并至主目录

---

## 核心特性

1. **轻量级** — 无需 API Key，最小化配置
2. **Brownfield 优先** — 不仅适用于 0→1 新项目，在修改现有系统（1→n）时表现尤为出色，通过 Spec Delta 管理变更
3. **双文件夹模型** — `openspec/specs/`（当前真实状态）与 `openspec/changes/`（待定更新）分离
4. **工具无关** — 支持 25+ AI 编码助手，不锁定特定工具或模型
5. **可追溯** — 提案、任务、规范差异共存，归档后自动合并回主规范
6. **Spec 即代码** — 规范文件与代码一起存储在仓库中，通过 Git 管理

---

## 价值主张

| 价值 | 说明 |
|------|------|
| 🤝 先对齐再编码 | 人与 AI 在写代码前先就规范达成共识，避免方向性偏差导致大量返工 |
| 📁 井然有序 | 每个变更都有独立文件夹，包含提案、规范、设计和任务四个产物 |
| 🌊 流动灵活 | 随时更新任意产物，无需遵守僵化的阶段门控 |
| 🔧 工具生态广泛 | 通过 Slash 命令支持 25+ AI 助手 |

---

## 与同类产品对比

| 维度 | OpenSpec | Spec Kit (GitHub) | Kiro (AWS) | 不用规范 |
|------|----------|-------------------|------------|----------|
| **重量** | 轻量 | 重量级 | 中等 | 无 |
| **阶段门控** | 灵活迭代 | 严格门控 | 有 | 无 |
| **环境依赖** | Node.js | Python | IDE 锁定 | 无 |
| **Brownfield** | 优先支持 | 弱 | 一般 | N/A |
| **工具绑定** | 工具无关（25+） | 工具无关 | 仅限其 IDE + Claude | 取决于提示词 |
| **规范持久化** | 是（活文档） | 是 | 部分 | 否（只在聊天中） |
| **适合场景** | 新项目 + 存量项目 | 全新项目 (0→1) | 特定工具链用户 | 个人小实验 |

### vs. Spec Kit (GitHub)

Spec Kit 全面但重量级，有严格阶段门控、大量 Markdown 文件、Python 环境依赖。OpenSpec 更轻量，让你自由迭代。

### vs. Kiro (AWS)

Kiro 功能强大但锁定在其 IDE 内，且仅限 Claude 模型。OpenSpec 支持你现有的全部工具。

### vs. 不用规范

没有规范的 AI 编码意味着模糊提示和不可预测的结果。OpenSpec 在不增加繁文缛节的前提下带来可预测性。

---

## 适用场景

- **个人开发者** — 摆脱"AI 失忆症"，需求可追溯
- **团队协作** — 统一代码风格和开发规范，通过 Git PR 协作审查
- **遗留系统重构** — 拆分 Proposal 逐步重构，每次变更可追踪
- **中大型项目长期维护** — 归档后的 Spec 成为项目知识库

> ThoughtWorks 技术雷达已将 OpenSpec 列入推荐评估工具，评价其"将 AI 编码从艺术创作升级为规范工程"。

---

## 常见问题

**Q: OpenSpec 和 AI 助手自带的 plan mode 有什么区别？**
Plan mode 适用于单次聊天会话。OpenSpec 专注跨多会话的计划，或需要与他人共享的计划。它是贯穿整个开发生命周期的工具。

**Q: 为什么用 Spec 而不直接写详细的 Prompt？**
Spec 用于对齐。在写代码之前结构化你的思考。你不会让建筑师在没有蓝图的情况下盖房子，这里同理。

**Q: 能在现有代码库上使用吗？**
可以。规范随着你构建而创建。按需创建 Spec，逐步构建。不必一次性为所有现有代码生成 Spec。

**Q: 切换 AI 工具时怎么办？**
OpenSpec 的目标是通用规划层，支持任何编码工具。编码工具在快速演进，你的规范不应该绑定特定工具。

**Q: 规范放在哪里？**
在代码库里，应该随代码一起提交（Check-in）。它们提供了系统如何工作以及设计意图的可见性。

**Q: 团队如何分享和协作规范？**
规范和变更都在代码中，推荐通过 Git 协作 — PR、审查、常规工作流。正在构建面向大型代码库、多仓库系统、微服务的更深层团队功能。

**Q: 这是瀑布模型吗？**
不。瀑布失败是因为僵化的计划和数月的预先规划。OpenSpec 让你快速达到足够好的计划并开始编码 — 最小努力，轻量流程。情况变化时？更新 Spec 继续前进。

**Q: 适合"氛围编程"（Vibe Coding）吗？**
取决于你。如果你寻找一劳永逸的魔法工具，这不是它。Spec 只在你会阅读、思考和参与其中时才起作用。

---

## 支持的 AI 工具（部分）

**原生支持（内置自定义 Slash 命令）：**
Claude Code、Cursor、Codex、GitHub Copilot、OpenCode、Windsurf、Gemini CLI、Antigravity、Cline、RooCode、Kilo Code、Amazon Q、Qoder、Auggie CLI、Qwen Code、CodeBuddy、CoStrict、Crush、Factory Droid、iFlow（共 25+）

---

## 遥测说明

OpenSpec 收集匿名使用统计：仅收集命令名称和版本以了解使用模式。**不收集**参数、路径、内容或 PII。CI 环境中自动禁用。

**退出遥测：**
```bash
export OPENSPEC_TELEMETRY=0
# 或
export DO_NOT_TRACK=1
```

---

## 参考

- 官网: https://openspec.dev/
- GitHub: https://github.com/Fission-AI/OpenSpec
- 中文站: https://openspec.radebit.com/
