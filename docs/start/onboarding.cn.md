---
summary: "First-run onboarding flow for Clawdbot (macOS app)"
read_when:
  - Designing the macOS onboarding assistant
  - Implementing auth or identity setup
---
# 引导（macOS 应用程序）

本文档描述了**当前**首次运行引导流程。目标是流畅的"第 0 天"体验：选择 Gateway 运行的位置，连接认证，运行向导，让代理引导自身。

## 页面顺序（当前）

1) 欢迎 + 安全通知
2) **Gateway 选择**（本地 / 远程 / 稍后配置）
3) **认证（Anthropic OAuth）** — 仅本地
4) **设置向导**（Gateway 驱动）
5) **权限**（TCC 提示）
6) **CLI**（可选）
7) **引导聊天**（专用会话）
8) 就绪

## 1) 本地 vs 远程

**Gateway** 在哪里运行？

- **本地（此 Mac）：** 引导可以在本地运行 OAuth 流程并写入凭据。
- **远程（通过 SSH/Tailnet）：** 引导**不**在本地运行 OAuth；凭据必须存在于网关主机上。
- **稍后配置：** 跳过设置并保持应用程序未配置。

Gateway 认证提示：
- 向导现在即使在 loopback 上也生成**令牌**，因此本地 WS 客户端必须进行身份验证。
- 如果你禁用认证，任何本地进程都可以连接；仅在完全受信任的机器上使用。
- 对于多机器访问或非 loopback 绑定，使用**令牌**。

## 2) 仅本地认证（Anthropic OAuth）

macOS 应用程序支持 Anthropic OAuth（Claude Pro/Max）。流程：

- 打开浏览器进行 OAuth（PKCE）
- 要求用户粘贴 `code#state` 值
- 将凭据写入 `~/.clawdbot/credentials/oauth.json`

其他提供者（OpenAI、自定义 API）目前通过环境变量或配置文件配置。

## 3) 设置向导（Gateway 驱动）

应用程序可以运行与 CLI 相同的设置向导。这使引导与 Gateway 端行为保持同步，并避免在 SwiftUI 中重复逻辑。

## 4) 权限

引导请求 TCC 权限，需要：

- 通知
- 辅助功能
- 屏幕录制
- 麦克风 / 语音识别
- 自动化（AppleScript）

## 5) CLI（可选）

应用程序可以通过 npm/pnpm 安装全局 `clawdbot` CLI，以便终端工作流和 launchd 任务开箱即用。

## 6) 引导聊天（专用会话）

设置后，应用程序打开专用引导聊天会话，以便代理可以介绍自己并指导后续步骤。这使首次运行指导与你的正常对话分开。

## 代理引导仪式

在第一次代理运行时，Clawdbot 引导工作区（默认 `~/clawd`）：

- 种子 `AGENTS.md`、`BOOTSTRAP.md`、`IDENTITY.md`、`USER.md`
- 运行简短的问答仪式（一次一个问题）
- 将身份 + 首选项写入 `IDENTITY.md`、`USER.md`、`SOUL.md`
- 完成后删除 `BOOTSTRAP.md`，因此它只运行一次

## 可选：Gmail hooks（手动）

Gmail Pub/Sub 设置目前是手动步骤。使用：

```bash
clawdbot webhooks gmail setup --account you@gmail.com
```

有关详细信息，请参见 [/automation/gmail-pubsub](/automation/gmail-pubsub）。

## 远程模式说明

当 Gateway 在另一台机器上运行时，凭据和工作区文件**在该主机上**。如果你需要在远程模式下进行 OAuth，请创建：

- `~/.clawdbot/credentials/oauth.json`
- `~/.clawdbot/agents/<agentId>/agent/auth-profiles.json`

在网关主机上。
