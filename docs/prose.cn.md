---
summary: "OpenProse: .prose workflows, slash commands, and state in Clawdbot"
read_when:
  - You want to run or write .prose workflows
  - You want to enable the OpenProse plugin
  - You need to understand state storage
---
# OpenProse

OpenProse 是一种便携式、Markdown 优先的工作流格式，用于编排 AI 会话。在 Clawdbot 中，它作为插件提供，安装 OpenProse 技能包加上 `/prose` 斜杠命令。程序位于 `.prose` 文件中，可以生成多个子代理，具有显式控制流。

官方网站：https://www.prose.md

## 它能做什么

- 多代理研究 + 具有显式并行性的综合。
- 可重复的批准安全工作流（代码审查、事件分类、内容管道）。
- 可重用的 `.prose` 程序，你可以在支持的代理运行时中运行。

## 安装 + 启用

捆绑插件默认禁用。启用 OpenProse：

```bash
clawdbot plugins enable open-prose
```

启用插件后重启 Gateway。

开发/本地检出：`clawdbot plugins install ./extensions/open-prose`

相关文档：[插件](/plugin)、[插件清单](/plugins/manifest)、[技能](/tools/skills)。

## 斜杠命令

OpenProse 将 `/prose` 注册为用户可调用的技能命令。它路由到 OpenProse VM 指令并在底层使用 Clawdbot 工具。

常见命令：

```
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

## 示例：简单的 `.prose` 文件

```prose
# Research + synthesis with two agents running in parallel.

input topic: "What should we research?"

agent researcher:
  model: sonnet
  prompt: "You research thoroughly and cite sources."

agent writer:
  model: opus
  prompt: "You write a concise summary."

parallel:
  findings = session: researcher
    prompt: "Research {topic}."
  draft = session: writer
    prompt: "Summarize {topic}."

session "Merge the findings + draft into a final answer."
context: { findings, draft }
```

## 文件位置

OpenProse 在你的工作区下的 `.prose/` 中保持状态：

```
.prose/
├── .env
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose
│       ├── state.md
│       ├── bindings/
│       └── agents/
└── agents/
```

用户级持久化代理位于：

```
~/.prose/agents/
```

## 状态模式

OpenProse 支持多个状态后端：

- **filesystem**（默认）：`.prose/runs/...`
- **in-context**：瞬态，用于小程序
- **sqlite**（实验性）：需要 `sqlite3` 二进制文件
- **postgres**（实验性）：需要 `psql` 和连接字符串

注意：
- sqlite/postgres 是选择加入且实验性的。
- postgres 凭据流入子代理日志；使用专用的、最小权限的数据库。

## 远程程序

`/prose run <handle/slug>` 解析为 `https://p.prose.md/<handle>/<slug>`。
直接 URL 按原样获取。这使用 `web_fetch` 工具（或用于 POST 的 `exec`）。

## Clawdbot 运行时映射

OpenProse 程序映射到 Clawdbot 原语：

| OpenProse 概念 | Clawdbot 工具 |
| --- | --- |
| Spawn session / Task tool | `sessions_spawn` |
| File read/write | `read` / `write` |
| Web fetch | `web_fetch` |

如果你的工具允许列表阻止这些工具，OpenProse 程序将失败。参见 [技能配置](/tools/skills-config)。

## 安全 + 批准

将 `.prose` 文件视为代码。运行前审查。使用 Clawdbot 工具允许列表和批准门控来控制副作用。

对于确定性的、批准门控的工作流，与 [Lobster](/tools/lobster) 进行比较。
