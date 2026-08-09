---
excalidraw-plugin: parsed
excalidraw-json: |
  {
    "type": "excalidraw",
    "version": 2,
    "source": "https://marketplace.obsidian.md/plugins/excalidraw",
    "elements": [
      {
        "type": "text",
        "text": "Hermes Agent 完整架构流程图",
        "x": 380,
        "y": -80,
        "width": 500,
        "height": 35,
        "fontSize": 28
      },
      {
        "type": "text",
        "text": "AIAgent 系统架构 · 数据流 · 子系统全景",
        "x": 420,
        "y": -45,
        "width": 420,
        "height": 20,
        "fontSize": 13,
        "strokeColor": "#666"
      },
      {
        "type": "line",
        "x": 30,
        "y": -15,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 1050,
            "y": 0
          }
        ],
        "strokeColor": "#ccc"
      },
      {
        "type": "rectangle",
        "x": 350,
        "y": 5,
        "width": 500,
        "height": 32,
        "fillStyle": "solid",
        "strokeColor": "#0d47a1",
        "backgroundColor": "#e3f2fd"
      },
      {
        "type": "text",
        "text": "1. Entry Points (入口层)",
        "x": 460,
        "y": 11,
        "fontSize": 15,
        "strokeColor": "#0d47a1"
      },
      {
        "type": "rectangle",
        "x": 30,
        "y": 55,
        "width": 155,
        "height": 50,
        "fillStyle": "solid",
        "strokeColor": "#1565c0",
        "backgroundColor": "#bbdefb"
      },
      {
        "type": "text",
        "text": "CLI (cli.py)",
        "x": 50,
        "y": 63,
        "fontSize": 14
      },
      {
        "type": "text",
        "text": "命令行交互入口",
        "x": 55,
        "y": 84,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "rectangle",
        "x": 210,
        "y": 55,
        "width": 380,
        "height": 75,
        "fillStyle": "solid",
        "strokeColor": "#1565c0",
        "backgroundColor": "#bbdefb"
      },
      {
        "type": "text",
        "text": "Gateway (21平台)",
        "x": 300,
        "y": 63,
        "fontSize": 14
      },
      {
        "type": "text",
        "text": "telegram · discord · slack · whatsapp · signal · matrix · mattermost",
        "x": 220,
        "y": 84,
        "fontSize": 10,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "email · sms · dingtalk · feishu · wecom · weixin · qqbot · yuanbao · webhook",
        "x": 215,
        "y": 100,
        "fontSize": 10,
        "strokeColor": "#555"
      },
      {
        "type": "rectangle",
        "x": 620,
        "y": 55,
        "width": 200,
        "height": 50,
        "fillStyle": "solid",
        "strokeColor": "#1565c0",
        "backgroundColor": "#bbdefb"
      },
      {
        "type": "text",
        "text": "ACP (IDE集成)",
        "x": 650,
        "y": 63,
        "fontSize": 14
      },
      {
        "type": "text",
        "text": "VS Code / Zed / JetBrains",
        "x": 630,
        "y": 84,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "rectangle",
        "x": 850,
        "y": 55,
        "width": 200,
        "height": 50,
        "fillStyle": "solid",
        "strokeColor": "#1565c0",
        "backgroundColor": "#bbdefb"
      },
      {
        "type": "text",
        "text": "Cron + Batch + API",
        "x": 865,
        "y": 63,
        "fontSize": 14
      },
      {
        "type": "text",
        "text": "定时任务 · 批量 · HTTP API",
        "x": 860,
        "y": 84,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "arrow",
        "x": 107,
        "y": 105,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 55
          }
        ],
        "strokeColor": "#1565c0"
      },
      {
        "type": "arrow",
        "x": 400,
        "y": 130,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 30
          }
        ],
        "strokeColor": "#1565c0"
      },
      {
        "type": "arrow",
        "x": 720,
        "y": 105,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 55
          }
        ],
        "strokeColor": "#1565c0"
      },
      {
        "type": "arrow",
        "x": 950,
        "y": 105,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 55
          }
        ],
        "strokeColor": "#1565c0"
      },
      {
        "type": "line",
        "x": 30,
        "y": 162,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 1050,
            "y": 0
          }
        ],
        "strokeColor": "#e0e0e0"
      },
      {
        "type": "rectangle",
        "x": 350,
        "y": 175,
        "width": 500,
        "height": 32,
        "fillStyle": "solid",
        "strokeColor": "#b71c1c",
        "backgroundColor": "#ffcdd2"
      },
      {
        "type": "text",
        "text": "2. AIAgent 核心 (run_agent.py ~9431行)",
        "x": 380,
        "y": 181,
        "fontSize": 15,
        "strokeColor": "#b71c1c"
      },
      {
        "type": "rectangle",
        "x": 150,
        "y": 225,
        "width": 750,
        "height": 42,
        "fillStyle": "solid",
        "strokeColor": "#c62828",
        "backgroundColor": "#ffebee"
      },
      {
        "type": "text",
        "text": "AIAgent 初始化 — 7阶段",
        "x": 400,
        "y": 230,
        "fontSize": 15
      },
      {
        "type": "text",
        "text": "配置 → API模式检测 → 回调注册 → LLM客户端 → 工具加载 → 记忆初始化 → 压缩器",
        "x": 160,
        "y": 252,
        "fontSize": 12,
        "strokeColor": "#555"
      },
      {
        "type": "rectangle",
        "x": 150,
        "y": 285,
        "width": 430,
        "height": 40,
        "fillStyle": "solid",
        "strokeColor": "#c62828",
        "backgroundColor": "#ffebee"
      },
      {
        "type": "text",
        "text": "run_conversation() 主循环 (6800-9431行)",
        "x": 180,
        "y": 292,
        "fontSize": 14
      },
      {
        "type": "text",
        "text": "消息 → Prompt → API → 响应 → 工具 → 压缩 → 循环",
        "x": 155,
        "y": 312,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "rectangle",
        "x": 610,
        "y": 285,
        "width": 440,
        "height": 40,
        "fillStyle": "solid",
        "strokeColor": "#c62828",
        "backgroundColor": "#ffebee"
      },
      {
        "type": "text",
        "text": "8个回调系统",
        "x": 760,
        "y": 290,
        "fontSize": 14
      },
      {
        "type": "text",
        "text": "thinking | tool_progress | reasoning | clarify | step | stream | tool_gen | status",
        "x": 615,
        "y": 312,
        "fontSize": 9,
        "strokeColor": "#555"
      },
      {
        "type": "rectangle",
        "x": 150,
        "y": 340,
        "width": 900,
        "height": 30,
        "fillStyle": "solid",
        "strokeColor": "#c62828",
        "backgroundColor": "#ffeeee"
      },
      {
        "type": "text",
        "text": "_interruptible_api_call: 后台线程HTTP + 主线程中断监听 · 中断丢弃响应保证原子性",
        "x": 170,
        "y": 347,
        "fontSize": 12,
        "strokeColor": "#555"
      },
      {
        "type": "line",
        "x": 30,
        "y": 385,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 1050,
            "y": 0
          }
        ],
        "strokeColor": "#e0e0e0"
      },
      {
        "type": "rectangle",
        "x": 350,
        "y": 400,
        "width": 500,
        "height": 32,
        "fillStyle": "solid",
        "strokeColor": "#4a148c",
        "backgroundColor": "#f3e5f5"
      },
      {
        "type": "text",
        "text": "3. 三条核心子系统 — Prompt · Provider · Tool",
        "x": 380,
        "y": 406,
        "fontSize": 15,
        "strokeColor": "#4a148c"
      },
      {
        "type": "rectangle",
        "x": 30,
        "y": 450,
        "width": 330,
        "height": 200,
        "fillStyle": "solid",
        "strokeColor": "#1b5e20",
        "backgroundColor": "#e8f5e9"
      },
      {
        "type": "text",
        "text": "Prompt System",
        "x": 120,
        "y": 460,
        "fontSize": 15,
        "strokeColor": "#1b5e20"
      },
      {
        "type": "text",
        "text": "三层组装: stable → context → volatile",
        "x": 45,
        "y": 486,
        "fontSize": 12,
        "strokeColor": "#2e7d32"
      },
      {
        "type": "text",
        "text": "stable: SOUL.md + 行为指导",
        "x": 50,
        "y": 505,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "context: .hermes.md / AGENTS.md",
        "x": 50,
        "y": 521,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "volatile: MEMORY.md + USER.md + 时间戳",
        "x": 50,
        "y": 537,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "line",
        "x": 50,
        "y": 555,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 290,
            "y": 0
          }
        ],
        "strokeColor": "#a5d6a7"
      },
      {
        "type": "text",
        "text": "Ephemeral injection (不破坏缓存)",
        "x": 55,
        "y": 563,
        "fontSize": 11,
        "strokeColor": "#2e7d32"
      },
      {
        "type": "line",
        "x": 50,
        "y": 580,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 290,
            "y": 0
          }
        ],
        "strokeColor": "#a5d6a7"
      },
      {
        "type": "text",
        "text": "双重压缩: Agent 50% + Gateway 85%",
        "x": 50,
        "y": 590,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "四阶段: 清理 → 边界 → 摘要 → 组装",
        "x": 55,
        "y": 608,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "Anthropic Cache: system_and_3策略",
        "x": 55,
        "y": 626,
        "fontSize": 11,
        "strokeColor": "#888"
      },
      {
        "type": "rectangle",
        "x": 380,
        "y": 450,
        "width": 330,
        "height": 200,
        "fillStyle": "solid",
        "strokeColor": "#e65100",
        "backgroundColor": "#fff3e0"
      },
      {
        "type": "text",
        "text": "Provider Resolution",
        "x": 450,
        "y": 460,
        "fontSize": 15,
        "strokeColor": "#e65100"
      },
      {
        "type": "text",
        "text": "3种API → 收敛到内部OpenAI格式",
        "x": 395,
        "y": 486,
        "fontSize": 12,
        "strokeColor": "#f57c00"
      },
      {
        "type": "text",
        "text": "chat_completions (OpenRouter等)",
        "x": 400,
        "y": 505,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "codex_responses (OpenAI Codex)",
        "x": 400,
        "y": 521,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "anthropic_messages (Anthropic)",
        "x": 400,
        "y": 537,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "line",
        "x": 400,
        "y": 555,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 290,
            "y": 0
          }
        ],
        "strokeColor": "#ffcc80"
      },
      {
        "type": "text",
        "text": "28+ provider · Fallback Chain",
        "x": 415,
        "y": 563,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "line",
        "x": 400,
        "y": 580,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 290,
            "y": 0
          }
        ],
        "strokeColor": "#ffcc80"
      },
      {
        "type": "text",
        "text": "IterationBudget: 默认90 · 子agent50",
        "x": 395,
        "y": 590,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "70%/90% 压力预警",
        "x": 440,
        "y": 608,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "execute_code 免计预算",
        "x": 430,
        "y": 626,
        "fontSize": 11,
        "strokeColor": "#888"
      },
      {
        "type": "rectangle",
        "x": 730,
        "y": 450,
        "width": 330,
        "height": 200,
        "fillStyle": "solid",
        "strokeColor": "#6a1b9a",
        "backgroundColor": "#f3e5f5"
      },
      {
        "type": "text",
        "text": "Tool System",
        "x": 840,
        "y": 460,
        "fontSize": 15,
        "strokeColor": "#6a1b9a"
      },
      {
        "type": "text",
        "text": "ToolRegistry 单例 · 自注册",
        "x": 780,
        "y": 486,
        "fontSize": 12,
        "strokeColor": "#7b1fa2"
      },
      {
        "type": "text",
        "text": "70+ tools · 28 toolsets",
        "x": 800,
        "y": 505,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "并发: ThreadPoolExecutor",
        "x": 790,
        "y": 521,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "交互式工具强制顺序",
        "x": 800,
        "y": 537,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "line",
        "x": 750,
        "y": 555,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 290,
            "y": 0
          }
        ],
        "strokeColor": "#ce93d8"
      },
      {
        "type": "text",
        "text": "4个Agent级拦截:",
        "x": 815,
        "y": 563,
        "fontSize": 11,
        "strokeColor": "#7b1fa2"
      },
      {
        "type": "text",
        "text": "todo · memory · session_search · delegate",
        "x": 750,
        "y": 580,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "line",
        "x": 750,
        "y": 598,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 290,
            "y": 0
          }
        ],
        "strokeColor": "#ce93d8"
      },
      {
        "type": "text",
        "text": "Terminal(6) Browser(5) Web(4) MCP",
        "x": 755,
        "y": 608,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "local docker ssh modal daytona singularity",
        "x": 750,
        "y": 626,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "arrow",
        "x": 195,
        "y": 650,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 28
          }
        ],
        "strokeColor": "#1b5e20"
      },
      {
        "type": "arrow",
        "x": 545,
        "y": 650,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 28
          }
        ],
        "strokeColor": "#e65100"
      },
      {
        "type": "arrow",
        "x": 895,
        "y": 650,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 28
          }
        ],
        "strokeColor": "#6a1b9a"
      },
      {
        "type": "line",
        "x": 30,
        "y": 685,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 1050,
            "y": 0
          }
        ],
        "strokeColor": "#e0e0e0"
      },
      {
        "type": "rectangle",
        "x": 350,
        "y": 700,
        "width": 500,
        "height": 32,
        "fillStyle": "solid",
        "strokeColor": "#e65100",
        "backgroundColor": "#fff8e1"
      },
      {
        "type": "text",
        "text": "4. 决策与循环",
        "x": 530,
        "y": 706,
        "fontSize": 15,
        "strokeColor": "#e65100"
      },
      {
        "type": "diamond",
        "x": 400,
        "y": 748,
        "width": 140,
        "height": 60,
        "fillStyle": "solid",
        "strokeColor": "#e65100",
        "backgroundColor": "#fffde7"
      },
      {
        "type": "text",
        "text": "tool_calls?",
        "x": 430,
        "y": 768,
        "fontSize": 13,
        "strokeColor": "#e65100"
      },
      {
        "type": "text",
        "text": "Yes →",
        "x": 550,
        "y": 760,
        "fontSize": 11,
        "strokeColor": "#e65100"
      },
      {
        "type": "arrow",
        "x": 540,
        "y": 778,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 70,
            "y": 0
          }
        ],
        "strokeColor": "#e65100"
      },
      {
        "type": "text",
        "text": "No ↓",
        "x": 458,
        "y": 815,
        "fontSize": 11,
        "strokeColor": "#e65100"
      },
      {
        "type": "arrow",
        "x": 470,
        "y": 808,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 0,
            "y": 25
          }
        ],
        "strokeColor": "#e65100"
      },
      {
        "type": "rectangle",
        "x": 610,
        "y": 748,
        "width": 350,
        "height": 55,
        "fillStyle": "solid",
        "strokeColor": "#e65100",
        "backgroundColor": "#fff3e0"
      },
      {
        "type": "text",
        "text": "Tool Loop 工具循环",
        "x": 700,
        "y": 755,
        "fontSize": 14,
        "strokeColor": "#e65100"
      },
      {
        "type": "text",
        "text": "并发/顺序执行 → 追加tool结果 → 回到API调用",
        "x": 620,
        "y": 778,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "↻ 回到 run_conversation 步骤5",
        "x": 660,
        "y": 793,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "rectangle",
        "x": 250,
        "y": 833,
        "width": 500,
        "height": 35,
        "fillStyle": "solid",
        "strokeColor": "#c62828",
        "backgroundColor": "#ffebee"
      },
      {
        "type": "text",
        "text": "Session持久化 + Memory刷写 → 返回结果",
        "x": 320,
        "y": 840,
        "fontSize": 14,
        "strokeColor": "#b71c1c"
      },
      {
        "type": "line",
        "x": 30,
        "y": 885,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 1050,
            "y": 0
          }
        ],
        "strokeColor": "#e0e0e0"
      },
      {
        "type": "rectangle",
        "x": 350,
        "y": 900,
        "width": 500,
        "height": 32,
        "fillStyle": "solid",
        "strokeColor": "#37474f",
        "backgroundColor": "#eceff1"
      },
      {
        "type": "text",
        "text": "5. 基础设施层",
        "x": 525,
        "y": 906,
        "fontSize": 15,
        "strokeColor": "#37474f"
      },
      {
        "type": "rectangle",
        "x": 30,
        "y": 950,
        "width": 330,
        "height": 100,
        "fillStyle": "solid",
        "strokeColor": "#455a64",
        "backgroundColor": "#eceff1"
      },
      {
        "type": "text",
        "text": "Session Storage",
        "x": 110,
        "y": 958,
        "fontSize": 14,
        "strokeColor": "#37474f"
      },
      {
        "type": "text",
        "text": "SQLite + WAL + FTS5全文搜索",
        "x": 55,
        "y": 980,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "sessions · messages · messages_fts",
        "x": 55,
        "y": 996,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "写入竞争: 抖动重试15次 + WAL checkpoint",
        "x": 40,
        "y": 1014,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "text",
        "text": "Session血缘: parent_session_id 链",
        "x": 60,
        "y": 1030,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "rectangle",
        "x": 380,
        "y": 950,
        "width": 300,
        "height": 100,
        "fillStyle": "solid",
        "strokeColor": "#455a64",
        "backgroundColor": "#eceff1"
      },
      {
        "type": "text",
        "text": "Memory System",
        "x": 460,
        "y": 958,
        "fontSize": 14,
        "strokeColor": "#37474f"
      },
      {
        "type": "text",
        "text": "MEMORY.md / USER.md 持久化",
        "x": 400,
        "y": 980,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "Memory Provider 插件 (Honcho)",
        "x": 405,
        "y": 996,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "磁盘更新 ≠ Prompt更新 (缓存优先)",
        "x": 400,
        "y": 1014,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "text",
        "text": "on_session_end → flush memories",
        "x": 410,
        "y": 1030,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "rectangle",
        "x": 700,
        "y": 950,
        "width": 360,
        "height": 100,
        "fillStyle": "solid",
        "strokeColor": "#455a64",
        "backgroundColor": "#eceff1"
      },
      {
        "type": "text",
        "text": "Plugin & Others",
        "x": 810,
        "y": 958,
        "fontSize": 14,
        "strokeColor": "#37474f"
      },
      {
        "type": "text",
        "text": "3种发现源: 用户/项目/pip",
        "x": 730,
        "y": 980,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "Memory Provider · Context Engine (单例)",
        "x": 720,
        "y": 996,
        "fontSize": 11,
        "strokeColor": "#555"
      },
      {
        "type": "text",
        "text": "Gateway Hooks · Background Review",
        "x": 725,
        "y": 1014,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "text",
        "text": "Cron第一等公民 · Trajectories训练数据",
        "x": 720,
        "y": 1030,
        "fontSize": 10,
        "strokeColor": "#888"
      },
      {
        "type": "line",
        "x": 30,
        "y": 1065,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 1050,
            "y": 0
          }
        ],
        "strokeColor": "#999"
      },
      {
        "type": "text",
        "text": "设计原则: 缓存优先 | 优雅降级 | 原子性 | 入口无关 | 同步核心+异步边界",
        "x": 200,
        "y": 1078,
        "fontSize": 13,
        "strokeColor": "#37474f"
      },
      {
        "type": "line",
        "x": 30,
        "y": 1100,
        "points": [
          {
            "x": 0,
            "y": 0
          },
          {
            "x": 1050,
            "y": 0
          }
        ],
        "strokeColor": "#999"
      },
      {
        "type": "text",
        "text": "数据流: 用户 → Entry → run_conversation() → Prompt → Provider → API → Tool → 压缩 → Session → 返回",
        "x": 100,
        "y": 1115,
        "fontSize": 11,
        "strokeColor": "#888"
      }
    ],
    "appState": {
      "gridSize": 20,
      "viewBackgroundColor": "#ffffff"
    }
  }
---

# Hermes Agent 完整架构流程图

> 在 Obsidian 中用 Excalidraw 插件打开查看交互式架构图
