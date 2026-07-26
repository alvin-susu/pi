# 自定义服务提供商

扩展可以通过 `pi.registerProvider()` 注册自定义模型服务提供商，从而实现：

- **代理**——通过企业代理或 API 网关转发请求
- **自定义端点**——使用自行托管或私有部署的模型
- **OAuth/SSO**——为企业服务提供商添加认证流程
- **自定义 API**——为非标准 LLM API 实现流式传输

<a id="example-extensions"></a>

## 扩展示例

以下目录提供了完整的服务提供商示例：

- [`examples/extensions/custom-provider-anthropic/`](../examples/extensions/custom-provider-anthropic/)
- [`examples/extensions/custom-provider-gitlab-duo/`](../examples/extensions/custom-provider-gitlab-duo/)

## 目录

- [扩展示例](#example-extensions)
- [快速参考](#quick-reference)
- [覆盖现有服务提供商](#override-existing-provider)
- [注册新服务提供商](#register-new-provider)
- [注销服务提供商](#unregister-provider)
- [OAuth 支持](#oauth-support)
- [自定义流式 API](#custom-streaming-api)
- [上下文溢出错误](#context-overflow-errors)
- [测试实现](#testing-your-implementation)
- [配置参考](#config-reference)
- [模型定义参考](#model-definition-reference)

<a id="quick-reference"></a>

## 快速参考

扩展既可以注册完整的 pi-ai `Provider`，也可以使用旧版服务提供商配置形式。需要自定义认证、筛选、刷新或流式传输行为时，应优先注册完整的服务提供商。Pi 会将 `models.json` 中的覆盖配置叠加到已注册的原生服务提供商之上。

```typescript
import { createProvider, openAICompletionsApi } from "@earendil-works/pi-ai";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerProvider(createProvider({
    id: "native-local",
    name: "Native Local",
    baseUrl: "http://localhost:8080/v1",
    auth: {
      apiKey: {
        name: "Local server API key",
        async login(interaction) {
          return {
            type: "api_key",
            key: await interaction.prompt({ type: "secret", message: "API key" })
          };
        },
        async resolve({ credential }) {
          return credential?.key
            ? { auth: { apiKey: credential.key }, source: "stored API key" }
            : undefined;
        }
      }
    },
    models: [],
    api: openAICompletionsApi()
  }));

  // 旧版服务提供商配置形式：
  // 覆盖现有服务提供商的 baseUrl
  pi.registerProvider("anthropic", {
    baseUrl: "https://proxy.example.com"
  });

  // 注册包含模型的新服务提供商
  pi.registerProvider("my-provider", {
    name: "My Provider",
    baseUrl: "https://api.example.com",
    apiKey: "$MY_API_KEY",
    api: "openai-completions",
    models: [
      {
        id: "my-model",
        name: "My Model",
        reasoning: false,
        input: ["text", "image"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 4096
      }
    ]
  });
}
```

扩展工厂函数也可以是 `async`。如果需要动态发现模型，应在工厂函数中获取并注册模型，而不是在 `session_start` 中执行。pi 会等待工厂函数完成后再继续启动，因此交互模式启动时和执行 `pi --list-models` 时，该服务提供商都已经可用。

<a id="override-existing-provider"></a>

## 覆盖现有服务提供商

最简单的用法是通过代理重定向现有服务提供商：

```typescript
// 此后所有 Anthropic 请求都经由该代理
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com"
});

// 为 OpenAI 请求添加自定义请求头
pi.registerProvider("openai", {
  headers: {
    "X-Custom-Header": "value"
  }
});

// 同时设置 baseUrl 和 headers
pi.registerProvider("google", {
  baseUrl: "https://ai-gateway.corp.com/google",
  headers: {
    "X-Corp-Auth": "$CORP_AUTH_TOKEN"  // 环境变量或字面值
  }
});
```

只提供 `baseUrl` 和/或 `headers`（不提供 `models`）时，会保留该服务提供商的全部现有模型，并让它们使用新的端点。

<a id="register-new-provider"></a>

## 注册新服务提供商

要添加全新的服务提供商，请同时指定 `models` 和其他必要配置。

如果模型列表来自远程端点，请使用异步扩展工厂函数：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const payload = (await response.json()) as {
    data: Array<{
      id: string;
      name?: string;
      context_window?: number;
      max_tokens?: number;
    }>;
  };

  pi.registerProvider("local-openai", {
    baseUrl: "http://localhost:1234/v1",
    apiKey: "$LOCAL_OPENAI_API_KEY",
    api: "openai-completions",
    models: payload.data.map((model) => ({
      id: model.id,
      name: model.name ?? model.id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: model.context_window ?? 128000,
      maxTokens: model.max_tokens ?? 4096,
    })),
  });
}
```

这样可以在启动完成前注册获取到的模型。

```typescript
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com/v1",
  apiKey: "$MY_LLM_API_KEY",  // 引用环境变量
  api: "openai-completions",  // 使用哪种流式 API
  models: [
    {
      id: "my-llm-large",
      name: "My LLM Large",
      reasoning: true,        // 支持扩展思考
      input: ["text", "image"],
      cost: {
        input: 3.0,           // 美元/百万 Token
        output: 15.0,
        cacheRead: 0.3,
        cacheWrite: 3.75
      },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});
```

提供 `models` 后，它会**替换**该服务提供商的全部现有模型。

`apiKey` 和自定义请求头的值采用与 `models.json` 相同的配置语法：值以 `!command` 开头时，会将整个值作为命令执行；`$ENV_VAR` 和 `${ENV_VAR}` 用于插值环境变量；`$$` 产生字面量 `$`；`$!` 产生字面量 `!`。

<a id="unregister-provider"></a>

## 注销服务提供商

使用 `pi.unregisterProvider(name)` 可以移除此前通过 `pi.registerProvider(name, ...)` 注册的服务提供商：

```typescript
// 注册
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com/v1",
  apiKey: "$MY_LLM_API_KEY",
  api: "openai-completions",
  models: [
    {
      id: "my-llm-large",
      name: "My LLM Large",
      reasoning: true,
      input: ["text", "image"],
      cost: { input: 3.0, output: 15.0, cacheRead: 0.3, cacheWrite: 3.75 },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});

// 稍后将其移除
pi.unregisterProvider("my-llm");
```

注销操作会移除该服务提供商的动态模型、API 密钥回退值、OAuth 服务提供商注册信息和自定义流处理函数注册信息。此前被覆盖的内置模型或服务提供商行为会恢复。

在扩展初次加载阶段之后调用该方法，变更也会立即生效，无需执行 `/reload`。

### API 类型

`api` 字段决定使用哪一种流式传输实现：

| API | 适用场景 |
|-----|---------|
| `anthropic-messages` | Anthropic Claude API 及其兼容接口 |
| `openai-completions` | OpenAI Chat Completions API 及其兼容接口 |
| `openai-responses` | OpenAI Responses API |
| `azure-openai-responses` | Azure OpenAI Responses API |
| `openai-codex-responses` | OpenAI Codex Responses API |
| `mistral-conversations` | Mistral SDK Conversations/Chat 流式接口 |
| `google-generative-ai` | Google Generative AI API |
| `google-vertex` | Google Vertex AI API |
| `bedrock-converse-stream` | Amazon Bedrock Converse API |

大多数兼容 OpenAI 的服务提供商都可以使用 `openai-completions`。模型特有的思考级别使用模型级别的 `thinkingLevelMap` 配置，服务提供商的特殊兼容行为则使用 `compat` 配置。`xhigh` 和 `max` 级别需要显式启用，映射项不能为 `null`，且不同有效级别之间可以存在不受支持的空档：

```typescript
models: [{
  id: "custom-model",
  // ...
  reasoning: true,
  thinkingLevelMap: {              // 将 pi 级别映射到服务提供商的值；null 会隐藏不支持的级别
    minimal: null,
    low: null,
    medium: null,
    high: "default",
    xhigh: null,
    max: "max"
  },
  compat: {
    supportsDeveloperRole: false,   // 使用 "system" 而不是 "developer"
    supportsReasoningEffort: true,
    maxTokensField: "max_tokens",   // 使用此字段而不是 "max_completion_tokens"
    requiresToolResultName: true,   // 工具结果必须包含 name 字段
    thinkingFormat: "qwen",         // 顶层 enable_thinking: true
    cacheControlFormat: "anthropic" // Anthropic 风格的 cache_control 标记
  }
}]
```

OpenRouter 风格的 `reasoning: { effort }` 控制使用 `openrouter`。Together 风格的 `reasoning: { enabled }` 控制使用 `together`；启用 `supportsReasoningEffort` 时，还会发送 `reasoning_effort`。对于读取 `chat_template_kwargs.enable_thinking` 且需要 `preserve_thinking` 的本地 Qwen 兼容服务器，请使用 `qwen-chat-template`。
对于通过系统提示词、最后一个工具定义，以及最后一段用户、助手或工具结果文本内容上的 `cache_control` 提供 Anthropic 风格提示词缓存的 OpenAI 兼容服务提供商，请使用 `cacheControlFormat: "anthropic"`。

对于使用 `api: "anthropic-messages"` 的 Anthropic 兼容服务提供商，如果其上游模型要求自适应思考（`thinking.type: "adaptive"` 加 `output_config.effort`），请在模型或服务提供商上设置 `compat.forceAdaptiveThinking: true`。内置的自适应 Claude 模型会自动设置此项。只有当服务提供商会返回空的思考签名，并在重放时要求 `signature: ""`，才应设置 `compat.allowEmptySignature: true`。

> 迁移说明：Mistral 已从 `openai-completions` 迁移到 `mistral-conversations`。
> 原生 Mistral 模型请使用 `mistral-conversations`。
> 如果确实需要通过 `openai-completions` 路由 Mistral 兼容端点或自定义端点，请根据需要显式设置 `compat` 开关。

### 认证请求头

如果服务提供商要求使用 `Authorization: Bearer <key>`，但没有使用标准 API，请设置 `authHeader: true`：

```typescript
pi.registerProvider("custom-api", {
  baseUrl: "https://api.example.com",
  apiKey: "$MY_API_KEY",
  authHeader: true,  // 添加 Authorization: Bearer 请求头
  api: "openai-completions",
  models: [...]
});
```

每次请求都会重新解析密钥。如果请求中显式提供了 `Authorization` 请求头，则优先使用显式值。

<a id="oauth-support"></a>

## OAuth 支持

可以添加与 `/login` 集成的 OAuth/SSO 认证：

```typescript
import type { OAuthCredentials, OAuthLoginCallbacks } from "@earendil-works/pi-ai";

pi.registerProvider("corporate-ai", {
  baseUrl: "https://ai.corp.com/v1",
  api: "openai-responses",
  models: [...],
  oauth: {
    name: "Corporate AI (SSO)",

    async login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials> {
      const method = await callbacks.onSelect({
        message: "Select login method:",
        options: [
          { id: "browser", label: "Browser OAuth" },
          { id: "device", label: "Device code" }
        ]
      });
      if (!method) throw new Error("Login cancelled");

      let code: string;
      if (method === "device") {
        callbacks.onDeviceCode({
          userCode: "ABCD-1234",
          verificationUri: "https://sso.corp.com/device",
          intervalSeconds: 5,
          expiresInSeconds: 900
        });
        code = await pollDeviceCodeUntilComplete();
      } else {
        callbacks.onAuth({ url: "https://sso.corp.com/authorize?..." });
        code = await callbacks.onPrompt({ message: "Enter SSO code:" });
      }

      // 用授权码换取 Token（由你实现）
      const tokens = await exchangeCodeForTokens(code);

      return {
        refresh: tokens.refreshToken,
        access: tokens.accessToken,
        expires: Date.now() + tokens.expiresIn * 1000
      };
    },

    async refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials> {
      const tokens = await refreshAccessToken(credentials.refresh);
      return {
        refresh: tokens.refreshToken ?? credentials.refresh,
        access: tokens.accessToken,
        expires: Date.now() + tokens.expiresIn * 1000
      };
    },

    getApiKey(credentials: OAuthCredentials): string {
      return credentials.access;
    }
  }
});
```

注册后，用户可以通过 `/login corporate-ai` 完成认证。

### OAuthLoginCallbacks

`callbacks` 对象为服务提供商自己的认证流程提供与界面无关的交互能力：

```typescript
interface OAuthLoginCallbacks {
  // 在浏览器中打开 URL（用于 OAuth 重定向）
  onAuth(params: { url: string }): void;

  // 显示设备码（用于设备授权流程）
  onDeviceCode(params: {
    userCode: string;
    verificationUri: string;
    intervalSeconds?: number;
    expiresInSeconds?: number;
  }): void;

  // 显示临时进度
  onProgress?(message: string): void;

  // 提示用户输入（用于手动输入 Token）
  onPrompt(params: { message: string }): Promise<string>;

  // 显示交互式选择器，例如选择浏览器 OAuth 或设备码
  onSelect(params: {
    message: string;
    options: { id: string; label: string }[];
  }): Promise<string | undefined>;
}
```

### OAuthCredentials

凭据会持久化到 `~/.pi/agent/auth.json`：

```typescript
interface OAuthCredentials {
  refresh: string;   // 刷新 Token（供 refreshToken() 使用）
  access: string;    // 访问 Token（由 getApiKey() 返回）
  expires: number;   // 到期时间戳，单位为毫秒
}
```

<a id="custom-streaming-api"></a>

## 自定义流式 API

对于使用非标准 API 的服务提供商，需要实现 `streamSimple`。编写实现前，请先研究现有的服务提供商实现：

**参考实现：**

- [anthropic.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/providers/anthropic.ts) - Anthropic Messages API
- [mistral.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/providers/mistral.ts) - Mistral Conversations API
- [openai-completions.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/providers/openai-completions.ts) - OpenAI Chat Completions
- [openai-responses.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/providers/openai-responses.ts) - OpenAI Responses API
- [google.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/providers/google.ts) - Google Generative AI
- [amazon-bedrock.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/providers/amazon-bedrock.ts) - AWS Bedrock

### 流处理模式

所有服务提供商都遵循相同的模式：

```typescript
import {
  type AssistantMessage,
  type AssistantMessageEventStream,
  type Context,
  type Model,
  type SimpleStreamOptions,
  calculateCost,
  createAssistantMessageEventStream,
} from "@earendil-works/pi-ai";

function streamMyProvider(
  model: Model<any>,
  context: Context,
  options?: SimpleStreamOptions
): AssistantMessageEventStream {
  const stream = createAssistantMessageEventStream();

  (async () => {
    // 初始化输出消息
    const output: AssistantMessage = {
      role: "assistant",
      content: [],
      api: model.api,
      provider: model.provider,
      model: model.id,
      usage: {
        input: 0,
        output: 0,
        cacheRead: 0,
        cacheWrite: 0,
        totalTokens: 0,
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
      },
      stopReason: "stop",
      timestamp: Date.now(),
    };

    try {
      // 推送开始事件
      stream.push({ type: "start", partial: output });

      // 发起 API 请求并处理响应……
      // 收到内容时推送相应事件……

      // 推送完成事件
      stream.push({
        type: "done",
        reason: output.stopReason as "stop" | "length" | "toolUse",
        message: output
      });
      stream.end();
    } catch (error) {
      output.stopReason = options?.signal?.aborted ? "aborted" : "error";
      output.errorMessage = error instanceof Error ? error.message : String(error);
      stream.push({ type: "error", reason: output.stopReason, error: output });
      stream.end();
    }
  })();

  return stream;
}
```

### 事件类型

通过 `stream.push()` 按以下顺序推送事件：

1. `{ type: "start", partial: output }`——流开始

2. 内容事件（可以重复；每个内容块都通过 `contentIndex` 跟踪）：
   - `{ type: "text_start", contentIndex, partial }`——文本块开始
   - `{ type: "text_delta", contentIndex, delta, partial }`——文本片段
   - `{ type: "text_end", contentIndex, content, partial }`——文本块结束
   - `{ type: "thinking_start", contentIndex, partial }`——思考开始
   - `{ type: "thinking_delta", contentIndex, delta, partial }`——思考片段
   - `{ type: "thinking_end", contentIndex, content, partial }`——思考结束
   - `{ type: "toolcall_start", contentIndex, partial }`——工具调用开始
   - `{ type: "toolcall_delta", contentIndex, delta, partial }`——工具调用 JSON 片段
   - `{ type: "toolcall_end", contentIndex, toolCall, partial }`——工具调用结束

3. `{ type: "done", reason, message }` 或 `{ type: "error", reason, error }`——流结束

每个事件的 `partial` 字段都包含 `AssistantMessage` 的当前状态。收到数据时先更新 `output.content`，然后将 `output` 作为 `partial` 放入事件。

### 内容块

收到内容块时，将其添加到 `output.content`：

```typescript
// 文本块
output.content.push({ type: "text", text: "" });
stream.push({ type: "text_start", contentIndex: output.content.length - 1, partial: output });

// 收到文本时
const block = output.content[contentIndex];
if (block.type === "text") {
  block.text += delta;
  stream.push({ type: "text_delta", contentIndex, delta, partial: output });
}

// 内容块完成时
stream.push({ type: "text_end", contentIndex, content: block.text, partial: output });
```

### 工具调用

工具调用需要累积并解析 JSON：

```typescript
// 开始工具调用
output.content.push({
  type: "toolCall",
  id: toolCallId,
  name: toolName,
  arguments: {}
});
stream.push({ type: "toolcall_start", contentIndex: output.content.length - 1, partial: output });

// 累积 JSON
let partialJson = "";
partialJson += jsonDelta;
try {
  block.arguments = JSON.parse(partialJson);
} catch {}
stream.push({ type: "toolcall_delta", contentIndex, delta: jsonDelta, partial: output });

// 完成
stream.push({
  type: "toolcall_end",
  contentIndex,
  toolCall: { type: "toolCall", id, name, arguments: block.arguments },
  partial: output
});
```

### 用量与费用

根据 API 响应更新用量并计算费用：

```typescript
output.usage.input = response.usage.input_tokens;
output.usage.output = response.usage.output_tokens;
output.usage.cacheRead = response.usage.cache_read_tokens ?? 0;
output.usage.cacheWrite = response.usage.cache_write_tokens ?? 0;
output.usage.totalTokens = output.usage.input + output.usage.output +
                           output.usage.cacheRead + output.usage.cacheWrite;
calculateCost(model, output.usage);
```

<a id="context-overflow-errors"></a>

### 上下文溢出错误

当请求超出模型的上下文窗口时，pi 可以通过压缩对话并重试来自动恢复。只有当 pi 将该故障识别为上下文溢出时，才会启用这种恢复机制。

pi 会检查最终生成的助手消息：

- `stopReason === "error"`
- `errorMessage` 匹配 pi 已知的某一种溢出模式（参见 [`packages/ai/src/utils/overflow.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/utils/overflow.ts)）

如果服务提供商返回的溢出错误消息无法被 pi 识别，请在注册该服务提供商的同一个扩展中对错误进行规范化。使用 `message_end` 处理函数改写助手消息，使其 `errorMessage` 以 pi 能够识别的短语开头。通用回退值 `context_length_exceeded` 是最稳妥的选择。

```typescript
const MY_PROVIDER_OVERFLOW_PATTERN = /your provider's overflow phrase/i;

export default function (pi: ExtensionAPI) {
  pi.registerProvider("my-provider", { /* ... */ });

  pi.on("message_end", (event, ctx) => {
    const message = event.message;
    if (message.role !== "assistant") return;
    if (message.stopReason !== "error") return;
    if (
      message.provider !== "my-provider" &&
      ctx.model?.provider !== "my-provider"
    )
      return;

    const errorMessage = message.errorMessage ?? "";
    if (errorMessage.includes("context_length_exceeded")) return;
    if (!MY_PROVIDER_OVERFLOW_PATTERN.test(errorMessage)) return;

    return {
      message: {
        ...message,
        errorMessage: `context_length_exceeded: ${errorMessage}`,
      },
    };
  });
}
```

`message_end` 会在 pi 为自动压缩记录助手消息之前运行，因此 pi 实际检查的是改写后的 `errorMessage`。配置完成后，pi 会：

1. 根据 `errorMessage` 识别溢出。
2. 从当前上下文中移除失败的助手消息。
3. 执行上下文压缩。
4. 重试一次请求。

改写错误时必须严格限定范围：

- 通过 `message.provider` 和 `ctx.model?.provider` 将处理范围限制在自己的服务提供商，避免改动其他服务提供商的无关错误。
- 匹配该服务提供商特有的错误模式，而不是 pi 的通用溢出模式。如果错误地改写限流或节流错误（`rate limit`、`too many requests`），就会误触发上下文压缩，而不是走 pi 正常的退避重试流程。
- 如果 `errorMessage` 已包含 `context_length_exceeded`，则跳过处理，保证处理函数具有幂等性。

### 注册

注册流处理函数：

```typescript
pi.registerProvider("my-provider", {
  baseUrl: "https://api.example.com",
  apiKey: "$MY_API_KEY",
  api: "my-custom-api",
  models: [...],
  streamSimple: streamMyProvider
});
```

<a id="testing-your-implementation"></a>

## 测试实现

请使用内置服务提供商所采用的同一套测试来验证自己的服务提供商。可从 [packages/ai/test/](https://github.com/earendil-works/pi-mono/tree/main/packages/ai/test) 复制并改造以下测试文件：

| 测试 | 用途 |
|------|---------|
| `stream.test.ts` | 基础流式传输和文本输出 |
| `tokens.test.ts` | Token 计数和用量 |
| `abort.test.ts` | `AbortSignal` 处理 |
| `empty.test.ts` | 空响应和最小响应 |
| `context-overflow.test.ts` | 上下文窗口限制 |
| `image-limits.test.ts` | 图像输入处理 |
| `unicode-surrogate.test.ts` | Unicode 边界情况 |
| `tool-call-without-result.test.ts` | 工具调用边界情况 |
| `image-tool-result.test.ts` | 工具结果中的图像 |
| `total-tokens.test.ts` | 总 Token 数计算 |
| `cross-provider-handoff.test.ts` | 服务提供商之间的上下文交接 |

使用自己的服务提供商/模型组合运行这些测试，以验证兼容性。

<a id="config-reference"></a>

## 配置参考

```typescript
interface ProviderConfig {
  /** 服务提供商在 /login 等界面中显示的名称。 */
  name?: string;

  /** API 端点 URL。定义模型时必填。 */
  baseUrl?: string;

  /** API 密钥字面值、环境变量插值（$ENV_VAR 或 ${ENV_VAR}）或 !command。定义模型时必填（使用 oauth 时除外）。 */
  apiKey?: string;

  /** 流式传输使用的 API 类型。定义模型时，必须在服务提供商或模型级别提供。 */
  api?: Api;

  /** 非标准 API 的自定义流式传输实现。 */
  streamSimple?: (
    model: Model<Api>,
    context: Context,
    options?: SimpleStreamOptions
  ) => AssistantMessageEventStream;

  /** 请求中包含的自定义请求头。其值采用与 apiKey 相同的解析语法。 */
  headers?: Record<string, string>;

  /** 如果为 true，则使用解析后的 API 密钥添加 Authorization: Bearer 请求头。 */
  authHeader?: boolean;

  /** 要注册的模型。提供后会替换该服务提供商的全部现有模型。 */
  models?: ProviderModelConfig[];

  /** 为 /login 提供支持的 OAuth 服务提供商。 */
  oauth?: {
    name: string;
    login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
    refreshToken(credentials: OAuthCredentials): Promise<OAuthCredentials>;
    getApiKey(credentials: OAuthCredentials): string;
  };
}
```

<a id="model-definition-reference"></a>

## 模型定义参考

```typescript
interface ProviderModelConfig {
  /** 模型 ID（例如 "claude-sonnet-4-20250514"）。 */
  id: string;

  /** 显示名称（例如 "Claude 4 Sonnet"）。 */
  name: string;

  /** 为该模型覆盖 API 类型。 */
  api?: Api;

  /** 为该模型覆盖 API 端点 URL。 */
  baseUrl?: string;

  /** 模型是否支持扩展思考。 */
  reasoning: boolean;

  /** 将 pi 的思考级别映射到服务提供商/模型特有的值；null 表示不支持该级别。 */
  thinkingLevelMap?: Partial<Record<"off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max", string | null>>;

  /** 支持的输入类型。 */
  input: ("text" | "image")[];

  /** 每百万 Token 的费用（用于用量跟踪）。 */
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };

  /** 上下文窗口的最大 Token 数。 */
  contextWindow: number;

  /** 最大输出 Token 数。 */
  maxTokens: number;

  /** 该模型特有的自定义请求头。 */
  headers?: Record<string, string>;

  /** 所选 API 的兼容性设置。 */
  compat?: {
    // openai-completions
    supportsStore?: boolean;
    supportsDeveloperRole?: boolean;
    supportsReasoningEffort?: boolean;
    supportsUsageInStreaming?: boolean;
    supportsStrictMode?: boolean;
    supportsOpenAIGrammarTools?: boolean; // 适用于 openai-completions/openai-responses；false 时退化为普通函数工具
    maxTokensField?: "max_completion_tokens" | "max_tokens";
    requiresToolResultName?: boolean;
    requiresAssistantAfterToolResult?: boolean;
    requiresThinkingAsText?: boolean;
    requiresReasoningContentOnAssistantMessages?: boolean;
    thinkingFormat?: "openai" | "openrouter" | "deepseek" | "together" | "zai" | "qwen" | "chat-template" | "qwen-chat-template" | "string-thinking" | "ant-ling";
    chatTemplateKwargs?: Record<string, string | number | boolean | null | { "$var": "thinking.enabled" | "thinking.effort"; omitWhenOff?: boolean }>;
    cacheControlFormat?: "anthropic";
    sessionAffinityFormat?: "openai" | "openai-nosession" | "openrouter";
    sendSessionAffinityHeaders?: boolean;

    // anthropic-messages
    supportsEagerToolInputStreaming?: boolean;
    supportsLongCacheRetention?: boolean;
    sendSessionAffinityHeaders?: boolean;
    supportsCacheControlOnTools?: boolean;
    forceAdaptiveThinking?: boolean;
    allowEmptySignature?: boolean;
    supportsStrictTools?: boolean;
  };
}
```

`openrouter` 发送 `reasoning: { effort }`。`deepseek` 发送 `thinking: { type: "enabled" | "disabled" }`，启用时还会发送 `reasoning_effort`。`together` 发送 `reasoning: { enabled }`；启用 `supportsReasoningEffort` 时也会发送 `reasoning_effort`。`qwen` 用于 DashScope 风格的顶层 `enable_thinking`。对于读取 `chat_template_kwargs.enable_thinking` 且需要 `preserve_thinking` 的本地 Qwen 兼容服务器，请使用 `qwen-chat-template`。需要可配置 `chat_template_kwargs` 时使用 `chat-template`；例如，通过 vLLM 部署的 DeepSeek V3.x 可配置 `chatTemplateKwargs: { "thinking": { "$var": "thinking.enabled" } }`。
`cacheControlFormat: "anthropic"` 会把 Anthropic 风格的 `cache_control` 标记应用到系统提示词、最后一个工具定义，以及最后一段用户、助手或工具结果文本内容。
