> Pi 可以帮助你创建 Pi 包。你可以让它把扩展、技能、提示词模板或主题打包在一起。

# Pi 包

Pi 包用于打包扩展、技能、提示词模板和主题，以便通过 npm 或 Git 共享。包可以在 `package.json` 的 `pi` 键下声明资源，也可以使用约定目录。

## 目录

- [安装与管理](#install-and-manage)
- [包来源](#package-sources)
- [创建 Pi 包](#creating-a-pi-package)
- [包结构](#package-structure)
- [依赖项](#dependencies)
- [包筛选](#package-filtering)
- [启用和禁用资源](#enable-and-disable-resources)
- [作用域与去重](#scope-and-deduplication)

<a id="install-and-manage"></a>

## 安装与管理

> **安全提示：** Pi 包以完整的系统访问权限运行。扩展可以执行任意代码，技能也可以指示模型执行包括运行可执行文件在内的任意操作。安装第三方包之前，请先检查其源代码。

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo  # 也支持原始 URL
pi install /absolute/path/to/package
pi install ./relative/path/to/package

pi remove npm:@foo/bar
pi list                     # 显示设置中已安装的包
pi update                   # 仅更新 Pi
pi update --all             # 更新 Pi 和包，并同步固定的 Git 引用
pi update --extensions      # 仅更新包并同步固定的 Git 引用
pi update --models          # 仅刷新模型目录
pi update --self            # 仅更新 Pi
pi update --self --force    # 即使已经是当前版本，也重新安装 Pi
pi update npm:@foo/bar      # 更新单个包
pi update --extension npm:@foo/bar
```

这些命令用于管理 Pi 包，`pi update` 还可以更新 Pi CLI 本身。要卸载 Pi，请参阅[快速开始](quickstart-zh.md#uninstall)。

默认情况下，`install` 和 `remove` 会写入用户设置（`~/.pi/agent/settings.json`）。使用 `-l` 可以改为写入项目设置（`.pi/settings.json`）。项目设置可以与团队共享；项目受信任后，Pi 会在启动时自动安装其中缺少的包。

如果只想试用包而不正式安装，可以使用 `--extension` 或 `-e`。包会安装到临时目录，并且只在本次运行中有效：

```bash
pi -e npm:@foo/bar
pi -e git:github.com/user/repo
```

<a id="package-sources"></a>

## 包来源

Pi 在设置和 `pi install` 中接受三种来源。

### npm

```
npm:@scope/pkg@1.2.3
npm:pkg
```

- 带版本的说明符会被固定；执行包更新（`pi update --extensions`、`pi update --all`）时会跳过它们。
- 用户级安装位于 `~/.pi/agent/npm/`。
- 项目级安装位于 `.pi/npm/`。
- 在 `settings.json` 中设置 `npmCommand`，可以让 npm 包查询和安装操作固定使用 `mise`、`asdf` 等特定包装命令。

示例：

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

### git

```
git:github.com/user/repo@v1
git:git@github.com:user/repo@v1
https://github.com/user/repo@v1
ssh://git@github.com/user/repo@v1
```

- 不带 `git:` 前缀时，只接受带协议的 URL（`https://`、`http://`、`ssh://`、`git://`）。
- 带有 `git:` 前缀时，可以使用简写格式，包括 `github.com/user/repo` 和 `git@github.com:user/repo`。
- 同时支持 HTTPS 和 SSH URL。
- SSH URL 会自动使用已配置的 SSH 密钥，并遵循 `~/.ssh/config`。
- 在 CI 等非交互运行环境中，可以设置 `GIT_TERMINAL_PROMPT=0` 禁用凭据提示，并设置 `GIT_SSH_COMMAND`（例如 `ssh -o BatchMode=yes -o ConnectTimeout=5`），以便在失败时快速退出。
- 引用必须是固定的标签或提交。`pi update --extensions` 和 `pi update --all` 不会将其移动到更新的引用，但会使现有克隆与已配置的引用保持一致。
- 使用 `pi install git:host/user/repo@new-ref` 可以更新设置，并将现有包移动到新的固定引用。
- 全局包克隆到 `~/.pi/agent/git/<host>/<path>`，项目包克隆到 `.pi/git/<host>/<path>`。
- 如果同步操作更改了检出内容，Pi 会重置并清理该克隆；如果其中存在 `package.json`，还会运行 `npm install`。

**SSH 示例：**

```bash
# git@host:path 简写（需要 git: 前缀）
pi install git:git@github.com:user/repo

# ssh:// 协议格式
pi install ssh://git@github.com/user/repo

# 带版本引用
pi install git:git@github.com:user/repo@v1.0.0
```

### 本地路径

```
/absolute/path/to/package
./relative/path/to/package
```

本地路径指向磁盘上的文件或目录，Pi 会将路径直接加入设置，而不会复制内容。相对路径以其所在的设置文件为基准解析。如果路径指向文件，则将其作为单个扩展加载；如果指向目录，则按照包规则加载资源。

<a id="creating-a-pi-package"></a>

## 创建 Pi 包

可以在 `package.json` 中添加 `pi` 清单，也可以使用约定目录。加入 `pi-package` 关键字，便于他人发现该包。

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

路径相对于包根目录。数组支持 Glob 模式和 `!排除模式`。

### 包画廊元数据

[包画廊](https://pi.dev/packages)会展示带有 `pi-package` 标签的包。添加 `video` 或 `image` 字段可以显示预览：

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "video": "https://example.com/demo.mp4",
    "image": "https://example.com/screenshot.png"
  }
}
```

- **video**：只支持 MP4。在桌面端，鼠标悬停时会自动播放；点击后打开全屏播放器。
- **image**：支持 PNG、JPEG、GIF 或 WebP，以静态预览形式显示。

如果两者同时设置，视频优先。

<a id="package-structure"></a>

## 包结构

### 约定目录

如果没有 `pi` 清单，Pi 会自动从以下目录发现资源：

- `extensions/`：加载 `.ts` 和 `.js` 文件
- `skills/`：递归查找包含 `SKILL.md` 的目录，并将顶层 `.md` 文件作为技能加载
- `prompts/`：加载 `.md` 文件
- `themes/`：加载 `.json` 文件

<a id="dependencies"></a>

## 依赖项

第三方运行时依赖项应放在 `package.json` 的 `dependencies` 中。不负责注册扩展、技能、提示词模板或主题的依赖项，也应放在 `dependencies` 中。Pi 从 npm 或 Git 安装包时会运行 `npm install`，因此这些依赖项会自动安装。

Pi 已经为扩展和技能提供核心包。如果需要导入以下任意包，请使用 `"*"` 版本范围将其列入 `peerDependencies`，并且不要将其打包：`@earendil-works/pi-ai`、`@earendil-works/pi-agent-core`、`@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui`、`typebox`。

其他 Pi 包必须一并打入你的 tar 包。将它们加入 `dependencies` 和 `bundledDependencies`，然后通过 `node_modules/` 路径引用其资源。Pi 会以相互独立的模块根目录加载各个包，因此不同安装之间不会发生模块冲突或共享。

示例：

```json
{
  "dependencies": {
    "shitty-extensions": "^1.0.1"
  },
  "bundledDependencies": ["shitty-extensions"],
  "pi": {
    "extensions": ["extensions", "node_modules/shitty-extensions/extensions"],
    "skills": ["skills", "node_modules/shitty-extensions/skills"]
  }
}
```

<a id="package-filtering"></a>

## 包筛选

在设置中使用对象形式，可以筛选包要加载的内容：

```json
{
  "packages": [
    "npm:simple-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"],
      "themes": ["+themes/legacy.json"]
    }
  ]
}
```

`+path` 和 `-path` 是相对于包根目录的精确路径。

- 省略某个键，会加载该类型的全部资源。
- 使用 `[]`，不会加载该类型的任何资源。
- `!pattern` 排除匹配项。
- `+path` 强制包含精确路径。
- `-path` 强制排除精确路径。
- 筛选器叠加在清单之上，只能进一步缩小清单已经允许的范围。

<a id="enable-and-disable-resources"></a>

## 启用和禁用资源

使用 `pi config` 可以启用或禁用已安装包和本地目录中的扩展、技能、提示词模板与主题。`pi config` 默认从全局设置（`~/.pi/agent/settings.json`）开始；按 Tab 可以在全局模式和项目本地模式之间切换。使用 `pi config -l` 可以从项目覆盖设置（`.pi/settings.json`）开始，并以暗色显示继承的全局资源。

<a id="scope-and-deduplication"></a>

## 作用域与去重

同一个包可以同时出现在全局设置和项目设置中。如果两处存在相同的包，项目条目优先；但如果项目条目设置了 `autoload: false`，则会把它作为全局条目的增量变更应用。包的身份按以下规则确定：

- npm：包名
- Git：去掉引用后的仓库 URL
- 本地路径：解析后的绝对路径
