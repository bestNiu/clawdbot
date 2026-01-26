---
summary: "Skills config schema and examples"
read_when:
  - Adding or modifying skills config
  - Adjusting bundled allowlist or install behavior
---
# 技能配置

所有技能相关配置位于 `~/.clawdbot/clawdbot.json` 中的 `skills` 下。

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: [
        "~/Projects/agent-scripts/skills",
        "~/Projects/oss/some-skill-pack/skills"
      ],
      watch: true,
      watchDebounceMs: 250
    },
    install: {
      preferBrew: true,
      nodeManager: "npm" // npm | pnpm | yarn | bun（Gateway 运行时仍为 Node；不推荐 bun）
    },
    entries: {
      "nano-banana-pro": {
        enabled: true,
        apiKey: "GEMINI_KEY_HERE",
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE"
        }
      },
      peekaboo: { enabled: true },
      sag: { enabled: false }
    }
  }
}
```

## 字段

- `allowBundled`：**捆绑**技能的可选允许列表。当设置时，仅列表中的捆绑技能符合条件（托管/工作区技能不受影响）。
- `load.extraDirs`：要扫描的额外技能目录（最低优先级）。
- `load.watch`：监视技能文件夹并刷新技能快照（默认：true）。
- `load.watchDebounceMs`：技能监视器事件的防抖（毫秒）（默认：250）。
- `install.preferBrew`：可用时首选 brew 安装程序（默认：true）。
- `install.nodeManager`：节点安装程序首选项（`npm` | `pnpm` | `yarn` | `bun`，默认：npm）。
  这仅影响**技能安装**；Gateway 运行时仍应为 Node（Bun 不推荐用于 WhatsApp/Telegram）。
- `entries.<skillKey>`：每个技能覆盖。

每个技能字段：
- `enabled`：设置 `false` 以禁用技能，即使它是捆绑/安装的。
- `env`：为代理运行注入的环境变量（仅当尚未设置时）。
- `apiKey`：为声明主要环境变量的技能提供可选便利。

## 说明

- `entries` 下的键默认映射到技能名称。如果技能定义 `metadata.clawdbot.skillKey`，请改用该键。
- 当启用监视器时，对技能的更改在下一个代理回合时拾取。

### 沙箱化技能 + 环境变量

当会话被**沙箱化**时，技能进程在 Docker 内运行。沙箱**不**继承主机 `process.env`。

使用以下之一：
- `agents.defaults.sandbox.docker.env`（或每个代理 `agents.list[].sandbox.docker.env`）
- 将 env 烘焙到你的自定义沙箱镜像中

全局 `env` 和 `skills.entries.<skill>.env/apiKey` 仅适用于**主机**运行。
