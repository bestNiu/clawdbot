---
summary: "End-to-end guide for running Clawdbot as a personal assistant with safety cautions"
read_when:
  - Onboarding a new assistant instance
  - Reviewing safety/permission implications
---
# 使用 Clawdbot 构建个人助手（Clawd 风格）

Clawdbot 是 **Pi** 代理的 WhatsApp + Telegram + Discord + iMessage 网关。插件添加 Mattermost。本指南是"个人助手"设置：一个专用的 WhatsApp 号码，行为类似于你的始终在线代理。

## ⚠️ 安全第一

你正在将代理置于可以：
- 在你的机器上运行命令（取决于你的 Pi 工具设置）
- 读取/写入工作区中的文件
- 通过 WhatsApp/Telegram/Discord/Mattermost（插件）发送消息回来

从保守开始：
- 始终设置 `channels.whatsapp.allowFrom`（永远不要在你的个人 Mac 上运行对世界开放）。
- 为助手使用专用的 WhatsApp 号码。
- 心跳现在默认为每 30 分钟。通过设置 `agents.defaults.heartbeat.every: "0m"` 禁用它，直到你信任设置。

## 先决条件

- Node **22+**
- PATH 上可用的 Clawdbot（推荐：全局安装）
- 用于助手的第二个电话号码（SIM/eSIM/预付费）

```bash
npm install -g clawdbot@latest
# 或：pnpm add -g clawdbot@latest
```

从源代码（开发）：

```bash
git clone https://github.com/clawdbot/clawdbot.git
cd clawdbot
pnpm install
pnpm ui:build # 首次运行自动安装 UI 依赖
pnpm build
pnpm link --global
```

## 双手机设置（推荐）

你想要这个：

```
你的手机（个人）          第二部手机（助手）
┌─────────────────┐           ┌─────────────────┐
│  你的 WhatsApp  │  ──────▶  │  助手 WA        │
│  +1-555-YOU     │  消息     │  +1-555-CLAWD   │
└─────────────────┘           └────────┬────────┘
                                       │ 通过 QR 链接
                                       ▼
                              ┌─────────────────┐
                              │  你的 Mac       │
                              │  (clawdbot)     │
                              │    Pi 代理      │
                              └─────────────────┘
```

如果你将个人 WhatsApp 链接到 Clawdbot，发给你的每条消息都成为"代理输入"。这很少是你想要的。

## 5 分钟快速开始

1) 配对 WhatsApp Web（显示 QR；用助手手机扫描）：

```bash
clawdbot channels login
```

2) 启动 Gateway（保持运行）：

```bash
clawdbot gateway --port 18789
```

3) 在 `~/.clawdbot/clawdbot.json` 中放置最小配置：

```json5
{
  channels: { whatsapp: { allowFrom: ["+15555550123"] } }
}
```

现在从你允许列表中的手机向助手号码发送消息。

当引导完成时，我们自动打开带有你的网关令牌的仪表板并打印令牌化链接。稍后重新打开：`clawdbot dashboard`。

## 给代理一个工作区（AGENTS）

Clawd 从其工作区目录读取操作指令和"记忆"。

默认情况下，Clawdbot 使用 `~/clawd` 作为代理工作区，并将在设置/第一次代理运行时自动创建它（加上启动器 `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`）。`BOOTSTRAP.md` 仅在工作区全新时创建（删除后不应返回）。

提示：将此文件夹视为 Clawd 的"记忆"，并使其成为 git 仓库（理想情况下是私有的），以便你的 `AGENTS.md` + 记忆文件得到备份。如果安装了 git，全新工作区会自动初始化。

```bash
clawdbot setup
```

完整工作区布局 + 备份指南：[代理工作区](/concepts/agent-workspace）
记忆工作流：[记忆](/concepts/memory）

可选：使用 `agents.defaults.workspace` 选择不同的工作区（支持 `~`）。

```json5
{
  agent: {
    workspace: "~/clawd"
  }
}
```

如果你已经从仓库提供自己的工作区文件，可以完全禁用引导文件创建：

```json5
{
  agent: {
    skipBootstrap: true
  }
}
```

## 将其转变为"助手"的配置

Clawdbot 默认为良好的助手设置，但你通常需要调整：
- `SOUL.md` 中的角色/指令
- 思考默认值（如果需要）
- 心跳（一旦你信任它）

示例：

```json5
{
  logging: { level: "info" },
  agent: {
    model: "anthropic/claude-opus-4-5",
    workspace: "~/clawd",
    thinkingDefault: "high",
    timeoutSeconds: 1800,
    // 从 0 开始；稍后启用。
    heartbeat: { every: "0m" }
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true }
      }
    }
  },
  routing: {
    groupChat: {
      mentionPatterns: ["@clawd", "clawd"]
    }
  },
  session: {
    scope: "per-sender",
    resetTriggers: ["/new", "/reset"],
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 10080
    }
  }
}
```

## 会话和记忆

- 会话文件：`~/.clawdbot/agents/<agentId>/sessions/{{SessionId}}.jsonl`
- 会话元数据（token 使用、最后路由等）：`~/.clawdbot/agents/<agentId>/sessions/sessions.json`（传统：`~/.clawdbot/sessions/sessions.json`）
- `/new` 或 `/reset` 为该聊天启动新会话（可通过 `resetTriggers` 配置）。如果单独发送，代理回复简短的问候以确认重置。
- `/compact [instructions]` 压缩会话上下文并报告剩余上下文预算。

## 心跳（主动模式）

默认情况下，Clawdbot 每 30 分钟运行一次心跳，提示：
`如果存在 HEARTBEAT.md（工作区上下文），请阅读它。严格遵守。不要推断或重复之前聊天中的旧任务。如果不需要注意，回复 HEARTBEAT_OK。`
设置 `agents.defaults.heartbeat.every: "0m"` 以禁用。

- 如果 `HEARTBEAT.md` 存在但实际上是空的（只有空行和像 `# Heading` 这样的 markdown 标题），Clawdbot 跳过心跳运行以节省 API 调用。
- 如果文件缺失，心跳仍会运行，模型决定做什么。
- 如果代理回复 `HEARTBEAT_OK`（可选地带有短填充；参见 `agents.defaults.heartbeat.ackMaxChars`），Clawdbot 抑制该心跳的出站传递。
- 心跳运行完整的代理回合 — 较短的间隔消耗更多 tokens。

```json5
{
  agent: {
    heartbeat: { every: "30m" }
  }
}
```

## 媒体输入和输出

入站附件（图像/音频/文档）可以通过模板显示给你的命令：
- `{{MediaPath}}`（本地临时文件路径）
- `{{MediaUrl}}`（伪 URL）
- `{{Transcript}}`（如果启用了音频转录）

来自代理的出站附件：在自己的行上包含 `MEDIA:<path-or-url>`（无空格）。示例：

```
这是截图。
MEDIA:/tmp/screenshot.png
```

Clawdbot 提取这些并将它们作为媒体与文本一起发送。

## 操作清单

```bash
clawdbot status          # 本地状态（凭据、会话、排队事件）
clawdbot status --all    # 完整诊断（只读、可粘贴）
clawdbot status --deep   # 添加网关健康探测（Telegram + Discord）
clawdbot health --json   # 网关健康快照（WS）
```

日志位于 `/tmp/clawdbot/` 下（默认：`clawdbot-YYYY-MM-DD.log`）。

## 后续步骤

- WebChat：[WebChat](/web/webchat）
- Gateway 操作：[Gateway 运行手册](/gateway）
- Cron + 唤醒：[Cron 作业](/automation/cron-jobs）
- macOS 菜单栏伴侣：[Clawdbot macOS 应用程序](/platforms/macos）
- iOS 节点应用程序：[iOS 应用程序](/platforms/ios）
- Android 节点应用程序：[Android 应用程序](/platforms/android）
- Windows 状态：[Windows (WSL2)](/platforms/windows）
- Linux 状态：[Linux 应用程序](/platforms/linux）
- 安全：[安全](/gateway/security）
