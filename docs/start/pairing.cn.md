---
summary: "Pairing overview: approve who can DM you + which nodes can join"
read_when:
  - Setting up DM access control
  - Pairing a new iOS/Android node
  - Reviewing Clawdbot security posture
---

# 配对

"配对"是 Clawdbot 的显式**所有者批准**步骤。
它在两个地方使用：

1) **私信配对**（允许谁与机器人对话）
2) **节点配对**（允许哪些设备/节点加入网关网络）

安全上下文：[安全](/gateway/security）

## 1) 私信配对（入站聊天访问）

当通道配置了私信策略 `pairing` 时，未知发送者会收到短代码，他们的消息在批准之前**不会被处理**。

默认私信策略记录在：[安全](/gateway/security）

配对代码：
- 8 个字符，大写，无歧义字符（`0O1I`）。
- **1 小时后过期**。机器人仅在创建新请求时发送配对消息（每个发送者大约每小时一次）。
- 待处理的私信配对请求默认每个通道限制为**3 个**；其他请求将被忽略，直到一个过期或被批准。

### 批准发送者

```bash
clawdbot pairing list telegram
clawdbot pairing approve telegram <CODE>
```

支持的通道：`telegram`、`whatsapp`、`signal`、`imessage`、`discord`、`slack`。

### 状态存储位置

存储在 `~/.clawdbot/credentials/` 下：
- 待处理请求：`<channel>-pairing.json`
- 已批准允许列表存储：`<channel>-allowFrom.json`

将这些视为敏感（它们控制对你的助手的访问）。


## 2) 节点设备配对（iOS/Android/macOS/无头节点）

节点作为**设备**以 `role: node` 连接到 Gateway。Gateway 创建一个必须批准的设备配对请求。

### 批准节点设备

```bash
clawdbot devices list
clawdbot devices approve <requestId>
clawdbot devices reject <requestId>
```

### 状态存储位置

存储在 `~/.clawdbot/devices/` 下：
- `pending.json`（短期；待处理请求过期）
- `paired.json`（配对设备 + 令牌）

### 说明

- 传统 `node.pair.*` API（CLI：`clawdbot nodes pending/approve`）是单独的网关拥有的配对存储。WS 节点仍需要设备配对。


## 相关文档

- 安全模型 + 提示注入：[安全](/gateway/security）
- 安全更新（运行 doctor）：[更新](/install/updating）
- 通道配置：
  - Telegram：[Telegram](/channels/telegram）
  - WhatsApp：[WhatsApp](/channels/whatsapp）
  - Signal：[Signal](/channels/signal）
  - iMessage：[iMessage](/channels/imessage）
  - Discord：[Discord](/channels/discord）
  - Slack：[Slack](/channels/slack）
