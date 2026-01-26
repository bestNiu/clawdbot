---
summary: "Exec approvals, allowlists, and sandbox escape prompts"
read_when:
  - Configuring exec approvals or allowlists
  - Implementing exec approval UX in the macOS app
  - Reviewing sandbox escape prompts and implications
---

# Exec 批准

Exec 批准是**伴侣应用程序 / 节点主机护栏**，用于让沙箱化代理在真实主机（`gateway` 或 `node`）上运行命令。将其视为安全互锁：命令仅在策略 + 允许列表 +（可选）用户批准都同意时才允许。Exec 批准是**除了**工具策略和提升门控之外的（除非提升设置为 `full`，这会跳过批准）。有效策略是 `tools.exec.*` 和批准默认值的**更严格**；如果省略批准字段，使用 `tools.exec` 值。

如果伴侣应用程序 UI **不可用**，任何需要提示的请求都由**ask 回退**解析（默认：拒绝）。

## 它适用的位置

Exec 批准在执行主机上本地强制执行：
- **网关主机** → 网关机器上的 `clawdbot` 进程
- **节点主机** → 节点运行器（macOS 伴侣应用程序或无头节点主机）

macOS 拆分：
- **节点主机服务**通过本地 IPC 将 `system.run` 转发到 **macOS 应用程序**。
- **macOS 应用程序**强制执行批准 + 在 UI 上下文中执行命令。

## 设置和存储

批准位于执行主机上的本地 JSON 文件中：

`~/.clawdbot/exec-approvals.json`

示例模式：
```json
{
  "version": 1,
  "socket": {
    "path": "~/.clawdbot/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

## 策略旋钮

### 安全（`exec.security`）
- **deny**：阻止所有主机 exec 请求。
- **allowlist**：仅允许允许列表的命令。
- **full**：允许所有内容（等同于提升）。

### Ask（`exec.ask`）
- **off**：永远不提示。
- **on-miss**：仅在允许列表不匹配时提示。
- **always**：在每个命令上提示。

### Ask 回退（`askFallback`）
如果需要提示但无法访问 UI，回退决定：
- **deny**：阻止。
- **allowlist**：仅当允许列表匹配时才允许。
- **full**：允许。

## 允许列表（每个代理）

允许列表是**每个代理**。如果存在多个代理，在 macOS 应用程序中切换你正在编辑的代理。模式是**不区分大小写的 glob 匹配**。
模式应解析为**二进制路径**（仅基本名称的条目被忽略）。
传统 `agents.default` 条目在加载时迁移到 `agents.main`。

示例：
- `~/Projects/**/bin/bird`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

每个允许列表条目跟踪：
- **id** 用于 UI 身份的稳定 UUID（可选）
- **最后使用**时间戳
- **最后使用的命令**
- **最后解析的路径**

## 自动允许技能 CLI

当启用**自动允许技能 CLI**时，已知技能引用的可执行文件被视为在节点上允许列表（macOS 节点或无头节点主机）。这使用 Gateway RPC 上的 `skills.bins` 来获取技能二进制列表。如果你想要严格的手动允许列表，请禁用此功能。

## 安全二进制文件（仅 stdin）

`tools.exec.safeBins` 定义**仅 stdin**二进制文件的小列表（例如 `jq`），可以在允许列表模式下运行**而无需**显式允许列表条目。安全二进制文件拒绝位置文件参数和类似路径的 token，因此它们只能对传入流进行操作。
Shell 链接和重定向在允许列表模式下不会自动允许。

当每个顶级段满足允许列表（包括安全二进制文件或技能自动允许）时，允许 Shell 链接（`&&`、`||`、`;`）。重定向在允许列表模式下仍然不受支持。

默认安全二进制文件：`jq`、`grep`、`cut`、`sort`、`uniq`、`head`、`tail`、`tr`、`wc`。

## 控制 UI 编辑

使用 **控制 UI → 节点 → Exec 批准**卡片编辑默认值、每个代理覆盖和允许列表。选择一个作用域（默认值或代理），调整策略，添加/删除允许列表模式，然后**保存**。UI 显示每个模式的**最后使用**元数据，以便你可以保持列表整洁。

目标选择器选择**Gateway**（本地批准）或**节点**。节点必须宣传 `system.execApprovals.get/set`（macOS 应用程序或无头节点主机）。
如果节点尚未宣传 exec 批准，直接编辑其本地 `~/.clawdbot/exec-approvals.json`。

CLI：`clawdbot approvals` 支持网关或节点编辑（参见 [批准 CLI](/cli/approvals））。

## 批准流程

当需要提示时，网关向操作员客户端广播 `exec.approval.requested`。
控制 UI 和 macOS 应用程序通过 `exec.approval.resolve` 解析它，然后网关将批准的请求转发到节点主机。

当需要批准时，exec 工具立即返回批准 id。使用该 id 来关联后续系统事件（`Exec finished` / `Exec denied`）。如果超时前没有决定，请求被视为批准超时并作为拒绝原因显示。

确认对话框包括：
- 命令 + 参数
- cwd
- 代理 id
- 解析的可执行路径
- 主机 + 策略元数据

操作：
- **允许一次** → 立即运行
- **始终允许** → 添加到允许列表 + 运行
- **拒绝** → 阻止

## 批准转发到聊天通道

你可以将 exec 批准提示转发到任何聊天通道（包括插件通道）并使用 `/approve` 批准它们。这使用正常的出站传递管道。

配置：
```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session", // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["discord"], // 子字符串或正则表达式
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" }
      ]
    }
  }
}
```

在聊天中回复：
```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

### macOS IPC 流程
```
Gateway -> Node Service (WS)
                 |  IPC (UDS + token + HMAC + TTL)
                 v
             Mac App (UI + approvals + system.run)
```

安全说明：
- Unix socket 模式 `0600`，令牌存储在 `exec-approvals.json` 中。
- 同 UID 对等检查。
- 挑战/响应（nonce + HMAC 令牌 + 请求哈希）+ 短 TTL。

## 系统事件

Exec 生命周期显示为系统消息：
- `Exec running`（仅当命令超过运行通知阈值时）
- `Exec finished`
- `Exec denied`

这些在节点报告事件后发布到代理的会话。
网关主机 exec 批准在命令完成时发出相同的生命周期事件（可选地当运行时间超过阈值时）。
批准门控 exec 在这些消息中重用批准 id 作为 `runId`，以便于关联。

## 影响

- **full** 很强大；在可能时首选允许列表。
- **ask** 让你保持循环，同时仍允许快速批准。
- 每个代理允许列表防止一个代理的批准泄漏到其他代理。

相关：
- [Exec 工具](/tools/exec）
- [提升模式](/tools/elevated）
- [技能](/tools/skills）
