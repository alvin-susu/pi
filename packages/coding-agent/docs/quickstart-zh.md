# 快速开始

本文将引导你完成安装，并开始第一个真正可用的 Pi 会话。

## 安装

Pi 以 npm 包的形式发布：

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` 会在安装期间禁用依赖项的生命周期脚本。通过 npm 正常安装 Pi 不需要运行安装脚本。

<a id="uninstall"></a>

### 卸载

请使用当初安装 Pi 的包管理器进行卸载。curl 安装脚本在内部使用 npm 全局安装，因此通过 curl 或 npm 安装的 Pi 都应使用 npm 卸载：

```bash
# curl 安装脚本或 npm install -g
npm uninstall -g @earendil-works/pi-coding-agent

# pnpm
pnpm remove -g @earendil-works/pi-coding-agent

# Yarn
yarn global remove @earendil-works/pi-coding-agent

# Bun
bun uninstall -g @earendil-works/pi-coding-agent
```

卸载 Pi 不会删除 `~/.pi/agent/` 中的设置、凭据、会话和已安装的 Pi 包。

然后在希望 Pi 操作的项目目录中启动：

```bash
cd /path/to/project
pi
```

## 认证

Pi 可以通过 `/login` 使用订阅制服务提供商，也可以通过环境变量或认证文件使用 API 密钥型服务提供商。

### 方案一：登录订阅账号

启动 Pi 并运行：

```text
/login
```

然后选择服务提供商。内置的订阅登录方式包括 Claude Pro/Max、ChatGPT Plus/Pro（Codex）和 GitHub Copilot。

### 方案二：API 密钥

启动 Pi 前设置 API 密钥：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

也可以运行 `/login` 并选择 API 密钥型服务提供商，将密钥保存到 `~/.pi/agent/auth.json`。

所有受支持的服务提供商、环境变量和云服务配置详见[服务提供商](providers-zh.md)。

## 第一个会话

Pi 启动后，输入请求并按 Enter：

```text
总结这个仓库，并告诉我如何运行项目检查。
```

默认情况下，Pi 会向模型提供四个工具：

- `read`——读取文件
- `write`——创建或覆盖文件
- `edit`——修改文件
- `bash`——运行 Shell 命令

还可以通过工具选项启用其他内置只读工具（`grep`、`find`、`ls`）。Pi 在当前工作目录中运行，并且可以修改其中的文件。如果希望方便地回滚，请使用 Git 或其他检查点工作流程。

## 为 Pi 提供项目说明

Pi 会在启动时加载上下文文件。可以添加 `AGENTS.md`，说明它在项目中应如何工作：

```markdown
# 项目说明

- 修改代码后运行 `npm run check`。
- 不要在本地运行生产环境迁移。
- 回答保持简洁。
```

Pi 会加载：

- 用作全局说明的 `~/.pi/agent/AGENTS.md`
- 父目录和当前目录中的 `AGENTS.md` 或 `CLAUDE.md`

修改上下文文件后，请重启 Pi 或运行 `/reload`。

## 常用操作

### 引用文件

在编辑器中输入 `@` 可以模糊搜索文件，也可以通过命令行传入文件：

```bash
pi @README.md "总结这个文件"
pi @src/app.ts @src/app.test.ts "一起检查这两个文件"
```

可以使用 Ctrl+V（Windows 上为 Alt+V）粘贴图片或文本；在支持的终端中，还可以直接拖入图片。

### 运行 Shell 命令

在交互模式中：

```text
!npm run lint
```

命令输出会发送给模型。使用 `!!command` 可以执行命令，但不将输出加入模型上下文。

### 切换模型

使用 `/model` 或 Ctrl+L 选择模型。使用 Shift+Tab 循环切换推理级别。使用 Ctrl+P / Shift+Ctrl+P 在作用域模型之间循环切换。

### 稍后继续

会话会自动保存：

```bash
pi -c                  # 继续最近的会话
pi -r                  # 浏览以前的会话
pi --name "my task"    # 启动时设置会话显示名称
pi --session <path|id> # 打开指定会话
```

在 Pi 内部，可以使用 `/resume`、`/new`、`/tree`、`/fork` 和 `/clone` 管理会话。

### 非交互模式

对于一次性提示词：

```bash
pi -p "总结这个代码库"
cat README.md | pi -p "总结这段文字"
pi -p @screenshot.png "这张图片里有什么？"
```

使用 `--mode json` 输出 JSON 事件，或使用 `--mode rpc` 进行进程集成。

## 后续阅读

- [使用 Pi](usage-zh.md)——交互模式、斜杠命令、会话、上下文文件和 CLI 参考。
- [服务提供商](providers-zh.md)——认证和模型配置。
- [设置](settings-zh.md)——全局配置和项目配置。
- [键位绑定](keybindings-zh.md)——快捷键和自定义方法。
- [Pi 包](packages-zh.md)——安装共享的扩展、技能、提示词和主题。

平台说明：[Windows](windows-zh.md)、[Termux](termux-zh.md)、[tmux](tmux-zh.md)、[终端配置](terminal-setup-zh.md)、[Shell 别名](shell-aliases-zh.md)。
