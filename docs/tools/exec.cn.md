---
summary: "Exec tool usage, stdin modes, and TTY support"
read_when:
  - Using or modifying the exec tool
  - Debugging stdin or TTY behavior
---

# Exec 工具

在工作区中运行 shell 命令。通过 `process` 支持前台 + 后台执行。
如果 `process` 被拒绝，`exec` 同步运行并忽略 `yieldMs`/`background`。
后台会话按代理作用域；`process` 仅看到来自同一代理的会话。

## 参数

- `command`（必需）
- `workdir`（默认为 cwd）
- `env`（键/值覆盖）
- `yieldMs`（默认 10000）：延迟后自动后台
- `background`（布尔值）：立即后台
- `timeout`（秒，默认 1800）：过期时杀死
- `pty`（布尔值）：在可用时在伪终端中运行（仅 TTY CLI、编码代理、终端 UI）
- `host`（`sandbox | gateway | node`）：执行位置
- `security`（`deny | allowlist | full`）：`gateway`/`node` 的强制执行模式
- `ask`（`off | on-miss | always`）：`gateway`/`node` 的批准提示
- `node`（字符串）：`host=node` 的节点 id/名称
- `elevated`（布尔值）：请求提升模式（网关主机）；`security=full` 仅在提升解析为 `full` 时强制

注意：
- `host` 默认为 `sandbox`。
- 当沙箱化关闭时，`elevated` 被忽略（exec 已经在主机上运行）。
- `gateway`/`node` 批准由 `~/.clawdbot/exec-approvals.json` 控制。
- `node` 需要配对节点（伴侣应用程序或无头节点主机）。
- 如果多个节点可用，设置 `exec.node` 或 `tools.exec.node` 以选择一个。
- 在非 Windows 主机上，exec 在设置时使用 `SHELL`；如果 `SHELL` 是 `fish`，它首选 `PATH` 中的 `bash`（或 `sh`）以避免 fish 不兼容的脚本，如果两者都不存在，则回退到 `SHELL`。

## 配置

- `tools.exec.notifyOnExit`（默认：true）：当为 true 时，后台 exec 会话排队系统事件并在退出时请求心跳。
- `tools.exec.approvalRunningNoticeMs`（默认：10000）：当批准门控 exec 运行超过此时发出单个"运行中"通知（0 禁用）。
- `tools.exec.host`（默认：`sandbox`）
- `tools.exec.security`（默认：未设置时为沙箱 `deny`，网关 + 节点 `allowlist`）
- `tools.exec.ask`（默认：`on-miss`）
- `tools.exec.node`（默认：未设置）
- `tools.exec.pathPrepend`：为 exec 运行前置到 `PATH` 的目录列表。
- `tools.exec.safeBins`：可以在没有显式允许列表条目的情况下运行的仅 stdin 安全二进制文件。

示例：
```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"]
    }
  }
}
```

### PATH 处理

- `host=gateway`：将你的登录 shell `PATH` 合并到 exec 环境中（除非 exec 调用已经设置 `env.PATH`）。守护进程本身仍以最小 `PATH` 运行：
  - macOS：`/opt/homebrew/bin`、`/usr/local/bin`、`/usr/bin`、`/bin`
  - Linux：`/usr/local/bin`、`/usr/bin`、`/bin`
- `host=sandbox`：在容器内运行 `sh -lc`（登录 shell），因此 `/etc/profile` 可能重置 `PATH`。
  Clawdbot 在配置文件源之后前置 `env.PATH`；`tools.exec.pathPrepend` 也适用于此。
- `host=node`：仅发送你传递的 env 覆盖到节点。`tools.exec.pathPrepend` 仅在 exec 调用已经设置 `env.PATH` 时适用。无头节点主机仅在它前置节点主机 PATH 时接受 `PATH`（无替换）。macOS 节点完全删除 `PATH` 覆盖。

每个代理节点绑定（在配置中使用代理列表索引）：

```bash
clawdbot config get agents.list
clawdbot config set agents.list[0].tools.exec.node "node-id-or-name"
```

控制 UI：节点选项卡包括一个小的"Exec 节点绑定"面板，用于相同的设置。

## 会话覆盖（`/exec`）

使用 `/exec` 为 `host`、`security`、`ask` 和 `node` 设置**每个会话**默认值。
发送不带参数的 `/exec` 以显示当前值。

示例：
```
/exec host=gateway security=allowlist ask=on-miss node=mac-1
```

## Exec 批准（伴侣应用程序 / 节点主机）

沙箱化代理可以在 `exec` 在网关或节点主机上运行之前需要每个请求批准。
有关策略、允许列表和 UI 流程，请参见 [Exec 批准](/tools/exec-approvals）。

当需要批准时，exec 工具立即返回 `status: "approval-pending"` 和批准 id。一旦批准（或拒绝 / 超时），Gateway 发出系统事件（`Exec finished` / `Exec denied`）。如果命令在 `tools.exec.approvalRunningNoticeMs` 后仍在运行，则发出单个 `Exec running` 通知。

## 允许列表 + 安全二进制文件

允许列表强制执行仅匹配**解析的二进制路径**（无基本名称匹配）。当 `security=allowlist` 时，shell 命令仅在每个管道段都被允许列表或安全二进制文件时才自动允许。链接（`;`、`&&`、`||`）和重定向在允许列表模式下被拒绝。

## 示例

前台：
```json
{"tool":"exec","command":"ls -la"}
```

后台 + 轮询：
```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

发送键（tmux 风格）：
```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

提交（仅发送 CR）：
```json
{"tool":"process","action":"submit","sessionId":"<id>"}
```

粘贴（默认括号）：
```json
{"tool":"process","action":"paste","sessionId":"<id>","text":"line1\nline2\n"}
```

## apply_patch（实验性）

`apply_patch` 是 `exec` 的子工具，用于结构化多文件编辑。
显式启用它：

```json5
{
  tools: {
    exec: {
      applyPatch: { enabled: true, allowModels: ["gpt-5.2"] }
    }
  }
}
```

注意：
- 仅适用于 OpenAI/OpenAI Codex 模型。
- 工具策略仍然适用；`allow: ["exec"]` 隐式允许 `apply_patch`。
- 配置位于 `tools.exec.applyPatch` 下。
