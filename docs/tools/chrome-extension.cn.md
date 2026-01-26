---
summary: "Chrome extension: let Clawdbot drive your existing Chrome tab"
read_when:
  - You want the agent to drive an existing Chrome tab (toolbar button)
  - You need remote Gateway + local browser automation via Tailscale
  - You want to understand the security implications of browser takeover
---

# Chrome 扩展（浏览器中继）

Clawdbot Chrome 扩展允许代理控制你**现有的 Chrome 标签页**（你的正常 Chrome 窗口），而不是启动单独的 clawd 管理的 Chrome 配置文件。

通过**单个 Chrome 工具栏按钮**附加/分离。

## 它是什么（概念）

有三个部分：
- **浏览器控制服务器**（HTTP）：代理/工具调用的 API（`browser.controlUrl`）
- **本地中继服务器**（loopback CDP）：在控制服务器和扩展之间桥接（默认 `http://127.0.0.1:18792`）
- **Chrome MV3 扩展**：使用 `chrome.debugger` 附加到活动标签页，并将 CDP 消息管道传输到中继

然后 Clawdbot 通过正常的 `browser` 工具界面控制附加的标签页（选择正确的配置文件）。

## 安装 / 加载（解压缩）

1) 将扩展安装到稳定的本地路径：

```bash
clawdbot browser extension install
```

2) 打印已安装的扩展目录路径：

```bash
clawdbot browser extension path
```

3) Chrome → `chrome://extensions`
- 启用"开发者模式"
- "加载解压缩的扩展" → 选择上面打印的目录

4) 固定扩展。

## 更新（无构建步骤）

扩展作为静态文件随 Clawdbot 发布（npm 包）提供。没有单独的"构建"步骤。

升级 Clawdbot 后：
- 重新运行 `clawdbot browser extension install` 以刷新 Clawdbot 状态目录下安装的文件。
- Chrome → `chrome://extensions` → 在扩展上点击"重新加载"。

## 使用它（无需额外配置）

Clawdbot 附带一个名为 `chrome` 的内置浏览器配置文件，该配置文件在默认端口上定位扩展中继。

使用它：
- CLI：`clawdbot browser --browser-profile chrome tabs`
- 代理工具：`browser` 与 `profile="chrome"`

如果你想要不同的名称或不同的中继端口，创建你自己的配置文件：

```bash
clawdbot browser create-profile \
  --name my-chrome \
  --driver extension \
  --cdp-url http://127.0.0.1:18792 \
  --color "#00AA00"
```

## 附加 / 分离（工具栏按钮）

- 打开你想要 Clawdbot 控制的标签页。
- 点击扩展图标。
  - 附加时徽章显示 `ON`。
- 再次点击以分离。

## 它控制哪个标签页？

- 它**不**自动控制"你正在查看的任何标签页"。
- 它仅控制**你通过点击工具栏按钮明确附加的标签页**。
- 要切换：打开另一个标签页并在那里点击扩展图标。

## 徽章 + 常见错误

- `ON`：已附加；Clawdbot 可以驱动该标签页。
- `…`：正在连接到本地中继。
- `!`：中继无法访问（最常见：此机器上未运行浏览器中继服务器）。

如果你看到 `!`：
- 确保 Gateway 在本地运行（默认设置），或在此机器上运行 `clawdbot browser serve`（远程网关设置）。
- 打开扩展选项页面；它显示中继是否可访问。

## 我需要 `clawdbot browser serve` 吗？

### 本地 Gateway（与 Chrome 同一机器）— 通常**不需要**

如果 Gateway 与 Chrome 在同一机器上运行，并且你的 `browser.controlUrl` 是 loopback（默认），你通常**不需要** `clawdbot browser serve`。

Gateway 的内置浏览器控制服务器将在 `http://127.0.0.1:18791/` 上启动，Clawdbot 将在 `http://127.0.0.1:18792/` 上自动启动本地中继服务器。

### 远程 Gateway（Gateway 在其他地方运行）— **需要**

如果你的 Gateway 在另一台机器上运行，在运行 Chrome 的机器上运行 `clawdbot browser serve`（并通过 Tailscale Serve / TLS 发布它）。参见下面的部分。

## 沙箱化（工具容器）

如果你的代理会话被沙箱化（`agents.defaults.sandbox.mode != "off"`），`browser` 工具可能被限制：

- 默认情况下，沙箱化会话通常定位**沙箱浏览器**（`target="sandbox"`），而不是你的主机 Chrome。
- Chrome 扩展中继接管需要控制**主机**浏览器控制服务器。

选项：
- 最简单：从**非沙箱化**会话/代理使用扩展。
- 或允许沙箱化会话的主机浏览器控制：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        browser: {
          allowHostControl: true
        }
      }
    }
  }
}
```

然后确保工具不被工具策略拒绝，并且（如果需要）使用 `target="host"` 调用 `browser`。

调试：`clawdbot sandbox explain`

## 远程 Gateway（推荐：Tailscale Serve）

目标：Gateway 在一台机器上运行，但 Chrome 在其他地方运行。

在**浏览器机器**上：

```bash
clawdbot browser serve --bind 127.0.0.1 --port 18791 --token <token>
tailscale serve https / http://127.0.0.1:18791
```

在**Gateway 机器**上：
- 将 `browser.controlUrl` 设置为 HTTPS Serve URL（MagicDNS/ts.net）。
- 提供令牌（首选 env）：

```bash
export CLAWDBOT_BROWSER_CONTROL_TOKEN="<token>"
```

然后代理可以通过调用远程 `browser.controlUrl` API 来驱动浏览器，而扩展 + 中继保持在浏览器机器上的本地。

## "扩展路径"如何工作

`clawdbot browser extension path` 打印包含扩展文件的**已安装**磁盘目录。

CLI 故意**不**打印 `node_modules` 路径。始终首先运行 `clawdbot browser extension install` 以将扩展复制到 Clawdbot 状态目录下的稳定位置。

如果你移动或删除该安装目录，Chrome 会将扩展标记为损坏，直到你从有效路径重新加载它。

## 安全影响（阅读此内容）

这很强大且有风险。将其视为给模型"在你的浏览器上动手"。

- 扩展使用 Chrome 的调试器 API（`chrome.debugger`）。附加时，模型可以：
  - 在该标签页中点击/输入/导航
  - 读取页面内容
  - 访问标签页的已登录会话可以访问的任何内容
- **这不是隔离的**，就像专用的 clawd 管理的配置文件一样。
  - 如果你附加到日常驱动程序配置文件/标签页，你正在授予对该账户状态的访问权限。

建议：
- 首选专用 Chrome 配置文件（与你的个人浏览分开）用于扩展中继使用。
- 保持浏览器控制服务器仅 tailnet（Tailscale）并要求令牌。
- 避免通过 LAN（`0.0.0.0`）暴露浏览器控制，避免 Funnel（公共）。

相关：
- 浏览器工具概述：[浏览器](/tools/browser）
- 安全审计：[安全](/gateway/security）
- Tailscale 设置：[Tailscale](/gateway/tailscale）
