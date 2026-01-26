---
summary: "Skills: managed vs workspace, gating rules, and config/env wiring"
read_when:
  - Adding or modifying skills
  - Changing skill gating or load rules
---
# 技能（Clawdbot）

Clawdbot 使用**[AgentSkills](https://agentskills.io) 兼容**技能文件夹来教代理如何使用工具。每个技能是一个包含 `SKILL.md` 的目录，带有 YAML frontmatter 和指令。Clawdbot 加载**捆绑技能**加上可选的本地覆盖，并根据环境、配置和二进制存在在加载时过滤它们。

## 位置和优先级

技能从**三个**位置加载：

1) **捆绑技能**：随安装提供（npm 包或 Clawdbot.app）
2) **托管/本地技能**：`~/.clawdbot/skills`
3) **工作区技能**：`<workspace>/skills`

如果技能名称冲突，优先级是：

`<workspace>/skills`（最高）→ `~/.clawdbot/skills` → 捆绑技能（最低）

此外，你可以通过 `~/.clawdbot/clawdbot.json` 中的 `skills.load.extraDirs` 配置额外的技能文件夹（最低优先级）。

## 每个代理 vs 共享技能

在**多代理**设置中，每个代理都有自己的工作区。这意味着：

- **每个代理技能**位于 `<workspace>/skills`，仅适用于该代理。
- **共享技能**位于 `~/.clawdbot/skills`（托管/本地），并且对**同一机器上的所有代理**可见。
- **共享文件夹**也可以通过 `skills.load.extraDirs` 添加（最低优先级），如果你想要多个代理使用的通用技能包。

如果同一技能名称存在于多个位置，通常的优先级适用：工作区获胜，然后是托管/本地，然后是捆绑。

## 插件 + 技能

插件可以通过在 `clawdbot.plugin.json` 中列出 `skills` 目录来提供自己的技能（相对于插件根的路径）。插件技能在启用插件时加载，并参与正常的技能优先级规则。你可以通过插件配置条目上的 `metadata.clawdbot.requires.config` 来门控它们。有关发现/配置，请参见 [插件](/plugin），有关这些技能教授的工具界面，请参见 [工具](/tools）。

## ClawdHub（安装 + 同步）

ClawdHub 是 Clawdbot 的公共技能注册表。在 https://clawdhub.com 浏览。使用它来发现、安装、更新和备份技能。
完整指南：[ClawdHub](/tools/clawdhub）。

常见流程：

- 将技能安装到你的工作区：
  - `clawdhub install <skill-slug>`
- 更新所有已安装的技能：
  - `clawdhub update --all`
- 同步（扫描 + 发布更新）：
  - `clawdhub sync --all`

默认情况下，`clawdhub` 安装到当前工作目录下的 `./skills`（或回退到配置的 Clawdbot 工作区）。Clawdbot 在下一个会话中将其作为 `<workspace>/skills` 拾取。

## 格式（AgentSkills + Pi 兼容）

`SKILL.md` 必须至少包括：

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
---
```

注意：
- 我们遵循 AgentSkills 规范进行布局/意图。
- 嵌入式代理使用的解析器仅支持**单行**frontmatter 键。
- `metadata` 应该是**单行 JSON 对象**。
- 在指令中使用 `{baseDir}` 来引用技能文件夹路径。
- 可选 frontmatter 键：
  - `homepage` — 在 macOS Skills UI 中显示为"Website"的 URL（也通过 `metadata.clawdbot.homepage` 支持）。
  - `user-invocable` — `true|false`（默认：`true`）。当 `true` 时，技能作为用户斜杠命令公开。
  - `disable-model-invocation` — `true|false`（默认：`false`）。当 `true` 时，技能从模型提示中排除（仍可通过用户调用获得）。
  - `command-dispatch` — `tool`（可选）。当设置为 `tool` 时，斜杠命令绕过模型并直接分派到工具。
  - `command-tool` — 当设置 `command-dispatch: tool` 时要调用的工具名称。
  - `command-arg-mode` — `raw`（默认）。对于工具分派，将原始 args 字符串转发到工具（无核心解析）。

    工具使用参数调用：
    `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`。

## 门控（加载时过滤器）

Clawdbot 使用 `metadata`（单行 JSON）**在加载时过滤**技能：

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
metadata: {"clawdbot":{"requires":{"bins":["uv"],"env":["GEMINI_API_KEY"],"config":["browser.enabled"]},"primaryEnv":"GEMINI_API_KEY"}}
---
```

`metadata.clawdbot` 下的字段：
- `always: true` — 始终包含技能（跳过其他门控）。
- `emoji` — macOS Skills UI 使用的可选表情符号。
- `homepage` — macOS Skills UI 中显示为"Website"的可选 URL。
- `os` — 可选平台列表（`darwin`、`linux`、`win32`）。如果设置，技能仅在这些操作系统上符合条件。
- `requires.bins` — 列表；每个必须在 `PATH` 上存在。
- `requires.anyBins` — 列表；至少一个必须在 `PATH` 上存在。
- `requires.env` — 列表；环境变量必须存在**或**在配置中提供。
- `requires.config` — 必须为真的 `clawdbot.json` 路径列表。
- `primaryEnv` — 与 `skills.entries.<name>.apiKey` 关联的环境变量名称。
- `install` — macOS Skills UI 使用的可选安装程序规范数组（brew/node/go/uv/download）。

关于沙箱化的说明：
- `requires.bins` 在技能加载时在**主机**上检查。
- 如果代理被沙箱化，二进制文件也必须在**容器内**存在。
  通过 `agents.defaults.sandbox.docker.setupCommand`（或自定义镜像）安装它。
  `setupCommand` 在创建容器后运行一次。
  包安装还需要网络出口、可写根文件系统和沙箱中的 root 用户。
  示例：`summarize` 技能（`skills/summarize/SKILL.md`）需要在沙箱容器中的 `summarize` CLI 才能在那里运行。

安装程序示例：

```markdown
---
name: gemini
description: Use Gemini CLI for coding assistance and Google search lookups.
metadata: {"clawdbot":{"emoji":"♊️","requires":{"bins":["gemini"]},"install":[{"id":"brew","kind":"brew","formula":"gemini-cli","bins":["gemini"],"label":"Install Gemini CLI (brew)"}]}}
---
```

注意：
- 如果列出了多个安装程序，网关选择**单个**首选选项（brew 可用时，否则 node）。
- 如果所有安装程序都是 `download`，Clawdbot 列出每个条目，以便你可以看到可用的工件。
- 安装程序规范可以包括 `os: ["darwin"|"linux"|"win32"]` 以按平台过滤选项。
- Node 安装遵守 `clawdbot.json` 中的 `skills.install.nodeManager`（默认：npm；选项：npm/pnpm/yarn/bun）。
  这仅影响**技能安装**；Gateway 运行时仍应为 Node（Bun 不推荐用于 WhatsApp/Telegram）。
- Go 安装：如果缺少 `go` 且 `brew` 可用，网关首先通过 Homebrew 安装 Go，并在可能时将 `GOBIN` 设置为 Homebrew 的 `bin`。
 - 下载安装：`url`（必需）、`archive`（`tar.gz` | `tar.bz2` | `zip`）、`extract`（默认：检测到归档时自动）、`stripComponents`、`targetDir`（默认：`~/.clawdbot/tools/<skillKey>`）。

如果不存在 `metadata.clawdbot`，技能始终符合条件（除非在配置中禁用或由捆绑技能的 `skills.allowBundled` 阻止）。

## 配置覆盖（`~/.clawdbot/clawdbot.json`）

捆绑/托管技能可以切换并提供环境值：

```json5
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
          endpoint: "https://example.invalid",
          model: "nano-pro"
        }
      },
      peekaboo: { enabled: true },
      sag: { enabled: false }
    }
  }
}
```

注意：如果技能名称包含连字符，请引用键（JSON5 允许引用的键）。

配置键默认匹配**技能名称**。如果技能定义 `metadata.clawdbot.skillKey`，请在 `skills.entries` 下使用该键。

规则：
- `enabled: false` 禁用技能，即使它是捆绑/安装的。
- `env`：**仅当**变量尚未在进程中设置时才注入。
- `apiKey`：为声明 `metadata.clawdbot.primaryEnv` 的技能提供便利。
- `config`：自定义每个技能字段的可选包；自定义键必须位于此处。
- `allowBundled`：**捆绑**技能的可选允许列表。如果设置，仅列表中的捆绑技能符合条件（托管/工作区技能不受影响）。

## 环境注入（每个代理运行）

当代理运行开始时，Clawdbot：
1) 读取技能元数据。
2) 将任何 `skills.entries.<key>.env` 或 `skills.entries.<key>.apiKey` 应用到 `process.env`。
3) 使用**符合条件的**技能构建系统提示。
4) 运行结束后恢复原始环境。

这是**作用域到代理运行**，不是全局 shell 环境。

## 会话快照（性能）

Clawdbot 在**会话开始时**快照符合条件的技能，并在同一会话的后续回合中重用该列表。对技能或配置的更改在下一个新会话时生效。

当启用技能监视器或出现新的符合条件的远程节点时，技能也可以在中途刷新（见下文）。将其视为**热重载**：刷新的列表在下一个代理回合时拾取。

## 远程 macOS 节点（Linux 网关）

如果 Gateway 在 Linux 上运行但**macOS 节点**已连接**且允许 `system.run`**（Exec 批准安全未设置为 `deny`），Clawdbot 可以在该节点上存在所需二进制文件时将仅 macOS 技能视为符合条件。代理应该通过 `nodes` 工具（通常是 `nodes.run`）执行这些技能。

这依赖于节点报告其命令支持以及通过 `system.run` 的二进制探测。如果 macOS 节点稍后离线，技能保持可见；调用可能失败，直到节点重新连接。

## 技能监视器（自动刷新）

默认情况下，Clawdbot 监视技能文件夹，并在 `SKILL.md` 文件更改时更新技能快照。在 `skills.load` 下配置：

```json5
{
  skills: {
    load: {
      watch: true,
      watchDebounceMs: 250
    }
  }
}
```

## Token 影响（技能列表）

当技能符合条件时，Clawdbot 将可用技能的紧凑 XML 列表注入到系统提示中（通过 `pi-coding-agent` 中的 `formatSkillsForPrompt`）。成本是确定性的：

- **基础开销（仅当 ≥1 技能时）：** 195 个字符。
- **每个技能：** 97 个字符 + XML 转义的 `<name>`、`<description>` 和 `<location>` 值的长度。

公式（字符）：

```
total = 195 + Σ (97 + len(name_escaped) + len(description_escaped) + len(location_escaped))
```

注意：
- XML 转义将 `& < > " '` 扩展为实体（`&amp;`、`&lt;` 等），增加长度。
- Token 计数因模型 tokenizer 而异。粗略的 OpenAI 风格估计约为 ~4 字符/token，因此**97 字符 ≈ 24 tokens** 每个技能加上你的实际字段长度。

## 托管技能生命周期

Clawdbot 将基线技能集作为**捆绑技能**随安装提供（npm 包或 Clawdbot.app）。`~/.clawdbot/skills` 用于本地覆盖（例如，在不更改捆绑副本的情况下固定/修补技能）。工作区技能是用户拥有的，并在名称冲突时覆盖两者。

## 配置参考

有关完整配置模式，请参见 [技能配置](/tools/skills-config）。

## 寻找更多技能？

浏览 https://clawdhub.com。

---
