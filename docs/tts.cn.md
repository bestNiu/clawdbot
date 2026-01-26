---
summary: "Text-to-speech (TTS) for outbound replies"
read_when:
  - Enabling text-to-speech for replies
  - Configuring TTS providers or limits
  - Using /tts commands
---

# 文本转语音（TTS）

Clawdbot 可以使用 ElevenLabs、OpenAI 或 Edge TTS 将出站回复转换为音频。
它可以在 Clawdbot 可以发送音频的任何地方工作；Telegram 获得一个圆形语音笔记气泡。

## 支持的服务

- **ElevenLabs**（主要或回退提供者）
- **OpenAI**（主要或回退提供者；也用于摘要）
- **Edge TTS**（主要或回退提供者；使用 `node-edge-tts`，当没有 API 密钥时默认）

### Edge TTS 说明

Edge TTS 通过 `node-edge-tts` 库使用 Microsoft Edge 的在线神经 TTS 服务。它是一个托管服务（非本地），使用 Microsoft 的端点，不需要 API 密钥。`node-edge-tts` 公开语音配置选项和输出格式，但并非所有选项都由 Edge 服务支持。

因为 Edge TTS 是一个没有发布 SLA 或配额的公共 Web 服务，将其视为尽力而为。如果你需要保证的限制和支持，请使用 OpenAI 或 ElevenLabs。
Microsoft 的 Speech REST API 文档记录了每个请求 10 分钟的音频限制；Edge TTS 不发布限制，因此假设类似或更低的限制。

## 可选密钥

如果你想要 OpenAI 或 ElevenLabs：
- `ELEVENLABS_API_KEY`（或 `XI_API_KEY`）
- `OPENAI_API_KEY`

Edge TTS **不**需要 API 密钥。如果未找到 API 密钥，Clawdbot 默认为 Edge TTS（除非通过 `messages.tts.edge.enabled=false` 禁用）。

如果配置了多个提供者，首先使用选定的提供者，其他作为回退选项。
自动摘要使用配置的 `summaryModel`（或 `agents.defaults.model.primary`），因此如果你启用摘要，该提供者也必须经过认证。

## 服务链接

- [OpenAI Text-to-Speech 指南](https://platform.openai.com/docs/guides/text-to-speech)
- [OpenAI Audio API 参考](https://platform.openai.com/docs/api-reference/audio)
- [ElevenLabs Text to Speech](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [ElevenLabs 认证](https://elevenlabs.io/docs/api-reference/authentication)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [Microsoft Speech 输出格式](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)

## 默认启用吗？

不。自动 TTS 默认**关闭**。在配置中使用 `messages.tts.auto` 或在每个会话中使用 `/tts always`（别名：`/tts on`）启用它。

Edge TTS 在 TTS 开启后**默认启用**，当没有 OpenAI 或 ElevenLabs API 密钥时自动使用。

## 配置

TTS 配置位于 `clawdbot.json` 中的 `messages.tts` 下。
完整模式在 [Gateway 配置](/gateway/configuration) 中。

### 最小配置（启用 + 提供者）

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "elevenlabs"
    }
  }
}
```

### OpenAI 主要，ElevenLabs 回退

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "openai",
      summaryModel: "openai/gpt-4.1-mini",
      modelOverrides: {
        enabled: true
      },
      openai: {
        apiKey: "openai_api_key",
        model: "gpt-4o-mini-tts",
        voice: "alloy"
      },
      elevenlabs: {
        apiKey: "elevenlabs_api_key",
        baseUrl: "https://api.elevenlabs.io",
        voiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0
        }
      }
    }
  }
}
```

### Edge TTS 主要（无 API 密钥）

```json5
{
  messages: {
    tts: {
      auto: "always",
      provider: "edge",
      edge: {
        enabled: true,
        voice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        rate: "+10%",
        pitch: "-5%"
      }
    }
  }
}
```

### 禁用 Edge TTS

```json5
{
  messages: {
    tts: {
      edge: {
        enabled: false
      }
    }
  }
}
```

### 自定义限制 + 首选项路径

```json5
{
  messages: {
    tts: {
      auto: "always",
      maxTextLength: 4000,
      timeoutMs: 30000,
      prefsPath: "~/.clawdbot/settings/tts.json"
    }
  }
}
```

### 仅在入站语音笔记后回复音频

```json5
{
  messages: {
    tts: {
      auto: "inbound"
    }
  }
}
```

### 禁用长回复的自动摘要

```json5
{
  messages: {
    tts: {
      auto: "always"
    }
  }
}
```

然后运行：

```
/tts summary off
```

### 字段说明

- `auto`：自动 TTS 模式（`off`、`always`、`inbound`、`tagged`）。
  - `inbound` 仅在入站语音笔记后发送音频。
  - `tagged` 仅在回复包含 `[[tts]]` 标签时发送音频。
- `enabled`：传统切换（doctor 将此迁移到 `auto`）。
- `mode`：`"final"`（默认）或 `"all"`（包括工具/块回复）。
- `provider`：`"elevenlabs"`、`"openai"` 或 `"edge"`（回退是自动的）。
- 如果 `provider` **未设置**，Clawdbot 首选 `openai`（如果有密钥），然后 `elevenlabs`（如果有密钥），否则 `edge`。
- `summaryModel`：用于自动摘要的可选廉价模型；默认为 `agents.defaults.model.primary`。
  - 接受 `provider/model` 或配置的模型别名。
- `modelOverrides`：允许模型发出 TTS 指令（默认开启）。
- `maxTextLength`：TTS 输入的硬上限（字符）。如果超过，`/tts audio` 失败。
- `timeoutMs`：请求超时（毫秒）。
- `prefsPath`：覆盖本地首选项 JSON 路径（提供者/限制/摘要）。
- `apiKey` 值回退到环境变量（`ELEVENLABS_API_KEY`/`XI_API_KEY`、`OPENAI_API_KEY`）。
- `elevenlabs.baseUrl`：覆盖 ElevenLabs API 基础 URL。
- `elevenlabs.voiceSettings`：
  - `stability`、`similarityBoost`、`style`：`0..1`
  - `useSpeakerBoost`：`true|false`
  - `speed`：`0.5..2.0`（1.0 = 正常）
- `elevenlabs.applyTextNormalization`：`auto|on|off`
- `elevenlabs.languageCode`：2 字母 ISO 639-1（例如 `en`、`de`）
- `elevenlabs.seed`：整数 `0..4294967295`（尽力而为的确定性）
- `edge.enabled`：允许 Edge TTS 使用（默认 `true`；无 API 密钥）。
- `edge.voice`：Edge 神经语音名称（例如 `en-US-MichelleNeural`）。
- `edge.lang`：语言代码（例如 `en-US`）。
- `edge.outputFormat`：Edge 输出格式（例如 `audio-24khz-48kbitrate-mono-mp3`）。
  - 有关有效值，请参见 Microsoft Speech 输出格式；并非所有格式都由 Edge 支持。
- `edge.rate` / `edge.pitch` / `edge.volume`：百分比字符串（例如 `+10%`、`-5%`）。
- `edge.saveSubtitles`：在音频文件旁边写入 JSON 字幕。
- `edge.proxy`：Edge TTS 请求的代理 URL。
- `edge.timeoutMs`：请求超时覆盖（毫秒）。

## 模型驱动的覆盖（默认开启）

默认情况下，模型**可以**为单个回复发出 TTS 指令。
当 `messages.tts.auto` 为 `tagged` 时，需要这些指令来触发音频。

启用时，模型可以发出 `[[tts:...]]` 指令来覆盖单个回复的语音，加上可选的 `[[tts:text]]...[[/tts:text]]` 块来提供仅应出现在音频中的表达标签（笑声、歌唱提示等）。

示例回复负载：

```
Here you go.

[[tts:provider=elevenlabs voiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](laughs) Read the song once more.[[/tts:text]]
```

可用指令键（启用时）：
- `provider`（`openai` | `elevenlabs` | `edge`）
- `voice`（OpenAI 语音）或 `voiceId`（ElevenLabs）
- `model`（OpenAI TTS 模型或 ElevenLabs 模型 ID）
- `stability`、`similarityBoost`、`style`、`speed`、`useSpeakerBoost`
- `applyTextNormalization`（`auto|on|off`）
- `languageCode`（ISO 639-1）
- `seed`

禁用所有模型覆盖：

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: false
      }
    }
  }
}
```

可选允许列表（禁用特定覆盖，同时保持标签启用）：

```json5
{
  messages: {
    tts: {
      modelOverrides: {
        enabled: true,
        allowProvider: false,
        allowSeed: false
      }
    }
  }
}
```

## 每个用户的首选项

斜杠命令将本地覆盖写入 `prefsPath`（默认：`~/.clawdbot/settings/tts.json`，使用 `CLAWDBOT_TTS_PREFS` 或 `messages.tts.prefsPath` 覆盖）。

存储的字段：
- `enabled`
- `provider`
- `maxLength`（摘要阈值；默认 1500 字符）
- `summarize`（默认 `true`）

这些覆盖该主机的 `messages.tts.*`。

## 输出格式（固定）

- **Telegram**：Opus 语音笔记（来自 ElevenLabs 的 `opus_48000_64`，来自 OpenAI 的 `opus`）。
  - 48kHz / 64kbps 是语音笔记的良好权衡，并且是圆形气泡所必需的。
- **其他通道**：MP3（来自 ElevenLabs 的 `mp3_44100_128`，来自 OpenAI 的 `mp3`）。
  - 44.1kHz / 128kbps 是语音清晰度的默认平衡。
- **Edge TTS**：使用 `edge.outputFormat`（默认 `audio-24khz-48kbitrate-mono-mp3`）。
  - `node-edge-tts` 接受 `outputFormat`，但并非所有格式都从 Edge 服务可用。
  - 输出格式值遵循 Microsoft Speech 输出格式（包括 Ogg/WebM Opus）。
  - Telegram `sendVoice` 接受 OGG/MP3/M4A；如果你需要保证的 Opus 语音笔记，请使用 OpenAI/ElevenLabs。
  - 如果配置的 Edge 输出格式失败，Clawdbot 使用 MP3 重试。

OpenAI/ElevenLabs 格式是固定的；Telegram 期望 Opus 用于语音笔记 UX。

## 自动 TTS 行为

启用时，Clawdbot：
- 如果回复已包含媒体或 `MEDIA:` 指令，则跳过 TTS。
- 跳过非常短的回复（< 10 字符）。
- 启用时使用 `agents.defaults.model.primary`（或 `summaryModel`）总结长回复。
- 将生成的音频附加到回复。

如果回复超过 `maxLength` 且摘要关闭（或摘要模型没有 API 密钥），音频被跳过并发送正常文本回复。

## 流程图

```
Reply -> TTS enabled?
  no  -> send text
  yes -> has media / MEDIA: / short?
          yes -> send text
          no  -> length > limit?
                   no  -> TTS -> attach audio
                   yes -> summary enabled?
                            no  -> send text
                            yes -> summarize (summaryModel or agents.defaults.model.primary)
                                      -> TTS -> attach audio
```

## 斜杠命令使用

有一个命令：`/tts`。
有关启用详细信息，请参见 [斜杠命令](/tools/slash-commands)。

Discord 注意：`/tts` 是内置的 Discord 命令，因此 Clawdbot 在那里注册 `/voice` 作为本机命令。文本 `/tts ...` 仍然有效。

```
/tts off
/tts always
/tts inbound
/tts tagged
/tts status
/tts provider openai
/tts limit 2000
/tts summary off
/tts audio Hello from Clawdbot
```

注意：
- 命令需要授权的发送者（允许列表/所有者规则仍然适用）。
- 必须启用 `commands.text` 或本机命令注册。
- `off|always|inbound|tagged` 是每个会话的切换（`/tts on` 是 `/tts always` 的别名）。
- `limit` 和 `summary` 存储在本地首选项中，不在主配置中。
- `/tts audio` 生成一次性音频回复（不切换 TTS 开启）。

## 代理工具

`tts` 工具将文本转换为语音并返回 `MEDIA:` 路径。当结果是 Telegram 兼容时，工具包括 `[[audio_as_voice]]`，因此 Telegram 发送语音气泡。

## Gateway RPC

Gateway 方法：
- `tts.status`
- `tts.enable`
- `tts.disable`
- `tts.convert`
- `tts.setProvider`
- `tts.providers`
