# Clawdbot 深度学习教程

> **从源码出发，深入核心架构思路与技术实现**

本教程不是概念罗列，而是从源码追踪出发，带你理解这个项目**为什么这样设计**，以及**关键代码路径是怎么工作的**。

---

## 目录

**第一部分：核心设计决策**
1. [为什么 Gateway 是唯一真相来源](#1-为什么-gateway-是唯一真相来源)
2. [消息路由的确定性设计](#2-消息路由的确定性设计)
3. [Session Key 与身份隔离](#3-session-key-与身份隔离)

**第二部分：消息处理的真实流程**
4. [从 WhatsApp 消息到 AI 回复的完整路径](#4-从-whatsapp-消息到-ai-回复的完整路径)
5. [消息预处理的关键逻辑](#5-消息预处理的关键逻辑)
6. [流式输出与分块发送](#6-流式输出与分块发送)

**第三部分：Agent 运行机制**
7. [双层队列：为什么需要序列化执行](#7-双层队列为什么需要序列化执行)
8. [Auth Profile 轮换与故障转移](#8-auth-profile-轮换与故障转移)
9. [系统提示词的动态构建](#9-系统提示词的动态构建)

**第四部分：内存与检索系统**
10. [RAG 的真实实现：混合搜索](#10-rag-的真实实现混合搜索)
11. [会话压缩与内存刷新](#11-会话压缩与内存刷新)

**第五部分：插件系统**
12. [运行时 TypeScript 加载](#12-运行时-typescript-加载)
13. [如何编写一个消息渠道插件](#13-如何编写一个消息渠道插件)

**第六部分：实战指南**
14. [必读源码文件清单](#14-必读源码文件清单)
15. [调试技巧与日志追踪](#15-调试技巧与日志追踪)
16. [开发环境搭建](#16-开发环境搭建)

---

# 第一部分：核心设计决策

## 1. 为什么 Gateway 是唯一真相来源

### 1.1 问题背景

当你有多个客户端（macOS app、CLI、WebChat、iOS node）同时连接时，如何保证它们看到的状态是一致的？

**错误的做法**：每个客户端各自保存状态，通过消息同步。
**Clawdbot 的做法**：只有 Gateway 保存状态，客户端都是 "dumb terminals"。

### 1.2 源码证据

```typescript
// src/gateway/server.impl.ts 第147行
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer> {
  // 确保所有默认端口派生都能看到实际运行端口
  process.env.CLAWDBOT_GATEWAY_PORT = String(port);
  
  // 配置快照 - Gateway 是配置的唯一读取者
  let configSnapshot = await readConfigFileSnapshot();
  
  // 检查遗留配置并自动迁移
  if (configSnapshot.legacyIssues.length > 0) {
    const { config: migrated, changes } = migrateLegacyConfig(configSnapshot.parsed);
    await writeConfigFile(migrated);
  }
  // ...
}
```

**关键洞察**：

1. **端口注入环境变量** - 所有子组件通过 `CLAWDBOT_GATEWAY_PORT` 知道 Gateway 在哪
2. **配置迁移** - Gateway 启动时自动处理旧配置，客户端不需要关心
3. **单点写入** - 只有 Gateway 能写配置，客户端通过 RPC 请求修改

### 1.3 WebSocket 协议设计

```typescript
// 协议帧类型（来自 src/gateway/protocol/）
type Frame = 
  | { type: "req"; id: string; method: string; params: object }  // 请求
  | { type: "res"; id: string; ok: boolean; payload?: any }      // 响应
  | { type: "event"; event: string; payload: any; seq?: number } // 服务端推送

// 连接生命周期
// 1. 客户端必须先发 connect 帧
// 2. Gateway 返回 hello-ok，包含当前 presence + health 快照
// 3. 之后客户端可以发任意 RPC 请求
// 4. Gateway 主动推送 event（不需要请求）
```

**这解决了什么问题？**

- **状态同步**：hello-ok 返回完整快照，客户端不需要多次查询
- **实时更新**：event 推送让客户端立即知道变化
- **幂等性**：req 需要 idempotency key，Gateway 维护去重缓存

### 1.4 为什么只能有一个 WhatsApp 连接

```typescript
// src/web/auto-reply/monitor.ts
export async function monitorWebChannel(
  verbose: boolean,
  listenerFactory: typeof monitorWebInbox | undefined = monitorWebInbox,
  // ...
) {
  // Baileys 库要求单一会话
  // 如果多个进程同时连接，WhatsApp 会踢掉旧连接
}
```

这是 WhatsApp Web 协议的限制，不是 Clawdbot 的设计选择。但 Clawdbot 顺应这个限制，把**所有状态集中到 Gateway**。

---

## 2. 消息路由的确定性设计

### 2.1 核心问题

当一条消息从 Telegram 群组进来，应该发给哪个 Agent？应该用什么 Session Key？

**Clawdbot 的答案**：路由是**配置驱动的确定性算法**，不是 AI 决定的。

### 2.2 路由优先级（源码追踪）

```typescript
// src/routing/resolve-route.ts 第142-209行
export function resolveAgentRoute(input: ResolveAgentRouteInput): ResolvedAgentRoute {
  const channel = normalizeToken(input.channel);
  const accountId = normalizeAccountId(input.accountId);
  const peer = input.peer ? { kind: input.peer.kind, id: normalizeId(input.peer.id) } : null;
  const guildId = normalizeId(input.guildId);
  const teamId = normalizeId(input.teamId);

  // 1. 首先过滤出匹配 channel + accountId 的 bindings
  const bindings = listBindings(input.cfg).filter((binding) => {
    if (!matchesChannel(binding.match, channel)) return false;
    return matchesAccountId(binding.match?.accountId, accountId);
  });

  const dmScope = input.cfg.session?.dmScope ?? "main";
  const identityLinks = input.cfg.session?.identityLinks;

  // 选择函数：根据 agentId 生成完整的路由结果
  const choose = (agentId: string, matchedBy: ResolvedAgentRoute["matchedBy"]) => {
    const resolvedAgentId = pickFirstExistingAgentId(input.cfg, agentId);
    const sessionKey = buildAgentSessionKey({
      agentId: resolvedAgentId,
      channel,
      peer,
      dmScope,
      identityLinks,
    }).toLowerCase();
    return { agentId: resolvedAgentId, channel, accountId, sessionKey, matchedBy };
  };

  // 优先级 1: 精确 peer 匹配（最高优先级）
  if (peer) {
    const peerMatch = bindings.find((b) => matchesPeer(b.match, peer));
    if (peerMatch) return choose(peerMatch.agentId, "binding.peer");
  }

  // 优先级 2: Discord Guild 匹配
  if (guildId) {
    const guildMatch = bindings.find((b) => matchesGuild(b.match, guildId));
    if (guildMatch) return choose(guildMatch.agentId, "binding.guild");
  }

  // 优先级 3: Slack Team 匹配
  if (teamId) {
    const teamMatch = bindings.find((b) => matchesTeam(b.match, teamId));
    if (teamMatch) return choose(teamMatch.agentId, "binding.team");
  }

  // 优先级 4: Account 匹配（非通配符）
  const accountMatch = bindings.find(
    (b) => b.match?.accountId?.trim() !== "*" && !b.match?.peer && !b.match?.guildId
  );
  if (accountMatch) return choose(accountMatch.agentId, "binding.account");

  // 优先级 5: Channel 通配符匹配
  const anyAccountMatch = bindings.find(
    (b) => b.match?.accountId?.trim() === "*" && !b.match?.peer
  );
  if (anyAccountMatch) return choose(anyAccountMatch.agentId, "binding.channel");

  // 优先级 6: 默认 Agent（兜底）
  return choose(resolveDefaultAgentId(input.cfg), "default");
}
```

### 2.3 配置示例解读

```json5
// ~/.clawdbot/clawdbot.json
{
  // 定义多个 Agent
  agents: {
    list: [
      { id: "main", workspace: "~/clawd" },
      { id: "support", workspace: "~/clawd-support" },
      { id: "work", workspace: "~/clawd-work" }
    ]
  },
  
  // 路由绑定
  bindings: [
    // Slack T123 团队的所有消息 → support agent
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    
    // Telegram 群组 -100123 → work agent
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "work" },
    
    // 特定用户的私聊 → 指定 agent
    { match: { channel: "whatsapp", peer: { kind: "dm", id: "+1234567890" } }, agentId: "main" }
  ]
}
```

### 2.4 为什么不让 AI 决定路由？

1. **可预测性**：用户发消息后，能确定它会发给谁
2. **安全性**：AI 不能绕过路由规则访问其他 Agent 的上下文
3. **调试性**：出问题时能追踪 `matchedBy` 字段知道为什么路由到这里

---

## 3. Session Key 与身份隔离

### 3.1 Session Key 是什么？

Session Key 是**会话上下文的唯一标识符**。不同的 Session Key 意味着：
- 不同的对话历史
- 不同的 token 计数
- 不同的压缩状态

### 3.2 Session Key 生成规则（源码）

```typescript
// src/routing/session-key.ts 第110-149行
export function buildAgentPeerSessionKey(params: {
  agentId: string;
  mainKey?: string;
  channel: string;
  peerKind?: "dm" | "group" | "channel" | null;
  peerId?: string | null;
  identityLinks?: Record<string, string[]>;
  dmScope?: "main" | "per-peer" | "per-channel-peer";
}): string {
  const peerKind = params.peerKind ?? "dm";
  
  // ========== DM 处理 ==========
  if (peerKind === "dm") {
    const dmScope = params.dmScope ?? "main";
    let peerId = (params.peerId ?? "").trim();
    
    // 身份链接：把多个渠道的 ID 映射到同一个人
    const linkedPeerId = dmScope === "main" ? null : resolveLinkedPeerId({
      identityLinks: params.identityLinks,
      channel: params.channel,
      peerId,
    });
    if (linkedPeerId) peerId = linkedPeerId;
    peerId = peerId.toLowerCase();
    
    // dmScope = "per-channel-peer": 按渠道+发送者隔离
    // 例如: agent:main:telegram:dm:123456
    if (dmScope === "per-channel-peer" && peerId) {
      const channel = (params.channel ?? "").trim().toLowerCase() || "unknown";
      return `agent:${normalizeAgentId(params.agentId)}:${channel}:dm:${peerId}`;
    }
    
    // dmScope = "per-peer": 按发送者隔离（跨渠道共享）
    // 例如: agent:main:dm:alice
    if (dmScope === "per-peer" && peerId) {
      return `agent:${normalizeAgentId(params.agentId)}:dm:${peerId}`;
    }
    
    // dmScope = "main": 所有 DM 共享一个会话（默认）
    // 例如: agent:main:main
    return buildAgentMainSessionKey({ agentId: params.agentId, mainKey: params.mainKey });
  }
  
  // ========== 群组/频道处理（总是隔离）==========
  // 例如: agent:main:telegram:group:-100123
  const channel = (params.channel ?? "").trim().toLowerCase() || "unknown";
  const peerId = ((params.peerId ?? "").trim() || "unknown").toLowerCase();
  return `agent:${normalizeAgentId(params.agentId)}:${channel}:${peerKind}:${peerId}`;
}
```

### 3.3 三种 DM 隔离模式的使用场景

| 模式 | Session Key 示例 | 使用场景 |
|------|-----------------|---------|
| `main` | `agent:main:main` | 个人助手：所有私聊共享上下文 |
| `per-peer` | `agent:main:dm:alice` | 多用户：每人独立，但跨渠道共享 |
| `per-channel-peer` | `agent:main:telegram:dm:123` | 企业：严格隔离，同一人在不同渠道也分开 |

### 3.4 身份链接的妙用

```json5
// 配置示例
{
  session: {
    dmScope: "per-peer",
    identityLinks: {
      "alice": ["telegram:123456789", "discord:987654321"],
      "bob": ["whatsapp:+1234567890", "slack:U123456"]
    }
  }
}
```

当 `telegram:123456789` 发消息时，Session Key 会变成 `agent:main:dm:alice`。
当 `discord:987654321` 发消息时，Session Key **也是** `agent:main:dm:alice`。

这样 Alice 无论用哪个渠道，都能和 AI 继续之前的对话。

---

# 第二部分：消息处理的真实流程

## 4. 从 WhatsApp 消息到 AI 回复的完整路径

### 4.1 调用链追踪

```
┌─────────────────────────────────────────────────────────────────┐
│  WhatsApp Web (Baileys SDK)                                     │
│  └─ WebSocket 收到消息                                           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  src/web/auto-reply/monitor.ts                                  │
│  └─ monitorWebChannel()                                         │
│      └─ 注册消息监听器                                           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  src/web/auto-reply/monitor/on-message.ts                       │
│  └─ createWebOnMessageHandler()                                 │
│      └─ 消息过滤（群组权限、allowFrom 检查）                      │
│      └─ 解析路由 → resolveAgentRoute()                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  src/web/auto-reply/monitor/process-message.ts                  │
│  └─ processMessage()                                            │
│      └─ 构建入站消息体（包含群聊历史）                            │
│      └─ 回声检测（避免响应自己发的消息）                          │
│      └─ 发送 ack 反应                                            │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  src/auto-reply/reply/get-reply.ts                              │
│  └─ getReplyFromConfig()                                        │
│      └─ 加载配置、解析 Agent                                     │
│      └─ 媒体理解（图片/音频/视频）                               │
│      └─ 链接理解（URL 内容提取）                                 │
│      └─ 初始化会话状态                                           │
│      └─ 解析指令（/think, /verbose 等）                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  src/auto-reply/reply/get-reply-run.ts                          │
│  └─ runPreparedReply()                                          │
│      └─ 调用 Agent 运行时                                        │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  src/agents/pi-embedded-runner/run.ts                           │
│  └─ runEmbeddedPiAgent()                                        │
│      └─ 队列序列化（会话队列 + 全局队列）                         │
│      └─ 解析模型和认证                                           │
│      └─ 构建系统提示词                                           │
│      └─ 调用 LLM API                                             │
│      └─ 执行工具调用                                             │
│      └─ 流式输出处理                                             │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  src/web/auto-reply/deliver-reply.ts                            │
│  └─ deliverWebReply()                                           │
│      └─ 分块发送（长消息切分）                                   │
│      └─ 发送到 WhatsApp                                          │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 每个节点做什么？

| 文件 | 核心职责 | 关键决策 |
|------|---------|---------|
| `monitor.ts` | 建立 WebSocket 连接 | 重连策略、keep-alive |
| `on-message.ts` | 消息过滤 | 是否响应这条消息 |
| `process-message.ts` | 消息预处理 | 上下文聚合、回声检测 |
| `get-reply.ts` | 主入口 | 配置加载、指令解析 |
| `run.ts` | Agent 执行 | 队列、认证、LLM 调用 |

---

## 5. 消息预处理的关键逻辑

### 5.1 群聊历史上下文聚合

```typescript
// src/web/auto-reply/monitor/process-message.ts 第133-172行
export async function processMessage(params) {
  // 构建基础入站消息体
  let combinedBody = buildInboundLine({
    cfg: params.cfg,
    msg: params.msg,
    agentId: params.route.agentId,
    previousTimestamp,
    envelope: envelopeOptions,
  });

  // 如果是群聊，附加历史消息作为上下文
  if (params.msg.chatType === "group") {
    const history = params.groupHistory ?? params.groupHistories.get(params.groupHistoryKey) ?? [];
    
    if (history.length > 0) {
      // 把历史消息转换成统一格式
      const historyEntries: HistoryEntry[] = history.map((m) => ({
        sender: m.sender,
        body: m.body,
        timestamp: m.timestamp,
        messageId: m.id,
      }));
      
      // 构建包含历史的完整消息体
      combinedBody = buildHistoryContextFromEntries({
        entries: historyEntries,
        currentMessage: combinedBody,
        excludeLast: false,
        formatEntry: (entry) => {
          // 每条历史消息的格式化
          const bodyWithId = entry.messageId
            ? `${entry.body}\n[message_id: ${entry.messageId}]`
            : entry.body;
          return formatInboundEnvelope({
            channel: "WhatsApp",
            from: conversationId,
            timestamp: entry.timestamp,
            body: bodyWithId,
            chatType: "group",
            senderLabel: entry.sender,
          });
        },
      });
    }
  }
  // ...
}
```

**为什么需要群聊历史？**

在群聊中，AI 需要知道"谁说了什么"才能正确回复。比如：

```
Alice: 明天几点开会？
Bob: 10点
你: @Clawd 帮我总结一下
```

AI 需要看到前面的对话才能知道要总结什么。

### 5.2 回声检测：避免无限循环

```typescript
// src/web/auto-reply/monitor/process-message.ts 第174-183行

// 用 sessionKey + combinedBody 的 hash 作为 echo key
const combinedEchoKey = params.buildCombinedEchoKey({
  sessionKey: params.route.sessionKey,
  combinedBody,
});

// 如果这条消息是我们自己发的，跳过处理
if (params.echoHas(combinedEchoKey)) {
  logVerbose("Skipping auto-reply: detected echo for combined message");
  params.echoForget(combinedEchoKey);  // 消费掉这个 echo 标记
  return false;
}
```

**工作原理**：

1. 发送消息时，记录 `echo key = hash(sessionKey + body)`
2. 收到消息时，检查是否匹配已记录的 echo key
3. 如果匹配，说明是自己发的，跳过处理
4. 消费后删除，避免误判

### 5.3 即时反馈：ack 反应

```typescript
// 发送 ack 反应让用户知道消息已收到
maybeSendAckReaction({
  cfg: params.cfg,
  msg: params.msg,
  agentId: params.route.agentId,
  sessionKey: params.route.sessionKey,
  conversationId,
  verbose: params.verbose,
  accountId: params.route.accountId,
});
```

在 AI 处理期间，用户会看到一个表情反应（比如 👀），知道消息正在处理中。

---

## 6. 流式输出与分块发送

### 6.1 为什么需要分块？

1. **平台限制**：WhatsApp 单条消息最多 65536 字符
2. **用户体验**：太长的消息难以阅读
3. **Markdown 格式**：不能在代码块中间切断

### 6.2 EmbeddedBlockChunker 的核心算法

```typescript
// src/agents/pi-embedded-block-chunker.ts

class EmbeddedBlockChunker {
  private buffer: string = "";
  private minChars: number;   // 最小字符数才发送
  private maxChars: number;   // 最大字符数必须切分
  private breakPreference: "paragraph" | "newline" | "sentence" | "whitespace";

  emit(text: string) {
    this.buffer += text;
    
    // 低于最小值不发送（除非是最后一块）
    if (this.buffer.length < this.minChars) return;
    
    // 寻找合适的断点
    const breakPoint = this.findBreakPoint();
    
    // 特殊处理：代码块不能在中间切分
    if (this.isInsideCodeFence()) {
      // 方案：关闭代码块 → 发送 → 重新打开代码块
      const closeTag = "\n```\n";
      const reopenTag = "\n```" + this.currentFenceLanguage + "\n";
      // ...
    }
  }

  private findBreakPoint(): number {
    // 按优先级寻找断点
    // 1. 段落边界（两个换行）
    // 2. 单个换行
    // 3. 句号
    // 4. 空格
    // 5. 硬切（maxChars 位置）
  }
}
```

### 6.3 块流式传输 vs 最终发送

```typescript
// 配置选项
{
  agents: {
    defaults: {
      // 是否启用块流式传输
      blockStreamingDefault: "on" | "off",
      
      // 触发时机
      blockStreamingBreak: "text_end" | "message_end",
      
      // 分块参数
      blockStreamingChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph"
      }
    }
  }
}
```

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `text_end` | 每个文本块结束就发 | 需要即时反馈 |
| `message_end` | 全部生成完再发 | 内容需要完整性 |

---

# 第三部分：Agent 运行机制

## 7. 双层队列：为什么需要序列化执行

### 7.1 问题场景

用户快速发了 3 条消息：
```
用户: 1+1=?
用户: 2+2=?
用户: 3+3=?
```

如果并行处理，可能出现：
- 第 3 条的回复先到
- 上下文污染（三个回复混在一起）
- API 限流（同时请求太多）

### 7.2 双层队列设计

```typescript
// src/agents/pi-embedded-runner/run.ts 第69-89行
export async function runEmbeddedPiAgent(params): Promise<EmbeddedPiRunResult> {
  // 会话级别队列：确保同一会话内的消息按顺序处理
  const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
  
  // 全局队列：控制总体并发
  const globalLane = resolveGlobalLane(params.lane);
  
  // 嵌套入队：先进会话队列，再进全局队列
  return enqueueSession(() =>
    enqueueGlobal(async () => {
      // 真正的执行逻辑
      const started = Date.now();
      const resolvedWorkspace = resolveUserPath(params.workspaceDir);
      // ...
    })
  );
}
```

### 7.3 队列的工作方式

```
消息1 ──┐                    ┌──► 执行
消息2 ──┼── 会话队列(FIFO) ──┼──► 等待
消息3 ──┘                    └──► 等待

会话A ──┐                    ┌──► 执行
会话B ──┼── 全局队列(并发N) ──┼──► 执行
会话C ──┘                    └──► 等待
```

- **会话队列**：FIFO，同一会话内严格顺序
- **全局队列**：有限并发，防止系统过载

---

## 8. Auth Profile 轮换与故障转移

### 8.1 问题场景

- 一个 API key 被限流了
- OAuth token 过期了
- 主模型不可用

### 8.2 轮换策略（源码）

```typescript
// src/agents/pi-embedded-runner/run.ts 第266-292行

// 1. 获取 profile 候选列表
const profileOrder = resolveAuthProfileOrder({
  cfg: params.config,
  store: authStore,
  provider,
  preferredProfile: preferredProfileId,
});

// 2. 逐个尝试
try {
  while (profileIndex < profileCandidates.length) {
    const candidate = profileCandidates[profileIndex];
    
    // 检查冷却期：失败的 profile 暂时跳过
    if (candidate && isProfileInCooldown(authStore, candidate)) {
      profileIndex += 1;
      continue;
    }
    
    // 尝试使用这个 profile
    await applyApiKeyInfo(profileCandidates[profileIndex]);
    break;
  }
  
  // 所有 profile 都在冷却期
  if (profileIndex >= profileCandidates.length) {
    throwAuthProfileFailover({ allInCooldown: true });
  }
} catch (err) {
  // 当前 profile 失败，尝试下一个
  const advanced = await advanceAuthProfile();
  if (!advanced) {
    // 没有更多 profile 可用，尝试 fallback 模型
    throwAuthProfileFailover({ allInCooldown: false, error: err });
  }
}
```

### 8.3 错误分类

```typescript
// src/agents/pi-embedded-helpers.ts

// 根据错误消息判断故障类型
export function classifyFailoverReason(message: string): FailoverReason | null {
  if (isAuthErrorMessage(message)) return "auth";
  if (isRateLimitErrorMessage(message)) return "rate_limit";
  if (isContextOverflowError(message)) return "context_overflow";
  if (isTimeoutErrorMessage(message)) return "timeout";
  return null;
}
```

不同的错误类型有不同的处理策略：
- `auth`：切换 profile
- `rate_limit`：等待或切换
- `context_overflow`：触发压缩
- `timeout`：重试或切换

### 8.4 冷却期机制

```typescript
// 失败后标记冷却
markAuthProfileFailure(authStore, profileId, {
  cooldownMs: 60000,  // 60秒冷却
  reason: "rate_limit"
});

// 成功后标记正常
markAuthProfileGood(authStore, profileId);
```

---

## 9. 系统提示词的动态构建

### 9.1 提示词来源

系统提示词不是一个固定字符串，而是**动态组装**的：

```typescript
// src/auto-reply/reply/get-reply.ts

// 1. 加载工作区文件
const workspace = await ensureAgentWorkspace({
  dir: workspaceDirRaw,
  ensureBootstrapFiles: true,  // 确保 AGENTS.md 等文件存在
});

// 2. 加载身份文件
// IDENTITY.md - 助手身份
// AGENTS.md - 工作规范
// TOOLS.md - 工具说明
// SOUL.md - 性格特点
```

### 9.2 文件加载顺序

```
系统提示词 =
  基础提示词（Clawdbot 内置）
  + IDENTITY.md（如果存在）
  + AGENTS.md（如果存在）
  + TOOLS.md（如果存在）
  + SOUL.md（如果存在）
  + Skills 提示词（符合条件的技能）
  + Bootstrap 上下文（动态生成）
  + 日期时间
```

### 9.3 Skills 的动态过滤

```typescript
// 技能加载条件
interface SkillGating {
  // OS 限制
  os?: ("darwin" | "linux" | "win32")[];
  
  // 依赖检查
  requires?: {
    bins?: string[];      // 全部需要存在
    anyBins?: string[];   // 至少一个存在
    env?: string[];       // 环境变量
    config?: string[];    // 配置路径
  };
}
```

只有**满足所有条件**的技能才会被加入系统提示词。

---

# 第四部分：内存与检索系统

## 10. RAG 的真实实现：混合搜索

### 10.1 为什么需要混合搜索？

**纯向量搜索的问题**：
- "帮我找 ID a828e60 的记录" → 向量搜索可能找不到，因为 ID 没有语义

**纯关键词搜索的问题**：
- "运行网关的机器" → 找不到 "Mac Studio 网关主机"，因为词不一样

**混合搜索**：两个都用，取长补短。

### 10.2 实现原理

```typescript
// src/memory/hybrid.ts

function hybridSearch(query: string, options: SearchOptions): SearchResult[] {
  const candidatePool = options.maxResults * options.candidateMultiplier;
  
  // 1. 向量搜索：按语义相似度
  const vectorResults = vectorSearch(queryEmbedding, candidatePool);
  // 返回: [{ chunkId, vectorScore: 0.92 }, { chunkId, vectorScore: 0.87 }, ...]
  
  // 2. BM25 搜索：按关键词匹配
  const textResults = bm25Search(query, candidatePool);
  // 返回: [{ chunkId, bm25Rank: 1 }, { chunkId, bm25Rank: 2 }, ...]
  
  // 3. 归一化 BM25 分数
  // BM25 返回的是排名，需要转换成 0-1 的分数
  const normalizedText = textResults.map(r => ({
    ...r,
    textScore: 1 / (1 + Math.max(0, r.bm25Rank))
    // rank=1 → score=0.5
    // rank=2 → score=0.33
    // rank=10 → score=0.09
  }));
  
  // 4. 按 chunkId 合并
  const merged = unionByChunkId(vectorResults, normalizedText);
  
  // 5. 加权计算最终分数
  return merged.map(r => ({
    ...r,
    finalScore: vectorWeight * r.vectorScore + textWeight * r.textScore
    // 默认: vectorWeight=0.7, textWeight=0.3
  })).sort((a, b) => b.finalScore - a.finalScore)
    .slice(0, options.maxResults);
}
```

### 10.3 配置示例

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        // 嵌入提供商
        provider: "openai",  // openai | gemini | local
        model: "text-embedding-3-small",
        
        // 混合搜索配置
        query: {
          hybrid: {
            enabled: true,
            vectorWeight: 0.7,
            textWeight: 0.3,
            candidateMultiplier: 4  // 每种搜索取 maxResults*4 个候选
          }
        },
        
        // 索引存储
        store: {
          path: "~/.clawdbot/memory/{agentId}.sqlite",
          vector: {
            enabled: true  // 使用 sqlite-vec 加速
          }
        }
      }
    }
  }
}
```

### 10.4 索引更新策略

```typescript
// 监视文件变化
watcher.on("change", (path) => {
  // 防抖 1.5 秒
  debounce(() => {
    markIndexDirty();
  }, 1500);
});

// 会话开始时同步
onSessionStart(() => {
  if (isIndexDirty()) {
    await syncIndex();
  }
});

// 搜索时按需同步
onSearch(() => {
  if (isIndexDirty()) {
    await syncIndex();
  }
});
```

---

## 11. 会话压缩与内存刷新

### 11.1 为什么需要压缩？

每个模型有上下文窗口限制（比如 200K tokens）。长对话会超限。

### 11.2 自动压缩触发条件

```typescript
// 触发条件：tokens > contextWindow - reserveTokensFloor
// 例如：200K - 20K = 180K tokens 时触发

// src/agents/compaction.ts
if (sessionTokens > contextWindow - reserveTokensFloor) {
  await compactSession();
}
```

### 11.3 压缩流程

```
1. 检测到即将超限
       ↓
2. 触发内存刷新（可选）
   └─ 让模型把重要内容写入 memory/YYYY-MM-DD.md
       ↓
3. 生成压缩摘要
   └─ 调用模型总结旧对话
       ↓
4. 替换旧历史
   └─ 压缩摘要 + 最近 N 条消息
       ↓
5. 持久化
   └─ 写入 JSONL 文件
```

### 11.4 内存刷新配置

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,  // 提前 4K tokens 触发
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

---

# 第五部分：插件系统

## 12. 运行时 TypeScript 加载

### 12.1 为什么用 jiti？

插件是 TypeScript 文件，但 Node.js 不能直接执行 TypeScript。

**传统方案**：先编译成 JS，再加载
**Clawdbot 方案**：用 jiti 运行时编译

```typescript
// src/plugins/loader.ts 第164行
export function loadClawdbotPlugins(options): PluginRegistry {
  // 创建 jiti 实例
  const jiti = createJiti(process.cwd(), {
    alias: {
      "clawdbot/plugin-sdk": resolvePluginSdkAlias(),  // 让插件能 import SDK
    },
  });
  
  // 加载插件（TypeScript 文件）
  const moduleExport = jiti(pluginPath);
  
  // 解析导出
  const { definition, register } = resolvePluginModuleExport(moduleExport);
  
  // 注册
  if (register) {
    const api = createApi(record);
    register(api);
  }
}
```

### 12.2 插件发现顺序

```typescript
// 优先级从高到低
const pluginScanOrder = [
  // 1. 配置指定的路径
  "plugins.load.paths",
  
  // 2. 工作区扩展
  "<workspace>/.clawdbot/extensions/*.ts",
  "<workspace>/.clawdbot/extensions/*/index.ts",
  
  // 3. 全局扩展
  "~/.clawdbot/extensions/*.ts",
  "~/.clawdbot/extensions/*/index.ts",
  
  // 4. 捆绑扩展（默认禁用）
  "<clawdbot>/extensions/*"
];
```

### 12.3 插件 API

```typescript
// 插件可以使用的 API
interface PluginAPI {
  // 日志
  logger: Logger;
  
  // 配置
  config: ClawdbotConfig;
  
  // 注册能力
  registerGatewayMethod(name: string, handler: MethodHandler): void;
  registerTool(tool: ToolDefinition): void;
  registerCli(setup: CliSetup): void;
  registerChannel(channel: ChannelPlugin): void;
  registerProvider(provider: ProviderPlugin): void;
  registerCommand(command: CommandDefinition): void;
  registerService(service: ServiceDefinition): void;
  registerHook(hook: HookDefinition): void;
}
```

---

## 13. 如何编写一个消息渠道插件

### 13.1 最小可用示例

```typescript
// ~/.clawdbot/extensions/my-channel/index.ts

import type { ChannelPlugin, PluginAPI } from "clawdbot/plugin-sdk";

const myChannel: ChannelPlugin = {
  id: "my-channel",
  
  // 元数据：控制 UI 显示
  meta: {
    id: "my-channel",
    label: "My Channel",
    selectionLabel: "My Channel (API)",
    docsPath: "/channels/my-channel",
    blurb: "My custom messaging channel.",
    aliases: ["mc"],
  },
  
  // 能力声明
  capabilities: { 
    chatTypes: ["direct", "group"],
    media: true,
    threads: false,
  },
  
  // 配置读取
  config: {
    listAccountIds: (cfg) => 
      Object.keys(cfg.channels?.["my-channel"]?.accounts ?? {}),
    resolveAccount: (cfg, accountId) =>
      cfg.channels?.["my-channel"]?.accounts?.[accountId ?? "default"],
  },
  
  // 出站消息
  outbound: {
    deliveryMode: "direct",
    sendText: async ({ to, text, accountId }) => {
      // 调用你的 API 发送消息
      const result = await myApi.send(to, text);
      return { ok: result.success, messageId: result.id };
    },
    sendMedia: async ({ to, media, accountId }) => {
      // 发送媒体
    },
  },
  
  // 生命周期
  gateway: {
    start: async (ctx) => {
      // 建立连接，注册消息监听
      const client = await myApi.connect(ctx.config);
      client.on("message", (msg) => {
        ctx.onInboundMessage(normalizeMessage(msg));
      });
    },
    stop: async () => {
      // 断开连接
      await myApi.disconnect();
    },
  },
};

// 导出注册函数
export default function(api: PluginAPI) {
  api.registerChannel({ plugin: myChannel });
}
```

### 13.2 配置文件

```json5
// ~/.clawdbot/clawdbot.json
{
  channels: {
    "my-channel": {
      accounts: {
        default: {
          apiKey: "xxx",
          enabled: true
        }
      }
    }
  }
}
```

### 13.3 Manifest 文件

```json
// ~/.clawdbot/extensions/my-channel/clawdbot.plugin.json
{
  "id": "my-channel",
  "name": "My Channel Plugin",
  "version": "1.0.0",
  "description": "Custom messaging channel",
  "configSchema": {
    "type": "object",
    "properties": {
      "apiKey": { "type": "string" }
    }
  }
}
```

---

# 第六部分：实战指南

## 14. 必读源码文件清单

按这个顺序阅读，能最快理解项目：

### 14.1 第一层：入口和路由

| 文件 | 行数 | 内容 |
|------|------|------|
| `src/index.ts` | ~95 | CLI 入口，全局初始化 |
| `src/routing/resolve-route.ts` | ~210 | 消息路由算法 |
| `src/routing/session-key.ts` | ~212 | Session Key 生成 |

### 14.2 第二层：消息处理

| 文件 | 行数 | 内容 |
|------|------|------|
| `src/web/auto-reply/monitor/process-message.ts` | ~400 | WhatsApp 消息处理 |
| `src/auto-reply/reply/get-reply.ts` | ~300 | 回复获取主入口 |
| `src/telegram/bot-handlers.ts` | ~600 | Telegram 消息处理 |

### 14.3 第三层：Agent 运行

| 文件 | 行数 | 内容 |
|------|------|------|
| `src/agents/pi-embedded-runner/run.ts` | ~650 | Agent 运行核心 |
| `src/agents/pi-embedded-subscribe.ts` | ~500 | 事件订阅和流式处理 |
| `src/agents/pi-embedded-block-chunker.ts` | ~300 | 块分割算法 |

### 14.4 第四层：Gateway

| 文件 | 行数 | 内容 |
|------|------|------|
| `src/gateway/server.impl.ts` | ~600 | Gateway 启动 |
| `src/gateway/server-methods.ts` | ~200 | RPC 方法注册 |
| `src/gateway/server-chat.ts` | ~270 | Chat 事件处理 |

### 14.5 第五层：插件

| 文件 | 行数 | 内容 |
|------|------|------|
| `src/plugins/loader.ts` | ~450 | 插件加载 |
| `src/plugins/registry.ts` | ~300 | 插件注册表 |
| `src/plugins/runtime/index.ts` | ~200 | 插件运行时 |

---

## 15. 调试技巧与日志追踪

### 15.1 启用详细日志

```bash
# 方法1：CLI 参数
clawdbot gateway --verbose

# 方法2：环境变量
CLAWDBOT_VERBOSE=1 clawdbot gateway

# 方法3：调试特定子系统
DEBUG=clawdbot:agent clawdbot gateway
```

### 15.2 查看实时日志

```bash
# macOS 统一日志
./scripts/clawlog.sh -f

# 或者直接看日志文件
tail -f /tmp/clawdbot-gateway.log
```

### 15.3 追踪消息流程

在关键位置加 `console.log`：

```typescript
// src/web/auto-reply/monitor/process-message.ts
console.log("[TRACE] Processing message:", {
  from: params.msg.from,
  body: params.msg.body?.slice(0, 50),
  sessionKey: params.route.sessionKey,
});
```

### 15.4 检查会话状态

```bash
# 列出所有会话
clawdbot sessions list

# 查看特定会话
clawdbot sessions get agent:main:main

# 清除会话
clawdbot sessions reset agent:main:main
```

---

## 16. 开发环境搭建

### 16.1 基础环境

```bash
# 1. 克隆代码
git clone https://github.com/clawdbot/clawdbot.git
cd clawdbot

# 2. 安装依赖
pnpm install

# 3. 构建 UI
pnpm ui:build

# 4. 构建项目
pnpm build

# 5. 运行开发模式
pnpm gateway:watch
```

### 16.2 常用命令

```bash
# 类型检查
pnpm build

# 运行测试
pnpm test

# 代码检查
pnpm lint

# 开发模式（自动重载）
pnpm gateway:watch

# 直接运行 TypeScript
pnpm clawdbot gateway --verbose
```

### 16.3 测试特定模块

```bash
# 运行特定测试文件
pnpm test src/routing/resolve-route.test.ts

# 运行匹配模式的测试
pnpm test -t "session key"

# 带覆盖率
pnpm test:coverage
```

### 16.4 调试技巧

```bash
# VS Code 调试配置
# .vscode/launch.json
{
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Gateway",
      "runtimeExecutable": "pnpm",
      "runtimeArgs": ["clawdbot", "gateway", "--verbose"],
      "skipFiles": ["<node_internals>/**"]
    }
  ]
}
```

---

## 附录：设计模式速查

| 模式 | 使用位置 | 代码文件 | 解决的问题 |
|------|---------|---------|-----------|
| 单例 | Gateway 实例 | `server.impl.ts` | 确保只有一个 WhatsApp 连接 |
| 命令队列 | Agent 运行 | `run.ts` | 序列化执行，避免竞态 |
| 策略 | 渠道适配器 | `channels/plugins/` | 统一接口，支持多渠道 |
| 观察者 | WebSocket 事件 | `server-chat.ts` | 多客户端同步 |
| 工厂 | 插件加载 | `loader.ts` | 运行时创建插件实例 |
| 责任链 | 消息处理 | `get-reply.ts` | 层层过滤和转换 |
| 装饰器 | 日志注入 | `subsystem.ts` | 添加上下文信息 |

---

## 结语

这份教程试图从**源码出发**，带你理解 Clawdbot 的核心设计。

**记住几个关键点**：

1. **Gateway 是老大** - 所有状态在 Gateway，客户端都是终端
2. **路由是确定的** - 配置驱动，不是 AI 决定
3. **队列保证顺序** - 双层队列避免竞态
4. **混合搜索** - 向量 + BM25，取长补短
5. **插件即代码** - jiti 运行时加载 TypeScript

如果你要深入开发，建议按照"必读源码文件清单"的顺序阅读代码，边读边跑测试，很快就能上手。

---

*文档版本: 2026.01.26*
*基于 Clawdbot 源码分析*
