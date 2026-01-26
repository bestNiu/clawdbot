---
summary: "Terminal UI (TUI): connect to the Gateway from any machine"
read_when:
  - You want a beginner-friendly walkthrough of the TUI
  - You need the complete list of TUI features, commands, and shortcuts
---
# TUI（终端 UI）

## 快速开始
1) 启动 Gateway。
```bash
clawdbot gateway
```
2) 打开 TUI。
```bash
clawdbot tui
```
3) 输入消息并按 Enter。

远程 Gateway：
```bash
clawdbot tui --url ws://<host>:<port> --token <gateway-token>
```
如果你的 Gateway 使用密码认证，请使用 `--password`。

## 你看到的内容
- 标头：连接 URL、当前代理、当前会话。
- 聊天日志：用户消息、助手回复、系统通知、工具卡片。
- 状态行：连接/运行状态（连接中、运行中、流式传输、空闲、错误）。
- 页脚：连接状态 + 代理 + 会话 + 模型 + 思考/详细/推理 + token 计数 + 传递。
- 输入：带自动完成的文本编辑器。

## 心理模型：代理 + 会话
- 代理是唯一的 slug（例如 `main`、`research`）。Gateway 公开列表。
- 会话属于当前代理。
- 会话键存储为 `agent:<agentId>:<sessionKey>`。
  - 如果你输入 `/session main`，TUI 将其扩展为 `agent:<currentAgent>:main`。
  - 如果你输入 `/session agent:other:main`，你显式切换到该代理会话。
- 会话作用域：
  - `per-sender`（默认）：每个代理有许多会话。
  - `global`：TUI 始终使用 `global` 会话（选择器可能为空）。
- 当前代理 + 会话始终在页脚中可见。

## 发送 + 传递
- 消息发送到 Gateway；默认关闭到提供者的传递。
- 开启传递：
  - `/deliver on`
  - 或设置面板
  - 或使用 `clawdbot tui --deliver` 启动

## 选择器 + 覆盖层
- 模型选择器：列出可用模型并设置会话覆盖。
- 代理选择器：选择不同的代理。
- 会话选择器：仅显示当前代理的会话。
- 设置：切换传递、工具输出展开和思考可见性。

## 键盘快捷键
- Enter：发送消息
- Esc：中止活动运行
- Ctrl+C：清除输入（按两次退出）
- Ctrl+D：退出
- Ctrl+L：模型选择器
- Ctrl+G：代理选择器
- Ctrl+P：会话选择器
- Ctrl+O：切换工具输出展开
- Ctrl+T：切换思考可见性（重新加载历史）

## 斜杠命令
核心：
- `/help`
- `/status`
- `/agent <id>`（或 `/agents`）
- `/session <key>`（或 `/sessions`）
- `/model <provider/model>`（或 `/models`）

会话控制：
- `/think <off|minimal|low|medium|high>`
- `/verbose <on|full|off>`
- `/reasoning <on|off|stream>`
- `/usage <off|tokens|full>`
- `/elevated <on|off|ask|full>`（别名：`/elev`）
- `/activation <mention|always>`
- `/deliver <on|off>`

会话生命周期：
- `/new` 或 `/reset`（重置会话）
- `/abort`（中止活动运行）
- `/settings`
- `/exit`

其他 Gateway 斜杠命令（例如，`/context`）转发到 Gateway 并显示为系统输出。参见 [斜杠命令](/tools/slash-commands)。

## 本地 shell 命令
- 在行前加上 `!` 以在 TUI 主机上运行本地 shell 命令。
- TUI 每个会话提示一次以允许本地执行；拒绝保持 `!` 在该会话中禁用。
- 命令在 TUI 工作目录中的全新、非交互式 shell 中运行（无持久 `cd`/env）。
- 单独的 `!` 作为正常消息发送；前导空格不触发本地 exec。

## 工具输出
- 工具调用显示为带有参数 + 结果的卡片。
- Ctrl+O 在折叠/展开视图之间切换。
- 工具运行时，部分更新流式传输到同一卡片。

## 历史 + 流式传输
- 连接时，TUI 加载最新历史（默认 200 条消息）。
- 流式响应在最终确定之前就地更新。
- TUI 还监听代理工具事件以获取更丰富的工具卡片。

## 连接详细信息
- TUI 以 `mode: "tui"` 注册到 Gateway。
- 重新连接显示系统消息；事件间隙在日志中显示。

## 选项
- `--url <url>`：Gateway WebSocket URL（默认为配置或 `ws://127.0.0.1:<port>`）
- `--token <token>`：Gateway 令牌（如果需要）
- `--password <password>`：Gateway 密码（如果需要）
- `--session <key>`：会话键（默认：`main`，或当作用域为 global 时为 `global`）
- `--deliver`：将助手回复传递到提供者（默认关闭）
- `--thinking <level>`：覆盖发送的思考级别
- `--timeout-ms <ms>`：代理超时（毫秒）（默认为 `agents.defaults.timeoutSeconds`）

## 故障排除

发送消息后没有输出：
- 在 TUI 中运行 `/status` 以确认 Gateway 已连接且空闲/忙碌。
- 检查 Gateway 日志：`clawdbot logs --follow`。
- 确认代理可以运行：`clawdbot status` 和 `clawdbot models status`。
- 如果你期望聊天通道中的消息，启用传递（`/deliver on` 或 `--deliver`）。
- `--history-limit <n>`：要加载的历史条目（默认 200）

## 故障排除
- `disconnected`：确保 Gateway 正在运行，并且你的 `--url/--token/--password` 正确。
- 选择器中没有代理：检查 `clawdbot agents list` 和你的路由配置。
- 空会话选择器：你可能处于 global 作用域或还没有会话。
