> pi 可以创建 TUI 组件。你可以让它针对自己的使用场景构建组件。

# TUI 组件

扩展和自定义工具可以渲染自定义 TUI 组件，构成交互式用户界面。本文介绍组件系统及其可用的基础构件。

**源代码：** [`@earendil-works/pi-tui`](https://github.com/earendil-works/pi-mono/tree/main/packages/tui)

<a id="component-interface"></a>

## 组件接口

所有组件都实现以下接口：

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

| 方法 | 说明 |
|--------|-------------|
| `render(width)` | 返回字符串数组，每个元素对应一行。每一行的宽度都**不得超过 `width`**。 |
| `handleInput?(data)` | 组件获得焦点时接收键盘输入。 |
| `wantsKeyRelease?` | 如果为 `true`，组件会接收按键释放事件（Kitty 键盘协议）。默认值：`false`。 |
| `invalidate()` | 清除缓存的渲染状态。主题变化时会调用。 |

TUI 会在每个已渲染行的末尾追加完整的 SGR 重置序列和 OSC 8 重置序列。样式不会跨行延续。如果输出带样式的多行文本，请逐行重新应用样式，或使用 `wrapTextWithAnsi()`，以便每个换行后的文本行都保留样式。

<a id="focusable-interface-ime-support"></a>

## Focusable 接口（支持输入法）

需要显示文本光标并支持 IME（Input Method Editor，输入法编辑器）的组件，应实现 `Focusable` 接口：

```typescript
import { CURSOR_MARKER, type Component, type Focusable } from "@earendil-works/pi-tui";

class MyInput implements Component, Focusable {
  focused: boolean = false;  // 焦点变化时由 TUI 设置

  render(width: number): string[] {
    const marker = this.focused ? CURSOR_MARKER : "";
    // 在模拟光标之前输出标记
    return [`> ${beforeCursor}${marker}\x1b[7m${atCursor}\x1b[27m${afterCursor}`];
  }
}
```

当 `Focusable` 组件获得焦点时，TUI 会：

1. 将组件的 `focused` 设为 `true`
2. 在渲染输出中查找 `CURSOR_MARKER`（一种零宽度 APC 转义序列）
3. 将终端的硬件光标定位到该处
4. 仅在启用 `showHardwareCursor` 时显示硬件光标

默认情况下光标保持隐藏。这样既能继续渲染模拟光标，又能为会根据隐藏光标跟踪输入法候选窗口的终端定位硬件光标。部分终端要求硬件光标可见才能正确定位输入法；可通过 `showHardwareCursor`、`setShowHardwareCursor(true)` 或 `PI_HARDWARE_CURSOR=1` 启用。内置的 `Editor` 和 `Input` 组件已经实现该接口。

### 包含输入组件的容器组件

如果容器组件（对话框、选择器等）包含 `Input` 或 `Editor` 子组件，容器必须实现 `Focusable`，并把焦点状态传递给子组件。否则，使用输入法时硬件光标无法正确定位。

```typescript
import { Container, type Focusable, Input } from "@earendil-works/pi-tui";

class SearchDialog extends Container implements Focusable {
  private searchInput: Input;

  // Focusable 实现——将状态传给子输入组件，以便定位输入法光标
  private _focused = false;
  get focused(): boolean {
    return this._focused;
  }
  set focused(value: boolean) {
    this._focused = value;
    this.searchInput.focused = value;
  }

  constructor() {
    super();
    this.searchInput = new Input();
    this.addChild(this.searchInput);
  }
}
```

如果不传递焦点状态，使用中文、日文、韩文等输入法输入时，候选窗口会显示在屏幕上的错误位置。

<a id="using-components"></a>

## 使用组件

**在扩展中**通过 `ctx.ui.custom()` 使用：

```typescript
pi.on("session_start", async (_event, ctx) => {
  const result = await ctx.ui.custom<string | null>((tui, theme, keybindings, done) =>
    new MyComponent({
      theme,
      keybindings,
      onChange: () => tui.requestRender(),
      onSelect: (value) => done(value),
      onCancel: () => done(null),
    })
  );
});
```

**在自定义工具中**同样通过 `ctx.ui.custom()` 使用：

```typescript
async execute(toolCallId, params, signal, onUpdate, ctx) {
  const result = await ctx.ui.custom<string | null>((tui, theme, keybindings, done) =>
    new MyComponent({
      theme,
      keybindings,
      onChange: () => tui.requestRender(),
      onSelect: (value) => done(value),
      onCancel: () => done(null),
    })
  );
  // 使用结果……
}
```

<a id="overlays"></a>

## 浮层

浮层会在现有内容上方渲染组件，而不会清空屏幕。调用 `ctx.ui.custom()` 时传入 `{ overlay: true }`：

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new MyDialog({ onClose: done }),
  { overlay: true }
);
```

使用 `overlayOptions` 控制位置和尺寸：

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new SidePanel({ onClose: done }),
  {
    overlay: true,
    overlayOptions: {
      // 尺寸：数值或百分比字符串
      width: "50%",          // 终端宽度的 50%
      minWidth: 40,          // 最少 40 列
      maxHeight: "80%",      // 最多为终端高度的 80%

      // 位置：基于锚点（默认值："center"）
      anchor: "right-center", // 9 个位置：center、top-left、top-center 等
      offsetX: -2,            // 相对锚点的偏移量
      offsetY: 0,

      // 也可以按百分比或绝对位置定位
      row: "25%",            // 距顶部 25%
      col: 10,               // 第 10 列

      // 外边距
      margin: 2,             // 四边相同，或使用 { top, right, bottom, left }

      // 响应式行为：在较窄的终端中隐藏
      visible: (termWidth, termHeight) => termWidth >= 80,
    },
    // 获取句柄，以编程方式控制焦点和可见性
    onHandle: (handle) => {
      // handle.focus()——让该浮层获得焦点并移到视觉最前方
      // handle.unfocus()——释放输入，由常规回退逻辑处理
      // handle.unfocus({ target })——将输入释放给指定组件或 null
      // handle.setHidden(true/false)——切换可见性
      // handle.hide()——永久移除
    },
  }
);
```

### 浮层焦点

获得焦点且可见的浮层在临时显示非浮层界面时仍会保留输入所有权。如果浮层又打开一个未设置 `{ overlay: true }` 的 `ctx.ui.custom()` 组件，该替代界面在活动期间会接收输入；关闭后，原来获得焦点的浮层可以重新接管输入。

如果可见浮层不应继续占有输入，并希望 TUI 回退到另一个能够捕获输入的可见浮层或之前的焦点目标，请调用 `handle.unfocus()`。如果浮层保持可见，但希望由特定组件接收输入，请调用 `handle.unfocus({ target })`。传入 `{ target: null }` 会刻意让所有组件都失去焦点，直到再次设置焦点。

### 浮层生命周期

浮层组件关闭时会被销毁。不要复用旧引用，应创建新实例：

```typescript
// 错误——引用已经失效
let menu: MenuComponent;
await ctx.ui.custom((_, __, ___, done) => {
  menu = new MenuComponent(done);
  return menu;
}, { overlay: true });
setActiveComponent(menu);  // 已销毁

// 正确——再次调用以重新显示
const showMenu = () => ctx.ui.custom((_, __, ___, done) =>
  new MenuComponent(done), { overlay: true });

await showMenu();  // 第一次显示
await showMenu();  // “返回”时直接再次调用
```

有关锚点、外边距、堆叠、响应式可见性和动画的完整示例，请参阅 [overlay-qa-tests.ts](../examples/extensions/overlay-qa-tests.ts)。

<a id="built-in-components"></a>

## 内置组件

从 `@earendil-works/pi-tui` 导入：

```typescript
import { Text, Box, Container, Spacer, Markdown } from "@earendil-works/pi-tui";
```

### Text

支持自动换行的多行文本。

```typescript
const text = new Text(
  "你好，世界",      // 内容
  1,                // paddingX（默认值：1）
  1,                // paddingY（默认值：1）
  (s) => bgGray(s)  // 可选的背景函数
);
text.setText("已更新");
```

### Box

带内边距和背景色的容器。

```typescript
const box = new Box(
  1,                // paddingX
  1,                // paddingY
  (s) => bgGray(s)  // 背景函数
);
box.addChild(new Text("内容", 0, 0));
box.setBgFn((s) => bgBlue(s));
```

### Container

纵向排列子组件。

```typescript
const container = new Container();
container.addChild(component1);
container.addChild(component2);
container.removeChild(component1);
```

### Spacer

纵向留白。

```typescript
const spacer = new Spacer(2);  // 两个空行
```

### Markdown

渲染带语法高亮的 Markdown。

```typescript
const md = new Markdown(
  "# 标题\n\n一些**粗体**文本",
  1,        // paddingX
  1,        // paddingY
  theme     // MarkdownTheme（见下文）
);
md.setText("已更新的 Markdown");
```

### Image

在支持的终端中渲染图像（Kitty、iTerm2、Ghostty、WezTerm、Warp）。

```typescript
const image = new Image(
  base64Data,   // Base64 编码的图像
  "image/png",  // MIME 类型
  theme,        // ImageTheme
  { maxWidthCells: 80, maxHeightCells: 24 }
);
```

<a id="keyboard-input"></a>

## 键盘输入

使用 `matchesKey()` 检测按键：

```typescript
import { matchesKey, Key } from "@earendil-works/pi-tui";

handleInput(data: string) {
  if (matchesKey(data, Key.up)) {
    this.selectedIndex--;
  } else if (matchesKey(data, Key.enter)) {
    this.onSelect?.(this.selectedIndex);
  } else if (matchesKey(data, Key.escape)) {
    this.onCancel?.();
  } else if (matchesKey(data, Key.ctrl("c"))) {
    // Ctrl+C
  }
}
```

**按键标识符**（使用 `Key.*` 可获得自动补全，也可以使用字符串字面值）：

- 基本按键：`Key.enter`、`Key.escape`、`Key.tab`、`Key.space`、`Key.backspace`、`Key.delete`、`Key.home`、`Key.end`
- 方向键：`Key.up`、`Key.down`、`Key.left`、`Key.right`
- 带修饰键：`Key.ctrl("c")`、`Key.shift("tab")`、`Key.alt("left")`、`Key.ctrlShift("p")`
- 也支持字符串形式：`"enter"`、`"ctrl+c"`、`"shift+tab"`、`"ctrl+shift+p"`

<a id="line-width"></a>

## 行宽

**重要：** `render()` 返回的每一行都不得超过 `width` 参数指定的宽度。

```typescript
import { visibleWidth, truncateToWidth } from "@earendil-works/pi-tui";

render(width: number): string[] {
  // 截断过长的行
  return [truncateToWidth(this.text, width)];
}
```

实用函数：

- `visibleWidth(str)`——获取显示宽度（忽略 ANSI 代码）
- `truncateToWidth(str, width, ellipsis?)`——截断文本，可选择添加省略号
- `wrapTextWithAnsi(str, width)`——自动换行并保留 ANSI 代码

<a id="creating-custom-components"></a>

## 创建自定义组件

以下示例实现一个交互式选择器：

```typescript
import {
  matchesKey, Key,
  truncateToWidth, visibleWidth
} from "@earendil-works/pi-tui";

class MySelector {
  private items: string[];
  private selected = 0;
  private cachedWidth?: number;
  private cachedLines?: string[];

  public onSelect?: (item: string) => void;
  public onCancel?: () => void;

  constructor(items: string[]) {
    this.items = items;
  }

  handleInput(data: string): void {
    if (matchesKey(data, Key.up) && this.selected > 0) {
      this.selected--;
      this.invalidate();
    } else if (matchesKey(data, Key.down) && this.selected < this.items.length - 1) {
      this.selected++;
      this.invalidate();
    } else if (matchesKey(data, Key.enter)) {
      this.onSelect?.(this.items[this.selected]);
    } else if (matchesKey(data, Key.escape)) {
      this.onCancel?.();
    }
  }

  render(width: number): string[] {
    if (this.cachedLines && this.cachedWidth === width) {
      return this.cachedLines;
    }

    this.cachedLines = this.items.map((item, i) => {
      const prefix = i === this.selected ? "> " : "  ";
      return truncateToWidth(prefix + item, width);
    });
    this.cachedWidth = width;
    return this.cachedLines;
  }

  invalidate(): void {
    this.cachedWidth = undefined;
    this.cachedLines = undefined;
  }
}
```

在扩展中的用法：

```typescript
pi.registerCommand("pick", {
  description: "选择一项",
  handler: async (_args, ctx) => {
    const items = ["选项 A", "选项 B", "选项 C"];
    const selected = await ctx.ui.custom<string | null>((tui, _theme, _keybindings, done) => {
      const selector = new MySelector(items);
      selector.onSelect = done;
      selector.onCancel = () => done(null);

      return {
        render: (width) => selector.render(width),
        handleInput: (data) => {
          selector.handleInput(data);
          tui.requestRender();
        },
        invalidate: () => selector.invalidate(),
      };
    });

    if (selected !== null) {
      ctx.ui.notify(`已选择：${selected}`, "info");
    }
  }
});
```

<a id="theming"></a>

## 主题

组件通过主题对象设置样式。

**在 `renderCall`/`renderResult` 中**，请使用 `theme` 参数：

```typescript
renderResult(result, options, theme, context) {
  // 使用 theme.fg() 设置前景色
  return new Text(theme.fg("success", "完成！"), 0, 0);

  // 使用 theme.bg() 设置背景色
  const styled = theme.bg("toolPendingBg", theme.fg("accent", "text"));
}
```

**前景色**（`theme.fg(color, text)`）：

| 类别 | 颜色 |
|----------|--------|
| 通用 | `text`, `accent`, `muted`, `dim` |
| 状态 | `success`, `error`, `warning` |
| 边框 | `border`, `borderAccent`, `borderMuted` |
| 消息 | `userMessageText`, `customMessageText`, `customMessageLabel` |
| 工具 | `toolTitle`, `toolOutput` |
| 差异 | `toolDiffAdded`, `toolDiffRemoved`, `toolDiffContext` |
| Markdown | `mdHeading`, `mdLink`, `mdLinkUrl`, `mdCode`, `mdCodeBlock`, `mdCodeBlockBorder`, `mdQuote`, `mdQuoteBorder`, `mdHr`, `mdListBullet` |
| 语法 | `syntaxComment`, `syntaxKeyword`, `syntaxFunction`, `syntaxVariable`, `syntaxString`, `syntaxNumber`, `syntaxType`, `syntaxOperator`, `syntaxPunctuation` |
| 思考 | `thinkingOff`, `thinkingMinimal`, `thinkingLow`, `thinkingMedium`, `thinkingHigh`, `thinkingXhigh`, `thinkingMax` |
| 模式 | `bashMode` |

**背景色**（`theme.bg(color, text)`）：

`selectedBg`, `userMessageBg`, `customMessageBg`, `toolPendingBg`, `toolSuccessBg`, `toolErrorBg`

**对于 Markdown**，请使用 `getMarkdownTheme()`：

```typescript
import { getMarkdownTheme } from "@earendil-works/pi-coding-agent";
import { Markdown } from "@earendil-works/pi-tui";

renderResult(result, options, theme, context) {
  const mdTheme = getMarkdownTheme();
  return new Markdown(result.details.markdown, 0, 0, mdTheme);
}
```

**对于自定义组件**，请定义自己的主题接口：

```typescript
interface MyTheme {
  selected: (s: string) => string;
  normal: (s: string) => string;
}
```

<a id="debug-logging"></a>

## 调试日志

设置 `PI_TUI_WRITE_LOG`，可以捕获写入标准输出的原始 ANSI 流。

```bash
PI_TUI_WRITE_LOG=/tmp/tui-ansi.log npx tsx packages/tui/test/chat-simple.ts
```

<a id="performance"></a>

## 性能

尽可能缓存渲染输出：

```typescript
class CachedComponent {
  private cachedWidth?: number;
  private cachedLines?: string[];

  render(width: number): string[] {
    if (this.cachedLines && this.cachedWidth === width) {
      return this.cachedLines;
    }
    // ……计算各行……
    this.cachedWidth = width;
    this.cachedLines = lines;
    return lines;
  }

  invalidate(): void {
    this.cachedWidth = undefined;
    this.cachedLines = undefined;
  }
}
```

状态变化时调用 `invalidate()`，然后使用注入的 `tui.requestRender()` 触发重新渲染。

<a id="invalidation-and-theme-changes"></a>

## 缓存失效与主题变化

主题变化时，TUI 会对所有组件调用 `invalidate()` 来清除缓存。组件必须正确实现 `invalidate()`，才能确保主题变化生效。

### 问题

如果组件事先通过 `theme.fg()`、`theme.bg()` 等函数将主题颜色写入字符串并缓存起来，那么缓存字符串中包含的是旧主题的 ANSI 转义代码。如果组件还单独保存了这些带主题样式的内容，仅清除渲染缓存并不足以更新主题。

**错误做法**（主题颜色不会更新）：

```typescript
class BadComponent extends Container {
  private content: Text;

  constructor(message: string, theme: Theme) {
    super();
    // Text 组件中存储了预先写入主题颜色的文本
    this.content = new Text(theme.fg("accent", message), 1, 0);
    this.addChild(this.content);
  }
  // 没有覆盖 invalidate——父类的 invalidate 只会清除
  // 子组件的渲染缓存，不会重建预先带入样式的内容
}
```

### 解决方案

使用主题颜色构建内容的组件，必须在调用 `invalidate()` 时重新构建这些内容：

```typescript
class GoodComponent extends Container {
  private message: string;
  private content: Text;

  constructor(message: string) {
    super();
    this.message = message;
    this.content = new Text("", 1, 0);
    this.addChild(this.content);
    this.updateDisplay();
  }

  private updateDisplay(): void {
    // 使用当前主题重新构建内容
    this.content.setText(theme.fg("accent", this.message));
  }

  override invalidate(): void {
    super.invalidate();   // 清除子组件缓存
    this.updateDisplay(); // 使用新主题重新构建
  }
}
```

### 模式：失效时重建

对于内容复杂的组件：

```typescript
class ComplexComponent extends Container {
  private data: SomeData;

  constructor(data: SomeData) {
    super();
    this.data = data;
    this.rebuild();
  }

  private rebuild(): void {
    this.clear();  // 移除所有子组件

    // 使用当前主题构建界面
    this.addChild(new Text(theme.fg("accent", theme.bold("Title")), 1, 0));
    this.addChild(new Spacer(1));

    for (const item of this.data.items) {
      const color = item.active ? "success" : "muted";
      this.addChild(new Text(theme.fg(color, item.label), 1, 0));
    }
  }

  override invalidate(): void {
    super.invalidate();
    this.rebuild();
  }
}
```

### 适用场景

以下情况需要使用此模式：

1. **预先写入主题颜色**——使用 `theme.fg()` 或 `theme.bg()` 创建带样式字符串，并将其存储在子组件中
2. **语法高亮**——使用会应用主题语法颜色的 `highlightCode()`
3. **复杂布局**——构建嵌入主题颜色的子组件树

以下情况**不需要**使用此模式：

1. **使用主题回调**——传入 `(text) => theme.fg("accent", text)` 这类在渲染时调用的函数
2. **简单容器**——只组合其他组件，不添加带主题样式的内容
3. **无状态渲染**——每次调用 `render()` 都重新计算带主题样式的输出，不使用缓存

<a id="common-patterns"></a>

## 常用模式

以下模式涵盖了扩展中最常见的界面需求。**请复用这些模式，不必从头实现。**

### 模式 1：选择对话框（SelectList）

用于让用户从选项列表中选择一项。使用 `@earendil-works/pi-tui` 中的 `SelectList`，并用 `DynamicBorder` 绘制边框。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { DynamicBorder } from "@earendil-works/pi-coding-agent";
import { Container, type SelectItem, SelectList, Text } from "@earendil-works/pi-tui";

pi.registerCommand("pick", {
  handler: async (_args, ctx) => {
    const items: SelectItem[] = [
      { value: "opt1", label: "选项 1", description: "第一个选项" },
      { value: "opt2", label: "选项 2", description: "第二个选项" },
      { value: "opt3", label: "选项 3" },  // description 可选
    ];

    const result = await ctx.ui.custom<string | null>((tui, theme, _kb, done) => {
      const container = new Container();

      // 上边框
      container.addChild(new DynamicBorder((s: string) => theme.fg("accent", s)));

      // 标题
      container.addChild(new Text(theme.fg("accent", theme.bold("请选择")), 1, 0));

      // 配置主题的 SelectList
      const selectList = new SelectList(items, Math.min(items.length, 10), {
        selectedPrefix: (t) => theme.fg("accent", t),
        selectedText: (t) => theme.fg("accent", t),
        description: (t) => theme.fg("muted", t),
        scrollInfo: (t) => theme.fg("dim", t),
        noMatch: (t) => theme.fg("warning", t),
      });
      selectList.onSelect = (item) => done(item.value);
      selectList.onCancel = () => done(null);
      container.addChild(selectList);

      // 帮助文本
      container.addChild(new Text(theme.fg("dim", "↑↓ 移动 • Enter 选择 • Esc 取消"), 1, 0));

      // 下边框
      container.addChild(new DynamicBorder((s: string) => theme.fg("accent", s)));

      return {
        render: (w) => container.render(w),
        invalidate: () => container.invalidate(),
        handleInput: (data) => { selectList.handleInput(data); tui.requestRender(); },
      };
    });

    if (result) {
      ctx.ui.notify(`已选择：${result}`, "info");
    }
  },
});
```

**示例：** [preset.ts](../examples/extensions/preset.ts)、[tools.ts](../examples/extensions/tools.ts)

### 模式 2：可取消的异步操作（BorderedLoader）

用于耗时且应允许取消的操作。`BorderedLoader` 会显示加载动画，并处理 Escape 取消操作。

```typescript
import { BorderedLoader } from "@earendil-works/pi-coding-agent";

pi.registerCommand("fetch", {
  handler: async (_args, ctx) => {
    const result = await ctx.ui.custom<string | null>((tui, theme, _kb, done) => {
      const loader = new BorderedLoader(tui, theme, "正在获取数据……");
      loader.onAbort = () => done(null);

      // 执行异步工作
      fetchData(loader.signal)
        .then((data) => done(data))
        .catch(() => done(null));

      return loader;
    });

    if (result === null) {
      ctx.ui.notify("已取消", "info");
    } else {
      ctx.ui.setEditorText(result);
    }
  },
});
```

**示例：** [qna.ts](../examples/extensions/qna.ts)、[handoff.ts](../examples/extensions/handoff.ts)

### 模式 3：设置/开关（SettingsList）

用于切换多项设置。使用 `@earendil-works/pi-tui` 中的 `SettingsList`，并配合 `getSettingsListTheme()`。

```typescript
import { getSettingsListTheme } from "@earendil-works/pi-coding-agent";
import { Container, type SettingItem, SettingsList, Text } from "@earendil-works/pi-tui";

pi.registerCommand("settings", {
  handler: async (_args, ctx) => {
    const items: SettingItem[] = [
      { id: "verbose", label: "详细模式", currentValue: "off", values: ["on", "off"] },
      { id: "color", label: "彩色输出", currentValue: "on", values: ["on", "off"] },
    ];

    await ctx.ui.custom((_tui, theme, _kb, done) => {
      const container = new Container();
      container.addChild(new Text(theme.fg("accent", theme.bold("设置")), 1, 1));

      const settingsList = new SettingsList(
        items,
        Math.min(items.length + 2, 15),
        getSettingsListTheme(),
        (id, newValue) => {
          // 处理值变化
          ctx.ui.notify(`${id} = ${newValue}`, "info");
        },
        () => done(undefined),  // 关闭时
        { enableSearch: true }, // 可选：按标签启用模糊搜索
      );
      container.addChild(settingsList);

      return {
        render: (w) => container.render(w),
        invalidate: () => container.invalidate(),
        handleInput: (data) => settingsList.handleInput?.(data),
      };
    });
  },
});
```

**示例：** [tools.ts](../examples/extensions/tools.ts)

### 模式 4：常驻状态指示器

在页脚中显示跨渲染周期保留的状态，适合用作模式指示器。

```typescript
// 设置状态（显示在页脚中）
ctx.ui.setStatus("my-ext", ctx.ui.theme.fg("accent", "● 活动"));

// 清除状态
ctx.ui.setStatus("my-ext", undefined);
```

**示例：** [status-line.ts](../examples/extensions/status-line.ts)、[plan-mode/index.ts](../examples/extensions/plan-mode/index.ts)、[preset.ts](../examples/extensions/preset.ts)

### 模式 4b：自定义工作状态指示器

自定义 pi 以流式方式生成响应时显示的行内工作状态指示器。

```typescript
// 静态指示器
ctx.ui.setWorkingIndicator({ frames: [ctx.ui.theme.fg("accent", "●")] });

// 自定义动画指示器
ctx.ui.setWorkingIndicator({
  frames: [
    ctx.ui.theme.fg("dim", "·"),
    ctx.ui.theme.fg("muted", "•"),
    ctx.ui.theme.fg("accent", "●"),
    ctx.ui.theme.fg("muted", "•"),
  ],
  intervalMs: 120,
});

// 完全隐藏指示器
ctx.ui.setWorkingIndicator({ frames: [] });

// 恢复 pi 的默认加载动画
ctx.ui.setWorkingIndicator();
```

此设置只影响正常流式响应期间的工作状态指示器。上下文压缩和重试加载器仍使用内置样式。自定义帧会原样渲染，因此扩展需要自行添加所需颜色。

**示例：** [working-indicator.ts](../examples/extensions/working-indicator.ts)

### 模式 5：编辑器上方/下方的小组件

在输入编辑器上方或下方显示常驻内容，适合待办事项列表、进度等信息。

```typescript
// 简单字符串数组（默认显示在编辑器上方）
ctx.ui.setWidget("my-widget", ["第 1 行", "第 2 行"]);

// 在编辑器下方渲染
ctx.ui.setWidget("my-widget", ["第 1 行", "第 2 行"], { placement: "belowEditor" });

// 也可以使用主题
ctx.ui.setWidget("my-widget", (_tui, theme) => {
  const lines = items.map((item, i) =>
    item.done
      ? theme.fg("success", "✓ ") + theme.fg("muted", item.text)
      : theme.fg("dim", "○ ") + item.text
  );
  return {
    render: () => lines,
    invalidate: () => {},
  };
});

// 清除
ctx.ui.setWidget("my-widget", undefined);
```

**示例：** [plan-mode/index.ts](../examples/extensions/plan-mode/index.ts)

### 模式 6：自定义页脚

替换页脚。`footerData` 提供扩展无法通过其他方式访问的数据。

```typescript
ctx.ui.setFooter((tui, theme, footerData) => ({
  invalidate() {},
  render(width: number): string[] {
    // footerData.getGitBranch(): string | null
    // footerData.getExtensionStatuses(): ReadonlyMap<string, string>
    return [`${ctx.model?.id} (${footerData.getGitBranch() || "无 Git"})`];
  },
  dispose: footerData.onBranchChange(() => tui.requestRender()), // 响应式更新
}));

ctx.ui.setFooter(undefined); // 恢复默认页脚
```

可以通过 `ctx.sessionManager.getBranch()` 和 `ctx.model` 获取 Token 统计信息。

**示例：** [custom-footer.ts](../examples/extensions/custom-footer.ts)

### 模式 7：自定义编辑器（Vim 模式等）

使用自定义实现替换主输入编辑器。适用于 Vim 模态编辑、Emacs 等不同的按键绑定，或专门的输入处理逻辑。

```typescript
import { CustomEditor, type ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { matchesKey, truncateToWidth } from "@earendil-works/pi-tui";

type Mode = "normal" | "insert";

class VimEditor extends CustomEditor {
  private mode: Mode = "insert";

  handleInput(data: string): void {
    // Escape：切换到普通模式，或交给应用处理
    if (matchesKey(data, "escape")) {
      if (this.mode === "insert") {
        this.mode = "normal";
        return;
      }
      // 普通模式下，Escape 会中止智能体（由 CustomEditor 处理）
      super.handleInput(data);
      return;
    }

    // 插入模式：将所有输入交给 CustomEditor
    if (this.mode === "insert") {
      super.handleInput(data);
      return;
    }

    // 普通模式：Vim 风格导航
    switch (data) {
      case "i": this.mode = "insert"; return;
      case "h": super.handleInput("\x1b[D"); return; // 左
      case "j": super.handleInput("\x1b[B"); return; // 下
      case "k": super.handleInput("\x1b[A"); return; // 上
      case "l": super.handleInput("\x1b[C"); return; // 右
    }
    // 将未处理的按键（Ctrl+C 等）交给父类，但过滤可打印字符
    if (data.length === 1 && data.charCodeAt(0) >= 32) return;
    super.handleInput(data);
  }

  render(width: number): string[] {
    const lines = super.render(width);
    // 在下边框添加模式指示器（使用 truncateToWidth，安全截断带 ANSI 代码的文本）
    if (lines.length > 0) {
      const label = this.mode === "normal" ? " NORMAL " : " INSERT ";
      const lastLine = lines[lines.length - 1]!;
      // 将 "" 作为省略号传入，避免截断时添加 "..."
      lines[lines.length - 1] = truncateToWidth(lastLine, width - label.length, "") + label;
    }
    return lines;
  }
}

export default function (pi: ExtensionAPI) {
  pi.on("session_start", (_event, ctx) => {
    // 工厂函数从应用接收 TUI、主题和按键绑定
    ctx.ui.setEditorComponent((tui, theme, keybindings) =>
      new VimEditor(tui, theme, keybindings)
    );
  });
}
```

**要点：**

- **扩展 `CustomEditor`**（而不是基础 `Editor`），以获得应用的按键绑定，例如按 Escape 中止、按 Ctrl+D 退出、切换模型等
- 对没有自行处理的按键，**调用 `super.handleInput(data)`**
- **工厂模式**：`setEditorComponent` 接收一个工厂函数，该函数可获得 `tui`、`theme` 和 `keybindings`
- **传入 `undefined`** 可恢复默认编辑器：`ctx.ui.setEditorComponent(undefined)`

**示例：** [modal-editor.ts](../examples/extensions/modal-editor.ts)

<a id="key-rules"></a>

## 关键规则

1. **始终使用回调提供的主题**——不要直接导入主题。应使用 `ctx.ui.custom((tui, theme, keybindings, done) => ...)` 回调中的 `theme`。

2. **始终为 DynamicBorder 的颜色参数标注类型**——写成 `(s: string) => theme.fg("accent", s)`，不要写成 `(s) => theme.fg("accent", s)`。

3. **状态变化后调用 `tui.requestRender()`**——在 `handleInput` 中更新状态后，调用 `tui.requestRender()`。

4. **返回包含三个方法的对象**——自定义组件需要提供 `{ render, invalidate, handleInput }`。

5. **使用现有组件**——`SelectList`、`SettingsList`、`BorderedLoader` 可以覆盖 90% 的使用场景，无需重复实现。

<a id="examples"></a>

## 示例

- **选择界面**：[examples/extensions/preset.ts](../examples/extensions/preset.ts)——使用 DynamicBorder 绘制边框的 SelectList
- **可取消的异步操作**：[examples/extensions/qna.ts](../examples/extensions/qna.ts)——用于 LLM 调用的 BorderedLoader
- **设置开关**：[examples/extensions/tools.ts](../examples/extensions/tools.ts)——用于启用/禁用工具的 SettingsList
- **状态指示器**：[examples/extensions/plan-mode/index.ts](../examples/extensions/plan-mode/index.ts)——`setStatus` 和 `setWidget`
- **工作状态指示器**：[examples/extensions/working-indicator.ts](../examples/extensions/working-indicator.ts)——`setWorkingIndicator`
- **自定义页脚**：[examples/extensions/custom-footer.ts](../examples/extensions/custom-footer.ts)——通过 `setFooter` 显示统计信息
- **自定义编辑器**：[examples/extensions/modal-editor.ts](../examples/extensions/modal-editor.ts)——类似 Vim 的模态编辑
- **贪吃蛇游戏**：[examples/extensions/snake.ts](../examples/extensions/snake.ts)——包含键盘输入和游戏循环的完整游戏
- **自定义工具渲染**：[examples/extensions/todo.ts](../examples/extensions/todo.ts)——`renderCall` 和 `renderResult`
