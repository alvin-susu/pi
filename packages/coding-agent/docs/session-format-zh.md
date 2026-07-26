# 会话文件格式

会话以 JSONL（JSON Lines）文件存储。每一行都是带有 `type` 字段的 JSON 对象。会话条目通过 `id`/`parentId` 字段组成树形结构，因此无需创建新文件即可在原会话中产生分支。

## 文件位置

```
~/.pi/agent/sessions/--<path>--/<timestamp>_<uuid>.jsonl
```

其中 `<path>` 是将工作目录中的 `/` 替换为 `-` 后得到的路径。

## 删除会话

删除 `~/.pi/agent/sessions/` 下相应的 `.jsonl` 文件即可移除会话。

Pi 也支持在 `/resume` 中交互式删除会话：选中一个会话，按 `Ctrl+D`，然后确认。如果系统中提供了 `trash` 命令行工具，pi 会优先使用它，避免永久删除文件。

## 会话版本

会话文件头中包含版本字段：

- **版本 1**：线性条目序列（旧格式，加载时自动迁移）
- **版本 2**：通过 `id`/`parentId` 关联的树形结构
- **版本 3**：将 `hookMessage` 角色重命名为 `custom`（统一扩展机制）

加载已有会话时，会自动将其迁移到当前版本（v3）。

## 源文件

GitHub 上的源代码（[pi-mono](https://github.com/earendil-works/pi-mono)）：

- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts)——会话条目类型和 `SessionManager`
- [`packages/coding-agent/src/core/messages.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/messages.ts)——扩展消息类型（`BashExecutionMessage`、`CustomMessage` 等）
- [`packages/ai/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/ai/src/types.ts)——基础消息类型（`UserMessage`、`AssistantMessage`、`ToolResultMessage`）
- [`packages/agent/src/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/agent/src/types.ts)——`AgentMessage` 联合类型

如需查看项目中安装的 TypeScript 类型定义，请检查 `node_modules/@earendil-works/pi-coding-agent/dist/` 和 `node_modules/@earendil-works/pi-ai/dist/`。

## 消息类型

会话条目包含 `AgentMessage` 对象。若要解析会话或编写扩展，必须理解这些类型。

### 内容块

消息包含由不同类型内容块组成的数组：

```typescript
interface TextContent {
  type: "text";
  text: string;
}

interface ImageContent {
  type: "image";
  data: string;      // Base64 编码
  mimeType: string;  // 例如 "image/jpeg"、"image/png"
}

interface ThinkingContent {
  type: "thinking";
  thinking: string;
}

interface ToolCall {
  type: "toolCall";
  id: string;
  name: string;
  arguments: Record<string, any>;
}
```

### 基础消息类型（来自 pi-ai）

```typescript
interface UserMessage {
  role: "user";
  content: string | (TextContent | ImageContent)[];
  timestamp: number;  // Unix 时间戳，单位为毫秒
}

interface AssistantMessage {
  role: "assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api: string;
  provider: string;
  model: string;
  usage: Usage;
  stopReason: "stop" | "length" | "toolUse" | "error" | "aborted";
  errorMessage?: string;
  timestamp: number;
}

interface ToolResultMessage {
  role: "toolResult";
  toolCallId: string;
  toolName: string;
  content: (TextContent | ImageContent)[];
  details?: any;      // 工具特有的元数据
  usage?: Usage;      // 工具内部调用 LLM 所产生的用量
  isError: boolean;
  timestamp: number;
}

interface Usage {
  input: number;
  output: number;
  cacheRead: number;
  cacheWrite: number;
  totalTokens: number;
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
    total: number;
  };
}
```

### 扩展消息类型（来自 pi-coding-agent）

```typescript
interface BashExecutionMessage {
  role: "bashExecution";
  command: string;
  output: string;
  exitCode: number | undefined;
  cancelled: boolean;
  truncated: boolean;
  fullOutputPath?: string;
  excludeFromContext?: boolean;  // 以 !! 开头的命令为 true
  timestamp: number;
}

interface CustomMessage {
  role: "custom";
  customType: string;            // 扩展标识符
  content: string | (TextContent | ImageContent)[];
  display: boolean;              // 是否在 TUI 中显示
  details?: any;                 // 扩展特有的元数据
  timestamp: number;
}

interface BranchSummaryMessage {
  role: "branchSummary";
  summary: string;
  fromId: string;                // 创建分支时所在的条目
  timestamp: number;
}

interface CompactionSummaryMessage {
  role: "compactionSummary";
  summary: string;
  tokensBefore: number;
  timestamp: number;
}
```

### AgentMessage 联合类型

```typescript
type AgentMessage =
  | UserMessage
  | AssistantMessage
  | ToolResultMessage
  | BashExecutionMessage
  | CustomMessage
  | BranchSummaryMessage
  | CompactionSummaryMessage;
```

## 条目基类

除 `SessionHeader` 外，所有条目都扩展自 `SessionEntryBase`：

```typescript
interface SessionEntryBase {
  type: string;
  id: string;           // 8 位十六进制 ID
  parentId: string | null;  // 父条目 ID（首个条目为 null）
  timestamp: string;    // ISO 时间戳
}
```

## 条目类型

### SessionHeader

文件的第一行，只包含元数据，不属于条目树（没有 `id`/`parentId`）。

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project"}
```

对于存在父会话的会话（通过 `/fork`、`/clone` 或 `newSession({ parentSession })` 创建）：

```json
{"type":"session","version":3,"id":"uuid","timestamp":"2024-12-03T14:00:00.000Z","cwd":"/path/to/project","parentSession":"/path/to/original/session.jsonl"}
```

### SessionMessageEntry

对话中的一条消息。`message` 字段包含一个 `AgentMessage`。

```json
{"type":"message","id":"a1b2c3d4","parentId":"prev1234","timestamp":"2024-12-03T14:00:01.000Z","message":{"role":"user","content":"Hello"}}
{"type":"message","id":"b2c3d4e5","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:00:02.000Z","message":{"role":"assistant","content":[{"type":"text","text":"Hi!"}],"provider":"anthropic","model":"claude-sonnet-4-5","usage":{...},"stopReason":"stop"}}
{"type":"message","id":"c3d4e5f6","parentId":"b2c3d4e5","timestamp":"2024-12-03T14:00:03.000Z","message":{"role":"toolResult","toolCallId":"call_123","toolName":"bash","content":[{"type":"text","text":"output"}],"isError":false}}
```

### ModelChangeEntry

当用户在会话过程中切换模型时产生。

```json
{"type":"model_change","id":"d4e5f6g7","parentId":"c3d4e5f6","timestamp":"2024-12-03T14:05:00.000Z","provider":"openai","modelId":"gpt-4o"}
```

### ThinkingLevelChangeEntry

当用户更改思考/推理级别时产生。

```json
{"type":"thinking_level_change","id":"e5f6g7h8","parentId":"d4e5f6g7","timestamp":"2024-12-03T14:06:00.000Z","thinkingLevel":"high"}
```

### CompactionEntry

压缩上下文时创建，用于存储早期消息的摘要。

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","timestamp":"2024-12-03T14:10:00.000Z","summary":"User discussed X, Y, Z...","firstKeptEntryId":"c3d4e5f6","tokensBefore":50000}
```

新版运行框架生成的压缩条目会直接嵌入压缩后保留的上下文，不再仅依赖 `firstKeptEntryId`：

```json
{"type":"compaction","id":"f6g7h8i9","parentId":"e5f6g7h8","timestamp":"2024-12-03T14:10:00.000Z","summary":"User discussed X, Y, Z...","tokensBefore":50000,"retainedTail":[{"role":"user","content":"latest request"},{"role":"assistant","content":[{"type":"text","text":"latest reply"}],"provider":"anthropic","model":"claude-sonnet-4-5","usage":{...},"stopReason":"stop"}]}
```

可选字段：

- `usage`：生成摘要时的 LLM 用量；会计入会话的 Token 和费用总计
- `retainedTail`：压缩后保留下来的实体化 `AgentMessage[]`。该字段仅为兼容旧会话而设为可选。新版运行框架生成的压缩条目会包含此字段，因此可以直接从这个检查点重建上下文，无需向前遍历压缩条目之前的旧条目。
- `details`：与实现相关的数据（例如默认实现中的 `{ readFiles: string[], modifiedFiles: string[] }`，或扩展自定义的数据）
- `fromHook`：由扩展生成时为 `true`，由 pi 生成时为 `false`/`undefined`（沿用的旧字段名）
- `firstKeptEntryId`：用于兼容旧版条目格式

### BranchSummaryEntry

通过 `/tree` 切换分支时创建。LLM 会对离开分支中截至共同祖先的内容生成摘要，以保留被放弃路径中的上下文。

```json
{"type":"branch_summary","id":"g7h8i9j0","parentId":"a1b2c3d4","timestamp":"2024-12-03T14:15:00.000Z","fromId":"f6g7h8i9","summary":"Branch explored approach A..."}
```

可选字段：

- `usage`：生成摘要时的 LLM 用量；会计入会话的 Token 和费用总计
- `details`：默认实现中的文件跟踪数据（`{ readFiles: string[], modifiedFiles: string[] }`），或扩展自定义的数据
- `fromHook`：由扩展生成时为 `true`，由 pi 生成时为 `false`/`undefined`（沿用的旧字段名）

### CustomEntry

用于持久化扩展状态，**不会**进入 LLM 上下文。

```json
{"type":"custom","id":"h8i9j0k1","parentId":"g7h8i9j0","timestamp":"2024-12-03T14:20:00.000Z","customType":"my-extension","data":{"count":42}}
```

重新加载时，可通过 `customType` 识别扩展自己的条目。交互模式可以使用 `pi.registerEntryRenderer(customType, renderer)` 渲染自定义条目，但这些条目仍不会进入 LLM 上下文。

### CustomMessageEntry

由扩展注入、**会**进入 LLM 上下文的消息。

```json
{"type":"custom_message","id":"i9j0k1l2","parentId":"h8i9j0k1","timestamp":"2024-12-03T14:25:00.000Z","customType":"my-extension","content":"Injected context...","display":true}
```

字段：

- `content`：字符串或 `(TextContent | ImageContent)[]`（与 `UserMessage` 相同）
- `display`：`true` 表示以独立样式显示在 TUI 中，`false` 表示隐藏
- `details`：可选的扩展专用元数据（不会发送给 LLM）

### LabelEntry

用户在条目上设置的书签或标记。

```json
{"type":"label","id":"j0k1l2m3","parentId":"i9j0k1l2","timestamp":"2024-12-03T14:30:00.000Z","targetId":"a1b2c3d4","label":"checkpoint-1"}
```

将 `label` 设为 `undefined` 即可清除标签。

### SessionInfoEntry

会话元数据（例如用户定义的显示名称）。可通过 `/name`、`--name` / `-n`，或扩展中的 `pi.setSessionName()` 设置。

```json
{"type":"session_info","id":"k1l2m3n4","parentId":"j0k1l2m3","timestamp":"2024-12-03T14:35:00.000Z","name":"Refactor auth module"}
```

设置会话名称后，会话选择器（`/resume`）将显示该名称，而不是首条消息。

## 树形结构

条目组成一棵树：

- 首个条目的 `parentId` 为 `null`
- 后续每个条目都通过 `parentId` 指向父条目
- 创建分支时，会从较早的条目生成新的子条目
- “叶节点”表示当前在树中的位置

```
[用户消息] ─── [助手] ─── [用户消息] ─── [助手] ─┬─ [用户消息] ← 当前叶节点
                                                            │
                                                            └─ [branch_summary] ─── [用户消息] ← 另一分支
```

## 构建上下文

`buildContextEntries()` 从当前叶节点向根节点遍历，在应用上下文压缩结果的同时生成当前有效的条目列表：

1. 收集路径上的所有条目
2. 如果路径上存在 `CompactionEntry`：
   - 首先加入压缩条目
   - 如果存在 `retainedTail`，则将该条目视为自包含的检查点，并加入压缩后的条目
   - 否则，加入从 `firstKeptEntryId` 到压缩条目之间的条目
   - 最后加入压缩后的条目
3. 保留所选范围内的非消息条目，以便交互模式渲染

`buildSessionContext()` 基于该条目列表生成供 LLM 使用的消息列表：

1. 从完整路径中提取当前模型和思考级别设置
2. 将选中的条目转换为消息：
   - `message` -> 已存储的 `AgentMessage`
   - `compaction` -> `compactionSummary`；如存在，再加上 `retainedTail`
   - `branch_summary` -> `branchSummary`
   - `custom_message` -> `CustomMessage`
   - `custom` -> 不生成上下文消息

这样，新版压缩条目就能作为自包含的检查点。`retainedTail` 设为可选，仅用于确保只存储 `firstKeptEntryId` 的旧会话仍可正确加载。

## 解析示例

```typescript
import { readFileSync } from "fs";

const lines = readFileSync("session.jsonl", "utf8").trim().split("\n");

for (const line of lines) {
  const entry = JSON.parse(line);

  switch (entry.type) {
    case "session":
      console.log(`Session v${entry.version ?? 1}: ${entry.id}`);
      break;
    case "message":
      console.log(`[${entry.id}] ${entry.message.role}: ${JSON.stringify(entry.message.content)}`);
      break;
    case "compaction":
      console.log(`[${entry.id}] Compaction: ${entry.tokensBefore} tokens summarized`);
      break;
    case "branch_summary":
      console.log(`[${entry.id}] Branch from ${entry.fromId}`);
      break;
    case "custom":
      console.log(`[${entry.id}] Custom (${entry.customType}): ${JSON.stringify(entry.data)}`);
      break;
    case "custom_message":
      console.log(`[${entry.id}] Extension message (${entry.customType}): ${entry.content}`);
      break;
    case "label":
      console.log(`[${entry.id}] Label "${entry.label}" on ${entry.targetId}`);
      break;
    case "model_change":
      console.log(`[${entry.id}] Model: ${entry.provider}/${entry.modelId}`);
      break;
    case "thinking_level_change":
      console.log(`[${entry.id}] Thinking: ${entry.thinkingLevel}`);
      break;
  }
}
```

## SessionManager API

以下是以编程方式操作会话时使用的主要方法。

### 静态创建方法

- `SessionManager.create(cwd, sessionDir?)`——创建新会话
- `SessionManager.open(path, sessionDir?)`——打开已有会话文件
- `SessionManager.continueRecent(cwd, sessionDir?)`——继续最近的会话；不存在时创建新会话
- `SessionManager.inMemory(cwd?)`——创建不持久化到文件的会话
- `SessionManager.forkFrom(sourcePath, targetCwd, sessionDir?)`——从另一个项目的会话派生新会话

### 静态列举方法

- `SessionManager.list(cwd, sessionDir?, onProgress?)`——列出指定目录的会话
- `SessionManager.listAll(onProgress?)`——列出所有项目中的全部会话

### 实例方法——会话管理

- `newSession(options?)`——启动新会话（选项：`{ parentSession?: string }`）
- `setSessionFile(path)`——切换到其他会话文件
- `createBranchedSession(leafId)`——将分支提取到新的会话文件

### 实例方法——追加条目

以下方法均返回条目 ID：

- `appendMessage(message)`——添加消息
- `appendThinkingLevelChange(level)`——记录思考级别变化
- `appendModelChange(provider, modelId)`——记录模型变化
- `appendCompaction(summary, firstKeptEntryId, tokensBefore, details?, fromHook?)`——添加压缩条目
- `appendCustomEntry(customType, data?)`——添加扩展状态（不进入上下文）
- `appendSessionInfo(name)`——设置会话显示名称
- `appendCustomMessageEntry(customType, content, display, details?)`——添加扩展消息（进入上下文）
- `appendLabelChange(targetId, label)`——设置或清除标签

### 实例方法——树导航

- `getLeafId()`——获取当前位置
- `getLeafEntry()`——获取当前叶条目
- `getEntry(id)`——按 ID 获取条目
- `getBranch(fromId?)`——从指定条目遍历到根节点
- `getTree()`——获取完整树结构
- `getChildren(parentId)`——获取直接子条目
- `getLabel(id)`——获取条目的标签
- `branch(entryId)`——将叶节点移至较早的条目
- `resetLeaf()`——将叶节点重置为 `null`（位于所有条目之前）
- `branchWithSummary(entryId, summary, details?, fromHook?)`——携带上下文摘要创建分支

### 实例方法——上下文与信息

- `buildContextEntries()`——获取应用压缩后的当前分支条目
- `buildSessionContext()`——获取供 LLM 使用的消息、`thinkingLevel` 和模型
- `getEntries()`——获取所有条目（不含文件头）
- `getHeader()`——获取会话文件头元数据
- `getSessionName()`——从最新的 `session_info` 条目获取显示名称
- `getCwd()`——获取工作目录
- `getSessionDir()`——获取会话存储目录
- `getSessionId()`——获取会话 UUID
- `getSessionFile()`——获取会话文件路径（内存会话为 `undefined`）
- `isPersisted()`——判断会话是否已保存到磁盘
