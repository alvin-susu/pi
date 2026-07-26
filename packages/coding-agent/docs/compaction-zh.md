# 上下文压缩与分支摘要

LLM 的上下文窗口有限。对话过长时，Pi 会通过上下文压缩总结较早的内容，同时保留近期工作。本文同时介绍自动上下文压缩和分支摘要。

**源文件**（[pi-mono](https://github.com/earendil-works/pi-mono)）：

- [`packages/coding-agent/src/core/compaction/compaction.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)——自动上下文压缩逻辑
- [`packages/coding-agent/src/core/compaction/branch-summarization.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)——分支摘要
- [`packages/coding-agent/src/core/compaction/utils.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/utils.ts)——共享工具（文件跟踪、序列化）
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts)——条目类型（`CompactionEntry`、`BranchSummaryEntry`）
- [`packages/coding-agent/src/core/extensions/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts)——扩展事件类型

要查看项目中的 TypeScript 定义，请检查 `node_modules/@earendil-works/pi-coding-agent/dist/`。

## 概览

Pi 提供两种摘要机制：

| 机制 | 触发条件 | 用途 |
|-----------|---------|---------|
| 上下文压缩 | 上下文超过阈值，或执行 `/compact` | 总结旧消息，释放上下文空间 |
| 分支摘要 | 使用 `/tree` 导航 | 切换分支时保留原分支的上下文 |

两种机制使用相同的结构化摘要格式，并以累积方式跟踪文件操作。上下文压缩和分支摘要请求会使用新的路由会话 ID；如果服务提供商支持，还会禁用提示词缓存写入，因为这类一次性提示词几乎不会被复用。

## 上下文压缩

### 触发时机

满足以下条件时触发自动上下文压缩：

```
contextTokens > contextWindow - reserveTokens
```

`reserveTokens` 默认为 16384 个 Token（可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置），用于为 LLM 回复预留空间。

也可以使用 `/compact [instructions]` 手动触发，并通过可选指令指定摘要的关注点。

### 工作原理

1. **查找切分点**：从最新消息开始向前遍历并累加 Token 估算值，直到达到 `keepRecentTokens`（默认为 20k，可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）
2. **提取消息**：收集从上一个保留边界（或会话开始处）到切分点之间的消息
3. **生成摘要**：调用 LLM，以结构化格式生成摘要；如果存在上一次摘要，则将其作为迭代上下文传入
4. **追加条目**：保存包含摘要和 `firstKeptEntryId` 的 `CompactionEntry`
5. **重新加载**：重新加载会话，使用摘要和从 `firstKeptEntryId` 开始的消息

```
压缩前：

  entry:  0     1     2     3      4     5     6      7      8     9
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┘
                └────────┬───────┘ └──────────────┬──────────────┘
               messagesToSummarize              保留消息
                                   ↑
                          firstKeptEntryId (entry 4)

压缩后（追加新条目）：

  entry:  0     1     2     3      4     5     6      7      8     9     10
        ┌─────┬─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│ cmp │
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┴─────┘
               └──────────┬──────┘ └──────────────────────┬───────────────────┘
                   不发送给 LLM                       发送给 LLM
                                                         ↑
                                              starts from firstKeptEntryId

LLM 看到的内容：

  ┌────────┬─────────┬─────┬─────┬──────┬──────┬─────┬──────┐
  │ system │ summary │ usr │ ass │ tool │ tool │ ass │ tool │
  └────────┴─────────┴─────┴─────┴──────┴──────┴─────┴──────┘
       ↑         ↑      └─────────────────┬────────────────┘
    提示词   来自 cmp          从 firstKeptEntryId 开始的消息
```

反复执行压缩时，摘要范围从上一次压缩的保留边界（`firstKeptEntryId`）开始，而不是从压缩条目本身开始。如果在当前路径中找不到该保留条目，则回退到上一次压缩后的下一个条目。这样，之前压缩时保留下来的消息也会进入下一次摘要，从而继续得到保留。写入新的 `CompactionEntry` 之前，Pi 还会根据重建后的会话上下文重新计算 `tokensBefore`，使 Token 数准确反映此次实际替换的压缩前上下文。

### 拆分轮次

一个“轮次”从用户消息开始，包含此后所有助手回复和工具调用，直到下一条用户消息为止。通常，上下文压缩会在轮次边界处切分。

如果单个轮次超过 `keepRecentTokens`，切分点会落在轮次中间的某条助手消息处。这称为“拆分轮次”：

```
拆分轮次（单个超长轮次超过预算）：

  entry:  0     1     2      3     4      5      6     7      8
        ┌─────┬─────┬─────┬──────┬─────┬──────┬──────┬─────┬──────┐
        │ hdr │ usr │ ass │ tool │ ass │ tool │ tool │ ass │ tool │
        └─────┴─────┴─────┴──────┴─────┴──────┴──────┴─────┴──────┘
                ↑                                     ↑
         turnStartIndex = 1                  firstKeptEntryId = 7
                │                                     │
                └──── turnPrefixMessages (1-6) ───────┘
                                                      └── 保留 (7-8)

  isSplitTurn = true
  messagesToSummarize = []  （之前没有完整轮次）
  turnPrefixMessages = [usr, ass, tool, ass, tool, tool]
```

对于拆分轮次，Pi 会生成两份摘要并将其合并：

1. **历史摘要**：之前的上下文（如果存在）
2. **轮次前缀摘要**：被拆分轮次的前半部分

### 切分点规则

以下位置可以作为切分点：

- 用户消息
- 助手消息
- BashExecution 消息
- 自定义消息（custom_message、branch_summary）

绝不能在工具结果处切分，因为工具结果必须与对应的工具调用保存在一起。

### CompactionEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface CompactionEntry<T = unknown> {
  type: "compaction";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  usage?: Usage;       // 生成摘要时的 LLM 用量
  fromHook?: boolean;  // 由扩展提供时为 true（旧字段名）
  details?: T;         // 特定于实现的数据
}

// 默认上下文压缩使用以下 details（来自 compaction.ts）：
interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

扩展可以在 `details` 中存储任何可序列化为 JSON 的数据。默认上下文压缩会跟踪文件操作，但自定义扩展实现可以使用自己的结构。对于自动生成和由扩展提供的摘要，如果存在 LLM `usage`，都会将其保存下来，使会话总用量包含摘要生成工作。

具体实现请参阅 [`prepareCompaction()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) 和 [`compact()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)。以编程方式直接生成摘要时，`generateSummary()` 返回摘要文本，`generateSummaryWithUsage()` 返回 `{ text, usage }`。

## 分支摘要

### 触发时机

使用 `/tree` 导航到其他分支时，Pi 会询问是否总结刚刚离开的工作。生成的摘要会把原分支的上下文注入新分支。

### 工作原理

1. **查找共同祖先**：旧位置和新位置共有的最深节点
2. **收集条目**：从旧叶节点向上遍历到共同祖先
3. **按预算准备内容**：在 Token 预算内纳入消息，优先保留较新的消息
4. **生成摘要**：调用 LLM，以结构化格式生成摘要
5. **追加条目**：在导航目标位置保存 `BranchSummaryEntry`

```
导航前的会话树：

         ┌─ B ─ C ─ D（即将离开的旧叶节点）
    A ───┤
         └─ E ─ F（目标）

共同祖先：A
需要总结的条目：B、C、D

生成摘要并完成导航后：

         ┌─ B ─ C ─ D ─ [B、C、D 的摘要]
    A ───┤
         └─ E ─ F（新叶节点）
```

### 累积文件跟踪

上下文压缩和分支摘要都会累积跟踪文件。生成摘要时，Pi 会从以下位置提取文件操作：

- 被总结消息中的工具调用
- 以前的上下文压缩或分支摘要 `details`（如果存在）

因此，文件跟踪信息会跨多次上下文压缩或嵌套分支摘要不断累积，从而保留已读取和已修改文件的完整历史。

### BranchSummaryEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface BranchSummaryEntry<T = unknown> {
  type: "branch_summary";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  fromId: string;      // 导航前所在的条目
  usage?: Usage;       // 生成摘要时的 LLM 用量
  fromHook?: boolean;  // 由扩展提供时为 true（旧字段名）
  details?: T;         // 特定于实现的数据
}

// 默认分支摘要使用以下 details（来自 branch-summarization.ts）：
interface BranchSummaryDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

与上下文压缩相同，扩展可以在 `details` 中存储自定义数据。

具体实现请参阅 [`collectEntriesForBranchSummary()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)、[`prepareBranchEntries()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) 和 [`generateBranchSummary()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)。

## 摘要格式

上下文压缩和分支摘要使用相同的结构化格式。以下英文标题是实际摘要格式的一部分，因此保持原样：

```markdown
## Goal
[用户希望完成的目标]

## Constraints & Preferences
- [用户提出的要求]

## Progress
### Done
- [x] [已完成的任务]

### In Progress
- [ ] [当前工作]

### Blocked
- [存在的问题（如有）]

## Key Decisions
- **[决策]**：[理由]

## Next Steps
1. [接下来应执行的操作]

## Critical Context
- [继续工作所需的数据]

<read-files>
path/to/file1.ts
path/to/file2.ts
</read-files>

<modified-files>
path/to/changed.ts
</modified-files>
```

### 消息序列化

生成摘要之前，Pi 会通过 [`serializeConversation()`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/compaction/utils.ts) 将消息序列化为文本。以下角色标记是实际序列化格式的一部分，因此保持英文：

```
[User]: 用户输入的内容
[Assistant thinking]: 内部推理
[Assistant]: 回复文本
[Assistant tool calls]: read(path="foo.ts"); edit(path="bar.ts", ...)
[Tool result]: 工具输出
```

这样可以防止模型将其误认为需要继续的对话。

序列化期间，工具结果会被截断到 2000 个字符。超出限制的内容会替换为一个标记，说明截断了多少字符。工具结果（尤其是 `read` 和 `bash` 的输出）通常是上下文体积的主要来源，这种处理可以将摘要请求控制在合理的 Token 预算内。

## 通过扩展自定义摘要

扩展可以拦截和自定义上下文压缩与分支摘要。事件类型定义详见 [`extensions/types.ts`](https://github.com/earendil-works/pi-mono/blob/main/packages/coding-agent/src/core/extensions/types.ts)。

### session_before_compact

在自动上下文压缩或 `/compact` 之前触发。处理程序可以取消操作，或提供自定义摘要。参阅类型文件中的 `SessionBeforeCompactEvent` 和 `CompactionPreparation`。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, reason, willRetry, signal } = event;

  // preparation.messagesToSummarize - 需要总结的消息
  // preparation.turnPrefixMessages - 拆分轮次的前缀（当 isSplitTurn 时）
  // preparation.previousSummary - 上一次上下文压缩摘要
  // preparation.fileOps - 提取出的文件操作
  // preparation.tokensBefore - 压缩前的上下文 Token 数
  // preparation.firstKeptEntryId - 保留消息的起点
  // preparation.settings - 上下文压缩设置

  // branchEntries - 当前分支上的全部条目（供自定义状态使用）
  // reason - "manual"（/compact）、"threshold" 或 "overflow"
  // willRetry - 压缩后是否重试被中止的轮次（上下文溢出恢复）
  // signal - AbortSignal（传递给 LLM 调用）

  // 取消：
  return { cancel: true };

  // 自定义摘要：
  return {
    compaction: {
      summary: "Your summary...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      // usage: summaryResponse.usage, // 可选；会计入会话总用量
      details: { /* 自定义数据 */ },
    }
  };
});
```

#### 将消息转换为文本

要使用自己的模型生成摘要，请通过 `serializeConversation` 将消息转换为文本：

```typescript
import { convertToLlm, serializeConversation } from "@earendil-works/pi-coding-agent";

pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;

  // 将 AgentMessage[] 转换为 Message[]，再序列化为文本
  const conversationText = serializeConversation(
    convertToLlm(preparation.messagesToSummarize)
  );
  // 返回：
  // [User]: 消息文本
  // [Assistant thinking]: 思考内容
  // [Assistant]: 回复文本
  // [Assistant tool calls]: read(path="..."); bash(command="...")
  // [Tool result]: 输出文本

  // 现在将文本发送给自己的模型生成摘要
  const { summary, usage } = await myModel.summarize(conversationText);

  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      usage,
    }
  };
});
```

使用其他模型的完整示例请参阅 [custom-compaction.ts](../examples/extensions/custom-compaction.ts)。

### session_before_tree

在 `/tree` 导航之前触发。无论用户是否选择生成摘要，该事件都会触发。处理程序可以取消导航，或提供自定义摘要。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;

  // preparation.targetId - 导航目标
  // preparation.oldLeafId - 当前所在位置（即将离开）
  // preparation.commonAncestorId - 共同祖先
  // preparation.entriesToSummarize - 将要总结的条目
  // preparation.userWantsSummary - 用户是否选择生成摘要

  // 完全取消导航：
  return { cancel: true };

  // 提供自定义摘要（仅在 userWantsSummary 为 true 时使用）：
  if (preparation.userWantsSummary) {
    return {
      summary: {
        summary: "Your summary...",
        // usage: summaryResponse.usage, // 可选；会计入会话总用量
        details: { /* 自定义数据 */ },
      }
    };
  }
});
```

参阅类型文件中的 `SessionBeforeTreeEvent` 和 `TreePreparation`。

## 设置

在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置上下文压缩：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| 设置 | 默认值 | 说明 |
|---------|---------|-------------|
| `enabled` | `true` | 启用自动上下文压缩 |
| `reserveTokens` | `16384` | 为 LLM 回复保留的 Token 数 |
| `keepRecentTokens` | `20000` | 保留且不纳入摘要的近期 Token 数 |

使用 `"enabled": false` 可以禁用自动上下文压缩；之后仍可通过 `/compact` 手动压缩。
