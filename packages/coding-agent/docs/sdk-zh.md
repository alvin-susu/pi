> pi 可以帮助你使用 SDK。你可以让它针对自己的使用场景构建集成。

# SDK

SDK 提供以编程方式使用 pi 智能体能力的接口。可以用它将 pi 嵌入其他应用、构建自定义界面，或接入自动化工作流。

**使用场景示例：**

- 构建自定义界面（Web、桌面端、移动端）
- 将智能体能力集成到现有应用中
- 创建使用智能体推理的自动化流水线
- 构建能够创建子智能体的自定义工具
- 以编程方式测试智能体行为

从最简用法到完全控制的可运行示例，请参阅 [examples/sdk/](../examples/sdk/)。

<a id="quick-start"></a>

## 快速开始

```typescript
import { createAgentSession, ModelRuntime, SessionManager } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();
const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  modelRuntime,
});

session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("当前目录中有哪些文件？");
```

<a id="installation"></a>

## 安装

```bash
npm install @earendil-works/pi-coding-agent
```

SDK 已包含在主包中，无需单独安装。

<a id="core-concepts"></a>

## 核心概念

### createAgentSession()

这是创建单个 `AgentSession` 的主要工厂函数。

`createAgentSession()` 使用 `ResourceLoader` 提供扩展、技能、提示词模板、主题和上下文文件。如果没有传入加载器，则使用按标准规则发现资源的 `DefaultResourceLoader`。

```typescript
import { createAgentSession, SessionManager } from "@earendil-works/pi-coding-agent";

// 最简用法：使用 DefaultResourceLoader 和各项默认值
const { session } = await createAgentSession();

// 自定义用法：覆盖特定选项
const { session } = await createAgentSession({
  model: myModel,
  tools: ["read", "bash"],
  sessionManager: SessionManager.inMemory(),
});
```

### AgentSession

会话负责管理智能体生命周期、消息历史、模型状态、上下文压缩和事件流。

```typescript
interface AgentSession {
  // 发送提示词并等待完成
  prompt(text: string, options?: PromptOptions): Promise<void>;

  // 在流式响应期间将消息加入队列
  steer(text: string): Promise<void>;
  followUp(text: string): Promise<void>;

  // 订阅事件（返回取消订阅函数）
  subscribe(listener: (event: AgentSessionEvent) => void): () => void;

  // 会话信息
  sessionFile: string | undefined;
  sessionId: string;

  // 模型控制
  setModel(model: Model): Promise<void>;
  setThinkingLevel(level: ThinkingLevel): void;
  cycleModel(): Promise<ModelCycleResult | undefined>;
  cycleThinkingLevel(): ThinkingLevel | undefined;

  // 访问状态
  agent: Agent;
  model: Model | undefined;
  thinkingLevel: ThinkingLevel;
  messages: AgentMessage[];
  isStreaming: boolean;

  // 在当前会话文件的条目树中原位导航
  navigateTree(targetId: string, options?: { summarize?: boolean; customInstructions?: string; replaceInstructions?: boolean; label?: string }): Promise<{ editorText?: string; cancelled: boolean }>;

  // 上下文压缩
  compact(customInstructions?: string): Promise<CompactionResult>;
  abortCompaction(): void;

  // 中止当前操作
  abort(): Promise<void>;

  // 清理
  dispose(): void;
}
```

新建会话、恢复会话、派生会话和导入会话等替换当前会话的 API 位于 `AgentSessionRuntime`，而不是 `AgentSession`。

### createAgentSessionRuntime() 与 AgentSessionRuntime

需要替换当前活动会话并重建与 `cwd` 绑定的运行时状态时，请使用运行时 API。
内置的交互模式、打印模式和 RPC 模式也使用这一层。

`createAgentSessionRuntime()` 接收运行时工厂函数，以及初始的 `cwd`/会话目标。工厂函数通过闭包捕获进程级固定输入，为实际工作目录重新创建与 `cwd` 绑定的服务，基于这些服务解析会话选项，然后返回完整的运行时结果。

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({
      services,
      sessionManager,
      sessionStartEvent,
    })),
    services,
    diagnostics: services.diagnostics,
  };
};

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});
```

`AgentSessionRuntime` 负责在以下操作中替换活动运行时：

- `newSession()`
- `switchSession()`
- `fork()`
- 通过 `fork(entryId, { position: "at" })` 执行的克隆流程
- `importFromJsonl()`

重要行为：

- 执行上述操作后，`runtime.session` 会发生变化
- 事件订阅绑定到特定的 `AgentSession`，所以替换后需要重新订阅
- 如果使用扩展，请为新会话再次调用 `runtime.session.bindExtensions(...)`
- 创建过程会通过 `runtime.diagnostics` 返回诊断信息
- 如果创建或替换运行时失败，方法会抛出异常，由调用方决定如何处理

```typescript
let session = runtime.session;
let unsubscribe = session.subscribe(() => {});

await runtime.newSession();

unsubscribe();
session = runtime.session;
unsubscribe = session.subscribe(() => {});
```

### 提示词与消息排队

`PromptOptions` 控制提示词展开、流式响应期间的排队行为，以及提示词预检通知：

```typescript
interface PromptOptions {
  expandPromptTemplates?: boolean;
  images?: ImageContent[];
  streamingBehavior?: "steer" | "followUp";
  source?: InputSource;
  preflightResult?: (success: boolean) => void;
}
```

每次调用 `prompt()`，`preflightResult` 都会被调用一次：

- 提示词已被接受、进入队列或立即得到处理时，参数为 `true`
- 提示词在接受前被预检拒绝时，参数为 `false`

该回调会在 `prompt()` 的 Promise 完成前触发。提示词一旦被接受，`prompt()` 仍会等到整个运行过程（包括重试）结束后才完成。接受后发生的故障会通过正常的事件流和消息流报告，而不会调用 `preflightResult(false)`。

`prompt()` 方法负责处理提示词模板、扩展命令和消息发送：

```typescript
// 基础提示词（当前未进行流式响应时）
await session.prompt("这里有哪些文件？");

// 携带图像
await session.prompt("这张图中有什么？", {
  images: [{ type: "image", source: { type: "base64", mediaType: "image/png", data: "..." } }]
});

// 流式响应期间：必须指定消息的排队方式
await session.prompt("停下，改为执行这项任务", { streamingBehavior: "steer" });
await session.prompt("完成后，再检查一下 X", { streamingBehavior: "followUp" });
```

**行为：**

- **扩展命令**（例如 `/mycommand`）：即使正在进行流式响应，也会立即执行。扩展命令通过 `pi.sendMessage()` 自行管理与 LLM 的交互。
- **基于文件的提示词模板**（来自 `.md` 文件）：发送或加入队列前，会先展开为模板内容。
- **流式响应期间未指定 `streamingBehavior`**：抛出错误。请直接使用 `steer()` 或 `followUp()`，或指定该选项。
- **`preflightResult(true)`**：表示提示词已被接受、进入队列或立即得到处理。
- **`preflightResult(false)`**：表示提示词在接受前被预检拒绝。

要在流式响应期间显式排队：

```typescript
// 将引导消息加入队列，在当前助手轮次完成工具调用后发送
await session.steer("新指令");

// 等待智能体完成（仅在智能体停止后发送）
await session.followUp("完成后，再执行这项任务");
```

`steer()` 和 `followUp()` 都会展开基于文件的提示词模板，但遇到扩展命令时会报错，因为扩展命令不能进入队列。

### Agent 与 AgentState

`Agent` 类（来自 `@earendil-works/pi-agent-core`）负责处理与 LLM 的核心交互，可通过 `session.agent` 访问。

```typescript
// 访问当前状态
const state = session.agent.state;

// state.messages: AgentMessage[]——对话历史
// state.model: Model——当前模型
// state.thinkingLevel: ThinkingLevel——当前思考级别
// state.systemPrompt: string——系统提示词
// state.tools: AgentTool[]——可用工具
// state.streamingMessage?: AgentMessage——当前尚未完成的助手消息
// state.errorMessage?: string——最近一次助手错误

// 替换消息（适用于创建分支或恢复状态）
session.agent.state.messages = messages; // 复制顶层数组

// 替换工具
session.agent.state.tools = tools; // 复制顶层数组

// 等待智能体完成处理
await session.agent.waitForIdle();
```

<a id="events"></a>

### 事件

订阅事件以接收流式输出和生命周期通知。

```typescript
session.subscribe((event) => {
  switch (event.type) {
    // 助手的流式文本
    case "message_update":
      if (event.assistantMessageEvent.type === "text_delta") {
        process.stdout.write(event.assistantMessageEvent.delta);
      }
      if (event.assistantMessageEvent.type === "thinking_delta") {
        // 思考输出（已启用思考时）
      }
      break;

    // 工具执行
    case "tool_execution_start":
      console.log(`Tool: ${event.toolName}`);
      break;
    case "tool_execution_update":
      // 流式工具输出
      break;
    case "tool_execution_end":
      console.log(`Result: ${event.isError ? "error" : "success"}`);
      break;

    // 消息生命周期
    case "message_start":
      // 新消息开始
      break;
    case "message_end":
      // 消息完成
      break;

    // 智能体生命周期
    case "agent_start":
      // 智能体开始处理提示词
      break;
    case "agent_end":
      // 智能体完成（event.messages 包含新消息）
      break;

    // 轮次生命周期（一次 LLM 响应及其工具调用）
    case "turn_start":
      break;
    case "turn_end":
      // event.message：助手响应
      // event.toolResults：本轮的工具结果
      break;

    // 会话事件（队列、上下文压缩、重试）
    case "queue_update":
      console.log(event.steering, event.followUp);
      break;
    case "compaction_start":
    case "compaction_end":
    case "auto_retry_start":
    case "auto_retry_end":
    case "summarization_retry_scheduled":
    case "summarization_retry_attempt_start":
    case "summarization_retry_finished":
      break;
  }
});
```

<a id="options-reference"></a>

## 选项参考

### 目录

```typescript
const { session } = await createAgentSession({
  // DefaultResourceLoader 发现资源时使用的工作目录
  cwd: process.cwd(), // 默认值

  // 全局配置目录
  agentDir: "~/.pi/agent", // 默认值（会展开 ~）
});
```

`DefaultResourceLoader` 将 `cwd` 用于：

- 项目扩展（`.pi/extensions/`）
- 项目技能：
  - `.pi/skills/`
  - `cwd` 及其祖先目录中的 `.agents/skills/`（在 Git 仓库中最多查找到仓库根目录，否则查找到文件系统根目录）
- 项目提示词（`.pi/prompts/`）
- 上下文文件（从 `cwd` 向上查找 `AGENTS.md`）
- 会话目录命名

`DefaultResourceLoader` 将 `agentDir` 用于：

- 全局扩展（`extensions/`）
- 全局技能：
  - `agentDir` 下的 `skills/`（例如 `~/.pi/agent/skills/`）
  - `~/.agents/skills/`
- 全局提示词（`prompts/`）
- 全局上下文文件（`AGENTS.md`）
- 设置（`settings.json`）
- 自定义模型（`models.json`）
- 凭据（`auth.json`）
- 会话（`sessions/`）

传入自定义 `ResourceLoader` 后，`cwd` 和 `agentDir` 不再控制资源发现，但仍会影响会话命名和工具路径解析。

### Model

```typescript
import { getModel } from "@earendil-works/pi-ai";
import { ModelRuntime } from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create();

// 查找特定内置模型（不检查 API 密钥是否存在）
const opus = getModel("anthropic", "claude-opus-4-5");
if (!opus) throw new Error("找不到模型");

// 按服务提供商/id 查找任意模型，包括 models.json 中的自定义模型
// （不检查 API 密钥是否存在）
const customModel = modelRuntime.getModel("my-provider", "my-model");

// 只获取已配置有效认证的模型
const available = await modelRuntime.getAvailable();

const { session } = await createAgentSession({
  model: opus,
  thinkingLevel: "medium", // off、minimal、low、medium、high、xhigh、max

  // 循环切换使用的模型（交互模式下按 Ctrl+P）
  scopedModels: [
    { model: opus, thinkingLevel: "high" },
    { model: haiku, thinkingLevel: "off" },
  ],

  modelRuntime,
});
```

如果没有提供模型：

1. 如果正在继续已有会话，尝试从会话中恢复
2. 使用设置中的默认模型
3. 回退到第一个可用模型

要采用与 CLI 相同的模型解析方式，请使用导出的解析辅助函数：

```typescript
import {
  resolveCliModel,
  resolveModelScopeWithDiagnostics,
} from "@earendil-works/pi-coding-agent";

const cliModel = resolveCliModel({
  cliModel: "anthropic/claude-opus-4-5:high",
  modelRuntime,
});
if (cliModel.error) throw new Error(cliModel.error);
if (cliModel.warning) console.warn(cliModel.warning);

const { scopedModels, diagnostics } = await resolveModelScopeWithDiagnostics(
  ["anthropic/*:high", "gpt-5"],
  modelRuntime,
);
for (const diagnostic of diagnostics) {
  console.warn(diagnostic.message);
}
```

`resolveCliModel()` 会使用全部已注册模型，因此通过 `--api-key` 首次配置时，即使还没有已存储的认证信息，也能解析模型。`resolveModelScopeWithDiagnostics()` 的语义与 `--models` 和 `enabledModels` 一致，但会返回警告，而不是直接打印。

> 参见 [examples/sdk/02-custom-model.ts](../examples/sdk/02-custom-model.ts)

### API 密钥与 OAuth

认证信息解析优先级（由 `ModelRuntime` 处理）：

1. 运行时覆盖值（通过 `setRuntimeApiKey` 设置，不持久化）
2. `auth.json` 中保存的凭据（API 密钥或 OAuth Token）
3. 环境变量（`ANTHROPIC_API_KEY`、`OPENAI_API_KEY` 等）
4. 回退解析器（用于解析 `models.json` 中自定义服务提供商的密钥）

```typescript
import { InMemoryCredentialStore } from "@earendil-works/pi-ai";
import { createAgentSession, ModelRuntime } from "@earendil-works/pi-coding-agent";

// 默认使用 ~/.pi/agent/auth.json 和 ~/.pi/agent/models.json
const modelRuntime = await ModelRuntime.create();

// 服务提供商自己的认证方法和当前状态
for (const provider of modelRuntime.getProviders()) {
  const status = await modelRuntime.checkAuth(provider.id);
  console.log(provider.name, provider.auth, status);
}

// 覆盖运行时 API 密钥（不持久化到磁盘）
modelRuntime.setRuntimeApiKey("anthropic", "sk-my-temp-key");

// 自定义凭据和模型文件位置
const customRuntime = await ModelRuntime.create({
  authPath: "/my/app/auth.json",
  modelsPath: "/my/app/models.json",
});

// 也可以注入任意 pi-ai CredentialStore
const credentials = new InMemoryCredentialStore();
const inMemoryRuntime = await ModelRuntime.create({ credentials });

const { session } = await createAgentSession({
  modelRuntime: customRuntime,
});
```

> 参见 [examples/sdk/09-api-keys-and-oauth.ts](../examples/sdk/09-api-keys-and-oauth.ts)

### 系统提示词

使用 `ResourceLoader` 覆盖系统提示词：

```typescript
import { createAgentSession, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  systemPromptOverride: () => "你是一名乐于助人的助手。",
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> 参见 [examples/sdk/03-custom-prompt.ts](../examples/sdk/03-custom-prompt.ts)

### 工具

指定要启用的内置工具：

- 内置工具名称：`read`、`bash`、`edit`、`write`、`grep`、`find`、`ls`
- 默认启用的内置工具：`read`、`bash`、`edit`、`write`
- `noTools: "all"` 禁用所有工具
- `noTools: "builtin"` 禁用默认内置工具，但保留扩展工具和自定义工具
- `excludeTools` 在应用 `tools` 允许列表后，再禁用指定名称的内置工具、扩展工具或自定义工具

`edit` 工具通过 `details.diff` 返回供 Pi TUI 显示的差异，并通过 `details.patch` 返回供 SDK 使用方处理的标准统一差异补丁。

```typescript
import { createAgentSession } from "@earendil-works/pi-coding-agent";

// 只读模式
const { session } = await createAgentSession({
  tools: ["read", "grep", "find", "ls"],
});

// 选择特定工具
const { session } = await createAgentSession({
  tools: ["read", "bash", "grep"],
});

// 禁用一个工具，保留其他工具
const { session } = await createAgentSession({
  excludeTools: ["ask_question"],
});
```

#### 在自定义 cwd 中使用工具

传入自定义 `cwd` 后，`createAgentSession()` 会为该工作目录构建所选的内置工具。

```typescript
import { createAgentSession, SessionManager } from "@earendil-works/pi-coding-agent";

const cwd = "/path/to/project";

// 在自定义 cwd 中使用默认工具
const { session } = await createAgentSession({
  cwd,
  sessionManager: SessionManager.inMemory(cwd),
});

// 或在自定义 cwd 中选择特定工具
const { session } = await createAgentSession({
  cwd,
  tools: ["read", "bash", "grep"],
  sessionManager: SessionManager.inMemory(cwd),
});
```

> 参见 [examples/sdk/05-tools.ts](../examples/sdk/05-tools.ts)

### 自定义工具

```typescript
import { Type } from "typebox";
import { createAgentSession, defineTool } from "@earendil-works/pi-coding-agent";

// 内联自定义工具
const myTool = defineTool({
  name: "my_tool",
  label: "我的工具",
  description: "执行一项有用的操作",
  parameters: Type.Object({
    input: Type.String({ description: "输入值" }),
  }),
  execute: async (_toolCallId, params) => ({
    content: [{ type: "text", text: `结果：${params.input}` }],
    details: {},
  }),
});

// 直接传入自定义工具
const { session } = await createAgentSession({
  customTools: [myTool],
});
```

对于独立的工具定义以及 `customTools: [myTool]` 这类数组，请使用 `defineTool()`。内联调用 `pi.registerTool({ ... })` 时，参数类型已经可以正确推断。

通过 `customTools` 传入的自定义工具会与扩展注册的工具合并。由 `ResourceLoader` 加载的扩展也可以通过 `pi.registerTool()` 注册工具。

如果传入了 `tools`，需要在其中列出每个要启用的自定义工具或扩展工具名称，例如 `tools: ["read", "bash", "my_tool"]`。

> 参见 [examples/sdk/05-tools.ts](../examples/sdk/05-tools.ts)

### 扩展

扩展由 `ResourceLoader` 加载。`DefaultResourceLoader` 会从 `~/.pi/agent/extensions/`、`.pi/extensions/` 以及 `settings.json` 配置的扩展来源中发现扩展。

```typescript
import { createAgentSession, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  additionalExtensionPaths: ["/path/to/my-extension.ts"],
  extensionFactories: [
    (pi) => {
      pi.on("agent_start", () => {
        console.log("[内联扩展] 智能体正在启动");
      });
    },
  ],
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

扩展可以注册工具、订阅事件、添加命令等。完整 API 请参阅 [extensions.md](extensions-zh.md)。

**具名内联扩展：** 默认情况下，内联工厂函数在启动时的扩展列表中显示为 `<inline:1>`、`<inline:2>` 等。要显示更具描述性的名称，请包装工厂函数：

```typescript
import type { InlineExtension } from "@earendil-works/pi-coding-agent";

const myProvider: InlineExtension = {
  name: "my-provider",
  factory: (pi) => {
    pi.on("agent_start", () => {
      console.log("[my-provider] 智能体正在启动");
    });
  },
};

const loader = new DefaultResourceLoader({
  extensionFactories: [myProvider],
});
```

这样会显示为 `<inline:my-provider>`，而不是 `<inline:1>`。为了向后兼容，仍然接受未包装的工厂函数。

**事件总线：** 扩展可以通过 `pi.events` 通信。如果需要从扩展外部发送或监听事件，请将共享的 `eventBus` 传给 `DefaultResourceLoader`：

```typescript
import { createEventBus, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const eventBus = createEventBus();
const loader = new DefaultResourceLoader({
  eventBus,
});
await loader.reload();

eventBus.on("my-extension:status", (data) => console.log(data));
```

> 参见 [examples/sdk/06-extensions.ts](../examples/sdk/06-extensions.ts) 和 [docs/extensions.md](extensions-zh.md)

<a id="skills"></a>

### 技能

```typescript
import {
  createAgentSession,
  DefaultResourceLoader,
  type Skill,
} from "@earendil-works/pi-coding-agent";

const customSkill: Skill = {
  name: "my-skill",
  description: "自定义指令",
  filePath: "/path/to/SKILL.md",
  baseDir: "/path/to",
  source: "custom",
};

const loader = new DefaultResourceLoader({
  skillsOverride: (current) => ({
    skills: [...current.skills, customSkill],
    diagnostics: current.diagnostics,
  }),
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> 参见 [examples/sdk/04-skills.ts](../examples/sdk/04-skills.ts)

### 上下文文件

```typescript
import { createAgentSession, DefaultResourceLoader } from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  agentsFilesOverride: (current) => ({
    agentsFiles: [
      ...current.agentsFiles,
      { path: "/virtual/AGENTS.md", content: "# 规则\n\n- 保持简洁" },
    ],
  }),
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> 参见 [examples/sdk/07-context-files.ts](../examples/sdk/07-context-files.ts)

### 斜杠命令

```typescript
import {
  createAgentSession,
  DefaultResourceLoader,
  type PromptTemplate,
} from "@earendil-works/pi-coding-agent";

const customCommand: PromptTemplate = {
  name: "deploy",
  description: "部署应用",
  source: "(custom)",
  content: "# 部署\n\n1. 构建\n2. 测试\n3. 部署",
};

const loader = new DefaultResourceLoader({
  promptsOverride: (current) => ({
    prompts: [...current.prompts, customCommand],
    diagnostics: current.diagnostics,
  }),
});
await loader.reload();

const { session } = await createAgentSession({ resourceLoader: loader });
```

> 参见 [examples/sdk/08-prompt-templates.ts](../examples/sdk/08-prompt-templates.ts)

### 会话管理

会话使用通过 `id`/`parentId` 关联的树形结构，因此可以在原会话中直接创建分支。

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSession,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

// 内存会话（不持久化）
const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
});

// 新建持久化会话
const { session: persisted } = await createAgentSession({
  sessionManager: SessionManager.create(process.cwd()),
});

// 继续最近的会话
const { session: continued, modelFallbackMessage } = await createAgentSession({
  sessionManager: SessionManager.continueRecent(process.cwd()),
});
if (modelFallbackMessage) {
  console.log("提示：", modelFallbackMessage);
}

// 打开指定文件
const { session: opened } = await createAgentSession({
  sessionManager: SessionManager.open("/path/to/session.jsonl"),
});

// 列出会话
const currentProjectSessions = await SessionManager.list(process.cwd());
const allSessions = await SessionManager.listAll(process.cwd());

// 用于 /new、/resume、/fork、/clone 和导入流程的会话替换 API。
const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({
      services,
      sessionManager,
      sessionStartEvent,
    })),
    services,
    diagnostics: services.diagnostics,
  };
};

const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

// 使用新会话替换当前活动会话
await runtime.newSession();

// 使用另一个已保存的会话替换当前活动会话
await runtime.switchSession("/path/to/session.jsonl");

// 使用从指定用户条目派生的会话替换当前活动会话
await runtime.fork("entry-id");

// 克隆经过指定条目的当前活动路径
await runtime.fork("entry-id", { position: "at" });
```

**SessionManager 树形结构 API：**

```typescript
const sm = SessionManager.open("/path/to/session.jsonl");

// 列出会话
const currentProjectSessions = await SessionManager.list(process.cwd());
const allSessions = await SessionManager.listAll(process.cwd());

// 遍历条目树
const entries = sm.getEntries();        // 所有条目（不含文件头）
const tree = sm.getTree();              // 完整树形结构
const path = sm.getPath();              // 从根节点到当前叶节点的路径
const leaf = sm.getLeafEntry();         // 当前叶条目
const entry = sm.getEntry(id);          // 按 ID 获取条目
const children = sm.getChildren(id);    // 条目的直接子条目

// 标签
const label = sm.getLabel(id);          // 获取条目的标签
sm.appendLabelChange(id, "checkpoint"); // 设置标签

// 创建分支
sm.branch(entryId);                     // 将叶节点移至较早的条目
sm.branchWithSummary(id, "摘要……");      // 携带上下文摘要创建分支
sm.createBranchedSession(leafId);       // 将路径提取到新文件
```

> 参见 [examples/sdk/11-sessions.ts](../examples/sdk/11-sessions.ts) 和[会话格式](session-format-zh.md)

### 设置管理

```typescript
import { createAgentSession, SettingsManager, SessionManager } from "@earendil-works/pi-coding-agent";

// 默认：从文件加载（合并全局设置和项目设置）
const { session } = await createAgentSession({
  settingsManager: SettingsManager.create(),
});

// 覆盖设置
const settingsManager = SettingsManager.create();
settingsManager.applyOverrides({
  compaction: { enabled: false },
  retry: { enabled: true, maxRetries: 5 },
});
const { session } = await createAgentSession({ settingsManager });

// 内存设置（不执行文件 I/O，适用于测试）
const { session } = await createAgentSession({
  settingsManager: SettingsManager.inMemory({ compaction: { enabled: false } }),
  sessionManager: SessionManager.inMemory(),
});

// 自定义目录
const { session } = await createAgentSession({
  settingsManager: SettingsManager.create("/custom/cwd", "/custom/agent"),
});
```

**静态工厂函数：**

- `SettingsManager.create(cwd?, agentDir?)`——从文件加载
- `SettingsManager.inMemory(settings?)`——不执行文件 I/O

**项目特定设置：**

设置会从以下两个位置加载并合并：

1. 全局：`~/.pi/agent/settings.json`
2. 项目：`<cwd>/.pi/settings.json`

项目设置会覆盖全局设置。嵌套对象按键合并。默认情况下，设置方法修改全局设置。

**持久化与错误处理语义：**

- 对内存状态的设置读取/写入方法是同步的。
- 设置方法会将持久化写入操作异步加入队列。
- 需要建立持久化边界时，请调用 `await settingsManager.flush()`，例如进程退出前，或测试中断言文件内容前。
- `SettingsManager` 不会打印设置 I/O 错误。请使用 `settingsManager.drainErrors()` 获取错误，并在应用层报告。

> 参见 [examples/sdk/10-settings.ts](../examples/sdk/10-settings.ts)

## ResourceLoader

使用 `DefaultResourceLoader` 发现扩展、技能、提示词、主题和上下文文件。

```typescript
import {
  DefaultResourceLoader,
  getAgentDir,
} from "@earendil-works/pi-coding-agent";

const loader = new DefaultResourceLoader({
  cwd,
  agentDir: getAgentDir(),
});
await loader.reload();

const extensions = loader.getExtensions();
const skills = loader.getSkills();
const prompts = loader.getPrompts();
const themes = loader.getThemes();
const contextFiles = loader.getAgentsFiles().agentsFiles;
```

<a id="return-value"></a>

## 返回值

`createAgentSession()` 返回：

```typescript
interface CreateAgentSessionResult {
  // 会话
  session: AgentSession;

  // 扩展加载结果（用于配置运行器）
  extensionsResult: LoadExtensionsResult;

  // 无法恢复会话模型时的警告
  modelFallbackMessage?: string;
}

interface LoadExtensionsResult {
  extensions: Extension[];
  errors: Array<{ path: string; error: string }>;
  runtime: ExtensionRuntime;
}
```

<a id="complete-example"></a>

## 完整示例

```typescript
import { getModel } from "@earendil-works/pi-ai";
import { Type } from "typebox";
import {
  createAgentSession,
  DefaultResourceLoader,
  defineTool,
  ModelRuntime,
  SessionManager,
  SettingsManager,
} from "@earendil-works/pi-coding-agent";

const modelRuntime = await ModelRuntime.create({
  authPath: "/custom/agent/auth.json",
  modelsPath: "/custom/agent/models.json",
});
if (process.env.MY_KEY) {
  modelRuntime.setRuntimeApiKey("anthropic", process.env.MY_KEY);
}

// 内联工具
const statusTool = defineTool({
  name: "status",
  label: "状态",
  description: "获取系统状态",
  parameters: Type.Object({}),
  execute: async () => ({
    content: [{ type: "text", text: `运行时间：${process.uptime()} 秒` }],
    details: {},
  }),
});

const model = getModel("anthropic", "claude-opus-4-5");
if (!model) throw new Error("找不到模型");

// 带覆盖项的内存设置
const settingsManager = SettingsManager.inMemory({
  compaction: { enabled: false },
  retry: { enabled: true, maxRetries: 2 },
});

const loader = new DefaultResourceLoader({
  cwd: process.cwd(),
  agentDir: "/custom/agent",
  settingsManager,
  systemPromptOverride: () => "你是一名精简的助手。保持简洁。",
});
await loader.reload();

const { session } = await createAgentSession({
  cwd: process.cwd(),
  agentDir: "/custom/agent",

  model,
  thinkingLevel: "off",
  modelRuntime,

  tools: ["read", "bash", "status"],
  customTools: [statusTool],
  resourceLoader: loader,

  sessionManager: SessionManager.inMemory(),
  settingsManager,
});

session.subscribe((event) => {
  if (event.type === "message_update" && event.assistantMessageEvent.type === "text_delta") {
    process.stdout.write(event.assistantMessageEvent.delta);
  }
});

await session.prompt("获取状态并列出文件。");
```

<a id="run-modes"></a>

## 运行模式

SDK 导出了多种运行模式实用函数，可基于 `createAgentSession()` 构建自定义界面：

### InteractiveMode

完整的 TUI 交互模式，包含编辑器、对话历史和所有内置命令：

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  InteractiveMode,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};
const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

const mode = new InteractiveMode(runtime, {
  migratedProviders: [],
  modelFallbackMessage: undefined,
  initialMessage: "你好",
  initialImages: [],
  initialMessages: [],
});

await mode.run();
```

### runPrintMode

单次运行模式：发送提示词、输出结果，然后退出：

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  runPrintMode,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};
const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

await runPrintMode(runtime, {
  mode: "text",
  initialMessage: "你好",
  initialImages: [],
  messages: ["后续消息"],
});
```

### runRpcMode

用于子进程集成的 JSON-RPC 模式：

```typescript
import {
  type CreateAgentSessionRuntimeFactory,
  createAgentSessionFromServices,
  createAgentSessionRuntime,
  createAgentSessionServices,
  getAgentDir,
  runRpcMode,
  SessionManager,
} from "@earendil-works/pi-coding-agent";

const createRuntime: CreateAgentSessionRuntimeFactory = async ({ cwd, sessionManager, sessionStartEvent }) => {
  const services = await createAgentSessionServices({ cwd });
  return {
    ...(await createAgentSessionFromServices({ services, sessionManager, sessionStartEvent })),
    services,
    diagnostics: services.diagnostics,
  };
};
const runtime = await createAgentSessionRuntime(createRuntime, {
  cwd: process.cwd(),
  agentDir: getAgentDir(),
  sessionManager: SessionManager.create(process.cwd()),
});

await runRpcMode(runtime);
```

JSON 协议请参阅 [RPC 文档](rpc-zh.md)。

<a id="rpc-mode-alternative"></a>

## RPC 模式替代方案

如果需要基于子进程进行集成，但不想使用 SDK 构建，可以直接使用 CLI：

```bash
pi --mode rpc --no-session
```

JSON 协议请参阅 [RPC 文档](rpc-zh.md)。

以下情况优先使用 SDK：

- 需要类型安全
- 集成代码与 pi 位于同一个 Node.js 进程
- 需要直接访问智能体状态
- 希望以编程方式自定义工具或扩展

以下情况优先使用 RPC 模式：

- 从其他编程语言进行集成
- 需要进程隔离
- 正在构建与编程语言无关的客户端

<a id="exports"></a>

## 导出内容

主入口导出以下内容：

```typescript
// 工厂函数
createAgentSession
createAgentSessionRuntime
AgentSessionRuntime

// 认证与模型
ModelRuntime // 实现 pi-ai Models 并负责存储凭据
ModelRegistry // 面向扩展兼容性的同步门面
resolveCliModel
resolveModelScopeWithDiagnostics

// 资源加载
DefaultResourceLoader
type ResourceLoader
createEventBus

// 常量与辅助函数
CONFIG_DIR_NAME
defineTool
getAgentDir
getPackageDir
getReadmePath
getDocsPath
getExamplesPath

// 会话管理
SessionManager
SettingsManager

// 工具工厂函数
createCodingTools
createReadOnlyTools
createReadTool, createBashTool, createEditTool, createWriteTool
createGrepTool, createFindTool, createLsTool

// 类型
type CreateAgentSessionOptions
type CreateAgentSessionResult
type ExtensionFactory
type InlineExtension
type ExtensionAPI
type ToolDefinition
type Skill
type PromptTemplate
type Tool
```

扩展类型的完整 API 请参阅 [extensions.md](extensions-zh.md)。
