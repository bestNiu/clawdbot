---
summary: "Agent tool surface for Clawdbot (browser, canvas, nodes, message, cron) replacing legacy `clawdbot-*` skills"
read_when:
  - Adding or modifying agent tools
  - Retiring or changing `clawdbot-*` skills
---

# 工具（Clawdbot）

Clawdbot 为浏览器、canvas、节点和 cron 公开**一流的代理工具**。
这些替换了旧的 `clawdbot-*` 技能：工具是类型化的，无 shell 调用，代理应该直接依赖它们。

## 禁用工具

你可以通过 `clawdbot.json` 中的 `tools.allow` / `tools.deny` 全局允许/拒绝工具（拒绝获胜）。这防止不允许的工具被发送到模型提供者。

```json5
{
  tools: { deny: ["browser"] }
}
```

注意：
- 匹配不区分大小写。
- 支持 `*` 通配符（`"*"` 表示所有工具）。
- 如果 `tools.allow` 仅引用未知或未加载的插件工具名称，Clawdbot 记录警告并忽略允许列表，以便核心工具保持可用。

## 工具配置文件（基础允许列表）

`tools.profile` 在 `tools.allow`/`tools.deny` 之前设置**基础工具允许列表**。
每个代理覆盖：`agents.list[].tools.profile`。

配置文件：
- `minimal`：仅 `session_status`
- `coding`：`group:fs`、`group:runtime`、`group:sessions`、`group:memory`、`image`
- `messaging`：`group:messaging`、`sessions_list`、`sessions_history`、`sessions_send`、`session_status`
- `full`：无限制（与未设置相同）

示例（默认仅消息，也允许 Slack + Discord 工具）：
```json5
{
  tools: {
    profile: "messaging",
    allow: ["slack", "discord"]
  }
}
```

示例（编码配置文件，但到处拒绝 exec/process）：
```json5
{
  tools: {
    profile: "coding",
    deny: ["group:runtime"]
  }
}
```

示例（全局编码配置文件，仅消息支持代理）：
```json5
{
  tools: { profile: "coding" },
  agents: {
    list: [
      {
        id: "support",
        tools: { profile: "messaging", allow: ["slack"] }
      }
    ]
  }
}
```

## 提供者特定工具策略

使用 `tools.byProvider` 为特定提供者（或单个 `provider/model`）**进一步限制**工具，而无需更改全局默认值。
每个代理覆盖：`agents.list[].tools.byProvider`。

这在基础工具配置文件**之后**和允许/拒绝列表**之前**应用，因此它只能缩小工具集。
提供者键接受 `provider`（例如 `google-antigravity`）或 `provider/model`（例如 `openai/gpt-5.2`）。

示例（保持全局编码配置文件，但 Google Antigravity 的最小工具）：
```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" }
    }
  }
}
```

示例（不稳定端点的提供者/模型特定允许列表）：
```json5
{
  tools: {
    allow: ["group:fs", "group:runtime", "sessions_list"],
    byProvider: {
      "openai/gpt-5.2": { allow: ["group:fs", "sessions_list"] }
    }
  }
}
```

示例（单个提供者的代理特定覆盖）：
```json5
{
  agents: {
    list: [
      {
        id: "support",
        tools: {
          byProvider: {
            "google-antigravity": { allow: ["message", "sessions_list"] }
          }
        }
      }
    ]
  }
}
```

## 工具组（简写）

工具策略（全局、代理、沙箱）支持扩展到多个工具的 `group:*` 条目。
在 `tools.allow` / `tools.deny` 中使用这些。

可用组：
- `group:runtime`：`exec`、`bash`、`process`
- `group:fs`：`read`、`write`、`edit`、`apply_patch`
- `group:sessions`：`sessions_list`、`sessions_history`、`sessions_send`、`sessions_spawn`、`session_status`
- `group:memory`：`memory_search`、`memory_get`
- `group:web`：`web_search`、`web_fetch`
- `group:ui`：`browser`、`canvas`
- `group:automation`：`cron`、`gateway`
- `group:messaging`：`message`
- `group:nodes`：`nodes`
- `group:clawdbot`：所有内置 Clawdbot 工具（不包括提供者插件）

示例（仅允许文件工具 + 浏览器）：
```json5
{
  tools: {
    allow: ["group:fs", "browser"]
  }
}
```

## 插件 + 工具

插件可以注册**额外工具**（和 CLI 命令），超出核心集。
有关安装 + 配置，请参见 [插件](/plugin），有关工具使用指导如何注入到提示中，请参见 [技能](/tools/skills）。某些插件随工具一起提供自己的技能（例如，voice-call 插件）。

可选插件工具：
- [Lobster](/tools/lobster）：具有可恢复批准的类型化工作流运行时（需要网关主机上的 Lobster CLI）。
- [LLM Task](/tools/llm-task）：仅 JSON 的 LLM 步骤，用于结构化工作流输出（可选模式验证）。

## 工具清单

### `apply_patch`
跨一个或多个文件应用结构化补丁。用于多文件或多块编辑，其中单个 `edit` 调用会很脆弱。
实验性：通过 `tools.exec.applyPatch.enabled` 启用（仅 OpenAI 模型）。

### `exec`
在工作区中运行 shell 命令。

核心参数：
- `command`（必需）
- `yieldMs`（超时后自动后台，默认 10000）
- `background`（立即后台）
- `timeout`（秒；如果超过则杀死进程，默认 1800）
- `elevated`（布尔值；如果启用/允许提升模式，则在主机上运行；仅在代理被沙箱化时更改行为）
- `host`（`sandbox | gateway | node`）
- `security`（`deny | allowlist | full`）
- `ask`（`off | on-miss | always`）
- `node`（`host=node` 的节点 id/名称）
- 需要真正的 TTY？设置 `pty: true`。

注意：
- 后台时返回 `status: "running"` 和 `sessionId`。
- 使用 `process` 轮询/记录/写入/杀死/清除后台会话。
- 如果 `process` 被拒绝，`exec` 同步运行并忽略 `yieldMs`/`background`。
- `elevated` 由 `tools.elevated` 加上任何 `agents.list[].tools.elevated` 覆盖门控（两者都必须允许），并且是 `host=gateway` + `security=full` 的别名。
- `elevated` 仅在代理被沙箱化时更改行为（否则是无操作）。
- `host=node` 可以定位 macOS 伴侣应用程序或无头节点主机（`clawdbot node run`）。
- 网关/节点批准和允许列表：[Exec 批准](/tools/exec-approvals）。

### `process`
管理后台 exec 会话。

核心操作：
- `list`、`poll`、`log`、`write`、`kill`、`clear`、`remove`

注意：
- `poll` 在完成时返回新输出和退出状态。
- `log` 支持基于行的 `offset`/`limit`（省略 `offset` 以获取最后 N 行）。
- `process` 按代理作用域；来自其他代理的会话不可见。

### `web_search`
使用 Brave Search API 搜索网络。

核心参数：
- `query`（必需）
- `count`（1–10；默认来自 `tools.web.search.maxResults`）

注意：
- 需要 Brave API 密钥（推荐：`clawdbot configure --section web`，或设置 `BRAVE_API_KEY`）。
- 通过 `tools.web.search.enabled` 启用。
- 响应被缓存（默认 15 分钟）。
- 有关设置，请参见 [Web 工具](/tools/web）。

### `web_fetch`
从 URL 获取并提取可读内容（HTML → markdown/text）。

核心参数：
- `url`（必需）
- `extractMode`（`markdown` | `text`）
- `maxChars`（截断长页面）

注意：
- 通过 `tools.web.fetch.enabled` 启用。
- 响应被缓存（默认 15 分钟）。
- 对于 JS 密集型站点，首选浏览器工具。
- 有关设置，请参见 [Web 工具](/tools/web）。
- 有关可选的反机器人回退，请参见 [Firecrawl](/tools/firecrawl）。

### `browser`
控制专用的 clawd 浏览器。

核心操作：
- `status`、`start`、`stop`、`tabs`、`open`、`focus`、`close`
- `snapshot`（aria/ai）
- `screenshot`（返回图像块 + `MEDIA:<path>`）
- `act`（UI 操作：click/type/press/hover/drag/select/fill/resize/wait/evaluate）
- `navigate`、`console`、`pdf`、`upload`、`dialog`

配置文件管理：
- `profiles` — 列出所有带状态的浏览器配置文件
- `create-profile` — 创建具有自动分配端口（或 `cdpUrl`）的新配置文件
- `delete-profile` — 停止浏览器，删除用户数据，从配置中删除（仅本地）
- `reset-profile` — 在配置文件的端口上杀死孤立进程（仅本地）

常见参数：
- `controlUrl`（从配置默认）
- `profile`（可选；默认为 `browser.defaultProfile`）
注意：
- 需要 `browser.enabled=true`（默认为 `true`；设置 `false` 以禁用）。
- 除非显式传递 `controlUrl`，否则使用 `browser.controlUrl`。
- 所有操作都接受可选的 `profile` 参数以支持多实例。
- 当省略 `profile` 时，使用 `browser.defaultProfile`（默认为 "chrome"）。
- 配置文件名称：仅小写字母数字 + 连字符（最多 64 个字符）。
- 端口范围：18800-18899（最多 ~100 个配置文件）。
- 远程配置文件是仅附加（无启动/停止/重置）。
- 当安装了 Playwright 时，`snapshot` 默认为 `ai`；使用 `aria` 获取可访问性树。
- `snapshot` 还支持角色快照选项（`interactive`、`compact`、`depth`、`selector`），它们返回像 `e12` 这样的引用。
- `act` 需要来自 `snapshot` 的 `ref`（来自 AI 快照的数字 `12`，或来自角色快照的 `e12`）；对于罕见的 CSS 选择器需求，使用 `evaluate`。
- 默认避免 `act` → `wait`；仅在特殊情况下使用它（没有可靠的 UI 状态可等待）。
- `upload` 可以选择传递 `ref` 以在准备后自动点击。
- `upload` 还支持 `inputRef`（aria ref）或 `element`（CSS 选择器）直接设置 `<input type="file">`。

### `canvas`
驱动节点 Canvas（present、eval、snapshot、A2UI）。

核心操作：
- `present`、`hide`、`navigate`、`eval`
- `snapshot`（返回图像块 + `MEDIA:<path>`）
- `a2ui_push`、`a2ui_reset`

注意：
- 在底层使用网关 `node.invoke`。
- 如果未提供 `node`，工具选择默认值（单个连接的节点或本地 mac 节点）。
- A2UI 仅为 v0.8（无 `createSurface`）；CLI 拒绝带有行错误的 v0.9 JSONL。
- 快速测试：`clawdbot nodes canvas a2ui push --node <id> --text "Hello from A2UI"`。

### `nodes`
发现并定位配对节点；发送通知；捕获相机/屏幕。

核心操作：
- `status`、`describe`
- `pending`、`approve`、`reject`（配对）
- `notify`（macOS `system.notify`）
- `run`（macOS `system.run`）
- `camera_snap`、`camera_clip`、`screen_record`
- `location_get`

注意：
- 相机/屏幕命令需要节点应用程序处于前台。
- 图像返回图像块 + `MEDIA:<path>`。
- 视频返回 `FILE:<path>`（mp4）。
- 位置返回 JSON 负载（lat/lon/accuracy/timestamp）。
- `run` 参数：`command` argv 数组；可选的 `cwd`、`env`（`KEY=VAL`）、`commandTimeoutMs`、`invokeTimeoutMs`、`needsScreenRecording`。

示例（`run`）：
```json
{
  "action": "run",
  "node": "office-mac",
  "command": ["echo", "Hello"],
  "env": ["FOO=bar"],
  "commandTimeoutMs": 12000,
  "invokeTimeoutMs": 45000,
  "needsScreenRecording": false
}
```

### `image`
使用配置的图像模型分析图像。

核心参数：
- `image`（必需的路径或 URL）
- `prompt`（可选；默认为 "Describe the image."）
- `model`（可选覆盖）
- `maxBytesMb`（可选大小上限）

注意：
- 仅当配置了 `agents.defaults.imageModel`（主要或回退）时可用，或当可以从默认模型 + 配置的认证推断隐式图像模型时（尽力而为配对）。
- 直接使用图像模型（独立于主聊天模型）。

### `message`
跨 Discord/Google Chat/Slack/Telegram/WhatsApp/Signal/iMessage/MS Teams 发送消息和通道操作。

核心操作：
- `send`（文本 + 可选媒体；MS Teams 还支持用于 Adaptive Cards 的 `card`）
- `poll`（WhatsApp/Discord/MS Teams 投票）
- `react` / `reactions` / `read` / `edit` / `delete`
- `pin` / `unpin` / `list-pins`
- `permissions`
- `thread-create` / `thread-list` / `thread-reply`
- `search`
- `sticker`
- `member-info` / `role-info`
- `emoji-list` / `emoji-upload` / `sticker-upload`
- `role-add` / `role-remove`
- `channel-info` / `channel-list`
- `voice-status`
- `event-list` / `event-create`
- `timeout` / `kick` / `ban`

注意：
- `send` 通过 Gateway 路由 WhatsApp；其他通道直接进行。
- `poll` 对 WhatsApp 和 MS Teams 使用 Gateway；Discord 投票直接进行。
- 当消息工具调用绑定到活动聊天会话时，发送被限制到该会话的目标，以避免跨上下文泄漏。

### `cron`
管理 Gateway cron 作业和唤醒。

核心操作：
- `status`、`list`
- `add`、`update`、`remove`、`run`、`runs`
- `wake`（排队系统事件 + 可选立即心跳）

注意：
- `add` 期望完整的 cron 作业对象（与 `cron.add` RPC 相同的模式）。
- `update` 使用 `{ id, patch }`。

### `gateway`
重启或对正在运行的 Gateway 进程应用更新（就地）。

核心操作：
- `restart`（授权 + 发送 `SIGUSR1` 进行进程内重启；`clawdbot gateway` 就地重启）
- `config.get` / `config.schema`
- `config.apply`（验证 + 写入配置 + 重启 + 唤醒）
- `config.patch`（合并部分更新 + 重启 + 唤醒）
- `update.run`（运行更新 + 重启 + 唤醒）

注意：
- 使用 `delayMs`（默认为 2000）以避免中断正在进行的回复。
- `restart` 默认禁用；使用 `commands.restart: true` 启用。

### `sessions_list` / `sessions_history` / `sessions_send` / `sessions_spawn` / `session_status`
列出会话，检查转录历史，或发送到另一个会话。

核心参数：
- `sessions_list`：`kinds?`、`limit?`、`activeMinutes?`、`messageLimit?`（0 = 无）
- `sessions_history`：`sessionKey`（或 `sessionId`）、`limit?`、`includeTools?`
- `sessions_send`：`sessionKey`（或 `sessionId`）、`message`、`timeoutSeconds?`（0 = 即发即弃）
- `sessions_spawn`：`task`、`label?`、`agentId?`、`model?`、`runTimeoutSeconds?`、`cleanup?`
- `session_status`：`sessionKey?`（默认当前；接受 `sessionId`）、`model?`（`default` 清除覆盖）

注意：
- `main` 是规范的直接聊天键；global/unknown 被隐藏。
- `messageLimit > 0` 获取每个会话的最后 N 条消息（工具消息被过滤）。
- 当 `timeoutSeconds > 0` 时，`sessions_send` 等待最终完成。
- 传递/通知在完成后发生并且是尽力而为；`status: "ok"` 确认代理运行完成，而不是通知已传递。
- `sessions_spawn` 启动子代理运行并将通知回复发布回请求者聊天。
- `sessions_spawn` 是非阻塞的，并立即返回 `status: "accepted"`。
- `sessions_send` 运行回复返回 ping-pong（回复 `REPLY_SKIP` 停止；通过 `session.agentToAgent.maxPingPongTurns` 最大回合数，0–5）。
- ping-pong 后，目标代理运行**通知步骤**；回复 `ANNOUNCE_SKIP` 以抑制通知。

### `agents_list`
列出当前会话可以通过 `sessions_spawn` 定位的代理 id。

注意：
- 结果限制为每个代理允许列表（`agents.list[].subagents.allowAgents`）。
- 当配置 `["*"]` 时，工具包括所有配置的代理并标记 `allowAny: true`。

## 参数（常见）

网关支持的工具（`canvas`、`nodes`、`cron`）：
- `gatewayUrl`（默认 `ws://127.0.0.1:18789`）
- `gatewayToken`（如果启用认证）
- `timeoutMs`

浏览器工具：
- `controlUrl`（从配置默认）

## 推荐的代理流程

浏览器自动化：
1) `browser` → `status` / `start`
2) `snapshot`（ai 或 aria）
3) `act`（click/type/press）
4) 如果需要视觉确认，则 `screenshot`

Canvas 渲染：
1) `canvas` → `present`
2) `a2ui_push`（可选）
3) `snapshot`

节点定位：
1) `nodes` → `status`
2) 在所选节点上 `describe`
3) `notify` / `run` / `camera_snap` / `screen_record`

## 安全

- 避免直接 `system.run`；仅在明确用户同意时使用 `nodes` → `run`。
- 尊重用户对相机/屏幕捕获的同意。
- 在调用媒体命令之前使用 `status/describe` 确保权限。

## 工具如何呈现给代理

工具通过两个并行通道公开：

1) **系统提示文本**：人类可读的列表 + 指导。
2) **工具模式**：发送到模型 API 的结构化函数定义。

这意味着代理既看到"存在哪些工具"，也看到"如何调用它们"。如果工具不出现在系统提示或模式中，模型无法调用它。
