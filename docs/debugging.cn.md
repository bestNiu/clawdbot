---
summary: "Debugging tools: watch mode, raw model streams, and tracing reasoning leakage"
read_when:
  - You need to inspect raw model output for reasoning leakage
  - You want to run the Gateway in watch mode while iterating
  - You need a repeatable debugging workflow
---

# 调试

本页介绍用于流式输出的调试助手，特别是当提供者将推理混合到普通文本中时。

## 运行时调试覆盖

使用聊天中的 `/debug` 设置**仅运行时**配置覆盖（内存，非磁盘）。
`/debug` 默认禁用；使用 `commands.debug: true` 启用。
当你需要切换晦涩的设置而不编辑 `clawdbot.json` 时，这很方便。

示例：

```
/debug show
/debug set messages.responsePrefix="[clawdbot]"
/debug unset messages.responsePrefix
/debug reset
```

`/debug reset` 清除所有覆盖并返回到磁盘配置。

## Gateway 监视模式

为了快速迭代，在文件监视器下运行网关：

```bash
pnpm gateway:watch --force
```

这映射到：

```bash
tsx watch src/entry.ts gateway --force
```

在 `gateway:watch` 之后添加任何网关 CLI 标志，它们将在每次重启时传递。

## 开发配置文件 + 开发网关 (--dev)

使用开发配置文件隔离状态并启动安全的、可丢弃的设置进行调试。有**两个** `--dev` 标志：

- **全局 `--dev`（配置文件）：** 在 `~/.clawdbot-dev` 下隔离状态，并将网关端口默认为 `19001`（派生端口随其移动）。
- **`gateway --dev`：告诉 Gateway 在缺少时自动创建默认配置 + 工作区**（并跳过 BOOTSTRAP.md）。

推荐流程（开发配置文件 + 开发引导）：

```bash
pnpm gateway:dev
CLAWDBOT_PROFILE=dev clawdbot tui
```

如果你还没有全局安装，请通过 `pnpm clawdbot ...` 运行 CLI。

这做什么：

1) **配置文件隔离**（全局 `--dev`）
   - `CLAWDBOT_PROFILE=dev`
   - `CLAWDBOT_STATE_DIR=~/.clawdbot-dev`
   - `CLAWDBOT_CONFIG_PATH=~/.clawdbot-dev/clawdbot.json`
   - `CLAWDBOT_GATEWAY_PORT=19001`（浏览器/canvas 相应移动）

2) **开发引导**（`gateway --dev`）
   - 如果缺少，写入最小配置（`gateway.mode=local`，绑定环回）。
   - 将 `agent.workspace` 设置为开发工作区。
   - 设置 `agent.skipBootstrap=true`（无 BOOTSTRAP.md）。
   - 如果缺少，播种工作区文件：
     `AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`。
   - 默认身份：**C3‑PO**（协议机器人）。
   - 在开发模式下跳过通道提供者（`CLAWDBOT_SKIP_CHANNELS=1`）。

重置流程（全新开始）：

```bash
pnpm gateway:dev:reset
```

注意：`--dev` 是一个**全局**配置文件标志，会被某些运行器消耗。
如果你需要明确说明，请使用环境变量形式：

```bash
CLAWDBOT_PROFILE=dev clawdbot gateway --dev --reset
```

`--reset` 擦除配置、凭据、会话和开发工作区（使用 `trash`，而不是 `rm`），然后重新创建默认开发设置。

提示：如果非开发网关已在运行（launchd/systemd），请先停止它：

```bash
clawdbot gateway stop
```

## 原始流日志记录（Clawdbot）

Clawdbot 可以在任何过滤/格式化之前记录**原始助手流**。
这是查看推理是否作为纯文本增量到达（或作为单独的思考块）的最佳方法。

通过 CLI 启用：

```bash
pnpm gateway:watch --force --raw-stream
```

可选路径覆盖：

```bash
pnpm gateway:watch --force --raw-stream --raw-stream-path ~/.clawdbot/logs/raw-stream.jsonl
```

等效环境变量：

```bash
CLAWDBOT_RAW_STREAM=1
CLAWDBOT_RAW_STREAM_PATH=~/.clawdbot/logs/raw-stream.jsonl
```

默认文件：

`~/.clawdbot/logs/raw-stream.jsonl`

## 原始块日志记录（pi-mono）

要捕获在解析为块之前的**原始 OpenAI 兼容块**，pi-mono 公开了一个单独的记录器：

```bash
PI_RAW_STREAM=1
```

可选路径：

```bash
PI_RAW_STREAM_PATH=~/.pi-mono/logs/raw-openai-completions.jsonl
```

默认文件：

`~/.pi-mono/logs/raw-openai-completions.jsonl`

> 注意：这仅由使用 pi-mono 的 `openai-completions` 提供者的进程发出。

## 安全说明

- 原始流日志可能包括完整的提示、工具输出和用户数据。
- 保持日志本地，调试后删除它们。
- 如果你共享日志，请先清理秘密和 PII。
