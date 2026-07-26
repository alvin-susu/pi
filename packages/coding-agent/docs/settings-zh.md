# 设置

Pi 使用 JSON 设置文件，项目设置会覆盖全局设置。

| 位置 | 作用域 |
|----------|-------|
| `~/.pi/agent/settings.json` | 全局（所有项目） |
| `.pi/settings.json` | 项目（当前目录） |

可以直接编辑文件，也可以使用 `/settings` 配置常用选项。

## 项目信任

以交互模式启动时，如果项目文件夹包含项目本地设置、资源或项目 `.agents/skills`，并且 `~/.pi/agent/trust.json` 中没有针对该文件夹或其父文件夹的已保存决定，Pi 会在信任该项目之前询问用户。信任项目后，Pi 可以加载 `.pi/settings.json` 和 `.pi` 资源、安装缺少的项目包，并执行项目扩展。

非交互模式（`-p`、`--mode json` 和 `--mode rpc`）不会显示信任提示。如果没有适用的已保存决定，它们会使用全局设置中的 `defaultProjectTrust`：`ask`（默认值）和 `never` 会忽略这些项目资源，`always` 则会信任它们。可以传入 `--approve`/`-a` 或 `--no-approve`/`-na`，仅针对本次运行覆盖项目信任。

如果没有扩展或已保存决定可用，则由 `defaultProjectTrust` 控制后备行为。可以在 `~/.pi/agent/settings.json` 中将其设为 `"ask"`、`"always"` 或 `"never"`，也可以通过 `/settings` 更改。

`pi config` 和包命令使用相同的项目信任流程，但 `pi update` 绝不会显示提示。传入 `--approve` 可以在一次命令中信任项目本地设置，传入 `--no-approve` 则会忽略它们。

在交互模式中使用 `/trust`，可以为以后的会话保存项目信任决定，其中也可以选择信任直接父目录。该命令只写入 `~/.pi/agent/trust.json`，不会重新加载当前会话，因此需要重启 Pi 才能使更改生效。

## 全部设置

### 模型与推理

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `defaultProvider` | string | - | 默认服务提供商（例如 `"anthropic"`、`"openai"`） |
| `defaultModel` | string | - | 默认模型 ID |
| `defaultThinkingLevel` | string | - | `"off"`, `"minimal"`, `"low"`, `"medium"`, `"high"`, `"xhigh"`, `"max"` |
| `hideThinkingBlock` | boolean | `false` | 在输出中隐藏思考内容块 |
| `showCacheMissNotices` | boolean | `false` | 当提示词缓存出现明显未命中时，在对话记录中显示提示 |
| `thinkingBudgets` | object | - | 为每个推理级别自定义 Token 预算 |

#### thinkingBudgets

```json
{
  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 10240,
    "high": 32768
  }
}
```

### UI 与显示

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `theme` | string | `"dark"` | 主题名称（`"dark"`、`"light"` 或自定义主题） |
| `externalEditor` | string | 依次为 `$VISUAL`、`$EDITOR`、Windows 上的记事本或其他平台上的 `nano` | Ctrl+G 打开的外部编辑器命令；优先于环境变量 |
| `quietStartup` | boolean | `false` | 隐藏启动标题区 |
| `defaultProjectTrust` | string | `"ask"` | 项目信任的后备行为：`"ask"`、`"always"` 或 `"never"`。仅限全局设置 |
| `collapseChangelog` | boolean | `false` | 更新后显示精简的变更日志 |
| `enableInstallTelemetry` | boolean | `true` | 首次安装或根据变更日志检测到更新后，发送匿名的安装/更新版本上报。该设置不控制更新检查 |
| `enableAnalytics` | boolean | `false` | 选择加入分析数据共享。目前只会在实验性首次配置（`PI_EXPERIMENTAL=1`）期间询问 |
| `trackingId` | string | - | 分析跟踪标识符；启用 `enableAnalytics` 时生成 |
| `doubleEscapeAction` | string | `"tree"` | 连按两次 Escape 时执行的操作：`"tree"`、`"fork"` 或 `"none"` |
| `treeFilterMode` | string | `"default"` | `/tree` 的默认筛选器：`"default"`、`"no-tools"`、`"user-only"`、`"labeled-only"`、`"all"` |
| `editorPaddingX` | number | `0` | 输入编辑器的水平内边距（0–3） |
| `outputPad` | number | `1` | 用户消息、助手消息和思考内容的水平内边距（0 或 1） |
| `autocompleteMaxVisible` | number | `5` | 自动补全下拉列表最多显示的条目数（3–20） |
| `showHardwareCursor` | boolean | `false` | TUI 为支持输入法而定位光标时，显示终端硬件光标 |

使用 VS Code 时，请加入 `--wait`，以便编辑器退出后 Pi 继续运行：

```json
{
  "externalEditor": "code --wait"
}
```

### 遥测和更新检查

`enableInstallTelemetry` 只控制发送到 `https://pi.dev/api/report-install` 的匿名安装/更新上报。退出遥测不会禁用更新检查；Pi 仍可能请求 `https://pi.dev/api/latest-version` 查找最新版本。

设置 `PI_SKIP_VERSION_CHECK=1` 可以禁用 Pi 版本更新检查。使用 `--offline` 或 `PI_OFFLINE=1` 可以禁用本文提到的全部启动网络操作，包括更新检查、包更新检查和安装/更新遥测。

### 网络

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `httpProxy` | string | - | 同时用作 `HTTP_PROXY` 和 `HTTPS_PROXY` 的 HTTP 代理 URL。仅限全局设置。 |

```json
{
  "httpProxy": "http://127.0.0.1:7890"
}
```

### 警告

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `warnings.anthropicExtraUsage` | boolean | `true` | Anthropic 订阅认证可能使用付费额外用量时显示警告 |

```json
{
  "warnings": {
    "anthropicExtraUsage": false
  }
}
```

### 上下文压缩

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `compaction.enabled` | boolean | `true` | 启用自动上下文压缩 |
| `compaction.reserveTokens` | number | `16384` | 为 LLM 回复保留的 Token 数 |
| `compaction.keepRecentTokens` | number | `20000` | 保留且不纳入摘要的近期 Token 数 |

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

### 分支摘要

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `branchSummary.reserveTokens` | number | `16384` | 为分支摘要保留的 Token 数 |
| `branchSummary.skipPrompt` | boolean | `false` | 使用 `/tree` 导航时跳过“是否总结分支？”提示（默认不生成摘要） |

### 重试

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `retry.enabled` | boolean | `true` | 发生暂时性错误时，启用智能体层自动重试 |
| `retry.maxRetries` | number | `3` | 智能体层最大重试次数 |
| `retry.baseDelayMs` | number | `2000` | 智能体层指数退避的基础延迟（2 秒、4 秒、8 秒） |
| `retry.provider.timeoutMs` | number | SDK 默认值 | 服务提供商/SDK 请求超时，单位为毫秒 |
| `retry.provider.maxRetries` | number | `0` | 服务提供商/SDK 重试次数 |
| `retry.provider.maxRetryDelayMs` | number | `60000` | 失败前允许服务器要求的最长等待时间（60 秒） |

如果服务提供商要求的重试延迟超过 `retry.provider.maxRetryDelayMs`，请求会立即失败并给出明确错误，而不是静默等待。设为 `0` 可以禁用该限制。

除非明确需要服务提供商层重试，否则应将 `retry.provider.maxRetries` 保持为 `0`。如果设为大于 `0`，SDK/服务提供商可能会在 Pi 收到错误之前自行处理超出用量限额的错误；在某些情况下，这会使智能体一直阻塞到服务提供商配额重置。

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000,
    "provider": {
      "timeoutMs": 3600000,
      "maxRetries": 0,
      "maxRetryDelayMs": 60000
    }
  }
}
```

### 消息投递

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `steeringMode` | string | `"one-at-a-time"` | 引导消息的发送方式：`"all"` 或 `"one-at-a-time"` |
| `followUpMode` | string | `"one-at-a-time"` | 后续消息的发送方式：`"all"` 或 `"one-at-a-time"` |
| `transport` | string | `"auto"` | 对支持多种传输方式的服务提供商，首选的传输方式：`"sse"`、`"websocket"`、`"websocket-cached"` 或 `"auto"` |
| `httpIdleTimeoutMs` | number | `300000` | HTTP 请求头/正文空闲超时，单位为毫秒；也供显式设置流空闲超时的服务提供商使用。设为 `0` 可禁用。 |
| `websocketConnectTimeoutMs` | number | `15000` | 对支持 WebSocket 传输的服务提供商，WebSocket 连接/打开握手的超时时间，单位为毫秒。设为 `0` 可禁用。 |

### 终端与图片

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `terminal.showImages` | boolean | `true` | 在终端中显示图片（如果支持） |
| `terminal.imageWidthCells` | number | `60` | 行内图片的首选宽度，以终端字符单元为单位 |
| `terminal.clearOnShrink` | boolean | `false` | 内容缩小时清除空行（可能导致闪烁） |
| `images.autoResize` | boolean | `true` | 将图片最大尺寸调整为 2000×2000 |
| `images.blockImages` | boolean | `false` | 阻止向 LLM 发送任何图片 |

### Shell

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `shellPath` | string | - | 自定义 Shell 路径（例如 Windows 上的 Cygwin）；支持以 `~` 表示主目录 |
| `shellCommandPrefix` | string | - | 添加到每条 bash 命令之前的前缀（例如 `"shopt -s expand_aliases"`） |
| `npmCommand` | string[] | - | npm 包查询/安装操作使用的命令参数数组（例如 `["mise", "exec", "node@20", "--", "npm"]`） |

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

`npmCommand` 用于所有 npm 包管理操作，包括安装、卸载以及 Git 包内部的依赖项安装。用户作用域的 npm 包安装到 `~/.pi/agent/npm/`；项目作用域的 npm 包安装到 `.pi/npm/`。请按照进程实际启动时的 argv 形式逐项填写。配置 `npmCommand` 后，安装 Git 包依赖项时只使用普通的 `install`，避免向包装命令或其他包管理器传递 npm 专用标志。

### 会话

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `sessionDir` | string | - | 会话文件存储目录。支持绝对路径、相对路径和 `~`。 |

```json
{ "sessionDir": ".pi/sessions" }
```

如果多个来源都指定了会话目录，优先级依次为 `--session-dir`、`PI_CODING_AGENT_SESSION_DIR`、settings.json 中的 `sessionDir`。

### 模型循环切换

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `enabledModels` | string[] | - | 供 Ctrl+P 循环切换的模型模式（格式与 CLI `--models` 标志相同） |

```json
{
  "enabledModels": ["claude-*", "gpt-4o", "gemini-2*"]
}
```

### Markdown

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `markdown.codeBlockIndent` | string | `"  "` | 代码块缩进 |

### 资源

以下设置定义扩展、技能、提示词和主题的加载位置。

`~/.pi/agent/settings.json` 中的路径相对于 `~/.pi/agent` 解析；`.pi/settings.json` 中的路径相对于 `.pi` 解析。同时支持绝对路径和 `~`。

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `packages` | array | `[]` | 用于加载资源的 npm/Git 包 |
| `extensions` | string[] | `[]` | 本地扩展文件路径或目录 |
| `skills` | string[] | `[]` | 本地技能文件路径或目录 |
| `prompts` | string[] | `[]` | 本地提示词模板路径或目录 |
| `themes` | string[] | `[]` | 本地主题文件路径或目录 |
| `enableSkillCommands` | boolean | `true` | 将技能注册为 `/skill:name` 命令 |

数组支持 Glob 模式和排除规则。使用 `!pattern` 排除匹配项；使用 `+path` 强制包含精确路径；使用 `-path` 强制排除精确路径。

#### packages

字符串形式会加载包中的全部资源：

```json
{
  "packages": ["pi-skills", "@org/my-extension"]
}
```

对象形式可以筛选要加载的资源：

```json
{
  "packages": [
    {
      "source": "pi-skills",
      "skills": ["brave-search", "transcribe"],
      "extensions": []
    }
  ]
}
```

包管理详情请参阅 [packages.md](packages-zh.md)。

## 示例

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "theme": "dark",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  },
  "enabledModels": ["claude-*", "gpt-4o"],
  "warnings": {
    "anthropicExtraUsage": true
  },
  "packages": ["pi-skills"]
}
```

## 项目覆盖设置

项目设置（`.pi/settings.json`）会覆盖全局设置。嵌套对象采用合并方式：

```json
// ~/.pi/agent/settings.json（全局）
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 16384 }
}

// .pi/settings.json（项目）
{
  "compaction": { "reserveTokens": 8192 }
}

// 结果
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 8192 }
}
```
