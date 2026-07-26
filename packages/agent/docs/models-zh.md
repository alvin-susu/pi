# Models 架构

本文介绍下一轮 `pi-ai` 模型/服务提供商重构的目标设计。这里描述的是期望形态，而不是当前实现；内容应足够完整，使开发者可以在新会话中直接开始实现。

目标：

- `Models` 只是简单的运行时服务提供商集合。
- 具体服务提供商负责元数据、身份验证、模型列表和流式传输行为。
- API 实现位于 `src/api/` 下，可以复用并延迟加载。
- 具体服务提供商工厂位于 `src/providers/` 下。
- 用户可以只导入所需的服务提供商。
- 导入服务提供商时不能立即导入体积较大的 SDK。
- 将动态模型列表作为一等能力：同步读取最近已知列表，通过显式的异步 `refresh` 获取最新列表。
- `models.json` 和扩展通过封装服务提供商来叠加能力，不随意修改服务提供商内部状态。
- 旧的全局 API 只保留在显式且临时的 `/compat` 入口点中。

本轮 `pi-ai` 工作暂不包含：

- 暂不迁移 coding-agent 的 `ModelRegistry`。
- 不在 `Models` 内保留流/API 注册表。
- 暂不实现 Web OAuth 流程。
- 图像生成采用与对话侧对应的设计（`images-models.ts` 中的 `ImagesModels`/`ImagesProvider`）；旧的全局图像 API（`images.ts`、`images-api-registry.ts`）保留在兼容入口中。

## 包布局

目标源码布局：

```txt
packages/ai/src/
  index.ts                    # 只导出核心内容；不导入内置服务提供商
  models.ts                   # Models 运行时、Provider
  images-models.ts            # ImagesModels 运行时、ImagesProvider（与 models.ts 对应）
  compat.ts                   # 临时的旧 API 兼容入口
  auth/                       # 身份验证方法类型、辅助函数、共享 resolveProviderAuth()、登录回调
  api/                        # API 实现和延迟加载封装器
    openai-completions.ts     # 真实实现；导入 SDK，导出 stream/streamSimple
    openai-completions.lazy.ts
    openai-responses.ts
    openai-responses.lazy.ts
    openai-codex-responses.ts
    openai-codex-responses.lazy.ts
    azure-openai-responses.ts
    azure-openai-responses.lazy.ts
    anthropic-messages.ts
    anthropic-messages.lazy.ts
    google-generative-ai.ts
    google-generative-ai.lazy.ts
    google-vertex.ts
    google-vertex.lazy.ts
    mistral-conversations.ts
    mistral-conversations.lazy.ts
    bedrock-converse-stream.ts
    bedrock-converse-stream.lazy.ts
    openrouter-images.ts      # 图像生成 API 实现
    openrouter-images.lazy.ts
    lazy.ts                   # lazyStream()/lazyApi() 辅助函数
    （共享辅助模块：openai-responses-shared、google-shared、transform-messages……）
  providers/                  # 具体服务提供商工厂及各服务提供商的模型目录
    openai.ts
    openai.models.ts          # 生成的 OpenAI 模型目录
    openai-codex.ts
    openai-codex.models.ts
    anthropic.ts
    anthropic.models.ts
    google.ts
    google.models.ts
    ……每个内置服务提供商对应一组文件……
    openrouter-images.ts      # 图像生成服务提供商工厂
    faux.ts                   # 测试服务提供商工厂
    all.ts                    # 显式聚合入口：builtinModels()、builtinImagesModels()、getBuiltin*()
  auth/oauth/                 # 标准 OAuth 实现（Node），延迟加载
```

`src/index.ts` 必须只包含核心内容，不能导入：

- 生成的模型目录
- 内置服务提供商工厂
- 服务提供商 SDK 实现
- Node 专用 OAuth 模块
- `providers/all`
- `compat`

服务提供商、API 和兼容入口点均通过显式子路径导出。

## 公开用法

使用单个服务提供商的最小示例：

```ts
import { createModels } from "@earendil-works/pi-ai";
import { openaiProvider } from "@earendil-works/pi-ai/providers/openai";

const models = createModels();
models.setProvider(openaiProvider());

const model = models.getModel("openai", "gpt-4o-mini");
if (!model) throw new Error("未找到模型");

const response = await models.complete(model, context);
```

多个服务提供商：

```ts
const models = createModels();
models.setProvider(openaiProvider());
models.setProvider(openrouterProvider());
```

全部内置服务提供商（显式使用元数据较重的入口点）：

```ts
import { builtinModels } from "@earendil-works/pi-ai/providers/all";

const models = builtinModels();
```

`providers/all` 可以导入所有服务提供商元数据/模型目录，但仍不能立即导入 SDK 实现；服务提供商的数据流使用延迟加载封装器。

## 核心运行时：Models

`Models` 是服务提供商集合，同时负责应用身份验证信息，并提供便捷的流式传输方法。它不包含流注册表，也不包含身份验证解析策略对象。

```ts
export function createModels(options?: {
  /** 由应用管理的凭据存储。默认使用内存存储。 */
  credentials?: CredentialStore;
  /** 身份验证解析所需的环境访问能力（环境变量、文件是否存在）。默认基于 process.env/node:fs；测试和非 Node 宿主可注入实现。 */
  authContext?: AuthContext;
}): MutableModels;

export interface Models {
  getProviders(): readonly Provider[];
  getProvider(id: string): Provider | undefined;

  /** 同步读取最近已知的模型。尽力而为：如果某个服务提供商的 getModels() 抛错，则不返回该服务提供商的模型。 */
  getModels(provider?: string): readonly Model<Api>[];
  /** 动态列表的准确类型是 Model<Api>；使用 hasApi() 类型守卫进行收窄。 */
  getModel(provider: string, id: string): Model<Api> | undefined;

  /**
   * 要求动态服务提供商重新获取模型列表。传入服务提供商 ID 时，
   * 如果该服务提供商失败则拒绝；不传入时并发刷新全部服务提供商，
   * 并尽力完成。静态服务提供商不执行任何操作。
   */
  refresh(provider?: string): Promise<void>;

  /**
   * 解析模型请求所需的身份验证信息，并包含供状态 UI 使用的来源标签。
   * 服务提供商未知或未配置时解析为 undefined。失败时以 ModelsError
   * 拒绝（刷新失败使用 "oauth"，API 密钥/存储失败使用 "auth"）；
   * 状态/可用性 UI 会捕获拒绝并显示“需要重新登录”，而不是视为未配置。
   */
  getAuth(model: Model<Api>): Promise<AuthResult | undefined>;

  stream<TApi extends Api>(
    model: Model<TApi>,
    context: Context,
    options?: ApiStreamOptions<TApi>,
  ): AssistantMessageEventStream;

  complete<TApi extends Api>(
    model: Model<TApi>,
    context: Context,
    options?: ApiStreamOptions<TApi>,
  ): Promise<AssistantMessage>;

  streamSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
  completeSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): Promise<AssistantMessage>;
}

export interface MutableModels extends Models {
  /** 按 provider.id 插入或替换。服务提供商 ID 必须唯一。 */
  setProvider(provider: Provider): void;
  deleteProvider(id: string): void;
  clearProviders(): void;
}
```

已移除的概念：

```txt
不再提供 Models.setStreamFunctions() / getStreamFunctions()
不再将 api-registry 用作真正的分发机制
不再提供 Models.provider(id) 构建器，也不再提供 setModel/upsertModel/patchModel 生命周期
不再提供 ModelAuthResolver / setAuthResolver——解析策略固定，通过注入提供存储
```

如果应用需要不同的身份验证策略，可以封装服务提供商（封装身份验证方法或 `getModels`），也可以在流式传输选项中显式传入请求身份验证信息。

## 服务提供商

服务提供商是具体的运行时单元，负责 ID、名称、基础元数据、身份验证方法、模型列表和流式传输行为。

`Provider` 以其模型所使用的 API 为泛型参数。具体工厂会声明自身产出的类型（`openaiProvider(): Provider<"openai-responses" | "openai-completions">`），使直接使用工厂的用户获得类型化模型列表。`Models` 集合则以 `Provider<Api>` 保存服务提供商。

```ts
export interface Provider<TApi extends Api = Api> {
  readonly id: string;
  readonly name: string;

  readonly baseUrl?: string;
  readonly headers?: Record<string, string>;

  /**
   * 必填：apiKey/oauth 至少提供一个。即使是使用环境凭据的服务提供商
   *（环境变量、AWS 配置文件、ADC）或无密钥本地服务器，也要提供 apiKey
   * 身份验证，由其 resolve() 报告服务提供商是否已配置。
   * getAuth() 返回 undefined 表示未配置。
   */
  readonly auth: ProviderAuth;

  /** 当前已知模型，同步返回。静态服务提供商返回模型目录；动态服务提供商返回最近一次刷新的结果（首次刷新前为空）。 */
  getModels(): readonly Model<TApi>[];

  /** 仅供动态服务提供商使用：获取并更新模型列表。并发调用共享同一个正在执行的获取任务。 */
  refreshModels?(): Promise<void>;

  stream<T extends TApi>(model: Model<T>, context: Context, options?: ApiStreamOptions<T>): AssistantMessageEventStream;

  streamSimple(model: Model<TApi>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}
```

不存在 `Provider.api` 字段。`model.api` 携带 API 标识，由服务提供商在内部进行分发（参见 `createProvider()`）。

保留 `Model.api`：现有元数据和测试都在使用它，它也有助于诊断，而且构造服务提供商时要依据它选择 API 实现。不过，`Models` 绝不会根据该字段分发；分发由服务提供商负责。

### 类型化的流式传输选项

完整的流式传输选项因 API 而异。`Model<TApi>` 可以根据 API 派生选项类型，从而体现其价值：

```ts
// types.ts——从 API 实现模块进行的纯类型导入会被擦除，因此不会妨碍摇树优化
export interface ApiOptionsMap {
  "anthropic-messages": AnthropicOptions;
  "openai-completions": OpenAICompletionsOptions;
  "openai-responses": OpenAIResponsesOptions;
  "openai-codex-responses": OpenAICodexResponsesOptions;
  "azure-openai-responses": AzureOpenAIResponsesOptions;
  "google-generative-ai": GoogleOptions;
  "google-vertex": GoogleVertexOptions;
  "mistral-conversations": MistralOptions;
  "bedrock-converse-stream": BedrockOptions;
}

export type ApiStreamOptions<TApi extends Api> = TApi extends keyof ApiOptionsMap
  ? ApiOptionsMap[TApi]
  : StreamOptions & Record<string, unknown>;
```

自定义 API 字符串会回退到通用结构。

### 类型化的模型收窄

运行时模型列表是动态的，因此 `models.getModel()`/`getModels()` 会如实返回 `Model<Api>`。类型能力在以下三处得到增强：

1. **`hasApi()` 类型守卫**——对动态查找结果执行运行时检查并收窄类型，无需盲目断言：

   ```ts
   export function hasApi<TApi extends Api>(model: Model<Api>, api: TApi): model is Model<TApi>;

   const model = models.getModel("anthropic", "claude-opus-4-7");
   if (model && hasApi(model, "anthropic-messages")) {
     // model: Model<"anthropic-messages">，流式传输选项具有完整类型
   }
   ```

2. **`getBuiltinModel()`**——同步查询生成的模型目录，并提供类型化重载：`(provider, id) -> Model<精确的 API 字面量>`。适合查询硬编码的已知模型。

3. **`Provider<TApi>` 工厂**——不经过 `Models` 集合、直接使用服务提供商时，可以获得类型化模型列表。

有意不做的事情：若要让 `models.getModel(provider, ...)` 与类型化的服务提供商/模型 ID 绑定，就必须在静态阶段知道可变运行时集合中安装了哪些服务提供商。运行框架路径（`streamSimple` + `SimpleStreamOptions`）与具体 API 无关，不受影响。

作为对比，Vercel AI SDK 将实现附加到模型对象上，从而消除了分发类型问题，但也使模型无法序列化（会话、RPC 和模型目录无法再使用普通数据）；其 `providerOptions` 参数包是 `Record<string, JSON>`，仅通过 `satisfies` 约定进行检查。采用普通数据模型并由服务提供商负责行为，可以在关键位置保留更强的类型能力。

### 名称冲突

`types.ts` 当前导出 `type Provider = KnownProvider | string`（表示服务提供商 ID）。应将该别名重命名为 `ProviderId` 并修正调用位置，上述 `Provider` 接口则使用 `Provider` 这一名称。

## 服务提供商模型列表

读取采用同步方式，获取最新数据则是显式的异步操作。`Provider.getModels()` 返回当前已知列表：静态服务提供商返回完整模型目录，动态服务提供商返回最近一次刷新的列表（例如 llama.cpp、OpenRouter 实时列表）。动态服务提供商通过 `refreshModels()` 获取数据。

之所以进行这种拆分，是因为同步或异步联合类型（`Promise<T> | T`）容易让代码暗中形成同步假设，一旦遇到第一个异步服务提供商便会出错；如果只允许异步读取，又会迫使所有消费者（UI 列表、扩展的 `find`/`getAll` 接口）为几乎总是静态的数据处理 Promise。同步读取配合显式刷新，既能让数据可能过期这一事实清晰可见，又能保持单一契约：`getModels()` 表示最近已知值，`refresh()` 表示使其更新。获取到的列表在返回的一刻就可能过期，明确命名刷新时点才符合实际情况。

应用负责刷新生命周期，例如启动、重新加载注册表或打开模型选择器。对新鲜度有严格要求的查询分两步执行：`await models.refresh("llamacpp"); models.getModel("llamacpp", id)`。

动态刷新必须是无副作用的发现过程：

```txt
允许：获取 /v1/models、枚举本地模型目录、刷新已缓存的远程模型列表
不允许：加载模型、下载模型、修改服务器状态、发起探测请求
```

服务提供商专用的模型生命周期操作（加载/卸载）属于应用或服务提供商管理命令，不应放在 `refreshModels()` 中。

## 流式传输路径

`Models.stream()` 根据 `model.provider` 查找服务提供商，解析身份验证信息，将其合并进请求选项，再委托给服务提供商：

```ts
function stream(model, context, options) {
  const provider = this.getProvider(model.provider);
  if (!provider) {
    // 生成错误流，而不是抛出错误——参见“错误行为”
  }

  // 异步准备工作在返回的流内部进行（lazyStream 模式）
  const resolution = await this.getAuth(model);
  const requestModel = resolution?.auth.baseUrl ? { ...model, baseUrl: resolution.auth.baseUrl } : model;
  const requestOptions = mergeAuth(options, resolution?.auth); // 每个字段都以显式选项为准

  return provider.stream(requestModel, context, requestOptions);
}
```

`stream()` 同步返回 `AssistantMessageEventStream`；异步准备工作（身份验证解析、模块延迟加载）在返回的流内部进行。目前的 `register-builtins.ts` 中已经存在这种转发模式（`createLazyStream`）；应将其提取为 `src/api/lazy.ts` 中的 `lazyStream()`。

请求热路径不对模型进行规范化：`stream()` 原样使用传入的模型对象。如果应用需要最新的模型元数据，应在开始轮次前刷新服务提供商并重新读取（`await models.refresh(p); models.getModel(p, id)`）。

## `src/api` 下的 API 实现

API 实现是可复用的流式传输行为，不是服务提供商。

统一的导出契约——每个真实实现模块只导出：

```ts
// src/api/anthropic-messages.ts——导入 SDK
export function stream(model, context, options) { ... }
export function streamSimple(model, context, options) { ... }
```

这样模块本身就能满足 `ProviderStreams`，使延迟加载封装器可以使用一个通用辅助函数，而不必为每个 API 编写专用衔接代码。`ProviderStreams` 是不带具体类型的分发结构（实现模块导出具体类型的函数，无法赋给泛型方法）；每个 API 的选项类型由模块本身提供，并通过 `ApiStreamOptions` 体现在 `Provider.stream()` 上：

```ts
export interface ProviderStreams {
  stream(model: Model<Api>, context: Context, options?: StreamOptions): AssistantMessageEventStream;
  streamSimple(model: Model<Api>, context: Context, options?: SimpleStreamOptions): AssistantMessageEventStream;
}

// src/api/lazy.ts
export function lazyApi(load: () => Promise<ProviderStreams>): ProviderStreams;

// src/api/anthropic-messages.lazy.ts
export const anthropicMessagesApi = (): ProviderStreams => lazyApi(() => import("./anthropic-messages.ts"));
```

导入链：

```txt
服务提供商模块 → API 延迟加载封装器 → 动态导入（真实 API 实现）→ SDK 依赖
```

备注：

- Bedrock 在其延迟加载封装器中保留仅适用于 Node 的动态导入技巧（`importNodeOnlyProvider`、重写 `.ts`/`.js` 模块说明符）。Bun 构建所用的 `setBedrockProviderModule()` 移入 Bedrock 延迟加载封装器模块。
- 共享辅助模块（`openai-responses-shared.ts`、`google-shared.ts`、`transform-messages.ts`、提示缓存、Copilot 请求头）与实现一起移至 `src/api/`。

## 多个具体服务提供商共享 API 实现

许多具体服务提供商共享同一种 API 实现（例如使用 OpenAI Completions 的 OpenRouter、Groq、Cerebras、xAI、ZAI 等）。它们通过引用共享延迟加载的 API 对象：

```ts
import { openAICompletionsApi } from "../api/openai-completions.lazy.ts";

export function openrouterProvider(): Provider {
  return createProvider({
    id: "openrouter",
    name: "OpenRouter",
    baseUrl: "https://openrouter.ai/api/v1",
    auth: { apiKey: envApiKeyAuth("OpenRouter API key", ["OPENROUTER_API_KEY"]) },
    models: OPENROUTER_MODELS,
    api: openAICompletionsApi(),
  });
}
```

这借鉴了 Vercel AI SDK 的一个实用特性：用户导入具体服务提供商，共享协议的实现则留在内部。

## 身份验证

请求身份验证的输出保持精简：

```ts
export interface ModelAuth {
  apiKey?: string;
  headers?: Record<string, string>;
  baseUrl?: string;
}
```

如果某个值无法表示为 `apiKey`、`headers` 或 `baseUrl`，它就属于服务提供商配置，而不是身份验证信息（Vertex 的项目/位置、Bedrock 的区域/配置文件、Azure 的 `apiVersion` 都是服务提供商工厂选项）。

### 服务提供商身份验证

`Provider.auth` 只有两个槽位；实际服务提供商最多有一条 API 密钥路径和一条 OAuth 路径。槽位名称本身就能支持 UI 区分 OAuth 与 API 密钥，无需 `kind` 判别字段或方法 ID：

```ts
export interface ProviderAuth {
  apiKey?: ApiKeyAuth; // 已存储密钥/服务提供商环境配置 + 外部环境/文件/ADC/IAM
  oauth?: OAuthAuth;   // 登录流程 + 刷新
}

export interface ApiKeyAuth {
  name: string; // “Anthropic API key”

  /** 交互式设置（提示输入密钥/服务提供商环境配置）。不存在时表示只使用外部凭据（环境变量、ADC、IAM）。 */
  login?(interaction: AuthInteraction): Promise<ApiKeyCredential>;

  /**
   * 从已存储凭据和/或外部来源解析身份验证信息，并逐字段合并
   *（credential.key ?? env("...")、credential.env?.NAME ?? env("...")）。
   * undefined 表示未配置。
   */
  resolve(input: {
    model: Model<Api>;
    ctx: AuthContext;
    credential?: ApiKeyCredential;
  }): Promise<AuthResult | undefined>;
}

export interface OAuthAuth {
  name: string; // “Anthropic (Claude Pro/Max)”

  login(interaction: AuthInteraction): Promise<OAuthCredential>;

  /** 使用刷新 token 换取新凭据。该方法会发起网络调用，失败时抛出错误（invalid_grant 等），并在存储锁内运行。 */
  refresh(credential: OAuthCredential): Promise<OAuthCredential>;

  /** 根据有效凭据无副作用地派生请求身份验证信息，支持 Copilot 这类按凭据确定 baseUrl 的情况。采用异步形式，以便延迟加载封装器载入实现。 */
  toAuth(credential: OAuthCredential): Promise<ModelAuth>;
}

export interface AuthResult {
  auth: ModelAuth;
  /** 供状态 UI 使用的可读标签，例如 "ANTHROPIC_API_KEY"、"OAuth"、"~/.aws/credentials"。 */
  source?: string;
}

export interface AuthContext {
  env(name: string): Promise<string | undefined>;
  fileExists(path: string): Promise<boolean>; // 支持开头的 ~
}
```

将 `refresh` 与 `toAuth` 拆分后，`Models` 可以直接负责加锁刷新模式，无需复杂的闭包技巧：`refresh` 生成凭据，`toAuth` 则根据最终存入的凭据派生请求身份验证信息。

OAuth 实现直接使用不依赖具体服务提供商的 `AuthInteraction` 协议。回调服务器流程会同时等待服务器回调和 `manual_code` 提示；如果服务器回调先完成，就中止提示。因此，UI 不需要服务提供商专用回调，也不需要静态的回调服务器标志。

### 凭据

每个服务提供商保存一份带类型标记的凭据，其结构与目前的 auth.json 完全一致（每个服务提供商 ID 对应 `type: "api_key" | "oauth"`）：

```ts
export interface ApiKeyCredential {
  type: "api_key";
  key?: string;
  env?: ProviderEnv; // 例如 Cloudflare 账户/网关 ID、Azure/Vertex/Bedrock 作用域配置
}

export interface OAuthCredential extends OAuthCredentials {
  type: "oauth"; // 继承 OAuthCredentials 中的 access、refresh、expires
}

export type Credential = ApiKeyCredential | OAuthCredential;
```

`ApiKeyCredential.env` 可以与密钥一起存储服务提供商作用域内的环境/配置值，也可以只存储这些值。`ApiKeyAuth.resolve()` 逐字段合并，例如 `credential.key ?? env("CLOUDFLARE_API_KEY")`、`credential.env?.CLOUDFLARE_ACCOUNT_ID ?? env("CLOUDFLARE_ACCOUNT_ID")` 等。凭据判别值有意与当前 `auth.json` 中的 `api_key` 保持一致，使文件存储无需进行可能丢失信息的类型转换。

### 凭据存储

存储由应用注入；`pi-ai` 默认提供内存实现。以服务提供商 ID 为键，每个服务提供商保存一份凭据：

```ts
export interface CredentialStore {
  /** 读取已存储的凭据，凭据可能已经过期。供显示/状态使用；请求身份验证信息由 Models.getAuth() 提供。 */
  read(providerId: string): Promise<Credential | undefined>;

  /**
   * 串行化写入——唯一的写入路径。fn 可以看到当前凭据，
   * 因为正确的写入（刷新、刷新期间登录）依赖该值；
   * 返回新凭据，或返回 undefined 以保持条目不变。
   * 每个服务提供商 ID 互斥；如果底层存储支持（例如文件锁），也要跨进程互斥。
   * 最终解析为写入后的凭据。
   */
  modify(
    providerId: string,
    fn: (current: Credential | undefined) => Promise<Credential | undefined>,
  ): Promise<Credential | undefined>;

  /** 移除凭据（登出）。与 modify 串行执行。 */
  delete(providerId: string): Promise<void>;
}
```

这里有意不提供 `set`：非串行化写入路径容易造成“读—改—写”竞争，例如刷新期间登录覆盖刚刷新的凭据，或 token 被重复刷新。调用方式：

```ts
await store.modify(pid, async () => credential);      // 登录：存储该凭据
await store.read(pid);                                // 状态 UI（“已通过 OAuth 登录”）
await store.delete(pid);                              // 登出
// 刷新的“读—改—写”操作在 Models.getAuth 内部进行
```

错误语义：条目不存在时，`read` 解析为 `undefined`；只有存储失败时方法才会拒绝，`Models` 会将此类拒绝封装为代码为 `"auth"` 的 `ModelsError`。尽力提供内存视图、并在内部记录持久化错误的存储（即当前 `AuthStorage` 的行为）也是有效实现。

### 解析策略（固定）

`Models.getAuth(model)` 使用决策树，而不是循环。服务提供商一旦存在已存储凭据，就由该凭据完全决定身份验证；只有没有存储任何凭据时，才查询外部环境/环境变量（与 AuthStorage 保持一致：刷新失败或凭据类型不匹配时，不会静默回退到环境变量）：

```ts
const stored = await store.read(provider.id);
if (stored) {
  if (stored.type === "oauth" && provider.auth.oauth) {
    const oauth = provider.auth.oauth;
    let credential = stored;
    if (Date.now() >= credential.expires) {                 // 乐观检查，无需加锁
      const post = await store.modify(provider.id, async (current) => {
        if (current?.type !== "oauth") return undefined;    // 此期间已登出
        return Date.now() >= current.expires                // 锁内的权威检查
          ? oauth.refresh(current)                          // 抛出错误 → ModelsError("oauth")
          : undefined;                                      // 另一个进程/请求已经刷新
      });
      if (post?.type !== "oauth") return undefined;
      credential = post;
    }
    return { auth: await oauth.toAuth(credential), source: "OAuth" };
  }
  if (stored.type === "api_key" && provider.auth.apiKey) {
    return provider.auth.apiKey.resolve({ model, ctx, credential: stored });
  }
  return undefined; // 已存储凭据没有匹配处理器时，阻止使用外部凭据
}
return provider.auth.apiKey?.resolve({ model, ctx, credential: undefined }); // 外部凭据
```

特性：

- 采用双重检查锁定，与当前的 `refreshOAuthTokenWithLock` 相同：有效 token 只需执行一次 `read`，无需加锁；过期 token 会加锁并在锁内复查，全局只刷新一次，并在释放锁前持久化。
- 显式请求身份验证信息（流式传输选项中的 `apiKey`/`headers`）会在 `stream()` 中逐字段覆盖合并，优先级最高。
- 刷新失败时以 `ModelsError("oauth")` 拒绝；已存储凭据保持不变，以便重试。请求路径将其作为带真实原因（“运行 /login”）的流错误公开；状态/可用性 UI 会捕获拒绝并显示“需要重新登录”——这是 `getAuth` 中记录的契约。

### 替换 AuthStorage

coding-agent 的最终状态是删除 AuthStorage，将其能力映射为一个 `CredentialStore` 实现及若干组合层。

当前 `getApiKey` 的优先级及其在新设计中的归属：

| 当前 AuthStorage | 新设计 |
|---|---|
| 运行时覆盖值（CLI `--api-key`） | `withRuntimeOverrides(store, overrides)` 装饰器：`read` 将覆盖值作为 `ApiKeyCredential` 返回，绝不持久化 |
| 已存储的 `api_key`（通过 `resolveConfigValue` 支持 `$ENV`/`!command`） | 已存储的 `ApiKeyCredential`；coding-agent 的适配器/装饰器在 `read` 时解析配置值（命令执行仍由应用策略决定） |
| 已存储的 `oauth` + 加锁刷新，失败时返回 undefined | 使用上述 `getAuth` 决策树；失败时带原因拒绝，而不是静默变为未配置 |
| 环境变量（仅在没有已存储值时） | `apiKey.resolve` 的外部凭据分支 |
| `fallbackResolver`（models.json 自定义服务提供商） | 删除——自定义服务提供商自行携带 `auth.apiKey` |

```txt
FileCredentialStore        移植 AuthStorage 的锁后端：read = 内存快照，
                           modify = withLockAsync（重新读取、fn、合并写入），delete，
                           内部错误记录（相当于 drainErrors）
└─ withConfigValues        在读取时处理 $ENV / !command
   └─ withRuntimeOverrides --api-key
      └─ createModels({ credentials: store })

登录/登出 UI               provider.auth.{oauth,apiKey}.login(interaction) + store.modify/delete
状态 UI                    store.read(pid) + getAuth try/catch（拒绝时显示“需要 /login”）
getOAuthProviders          检查已注册服务提供商中是否存在 provider.auth.oauth
```

### 登录回调

同一个接口同时支持 API 密钥和 OAuth 登录：

```ts
export interface AuthInteraction {
  /** 中止整个登录流程。取消单个提示时使用 AuthPrompt.signal。 */
  signal?: AbortSignal;

  prompt(prompt: AuthPrompt): Promise<string>;
  notify(event: AuthEvent): void;
}

/** 当带外事件完成当前步骤时，流程可以通过 `signal` 取消尚未完成的提示。 */
export type AuthPrompt = { signal?: AbortSignal } & (
  | { type: "text"; message: string; placeholder?: string }
  | { type: "secret"; message: string; placeholder?: string }
  | { type: "select"; message: string; options: readonly { id: string; label: string; description?: string }[] }
  | { type: "manual_code"; message: string; placeholder?: string }
);

export type AuthEvent =
  | { type: "auth_url"; url: string; instructions?: string }
  | { type: "device_code"; userCode: string; verificationUri: string; intervalSeconds?: number; expiresInSeconds?: number }
  | { type: "progress"; message: string };
```

`prompt()` 返回输入或选择的字符串（`select` 返回选项 ID）。流程通过设置 `AuthPrompt.signal`，同时等待 `manual_code` 提示和回调服务器；如果回调先完成，则中止提示。

### 附加 OAuth

支持 OAuth 的服务提供商始终附加该能力，不提供工厂开关。流程采用延迟加载，因此声明支持 OAuth 在实际执行 `login()`/`refresh()` 前没有成本；从不登录的宿主也永远不会加载它。

```ts
export function anthropicProvider(): Provider {
  return createProvider({
    id: "anthropic",
    name: "Anthropic",
    baseUrl: "https://api.anthropic.com/v1",
    auth: {
      apiKey: envApiKeyAuth("Anthropic API key", ["ANTHROPIC_API_KEY"]),
      oauth: lazyOAuth({
        name: "Anthropic (Claude Pro/Max)",
        load: () => import("../auth/oauth/anthropic.ts").then((m) => m.anthropicOAuth),
      }),
    },
    models: ANTHROPIC_MODELS,
    api: anthropicMessagesApi(),
  });
}
```

`lazyOAuth()` 封装动态导入的 `OAuthAuth`，让服务提供商定义可以声明支持 OAuth 而无需导入其实现（`toAuth` 之所以是异步方法，正是出于这一原因）：

```ts
export function lazyOAuth(input: {
  name: string;
  load: () => Promise<OAuthAuth>;
}): OAuthAuth;
```

OAuth 不能迫使浏览器包引入 Node 专用代码（`node:http`、`node:crypto`）：`lazyOAuth()` 内的动态导入采用与 Bedrock 延迟加载封装器相同、对打包器不透明的变量模块说明符技巧。浏览器宿主不会触发加载，因为不存在已存储的 Node OAuth 凭据，也不会运行登录流程。如果以后实现 Web OAuth（sitegeist 已证明 Web Crypto PKCE、身份验证标签页、通过 fetch 交换 token、设备代码轮询等方案可行），它只需提供另一种 `OAuthAuth` 实现，无需预留选项值。

`src/auth/oauth/` 中的内置流程直接实现 `OAuthAuth` 和 `AuthInteraction`，同时保持面向 Node 且延迟加载。Copilot 通过 `toAuth().baseUrl` 派生对应凭据的请求端点。

## 服务提供商封装器与 models.json

`models.json` 是服务提供商封装层，不会原地修改服务提供商：

```ts
function withProviderOverrides(base: Provider, overrides: ProviderOverrides): Provider {
  return {
    ...base,
    name: overrides.name ?? base.name,
    baseUrl: overrides.baseUrl ?? base.baseUrl,
    headers: mergeHeaders(base.headers, overrides.headers),

    getModels: () => applyModelOverrides(base.getModels(), overrides.models),
    refreshModels: base.refreshModels?.bind(base),

    stream: base.stream,
    streamSimple: base.streamSimple,
  };
}
```

这种设计可以与动态服务提供商组合，因为 `getModels()` 委托给基础来源，`refreshModels()` 则直接透传。

models.json 中的请求身份验证配置（`$ENV`、`!command`、内联密钥）仍是由应用管理的旁路状态，应用可以将其作为显式请求身份验证信息公开，也可以为已封装服务提供商的 `auth.apiKey` 设置自定义 `ApiKeyAuth`。

## 自定义服务提供商：createProvider()

一个辅助函数负责用各组成部分构建服务提供商，同时支持单 API 和混合 API 服务提供商：

```ts
export function createProvider(input: {
  id: string;
  name?: string;                 // 默认值：id
  baseUrl?: string;
  headers?: Record<string, string>;
  auth: ProviderAuth;            // 必填，apiKey/oauth 至少提供一个（不存在“无需身份验证”的服务提供商）
  /** 初始模型列表（纯动态服务提供商为空）。 */
  models: readonly Model<Api>[];
  /** 动态服务提供商：获取当前列表；createProvider 会保存结果，并合并正在执行的重复调用。 */
  refreshModels?: () => Promise<readonly Model<Api>[]>;
  /** 单一实现，或供混合 API 服务提供商使用、以 model.api 为键的映射。 */
  api: ProviderStreams | Record<string, ProviderStreams>;
}): Provider;
```

- 单一 `api`：所有模型都通过它进行流式传输。
- 映射形式的 `api`：`stream()`/`streamSimple()` 根据 `model.api` 分发；API 未知时生成流错误。

必须支持混合 API 的自定义服务提供商（类似 opencode Go/Zen 的服务提供商会在同一个服务提供商 ID 下公开由不同 API 支持的模型）。

内置服务提供商工厂在内部使用 `createProvider()`；models.json 中的自定义服务提供商可直接映射到它：

```json
{
  "providers": {
    "my-openai-proxy": {
      "api": "openai-completions",
      "baseUrl": "https://proxy.example/v1",
      "models": [ ... ]
    }
  }
}
```

## 兼容入口点

在 coding-agent 迁移完成并删除旧 API 之前，`@earendil-works/pi-ai/compat` 会保留旧的全局 API 接口。新代码绝不导入该入口。

需要保留的旧语义：对于自定义服务提供商、修改过的模型，以及覆盖内置 API 实现的测试/扩展，全局 `stream()` 仍可通过旧版 API 注册表，依据 `model.api` 进行分发。

- `stream/complete/streamSimple/completeSimple(model, ctx, opts)`：真正匹配内置服务提供商/模型/API 的请求通过单例 `builtinModels()` 集合路由，从而与新运行时共享服务提供商的身份验证、环境和 `baseUrl` 行为。未知服务提供商、修改过的模型或被覆盖的 API 注册项，则回退到 API 注册表分发并注入 `getEnvApiKey`。
- 内置 API 注册的副作用从根入口移入兼容入口。已有注册项的 API ID 会被跳过，因为兼容入口加载前，测试或扩展可能已经注册了覆盖实现。`registerApiProvider()`/`unregisterApiProviders()` 继续操作兼容入口内部的注册表；`resetApiProviders()` 则清空并重新注册内置 API。
- 同步的 `getModel/getModels/getProviders` 成为 `providers/all` 中 `getBuiltinModel/getBuiltinModels/getBuiltinProviders` 的弃用别名（这些方法始终只是读取生成的模型目录——已经验证，从来没有代码修改旧的 `modelRegistry`）。
- 重新导出每个 API 的延迟加载流封装器（包括 `setBedrockProviderModule`）、`env-api-keys.ts` 以及图像生成注册表/模型目录；这些内容都不再保留于根入口。
- `export * from "./index.ts"`：兼容入口是核心入口的严格超集，因此消费者可以整体替换文件中的导入路径，无需逐个修改符号。

coding-agent（以及过渡期的 agent 包）会将这些符号的导入路径从 `@earendil-works/pi-ai` 改为 `@earendil-works/pi-ai/compat`（只修改导入路径），在 ModelManager 迁移前不作其他改动。

扩展宽限期：coding-agent 扩展加载器（jiti 别名 + Bun `virtualModules`）会将根模块说明符 `@earendil-works/pi-ai` 解析到兼容入口。现有用户扩展即使使用旧的全局 API（`complete`、`getModel`、`registerApiProvider` 等），无需修改仍能在运行时工作；只有 ModelManager 迁移时移除兼容入口后才会失效，届时会在更新日志中提供迁移指南。类型检查会起到提示作用：编辑器将根入口解析为精简的核心类型，因此需要通过类型检查的扩展源码必须从 `/compat` 导入旧全局 API，仓库中的示例扩展也会演示这一做法。

## 内置静态辅助函数

只读取生成模型目录的类型化同步辅助函数与目录放在一起，并从 `providers/all` 导出：

```ts
getBuiltinModel(provider, id)   // 同步；使用生成模型目录提供的类型化重载
getBuiltinModels(provider)      // 同步
getBuiltinProviders()           // 同步
```

通过 `Models` 实例进行的运行时查询，会同步读取服务提供商最近已知的列表：`models.getModel(...)`。对新鲜度有严格要求的调用者应先运行 `await models.refresh(provider)`。

通过更新 `packages/ai/scripts/generate-models.ts`，按服务提供商拆分生成的模型目录（`providers/<id>.models.ts`）。如果本轮修改生成器的工作量过大，可以推迟拆分；`providers/all` 和服务提供商工厂可以暂时导入单体的 `models.generated.ts`，并依赖 `sideEffects: false` 进行裁剪。

## 摇树优化与延迟导入

规则：

1. `@earendil-works/pi-ai` 主入口只包含核心内容。
2. 服务提供商模块只导入自身模型目录、身份验证辅助函数和 API 延迟加载封装器。
3. API 延迟加载封装器动态导入真实 API 实现。
4. 真实 API 实现导入 SDK 依赖。
5. OAuth 实现始终通过 `lazyOAuth()` 附加，并隐藏在对打包器不透明的动态导入之后进行延迟加载；服务提供商元数据绝不立即导入 Node 专用 OAuth 代码。
6. `providers/all` 导入每个内置服务提供商工厂和所有模型目录，它是显式的重型入口点。
7. 服务提供商模块无副作用；导入服务提供商不会在全局注册任何内容。
8. `package.json` 的 `sideEffects` 只列出有副作用的兼容/图像注册文件；根模块和服务提供商模块仍可进行摇树优化。
9. 使用代码拆分时，服务提供商 SDK 保留在延迟加载的代码块中。不使用代码拆分时，打包器会将静态可达的延迟 API 实现合并进单个包；此时 `providers/all` 会带入全部静态可见的 SDK。Bedrock 是例外：其 AWS SDK 实现隐藏在对打包器不透明的 Node 专用导入之后，独立的单文件包需要使用 `setBedrockProviderModule()`。

导出映射草图：

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./compat": "./dist/compat.js",
    "./providers/all": "./dist/providers/all.js",
    "./providers/openai": "./dist/providers/openai.js",
    "./providers/anthropic": "./dist/providers/anthropic.js",
    "./providers/*": "./dist/providers/*.js",
    "./api/*": "./dist/api/*.js"
  }
}
```

浏览器冒烟检查（`scripts/check-browser-smoke.mjs`）必须持续通过：打包核心入口点以及任何非 Node 服务提供商入口点时，都不能带入 `node:http`/`node:crypto`。

## AgentHarness 集成

`AgentHarness` 接收一个 `Models` 实例。

- `AgentHarnessOptions.models` 为必填项。
- 运行框架不会将 `Models` 快照保存到轮次状态。
- 请求路径调用 `this.models.streamSimple(model, context, options)`；上下文压缩/分支摘要路径也一样。
- 请求路径绝不会调用异步 `models.getModel()` 对模型进行规范化；如果模型元数据需要刷新，应用应在开始轮次前更新选中的模型。
- 运行框架测试会创建 `createModels()`，并安装模拟服务提供商（`providers/faux` 中的 `fauxProvider()` 工厂）。

## coding-agent 下一阶段（不属于本轮）

coding-agent 分层构建服务提供商，并按会话进行绑定：

```txt
内置服务提供商（builtinModels）
→ models.json 服务提供商封装器/自定义服务提供商（createProvider）
→ 扩展提供的服务提供商封装器/新增服务提供商
```

```ts
sessionModels.clearProviders();
for (const provider of layeredProviders) sessionModels.setProvider(provider);
```

coding-agent 负责：取代 AuthStorage 的 `FileCredentialStore` 及装饰器（参见“替换 AuthStorage”）、models.json 身份验证旁路状态（`$ENV`、`!command`）、命令执行策略、服务提供商状态标签（来自 `AuthResult.source`）、登录/登出 UI（通过 `prompt()`/`notify()` 驱动 `auth.{apiKey,oauth}.login()`）、扩展生命周期，以及服务提供商管理斜杠命令。

当前过渡状态：

- `AgentHarness` 已经接受 `Models` 实例，并将其用于轮次流式传输、上下文压缩和分支摘要。
- coding-agent 尚未使用 `AgentHarness`；`AgentSession` 仍通过 `streamFn` 驱动底层 `Agent`。
- coding-agent 仍在使用旧版 `AuthStorage` + `ModelRegistry`，并通过 `@earendil-works/pi-ai/compat` 导入旧的 pi-ai 全局 API。
- 扩展加载器仍将 pi-ai 根入口别名指向 `/compat`，作为旧扩展的运行时宽限期。

## 实现待办事项

事项完成后应及时勾选，并持续更新本列表；恢复会话时以此处记录的工作状态为准。

### 阶段 1——核心类型/运行时

- [x] 将 `types.ts` 的 `Provider` 别名重命名为 `ProviderId`，并修正调用位置。
- [x] 向 `types.ts` 增加 `ApiOptionsMap` 和 `ApiStreamOptions<TApi>`（纯类型导入）。
- [x] 新建 `models.ts`：包含 `Provider<TApi>` 接口、`hasApi()` 守卫、`ModelsError` 及错误代码。身份验证类型放在 `src/auth/types.ts` 中（`ProviderAuth` = `{ apiKey?, oauth? }`、凭据、`CredentialStore`〔`read`/`modify`/`delete`，每个服务提供商一份凭据〕、`AuthResult`、`AuthContext`、`ModelAuth`、登录回调）；内存存储放在 `src/auth/credential-store.ts`；默认上下文放在 `src/auth/context.ts`（使用浏览器安全的 node:fs 技巧）；`lazyStream()` 放在 `src/api/lazy.ts`。
- [x] 实现 `Models`/`MutableModels`/`createModels({ credentials?, authContext? })`：包含服务提供商映射、同步 `getModel(s)`（按服务提供商隔离失败）、显式异步 `refresh(provider?)`、`getAuth`（决策树、双重检查加锁刷新），以及逐字段合并身份验证信息的 `stream/complete/streamSimple/completeSimple`。测试：`packages/ai/test/models-runtime.test.ts`。
- [x] 保留元数据辅助函数：`calculateCost`、`getSupportedThinkingLevels`、`clampThinkingLevel`、`modelsAreEqual`。

### 阶段 2——`src/api/`

- [x] 将流式传输实现从 `src/providers/` 移至 `src/api/`，并按 API ID 重命名（例如 `anthropic.ts` → `api/anthropic-messages.ts`）。
- [x] 统一每个实现模块，使其只导出 `stream` 和 `streamSimple`。
- [x] 将共享辅助模块（`openai-responses-shared`、`google-shared`、`transform-messages`、`openai-prompt-cache`、`github-copilot-headers`、`cloudflare`、`simple-options`）移至 `src/api/`。
- [x] 将 `lazyStream()`/`lazyApi()` 提取到 `src/api/lazy.ts`。
- [x] 为每个 API 增加 `*.lazy.ts` 封装器；Bedrock 保留 Node 专用导入技巧和 `setBedrockProviderModule()`。
- [x] 删除 `providers/register-builtins.ts`。阶段 5 的兼容层完成前，内置 API 注册表的注册逻辑暂时放在 `stream.ts` 中；API 延迟加载封装器从根入口导出。

### 阶段 3——服务提供商工厂与模型目录

- [x] 在 `src/auth/helpers.ts` 中实现身份验证辅助函数：`envApiKeyAuth()`（包含秘密信息提示式 `login`）、`lazyOAuth()`。OAuth 流程通过 `auth/oauth/load.ts` 加载（对打包器不透明的动态导入）；其引用的 `OAuthAuth` 导出在阶段 4 完成。
- [x] 在 `models.ts` 中实现 `createProvider()`（支持单一和混合 `api` 映射，根据 `model.api` 分发，API 未知时生成流错误）。
- [x] 在 `src/providers/` 下为所有内置模型目录服务提供商建立独立工厂；通过 `lazyOAuth()` 附加 OAuth（Anthropic、OpenAI Codex、GitHub Copilot）；为 amazon-bedrock（AWS 环境变量/配置文件）和 google-vertex（密钥或 ADC + 项目 + 位置）提供外部凭据式 `ApiKeyAuth`。
- [x] 在 `providers/all.ts` 中重新导出 `builtinProviders()`、`builtinModels()`、`getBuiltinModel/getBuiltinModels/getBuiltinProviders`。
- [x] 为测试建立模拟服务提供商工厂（`providers/faux.ts` 中的 `fauxProvider()`）；在兼容层删除前保留旧版 `registerFauxProvider()`。
- [x] 通过 `scripts/generate-models.ts` 按服务提供商拆分生成的模型目录（`providers/<id>.models.ts`）；`models.generated.ts` 改为生成的聚合文件。

### 阶段 4——适配 OAuth

- [x] 内置实现位于 `auth/oauth/` 下，通过 `AuthInteraction.prompt()`/`notify()` 直接实现 `OAuthAuth`。它们是服务提供商私有实现，由服务提供商工厂延迟加载。
- [x] 回调服务器流程同时等待 `manual_code` 提示；流程完成后通过 `AuthPrompt.signal` 中止提示。公开的 `oauth` 子路径只保留 coding-agent 扩展兼容类型。

### 阶段 5——打包

- [x] `index.ts` 只包含核心内容且无副作用（不包含模型目录、服务提供商工厂、API 注册表、env-api-keys、图像、OAuth、兼容层）。类型化模型目录读取方法（`getBuiltin*`）在 `providers/all.ts` 中实现；`models.ts` 不再导入 `models.generated.ts`。
- [x] `compat.ts`：包含 index 的全部内容以及旧版 API 分发全局方法、已弃用的 `getModel/getModels/getProviders` 别名、API 延迟加载封装器 + `setBedrockProviderModule`、`getEnvApiKey` 和图像功能。注册副作用位于此处（已有项则跳过）。
- [x] 建立子路径导出映射（`./compat`、`./providers/*`、`./api/*`）；将有副作用的模块（兼容层、图像注册）列入 `sideEffects` 数组，而不是将其设为 `false`。
- [x] 浏览器冒烟检查（入口现在从 `/compat` 导入旧全局方法）和 shrinkwrap 检查均通过。内部旧全局方法的导入已改为 `/compat`（agent/coding-agent/examples 中共 42 个文件；Vitest 配置将 `/compat` 别名指向源码；spawn-CLI 测试解析工作区构建产物，因此重新构建了 `packages/ai` 和 `packages/agent`）。

### 阶段 6——AgentHarness

- [x] `AgentHarnessOptions.models` 为必填项（运行框架上提供 `readonly models`）；运行框架流式传输路径使用 `models.streamSimple()`。以结构类型重新定义 `StreamFn`，不再依赖兼容类型；`Models.streamSimple` 满足该类型。
- [x] 上下文压缩/分支摘要接收运行框架的 `Models` 实例。彻底移除 `getApiKeyAndHeaders`——`Models` 是唯一的身份验证路径；每个请求的密钥解析改由集合中的服务提供商身份验证处理。`compact()`/`generateSummary()`/`generateBranchSummary()` 不再具有显式 `apiKey`/`headers` 参数。
- [x] 运行框架测试使用 `createModels()` + `fauxProvider()`，并为每个模拟服务提供商设置唯一 ID；不再使用全局 API 注册表状态，也无需注销记录。

### 阶段 7——coding-agent 桥接（最小范围）

- [x] 将旧全局方法的导入改为 `@earendil-works/pi-ai/compat`（随阶段 5 完成；兼容层是超集，因此只需修改路径）。扩展加载器将 pi-ai 根入口解析到兼容层，作为运行时宽限期。
- [x] 原先在此规划的其他所有事项，都要等 coding-agent 真正通过 `Models` 实例进行流式传输后才能开展——coding-agent 的 `AgentSession` 通过 `streamFn` 驱动底层 `Agent`，而不是使用运行框架——因此移至阶段 9。

### 阶段 8——收尾

- [x] 更新/增加测试并运行受影响的测试套件（各阶段均随实现提交测试；`./test.sh` 始终通过）。
- [x] `packages/ai/CHANGELOG.md`：在 `### Breaking Changes` 中提供迁移指南（兼容入口、`Provider` → `ProviderId`、API 模块迁移），并在 `### Added` 中记录新的 Models/服务提供商/身份验证 API。
- [x] `packages/coding-agent/CHANGELOG.md`：为扩展作者增加 `### Changed` 条目——运行时不受影响（加载器将 pi-ai 根入口解析到兼容层）；类型检查会提示改用 `/compat` 或新 API；以后移除时提供迁移指南。
- [x] `packages/agent/CHANGELOG.md`：在 `### Breaking Changes` 中记录 `AgentHarnessOptions.models` 改为必填、上下文压缩签名变更，以及结构类型化的 `StreamFn`。
- [x] `npm run check` 无错误。

### 阶段 9——coding-agent 迁移到 Models + CredentialStore（本轮范围）

coding-agent 使用 `FileCredentialStore` + `MutableModels` 集合替换 AuthStorage 和 ModelRegistry 的内部实现。AgentSession 本身保留（采用 AgentHarness 属于 Pi 2.0 的工作），只替换其模型/身份验证基础设施。各层严格保持单向依赖：

```txt
FileCredentialStore（auth.json、加锁、解析 $ENV/!command）+ 显式 --api-key 覆盖层
        ↑
MutableModels：内置工厂（根据 models.json 配置封装）+ 自定义服务提供商（models.json ∪ 扩展）
        ↑
ModelRegistry：兼容门面——将最近已知值的同步读取委托给集合；为扩展 + UI 提供 registerProvider/login/logout/status
        ↑
AgentSession / SDK / 交互模式（通过 models 进行流式传输；只等待身份验证/刷新路径）
```

设计决策：

- 删除 `AuthStorage` 类型——否则它会依赖服务提供商身份验证，而服务提供商身份验证又依赖其存储，形成循环依赖。将其接口拆分：`get`/`set`/`remove` → `CredentialStore`；`getApiKey` → `Models.getAuth`；`login`/`logout`/`getAuthStatus` → 基于 `provider.auth.oauth` + 存储实现的 ModelRegistry 门面方法。
- `FileCredentialStore` 自包含路径、锁定、解析/写入、chmod 和错误缓冲逻辑，并负责 `auth.json` 语义，包括解析已存储 API 密钥凭据中的 `$ENV`/`!command`。持久化值保持原始形式；解析时返回副本供身份验证使用。
- 运行时 `--api-key` 覆盖值通过显式的存储覆盖层实现（读取覆盖值时，将其视为临时存储的 API 密钥凭据，从而遮蔽已存储 OAuth——与当前优先级一致）。确保每个已注册服务提供商都有 `apiKey` 身份验证槽位，使覆盖值也能应用于只支持 OAuth 的服务提供商。
- 为兼容 SDK 和扩展，`ModelRegistry.getAll`/`find`/`getAvailable` 保持同步；它们委托给集合中最近已知的同步模型列表和快速的“看似已配置”状态检查。动态服务提供商通过显式异步 `refresh()` 更新，请求身份验证则继续通过 `getApiKeyAndHeaders()`/`Models.getAuth()` 异步执行。扩展还会直接获得集合，作为面向未来的 API。
- models.json 保持全部现有功能，通过装饰服务提供商实现：封装内置工厂，使 `getModels()` 应用服务提供商 `baseUrl`/`compat` 覆盖、`modelOverrides` 和自定义模型合并（异步安全）；服务提供商的 `apiKey`/`headers`/`authHeader` 配置转为该服务提供商的 `ApiKeyAuth`（优先使用配置，回退到工厂身份验证）；解析错误保留 `getError()` 语义。
- 与扩展 `ProviderConfig` 保持一致：提供按服务提供商区分的 `streamSimple`，将旧版扩展 OAuth 回调适配为 `OAuthAuth`，并支持按服务提供商完整替换模型。旧版 `registerApiProvider` 写入继续只存在于兼容层，供调用全局 `complete()` 的消费者使用；兼容层删除时一并移除。
- Copilot：在封装后的 `getModels()` 中应用已存储凭据的 `baseUrl`，确保扩展可见的模型仍然正确；每个请求还会应用 `toAuth().baseUrl`。
- Cloudflare：替换服务提供商身份验证信息（密钥 + 从凭据 `env` 或外部 `AuthContext.env()` 读取的 `CLOUDFLARE_ACCOUNT_ID`/`CLOUDFLARE_GATEWAY_ID` → `ModelAuth.baseUrl`）。内置兼容调用通过 `Models` 路由，因此使用相同的服务提供商身份验证路径。

新会话中的实施顺序：

1. [x] 首先重构 pi-ai：`Provider.getModels()` 同步 + 可选 `refreshModels()`；`Models.getModels`/`getModel` 同步，`Models.refresh(provider?)` 异步；`createProvider` 接受 `models` 数组 + 可选的 `refreshModels` 获取函数（合并正在执行的重复请求）。这推翻了阶段 1 的异步列表决策——理由参见“服务提供商模型列表”（同步或异步联合类型会滋生隐蔽的同步假设；只允许异步又会破坏扩展 `find`/`getAll` 等同步消费者接口）。
2. [x] 在 pi-ai 工厂中实现 Cloudflare 服务提供商身份验证：Workers AI 和 AI Gateway 校验必需的账户/网关环境变量或配置，并通过服务提供商身份验证返回解析后的 `baseUrl`、服务提供商作用域环境配置，以及请求头抑制/覆盖元数据。
3. [ ] 在 coding-agent 中增加 `FileCredentialStore`。
   - 将 pi-ai 的 `CredentialStore` 接口实现为自包含的 `auth.json` 存储；不要依赖旧的 `AuthStorageBackend` 抽象，但可以移植其锁定/重试语义。
   - 保留现有文件格式。`ApiKeyCredential` 使用 `{ type: "api_key", key?, env? }`，与当前 `auth.json` 一致；不要将 `env` 转换为元数据，也不要重写判别字段。
   - 使用注入的执行/配置环境，默认解析已存储 API 密钥 `key` 和 `env` 值中的 `$ENV`/`!command`。`$ENV` 应通过该环境查询；`!command` 应通过共享 shell 执行路径运行，而不是直接使用 `execSync`。
   - 持久化原始配置值；返回给身份验证逻辑的已解析凭据必须是副本，除非调用者显式存入新值，否则不能改写 `$ENV`/`!command` 字符串。
   - `read(provider)` 返回当前凭据快照，并记录解析/存储错误，以保持状态 UI 行为一致。
   - `modify(provider, fn)` 必须加锁、重新读取、运行 `fn`、合并写入服务提供商条目、执行 `chmod 0600`，并返回写入后的凭据。
   - `delete(provider)` 必须加锁，并且只移除该服务提供商的条目。
   - 增加文件存储和内存存储测试，覆盖锁定/“读—改—写”行为、解析配置值的 `api_key` 读取、OAuth 读取、保留服务提供商 `env`、删除、解析错误和类似并发刷新的修改。
4. [ ] 增加符合 coding-agent 策略的运行时覆盖层。
   - `withRuntimeOverrides(store, overrides)` 实现 CLI `--api-key`：对于每个被覆盖的服务提供商，读取时返回临时的 `{ type: "api_key", key }`，遮蔽已存储的 OAuth/API 凭据，但不进行持久化。
   - 运行时覆盖值也必须应用于支持 OAuth 的服务提供商；coding-agent 中注册的每个服务提供商都必须保留或增加 `apiKey` 身份验证槽位，使覆盖层真正有效。
   - 测试覆盖以下优先级：运行时覆盖值 > 已存储凭据 > models.json 配置身份验证 > 服务提供商外部环境；已存储凭据会阻止回退到外部环境。
5. [ ] 为 `models.json` 构建服务提供商装饰辅助函数。
   - 从内置服务提供商工厂开始，而不是从生成的模型数组开始。
   - 封装服务提供商的 `getModels()`，使每次同步读取都应用服务提供商级 `baseUrl`/`headers`/`compat`、每个模型的 `modelOverrides`，以及自定义模型合并。
   - 保留 `refreshModels()` 透传，使动态服务提供商能够与装饰器组合。
   - 将服务提供商的 `apiKey`/`headers`/`authHeader` models.json 配置转换为封装后的 `ApiKeyAuth`：优先解析配置值，随后回退到基础服务提供商身份验证。
   - 带有 `models` 的自定义服务提供商使用 `createProvider()`，并传入适当的 API 延迟加载封装器或扩展提供的流式传输实现。
   - 解析错误必须保留当前的 `ModelRegistry.getError()` 行为：内置服务提供商仍然可用，同时错误保持可见。
6. [ ] 封装 Copilot 的 `getModels()` baseUrl。
   - GitHub Copilot OAuth 的 `toAuth()` 已经为流式传输返回每个凭据对应的请求 `baseUrl`。
   - 存在 OAuth 凭据时，封装 Copilot 服务提供商的 `getModels()`，使扩展/UI 可见的模型元数据也携带已验证账户的基础 URL。
   - 保持 Copilot 的 API 密钥/环境 token 行为不变。
   - 增加测试，覆盖登录前、获得 OAuth 凭据后、刷新或 baseUrl 变更后，以及登出后的模型元数据。
7. [x] 扩展 OAuth 适配器。
   - 只保留 coding-agent `ProviderConfig.oauth` 所需的旧版回调/凭据声明。
   - `login` 将旧版回调/事件映射到 `AuthInteraction.prompt()`/`notify()`。
   - `refreshToken` 映射到 `refresh`；`getApiKey` 映射到 `toAuth`。
   - 保留 pi-ai 仅含类型的 `oauth` 入口和扩展加载器别名。
8. [ ] 基于 `MutableModels` 重建 coding-agent `ModelRegistry`。
   - 它持有一个 `MutableModels` 实例，该实例由装饰后的内置服务提供商 + models.json 自定义服务提供商 + 扩展服务提供商构成。
   - `getAll()`、`find()` 和 `getAvailable()` 继续作为同步兼容方法，读取最近已知模型列表并快速检查“看似已配置”的身份验证状态。不要破坏这些读取方法面向扩展的 `modelRegistry` 接口。
   - `refresh()` 是显式的异步新鲜度边界：重建服务提供商层，并在需要时调用 `models.refresh()`；除兼容层宽限行为外，新路径不应重置全局 API 注册表。
   - `registerProvider()`/`unregisterProvider()` 修改服务提供商层并重建集合。
   - 门面的身份验证操作（`login`、`logout`、服务提供商状态、可用 OAuth 服务提供商）驱动 `provider.auth.{apiKey,oauth}` 和 `CredentialStore`；不再保留 `AuthStorage` 类型。
   - 旧版 `registerApiProvider` 写入只供 `/compat` 调用者使用，并在阶段 10 移除。
9. [ ] 重新连接消费者。
   - `AgentSession` 的流函数通过 `ModelRegistry`/`Models` 解析，不再使用 `getApiKeyAndHeaders()` + 兼容层全局方法。
   - SDK 选项使用 `credentials?: CredentialStore` 或基于 agent 目录的默认值取代 `authStorage`；更新 `sdk.md` 和示例。
   - `model-resolver`、`--list-models`、模型选择器、登录/登出/状态 UI 及服务提供商归属信息都使用最近已知模型的同步读取，只等待显式刷新/身份验证操作。
   - CLI `--api-key` 填充运行时覆盖装饰器，不再修改 `AuthStorage`。
   - 阶段 10 前保留扩展加载器将根入口指向兼容层的别名，但将新集合/门面作为面向未来的 API 公开。
10. [ ] 测试迁移并验证真实服务提供商。
    - 为 `FileCredentialStore`、运行时覆盖层、服务提供商装饰、扩展 OAuth 适配器、基于 Models 的 ModelRegistry 门面和消费者重新连接编写单元测试。
    - 编写回归测试，覆盖 Cloudflare 账户/网关环境配置、Copilot OAuth baseUrl 封装、运行时 `--api-key` 优先级、`$ENV`/`!command` 解析，以及已存储凭据阻止回退到外部环境。
    - 更新现有测试，覆盖最近已知值的同步 `ModelRegistry.getAll/find/getAvailable` 以及显式异步刷新行为。
    - 运行针对性的非端到端测试套件，并通过 tmux 验证真实服务提供商的登录流程（Anthropic OAuth/API 密钥、OpenAI Codex OAuth、GitHub Copilot OAuth、Cloudflare AI Gateway；如果有凭据，也验证 Bedrock）。

### 阶段 10——删除兼容层（Pi 2.0 时期，单独实施）

- [ ] AgentSession → AgentHarness；删除注册表门面，改用运行框架的 `Models`。
- [ ] 将全部内部 `/compat` 导入迁移到新 API：包括每个包的源码、所有测试和示例扩展（之后由示例演示新 API）。到那时，仓库内部不能再导入 `/compat`。
- [ ] 删除 `/compat`、`env-api-keys.ts`、扩展加载器中将根入口指向兼容层的别名，以及兼容层内部的旧版 API 注册表。旧版 OAuth 注册表/服务提供商接口已经删除；只保留纯类型的 `oauth` 入口以兼容扩展。

### 推迟事项/后续工作

- [ ] 将 Web OAuth 实现（sitegeist 风格）作为另一种 `OAuthAuth`。
- [x] 重新设计图像 API：`ImagesModels`/`ImagesProvider`/`createImagesProvider` 与对话侧设计保持对应（同步读取、显式刷新、生成方法绝不拒绝）；通过 `auth/resolve.ts` 中独立的 `resolveProviderAuth()` 与对话侧共享身份验证解析（该模块也负责 `ModelsError`；两个集合都将存储/上下文作为参数传入，不使用解析器对象）。在 `providers/all` 中提供 `openrouterImagesProvider()` 工厂 + `builtinImagesProviders()`/`builtinImagesModels()`；实现移至 `api/openrouter-images.ts`，并增加延迟加载封装器。旧的全局图像 API（注册表 + `getImageModel*` + `generateImages`）保留在兼容层；将 types.ts 中的 `ImagesProvider` ID 别名重命名为 `ImagesProviderId`（对应 `Provider` → `ProviderId`）。

## 错误行为

`undefined` 表示未找到或未配置。真正的失败会使 Promise 拒绝或转为流错误。

```ts
export type ModelsErrorCode =
  | "model_source"      // 刷新服务提供商模型失败
  | "model_validation"  // 模型对象无效
  | "provider"          // 服务提供商未知、分发失败
  | "stream"            // 流准备失败
  | "auth"              // 身份验证解析失败
  | "oauth";            // OAuth 登录/刷新失败
```

- 如果异步准备失败，`Models.stream()` 会生成流错误（错误事件 + 错误结果）；返回流之后不会抛出错误。
- `Models.getModels()` 是尽力而为的同步读取：如果某个服务提供商的 `getModels()` 抛错，则不返回该服务提供商的模型。`Models.refresh(provider)` 会在该服务提供商获取失败时拒绝；`Models.refresh()`（全部服务提供商）并发执行并尽力完成。需要明确得知列表失败的应用应只刷新单个服务提供商。
- 身份验证解析和凭据存储失败会明确拒绝（`ModelsError` 代码 `auth`/`oauth`）；失败后静默回退到另一条身份验证路径可能造成意外计费。只要存在已存储凭据，就始终阻止回退到外部环境/环境变量，包括刷新失败之后。
- 状态/可用性 UI 会捕获 `getAuth` 的拒绝并显示“需要重新登录”，不会将拒绝视为“未配置”。
