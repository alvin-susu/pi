# 服务提供商

Pi 通过 OAuth 支持订阅制服务提供商，并通过环境变量或认证文件支持 API 密钥型服务提供商。Pi 随附内置模型目录；已配置的服务提供商可以刷新较新的目录，并将其缓存在 `~/.pi/agent/models-store.json` 中，供离线使用。

## 目录

- [订阅](#subscriptions)
- [API 密钥](#api-keys)
- [认证文件](#auth-file)
- [云服务提供商](#cloud-providers)
- [llama.cpp](#llamacpp)
- [自定义服务提供商](#custom-providers)
- [解析顺序](#resolution-order)

<a id="subscriptions"></a>

## 订阅

在交互模式中使用 `/login`，然后选择服务提供商：

- ChatGPT Plus/Pro (Codex)
- Claude Pro/Max
- GitHub Copilot
- xAI（Grok/X 订阅）
- OpenRouter（通过 OAuth 签发 API 密钥，费用从 OpenRouter 余额中扣除）
- Radius

使用 `/logout` 清除凭据。Token 保存在 `~/.pi/agent/auth.json` 中，过期后会自动刷新。OpenRouter 的处理方式不同：它会签发一个由用户控制、不会自动过期的 API 密钥。

### OpenAI Codex

- 需要 ChatGPT Plus 或 Pro 订阅
- 已获 OpenAI 官方认可：[面向开源软件的 Codex](https://developers.openai.com/community/codex-for-oss)

### Claude Pro/Max

Claude Pro/Max 账号可以使用 Anthropic 订阅认证。第三方运行框架产生的用量计入[额外用量](https://claude.ai/settings/usage)，按 Token 计费，不占用 Claude 套餐限额。

### GitHub Copilot

- 如果使用 github.com，直接按 Enter；否则输入 GitHub Enterprise Server 域名
- 如果出现“model not supported（模型不受支持）”，请在 VS Code 中启用该模型：Copilot Chat → 模型选择器 → 选择模型 → “Enable（启用）”

### xAI（Grok/X 订阅）

- 运行 `/login xai`，然后选择 **Use a subscription（使用订阅）**
- 仍可通过 **Use an API key（使用 API 密钥）** 配置 `XAI_API_KEY`

### OpenRouter

- 运行 `/login openrouter`，然后选择 **Sign in with OpenRouter（使用 OpenRouter 登录）**，打开 OpenRouter PKCE 授权流程
- 授权过程会创建一个由用户控制的 OpenRouter API 密钥，费用从 OpenRouter 余额中扣除
- 仍可通过 **Use an API key（使用 API 密钥）** 配置 `OPENROUTER_API_KEY`

### Radius

Radius 是动态 `pi-messages` 网关。`/login radius` 将 OAuth Token 保存到 `auth.json`；网关目录会独立刷新，并缓存在 `models-store.json` 中。可以在 `models.json` 中使用 `"oauth": "radius"` 和网关 `baseUrl` 声明自定义 Radius 网关。

<a id="api-keys"></a>

## API 密钥

<a id="environment-variables-or-auth-file"></a>

### 环境变量或认证文件

在交互模式中使用 `/login` 并选择服务提供商，可以将 API 密钥保存到 `auth.json`；也可以通过环境变量设置凭据：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

| 服务提供商 | 环境变量 | `auth.json` 键 |
|----------|----------------------|------------------|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| Ant Ling | `ANT_LING_API_KEY` | `ant-ling` |
| Azure OpenAI Responses | `AZURE_OPENAI_API_KEY` | `azure-openai-responses` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| NVIDIA NIM | `NVIDIA_API_KEY` | `nvidia` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| Amazon Bedrock | `AWS_BEARER_TOKEN_BEDROCK` | `amazon-bedrock` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| Groq | `GROQ_API_KEY` | `groq` |
| Cerebras | `CEREBRAS_API_KEY` | `cerebras` |
| Cloudflare AI Gateway | `CLOUDFLARE_API_KEY` (+ `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_GATEWAY_ID`) | `cloudflare-ai-gateway` |
| Cloudflare Workers AI | `CLOUDFLARE_API_KEY` (+ `CLOUDFLARE_ACCOUNT_ID`) | `cloudflare-workers-ai` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` | `vercel-ai-gateway` |
| ZAI Coding Plan（全球） | `ZAI_API_KEY` | `zai` |
| ZAI Coding Plan（中国） | `ZAI_CODING_CN_API_KEY` | `zai-coding-cn` |
| OpenCode Zen | `OPENCODE_API_KEY` | `opencode` |
| OpenCode Go | `OPENCODE_API_KEY` | `opencode-go` |
| Radius | `RADIUS_API_KEY` | `radius` |
| Hugging Face | `HF_TOKEN` | `huggingface` |
| Fireworks | `FIREWORKS_API_KEY` | `fireworks` |
| Together AI | `TOGETHER_API_KEY` | `together` |
| Kimi For Coding | `KIMI_API_KEY` | `kimi-coding` |
| MiniMax | `MINIMAX_API_KEY` | `minimax` |
| MiniMax（中国） | `MINIMAX_CN_API_KEY` | `minimax-cn` |
| Qwen Token Plan | `QWEN_TOKEN_PLAN_API_KEY` | `qwen-token-plan` |
| Qwen Token Plan（中国） | `QWEN_TOKEN_PLAN_CN_API_KEY` | `qwen-token-plan-cn` |
| Xiaomi MiMo | `XIAOMI_API_KEY` | `xiaomi` |
| Xiaomi MiMo Token Plan（中国） | `XIAOMI_TOKEN_PLAN_CN_API_KEY` | `xiaomi-token-plan-cn` |
| Xiaomi MiMo Token Plan（阿姆斯特丹） | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` | `xiaomi-token-plan-ams` |
| Xiaomi MiMo Token Plan（新加坡） | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` | `xiaomi-token-plan-sgp` |

环境变量和 `auth.json` 键的权威定义，请参阅 [`packages/ai/src/env-api-keys.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/env-api-keys.ts) 中的 [`const envMap`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/env-api-keys.ts)。

<a id="auth-file"></a>

#### 认证文件

将凭据保存在 `~/.pi/agent/auth.json` 中：

```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-..." },
  "ant-ling": { "type": "api_key", "key": "..." },
  "openai": { "type": "api_key", "key": "sk-..." },
  "deepseek": { "type": "api_key", "key": "sk-..." },
  "nvidia": { "type": "api_key", "key": "nvapi-..." },
  "google": { "type": "api_key", "key": "..." },
  "opencode": { "type": "api_key", "key": "..." },
  "opencode-go": { "type": "api_key", "key": "..." },
  "together": { "type": "api_key", "key": "..." },
  "qwen-token-plan":  { "type": "api_key", "key": "sk-sp-..." },
  "qwen-token-plan-cn": { "type": "api_key", "key": "sk-sp-..." },
  "xiaomi": { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-cn":  { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-ams": { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-sgp": { "type": "api_key", "key": "..." }
}
```

该文件以 `0600` 权限创建（只有当前用户可以读写）。认证文件中的凭据优先于环境变量。

API 密钥凭据还可以包含限定于该服务提供商的环境值。解析凭据密钥、服务提供商/模型请求头，以及 Cloudflare 账号 ID、Azure OpenAI 设置、Vertex 项目/区域、Bedrock 设置、`PI_CACHE_RETENTION`、`HTTP_PROXY`/`HTTPS_PROXY` 等服务提供商配置时，这些值的优先级高于进程环境变量。

```json
{
  "cloudflare-ai-gateway": {
    "type": "api_key",
    "key": "$CLOUDFLARE_API_KEY",
    "env": {
      "CLOUDFLARE_API_KEY": "...",
      "CLOUDFLARE_ACCOUNT_ID": "account-id",
      "CLOUDFLARE_GATEWAY_ID": "gateway-id"
    }
  }
}
```

如果 Pi 应使用与项目 Shell 环境不同的服务提供商设置，请使用这种方式。

### 密钥值解析

`key` 字段支持执行命令、环境变量插值和字面量：

- **Shell 命令：** 如果值以 `"!command"` 开头，会将整个值作为命令执行，并使用其标准输出（在进程生命周期内缓存）
  ```json
  { "type": "api_key", "key": "!security find-generic-password -ws 'anthropic'" }
  { "type": "api_key", "key": "!op read 'op://vault/item/credential'" }
  ```
- **环境变量插值：** `"$ENV_VAR"` 或 `"${ENV_VAR}"` 使用对应环境变量的值。较长的字面量中也可以使用插值。
  ```json
  { "type": "api_key", "key": "$MY_ANTHROPIC_KEY" }
  { "type": "api_key", "key": "${KEY_PREFIX}_${KEY_SUFFIX}" }
  ```
  `$FOO_BAR` 表示变量 `FOO_BAR`；如果 `BAR` 是字面文本，请使用 `${FOO}_BAR`。缺少环境变量时，该值无法解析。
- **转义：** `"$$"` 生成字面量 `"$"`；`"$!"` 生成字面量 `"!"`，但不会触发命令执行。
  ```json
  { "type": "api_key", "key": "$$literal-dollar-prefix" }
  { "type": "api_key", "key": "$!literal-bang-prefix" }
  ```
- **字面值：** 直接使用。`MY_API_KEY` 之类的纯大写字符串也是字面量；要引用环境变量，请使用 `$MY_API_KEY`。
  ```json
  { "type": "api_key", "key": "sk-ant-..." }
  { "type": "api_key", "key": "public" }
  ```

通过 `/login` 获得的 OAuth 凭据也保存在此处，并由 Pi 自动管理。

<a id="cloud-providers"></a>

## 云服务提供商

### Azure OpenAI

```bash
export AZURE_OPENAI_API_KEY=...
export AZURE_OPENAI_BASE_URL=https://your-resource.ai.azure.com
# 也支持：https://your-resource.cognitiveservices.azure.com
# 也支持：https://your-resource.openai.azure.com
# 根端点会自动规范化为 /openai/v1
# 也可以使用资源名称代替基础 URL
export AZURE_OPENAI_RESOURCE_NAME=your-resource

# 可选
export AZURE_OPENAI_API_VERSION=2024-02-01
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP=gpt-4=my-gpt4,gpt-4o=my-gpt4o
```

### Amazon Bedrock

使用 `/login amazon-bedrock` 保存 Bedrock API 密钥，或配置以下任一种 AWS 环境凭据来源：

```bash
# 方案一：AWS 配置文件
export AWS_PROFILE=your-profile

# 方案二：IAM 密钥
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...

# 方案三：Bearer Token
export AWS_BEARER_TOKEN_BEDROCK=...

# 可选区域（默认为 us-east-1）
export AWS_REGION=us-west-2
```

还支持 ECS 任务角色（`AWS_CONTAINER_CREDENTIALS_*`）和 IRSA（`AWS_WEB_IDENTITY_TOKEN_FILE`）。

```bash
pi --provider amazon-bedrock --model us.anthropic.claude-sonnet-4-20250514-v1:0
```

如果 Claude 模型 ID 中包含可识别的模型名称（基础模型和系统定义的推理配置文件），Pi 会自动启用提示词缓存。对于应用程序推理配置文件（其 ARN 不包含模型名称），请设置 `AWS_BEDROCK_FORCE_CACHE=1` 以启用缓存点：

```bash
export AWS_BEDROCK_FORCE_CACHE=1
pi --provider amazon-bedrock --model arn:aws:bedrock:us-east-1:123456789012:application-inference-profile/abc123
```

连接 Bedrock API 代理时，可以使用以下环境变量：

```bash
# 设置 Bedrock 代理 URL（标准 AWS SDK 环境变量）
export AWS_ENDPOINT_URL_BEDROCK_RUNTIME=https://my.corp.proxy/bedrock

# 如果代理不需要认证，请设置
export AWS_BEDROCK_SKIP_AUTH=1

# 如果代理只支持 HTTP/1.1，请设置
export AWS_BEDROCK_FORCE_HTTP1=1
```

### Cloudflare AI Gateway

可以通过 `/login` 设置 `CLOUDFLARE_API_KEY`。账号 ID 和网关短名称可以通过环境变量设置，也可以写入 `auth.json` 中 API 密钥凭据的 `env` 对象。

```bash
export CLOUDFLARE_API_KEY=...           # 或使用 /login
export CLOUDFLARE_ACCOUNT_ID=...
export CLOUDFLARE_GATEWAY_ID=...        # 在 dash.cloudflare.com → AI → AI Gateway 中创建
pi --provider cloudflare-ai-gateway --model "claude-sonnet-4-5"
```

通过 Cloudflare AI Gateway 将请求路由到 OpenAI、Anthropic 和 Workers AI。Workers AI 使用统一 API（`/compat`）和带前缀的模型 ID（`workers-ai/@cf/...`）。OpenAI 使用 OpenAI 直通路由（`/openai`）以及 `gpt-5.1` 等原生 OpenAI 模型 ID。Anthropic 使用 Anthropic 直通路由（`/anthropic`）以及 `claude-sonnet-4-5` 等原生 Anthropic 模型 ID。

AI Gateway 认证将 `CLOUDFLARE_API_KEY` 用作 `cf-aig-authorization`。上游认证可以采用以下方式之一：

| 模式 | 请求认证 | 上游认证 |
|------|--------------|---------------|
| Workers AI | 仅 Cloudflare Token | Cloudflare 原生认证 |
| 统一计费 | 仅 Cloudflare Token | Cloudflare 处理上游认证并扣除余额 |
| 已存储 BYOK | 仅 Cloudflare Token | Cloudflare 注入保存在 AI Gateway 控制台中的服务提供商密钥 |
| 内联 BYOK | Cloudflare Token 加上游 `Authorization` 请求头 | 由请求提供上游服务提供商密钥 |

正常使用 Pi 时，建议选择统一计费或已存储 BYOK。内联 BYOK 需要为 Cloudflare AI Gateway 服务提供商额外配置上游 `Authorization` 请求头，例如通过 `models.json` 中的服务提供商/模型覆盖配置。

### Cloudflare Workers AI

可以通过 `/login` 设置 `CLOUDFLARE_API_KEY`。`CLOUDFLARE_ACCOUNT_ID` 可以通过环境变量设置，也可以写入 `auth.json` 中 API 密钥凭据的 `env` 对象。

```bash
export CLOUDFLARE_API_KEY=...           # 或使用 /login
export CLOUDFLARE_ACCOUNT_ID=...
pi --provider cloudflare-workers-ai --model "@cf/moonshotai/kimi-k2.6"
```

Pi 会自动设置 `x-session-affinity`，以便获得[前缀缓存](https://developers.cloudflare.com/workers-ai/features/prompt-caching/)优惠。

### Google Vertex AI

使用应用默认凭据（Application Default Credentials）：

```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT=your-project
export GOOGLE_CLOUD_LOCATION=us-central1
```

也可以将 `GOOGLE_APPLICATION_CREDENTIALS` 指向服务账号密钥文件。

<a id="llamacpp"></a>

## llama.cpp

Pi 支持 llama.cpp 路由服务器。使用 `/login llama.cpp` 进行配置，通过 `/llama` 管理已加载模型，并通过 `/model` 选择已加载模型。

服务器配置、模型目录布局、环境变量和命令用法详见 [llama.cpp](llama-cpp-zh.md)。

<a id="custom-providers"></a>

## 自定义服务提供商

**通过 models.json：** 添加 Ollama、LM Studio、vLLM，或任何使用受支持 API（OpenAI Completions、OpenAI Responses、Anthropic Messages、Google Generative AI）的服务提供商。参阅 [models.md](models-zh.md)。

**通过扩展：** 如果服务提供商需要自定义 API 实现或 OAuth 流程，请创建扩展。参阅 [custom-provider.md](custom-provider-zh.md) 和 [examples/extensions/custom-provider-gitlab-duo](../examples/extensions/custom-provider-gitlab-duo/)。

<a id="resolution-order"></a>

## 解析顺序

解析服务提供商凭据时，优先级依次为：

1. CLI `--api-key` 标志
2. `auth.json` 条目（API 密钥或 OAuth Token）
3. 环境变量
4. `models.json` 中的自定义服务提供商密钥
