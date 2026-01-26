---
summary: "Where Clawdbot loads environment variables and the precedence order"
read_when:
  - You need to know which env vars are loaded, and in what order
  - You are debugging missing API keys in the Gateway
  - You are documenting provider auth or deployment environments
---
# 环境变量

Clawdbot 从多个源拉取环境变量。规则是**永远不覆盖现有值**。

## 优先级（最高 → 最低）

1) **进程环境**（Gateway 进程从父 shell/守护进程已有的内容）。
2) **当前工作目录中的 `.env`**（dotenv 默认；不覆盖）。
3) **全局 `.env`** 位于 `~/.clawdbot/.env`（即 `$CLAWDBOT_STATE_DIR/.env`；不覆盖）。
4) **配置 `env` 块**在 `~/.clawdbot/clawdbot.json` 中（仅在缺少时应用）。
5) **可选的登录 shell 导入**（`env.shellEnv.enabled` 或 `CLAWDBOT_LOAD_SHELL_ENV=1`），仅对缺少的预期键应用。

如果配置文件完全缺失，则跳过步骤 4；如果启用，shell 导入仍会运行。

## 配置 `env` 块

两种等效的设置内联环境变量的方法（两者都是非覆盖的）：

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-..."
    }
  }
}
```

## Shell 环境导入

`env.shellEnv` 运行你的登录 shell 并仅导入**缺少的**预期键：

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000
    }
  }
}
```

环境变量等效项：
- `CLAWDBOT_LOAD_SHELL_ENV=1`
- `CLAWDBOT_SHELL_ENV_TIMEOUT_MS=15000`

## 配置中的环境变量替换

你可以使用 `${VAR_NAME}` 语法在配置字符串值中直接引用环境变量：

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}"
      }
    }
  }
}
```

有关完整详细信息，请参见 [配置：环境变量替换](/gateway/configuration#env-var-substitution-in-config)。

## 相关

- [Gateway 配置](/gateway/configuration)
- [FAQ：环境变量和 .env 加载](/help/faq#env-vars-and-env-loading)
- [模型概述](/concepts/models)
