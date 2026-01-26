# 创建自定义技能 🛠

Clawdbot 设计为易于扩展。"技能"是向助手添加新功能的主要方式。

## 什么是技能？
技能是一个包含 `SKILL.md` 文件的目录（向 LLM 提供指令和工具定义），可选地包含一些脚本或资源。

## 逐步：你的第一个技能

### 1. 创建目录
技能位于你的工作区，通常是 `~/clawd/skills/`。为你的技能创建一个新文件夹：
```bash
mkdir -p ~/clawd/skills/hello-world
```

### 2. 定义 `SKILL.md`
在该目录中创建 `SKILL.md` 文件。此文件使用 YAML frontmatter 作为元数据，使用 Markdown 作为指令。

```markdown
---
name: hello_world
description: A simple skill that says hello.
---

# Hello World Skill
When the user asks for a greeting, use the `echo` tool to say "Hello from your custom skill!".
```

### 3. 添加工具（可选）
你可以在 frontmatter 中定义自定义工具，或指示代理使用现有系统工具（如 `bash` 或 `browser`）。

### 4. 刷新 Clawdbot
要求你的代理"刷新技能"或重启网关。Clawdbot 将发现新目录并索引 `SKILL.md`。

## 最佳实践
- **简洁**：指示模型*做什么*，而不是如何成为 AI。
- **安全第一**：如果你的技能使用 `bash`，确保提示不允许来自不受信任用户输入的任意命令注入。
- **本地测试**：使用 `clawdbot agent --message "use my new skill"` 进行测试。

## 共享技能
你也可以浏览技能并贡献到 [ClawdHub](https://clawdhub.com）。
