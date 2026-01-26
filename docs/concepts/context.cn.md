---
summary: "Context: what the model sees, how it is built, and how to inspect it"
read_when:
  - You want to understand what "context" means in Clawdbot
  - You are debugging why the model "knows" something (or forgot it)
  - You want to reduce context overhead (/context, /status, /compact)
---
# 上下文

"上下文"是**Clawdbot 为运行发送给模型的所有内容**。它受模型的**上下文窗口**（token 限制）限制。

初学者心理模型：
- **系统提示**（Clawdbot 构建）：规则、工具、技能列表、时间/运行时和注入的工作区文件。
- **对话历史**：你的消息 + 此会话的助手消息。
- **工具调用/结果 + 附件**：命令输出、文件读取、图像/音频等。

上下文*与"记忆"不同*：记忆可以存储在磁盘上并在以后重新加载；上下文是模型当前窗口内的内容。

## 快速开始（检查上下文）

- `/status` → 快速"我的窗口有多满？"视图 + 会话设置。
- `/context list` → 注入的内容 + 粗略大小（每个文件 + 总计）。
- `/context detail` → 更深入的分解：每个文件、每个工具模式大小、每个技能条目大小和系统提示大小。
- `/usage tokens` → 在每个回复后附加每次回复使用页脚。
- `/compact` → 将较旧的历史总结为紧凑条目以释放窗口空间。

另请参见：[斜杠命令](/tools/slash-commands)、[Token 使用和成本](/token-use)、[压缩](/concepts/compaction)。

## 示例输出

值因模型、提供者、工具策略和工作区中的内容而异。

### `/context list`

```
🧠 Context breakdown
Workspace: <workspaceDir>
Bootstrap max/file: 20,000 chars
Sandbox: mode=non-main sandboxed=false
System prompt (run): 38,412 chars (~9,603 tok) (Project Context 23,901 chars (~5,976 tok))

Injected workspace files:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Skills list (system prompt text): 2,184 chars (~546 tok) (12 skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Tool list (system prompt text): 1,032 chars (~258 tok)
Tool schemas (JSON): 31,988 chars (~7,997 tok) (counts toward context; not shown as text)
Tools: (same as above)

Session tokens (cached): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Context breakdown (detailed)
…
Top skills (prompt entry size):
- frontend-design: 412 chars (~103 tok)
- oracle: 401 chars (~101 tok)
… (+10 more skills)

Top tools (schema size):
- browser: 9,812 chars (~2,453 tok)
- exec: 6,240 chars (~1,560 tok)
… (+N more tools)
```

## 计入上下文窗口的内容

模型接收的所有内容都计入，包括：
- 系统提示（所有部分）。
- 对话历史。
- 工具调用 + 工具结果。
- 附件/转录（图像/音频/文件）。
- 压缩摘要和修剪工件。
- 提供者"包装器"或隐藏标头（不可见，仍被计算）。

## Clawdbot 如何构建系统提示

系统提示是**Clawdbot 拥有的**，每次运行都重建。它包括：
- 工具列表 + 简短描述。
- 技能列表（仅元数据；见下文）。
- 工作区位置。
- 时间（UTC + 转换的用户时间，如果配置）。
- 运行时元数据（主机/OS/模型/思考）。
- 在**项目上下文**下注入的工作区引导文件。

完整分解：[系统提示](/concepts/system-prompt)。

## 注入的工作区文件（项目上下文）

默认情况下，Clawdbot 注入一组固定的工作区文件（如果存在）：
- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（仅首次运行）

大文件使用 `agents.defaults.bootstrapMaxChars`（默认 `20000` 字符）按文件截断。`/context` 显示**原始 vs 注入**大小以及是否发生截断。

## 技能：注入的内容 vs 按需加载

系统提示包括一个紧凑的**技能列表**（名称 + 描述 + 位置）。此列表有实际开销。

技能说明*不*默认包含。模型应在**仅在需要时**`read` 技能的 `SKILL.md`。

## 工具：有两个成本

工具以两种方式影响上下文：
1) 系统提示中的**工具列表文本**（你看到的"工具"）。
2) **工具模式**（JSON）。这些被发送到模型，以便它可以调用工具。即使你看不到它们作为纯文本，它们也计入上下文。

`/context detail` 分解最大的工具模式，以便你可以看到什么占主导地位。

## 命令、指令和"内联快捷方式"

斜杠命令由 Gateway 处理。有几种不同的行为：
- **独立命令**：仅 `/...` 的消息作为命令运行。
- **指令**：`/think`、`/verbose`、`/reasoning`、`/elevated`、`/model`、`/queue` 在模型看到消息之前被剥离。
  - 仅指令的消息持久化会话设置。
  - 正常消息中的内联指令充当每条消息的提示。
- **内联快捷方式**（仅允许列表中的发送者）：正常消息中的某些 `/...` token 可以立即运行（示例："hey /status"），并在模型看到剩余文本之前被剥离。

详细信息：[斜杠命令](/tools/slash-commands)。

## 会话、压缩和修剪（持久化的内容）

跨消息持久化的内容取决于机制：
- **正常历史**在会话转录本中持久化，直到被策略压缩/修剪。
- **压缩**将摘要持久化到转录本中，并保持最近的消息完整。
- **修剪**从运行的*内存中*提示中删除旧工具结果，但不重写转录本。

文档：[会话](/concepts/session)、[压缩](/concepts/compaction)、[会话修剪](/concepts/session-pruning)。

## `/context` 实际报告的内容

`/context` 在可用时优先使用最新的**运行构建**系统提示报告：
- `System prompt (run)` = 从最后一次嵌入式（工具能力）运行捕获并持久化在会话存储中。
- `System prompt (estimate)` = 当不存在运行报告时（或通过不生成报告的 CLI 后端运行时）动态计算。

无论哪种方式，它都报告大小和主要贡献者；它**不**转储完整的系统提示或工具模式。
