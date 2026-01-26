---
summary: "Agent runtime (embedded p-mono), workspace contract, and session bootstrap"
read_when:
  - Changing agent runtime, workspace bootstrap, or session behavior
---
# 代理运行时 🤖

Clawdbot 运行一个从 **p-mono** 派生的嵌入式代理运行时。

## 工作区（必需）

Clawdbot 使用单个代理工作区目录（`agents.defaults.workspace`）作为代理的**唯一**工作目录（`cwd`），用于工具和上下文。

推荐：如果缺少，使用 `clawdbot setup` 创建 `~/.clawdbot/clawdbot.json` 并初始化工作区文件。

完整工作区布局 + 备份指南：[代理工作区](/concepts/agent-workspace)

如果启用了 `agents.defaults.sandbox`，非主会话可以在 `agents.defaults.sandbox.workspaceRoot` 下使用每个会话的工作区覆盖此设置（参见 [Gateway 配置](/gateway/configuration)）。

## 引导文件（注入）

在 `agents.defaults.workspace` 内，Clawdbot 期望这些用户可编辑的文件：
- `AGENTS.md` — 操作说明 + "记忆"
- `SOUL.md` — 角色、边界、语调
- `TOOLS.md` — 用户维护的工具说明（例如 `imsg`、`sag`、约定）
- `BOOTSTRAP.md` — 一次性首次运行仪式（完成后删除）
- `IDENTITY.md` — 代理名称/氛围/表情符号
- `USER.md` — 用户配置文件 + 首选地址

在新会话的第一轮中，Clawdbot 将这些文件的内容直接注入到代理上下文中。

空白文件被跳过。大文件被修剪和截断，并带有标记，以便提示保持精简（读取文件以获取完整内容）。

如果文件缺失，Clawdbot 注入单个"缺失文件"标记行（`clawdbot setup` 将创建安全的默认模板）。

`BOOTSTRAP.md` 仅为**全新工作区**创建（不存在其他引导文件）。如果你在完成仪式后删除它，它不应在后续重启时重新创建。

要完全禁用引导文件创建（对于预播种的工作区），请设置：

```json5
{ agent: { skipBootstrap: true } }
```

## 内置工具

核心工具（read/exec/edit/write 和相关系统工具）始终可用，受工具策略约束。`apply_patch` 是可选的，由 `tools.exec.applyPatch` 门控。`TOOLS.md` **不**控制哪些工具存在；它是关于*你*希望如何使用它们的指导。

## 技能

Clawdbot 从三个位置加载技能（工作区在名称冲突时获胜）：
- 捆绑（随安装一起提供）
- 托管/本地：`~/.clawdbot/skills`
- 工作区：`<workspace>/skills`

技能可以通过配置/env 门控（参见 [Gateway 配置](/gateway/configuration) 中的 `skills`）。

## p-mono 集成

Clawdbot 重用 p-mono 代码库的部分（模型/工具），但**会话管理、发现和工具布线是 Clawdbot 拥有的**。

- 没有 p-coding 代理运行时。
- 不咨询 `~/.pi/agent` 或 `<workspace>/.pi` 设置。

## 会话

会话转录本存储为 JSONL：
- `~/.clawdbot/agents/<agentId>/sessions/<SessionId>.jsonl`

会话 ID 是稳定的，由 Clawdbot 选择。
不读取传统 Pi/Tau 会话文件夹。

## 流式传输时的转向

当队列模式为 `steer` 时，入站消息被注入到当前运行中。
队列在**每次工具调用后**检查；如果存在排队的消息，则跳过当前助手消息的剩余工具调用（工具结果错误，显示"Skipped due to queued user message."），然后在下一个助手响应之前注入排队的用户消息。

当队列模式为 `followup` 或 `collect` 时，入站消息被保留，直到当前回合结束，然后使用排队的负载启动新的代理回合。有关模式 + 防抖/上限行为，请参见 [队列](/concepts/queue)。

块流式传输在完成时立即发送已完成的助手块；它**默认关闭**（`agents.defaults.blockStreamingDefault: "off"`）。
通过 `agents.defaults.blockStreamingBreak` 调整边界（`text_end` vs `message_end`；默认为 text_end）。
使用 `agents.defaults.blockStreamingChunk` 控制软块分块（默认为 800–1200 字符；优先段落中断，然后是换行；句子最后）。
使用 `agents.defaults.blockStreamingCoalesce` 合并流式块以减少单行垃圾邮件（发送前基于空闲的合并）。非 Telegram 通道需要显式 `*.blockStreaming: true` 以启用块回复。
详细工具摘要在工具开始时发出（无防抖）；控制 UI 在可用时通过代理事件流式传输工具输出。
更多详细信息：[流式传输 + 分块](/concepts/streaming)。

## 模型引用

配置中的模型引用（例如 `agents.defaults.model` 和 `agents.defaults.models`）通过在**第一个** `/` 上拆分来解析。

- 配置模型时使用 `provider/model`。
- 如果模型 ID 本身包含 `/`（OpenRouter 风格），请包含提供者前缀（示例：`openrouter/moonshotai/kimi-k2`）。
- 如果你省略提供者，Clawdbot 将输入视为别名或**默认提供者**的模型（仅在模型 ID 中没有 `/` 时有效）。

## 配置（最小）

至少设置：
- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom`（强烈推荐）

---

*下一步：[群聊](/concepts/group-messages)* 🦞
