---
summary: "Sub-agents: spawning isolated agent runs that announce results back to the requester chat"
read_when:
  - You want background/parallel work via the agent
  - You are changing sessions_spawn or sub-agent tool policy
---

# 子代理

子代理是从现有代理运行生成的后台代理运行。它们在自己的会话中运行（`agent:<agentId>:subagent:<uuid>`），并在完成时**通知**其结果回请求者聊天通道。

## 斜杠命令

使用 `/subagents` 检查或控制**当前会话**的子代理运行：
- `/subagents list`
- `/subagents stop <id|#|all>`
- `/subagents log <id|#> [limit] [tools]`
- `/subagents info <id|#>`
- `/subagents send <id|#> <message>`

`/subagents info` 显示运行元数据（状态、时间戳、会话 id、转录本路径、清理）。

主要目标：
- 并行化"研究 / 长任务 / 慢工具"工作而不阻塞主运行。
- 默认保持子代理隔离（会话分离 + 可选沙箱化）。
- 保持工具界面难以滥用：子代理默认**不**获得会话工具。
- 避免嵌套扇出：子代理无法生成子代理。

成本说明：每个子代理有**自己的**上下文和 token 使用。对于繁重或重复的任务，为子代理设置更便宜的模型，并将主代理保持在更高质量的模型上。
你可以通过 `agents.defaults.subagents.model` 或每个代理覆盖来配置此。

## 工具

使用 `sessions_spawn`：
- 启动子代理运行（`deliver: false`，全局通道：`subagent`）
- 然后运行通知步骤并将通知回复发布到请求者聊天通道
- 默认模型：继承调用者，除非你设置 `agents.defaults.subagents.model`（或每个代理 `agents.list[].subagents.model`）；显式 `sessions_spawn.model` 仍然获胜。

工具参数：
- `task`（必需）
- `label?`（可选）
- `agentId?`（可选；如果允许，在另一个代理 id 下生成）
- `model?`（可选；覆盖子代理模型；无效值被跳过，子代理在默认模型上运行，工具结果中带有警告）
- `thinking?`（可选；覆盖子代理运行的思考级别）
- `runTimeoutSeconds?`（默认 `0`；当设置时，子代理运行在 N 秒后中止）
- `cleanup?`（`delete|keep`，默认 `keep`）

允许列表：
- `agents.list[].subagents.allowAgents`：可以通过 `agentId` 定位的代理 id 列表（`["*"]` 以允许任何）。默认：仅请求者代理。

发现：
- 使用 `agents_list` 查看当前允许用于 `sessions_spawn` 的代理 id。

自动归档：
- 子代理会话在 `agents.defaults.subagents.archiveAfterMinutes`（默认：60）后自动归档。
- 归档使用 `sessions.delete` 并将转录本重命名为 `*.deleted.<timestamp>`（同一文件夹）。
- `cleanup: "delete"` 在通知后立即归档（仍通过重命名保留转录本）。
- 自动归档是尽力而为；如果网关重启，待处理的计时器会丢失。
- `runTimeoutSeconds` **不**自动归档；它仅停止运行。会话保持直到自动归档。

## 认证

子代理认证通过**代理 id**解析，而不是会话类型：
- 子代理会话键是 `agent:<agentId>:subagent:<uuid>`。
- 认证存储从该代理的 `agentDir` 加载。
- 主代理的认证配置文件作为**回退**合并；代理配置文件在冲突时覆盖主配置文件。

注意：合并是加法的，因此主配置文件始终可用作回退。每个代理的完全隔离认证尚不支持。

## 通知

子代理通过通知步骤报告回来：
- 通知步骤在子代理会话内运行（不是请求者会话）。
- 如果子代理完全回复 `ANNOUNCE_SKIP`，则不发布任何内容。
- 否则，通知回复通过后续 `agent` 调用（`deliver=true`）发布到请求者聊天通道。
- 通知回复在可用时保留线程/主题路由（Slack 线程、Telegram 主题、Matrix 线程）。
- 通知消息被规范化为稳定模板：
  - `Status:` 从运行结果派生（`success`、`error`、`timeout` 或 `unknown`）。
  - `Result:` 来自通知步骤的摘要内容（如果缺失则为 `(not available)`）。
  - `Notes:` 错误详细信息和其他有用上下文。
- `Status` 不是从模型输出推断的；它来自运行时结果信号。

通知负载在末尾包括统计行（即使被包装）：
- 运行时（例如，`runtime 5m12s`）
- Token 使用（输入/输出/总计）
- 当配置模型定价时的估计成本（`models.providers.*.models[].cost`）
- `sessionKey`、`sessionId` 和转录本路径（以便主代理可以通过 `sessions_history` 获取历史或检查磁盘上的文件）

## 工具策略（子代理工具）

默认情况下，子代理获得**除会话工具外的所有工具**：
- `sessions_list`
- `sessions_history`
- `sessions_send`
- `sessions_spawn`

通过配置覆盖：

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 1
      }
    }
  },
  tools: {
    subagents: {
      tools: {
        // deny 获胜
        deny: ["gateway", "cron"],
        // 如果设置了 allow，它变为仅允许（deny 仍然获胜）
        // allow: ["read", "exec", "process"]
      }
    }
  }
}
```

## 并发

子代理使用专用的进程内队列通道：
- 通道名称：`subagent`
- 并发：`agents.defaults.subagents.maxConcurrent`（默认 `8`）

## 停止

- 在请求者聊天中发送 `/stop` 中止请求者会话并停止从中生成的任何活动子代理运行。

## 限制

- 子代理通知是**尽力而为**。如果网关重启，待处理的"通知回来"工作会丢失。
- 子代理仍共享相同的网关进程资源；将 `maxConcurrent` 视为安全阀。
- `sessions_spawn` 始终是非阻塞的：它立即返回 `{ status: "accepted", runId, childSessionKey }`。
- 子代理上下文仅注入 `AGENTS.md` + `TOOLS.md`（无 `SOUL.md`、`IDENTITY.md`、`USER.md`、`HEARTBEAT.md` 或 `BOOTSTRAP.md`）。
