# 自定义模型

通过 `~/.pi/agent/models.json` 添加自定义服务提供商和模型（Ollama、vLLM、LM Studio、代理服务等）。

## 目录

- [最小示例](#minimal-example)
- [完整示例](#full-example)
- [支持的 API](#supported-apis)
- [服务提供商配置](#provider-configuration)
- [模型配置](#model-configuration)
- [覆盖内置服务提供商](#overriding-built-in-providers)
- [按模型覆盖配置](#per-model-overrides)
- [Anthropic Messages 兼容性](#anthropic-messages-compatibility)
- [OpenAI 兼容性](#openai-compatibility)

<a id="minimal-example"></a>

## 最小示例

对于本地模型（Ollama、LM Studio、vLLM），每个模型只需配置 `id`：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

`apiKey` 是占位值，Ollama 会忽略它。但 pi 仍会把这些模型视为需要认证，只有认证后才会在 `/model` 中显示。因此，对于无需密钥的本地服务器，可以保留一个虚拟值、使用 `/login` 为该服务提供商保存密钥，或在选择模型时传入 `--api-key`。

部分兼容 OpenAI 的服务器无法识别推理模型使用的 `developer` 角色。对于这类服务提供商，将 `compat.supportsDeveloperRole` 设为 `false`，pi 就会改用 `system` 消息发送系统提示词。如果服务器也不支持 `reasoning_effort`，还应将 `compat.supportsReasoningEffort` 设为 `false`。

可以在服务提供商级别设置 `compat`，使其作用于全部模型；也可以在模型级别设置，为特定模型覆盖配置。这类设置通常适用于 Ollama、vLLM、SGLang 等兼容 OpenAI 的服务器。

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "gpt-oss:20b",
          "reasoning": true
        }
      ]
    }
  }
}
```

<a id="full-example"></a>

## 完整示例

需要指定具体参数时，可以覆盖默认值：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "llama3.1:8b",
          "name": "Llama 3.1 8B (Local)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 32000,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

每次打开 `/model` 时都会重新加载该文件。因此可以在会话过程中修改，无需重启。

## Google AI Studio 示例

使用 `google-generative-ai` 并配置 `baseUrl`，即可添加 Google AI Studio 中的模型，包括自定义 Gemma 4 条目：

```json
{
  "providers": {
    "my-google": {
      "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
      "api": "google-generative-ai",
      "apiKey": "$GEMINI_API_KEY",
      "models": [
        {
          "id": "gemma-4-31b-it",
          "name": "Gemma 4 31B",
          "input": ["text", "image"],
          "contextWindow": 262144,
          "reasoning": true
        }
      ]
    }
  }
}
```

向 `google-generative-ai` API 类型添加自定义模型时，必须提供 `baseUrl`。

<a id="supported-apis"></a>

## 支持的 API

| API | 说明 |
|-----|-------------|
| `openai-completions` | OpenAI Chat Completions（兼容性最广） |
| `openai-responses` | OpenAI Responses API |
| `anthropic-messages` | Anthropic Messages API |
| `google-generative-ai` | Google Generative AI |

可以在服务提供商级别设置 `api`（作为所有模型的默认值），也可以在模型级别设置（按模型覆盖）。

<a id="provider-configuration"></a>

## 服务提供商配置

| 字段 | 说明 |
|-------|-------------|
| `baseUrl` | API 端点 URL |
| `api` | API 类型（见上文） |
| `apiKey` | 可选的 API 密钥配置（取值解析方式见下文）。如果认证信息由 `/login`/`auth.json` 或命令行参数 `--api-key` 提供，请省略此字段。 |
| `oauth` | 动态 OAuth 服务提供商类型。目前支持 `"radius"`；必须同时提供网关的 `baseUrl`。 |
| `headers` | 自定义请求头（取值解析方式见下文） |
| `authHeader` | 设为 `true` 时，自动添加 `Authorization: Bearer <apiKey>` |
| `models` | 模型配置数组 |
| `modelOverrides` | 针对该服务提供商的内置模型或扩展注册模型，按模型覆盖配置 |

对于包含 `models` 的服务提供商，非内置服务提供商的配置必须提供 `baseUrl`，并在服务提供商或模型级别设置 `api`。加载文件本身不要求提供 `apiKey`；通过 `/login`/`auth.json`、命令行参数 `--api-key` 或服务提供商的 `apiKey` 配置认证后，模型才会变为可用。若未配置认证，模型仍会加载，但不会作为可用模型出现在 `/model` 和 `--list-models` 中。

### 取值解析

`apiKey` 和 `headers` 字段支持执行命令、插值环境变量和使用字面值：

- **Shell 命令：** 值以 `"!command"` 开头时，会将整个值作为命令执行，并使用其标准输出
  ```json
  "apiKey": "!security find-generic-password -ws 'anthropic'"
  "apiKey": "!op read 'op://vault/item/credential'"
  ```
- **环境变量插值：** `"$ENV_VAR"` 或 `"${ENV_VAR}"` 使用对应变量的值。较长的字面值中也可以进行插值。
  ```json
  "apiKey": "$MY_API_KEY"
  "apiKey": "${KEY_PREFIX}_${KEY_SUFFIX}"
  ```
  `$FOO_BAR` 指变量 `FOO_BAR`；如果 `BAR` 是字面文本，应写作 `${FOO}_BAR`。环境变量不存在时，该值无法解析。
- **转义：** `"$$"` 产生字面量 `"$"`；`"$!"` 产生字面量 `"!"`，且不会触发命令执行。
  ```json
  "apiKey": "$$literal-dollar-prefix"
  "apiKey": "$!literal-bang-prefix"
  ```
- **字面值：** 直接使用。`MY_API_KEY` 这样的纯大写字符串仍是字面值；要引用环境变量，请写成 `$MY_API_KEY`。
  ```json
  "apiKey": "sk-..."
  ```

对于 `models.json`，Shell 命令会在发起请求时解析。pi 刻意不为任意命令内置 TTL（存活时间）、旧值复用或故障恢复逻辑，因为不同命令需要不同的缓存与失败处理策略，pi 无法推断合适的做法。

如果命令执行缓慢、成本较高、受到速率限制，或在临时故障时应继续使用旧值，请使用自己的脚本或命令封装它，并在其中实现所需的缓存或 TTL 策略。

`/model` 检查可用性时只判断认证配置是否存在，不会执行 Shell 命令。

### 自定义请求头

```json
{
  "providers": {
    "custom-proxy": {
      "baseUrl": "https://proxy.example.com/v1",
      "apiKey": "$MY_API_KEY",
      "api": "anthropic-messages",
      "headers": {
        "x-portkey-api-key": "$PORTKEY_API_KEY",
        "x-secret": "!op read 'op://vault/item/secret'"
      },
      "models": [...]
    }
  }
}
```

<a id="model-configuration"></a>

## 模型配置

| 字段 | 必填 | 默认值 | 说明 |
|-------|----------|---------|-------------|
| `id` | 是 | — | 模型标识符（原样传给 API） |
| `name` | 否 | `id` | 便于阅读的模型名称。用于匹配（`--model` 模式），并作为模型的次要详情文本显示。 |
| `api` | 否 | 服务提供商的 `api` | 为该模型覆盖服务提供商的 API |
| `reasoning` | 否 | `false` | 是否支持扩展思考 |
| `thinkingLevelMap` | 否 | 省略 | 将 pi 的思考级别映射到服务提供商的值，并标记不支持的级别（见下文） |
| `input` | 否 | `["text"]` | 输入类型：`["text"]` 或 `["text", "image"]` |
| `contextWindow` | 否 | `128000` | 上下文窗口大小，单位为 Token |
| `maxTokens` | 否 | `16384` | 最大输出 Token 数 |
| `cost` | 否 | 全部为零 | 每百万 Token 的费率；可选配置作用于整个请求的输入定价分档 |
| `compat` | 否 | 服务提供商的 `compat` | 服务提供商兼容性覆盖配置。若模型和服务提供商两级均有设置，则合并二者。 |

费用分档提供一套完整的替代费率。当总输入用量（`input + cacheRead + cacheWrite`）超过 `inputTokensAbove` 时，这套费率将作用于整个请求。多个分档同时匹配时，以阈值最高者为准。

```json
{
  "cost": {
    "input": 5,
    "output": 30,
    "cacheRead": 0.5,
    "cacheWrite": 6.25,
    "tiers": [
      {
        "inputTokensAbove": 272000,
        "input": 10,
        "output": 45,
        "cacheRead": 1,
        "cacheWrite": 12.5
      }
    ]
  }
}
```

当前行为：

- `/model`、`--list-models` 和交互界面页脚均按模型 `id` 显示条目。
- 配置的 `name` 用于模型匹配和次要详情文本，不会取代页脚或状态栏中的模型 ID。

### 思考级别映射

在模型上使用 `thinkingLevelMap` 描述该模型特有的思考控制方式。键为 pi 的思考级别：`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max`。映射中可以缺少中间级别；例如，模型可以提供 `high` 和 `max`，但不提供 `xhigh`。

值有三种状态：

| 值 | 含义 |
|-------|---------|
| 省略 | 截至 `high` 的标准级别使用服务提供商的默认映射；不支持扩展级别 `xhigh` 和 `max` |
| 字符串 | 支持该级别，并将此值发送给服务提供商 |
| `null` | 不支持该级别；在界面中隐藏、遍历时跳过，或在取值时限制到其他有效级别 |

以下模型仅支持关闭推理以及 `high`、`max` 两个推理级别：

```json
{
  "id": "deepseek-v4-pro",
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": null,
    "low": null,
    "medium": null,
    "high": "high",
    "xhigh": null,
    "max": "max"
  }
}
```

以下模型无法关闭思考：

```json
{
  "id": "always-thinking-model",
  "reasoning": true,
  "thinkingLevelMap": {
    "off": null
  }
}
```

迁移说明：旧配置如果使用了 `compat.reasoningEffortMap`，应将该映射移到模型级别的 `thinkingLevelMap`。不应出现在界面中的级别请设为 `null`。

<a id="overriding-built-in-providers"></a>

## 覆盖内置服务提供商

可以让内置服务提供商经由代理转发，而无需重新定义模型：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1"
    }
  }
}
```

所有内置 Anthropic 模型仍然可用，已有的 OAuth 或 API 密钥认证也会继续生效。

要将自定义模型合并到内置服务提供商中，请加入 `models` 数组：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1",
      "apiKey": "$ANTHROPIC_API_KEY",
      "api": "anthropic-messages",
      "models": [...]
    }
  }
}
```

合并规则：

- 保留内置模型。
- 在同一服务提供商内，按 `id` 新增或更新自定义模型。
- 如果自定义模型的 `id` 与内置模型相同，自定义模型会替换该内置模型。
- 如果自定义模型的 `id` 是新的，则将其添加到内置模型旁。

<a id="per-model-overrides"></a>

## 按模型覆盖配置

使用 `modelOverrides` 可以自定义内置模型和匹配的扩展注册模型，而不必替换服务提供商的完整模型列表。

```json
{
  "providers": {
    "openrouter": {
      "modelOverrides": {
        "anthropic/claude-sonnet-4": {
          "name": "Claude Sonnet 4 (Bedrock Route)",
          "compat": {
            "openRouterRouting": {
              "only": ["amazon-bedrock"]
            }
          }
        }
      }
    }
  }
}
```

`modelOverrides` 支持为每个模型设置以下字段：`name`、`reasoning`、`thinkingLevelMap`、`input`、`cost`（可只提供部分字段）、`contextWindow`、`maxTokens`、`headers`、`compat`。

直接使用 OpenAI GPT-5.6 Sol、Terra 和 Luna 时，默认上下文窗口为 `272000`，以使请求保持在 OpenAI 的短上下文定价分档内。若要启用 OpenAI 的 105 万 Token 上下文窗口，请为使用的每个模型增大该值：

```json
{
  "providers": {
    "openai": {
      "modelOverrides": {
        "gpt-5.6-sol": {
          "contextWindow": 1050000
        }
      }
    }
  }
}
```

覆盖上下文窗口不会改变内置定价元数据。当总输入超过 27.2 万 Token 时，整个请求都会采用 GPT-5.6 的长上下文费率。需要时，可对 `gpt-5.6-terra` 或 `gpt-5.6-luna` 应用相同的覆盖配置。

行为说明：

- `modelOverrides` 会作用于内置服务提供商模型，以及 `id` 匹配的扩展注册模型。
- 未知的模型 ID 会被忽略。
- 可以将服务提供商级别的 `baseUrl`/`headers` 与 `modelOverrides` 组合使用。
- 覆盖 `name` 只会改变模型匹配和次要详情文本；页脚和主要模型列表仍显示模型 `id`。
- 如果服务提供商还定义了 `models`，自定义模型会在应用内置模型覆盖配置后合并。`id` 相同的自定义模型会替换已经覆盖过配置的内置模型条目。

<a id="anthropic-messages-compatibility"></a>

## Anthropic Messages 兼容性

对于使用 `api: "anthropic-messages"` 的服务提供商或代理，可通过 `compat` 控制 Anthropic 特有的请求兼容行为。

默认情况下，pi 会为每个工具发送 `eager_input_streaming: true`。如果代理或兼容 Anthropic 的后端拒绝该字段，请将 `supportsEagerToolInputStreaming` 设为 `false`。此后 pi 会省略 `tools[].eager_input_streaming`，并改为在启用工具的请求中发送旧版 `fine-grained-tool-streaming-2025-05-14` Beta 请求头。

部分 Anthropic 模型要求使用自适应思考（`thinking.type: "adaptive"` 加 `output_config.effort`），而不是旧版基于预算的思考请求体。内置模型会自动配置这一行为。对于路由到这些模型的自定义服务提供商或别名，请将 `forceAdaptiveThinking` 设为 `true`。

部分兼容 Anthropic 的服务提供商会返回签名为空的思考块，并且在重放消息时仍要求带上这些块。只有对这类服务提供商才应将 `allowEmptySignature` 设为 `true`；Anthropic 官方接口会拒绝空的思考签名。

内置 Anthropic 模型已在模型元数据中启用 `supportsStrictTools`。如果自定义的 Anthropic 兼容模型端点接受严格的 JSON Schema 工具定义，则必须将此项设为 `true`。

```json
{
  "providers": {
    "anthropic-proxy": {
      "baseUrl": "https://proxy.example.com",
      "api": "anthropic-messages",
      "apiKey": "$ANTHROPIC_PROXY_KEY",
      "compat": {
        "supportsEagerToolInputStreaming": false,
        "supportsLongCacheRetention": true,
        "forceAdaptiveThinking": true,
        "allowEmptySignature": true
      },
      "models": [
        {
          "id": "claude-opus-4-7",
          "reasoning": true,
          "input": ["text", "image"]
        }
      ]
    }
  }
}
```

| 字段 | 说明 |
|-------|-------------|
| `supportsEagerToolInputStreaming` | 服务提供商是否接受每个工具上的 `eager_input_streaming`。默认值：`true`。设为 `false` 后会省略该字段，并在启用工具的请求中使用旧版细粒度工具流式传输 Beta 请求头。 |
| `supportsLongCacheRetention` | 当缓存保留策略为 `long` 时，服务提供商是否接受 Anthropic 长时缓存（`cache_control.ttl: "1h"`）。默认值：`true`。 |
| `sendSessionAffinityHeaders` | 启用缓存时，是否根据会话 ID 发送 `x-session-affinity`。默认行为：对已知服务提供商自动检测。 |
| `supportsCacheControlOnTools` | 服务提供商是否接受工具定义上的 Anthropic 风格 `cache_control` 标记。默认值：`true`。 |
| `forceAdaptiveThinking` | 是否为该模型发送自适应思考配置（`thinking.type: "adaptive"` 加 `output_config.effort`）。内置的自适应模型会自动设置此项。默认值：`false`。 |
| `allowEmptySignature` | 是否以 `signature: ""` 的形式重放空的思考签名，而不是将思考内容转换为文本。默认值：`false`。 |
| `supportsStrictTools` | 服务提供商是否接受严格的 JSON Schema 工具定义。默认值：`false`；内置 Anthropic 模型会在生成的元数据中启用此项。 |

<a id="openai-compatibility"></a>

## OpenAI 兼容性

对于只实现了部分 OpenAI 兼容能力的服务提供商，请使用 `compat` 字段。

- 服务提供商级别的 `compat` 为该服务提供商下的所有模型提供默认值。
- 模型级别的 `compat` 为该模型覆盖服务提供商级别的值。

```json
{
  "providers": {
    "local-llm": {
      "baseUrl": "http://localhost:8080/v1",
      "api": "openai-completions",
      "compat": {
        "supportsUsageInStreaming": false,
        "maxTokensField": "max_tokens"
      },
      "models": [...]
    }
  }
}
```

| 字段 | 说明 |
|-------|-------------|
| `supportsStore` | 服务提供商是否支持 `store` 字段 |
| `supportsDeveloperRole` | 使用 `developer` 角色还是 `system` 角色 |
| `supportsReasoningEffort` | 是否支持 `reasoning_effort` 参数 |
| `supportsUsageInStreaming` | 是否支持 `stream_options: { include_usage: true }`（默认值：`true`） |
| `maxTokensField` | 使用 `max_completion_tokens` 还是 `max_tokens` |
| `requiresToolResultName` | 工具结果消息中是否必须包含 `name` |
| `requiresAssistantAfterToolResult` | 工具结果后出现用户消息时，是否要在用户消息前插入一条助手消息 |
| `requiresThinkingAsText` | 是否将思考块转换为纯文本 |
| `requiresReasoningContentOnAssistantMessages` | 启用推理时，是否要在所有重放的助手消息中加入空的 `reasoning_content` |
| `thinkingFormat` | 使用 `reasoning_effort`、`openrouter`、`deepseek`、`together`、`zai`、`qwen`、`chat-template` 或 `qwen-chat-template` 格式的思考参数 |
| `chatTemplateKwargs` | `thinkingFormat: "chat-template"` 使用的 `chat_template_kwargs` 值；若要由 pi 控制思考值，请使用 `{ "$var": "thinking.enabled" }` 或 `{ "$var": "thinking.effort" }` |
| `cacheControlFormat` | 在系统提示词、最后一个工具定义，以及最后一段用户、助手或工具结果文本内容上使用 Anthropic 风格的 `cache_control` 标记。目前仅支持 `anthropic`。 |
| `sendSessionAffinityHeaders` | 对 `openai-completions` 而言，启用缓存时是否根据会话 ID 发送会话亲和性请求头。默认值：`false`。 |
| `sessionAffinityFormat` | `openai-completions` 和 `openai-responses` 使用的会话亲和性请求头格式：`openai` 发送 `session_id`/`x-client-request-id`（Completions 还发送 `x-session-affinity`）；`openai-nosession` 省略名称中含下划线的 `session_id` 请求头；`openrouter` 发送 `x-session-id`。此配置不影响请求体中的 `prompt_cache_key` 参数。默认行为：自动检测。 |
| `supportsStrictMode` | 服务提供商是否接受严格的 JSON Schema 函数工具定义。默认值取决于 API；内置 OpenAI 模型带有明确的能力元数据。 |
| `supportsOpenAIGrammarTools` | 兼容 OpenAI 的 API 是否支持自定义 Lark/正则表达式语法工具。设为 `false` 时，受语法约束的工具会退化为普通函数工具。默认值：`false`；内置模型目录为 OpenAI、OpenAI Codex、Azure OpenAI、GitHub Copilot、opencode 和 Cloudflare AI Gateway 上的 GPT-5+ 模型启用了此能力。 |
| `deferredToolsMode` | 使用服务提供商特有的延迟工具序列化格式。目前只支持 `"kimi"`，对应 Kimi 的 OpenAI 兼容 Chat Completions 格式。 |
| `supportsLongCacheRetention` | 当缓存保留策略为 `long` 时，服务提供商是否接受长时缓存：OpenAI 提示词缓存使用 `prompt_cache_retention: "24h"`；当 `cacheControlFormat` 为 `anthropic` 时使用 `cache_control.ttl: "1h"`。默认值：`true`。 |
| `openRouterRouting` | OpenRouter 服务提供商的路由偏好。该对象会原样放入 [OpenRouter API 请求](https://openrouter.ai/docs/guides/routing/provider-selection)的 `provider` 字段。 |
| `vercelGatewayRouting` | 用于选择服务提供商的 Vercel AI Gateway 路由配置（`only`、`order`） |

`openrouter` 使用 `reasoning: { effort }`。`together` 使用 `reasoning: { enabled }`，且在启用 `supportsReasoningEffort` 时还会使用 `reasoning_effort`。`qwen` 使用顶层字段 `enable_thinking`。对于要求提供 `chat_template_kwargs.enable_thinking` 和 `preserve_thinking` 的本地 Qwen 兼容服务器，请使用 `qwen-chat-template`。对于需要自定义 `chat_template_kwargs` 的 vLLM/Hugging Face 对话模板，请使用 `chat-template`；例如，DeepSeek V3.x 模板可配置 `chatTemplateKwargs: { "thinking": { "$var": "thinking.enabled" } }`。

`cacheControlFormat: "anthropic"` 适用于兼容 OpenAI、但通过文本内容和工具定义上的 `cache_control` 标记提供 Anthropic 风格提示词缓存的服务提供商。

示例：

```json
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "apiKey": "$OPENROUTER_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "openrouter/anthropic/claude-3.5-sonnet",
          "name": "OpenRouter Claude 3.5 Sonnet",
          "compat": {
            "openRouterRouting": {
              "allow_fallbacks": true,
              "require_parameters": false,
              "data_collection": "deny",
              "zdr": true,
              "enforce_distillable_text": false,
              "order": ["anthropic", "amazon-bedrock", "google-vertex"],
              "only": ["anthropic", "amazon-bedrock"],
              "ignore": ["gmicloud", "friendli"],
              "quantizations": ["fp16", "bf16"],
              "sort": {
                "by": "price",
                "partition": "model"
              },
              "max_price": {
                "prompt": 10,
                "completion": 20
              },
              "preferred_min_throughput": {
                "p50": 100,
                "p90": 50
              },
              "preferred_max_latency": {
                "p50": 1,
                "p90": 3,
                "p99": 5
              }
            }
          }
        }
      ]
    }
  }
}
```

Vercel AI Gateway 示例：

```json
{
  "providers": {
    "vercel-ai-gateway": {
      "baseUrl": "https://ai-gateway.vercel.sh/v1",
      "apiKey": "$AI_GATEWAY_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "moonshotai/kimi-k2.5",
          "name": "Kimi K2.5 (Fireworks via Vercel)",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 0.6, "output": 3, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 262144,
          "maxTokens": 262144,
          "compat": {
            "vercelGatewayRouting": {
              "only": ["fireworks", "novita"],
              "order": ["fireworks", "novita"]
            }
          }
        }
      ]
    }
  }
}
```
