---
summary: "Setup guide: keep your Clawdbot setup tailored while staying up-to-date"
read_when:
  - Setting up a new machine
  - You want "latest + greatest" without breaking your personal setup
---

# 设置

最后更新：2026-01-01

## TL;DR
- **定制位于仓库外：** `~/clawd`（工作区）+ `~/.clawdbot/clawdbot.json`（配置）。
- **稳定工作流：** 安装 macOS 应用程序；让它运行捆绑的 Gateway。
- **前沿工作流：** 通过 `pnpm gateway:watch` 自己运行 Gateway，然后让 macOS 应用程序以本地模式附加。

## 先决条件（从源代码）
- Node `>=22`
- `pnpm`
- Docker（可选；仅用于容器化设置/e2e — 参见 [Docker](/install/docker））

## 定制策略（以便更新不会造成损害）

如果你想要"100% 为我定制"*并且*易于更新，请将你的自定义保留在：

- **配置：** `~/.clawdbot/clawdbot.json`（JSON/JSON5 风格）
- **工作区：** `~/clawd`（技能、提示、记忆；使其成为私有 git 仓库）

引导一次：

```bash
clawdbot setup
```

从此仓库内部，使用本地 CLI 入口：

```bash
clawdbot setup
```

如果你还没有全局安装，请通过 `pnpm clawdbot setup` 运行它。

## 稳定工作流（macOS 应用程序优先）

1) 安装 + 启动 **Clawdbot.app**（菜单栏）。
2) 完成引导/权限清单（TCC 提示）。
3) 确保 Gateway 是**本地**且正在运行（应用程序管理它）。
4) 链接界面（示例：WhatsApp）：

```bash
clawdbot channels login
```

5) 健全性检查：

```bash
clawdbot health
```

如果你的构建中不可用引导：
- 运行 `clawdbot setup`，然后 `clawdbot channels login`，然后手动启动 Gateway（`clawdbot gateway`）。

## 前沿工作流（终端中的 Gateway）

目标：在 TypeScript Gateway 上工作，获得热重载，保持 macOS 应用程序 UI 附加。

### 0) （可选）也从源代码运行 macOS 应用程序

如果你也希望 macOS 应用程序处于前沿：

```bash
./scripts/restart-mac.sh
```

### 1) 启动开发 Gateway

```bash
pnpm install
pnpm gateway:watch
```

`gateway:watch` 在监视模式下运行网关，并在 TypeScript 更改时重新加载。

### 2) 将 macOS 应用程序指向你正在运行的 Gateway

在 **Clawdbot.app** 中：

- 连接模式：**本地**
应用程序将附加到配置端口上正在运行的网关。

### 3) 验证

- 应用内 Gateway 状态应显示**"使用现有网关 …"**
- 或通过 CLI：

```bash
clawdbot health
```

### 常见陷阱
- **错误端口：** Gateway WS 默认为 `ws://127.0.0.1:18789`；保持应用程序 + CLI 在同一端口。
- **状态存储位置：**
  - 凭据：`~/.clawdbot/credentials/`
  - 会话：`~/.clawdbot/agents/<agentId>/sessions/`
  - 日志：`/tmp/clawdbot/`

## 更新（不破坏你的设置）

- 将 `~/clawd` 和 `~/.clawdbot/` 保留为"你的东西"；不要将个人提示/配置放入 `clawdbot` 仓库。
- 更新源代码：`git pull` + `pnpm install`（当 lockfile 更改时）+ 继续使用 `pnpm gateway:watch`。

## Linux（systemd 用户服务）

Linux 安装使用 systemd **用户**服务。默认情况下，systemd 在注销/空闲时停止用户服务，这会杀死 Gateway。引导尝试为你启用 lingering（可能提示 sudo）。如果它仍然关闭，请运行：

```bash
sudo loginctl enable-linger $USER
```

对于始终在线或多用户服务器，考虑使用**系统**服务而不是用户服务（不需要 lingering）。有关 systemd 说明，请参见 [Gateway 运行手册](/gateway）。

## 相关文档

- [Gateway 运行手册](/gateway）（标志、监督、端口）
- [Gateway 配置](/gateway/configuration）（配置模式 + 示例）
- [Discord](/channels/discord）和 [Telegram](/channels/telegram）（回复标签 + replyToMode 设置）
- [Clawdbot 助手设置](/start/clawd）
- [macOS 应用程序](/platforms/macos）（网关生命周期）
