---
summary: "Manual logins for browser automation + X/Twitter posting"
read_when:
  - You need to log into sites for browser automation
  - You want to post updates to X/Twitter
---

# 浏览器登录 + X/Twitter 发布

## 手动登录（推荐）

当站点需要登录时，在**主机**浏览器配置文件（clawd 浏览器）中**手动登录**。

**不要**给模型你的凭据。自动登录经常触发反机器人防御，并可能锁定账户。

返回主浏览器文档：[浏览器](/tools/browser）。

## 使用哪个 Chrome 配置文件？

Clawdbot 控制**专用 Chrome 配置文件**（名为 `clawd`，橙色 UI）。这与你的日常浏览器配置文件分开。

两种简单的方法来访问它：

1) **要求代理打开浏览器**，然后自己登录。
2) **通过 CLI 打开它**：

```bash
clawdbot browser start
clawdbot browser open https://x.com
```

如果你有多个配置文件，传递 `--browser-profile <name>`（默认是 `clawd`）。

## X/Twitter：推荐流程

- **读取/搜索/线程**：使用 **bird** CLI 技能（无浏览器，稳定）。
  - 仓库：https://github.com/steipete/bird
- **发布更新**：使用**主机**浏览器（手动登录）。

## 沙箱化 + 主机浏览器访问

沙箱化浏览器会话**更可能**触发机器人检测。对于 X/Twitter（和其他严格站点），首选**主机**浏览器。

如果代理被沙箱化，浏览器工具默认为沙箱。要允许主机控制：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        browser: {
          allowHostControl: true
        }
      }
    }
  }
}
```

然后定位主机浏览器：

```bash
clawdbot browser open https://x.com --browser-profile clawd --target host
```

或为发布更新的代理禁用沙箱化。
