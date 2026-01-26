---
summary: "Testing kit: unit/e2e/live suites, Docker runners, and what each test covers"
read_when:
  - Running tests locally or in CI
  - Adding regressions for model/provider bugs
  - Debugging gateway + agent behavior
---

# 测试

Clawdbot 有三个 Vitest 套件（单元/集成、e2e、live）和一组小型 Docker 运行器。

本文档是“我们如何测试”指南：
- 每个套件涵盖的内容（以及它故意*不*涵盖的内容）
- 常见工作流（本地、推送前、调试）要运行的命令
- Live 测试如何发现凭据并选择模型/提供者
- 如何为真实世界的模型/提供者问题添加回归

## 快速开始

大多数时候：
- 完整门控（推送前预期）：`pnpm lint && pnpm build && pnpm test`

当你接触测试或想要额外信心时：
- 覆盖率门控：`pnpm test:coverage`
- E2E 套件：`pnpm test:e2e`

调试真实提供者/模型时（需要真实凭据）：
- Live 套件（模型 + 网关工具/图像探测）：`pnpm test:live`

提示：当你只需要一个失败的案例时，最好通过下面描述的 allowlist 环境变量缩小 live 测试。

## 测试套件（在哪里运行什么）

将套件视为“增加真实性”（以及增加不稳定性/成本）：

### 单元 / 集成（默认）

- 命令：`pnpm test`
- 配置：`vitest.config.ts`
- 文件：`src/**/*.test.ts`
- 范围：
  - 纯单元测试
  - 进程内集成测试（网关认证、路由、工具、解析、配置）
  - 已知错误的确定性回归
- 期望：
  - 在 CI 中运行
  - 不需要真实密钥
  - 应该快速且稳定

### E2E（网关冒烟）

- 命令：`pnpm test:e2e`
- 配置：`vitest.e2e.config.ts`
- 文件：`src/**/*.e2e.test.ts`
- 范围：
  - 多实例网关端到端行为
  - WebSocket/HTTP 界面、节点配对和更重的网络
- 期望：
  - 在 CI 中运行（在管道中启用时）
  - 不需要真实密钥
  - 比单元测试有更多移动部件（可能更慢）

### Live（真实提供者 + 真实模型）

- 命令：`pnpm test:live`
- 配置：`vitest.live.config.ts`
- 文件：`src/**/*.live.test.ts`
- 默认：**启用**通过 `pnpm test:live`（设置 `CLAWDBOT_LIVE_TEST=1`）
- 范围：
  - “这个提供者/模型今天是否真的与真实凭据一起工作？”
  - 捕获提供者格式更改、工具调用怪癖、认证问题和速率限制行为
- 期望：
  - 设计上不是 CI 稳定的（真实网络、真实提供者策略、配额、中断）
  - 花费金钱 / 使用速率限制
  - 最好运行缩小的子集而不是“所有内容”
  - Live 运行将源 `~/.profile` 以获取缺少的 API 密钥
  - Anthropic 密钥轮换：设置 `CLAWDBOT_LIVE_ANTHROPIC_KEYS="sk-...,sk-..."`（或 `CLAWDBOT_LIVE_ANTHROPIC_KEY=sk-...`）或多个 `ANTHROPIC_API_KEY*` 变量；测试将在速率限制时重试

## 我应该运行哪个套件？

使用此决策表：
- 编辑逻辑/测试：运行 `pnpm test`（如果你更改了很多，则运行 `pnpm test:coverage`）
- 接触网关网络 / WS 协议 / 配对：添加 `pnpm test:e2e`
- 调试“我的机器人已关闭”/ 提供者特定故障 / 工具调用：运行缩小的 `pnpm test:live`

## Live：模型冒烟（配置文件密钥）

Live 测试分为两层，以便我们可以隔离故障：
- “直接模型”告诉我们提供者/模型是否可以使用给定密钥回答。
- “网关冒烟”告诉我们完整的网关+代理管道是否适用于该模型（会话、历史、工具、沙箱策略等）。

### 第 1 层：直接模型完成（无网关）

- 测试：`src/agents/models.profiles.live.test.ts`
- 目标：
  - 枚举发现的模型
  - 使用 `getApiKeyForModel` 选择你有凭据的模型
  - 每个模型运行一个小完成（并在需要时进行定向回归）
- 如何启用：
  - `pnpm test:live`（或如果直接调用 Vitest，则 `CLAWDBOT_LIVE_TEST=1`）
  - 设置 `CLAWDBOT_LIVE_MODELS=modern`（或 `all`，modern 的别名）以实际运行此套件；否则跳过以保持 `pnpm test:live` 专注于网关冒烟
- 如何选择模型：
  - `CLAWDBOT_LIVE_MODELS=modern` 运行现代 allowlist（Opus/Sonnet/Haiku 4.5、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.1、Grok 4）
  - `CLAWDBOT_LIVE_MODELS=all` 是现代 allowlist 的别名
  - 或 `CLAWDBOT_LIVE_MODELS="openai/gpt-5.2,anthropic/claude-opus-4-5,..."`（逗号 allowlist）
- 如何选择提供者：
  - `CLAWDBOT_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"`（逗号 allowlist）
- 密钥来源：
  - 默认：配置文件存储和 env 回退
  - 设置 `CLAWDBOT_LIVE_REQUIRE_PROFILE_KEYS=1` 以强制**仅配置文件存储**
- 为什么存在：
  - 将“提供者 API 已损坏 / 密钥无效”与“网关代理管道已损坏”分开
  - 包含小的、隔离的回归（示例：OpenAI Responses/Codex Responses 推理重放 + 工具调用流）

### 第 2 层：Gateway + 开发代理冒烟（“@clawdbot”实际做什么）

- 测试：`src/gateway/gateway-models.profiles.live.test.ts`
- 目标：
  - 启动进程内网关
  - 创建/修补 `agent:dev:*` 会话（每次运行模型覆盖）
  - 迭代有密钥的模型并断言：
    - “有意义的”响应（无工具）
    - 真实工具调用有效（读取探测）
    - 可选额外工具探测（exec+read 探测）
    - OpenAI 回归路径（仅工具调用 → 后续）保持工作
- 探测详细信息（以便你可以快速解释故障）：
  - `read` 探测：测试在工作区中写入一个 nonce 文件，并要求代理 `read` 它并回显 nonce。
  - `exec+read` 探测：测试要求代理 `exec`-写入一个 nonce 到临时文件，然后 `read` 它回来。
  - 图像探测：测试附加一个生成的 PNG（cat + 随机代码）并期望模型返回 `cat <CODE>`。
  - 实现参考：`src/gateway/gateway-models.profiles.live.test.ts` 和 `src/gateway/live-image-probe.ts`。
- 如何启用：
  - `pnpm test:live`（或如果直接调用 Vitest，则 `CLAWDBOT_LIVE_TEST=1`）
- 如何选择模型：
  - 默认：现代 allowlist（Opus/Sonnet/Haiku 4.5、GPT-5.x + Codex、Gemini 3、GLM 4.7、MiniMax M2.1、Grok 4）
  - `CLAWDBOT_LIVE_GATEWAY_MODELS=all` 是现代 allowlist 的别名
  - 或设置 `CLAWDBOT_LIVE_GATEWAY_MODELS="provider/model"`（或逗号列表）以缩小
- 如何选择提供者（避免“OpenRouter 所有内容”）：
  - `CLAWDBOT_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"`（逗号 allowlist）
- 工具 + 图像探测在此 live 测试中始终开启：
  - `read` 探测 + `exec+read` 探测（工具压力）
  - 当模型宣传图像输入支持时运行图像探测
  - 流程（高级）：
    - 测试生成一个带有“CAT”+ 随机代码的小 PNG（`src/gateway/live-image-probe.ts`）
    - 通过 `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` 发送
    - Gateway 将附件解析为 `images[]`（`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`）
    - 嵌入式代理将多模态用户消息转发到模型
    - 断言：回复包含 `cat` + 代码（OCR 容差：允许小错误）

提示：要查看你可以在机器上测试的内容（以及确切的 `provider/model` ID），请运行：

```bash
clawdbot models list
clawdbot models list --json
```

## Live：Anthropic setup-token 冒烟

- 测试：`src/agents/anthropic.setup-token.live.test.ts`
- 目标：验证 Claude Code CLI setup-token（或粘贴的 setup-token 配置文件）可以完成 Anthropic 提示。
- 启用：
  - `pnpm test:live`（或如果直接调用 Vitest，则 `CLAWDBOT_LIVE_TEST=1`）
  - `CLAWDBOT_LIVE_SETUP_TOKEN=1`
- 令牌源（选择一个）：
  - 配置文件：`CLAWDBOT_LIVE_SETUP_TOKEN_PROFILE=anthropic:setup-token-test`
  - 原始令牌：`CLAWDBOT_LIVE_SETUP_TOKEN_VALUE=sk-ant-oat01-...`
- 模型覆盖（可选）：
  - `CLAWDBOT_LIVE_SETUP_TOKEN_MODEL=anthropic/claude-opus-4-5`

设置示例：

```bash
clawdbot models auth paste-token --provider anthropic --profile-id anthropic:setup-token-test
CLAWDBOT_LIVE_SETUP_TOKEN=1 CLAWDBOT_LIVE_SETUP_TOKEN_PROFILE=anthropic:setup-token-test pnpm test:live src/agents/anthropic.setup-token.live.test.ts
```

## Live：CLI 后端冒烟（Claude Code CLI 或其他本地 CLI）

- 测试：`src/gateway/gateway-cli-backend.live.test.ts`
- 目标：使用本地 CLI 后端验证 Gateway + 代理管道，而不触及你的默认配置。
- 启用：
  - `pnpm test:live`（或如果直接调用 Vitest，则 `CLAWDBOT_LIVE_TEST=1`）
  - `CLAWDBOT_LIVE_CLI_BACKEND=1`
- 默认值：
  - 模型：`claude-cli/claude-sonnet-4-5`
  - 命令：`claude`
  - 参数：`["-p","--output-format","json","--dangerously-skip-permissions"]`
- 覆盖（可选）：
  - `CLAWDBOT_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-opus-4-5"`
  - `CLAWDBOT_LIVE_CLI_BACKEND_MODEL="codex-cli/gpt-5.2-codex"`
  - `CLAWDBOT_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `CLAWDBOT_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","json","--permission-mode","bypassPermissions"]'`
  - `CLAWDBOT_LIVE_CLI_BACKEND_CLEAR_ENV='["ANTHROPIC_API_KEY","ANTHROPIC_API_KEY_OLD"]'`
  - `CLAWDBOT_LIVE_CLI_BACKEND_IMAGE_PROBE=1` 发送真实图像附件（路径注入到提示中）。
  - `CLAWDBOT_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` 将图像文件路径作为 CLI 参数传递，而不是提示注入。
  - `CLAWDBOT_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"`（或 `"list"`）控制当设置 `IMAGE_ARG` 时如何传递图像参数。
  - `CLAWDBOT_LIVE_CLI_BACKEND_RESUME_PROBE=1` 发送第二个回合并验证恢复流。
- `CLAWDBOT_LIVE_CLI_BACKEND_DISABLE_MCP_CONFIG=0` 保持 Claude Code CLI MCP 配置启用（默认使用临时空文件禁用 MCP 配置）。

示例：

```bash
CLAWDBOT_LIVE_CLI_BACKEND=1 \
  CLAWDBOT_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-5" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

### 推荐的 live 配方

窄的、明确的 allowlist 最快且最不稳定：

- 单个模型，直接（无网关）：
  - `CLAWDBOT_LIVE_MODELS="openai/gpt-5.2" pnpm test:live src/agents/models.profiles.live.test.ts`

- 单个模型，网关冒烟：
  - `CLAWDBOT_LIVE_GATEWAY_MODELS="openai/gpt-5.2" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- 跨多个提供者的工具调用：
  - `CLAWDBOT_LIVE_GATEWAY_MODELS="openai/gpt-5.2,anthropic/claude-opus-4-5,google/gemini-3-flash-preview,zai/glm-4.7,minimax/minimax-m2.1" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- Google 焦点（Gemini API 密钥 + Antigravity）：
  - Gemini（API 密钥）：`CLAWDBOT_LIVE_GATEWAY_MODELS="google/gemini-3-flash-preview" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity（OAuth）：`CLAWDBOT_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-5-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

注意：
- `google/...` 使用 Gemini API（API 密钥）。
- `google-antigravity/...` 使用 Antigravity OAuth 桥接（Cloud Code Assist 风格的代理端点）。
- `google-gemini-cli/...` 使用你机器上的本地 Gemini CLI（单独的认证 + 工具怪癖）。
- Gemini API vs Gemini CLI：
  - API：Clawdbot 通过 HTTP 调用 Google 的托管 Gemini API（API 密钥 / 配置文件认证）；这是大多数用户所说的“Gemini”。
  - CLI：Clawdbot 调用本地 `gemini` 二进制文件；它有自己的认证，可能表现不同（流式传输/工具支持/版本偏差）。

## Live：模型矩阵（我们涵盖的内容）

没有固定的“CI 模型列表”（live 是选择加入的），但这些是在有密钥的开发机器上定期涵盖的**推荐**模型。

### 现代冒烟集（工具调用 + 图像）

这是我们希望保持工作的“常见模型”运行：
- OpenAI（非 Codex）：`openai/gpt-5.2`（可选：`openai/gpt-5.1`）
- OpenAI Codex：`openai-codex/gpt-5.2`（可选：`openai-codex/gpt-5.2-codex`）
- Anthropic：`anthropic/claude-opus-4-5`（或 `anthropic/claude-sonnet-4-5`）
- Google（Gemini API）：`google/gemini-3-pro-preview` 和 `google/gemini-3-flash-preview`（避免较旧的 Gemini 2.x 模型）
- Google（Antigravity）：`google-antigravity/claude-opus-4-5-thinking` 和 `google-antigravity/gemini-3-flash`
- Z.AI（GLM）：`zai/glm-4.7`
- MiniMax：`minimax/minimax-m2.1`

使用工具 + 图像运行网关冒烟：
`CLAWDBOT_LIVE_GATEWAY_MODELS="openai/gpt-5.2,openai-codex/gpt-5.2,anthropic/claude-opus-4-5,google/gemini-3-pro-preview,google/gemini-3-flash-preview,google-antigravity/claude-opus-4-5-thinking,google-antigravity/gemini-3-flash,zai/glm-4.7,minimax/minimax-m2.1" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

### 基线：工具调用（Read + 可选 Exec）

每个提供者系列至少选择一个：
- OpenAI：`openai/gpt-5.2`（或 `openai/gpt-5-mini`）
- Anthropic：`anthropic/claude-opus-4-5`（或 `anthropic/claude-sonnet-4-5`）
- Google：`google/gemini-3-flash-preview`（或 `google/gemini-3-pro-preview`）
- Z.AI（GLM）：`zai/glm-4.7`
- MiniMax：`minimax/minimax-m2.1`

可选额外覆盖（最好有）：
- xAI：`xai/grok-4`（或最新可用）
- Mistral：`mistral/`…（选择一个你已启用的“工具”能力模型）
- Cerebras：`cerebras/`…（如果你有访问权限）
- LM Studio：`lmstudio/`…（本地；工具调用取决于 API 模式）

### 视觉：图像发送（附件 → 多模态消息）

在 `CLAWDBOT_LIVE_GATEWAY_MODELS` 中至少包含一个支持图像的模型（Claude/Gemini/OpenAI 视觉能力变体等）以执行图像探测。

### 聚合器 / 备用网关

如果你有启用的密钥，我们还支持通过以下方式测试：
- OpenRouter：`openrouter/...`（数百个模型；使用 `clawdbot models scan` 查找工具+图像能力候选）
- OpenCode Zen：`opencode/...`（通过 `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY` 认证）

你可以在 live 矩阵中包含的更多提供者（如果你有凭据/配置）：
- 内置：`openai`、`openai-codex`、`anthropic`、`google`、`google-vertex`、`google-antigravity`、`google-gemini-cli`、`zai`、`openrouter`、`opencode`、`xai`、`groq`、`cerebras`、`mistral`、`github-copilot`
- 通过 `models.providers`（自定义端点）：`minimax`（云/API），以及任何 OpenAI/Anthropic 兼容代理（LM Studio、vLLM、LiteLLM 等）

提示：不要尝试在文档中硬编码“所有模型”。权威列表是 `discoverModels(...)` 在你的机器上返回的任何内容 + 任何可用的密钥。

## 凭据（永远不提交）

Live 测试以与 CLI 相同的方式发现凭据。实际含义：
- 如果 CLI 工作，live 测试应该找到相同的密钥。
- 如果 live 测试说“无凭据”，请以与调试 `clawdbot models list` / 模型选择相同的方式调试。

- 配置文件存储：`~/.clawdbot/credentials/`（首选；测试中“配置文件密钥”的含义）
- 配置：`~/.clawdbot/clawdbot.json`（或 `CLAWDBOT_CONFIG_PATH`）

如果你想要依赖 env 密钥（例如在你的 `~/.profile` 中导出的），请在 `source ~/.profile` 后运行本地测试，或使用下面的 Docker 运行器（它们可以将 `~/.profile` 挂载到容器中）。

## Deepgram live（音频转录）

- 测试：`src/media-understanding/providers/deepgram/audio.live.test.ts`
- 启用：`DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live src/media-understanding/providers/deepgram/audio.live.test.ts`

## Docker 运行器（可选的“在 Linux 中工作”检查）

这些在仓库 Docker 镜像内运行 `pnpm test:live`，挂载你的本地配置目录和工作区（如果挂载，则源 `~/.profile`）：

- 直接模型：`pnpm test:docker:live-models`（脚本：`scripts/test-live-models-docker.sh`）
- Gateway + 开发代理：`pnpm test:docker:live-gateway`（脚本：`scripts/test-live-gateway-models-docker.sh`）
- 引导向导（TTY，完整脚手架）：`pnpm test:docker:onboard`（脚本：`scripts/e2e/onboard-docker.sh`）
- Gateway 网络（两个容器，WS 认证 + 健康）：`pnpm test:docker:gateway-network`（脚本：`scripts/e2e/gateway-network-docker.sh`）
- 插件（自定义扩展加载 + 注册表冒烟）：`pnpm test:docker:plugins`（脚本：`scripts/e2e/plugins-docker.sh`）

有用的环境变量：

- `CLAWDBOT_CONFIG_DIR=...`（默认：`~/.clawdbot`）挂载到 `/home/node/.clawdbot`
- `CLAWDBOT_WORKSPACE_DIR=...`（默认：`~/clawd`）挂载到 `/home/node/clawd`
- `CLAWDBOT_PROFILE_FILE=...`（默认：`~/.profile`）挂载到 `/home/node/.profile` 并在运行测试之前源
- `CLAWDBOT_LIVE_GATEWAY_MODELS=...` / `CLAWDBOT_LIVE_MODELS=...` 以缩小运行
- `CLAWDBOT_LIVE_REQUIRE_PROFILE_KEYS=1` 以确保凭据来自配置文件存储（不是 env）

## 文档健全性

在文档编辑后运行文档检查：`pnpm docs:list`。

## 离线回归（CI 安全）

这些是“真实管道”回归，没有真实提供者：
- Gateway 工具调用（模拟 OpenAI，真实网关 + 代理循环）：`src/gateway/gateway.tool-calling.mock-openai.test.ts`
- Gateway 向导（WS `wizard.start`/`wizard.next`，写入配置 + 强制认证）：`src/gateway/gateway.wizard.e2e.test.ts`

## 代理可靠性评估（技能）

我们已经有一些 CI 安全测试，其行为类似于“代理可靠性评估”：
- 通过真实网关 + 代理循环进行模拟工具调用（`src/gateway/gateway.tool-calling.mock-openai.test.ts`）。
- 端到端向导流，验证会话布线和配置效果（`src/gateway/gateway.wizard.e2e.test.ts`）。

技能仍然缺少的内容（参见 [技能](/tools/skills)）：
- **决策：** 当技能在提示中列出时，代理是否选择正确的技能（或避免不相关的技能）？
- **合规性：** 代理是否在使用前读取 `SKILL.md` 并遵循必需的步骤/参数？
- **工作流契约：** 多回合场景，断言工具顺序、会话历史延续和沙箱边界。

未来的评估应该首先保持确定性：
- 使用模拟提供者的场景运行器，断言工具调用 + 顺序、技能文件读取和会话布线。
- 一套小的、专注于技能的场景（使用 vs 避免、门控、提示注入）。
- 可选的 live 评估（选择加入，env 门控）仅在 CI 安全套件就位后。

## 添加回归（指导）

当你修复在 live 中发现的提供者/模型问题时：
- 如果可能，添加 CI 安全回归（模拟/存根提供者，或捕获确切的请求形状转换）
- 如果它本质上是仅 live 的（速率限制、认证策略），保持 live 测试窄并通过环境变量选择加入
- 最好针对捕获错误的最小层：
  - 提供者请求转换/重放错误 → 直接模型测试
  - 网关会话/历史/工具管道错误 → 网关 live 冒烟或 CI 安全网关模拟测试
