---
title: Lobster
summary: "Typed workflow runtime for Clawdbot with resumable approval gates."
description: Typed workflow runtime for Clawdbot — composable pipelines with approval gates.
read_when:
  - You want deterministic multi-step workflows with explicit approvals
  - You need to resume a workflow without re-running earlier steps
---

# Lobster

Lobster 是一个工作流 shell，允许 Clawdbot 将多步骤工具序列作为单个确定性操作运行，具有显式批准检查点。

## Hook

你的助手可以构建管理自己的工具。请求一个工作流，30 分钟后你就有了一个 CLI 加上作为单个调用运行的管道。Lobster 是缺失的部分：确定性管道、显式批准和可恢复状态。

## 为什么

今天，复杂的工作流需要许多来回的工具调用。每次调用都消耗 tokens，LLM 必须编排每个步骤。Lobster 将该编排移动到类型化运行时：

- **一个调用而不是许多**：Clawdbot 运行一个 Lobster 工具调用并获得结构化结果。
- **内置批准**：副作用（发送电子邮件、发布评论）暂停工作流，直到明确批准。
- **可恢复**：暂停的工作流返回一个令牌；批准并恢复而不重新运行所有内容。

## 为什么是 DSL 而不是普通程序？

Lobster 故意很小。目标不是"新语言"，它是一个可预测的、AI 友好的管道规范，具有一流的批准和恢复令牌。

- **批准/恢复是内置的**：普通程序可以提示人类，但它不能*暂停并使用持久令牌恢复*，除非你自己发明该运行时。
- **确定性 + 可审计性**：管道是数据，因此它们易于记录、差异、重放和审查。
- **AI 的受限界面**：小语法 + JSON 管道减少了"创造性"代码路径，使验证变得现实。
- **内置安全策略**：超时、输出上限、沙箱检查和允许列表由运行时强制执行，而不是每个脚本。
- **仍然可编程**：每个步骤可以调用任何 CLI 或脚本。如果你想要 JS/TS，从代码生成 `.lobster` 文件。

## 工作原理

Clawdbot 在**工具模式**下启动本地 `lobster` CLI 并从 stdout 解析 JSON 信封。
如果管道暂停等待批准，工具返回 `resumeToken`，以便你稍后继续。

## 模式：小型 CLI + JSON 管道 + 批准

构建说 JSON 的小型命令，然后将它们链接到单个 Lobster 调用中。（下面的示例命令名称 — 替换为你自己的。）

```bash
inbox list --json
inbox categorize --json
inbox apply --json
```

```json
{
  "action": "run",
  "pipeline": "exec --json --shell 'inbox list --json' | exec --stdin json --shell 'inbox categorize --json' | exec --stdin json --shell 'inbox apply --json' | approve --preview-from-stdin --limit 5 --prompt 'Apply changes?'",
  "timeoutMs": 30000
}
```

如果管道请求批准，使用令牌恢复：

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

AI 触发工作流；Lobster 执行步骤。批准门控保持副作用显式和可审计。

示例：将输入项映射到工具调用：

```bash
gog.gmail.search --query 'newer_than:1d' \
  | clawd.invoke --tool message --action send --each --item-key message --args-json '{"provider":"telegram","to":"..."}'
```

## 仅 JSON LLM 步骤（llm-task）

对于需要**结构化 LLM 步骤**的工作流，启用可选的 `llm-task` 插件工具并从 Lobster 调用它。这使工作流保持确定性，同时仍允许你使用模型进行分类/摘要/草稿。

启用工具：

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  },
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": { "allow": ["llm-task"] }
      }
    ]
  }
}
```

在管道中使用它：

```lobster
clawd.invoke --tool llm-task --action json --args-json '{
  "prompt": "Given the input email, return intent and draft.",
  "input": { "subject": "Hello", "body": "Can you help?" },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

有关详细信息和配置选项，请参见 [LLM Task](/tools/llm-task）。

## 工作流文件（.lobster）

Lobster 可以运行带有 `name`、`args`、`steps`、`env`、`condition` 和 `approval` 字段的 YAML/JSON 工作流文件。在 Clawdbot 工具调用中，将 `pipeline` 设置为文件路径。

```yaml
name: inbox-triage
args:
  tag:
    default: "family"
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

注意：

- `stdin: $step.stdout` 和 `stdin: $step.json` 传递先前步骤的输出。
- `condition`（或 `when`）可以在 `$step.approved` 上门控步骤。

## 安装 Lobster

在运行 Clawdbot Gateway 的**同一主机**上安装 Lobster CLI（参见 [Lobster 仓库](https://github.com/clawdbot/lobster）），并确保 `lobster` 在 `PATH` 上。
如果你想使用自定义二进制位置，在工具调用中传递**绝对** `lobsterPath`。

## 启用工具

Lobster 是一个**可选**插件工具（默认不启用）。按代理允许它：

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "tools": {
          "allow": ["lobster"]
        }
      }
    ]
  }
}
```

如果每个代理都应该看到它，你也可以使用 `tools.allow` 全局允许它。

注意：允许列表对于可选插件是选择加入的。如果你的允许列表仅命名插件工具（如 `lobster`），Clawdbot 保持核心工具启用。要限制核心工具，也在允许列表中包含你想要的核心工具或组。

## 示例：电子邮件分类

没有 Lobster：
```
User: "Check my email and draft replies"
→ clawd calls gmail.list
→ LLM summarizes
→ User: "draft replies to #2 and #5"
→ LLM drafts
→ User: "send #2"
→ clawd calls gmail.send
(repeat daily, no memory of what was triaged)
```

使用 Lobster：
```json
{
  "action": "run",
  "pipeline": "email.triage --limit 20",
  "timeoutMs": 30000
}
```

返回 JSON 信封（截断）：
```json
{
  "ok": true,
  "status": "needs_approval",
  "output": [{ "summary": "5 need replies, 2 need action" }],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "Send 2 draft replies?",
    "items": [],
    "resumeToken": "..."
  }
}
```

用户批准 → 恢复：
```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

一个工作流。确定性。安全。

## 工具参数

### `run`

在工具模式下运行管道。

```json
{
  "action": "run",
  "pipeline": "gog.gmail.search --query 'newer_than:1d' | email.triage",
  "cwd": "/path/to/workspace",
  "timeoutMs": 30000,
  "maxStdoutBytes": 512000
}
```

使用参数运行工作流文件：

```json
{
  "action": "run",
  "pipeline": "/path/to/inbox-triage.lobster",
  "argsJson": "{\"tag\":\"family\"}"
}
```

### `resume`

批准后继续暂停的工作流。

```json
{
  "action": "resume",
  "token": "<resumeToken>",
  "approve": true
}
```

### 可选输入

- `lobsterPath`：Lobster 二进制文件的绝对路径（省略以使用 `PATH`）。
- `cwd`：管道的工作目录（默认为当前进程工作目录）。
- `timeoutMs`：如果超过此持续时间，则杀死子进程（默认：20000）。
- `maxStdoutBytes`：如果 stdout 超过此大小，则杀死子进程（默认：512000）。
- `argsJson`：传递给 `lobster run --args-json` 的 JSON 字符串（仅工作流文件）。

## 输出信封

Lobster 返回带有三种状态之一的 JSON 信封：

- `ok` → 成功完成
- `needs_approval` → 暂停；需要 `requiresApproval.resumeToken` 来恢复
- `cancelled` → 明确拒绝或取消

工具在 `content`（漂亮的 JSON）和 `details`（原始对象）中都显示信封。

## 批准

如果存在 `requiresApproval`，检查提示并决定：

- `approve: true` → 恢复并继续副作用
- `approve: false` → 取消并完成工作流

使用 `approve --preview-from-stdin --limit N` 将 JSON 预览附加到批准请求，而无需自定义 jq/heredoc 粘合。恢复令牌现在是紧凑的：Lobster 在其状态目录下存储工作流恢复状态并返回小令牌键。

## OpenProse

OpenProse 与 Lobster 配合良好：使用 `/prose` 编排多代理准备，然后运行 Lobster 管道以进行确定性批准。如果 Prose 程序需要 Lobster，通过 `tools.subagents.tools` 为子代理允许 `lobster` 工具。参见 [OpenProse](/prose）。

## 安全

- **仅本地子进程** — 插件本身没有网络调用。
- **无密钥** — Lobster 不管理 OAuth；它调用执行此操作的 clawd 工具。
- **沙箱感知** — 当工具上下文被沙箱化时禁用。
- **强化** — 如果指定，`lobsterPath` 必须是绝对的；强制执行超时和输出上限。

## 故障排除

- **`lobster subprocess timed out`** → 增加 `timeoutMs`，或拆分长管道。
- **`lobster output exceeded maxStdoutBytes`** → 提高 `maxStdoutBytes` 或减少输出大小。
- **`lobster returned invalid JSON`** → 确保管道在工具模式下运行并仅打印 JSON。
- **`lobster failed (code …)`** → 在终端中运行相同的管道以检查 stderr。

## 了解更多

- [插件](/plugin）
- [插件工具编写](/plugins/agent-tools）

## 案例研究：社区工作流

一个公共示例：一个"第二大脑" CLI + Lobster 管道，管理三个 Markdown 库（个人、合作伙伴、共享）。CLI 为统计、收件箱列表和过期扫描发出 JSON；Lobster 将这些命令链接到工作流，如 `weekly-review`、`inbox-triage`、`memory-consolidation` 和 `shared-task-sync`，每个都有批准门控。AI 在可用时处理判断（分类），并在不可用时回退到确定性规则。

- 线程：https://x.com/plattenschieber/status/2014508656335770033
- 仓库：https://github.com/bloomedai/brain-cli
