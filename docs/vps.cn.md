---
summary: "VPS hosting hub for Clawdbot (Railway/Fly/Hetzner/exe.dev)"
read_when:
  - You want to run the Gateway in the cloud
  - You need a quick map of VPS/hosting guides
---
# VPS 托管

此中心链接到支持的 VPS/托管指南，并解释云部署在高级别上的工作原理。

## 选择提供者

- **Railway**（一键 + 浏览器设置）：[Railway](/railway)
- **Fly.io**：[Fly.io](/platforms/fly)
- **Hetzner (Docker)**：[Hetzner](/platforms/hetzner)
- **GCP (Compute Engine)**：[GCP](/platforms/gcp)
- **exe.dev**（VM + HTTPS 代理）：[exe.dev](/platforms/exe-dev)
- **AWS (EC2/Lightsail/free tier)**：也很好用。视频指南：
  https://x.com/techfrenAJ/status/2014934471095812547

## 云设置如何工作

- **Gateway 在 VPS 上运行**并拥有状态 + 工作区。
- 你通过**控制 UI** 或 **Tailscale/SSH** 从笔记本电脑/手机连接。
- 将 VPS 视为真相来源并**备份**状态 + 工作区。

远程访问：[Gateway 远程](/gateway/remote)  
平台中心：[平台](/platforms)

## 将节点与 VPS 一起使用

你可以将 Gateway 保留在云中，并在本地设备（Mac/iOS/Android/无头）上配对**节点**。节点提供本地屏幕/相机/canvas 和 `system.run` 功能，而 Gateway 保持在云中。

文档：[节点](/nodes)、[节点 CLI](/cli/nodes)
