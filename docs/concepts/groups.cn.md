---
summary: "Group chat behavior across surfaces (WhatsApp/Telegram/Discord/Slack/Signal/iMessage/Microsoft Teams)"
read_when:
  - Changing group chat behavior or mention gating
---
# 群组

Clawdbot 在所有界面上一致地处理群聊：WhatsApp、Telegram、Discord、Slack、Signal、iMessage、Microsoft Teams。

## 初学者介绍（2 分钟）
Clawdbot"生活"在你自己的消息账户上。没有单独的 WhatsApp 机器人用户。
如果**你**在群组中，Clawdbot 可以看到该群组并在那里响应。

默认行为：
- 群组受限（`groupPolicy: "allowlist"`）。
- 回复需要提及，除非你显式禁用提及门控。

翻译：允许列表中的发送者可以通过提及它来触发 Clawdbot。

> TL;DR
> - **私信访问**由 `*.allowFrom` 控制。
> - **群组访问**由 `*.groupPolicy` + 允许列表（`*.groups`、`*.groupAllowFrom`）控制。
> - **回复触发**由提及门控（`requireMention`、`/activation`）控制。

快速流程（群组消息发生的情况）：
```
groupPolicy? disabled -> drop
groupPolicy? allowlist -> group allowed? no -> drop
requireMention? yes -> mentioned? no -> store for context only
otherwise -> reply
```

![Group message flow](/images/groups-flow.svg)

如果你想要...
| 目标 | 设置什么 |
|------|-------------|
| 允许所有群组但仅在 @mentions 时回复 | `groups: { "*": { requireMention: true } }` |
| 禁用所有群组回复 | `groupPolicy: "disabled"` |
| 仅特定群组 | `groups: { "<group-id>": { ... } }`（无 `"*"` 键） |
| 只有你可以触发群组 | `groupPolicy: "allowlist"`，`groupAllowFrom: ["+1555..."]` |

## 会话键
- 群组会话使用 `agent:<agentId>:<channel>:group:<id>` 会话键（房间/通道使用 `agent:<agentId>:<channel>:channel:<id>`）。
- Telegram 论坛主题将 `:topic:<threadId>` 添加到群组 ID，以便每个主题都有自己的会话。
- 直接聊天使用主会话（或如果配置则为每个发送者）。
- 群组会话跳过心跳。

## 模式：个人私信 + 公共群组（单代理）

是的 — 如果你的"个人"流量是**私信**，你的"公共"流量是**群组**，这很好用。

为什么：在单代理模式下，私信通常落在**主**会话键（`agent:main:main`）中，而群组始终使用**非主**会话键（`agent:main:<channel>:group:<id>`）。如果你使用 `mode: "non-main"` 启用沙箱，这些群组会话在 Docker 中运行，而你的主私信会话保持在主机上。

这给你一个代理"大脑"（共享工作区 + 记忆），但有两个执行姿态：
- **私信**：完整工具（主机）
- **群组**：沙箱 + 受限工具（Docker）

> 如果你需要真正独立的工作区/角色（"个人"和"公共"绝不能混合），使用第二个代理 + 绑定。参见 [多代理路由](/concepts/multi-agent)。

示例（主机上的私信，沙箱化的群组 + 仅消息工具）：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // 群组/通道是非主 -> 沙箱化
        scope: "session", // 最强隔离（每个群组/通道一个容器）
        workspaceAccess: "none"
      }
    }
  },
  tools: {
    sandbox: {
      tools: {
        // 如果 allow 非空，其他所有内容都被阻止（deny 仍然获胜）。
        allow: ["group:messaging", "group:sessions"],
        deny: ["group:runtime", "group:fs", "group:ui", "nodes", "cron", "gateway"]
      }
    }
  }
}
```

想要"群组只能看到文件夹 X"而不是"无主机访问"？保持 `workspaceAccess: "none"` 并仅将允许列表的路径挂载到沙箱：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
        docker: {
          binds: [
            // hostPath:containerPath:mode
            "~/FriendsShared:/data:ro"
          ]
        }
      }
    }
  }
}
```

相关：
- 配置键和默认值：[Gateway 配置](/gateway/configuration#agentsdefaultssandbox)
- 调试为什么工具被阻止：[沙箱 vs 工具策略 vs 提升](/gateway/sandbox-vs-tool-policy-vs-elevated)
- 绑定挂载详细信息：[沙箱](/gateway/sandboxing#custom-bind-mounts)

## 显示标签
- UI 标签在可用时使用 `displayName`，格式为 `<channel>:<token>`。
- `#room` 保留用于房间/通道；群聊使用 `g-<slug>`（小写，空格 -> `-`，保留 `#@+._-`）。

## 群组策略
控制每个通道如何处理群组/房间消息：

```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "disabled", // "open" | "disabled" | "allowlist"
      groupAllowFrom: ["+15551234567"]
    },
    telegram: {
      groupPolicy: "disabled",
      groupAllowFrom: ["123456789", "@username"]
    },
    signal: {
      groupPolicy: "disabled",
      groupAllowFrom: ["+15551234567"]
    },
    imessage: {
      groupPolicy: "disabled",
      groupAllowFrom: ["chat_id:123"]
    },
    msteams: {
      groupPolicy: "disabled",
      groupAllowFrom: ["user@org.com"]
    },
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "GUILD_ID": { channels: { help: { allow: true } } }
      }
    },
    slack: {
      groupPolicy: "allowlist",
      channels: { "#general": { allow: true } }
    },
    matrix: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["@owner:example.org"],
      groups: {
        "!roomId:example.org": { allow: true },
        "#alias:example.org": { allow: true }
      }
    }
  }
}
```

| 策略 | 行为 |
|--------|----------|
| `"open"` | 群组绕过允许列表；提及门控仍然适用。 |
| `"disabled"` | 完全阻止所有群组消息。 |
| `"allowlist"` | 仅允许匹配配置的允许列表的群组/房间。 |

注意：
- `groupPolicy` 与提及门控（需要 @mentions）分开。
- WhatsApp/Telegram/Signal/iMessage/Microsoft Teams：使用 `groupAllowFrom`（回退：显式 `allowFrom`）。
- Discord：允许列表使用 `channels.discord.guilds.<id>.channels`。
- Slack：允许列表使用 `channels.slack.channels`。
- Matrix：允许列表使用 `channels.matrix.groups`（房间 ID、别名或名称）。使用 `channels.matrix.groupAllowFrom` 限制发送者；也支持每个房间的 `users` 允许列表。
- 群组私信单独控制（`channels.discord.dm.*`、`channels.slack.dm.*`）。
- Telegram 允许列表可以匹配用户 ID（`"123456789"`、`"telegram:123456789"`、`"tg:123456789"`）或用户名（`"@alice"` 或 `"alice"`）；前缀不区分大小写。
- 默认是 `groupPolicy: "allowlist"`；如果你的群组允许列表为空，群组消息被阻止。

快速心理模型（群组消息的评估顺序）：
1) `groupPolicy`（open/disabled/allowlist）
2) 群组允许列表（`*.groups`、`*.groupAllowFrom`、特定通道的允许列表）
3) 提及门控（`requireMention`、`/activation`）

## 提及门控（默认）
群组消息需要提及，除非每个群组覆盖。默认值位于每个子系统下的 `*.groups."*"`。

回复机器人消息算作隐式提及（当通道支持回复元数据时）。这适用于 Telegram、WhatsApp、Slack、Discord 和 Microsoft Teams。

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
        "123@g.us": { requireMention: false }
      }
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "123456789": { requireMention: false }
      }
    },
    imessage: {
      groups: {
        "*": { requireMention: true },
        "123": { requireMention: false }
      }
    }
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          mentionPatterns: ["@clawd", "clawdbot", "\\+15555550123"],
          historyLimit: 50
        }
      }
    ]
  }
}
```

注意：
- `mentionPatterns` 是不区分大小写的正则表达式。
- 提供显式提及的界面仍然通过；模式是回退。
- 每个代理覆盖：`agents.list[].groupChat.mentionPatterns`（当多个代理共享群组时有用）。
- 仅在可能进行提及检测时强制执行提及门控（本机提及或配置了 `mentionPatterns`）。
- Discord 默认值位于 `channels.discord.guilds."*"`（每个公会/通道可覆盖）。
- 群组历史上下文在所有通道上统一包装，并且是**仅待处理的**（由于提及门控而跳过的消息）；使用 `messages.groupChat.historyLimit` 作为全局默认值，使用 `channels.<channel>.historyLimit`（或 `channels.<channel>.accounts.*.historyLimit`）进行覆盖。设置 `0` 以禁用。

## 群组允许列表
当配置 `channels.whatsapp.groups`、`channels.telegram.groups` 或 `channels.imessage.groups` 时，键充当群组允许列表。使用 `"*"` 允许所有群组，同时仍设置默认提及行为。

常见意图（复制/粘贴）：

1) 禁用所有群组回复
```json5
{
  channels: { whatsapp: { groupPolicy: "disabled" } }
}
```

2) 仅允许特定群组（WhatsApp）
```json5
{
  channels: {
    whatsapp: {
      groups: {
        "123@g.us": { requireMention: true },
        "456@g.us": { requireMention: false }
      }
    }
  }
}
```

3) 允许所有群组但需要提及（显式）
```json5
{
  channels: {
    whatsapp: {
      groups: { "*": { requireMention: true } }
    }
  }
}
```

4) 只有所有者可以在群组中触发（WhatsApp）
```json5
{
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
      groups: { "*": { requireMention: true } }
    }
  }
}
```

## 激活（仅所有者）
群组所有者可以切换每个群组的激活：
- `/activation mention`
- `/activation always`

所有者由 `channels.whatsapp.allowFrom`（或未设置时的机器人自己的 E.164）确定。将命令作为独立消息发送。其他界面目前忽略 `/activation`。

## 上下文字段
群组入站负载设置：
- `ChatType=group`
- `GroupSubject`（如果已知）
- `GroupMembers`（如果已知）
- `WasMentioned`（提及门控结果）
- Telegram 论坛主题还包括 `MessageThreadId` 和 `IsForum`。

代理系统提示在新群组会话的第一轮中包含群组介绍。它提醒模型像人类一样响应，避免 Markdown 表格，并避免输入字面 `\n` 序列。

## iMessage 特定
- 路由或允许列表时首选 `chat_id:<id>`。
- 列出聊天：`imsg chats --limit 20`。
- 群组回复始终返回到同一个 `chat_id`。

## WhatsApp 特定
有关 WhatsApp 特定行为（历史注入、提及处理详细信息），请参见 [群组消息](/concepts/group-messages)。
