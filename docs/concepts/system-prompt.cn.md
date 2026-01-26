---
summary: "What the Clawdbot system prompt contains and how it is assembled"
read_when:
  - Editing system prompt text, tools list, or time/heartbeat sections
  - Changing workspace bootstrap or skills injection behavior
---
# 系统提示

Clawdbot 为每个代理运行构建自定义系统提示。提示是**Clawdbot 拥有的**，不使用 p-coding-agent 默认提示。

提示由 Clawdbot 组装并注入到每个代理运行中。

## 结构

提示有意紧凑并使用固定部分：

- **工具**：当前工具列表 + 简短描述。
- **技能**（可用时）：告诉模型如何按需加载技能说明。
- **Clawdbot 自我更新**：如何运行 `config.apply` 和 `update.run`。
- **工作区**：工作目录（`agents.defaults.workspace`）。
- **文档**：Clawdbot 文档的本地路径（仓库或 npm 包）以及何时读取它们。
- **工作区文件（注入）**：指示下面包含引导文件。
- **沙箱**（启用时）：指示沙箱运行时、沙箱路径以及是否可用提升的 exec。
- **当前日期和时间**：用户本地时间、时区和时间格式。
- **回复标签**：支持提供者的可选回复标签语法。
- **心跳**：心跳提示和确认行为。
- **运行时**：主机、OS、节点、模型、仓库根（检测到时）、思考级别（一行）。
- **推理**：当前可见性级别 + /reasoning 切换提示。

## 提示模式

Clawdbot 可以为子代理渲染较小的系统提示。运行时为每次运行设置 `promptMode`（不是面向用户的配置）：

- `full`（默认）：包括上述所有部分。
- `minimal`：用于子代理；省略**技能**、**记忆召回**、**Clawdbot 自我更新**、**模型别名**、**用户身份**、**回复标签**、**消息传递**、**静默回复**和**心跳**。工具、工作区、沙箱、当前日期和时间（已知时）、运行时和注入的上下文保持可用。
- `none`：仅返回基本身份行。

当 `promptMode=minimal` 时，额外的注入提示标记为**子代理上下文**而不是**群聊上下文**。

## 工作区引导注入

引导文件被修剪并附加在**项目上下文**下，以便模型看到身份和配置文件上下文，而无需显式读取：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（仅在全新工作区上）

大文件用标记截断。每个文件的最大大小由 `agents.defaults.bootstrapMaxChars`（默认：20000）控制。缺失的文件注入一个简短的缺失文件标记。

内部钩子可以通过 `agent:bootstrap` 拦截此步骤，以改变或替换注入的引导文件（例如将 `SOUL.md` 交换为备用角色）。

要检查每个注入文件贡献多少（原始 vs 注入、截断，加上工具模式开销），使用 `/context list` 或 `/context detail`。参见 [上下文](/concepts/context)。

## 时间处理

当已知用户时区时，系统提示包括专用的**当前日期和时间**部分。为了保持提示缓存稳定，它现在仅包括**时区**（无动态时钟或时间格式）。

当代理需要当前时间时，使用 `session_status`；状态卡包括一个时间戳行。

配置：

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat`（`auto` | `12` | `24`）

有关完整行为详细信息，请参见 [日期和时间](/date-time)。

## 技能

当存在符合条件的技能时，Clawdbot 注入一个紧凑的**可用技能列表**（`formatSkillsForPrompt`），其中包括每个技能的**文件路径**。提示指示模型使用 `read` 在列出的位置（工作区、托管或捆绑）加载 SKILL.md。如果没有符合条件的技能，则省略技能部分。

```
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
  </skill>
</available_skills>
```

这保持基本提示小，同时仍启用定向技能使用。

## 文档

可用时，系统提示包括一个**文档**部分，指向本地 Clawdbot 文档目录（仓库工作区中的 `docs/` 或捆绑的 npm 包文档），还注明公共镜像、源仓库、社区 Discord 和 ClawdHub (https://clawdhub.com) 用于技能发现。提示指示模型首先查阅本地文档以了解 Clawdbot 行为、命令、配置或架构，并在可能时运行 `clawdbot status`（仅在缺少访问权限时询问用户）。
