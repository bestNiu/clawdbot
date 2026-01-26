---
summary: "CLI onboarding wizard: guided setup for gateway, workspace, channels, and skills"
read_when:
  - Running or configuring the onboarding wizard
  - Setting up a new machine
---
# 引导向导（CLI）

引导向导是在 macOS、Linux 或 Windows（通过 WSL2；强烈推荐）上设置 Clawdbot 的**推荐**方式。
它在一个引导流程中配置本地 Gateway 或远程 Gateway 连接，以及通道、技能和工作区默认值。

主要入口点：

```bash
clawdbot onboard
```

后续重新配置：

```bash
clawdbot configure
```

推荐：设置 Brave Search API 密钥，以便代理可以使用 `web_search`（`web_fetch` 无需密钥即可工作）。最简单路径：`clawdbot configure --section web`，它存储 `tools.web.search.apiKey`。文档：[Web 工具](/tools/web）。

## QuickStart vs Advanced

向导以 **QuickStart**（默认值）vs **Advanced**（完全控制）开始。

**QuickStart** 保留默认值：
- 本地网关（loopback）
- 工作区默认（或现有工作区）
- 网关端口 **18789**
- 网关认证 **令牌**（自动生成，即使在 loopback 上）
- Tailscale 暴露 **关闭**
- Telegram + WhatsApp 私信默认为**允许列表**（系统会提示你输入电话号码）

**Advanced** 暴露每个步骤（模式、工作区、网关、通道、守护进程、技能）。

## 向导做什么

**本地模式（默认）**引导你完成：
  - 模型/认证（OpenAI Code (Codex) 订阅 OAuth、Anthropic API 密钥（推荐）或 setup-token（粘贴），以及 MiniMax/GLM/Moonshot/AI Gateway 选项）
- 工作区位置 + 引导文件
- 网关设置（端口/绑定/认证/tailscale）
- 提供者（Telegram、WhatsApp、Discord、Google Chat、Mattermost（插件）、Signal）
- 守护进程安装（LaunchAgent / systemd 用户单元）
- 健康检查
- 技能（推荐）

**远程模式**仅配置本地客户端以连接到其他地方的 Gateway。
它**不**在远程主机上安装或更改任何内容。

要添加更多隔离代理（单独的工作区 + 会话 + 认证），请使用：

```bash
clawdbot agents add <name>
```

提示：`--json` **不**意味着非交互模式。对于脚本，请使用 `--non-interactive`（和 `--workspace`）。

## 流程详细信息（本地）

1) **现有配置检测**
   - 如果 `~/.clawdbot/clawdbot.json` 存在，选择**保留 / 修改 / 重置**。
   - 重新运行向导**不**擦除任何内容，除非你明确选择**重置**（或传递 `--reset`）。
   - 如果配置无效或包含传统键，向导停止并要求你在继续之前运行 `clawdbot doctor`。
   - 重置使用 `trash`（永远不使用 `rm`）并提供作用域：
     - 仅配置
     - 配置 + 凭据 + 会话
     - 完全重置（也删除工作区）

2) **模型/认证**
   - **Anthropic API 密钥（推荐）**：如果存在则使用 `ANTHROPIC_API_KEY` 或提示输入密钥，然后保存以供守护进程使用。
   - **Anthropic OAuth（Claude Code CLI）**：在 macOS 上，向导检查 Keychain 项 "Claude Code-credentials"（选择"始终允许"以便 launchd 启动不会阻塞）；在 Linux/Windows 上，如果存在，它重用 `~/.claude/.credentials.json`。
   - **Anthropic 令牌（粘贴 setup-token）**：在任何机器上运行 `claude setup-token`，然后粘贴令牌（你可以命名它；空白 = 默认）。
   - **OpenAI Code (Codex) 订阅（Codex CLI）**：如果 `~/.codex/auth.json` 存在，向导可以重用它。
   - **OpenAI Code (Codex) 订阅（OAuth）**：浏览器流程；粘贴 `code#state`。
     - 当模型未设置或 `openai/*` 时，将 `agents.defaults.model` 设置为 `openai-codex/gpt-5.2`。
   - **OpenAI API 密钥**：如果存在则使用 `OPENAI_API_KEY` 或提示输入密钥，然后将其保存到 `~/.clawdbot/.env`，以便 launchd 可以读取它。
   - **OpenCode Zen（多模型代理）**：提示输入 `OPENCODE_API_KEY`（或 `OPENCODE_ZEN_API_KEY`，在 https://opencode.ai/auth 获取）。
   - **API 密钥**：为你存储密钥。
   - **Vercel AI Gateway（多模型代理）**：提示输入 `AI_GATEWAY_API_KEY`。
   - 更多详细信息：[Vercel AI Gateway](/providers/vercel-ai-gateway）
   - **MiniMax M2.1**：配置自动写入。
   - 更多详细信息：[MiniMax](/providers/minimax）
   - **Synthetic（Anthropic 兼容）**：提示输入 `SYNTHETIC_API_KEY`。
   - 更多详细信息：[Synthetic](/providers/synthetic）
   - **Moonshot (Kimi K2)**：配置自动写入。
   - **Kimi Code**：配置自动写入。
   - 更多详细信息：[Moonshot AI (Kimi + Kimi Code)](/providers/moonshot）
   - **跳过**：尚未配置认证。
   - 从检测到的选项中选择默认模型（或手动输入提供者/模型）。
   - 向导运行模型检查，如果配置的模型未知或缺少认证，则警告。
  - OAuth 凭据位于 `~/.clawdbot/credentials/oauth.json`；认证配置文件位于 `~/.clawdbot/agents/<agentId>/agent/auth-profiles.json`（API 密钥 + OAuth）。
   - 更多详细信息：[/concepts/oauth](/concepts/oauth）

3) **工作区**
   - 默认 `~/clawd`（可配置）。
   - 为代理引导仪式种子所需的工作区文件。
   - 完整工作区布局 + 备份指南：[代理工作区](/concepts/agent-workspace）

4) **Gateway**
   - 端口、绑定、认证模式、tailscale 暴露。
   - 认证推荐：即使在 loopback 上也保留**令牌**，以便本地 WS 客户端必须进行身份验证。
   - 仅在你完全信任每个本地进程时禁用认证。
   - 非 loopback 绑定仍需要认证。

5) **通道**
  - WhatsApp：可选 QR 登录。
  - Telegram：机器人令牌。
  - Discord：机器人令牌。
  - Google Chat：服务账户 JSON + webhook 受众。
  - Mattermost（插件）：机器人令牌 + 基础 URL。
   - Signal：可选 `signal-cli` 安装 + 账户配置。
   - iMessage：本地 `imsg` CLI 路径 + DB 访问。
  - 私信安全：默认是配对。第一条私信发送代码；通过 `clawdbot pairing approve <channel> <code>` 批准或使用允许列表。

6) **守护进程安装**
   - macOS：LaunchAgent
     - 需要登录的用户会话；对于无头，使用自定义 LaunchDaemon（不提供）。
   - Linux（以及通过 WSL2 的 Windows）：systemd 用户单元
     - 向导尝试通过 `loginctl enable-linger <user>` 启用 lingering，以便 Gateway 在注销后保持运行。
     - 可能提示 sudo（写入 `/var/lib/systemd/linger`）；它首先尝试不使用 sudo。
   - **运行时选择：** Node（推荐；WhatsApp/Telegram 需要）。**不推荐** Bun。

7) **健康检查**
   - 启动 Gateway（如果需要）并运行 `clawdbot health`。
   - 提示：`clawdbot status --deep` 将网关健康探测添加到状态输出（需要可访问的网关）。

8) **技能（推荐）**
   - 读取可用技能并检查要求。
   - 让你选择节点管理器：**npm / pnpm**（不推荐 bun）。
   - 安装可选依赖项（某些在 macOS 上使用 Homebrew）。

9) **完成**
   - 摘要 + 后续步骤，包括用于额外功能的 iOS/Android/macOS 应用程序。
  - 如果未检测到 GUI，向导打印控制 UI 的 SSH 端口转发说明，而不是打开浏览器。
  - 如果控制 UI 资源缺失，向导尝试构建它们；回退是 `pnpm ui:build`（自动安装 UI 依赖项）。

## 远程模式

远程模式配置本地客户端以连接到其他地方的 Gateway。

你将设置的内容：
- 远程 Gateway URL（`ws://...`）
- 如果远程 Gateway 需要认证，则为令牌（推荐）

注意：
- 不执行远程安装或守护进程更改。
- 如果 Gateway 仅 loopback，请使用 SSH 隧道或 tailnet。
- 发现提示：
  - macOS：Bonjour（`dns-sd`）
  - Linux：Avahi（`avahi-browse`）

## 添加另一个代理

使用 `clawdbot agents add <name>` 创建具有自己的工作区、会话和认证配置文件的单独代理。在没有 `--workspace` 的情况下运行会启动向导。

它设置的内容：
- `agents.list[].name`
- `agents.list[].workspace`
- `agents.list[].agentDir`

注意：
- 默认工作区遵循 `~/clawd-<agentId>`。
- 添加 `bindings` 以路由入站消息（向导可以执行此操作）。
- 非交互标志：`--model`、`--agent-dir`、`--bind`、`--non-interactive`。

## 非交互模式

使用 `--non-interactive` 自动化或编写引导脚本：

```bash
clawdbot onboard --non-interactive \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

添加 `--json` 以获取机器可读摘要。

Gemini 示例：

```bash
clawdbot onboard --non-interactive \
  --mode local \
  --auth-choice gemini-api-key \
  --gemini-api-key "$GEMINI_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Z.AI 示例：

```bash
clawdbot onboard --non-interactive \
  --mode local \
  --auth-choice zai-api-key \
  --zai-api-key "$ZAI_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Vercel AI Gateway 示例：

```bash
clawdbot onboard --non-interactive \
  --mode local \
  --auth-choice ai-gateway-api-key \
  --ai-gateway-api-key "$AI_GATEWAY_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Moonshot 示例：

```bash
clawdbot onboard --non-interactive \
  --mode local \
  --auth-choice moonshot-api-key \
  --moonshot-api-key "$MOONSHOT_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

Synthetic 示例：

```bash
clawdbot onboard --non-interactive \
  --mode local \
  --auth-choice synthetic-api-key \
  --synthetic-api-key "$SYNTHETIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

OpenCode Zen 示例：

```bash
clawdbot onboard --non-interactive \
  --mode local \
  --auth-choice opencode-zen \
  --opencode-zen-api-key "$OPENCODE_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback
```

添加代理（非交互）示例：

```bash
clawdbot agents add work \
  --workspace ~/clawd-work \
  --model openai/gpt-5.2 \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

## Gateway 向导 RPC

Gateway 通过 RPC（`wizard.start`、`wizard.next`、`wizard.cancel`、`wizard.status`）公开向导流程。
客户端（macOS 应用程序、控制 UI）可以渲染步骤，而无需重新实现引导逻辑。

## Signal 设置（signal-cli）

向导可以从 GitHub 发布版安装 `signal-cli`：
- 下载适当的发布资源。
- 将其存储在 `~/.clawdbot/tools/signal-cli/<version>/` 下。
- 将 `channels.signal.cliPath` 写入你的配置。

注意：
- JVM 构建需要 **Java 21**。
- 当可用时使用本机构建。
- Windows 使用 WSL2；signal-cli 安装在 WSL 内遵循 Linux 流程。

## 向导写入的内容

`~/.clawdbot/clawdbot.json` 中的典型字段：
- `agents.defaults.workspace`
- `agents.defaults.model` / `models.providers`（如果选择了 Minimax）
- `gateway.*`（模式、绑定、认证、tailscale）
- `channels.telegram.botToken`、`channels.discord.token`、`channels.signal.*`、`channels.imessage.*`
- 当你在提示期间选择加入时，通道允许列表（Slack/Discord/Matrix/Microsoft Teams）（名称在可能时解析为 ID）。
- `skills.install.nodeManager`
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`

`clawdbot agents add` 写入 `agents.list[]` 和可选的 `bindings`。

WhatsApp 凭据位于 `~/.clawdbot/credentials/whatsapp/<accountId>/` 下。
会话存储在 `~/.clawdbot/agents/<agentId>/sessions/` 下。

某些通道作为插件提供。当你在引导期间选择一个时，向导会在可以配置之前提示安装它（npm 或本地路径）。

## 相关文档

- macOS 应用程序引导：[引导](/start/onboarding）
- 配置参考：[Gateway 配置](/gateway/configuration）
- 提供者：[WhatsApp](/channels/whatsapp）、[Telegram](/channels/telegram）、[Discord](/channels/discord）、[Google Chat](/channels/googlechat）、[Signal](/channels/signal）、[iMessage](/channels/imessage）
- 技能：[技能](/tools/skills）、[技能配置](/tools/skills-config）
