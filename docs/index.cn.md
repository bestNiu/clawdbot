---
summary: "Top-level overview of Clawdbot, features, and purpose"
read_when:
  - Introducing Clawdbot to newcomers
---
# Clawdbot 🦞

> *"EXFOLIATE! EXFOLIATE!"* — 一只太空龙虾，可能

<p align="center">
  <img src="whatsapp-clawd.jpg" alt="Clawdbot" width="420" />
</p>

<p align="center">
  <strong>任何操作系统 + WhatsApp/Telegram/Discord/iMessage 网关，用于 AI 代理（Pi）。</strong><br />
  插件添加 Mattermost 等。
  发送消息，获得代理响应 — 从你的口袋。
</p>

<p align="center">
  <a href="https://github.com/clawdbot/clawdbot">GitHub</a> ·
  <a href="https://github.com/clawdbot/clawdbot/releases">Releases</a> ·
  <a href="/">Docs</a> ·
  <a href="/start/clawd">Clawdbot 助手设置</a>
</p>

Clawdbot 桥接 WhatsApp（通过 WhatsApp Web / Baileys）、Telegram（Bot API / grammY）、Discord（Bot API / channels.discord.js）和 iMessage（imsg CLI）到像 [Pi](https://github.com/badlogic/pi-mono) 这样的编码代理。插件添加 Mattermost（Bot API + WebSocket）等。
Clawdbot 还为 [Clawd](https://clawd.me)（太空龙虾助手）提供支持。

## 从这里开始

- **从零开始新安装：** [入门指南](/start/getting-started)
- **引导设置（推荐）：** [向导](/start/wizard) (`clawdbot onboard`)
- **打开仪表板（本地 Gateway）：** http://127.0.0.1:18789/ （或 http://localhost:18789/）

如果 Gateway 在同一台计算机上运行，该链接会立即打开浏览器控制 UI。
如果失败，请先启动 Gateway：`clawdbot gateway`。

## 仪表板（浏览器控制 UI）

仪表板是用于聊天、配置、节点、会话等的浏览器控制 UI。
本地默认：http://127.0.0.1:18789/
远程访问：[Web 界面](/web) 和 [Tailscale](/gateway/tailscale)

## 工作原理

```
WhatsApp / Telegram / Discord / iMessage (+ 插件)
        │
        ▼
  ┌───────────────────────────┐
  │          Gateway          │  ws://127.0.0.1:18789 (仅环回)
  │     (单一来源)       │
  │                           │  http://<gateway-host>:18793
  │                           │    /__clawdbot__/canvas/ (Canvas 主机)
  └───────────┬───────────────┘
              │
              ├─ Pi 代理 (RPC)
              ├─ CLI (clawdbot …)
              ├─ Chat UI (SwiftUI)
              ├─ macOS 应用 (Clawdbot.app)
              ├─ iOS 节点通过 Gateway WS + 配对
              └─ Android 节点通过 Gateway WS + 配对
```

大多数操作通过 **Gateway**（`clawdbot gateway`）进行，这是一个长期运行的单一进程，拥有通道连接和 WebSocket 控制平面。

## 网络模型

- **每个主机一个 Gateway（推荐）**：这是唯一允许拥有 WhatsApp Web 会话的进程。如果你需要救援机器人或严格隔离，请使用隔离的配置文件和端口运行多个网关；参见 [多个网关](/gateway/multiple-gateways）。
- **环回优先**：Gateway WS 默认为 `ws://127.0.0.1:18789`。
  - 向导现在默认生成网关令牌（即使是环回）。
  - 对于 Tailnet 访问，运行 `clawdbot gateway --bind tailnet --token ...`（非环回绑定需要令牌）。
- **Nodes**：连接到 Gateway WebSocket（根据需要 LAN/tailnet/SSH）；传统 TCP 桥接已弃用/移除。
- **Canvas 主机**：`canvasHost.port`（默认 `18793`）上的 HTTP 文件服务器，为节点 WebViews 提供 `/__clawdbot__/canvas/`；参见 [Gateway 配置](/gateway/configuration) (`canvasHost`）。
- **远程使用**：SSH 隧道或 tailnet/VPN；参见 [远程访问](/gateway/remote) 和 [发现](/gateway/discovery）。

## 功能（高级）

- 📱 **WhatsApp 集成** — 使用 Baileys 进行 WhatsApp Web 协议
- ✈️ **Telegram Bot** — 通过 grammY 的私信 + 群组
- 🎮 **Discord Bot** — 通过 channels.discord.js 的私信 + 公会频道
- 🧩 **Mattermost Bot（插件）** — Bot 令牌 + WebSocket 事件
- 💬 **iMessage** — 本地 imsg CLI 集成（macOS）
- 🤖 **代理桥接** — Pi（RPC 模式）带工具流式传输
- ⏱️ **流式传输 + 分块** — 块流式传输 + Telegram 草稿流式传输详情（[/concepts/streaming](/concepts/streaming)）
- 🧠 **多代理路由** — 将提供者账户/对等点路由到隔离的代理（工作区 + 每个代理的会话）
- 🔐 **订阅认证** — Anthropic（Claude Pro/Max）+ OpenAI（ChatGPT/Codex）通过 OAuth
- 💬 **会话** — 直接聊天折叠到共享 `main`（默认）；群组是隔离的
- 👥 **群聊支持** — 默认基于提及；所有者可以切换 `/activation always|mention`
- 📎 **媒体支持** — 发送和接收图像、音频、文档
- 🎤 **语音笔记** — 可选转录钩子
- 🖥️ **WebChat + macOS 应用** — 本地 UI + 菜单栏伴侣用于操作和语音唤醒
- 📱 **iOS 节点** — 配对为节点并公开 Canvas 界面
- 📱 **Android 节点** — 配对为节点并公开 Canvas + Chat + Camera

注意：传统 Claude/Codex/Gemini/Opencode 路径已移除；Pi 是唯一的编码代理路径。

## 快速开始

运行时要求：**Node ≥ 22**。

```bash
# 推荐：全局安装 (npm/pnpm)
npm install -g clawdbot@latest
# 或：pnpm add -g clawdbot@latest

# 引导 + 安装服务 (launchd/systemd 用户服务)
clawdbot onboard --install-daemon

# 配对 WhatsApp Web（显示二维码）
clawdbot channels login

# Gateway 在引导后通过服务运行；仍可手动运行：
clawdbot gateway --port 18789
```

稍后在 npm 和 git 安装之间切换很容易：安装另一种风格并运行 `clawdbot doctor` 以更新网关服务入口点。

从源代码（开发）：

```bash
git clone https://github.com/clawdbot/clawdbot.git
cd clawdbot
pnpm install
pnpm ui:build # 首次运行自动安装 UI 依赖
pnpm build
clawdbot onboard --install-daemon
```

如果你还没有全局安装，请从仓库通过 `pnpm clawdbot ...` 运行引导步骤。

多实例快速开始（可选）：

```bash
CLAWDBOT_CONFIG_PATH=~/.clawdbot/a.json \
CLAWDBOT_STATE_DIR=~/.clawdbot-a \
clawdbot gateway --port 19001
```

发送测试消息（需要运行中的 Gateway）：

```bash
clawdbot message send --target +15555550123 --message "Hello from Clawdbot"
```

## 配置（可选）

配置位于 `~/.clawdbot/clawdbot.json`。

- 如果你**什么都不做**，Clawdbot 使用 RPC 模式下的捆绑 Pi 二进制文件，每个发送者会话。
- 如果你想锁定它，从 `channels.whatsapp.allowFrom` 和（对于群组）提及规则开始。

示例：

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } }
    }
  },
  messages: { groupChat: { mentionPatterns: ["@clawd"] } }
}
```

## 文档

- 从这里开始：
  - [文档中心（所有页面链接）](/start/hubs)
  - [帮助](/help) ← *常见修复 + 故障排除*
  - [配置](/gateway/configuration)
  - [配置示例](/gateway/configuration-examples)
  - [斜杠命令](/tools/slash-commands)
  - [多代理路由](/concepts/multi-agent)
  - [更新 / 回滚](/install/updating)
  - [配对（私信 + 节点）](/start/pairing)
  - [Nix 模式](/install/nix)
  - [Clawdbot 助手设置（Clawd）](/start/clawd)
  - [技能](/tools/skills)
  - [技能配置](/tools/skills-config)
  - [工作区模板](/reference/templates/AGENTS)
  - [RPC 适配器](/reference/rpc)
  - [Gateway 运行手册](/gateway)
  - [节点（iOS/Android）](/nodes)
  - [Web 界面（控制 UI）](/web)
  - [发现 + 传输](/gateway/discovery)
  - [远程访问](/gateway/remote)
- 提供者和 UX：
  - [WebChat](/web/webchat)
  - [控制 UI（浏览器）](/web/control-ui)
  - [Telegram](/channels/telegram)
  - [Discord](/channels/discord)
  - [Mattermost（插件）](/channels/mattermost)
  - [iMessage](/channels/imessage)
  - [群组](/concepts/groups)
  - [WhatsApp 群组消息](/concepts/group-messages)
  - [媒体：图像](/nodes/images)
  - [媒体：音频](/nodes/audio)
- 配套应用：
  - [macOS 应用](/platforms/macos)
  - [iOS 应用](/platforms/ios)
  - [Android 应用](/platforms/android)
  - [Windows (WSL2)](/platforms/windows)
  - [Linux 应用](/platforms/linux)
- 操作和安全：
  - [会话](/concepts/session)
  - [Cron 作业](/automation/cron-jobs)
  - [Webhooks](/automation/webhook)
  - [Gmail 钩子（Pub/Sub）](/automation/gmail-pubsub)
  - [安全](/gateway/security)
  - [故障排除](/gateway/troubleshooting)

## 名称

**Clawdbot = CLAW + TARDIS** — 因为每只太空龙虾都需要一台时间和空间机器。

---

*"We're all just playing with our own prompts."* — 一个 AI，可能对 token 上瘾

## 致谢

- **Peter Steinberger** ([@steipete](https://twitter.com/steipete)) — 创建者，龙虾低语者
- **Mario Zechner** ([@badlogicc](https://twitter.com/badlogicgames)) — Pi 创建者，安全渗透测试者
- **Clawd** — 要求更好名称的太空龙虾

## 核心贡献者

- **Maxim Vovshin** (@Hyaxia, 36747317+Hyaxia@users.noreply.github.com) — Blogwatcher 技能
- **Nacho Iacovino** (@nachoiacovino, nacho.iacovino@gmail.com) — 位置解析（Telegram + WhatsApp）

## 许可证

MIT — 像海洋中的龙虾一样自由 🦞

---

*"We're all just playing with our own prompts."* — 一个 AI，可能对 token 上瘾
