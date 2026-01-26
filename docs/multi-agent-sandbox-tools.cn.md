---
summary: "Per-agent sandbox + tool restrictions, precedence, and examples"
title: Multi-Agent Sandbox & Tools
read_when: "You want per-agent sandboxing or per-agent tool allow/deny policies in a multi-agent gateway."
status: active
---

# 多代理沙箱和工具配置

## 概述

多代理设置中的每个代理现在可以有自己的：
- **沙箱配置**（`agents.list[].sandbox` 覆盖 `agents.defaults.sandbox`）
- **工具限制**（`tools.allow` / `tools.deny`，加上 `agents.list[].tools`）

这允许你运行具有不同安全配置文件的多个代理：
- 具有完全访问权限的个人助手
- 具有受限工具的家庭/工作代理
- 在沙箱中的面向公众的代理

`setupCommand` 属于 `sandbox.docker`（全局或每个代理）下，在创建容器时运行一次。

认证是按代理的：每个代理从其自己的 `agentDir` 认证存储读取：

```
~/.clawdbot/agents/<agentId>/agent/auth-profiles.json
```

凭据**不**在代理之间共享。永远不要在代理之间重用 `agentDir`。
如果你想共享凭据，请将 `auth-profiles.json` 复制到另一个代理的 `agentDir`。

有关沙箱在运行时的行为，请参见 [沙箱化](/gateway/sandboxing）。
有关调试"为什么这被阻止？"，请参见 [沙箱 vs 工具策略 vs 提升](/gateway/sandbox-vs-tool-policy-vs-elevated）和 `clawdbot sandbox explain`。

---

## 配置示例

### 示例 1：个人 + 受限家庭代理

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "default": true,
        "name": "Personal Assistant",
        "workspace": "~/clawd",
        "sandbox": { "mode": "off" }
      },
      {
        "id": "family",
        "name": "Family Bot",
        "workspace": "~/clawd-family",
        "sandbox": {
          "mode": "all",
          "scope": "agent"
        },
        "tools": {
          "allow": ["read"],
          "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"]
        }
      }
    ]
  },
  "bindings": [
    {
      "agentId": "family",
      "match": {
        "provider": "whatsapp",
        "accountId": "*",
        "peer": {
          "kind": "group",
          "id": "120363424282127706@g.us"
        }
      }
    }
  ]
}
```

**结果：**
- `main` 代理：在主机上运行，完全工具访问
- `family` 代理：在 Docker 中运行（每个代理一个容器），仅 `read` 工具

---

### 示例 2：具有共享沙箱的工作代理

```json
{
  "agents": {
    "list": [
      {
        "id": "personal",
        "workspace": "~/clawd-personal",
        "sandbox": { "mode": "off" }
      },
      {
        "id": "work",
        "workspace": "~/clawd-work",
        "sandbox": {
          "mode": "all",
          "scope": "shared",
          "workspaceRoot": "/tmp/work-sandboxes"
        },
        "tools": {
          "allow": ["read", "write", "apply_patch", "exec"],
          "deny": ["browser", "gateway", "discord"]
        }
      }
    ]
  }
}
```

---

### 示例 2b：全局编码配置文件 + 仅消息代理

```json
{
  "tools": { "profile": "coding" },
  "agents": {
    "list": [
      {
        "id": "support",
        "tools": { "profile": "messaging", "allow": ["slack"] }
      }
    ]
  }
}
```

**结果：**
- 默认代理获得编码工具
- `support` 代理仅消息（+ Slack 工具）

---

### 示例 3：每个代理的不同沙箱模式

```json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",  // 全局默认
        "scope": "session"
      }
    },
    "list": [
      {
        "id": "main",
        "workspace": "~/clawd",
        "sandbox": {
          "mode": "off"  // 覆盖：main 永远不沙箱化
        }
      },
      {
        "id": "public",
        "workspace": "~/clawd-public",
        "sandbox": {
          "mode": "all",  // 覆盖：public 始终沙箱化
          "scope": "agent"
        },
        "tools": {
          "allow": ["read"],
          "deny": ["exec", "write", "edit", "apply_patch"]
        }
      }
    ]
  }
}
```

---

## 配置优先级

当全局（`agents.defaults.*`）和代理特定（`agents.list[].*`）配置都存在时：

### 沙箱配置
代理特定设置覆盖全局：
```
agents.list[].sandbox.mode > agents.defaults.sandbox.mode
agents.list[].sandbox.scope > agents.defaults.sandbox.scope
agents.list[].sandbox.workspaceRoot > agents.defaults.sandbox.workspaceRoot
agents.list[].sandbox.workspaceAccess > agents.defaults.sandbox.workspaceAccess
agents.list[].sandbox.docker.* > agents.defaults.sandbox.docker.*
agents.list[].sandbox.browser.* > agents.defaults.sandbox.browser.*
agents.list[].sandbox.prune.* > agents.defaults.sandbox.prune.*
```

**说明：**
- `agents.list[].sandbox.{docker,browser,prune}.*` 覆盖该代理的 `agents.defaults.sandbox.{docker,browser,prune}.*`（当沙箱作用域解析为 `"shared"` 时忽略）。

### 工具限制
过滤顺序是：
1. **工具配置文件**（`tools.profile` 或 `agents.list[].tools.profile`）
2. **提供者工具配置文件**（`tools.byProvider[provider].profile` 或 `agents.list[].tools.byProvider[provider].profile`）
3. **全局工具策略**（`tools.allow` / `tools.deny`）
4. **提供者工具策略**（`tools.byProvider[provider].allow/deny`）
5. **代理特定工具策略**（`agents.list[].tools.allow/deny`）
6. **代理提供者策略**（`agents.list[].tools.byProvider[provider].allow/deny`）
7. **沙箱工具策略**（`tools.sandbox.tools` 或 `agents.list[].tools.sandbox.tools`）
8. **子代理工具策略**（`tools.subagents.tools`，如果适用）

每个级别可以进一步限制工具，但不能授予早期级别拒绝的工具。
如果设置了 `agents.list[].tools.sandbox.tools`，它将替换该代理的 `tools.sandbox.tools`。
如果设置了 `agents.list[].tools.profile`，它将覆盖该代理的 `tools.profile`。
提供者工具键接受 `provider`（例如 `google-antigravity`）或 `provider/model`（例如 `openai/gpt-5.2`）。

### 工具组（简写）

工具策略（全局、代理、沙箱）支持 `group:*` 条目，这些条目扩展到多个具体工具：

- `group:runtime`：`exec`、`bash`、`process`
- `group:fs`：`read`、`write`、`edit`、`apply_patch`
- `group:sessions`：`sessions_list`、`sessions_history`、`sessions_send`、`sessions_spawn`、`session_status`
- `group:memory`：`memory_search`、`memory_get`
- `group:ui`：`browser`、`canvas`
- `group:automation`：`cron`、`gateway`
- `group:messaging`：`message`
- `group:nodes`：`nodes`
- `group:clawdbot`：所有内置 Clawdbot 工具（不包括提供者插件）

### 提升模式
`tools.elevated` 是全局基线（基于发送者的允许列表）。`agents.list[].tools.elevated` 可以进一步限制特定代理的提升（两者都必须允许）。

缓解模式：
- 拒绝不受信任代理的 `exec`（`agents.list[].tools.deny: ["exec"]`）
- 避免允许列表路由到受限代理的发送者
- 如果你只想要沙箱执行，全局禁用提升（`tools.elevated.enabled: false`）
- 对于敏感配置文件，每个代理禁用提升（`agents.list[].tools.elevated.enabled: false`）

---

## 从单代理迁移

**之前（单代理）：**
```json
{
  "agents": {
    "defaults": {
      "workspace": "~/clawd",
      "sandbox": {
        "mode": "non-main"
      }
    }
  },
  "tools": {
    "sandbox": {
      "tools": {
        "allow": ["read", "write", "apply_patch", "exec"],
        "deny": []
      }
    }
  }
}
```

**之后（具有不同配置文件的多代理）：**
```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "default": true,
        "workspace": "~/clawd",
        "sandbox": { "mode": "off" }
      }
    ]
  }
}
```

传统 `agent.*` 配置由 `clawdbot doctor` 迁移；将来首选 `agents.defaults` + `agents.list`。

---

## 工具限制示例

### 只读代理
```json
{
  "tools": {
    "allow": ["read"],
    "deny": ["exec", "write", "edit", "apply_patch", "process"]
  }
}
```

### 安全执行代理（无文件修改）
```json
{
  "tools": {
    "allow": ["read", "exec", "process"],
    "deny": ["write", "edit", "apply_patch", "browser", "gateway"]
  }
}
```

### 仅通信代理
```json
{
  "tools": {
    "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
    "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
  }
}
```

---

## 常见陷阱："non-main"

`agents.defaults.sandbox.mode: "non-main"` 基于 `session.mainKey`（默认 `"main"`），不是代理 id。群组/通道会话总是获得自己的键，因此它们被视为 non-main 并将被沙箱化。如果你希望代理永远不沙箱化，请设置 `agents.list[].sandbox.mode: "off"`。

---

## 测试

配置多代理沙箱和工具后：

1. **检查代理解析：**
   ```exec
   clawdbot agents list --bindings
   ```

2. **验证沙箱容器：**
   ```exec
   docker ps --filter "label=clawdbot.sandbox=1"
   ```

3. **测试工具限制：**
   - 发送需要受限工具的消息
   - 验证代理无法使用被拒绝的工具

4. **监视日志：**
   ```exec
   tail -f "${CLAWDBOT_STATE_DIR:-$HOME/.clawdbot}/logs/gateway.log" | grep -E "routing|sandbox|tools"
   ```

---

## 故障排除

### 尽管 `mode: "all"`，代理未沙箱化
- 检查是否有全局 `agents.defaults.sandbox.mode` 覆盖它
- 代理特定配置优先，因此设置 `agents.list[].sandbox.mode: "all"`

### 尽管拒绝列表，工具仍然可用
- 检查工具过滤顺序：全局 → 代理 → 沙箱 → 子代理
- 每个级别只能进一步限制，不能授予回来
- 使用日志验证：`[tools] filtering tools for agent:${agentId}`

### 容器未按代理隔离
- 在代理特定沙箱配置中设置 `scope: "agent"`
- 默认是 `"session"`，它为每个会话创建一个容器

---

## 另请参见

- [多代理路由](/concepts/multi-agent）
- [沙箱配置](/gateway/configuration#agentsdefaults-sandbox）
- [会话管理](/concepts/session）
