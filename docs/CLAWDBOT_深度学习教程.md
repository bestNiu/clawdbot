# Clawdbot 深度学习教程

> 从架构设计到技术实现的全方位解析

---

## 目录

1. [项目概述](#1-项目概述)
2. [核心架构设计](#2-核心架构设计)
3. [消息处理流水线](#3-消息处理流水线)
4. [Agent 智能体系统](#4-agent-智能体系统)
5. [会话管理与内存系统](#5-会话管理与内存系统)
6. [插件系统架构](#6-插件系统架构)
7. [多渠道集成](#7-多渠道集成)
8. [工具与技能系统](#8-工具与技能系统)
9. [RAG 与向量检索](#9-rag-与向量检索)
10. [Workflow 工作流设计](#10-workflow-工作流设计)
11. [源码目录深度解析](#11-源码目录深度解析)
12. [开发实践指南](#12-开发实践指南)
13. [扩展与定制](#13-扩展与定制)
14. [部署与运维](#14-部署与运维)

---

## 1. 项目概述

### 1.1 什么是 Clawdbot

Clawdbot 是一个**个人 AI 助手平台**，核心设计理念是：

- **本地优先**：运行在你自己的设备上
- **多渠道统一**：支持 WhatsApp、Telegram、Slack、Discord、Signal、iMessage 等
- **单一控制面**：Gateway 作为唯一的控制平面
- **可扩展**：通过插件系统支持任意扩展

### 1.2 核心设计哲学

```
┌─────────────────────────────────────────────────────────────┐
│                     设计原则                                 │
├─────────────────────────────────────────────────────────────┤
│ 1. Gateway 是唯一真相来源 (Single Source of Truth)          │
│ 2. 消息路由是确定性的，不由模型决定                          │
│ 3. 会话状态持久化在 Gateway 主机上                           │
│ 4. 插件作为可信代码在进程内运行                              │
│ 5. 工具执行发生在 Gateway 主机或连接的节点上                 │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 技术栈概览

| 层级 | 技术选型 | 说明 |
|------|----------|------|
| 运行时 | Node.js 22+ / Bun | 支持 TypeScript 直接执行 |
| 语言 | TypeScript (ESM) | 严格类型，避免 any |
| 构建 | tsc / pnpm | 输出到 dist/ |
| 测试 | Vitest | V8 覆盖率阈值 70% |
| Lint | Oxlint / Oxfmt | 快速格式化 |
| 原生应用 | SwiftUI (macOS/iOS) / Kotlin (Android) | 伴侣应用 |

---

## 2. 核心架构设计

### 2.1 系统架构图

```
WhatsApp / Telegram / Slack / Discord / Signal / iMessage / WebChat
                        │
                        ▼
         ┌──────────────────────────────┐
         │           Gateway            │
         │        (控制平面)             │
         │    ws://127.0.0.1:18789      │
         └──────────────┬───────────────┘
                        │
          ┌─────────────┼─────────────┐
          │             │             │
          ▼             ▼             ▼
     ┌─────────┐  ┌─────────┐  ┌─────────┐
     │Pi Agent │  │  CLI    │  │ WebChat │
     │  (RPC)  │  │         │  │   UI    │
     └─────────┘  └─────────┘  └─────────┘
          │
          ▼
     ┌─────────────────────────────────┐
     │         Nodes (设备节点)         │
     │   macOS / iOS / Android         │
     │   Canvas / Camera / Location    │
     └─────────────────────────────────┘
```

### 2.2 Gateway 核心组件

Gateway 是整个系统的**控制平面**，负责：

```typescript
// Gateway 核心职责
interface GatewayResponsibilities {
  // 1. 维护消息渠道连接
  channelConnections: {
    whatsapp: BaileysSession;    // WhatsApp Web 协议
    telegram: GrammyBot;         // Telegram Bot API
    discord: DiscordJS;          // Discord.js
    slack: BoltApp;              // Slack Bolt
    // ...更多渠道
  };

  // 2. WebSocket API 服务
  wsServer: {
    port: 18789;
    frames: JSONPayload[];       // 类型化的请求/响应/事件
    validation: JSONSchema;      // 入站帧验证
  };

  // 3. 会话管理
  sessions: SessionStore;        // 会话状态持久化
  
  // 4. Agent 运行时
  agentRuntime: PiAgentCore;     // 嵌入式 Agent
  
  // 5. 工具执行
  toolExecution: ToolRegistry;   // 工具注册与执行
}
```

### 2.3 WebSocket 协议设计

```typescript
// 协议帧类型
type Frame = 
  | { type: "req"; id: string; method: string; params: object }  // 请求
  | { type: "res"; id: string; ok: boolean; payload?: any; error?: any }  // 响应
  | { type: "event"; event: string; payload: any; seq?: number };  // 事件

// 连接生命周期
/*
 Client                    Gateway
   |                          |
   |---- req:connect -------->|
   |<------ res (ok) ---------|   (hello-ok 携带 presence + health 快照)
   |                          |
   |<------ event:presence ---|
   |<------ event:tick -------|
   |                          |
   |------- req:agent ------->|
   |<------ res:agent --------|   (ack: {runId, status:"accepted"})
   |<------ event:agent ------|   (streaming)
   |<------ res:agent --------|   (final: {runId, status, summary})
*/
```

### 2.4 节点 (Nodes) 架构

节点是连接到 Gateway 的设备，提供本地能力：

```typescript
interface Node {
  role: "node";
  deviceId: string;
  
  // 能力声明
  capabilities: {
    commands: string[];  // 支持的命令
    permissions: PermissionMap;  // TCC 权限状态
  };
  
  // 可用命令
  commands: {
    "canvas.*": CanvasCommands;
    "camera.*": CameraCommands;
    "screen.record": ScreenRecordCommand;
    "location.get": LocationCommand;
    "system.run": SystemRunCommand;  // macOS only
    "system.notify": SystemNotifyCommand;
  };
}
```

---

## 3. 消息处理流水线

### 3.1 入站消息流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    入站消息处理流水线                            │
└─────────────────────────────────────────────────────────────────┘

1. 渠道接收
   WhatsApp/Telegram/... ─→ 原始消息
                              │
2. 消息规范化               ▼
   ┌────────────────────────────────────────┐
   │  InboundMessage {                      │
   │    channel: "whatsapp",                │
   │    from: "+1234567890",                │
   │    body: "Hello",                      │
   │    media?: MediaAttachment[],          │
   │    replyTo?: QuotedMessage             │
   │  }                                     │
   └────────────────────────────────────────┘
                              │
3. 访问控制                 ▼
   ┌────────────────────────────────────────┐
   │  - DM 配对检查 (pairing)               │
   │  - 允许列表验证 (allowFrom)            │
   │  - 群组权限检查 (groups)               │
   └────────────────────────────────────────┘
                              │
4. 路由决策                 ▼
   ┌────────────────────────────────────────┐
   │  路由优先级:                           │
   │  1. 精确 peer 匹配 (bindings)          │
   │  2. Guild 匹配 (Discord)               │
   │  3. Team 匹配 (Slack)                  │
   │  4. Account 匹配                       │
   │  5. Channel 匹配                       │
   │  6. 默认 Agent                         │
   └────────────────────────────────────────┘
                              │
5. 会话解析                 ▼
   ┌────────────────────────────────────────┐
   │  SessionKey 生成规则:                  │
   │  - DM: agent:<agentId>:<mainKey>       │
   │  - 群组: agent:<agentId>:<ch>:group:<id>│
   │  - 频道: agent:<agentId>:<ch>:channel:<id>│
   └────────────────────────────────────────┘
                              │
6. 命令检测                 ▼
   ┌────────────────────────────────────────┐
   │  斜杠命令处理:                         │
   │  /status, /new, /reset, /model, ...    │
   │  → 直接执行，不调用 Agent              │
   └────────────────────────────────────────┘
                              │
7. Agent 调用               ▼
   ┌────────────────────────────────────────┐
   │  agentCommand() → runEmbeddedPiAgent() │
   └────────────────────────────────────────┘
```

### 3.2 出站消息流程

```typescript
// 出站路由是确定性的
interface OutboundRouting {
  // 消息路由回原始渠道
  routeReply(sessionKey: string, message: string): void {
    const origin = this.sessions.getOrigin(sessionKey);
    const channel = this.channels.get(origin.channel);
    
    // 分块处理 (长消息)
    const chunks = this.chunker.split(message, {
      maxChars: channel.textChunkLimit,
      breakPreference: "paragraph"
    });
    
    // 逐块发送
    for (const chunk of chunks) {
      await channel.send(origin.to, chunk);
    }
  }
}
```

### 3.3 流式输出与分块

```typescript
// 块流式传输 (Block Streaming)
interface BlockStreamingConfig {
  // 默认关闭，需要显式启用
  blockStreamingDefault: "on" | "off";
  
  // 触发时机
  blockStreamingBreak: "text_end" | "message_end";
  
  // 分块参数
  blockStreamingChunk: {
    minChars: number;   // 最小字符数
    maxChars: number;   // 最大字符数
    breakPreference: "paragraph" | "newline" | "sentence" | "whitespace";
  };
  
  // 合并策略
  blockStreamingCoalesce: {
    minChars: number;
    maxChars: number;
    idleMs: number;     // 空闲等待时间
  };
}
```

---

## 4. Agent 智能体系统

### 4.1 Agent Loop 核心流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent Loop 生命周期                          │
└─────────────────────────────────────────────────────────────────┘

1. 入口点
   ├─ Gateway RPC: agent / agent.wait
   └─ CLI: clawdbot agent

2. 预处理阶段
   ┌─────────────────────────────────────┐
   │  • 解析 session (sessionKey/Id)     │
   │  • 持久化 session 元数据             │
   │  • 返回 { runId, acceptedAt }       │
   └─────────────────────────────────────┘
              │
              ▼
3. agentCommand 执行
   ┌─────────────────────────────────────┐
   │  • 解析 model + thinking/verbose    │
   │  • 加载 skills 快照                  │
   │  • 调用 runEmbeddedPiAgent()        │
   │  • 发出 lifecycle end/error         │
   └─────────────────────────────────────┘
              │
              ▼
4. runEmbeddedPiAgent 运行
   ┌─────────────────────────────────────┐
   │  • 队列序列化 (per-session + global)│
   │  • 解析 model + auth profile         │
   │  • 构建 pi session                   │
   │  • 订阅 pi events                    │
   │  • 流式输出 assistant/tool deltas    │
   │  • 超时控制 → abort                  │
   └─────────────────────────────────────┘
              │
              ▼
5. 事件流
   ┌─────────────────────────────────────┐
   │  stream: "lifecycle"                 │
   │    └─ phase: "start" | "end" | "error"│
   │  stream: "assistant"                 │
   │    └─ text deltas                    │
   │  stream: "tool"                      │
   │    └─ tool start/update/end          │
   └─────────────────────────────────────┘
```

### 4.2 系统提示词构建

```typescript
// 系统提示词组装流程
interface SystemPromptBuilder {
  buildPrompt(ctx: AgentContext): string {
    const parts: string[] = [];
    
    // 1. 基础提示词
    parts.push(CLAWDBOT_BASE_PROMPT);
    
    // 2. 身份文件 (IDENTITY.md)
    if (ctx.workspace.hasFile("IDENTITY.md")) {
      parts.push(ctx.workspace.read("IDENTITY.md"));
    }
    
    // 3. Agent 文件 (AGENTS.md)
    if (ctx.workspace.hasFile("AGENTS.md")) {
      parts.push(ctx.workspace.read("AGENTS.md"));
    }
    
    // 4. 工具文件 (TOOLS.md)
    if (ctx.workspace.hasFile("TOOLS.md")) {
      parts.push(ctx.workspace.read("TOOLS.md"));
    }
    
    // 5. 灵魂文件 (SOUL.md)
    if (ctx.workspace.hasFile("SOUL.md")) {
      parts.push(ctx.workspace.read("SOUL.md"));
    }
    
    // 6. Skills 提示词
    const eligibleSkills = this.filterSkills(ctx);
    parts.push(formatSkillsForPrompt(eligibleSkills));
    
    // 7. Bootstrap 上下文
    parts.push(this.buildBootstrapContext(ctx));
    
    // 8. 日期时间
    parts.push(formatDateTime());
    
    return parts.join("\n\n");
  }
}
```

### 4.3 工具执行模型

```typescript
// 工具注册与执行
interface ToolRegistry {
  // 内置工具
  builtinTools: {
    bash: BashTool;              // Shell 执行
    read: ReadFileTool;          // 文件读取
    write: WriteFileTool;        // 文件写入
    edit: EditFileTool;          // 文件编辑
    browser: BrowserTool;        // 浏览器控制
    canvas: CanvasTool;          // Canvas 操作
    nodes: NodesTool;            // 节点控制
    cron: CronTool;              // 定时任务
    sessions_list: SessionsListTool;
    sessions_history: SessionsHistoryTool;
    sessions_send: SessionsSendTool;
    sessions_spawn: SessionsSpawnTool;
    memory_search: MemorySearchTool;
    memory_get: MemoryGetTool;
  };
  
  // 插件工具
  pluginTools: Map<string, PluginTool>;
  
  // 工具执行
  async execute(toolName: string, params: object): Promise<ToolResult> {
    const tool = this.resolve(toolName);
    
    // 权限检查
    if (!this.checkPermissions(tool, params)) {
      throw new PermissionDeniedError();
    }
    
    // 沙箱执行 (如果配置)
    if (this.shouldSandbox(tool)) {
      return this.executeSandboxed(tool, params);
    }
    
    return tool.execute(params);
  }
}
```

### 4.4 Model Failover 机制

```typescript
// 模型选择与故障转移
interface ModelFailover {
  // 选择顺序
  selectionOrder: [
    "agents.defaults.model.primary",      // 1. 主模型
    "agents.defaults.model.fallbacks[0]", // 2. 第一个备用
    "agents.defaults.model.fallbacks[1]", // 3. 第二个备用
    // ...
  ];
  
  // Auth Profile 轮换
  authRotation: {
    // 同一提供商内先轮换 auth profile
    // 再切换到下一个模型
    strategy: "rotate-auth-first";
    
    // 冷却机制
    cooldown: {
      profileFailureMs: 60000;  // profile 失败冷却
      modelFailureMs: 300000;   // model 失败冷却
    };
  };
  
  // 图像模型回退
  imageModelFallback: {
    // 当主模型不支持图像时使用
    primary: "agents.defaults.imageModel.primary";
    fallbacks: "agents.defaults.imageModel.fallbacks";
  };
}
```

---

## 5. 会话管理与内存系统

### 5.1 会话 (Session) 架构

```typescript
// 会话存储结构
interface SessionStore {
  // 存储位置
  path: "~/.clawdbot/agents/<agentId>/sessions/sessions.json";
  
  // 会话映射
  sessions: Map<SessionKey, SessionEntry>;
}

interface SessionEntry {
  sessionId: string;
  sessionKey: string;
  updatedAt: number;
  
  // Token 统计
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;
  contextTokens: number;
  
  // 来源元数据
  origin: {
    label: string;
    provider: string;
    from: string;
    to: string;
    accountId?: string;
    threadId?: string;
  };
  
  // 群组信息
  displayName?: string;
  channel?: string;
  subject?: string;
  room?: string;
  space?: string;
}
```

### 5.2 会话 Key 生成规则

```typescript
// SessionKey 生成策略
type DMScope = "main" | "per-peer" | "per-channel-peer";

function generateSessionKey(ctx: InboundContext): string {
  const { agentId, channel, chatType, peerId, groupId, threadId } = ctx;
  
  if (chatType === "direct") {
    switch (config.session.dmScope) {
      case "main":
        // 所有 DM 共享一个会话
        return `agent:${agentId}:${config.session.mainKey}`;
      case "per-peer":
        // 按发送者隔离
        return `agent:${agentId}:dm:${peerId}`;
      case "per-channel-peer":
        // 按渠道+发送者隔离
        return `agent:${agentId}:${channel}:dm:${peerId}`;
    }
  }
  
  if (chatType === "group") {
    let key = `agent:${agentId}:${channel}:group:${groupId}`;
    // Telegram 论坛主题
    if (threadId) {
      key += `:topic:${threadId}`;
    }
    return key;
  }
  
  // 频道/房间
  return `agent:${agentId}:${channel}:channel:${groupId}`;
}
```

### 5.3 会话重置策略

```typescript
// 会话重置配置
interface SessionResetConfig {
  reset: {
    mode: "daily" | "idle";
    atHour: number;        // daily 模式: 每天几点重置 (本地时间)
    idleMinutes: number;   // idle 模式: 空闲多久后重置
  };
  
  // 按类型覆盖
  resetByType: {
    dm: ResetPolicy;
    group: ResetPolicy;
    thread: ResetPolicy;
  };
  
  // 按渠道覆盖
  resetByChannel: {
    [channel: string]: ResetPolicy;
  };
  
  // 重置触发词
  resetTriggers: ["/new", "/reset"];
}
```

### 5.4 内存系统 (Memory)

#### 5.4.1 内存文件布局

```
<workspace>/
├── MEMORY.md              # 精选长期记忆 (仅主 DM 会话加载)
└── memory/
    ├── 2026-01-01.md      # 每日日志 (仅追加)
    ├── 2026-01-02.md
    └── ...
```

#### 5.4.2 自动内存刷新

```typescript
// 预压缩内存刷新
interface MemoryFlushConfig {
  compaction: {
    memoryFlush: {
      enabled: true;
      
      // 触发阈值: contextWindow - reserveTokensFloor - softThresholdTokens
      softThresholdTokens: 4000;
      
      // 系统提示词
      systemPrompt: "Session nearing compaction. Store durable memories now.";
      
      // 用户提示词
      prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store.";
    };
  };
}
```

#### 5.4.3 向量内存搜索 (RAG)

```typescript
// 内存搜索配置
interface MemorySearchConfig {
  provider: "openai" | "gemini" | "local";
  model: string;
  
  // 远程嵌入配置
  remote: {
    baseUrl?: string;
    apiKey?: string;
    headers?: Record<string, string>;
    batch: {
      enabled: true;
      concurrency: 2;
    };
  };
  
  // 本地嵌入配置
  local: {
    modelPath: "hf:ggml-org/embeddinggemma-300M-GGUF/...";
    modelCacheDir?: string;
  };
  
  // 混合搜索
  query: {
    hybrid: {
      enabled: true;
      vectorWeight: 0.7;
      textWeight: 0.3;
      candidateMultiplier: 4;
    };
  };
  
  // 索引存储
  store: {
    path: "~/.clawdbot/memory/<agentId>.sqlite";
    vector: {
      enabled: true;  // sqlite-vec 加速
    };
  };
}
```

### 5.5 压缩 (Compaction)

```typescript
// 上下文压缩机制
interface CompactionConfig {
  // 触发条件
  trigger: {
    // 当 session tokens 超过 contextWindow - reserveTokensFloor 时触发
    reserveTokensFloor: 20000;
  };
  
  // 压缩策略
  strategy: {
    // 保留最近的消息
    keepRecentMessages: number;
    
    // 压缩为摘要
    summaryPrompt: string;
  };
  
  // 压缩结果持久化到 JSONL
  persistence: {
    // 压缩摘要存入会话历史
    // 未来请求使用: 压缩摘要 + 最近消息
  };
}
```

---

## 6. 插件系统架构

### 6.1 插件发现与加载

```typescript
// 插件加载顺序 (优先级从高到低)
const pluginScanOrder = [
  // 1. 配置路径
  "plugins.load.paths",
  
  // 2. 工作区扩展
  "<workspace>/.clawdbot/extensions/*.ts",
  "<workspace>/.clawdbot/extensions/*/index.ts",
  
  // 3. 全局扩展
  "~/.clawdbot/extensions/*.ts",
  "~/.clawdbot/extensions/*/index.ts",
  
  // 4. 捆绑扩展 (默认禁用)
  "<clawdbot>/extensions/*"
];
```

### 6.2 插件 Manifest

```json
// clawdbot.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  
  // 配置 Schema
  "configSchema": {
    "type": "object",
    "properties": {
      "apiKey": { "type": "string" },
      "region": { "type": "string" }
    }
  },
  
  // UI 提示
  "uiHints": {
    "apiKey": { "label": "API Key", "sensitive": true },
    "region": { "label": "Region", "placeholder": "us-east-1" }
  }
}
```

### 6.3 插件 API

```typescript
// 插件注册 API
interface PluginAPI {
  // 注册 Gateway RPC 方法
  registerGatewayMethod(name: string, handler: MethodHandler): void;
  
  // 注册 HTTP 处理器
  registerHttpHandler(path: string, handler: HttpHandler): void;
  
  // 注册 Agent 工具
  registerTool(tool: ToolDefinition): void;
  
  // 注册 CLI 命令
  registerCli(setup: CliSetup, options: CliOptions): void;
  
  // 注册后台服务
  registerService(service: ServiceDefinition): void;
  
  // 注册消息渠道
  registerChannel(channel: ChannelPlugin): void;
  
  // 注册模型提供商
  registerProvider(provider: ProviderPlugin): void;
  
  // 注册自动回复命令
  registerCommand(command: CommandDefinition): void;
}
```

### 6.4 消息渠道插件示例

```typescript
// 最小渠道插件
const myChannel: ChannelPlugin = {
  id: "acmechat",
  
  meta: {
    id: "acmechat",
    label: "AcmeChat",
    selectionLabel: "AcmeChat (API)",
    docsPath: "/channels/acmechat",
    blurb: "AcmeChat messaging channel.",
    aliases: ["acme"]
  },
  
  capabilities: { 
    chatTypes: ["direct", "group"] 
  },
  
  config: {
    listAccountIds: (cfg) => Object.keys(cfg.channels?.acmechat?.accounts ?? {}),
    resolveAccount: (cfg, accountId) =>
      cfg.channels?.acmechat?.accounts?.[accountId ?? "default"]
  },
  
  outbound: {
    deliveryMode: "direct",
    sendText: async ({ to, text }) => {
      // 实现发送逻辑
      await acmeApi.send(to, text);
      return { ok: true };
    }
  },
  
  // 可选适配器
  gateway: {
    start: async (ctx) => { /* 启动连接 */ },
    stop: async () => { /* 停止连接 */ }
  }
};

export default function(api: PluginAPI) {
  api.registerChannel({ plugin: myChannel });
}
```

### 6.5 插件 Hooks

```typescript
// 插件 Hook 点
interface PluginHooks {
  // Agent 生命周期
  before_agent_start: (ctx: AgentContext) => void;
  agent_end: (ctx: AgentContext, result: AgentResult) => void;
  
  // 工具生命周期
  before_tool_call: (tool: string, params: object) => object;
  after_tool_call: (tool: string, result: ToolResult) => ToolResult;
  tool_result_persist: (result: ToolResult) => ToolResult;
  
  // 压缩生命周期
  before_compaction: (session: Session) => void;
  after_compaction: (session: Session, summary: string) => void;
  
  // 消息生命周期
  message_received: (msg: InboundMessage) => void;
  message_sending: (msg: OutboundMessage) => OutboundMessage;
  message_sent: (msg: OutboundMessage, result: SendResult) => void;
  
  // 会话生命周期
  session_start: (session: Session) => void;
  session_end: (session: Session) => void;
  
  // Gateway 生命周期
  gateway_start: () => void;
  gateway_stop: () => void;
}
```

---

## 7. 多渠道集成

### 7.1 渠道抽象层

```typescript
// 统一渠道接口
interface ChannelAdapter {
  // 元数据
  id: string;
  label: string;
  
  // 能力声明
  capabilities: {
    chatTypes: ("direct" | "group" | "channel")[];
    media: boolean;
    threads: boolean;
    reactions: boolean;
    streaming: boolean;
  };
  
  // 生命周期
  start(ctx: ChannelContext): Promise<void>;
  stop(): Promise<void>;
  
  // 出站
  sendText(to: string, text: string): Promise<SendResult>;
  sendMedia(to: string, media: MediaAttachment): Promise<SendResult>;
  
  // 可选能力
  typing?(to: string): Promise<void>;
  react?(messageId: string, emoji: string): Promise<void>;
}
```

### 7.2 内置渠道实现

| 渠道 | 协议/SDK | 关键文件 |
|------|----------|----------|
| WhatsApp | Baileys (Web 协议) | `src/web/` |
| Telegram | grammY (Bot API) | `src/telegram/` |
| Discord | discord.js | `src/discord/` |
| Slack | Bolt | `src/slack/` |
| Signal | signal-cli | `src/signal/` |
| iMessage | imsg CLI | `src/imessage/` |

### 7.3 渠道路由配置

```json5
// ~/.clawdbot/clawdbot.json
{
  channels: {
    whatsapp: {
      allowFrom: ["+1234567890"],
      groups: { "*": { requireMention: true } }
    },
    telegram: {
      botToken: "123456:ABC...",
      groups: { "*": { requireMention: true } }
    },
    discord: {
      token: "...",
      guilds: {
        "123456789": { enabled: true }
      }
    }
  },
  
  // 路由绑定
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "main" }
  ]
}
```

### 7.4 群组激活模式

```typescript
// 群组消息处理
interface GroupActivation {
  // 激活模式
  mode: "mention" | "always";
  
  // mention 模式: 需要 @ 触发
  mentionPatterns: ["@clawd", "@clawdbot"];
  
  // 可通过 /activation 切换
  // /activation mention
  // /activation always
}
```

---

## 8. 工具与技能系统

### 8.1 技能 (Skills) 架构

```
<workspace>/skills/
└── my-skill/
    ├── SKILL.md          # 技能定义 (必需)
    ├── handler.ts        # 可选处理器
    └── scripts/          # 可选脚本
```

### 8.2 SKILL.md 格式

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
metadata: {"clawdbot":{"requires":{"bins":["uv"],"env":["GEMINI_API_KEY"]},"primaryEnv":"GEMINI_API_KEY"}}
---

## 使用说明

当用户请求生成或编辑图像时，使用此技能...

## 命令

```bash
uv run --script {baseDir}/generate.py --prompt "..."
```
```

### 8.3 技能门控 (Gating)

```typescript
// 技能加载时过滤
interface SkillGating {
  // metadata.clawdbot 字段
  clawdbot: {
    // 总是包含
    always?: true;
    
    // OS 限制
    os?: ("darwin" | "linux" | "win32")[];
    
    // 依赖检查
    requires?: {
      bins?: string[];      // 全部需要存在
      anyBins?: string[];   // 至少一个存在
      env?: string[];       // 环境变量或配置
      config?: string[];    // 配置路径必须为真
    };
    
    // 主要环境变量
    primaryEnv?: string;
    
    // 安装器
    install?: InstallerSpec[];
  };
}
```

### 8.4 技能配置

```json5
// ~/.clawdbot/clawdbot.json
{
  skills: {
    entries: {
      "nano-banana-pro": {
        enabled: true,
        apiKey: "GEMINI_KEY_HERE",
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE"
        },
        config: {
          endpoint: "https://...",
          model: "nano-pro"
        }
      }
    },
    
    load: {
      watch: true,
      watchDebounceMs: 250,
      extraDirs: ["~/shared-skills"]
    },
    
    // 仅允许特定捆绑技能
    allowBundled: ["peekaboo", "camsnap"]
  }
}
```

### 8.5 内置工具列表

| 工具 | 功能 | 沙箱支持 |
|------|------|----------|
| `bash` | Shell 命令执行 | ✓ |
| `read` | 文件读取 | ✓ |
| `write` | 文件写入 | ✓ |
| `edit` | 文件编辑 | ✓ |
| `browser` | 浏览器控制 | ✗ |
| `canvas` | Canvas 操作 | ✗ |
| `nodes` | 节点控制 | ✗ |
| `cron` | 定时任务 | ✗ |
| `sessions_list` | 列出会话 | ✓ |
| `sessions_history` | 会话历史 | ✓ |
| `sessions_send` | 发送到会话 | ✓ |
| `sessions_spawn` | 创建子代理 | ✓ |
| `memory_search` | 内存搜索 | ✓ |
| `memory_get` | 获取内存 | ✓ |

---

## 9. RAG 与向量检索

### 9.1 RAG 系统概述

Clawdbot 的 RAG 系统基于**内存文件的向量索引**：

```
┌─────────────────────────────────────────────────────────────────┐
│                      RAG 流水线                                  │
└─────────────────────────────────────────────────────────────────┘

1. 索引阶段
   ┌─────────────────────────────────────┐
   │  MEMORY.md + memory/*.md            │
   │           │                         │
   │           ▼                         │
   │  Markdown 分块 (~400 tokens)        │
   │  80 tokens 重叠                     │
   │           │                         │
   │           ▼                         │
   │  嵌入生成 (OpenAI/Gemini/Local)     │
   │           │                         │
   │           ▼                         │
   │  SQLite 向量存储                    │
   │  (sqlite-vec 加速)                  │
   └─────────────────────────────────────┘

2. 检索阶段
   ┌─────────────────────────────────────┐
   │  用户查询                           │
   │           │                         │
   │           ▼                         │
   │  查询嵌入                           │
   │           │                         │
   │           ▼                         │
   │  混合搜索                           │
   │  ├─ 向量相似性 (70%)               │
   │  └─ BM25 关键词 (30%)              │
   │           │                         │
   │           ▼                         │
   │  排序 + Top-K 返回                  │
   └─────────────────────────────────────┘

3. 生成阶段
   ┌─────────────────────────────────────┐
   │  检索结果 + 用户问题                │
   │           │                         │
   │           ▼                         │
   │  构建增强提示词                     │
   │           │                         │
   │           ▼                         │
   │  LLM 生成回答                       │
   └─────────────────────────────────────┘
```

### 9.2 混合搜索算法

```typescript
// 混合搜索实现
interface HybridSearch {
  search(query: string, maxResults: number): SearchResult[] {
    const candidatePool = maxResults * candidateMultiplier;
    
    // 1. 向量检索
    const vectorResults = this.vectorSearch(query, candidatePool);
    
    // 2. BM25 检索
    const textResults = this.bm25Search(query, candidatePool);
    
    // 3. 分数转换
    const normalizedText = textResults.map(r => ({
      ...r,
      textScore: 1 / (1 + Math.max(0, r.bm25Rank))
    }));
    
    // 4. 按块 ID 合并
    const merged = this.unionByChunkId(vectorResults, normalizedText);
    
    // 5. 加权打分
    return merged.map(r => ({
      ...r,
      finalScore: vectorWeight * r.vectorScore + textWeight * r.textScore
    })).sort((a, b) => b.finalScore - a.finalScore)
      .slice(0, maxResults);
  }
}
```

### 9.3 嵌入缓存

```typescript
// 嵌入缓存策略
interface EmbeddingCache {
  // SQLite 中缓存块嵌入
  cache: {
    enabled: true;
    maxEntries: 50000;
  };
  
  // 缓存键: 块内容 hash + 模型 + 提供商
  cacheKey(chunk: Chunk): string {
    return hash(`${chunk.content}:${provider}:${model}`);
  }
  
  // 重新索引触发器
  reindexTriggers: [
    "provider change",
    "model change",
    "endpoint change",
    "chunking params change"
  ];
}
```

### 9.4 会话内存搜索 (实验性)

```json5
// 启用会话转录本索引
{
  agents: {
    defaults: {
      memorySearch: {
        experimental: { sessionMemory: true },
        sources: ["memory", "sessions"],
        sync: {
          sessions: {
            deltaBytes: 100000,   // ~100 KB
            deltaMessages: 50     // JSONL 行数
          }
        }
      }
    }
  }
}
```

---

## 10. Workflow 工作流设计

### 10.1 队列与并发控制

```typescript
// Agent 运行队列
interface QueueSystem {
  // 每会话队列
  sessionLane: Map<SessionKey, Queue>;
  
  // 全局队列
  globalLane: Queue;
  
  // 队列模式
  queueMode: "collect" | "steer" | "followup";
  
  // 序列化策略
  serialization: {
    // 同一会话内的运行串行执行
    perSession: true;
    
    // 可选的全局序列化
    global: false;
  };
}
```

### 10.2 Cron 定时任务

```typescript
// 定时任务配置
interface CronJob {
  id: string;
  schedule: string;  // cron 表达式
  
  // 任务类型
  type: "agent" | "webhook" | "script";
  
  // Agent 任务
  agent?: {
    message: string;
    agentId?: string;
    
    // 会话隔离
    sessionIsolation: true;  // 每次运行新 sessionId
  };
  
  // 输出路由
  output?: {
    channel: string;
    to: string;
  };
}
```

### 10.3 Webhook 触发

```json5
// Webhook 配置
{
  webhooks: {
    "my-hook": {
      enabled: true,
      secret: "...",  // HMAC 签名验证
      
      // 触发 Agent
      agent: {
        message: "Process webhook: {{body}}",
        sessionKey: "hook:my-hook"
      }
    }
  }
}
```

### 10.4 Gmail Pub/Sub 集成

```typescript
// Gmail 触发器
interface GmailTrigger {
  // Google Cloud Pub/Sub 订阅
  subscription: {
    projectId: string;
    topicName: string;
  };
  
  // 过滤规则
  filters: {
    from?: string[];
    subject?: RegExp;
    labels?: string[];
  };
  
  // 触发 Agent
  onMatch: {
    message: "New email from {{from}}: {{subject}}";
    includeBody: boolean;
  };
}
```

### 10.5 多代理协作

```typescript
// sessions_* 工具实现多代理协作
interface MultiAgentWorkflow {
  // 列出活跃会话
  sessions_list(): SessionInfo[];
  
  // 获取会话历史
  sessions_history(sessionKey: string): Message[];
  
  // 发送到另一个会话
  sessions_send(params: {
    sessionKey: string;
    message: string;
    replyBack?: boolean;  // 是否等待回复
    announceStep?: boolean;
  }): SendResult;
  
  // 创建子代理
  sessions_spawn(params: {
    agentId?: string;
    message: string;
    model?: string;
    waitForCompletion?: boolean;
  }): SpawnResult;
}
```

---

## 11. 源码目录深度解析

### 11.1 目录结构总览

```
src/
├── agents/              # Agent 核心实现
│   ├── pi-embedded-*.ts   # 嵌入式 Pi Agent
│   ├── auth-profiles*.ts  # 认证配置管理
│   ├── bash-tools*.ts     # Shell 工具
│   ├── model-*.ts         # 模型选择/故障转移
│   └── compaction.ts      # 上下文压缩
│
├── gateway/             # Gateway 服务器
│   ├── server.ts          # WebSocket 服务器
│   ├── protocol/          # 协议定义
│   ├── rpc/              # RPC 方法实现
│   └── health.ts         # 健康检查
│
├── channels/            # 渠道抽象层
│   ├── types.ts           # 通用类型
│   ├── routing.ts         # 路由逻辑
│   └── adapter.ts         # 适配器基类
│
├── telegram/            # Telegram 实现
├── discord/             # Discord 实现
├── slack/               # Slack 实现
├── signal/              # Signal 实现
├── imessage/            # iMessage 实现
├── web/                 # WhatsApp Web 实现
│
├── cli/                 # CLI 入口
│   ├── index.ts           # 主入口
│   └── commands/          # 命令实现
│
├── commands/            # 斜杠命令处理
│   ├── auto-reply/        # 自动回复命令
│   └── handlers/          # 命令处理器
│
├── config/              # 配置系统
│   ├── schema.ts          # JSON Schema
│   ├── loader.ts          # 配置加载
│   └── validation.ts      # 验证逻辑
│
├── plugins/             # 插件系统
│   ├── loader.ts          # 插件加载
│   ├── api.ts             # 插件 API
│   └── registry.ts        # 插件注册表
│
├── sessions/            # 会话管理
│   ├── store.ts           # 会话存储
│   └── manager.ts         # 会话管理器
│
├── memory/              # 内存系统
│   ├── indexer.ts         # 向量索引
│   ├── search.ts          # 搜索实现
│   └── tools.ts           # 内存工具
│
├── browser/             # 浏览器控制
├── canvas-host/         # Canvas 宿主
├── cron/                # 定时任务
├── hooks/               # Hook 系统
├── media/               # 媒体处理
├── security/            # 安全模块
└── utils/               # 工具函数
```

### 11.2 关键文件解析

#### `src/agents/pi-embedded-runner.ts`
```typescript
// Agent 运行的核心入口
export async function runEmbeddedPiAgent(params: RunParams): Promise<RunResult> {
  // 1. 队列序列化
  await acquireLock(params.sessionKey);
  
  // 2. 解析模型和认证
  const modelConfig = await resolveModel(params);
  const authProfile = await resolveAuthProfile(modelConfig);
  
  // 3. 构建 session
  const session = await buildPiSession(params, modelConfig);
  
  // 4. 订阅事件
  const subscription = subscribeEmbeddedPiSession(session, params.callbacks);
  
  // 5. 执行并返回
  try {
    const result = await session.run();
    return result;
  } finally {
    await releaseLock(params.sessionKey);
  }
}
```

#### `src/gateway/server.ts`
```typescript
// Gateway WebSocket 服务器
export class GatewayServer {
  private wss: WebSocketServer;
  private clients: Map<string, Client>;
  
  async start(config: GatewayConfig) {
    this.wss = new WebSocketServer({ port: config.port });
    
    this.wss.on('connection', (ws, req) => {
      this.handleConnection(ws, req);
    });
  }
  
  private handleConnection(ws: WebSocket, req: IncomingMessage) {
    // 1. 等待 connect 帧
    ws.once('message', async (data) => {
      const frame = JSON.parse(data.toString());
      
      if (frame.type !== 'req' || frame.method !== 'connect') {
        ws.close(4001, 'First frame must be connect');
        return;
      }
      
      // 2. 验证认证
      if (!this.validateAuth(frame.params)) {
        ws.close(4003, 'Unauthorized');
        return;
      }
      
      // 3. 注册客户端
      const client = this.registerClient(ws, frame.params);
      
      // 4. 发送 hello-ok
      this.sendResponse(client, frame.id, {
        presence: this.getPresence(),
        health: this.getHealth()
      });
    });
  }
}
```

#### `src/memory/search.ts`
```typescript
// 内存搜索实现
export class MemorySearchEngine {
  private indexer: MemoryIndexer;
  private db: Database;
  
  async search(query: string, options: SearchOptions): Promise<SearchResult[]> {
    // 1. 生成查询嵌入
    const queryEmbedding = await this.embed(query);
    
    // 2. 向量搜索
    const vectorResults = await this.vectorSearch(queryEmbedding, options);
    
    // 3. BM25 搜索 (如果启用混合)
    let textResults: TextSearchResult[] = [];
    if (options.hybrid?.enabled) {
      textResults = await this.bm25Search(query, options);
    }
    
    // 4. 合并结果
    return this.mergeResults(vectorResults, textResults, options);
  }
  
  private async vectorSearch(embedding: number[], options: SearchOptions) {
    // 使用 sqlite-vec 进行向量搜索
    return this.db.prepare(`
      SELECT chunk_id, content, file_path, line_start, line_end,
             vec_distance_cosine(embedding, ?) as distance
      FROM chunks
      ORDER BY distance
      LIMIT ?
    `).all(embedding, options.limit);
  }
}
```

---

## 12. 开发实践指南

### 12.1 开发环境设置

```bash
# 1. 克隆仓库
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

### 12.2 常用开发命令

```bash
# 类型检查 + 构建
pnpm build

# 运行测试
pnpm test
pnpm test:coverage

# 代码检查
pnpm lint
pnpm format

# 运行 CLI (开发模式)
pnpm clawdbot gateway --verbose
pnpm clawdbot agent --message "Hello"

# 实时测试 (需要 API 密钥)
CLAWDBOT_LIVE_TEST=1 pnpm test:live
```

### 12.3 添加新渠道

```typescript
// 1. 创建渠道目录
// src/my-channel/
// ├── index.ts
// ├── adapter.ts
// └── types.ts

// 2. 实现适配器
// src/my-channel/adapter.ts
export class MyChannelAdapter implements ChannelAdapter {
  id = "my-channel";
  label = "My Channel";
  
  capabilities = {
    chatTypes: ["direct", "group"],
    media: true,
    threads: false,
    reactions: false,
    streaming: false
  };
  
  async start(ctx: ChannelContext) {
    // 初始化连接
  }
  
  async stop() {
    // 关闭连接
  }
  
  async sendText(to: string, text: string) {
    // 发送消息
    return { ok: true, messageId: "..." };
  }
}

// 3. 注册渠道
// src/channels/registry.ts
import { MyChannelAdapter } from "../my-channel";

registerChannel(new MyChannelAdapter());
```

### 12.4 添加新工具

```typescript
// 1. 定义工具 Schema
const myToolSchema = {
  name: "my_tool",
  description: "My custom tool",
  parameters: {
    type: "object",
    properties: {
      input: { type: "string", description: "Input value" }
    },
    required: ["input"]
  }
};

// 2. 实现工具
async function myToolHandler(params: { input: string }): Promise<ToolResult> {
  // 工具逻辑
  const result = await doSomething(params.input);
  
  return {
    ok: true,
    output: result
  };
}

// 3. 注册工具
toolRegistry.register({
  schema: myToolSchema,
  handler: myToolHandler,
  permissions: {
    sandbox: true,  // 是否支持沙箱执行
    elevated: false // 是否需要提权
  }
});
```

### 12.5 添加新技能

```markdown
<!-- skills/my-skill/SKILL.md -->
---
name: my-skill
description: My custom skill for doing X
metadata: {"clawdbot":{"requires":{"bins":["my-cli"],"env":["MY_API_KEY"]},"primaryEnv":"MY_API_KEY"}}
---

## 使用方法

当用户请求做 X 时，使用以下命令:

```bash
my-cli --input "{用户输入}" --api-key "$MY_API_KEY"
```

## 示例

用户: "帮我做 X"
助手: 我来使用 my-skill 帮你完成...
```

### 12.6 测试最佳实践

```typescript
// 1. 单元测试
import { describe, it, expect, vi } from 'vitest';
import { myFunction } from './my-module';

describe('myFunction', () => {
  it('should handle normal input', () => {
    expect(myFunction('input')).toBe('expected');
  });
  
  it('should handle edge cases', () => {
    expect(myFunction('')).toBe('');
    expect(myFunction(null)).toBeNull();
  });
});

// 2. 集成测试
describe('MyChannel integration', () => {
  it('should send and receive messages', async () => {
    const adapter = new MyChannelAdapter();
    await adapter.start(mockContext);
    
    const result = await adapter.sendText('user123', 'Hello');
    expect(result.ok).toBe(true);
    
    await adapter.stop();
  });
});

// 3. E2E 测试
describe('E2E: Agent workflow', () => {
  it('should complete full agent loop', async () => {
    const gateway = await startTestGateway();
    const client = await connectTestClient(gateway);
    
    const result = await client.call('agent', {
      message: 'Test message',
      sessionKey: 'test:session'
    });
    
    expect(result.status).toBe('completed');
  });
});
```

---

## 13. 扩展与定制

### 13.1 自定义 Agent 身份

```markdown
<!-- ~/clawd/IDENTITY.md -->
# 我的 AI 助手

你是一个专业的技术助手，专注于:
- 软件开发
- 系统架构
- 代码审查

## 性格特点
- 简洁直接
- 注重实践
- 善于举例

## 回复风格
- 使用代码块展示代码
- 分步骤解释复杂问题
- 主动提供最佳实践建议
```

### 13.2 自定义工具集

```markdown
<!-- ~/clawd/TOOLS.md -->
# 可用工具

## 项目特定工具

### deploy
部署到生产环境
```bash
./scripts/deploy.sh <env>
```

### test-api
测试 API 端点
```bash
curl -X POST $API_URL/endpoint -d '...'
```

## 注意事项
- 部署前必须运行测试
- 敏感操作需要确认
```

### 13.3 自定义消息处理

```typescript
// 插件方式自定义消息处理
export default function(api: PluginAPI) {
  // 入站消息 Hook
  api.registerHook('message_received', (msg) => {
    // 记录所有消息
    logger.info('Received:', msg.body);
    
    // 可以修改消息
    if (msg.body.includes('敏感词')) {
      msg.body = msg.body.replace('敏感词', '***');
    }
  });
  
  // 出站消息 Hook
  api.registerHook('message_sending', (msg) => {
    // 添加签名
    msg.text += '\n\n— AI Assistant';
    return msg;
  });
}
```

### 13.4 自定义模型提供商

```typescript
// 注册自定义模型提供商
api.registerProvider({
  id: "my-provider",
  label: "My AI Provider",
  
  auth: [
    {
      id: "api-key",
      label: "API Key",
      kind: "api-key",
      run: async (ctx) => {
        const apiKey = await ctx.prompter.text('Enter API Key:');
        
        return {
          profiles: [{
            profileId: "my-provider:default",
            credential: {
              type: "api-key",
              provider: "my-provider",
              apiKey
            }
          }]
        };
      }
    }
  ],
  
  // 模型配置
  models: {
    "my-model-v1": {
      contextWindow: 128000,
      supportsImages: true,
      supportsTools: true
    }
  }
});
```

---

## 14. 部署与运维

### 14.1 部署架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     生产部署架构                                 │
└─────────────────────────────────────────────────────────────────┘

                    ┌─────────────┐
                    │   Internet  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Tailscale  │
                    │  Funnel     │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐    │    ┌───────▼───────┐
       │ Gateway Host │    │    │  Device Nodes │
       │   (Linux)    │    │    │(macOS/iOS/...)│
       │              │    │    │               │
       │ - Gateway    │◄───┼───►│ - Canvas      │
       │ - Channels   │    │    │ - Camera      │
       │ - Agent      │    │    │ - Location    │
       │ - Storage    │    │    │ - System.run  │
       └──────────────┘    │    └───────────────┘
                           │
                    ┌──────▼──────┐
                    │   Clients   │
                    │(WebChat/CLI)│
                    └─────────────┘
```

### 14.2 安装方式

```bash
# 方式 1: npm 全局安装 (推荐)
npm install -g clawdbot@latest
clawdbot onboard --install-daemon

# 方式 2: Docker
docker run -d \
  -v ~/.clawdbot:/root/.clawdbot \
  -p 18789:18789 \
  clawdbot/clawdbot:latest

# 方式 3: 从源码
git clone https://github.com/clawdbot/clawdbot.git
cd clawdbot
pnpm install && pnpm build
clawdbot onboard --install-daemon
```

### 14.3 远程访问配置

```json5
// Tailscale Serve/Funnel 配置
{
  gateway: {
    bind: "loopback",  // 保持绑定 loopback
    tailscale: {
      mode: "funnel",  // serve (仅 tailnet) 或 funnel (公网)
      resetOnExit: true
    },
    auth: {
      mode: "password",  // funnel 必须设置密码
      password: "..."
    }
  }
}
```

### 14.4 监控与日志

```bash
# 查看 Gateway 日志
clawdbot logs -f

# 健康检查
clawdbot health

# 状态概览
clawdbot status --all

# 深度状态 (包含探测)
clawdbot status --deep

# macOS 统一日志
./scripts/clawlog.sh -f  # follow 模式
```

### 14.5 故障排除

```bash
# 运行诊断
clawdbot doctor

# 自动修复
clawdbot doctor --fix

# 检查渠道状态
clawdbot channels status --probe

# 验证配置
clawdbot config validate

# 重置 WhatsApp 会话
clawdbot channels logout whatsapp
clawdbot channels login whatsapp
```

### 14.6 备份与恢复

```bash
# 备份关键数据
tar -czf clawdbot-backup.tar.gz \
  ~/.clawdbot/clawdbot.json \
  ~/.clawdbot/credentials/ \
  ~/.clawdbot/agents/ \
  ~/.clawdbot/sessions/

# 恢复
tar -xzf clawdbot-backup.tar.gz -C ~/
```

---

## 附录

### A. 配置参考速查

```json5
// ~/.clawdbot/clawdbot.json 完整示例
{
  // Gateway 配置
  gateway: {
    port: 18789,
    bind: "loopback"
  },
  
  // Agent 配置
  agents: {
    defaults: {
      workspace: "~/clawd",
      model: {
        primary: "anthropic/claude-opus-4-5",
        fallbacks: ["openai/gpt-5.2"]
      },
      sandbox: { mode: "non-main" }
    }
  },
  
  // 渠道配置
  channels: {
    whatsapp: { allowFrom: ["+1234567890"] },
    telegram: { botToken: "..." }
  },
  
  // 会话配置
  session: {
    dmScope: "main",
    reset: { mode: "daily", atHour: 4 }
  },
  
  // 技能配置
  skills: {
    entries: { peekaboo: { enabled: true } }
  },
  
  // 插件配置
  plugins: {
    entries: { "voice-call": { enabled: true } }
  }
}
```

### B. 常用 CLI 命令

| 命令 | 说明 |
|------|------|
| `clawdbot onboard` | 引导设置 |
| `clawdbot gateway` | 启动 Gateway |
| `clawdbot agent -m "..."` | 发送消息给 Agent |
| `clawdbot channels status` | 渠道状态 |
| `clawdbot models status` | 模型状态 |
| `clawdbot sessions list` | 会话列表 |
| `clawdbot plugins list` | 插件列表 |
| `clawdbot doctor` | 诊断问题 |

### C. 斜杠命令

| 命令 | 说明 |
|------|------|
| `/status` | 会话状态 |
| `/new` | 重置会话 |
| `/model` | 切换模型 |
| `/think <level>` | 设置思考级别 |
| `/verbose on/off` | 详细模式 |
| `/compact` | 手动压缩 |
| `/stop` | 中止当前运行 |

### D. 环境变量

| 变量 | 说明 |
|------|------|
| `CLAWDBOT_CONFIG_PATH` | 配置文件路径 |
| `CLAWDBOT_STATE_DIR` | 状态目录 |
| `CLAWDBOT_GATEWAY_TOKEN` | Gateway 认证令牌 |
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 |
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot 令牌 |
| `DISCORD_BOT_TOKEN` | Discord Bot 令牌 |

---

## 结语

Clawdbot 是一个功能强大且设计精良的个人 AI 助手平台。通过本教程，你应该能够：

1. **理解核心架构**：Gateway 作为控制平面，统一管理所有渠道和会话
2. **掌握消息流程**：从入站路由到出站分发的完整流水线
3. **深入 Agent 系统**：了解 Agent Loop、工具执行和模型管理
4. **运用内存系统**：包括会话管理、向量搜索和上下文压缩
5. **扩展平台能力**：通过插件系统添加新渠道、工具和功能
6. **进行开发实践**：搭建环境、编写测试、部署运维

希望这份教程能帮助你全面掌握 Clawdbot，并在此基础上进行二次开发或教学分享！

---

*文档版本: 2026.01.26*
*基于 Clawdbot 源码分析*
