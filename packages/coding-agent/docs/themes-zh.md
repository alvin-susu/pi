> Pi 可以创建主题。你可以让它为自己的终端环境构建一个主题。

# 主题

主题是用于定义 TUI 配色的 JSON 文件。

## 目录

- [存放位置](#locations)
- [选择主题](#selecting-a-theme)
- [创建自定义主题](#creating-a-custom-theme)
- [主题格式](#theme-format)
- [颜色 Token](#color-tokens)
- [颜色值](#color-values)
- [建议](#tips)

<a id="locations"></a>

## 存放位置

Pi 会从以下位置加载主题：

- 内置：`dark`、`light`
- 全局：`~/.pi/agent/themes/*.json`
- 项目：`.pi/themes/*.json`（仅在项目受信任后加载）
- 包：`themes/` 目录，或 `package.json` 中的 `pi.themes` 条目
- 设置：`themes` 数组中指定的文件或目录
- CLI：`--theme <path>`（可重复指定）

使用 `--no-themes` 可以禁用自动发现。

<a id="selecting-a-theme"></a>

## 选择主题

通过 `/settings` 或在 `settings.json` 中选择主题：

```json
{
  "theme": "my-theme"
}
```

首次运行时，Pi 会检测终端背景，并默认选择 `dark` 或 `light`。

<a id="creating-a-custom-theme"></a>

## 创建自定义主题

1. 创建主题文件：

```bash
mkdir -p ~/.pi/agent/themes
vim ~/.pi/agent/themes/my-theme.json
```

2. 定义主题所需的全部颜色（参阅[颜色 Token](#color-tokens)）：

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "primary": "#00aaff",
    "secondary": 242
  },
  "colors": {
    "accent": "primary",
    "border": "primary",
    "borderAccent": "#00ffff",
    "borderMuted": "secondary",
    "success": "#00ff00",
    "error": "#ff0000",
    "warning": "#ffff00",
    "muted": "secondary",
    "dim": 240,
    "text": "",
    "thinkingText": "secondary",
    "selectedBg": "#2d2d30",
    "userMessageBg": "#2d2d30",
    "userMessageText": "",
    "customMessageBg": "#2d2d30",
    "customMessageText": "",
    "customMessageLabel": "primary",
    "toolPendingBg": "#1e1e2e",
    "toolSuccessBg": "#1e2e1e",
    "toolErrorBg": "#2e1e1e",
    "toolTitle": "primary",
    "toolOutput": "",
    "mdHeading": "#ffaa00",
    "mdLink": "primary",
    "mdLinkUrl": "secondary",
    "mdCode": "#00ffff",
    "mdCodeBlock": "",
    "mdCodeBlockBorder": "secondary",
    "mdQuote": "secondary",
    "mdQuoteBorder": "secondary",
    "mdHr": "secondary",
    "mdListBullet": "#00ffff",
    "toolDiffAdded": "#00ff00",
    "toolDiffRemoved": "#ff0000",
    "toolDiffContext": "secondary",
    "syntaxComment": "secondary",
    "syntaxKeyword": "primary",
    "syntaxFunction": "#00aaff",
    "syntaxVariable": "#ffaa00",
    "syntaxString": "#00ff00",
    "syntaxNumber": "#ff00ff",
    "syntaxType": "#00aaff",
    "syntaxOperator": "primary",
    "syntaxPunctuation": "secondary",
    "thinkingOff": "secondary",
    "thinkingMinimal": "primary",
    "thinkingLow": "#00aaff",
    "thinkingMedium": "#00ffff",
    "thinkingHigh": "#ff00ff",
    "thinkingXhigh": "#ff0000",
    "thinkingMax": "#ff0088",
    "bashMode": "#ffaa00"
  }
}
```

3. 通过 `/settings` 选择该主题。

**热重载：** 编辑当前启用的自定义主题文件时，Pi 会自动重新加载，以便立即查看视觉效果。

<a id="theme-format"></a>

## 主题格式

```json
{
  "$schema": "https://raw.githubusercontent.com/earendil-works/pi/main/packages/coding-agent/src/modes/interactive/theme/theme-schema.json",
  "name": "my-theme",
  "vars": {
    "blue": "#0066cc",
    "gray": 242
  },
  "colors": {
    "accent": "blue",
    "muted": "gray",
    "text": "",
    ...
  }
}
```

- `name` 为必填项，必须唯一，并且不能包含 `/`。
- `vars` 可选。可以在这里定义可复用的颜色，然后在 `colors` 中引用。
- `colors` 必须定义全部 51 个必需 Token。`thinkingMax` 可选；省略时使用 `thinkingXhigh`。

`$schema` 字段用于启用编辑器自动补全和验证。

<a id="color-tokens"></a>

## 颜色 Token

每个主题都必须定义全部 51 个必需的颜色 Token。为了兼容现有主题，`thinkingMax` 是可选项；省略时使用 `thinkingXhigh`。

### 核心 UI（11 种颜色）

| Token | 用途 |
|-------|---------|
| `accent` | 主强调色（Logo、选中项、光标） |
| `border` | 普通边框 |
| `borderAccent` | 高亮边框 |
| `borderMuted` | 弱化边框（编辑器） |
| `success` | 成功状态 |
| `error` | 错误状态 |
| `warning` | 警告状态 |
| `muted` | 次要文本 |
| `dim` | 第三级文本 |
| `text` | 默认文本（通常为 `""`） |
| `thinkingText` | 思考内容块文本 |

### 背景与内容（11 种颜色）

| Token | 用途 |
|-------|---------|
| `selectedBg` | 选中行的背景 |
| `userMessageBg` | 用户消息背景 |
| `userMessageText` | 用户消息文本 |
| `customMessageBg` | 扩展消息背景 |
| `customMessageText` | 扩展消息文本 |
| `customMessageLabel` | 扩展消息标签 |
| `toolPendingBg` | 工具框（等待中） |
| `toolSuccessBg` | 工具框（成功） |
| `toolErrorBg` | 工具框（错误） |
| `toolTitle` | 工具标题 |
| `toolOutput` | 工具输出文本 |

### Markdown（10 种颜色）

| Token | 用途 |
|-------|---------|
| `mdHeading` | 标题 |
| `mdLink` | 链接文本 |
| `mdLinkUrl` | 链接 URL |
| `mdCode` | 行内代码 |
| `mdCodeBlock` | 代码块内容 |
| `mdCodeBlockBorder` | 代码块围栏 |
| `mdQuote` | 引用块文本 |
| `mdQuoteBorder` | 引用块边框 |
| `mdHr` | 水平分隔线 |
| `mdListBullet` | 列表项目符号 |

### 工具差异（3 种颜色）

| Token | 用途 |
|-------|---------|
| `toolDiffAdded` | 新增行 |
| `toolDiffRemoved` | 删除行 |
| `toolDiffContext` | 上下文行 |

### 语法高亮（9 种颜色）

| Token | 用途 |
|-------|---------|
| `syntaxComment` | 注释 |
| `syntaxKeyword` | 关键字 |
| `syntaxFunction` | 函数名 |
| `syntaxVariable` | 变量 |
| `syntaxString` | 字符串 |
| `syntaxNumber` | 数字 |
| `syntaxType` | 类型 |
| `syntaxOperator` | 运算符 |
| `syntaxPunctuation` | 标点符号 |

### 推理级别边框（6 个必需项，1 个可选项）

用于表示推理级别的编辑器边框颜色（视觉层次从弱到强）：

| Token | 用途 |
|-------|---------|
| `thinkingOff` | 关闭推理 |
| `thinkingMinimal` | 最低推理级别 |
| `thinkingLow` | 低推理级别 |
| `thinkingMedium` | 中等推理级别 |
| `thinkingHigh` | 高推理级别 |
| `thinkingXhigh` | 超高推理级别 |
| `thinkingMax` | 最高推理级别；可选，省略时使用 `thinkingXhigh` |

### Bash 模式（1 种颜色）

| Token | 用途 |
|-------|---------|
| `bashMode` | Bash 模式（`!` 前缀）下的编辑器边框 |

### HTML 导出（可选）

`export` 部分控制 `/export` 所生成 HTML 的颜色。省略时，颜色会根据 `userMessageBg` 推导。

```json
{
  "export": {
    "pageBg": "#18181e",
    "cardBg": "#1e1e24",
    "infoBg": "#3c3728"
  }
}
```

<a id="color-values"></a>

## 颜色值

支持四种格式：

| 格式 | 示例 | 说明 |
|--------|---------|-------------|
| 十六进制 | `"#ff0000"` | 6 位十六进制 RGB |
| 256 色 | `39` | xterm 256 色调色板索引（0–255） |
| 变量 | `"primary"` | 引用 `vars` 中的条目 |
| 默认值 | `""` | 终端默认颜色 |

### 256 色调色板

- `0-15`：基础 ANSI 颜色（取决于终端）
- `16-231`：6×6×6 RGB 色立方（`16 + 36×R + 6×G + B`，其中 R、G、B 的取值为 0–5）
- `232-255`：灰度渐变

### 终端兼容性

Pi 使用 24 位 RGB 颜色。大多数现代终端都支持该格式（iTerm2、Kitty、WezTerm、Windows Terminal、VS Code）。对于只支持 256 色的旧终端，Pi 会回退到最接近的近似颜色。

检查真彩色支持：

```bash
echo $COLORTERM  # 应输出 "truecolor" 或 "24bit"
```

<a id="tips"></a>

## 建议

**深色终端：** 使用明亮、饱和度较高且对比度更强的颜色。

**浅色终端：** 使用更深、更柔和且对比度较低的颜色。

**色彩协调：** 从基础调色板（如 Nord、Gruvbox、Tokyo Night）开始，在 `vars` 中定义，并始终通过变量引用。

**测试：** 使用不同消息类型、工具状态、Markdown 内容和自动换行的长文本检查主题效果。

**VS Code：** 将 `terminal.integrated.minimumContrastRatio` 设为 `1`，以准确显示颜色。

## 示例

参阅内置主题：

- [dark.json](../src/modes/interactive/theme/dark.json)
- [light.json](../src/modes/interactive/theme/light.json)
