---
summary: "Direct `clawdbot agent` CLI runs (with optional delivery)"
read_when:
  - Adding or modifying the agent CLI entrypoint
---
# `clawdbot agent`（直接代理运行）

`clawdbot agent` 运行单个代理回合，无需入站聊天消息。
默认情况下，它**通过 Gateway**；添加 `--local` 以强制在当前机器上使用嵌入式运行时。

## 行为

- 必需：`--message <text>`
- 会话选择：
  - `--to <dest>` 派生会话键（群组/通道目标保留隔离；直接聊天折叠为 `main`），**或**
  - `--session-id <id>` 通过 id 重用现有会话，**或**
  - `--agent <id>` 直接定位配置的代理（使用该代理的 `main` 会话键）
- 运行与正常入站回复相同的嵌入式代理运行时。
- 思考/详细标志持久化到会话存储。
- 输出：
  - 默认：打印回复文本（加上 `MEDIA:<url>` 行）
  - `--json`：打印结构化负载 + 元数据
- 使用 `--deliver` + `--channel` 可选传递回通道（目标格式匹配 `clawdbot message --target`）。
- 使用 `--reply-channel`/`--reply-to`/`--reply-account` 覆盖传递而不更改会话。

如果 Gateway 无法访问，CLI **回退**到嵌入式本地运行。

## 示例

```bash
clawdbot agent --to +15555550123 --message "status update"
clawdbot agent --agent ops --message "Summarize logs"
clawdbot agent --session-id 1234 --message "Summarize inbox" --thinking medium
clawdbot agent --to +15555550123 --message "Trace logs" --verbose on --json
clawdbot agent --to +15555550123 --message "Summon reply" --deliver
clawdbot agent --agent ops --message "Generate report" --deliver --reply-channel slack --reply-to "#reports"
```

## 标志

- `--local`：本地运行（需要 shell 中的模型提供者 API 密钥）
- `--deliver`：将回复发送到所选通道
- `--channel`：传递通道（`whatsapp|telegram|discord|googlechat|slack|signal|imessage`，默认：`whatsapp`）
- `--reply-to`：传递目标覆盖
- `--reply-channel`：传递通道覆盖
- `--reply-account`：传递账户 id 覆盖
- `--thinking <off|minimal|low|medium|high|xhigh>`：持久化思考级别（仅 GPT-5.2 + Codex 模型）
- `--verbose <on|full|off>`：持久化详细级别
- `--timeout <seconds>`：覆盖代理超时
- `--json`：输出结构化 JSON
