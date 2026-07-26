> pi 可以创建扩展。你可以让它针对自己的使用场景构建扩展。

# 扩展

扩展是用于拓展 pi 行为的 TypeScript 模块。它们可以订阅生命周期事件、注册供 LLM 调用的自定义工具、添加命令等。

> **支持 `/reload` 的放置位置：** 将扩展放入 `~/.pi/agent/extensions/`（全局）或 `.pi/extensions/`（项目本地），pi 会自动发现它们。`pi -e ./path.ts` 只应用于快速测试。位于自动发现目录中的扩展可以通过 `/reload` 热重载。

**主要能力：**

- **自定义工具**——通过 `pi.registerTool()` 注册供 LLM 调用的工具
- **事件拦截**——阻止或修改工具调用、注入上下文、自定义上下文压缩
- **用户交互**——通过 `ctx.ui` 提示用户选择、确认、输入或接收通知
- **自定义 UI 组件**——通过 `ctx.ui.custom()` 创建支持键盘输入的完整 TUI 组件，实现复杂交互
- **自定义命令**——通过 `pi.registerCommand()` 注册 `/mycommand` 之类的命令
- **会话持久化**——通过 `pi.appendEntry()` 存储重启后仍保留的状态
- **自定义渲染**——控制工具调用、工具结果和消息在 TUI 中的呈现方式

**使用场景示例：**

- 权限关卡（执行 `rm -rf`、`sudo` 等命令前要求确认）
- Git 检查点（每轮暂存变更，创建分支时恢复）
- 路径保护（禁止写入 `.env`、`node_modules/`）
- 自定义上下文压缩（按自己的方式总结对话）
- 对话摘要（参见 `summarize.ts` 示例）
- 交互式工具（问答、向导、自定义对话框）
- 有状态工具（待办事项列表、连接池）
- 外部集成（文件监视器、Webhook、CI 触发器）
- 等待期间运行的游戏（参见 `snake.ts` 示例）

可运行实现请参阅 [examples/extensions/](../examples/extensions/)。

## 目录

- [快速开始](#quick-start)
- [扩展位置](#extension-locations)
- [可用导入](#available-imports)
- [编写扩展](#writing-an-extension)
  - [扩展组织形式](#extension-styles)
- [事件](#events)
  - [生命周期概览](#lifecycle-overview)
  - [资源事件](#resource-events)
  - [会话事件](#session-events)
  - [智能体事件](#agent-events)
  - [模型事件](#model-events)
  - [工具事件](#tool-events)
- [ExtensionContext](#extensioncontext)
- [ExtensionCommandContext](#extensioncommandcontext)
- [ExtensionAPI 方法](#extensionapi-methods)
- [状态管理](#state-management)
- [自定义工具](#custom-tools)
  - [动态加载工具](#dynamic-tool-loading)
- [自定义 UI](#custom-ui)
- [错误处理](#error-handling)
- [各模式下的行为](#mode-behavior)
- [示例索引](#examples-reference)

<a id="quick-start"></a>

## 快速开始

创建 `~/.pi/agent/extensions/my-extension.ts`：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // 响应事件
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });

  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  // 注册自定义工具
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone by name",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // 注册命令
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello ${args || "world"}!`, "info");
    },
  });
}
```

使用 `--extension`（或 `-e`）参数测试：

```bash
pi -e ./my-extension.ts
```

<a id="extension-locations"></a>

## 扩展位置

> **安全提示：** 扩展以当前用户的完整系统权限运行，并且可以执行任意代码。只安装来自可信来源的扩展。

pi 会自动从可信位置发现扩展。项目本地的 `.pi/extensions` 条目只有在信任项目后才会加载。

| 位置 | 作用域 |
|----------|-------|
| `~/.pi/agent/extensions/*.ts` | 全局（所有项目） |
| `~/.pi/agent/extensions/*/index.ts` | 全局（子目录） |
| `.pi/extensions/*.ts` | 项目本地 |
| `.pi/extensions/*/index.ts` | 项目本地（子目录） |

可通过 `settings.json` 添加其他路径：

```json
{
  "packages": [
    "npm:@foo/bar@1.0.0",
    "git:github.com/user/repo@v1"
  ],
  "extensions": [
    "/path/to/local/extension.ts",
    "/path/to/local/extension/dir"
  ]
}
```

要通过 npm 或 Git 将扩展作为 pi 包共享，请参阅 [packages.md](packages-zh.md)。

<a id="available-imports"></a>

## 可用导入

| 包 | 用途 |
|---------|---------|
| `@earendil-works/pi-coding-agent` | 扩展类型（`ExtensionAPI`、`ExtensionContext`、事件） |
| `typebox` | 工具参数的 Schema 定义 |
| `@earendil-works/pi-ai` | AI 实用函数（例如用于 Google 兼容枚举的 `StringEnum`） |
| `@earendil-works/pi-tui` | 用于自定义渲染的 TUI 组件 |

也可以使用 npm 依赖。在扩展旁边或其父目录中添加 `package.json`，运行 `npm install` 后，系统会自动解析从 `node_modules/` 导入的内容。

对于通过 `pi install` 安装的分发式 pi 包（npm 或 Git），运行时依赖必须放在 `dependencies` 中。默认情况下，安装包时采用生产环境安装（`npm install --omit=dev`），所以运行时无法使用 `devDependencies`。如果配置了 `npmCommand`，Git 包会使用普通的 `install`，以兼容命令包装器。

也可以使用 Node.js 内置模块（`node:fs`、`node:path` 等）。

<a id="writing-an-extension"></a>

## 编写扩展

扩展默认导出一个接收 `ExtensionAPI` 的工厂函数。该函数可以是同步或异步的：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // 订阅事件
  pi.on("event_name", async (event, ctx) => {
    // 通过 ctx.ui 与用户交互
    const ok = await ctx.ui.confirm("Title", "Are you sure?");
    ctx.ui.notify("Done!", "info");
    ctx.ui.setStatus("my-ext", "Processing...");  // Footer status
    ctx.ui.setWidget("my-ext", ["Line 1", "Line 2"]);  // Widget above editor (default)
  });

  // 注册工具、命令、快捷键和命令行参数
  pi.registerTool({ ... });
  pi.registerCommand("name", { ... });
  pi.registerShortcut("ctrl+x", { ... });
  pi.registerFlag("my-flag", { ... });
}
```

扩展通过 [jiti](https://github.com/unjs/jiti) 加载，因此 TypeScript 无需预先编译即可运行。

如果工厂函数返回 `Promise`，pi 会等待它完成后再继续启动。这意味着异步初始化会先于 `session_start` 和 `resources_discover` 完成，也会先于刷新通过 `pi.registerProvider()` 排队的服务提供商注册操作。

### 异步工厂函数

对于获取远程配置、动态发现可用模型等一次性启动工作，请使用异步工厂函数。

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

使用这种模式，正常启动过程和 `pi --list-models` 都可以使用获取到的模型。

### 长期运行的资源与关闭

某些调用会执行扩展工厂函数，但不会启动会话。因此，不要在工厂函数中启动进程、套接字、文件监视器或定时器等后台资源。

应推迟到 `session_start`，或真正需要该资源的命令、工具、事件中再启动后台资源。同时注册具有幂等性的 `session_shutdown` 处理函数，关闭自己启动的所有会话级资源。

<a id="extension-styles"></a>

### 扩展组织形式

**单文件**——最简单，适用于小型扩展：

```
~/.pi/agent/extensions/
└── my-extension.ts
```

**包含 index.ts 的目录**——适用于多文件扩展：

```
~/.pi/agent/extensions/
└── my-extension/
    ├── index.ts        # 入口（导出默认函数）
    ├── tools.ts        # 辅助模块
    └── utils.ts        # 辅助模块
```

**带依赖的包**——适用于需要 npm 包的扩展：

```
~/.pi/agent/extensions/
└── my-extension/
    ├── package.json    # 声明依赖和入口
    ├── package-lock.json
    ├── node_modules/   # 运行 npm install 后生成
    └── src/
        └── index.ts
```

```json
// package.json
{
  "name": "my-extension",
  "dependencies": {
    "zod": "^3.0.0",
    "chalk": "^5.0.0"
  },
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

在扩展目录中运行 `npm install`，此后即可自动解析从 `node_modules/` 导入的内容。

<a id="events"></a>

## 事件

<a id="lifecycle-overview"></a>

### 生命周期概览

```
pi 启动
  │
  ├─► project_trust（仅用户/全局扩展和 CLI 扩展；加载项目资源前）
  ├─► session_start { reason: "startup" }
  └─► resources_discover { reason: "startup" }
      │
      ▼
用户发送提示词 ────────────────────────────────────────────┐
  │                                                        │
  ├─►（先检查扩展命令；找到后绕过后续输入处理）             │
  ├─► input（可以拦截、转换或直接处理）                     │
  ├─►（未被处理时，展开技能/模板）                          │
  ├─► before_agent_start（可注入消息、修改系统提示词）
  ├─► agent_start                                          │
  ├─► message_start / message_update / message_end         │
  │                                                        │
  │   ┌─── 轮次（LLM 调用工具期间重复）────────────┐       │
  │   │                                            │       │
  │   ├─► turn_start                               │       │
  │   ├─► context（可修改消息）                    │       │
  │   ├─► before_provider_headers（可修改请求头）          |
  │   ├─► before_provider_request（可检查或替换请求体）
  │   ├─► after_provider_response（消费流之前，提供状态与请求头）
  │   │                                            │       │
  │   │   LLM 响应，可能调用工具：                 │       │
  │   │     ├─► tool_execution_start               │       │
  │   │     ├─► tool_call（可以阻止）              │       │
  │   │     ├─► tool_execution_update              │       │
  │   │     ├─► tool_result（可以修改）            │       │
  │   │     └─► tool_execution_end                 │       │
  │   │                                            │       │
  │   └─► turn_end                                 │       │
  │                                                        │
  ├─► agent_end                                            │
  └─► agent_settled（不再有重试、压缩或后续消息）          │
                                                           │
用户发送下一条提示词 ◄─────────────────────────────────────┘

/new（新会话）或 /resume（切换会话）
  ├─► session_before_switch（可以取消）
  ├─► session_shutdown
  ├─► session_start { reason: "new" | "resume", previousSessionFile? }
  └─► resources_discover { reason: "startup" }

/fork 或 /clone
  ├─► session_before_fork（可以取消）
  ├─► session_shutdown
  ├─► session_start { reason: "fork", previousSessionFile }
  └─► resources_discover { reason: "startup" }

/name 或 pi.setSessionName()
  └─► session_info_changed

/compact 或自动压缩
  ├─► session_before_compact（可以取消或自定义）
  └─► session_compact

/tree 导航
  ├─► session_before_tree（可以取消或自定义）
  └─► session_tree

/model 或 Ctrl+P（选择/循环切换模型）
  ├─► thinking_level_select（模型变化导致思考级别变化或受限时）
  └─► model_select

思考级别变化（设置、按键绑定、pi.setThinkingLevel()）
  └─► thinking_level_select

退出（Ctrl+C、Ctrl+D、SIGHUP、SIGTERM）
  └─► session_shutdown
```

### 启动事件

#### project_trust

在 pi 决定是否信任包含动态配置（`.pi` 或 `.agents/skills`）的项目前触发。启动时会触发；替换会话（例如 `/resume`）并进入当前进程尚未判定信任状态的 `cwd` 时也会触发。只有用户/全局扩展和 CLI `-e` 扩展参与；项目本地扩展要等信任状态确定后才会加载。

```typescript
pi.on("project_trust", async (event, ctx) => {
  // event.cwd——当前工作目录
  // ctx 是受限的信任上下文，只包含 cwd、mode、hasUI 和 select/confirm/input/notify UI 辅助方法
  if (await ctx.ui.confirm("Trust project?", event.cwd)) {
    return { trusted: "yes", remember: true };
  }
  return { trusted: "undecided" };
});
```

`project_trust` 处理函数必须返回 `{ trusted: "yes" | "no" | "undecided" }`。用户/全局扩展或 CLI 扩展返回 `"yes"` 或 `"no"` 后即拥有本次决定权；首个明确的“是/否”决定生效，并跳过内置的信任提示。使用 `remember: true` 可持久化明确决定，否则只作用于当前进程。返回 `"undecided"` 则交给后续处理函数或内置信任流程决定。提示用户前应检查 `ctx.hasUI`。如果没有处理函数返回明确决定，则继续正常的信任解析：先应用 `trust.json` 中保存的决定，再由 `defaultProjectTrust` 决定 pi 默认询问、信任还是拒绝。

<a id="resource-events"></a>

### 资源事件

#### resources_discover

在 `session_start` 后触发，使扩展可以提供额外的技能、提示词和主题路径。
启动流程使用 `reason: "startup"`，重载使用 `reason: "reload"`。

```typescript
pi.on("resources_discover", async (event, _ctx) => {
  // event.cwd——当前工作目录
  // event.reason——"startup" | "reload"
  return {
    skillPaths: ["/path/to/skills"],
    promptPaths: ["/path/to/prompts"],
    themePaths: ["/path/to/themes"],
  };
});
```

<a id="session-events"></a>

### 会话事件

会话存储机制和 SessionManager API 请参阅[会话格式](session-format-zh.md)。

#### session_start

启动、加载或重载会话时触发。

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason——"startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile——在 "new"、"resume" 和 "fork" 时存在
  ctx.ui.notify(`Session: ${ctx.sessionManager.getSessionFile() ?? "ephemeral"}`, "info");
});
```

#### session_info_changed

通过 `/name`、RPC 或 `pi.setSessionName()` 设置当前会话显示名称时触发。

```typescript
pi.on("session_info_changed", async (event, ctx) => {
  // event.name——规范化后的当前名称；清除后为 undefined
  ctx.ui.notify(`Session renamed: ${event.name ?? "(none)"}`, "info");
});
```

#### session_before_switch

启动新会话（`/new`）或切换会话（`/resume`）前触发。

```typescript
pi.on("session_before_switch", async (event, ctx) => {
  // event.reason——"new" 或 "resume"
  // event.targetSessionFile——要切换到的会话（仅 "resume" 时存在）

  if (event.reason === "new") {
    const ok = await ctx.ui.confirm("Clear?", "Delete all messages?");
    if (!ok) return { cancel: true };
  }
});
```

成功切换或新建会话后，pi 会为旧扩展实例触发 `session_shutdown`，随后为新会话重新加载并绑定扩展，最后触发带有 `reason: "new" | "resume"` 和 `previousSessionFile` 的 `session_start`。
请在 `session_shutdown` 中完成清理，并在 `session_start` 中重新建立内存状态。

#### session_before_fork

通过 `/fork` 派生会话或通过 `/clone` 克隆会话时触发。

```typescript
pi.on("session_before_fork", async (event, ctx) => {
  // event.entryId——选中条目的 ID
  // event.position——/fork 为 "before"，/clone 为 "at"
  return { cancel: true }; // 取消派生/克隆
  // 或者
  return { skipConversationRestore: true }; // 为将来的对话恢复控制预留
});
```

成功派生或克隆会话后，pi 会为旧扩展实例触发 `session_shutdown`，随后为新会话重新加载并绑定扩展，最后触发带有 `reason: "fork"` 和 `previousSessionFile` 的 `session_start`。
请在 `session_shutdown` 中完成清理，并在 `session_start` 中重新建立内存状态。

#### session_before_compact / session_compact

执行上下文压缩时触发。详情请参阅 [compaction.md](compaction-zh.md)。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, reason, willRetry, signal } = event;

  // reason——"manual"（/compact）、"threshold" 或 "overflow"
  // willRetry——压缩后是否重试被中止的轮次（溢出恢复）

  // 取消：
  return { cancel: true };

  // 自定义摘要：
  return {
    compaction: {
      summary: "...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      // usage: summaryResponse.usage, // 可选；计入会话总计
    }
  };
});

pi.on("session_compact", async (event, ctx) => {
  // event.compactionEntry——已保存的压缩条目
  // event.fromExtension——是否由扩展提供
  // event.reason——"manual"（/compact）、"threshold" 或 "overflow"
  // event.willRetry——压缩后是否重试被中止的轮次（溢出恢复）
});
```

#### session_before_tree / session_tree

使用 `/tree` 导航时触发。树导航概念请参阅[会话](sessions-zh.md)。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;
  return { cancel: true };
  // 或提供自定义摘要：
  return {
    summary: {
      summary: "...",
      // usage: summaryResponse.usage, // 可选；计入会话总计
      details: {},
    },
  };
});

pi.on("session_tree", async (event, ctx) => {
  // event.newLeafId、oldLeafId、summaryEntry、fromExtension
});
```

#### session_shutdown

已经启动的会话运行时销毁前触发。请使用此事件清理在 `session_start` 或其他会话级钩子中打开的资源。

```typescript
pi.on("session_shutdown", async (event, ctx) => {
  // event.reason——"quit" | "reload" | "new" | "resume" | "fork"
  // event.targetSessionFile——会话替换流程的目标会话
  // 清理、保存状态等
});
```

<a id="agent-events"></a>

### 智能体事件

#### before_agent_start

用户提交提示词之后、智能体循环开始之前触发。可以注入消息和/或修改系统提示词。

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  // event.prompt——用户的提示词文本
  // event.images——附带的图像（如有）
  // event.systemPrompt——传给当前处理函数的链式系统提示词
  //   （包含更早的 before_agent_start 处理函数所做的更改）
  // event.systemPromptOptions——用于构建系统提示词的结构化选项
  //   .customPrompt——任意自定义系统提示词（来自 --system-prompt、SYSTEM.md 或自定义模板）
  //   .selectedTools——当前提示词中启用的工具
  //   .toolSnippets——每个工具的单行说明
  //   .promptGuidelines——自定义规则条目
  //   .appendSystemPrompt——来自 --append-system-prompt 参数的文本
  //   .cwd——工作目录
  //   .contextFiles——AGENTS.md 文件和其他已加载的上下文文件
  //   .skills——已加载的技能

  return {
    // 注入持久消息（存入会话并发送给 LLM）
    message: {
      customType: "my-extension",
      content: "Additional context for the LLM",
      display: true,
    },
    // 替换本轮的系统提示词（在扩展之间链式传递）
    systemPrompt: event.systemPrompt + "\n\nExtra instructions for this turn...",
  };
});
```

`systemPromptOptions` 字段使扩展能够访问 Pi 构建系统提示词时使用的同一份结构化数据。这样，无需重新发现资源或解析命令行参数，就能检查 Pi 已加载的自定义提示词、规则、工具简介、上下文文件和技能。如果扩展需要在尊重用户配置的前提下，对系统提示词进行深入且有依据的修改，请使用该字段。

在 `before_agent_start` 中，`event.systemPrompt` 和 `ctx.getSystemPrompt()` 都反映截至当前处理函数为止的链式系统提示词。后续的 `before_agent_start` 处理函数仍可继续修改。

#### agent_start / agent_end / agent_settled

底层智能体运行开始时触发 `agent_start`，结束时触发 `agent_end`；但此后 Pi 仍可能自动重试、自动压缩后重试，或继续处理队列中的后续消息。如果状态集成需要确定 Pi 不会继续自动运行，请使用 `agent_settled`。

```typescript
pi.on("agent_start", async (_event, ctx) => {});

pi.on("agent_end", async (event, ctx) => {
  // event.messages——本次底层运行产生的消息
});

pi.on("agent_settled", async (_event, ctx) => {
  // 除非另一个扩展启动了新的运行，否则此处 ctx.isIdle() 为 true。
});
```

#### turn_start / turn_end

每个轮次（一次 LLM 响应及工具调用）触发。

```typescript
pi.on("turn_start", async (event, ctx) => {
  // event.turnIndex、event.timestamp
});

pi.on("turn_end", async (event, ctx) => {
  // event.turnIndex、event.message、event.toolResults
});
```

#### message_start / message_update / message_end

消息生命周期更新时触发。

- 用户、助手和 `toolResult` 消息都会触发 `message_start` 和 `message_end`。
- 助手消息的流式更新会触发 `message_update`。
- `message_end` 处理函数可以返回 `{ message }`，替换最终消息。替换后的消息必须保持相同的 `role`。

```typescript
pi.on("message_start", async (event, ctx) => {
  // event.message
});

pi.on("message_update", async (event, ctx) => {
  // event.message
  // event.assistantMessageEvent（逐 Token 流式事件）
});

pi.on("message_end", async (event, ctx) => {
  if (event.message.role !== "assistant") return;

  return {
    message: {
      ...event.message,
      usage: {
        ...event.message.usage,
        cost: {
          ...event.message.usage.cost,
          total: 0.123,
        },
      },
    },
  };
});
```

#### tool_execution_start / tool_execution_update / tool_execution_end

工具执行生命周期更新时触发。

在并行工具模式下：

- 预检阶段会按助手消息中的原始顺序触发 `tool_execution_start`
- 不同工具的 `tool_execution_update` 事件可能交错
- 每个工具完成最终处理后，会按工具完成顺序触发 `tool_execution_end`
- 最终的 `toolResult` 消息事件仍会稍后按助手消息中的原始顺序触发

```typescript
pi.on("tool_execution_start", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args
});

pi.on("tool_execution_update", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args, event.partialResult
});

pi.on("tool_execution_end", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.result, event.isError
});
```

#### context

每次调用 LLM 前触发。可以以非破坏方式修改消息。消息类型请参阅[会话格式](session-format-zh.md)。

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages——深拷贝，可以安全修改
  const filtered = event.messages.filter(m => !shouldPrune(m));
  return { messages: filtered };
});
```

#### before_provider_headers

传出 HTTP 请求头组装完成后触发。可以用它添加、覆盖或移除请求头。

处理函数直接修改 `event.headers`。将键设为字符串可添加或覆盖请求头；设为 `null` 可删除请求头。

```typescript
pi.on("before_provider_headers", (event, ctx) => {
  // 添加或覆盖——例如用于网关追踪/归因的会话 ID
  event.headers["x-session-id"] = ctx.sessionManager.getSessionId();

  // 删除 pi 为本次调用添加的跟踪请求头
  event.headers["X-OpenRouter-Title"] = null;
});
```

每个服务提供商请求只运行一次；重试时会复用相同的请求头，不会再次触发该钩子。

#### before_provider_request

服务提供商特有的请求体构建完成后、真正发送请求前触发。处理函数按扩展加载顺序执行。返回 `undefined` 会保持请求体不变；返回其他任何值，都会替换传给后续处理函数和实际请求的请求体。

该钩子可以改写或完全移除服务提供商级的系统指令。这些请求体级别的修改不会反映在 `ctx.getSystemPrompt()` 中；后者报告的是 Pi 的系统提示词字符串，而不是最终序列化后的服务提供商请求体。

```typescript
pi.on("before_provider_request", (event, ctx) => {
  console.log(JSON.stringify(event.payload, null, 2));

  // 可选：替换请求体
  // return { ...event.payload, temperature: 0 };
});
```

该钩子主要用于调试服务提供商序列化和缓存行为。

#### after_provider_response

收到 HTTP 响应后、开始消费其流式响应体前触发。处理函数按扩展加载顺序执行。

```typescript
pi.on("after_provider_response", (event, ctx) => {
  // event.status——HTTP 状态码
  // event.headers——规范化后的响应头
  if (event.status === 429) {
    console.log("rate limited", event.headers["retry-after"]);
  }
});
```

能否获取响应头取决于服务提供商和传输层。对 HTTP 响应进行了抽象的服务提供商可能不会暴露响应头。

<a id="model-events"></a>

### 模型事件

#### model_select

通过 `/model` 命令、循环切换模型（`Ctrl+P`）或恢复会话而改变模型时触发。

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model——新选中的模型
  // event.previousModel——上一个模型（首次选择时为 undefined）
  // event.source——"set" | "cycle" | "restore"

  const prev = event.previousModel
    ? `${event.previousModel.provider}/${event.previousModel.id}`
    : "none";
  const next = `${event.model.provider}/${event.model.id}`;

  ctx.ui.notify(`Model changed (${event.source}): ${prev} -> ${next}`, "info");
});
```

可以在活动模型变化时，使用此事件更新状态栏、页脚等 UI 元素，或执行模型特有的初始化。

#### thinking_level_select

思考级别变化时触发。该事件仅用于通知，处理函数的返回值会被忽略。

```typescript
pi.on("thinking_level_select", async (event, ctx) => {
  // event.level——新选中的思考级别
  // event.previousLevel——上一个思考级别

  ctx.ui.setStatus("thinking", `thinking: ${event.level}`);
});
```

当 `pi.setThinkingLevel()`、模型变化或内置思考级别控件改变活动思考级别时，可以使用此事件更新扩展 UI。

<a id="tool-events"></a>

### 工具事件

#### tool_call

在 `tool_execution_start` 之后、工具执行之前触发。**可以阻止工具执行。** 使用 `isToolCallEventType` 缩小类型范围并获得带类型的输入。

运行 `tool_call` 前，pi 会等待此前触发的智能体事件全部经过 `AgentSession` 处理完毕。因此，`ctx.sessionManager` 会更新到当前包含工具调用的助手消息。

在默认的并行工具执行模式下，同一助手消息中的同级工具调用会依次预检，然后并发执行。`tool_call` 无法保证能在 `ctx.sessionManager` 中看到同一助手消息内其他同级工具调用的结果。

`event.input` 可变。可以在执行前直接修改它，以修补工具参数。

行为保证：

- 对 `event.input` 的修改会影响实际工具执行
- 后续的 `tool_call` 处理函数可以看到早期处理函数所做的修改
- 修改后不会重新验证参数
- `tool_call` 的返回值只能通过 `{ block: true, reason?: string }` 控制是否阻止执行

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  // event.toolName——"bash"、"read"、"write"、"edit" 等
  // event.toolCallId
  // event.input——工具参数（可变）

  // 内置工具：无需类型参数
  if (isToolCallEventType("bash", event)) {
    // event.input 为 { command: string; timeout?: number }
    event.input.command = `source ~/.profile\n${event.input.command}`;

    if (event.input.command.includes("rm -rf")) {
      return { block: true, reason: "Dangerous command" };
    }
  }

  if (isToolCallEventType("read", event)) {
    // event.input 为 { path: string; offset?: number; limit?: number }
    console.log(`Reading: ${event.input.path}`);
  }
});
```

#### 为自定义工具输入添加类型

自定义工具应导出其输入类型：

```typescript
// my-extension.ts
export type MyToolInput = Static<typeof myToolSchema>;
```

通过显式类型参数使用 `isToolCallEventType`：

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";
import type { MyToolInput } from "my-extension";

pi.on("tool_call", (event) => {
  if (isToolCallEventType<"my_tool", MyToolInput>("my_tool", event)) {
    event.input.action;  // 已获得类型
  }
});
```

#### tool_result

工具执行完成后、触发 `tool_execution_end` 和最终工具结果消息事件前触发。**可以修改结果。**

在并行工具模式下，`tool_result` 和 `tool_execution_end` 可能按工具完成顺序交错触发，而最终的 `toolResult` 消息事件仍会稍后按助手消息中的原始顺序触发。

`tool_result` 处理函数像中间件一样串联：

- 处理函数按扩展加载顺序执行
- 每个处理函数都能看到上一个处理函数修改后的最新结果
- 处理函数可以返回部分补丁（`content`、`details`、`isError` 或 `usage`）；省略的字段保持当前值

处理函数内部执行嵌套异步工作时，请使用 `ctx.signal`。这样，按 Escape 可以取消扩展发起的模型调用、`fetch()` 和其他支持中止信号的操作。

```typescript
import { isBashToolResult } from "@earendil-works/pi-coding-agent";

pi.on("tool_result", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input
  // event.content, event.details, event.isError, event.usage

  if (isBashToolResult(event)) {
    // event.details 的类型为 BashToolDetails
  }

  const response = await fetch("https://example.com/summarize", {
    method: "POST",
    body: JSON.stringify({ content: event.content }),
    signal: ctx.signal,
  });

  // 修改结果：
  return { content: [...], details: {...}, isError: false, usage: nestedModelUsage };
});
```

### 用户 Bash 事件

#### user_bash

用户执行 `!` 或 `!!` 命令时触发。**可以拦截。**

```typescript
import { createLocalBashOperations } from "@earendil-works/pi-coding-agent";

pi.on("user_bash", (event, ctx) => {
  // event.command——Bash 命令
  // event.excludeFromContext——以 !! 开头时为 true
  // event.cwd——工作目录

  // 方案 1：提供自定义操作（例如 SSH）
  return { operations: remoteBashOps };

  // 方案 2：包装 pi 内置的本地 Bash 后端
  const local = createLocalBashOperations();
  return {
    operations: {
      exec(command, cwd, options) {
        return local.exec(`source ~/.profile\n${command}`, cwd, options);
      }
    }
  };

  // 方案 3：完全替换——直接返回结果
  return { result: { output: "...", exitCode: 0, cancelled: false, truncated: false } };
});
```

### 输入事件

#### input

收到用户输入时，在检查扩展命令之后、展开技能和模板之前触发。事件看到的是原始输入文本，所以 `/skill:foo` 和 `/template` 尚未展开。

**处理顺序：**

1. 先检查扩展命令（`/cmd`）；如果找到，则运行相应处理函数，并跳过输入事件
2. 触发 `input` 事件；可以拦截、转换或直接处理
3. 如果尚未处理，将技能命令（`/skill:name`）展开为技能内容
4. 如果仍未处理，将提示词模板（`/template`）展开为模板内容
5. 开始智能体处理（`before_agent_start` 等）

```typescript
pi.on("input", async (event, ctx) => {
  // event.text——原始输入（展开技能/模板前）
  // event.images——附带的图像（如有）
  // event.source——"interactive"（键入）、"rpc"（API）或 "extension"（通过 sendUserMessage）
  // event.streamingBehavior——"steer" | "followUp" | undefined
  //   空闲时为 undefined；流式响应期间打断时为 "steer"；
  //   排队到智能体完成后再发送的消息为 "followUp"

  // 转换：展开前改写输入
  if (event.text.startsWith("?quick "))
    return { action: "transform", text: `Respond briefly: ${event.text.slice(7)}` };

  // 处理：不使用 LLM 直接响应（扩展自行显示反馈）
  if (event.text === "ping") {
    ctx.ui.notify("pong", "info");
    return { action: "handled" };
  }

  // 按来源路由：跳过扩展注入消息的处理
  if (event.source === "extension") return { action: "continue" };

  // 在展开前拦截技能命令
  if (event.text.startsWith("/skill:")) {
    // 可以转换、阻止或放行
  }

  return { action: "continue" };  // 默认：放行并继续展开
});
```

**结果：**

- `continue`——保持原样并继续处理（处理函数不返回任何内容时的默认行为）
- `transform`——修改文本/图像，然后继续展开
- `handled`——完全跳过智能体（第一个返回该结果的处理函数生效）

转换会在多个处理函数之间链式应用。根据 `streamingBehavior` 路由的示例，请参阅 [input-transform.ts](../examples/extensions/input-transform.ts) 和 [input-transform-streaming.ts](../examples/extensions/input-transform-streaming.ts)。

## ExtensionContext

所有处理函数都会收到 `ctx: ExtensionContext`。

### ctx.ui

用于与用户交互的 UI 方法。完整说明请参阅[自定义 UI](#custom-ui)。

### ctx.mode

当前运行模式：`"tui"`、`"rpc"`、`"json"` 或 `"print"`。对于 `custom()`、组件工厂函数、终端输入和直接 TUI 渲染等仅限终端的功能，请使用 `ctx.mode === "tui"` 进行保护。

### ctx.hasUI

在 TUI 和 RPC 模式中为 `true`，在打印模式（`-p`）和 JSON 模式中为 `false`。对于同时支持 TUI 和 RPC 模式的对话框方法（`select`、`confirm`、`input`、`editor`）及发送后不等待的方法（`notify`、`setStatus`、`setWidget`、`setTitle`、`setEditorText`），请使用该属性进行保护。在 RPC 模式中，部分 TUI 特有方法不执行任何操作或返回默认值（参见 [rpc.md](rpc-zh.md#extension-ui-protocol)）。

### ctx.cwd

当前工作目录。

构建项目本地配置路径时，请使用 `CONFIG_DIR_NAME`，不要硬编码 `.pi`。更换品牌的发行版可能使用其他配置目录名称。

```typescript
import { CONFIG_DIR_NAME, type ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { join } from "node:path";

export default function (pi: ExtensionAPI) {
  pi.on("session_start", (_event, ctx) => {
    const projectConfigPath = join(ctx.cwd, CONFIG_DIR_NAME, "my-extension.json");
    // ...
  });
}
```

### ctx.isProjectTrusted()

返回当前会话上下文是否已启用项目本地信任。它不仅考虑全局信任存储中保存的决定，还包括临时信任决定和 CLI 信任覆盖项。

读取只应对可信项目生效的项目本地扩展配置前，请先调用此方法。

### ctx.sessionManager

以只读方式访问会话状态。完整的 SessionManager API 和条目类型请参阅[会话格式](session-format-zh.md)。

对于 `tool_call`，运行处理函数前会将该状态同步到当前助手消息。但在并行工具执行模式下，仍无法保证其中包含同一助手消息内其他同级工具调用的结果。

```typescript
ctx.sessionManager.getEntries()             // 所有条目
ctx.sessionManager.getBranch()              // 当前分支
ctx.sessionManager.buildContextEntries()    // 应用上下文压缩后的活动分支条目
ctx.sessionManager.getLeafId()              // 当前叶条目 ID
```

### ctx.modelRegistry / ctx.model / ctx.thinkingLevel

访问模型、服务提供商和解析后的认证信息。`ctx.modelRegistry.getProvider(id)` 返回实际生效的 pi-ai 服务提供商；`getProviderAuth(id)` 无需加载模型，即可解析其当前 API 密钥、请求头、基础 URL 和服务提供商作用域环境。`ctx.model` 是活动模型，`ctx.thinkingLevel` 是其当前实际生效的思考级别。

### ctx.signal

当前智能体的中止信号；没有活动的智能体轮次时为 `undefined`。

扩展处理函数启动支持中止的嵌套工作时，请使用该信号，例如：

- `fetch(..., { signal: ctx.signal })`
- 接受 `signal` 的模型调用
- 接受 `AbortSignal` 的文件或进程辅助函数

在 `tool_call`、`tool_result`、`message_update`、`turn_end` 等活动轮次事件期间，通常会定义 `ctx.signal`。
在会话事件、扩展命令和 pi 空闲时触发的快捷键等空闲或非轮次上下文中，它通常为 `undefined`。

```typescript
pi.on("tool_result", async (event, ctx) => {
  const response = await fetch("https://example.com/api", {
    method: "POST",
    body: JSON.stringify(event),
    signal: ctx.signal,
  });

  const data = await response.json();
  return { details: data };
});
```

### ctx.isIdle() / ctx.abort() / ctx.hasPendingMessages()

流程控制辅助方法。当 Pi 正在处理智能体运行、自动重试、自动压缩重试或队列中的后续操作时，`ctx.isIdle()` 为 `false`。

### ctx.shutdown()

请求平稳关闭 pi。

- **交互模式：** 延迟到智能体空闲后执行，即处理完队列中的所有引导消息和后续消息后。
- **RPC 模式：** 延迟到下一个空闲状态执行，即完成当前命令响应、等待下一条命令时。
- **打印模式：** 不执行任何操作。所有提示词处理完毕后，进程会自动退出。

退出前会向所有扩展触发 `session_shutdown` 事件。所有上下文（事件处理函数、工具、命令、快捷键）中均可调用。

```typescript
pi.on("tool_call", (event, ctx) => {
  if (isFatal(event.input)) {
    ctx.shutdown();
  }
});
```

### ctx.getContextUsage()

返回活动模型当前的上下文用量。优先使用最近一条助手消息的用量；如果需要，再估算末尾消息的 Token 数。

```typescript
const usage = ctx.getContextUsage();
if (usage && usage.tokens > 100_000) {
  // ...
}
```

### ctx.compact()

触发上下文压缩，但不等待完成。使用 `onComplete` 和 `onError` 执行后续操作。

```typescript
ctx.compact({
  customInstructions: "Focus on recent changes",
  onComplete: (result) => {
    ctx.ui.notify("Compaction completed", "info");
  },
  onError: (error) => {
    ctx.ui.notify(`Compaction failed: ${error.message}`, "error");
  },
});
```

### ctx.getSystemPrompt()

返回 Pi 当前的系统提示词字符串。

- 在 `before_agent_start` 期间，它反映本轮截至当前已经链式应用的系统提示词更改。
- 不包含后续 `context` 事件对消息所做的修改。
- 不包含 `before_provider_request` 对请求体所做的改写。
- 如果加载顺序更靠后的扩展在当前扩展之后运行，它们仍然可以改变最终发送的内容。

```typescript
pi.on("before_agent_start", (event, ctx) => {
  const prompt = ctx.getSystemPrompt();
  console.log(`System prompt length: ${prompt.length}`);
});
```

## ExtensionCommandContext

命令处理函数接收 `ExtensionCommandContext`。它在 `ExtensionContext` 基础上增加了会话控制方法。这些方法只在命令中可用，因为从事件处理函数中调用可能导致死锁。

### ctx.getSystemPromptOptions()

返回 Pi 当前用于构建系统提示词的基础输入。

```typescript
const options = ctx.getSystemPromptOptions();
const contextPaths = options.contextFiles?.map((file) => file.path) ?? [];
```

其结构和可变性与 `before_agent_start` 的 `event.systemPromptOptions` 相同，包括自定义提示词、活动工具、工具简介、提示词规则、追加的系统提示词文本、`cwd`、已加载的上下文文件和技能。它可能包含上下文文件的完整内容，因此应将其视为扩展内部的敏感数据，不要通过命令列表、日志或自动补全元数据对外暴露。

该方法报告当前的基础提示词输入，不包含每轮 `before_agent_start` 链式应用的系统提示词更改、后续 `context` 事件的消息修改或 `before_provider_request` 的请求体改写。

### ctx.waitForIdle()

等待智能体完全结束，包括自动重试、自动压缩重试和队列中的后续操作：

```typescript
pi.registerCommand("my-cmd", {
  handler: async (args, ctx) => {
    await ctx.waitForIdle();
    // 智能体现已空闲，可以安全修改会话
  },
});
```

### ctx.newSession(options?)

创建新会话：

```typescript
const parentSession = ctx.sessionManager.getSessionFile();
const kickoff = "Continue in the replacement session";

const result = await ctx.newSession({
  parentSession,
  setup: async (sm) => {
    sm.appendMessage({
      role: "user",
      content: [{ type: "text", text: "Context from previous session..." }],
      timestamp: Date.now(),
    });
  },
  withSession: async (ctx) => {
    // 此处只使用替换后会话的 ctx。
    await ctx.sendUserMessage(kickoff);
  },
});

if (result.cancelled) {
  // 某个扩展取消了新建会话
}
```

选项：

- `parentSession`：要记录在新会话文件头中的父会话文件
- `setup`：在运行 `withSession` 前修改新会话的 `SessionManager`
- `withSession`：使用全新的替换后会话上下文执行切换后的工作。不要使用捕获的旧 `pi` 或命令 `ctx`；参见[会话替换生命周期与常见陷阱](#session-replacement-lifecycle-and-footguns)。

### ctx.fork(entryId, options?)

从指定条目派生，创建新的会话文件：

```typescript
const result = await ctx.fork("entry-id-123", {
  withSession: async (ctx) => {
    // 此处只使用替换后会话的 ctx。
    ctx.ui.notify("Now in the forked session", "info");
  },
});
if (result.cancelled) {
  // 某个扩展取消了派生操作
}

const cloneResult = await ctx.fork("entry-id-456", { position: "at" });
if (cloneResult.cancelled) {
  // 某个扩展取消了克隆操作
}
```

选项：

- `position: "before"`（默认值）：在选中的用户消息之前派生，并将该提示词恢复到编辑器中
- `position: "at"`：复制经过选中条目的活动路径，不恢复编辑器文本
- `withSession`：使用全新的替换后会话上下文执行切换后的工作。不要使用捕获的旧 `pi` 或命令 `ctx`；参见[会话替换生命周期与常见陷阱](#session-replacement-lifecycle-and-footguns)。

### ctx.navigateTree(targetId, options?)

导航到会话树中的另一个位置：

```typescript
const result = await ctx.navigateTree("entry-id-456", {
  summarize: true,
  customInstructions: "Focus on error handling changes",
  replaceInstructions: false, // true = replace default prompt entirely
  label: "review-checkpoint",
});
```

选项：

- `summarize`：是否为放弃的分支生成摘要
- `customInstructions`：提供给摘要生成器的自定义指令
- `replaceInstructions`：如果为 `true`，`customInstructions` 会替换默认提示词，而不是追加到其后
- `label`：附加到分支摘要条目的标签；不生成摘要时则附加到目标条目

### ctx.switchSession(sessionPath, options?)

切换到另一个会话文件：

```typescript
const result = await ctx.switchSession("/path/to/session.jsonl", {
  withSession: async (ctx) => {
    await ctx.sendUserMessage("Resume work in the replacement session");
  },
});
if (result.cancelled) {
  // 某个扩展通过 session_before_switch 取消了切换
}
```

选项：

- `withSession`：使用全新的替换后会话上下文执行切换后的工作。不要使用捕获的旧 `pi` 或命令 `ctx`；参见[会话替换生命周期与常见陷阱](#session-replacement-lifecycle-and-footguns)。

要发现可用会话，请使用静态方法 `SessionManager.list()` 或 `SessionManager.listAll()`：

```typescript
import { SessionManager } from "@earendil-works/pi-coding-agent";

pi.registerCommand("switch", {
  description: "Switch to another session",
  handler: async (args, ctx) => {
    const sessions = await SessionManager.list(ctx.cwd);
    if (sessions.length === 0) return;
    const choice = await ctx.ui.select(
      "Pick session:",
      sessions.map(s => s.file),
    );
    if (choice) {
      await ctx.switchSession(choice, {
        withSession: async (ctx) => {
          ctx.ui.notify("Switched session", "info");
        },
      });
    }
  },
});
```

<a id="session-replacement-lifecycle-and-footguns"></a>

### 会话替换生命周期与常见陷阱

`withSession` 接收全新的 `ReplacedSessionContext`。该上下文扩展自 `ExtensionCommandContext`，并增加了绑定到替换后会话的异步 `sendMessage()` 和 `sendUserMessage()` 辅助方法。

生命周期与常见陷阱：

- 只有在旧会话已经触发 `session_shutdown`、旧运行时已经销毁、替换后会话已经重新绑定，并且新扩展实例已经收到 `session_start` 之后，才会运行 `withSession`。
- 回调仍在原闭包中执行，而不是在新扩展实例中执行。这意味着在 `withSession` 开始前，旧扩展实例可能已经执行了关闭清理。
- 替换会话后，捕获的旧 `pi` 或旧命令 `ctx` 中绑定会话的对象已经失效，继续使用会抛出错误。会话相关工作只能使用传给 `withSession` 的 `ctx`。
- 此前提取的原始对象仍需由扩展自行负责。例如，如果在替换前捕获 `const sm = ctx.sessionManager`，`sm` 仍然是旧的 `SessionManager` 对象。替换后不要复用。
- `withSession` 中的代码应假定所有被 `session_shutdown` 处理函数清除的状态都已不存在。只能捕获可以在关闭后安全保留的普通数据，例如字符串、ID 和序列化配置。

安全模式：

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const kickoff = "Continue from the replacement session";
    await ctx.newSession({
      withSession: async (ctx) => {
        await ctx.sendUserMessage(kickoff);
      },
    });
  },
});
```

不安全模式：

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const oldSessionManager = ctx.sessionManager;
    await ctx.newSession({
      withSession: async (_ctx) => {
        // 已失效的旧对象：不要这样做
        oldSessionManager.getSessionFile();
        pi.sendUserMessage("wrong");
      },
    });
  },
});
```

### ctx.reload()

运行与 `/reload` 相同的重载流程。

```typescript
pi.registerCommand("reload-runtime", {
  description: "Reload extensions, skills, prompts, themes, and context files",
  handler: async (_args, ctx) => {
    await ctx.reload();
    return;
  },
});
```

重要行为：

- `await ctx.reload()` 会为当前扩展运行时触发 `session_shutdown`
- 随后重新加载资源，并触发带有 `reason: "reload"` 的 `session_start` 和 `resources_discover`
- 当前正在运行的命令处理函数仍会在旧调用帧中继续执行
- `await ctx.reload()` 后的代码仍然来自重载前的版本
- `await ctx.reload()` 后的代码不得假定旧的扩展内存状态仍然有效
- 处理函数返回后，后续命令、事件和工具调用会使用新版本扩展

为确保行为可预测，应将重载视为该处理函数的终点（`await ctx.reload(); return;`）。

工具使用 `ExtensionContext` 运行，因此无法直接调用 `ctx.reload()`。请使用命令作为重载入口，再提供一个工具，将该命令作为后续用户消息加入队列。

以下工具可供 LLM 调用，以触发重载：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("reload-runtime", {
    description: "Reload extensions, skills, prompts, themes, and context files",
    handler: async (_args, ctx) => {
      await ctx.reload();
      return;
    },
  });

  pi.registerTool({
    name: "reload_runtime",
    label: "Reload Runtime",
    description: "Reload extensions, skills, prompts, themes, and context files",
    parameters: Type.Object({}),
    async execute() {
      pi.sendUserMessage("/reload-runtime", { deliverAs: "followUp" });
      return {
        content: [{ type: "text", text: "Queued /reload-runtime as a follow-up command." }],
      };
    },
  });
}
```

<a id="extensionapi-methods"></a>

## ExtensionAPI 方法

### pi.on(event, handler)

订阅事件。事件类型和返回值请参阅[事件](#events)。

### pi.registerTool(definition)

注册供 LLM 调用的自定义工具。完整说明请参阅[自定义工具](#custom-tools)。

`pi.registerTool()` 在扩展加载期间和启动后都可使用。可以在 `session_start`、命令处理函数或其他事件处理函数中调用。新工具会立即刷新到当前会话中，因此无需 `/reload` 就会出现在 `pi.getAllTools()` 中，并可供 LLM 调用。

使用 `pi.setActiveTools()` 可以在运行时启用或禁用工具，包括动态添加的工具。

使用 `promptSnippet` 可以让自定义工具在 `Available tools` 中显示为单行条目；使用 `promptGuidelines` 可以在工具活动时，将工具特有的条目追加到默认的 `Guidelines` 部分。

**重要：** `promptGuidelines` 条目会平铺追加到 `Guidelines` 部分，不带工具名称前缀。每条规则必须明确写出所指工具的名称。不要写“Use this tool when...”，因为 LLM 无法判断“this”指哪个工具；应写成“Use my_tool when...”。

完整示例请参阅 [dynamic-tools.ts](../examples/extensions/dynamic-tools.ts)。

```typescript
import { Type } from "typebox";
import { StringEnum } from "@earendil-works/pi-ai";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "What this tool does",
  promptSnippet: "Summarize or transform text according to action",
  promptGuidelines: ["Use my_tool when the user asks to summarize previously generated text."],
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),
    text: Type.Optional(Type.String()),
  }),
  prepareArguments(args) {
    // 可选的兼容转换层，在 Schema 验证前运行。
    // 返回符合当前 Schema 的结构，例如将旧字段
    // 折叠进当前参数对象。
    return args;
  },

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // 流式报告进度
    onUpdate?.({ content: [{ type: "text", text: "Working..." }] });

    return {
      content: [{ type: "text", text: "Done" }],
      details: { result: "..." },
    };
  },

  // 可选：自定义渲染
  renderCall(args, theme, context) { ... },
  renderResult(result, options, theme, context) { ... },
});
```

### pi.sendMessage(message, options?)

向会话中注入自定义消息。自定义消息会进入 LLM 上下文。对于需要持久保留、只在 TUI 中显示且不应发送给 LLM 的内容，请组合使用 [`pi.appendEntry()`](#piappendentrycustomtype-data) 和 [`pi.registerEntryRenderer()`](#piregisterentryrenderercustomtype-renderer)。

```typescript
pi.sendMessage({
  customType: "my-extension",
  content: "Message text",
  display: true,
  details: { ... },
}, {
  triggerTurn: true,
  deliverAs: "steer",
});
```

**选项：**

- `deliverAs`——发送模式：
  - `"steer"`（默认值）——流式响应期间将消息加入队列。在当前助手轮次完成工具调用后、下一次调用 LLM 前发送。
  - `"followUp"`——等待智能体完成。只有智能体不再有工具调用时才发送。
  - `"nextTurn"`——加入队列，等待下一条用户提示词。不会打断或触发任何操作。
- `triggerTurn: true`——如果智能体空闲，立即触发 LLM 响应。只适用于 `"steer"` 和 `"followUp"` 模式；在 `"nextTurn"` 模式中被忽略。

### pi.sendUserMessage(content, options?)

向智能体发送用户消息。`sendMessage()` 发送的是自定义消息，而此方法发送的是真实用户消息，效果如同用户亲自输入。它始终会触发新轮次。

```typescript
// 简单文本消息
pi.sendUserMessage("What is 2+2?");

// 使用内容数组（文本 + 图像）
pi.sendUserMessage([
  { type: "text", text: "Describe this image:" },
  { type: "image", source: { type: "base64", mediaType: "image/png", data: "..." } },
]);

// 流式响应期间——必须指定发送模式
pi.sendUserMessage("Focus on error handling", { deliverAs: "steer" });
pi.sendUserMessage("And then summarize", { deliverAs: "followUp" });
```

**选项：**

- `deliverAs`——智能体正在进行流式响应时必填：
  - `"steer"`——将消息加入队列，在当前助手轮次完成工具调用后发送
  - `"followUp"`——等待智能体完成所有工具调用

未进行流式响应时，消息会立即发送并触发新轮次。流式响应期间未设置 `deliverAs` 时，会抛出错误。

完整示例请参阅 [send-user-message.ts](../examples/extensions/send-user-message.ts)。

### pi.appendEntry(customType, data?)

持久化扩展数据。自定义条目**不会**进入 LLM 上下文。在交互模式中，配合 `pi.registerEntryRenderer()` 还可以将它们渲染到对话记录中。

```typescript
pi.appendEntry("my-state", { count: 42 });
pi.appendEntry("status-card", { title: "Indexed files", count: 17 });

// 重载时恢复
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      // 根据 entry.data 重建
    }
  }
});
```

### pi.setSessionName(name)

设置会话显示名称。设置后，会话选择器会显示该名称，而不是首条消息。

```typescript
pi.setSessionName("Refactor auth module");
```

### pi.getSessionName()

获取当前会话名称；未设置时为空。

```typescript
const name = pi.getSessionName();
if (name) {
  console.log(`Session: ${name}`);
}
```

### pi.setLabel(entryId, label)

设置或清除条目标签。标签是用户定义的标记，用于添加书签和导航，会显示在 `/tree` 选择器中。

```typescript
// 设置标签
pi.setLabel(entryId, "checkpoint-before-refactor");

// 清除标签
pi.setLabel(entryId, undefined);

// 通过 sessionManager 读取标签
const label = ctx.sessionManager.getLabel(entryId);
```

标签会持久化在会话中，重启后仍然保留。可以用它们标记对话树中的重要位置，例如轮次或检查点。

### pi.registerCommand(name, options)

注册命令。

如果多个扩展注册了同名命令，pi 会保留所有命令，并按加载顺序分配数字调用后缀，例如 `/review:1` 和 `/review:2`。

```typescript
pi.registerCommand("stats", {
  description: "Show session statistics",
  handler: async (args, ctx) => {
    const count = ctx.sessionManager.getEntries().length;
    ctx.ui.notify(`${count} entries`, "info");
  }
});
```

可选：为 `/command ...` 添加参数自动补全：

```typescript
import type { AutocompleteItem } from "@earendil-works/pi-tui";

pi.registerCommand("deploy", {
  description: "Deploy to an environment",
  getArgumentCompletions: (prefix: string): AutocompleteItem[] | null => {
    const envs = ["dev", "staging", "prod"];
    const items = envs.map((e) => ({ value: e, label: e }));
    const filtered = items.filter((i) => i.value.startsWith(prefix));
    return filtered.length > 0 ? filtered : null;
  },
  handler: async (args, ctx) => {
    ctx.ui.notify(`Deploying: ${args}`, "info");
  },
});
```

### pi.getCommands()

获取当前会话中可通过 `prompt` 调用的斜杠命令，包括扩展命令、提示词模板和技能命令。
列表顺序与 RPC `get_commands` 一致：先扩展，再模板，最后技能。

```typescript
const commands = pi.getCommands();
const bySource = commands.filter((command) => command.source === "extension");
const userScoped = commands.filter((command) => command.sourceInfo.scope === "user");
```

每个条目的结构如下：

```typescript
{
  name: string; // 可调用的命令名称，不含开头的斜杠；可能带有 "review:1" 之类的后缀
  description?: string;
  source: "extension" | "prompt" | "skill";
  sourceInfo: {
    path: string;
    source: string;
    scope: "user" | "project" | "temporary";
    origin: "package" | "top-level";
    baseDir?: string;
  };
}
```

请将 `sourceInfo` 作为规范的来源字段。不要根据命令名称或临时解析路径来推断归属。

这里不包含内置交互命令（例如 `/model` 和 `/settings`）。这些命令只在交互模式中处理，通过 `prompt` 发送时不会执行。

### pi.registerMessageRenderer(customType, renderer)

为指定 `customType` 的自定义消息注册 TUI 渲染器。自定义消息通过 `pi.sendMessage()` 创建，并会进入 LLM 上下文。参见[自定义 UI](#custom-ui)。

### pi.registerEntryRenderer(customType, renderer)

为指定 `customType` 的自定义条目注册 TUI 渲染器。自定义条目通过 `pi.appendEntry()` 创建，不会进入 LLM 上下文。

```typescript
import { Box, Text } from "@earendil-works/pi-tui";

pi.registerEntryRenderer("status-card", (entry, { expanded }, theme) => {
  const data = entry.data as { title: string; count: number };
  const box = new Box(1, 1, (text) => theme.bg("customMessageBg", text));
  box.addChild(new Text(`${theme.bold(data.title)}: ${data.count}`));
  if (expanded) {
    box.addChild(new Text(theme.fg("dim", JSON.stringify(data, null, 2))));
  }
  return box;
});

pi.appendEntry("status-card", { title: "Indexed files", count: 17 });
```

### pi.registerShortcut(shortcut, options)

注册键盘快捷键。快捷键格式和内置按键绑定请参阅 [keybindings.md](keybindings-zh.md)。

```typescript
pi.registerShortcut("ctrl+shift+p", {
  description: "Toggle plan mode",
  handler: async (ctx) => {
    ctx.ui.notify("Toggled!");
  },
});
```

### pi.registerFlag(name, options)

注册 CLI 参数。

```typescript
pi.registerFlag("plan", {
  description: "Start in plan mode",
  type: "boolean",
  default: false,
});

// 检查值
if (pi.getFlag("plan")) {
  // 计划模式已启用
}
```

### pi.exec(command, args, options?)

执行 Shell 命令。

```typescript
const result = await pi.exec("git", ["status"], { signal, timeout: 5000 });
// result.stdout, result.stderr, result.code, result.killed
```

### pi.getActiveTools() / pi.getAllTools() / pi.setActiveTools(names)

管理活动工具。内置工具和动态注册的工具都支持这些方法。`pi.getActiveTools()` 以 `string[]` 返回活动工具名称；`pi.getAllTools()` 返回所有已配置工具的元数据。

```typescript
const active = pi.getActiveTools(); // ["read", "bash", ...]
const all = pi.getAllTools();
// all = [{
//   name: "read",
//   description: "Read file contents...",
//   parameters: ...,
//   promptGuidelines: ["Use read to examine files instead of cat or sed."],
//   sourceInfo: { path: "<builtin:read>", source: "builtin", scope: "temporary", origin: "top-level" }
// }, ...]
const builtinTools = all.filter((t) => t.sourceInfo.source === "builtin");
const extensionTools = all.filter((t) => t.sourceInfo.source !== "builtin" && t.sourceInfo.source !== "sdk");
pi.setActiveTools([...new Set([...active, "my_custom_tool"])]); // 保留当前工具并启用 my_custom_tool
pi.setActiveTools(["read", "bash"]); // 切换为只读
```

`pi.getAllTools()` 返回 `name`、`description`、`parameters`、`promptGuidelines` 和 `sourceInfo`。

常见的 `sourceInfo.source` 值：

- 内置工具为 `builtin`
- 通过 `createAgentSession({ customTools })` 传入的工具为 `sdk`
- 扩展注册的工具使用扩展来源元数据

### pi.setModel(model)

设置当前模型。如果该模型没有可用的 API 密钥，则返回 `false`。自定义模型配置请参阅 [models.md](models-zh.md)。

```typescript
const model = ctx.modelRegistry.find("anthropic", "claude-sonnet-4-5");
if (model) {
  const success = await pi.setModel(model);
  if (!success) {
    ctx.ui.notify("No API key for this model", "error");
  }
}
```

### pi.getThinkingLevel() / pi.setThinkingLevel(level)

获取或设置思考级别。该级别会受模型能力限制；不支持推理的模型始终使用 `"off"`。发生变化时会触发 `thinking_level_select`。

```typescript
const current = pi.getThinkingLevel();  // "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max"
pi.setThinkingLevel("high");
```

### pi.events

用于扩展之间通信的共享事件总线：

```typescript
pi.events.on("my:event", (data) => { ... });
pi.events.emit("my:event", { ... });
```

### pi.registerProvider(name, config)

动态注册或覆盖模型服务提供商。适用于代理、自定义端点或团队统一的模型配置。

在扩展工厂函数中进行的调用会进入队列，并在运行器初始化后应用。在此之后进行的调用，例如用户配置流程完成后从命令处理函数中调用，会立即生效，无需 `/reload`。

动态服务提供商可以实现 `refreshModels`。Pi 会在刷新模型时调用它，通过服务提供商同步发布返回的列表，并传入规范的凭据、存储、网络和信号上下文。扩展可以自行决定是否通过 `context.store` 持久化模型目录；llama.cpp 之类的实时服务器可以忽略该存储。

如果扩展需要原生的服务提供商认证、筛选、刷新或流式传输行为，可以注册来自 `@earendil-works/pi-ai` 的完整 `Provider`。该服务提供商会成为组合基础，`models.json` 中的覆盖配置仍会叠加到其上。

```typescript
import { createProvider, openAICompletionsApi } from "@earendil-works/pi-ai";

const provider = createProvider({
  id: "local-server",
  name: "Local Server",
  baseUrl: "http://localhost:8080/v1",
  auth: {
    apiKey: {
      name: "Local server setup",
      async login(interaction) {
        return {
          type: "api_key",
          key: await interaction.prompt({ type: "secret", message: "API key" }),
        };
      },
      async resolve({ credential }) {
        return credential?.key
          ? { auth: { apiKey: credential.key }, source: "stored API key" }
          : undefined;
      },
    },
  },
  models: [],
  api: openAICompletionsApi(),
});

pi.registerProvider(provider);

// 注册包含自定义模型的新服务提供商
pi.registerProvider("my-proxy", {
  name: "My Proxy",
  baseUrl: "https://proxy.example.com",
  apiKey: "$PROXY_API_KEY",  // 引用环境变量
  api: "anthropic-messages",
  models: [
    {
      id: "claude-sonnet-4-20250514",
      name: "Claude 4 Sonnet (proxy)",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});

// 注册实时 llama.cpp 模型目录，但不持久化发现的模型
pi.registerProvider("llama.cpp", {
  baseUrl: "http://localhost:8080/v1",
  apiKey: "local",
  api: "openai-completions",
  async refreshModels({ signal }) {
    const response = await fetch("http://localhost:8080/v1/models", { signal });
    const { data } = await response.json();
    return data.map(({ id }) => ({
      id,
      name: id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 128000,
      maxTokens: 16384
    }));
  }
});

// 覆盖现有服务提供商的 baseUrl（保留所有模型）
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com"
});

// 注册支持 /login OAuth 认证的服务提供商
pi.registerProvider("corporate-ai", {
  baseUrl: "https://ai.corp.com",
  api: "openai-responses",
  models: [...],
  oauth: {
    name: "Corporate AI (SSO)",
    async login(callbacks) {
      // 自定义 OAuth 流程
      callbacks.onAuth({ url: "https://sso.corp.com/..." });
      const code = await callbacks.onPrompt({ message: "Enter code:" });
      return { refresh: code, access: code, expires: Date.now() + 3600000 };
    },
    async refreshToken(credentials) {
      // 刷新逻辑
      return credentials;
    },
    getApiKey(credentials) {
      return credentials.access;
    }
  }
});
```

对象形式接受完整的 pi-ai `Provider`，包括原生的 `auth`、`getModels`、`refreshModels`、`filterModels`、`stream` 和 `streamSimple` 行为。

**旧版配置选项：**

- `name`——服务提供商在 `/login` 等界面中显示的名称。
- `baseUrl`——API 端点 URL。定义模型时必填。
- `apiKey`——API 密钥字面值、环境变量插值（`$ENV_VAR` 或 `${ENV_VAR}`），或以 `!command` 开头的命令。定义模型时必填（提供 `oauth` 时除外）。`$$` 用于转义 `$`，`$!` 用于转义字面量 `!` 且不会触发命令执行。
- `api`——API 类型：`"anthropic-messages"`、`"openai-completions"`、`"openai-responses"` 等。
- `headers`——请求中包含的自定义请求头。
- `authHeader`——如果为 `true`，自动添加 `Authorization: Bearer` 请求头。
- `models`——模型定义数组。提供后会替换该服务提供商的所有现有模型。模型定义可以设置 `baseUrl`，为该模型覆盖服务提供商端点。
- `refreshModels`——异步动态发现回调。它返回的模型会替换扩展提供的模型。只有结果需要持久化时，才使用作用域内的 `context.store`。
- `oauth`——为 `/login` 提供支持的 OAuth 服务提供商配置。提供后，该服务提供商会出现在登录菜单中。
- `streamSimple`——非标准 API 的自定义流式传输实现。

自定义流式 API、OAuth 细节和模型定义参考等进阶主题，请参阅 [custom-provider.md](custom-provider-zh.md)。

### pi.unregisterProvider(name)

移除此前注册的服务提供商及其模型。被该服务提供商覆盖的内置模型会恢复。如果该服务提供商未注册，则不产生任何影响。

与 `registerProvider` 相同，在初次加载阶段之后调用时会立即生效，因此无需 `/reload`。

```typescript
pi.registerCommand("my-setup-teardown", {
  description: "Remove the custom proxy provider",
  handler: async (_args, _ctx) => {
    pi.unregisterProvider("my-proxy");
  },
});
```

<a id="state-management"></a>

## 状态管理

有状态扩展应将状态存储在工具结果的 `details` 中，以正确支持创建分支：

```typescript
export default function (pi: ExtensionAPI) {
  let items: string[] = [];

  // 根据会话重建状态
  pi.on("session_start", async (_event, ctx) => {
    items = [];
    for (const entry of ctx.sessionManager.getBranch()) {
      if (entry.type === "message" && entry.message.role === "toolResult") {
        if (entry.message.toolName === "my_tool") {
          items = entry.message.details?.items ?? [];
        }
      }
    }
  });

  pi.registerTool({
    name: "my_tool",
    // ...
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      items.push("new item");
      return {
        content: [{ type: "text", text: "Added" }],
        details: { items: [...items] },  // 存储起来供重建使用
      };
    },
  });
}
```

<a id="custom-tools"></a>

## 自定义工具

通过 `pi.registerTool()` 注册供 LLM 调用的工具。工具会出现在系统提示词中，并且可以使用自定义渲染。

使用 `promptSnippet` 可以在默认系统提示词的 `Available tools` 部分添加简短的单行条目。省略时，该部分不会列出自定义工具。

使用 `promptGuidelines` 可以在默认系统提示词的 `Guidelines` 部分添加工具特有的规则条目。只有工具处于活动状态时才会包含这些条目，例如调用 `pi.setActiveTools([...])` 后。

**重要：** `promptGuidelines` 条目会平铺追加到 `Guidelines` 部分，不带工具名称前缀，也不会按工具分组。每条规则必须明确写出所指工具的名称。不要写“Use this tool when...”，因为 LLM 无法判断“this”指哪个工具；应写成“Use my_tool when...”。

注意：部分模型会错误地在工具路径参数中保留 `@` 前缀。内置工具会在解析路径前移除开头的 `@`。如果自定义工具接受路径，也应规范化开头的 `@`。

如果自定义工具会修改文件，请使用 `withFileMutationQueue()`，使它与内置 `edit` 和 `write` 共用同一套按文件划分的队列。工具调用默认并行运行，因此这点很重要。如果不进入队列，两个工具可能读取同一份旧文件内容、分别计算不同更新，最后后写入的一方覆盖另一方。

典型故障场景：自定义工具编辑 `foo.ts`，而同一助手轮次中的内置 `edit` 也修改 `foo.ts`。如果自定义工具不进入队列，两者都可能读取原始 `foo.ts` 并分别应用更改，最终丢失其中一项更改。

传给 `withFileMutationQueue()` 的应是真实目标文件路径，而不是原始用户参数。请先基于 `ctx.cwd` 或工具工作目录将其解析为绝对路径。对于现有文件，辅助函数会通过 `realpath()` 取得规范路径，所以指向同一文件的符号链接别名会共用一个队列。对于新文件，由于尚无内容可供 `realpath()` 解析，会回退到解析后的绝对路径。

请将该目标路径上的整个修改时间窗口都放入队列，包括“读取—修改—写入”的完整逻辑，而不只是最后的写入步骤。

```typescript
import { withFileMutationQueue } from "@earendil-works/pi-coding-agent";
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { dirname, resolve } from "node:path";

async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
  const absolutePath = resolve(ctx.cwd, params.path);

  return withFileMutationQueue(absolutePath, async () => {
    await mkdir(dirname(absolutePath), { recursive: true });
    const current = await readFile(absolutePath, "utf8");
    const next = current.replace(params.oldText, params.newText);
    await writeFile(absolutePath, next, "utf8");

    return {
      content: [{ type: "text", text: `Updated ${params.path}` }],
      details: {},
    };
  });
}
```

### 工具定义

```typescript
import { Type } from "typebox";
import { StringEnum } from "@earendil-works/pi-ai";
import { Text } from "@earendil-works/pi-tui";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "What this tool does (shown to LLM)",
  promptSnippet: "List or add items in the project todo list",
  promptGuidelines: [
    "Use my_tool for todo planning instead of direct file edits when the user asks for a task list."
  ],
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),  // 使用 StringEnum 以兼容 Google
    text: Type.Optional(Type.String()),
  }),
  prepareArguments(args) {
    if (!args || typeof args !== "object") return args;
    const input = args as { action?: string; oldAction?: string };
    if (typeof input.oldAction === "string" && input.action === undefined) {
      return { ...input, action: input.oldAction };
    }
    return args;
  },

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // 检查是否已取消
    if (signal?.aborted) {
      return { content: [{ type: "text", text: "Cancelled" }] };
    }

    // 流式报告进度
    onUpdate?.({
      content: [{ type: "text", text: "Working..." }],
      details: { progress: 50 },
    });

    // 通过从扩展闭包捕获的 pi.exec 运行命令
    const result = await pi.exec("some-command", [], { signal });

    // 返回结果
    return {
      content: [{ type: "text", text: "Done" }],  // 发送给 LLM
      details: { data: result },                   // 用于渲染和状态
      // usage: nestedModelResponse.usage,          // 可选的嵌套 LLM 用量
      // 可选：如果本批次中每个最终工具结果都返回 terminate: true，
      // 则在该工具批次后停止。
      terminate: true,
    };
  },

  // 可选：自定义渲染
  renderCall(args, theme, context) { ... },
  renderResult(result, options, theme, context) { ... },
});
```

**用量计算：** 如果工具在内部调用了 LLM，请将这些调用合并后的 `Usage` 作为 `usage` 返回。Pi 会将其持久化在工具结果中，并计入页脚、`/session` 和 RPC 会话总计。`tool_result` 处理函数可以检查或替换该值。

**报告错误：** 要将工具执行标记为失败，即在结果上设置 `isError: true` 并报告给 LLM，请从 `execute` 中抛出错误。无论返回对象中包含哪些属性，只要正常返回值，就不会设置错误标记。

**提前终止：** 从 `execute()` 返回 `terminate: true`，可以提示系统在当前工具批次后跳过自动的后续 LLM 调用。只有当该批次中每个最终工具结果都要求终止时，此设置才会生效。以最终结构化输出工具调用结束智能体的最小示例，请参阅 [examples/extensions/structured-output.ts](../examples/extensions/structured-output.ts)。

```typescript
// 正确：通过抛出异常报告错误
async execute(toolCallId, params) {
  if (!isValid(params.input)) {
    throw new Error(`Invalid input: ${params.input}`);
  }
  return { content: [{ type: "text", text: "OK" }], details: {} };
}
```

**重要：** 字符串枚举请使用 `@earendil-works/pi-ai` 中的 `StringEnum`。`Type.Union`/`Type.Literal` 无法与 Google API 配合使用。

**参数准备：** `prepareArguments(args)` 可选。定义后，它会在 Schema 验证和 `execute()` 之前运行。当 pi 恢复旧会话，而其中存储的工具调用参数不再符合当前 Schema 时，可以用它模拟旧版可接受的输入结构。请返回希望由 `parameters` 验证的对象。对外公开的 Schema 应保持严格，不要仅为了让恢复的旧会话继续工作，就向 `parameters` 添加已弃用的兼容字段。

例如，旧会话可能包含顶层带有 `oldText` 和 `newText` 的 `edit` 工具调用，而当前 Schema 只接受 `edits: [{ oldText, newText }]`。

```typescript
pi.registerTool({
  name: "edit",
  label: "Edit",
  description: "Edit a single file using exact text replacement",
  parameters: Type.Object({
    path: Type.String(),
    edits: Type.Array(
      Type.Object({
        oldText: Type.String(),
        newText: Type.String(),
      }),
    ),
  }),
  prepareArguments(args) {
    if (!args || typeof args !== "object") return args;

    const input = args as {
      path?: string;
      edits?: Array<{ oldText: string; newText: string }>;
      oldText?: unknown;
      newText?: unknown;
    };

    if (typeof input.oldText !== "string" || typeof input.newText !== "string") {
      return args;
    }

    return {
      ...input,
      edits: [...(input.edits ?? []), { oldText: input.oldText, newText: input.newText }],
    };
  },
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // params 现在符合当前 Schema
    return {
      content: [{ type: "text", text: `Applying ${params.edits.length} edit block(s)` }],
      details: {},
    };
  },
});
```

### 覆盖内置工具

扩展可以注册同名工具，覆盖内置工具（`read`、`bash`、`edit`、`write`、`grep`、`find`、`ls`）。发生覆盖时，交互模式会显示警告。

```bash
# 扩展的 read 工具替换内置 read
pi -e ./tool-override.ts
```

也可以使用 `--no-builtin-tools`，在不加载任何内置工具的情况下启动，同时保持扩展工具启用：
```bash
# 不加载内置工具，只加载扩展工具
pi --no-builtin-tools -e ./my-extension.ts
```

通过日志记录和访问控制覆盖 `read` 的完整示例，请参阅 [examples/extensions/tool-override.ts](../examples/extensions/tool-override.ts)。

**渲染：** 内置渲染器按渲染位置分别继承。执行覆盖与渲染覆盖彼此独立。如果覆盖工具省略 `renderCall`，则使用内置 `renderCall`；如果省略 `renderResult`，则使用内置 `renderResult`；如果两者都省略，则自动使用完整的内置渲染器，包括语法高亮、差异显示等。这样可以包装内置工具以实现日志记录或访问控制，而无需重新实现 UI。

**提示词元数据：** `promptSnippet` 和 `promptGuidelines` 不会从内置工具继承。如果覆盖工具需要保留这些提示词指令，请在覆盖定义中显式提供。

**实现必须与结果的准确结构一致**，包括 `details` 类型。UI 和会话逻辑依赖这些结构进行渲染和状态跟踪。

内置工具实现：
- [read.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/tools/read.ts) - `ReadToolDetails`
- [bash.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/tools/bash.ts) - `BashToolDetails`
- [edit.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/tools/edit.ts)
- [write.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/tools/write.ts)
- [grep.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/tools/grep.ts) - `GrepToolDetails`
- [find.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/tools/find.ts) - `FindToolDetails`
- [ls.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/tools/ls.ts) - `LsToolDetails`

### 远程执行

内置工具支持可插拔操作，可将执行委托给远程系统（SSH、容器等）：

```typescript
import { createReadTool, createBashTool, type ReadOperations } from "@earendil-works/pi-coding-agent";

// 使用自定义操作创建工具
const remoteRead = createReadTool(cwd, {
  operations: {
    readFile: (path) => sshExec(remote, `cat ${path}`),
    access: (path) => sshExec(remote, `test -r ${path}`).then(() => {}),
  }
});

// 注册工具，并在执行时检查参数
pi.registerTool({
  ...remoteRead,
  async execute(id, params, signal, onUpdate, _ctx) {
    const ssh = getSshConfig();
    if (ssh) {
      const tool = createReadTool(cwd, { operations: createRemoteOps(ssh) });
      return tool.execute(id, params, signal, onUpdate);
    }
    return localRead.execute(id, params, signal, onUpdate);
  },
});
```

**操作接口：** `ReadOperations`、`WriteOperations`、`EditOperations`、`BashOperations`、`LsOperations`、`GrepOperations`、`FindOperations`

对于 `user_bash`，扩展可以通过 `createLocalBashOperations()` 复用 pi 的本地 Shell 后端，无需重新实现本地进程创建、Shell 解析和进程树终止。

Bash 工具还支持进程创建钩子，可在执行前调整命令、`cwd` 或环境变量：

```typescript
import { createBashTool } from "@earendil-works/pi-coding-agent";

const bashTool = createBashTool(cwd, {
  spawnHook: ({ command, cwd, env }) => ({
    command: `source ~/.profile\n${command}`,
    cwd: `/mnt/sandbox${cwd}`,
    env: { ...env, CI: "1" },
  }),
});
```

`createBashTool()` 通过 `PI_SESSION_ID`、`PI_SESSION_FILE`、`PI_PROVIDER`、`PI_MODEL` 和 `PI_REASONING_LEVEL` 向命令暴露当前会话。注入发生在 `spawnHook` 之前，因此钩子可以从 `env` 获取这些值；如上所示展开现有环境时，也会保留它们。设置 `exposeSessionEnvironment: false` 可以禁用：

```typescript
const bashTool = createBashTool(cwd, {
  exposeSessionEnvironment: false,
});
```

变量语义请参阅 [Bash 工具的会话环境](environment-variables-zh.md#bash-tool-session-environment)。带 `--ssh` 参数的完整 SSH 示例请参阅 [examples/extensions/ssh.ts](../examples/extensions/ssh.ts)。

### 输出截断

**工具必须截断输出**，避免占满 LLM 上下文。输出过大可能导致：

- 上下文溢出错误（提示词过长）
- 上下文压缩失败
- 模型性能下降

内置限制为 **50 KB**（约 1 万 Token）和 **2000 行**，以先达到的限制为准。请使用导出的截断实用函数：

```typescript
import {
  truncateHead,      // 保留开头 N 行/字节（适合读取文件、搜索结果）
  truncateTail,      // 保留末尾 N 行/字节（适合日志、命令输出）
  truncateLine,      // 将单行截断到 maxBytes，并添加省略号
  formatSize,        // 便于阅读的大小（例如 "50KB"、"1.5MB"）
  DEFAULT_MAX_BYTES, // 50KB
  DEFAULT_MAX_LINES, // 2000
} from "@earendil-works/pi-coding-agent";

async execute(toolCallId, params, signal, onUpdate, ctx) {
  const output = await runCommand();

  // 应用截断
  const truncation = truncateHead(output, {
    maxLines: DEFAULT_MAX_LINES,
    maxBytes: DEFAULT_MAX_BYTES,
  });

  let result = truncation.content;

  if (truncation.truncated) {
    // 将完整输出写入临时文件
    const tempFile = writeTempFile(output);

    // 告知 LLM 在哪里查找完整输出
    result += `\n\n[Output truncated: ${truncation.outputLines} of ${truncation.totalLines} lines`;
    result += ` (${formatSize(truncation.outputBytes)} of ${formatSize(truncation.totalBytes)}).`;
    result += ` Full output saved to: ${tempFile}]`;
  }

  return { content: [{ type: "text", text: result }] };
}
```

**要点：**

- 对开头内容重要的输出使用 `truncateHead`，例如搜索结果、文件读取内容
- 对末尾内容重要的输出使用 `truncateTail`，例如日志、命令输出
- 输出被截断时，始终告知 LLM，并说明在哪里查找完整版
- 在工具说明中写明截断限制

正确截断 `rg`（ripgrep）输出的完整包装示例，请参阅 [examples/extensions/truncated-tool.ts](../examples/extensions/truncated-tool.ts)。

### 多个工具

一个扩展可以注册多个共享状态的工具：

```typescript
export default function (pi: ExtensionAPI) {
  let connection = null;

  pi.registerTool({ name: "db_connect", ... });
  pi.registerTool({ name: "db_query", ... });
  pi.registerTool({ name: "db_close", ... });

  pi.on("session_shutdown", async () => {
    connection?.close();
  });
}
```

### 自定义渲染

工具可以提供 `renderCall` 和 `renderResult`，自定义 TUI 显示。完整组件 API 请参阅 [tui.md](tui-zh.md)；工具行的组合方式请参阅 [tool-execution.ts](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/modes/interactive/components/tool-execution.ts)。

默认情况下，工具输出包装在负责内边距和背景的 `Box` 中。定义的 `renderCall` 或 `renderResult` 必须返回 `Component`。如果某个渲染位置没有定义渲染器，`tool-execution.ts` 会为该位置使用回退渲染。

如果工具应自行渲染外壳，而不是使用默认 `Box`，请设置 `renderShell: "self"`。这适用于需要完全控制边框或背景行为的工具，例如工具结束后仍需保持视觉稳定的大型预览。

```typescript
pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "Custom shell example",
  parameters: Type.Object({}),
  renderShell: "self",
  async execute() {
    return { content: [{ type: "text", text: "ok" }], details: undefined };
  },
  renderCall(args, theme, context) {
    return new Text(theme.fg("accent", "my custom shell"), 0, 0);
  },
});
```

`renderCall` 和 `renderResult` 都会收到一个 `context` 对象，其中包含：

- `args`——当前工具调用参数
- `state`——`renderCall` 和 `renderResult` 之间共享的工具行局部状态
- `lastComponent`——该渲染位置此前返回的组件（如有）
- `invalidate()`——请求重新渲染该工具行
- `toolCallId`, `cwd`, `executionStarted`, `argsComplete`, `isPartial`, `expanded`, `showImages`, `isError`

使用 `context.state` 保存跨渲染位置共享的状态。如果希望跨多次渲染复用并修改同一组件，请将该位置的局部缓存保存在返回的组件实例上。

#### renderCall

渲染工具调用或标题：

```typescript
import { Text } from "@earendil-works/pi-tui";

renderCall(args, theme, context) {
  const text = (context.lastComponent as Text | undefined) ?? new Text("", 0, 0);
  let content = theme.fg("toolTitle", theme.bold("my_tool "));
  content += theme.fg("muted", args.action);
  if (args.text) {
    content += " " + theme.fg("dim", `"${args.text}"`);
  }
  text.setText(content);
  return text;
}
```

#### renderResult

渲染工具结果或输出：

```typescript
renderResult(result, { expanded, isPartial }, theme, context) {
  if (isPartial) {
    return new Text(theme.fg("warning", "Processing..."), 0, 0);
  }

  if (result.details?.error) {
    return new Text(theme.fg("error", `Error: ${result.details.error}`), 0, 0);
  }

  let text = theme.fg("success", "✓ Done");
  if (expanded && result.details?.items) {
    for (const item of result.details.items) {
      text += "\n  " + theme.fg("dim", item);
    }
  }
  return new Text(text, 0, 0);
}
```

如果某个渲染位置刻意不显示任何内容，请返回空的 `Component`，例如空的 `Container`。

#### 按键绑定提示

使用 `keyHint()` 显示遵循当前按键绑定配置的操作提示：

```typescript
import { keyHint } from "@earendil-works/pi-coding-agent";

renderResult(result, { expanded }, theme, context) {
  let text = theme.fg("success", "✓ Done");
  if (!expanded) {
    text += ` (${keyHint("app.tools.expand", "to expand")})`;
  }
  return new Text(text, 0, 0);
}
```

可用函数：

- `keyHint(keybinding, description)`——格式化已配置的按键绑定 ID，例如 `"app.tools.expand"` 或 `"tui.select.confirm"`
- `keyText(keybinding)`——返回某个按键绑定 ID 配置的原始按键文本
- `rawKeyHint(key, description)`——格式化原始按键字符串

请使用带命名空间的按键绑定 ID：

- 编程智能体的 ID 使用 `app.*` 命名空间，例如 `app.tools.expand`、`app.editor.external`、`app.session.rename`
- 共享 TUI 的 ID 使用 `tui.*` 命名空间，例如 `tui.select.confirm`、`tui.select.cancel`、`tui.input.tab`

按键绑定 ID 和默认值的完整列表请参阅 [keybindings.md](keybindings-zh.md)。`keybindings.json` 使用相同的命名空间 ID。

自定义编辑器和 `ctx.ui.custom()` 组件会收到注入的 `keybindings: KeybindingsManager` 参数。应直接使用该管理器，不要调用 `getKeybindings()` 或 `setKeybindings()`。

#### 最佳实践

- 使用内边距为 `(0, 0)` 的 `Text`；默认 `Box` 会处理内边距。
- 使用 `\n` 表示多行内容。
- 处理 `isPartial`，显示流式进度。
- 支持 `expanded`，按需显示详情。
- 保持默认视图紧凑。
- 在 `renderResult` 中读取 `context.args`，不要将参数复制到 `context.state`。
- 只将必须在调用和结果两个渲染位置之间共享的数据放入 `context.state`。
- 如果同一组件实例可以原位更新，请复用 `context.lastComponent`。
- 只有在默认带框外壳妨碍渲染时才使用 `renderShell: "self"`。在自行渲染外壳模式下，工具需要负责自己的边框、内边距和背景。

#### 回退行为

如果某个位置的渲染器未定义或抛出错误：

- `renderCall`：显示工具名称
- `renderResult`：显示 `content` 中的原始文本

<a id="dynamic-tool-loading"></a>

### 动态加载工具

扩展可以注册许多工具，但最初只启用少量工具。之后，某个工具可以在执行期间使用 `pi.setActiveTools()` 添加更多工具。Pi 会检测纯新增式变化，将新可用的工具名称记录在该工具结果上，并在下一次模型请求前应用更新后的活动工具集合。

所有模型都支持这种机制。原生支持延迟加载的模型会保留稳定的提示词前缀，并在工具结果所在位置加载新定义。其他模型使用下文所述的回退行为。

生命周期如下：

1. 使用 `pi.registerTool()` 注册所有工具，使它们出现在 `pi.getAllTools()` 中。
2. 保持 `search_tools` 等加载器工具为活动状态，让可搜索工具保持非活动状态。
3. 加载器执行期间调用 `pi.setActiveTools([...currentTools, ...matchingTools])`。变化必须是新增式的；同一次调用中不要移除当前活动工具。
4. Pi 将新增工具记录在加载器的工具结果上。
5. 下一次模型响应前，如果支持原生延迟加载，Pi 会通过该机制暴露新增定义；否则通过普通活动工具列表暴露。

无需返回服务提供商特有的工具引用，也无需将加载器标记为特殊搜索工具。活动工具集合发生变化本身就是信号。传给 `pi.setActiveTools()` 的名称必须已经注册，未知名称会被忽略。

#### 原生支持延迟加载的模型

- **Anthropic**
  - **模型：** 4.5 或更高版本的 Sonnet、Opus、Fable（不包括 Haiku）
  - **原生表示：** 延迟定义使用 `defer_loading`；加载位置使用 `tool_reference` 内容。
- **OpenAI**
  - **模型：** `gpt-5.4` 及更新系列
  - **原生表示：** Pi 会在加载位置添加已完成的客户端 `tool_search_call` 和 `tool_search_output` 条目。

对于已经验证的自定义模型或代理，`anthropic-messages` 可通过 `compat.supportsToolReferences: true` 启用原生处理；`openai-responses` 和 `openai-codex-responses` 可通过 `compat.supportsToolSearch: true` 启用。除非端点和模型确实接受相应的原生协议，否则请保持禁用。

#### 回退行为

对于其他所有模型和服务提供商，动态启用仍然有效：Pi 会在下一次请求中正常发送当前完整的活动工具列表。模型可以调用新启用的工具，但添加其定义可能使服务提供商缓存的提示词前缀失效。

当活动工具集合并非纯新增式变化时，例如用一组工具替换另一组工具，Pi 也会使用这种安全回退。因此可以移除工具，但移除操作不会使用延迟加载。

为了获得最佳缓存效果，请在整个会话期间保持加载器工具活动，并新增工具，而不是替换活动集合。还要注意，启用带有 `promptSnippet` 或 `promptGuidelines` 的工具会重建系统提示词；即使服务提供商支持延迟 Schema，这种系统提示词变化仍可能使前缀失效。延迟加载的工具通常应依赖其工具 `description`，并省略仅在活动时加入的提示词元数据。

#### 搜索工具示例

以下扩展注册两个可搜索工具，将它们从初始活动集合中移除，并只保留 `search_tools` 作为加载器。示例使用简单的关键词匹配，但搜索实现也可以使用 BM25、嵌入向量、远程目录或项目特有的路由。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

const SEARCHABLE_TOOL_NAMES = new Set(["lookup_weather", "search_issues"]);

export default function (pi: ExtensionAPI) {
  pi.registerTool({
    name: "lookup_weather",
    label: "Lookup Weather",
    description: "Look up the current weather for a city",
    parameters: Type.Object({ city: Type.String() }),
    async execute(_toolCallId, params) {
      return {
        content: [{ type: "text", text: `Weather for ${params.city}: sunny` }],
        details: {},
      };
    },
  });

  pi.registerTool({
    name: "search_issues",
    label: "Search Issues",
    description: "Search project issues by keyword",
    parameters: Type.Object({ query: Type.String() }),
    async execute(_toolCallId, params) {
      return {
        content: [{ type: "text", text: `No open issues matching ${params.query}` }],
        details: {},
      };
    },
  });

  pi.registerTool({
    name: "search_tools",
    label: "Search Tools",
    description: "Search for and enable tools relevant to a task",
    promptSnippet: "Search for additional tools when the active tools cannot perform the task",
    promptGuidelines: [
      "Use search_tools when a task requires a capability that is not currently available.",
    ],
    parameters: Type.Object({
      query: Type.String({ description: "Capability or task to search for" }),
      limit: Type.Optional(Type.Integer({ minimum: 1, maximum: 10 })),
    }),
    async execute(_toolCallId, params) {
      const terms = params.query.toLowerCase().split(/[^a-z0-9]+/).filter(Boolean);
      const matches = pi.getAllTools()
        .filter((tool) => SEARCHABLE_TOOL_NAMES.has(tool.name))
        .map((tool) => ({
          tool,
          score: terms.reduce(
            (score, term) =>
              score + (`${tool.name} ${tool.description}`.toLowerCase().includes(term) ? 1 : 0),
            0,
          ),
        }))
        .filter((match) => match.score > 0)
        .sort((a, b) => b.score - a.score)
        .slice(0, params.limit ?? 3)
        .map((match) => match.tool.name);

      if (matches.length === 0) {
        return {
          content: [{ type: "text", text: `No tools found for: ${params.query}` }],
          details: { matches: [] },
        };
      }

      const active = pi.getActiveTools();
      const added = matches.filter((name) => !active.includes(name));
      pi.setActiveTools([...new Set([...active, ...added])]);

      return {
        content: [{
          type: "text",
          text: added.length > 0
            ? `Loaded tools: ${added.join(", ")}`
            : `Matching tools already active: ${matches.join(", ")}`,
        }],
        details: { matches, added },
      };
    },
  });

  pi.on("session_start", () => {
    // 保持可搜索工具已注册但初始不活动。保留内置工具
    // 和其他扩展拥有的工具，同时保持加载器本身活动。
    const initialTools = pi.getActiveTools().filter(
      (name) => !SEARCHABLE_TOOL_NAMES.has(name),
    );
    pi.setActiveTools([...new Set([...initialTools, "search_tools"])]);
  });
}
```

`search_tools` 添加匹配项后，模型会在紧接着的请求中收到该定义。对于原生支持的模型，定义会锚定在搜索结果后方，不改变初始工具 Schema 前缀；对于其他模型，它会在同一个后续请求的普通工具列表中出现。

<a id="custom-ui"></a>

## 自定义 UI

扩展可以通过 `ctx.ui` 方法与用户交互，并自定义消息和工具的渲染方式。

**自定义组件请参阅 [tui.md](tui-zh.md)**，其中提供了可以直接复用的以下模式：

- 选择对话框（SelectList）
- 可取消的异步操作（BorderedLoader）
- 设置开关（SettingsList）
- 状态指示器（setStatus）
- 流式响应期间的工作消息、可见性和指示器（`setWorkingMessage`、`setWorkingVisible`、`setWorkingIndicator`）
- 编辑器上方/下方的小组件（setWidget）
- 叠加在内置斜杠命令/路径补全之上的自动补全提供程序（addAutocompleteProvider）
- 自定义页脚（setFooter）

### 对话框

```typescript
// 从选项中选择
const choice = await ctx.ui.select("Pick one:", ["A", "B", "C"]);

// 确认对话框
const ok = await ctx.ui.confirm("Delete?", "This cannot be undone");

// 文本输入
const name = await ctx.ui.input("Name:", "placeholder");

// 多行编辑器
const text = await ctx.ui.editor("Edit:", "prefilled text");

// 通知（不阻塞）
ctx.ui.notify("Done!", "info");  // "info" | "warning" | "error"
```

#### 带倒计时的定时对话框

对话框支持 `timeout` 选项，可以实时显示倒计时，并在超时后自动关闭：

```typescript
// 对话框显示“标题 (5s)”→“标题 (4s)”→……→倒计时到 0 时自动关闭
const confirmed = await ctx.ui.confirm(
  "Timed Confirmation",
  "This dialog will auto-cancel in 5 seconds. Confirm?",
  { timeout: 5000 }
);

if (confirmed) {
  // 用户已确认
} else {
  // 用户已取消或已超时
}
```

**超时后的返回值：**

- `select()` 返回 `undefined`
- `confirm()` 返回 `false`
- `input()` 返回 `undefined`

#### 使用 AbortSignal 手动关闭

如果需要更精细的控制，例如区分超时和用户取消，请使用 `AbortSignal`：

```typescript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

const confirmed = await ctx.ui.confirm(
  "Timed Confirmation",
  "This dialog will auto-cancel in 5 seconds. Confirm?",
  { signal: controller.signal }
);

clearTimeout(timeoutId);

if (confirmed) {
  // 用户已确认
} else if (controller.signal.aborted) {
  // 对话框已超时
} else {
  // 用户已取消（按 Escape 或选择“否”）
}
```

完整示例请参阅 [examples/extensions/timed-confirm.ts](../examples/extensions/timed-confirm.ts)。

### 小组件、状态与页脚

```typescript
// 页脚状态（清除前一直保留）
ctx.ui.setStatus("my-ext", "Processing...");
ctx.ui.setStatus("my-ext", undefined);  // 清除

// 工作加载器（流式响应期间显示）
ctx.ui.setWorkingMessage("Thinking deeply...");
ctx.ui.setWorkingMessage();  // 恢复默认值
ctx.ui.setWorkingVisible(false);  // 完全隐藏内置工作加载器行
ctx.ui.setWorkingVisible(true);   // 显示内置工作加载器行

// 工作状态指示器（流式响应期间显示）
ctx.ui.setWorkingIndicator({ frames: [ctx.ui.theme.fg("accent", "●")] });  // 静态圆点
ctx.ui.setWorkingIndicator({
  frames: [
    ctx.ui.theme.fg("dim", "·"),
    ctx.ui.theme.fg("muted", "•"),
    ctx.ui.theme.fg("accent", "●"),
    ctx.ui.theme.fg("muted", "•"),
  ],
  intervalMs: 120,
});
ctx.ui.setWorkingIndicator({ frames: [] });  // 隐藏指示器
ctx.ui.setWorkingIndicator();  // 恢复默认加载动画

// 编辑器上方的小组件（默认）
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"]);
// 编辑器下方的小组件
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"], { placement: "belowEditor" });
ctx.ui.setWidget("my-widget", (tui, theme) => new Text(theme.fg("accent", "Custom"), 0, 0));
ctx.ui.setWidget("my-widget", undefined);  // 清除

// 自定义页脚（完全替换内置页脚）
ctx.ui.setFooter((tui, theme) => ({
  render(width) { return [theme.fg("dim", "Custom footer")]; },
  invalidate() {},
}));
ctx.ui.setFooter(undefined);  // 恢复内置页脚

// 终端标题
ctx.ui.setTitle("pi - my-project");

// 编辑器文本
ctx.ui.setEditorText("Prefill text");
const current = ctx.ui.getEditorText();

// 粘贴到编辑器（触发粘贴处理，包括折叠大段内容）
ctx.ui.pasteToEditor("pasted content");

// 在内置提供程序之上叠加自定义自动补全行为
ctx.ui.addAutocompleteProvider((current) => ({
  triggerCharacters: ["#"],
  async getSuggestions(lines, line, col, options) {
    const beforeCursor = (lines[line] ?? "").slice(0, col);
    const match = beforeCursor.match(/(?:^|[ \t])#([^\s#]*)$/);
    if (!match) {
      return current.getSuggestions(lines, line, col, options);
    }

    return {
      prefix: `#${match[1] ?? ""}`,
      items: [{ value: "#2983", label: "#2983", description: "Extension API for autocomplete" }],
    };
  },
  applyCompletion(lines, line, col, item, prefix) {
    return current.applyCompletion(lines, line, col, item, prefix);
  },
  shouldTriggerFileCompletion(lines, line, col) {
    return current.shouldTriggerFileCompletion?.(lines, line, col) ?? true;
  },
}));

// 展开工具输出
const wasExpanded = ctx.ui.getToolsExpanded();
ctx.ui.setToolsExpanded(true);
ctx.ui.setToolsExpanded(wasExpanded);

// 自定义编辑器（Vim 模式、Emacs 模式等）
ctx.ui.setEditorComponent((tui, theme, keybindings) => new VimEditor(tui, theme, keybindings));
const currentEditor = ctx.ui.getEditorComponent();
ctx.ui.setEditorComponent((tui, theme, keybindings) =>
  new WrappedEditor(tui, theme, keybindings, currentEditor?.(tui, theme, keybindings))
);
ctx.ui.setEditorComponent(undefined);  // 恢复默认编辑器

// 主题管理（创建主题请参阅 themes.md）
const themes = ctx.ui.getAllThemes();  // [{ name: "dark", path: "/..." | undefined }, ...]
const lightTheme = ctx.ui.getTheme("light");  // 加载但不切换
const result = ctx.ui.setTheme("light");  // 按名称切换
if (!result.success) {
  ctx.ui.notify(`Failed: ${result.error}`, "error");
}
ctx.ui.setTheme(lightTheme!);  // 或通过 Theme 对象切换
ctx.ui.theme.fg("accent", "styled text");  // 访问当前主题
```

自定义工作状态指示器帧会原样渲染。如果需要颜色，请自行添加到帧字符串中，例如使用 `ctx.ui.theme.fg(...)`。

### 自动补全提供程序

使用 `ctx.ui.addAutocompleteProvider()` 可以在内置斜杠命令和路径补全提供程序之上叠加自定义自动补全逻辑。通过 `triggerCharacters` 设置 `$` 等自然触发字符。

典型模式：

- 检查光标前的文本
- 匹配扩展特有语法时返回自己的建议
- 否则委托给 `current.getSuggestions(...)`
- 除非需要自定义插入行为，否则将 `applyCompletion(...)` 委托给当前提供程序

```typescript
pi.on("session_start", (_event, ctx) => {
  ctx.ui.addAutocompleteProvider((current) => ({
    triggerCharacters: ["#"],
    async getSuggestions(lines, cursorLine, cursorCol, options) {
      const line = lines[cursorLine] ?? "";
      const beforeCursor = line.slice(0, cursorCol);
      const match = beforeCursor.match(/(?:^|[ \t])#([^\s#]*)$/);
      if (!match) {
        return current.getSuggestions(lines, cursorLine, cursorCol, options);
      }

      return {
        prefix: `#${match[1] ?? ""}`,
        items: [
          { value: "#2983", label: "#2983", description: "Extension API for registering custom @ autocomplete providers" },
          { value: "#2753", label: "#2753", description: "Reload stale resource settings" },
        ],
      };
    },

    applyCompletion(lines, cursorLine, cursorCol, item, prefix) {
      return current.applyCompletion(lines, cursorLine, cursorCol, item, prefix);
    },

    shouldTriggerFileCompletion(lines, cursorLine, cursorCol) {
      return current.shouldTriggerFileCompletion?.(lines, cursorLine, cursorCol) ?? true;
    },
  }));
});
```

完整示例请参阅 [github-issue-autocomplete.ts](../examples/extensions/github-issue-autocomplete.ts)。它使用 `gh issue list` 预加载最新的开放 GitHub Issue，并在本地筛选，以快速完成 `#...` 补全。该示例要求安装 GitHub CLI（`gh`），并位于 GitHub 仓库检出目录中。

### 自定义组件

对于复杂 UI，请使用 `ctx.ui.custom()`。它会暂时用自定义组件替换编辑器，直到调用 `done()`：

```typescript
import { Text, Component } from "@earendil-works/pi-tui";

const result = await ctx.ui.custom<boolean>((tui, theme, keybindings, done) => {
  const text = new Text("按 Enter 确认，按 Escape 取消", 1, 1);

  text.onKey = (key) => {
    if (key === "return") done(true);
    if (key === "escape") done(false);
    return true;
  };

  return text;
});

if (result) {
  // 用户按下了 Enter
}
```

回调接收：

- `tui`——TUI 实例，用于获取屏幕尺寸和管理焦点
- `theme`——用于设置样式的当前主题
- `keybindings`——应用按键绑定管理器，用于检查快捷键
- `done(value)`——调用后关闭组件并返回值

完整组件 API 请参阅 [tui.md](tui-zh.md)。

#### 浮层模式（实验性）

传入 `{ overlay: true }`，可将组件作为浮动模态框渲染在现有内容上方，而不清空屏幕：

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new MyOverlayComponent({ onClose: done }),
  { overlay: true }
);
```

对于锚点、外边距、百分比和响应式可见性等高级定位，请传入 `overlayOptions`。使用 `onHandle` 可以以编程方式控制焦点或可见性：

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new MyOverlayComponent({ onClose: done }),
  {
    overlay: true,
    overlayOptions: { anchor: "top-right", width: "50%", margin: 2 },
    onHandle: (handle) => {
      handle.focus(); // 让该浮层获得焦点并移到视觉最前方
      // handle.unfocus({ target: editorComponent }); // 将输入释放给指定组件
      // handle.setHidden(true/false); // 切换可见性
      // handle.hide(); // 永久移除
    }
  }
);
```

获得焦点且可见的浮层可以在临时非浮层自定义 UI 关闭后重新接管输入。如果希望浮层保持可见，但刻意让另一个组件继续接收输入，请调用 `handle.unfocus({ target })`。传入 `{ target: null }` 会释放浮层焦点，且不会让其他组件获得焦点。

完整的 `OverlayOptions` 和 `OverlayHandle` API 请参阅 [tui.md](tui-zh.md)，示例请参阅 [overlay-qa-tests.ts](../examples/extensions/overlay-qa-tests.ts)。

### 自定义编辑器

使用自定义实现替换主输入编辑器（Vim 模式、Emacs 模式等）：

```typescript
import { CustomEditor, type ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { matchesKey } from "@earendil-works/pi-tui";

class VimEditor extends CustomEditor {
  private mode: "normal" | "insert" = "insert";

  handleInput(data: string): void {
    if (matchesKey(data, "escape") && this.mode === "insert") {
      this.mode = "normal";
      return;
    }
    if (this.mode === "normal" && data === "i") {
      this.mode = "insert";
      return;
    }
    super.handleInput(data);  // 应用按键绑定 + 文本编辑
  }
}

export default function (pi: ExtensionAPI) {
  pi.on("session_start", (_event, ctx) => {
    ctx.ui.setEditorComponent((tui, theme, keybindings) =>
      new VimEditor(tui, theme, keybindings)
    );
  });
}
```

**要点：**

- 扩展 `CustomEditor`（而不是基础 `Editor`），以获得应用按键绑定，例如按 Escape 中止、按 Ctrl+D、切换模型
- 对没有自行处理的按键调用 `super.handleInput(data)`
- 工厂函数从应用接收 `tui`、`theme` 和 `keybindings`
- 调用 `setEditorComponent()` 前，使用 `ctx.ui.getEditorComponent()` 获取并包装此前配置的自定义编辑器
- 传入 `undefined` 可恢复默认编辑器：`ctx.ui.setEditorComponent(undefined)`

如果要与另一个已经替换编辑器的扩展组合，请在设置自己的工厂函数前捕获上一个工厂函数：

```typescript
const previous = ctx.ui.getEditorComponent();
ctx.ui.setEditorComponent((tui, theme, keybindings) =>
  new MyEditor(tui, theme, keybindings, { base: previous?.(tui, theme, keybindings) })
);
```

带模式指示器的完整示例请参阅 [tui.md](tui-zh.md) 的模式 7。

### 消息与条目渲染

为指定 `customType` 的消息注册自定义渲染器。需要进入 LLM 上下文的内容应使用消息渲染器：

```typescript
import { Text } from "@earendil-works/pi-tui";

pi.registerMessageRenderer("my-extension", (message, options, theme) => {
  const { expanded, outputPad } = options;
  let text = theme.fg("accent", `[${message.customType}] `);
  text += message.content;

  if (expanded && message.details) {
    text += "\n" + theme.fg("dim", JSON.stringify(message.details, null, 2));
  }

  return new Text(text, outputPad, 0);
});
```

消息通过 `pi.sendMessage()` 发送：

```typescript
pi.sendMessage({
  customType: "my-extension",  // 与 registerMessageRenderer 匹配
  content: "Status update",
  display: true,               // 在 TUI 中显示
  details: { ... },            // 可供渲染器使用
});
```

对于只应在 TUI 中显示、不应发送给 LLM 的内容，请改为渲染自定义条目：

```typescript
pi.registerEntryRenderer("my-card", (entry, options, theme) => {
  return new Text(theme.fg("accent", JSON.stringify(entry.data)));
});

pi.appendEntry("my-card", { status: "done" });
```

### 主题颜色

所有渲染函数都会收到 `theme` 对象。自定义主题的创建方法和完整调色板请参阅 [themes.md](themes-zh.md)。

```typescript
// 前景色
theme.fg("toolTitle", text)   // 工具名称
theme.fg("accent", text)      // 强调内容
theme.fg("success", text)     // 成功（绿色）
theme.fg("error", text)       // 错误（红色）
theme.fg("warning", text)     // 警告（黄色）
theme.fg("muted", text)       // 次要文本
theme.fg("dim", text)         // 三级文本

// 文本样式
theme.bold(text)
theme.italic(text)
theme.strikethrough(text)
```

在自定义工具渲染器中进行语法高亮：

```typescript
import { highlightCode, getLanguageFromPath } from "@earendil-works/pi-coding-agent";

// 使用显式指定的语言高亮代码
const highlighted = highlightCode("const x = 1;", "typescript", theme);

// 根据文件路径自动检测语言
const lang = getLanguageFromPath("/path/to/file.rs");  // "rust"
const highlighted = highlightCode(code, lang, theme);
```

<a id="error-handling"></a>

## 错误处理

- 扩展错误会写入日志，智能体继续运行
- `tool_call` 错误会阻止工具执行（故障安全）
- 工具 `execute` 必须通过抛出异常报告错误；抛出的异常会被捕获，并以 `isError: true` 报告给 LLM，随后继续执行

<a id="mode-behavior"></a>

## 各模式下的行为

| 模式 | `ctx.mode` | `ctx.hasUI` | 说明 |
|------|------------|-------------|-------|
| 交互模式 | `"tui"` | `true` | 提供终端渲染的完整 TUI |
| RPC（`--mode rpc`） | `"rpc"` | `true` | 通过 JSON 协议提供对话框和通知；`custom()` 返回 `undefined`。参见 [rpc.md](rpc-zh.md) |
| JSON（`--mode json`） | `"json"` | `false` | 将事件流写入标准输出；UI 方法不执行任何操作 |
| 打印模式（`-p`） | `"print"` | `false` | 扩展会运行，但无法提示用户 |

使用 TUI 特有功能（`custom()`、组件工厂函数、终端输入）前，请检查 `ctx.mode === "tui"`。使用同时支持 TUI 和 RPC 模式的对话框及通知方法前，请检查 `ctx.hasUI`。

<a id="examples-reference"></a>

## 示例索引

所有示例均位于 [examples/extensions/](../examples/extensions/)。

| 示例 | 说明 | 主要 API |
|---------|-------------|----------|
| **工具** |||
| `hello.ts` | 最简工具注册 | `registerTool` |
| `question.ts` | 与用户交互的工具 | `registerTool`, `ui.select` |
| `questionnaire.ts` | 多步骤向导工具 | `registerTool`, `ui.custom` |
| `todo.ts` | 支持持久化的有状态工具 | `registerTool`, `appendEntry`, `renderResult`, 会话事件 |
| `dynamic-tools.ts` | 启动后和命令运行期间注册工具 | `registerTool`, `session_start`, `registerCommand` |
| `structured-output.ts` | 带 `terminate: true` 的最终结构化输出工具 | `registerTool`, 终止型工具结果 |
| `truncated-tool.ts` | 输出截断示例 | `registerTool`, `truncateHead` |
| `tool-override.ts` | 覆盖内置 read 工具 | `registerTool`（与内置工具同名） |
| **命令** |||
| `pirate.ts` | 按轮次修改系统提示词 | `registerCommand`, `before_agent_start` |
| `summarize.ts` | 对话摘要命令 | `registerCommand`, `ui.custom` |
| `handoff.ts` | 跨服务提供商的模型交接 | `registerCommand`, `ui.editor`, `ui.custom` |
| `qna.ts` | 使用自定义 UI 的问答 | `registerCommand`, `ui.custom`, `setEditorText` |
| `send-user-message.ts` | 注入用户消息 | `registerCommand`, `sendUserMessage` |
| `reload-runtime.ts` | 重载命令与 LLM 工具交接 | `registerCommand`, `ctx.reload()`, `sendUserMessage` |
| `shutdown-command.ts` | 平稳关闭命令 | `registerCommand`, `shutdown()` |
| **事件与关卡** |||
| `permission-gate.ts` | 阻止危险命令 | `on("tool_call")`, `ui.confirm` |
| `project-trust.ts` | 从用户/全局扩展或 CLI 扩展决定或推迟项目信任 | `on("project_trust")`, 信任 UI, 必填的信任结果 |
| `protected-paths.ts` | 阻止写入特定路径 | `on("tool_call")` |
| `confirm-destructive.ts` | 确认会话变更 | `on("session_before_switch")`, `on("session_before_fork")` |
| `dirty-repo-guard.ts` | Git 工作区有未提交变更时发出警告 | `on("session_before_*")`, `exec` |
| `input-transform.ts` | 转换用户输入 | `on("input")` |
| `input-transform-streaming.ts` | 感知流式状态的输入转换 | `on("input")`, `streamingBehavior` |
| `model-status.ts` | 响应模型变化 | `on("model_select")`, `setStatus` |
| `provider-payload.ts` | 检查请求体和服务提供商响应头 | `on("before_provider_request")`, `on("after_provider_response")` |
| `system-prompt-header.ts` | 显示系统提示词信息 | `on("agent_start")`, `getSystemPrompt` |
| `claude-rules.ts` | 从文件加载规则 | `on("session_start")`, `on("before_agent_start")` |
| `prompt-customizer.ts` | 使用 `systemPromptOptions` 添加感知上下文的工具指引 | `on("before_agent_start")`, `BuildSystemPromptOptions` |
| `file-trigger.ts` | 文件监视器触发消息 | `sendMessage` |
| **上下文压缩与会话** |||
| `custom-compaction.ts` | 自定义上下文压缩摘要 | `on("session_before_compact")` |
| `trigger-compact.ts` | 手动触发上下文压缩 | `compact()` |
| `git-checkpoint.ts` | 在轮次中执行 Git 暂存 | `on("turn_start")`, `on("session_before_fork")`, `exec` |
| `git-merge-and-resolve.ts` | 获取、合并并解决冲突 | `on("agent_end")`, `exec`, `sendUserMessage` |
| `auto-commit-on-exit.ts` | 关闭时提交 | `on("session_shutdown")`, `exec` |
| **UI 组件** |||
| `status-line.ts` | 页脚状态指示器 | `setStatus`, 会话事件 |
| `working-indicator.ts` | 自定义流式工作状态指示器 | `setWorkingIndicator`, `registerCommand` |
| `github-issue-autocomplete.ts` | 使用 `gh issue list` 预加载最近开放的 Issue，在内置自动补全上添加 `#1234` Issue 补全 | `addAutocompleteProvider`, `on("session_start")`, `exec` |
| `custom-footer.ts` | 完全替换页脚 | `registerCommand`, `setFooter` |
| `custom-header.ts` | 替换启动标题 | `on("session_start")`, `setHeader` |
| `modal-editor.ts` | Vim 风格模态编辑器 | `setEditorComponent`, `CustomEditor` |
| `rainbow-editor.ts` | 自定义编辑器样式 | `setEditorComponent` |
| `widget-placement.ts` | 编辑器上方/下方的小组件 | `setWidget` |
| `overlay-test.ts` | 浮层组件 | 带浮层选项的 `ui.custom` |
| `overlay-qa-tests.ts` | 完整浮层测试 | `ui.custom`, 所有浮层选项 |
| `notify.ts` | 简单通知 | `ui.notify` |
| `timed-confirm.ts` | 带超时的对话框 | 带超时/信号的 `ui.confirm` |
| `mac-system-theme.ts` | 自动切换主题 | `setTheme`, `exec` |
| **复杂扩展** |||
| `plan-mode/` | 完整的计划模式实现 | 所有事件类型、`registerCommand`, `registerShortcut`, `registerFlag`, `setStatus`, `setWidget`, `sendMessage`, `setActiveTools` |
| `preset.ts` | 可保存的预设（模型、工具、思考） | `registerCommand`, `registerShortcut`, `registerFlag`, `setModel`, `setActiveTools`, `setThinkingLevel`, `appendEntry` |
| `tools.ts` | 用 UI 启用/禁用工具 | `registerCommand`, `setActiveTools`, `SettingsList`, 会话事件 |
| **远程与沙箱** |||
| `ssh.ts` | SSH 远程执行 | `registerFlag`, `on("user_bash")`, `on("before_agent_start")`, 工具操作 |
| `interactive-shell.ts` | 持久 Shell 会话 | `on("user_bash")` |
| `sandbox/` | 沙箱化工具执行 | 工具操作 |
| `gondolin/` | 将内置工具和 `!` 命令路由到 Gondolin 微型虚拟机 | 工具操作、覆盖内置工具、`on("user_bash")` |
| `subagent/` | 创建子智能体 | `registerTool`, `exec` |
| **游戏** |||
| `snake.ts` | 贪吃蛇游戏 | `registerCommand`, `ui.custom`, 键盘处理 |
| `space-invaders.ts` | 太空侵略者游戏 | `registerCommand`, `ui.custom` |
| `doom-overlay/` | 浮层中的 Doom | 带浮层的 `ui.custom` |
| **服务提供商** |||
| `custom-provider-anthropic/` | 自定义 Anthropic 代理 | `registerProvider` |
| `custom-provider-gitlab-duo/` | GitLab Duo 集成 | 带 OAuth 的 `registerProvider` |
| **消息与通信** |||
| `message-renderer.ts` | 自定义消息渲染 | `registerMessageRenderer`, `sendMessage` |
| `entry-renderer.ts` | 仅 TUI 的自定义条目渲染 | `registerEntryRenderer`, `appendEntry` |
| `event-bus.ts` | 扩展间事件 | `pi.events` |
| **会话元数据** |||
| `session-name.ts` | 为会话选择器命名会话 | `setSessionName`, `getSessionName` |
| `bookmark.ts` | 为 /tree 添加条目书签 | `setLabel` |
| **其他** |||
| `inline-bash.ts` | 工具调用中的内联 Bash | `on("tool_call")` |
| `bash-spawn-hook.ts` | 执行前调整 Bash 命令、cwd 和环境变量 | `createBashTool`, `spawnHook` |
| `with-deps/` | 带 npm 依赖的扩展 | 包含 `package.json` 的包结构 |
