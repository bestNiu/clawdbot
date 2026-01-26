---
summary: "Repository scripts: purpose, scope, and safety notes"
read_when:
  - Running scripts from the repo
  - Adding or changing scripts under ./scripts
---
# 脚本

`scripts/` 目录包含用于本地工作流和操作任务的助手脚本。
当任务明显与脚本相关时使用这些；否则首选 CLI。

## 约定

- 脚本是**可选的**，除非在文档或发布清单中引用。
- 当 CLI 界面存在时首选它们（示例：认证监控使用 `clawdbot models status --check`）。
- 假设脚本是主机特定的；在新机器上运行之前先阅读它们。

## Git 钩子

- `scripts/setup-git-hooks.js`：在 git 仓库内为 `core.hooksPath` 进行尽力而为的设置。
- `scripts/format-staged.js`：用于暂存 `src/` 和 `test/` 文件的预提交格式化器。

## 认证监控脚本

认证监控脚本在此处记录：
[/automation/auth-monitoring](/automation/auth-monitoring)

## 添加脚本时

- 保持脚本专注且有文档。
- 在相关文档中添加简短条目（如果缺失则创建一个）。
