# RPC 模式

RPC 模式允许通过标准输入/标准输出上的 JSON 协议，以无界面方式运行编程智能体。它适用于将智能体嵌入其他应用、IDE 或自定义界面。

**Node.js/TypeScript 用户注意：** 如果正在构建 Node.js 应用，请考虑直接使用 `@earendil-works/pi-coding-agent` 中的 `AgentSession`，而不是创建子进程。API 请参阅 [`src/core/agent-session.ts`](../src/core/agent-session.ts)。基于子进程的 TypeScript 客户端请参阅 [`src/modes/rpc/rpc-client.ts`](../src/modes/rpc/rpc-client.ts)。

## 启动 RPC 模式

```bash
pi --mode rpc [options]
```

常用选项：

- `--provider <name>`：设置 LLM 服务提供商（anthropic、openai、google 等）
- `--model <pattern>`：模型匹配模式或 ID（支持 `provider/id` 和可选的 `:<thinking>`）
- `--name <name>` / `-n <name>`：启动时设置会话显示名称
- `--no-session`：禁用会话持久化
- `--session-dir <path>`：自定义会话存储目录

## 协议概览

- **命令**：发送到标准输入的 JSON 对象，每行一个
- **响应**：包含 `type: "response"` 的 JSON 对象，用于表示命令成功或失败
- **事件**：以 JSONL 形式流式写入标准输出的智能体事件

所有命令都支持可选的 `id` 字段，用于关联请求和响应。如果提供了该字段，对应响应会包含相同的 `id`。`bash_execution_update` 事件也会包含其来源 `bash` 命令的 `id`。

### 消息分帧

RPC 模式采用严格的 JSONL 语义，只使用 LF（`\n`）分隔记录。

客户端必须注意：

- 只能按 `\n` 拆分记录
- 可以接受 `\r\n` 输入，但需要移除末尾的 `\r`
- 不要使用会把 Unicode 分隔符也视为换行符的通用逐行读取器

尤其是 Node 的 `readline`，它还会按 `U+2028` 和 `U+2029` 拆分，而这两个字符可以合法地出现在 JSON 字符串中，因此不符合 RPC 模式的协议要求。

## 命令

### 提示词

#### prompt

向智能体发送用户提示词。提示词被接受、加入队列或处理后，会返回命令响应；接受后，事件会继续异步流式传输。

```json
{"id": "req-1", "type": "prompt", "message": "Hello, world!"}
```

携带图像：
```json
{"type": "prompt", "message": "What's in this image?", "images": [{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}]}
```

**流式响应期间：** 如果智能体正在进行流式响应，必须指定 `streamingBehavior`，将消息加入队列：

```json
{"type": "prompt", "message": "New instruction", "streamingBehavior": "steer"}
```

- `"steer"`：在智能体运行期间将消息加入队列。当前助手轮次完成工具调用后、下一次调用 LLM 前发送。
- `"followUp"`：等待智能体完成。只有智能体停止后才发送消息。

如果智能体正在进行流式响应，但未指定 `streamingBehavior`，命令会返回错误。

**扩展命令：** 如果消息是扩展命令（例如 `/mycommand`），即使正在进行流式响应，也会立即执行。扩展命令通过 `pi.sendMessage()` 自行管理与 LLM 的交互。

**输入展开：** 技能命令（`/skill:name`）和提示词模板（`/template`）会在发送或加入队列前展开。

响应：
```json
{"id": "req-1", "type": "response", "command": "prompt", "success": true}
```

`success: true` 表示提示词已被接受、加入队列或立即得到处理。`success: false` 表示提示词在接受前被拒绝。接受后发生的故障通过正常的事件流和消息流报告，不会为同一请求 ID 返回第二个 `response`。

`images` 字段可选。每张图像都使用 `ImageContent` 格式：`{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}`。

#### steer

在智能体运行期间将引导消息加入队列。该消息会在当前助手轮次完成工具调用后、下一次调用 LLM 前发送。技能命令和提示词模板会展开。不允许使用扩展命令；请改用 `prompt`。

```json
{"type": "steer", "message": "Stop and do this instead"}
```

携带图像：
```json
{"type": "steer", "message": "Look at this instead", "images": [{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}]}
```

`images` 字段可选。每张图像都使用 `ImageContent` 格式（与 `prompt` 相同）。

响应：
```json
{"type": "response", "command": "steer", "success": true}
```

如何控制引导消息的处理方式，请参阅 [set_steering_mode](#set_steering_mode)。

#### follow_up

将后续消息加入队列，等智能体完成后再处理。只有当智能体不再有工具调用或引导消息时才会发送。技能命令和提示词模板会展开。不允许使用扩展命令；请改用 `prompt`。

```json
{"type": "follow_up", "message": "After you're done, also do this"}
```

携带图像：
```json
{"type": "follow_up", "message": "Also check this image", "images": [{"type": "image", "data": "base64-encoded-data", "mimeType": "image/png"}]}
```

`images` 字段可选。每张图像都使用 `ImageContent` 格式（与 `prompt` 相同）。

响应：
```json
{"type": "response", "command": "follow_up", "success": true}
```

如何控制后续消息的处理方式，请参阅 [set_follow_up_mode](#set_follow_up_mode)。

#### abort

中止当前智能体操作。

```json
{"type": "abort"}
```

响应：
```json
{"type": "response", "command": "abort", "success": true}
```

#### new_session

启动全新会话。`session_before_switch` 扩展事件处理函数可以取消该操作。

```json
{"type": "new_session"}
```

可选跟踪父会话：
```json
{"type": "new_session", "parentSession": "/path/to/parent-session.jsonl"}
```

响应：
```json
{"type": "response", "command": "new_session", "success": true, "data": {"cancelled": false}}
```

如果扩展取消了操作：
```json
{"type": "response", "command": "new_session", "success": true, "data": {"cancelled": true}}
```

### 状态

#### get_state

获取当前会话状态。

```json
{"type": "get_state"}
```

响应：
```json
{
  "type": "response",
  "command": "get_state",
  "success": true,
  "data": {
    "model": {...},
    "thinkingLevel": "medium",
    "isStreaming": false,
    "isCompacting": false,
    "steeringMode": "all",
    "followUpMode": "one-at-a-time",
    "sessionFile": "/path/to/session.jsonl",
    "sessionId": "abc123",
    "sessionName": "my-feature-work",
    "autoCompactionEnabled": true,
    "messageCount": 5,
    "pendingMessageCount": 0
  }
}
```

`model` 字段为完整的 [Model](#model) 对象或 `null`。`sessionName` 字段是通过 `set_session_name` 设置的显示名称；未设置时省略。

#### get_messages

获取对话中的全部消息。

```json
{"type": "get_messages"}
```

响应：
```json
{
  "type": "response",
  "command": "get_messages",
  "success": true,
  "data": {"messages": [...]}
}
```

消息为 `AgentMessage` 对象（参见[消息类型](#message-types)）。

### 模型

#### set_model

切换到指定模型。

```json
{"type": "set_model", "provider": "anthropic", "modelId": "claude-sonnet-4-20250514"}
```

响应中包含完整的 [Model](#model) 对象：
```json
{
  "type": "response",
  "command": "set_model",
  "success": true,
  "data": {...}
}
```

#### cycle_model

循环切换到下一个可用模型。如果只有一个可用模型，则返回 `null` 数据。

```json
{"type": "cycle_model"}
```

响应：
```json
{
  "type": "response",
  "command": "cycle_model",
  "success": true,
  "data": {
    "model": {...},
    "thinkingLevel": "medium",
    "isScoped": false
  }
}
```

`model` 字段为完整的 [Model](#model) 对象。

#### get_available_models

列出所有已配置的模型。

```json
{"type": "get_available_models"}
```

响应中包含由完整 [Model](#model) 对象组成的数组：
```json
{
  "type": "response",
  "command": "get_available_models",
  "success": true,
  "data": {
    "models": [...]
  }
}
```

### 思考

#### set_thinking_level

为支持推理/思考的模型设置相应级别。

```json
{"type": "set_thinking_level", "level": "high"}
```

级别：`"off"`、`"minimal"`、`"low"`、`"medium"`、`"high"`、`"xhigh"`、`"max"`

只有当前模型支持时，才会提供 `"xhigh"` 和 `"max"`。包括 GPT-5.6 在内的部分模型同时提供这两个级别。

响应：
```json
{"type": "response", "command": "set_thinking_level", "success": true}
```

#### cycle_thinking_level

循环切换可用的思考级别。如果模型不支持思考，则返回 `null` 数据。

```json
{"type": "cycle_thinking_level"}
```

响应：
```json
{
  "type": "response",
  "command": "cycle_thinking_level",
  "success": true,
  "data": {"level": "high"}
}
```

#### get_available_thinking_levels

列出当前模型支持的思考级别。对于不支持推理的模型，返回 `["off"]`。

```json
{"type": "get_available_thinking_levels"}
```

响应：
```json
{
  "type": "response",
  "command": "get_available_thinking_levels",
  "success": true,
  "data": {
    "levels": ["off", "minimal", "low", "medium", "high"]
  }
}
```

### 队列模式

#### set_steering_mode

控制如何发送来自 `steer` 的引导消息。

```json
{"type": "set_steering_mode", "mode": "one-at-a-time"}
```

模式：

- `"all"`：当前助手轮次完成工具调用后，发送全部引导消息
- `"one-at-a-time"`：每完成一个助手轮次，发送一条引导消息（默认值）

响应：
```json
{"type": "response", "command": "set_steering_mode", "success": true}
```

#### set_follow_up_mode

控制如何发送来自 `follow_up` 的后续消息。

```json
{"type": "set_follow_up_mode", "mode": "one-at-a-time"}
```

模式：

- `"all"`：智能体完成后，发送全部后续消息
- `"one-at-a-time"`：智能体每完成一次，发送一条后续消息（默认值）

响应：
```json
{"type": "response", "command": "set_follow_up_mode", "success": true}
```

### 上下文压缩

#### compact

手动压缩对话上下文，以减少 Token 用量。

```json
{"type": "compact"}
```

携带自定义指令：
```json
{"type": "compact", "customInstructions": "Focus on code changes"}
```

响应：
```json
{
  "type": "response",
  "command": "compact",
  "success": true,
  "data": {
    "summary": "Summary of conversation...",
    "firstKeptEntryId": "abc123",
    "tokensBefore": 150000,
    "estimatedTokensAfter": 32000,
    "usage": {
      "input": 32000,
      "output": 1200,
      "cacheRead": 0,
      "cacheWrite": 0,
      "totalTokens": 33200,
      "cost": {"input": 0.01, "output": 0.02, "cacheRead": 0, "cacheWrite": 0, "total": 0.03}
    },
    "details": {}
  }
}
```

`estimatedTokensAfter` 是对压缩后立即重建的消息上下文进行的启发式估算，并非服务提供商给出的精确 Token 数。`usage` 报告生成摘要所用的一次或多次 LLM 调用；自定义压缩处理函数可以省略该字段。

#### set_auto_compaction

启用或禁用在上下文接近满载时自动压缩。

```json
{"type": "set_auto_compaction", "enabled": true}
```

响应：
```json
{"type": "response", "command": "set_auto_compaction", "success": true}
```

### 重试

#### set_auto_retry

启用或禁用在发生临时错误（过载、限流、5xx）时自动重试。

```json
{"type": "set_auto_retry", "enabled": true}
```

响应：
```json
{"type": "response", "command": "set_auto_retry", "success": true}
```

#### abort_retry

中止正在进行的重试：取消等待，并停止后续重试。

```json
{"type": "abort_retry"}
```

响应：
```json
{"type": "response", "command": "abort_retry", "success": true}
```

### Bash

#### bash

执行 Shell 命令，并将输出加入对话上下文。命令运行期间，输出以 `bash_execution_update` 事件流式传输；响应中包含最终结果。

```json
{"id": "req-1", "type": "bash", "command": "ls -la"}
```

提供 `id`，可以将流式 `bash_execution_update` 事件与该命令关联。

响应：
```json
{
  "id": "req-1",
  "type": "response",
  "command": "bash",
  "success": true,
  "data": {
    "output": "total 48\ndrwxr-xr-x ...",
    "exitCode": 0,
    "cancelled": false,
    "truncated": false
  }
}
```

如果输出被截断，响应中还会包含 `fullOutputPath`：
```json
{
  "type": "response",
  "command": "bash",
  "success": true,
  "data": {
    "output": "truncated output...",
    "exitCode": 0,
    "cancelled": false,
    "truncated": true,
    "fullOutputPath": "/tmp/pi-bash-abc123.log"
  }
}
```

**Bash 结果如何传给 LLM：**

`bash` 命令立即执行并返回 `BashResult`。在内部，系统会创建 `BashExecutionMessage` 并将其存入智能体的消息状态。

发送下一条 `prompt` 命令时，所有消息（包括 `BashExecutionMessage`）都会先经过转换，再发送给 LLM。`BashExecutionMessage` 会转换为以下格式的 `UserMessage`：

````
Ran `ls -la`
```
total 48
drwxr-xr-x ...
```
````

这意味着：

1. Bash 输出会在**下一条提示词**中进入 LLM 上下文，而不是立即进入
2. 可以在发送提示词前执行多条 Bash 命令；所有输出都会进入上下文

#### abort_bash

中止正在运行的 Bash 命令。

```json
{"type": "abort_bash"}
```

响应：
```json
{"type": "response", "command": "abort_bash", "success": true}
```

### 会话

#### get_session_stats

获取 Token 用量、费用统计和当前上下文窗口用量。

```json
{"type": "get_session_stats"}
```

响应：
```json
{
  "type": "response",
  "command": "get_session_stats",
  "success": true,
  "data": {
    "sessionFile": "/path/to/session.jsonl",
    "sessionId": "abc123",
    "userMessages": 5,
    "assistantMessages": 5,
    "toolCalls": 12,
    "toolResults": 12,
    "totalMessages": 22,
    "tokens": {
      "input": 50000,
      "output": 10000,
      "cacheRead": 40000,
      "cacheWrite": 5000,
      "total": 105000
    },
    "cost": 0.45,
    "contextUsage": {
      "tokens": 60000,
      "contextWindow": 200000,
      "percent": 30
    }
  }
}
```

`tokens` 和 `cost` 包含整个会话中的助手消息、工具报告的用量，以及生成上下文压缩摘要和分支摘要产生的用量。`contextUsage` 包含用于上下文压缩和页脚显示的当前上下文窗口实际估算值。

如果模型或上下文窗口不可用，则省略 `contextUsage`。上下文压缩刚完成时，`contextUsage.tokens` 和 `contextUsage.percent` 为 `null`，直到压缩后新的助手响应提供有效的用量数据。

#### export_html

将会话导出为 HTML 文件。

```json
{"type": "export_html"}
```

指定自定义路径：
```json
{"type": "export_html", "outputPath": "/tmp/session.html"}
```

响应：
```json
{
  "type": "response",
  "command": "export_html",
  "success": true,
  "data": {"path": "/tmp/session.html"}
}
```

#### switch_session

加载另一个会话文件。`session_before_switch` 扩展事件处理函数可以取消该操作。

```json
{"type": "switch_session", "sessionPath": "/path/to/session.jsonl"}
```

响应：
```json
{"type": "response", "command": "switch_session", "success": true, "data": {"cancelled": false}}
```

如果扩展取消了切换：
```json
{"type": "response", "command": "switch_session", "success": true, "data": {"cancelled": true}}
```

#### fork

从当前分支中较早的一条用户消息派生新会话。`session_before_fork` 扩展事件处理函数可以取消该操作。响应会返回分支起点消息的文本。

```json
{"type": "fork", "entryId": "abc123"}
```

响应：
```json
{
  "type": "response",
  "command": "fork",
  "success": true,
  "data": {"text": "The original prompt text...", "cancelled": false}
}
```

如果扩展取消了派生操作：
```json
{
  "type": "response",
  "command": "fork",
  "success": true,
  "data": {"text": "The original prompt text...", "cancelled": true}
}
```

#### clone

在当前位置将当前活动分支复制为新会话。`session_before_fork` 扩展事件处理函数可以取消该操作。

```json
{"type": "clone"}
```

响应：
```json
{
  "type": "response",
  "command": "clone",
  "success": true,
  "data": {"cancelled": false}
}
```

如果扩展取消了克隆：
```json
{
  "type": "response",
  "command": "clone",
  "success": true,
  "data": {"cancelled": true}
}
```

#### get_fork_messages

获取可作为派生起点的用户消息。

```json
{"type": "get_fork_messages"}
```

响应：
```json
{
  "type": "response",
  "command": "get_fork_messages",
  "success": true,
  "data": {
    "messages": [
      {"entryId": "abc123", "text": "First prompt..."},
      {"entryId": "def456", "text": "Second prompt..."}
    ]
  }
}
```

#### get_entries

按追加顺序获取全部会话条目（不含会话文件头）。会话是由稳定 ID 组成的仅追加条目树，因此条目 ID 可以作为持久游标：将最后看到的条目 ID 作为 `since` 传入，即使客户端已经重启，也只会获取严格位于该条目之后的内容。与 `get_messages` 不同，该命令还包括压缩前的历史记录和已经放弃的分支。

```json
{"type": "get_entries"}
```

使用游标：
```json
{"type": "get_entries", "since": "abc123"}
```

响应：
```json
{
  "type": "response",
  "command": "get_entries",
  "success": true,
  "data": {
    "entries": [
      {"type": "message", "id": "def456", "parentId": "abc123", "timestamp": "...", "message": {"role": "user", "...": "..."}}
    ],
    "leafId": "def456"
  }
}
```

`leafId` 是当前叶条目的 ID（空会话为 `null`），因此客户端只需一次往返即可判断活动分支是否发生变化。如果 `since` 不匹配任何条目 ID，则响应中的 `success` 为 `false`。

#### get_tree

以条目树的形式获取会话。每个节点的结构为 `{entry, children, label?, labelTimestamp?}`。格式正确的会话只有一个根节点；孤立条目（父链已断裂）也会作为根节点出现。

```json
{"type": "get_tree"}
```

响应：
```json
{
  "type": "response",
  "command": "get_tree",
  "success": true,
  "data": {
    "tree": [
      {
        "entry": {"type": "message", "id": "abc123", "parentId": null, "...": "..."},
        "children": [
          {"entry": {"type": "message", "id": "def456", "parentId": "abc123", "...": "..."}, "children": []}
        ]
      }
    ],
    "leafId": "def456"
  }
}
```

#### get_last_assistant_text

获取最后一条助手消息的文本内容。

```json
{"type": "get_last_assistant_text"}
```

响应：
```json
{
  "type": "response",
  "command": "get_last_assistant_text",
  "success": true,
  "data": {"text": "The assistant's response..."}
}
```

如果不存在助手消息，则返回 `{"text": null}`。

#### set_session_name

设置当前会话的显示名称。该名称会出现在会话列表中，便于识别会话。

```json
{"type": "set_session_name", "name": "my-feature-work"}
```

响应：
```json
{
  "type": "response",
  "command": "set_session_name",
  "success": true
}
```

可通过 `get_state` 返回的 `sessionName` 字段获取当前会话名称。要在启动 RPC 模式时设置初始名称，请向 `pi --mode rpc` 进程传入 `--name <name>` 或 `-n <name>`。

### 命令

#### get_commands

获取可用命令（扩展命令、提示词模板和技能）。在命令名称前添加 `/`，即可通过 `prompt` 命令调用。

```json
{"type": "get_commands"}
```

响应：
```json
{
  "type": "response",
  "command": "get_commands",
  "success": true,
  "data": {
    "commands": [
      {"name": "session-name", "description": "Set or clear session name", "source": "extension", "path": "/home/user/.pi/agent/extensions/session.ts"},
      {"name": "fix-tests", "description": "Fix failing tests", "source": "prompt", "location": "project", "path": "/home/user/myproject/.pi/agent/prompts/fix-tests.md"},
      {"name": "skill:brave-search", "description": "Web search via Brave API", "source": "skill", "location": "user", "path": "/home/user/.pi/agent/skills/brave-search/SKILL.md"}
    ]
  }
}
```

每条命令包含：

- `name`：命令名称（使用 `/name` 调用）
- `description`：便于阅读的说明（扩展命令可以省略）
- `source`：命令类型：
  - `"extension"`：由扩展中的 `pi.registerCommand()` 注册
  - `"prompt"`：从提示词模板 `.md` 文件加载
  - `"skill"`：从技能目录加载（名称以 `skill:` 为前缀）
- `location`：加载位置（可选；扩展命令没有此字段）：
  - `"user"`：用户级（`~/.pi/agent/`）
  - `"project"`：项目级（`./.pi/agent/`）
  - `"path"`：通过 CLI 或设置显式指定的路径
- `path`：命令来源文件的绝对路径（可选）

**注意：** 其中不包含内置 TUI 命令（`/settings`、`/hotkeys` 等）。这些命令只在交互模式中处理，通过 `prompt` 发送时不会执行。

## 事件

智能体运行期间，事件会以 JSONL 形式流式写入标准输出。事件通常不包含 `id` 字段；如果来源 `bash` 命令提供了 `id`，则 `bash_execution_update` 会包含该 `id`。

### 事件类型

| 事件 | 说明 |
|-------|-------------|
| `agent_start` | 智能体开始处理 |
| `agent_end` | 一次底层智能体运行完成（后续仍可能重试、压缩或处理队列中的消息） |
| `agent_settled` | 智能体运行完全结束；不再有自动重试、压缩重试或队列中的后续操作 |
| `turn_start` | 新轮次开始 |
| `turn_end` | 轮次完成（包含助手消息和工具结果） |
| `message_start` | 消息开始 |
| `message_update` | 流式更新（文本/思考/工具调用增量） |
| `message_end` | 消息完成 |
| `bash_execution_update` | 直接 RPC Bash 命令的输出片段 |
| `tool_execution_start` | 工具开始执行 |
| `tool_execution_update` | 工具执行进度（流式输出） |
| `tool_execution_end` | 工具执行完成 |
| `queue_update` | 待处理的引导/后续消息队列发生变化 |
| `compaction_start` | 上下文压缩开始 |
| `compaction_end` | 上下文压缩完成 |
| `auto_retry_start` | 自动重试开始（发生临时错误后） |
| `auto_retry_end` | 自动重试完成（成功或最终失败） |
| `summarization_retry_scheduled` | 因上下文压缩或分支摘要生成发生临时错误，已安排重试 |
| `summarization_retry_attempt_start` | 重试生成摘要的请求开始 |
| `summarization_retry_finished` | 摘要重试循环完成 |
| `extension_error` | 扩展抛出错误 |

### agent_start

智能体开始处理提示词时触发。

```json
{"type": "agent_start"}
```

### agent_end

一次底层智能体运行完成时触发。事件包含本次运行产生的所有消息。如果 `willRetry` 为 `true`，随后会自动重试。

```json
{
  "type": "agent_end",
  "messages": [...],
  "willRetry": false
}
```

### agent_settled

整个会话级运行完全结束后触发。此时 Pi 不会再自动进行重试、压缩重试或处理队列中的后续消息。

```json
{"type": "agent_settled"}
```

### turn_start / turn_end

一个轮次由一次助手响应，以及由此产生的工具调用和结果组成。

```json
{"type": "turn_start"}
```

```json
{
  "type": "turn_end",
  "message": {...},
  "toolResults": [...]
}
```

### message_start / message_end

消息开始和完成时触发。`message` 字段包含一个 `AgentMessage`。

```json
{"type": "message_start", "message": {...}}
{"type": "message_end", "message": {...}}
```

### message_update（流式）

流式传输助手消息期间触发，同时包含尚未完成的消息和流式增量事件。

```json
{
  "type": "message_update",
  "message": {...},
  "assistantMessageEvent": {
    "type": "text_delta",
    "contentIndex": 0,
    "delta": "Hello ",
    "partial": {...}
  }
}
```

`assistantMessageEvent` 字段包含以下某一种增量类型：

| 类型 | 说明 |
|------|-------------|
| `start` | 消息生成开始 |
| `text_start` | 文本内容块开始 |
| `text_delta` | 文本内容片段 |
| `text_end` | 文本内容块结束 |
| `thinking_start` | 思考块开始 |
| `thinking_delta` | 思考内容片段 |
| `thinking_end` | 思考块结束 |
| `toolcall_start` | 工具调用开始 |
| `toolcall_delta` | 工具调用参数片段 |
| `toolcall_end` | 工具调用结束（包含完整的 `toolCall` 对象） |
| `done` | 消息完成（原因：`"stop"`、`"length"`、`"toolUse"`） |
| `error` | 发生错误（原因：`"aborted"`、`"error"`） |

流式文本响应示例：
```json
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_start","contentIndex":0,"partial":{...}}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":"Hello","partial":{...}}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_delta","contentIndex":0,"delta":" world","partial":{...}}}
{"type":"message_update","message":{...},"assistantMessageEvent":{"type":"text_end","contentIndex":0,"content":"Hello world","partial":{...}}}
```

### bash_execution_update

直接执行的 `bash` 命令每产生一个输出片段，就会触发一次该事件。`id` 与命令的 `id` 相同，使客户端能够将输出关联到正确的命令。

命令运行期间，事件会流式传输全部输出，即使最终 `bash` 响应中的 `output` 已被截断。

```json
{
  "type": "bash_execution_update",
  "id": "req-1",
  "delta": "total 48\n"
}
```

### tool_execution_start / tool_execution_update / tool_execution_end

工具开始执行、流式报告进度和完成执行时，分别触发这些事件。

```json
{
  "type": "tool_execution_start",
  "toolCallId": "call_abc123",
  "toolName": "bash",
  "args": {"command": "ls -la"}
}
```

执行期间，`tool_execution_update` 事件会流式传输部分结果，例如实时收到的 Bash 输出：

```json
{
  "type": "tool_execution_update",
  "toolCallId": "call_abc123",
  "toolName": "bash",
  "args": {"command": "ls -la"},
  "partialResult": {
    "content": [{"type": "text", "text": "partial output so far..."}],
    "details": {"truncation": null, "fullOutputPath": null}
  }
}
```

完成时：

```json
{
  "type": "tool_execution_end",
  "toolCallId": "call_abc123",
  "toolName": "bash",
  "result": {
    "content": [{"type": "text", "text": "total 48\n..."}],
    "details": {...}
  },
  "isError": false
}
```

使用 `toolCallId` 关联事件。`tool_execution_update` 中的 `partialResult` 包含截至当前累积的全部输出，而不只是本次增量，因此客户端每次更新时直接替换显示内容即可。

### queue_update

待处理的引导消息或后续消息队列发生变化时触发。

```json
{
  "type": "queue_update",
  "steering": ["Focus on error handling"],
  "followUp": ["After that, summarize the result"]
}
```

### compaction_start / compaction_end

无论手动还是自动执行上下文压缩，都会触发这些事件。

```json
{"type": "compaction_start", "reason": "threshold"}
```

`reason` 字段为 `"manual"`、`"threshold"` 或 `"overflow"`。

```json
{
  "type": "compaction_end",
  "reason": "threshold",
  "result": {
    "summary": "Summary of conversation...",
    "firstKeptEntryId": "abc123",
    "tokensBefore": 150000,
    "estimatedTokensAfter": 32000,
    "usage": {
      "input": 32000,
      "output": 1200,
      "cacheRead": 0,
      "cacheWrite": 0,
      "totalTokens": 33200,
      "cost": {"input": 0.01, "output": 0.02, "cacheRead": 0, "cacheWrite": 0, "total": 0.03}
    },
    "details": {}
  },
  "aborted": false,
  "willRetry": false
}
```

如果 `reason` 为 `"overflow"` 且压缩成功，`willRetry` 会为 `true`，智能体将自动重试提示词。

如果压缩被中止，`result` 为 `null`，`aborted` 为 `true`。

如果压缩失败（例如超出 API 配额），`result` 为 `null`，`aborted` 为 `false`，`errorMessage` 包含错误说明。

### auto_retry_start / auto_retry_end

发生临时错误（过载、限流、5xx）并触发自动重试时产生。

```json
{
  "type": "auto_retry_start",
  "attempt": 1,
  "maxAttempts": 3,
  "delayMs": 2000,
  "errorMessage": "529 {\"type\":\"error\",\"error\":{\"type\":\"overloaded_error\",\"message\":\"Overloaded\"}}"
}
```

```json
{
  "type": "auto_retry_end",
  "success": true,
  "attempt": 2
}
```

最终失败（超过最大重试次数）时：
```json
{
  "type": "auto_retry_end",
  "success": false,
  "attempt": 3,
  "finalError": "529 overloaded_error: Overloaded"
}
```

### summarization_retry_scheduled / summarization_retry_attempt_start / summarization_retry_finished

生成上下文压缩摘要或分支摘要时，如果服务提供商发生临时错误并触发重试，就会产生这些事件。这些事件采用与助手轮次自动重试相同的重试设置。

```json
{
  "type": "summarization_retry_scheduled",
  "attempt": 1,
  "maxAttempts": 3,
  "delayMs": 2000,
  "errorMessage": "terminated"
}
```

```json
{
  "type": "summarization_retry_attempt_start",
  "source": "compaction",
  "reason": "threshold"
}
```

对于分支摘要，`source` 为 `"branchSummary"`，且不存在 `reason`。

```json
{
  "type": "summarization_retry_finished"
}
```

### extension_error

扩展抛出错误时产生。

```json
{
  "type": "extension_error",
  "extensionPath": "/path/to/extension.ts",
  "event": "tool_call",
  "error": "Error message..."
}
```

<a id="extension-ui-protocol"></a>

## 扩展 UI 协议

扩展可以通过 `ctx.ui.select()`、`ctx.ui.confirm()` 等方法请求用户交互。在 RPC 模式中，这些调用会转换为建立在基础命令/事件流之上的请求/响应子协议。

扩展 UI 方法分为两类：

- **对话框方法**（`select`、`confirm`、`input`、`editor`）：向标准输出发送 `extension_ui_request`，然后阻塞，直到客户端通过标准输入发回 `id` 匹配的 `extension_ui_response`。
- **发送后不等待的方法**（`notify`、`setStatus`、`setWidget`、`setTitle`、`set_editor_text`）：向标准输出发送 `extension_ui_request`，但不等待响应。客户端可以显示或忽略这些信息。

如果对话框方法包含 `timeout` 字段，超时后智能体一侧会自动使用默认值结束等待。客户端无需自行跟踪超时。

部分 `ExtensionUIContext` 方法需要直接访问 TUI，因此在 RPC 模式中不受支持或功能有所退化：

- `custom()` 返回 `undefined`
- `setWorkingMessage()`、`setWorkingIndicator()`、`setFooter()`、`setHeader()`、`setEditorComponent()`、`setToolsExpanded()` 不执行任何操作
- `getEditorText()` 返回 `""`
- `getToolsExpanded()` 返回 `false`
- `pasteToEditor()` 委托给 `setEditorText()`（不处理粘贴或折叠）
- `getAllThemes()` 返回 `[]`
- `getTheme()` 返回 `undefined`
- `setTheme()` 返回 `{ success: false, error: "..." }`

注意：在 RPC 模式中，`ctx.mode` 为 `"rpc"`，`ctx.hasUI` 为 `true`，因为对话框方法和发送后不等待的方法可以通过扩展 UI 子协议正常工作。对于 `custom()` 这类需要真实终端的 TUI 特有功能，请使用 `ctx.mode === "tui"` 进行保护。

### 扩展 UI 请求（标准输出）

所有请求都包含 `type: "extension_ui_request"`、唯一的 `id` 和 `method` 字段。

#### select

提示用户从列表中选择。对话框方法的 `timeout` 字段以毫秒为单位；如果客户端未及时响应，智能体会自动以 `undefined` 结束等待。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-1",
  "method": "select",
  "title": "Allow dangerous command?",
  "options": ["Allow", "Block"],
  "timeout": 10000
}
```

预期响应：包含 `value`（选中的选项字符串）或 `cancelled: true` 的 `extension_ui_response`。

#### confirm

提示用户进行“是/否”确认。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-2",
  "method": "confirm",
  "title": "Clear session?",
  "message": "All messages will be lost.",
  "timeout": 5000
}
```

预期响应：包含 `confirmed: true/false` 或 `cancelled: true` 的 `extension_ui_response`。

#### input

提示用户输入自由格式文本。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-3",
  "method": "input",
  "title": "Enter a value",
  "placeholder": "type something..."
}
```

预期响应：包含 `value`（输入的文本）或 `cancelled: true` 的 `extension_ui_response`。

#### editor

打开多行文本编辑器，可以预先填入内容。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-4",
  "method": "editor",
  "title": "Edit some text",
  "prefill": "Line 1\nLine 2\nLine 3"
}
```

预期响应：包含 `value`（编辑后的文本）或 `cancelled: true` 的 `extension_ui_response`。

#### notify

显示通知。发送后不等待响应。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-5",
  "method": "notify",
  "message": "Command blocked by user",
  "notifyType": "warning"
}
```

`notifyType` 字段为 `"info"`、`"warning"` 或 `"error"`。省略时默认为 `"info"`。

#### setStatus

设置或清除页脚/状态栏中的状态条目。发送后不等待响应。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-6",
  "method": "setStatus",
  "statusKey": "my-ext",
  "statusText": "Turn 3 running..."
}
```

发送 `statusText: undefined`（或省略该字段）可以清除对应键的状态条目。

#### setWidget

设置或清除显示在编辑器上方或下方的小组件（由多行文本组成）。发送后不等待响应。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-7",
  "method": "setWidget",
  "widgetKey": "my-ext",
  "widgetLines": ["--- My Widget ---", "Line 1", "Line 2"],
  "widgetPlacement": "aboveEditor"
}
```

发送 `widgetLines: undefined`（或省略该字段）可以清除小组件。`widgetPlacement` 字段为 `"aboveEditor"`（默认值）或 `"belowEditor"`。RPC 模式只支持字符串数组，组件工厂函数会被忽略。

#### setTitle

设置终端窗口/标签页标题。发送后不等待响应。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-8",
  "method": "setTitle",
  "title": "pi - my project"
}
```

#### set_editor_text

设置输入编辑器中的文本。发送后不等待响应。

```json
{
  "type": "extension_ui_request",
  "id": "uuid-9",
  "method": "set_editor_text",
  "text": "prefilled text for the user"
}
```

### 扩展 UI 响应（标准输入）

只有对话框方法（`select`、`confirm`、`input`、`editor`）需要发送响应。响应的 `id` 必须与请求匹配。

#### 值响应（select、input、editor）

```json
{"type": "extension_ui_response", "id": "uuid-1", "value": "Allow"}
```

#### 确认响应（confirm）

```json
{"type": "extension_ui_response", "id": "uuid-2", "confirmed": true}
```

#### 取消响应（任意对话框）

关闭任意对话框。扩展会收到 `undefined`（select/input/editor）或 `false`（confirm）。

```json
{"type": "extension_ui_response", "id": "uuid-3", "cancelled": true}
```

## 错误处理

命令失败时，返回包含 `success: false` 的响应：

```json
{
  "type": "response",
  "command": "set_model",
  "success": false,
  "error": "Model not found: invalid/model"
}
```

解析错误：

```json
{
  "type": "response",
  "command": "parse",
  "success": false,
  "error": "Failed to parse command: Unexpected token..."
}
```

<a id="message-types"></a>

## 类型

源文件：

- [`packages/ai/src/types.ts`](../../ai/src/types.ts)——`Model`、`UserMessage`、`AssistantMessage`、`ToolResultMessage`
- [`packages/agent/src/types.ts`](../../agent/src/types.ts)——`AgentMessage`、`AgentEvent`
- [`src/core/messages.ts`](../src/core/messages.ts)——`BashExecutionMessage`
- [`src/modes/rpc/rpc-types.ts`](../src/modes/rpc/rpc-types.ts)——RPC 命令/响应类型、扩展 UI 请求/响应类型

### Model

```json
{
  "id": "claude-sonnet-4-20250514",
  "name": "Claude Sonnet 4",
  "api": "anthropic-messages",
  "provider": "anthropic",
  "baseUrl": "https://api.anthropic.com",
  "reasoning": true,
  "input": ["text", "image"],
  "contextWindow": 200000,
  "maxTokens": 16384,
  "cost": {
    "input": 3.0,
    "output": 15.0,
    "cacheRead": 0.3,
    "cacheWrite": 3.75
  }
}
```

### UserMessage

```json
{
  "role": "user",
  "content": "Hello!",
  "timestamp": 1733234567890,
  "attachments": []
}
```

`content` 字段可以是字符串，也可以是由 `TextContent`/`ImageContent` 内容块组成的数组。

### AssistantMessage

```json
{
  "role": "assistant",
  "content": [
    {"type": "text", "text": "Hello! How can I help?"},
    {"type": "thinking", "thinking": "User is greeting me..."},
    {"type": "toolCall", "id": "call_123", "name": "bash", "arguments": {"command": "ls"}}
  ],
  "api": "anthropic-messages",
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",
  "usage": {
    "input": 100,
    "output": 50,
    "cacheRead": 0,
    "cacheWrite": 0,
    "cost": {"input": 0.0003, "output": 0.00075, "cacheRead": 0, "cacheWrite": 0, "total": 0.00105}
  },
  "stopReason": "stop",
  "timestamp": 1733234567890
}
```

停止原因：`"stop"`、`"length"`、`"toolUse"`、`"error"`、`"aborted"`

### ToolResultMessage

```json
{
  "role": "toolResult",
  "toolCallId": "call_123",
  "toolName": "bash",
  "content": [{"type": "text", "text": "total 48\ndrwxr-xr-x ..."}],
  "usage": {
    "input": 100,
    "output": 50,
    "cacheRead": 0,
    "cacheWrite": 0,
    "totalTokens": 150,
    "cost": {"input": 0.0003, "output": 0.00075, "cacheRead": 0, "cacheWrite": 0, "total": 0.00105}
  },
  "isError": false,
  "timestamp": 1733234567890
}
```

`usage` 可选，用于报告工具内部执行的 LLM 工作。存在时，其用量会计入会话的 Token 和费用总计。

### BashExecutionMessage

由 `bash` RPC 命令创建，而不是由 LLM 工具调用创建：

```json
{
  "role": "bashExecution",
  "command": "ls -la",
  "output": "total 48\ndrwxr-xr-x ...",
  "exitCode": 0,
  "cancelled": false,
  "truncated": false,
  "fullOutputPath": null,
  "timestamp": 1733234567890
}
```

### Attachment

```json
{
  "id": "img1",
  "type": "image",
  "fileName": "photo.jpg",
  "mimeType": "image/jpeg",
  "size": 102400,
  "content": "base64-encoded-data...",
  "extractedText": null,
  "preview": null
}
```

## 示例：基础客户端（Python）

```python
import subprocess
import json

proc = subprocess.Popen(
    ["pi", "--mode", "rpc", "--no-session"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    text=True
)

def send(cmd):
    proc.stdin.write(json.dumps(cmd) + "\n")
    proc.stdin.flush()

def read_events():
    for line in proc.stdout:
        yield json.loads(line)

# 发送提示词
send({"type": "prompt", "message": "Hello!"})

# 处理事件
for event in read_events():
    if event.get("type") == "message_update":
        delta = event.get("assistantMessageEvent", {})
        if delta.get("type") == "text_delta":
            print(delta["delta"], end="", flush=True)

    if event.get("type") == "agent_end":
        print()
        break
```

## 示例：交互式客户端（Node.js）

完整的交互式示例请参阅 [`test/rpc-example.ts`](../test/rpc-example.ts)；带类型的客户端实现请参阅 [`src/modes/rpc/rpc-client.ts`](../src/modes/rpc/rpc-client.ts)。

扩展 UI 协议的完整处理示例请参阅 [`examples/rpc-extension-ui.ts`](../examples/rpc-extension-ui.ts)，它与 [`examples/extensions/rpc-demo.ts`](../examples/extensions/rpc-demo.ts) 扩展配合使用。

```javascript
const { spawn } = require("child_process");
const { StringDecoder } = require("string_decoder");

const agent = spawn("pi", ["--mode", "rpc", "--no-session"]);

function attachJsonlReader(stream, onLine) {
    const decoder = new StringDecoder("utf8");
    let buffer = "";

    stream.on("data", (chunk) => {
        buffer += typeof chunk === "string" ? chunk : decoder.write(chunk);

        while (true) {
            const newlineIndex = buffer.indexOf("\n");
            if (newlineIndex === -1) break;

            let line = buffer.slice(0, newlineIndex);
            buffer = buffer.slice(newlineIndex + 1);
            if (line.endsWith("\r")) line = line.slice(0, -1);
            onLine(line);
        }
    });

    stream.on("end", () => {
        buffer += decoder.end();
        if (buffer.length > 0) {
            onLine(buffer.endsWith("\r") ? buffer.slice(0, -1) : buffer);
        }
    });
}

attachJsonlReader(agent.stdout, (line) => {
    const event = JSON.parse(line);

    if (event.type === "message_update") {
        const { assistantMessageEvent } = event;
        if (assistantMessageEvent.type === "text_delta") {
            process.stdout.write(assistantMessageEvent.delta);
        }
    }
});

// 发送提示词
agent.stdin.write(JSON.stringify({ type: "prompt", message: "Hello" }) + "\n");

// 按 Ctrl+C 中止
process.on("SIGINT", () => {
    agent.stdin.write(JSON.stringify({ type: "abort" }) + "\n");
});
```
