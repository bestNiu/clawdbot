---
summary: "How Clawdbot rotates auth profiles and falls back across models"
read_when:
  - Diagnosing auth profile rotation, cooldowns, or model fallback behavior
  - Updating failover rules for auth profiles or models
---

# 模型故障转移

Clawdbot 在两个阶段处理故障：
1) **认证配置文件轮换**在当前提供者内。
2) **模型回退**到 `agents.defaults.model.fallbacks` 中的下一个模型。

本文档解释了运行时规则和支持它们的数据。

## 认证存储（密钥 + OAuth）

Clawdbot 对 API 密钥和 OAuth 令牌使用**认证配置文件**。

- 密钥位于 `~/.clawdbot/agents/<agentId>/agent/auth-profiles.json`（传统：`~/.clawdbot/agent/auth-profiles.json`）。
- 配置 `auth.profiles` / `auth.order` 是**元数据 + 路由仅**（无密钥）。
- 传统仅导入 OAuth 文件：`~/.clawdbot/credentials/oauth.json`（首次使用时导入到 `auth-profiles.json`）。

更多详细信息：[/concepts/oauth](/concepts/oauth)

凭据类型：
- `type: "api_key"` → `{ provider, key }`
- `type: "oauth"` → `{ provider, access, refresh, expires, email? }`（+ 某些提供者的 `projectId`/`enterpriseUrl`）

## 配置文件 ID

OAuth 登录创建不同的配置文件，以便多个账户可以共存。
- 默认：当没有可用电子邮件时为 `provider:default`。
- 带电子邮件的 OAuth：`provider:<email>`（例如 `google-antigravity:user@gmail.com`）。

配置文件位于 `~/.clawdbot/agents/<agentId>/agent/auth-profiles.json` 下的 `profiles`。

## 轮换顺序

当提供者有多个配置文件时，Clawdbot 按以下方式选择顺序：

1) **显式配置**：`auth.order[provider]`（如果设置）。
2) **配置的配置文件**：按提供者过滤的 `auth.profiles`。
3) **存储的配置文件**：`auth-profiles.json` 中该提供者的条目。

如果未配置显式顺序，Clawdbot 使用轮询顺序：
- **主键**：配置文件类型（**OAuth 在 API 密钥之前**）。
- **次键**：`usageStats.lastUsed`（最旧优先，在每个类型内）。
- **冷却/禁用的配置文件**移动到末尾，按最早过期排序。

### 会话粘性（缓存友好）

Clawdbot **为每个会话固定选择的认证配置文件**以保持提供者缓存温暖。
它**不**在每个请求上轮换。固定的配置文件被重用，直到：
- 会话被重置（`/new` / `/reset`）
- 压缩完成（压缩计数递增）
- 配置文件处于冷却/禁用状态

通过 `/model …@<profileId>` 的手动选择为该会话设置**用户覆盖**，并在新会话开始之前不会自动轮换。

自动固定的配置文件（由会话路由器选择）被视为**首选项**：它们首先尝试，但 Clawdbot 可能在速率限制/超时时轮换到另一个配置文件。
用户固定的配置文件保持锁定到该配置文件；如果它失败且配置了模型回退，Clawdbot 移动到下一个模型而不是切换配置文件。

### 为什么 OAuth 可能"看起来丢失"

如果你对同一提供者同时拥有 OAuth 配置文件和 API 密钥配置文件，轮询可以在消息之间切换它们，除非固定。要强制单个配置文件：
- 使用 `auth.order[provider] = ["provider:profileId"]` 固定，或
- 通过 `/model …` 使用配置文件覆盖（当你的 UI/聊天界面支持时）进行每个会话覆盖。

## 冷却

当配置文件由于认证/速率限制错误（或看起来像速率限制的超时）而失败时，Clawdbot 将其标记为冷却并移动到下一个配置文件。
格式/无效请求错误（例如 Cloud Code Assist 工具调用 ID 验证失败）被视为值得故障转移，并使用相同的冷却。

冷却使用指数退避：
- 1 分钟
- 5 分钟
- 25 分钟
- 1 小时（上限）

状态存储在 `auth-profiles.json` 下的 `usageStats` 中：

```json
{
  "usageStats": {
    "provider:profile": {
      "lastUsed": 1736160000000,
      "cooldownUntil": 1736160600000,
      "errorCount": 2
    }
  }
}
```

## 计费禁用

计费/信用失败（例如"信用不足"/"信用余额过低"）被视为值得故障转移，但它们通常不是瞬时的。而不是短冷却，Clawdbot 将配置文件标记为**禁用**（具有更长的退避）并轮换到下一个配置文件/提供者。

状态存储在 `auth-profiles.json` 中：

```json
{
  "usageStats": {
    "provider:profile": {
      "disabledUntil": 1736178000000,
      "disabledReason": "billing"
    }
  }
}
```

默认值：
- 计费退避从**5 小时**开始，每次计费失败时加倍，并上限为**24 小时**。
- 如果配置文件在**24 小时**内没有失败，退避计数器重置（可配置）。

## 模型回退

如果提供者的所有配置文件都失败，Clawdbot 移动到 `agents.defaults.model.fallbacks` 中的下一个模型。这适用于认证失败、速率限制和耗尽配置文件轮换的超时（其他错误不推进回退）。

当运行以模型覆盖（钩子或 CLI）开始时，回退在尝试任何配置的回退后仍以 `agents.defaults.model.primary` 结束。

## 相关配置

有关以下内容，请参见 [Gateway 配置](/gateway/configuration)：
- `auth.profiles` / `auth.order`
- `auth.cooldowns.billingBackoffHours` / `auth.cooldowns.billingBackoffHoursByProvider`
- `auth.cooldowns.billingMaxHours` / `auth.cooldowns.failureWindowHours`
- `agents.defaults.model.primary` / `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel` 路由

有关更广泛的模型选择和回退概述，请参见 [模型](/concepts/models)。
