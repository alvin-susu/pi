# 使用 Pi

本文汇总日常使用 Pi 时需要了解、但不适合放在快速开始页面中的详细信息。

## 交互模式

<p align="center"><img src="images/interactive-mode.png" alt="交互模式" width="600"></p>

界面主要分为四个区域：

- **启动标题区**——显示快捷键，以及已加载的上下文文件、提示词模板、技能和扩展
- **消息区**——显示用户消息、助手回复、工具调用、工具结果、通知、错误和扩展界面
- **编辑器**——输入内容的区域；边框颜色表示当前推理级别
- **页脚**——显示工作目录、会话名称、Token/缓存用量、费用、上下文用量和当前模型。总计数据包括助手回复、工具报告的用量以及摘要生成

编辑器可以暂时被 `/settings` 等内置界面或自定义扩展界面替换。

### 编辑器功能

| 功能 | 使用方法 |
|---------|-----|
| 引用文件 | 输入 `@` 对项目文件进行模糊搜索 |
| 路径补全 | 按 Tab 补全路径 |
| 多行输入 | Shift+Enter；Windows Terminal 中也可以使用 Ctrl+Enter |
| 复制回复 | Ctrl+X 复制最后一条助手消息；在 `/tree` 中复制所选消息 |
| 图片 | 使用 Ctrl+V（Windows 上为 Alt+V）粘贴，或将图片拖入终端 |
| Shell 命令 | `!command` 执行命令并将输出发送给模型 |
| 隐藏 Shell 命令 | `!!command` 执行命令，但不将输出发送给模型 |
| 外部编辑器 | Ctrl+G 打开 `externalEditor`、`$VISUAL`、`$EDITOR`、Windows 上的记事本，或其他平台上的 `nano` |

全部快捷键和自定义方法详见[键位绑定](keybindings-zh.md)。

## 斜杠命令

在编辑器中输入 `/` 可以打开命令补全。扩展可以注册自定义命令；技能以 `/skill:name` 的形式提供；提示词模板则通过 `/templatename` 展开。

| 命令 | 说明 |
|---------|-------------|
| `/login`, `/logout` | 管理 OAuth 或 API 密钥凭据 |
| [`/llama`](llama-cpp-zh.md) | 下载、加载和卸载 llama.cpp 路由模型 |
| `/model` | 切换模型 |
| `/scoped-models` | 启用/禁用 Ctrl+P 循环切换范围内的模型 |
| `/settings` | 配置推理级别、主题、消息投递和传输方式 |
| `/resume` | 从以前的会话中选择 |
| `/new` | 启动新会话 |
| `/name <name>` | 设置会话显示名称 |
| `/session` | 显示会话文件、ID、消息数、Token 和费用 |
| `/tree` | 跳转到会话中的任意节点并从那里继续 |
| `/trust` | 保存项目信任决定，供以后的会话使用 |
| `/fork` | 从以前的用户消息创建新会话 |
| `/clone` | 将当前活动分支复制为新会话 |
| `/compact [prompt]` | 手动压缩上下文，并可指定自定义指令 |
| `/copy` | 将最后一条助手消息复制到剪贴板 |
| `/export [file]` | 将会话导出为 HTML 或 JSONL |
| `/import <file>` | 从 JSONL 文件导入并恢复会话 |
| `/share` | 上传为私有 GitHub Gist，并生成可共享的 HTML 链接 |
| `/reload` | 重新加载键位绑定、扩展、技能、提示词、主题和上下文文件 |
| `/hotkeys` | 显示全部键盘快捷键 |
| `/changelog` | 显示版本历史 |
| `/quit` | 退出 Pi |

## 消息队列

智能体仍在工作时，也可以提交消息：

- **Enter** 将引导消息加入队列；当前助手轮次完成所有工具调用后投递。
- **Alt+Enter** 将后续消息加入队列；智能体完成全部工作后投递。
- **Escape** 中止当前操作，并将已入队消息恢复到编辑器。
- **Alt+Up** 将已入队消息取回编辑器。

Windows Terminal 默认将 Alt+Enter 用于全屏。如果希望 Pi 收到该快捷键，请按照[终端配置](terminal-setup-zh.md)中的说明重新映射。

可以在[设置](settings-zh.md)中通过 `steeringMode` 和 `followUpMode` 配置投递方式。

## 会话

会话会自动保存到 `~/.pi/agent/sessions/`，并按工作目录组织。

```bash
pi -c                  # 继续最近的会话
pi -r                  # 浏览并选择会话
pi --no-session        # 临时模式；不保存
pi --name "my task"    # 启动时设置会话显示名称
pi --session <path|id> # 使用指定会话文件或会话 ID
pi --fork <path|id>    # 将会话派生为新的会话文件
```

常用会话命令：

- `/session` 显示当前会话文件和 ID。
- `/tree` 用于在文件内部的会话树中导航，并可总结离开的分支。
- `/fork` 从较早的用户消息创建新会话。
- `/clone` 将当前活动分支复制为新的会话文件。
- `/compact` 总结较早的消息，以释放上下文空间。

详细信息请参阅[会话](sessions-zh.md)和[上下文压缩](compaction-zh.md)。

## 上下文文件

Pi 启动时会从以下位置加载 `AGENTS.md` 或 `CLAUDE.md`：

- 用于全局说明的 `~/.pi/agent/AGENTS.md`
- 从当前工作目录向上遍历的父目录
- 当前目录

上下文文件适合记录项目约定、命令、安全规则和偏好。使用 `--no-context-files` 或 `-nc` 可以禁用加载。

### 系统提示词文件

使用以下文件替换默认系统提示词：

- 项目级 `.pi/SYSTEM.md`
- 全局 `~/.pi/agent/SYSTEM.md`

在任一位置使用 `APPEND_SYSTEM.md`，可以在默认提示词后追加内容，而不替换它。

### 项目信任

以交互模式启动时，如果项目文件夹包含项目本地设置、资源或项目 `.agents/skills`，并且 `~/.pi/agent/trust.json` 中没有针对该文件夹或其父文件夹的已保存决定，Pi 会在信任该项目之前询问用户。信任项目后，Pi 可以加载 `.pi/settings.json` 和 `.pi` 资源、安装缺少的项目包，并执行项目扩展。

信任状态确定之前，Pi 只加载上下文文件、用户级/全局扩展和 CLI `-e` 扩展，以便它们处理 `project_trust` 事件。只有项目受信任后，才会加载项目本地扩展、由项目包管理的扩展和项目设置。如果切换到另一个工作目录中的会话，而该目录的信任状态尚未在当前进程中确定，也会采用同样的分阶段加载方式。

非交互模式（`-p`、`--mode json` 和 `--mode rpc`）不会显示信任提示。如果没有适用的已保存决定，它们会使用全局设置中的 `defaultProjectTrust`：`ask`（默认值）和 `never` 会忽略这些项目资源，`always` 则会信任它们。可以传入 `--approve`/`-a` 或 `--no-approve`/`-na`，仅针对本次运行覆盖项目信任。

如果没有扩展或已保存决定可用，则由 `defaultProjectTrust` 控制后备行为。可以在 `~/.pi/agent/settings.json` 中将其设为 `"ask"`、`"always"` 或 `"never"`，也可以通过 `/settings` 更改。

`pi config` 和包命令使用相同的项目信任流程，但 `pi update` 绝不会显示提示。传入 `--approve` 可以在一次命令中信任项目本地设置，传入 `--no-approve` 则会忽略它们。

在交互模式中使用 `/trust`，可以为以后的会话保存项目信任决定，其中也可以选择信任直接父目录。该命令只写入 `~/.pi/agent/trust.json`，不会重新加载当前会话，因此需要重启 Pi 才能使更改生效。

## 导出和共享会话

使用 `/export [file]` 将会话写入 HTML 文件。

使用 `/share` 上传私有 GitHub Gist，并生成可共享的 HTML 链接。

如果使用 Pi 参与开源项目，并希望发布会话以供模型、提示词、工具和评估研究使用，请参阅 [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf)。该工具可以将会话发布到 Hugging Face 数据集。

## CLI 参考

```bash
pi [options] [@files...] [messages...]
```

### 包命令

```bash
pi install <source> [-l]     # 安装包；-l 表示项目本地
pi remove <source> [-l]      # 移除包
pi uninstall <source> [-l]   # remove 的别名
pi update [source|self|pi]   # 仅更新 Pi，或更新单个包来源
pi update --all              # 更新 Pi 和包；同步固定的 Git 引用
pi update --extensions       # 仅更新包；同步固定的 Git 引用
pi update --models           # 仅刷新模型目录
pi update --self             # 仅更新 Pi
pi update --extension <src>  # 更新单个包
pi list                      # 列出已安装的包
pi config                    # 启用/禁用包资源
```

这些命令用于管理 Pi 包，`pi update` 还可以更新 Pi CLI 本身。要卸载 Pi，请参阅[快速开始](quickstart-zh.md#uninstall)。`pi config` 和项目包命令接受 `--approve`/`--no-approve`，以便仅在一次命令中信任或忽略项目本地设置。`pi update` 绝不会显示项目信任提示。

包来源和安全说明详见 [Pi 包](packages-zh.md)。

### 模式

| 标志 | 说明 |
|------|-------------|
| 默认 | 交互模式 |
| `-p`, `--print` | 输出回复后退出 |
| `--mode json` | 将全部事件输出为逐行 JSON；参阅 [JSON 模式](json-zh.md) |
| `--mode rpc` | 通过标准输入/标准输出运行 RPC 模式；参阅 [RPC 模式](rpc-zh.md) |
| `--export <in> [out]` | 将会话导出为 HTML |

在打印模式中，Pi 还会读取管道传入的标准输入，并将其合并到初始提示词中：

```bash
cat README.md | pi -p "总结这段文字"
```

### 模型选项

| 选项 | 说明 |
|--------|-------------|
| `--provider <name>` | 服务提供商，例如 `anthropic`、`openai` 或 `google` |
| `--model <pattern>` | 模型模式或 ID；支持 `provider/id` 和可选的 `:<thinking>` |
| `--api-key <key>` | API 密钥，优先于环境变量 |
| `--thinking <level>` | `off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max` |
| `--models <patterns>` | 用逗号分隔、供 Ctrl+P 循环切换的模型模式 |
| `--list-models [search]` | 列出可用模型 |

### 会话选项

| 选项 | 说明 |
|--------|-------------|
| `-c`, `--continue` | 继续最近的会话 |
| `-r`, `--resume` | 浏览并选择会话 |
| `--session <path\|id>` | 使用指定会话文件或部分 UUID |
| `--fork <path\|id>` | 将会话文件或部分 UUID 派生为新会话 |
| `--session-dir <dir>` | 自定义会话存储目录 |
| `--no-session` | 临时模式；不保存 |
| `--name <name>`, `-n <name>` | 启动时设置会话显示名称 |

### 工具选项

| 选项 | 说明 |
|--------|-------------|
| `--tools <list>`, `-t <list>` | 将指定的内置、扩展和自定义工具加入允许列表 |
| `--exclude-tools <list>`, `-xt <list>` | 禁用指定的内置、扩展和自定义工具 |
| `--no-builtin-tools`, `-nbt` | 禁用内置工具，但保留扩展/自定义工具 |
| `--no-tools`, `-nt` | 禁用全部工具 |

内置工具：`read`、`bash`、`edit`、`write`、`grep`、`find`、`ls`。

### 资源选项

| 选项 | 说明 |
|--------|-------------|
| `-e`, `--extension <source>` | 从路径、npm 或 Git 加载扩展；可重复指定 |
| `--no-extensions` | 禁用扩展发现 |
| `--skill <path>` | 加载技能；可重复指定 |
| `--no-skills` | 禁用技能发现 |
| `--prompt-template <path>` | 加载提示词模板；可重复指定 |
| `--no-prompt-templates` | 禁用提示词模板发现 |
| `--theme <path>` | 加载主题；可重复指定 |
| `--no-themes` | 禁用主题发现 |
| `--no-context-files`, `-nc` | 禁止发现 `AGENTS.md` 和 `CLAUDE.md` |

将 `--no-*` 与显式标志组合使用，可以忽略设置，只加载所需内容。例如：

```bash
pi --no-extensions -e ./my-extension.ts
```

### 其他选项

| 选项 | 说明 |
|--------|-------------|
| `--system-prompt <text>` | 替换默认提示词；仍会追加上下文文件和技能 |
| `--append-system-prompt <text>` | 追加到系统提示词 |
| `--verbose` | 强制显示详细启动信息 |
| `-a`, `--approve` | 在本次运行中信任项目本地文件 |
| `-na`, `--no-approve` | 在本次运行中忽略项目本地文件 |
| `-h`, `--help` | 显示帮助 |
| `-v`, `--version` | 显示版本 |

### 文件参数

在文件前加上 `@`，可以将其包含在消息中：

```bash
pi @prompt.md "回答这个问题"
pi -p @screenshot.png "这张图片里有什么？"
pi @code.ts @test.ts "检查这些文件"
```

### 示例

```bash
# 带初始提示词的交互模式
pi "列出 src/ 中的所有 .ts 文件"

# 非交互模式
pi -p "总结这个代码库"

# 通过管道传入标准输入的非交互模式
cat README.md | pi -p "总结这段文字"

# 已命名的一次性会话
pi --name "发布审查" -p "审查这个仓库"

# 使用不同模型
pi --provider openai --model gpt-4o "帮我重构"

# 带服务提供商前缀的模型
pi --model openai/gpt-4o "帮我重构"

# 带推理级别简写的模型
pi --model sonnet:high "解决这个复杂问题"

# 限定循环切换的模型
pi --models "claude-*,gpt-4o"

# 只读模式
pi --tools read,grep,find,ls -p "检查代码"

# 禁用一个扩展或内置工具，同时保留其他工具
pi --exclude-tools ask_question
```

## 设计原则

Pi 保持核心小巧，并将特定于工作流程的行为放入扩展、技能、提示词模板和包中。

Pi 有意不内置 MCP、子智能体、权限弹窗、计划模式、待办事项或后台 bash。你可以将这些工作流程构建或安装为扩展/包，也可以使用容器和 tmux 等外部工具。

完整的设计理由请阅读这篇[博客文章](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)。
