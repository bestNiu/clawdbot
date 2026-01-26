---
summary: "How Clawdbot builds prompt context and reports token usage + costs"
read_when:
  - Explaining token usage, costs, or context windows
  - Debugging context growth or compaction behavior
---
# Token 使用和成本

Clawdbot 跟踪**tokens**，而不是字符。Tokens 是模型特定的，但大多数 OpenAI 风格的模型平均每个 token 约 4 个字符用于英文文本。

## 系统提示如何构建

Clawdbot 在每次运行时组装自己的系统提示。它包括：

- 工具列表 + 简短描述
- 技能列表（仅元数据；说明在使用 `read` 时按需加载）
- 自我更新说明
- 工作区 + 引导文件（`AGENTS.md`、`SOUL.md`、`TOOLS.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md`，新时为 `BOOTSTRAP.md`）。大文件由 `agents.defaults.bootstrapMaxChars`（默认：20000）截断。
- 时间（UTC + 用户时区）
- 回复标签 + 心跳行为
- 运行时元数据（主机/OS/模型/思考）

有关完整分解，请参见 [系统提示](/concepts/system-prompt)。

## 上下文窗口中计算的内容

模型接收的所有内容都计入上下文限制：

- 系统提示（上面列出的所有部分）
- 对话历史（用户 + 助手消息）
- 工具调用和工具结果
- 附件/转录（图像、音频、文件）
- 压缩摘要和修剪工件
- 提供者包装器或安全标头（不可见，但仍被计算）

有关实际分解（每个注入文件、工具、技能和系统提示大小），请使用 `/context list` 或 `/context detail`。参见 [上下文](/concepts/context)。

## 如何查看当前 token 使用

在聊天中使用这些：

- `/status` → **丰富的表情符号状态卡**，包含会话模型、上下文使用、上次响应输入/输出 tokens，以及**估计成本**（仅 API 密钥）。
- `/usage off|tokens|full` → 在每个回复后附加**每次响应使用页脚**。
  - 每个会话持久化（存储为 `responseUsage`）。
  - OAuth 认证**隐藏成本**（仅 tokens）。
- `/usage cost` → 显示来自 Clawdbot 会话日志的本地成本摘要。

其他界面：

- **TUI/Web TUI：** 支持 `/status` + `/usage`。
- **CLI：** `clawdbot status --usage` 和 `clawdbot channels list` 显示提供者配额窗口（不是每次响应成本）。

## 成本估算（显示时）

成本从你的模型定价配置估算：

```
models.providers.<provider>.models[].cost
```

这些是**每 100 万 tokens 的 USD**，用于 `input`、`output`、`cacheRead` 和 `cacheWrite`。如果缺少定价，Clawdbot 仅显示 tokens。OAuth tokens 从不显示美元成本。

## 缓存 TTL 和修剪影响

提供者提示缓存仅适用于缓存 TTL 窗口内。Clawdbot 可以选择运行**缓存 TTL 修剪**：它在缓存 TTL 过期后修剪会话，然后重置缓存窗口，以便后续请求可以重新使用新缓存的上下文，而不是重新缓存完整历史。这会在会话在 TTL 后空闲时保持缓存写入成本较低。

在 [Gateway 配置](/gateway/configuration) 中配置它，并在 [会话修剪](/concepts/session-pruning) 中查看行为详细信息。

心跳可以在空闲间隙中保持缓存**温暖**。如果你的模型缓存 TTL 是 `1h`，将心跳间隔设置为略低于该值（例如，`55m`）可以避免重新缓存完整提示，降低缓存写入成本。

对于 Anthropic API 定价，缓存读取比输入 tokens 便宜得多，而缓存写入以更高的倍数计费。有关最新费率和 TTL 倍数，请参见 Anthropic 的提示缓存定价：
https://docs.anthropic.com/docs/build-with-claude/prompt-caching

### 示例：使用心跳保持 1h 缓存温暖

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-5"
    models:
      "anthropic/claude-opus-4-5":
        params:
          cacheControlTtl: "1h"
    heartbeat:
      every: "55m"
```

## 减少 token 压力的提示

- 使用 `/compact` 总结长会话。
- 在工作流中修剪大型工具输出。
- 保持技能描述简短（技能列表注入到提示中）。
- 对于冗长的、探索性的工作，首选较小的模型。

有关确切的技能列表开销公式，请参见 [技能](/tools/skills)。
