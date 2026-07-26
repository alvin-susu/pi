> Pi 可以创建提示词模板。你可以让它根据自己的工作流程构建一个模板。

# 提示词模板

提示词模板是可以展开为完整提示词的 Markdown 片段。在编辑器中输入 `/name` 即可调用模板，其中 `name` 是去掉 `.md` 后的文件名。

## 存放位置

Pi 会从以下位置加载提示词模板：

- 全局：`~/.pi/agent/prompts/*.md`
- 项目：`.pi/prompts/*.md`（仅在项目受信任后加载）
- 包：`prompts/` 目录，或 `package.json` 中的 `pi.prompts` 条目
- 设置：`prompts` 数组中指定的文件或目录
- CLI：`--prompt-template <path>`（可重复指定）

使用 `--no-prompt-templates` 可以禁用自动发现。

## 格式

```markdown
---
description: 检查已暂存的 Git 更改
---
检查已暂存的更改（`git diff --cached`），重点关注：
- Bug 和逻辑错误
- 安全问题
- 错误处理缺口
```

- 文件名会成为命令名。例如，`review.md` 对应 `/review`。
- `description` 可选。如果省略，则使用第一行非空内容。
- `argument-hint` 可选。设置后，自动补全下拉列表会在说明文字前显示参数提示。

### 参数提示

在 frontmatter（文档头元数据）中使用 `argument-hint`，可以在自动补全中显示预期参数。必填参数使用 `<尖括号>`，可选参数使用 `[方括号]`：

```markdown
---
description: 根据 URL 检查 PR，并对 Issue 和代码进行结构化分析
argument-hint: "<PR-URL>"
---
```

自动补全下拉列表中的效果如下：

```
→ pr   <PR-URL>       — 根据 URL 检查 PR，并对 Issue 和代码进行结构化分析
  is   <issue>        — 分析 GitHub Issue（Bug 或功能请求）
  wr   [instructions] — 端到端完成当前任务
  cl   — 发布前审查变更日志条目
```

## 使用方法

在编辑器中输入 `/`，然后输入模板名称。自动补全会列出可用模板及其说明。

```
/review                           # 展开 review.md
/component Button                 # 带参数展开
/component Button "click handler" # 多个参数
```

## 参数

模板支持位置参数、默认值和简单切片：

- `$1`、`$2`……表示位置参数
- `$@` 或 `$ARGUMENTS` 表示拼接后的全部参数
- `${1:-default}`：参数 1 存在且非空时使用该参数，否则使用 `default`
- `${@:-default}` 或 `${ARGUMENTS:-default}`：参数存在且非空时使用全部参数，否则使用 `default`
- `${@:N}`：从第 N 个位置开始的所有参数（位置从 1 开始）
- `${@:N:L}`：从位置 N 开始取 L 个参数

示例：

```markdown
---
description: 创建组件
---
创建一个名为 $1 的 React 组件，并实现以下功能：$@
```

默认值适合用于可选参数：

```markdown
使用 ${1:-7} 个要点总结当前状态。
```

用法：`/component Button "onClick handler" "disabled support"`

## 加载规则

- Pi 不会递归发现 `prompts/` 中的模板。
- 如果需要加载子目录中的模板，请通过 `prompts` 设置或包清单显式添加。
