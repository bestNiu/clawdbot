---
summary: "Logging overview: file logs, console output, CLI tailing, and the Control UI"
read_when:
  - You need a beginner-friendly overview of logging
  - You want to configure log levels or formats
  - You are troubleshooting and need to find logs quickly
---

# 日志记录

Clawdbot 在两个地方记录日志：

- **文件日志**（JSON 行）由 Gateway 写入。
- **控制台输出**显示在终端和控制 UI 中。

本页说明日志的位置、如何读取它们，以及如何配置日志级别和格式。

## 日志位置

默认情况下，Gateway 在以下位置写入滚动日志文件：

`/tmp/clawdbot/clawdbot-YYYY-MM-DD.log`

日期使用网关主机的本地时区。

你可以在 `~/.clawdbot/clawdbot.json` 中覆盖此设置：

```json
{
  "logging": {
    "file": "/path/to/clawdbot.log"
  }
}
```

## 如何读取日志

### CLI：实时跟踪（推荐）

使用 CLI 通过 RPC 跟踪网关日志文件：

```bash
clawdbot logs --follow
```

输出模式：

- **TTY 会话**：美观、彩色、结构化日志行。
- **非 TTY 会话**：纯文本。
- `--json`：行分隔的 JSON（每行一个日志事件）。
- `--plain`：在 TTY 会话中强制纯文本。
- `--no-color`：禁用 ANSI 颜色。

在 JSON 模式下，CLI 发出带 `type` 标签的对象：

- `meta`：流元数据（文件、游标、大小）
- `log`：解析的日志条目
- `notice`：截断 / 轮换提示
- `raw`：未解析的日志行

如果 Gateway 无法访问，CLI 会打印简短提示以运行：

```bash
clawdbot doctor
```

### 控制 UI（Web）

控制 UI 的**日志**选项卡使用 `logs.tail` 跟踪同一文件。
有关如何打开它，请参见 [/web/control-ui](/web/control-ui)。

### 仅通道日志

要过滤通道活动（WhatsApp/Telegram 等），请使用：

```bash
clawdbot channels logs --channel whatsapp
```

## 日志格式

### 文件日志（JSONL）

日志文件中的每一行都是一个 JSON 对象。CLI 和控制 UI 解析这些条目以呈现结构化输出（时间、级别、子系统、消息）。

### 控制台输出

控制台日志是**TTY 感知的**，并格式化为可读性：

- 子系统前缀（例如 `gateway/channels/whatsapp`）
- 级别着色（info/warn/error）
- 可选的紧凑或 JSON 模式

控制台格式由 `logging.consoleStyle` 控制。

## 配置日志记录

所有日志记录配置位于 `~/.clawdbot/clawdbot.json` 中的 `logging` 下。

```json
{
  "logging": {
    "level": "info",
    "file": "/tmp/clawdbot/clawdbot-YYYY-MM-DD.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": [
      "sk-.*"
    ]
  }
}
```

### 日志级别

- `logging.level`：**文件日志**（JSONL）级别。
- `logging.consoleLevel`：**控制台**详细级别。

`--verbose` 仅影响控制台输出；它不会更改文件日志级别。

### 控制台样式

`logging.consoleStyle`：

- `pretty`：人性化、彩色、带时间戳。
- `compact`：更紧凑的输出（最适合长时间会话）。
- `json`：每行 JSON（用于日志处理器）。

### 编辑

工具摘要可以在到达控制台之前编辑敏感令牌：

- `logging.redactSensitive`：`off` | `tools`（默认：`tools`）
- `logging.redactPatterns`：正则表达式字符串列表，用于覆盖默认集

编辑仅影响**控制台输出**，不会更改文件日志。

## 诊断 + OpenTelemetry

诊断是结构化的、机器可读的事件，用于模型运行**和**消息流遥测（webhooks、队列、会话状态）。它们**不**替换日志；它们存在是为了提供指标、跟踪和其他导出器。

诊断事件在进程中发出，但导出器仅在启用诊断 + 导出器插件时附加。

### OpenTelemetry vs OTLP

- **OpenTelemetry (OTel)**：用于跟踪、指标和日志的数据模型 + SDK。
- **OTLP**：用于将 OTel 数据导出到收集器/后端的传输协议。
- Clawdbot 今天通过 **OTLP/HTTP (protobuf)** 导出。

### 导出的信号

- **指标**：计数器 + 直方图（令牌使用、消息流、队列）。
- **跟踪**：模型使用 + webhook/消息处理的跨度。
- **日志**：当启用 `diagnostics.otel.logs` 时通过 OTLP 导出。日志量可能很高；请记住 `logging.level` 和导出器过滤器。

### 诊断事件目录

模型使用：
- `model.usage`：令牌、成本、持续时间、上下文、提供者/模型/通道、会话 ID。

消息流：
- `webhook.received`：每个通道的 webhook 入口。
- `webhook.processed`：webhook 处理 + 持续时间。
- `webhook.error`：webhook 处理程序错误。
- `message.queued`：为处理排队的消息。
- `message.processed`：结果 + 持续时间 + 可选错误。

队列 + 会话：
- `queue.lane.enqueue`：命令队列通道入队 + 深度。
- `queue.lane.dequeue`：命令队列通道出队 + 等待时间。
- `session.state`：会话状态转换 + 原因。
- `session.stuck`：会话卡住警告 + 年龄。
- `run.attempt`：运行重试/尝试元数据。
- `diagnostic.heartbeat`：聚合计数器（webhooks/队列/会话）。

### 启用诊断（无导出器）

如果你希望诊断事件可用于插件或自定义接收器，请使用此选项：

```json
{
  "diagnostics": {
    "enabled": true
  }
}
```

### 诊断标志（定向日志）

使用标志在不提高 `logging.level` 的情况下打开额外的、定向的调试日志。
标志不区分大小写，支持通配符（例如 `telegram.*` 或 `*`）。

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

环境变量覆盖（一次性）：

```
CLAWDBOT_DIAGNOSTICS=telegram.http,telegram.payload
```

注意：
- 标志日志转到标准日志文件（与 `logging.file` 相同）。
- 输出仍根据 `logging.redactSensitive` 进行编辑。
- 完整指南：[/diagnostics/flags](/diagnostics/flags)。

### 导出到 OpenTelemetry

诊断可以通过 `diagnostics-otel` 插件（OTLP/HTTP）导出。这适用于接受 OTLP/HTTP 的任何 OpenTelemetry 收集器/后端。

```json
{
  "plugins": {
    "allow": ["diagnostics-otel"],
    "entries": {
      "diagnostics-otel": {
        "enabled": true
      }
    }
  },
  "diagnostics": {
    "enabled": true,
    "otel": {
      "enabled": true,
      "endpoint": "http://otel-collector:4318",
      "protocol": "http/protobuf",
      "serviceName": "clawdbot-gateway",
      "traces": true,
      "metrics": true,
      "logs": true,
      "sampleRate": 0.2,
      "flushIntervalMs": 60000
    }
  }
}
```

注意：
- 你也可以使用 `clawdbot plugins enable diagnostics-otel` 启用插件。
- `protocol` 目前仅支持 `http/protobuf`。`grpc` 被忽略。
- 指标包括令牌使用、成本、上下文大小、运行持续时间，以及消息流计数器/直方图（webhooks、队列、会话状态、队列深度/等待）。
- 可以使用 `traces` / `metrics` 切换跟踪/指标（默认：开启）。跟踪包括模型使用跨度，以及启用时的 webhook/消息处理跨度。
- 当收集器需要认证时设置 `headers`。
- 支持环境变量：`OTEL_EXPORTER_OTLP_ENDPOINT`、`OTEL_SERVICE_NAME`、`OTEL_EXPORTER_OTLP_PROTOCOL`。

### 导出的指标（名称 + 类型）

模型使用：
- `clawdbot.tokens`（计数器，属性：`clawdbot.token`、`clawdbot.channel`、`clawdbot.provider`、`clawdbot.model`）
- `clawdbot.cost.usd`（计数器，属性：`clawdbot.channel`、`clawdbot.provider`、`clawdbot.model`）
- `clawdbot.run.duration_ms`（直方图，属性：`clawdbot.channel`、`clawdbot.provider`、`clawdbot.model`）
- `clawdbot.context.tokens`（直方图，属性：`clawdbot.context`、`clawdbot.channel`、`clawdbot.provider`、`clawdbot.model`）

消息流：
- `clawdbot.webhook.received`（计数器，属性：`clawdbot.channel`、`clawdbot.webhook`）
- `clawdbot.webhook.error`（计数器，属性：`clawdbot.channel`、`clawdbot.webhook`）
- `clawdbot.webhook.duration_ms`（直方图，属性：`clawdbot.channel`、`clawdbot.webhook`）
- `clawdbot.message.queued`（计数器，属性：`clawdbot.channel`、`clawdbot.source`）
- `clawdbot.message.processed`（计数器，属性：`clawdbot.channel`、`clawdbot.outcome`）
- `clawdbot.message.duration_ms`（直方图，属性：`clawdbot.channel`、`clawdbot.outcome`）

队列 + 会话：
- `clawdbot.queue.lane.enqueue`（计数器，属性：`clawdbot.lane`）
- `clawdbot.queue.lane.dequeue`（计数器，属性：`clawdbot.lane`）
- `clawdbot.queue.depth`（直方图，属性：`clawdbot.lane` 或 `clawdbot.channel=heartbeat`）
- `clawdbot.queue.wait_ms`（直方图，属性：`clawdbot.lane`）
- `clawdbot.session.state`（计数器，属性：`clawdbot.state`、`clawdbot.reason`）
- `clawdbot.session.stuck`（计数器，属性：`clawdbot.state`）
- `clawdbot.session.stuck_age_ms`（直方图，属性：`clawdbot.state`）
- `clawdbot.run.attempt`（计数器，属性：`clawdbot.attempt`）

### 导出的跨度（名称 + 关键属性）

- `clawdbot.model.usage`
  - `clawdbot.channel`、`clawdbot.provider`、`clawdbot.model`
  - `clawdbot.sessionKey`、`clawdbot.sessionId`
  - `clawdbot.tokens.*`（input/output/cache_read/cache_write/total）
- `clawdbot.webhook.processed`
  - `clawdbot.channel`、`clawdbot.webhook`、`clawdbot.chatId`
- `clawdbot.webhook.error`
  - `clawdbot.channel`、`clawdbot.webhook`、`clawdbot.chatId`、`clawdbot.error`
- `clawdbot.message.processed`
  - `clawdbot.channel`、`clawdbot.outcome`、`clawdbot.chatId`、`clawdbot.messageId`、`clawdbot.sessionKey`、`clawdbot.sessionId`、`clawdbot.reason`
- `clawdbot.session.stuck`
  - `clawdbot.state`、`clawdbot.ageMs`、`clawdbot.queueDepth`、`clawdbot.sessionKey`、`clawdbot.sessionId`

### 采样 + 刷新

- 跟踪采样：`diagnostics.otel.sampleRate`（0.0–1.0，仅根跨度）。
- 指标导出间隔：`diagnostics.otel.flushIntervalMs`（最小 1000ms）。

### 协议说明

- OTLP/HTTP 端点可以通过 `diagnostics.otel.endpoint` 或 `OTEL_EXPORTER_OTLP_ENDPOINT` 设置。
- 如果端点已包含 `/v1/traces` 或 `/v1/metrics`，则按原样使用。
- 如果端点已包含 `/v1/logs`，则按原样用于日志。
- `diagnostics.otel.logs` 为主记录器输出启用 OTLP 日志导出。

### 日志导出行为

- OTLP 日志使用写入 `logging.file` 的相同结构化记录。
- 遵循 `logging.level`（文件日志级别）。控制台编辑**不**适用于 OTLP 日志。
- 高容量安装应首选 OTLP 收集器采样/过滤。

## 故障排除提示

- **Gateway 无法访问？** 首先运行 `clawdbot doctor`。
- **日志为空？** 检查 Gateway 是否正在运行并写入 `logging.file` 中的文件路径。
- **需要更多详细信息？** 将 `logging.level` 设置为 `debug` 或 `trace` 并重试。
