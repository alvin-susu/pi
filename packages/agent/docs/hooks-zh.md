# AgentHarness 钩子设计

<!-- 从 jot 3utlzkxy 同步而来。后续请直接在仓库内编辑此文件。 -->

最终设计。

## 核心模型

事件通过仅存在于类型层面的幻影成员携带自身的结果类型：

```ts
declare const HookResult: unique symbol;

interface HookEvent<TType extends string, TResult = void> {
	type: TType;
	readonly [HookResult]?: TResult;
}

type ResultOf<E> = E extends { readonly [HookResult]?: infer R } ? R : void;

type HookHandler<E, Ctx> = (
	event: E,
	ctx: Ctx,
	signal?: AbortSignal,
) => ResultOf<E> | void | Promise<ResultOf<E> | void>;

type HookObserver<E, Ctx> = (
	event: E,
	ctx: Ctx,
	signal?: AbortSignal,
) => void | Promise<void>;
```

示例：

```ts
interface ContextEvent extends HookEvent<"context", { messages?: AgentMessage[] }> {
	type: "context";
	messages: AgentMessage[];
}

interface ToolCallEvent extends HookEvent<"tool_call", { block?: boolean; reason?: string }> {
	type: "tool_call";
	toolName: string;
	input: Record<string, unknown>;
}

interface MessageEndEvent extends HookEvent<"message_end"> {
	type: "message_end";
	message: AgentMessage;
}
```

无需结果映射，也无需规格表。每种事件类型自行定义其结果。

## 钩子接口

```ts
interface AgentHarnessHooks<E extends HookEvent<string, unknown>, Ctx> {
	context: Ctx;

	setContext(ctx: Ctx): void;

	observe(handler: HookObserver<E, Ctx>): () => void;

	on<TType extends E["type"]>(
		type: TType,
		handler: HookHandler<Extract<E, { type: TType }>, Ctx>,
	): () => void;

	emit<TEvent extends E>(
		event: TEvent,
		signal?: AbortSignal,
	): Promise<ResultOf<TEvent> | undefined>;

	addCleanup(cleanup: () => void | Promise<void>): () => void;

	clear(): Promise<void>;
	dispose(): Promise<void>;
}
```

需要注意的职责划分：

- `observe()` 能看到所有事件，但只能读取；其返回值会被忽略。
- `on(type, handler)` 参与对应事件的语义处理。
- `emit(event)` 是 `AgentHarness` 唯一调用的方法。
- `clear()` 移除观察器和处理器，并执行清理函数。

## 默认实现的内部结构

```ts
class DefaultAgentHarnessHooks<E extends HookEvent<string, unknown>, Ctx>
	implements AgentHarnessHooks<E, Ctx> {
	context: Ctx;

	private observers = new Set<HookObserver<E, Ctx>>();
	private handlers = new Map<string, Set<HookHandler<any, Ctx>>>();
	private cleanups = new Set<() => void | Promise<void>>();

	constructor(ctx: Ctx) {
		this.context = ctx;
	}

	setContext(ctx: Ctx): void {
		this.context = ctx;
	}

	observe(handler: HookObserver<E, Ctx>): () => void {
		this.observers.add(handler);
		return () => this.observers.delete(handler);
	}

	on(type, handler): () => void {
		let handlers = this.handlers.get(type);
		if (!handlers) {
			handlers = new Set();
			this.handlers.set(type, handlers);
		}
		handlers.add(handler);
		return () => handlers.delete(handler);
	}

	async emit(event, signal?) {
		for (const observer of this.observers) {
			await observer(event, this.context, signal);
		}

		switch (event.type) {
			case "context":
				return this.emitContext(event, signal);
			case "before_provider_request":
				return this.emitBeforeProviderRequest(event, signal);
			case "before_provider_payload":
				return this.emitBeforeProviderPayload(event, signal);
			case "before_agent_start":
				return this.emitBeforeAgentStart(event, signal);
			case "tool_call":
				return this.emitToolCall(event, signal);
			case "tool_result":
				return this.emitToolResult(event, signal);
			case "session_before_compact":
			case "session_before_tree":
				return this.emitFirstCancelOrLast(event, signal);
			default:
				await this.emitObservationHandlers(event, signal);
				return undefined;
		}
	}
}
```

由于 `Map<string, ...>` 会丢失具体类型信息，实现内部可以进行类型断言，但公开 API 仍须保持类型安全。

## 变更语义

### 观察

```ts
await hooks.emit({ type: "message_end", message }, signal);
```

观察器和 `message_end` 处理器都会运行。除非以后为该事件增加结果类型，否则返回值会被忽略。

### 上下文转换

处理器按顺序运行，每个处理器都会看到当前的消息列表。

```ts
let current = event;

for (const handler of handlers("context")) {
	const result = await handler(current, ctx, signal);
	if (result?.messages) {
		current = { ...current, messages: result.messages };
	}
}

return current.messages === event.messages ? undefined : { messages: current.messages };
```

### 服务提供商请求/载荷

按顺序执行转换，每个处理器都会看到前一个处理器的输出。

```ts
let current = event;

for (const handler of handlers("before_provider_payload")) {
	const result = await handler(current, ctx, signal);
	if (result !== undefined) {
		current = { ...current, payload: result.payload };
	}
}

return changed ? { payload: current.payload } : undefined;
```

### 智能体启动前

收集注入的消息，并依次转换系统提示词。

```ts
let systemPrompt = event.systemPrompt;
const messages = [];

for (const handler of handlers("before_agent_start")) {
	const result = await handler({ ...event, systemPrompt }, ctx, signal);
	if (result?.messages) messages.push(...result.messages);
	if (result?.systemPrompt !== undefined) systemPrompt = result.systemPrompt;
}

return messages.length || systemPrompt !== event.systemPrompt
	? { messages, systemPrompt }
	: undefined;
```

### 工具调用

按顺序执行；遇到阻止结果时提前退出。

```ts
for (const handler of handlers("tool_call")) {
	const result = await handler(event, ctx, signal);
	if (result?.block) return result;
}
```

### 工具结果

按顺序累积补丁，每个处理器都会看到应用现有补丁后的结果。

```ts
let current = event;
let modified = false;

for (const handler of handlers("tool_result")) {
	const result = await handler(current, ctx, signal);
	if (!result) continue;

	current = {
		...current,
		content: result.content ?? current.content,
		details: result.details ?? current.details,
		isError: result.isError ?? current.isError,
	};

	modified = true;
}

return modified
	? { content: current.content, details: current.details, isError: current.isError }
	: undefined;
```

### 会话操作前事件

按顺序执行；遇到取消结果时提前退出。

```ts
let last;

for (const handler of handlers(event.type)) {
	const result = await handler(event, ctx, signal);
	if (!result) continue;
	last = result;
	if (result.cancel) return result;
}

return last;
```

## 运行框架的用法

运行框架只需执行：

```ts
await this.hooks.emit(event, signal);
```

或者：

```ts
const result = await this.hooks.emit({ type: "context", messages }, signal);
return result?.messages ?? messages;
```

运行框架不存储处理器、不串联监听器，也不需要了解扩展策略。

## 上下文

上下文是普通对象，不会在每次发出事件时重新构建。

```ts
const hooks = new CodingAgentHooks({
	harness: harnessFacade,
	session: sessionFacade,
	ui: noUiFacade,
});
```

之后：

```ts
hooks.setContext({
	...hooks.context,
	ui: tuiFacade,
});
```

对于动态状态，应优先使用稳定的门面和方法，避免形成错综复杂的 getter：

```ts
interface CodingAgentHookContext {
	harness: HarnessFacade;
	session: SessionFacade;
	ui: UiFacade;
	models: ModelFacade;
}
```

每次运行对应的 `signal` 会作为处理器的第三个参数传入。

## 后续加载扩展

扩展加载逻辑可以放在运行框架旁，并负责构造钩子：

```ts
const hooks = await loadExtensions({
	paths,
	context,
	hooks: new CodingAgentHooks(context),
});
const harness = new AgentHarness({ ..., hooks });
```

加载器向钩子注册内容：

```ts
hooks.on("context", handler);
hooks.on("tool_call", handler);
hooks.addCleanup(cleanup);
```

重新加载时：

```ts
await hooks.clear();
const nextHooks = await loadExtensions(...);
harness.setHooks(nextHooks); // 如果支持此方法，则只能在空闲状态调用
```

## 设计审视

### 1. 必须明确错误策略

现有的 coding-agent 会捕获并报告扩展错误，然后继续执行。新钩子也需要相同的策略，可能采用：

```ts
errorMode: "continue" | "throw"
onError(error)
```

对于 coding-agent，默认值应为 `"continue"`。

### 2. 来源元数据很重要

现有运行器知道错误、资源或工具来自哪个扩展。如果不增加注册元数据或作用域，单纯使用 `on()` 会丢失这些信息。

可能需要：

```ts
const scope = hooks.createScope({ sourceInfo });
scope.on("context", handler);
scope.addCleanup(...);
```

或者使用 `on(type, handler, { sourceInfo })`。

### 3. 部分扩展能力属于注册表，而非钩子

以下能力不在 `emit()` 的覆盖范围内，应继续作为 `CodingAgentHooks` 或扩展宿主中的注册表：

- 工具
- 命令
- 快捷键
- 标志
- 消息渲染器
- 服务提供商注册项
- OAuth 服务提供商
- 自定义模型服务提供商

这样是合理的，因为它们不属于 `AgentHarness`。

### 4. 可以表示现有 coding-agent 事件

以下事件均不存在设计障碍：

- `context`
- `before_provider_request`
- `after_provider_response`
- `before_agent_start`
- `message_end`
- `tool_call`
- `tool_result`
- `input`
- `user_bash`
- `resources_discover`
- `session_before_*`
- `session_*`
- 模型/思考级别选择事件
- 智能体/轮次/消息/工具生命周期事件

它们会成为由 `CodingAgentHooks` 处理的其他事件类型。

### 5. 必须准确保留旧有语义

迁移 coding-agent 时，必须保留以下特殊行为：

- `input`：转换链；`handled` 会使其短路。
- `user_bash`：采用第一个有实际意义的结果。
- `message_end`：替换后的消息必须保持相同角色。
- `before_agent_start`：`ctx.getSystemPrompt()` 必须反映当前经链式转换后的提示词。
- `resources_discover`：汇总路径并保留扩展来源。
- `tool_call`：后续处理器仍能看到对参数所作的修改。
- `tool_result`：后续处理器能看到之前应用的补丁。

本设计可以支持以上行为，但默认钩子和 coding-agent 钩子的实现必须将这些语义明确编码进去。

### 6. `emit()` 的 switch 可能遗漏自定义变更事件

如果子类新增了一种会产生结果的事件，却忘记重写 `emit()`，该事件将只按观察事件处理。测试应能发现这类问题。如果实践证明这里容易出错，以后可以增加受保护的策略注册表，但初期不必引入。

### 7. 观察器语义有意保持受限

观察器只会看到一次最初发出的事件，不会看到每次中间变更。如果某项功能需要最终转换后的状态，应另行发出最终事件，或者使用该事件专用的处理器。

## 结论

该设计可以用于实现新的 coding-agent。它比当前运行器更简单，也能保持运行框架整洁。只要 `CodingAgentHooks` 增加能够感知来源的作用域、注册表、清理机制，并准确实现旧有事件语义，就能保留重要的扩展能力。

--- 评论 ---

关于“addCleanup(cleanup”的讨论串 hn2xk0tzhj
  [tmluyaub9v] 所有者 (2026-05-14T12:55:45.500Z)：应允许将 cleanup 作为可选参数一并传给 on/observe
