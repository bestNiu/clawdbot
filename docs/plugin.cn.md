---
summary: "Clawdbot plugins/extensions: discovery, config, and safety"
read_when:
  - Adding or modifying plugins/extensions
  - Documenting plugin install or load rules
---
# 插件（扩展）

## 快速开始（新接触插件？）

插件只是一个**小代码模块**，用额外功能（命令、工具和 Gateway RPC）扩展 Clawdbot。

大多数时候，当你想要一个尚未内置到核心 Clawdbot 的功能（或者你想将可选功能排除在主安装之外）时，你会使用插件。

快速路径：

1) 查看已加载的内容：

```bash
clawdbot plugins list
```

2) 安装官方插件（示例：Voice Call）：

```bash
clawdbot plugins install @clawdbot/voice-call
```

3) 重启 Gateway，然后在 `plugins.entries.<id>.config` 下配置。

有关具体示例插件，请参见 [Voice Call](/plugins/voice-call）。

## 可用插件（官方）

- Microsoft Teams 自 2026.1.15 起仅作为插件；如果你使用 Teams，请安装 `@clawdbot/msteams`。
- Memory (Core) — 捆绑的内存搜索插件（通过 `plugins.slots.memory` 默认启用）
- Memory (LanceDB) — 捆绑的长期内存插件（自动召回/捕获；设置 `plugins.slots.memory = "memory-lancedb"`）
- [Voice Call](/plugins/voice-call) — `@clawdbot/voice-call`
- [Zalo Personal](/plugins/zalouser) — `@clawdbot/zalouser`
- [Matrix](/channels/matrix) — `@clawdbot/matrix`
- [Nostr](/channels/nostr) — `@clawdbot/nostr`
- [Zalo](/channels/zalo) — `@clawdbot/zalo`
- [Microsoft Teams](/channels/msteams) — `@clawdbot/msteams`
- Google Antigravity OAuth（提供者认证）— 作为 `google-antigravity-auth` 捆绑（默认禁用）
- Gemini CLI OAuth（提供者认证）— 作为 `google-gemini-cli-auth` 捆绑（默认禁用）
- Qwen OAuth（提供者认证）— 作为 `qwen-portal-auth` 捆绑（默认禁用）
- Copilot Proxy（提供者认证）— 本地 VS Code Copilot Proxy 桥接；与内置 `github-copilot` 设备登录不同（捆绑，默认禁用）

Clawdbot 插件是通过 jiti 在运行时加载的**TypeScript 模块**。**配置验证不执行插件代码**；它使用插件清单和 JSON Schema。参见 [插件清单](/plugins/manifest）。

插件可以注册：

- Gateway RPC 方法
- Gateway HTTP 处理程序
- 代理工具
- CLI 命令
- 后台服务
- 可选配置验证
- **技能**（通过在插件清单中列出 `skills` 目录）
- **自动回复命令**（无需调用 AI 代理即可执行）

插件**在进程内**与 Gateway 一起运行，因此将它们视为受信任的代码。
工具编写指南：[插件代理工具](/plugins/agent-tools）。

## 运行时助手

插件可以通过 `api.runtime` 访问选定的核心助手。对于电话 TTS：

```ts
const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from Clawdbot",
  cfg: api.config,
});
```

注意：
- 使用核心 `messages.tts` 配置（OpenAI 或 ElevenLabs）。
- 返回 PCM 音频缓冲区 + 采样率。插件必须为提供者重新采样/编码。
- 电话不支持 Edge TTS。

## 发现和优先级

Clawdbot 按顺序扫描：

1) 配置路径
- `plugins.load.paths`（文件或目录）

2) 工作区扩展
- `<workspace>/.clawdbot/extensions/*.ts`
- `<workspace>/.clawdbot/extensions/*/index.ts`

3) 全局扩展
- `~/.clawdbot/extensions/*.ts`
- `~/.clawdbot/extensions/*/index.ts`

4) 捆绑扩展（随 Clawdbot 提供，**默认禁用**）
- `<clawdbot>/extensions/*`

捆绑插件必须通过 `plugins.entries.<id>.enabled` 或 `clawdbot plugins enable <id>` 显式启用。安装的插件默认启用，但可以以相同方式禁用。

每个插件必须在其根目录中包含 `clawdbot.plugin.json` 文件。如果路径指向文件，插件根目录是文件的目录，必须包含清单。

如果多个插件解析为相同的 id，上述顺序中的第一个匹配获胜，较低优先级的副本被忽略。

### 包包

插件目录可以包含带有 `clawdbot.extensions` 的 `package.json`：

```json
{
  "name": "my-pack",
  "clawdbot": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"]
  }
}
```

每个条目成为一个插件。如果包列出多个扩展，插件 id 变为 `name/<fileBase>`。

如果你的插件导入 npm 依赖项，请在该目录中安装它们，以便 `node_modules` 可用（`npm install` / `pnpm install`）。

### 通道目录元数据

通道插件可以通过 `clawdbot.channel` 和通过 `clawdbot.install` 的安装提示来宣传引导元数据。这使核心目录保持无数据。

示例：

```json
{
  "name": "@clawdbot/nextcloud-talk",
  "clawdbot": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (self-hosted)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Self-hosted chat via Nextcloud Talk webhook bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@clawdbot/nextcloud-talk",
      "localPath": "extensions/nextcloud-talk",
      "defaultChoice": "npm"
    }
  }
}
```

Clawdbot 还可以合并**外部通道目录**（例如，MPM 注册表导出）。将 JSON 文件放在以下位置之一：
- `~/.clawdbot/mpm/plugins.json`
- `~/.clawdbot/mpm/catalog.json`
- `~/.clawdbot/plugins/catalog.json`

或者将 `CLAWDBOT_PLUGIN_CATALOG_PATHS`（或 `CLAWDBOT_MPM_CATALOG_PATHS`）指向一个或多个 JSON 文件（逗号/分号/`PATH` 分隔）。每个文件应包含 `{ "entries": [ { "name": "@scope/pkg", "clawdbot": { "channel": {...}, "install": {...} } } ] }`。

## 插件 ID

默认插件 id：

- 包包：`package.json` `name`
- 独立文件：文件基本名称（`~/.../voice-call.ts` → `voice-call`）

如果插件导出 `id`，Clawdbot 使用它，但当它与配置的 id 不匹配时会警告。

## 配置

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-extension"] },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } }
    }
  }
}
```

字段：
- `enabled`：主切换（默认：true）
- `allow`：允许列表（可选）
- `deny`：拒绝列表（可选；拒绝获胜）
- `load.paths`：额外插件文件/目录
- `entries.<id>`：每个插件切换 + 配置

配置更改**需要重启网关**。

验证规则（严格）：
- `entries`、`allow`、`deny` 或 `slots` 中的未知插件 id 是**错误**。
- 未知的 `channels.<id>` 键是**错误**，除非插件清单声明了通道 id。
- 插件配置使用嵌入在 `clawdbot.plugin.json`（`configSchema`）中的 JSON Schema 进行验证。
- 如果插件被禁用，其配置会被保留并发出**警告**。

## 插件插槽（独占类别）

某些插件类别是**独占的**（一次只能激活一个）。使用 `plugins.slots` 选择哪个插件拥有该插槽：

```json5
{
  plugins: {
    slots: {
      memory: "memory-core" // 或 "none" 以禁用内存插件
    }
  }
}
```

如果多个插件声明 `kind: "memory"`，只有选定的一个加载。其他插件被禁用并显示诊断信息。

## 控制 UI（模式 + 标签）

控制 UI 使用 `config.schema`（JSON Schema + `uiHints`）来渲染更好的表单。

Clawdbot 在运行时根据发现的插件增强 `uiHints`：

- 为 `plugins.entries.<id>` / `.enabled` / `.config` 添加每个插件标签
- 在以下位置合并可选的插件提供的配置字段提示：`plugins.entries.<id>.config.<field>`

如果你希望插件配置字段显示良好的标签/占位符（并将密钥标记为敏感），请在插件清单中与 JSON Schema 一起提供 `uiHints`。

示例：

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": { "type": "string" },
      "region": { "type": "string" }
    }
  },
  "uiHints": {
    "apiKey": { "label": "API Key", "sensitive": true },
    "region": { "label": "Region", "placeholder": "us-east-1" }
  }
}
```

## CLI

```bash
clawdbot plugins list
clawdbot plugins info <id>
clawdbot plugins install <path>                 # 将本地文件/目录复制到 ~/.clawdbot/extensions/<id>
clawdbot plugins install ./extensions/voice-call # 相对路径可以
clawdbot plugins install ./plugin.tgz           # 从本地 tarball 安装
clawdbot plugins install ./plugin.zip           # 从本地 zip 安装
clawdbot plugins install -l ./extensions/voice-call # 链接（不复制）用于开发
clawdbot plugins install @clawdbot/voice-call # 从 npm 安装
clawdbot plugins update <id>
clawdbot plugins update --all
clawdbot plugins enable <id>
clawdbot plugins disable <id>
clawdbot plugins doctor
```

`plugins update` 仅适用于在 `plugins.installs` 下跟踪的 npm 安装。

插件还可以注册自己的顶级命令（示例：`clawdbot voicecall`）。

## 插件 API（概述）

插件导出：

- 函数：`(api) => { ... }`
- 对象：`{ id, name, configSchema, register(api) { ... } }`

## 插件 hooks

插件可以附带 hooks 并在运行时注册它们。这允许插件捆绑事件驱动的自动化，而无需单独的 hook 包安装。

### 示例

```
import { registerPluginHooksFromDir } from "clawdbot/plugin-sdk";

export default function register(api) {
  registerPluginHooksFromDir(api, "./hooks");
}
```

注意：
- Hook 目录遵循正常的 hook 结构（`HOOK.md` + `handler.ts`）。
- Hook 资格规则仍然适用（OS/bins/env/config 要求）。
- 插件管理的 hooks 在 `clawdbot hooks list` 中显示为 `plugin:<id>`。
- 你不能通过 `clawdbot hooks` 启用/禁用插件管理的 hooks；而是启用/禁用插件。

## 提供者插件（模型认证）

插件可以注册**模型提供者认证**流程，以便用户可以在 Clawdbot 内运行 OAuth 或 API 密钥设置（无需外部脚本）。

通过 `api.registerProvider(...)` 注册提供者。每个提供者公开一个或多个认证方法（OAuth、API 密钥、设备代码等）。这些方法支持：

- `clawdbot models auth login --provider <id> [--method <id>]`

示例：

```ts
api.registerProvider({
  id: "acme",
  label: "AcmeAI",
  auth: [
    {
      id: "oauth",
      label: "OAuth",
      kind: "oauth",
      run: async (ctx) => {
        // 运行 OAuth 流程并返回认证配置文件。
        return {
          profiles: [
            {
              profileId: "acme:default",
              credential: {
                type: "oauth",
                provider: "acme",
                access: "...",
                refresh: "...",
                expires: Date.now() + 3600 * 1000,
              },
            },
          ],
          defaultModel: "acme/opus-1",
        };
      },
    },
  ],
});
```

注意：
- `run` 接收一个 `ProviderAuthContext`，其中包含 `prompter`、`runtime`、`openUrl` 和 `oauth.createVpsAwareHandlers` 助手。
- 当你需要添加默认模型或提供者配置时，返回 `configPatch`。
- 返回 `defaultModel`，以便 `--set-default` 可以更新代理默认值。

### 注册消息通道

插件可以注册**通道插件**，其行为类似于内置通道（WhatsApp、Telegram 等）。通道配置位于 `channels.<id>` 下，由你的通道插件代码验证。

```ts
const myChannel = {
  id: "acmechat",
  meta: {
    id: "acmechat",
    label: "AcmeChat",
    selectionLabel: "AcmeChat (API)",
    docsPath: "/channels/acmechat",
    blurb: "demo channel plugin.",
    aliases: ["acme"],
  },
  capabilities: { chatTypes: ["direct"] },
  config: {
    listAccountIds: (cfg) => Object.keys(cfg.channels?.acmechat?.accounts ?? {}),
    resolveAccount: (cfg, accountId) =>
      (cfg.channels?.acmechat?.accounts?.[accountId ?? "default"] ?? { accountId }),
  },
  outbound: {
    deliveryMode: "direct",
    sendText: async () => ({ ok: true }),
  },
};

export default function (api) {
  api.registerChannel({ plugin: myChannel });
}
```

注意：
- 将配置放在 `channels.<id>` 下（不是 `plugins.entries`）。
- `meta.label` 用于 CLI/UI 列表中的标签。
- `meta.aliases` 添加用于规范化和 CLI 输入的备用 id。
- `meta.preferOver` 列出当两者都配置时要跳过自动启用的通道 id。
- `meta.detailLabel` 和 `meta.systemImage` 让 UI 显示更丰富的通道标签/图标。

### 编写新消息通道（逐步）

当你想要**新聊天界面**（"消息通道"）时使用此方法，而不是模型提供者。
模型提供者文档位于 `/providers/*` 下。

1) 选择 id + 配置形状
- 所有通道配置位于 `channels.<id>` 下。
- 对于多账户设置，首选 `channels.<id>.accounts.<accountId>`。

2) 定义通道元数据
- `meta.label`、`meta.selectionLabel`、`meta.docsPath`、`meta.blurb` 控制 CLI/UI 列表。
- `meta.docsPath` 应指向像 `/channels/<id>` 这样的文档页面。
- `meta.preferOver` 允许插件替换另一个通道（自动启用优先选择它）。
- `meta.detailLabel` 和 `meta.systemImage` 由 UI 用于详细文本/图标。

3) 实现所需的适配器
- `config.listAccountIds` + `config.resolveAccount`
- `capabilities`（聊天类型、媒体、线程等）
- `outbound.deliveryMode` + `outbound.sendText`（用于基本发送）

4) 根据需要添加可选适配器
- `setup`（向导）、`security`（私信策略）、`status`（健康/诊断）
- `gateway`（启动/停止/登录）、`mentions`、`threading`、`streaming`
- `actions`（消息操作）、`commands`（本机命令行为）

5) 在你的插件中注册通道
- `api.registerChannel({ plugin })`

最小配置示例：

```json5
{
  channels: {
    acmechat: {
      accounts: {
        default: { token: "ACME_TOKEN", enabled: true }
      }
    }
  }
}
```

最小通道插件（仅出站）：

```ts
const plugin = {
  id: "acmechat",
  meta: {
    id: "acmechat",
    label: "AcmeChat",
    selectionLabel: "AcmeChat (API)",
    docsPath: "/channels/acmechat",
    blurb: "AcmeChat messaging channel.",
    aliases: ["acme"],
  },
  capabilities: { chatTypes: ["direct"] },
  config: {
    listAccountIds: (cfg) => Object.keys(cfg.channels?.acmechat?.accounts ?? {}),
    resolveAccount: (cfg, accountId) =>
      (cfg.channels?.acmechat?.accounts?.[accountId ?? "default"] ?? { accountId }),
  },
  outbound: {
    deliveryMode: "direct",
    sendText: async ({ text }) => {
      // 在这里将 `text` 传递到你的通道
      return { ok: true };
    },
  },
};

export default function (api) {
  api.registerChannel({ plugin });
}
```

加载插件（扩展目录或 `plugins.load.paths`），重启网关，然后在配置中配置 `channels.<id>`。

### 代理工具

参见专用指南：[插件代理工具](/plugins/agent-tools）。

### 注册网关 RPC 方法

```ts
export default function (api) {
  api.registerGatewayMethod("myplugin.status", ({ respond }) => {
    respond(true, { ok: true });
  });
}
```

### 注册 CLI 命令

```ts
export default function (api) {
  api.registerCli(({ program }) => {
    program.command("mycmd").action(() => {
      console.log("Hello");
    });
  }, { commands: ["mycmd"] });
}
```

### 注册自动回复命令

插件可以注册自定义斜杠命令，这些命令**无需调用 AI 代理**即可执行。这对于切换命令、状态检查或不需要 LLM 处理的快速操作很有用。

```ts
export default function (api) {
  api.registerCommand({
    name: "mystatus",
    description: "Show plugin status",
    handler: (ctx) => ({
      text: `Plugin is running! Channel: ${ctx.channel}`,
    }),
  });
}
```

命令处理程序上下文：

- `senderId`：发送者的 ID（如果可用）
- `channel`：发送命令的通道
- `isAuthorizedSender`：发送者是否是授权用户
- `args`：命令后传递的参数（如果 `acceptsArgs: true`）
- `commandBody`：完整命令文本
- `config`：当前 Clawdbot 配置

命令选项：

- `name`：命令名称（不带前导 `/`）
- `description`：在命令列表中显示的帮助文本
- `acceptsArgs`：命令是否接受参数（默认：false）。如果为 false 且提供了参数，命令将不匹配，消息将传递给其他处理程序
- `requireAuth`：是否需要授权发送者（默认：true）
- `handler`：返回 `{ text: string }` 的函数（可以是异步）

带授权和参数的示例：

```ts
api.registerCommand({
  name: "setmode",
  description: "Set plugin mode",
  acceptsArgs: true,
  requireAuth: true,
  handler: async (ctx) => {
    const mode = ctx.args?.trim() || "default";
    await saveMode(mode);
    return { text: `Mode set to: ${mode}` };
  },
});
```

注意：
- 插件命令在内置命令和 AI 代理**之前**处理
- 命令全局注册并在所有通道中工作
- 命令名称不区分大小写（`/MyStatus` 匹配 `/mystatus`）
- 命令名称必须以字母开头，并且只能包含字母、数字、连字符和下划线
- 保留的命令名称（如 `help`、`status`、`reset` 等）不能被插件覆盖
- 跨插件的重复命令注册将失败并显示诊断错误

### 注册后台服务

```ts
export default function (api) {
  api.registerService({
    id: "my-service",
    start: () => api.logger.info("ready"),
    stop: () => api.logger.info("bye"),
  });
}
```

## 命名约定

- Gateway 方法：`pluginId.action`（示例：`voicecall.status`）
- 工具：`snake_case`（示例：`voice_call`）
- CLI 命令：kebab 或 camel，但避免与核心命令冲突

## 技能

插件可以在仓库中附带技能（`skills/<name>/SKILL.md`）。
使用 `plugins.entries.<id>.enabled`（或其他配置门控）启用它，并确保它存在于你的工作区/托管技能位置。

## 分发（npm）

推荐的打包：

- 主包：`clawdbot`（此仓库）
- 插件：`@clawdbot/*` 下的单独 npm 包（示例：`@clawdbot/voice-call`）

发布合同：

- 插件 `package.json` 必须包含一个或多个入口文件的 `clawdbot.extensions`。
- 入口文件可以是 `.js` 或 `.ts`（jiti 在运行时加载 TS）。
- `clawdbot plugins install <npm-spec>` 使用 `npm pack`，提取到 `~/.clawdbot/extensions/<id>/`，并在配置中启用它。
- 配置键稳定性：作用域包被规范化为 `plugins.entries.*` 的**非作用域** id。

## 示例插件：Voice Call

此仓库包含语音呼叫插件（Twilio 或日志回退）：

- 源代码：`extensions/voice-call`
- 技能：`skills/voice-call`
- CLI：`clawdbot voicecall start|status`
- 工具：`voice_call`
- RPC：`voicecall.start`、`voicecall.status`
- 配置（twilio）：`provider: "twilio"` + `twilio.accountSid/authToken/from`（可选 `statusCallbackUrl`、`twimlUrl`）
- 配置（dev）：`provider: "log"`（无网络）

有关设置和用法，请参见 [Voice Call](/plugins/voice-call）和 `extensions/voice-call/README.md`。

## 安全说明

插件在进程内与 Gateway 一起运行。将它们视为受信任的代码：

- 仅安装你信任的插件。
- 首选 `plugins.allow` 允许列表。
- 更改后重启 Gateway。

## 测试插件

插件可以（并且应该）附带测试：

- 仓库内插件可以在 `src/**` 下保留 Vitest 测试（示例：`src/plugins/voice-call.plugin.test.ts`）。
- 单独发布的插件应该运行自己的 CI（lint/build/test）并验证 `clawdbot.extensions` 指向构建的入口点（`dist/index.js`）。
