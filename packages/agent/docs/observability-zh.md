<!-- 从 jot qe0ikdqs 同步而来。后续请直接在仓库内编辑此文件。 -->

# Pi 可观测性设计笔记

## 目标

让 `packages/ai` 和 `packages/agent`/运行框架具备可观测性，同时不依赖 OpenTelemetry、Sentry 或任何 APM 厂商。

Pi 应发出稳定、结构化的生命周期事件。外部监听器可以将这些事件转换为 OTel span、Sentry span、日志、指标或自定义遥测数据。

## 心智模型

一条 trace 表示一棵具有因果关系的工作树，例如用户的一轮交互。

一个 span 表示树中一次有计时的操作。它通常通过 ID 而非对象指针来表示：

```ts
interface SpanRecord {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  name: string;
  startTime: number;
  endTime?: number;
  attributes: Record<string, unknown>;
  status: "ok" | "error";
}
```

示例树：

```text
traceId=t1 spanId=s1 parent=-  name=pi.agent.prompt
traceId=t1 spanId=s2 parent=s1 name=pi.agent.turn
traceId=t1 spanId=s3 parent=s2 name=pi.ai.provider.request
traceId=t1 spanId=s4 parent=s2 name=pi.agent.tool_call
traceId=t1 spanId=s5 parent=s4 name=pi.session.append_entry
```

## 异步上下文

JavaScript 只有一个事件循环，但多条异步调用链可能交错执行。并发情况下，单个全局 `currentContext` 无法正确工作。

`AsyncLocalStorage` 相当于 Node 为异步延续提供的 `ThreadLocal`。它能让并发操作各自保有独立的当前上下文：

```ts
await Promise.all([
  runWithPiContext({ userId: "alice" }, () => harness.prompt("A")),
  runWithPiContext({ userId: "bob" }, () => harness.prompt("B")),
]);
```

这样一来，深层代码也能读取当前活动异步调用链所对应的正确上下文。

Pi 必须能在 Node、Bun、浏览器、worker 及其他 JavaScript 运行时中运行，因此 ALS 不能作为核心抽象，而应作为一种运行时适配器。

## 核心设计

Pi 自身提供一个精简且不依赖具体运行时的可观测性抽象：

```ts
export interface PiObservabilityContext {
  traceId?: string;
  currentSpanId?: string;
  userContext?: Record<string, unknown>;
}

export interface PiObservabilityEvent {
  type: "start" | "end" | "error" | "event";
  name: string;
  traceId: string;
  spanId?: string;
  parentSpanId?: string;
  timestamp: number;
  durationMs?: number;
  context?: Record<string, unknown>;
  payload?: Record<string, unknown>;
  error?: { name: string; message: string };
}

export interface PiObservability {
  getContext(): PiObservabilityContext | undefined;
  runWithContext<T>(context: PiObservabilityContext, fn: () => T): T;
  emit(event: PiObservabilityEvent): void;
  hasSubscribers(): boolean;
}
```

公开 API：

```ts
export function configurePiObservability(observability: PiObservability): void;
export function subscribePiObservability(listener: (event: PiObservabilityEvent) => void): () => void;
export function runWithPiContext<T>(userContext: Record<string, unknown>, fn: () => T): T;
export function traceOperation<T>(name: string, payload: Record<string, unknown>, fn: () => T): T;
```

`traceOperation()` 的执行过程：

1. 读取当前上下文
2. 如果缺少 `traceId`，则创建一个
3. 创建新的 `spanId`
4. 将当前 span 作为 `parentSpanId`
5. 发出 `start`
6. 在子上下文中运行回调
7. 发出 `end` 或 `error`
8. 出错时重新抛出错误

伪代码：

```ts
function traceOperation<T>(name: string, payload: Record<string, unknown>, fn: () => T): T {
  const parent = getContext();
  const traceId = parent?.traceId ?? createId();
  const spanId = createId();
  const parentSpanId = parent?.currentSpanId;

  const child = { ...parent, traceId, currentSpanId: spanId };

  emit({ type: "start", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: parent?.userContext, payload });

  return runWithContext(child, () => {
    try {
      const result = fn();
      // 支持 Promise 的实现会在 Promise 敲定后发出 end/error。
      emit({ type: "end", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: child.userContext, payload });
      return result;
    } catch (error) {
      emit({ type: "error", name, traceId, spanId, parentSpanId, timestamp: Date.now(), context: child.userContext, payload, error: serializeError(error) });
      throw error;
    }
  });
}
```

## 运行时适配器

核心包不应导入 Node 专用 API。

可选实现：

- Node 适配器：使用 `AsyncLocalStorage` 管理上下文，并可选择通过 `diagnostics_channel` 发布事件。
- 浏览器/worker 后备方案：使用本地订阅者集合，并进行有限或手动的上下文传播。
- Bun/Deno 适配器：如果运行时提供专用的异步上下文机制，则使用该机制。

在 Node 中，可以将诊断通道用作被动事件总线：

```ts
import { channel } from "diagnostics_channel";
channel("pi.observability").publish(event);
```

订阅者无需对 Pi 进行 monkey patch，即可创建 OTel/Sentry span。

## Pi 发出的内容

Pi 只发出所发生事件的信息，不直接创建 OTel/Sentry span。

最初使用的最小事件名称集合：

```text
pi.agent.prompt
pi.agent.skill
pi.agent.prompt_template
pi.agent.compaction
pi.agent.branch_navigation
pi.agent.session.append_entry
pi.ai.provider.request
```

每项操作都会发出：

```text
start
end
error
```

后续可增加：

```text
pi.agent.turn
pi.agent.tool_call
pi.agent.queue_update
pi.ai.provider.retry
pi.ai.provider.first_token
pi.ai.provider.usage
pi.session.read
pi.session.write
```

## 最小插桩点

### packages/agent

封装以下方法：

- `AgentHarness.prompt()`
- `AgentHarness.skill()`
- `AgentHarness.promptFromTemplate()`
- `AgentHarness.compact()`
- `AgentHarness.navigateTree()`
- `Session.appendTypedEntry()` 或存储追加门面

示例：

```ts
return traceOperation(
  "pi.agent.prompt",
  {
    sessionId: turnState.sessionId,
    provider: turnState.model.provider,
    model: turnState.model.id,
    promptLength: text.length,
    imageCount: options?.images?.length ?? 0,
  },
  () => this.executeTurn(turnState, text, options),
);
```

会话写入：

```ts
return traceOperation(
  "pi.agent.session.append_entry",
  { entryType: entry.type },
  async () => {
    await this.unwrap(this.storage.appendEntry(entry));
    return entry.id;
  },
);
```

### packages/ai

封装服务提供商的公共调用边界：

- `streamSimple()`
- `completeSimple()`

示例：

```ts
return traceOperation(
  "pi.ai.provider.request",
  {
    api: model.api,
    provider: model.provider,
    model: model.id,
    sessionId: options.sessionId,
    reasoning: options.reasoning,
  },
  () => actualStreamSimple(model, context, options),
);
```

结束/错误事件的载荷可以包含安全的元数据：

- 停止原因
- 状态码
- 重试次数
- 输入/输出/总 token 数
- 总成本
- 中止/超时标志

## 安全与脱敏

默认载荷必须是安全的。

默认可安全记录：

- 服务提供商
- 模型
- API 标识符
- 会话 ID
- 条目类型
- 工具名称
- 状态码
- 停止原因
- token 数量
- 成本
- 耗时

默认不应记录：

- 提示词
- 补全文本
- 工具参数
- 工具结果
- shell 输出
- 文件内容
- 服务提供商请求载荷
- 服务提供商响应正文
- API 密钥
- 请求头

以后可以提供内容采集功能，但必须由用户主动启用，并配合显式的脱敏钩子。

## 监听器行为

可观测性功能绝不能影响 Pi 的执行。

订阅者错误应被忽略或隔离。运行框架钩子属于控制平面，可能影响执行；可观测性订阅者则是被动的，不能影响执行。

## 用户上下文

用户可以为某轮交互关联任意上下文：

```ts
await runWithPiContext(
  {
    userId: "u123",
    orgId: "acme",
    region: "eu",
  },
  () => harness.prompt("fix this"),
);
```

该异步调用链内发出的每个事件都会包含此上下文：

```ts
{
  type: "start",
  name: "pi.ai.provider.request",
  traceId: "t1",
  spanId: "s3",
  parentSpanId: "s1",
  context: {
    userId: "u123",
    orgId: "acme",
    region: "eu",
  },
  payload: {
    provider: "anthropic",
    model: "claude-sonnet-4",
  },
}
```

OTel 适配器可以将其映射为 span 属性；Sentry 适配器可以将其映射为 Sentry 上下文/span；用户也可以自行将其记录为 JSON。

## 包的组织方式

最初所需的最小包：

```text
packages/observability
  不依赖具体运行时的上下文 + traceOperation + subscribe
```

然后：

```text
packages/ai
  发出 pi.ai.* 事件

packages/agent
  发出 pi.agent.* / pi.session.* 事件
```

以后可选增加：

```text
packages/observability-node
  AsyncLocalStorage + diagnostics_channel 桥接

packages/otel
  订阅 Pi 事件并创建 OpenTelemetry span
```

## 核心主张

Pi 定义稳定、安全的事件契约，适配器决定事件的去向。

这样既能让 AI 包和运行框架具备可观测性，又不会使核心包与 OTel、Sentry、Node 专用 API 或 monkey patch 绑定。
