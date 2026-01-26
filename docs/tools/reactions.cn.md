---
summary: "Reaction semantics shared across channels"
read_when:
  - Working on reactions in any channel
---
# 反应工具

跨通道共享的反应语义：

- 添加反应时需要 `emoji`。
- `emoji=""` 在支持时删除机器人的反应。
- `remove: true` 在支持时删除指定的表情符号（需要 `emoji`）。

通道说明：

- **Discord/Slack**：空 `emoji` 删除消息上机器人的所有反应；`remove: true` 仅删除该表情符号。
- **Google Chat**：空 `emoji` 删除应用程序在消息上的反应；`remove: true` 仅删除该表情符号。
- **Telegram**：空 `emoji` 删除机器人的反应；`remove: true` 也删除反应，但仍需要非空 `emoji` 进行工具验证。
- **WhatsApp**：空 `emoji` 删除机器人反应；`remove: true` 映射到空表情符号（仍需要 `emoji`）。
- **Signal**：当启用 `channels.signal.reactionNotifications` 时，入站反应通知发出系统事件。
