---
summary: "Integrated browser control server + action commands"
read_when:
  - Adding agent-controlled browser automation
  - Debugging why clawd is interfering with your own Chrome
  - Implementing browser settings + lifecycle in the macOS app
---

# 浏览器（clawd 管理）

Clawdbot 可以运行代理控制的**专用 Chrome/Brave/Edge/Chromium 配置文件**。
它与你的个人浏览器隔离，通过小型本地控制服务器管理。

初学者视图：
- 将其视为**单独的、仅代理的浏览器**。
- `clawd` 配置文件**不**接触你的个人浏览器配置文件。
- 代理可以在安全通道中**打开标签页、读取页面、点击和输入**。
- 默认 `chrome` 配置文件通过扩展中继使用**系统默认 Chromium 浏览器**；切换到 `clawd` 以使用隔离的管理浏览器。

## 你得到什么

- 名为 **clawd** 的单独浏览器配置文件（默认橙色强调）。
- 确定性标签页控制（列出/打开/聚焦/关闭）。
- 代理操作（点击/输入/拖拽/选择）、快照、截图、PDF。
- 可选多配置文件支持（`clawd`、`work`、`remote`、...）。

此浏览器**不是**你的日常驱动程序。它是用于代理自动化和验证的安全、隔离界面。

## 快速开始

```bash
clawdbot browser --browser-profile clawd status
clawdbot browser --browser-profile clawd start
clawdbot browser --browser-profile clawd open https://example.com
clawdbot browser --browser-profile clawd snapshot
```

如果你看到"Browser disabled"，在配置中启用它（见下文）并重启 Gateway。

## 配置文件：`clawd` vs `chrome`

- `clawd`：管理的、隔离的浏览器（不需要扩展）。
- `chrome`：通过扩展中继到你的**系统浏览器**（需要 Clawdbot 扩展附加到标签页）。

如果你想要默认管理模式，设置 `browser.defaultProfile: "clawd"`。

## 配置

浏览器设置位于 `~/.clawdbot/clawdbot.json`。

```json5
{
  browser: {
    enabled: true,                    // 默认：true
    controlUrl: "http://127.0.0.1:18791",
    cdpUrl: "http://127.0.0.1:18792", // 默认为 controlUrl + 1
    remoteCdpTimeoutMs: 1500,         // 远程 CDP HTTP 超时（毫秒）
    remoteCdpHandshakeTimeoutMs: 3000, // 远程 CDP WebSocket 握手超时（毫秒）
    defaultProfile: "chrome",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      clawd: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" }
    }
  }
}
```

注意：
- `controlUrl` 默认为 `http://127.0.0.1:18791`。
- 如果你覆盖 Gateway 端口（`gateway.port` 或 `CLAWDBOT_GATEWAY_PORT`），默认浏览器端口会移动以保持在同一个"系列"中（control = gateway + 2）。
- 未设置时，`cdpUrl` 默认为 `controlUrl + 1`。
- `remoteCdpTimeoutMs` 适用于远程（非 loopback）CDP 可达性检查。
- `remoteCdpHandshakeTimeoutMs` 适用于远程 CDP WebSocket 可达性检查。
- `attachOnly: true` 意味着"永远不启动本地浏览器；仅在已运行时附加"。
- `color` + 每个配置文件的 `color` 为浏览器 UI 着色，以便你可以看到哪个配置文件处于活动状态。
- 默认配置文件是 `chrome`（扩展中继）。对于管理浏览器，使用 `defaultProfile: "clawd"`。
- 自动检测顺序：如果基于 Chromium，则为系统默认浏览器；否则 Chrome → Brave → Edge → Chromium → Chrome Canary。
- 本地 `clawd` 配置文件自动分配 `cdpPort`/`cdpUrl` — 仅对远程 CDP 设置这些。

## 使用 Brave（或其他基于 Chromium 的浏览器）

如果你的**系统默认**浏览器是基于 Chromium 的（Chrome/Brave/Edge 等），Clawdbot 自动使用它。设置 `browser.executablePath` 以覆盖自动检测：

CLI 示例：

```bash
clawdbot config set browser.executablePath "/usr/bin/google-chrome"
```

```json5
// macOS
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
  }
}

// Windows
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
  }
}

// Linux
{
  browser: {
    executablePath: "/usr/bin/brave-browser"
  }
}
```

## 本地 vs 远程控制

- **本地控制（默认）：** `controlUrl` 是 loopback（`127.0.0.1`/`localhost`）。Gateway 启动控制服务器并可以启动本地浏览器。
- **远程控制：** `controlUrl` 是非 loopback。Gateway **不**启动本地服务器；它假设你指向其他地方的现有服务器。
- **远程 CDP：** 设置 `browser.profiles.<name>.cdpUrl`（或 `browser.cdpUrl`）以附加到远程基于 Chromium 的浏览器。在这种情况下，Clawdbot 不会启动本地浏览器。

## 远程浏览器（控制服务器）

你可以在另一台机器上运行**浏览器控制服务器**，并使用远程 `controlUrl` 将 Gateway 指向它。这允许代理驱动主机外的浏览器（实验室盒子、VM、远程桌面等）。

关键点：
- **控制服务器**通过 **CDP** 与基于 Chromium 的浏览器（Chrome/Brave/Edge/Chromium）通信。
- **Gateway** 只需要 HTTP 控制 URL。
- 配置文件在**控制服务器**端解析。

示例：
```json5
{
  browser: {
    enabled: true,
    controlUrl: "http://10.0.0.42:18791",
    defaultProfile: "work"
  }
}
```

如果你希望 Gateway 直接与基于 Chromium 的浏览器实例对话而不使用远程控制服务器，请使用 `profiles.<name>.cdpUrl` 进行**远程 CDP**。

远程 CDP URL 可以包括认证：
- 查询令牌（例如，`https://provider.example?token=<token>`）
- HTTP Basic 认证（例如，`https://user:pass@provider.example`）

Clawdbot 在调用 `/json/*` 端点以及连接到 CDP WebSocket 时保留认证。首选环境变量或密钥管理器用于令牌，而不是将它们提交到配置文件。

### 节点浏览器代理（零配置默认）

如果你在拥有浏览器的机器上运行**节点主机**，Clawdbot 可以自动将浏览器工具调用路由到该节点，而无需任何自定义 `controlUrl` 设置。这是远程网关的默认路径。

注意：
- 节点主机通过**代理命令**公开其本地浏览器控制服务器。
- 配置文件来自节点自己的 `browser.profiles` 配置（与本地相同）。
- 如果你不想要它，请禁用：
  - 在节点上：`nodeHost.browserProxy.enabled=false`
  - 在网关上：`gateway.nodes.browser.mode="off"`

### Browserless（托管远程 CDP）

[Browserless](https://browserless.io) 是一个托管 Chromium 服务，通过 HTTPS 公开 CDP 端点。你可以将 Clawdbot 浏览器配置文件指向 Browserless 区域端点并使用 API 密钥进行认证。

示例：
```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "https://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00"
      }
    }
  }
}
```

注意：
- 将 `<BROWSERLESS_API_KEY>` 替换为你的真实 Browserless 令牌。
- 选择与你的 Browserless 账户匹配的区域端点（参见他们的文档）。

### 在浏览器机器上运行控制服务器

运行独立浏览器控制服务器（当 Gateway 是远程时推荐）：

```bash
# 在运行 Chrome/Brave/Edge 的机器上
clawdbot browser serve --bind <browser-host> --port 18791 --token <token>
```

然后将你的 Gateway 指向它：

```json5
{
  browser: {
    enabled: true,
    controlUrl: "http://<browser-host>:18791",

    // 选项 A（推荐）：在 Gateway 上保持令牌在 env 中
    // （避免将密钥写入配置文件）
    // controlToken: "<token>"
  }
}
```

并在 Gateway 环境中设置认证令牌：

```bash
export CLAWDBOT_BROWSER_CONTROL_TOKEN="<token>"
```

选项 B：将令牌存储在 Gateway 配置中（相同的共享令牌）：

```json5
{
  browser: {
    enabled: true,
    controlUrl: "http://<browser-host>:18791",
    controlToken: "<token>"
  }
}
```

## 安全

本节涵盖用于代理浏览器自动化的**浏览器控制服务器**（`browser.controlUrl`）。

关键想法：
- 将浏览器控制服务器视为管理 API：**仅私有网络**。
- 当服务器可以从机器外访问时，始终使用**令牌认证**。
- 首选**仅 Tailnet**连接而不是 LAN 暴露。

### 令牌（什么与什么共享？）

- `browser.controlToken` / `CLAWDBOT_BROWSER_CONTROL_TOKEN` **仅**用于对 `browser.controlUrl` 的浏览器控制 HTTP 请求进行认证。
- 它**不是** Gateway 令牌（`gateway.auth.token`）也**不是**节点配对令牌。
- 你*可以*重用相同的字符串值，但最好将它们分开以减少爆炸半径。

### 绑定（不要意外暴露到你的 LAN）

推荐：
- 将 `clawdbot browser serve` 绑定到 loopback（`127.0.0.1`）并通过 Tailscale 发布它。
- 或仅绑定到 Tailnet IP（永远不要 `0.0.0.0`）并要求令牌。

避免：
- `--bind 0.0.0.0`（LAN 可见）。即使有令牌认证，流量也是纯 HTTP，除非你还添加 TLS。

### TLS / HTTPS（推荐方法：在前端终止）

这里的最佳实践：保持 `clawdbot browser serve` 在 HTTP 上并在前端终止 TLS。

如果你已经在使用 Tailscale，你有两个好选项：

1) **仅 Tailnet，仍为 HTTP**（传输由 Tailscale 加密）：
- 保持 `controlUrl` 为 `http://…` 但确保它仅可通过你的 tailnet 访问。

2) **通过 Tailscale 提供 HTTPS**（良好的 UX：`https://…` URL）：

```bash
# 在浏览器机器上
clawdbot browser serve --bind 127.0.0.1 --port 18791 --token <token>
tailscale serve https / http://127.0.0.1:18791
```

然后将你的 Gateway 配置 `browser.controlUrl` 设置为 HTTPS URL（MagicDNS/ts.net）并继续使用相同的令牌。

注意：
- **不要**使用 Tailscale Funnel，除非你明确想要使端点公开。
- 有关 Tailnet 设置/背景，请参见 [Gateway web 界面](/web/index）和 [Gateway CLI](/cli/gateway）。

## 配置文件（多浏览器）

Clawdbot 支持多个命名配置文件（路由配置）。配置文件可以是：
- **clawd 管理**：具有自己的用户数据目录 + CDP 端口的专用基于 Chromium 的浏览器实例
- **远程**：显式 CDP URL（在其他地方运行的基于 Chromium 的浏览器）
- **扩展中继**：通过本地中继 + Chrome 扩展的现有 Chrome 标签页

默认值：
- 如果缺失，`clawd` 配置文件会自动创建。
- `chrome` 配置文件是为 Chrome 扩展中继内置的（默认指向 `http://127.0.0.1:18792`）。
- 本地 CDP 端口默认从 **18800–18899** 分配。
- 删除配置文件会将其本地数据目录移动到垃圾箱。

所有控制端点接受 `?profile=<name>`；CLI 使用 `--browser-profile`。

## Chrome 扩展中继（使用你现有的 Chrome）

Clawdbot 还可以通过本地 CDP 中继 + Chrome 扩展驱动**你现有的 Chrome 标签页**（无单独的"clawd" Chrome 实例）。

完整指南：[Chrome 扩展](/tools/chrome-extension）

流程：
- 你运行**浏览器控制服务器**（同一机器上的 Gateway，或 `clawdbot browser serve`）。
- 本地**中继服务器**在 loopback `cdpUrl` 上监听（默认：`http://127.0.0.1:18792`）。
- 你在要控制的标签页上点击 **Clawdbot Browser Relay** 扩展图标以附加（它不会自动附加）。
- 代理通过选择正确的配置文件，通过正常的 `browser` 工具控制该标签页。

如果 Gateway 与 Chrome 在同一机器上运行（默认设置），你通常**不需要** `clawdbot browser serve`。
仅在 Gateway 在其他地方运行时使用 `browser serve`（远程模式）。

### 沙箱化会话

如果代理会话被沙箱化，`browser` 工具可能默认为 `target="sandbox"`（沙箱浏览器）。
Chrome 扩展中继接管需要主机浏览器控制，因此要么：
- 运行未沙箱化的会话，或
- 设置 `agents.defaults.sandbox.browser.allowHostControl: true` 并在调用工具时使用 `target="host"`。

### 设置

1) 加载扩展（dev/unpacked）：

```bash
clawdbot browser extension install
```

- Chrome → `chrome://extensions` → 启用"开发者模式"
- "加载解压缩的扩展" → 选择 `clawdbot browser extension path` 打印的目录
- 固定扩展，然后在要控制的标签页上点击它（徽章显示 `ON`）。

2) 使用它：
- CLI：`clawdbot browser --browser-profile chrome tabs`
- 代理工具：`browser` 与 `profile="chrome"`

可选：如果你想要不同的名称或中继端口，创建你自己的配置文件：

```bash
clawdbot browser create-profile \
  --name my-chrome \
  --driver extension \
  --cdp-url http://127.0.0.1:18792 \
  --color "#00AA00"
```

注意：
- 此模式依赖 Playwright-on-CDP 进行大多数操作（截图/快照/操作）。
- 通过再次点击扩展图标分离。

## 隔离保证

- **专用用户数据目录**：永远不接触你的个人浏览器配置文件。
- **专用端口**：避免 `9222` 以防止与开发工作流冲突。
- **确定性标签页控制**：通过 `targetId` 定位标签页，而不是"最后一个标签页"。

## 浏览器选择

在本地启动时，Clawdbot 选择第一个可用的：
1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

你可以使用 `browser.executablePath` 覆盖。

平台：
- macOS：检查 `/Applications` 和 `~/Applications`。
- Linux：查找 `google-chrome`、`brave`、`microsoft-edge`、`chromium` 等。
- Windows：检查常见安装位置。

## 控制 API（可选）

如果你想直接集成，浏览器控制服务器公开小型 HTTP API：

- 状态/启动/停止：`GET /`、`POST /start`、`POST /stop`
- 标签页：`GET /tabs`、`POST /tabs/open`、`POST /tabs/focus`、`DELETE /tabs/:targetId`
- 快照/截图：`GET /snapshot`、`POST /screenshot`
- 操作：`POST /navigate`、`POST /act`
- 钩子：`POST /hooks/file-chooser`、`POST /hooks/dialog`
- 下载：`POST /download`、`POST /wait/download`
- 调试：`GET /console`、`POST /pdf`
- 调试：`GET /errors`、`GET /requests`、`POST /trace/start`、`POST /trace/stop`、`POST /highlight`
- 网络：`POST /response/body`
- 状态：`GET /cookies`、`POST /cookies/set`、`POST /cookies/clear`
- 状态：`GET /storage/:kind`、`POST /storage/:kind/set`、`POST /storage/:kind/clear`
- 设置：`POST /set/offline`、`POST /set/headers`、`POST /set/credentials`、`POST /set/geolocation`、`POST /set/media`、`POST /set/timezone`、`POST /set/locale`、`POST /set/device`

所有端点接受 `?profile=<name>`。

### Playwright 要求

某些功能（navigate/act/AI 快照/角色快照、元素截图、PDF）需要 Playwright。如果未安装 Playwright，这些端点返回清晰的 501 错误。ARIA 快照和基本截图仍适用于 clawd 管理的 Chrome。
对于 Chrome 扩展中继驱动程序，ARIA 快照和截图需要 Playwright。

如果你看到 `Playwright is not available in this gateway build`，安装完整的 Playwright 包（不是 `playwright-core`）并重启网关，或使用浏览器支持重新安装 Clawdbot。

## 工作原理（内部）

高级流程：
- 小型**控制服务器**接受 HTTP 请求。
- 它通过 **CDP** 连接到基于 Chromium 的浏览器（Chrome/Brave/Edge/Chromium）。
- 对于高级操作（点击/输入/快照/PDF），它在 CDP 之上使用 **Playwright**。
- 当缺少 Playwright 时，仅非 Playwright 操作可用。

此设计使代理保持在稳定、确定性的界面上，同时让你交换本地/远程浏览器和配置文件。

## CLI 快速参考

所有命令接受 `--browser-profile <name>` 以定位特定配置文件。
所有命令还接受 `--json` 以获取机器可读输出（稳定负载）。

基础：
- `clawdbot browser status`
- `clawdbot browser start`
- `clawdbot browser stop`
- `clawdbot browser tabs`
- `clawdbot browser tab`
- `clawdbot browser tab new`
- `clawdbot browser tab select 2`
- `clawdbot browser tab close 2`
- `clawdbot browser open https://example.com`
- `clawdbot browser focus abcd1234`
- `clawdbot browser close abcd1234`

检查：
- `clawdbot browser screenshot`
- `clawdbot browser screenshot --full-page`
- `clawdbot browser screenshot --ref 12`
- `clawdbot browser screenshot --ref e12`
- `clawdbot browser snapshot`
- `clawdbot browser snapshot --format aria --limit 200`
- `clawdbot browser snapshot --interactive --compact --depth 6`
- `clawdbot browser snapshot --efficient`
- `clawdbot browser snapshot --labels`
- `clawdbot browser snapshot --selector "#main" --interactive`
- `clawdbot browser snapshot --frame "iframe#main" --interactive`
- `clawdbot browser console --level error`
- `clawdbot browser errors --clear`
- `clawdbot browser requests --filter api --clear`
- `clawdbot browser pdf`
- `clawdbot browser responsebody "**/api" --max-chars 5000`

操作：
- `clawdbot browser navigate https://example.com`
- `clawdbot browser resize 1280 720`
- `clawdbot browser click 12 --double`
- `clawdbot browser click e12 --double`
- `clawdbot browser type 23 "hello" --submit`
- `clawdbot browser press Enter`
- `clawdbot browser hover 44`
- `clawdbot browser scrollintoview e12`
- `clawdbot browser drag 10 11`
- `clawdbot browser select 9 OptionA OptionB`
- `clawdbot browser download e12 /tmp/report.pdf`
- `clawdbot browser waitfordownload /tmp/report.pdf`
- `clawdbot browser upload /tmp/file.pdf`
- `clawdbot browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'`
- `clawdbot browser dialog --accept`
- `clawdbot browser wait --text "Done"`
- `clawdbot browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"`
- `clawdbot browser evaluate --fn '(el) => el.textContent' --ref 7`
- `clawdbot browser highlight e12`
- `clawdbot browser trace start`
- `clawdbot browser trace stop`

状态：
- `clawdbot browser cookies`
- `clawdbot browser cookies set session abc123 --url "https://example.com"`
- `clawdbot browser cookies clear`
- `clawdbot browser storage local get`
- `clawdbot browser storage local set theme dark`
- `clawdbot browser storage session clear`
- `clawdbot browser set offline on`
- `clawdbot browser set headers --json '{"X-Debug":"1"}'`
- `clawdbot browser set credentials user pass`
- `clawdbot browser set credentials --clear`
- `clawdbot browser set geo 37.7749 -122.4194 --origin "https://example.com"`
- `clawdbot browser set geo --clear`
- `clawdbot browser set media dark`
- `clawdbot browser set timezone America/New_York`
- `clawdbot browser set locale en-US`
- `clawdbot browser set device "iPhone 14"`

注意：
- `upload` 和 `dialog` 是**准备**调用；在触发选择器/对话框的点击/按下之前运行它们。
- `upload` 还可以通过 `--input-ref` 或 `--element` 直接设置文件输入。
- `snapshot`：
  - `--format ai`（安装 Playwright 时默认）：返回带有数字引用的 AI 快照（`aria-ref="<n>"`）。
  - `--format aria`：返回可访问性树（无引用；仅检查）。
  - `--efficient`（或 `--mode efficient`）：紧凑角色快照预设（interactive + compact + depth + 较低的 maxChars）。
  - 配置默认（仅工具/CLI）：设置 `browser.snapshotDefaults.mode: "efficient"` 以在调用者不传递模式时使用高效快照（参见 [Gateway 配置](/gateway/configuration#browser-clawd-managed-browser））。
  - 角色快照选项（`--interactive`、`--compact`、`--depth`、`--selector`）强制基于角色的快照，带有像 `ref=e12` 这样的引用。
  - `--frame "<iframe selector>"` 将角色快照作用域到 iframe（与像 `e12` 这样的角色引用配对）。
  - `--interactive` 输出平面、易于选择的交互元素列表（最适合驱动操作）。
  - `--labels` 添加仅视口截图，带有叠加的引用标签（打印 `MEDIA:<path>`）。
- `click`/`type`/等需要来自 `snapshot` 的 `ref`（数字 `12` 或角色引用 `e12`）。
  操作故意不支持 CSS 选择器。

## 快照和引用

Clawdbot 支持两种"快照"样式：

- **AI 快照（数字引用）**：`clawdbot browser snapshot`（默认；`--format ai`）
  - 输出：包含数字引用的文本快照。
  - 操作：`clawdbot browser click 12`、`clawdbot browser type 23 "hello"`。
  - 在内部，引用通过 Playwright 的 `aria-ref` 解析。

- **角色快照（角色引用如 `e12`）**：`clawdbot browser snapshot --interactive`（或 `--compact`、`--depth`、`--selector`、`--frame`）
  - 输出：带有 `[ref=e12]`（和可选的 `[nth=1]`）的基于角色的列表/树。
  - 操作：`clawdbot browser click e12`、`clawdbot browser highlight e12`。
  - 在内部，引用通过 `getByRole(...)` 解析（加上用于重复的 `nth()`）。
  - 添加 `--labels` 以包括带有叠加 `e12` 标签的视口截图。

引用行为：
- 引用**在导航之间不稳定**；如果某些内容失败，重新运行 `snapshot` 并使用新的引用。
- 如果角色快照是用 `--frame` 拍摄的，角色引用作用域到该 iframe，直到下一个角色快照。

## 等待增强

你可以等待的不仅仅是时间/文本：

- 等待 URL（Playwright 支持 glob）：
  - `clawdbot browser wait --url "**/dash"`
- 等待加载状态：
  - `clawdbot browser wait --load networkidle`
- 等待 JS 谓词：
  - `clawdbot browser wait --fn "window.ready===true"`
- 等待选择器变为可见：
  - `clawdbot browser wait "#main"`

这些可以组合：

```bash
clawdbot browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## 调试工作流

当操作失败时（例如"不可见"、"严格模式违规"、"被覆盖"）：

1. `clawdbot browser snapshot --interactive`
2. 使用 `click <ref>` / `type <ref>`（在交互模式下首选角色引用）
3. 如果仍然失败：`clawdbot browser highlight <ref>` 以查看 Playwright 正在定位的内容
4. 如果页面行为异常：
   - `clawdbot browser errors --clear`
   - `clawdbot browser requests --filter api --clear`
5. 对于深度调试：记录跟踪：
   - `clawdbot browser trace start`
   - 重现问题
   - `clawdbot browser trace stop`（打印 `TRACE:<path>`）

## JSON 输出

`--json` 用于脚本和结构化工具。

示例：

```bash
clawdbot browser status --json
clawdbot browser snapshot --interactive --json
clawdbot browser requests --filter api --json
clawdbot browser cookies --json
```

JSON 中的角色快照包括 `refs` 加上小的 `stats` 块（行/字符/引用/交互），以便工具可以推理负载大小和密度。

## 状态和环境旋钮

这些对于"使站点表现得像 X"工作流很有用：

- Cookies：`cookies`、`cookies set`、`cookies clear`
- 存储：`storage local|session get|set|clear`
- 离线：`set offline on|off`
- 标头：`set headers --json '{"X-Debug":"1"}'`（或 `--clear`）
- HTTP basic 认证：`set credentials user pass`（或 `--clear`）
- 地理位置：`set geo <lat> <lon> --origin "https://example.com"`（或 `--clear`）
- 媒体：`set media dark|light|no-preference|none`
- 时区 / 区域设置：`set timezone ...`、`set locale ...`
- 设备 / 视口：
  - `set device "iPhone 14"`（Playwright 设备预设）
  - `set viewport 1280 720`

## 安全和隐私

- clawd 浏览器配置文件可能包含已登录的会话；将其视为敏感。
- 有关登录和反机器人说明（X/Twitter 等），请参见 [浏览器登录 + X/Twitter 发布](/tools/browser-login）。
- 除非你故意暴露服务器，否则保持控制 URL 仅 loopback。
- 远程 CDP 端点很强大；对它们进行隧道和保护。

## 故障排除

对于 Linux 特定问题（特别是 snap Chromium），请参见 [浏览器故障排除](/tools/browser-linux-troubleshooting）。

## 代理工具 + 控制如何工作

代理获得**一个工具**用于浏览器自动化：
- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

它如何映射：
- `browser snapshot` 返回稳定的 UI 树（AI 或 ARIA）。
- `browser act` 使用快照 `ref` ID 来点击/输入/拖拽/选择。
- `browser screenshot` 捕获像素（完整页面或元素）。
- `browser` 接受：
  - `profile` 以选择命名浏览器配置文件（主机或远程控制服务器）。
  - `target`（`sandbox` | `host` | `custom`）以选择浏览器所在的位置。
  - `controlUrl` 隐式设置 `target: "custom"`（远程控制服务器）。
  - 在沙箱化会话中，`target: "host"` 需要 `agents.defaults.sandbox.browser.allowHostControl=true`。
  - 如果省略 `target`：沙箱化会话默认为 `sandbox`，非沙箱会话默认为 `host`。
  - 沙箱允许列表可以将 `target: "custom"` 限制到特定 URL/主机/端口。
  - 默认值：允许列表未设置（无限制），沙箱主机控制被禁用。

这使代理保持确定性并避免脆弱的选择器。
