---
summary: "一步一步教会你 Clawdbot 项目的完整学习路径"
read_when:
  - 想要深入学习 Clawdbot 项目
  - 准备贡献代码或扩展功能
  - 理解项目架构和实现细节
---

# Clawdbot 学习路径 🦞

> 一步一步教会你 Clawdbot 项目的完整指南

## 目录

1. [项目概述](#1-项目概述)
2. [环境准备](#2-环境准备)
3. [核心概念理解](#3-核心概念理解)
4. [代码结构探索](#4-代码结构探索)
5. [实践任务](#5-实践任务)
6. [高级主题](#6-高级主题)
7. [测试与调试](#7-测试与调试)
8. [贡献指南](#8-贡献指南)

---

## 1. 项目概述

### 1.1 什么是 Clawdbot？

Clawdbot 是一个**个人 AI 助手**，你可以运行在自己的设备上。它能够：
- 连接多个消息平台（WhatsApp、Telegram、Discord、Slack、iMessage 等）
- 使用 AI 代理（Pi）处理对话和任务
- 提供丰富的工具集（浏览器、Canvas、节点等）
- 支持多平台应用（macOS、iOS、Android）

### 1.2 核心价值

- **本地优先**：数据和控制权在你手中
- **多通道统一**：一个助手，多个消息平台
- **可扩展**：插件系统和技能平台
- **开发者友好**：TypeScript、清晰的架构、完善的文档

### 1.3 技术栈

- **运行时**：Node.js ≥22
- **语言**：TypeScript (ESM)
- **AI 代理**：Pi (p-mono)
- **通信协议**：WebSocket
- **移动端**：Swift (iOS/macOS), Kotlin (Android)
- **Web UI**：Lit + Hono

### 1.4 架构概览

```
消息平台 (WhatsApp/Telegram/Discord...)
        │
        ▼
  ┌───────────────────────────┐
  │        Gateway            │  WebSocket 控制平面
  │    (单一控制中心)          │  ws://127.0.0.1:18789
  └───────────┬───────────────┘
              │
              ├─ Pi Agent (RPC)
              ├─ CLI (clawdbot ...)
              ├─ Web UI (Control UI)
              ├─ macOS App
              └─ iOS/Android Nodes
```

**关键原则**：
- **一个 Gateway 进程**：管理所有通道连接
- **WebSocket 协议**：所有客户端通过 WS 连接
- **会话隔离**：每个对话有独立的会话状态
- **工具系统**：代理可以调用各种工具

---

## 2. 环境准备

### 2.1 基础要求

**必需**：
- Node.js ≥22.12.0
- Git
- 代码编辑器（推荐 VS Code）

**可选**（用于移动端开发）：
- Xcode (macOS/iOS)
- Android Studio (Android)
- pnpm（推荐用于开发）

### 2.2 安装项目

```bash
# 1. 克隆仓库
git clone https://github.com/clawdbot/clawdbot.git
cd clawdbot

# 2. 安装依赖
pnpm install

# 3. 构建 UI
pnpm ui:build  # 首次运行会自动安装 UI 依赖

# 4. 编译 TypeScript
pnpm build

# 5. 运行向导（设置基础配置）
pnpm clawdbot onboard --install-daemon
```

### 2.3 开发工具设置

**VS Code 推荐插件**：
- TypeScript and JavaScript Language Features
- ESLint / Oxlint
- Prettier / Oxfmt

**环境变量**（可选）：
```bash
# .env 文件（项目根目录）
CLAWDBOT_PROFILE=dev
CLAWDBOT_SKIP_CHANNELS=1  # 开发时跳过通道连接
```

### 2.4 验证安装

```bash
# 检查 CLI 是否工作
pnpm clawdbot --version

# 检查健康状态
pnpm clawdbot health

# 查看状态
pnpm clawdbot status
```

---

## 3. 核心概念理解

### 3.1 Gateway（网关）

**什么是 Gateway？**
- 长期运行的后台进程
- 管理所有消息通道连接
- 提供 WebSocket API 供客户端连接
- 是系统的"控制平面"

**关键文件**：
- `src/gateway/server.ts` - Gateway 服务器入口
- `src/gateway/server.impl.ts` - 服务器实现
- `docs/gateway/` - Gateway 文档

**学习任务**：
1. 阅读 `docs/concepts/architecture.md`
2. 查看 `src/gateway/server.impl.ts` 了解启动流程
3. 运行 `pnpm gateway:watch` 观察 Gateway 启动

**实践**：
```bash
# 启动 Gateway（开发模式）
pnpm gateway:watch

# 在另一个终端查看状态
pnpm clawdbot status

# 查看 Gateway 日志
pnpm clawdbot logs
```

### 3.2 Session（会话）

**什么是 Session？**
- 每个对话都有独立的会话
- 存储对话历史和上下文
- 支持会话隔离和重置策略

**会话键（Session Key）格式**：
- 直接消息：`agent:<agentId>:<mainKey>`（默认 `main`）
- 群组：`agent:<agentId>:<channel>:group:<id>`
- Cron：`cron:<job.id>`

**关键文件**：
- `src/sessions/` - 会话管理
- `docs/concepts/session.md` - 会话文档
- `docs/concepts/session-pruning.md` - 会话修剪

**学习任务**：
1. 阅读 `docs/concepts/session.md`
2. 查看 `src/sessions/session.ts` 了解会话存储
3. 理解会话重置策略（每日重置、空闲重置）

**实践**：
```bash
# 列出所有会话
pnpm clawdbot sessions

# 查看特定会话详情
pnpm clawdbot sessions --json | jq '.["agent:main:main"]'

# 发送 /status 命令查看会话状态
```

### 3.3 Agent（代理）

**什么是 Agent？**
- AI 代理运行时（基于 Pi/p-mono）
- 处理用户消息并生成回复
- 可以调用工具执行任务
- 维护工作空间和上下文

**工作空间（Workspace）**：
- 默认位置：`~/clawd`
- 包含引导文件：`AGENTS.md`, `SOUL.md`, `TOOLS.md`
- 代理的所有文件操作都在此目录

**关键文件**：
- `src/agents/` - 代理运行时
- `docs/concepts/agent.md` - 代理文档
- `docs/concepts/agent-workspace.md` - 工作空间文档

**学习任务**：
1. 阅读 `docs/concepts/agent.md`
2. 查看 `src/agents/agent.ts` 了解代理启动
3. 理解工作空间文件的作用

**实践**：
```bash
# 发送消息给代理
pnpm clawdbot agent --message "Hello, what can you do?"

# 查看工作空间
ls ~/clawd

# 编辑 AGENTS.md 自定义代理行为
```

### 3.4 Channel（通道）

**什么是 Channel？**
- 消息平台的集成（WhatsApp、Telegram、Discord 等）
- 处理消息收发
- 管理连接状态

**支持的通道**：
- **核心通道**：WhatsApp、Telegram、Discord、Slack、Signal、iMessage
- **扩展通道**：Microsoft Teams、Matrix、Zalo、BlueBubbles（插件）

**关键文件**：
- `src/channels/` - 通道抽象层
- `src/telegram/` - Telegram 实现
- `src/discord/` - Discord 实现
- `src/whatsapp/` - WhatsApp 实现
- `docs/channels/` - 通道文档

**学习任务**：
1. 阅读 `docs/channels/telegram.md` 了解一个通道的实现
2. 查看 `src/channels/channel.ts` 了解通道接口
3. 理解通道路由机制

**实践**：
```bash
# 登录 WhatsApp（显示 QR 码）
pnpm clawdbot channels login

# 查看通道状态
pnpm clawdbot channels status

# 配置通道允许列表
pnpm clawdbot config set channels.telegram.allowFrom '["+1234567890"]'
```

### 3.5 Tools（工具）

**什么是 Tools？**
- 代理可以调用的功能
- 包括：浏览器、文件操作、Canvas、节点等

**内置工具**：
- `bash` - 执行 shell 命令
- `read` / `write` / `edit` - 文件操作
- `browser` - 浏览器控制
- `canvas` - Canvas 操作
- `nodes` - 节点设备控制

**关键文件**：
- `src/agents/tools/` - 工具实现
- `docs/tools/` - 工具文档

**学习任务**：
1. 阅读 `docs/tools/exec.md` 了解执行工具
2. 查看 `src/agents/tools/bash.ts` 了解工具实现
3. 理解工具策略和权限控制

---

## 4. 代码结构探索

### 4.1 目录结构

```
clawdbot/
├── src/                    # TypeScript 源代码
│   ├── cli/               # CLI 命令
│   ├── gateway/           # Gateway 服务器
│   ├── agents/            # Agent 运行时
│   ├── channels/          # 通道抽象层
│   ├── telegram/          # Telegram 实现
│   ├── discord/           # Discord 实现
│   ├── whatsapp/          # WhatsApp 实现
│   ├── commands/          # CLI 命令实现
│   ├── web/               # Web UI
│   └── ...
├── apps/                   # 移动端应用
│   ├── macos/            # macOS 应用
│   ├── ios/              # iOS 应用
│   └── android/          # Android 应用
├── extensions/            # 扩展插件
├── docs/                  # 文档
├── skills/                # 技能
└── scripts/               # 构建脚本
```

### 4.2 关键入口点

**CLI 入口**：
- `src/entry.ts` - CLI 入口脚本
- `src/cli/run-main.ts` - CLI 主逻辑
- `src/cli/program.ts` - 命令定义

**Gateway 入口**：
- `src/gateway/server.ts` - Gateway 导出
- `src/gateway/server.impl.ts` - 服务器实现
- `src/commands/gateway.ts` - Gateway 命令

**Agent 入口**：
- `src/agents/agent.ts` - Agent 运行时
- `src/agents/rpc.ts` - RPC 适配器

### 4.3 代码阅读顺序

**第一阶段：理解 CLI**
1. `src/entry.ts` - 入口点
2. `src/cli/run-main.ts` - CLI 启动
3. `src/cli/program.ts` - 命令注册
4. `src/commands/agent.ts` - Agent 命令示例

**第二阶段：理解 Gateway**
1. `src/gateway/server.impl.ts` - Gateway 启动
2. `src/gateway/protocol.ts` - WebSocket 协议
3. `src/gateway/handlers/` - 请求处理

**第三阶段：理解 Agent**
1. `src/agents/agent.ts` - Agent 运行时
2. `src/agents/rpc.ts` - RPC 通信
3. `src/agents/tools/` - 工具实现

**第四阶段：理解 Channel**
1. `src/channels/channel.ts` - 通道接口
2. `src/telegram/telegram.ts` - Telegram 实现示例
3. `src/routing/` - 消息路由

### 4.4 代码风格

**TypeScript 规范**：
- 使用 ESM（`import`/`export`）
- 严格类型检查
- 避免 `any` 类型
- 文件长度建议 <700 LOC

**格式化**：
```bash
# 检查格式
pnpm format

# 自动修复
pnpm format:fix

# 检查 lint
pnpm lint

# 自动修复 lint
pnpm lint:fix
```

---

## 5. 实践任务

### 5.1 任务 1：添加一个简单的 CLI 命令

**目标**：创建一个 `clawdbot hello` 命令

**步骤**：
1. 在 `src/commands/` 创建 `hello.ts`
2. 在 `src/cli/program.ts` 注册命令
3. 测试命令

**代码示例**：
```typescript
// src/commands/hello.ts
import { Command } from "commander";

export function registerHelloCommand(program: Command) {
  program
    .command("hello")
    .description("Say hello")
    .option("--name <name>", "Name to greet", "World")
    .action((options) => {
      console.log(`Hello, ${options.name}!`);
    });
}
```

**测试**：
```bash
pnpm build
pnpm clawdbot hello --name "Clawdbot"
```

### 5.2 任务 2：理解消息流程

**目标**：追踪一条消息从接收到回复的完整流程

**步骤**：
1. 选择一个通道（如 Telegram）
2. 找到消息接收处理代码
3. 追踪到 Agent 调用
4. 追踪到回复发送

**关键文件**：
- `src/telegram/telegram.ts` - 消息接收
- `src/channels/channel.ts` - 通道处理
- `src/gateway/handlers/agent.ts` - Agent 调用
- `src/agents/agent.ts` - Agent 处理

**实践**：
1. 设置断点或添加日志
2. 发送一条测试消息
3. 观察日志输出

### 5.3 任务 3：添加一个简单的工具

**目标**：创建一个 `echo` 工具，回显输入

**步骤**：
1. 在 `src/agents/tools/` 创建 `echo.ts`
2. 注册工具
3. 测试工具调用

**代码示例**：
```typescript
// src/agents/tools/echo.ts
import { Tool } from "@mariozechner/pi-agent-core";

export const echoTool: Tool = {
  name: "echo",
  description: "Echo the input text",
  inputSchema: {
    type: "object",
    properties: {
      text: {
        type: "string",
        description: "Text to echo",
      },
    },
    required: ["text"],
  },
  handler: async ({ text }) => {
    return { output: text };
  },
};
```

### 5.4 任务 4：理解配置系统

**目标**：了解配置如何加载和应用

**步骤**：
1. 查看 `src/config/` 目录
2. 理解配置加载流程
3. 理解配置验证

**关键文件**：
- `src/config/config.ts` - 配置加载
- `docs/gateway/configuration.md` - 配置文档

**实践**：
```bash
# 查看当前配置
pnpm clawdbot config get

# 设置配置项
pnpm clawdbot config set agent.model "anthropic/claude-opus-4-5"

# 验证配置
pnpm clawdbot doctor
```

### 5.5 任务 5：理解 WebSocket 协议

**目标**：了解 Gateway WebSocket API

**步骤**：
1. 阅读 `docs/gateway/protocol.md`
2. 查看 `src/gateway/protocol.ts`
3. 使用 WebSocket 客户端连接测试

**实践**：
```bash
# 启动 Gateway
pnpm gateway:watch

# 使用 wscat 连接（需要安装：npm install -g wscat）
wscat -c ws://127.0.0.1:18789

# 发送连接消息
{"type":"req","id":"1","method":"connect","params":{}}
```

---

## 6. 高级主题

### 6.1 多代理路由

**概念**：将不同的消息路由到不同的代理

**学习资源**：
- `docs/concepts/multi-agent.md`
- `src/routing/` - 路由实现

**实践**：
```json5
// ~/.clawdbot/clawdbot.json
{
  routing: {
    agents: {
      main: { workspace: "~/clawd" },
      work: { workspace: "~/clawd-work" }
    },
    rules: [
      { agent: "work", match: { channel: "slack", accountId: "work-account" } }
    ]
  }
}
```

### 6.2 沙箱和安全

**概念**：隔离非主会话的执行环境

**学习资源**：
- `docs/gateway/security.md`
- `docs/gateway/sandboxing.md`
- `src/security/` - 安全实现

**实践**：
```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",  // 非主会话使用沙箱
        workspaceRoot: "~/.clawdbot/sandboxes"
      }
    }
  }
}
```

### 6.3 技能系统

**概念**：可安装的技能扩展代理能力

**学习资源**：
- `docs/tools/skills.md`
- `docs/tools/creating-skills.md`
- `skills/` - 内置技能

**实践**：
```bash
# 列出可用技能
pnpm clawdbot skills list

# 安装技能
pnpm clawdbot skills install <skill-name>

# 创建自定义技能
# 参考 skills/ 目录中的示例
```

### 6.4 插件系统

**概念**：扩展通道和功能

**学习资源**：
- `docs/plugin.md`
- `docs/refactor/plugin-sdk.md`
- `extensions/` - 扩展示例

**实践**：
查看 `extensions/msteams/` 了解如何创建通道扩展

### 6.5 移动端开发

**概念**：iOS/Android 节点应用

**学习资源**：
- `docs/platforms/ios.md`
- `docs/platforms/android.md`
- `apps/ios/` - iOS 代码
- `apps/android/` - Android 代码

**实践**：
```bash
# iOS
cd apps/ios
xcodegen generate
open Clawdbot.xcodeproj

# Android
cd apps/android
./gradlew :app:assembleDebug
```

---

## 7. 测试与调试

### 7.1 运行测试

```bash
# 运行所有测试
pnpm test

# 运行特定测试文件
pnpm test src/commands/agent.test.ts

# 监视模式
pnpm test:watch

# 覆盖率
pnpm test:coverage

# E2E 测试
pnpm test:e2e

# 实时测试（需要真实密钥）
CLAWDBOT_LIVE_TEST=1 pnpm test:live
```

### 7.2 调试技巧

**CLI 调试**：
```bash
# 使用 Node 调试器
node --inspect dist/entry.js gateway

# 使用 VS Code 调试配置
# 创建 .vscode/launch.json
```

**Gateway 调试**：
```bash
# 启用详细日志
pnpm gateway:watch --verbose

# 查看日志
pnpm clawdbot logs

# 使用 doctor 诊断
pnpm clawdbot doctor
```

**Agent 调试**：
```bash
# 启用详细模式
pnpm clawdbot agent --message "test" --verbose high

# 查看会话日志
cat ~/.clawdbot/agents/main/sessions/*.jsonl | jq
```

### 7.3 常见问题排查

**Gateway 无法启动**：
1. 检查端口是否被占用：`lsof -i :18789`
2. 检查配置：`pnpm clawdbot doctor`
3. 查看日志：`pnpm clawdbot logs`

**通道连接失败**：
1. 检查凭证：`pnpm clawdbot channels status`
2. 重新登录：`pnpm clawdbot channels login`
3. 查看通道特定文档

**Agent 无响应**：
1. 检查模型配置：`pnpm clawdbot config get | grep model`
2. 检查认证：`pnpm clawdbot health`
3. 查看 Agent 日志

---

## 8. 贡献指南

### 8.1 贡献流程

1. **Fork 仓库**
2. **创建分支**：`git checkout -b feature/my-feature`
3. **编写代码**：遵循代码风格
4. **运行测试**：`pnpm lint && pnpm build && pnpm test`
5. **提交 PR**：描述清楚改动和原因

### 8.2 代码规范

**提交信息**：
- 使用清晰的描述
- 遵循 Conventional Commits（可选）

**代码审查清单**：
- [ ] 代码通过 lint 检查
- [ ] 代码通过类型检查
- [ ] 添加了测试（如适用）
- [ ] 更新了文档（如适用）
- [ ] 遵循项目代码风格

### 8.3 文档贡献

**文档位置**：
- `docs/` - 用户文档
- `README.md` - 项目说明
- `CONTRIBUTING.md` - 贡献指南

**文档格式**：
- 使用 Markdown
- 遵循现有文档结构
- 添加适当的链接和示例

### 8.4 获取帮助

**资源**：
- GitHub Issues：报告问题和建议
- Discord：实时讨论
- 文档：`docs/` 目录

**提问技巧**：
1. 提供清晰的描述
2. 包含错误日志
3. 说明你的环境
4. 展示你尝试过的解决方案

---

## 学习检查清单

### 基础理解
- [ ] 理解 Gateway 的作用和架构
- [ ] 理解 Session 的概念和管理
- [ ] 理解 Agent 运行时和工作空间
- [ ] 理解 Channel 的集成方式
- [ ] 理解 Tools 系统

### 实践能力
- [ ] 能够设置开发环境
- [ ] 能够运行和调试 Gateway
- [ ] 能够添加简单的 CLI 命令
- [ ] 能够理解消息流程
- [ ] 能够配置和测试通道

### 高级能力
- [ ] 理解多代理路由
- [ ] 理解沙箱和安全机制
- [ ] 能够创建自定义技能
- [ ] 能够调试和排查问题
- [ ] 能够贡献代码

---

## 下一步

完成基础学习后，你可以：

1. **深入特定领域**：
   - 选择一个感兴趣的通道深入研究
   - 研究移动端应用开发
   - 探索技能系统

2. **参与项目**：
   - 修复简单的 bug
   - 添加新功能
   - 改进文档

3. **构建自己的扩展**：
   - 创建自定义技能
   - 开发通道扩展
   - 构建工具集成

---

## 推荐阅读顺序

### 第一周：基础理解
1. 阅读 `README.md` 和 `docs/index.md`
2. 阅读 `docs/concepts/architecture.md`
3. 阅读 `docs/start/getting-started.md`
4. 设置开发环境并运行示例

### 第二周：核心概念
1. 深入理解 Gateway：`docs/gateway/`
2. 理解 Session：`docs/concepts/session.md`
3. 理解 Agent：`docs/concepts/agent.md`
4. 理解 Channel：`docs/channels/`

### 第三周：实践开发
1. 完成实践任务 1-5
2. 阅读代码并理解实现
3. 尝试修改和扩展功能

### 第四周：高级主题
1. 学习多代理路由
2. 理解安全机制
3. 探索技能和插件系统

---

## 资源链接

- **官方文档**：https://docs.clawd.bot
- **GitHub**：https://github.com/clawdbot/clawdbot
- **Discord**：https://discord.gg/clawd
- **问题追踪**：https://github.com/clawdbot/clawdbot/issues

---

**祝你学习愉快！🦞**

如有问题，欢迎在 Discord 或 GitHub Issues 提问。
