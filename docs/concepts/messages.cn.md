---
summary: "Message flow, sessions, queueing, and reasoning visibility"
read_when:
  - Explaining how inbound messages become replies
  - Clarifying sessions, queueing modes, or streaming behavior
  - Documenting reasoning visibility and usage implications
---
# 消息

本页将 Clawdbot 如何处理入站消息、会话、排队、流式传输和推理可见性联系在一起。

## 消息流（高级）

```
Inbound message
  -> routing/bindings -> session key
  -> queue (if a run is active)
  -> agent run (streaming + tools)
  -> outbound replies (channel limits + chunking)
```

关键旋钮位于配置中：
- `messages.*` 用于前缀、排队和群组行为。
- `agents.defaults.*` 用于块流式传输和分块默认值。
- 通道覆盖（`channels.whatsapp.*`、`channels.telegram.*` 等）用于上限和流式传输切换。

有关完整模式，请参见 [配置](/gateway/configuration)。

## 入站去重

通道可以在重新连接后重新传递相同的消息。Clawdbot 保持一个短期缓存，由通道/账户/对等点/会话/消息 ID 键控，以便重复传递不会触发另一个代理运行。

## 入站防抖

来自**同一发送者**的快速连续消息可以通过 `messages.inbound` 批处理为单个代理回合。防抖作用域为每个通道 + 对话，并使用最新消息进行回复线程/ID。

配置（全局默认 + 每个通道覆盖）：
```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
        discord: 1500
      }
    }
  }
}
```

注意：
- 防抖适用于**仅文本**消息；媒体/附件立即刷新。
- 控制命令绕过防抖，因此它们保持独立。

## 会话和设备

会话由网关拥有，而不是客户端。
- 直接聊天折叠到代理主会话键。
- 群组/通道获得自己的会话键。
- 会话存储和转录本位于网关主机上。

多个设备/通道可以映射到同一个会话，但历史不会完全同步回每个客户端。建议：使用一个主设备进行长对话，以避免分歧上下文。控制 UI 和 TUI 始终显示网关支持的会话转录本，因此它们是真相来源。

详细信息：[会话管理](/concepts/session)。

## 入站正文和历史上下文

Clawdbot 将**提示正文**与**命令正文**分开：
- `Body`：发送给代理的提示文本。这可能包括通道信封和可选的历史包装器。
- `CommandBody`：用于指令/命令解析的原始用户文本。
- `RawBody`：`CommandBody` 的传统别名（为兼容性保留）。

当通道提供历史时，它使用共享包装器：
- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

对于**非直接聊天**（群组/通道/房间），**当前消息正文**以发送者标签为前缀（与历史条目使用的相同样式）。这使实时和排队/历史消息在代理提示中保持一致。

历史缓冲区是**仅待处理的**：它们包括*未*触发运行的群组消息（例如，提及门控消息），并**排除**已在会话转录本中的消息。

指令剥离仅适用于**当前消息**部分，因此历史保持完整。包装历史的通道应将 `CommandBody`（或 `RawBody`）设置为原始消息文本，并将 `Body` 保持为组合提示。
历史缓冲区可通过 `messages.groupChat.historyLimit`（全局默认）和每个通道覆盖（如 `channels.slack.historyLimit` 或 `channels.telegram.accounts.<id>.historyLimit`）配置（设置 `0` 以禁用）。

## 排队和后续

如果运行已激活，入站消息可以排队、引导到当前运行，或收集用于后续回合。

- 通过 `messages.queue`（和 `messages.queue.byChannel`）配置。
- 模式：`interrupt`、`steer`、`followup`、`collect`，加上积压变体。

详细信息：[排队](/concepts/queue)。

## 流式传输、分块和批处理

块流式传输在模型生成文本块时发送部分回复。
分块尊重通道文本限制并避免拆分围栏代码。

关键设置：
- `agents.defaults.blockStreamingDefault`（`on|off`，默认关闭）
- `agents.defaults.blockStreamingBreak`（`text_end|message_end`）
- `agents.defaults.blockStreamingChunk`（`minChars|maxChars|breakPreference`）
- `agents.defaults.blockStreamingCoalesce`（基于空闲的批处理）
- `agents.defaults.humanDelay`（块回复之间类似人类的暂停）
- 通道覆盖：`*.blockStreaming` 和 `*.blockStreamingCoalesce`（非 Telegram 通道需要显式 `*.blockStreaming: true`）

详细信息：[流式传输 + 分块](/concepts/streaming)。

## 推理可见性和 tokens

Clawdbot 可以暴露或隐藏模型推理：
- `/reasoning on|off|stream` 控制可见性。
- 推理内容在由模型生成时仍计入 token 使用。
- Telegram 支持将推理流式传输到草稿气泡中。

详细信息：[思考 + 推理指令](/tools/thinking) 和 [Token 使用](/token-use)。

## 前缀、线程和回复

出站消息格式集中在 `messages` 中：
- `messages.responsePrefix`（出站前缀）和 `channels.whatsapp.messagePrefix`（WhatsApp 入站前缀）
- 通过 `replyToMode` 和每个通道默认值进行回复线程

详细信息：[配置](/gateway/configuration#messages) 和通道文档。
