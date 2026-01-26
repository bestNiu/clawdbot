---
summary: "Elevated exec mode and /elevated directives"
read_when:
  - Adjusting elevated mode defaults, allowlists, or slash command behavior
---
# 提升模式（/elevated 指令）

## 它做什么
- `/elevated on` 在网关主机上运行并保持 exec 批准（与 `/elevated ask` 相同）。
- `/elevated full` 在网关主机上运行**并**自动批准 exec（跳过 exec 批准）。
- `/elevated ask` 在网关主机上运行但保持 exec 批准（与 `/elevated on` 相同）。
- `on`/`ask` **不**强制 `exec.security=full`；配置的安全/ask 策略仍然适用。
- 仅在代理被**沙箱化**时更改行为（否则 exec 已经在主机上运行）。
- 指令形式：`/elevated on|off|ask|full`、`/elev on|off|ask|full`。
- 仅接受 `on|off|ask|full`；其他任何内容返回提示且不更改状态。

## 它控制什么（以及不控制什么）
- **可用性门控**：`tools.elevated` 是全局基线。`agents.list[].tools.elevated` 可以进一步限制每个代理的提升（两者都必须允许）。
- **每个会话状态**：`/elevated on|off|ask|full` 为当前会话键设置提升级别。
- **内联指令**：消息内的 `/elevated on|ask|full` 仅适用于该消息。
- **群组**：在群组聊天中，提升指令仅在代理被提及时才被遵守。绕过提及要求的仅命令消息被视为已提及。
- **主机执行**：提升强制 `exec` 到网关主机；`full` 还设置 `security=full`。
- **批准**：`full` 跳过 exec 批准；`on`/`ask` 在允许列表/ask 规则要求时遵守它们。
- **未沙箱化代理**：位置无操作；仅影响门控、日志记录和状态。
- **工具策略仍然适用**：如果 `exec` 被工具策略拒绝，提升无法使用。

## 解析顺序
1. 消息上的内联指令（仅适用于该消息）。
2. 会话覆盖（通过发送仅指令消息设置）。
3. 全局默认（配置中的 `agents.defaults.elevatedDefault`）。

## 设置会话默认值
- 发送**仅**指令的消息（允许空格），例如 `/elevated full`。
- 发送确认回复（`Elevated mode set to full...` / `Elevated mode disabled.`）。
- 如果提升访问被禁用或发送者不在批准的允许列表上，指令回复可操作的错误且不更改会话状态。
- 发送 `/elevated`（或 `/elevated:`）不带参数以查看当前提升级别。

## 可用性 + 允许列表
- 功能门控：`tools.elevated.enabled`（即使代码支持它，默认也可以通过配置关闭）。
- 发送者允许列表：`tools.elevated.allowFrom` 带有每个提供者允许列表（例如 `discord`、`whatsapp`）。
- 每个代理门控：`agents.list[].tools.elevated.enabled`（可选；只能进一步限制）。
- 每个代理允许列表：`agents.list[].tools.elevated.allowFrom`（可选；当设置时，发送者必须匹配**两者**全局 + 每个代理允许列表）。
- Discord 回退：如果省略 `tools.elevated.allowFrom.discord`，使用 `channels.discord.dm.allowFrom` 列表作为回退。设置 `tools.elevated.allowFrom.discord`（即使是 `[]`）以覆盖。每个代理允许列表**不**使用回退。
- 所有门控必须通过；否则提升被视为不可用。

## 日志记录 + 状态
- 提升 exec 调用在 info 级别记录。
- 会话状态包括提升模式（例如 `elevated=ask`、`elevated=full`）。
