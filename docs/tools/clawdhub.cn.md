---
summary: "ClawdHub guide: public skills registry + CLI workflows"
read_when:
  - Introducing ClawdHub to new users
  - Installing, searching, or publishing skills
  - Explaining ClawdHub CLI flags and sync behavior
---

# ClawdHub

ClawdHub 是 **Clawdbot 的公共技能注册表**。这是一项免费服务：所有技能都是公开的、开放的，对所有人可见，用于共享和重用。技能只是一个包含 `SKILL.md` 文件的文件夹（加上支持文本文件）。你可以在 Web 应用程序中浏览技能，或使用 CLI 搜索、安装、更新和发布技能。

网站：[clawdhub.com](https://clawdhub.com）

## 适合谁（初学者友好）

如果你想为 Clawdbot 代理添加新功能，ClawdHub 是查找和安装技能的最简单方法。你不需要知道后端如何工作。你可以：

- 通过纯语言搜索技能。
- 将技能安装到你的工作区。
- 稍后用一个命令更新技能。
- 通过发布它们来备份你自己的技能。

## 快速开始（非技术）

1) 安装 CLI（参见下一节）。
2) 搜索你需要的内容：
   - `clawdhub search "calendar"`
3) 安装技能：
   - `clawdhub install <skill-slug>`
4) 启动新的 Clawdbot 会话，以便它拾取新技能。

## 安装 CLI

选择一个：

```bash
npm i -g clawdhub
```

```bash
pnpm add -g clawdhub
```

## 它如何融入 Clawdbot

默认情况下，CLI 将技能安装到当前工作目录下的 `./skills`。如果配置了 Clawdbot 工作区，`clawdhub` 会回退到该工作区，除非你覆盖 `--workdir`（或 `CLAWDHUB_WORKDIR`）。Clawdbot 从 `<workspace>/skills` 加载工作区技能，并将在**下一个**会话中拾取它们。如果你已经使用 `~/.clawdbot/skills` 或捆绑技能，工作区技能优先。

有关技能如何加载、共享和门控的更多详细信息，请参见 [技能](/tools/skills）。

## 服务提供的内容（功能）

- **公共浏览**技能及其 `SKILL.md` 内容。
- **搜索**由嵌入（向量搜索）提供支持，不仅仅是关键词。
- **版本控制**带有 semver、changelog 和标签（包括 `latest`）。
- **下载**作为每个版本的 zip。
- **星标和评论**用于社区反馈。
- **审核**钩子用于批准和审计。
- **CLI 友好 API**用于自动化和脚本编写。

## CLI 命令和参数

全局选项（适用于所有命令）：

- `--workdir <dir>`：工作目录（默认：当前目录；回退到 Clawdbot 工作区）。
- `--dir <dir>`：技能目录，相对于 workdir（默认：`skills`）。
- `--site <url>`：站点基础 URL（浏览器登录）。
- `--registry <url>`：注册表 API 基础 URL。
- `--no-input`：禁用提示（非交互）。
- `-V, --cli-version`：打印 CLI 版本。

认证：

- `clawdhub login`（浏览器流程）或 `clawdhub login --token <token>`
- `clawdhub logout`
- `clawdhub whoami`

选项：

- `--token <token>`：粘贴 API 令牌。
- `--label <label>`：为浏览器登录令牌存储的标签（默认：`CLI token`）。
- `--no-browser`：不打开浏览器（需要 `--token`）。

搜索：

- `clawdhub search "query"`
- `--limit <n>`：最大结果。

安装：

- `clawdhub install <slug>`
- `--version <version>`：安装特定版本。
- `--force`：如果文件夹已存在则覆盖。

更新：

- `clawdhub update <slug>`
- `clawdhub update --all`
- `--version <version>`：更新到特定版本（仅单个 slug）。
- `--force`：当本地文件不匹配任何已发布版本时覆盖。

列表：

- `clawdhub list`（读取 `.clawdhub/lock.json`）

发布：

- `clawdhub publish <path>`
- `--slug <slug>`：技能 slug。
- `--name <name>`：显示名称。
- `--version <version>`：Semver 版本。
- `--changelog <text>`：Changelog 文本（可以为空）。
- `--tags <tags>`：逗号分隔的标签（默认：`latest`）。

删除/取消删除（仅所有者/管理员）：

- `clawdhub delete <slug> --yes`
- `clawdhub undelete <slug> --yes`

同步（扫描本地技能 + 发布新/更新的）：

- `clawdhub sync`
- `--root <dir...>`：额外扫描根。
- `--all`：无提示上传所有内容。
- `--dry-run`：显示将上传的内容。
- `--bump <type>`：更新的 `patch|minor|major`（默认：`patch`）。
- `--changelog <text>`：非交互更新的 changelog。
- `--tags <tags>`：逗号分隔的标签（默认：`latest`）。
- `--concurrency <n>`：注册表检查（默认：4）。

## 代理的常见工作流

### 搜索技能

```bash
clawdhub search "postgres backups"
```

### 下载新技能

```bash
clawdhub install my-skill-pack
```

### 更新已安装的技能

```bash
clawdhub update --all
```

### 备份你的技能（发布或同步）

对于单个技能文件夹：

```bash
clawdhub publish ./my-skill --slug my-skill --name "My Skill" --version 1.0.0 --tags latest
```

要扫描并一次备份许多技能：

```bash
clawdhub sync --all
```

## 高级详细信息（技术）

### 版本控制和标签

- 每次发布创建一个新的 **semver** `SkillVersion`。
- 标签（如 `latest`）指向版本；移动标签让你可以回滚。
- Changelog 附加到每个版本，在同步或发布更新时可以为空。

### 本地更改 vs 注册表版本

更新使用内容哈希将本地技能内容与注册表版本进行比较。如果本地文件不匹配任何已发布版本，CLI 在覆盖前询问（或在非交互运行中需要 `--force`）。

### 同步扫描和回退根

`clawdhub sync` 首先扫描你的当前 workdir。如果未找到技能，它回退到已知的传统位置（例如 `~/clawdbot/skills` 和 `~/.clawdbot/skills`）。这旨在无需额外标志即可找到较旧的技能安装。

### 存储和 lockfile

- 已安装的技能记录在你的 workdir 下的 `.clawdhub/lock.json` 中。
- 认证令牌存储在 ClawdHub CLI 配置文件中（通过 `CLAWDHUB_CONFIG_PATH` 覆盖）。

### 遥测（安装计数）

当你在登录时运行 `clawdhub sync` 时，CLI 发送最小快照以计算安装计数。你可以完全禁用此功能：

```bash
export CLAWDHUB_DISABLE_TELEMETRY=1
```

## 环境变量

- `CLAWDHUB_SITE`：覆盖站点 URL。
- `CLAWDHUB_REGISTRY`：覆盖注册表 API URL。
- `CLAWDHUB_CONFIG_PATH`：覆盖 CLI 存储令牌/配置的位置。
- `CLAWDHUB_WORKDIR`：覆盖默认 workdir。
- `CLAWDHUB_DISABLE_TELEMETRY=1`：在 `sync` 上禁用遥测。
