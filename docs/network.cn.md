---
summary: "Network hub: gateway surfaces, pairing, discovery, and security"
read_when:
  - You need the network architecture + security overview
  - You are debugging local vs tailnet access or pairing
  - You want the canonical list of networking docs
---
# 网络中心

此中心链接了 Clawdbot 如何在 localhost、LAN 和 tailnet 上连接、配对和保护设备的核心文档。

## 核心模型

- [Gateway 架构](/concepts/architecture)
- [Gateway 协议](/gateway/protocol)
- [Gateway 运行手册](/gateway)
- [Web 界面 + 绑定模式](/web)

## 配对 + 身份

- [配对概述（私信 + 节点）](/start/pairing)
- [Gateway 拥有的节点配对](/gateway/pairing)
- [设备 CLI（配对 + 令牌轮换）](/cli/devices)
- [配对 CLI（私信批准）](/cli/pairing)

本地信任：
- 本地连接（环回或网关主机自己的 tailnet 地址）可以自动批准配对，以保持同主机 UX 顺畅。
- 非本地 tailnet/LAN 客户端仍需要明确的配对批准。

## 发现 + 传输

- [发现和传输](/gateway/discovery)
- [Bonjour / mDNS](/gateway/bonjour)
- [远程访问（SSH）](/gateway/remote)
- [Tailscale](/gateway/tailscale)

## 节点 + 传输

- [节点概述](/nodes)
- [桥接协议（传统节点）](/gateway/bridge-protocol)
- [节点运行手册：iOS](/platforms/ios)
- [节点运行手册：Android](/platforms/android)

## 安全

- [安全概述](/gateway/security)
- [Gateway 配置参考](/gateway/configuration)
- [故障排除](/gateway/troubleshooting)
- [Doctor](/gateway/doctor)
