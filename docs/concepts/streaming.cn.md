---
summary: "Streaming + chunking behavior (block replies, draft streaming, limits)"
read_when:
  - Explaining how streaming or chunking works on channels
  - Changing block streaming or channel chunking behavior
  - Debugging duplicate/early block replies or draft streaming
---
# 流式传输 + 分块

Clawdbot 有两个独立的"流式传输"层：
- **块流式传输（通道）：** 在助手写入时发出已完成的**块**。这些是正常的通道消息（不是 token 增量）。
- **类似 Token 的流式传输（仅 Telegram）：** 在生成时使用部分文本更新**草稿气泡**；最终消息在结束时发送。

今天**没有真正的 token 流式传输**到外部通道消息。Telegram 草稿流式传输是唯一的部分流界面。

## 块流式传输（通道消息）

块流式传输在可用时以粗块发送助手输出。

```
Model output
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ chunker emits blocks as buffer grows
       └─ (blockStreamingBreak=message_end)
            └─ chunker flushes at message_end
                   └─ channel send (block replies)
```
图例：
- `text_delta/events`：模型流事件（对于非流式模型可能是稀疏的）。
- `chunker`：`EmbeddedBlockChunker` 应用最小/最大边界 + 中断偏好。
- `channel send`：实际出站消息（块回复）。

**控制：**
- `agents.defaults.blockStreamingDefault`：`"on"`/`"off"`（默认关闭）。
- 通道覆盖：`*.blockStreaming`（以及每个账户变体）以强制每个通道 `"on"`/`"off"`。
- `agents.defaults.blockStreamingBreak`：`"text_end"` 或 `"message_end"`。
- `agents.defaults.blockStreamingChunk`：`{ minChars, maxChars, breakPreference? }`。
- `agents.defaults.blockStreamingCoalesce`：`{ minChars?, maxChars?, idleMs? }`（在发送前合并流式块）。
- 通道硬上限：`*.textChunkLimit`（例如，`channels.whatsapp.textChunkLimit`）。
- 通道分块模式：`*.chunkMode`（`length` 默认，`newline` 在长度分块之前在空行（段落边界）上拆分）。
- Discord 软上限：`channels.discord.maxLinesPerMessage`（默认 17）拆分高回复以避免 UI 裁剪。

**边界语义：**
- `text_end`：一旦 chunker 发出就流式传输块；在每个 `text_end` 刷新。
- `message_end`：等到助手消息完成，然后刷新缓冲的输出。

`message_end` 如果缓冲的文本超过 `maxChars`，仍使用 chunker，因此它可以在结束时发出多个块。

## 分块算法（低/高边界）

块分块由 `EmbeddedBlockChunker` 实现：
- **低边界：** 在缓冲区 >= `minChars` 之前不发出（除非强制）。
- **高边界：** 在 `maxChars` 之前优先拆分；如果强制，在 `maxChars` 拆分。
- **中断偏好：** `paragraph` → `newline` → `sentence` → `whitespace` → 硬中断。
- **代码围栏：** 永远不在围栏内拆分；当在 `maxChars` 强制时，关闭 + 重新打开围栏以保持 Markdown 有效。

`maxChars` 被限制到通道 `textChunkLimit`，因此你不能超过每个通道的上限。

## 合并（合并流式块）

当启用块流式传输时，Clawdbot 可以在发送之前**合并连续的块块**。这减少了"单行垃圾邮件"，同时仍提供渐进式输出。

- 合并等待**空闲间隙**（`idleMs`）然后刷新。
- 缓冲区由 `maxChars` 限制，如果超过它，将刷新。
- `minChars` 防止微小片段发送，直到累积足够的文本（最终刷新始终发送剩余文本）。
- 连接器从 `blockStreamingChunk.breakPreference` 派生（`paragraph` → `\n\n`，`newline` → `\n`，`sentence` → 空格）。
- 通过 `*.blockStreamingCoalesce`（包括每个账户配置）提供通道覆盖。
- 除非覆盖，否则 Signal/Slack/Discord 的默认合并 `minChars` 提升到 1500。

## 块之间类似人类的节奏

当启用块流式传输时，你可以在块回复之间添加**随机暂停**（在第一个块之后）。这使多气泡响应感觉更自然。

- 配置：`agents.defaults.humanDelay`（通过 `agents.list[].humanDelay` 覆盖每个代理）。
- 模式：`off`（默认），`natural`（800–2500ms），`custom`（`minMs`/`maxMs`）。
- 仅适用于**块回复**，不适用于最终回复或工具摘要。

## "流式块或所有内容"

这映射到：
- **流式块：** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"`（边走边发出）。非 Telegram 通道还需要 `*.blockStreaming: true`。
- **在结束时流式传输所有内容：** `blockStreamingBreak: "message_end"`（刷新一次，如果很长可能多个块）。
- **无块流式传输：** `blockStreamingDefault: "off"`（仅最终回复）。

**通道注意：** 对于非 Telegram 通道，块流式传输**关闭，除非**`*.blockStreaming` 显式设置为 `true`。Telegram 可以在没有块回复的情况下流式传输草稿（`channels.telegram.streamMode`）。

配置位置提醒：`blockStreaming*` 默认值位于 `agents.defaults` 下，而不是根配置。

## Telegram 草稿流式传输（类似 token）

Telegram 是唯一具有草稿流式传输的通道：
- 在**带主题的私聊**中使用 Bot API `sendMessageDraft`。
- `channels.telegram.streamMode`：`"partial" | "block" | "off"`。
  - `partial`：使用最新流文本更新草稿。
  - `block`：在分块块中更新草稿（相同的 chunker 规则）。
  - `off`：无草稿流式传输。
- 草稿分块配置（仅用于 `streamMode: "block"`）：`channels.telegram.draftChunk`（默认值：`minChars: 200`，`maxChars: 800`）。
- 草稿流式传输与块流式传输分开；块回复默认关闭，仅在非 Telegram 通道上通过 `*.blockStreaming: true` 启用。
- 最终回复仍然是正常消息。
- `/reasoning stream` 将推理写入草稿气泡（仅 Telegram）。

当草稿流式传输处于活动状态时，Clawdbot 禁用该回复的块流式传输以避免双重流式传输。

```
Telegram (private + topics)
  └─ sendMessageDraft (draft bubble)
       ├─ streamMode=partial → update latest text
       └─ streamMode=block   → chunker updates draft
  └─ final reply → normal message
```
图例：
- `sendMessageDraft`：Telegram 草稿气泡（不是真实消息）。
- `final reply`：正常 Telegram 消息发送。
