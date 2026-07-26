> Pi 可以创建技能。你可以让它根据自己的使用场景构建一个技能。

# 技能

技能是由智能体按需加载、相互独立的能力包。一个技能可以针对特定任务提供专用工作流程、环境配置说明、辅助脚本和参考文档。

Pi 实现了 [Agent Skills 标准](https://agentskills.io/specification)。对于大多数不符合规范的情况，Pi 只会发出警告，仍会以宽松方式加载。尽管该标准不允许技能名称与父目录不同，Pi 仍允许这样做，因为对于由多个智能体运行框架共享的技能目录而言，标准中的这一限制并不理想。

## 目录

- [存放位置](#locations)
- [技能的工作方式](#how-skills-work)
- [技能命令](#skill-commands)
- [技能结构](#skill-structure)
- [Frontmatter](#frontmatter)
- [验证](#validation)
- [示例](#example)
- [技能仓库](#skill-repositories)

<a id="locations"></a>

## 存放位置

> **安全提示：** 技能可以指示模型执行任意操作，也可能包含由模型调用的可执行代码。使用前请检查技能内容。

Pi 会从以下位置加载技能：

- 全局：
  - `~/.pi/agent/skills/`
  - `~/.agents/skills/`
- 项目（仅在项目受信任后）：
  - `.pi/skills/`
  - 当前工作目录及其祖先目录中的 `.agents/skills/`（最多查找到 Git 仓库根目录；如果不在仓库中，则查找到文件系统根目录）
- 包：`skills/` 目录，或 `package.json` 中的 `pi.skills` 条目
- 设置：`skills` 数组中指定的文件或目录
- CLI：`--skill <path>`（可重复指定；即使使用了 `--no-skills`，显式路径仍会追加加载）

发现规则：

- 在 `~/.pi/agent/skills/` 和 `.pi/skills/` 中，根目录下的 `.md` 文件会分别作为独立技能被发现
- 在所有技能位置中，Pi 会递归发现包含 `SKILL.md` 的目录
- 在 `~/.agents/skills/` 和项目 `.agents/skills/` 中，根目录下的 `.md` 文件会被忽略

使用 `--no-skills` 可以禁用自动发现，但通过 `--skill` 显式指定的路径仍会加载。

### 使用其他运行框架的技能

要使用 Claude Code 或 OpenAI Codex 的技能，请将相应目录加入设置：

```json
{
  "skills": [
    "~/.claude/skills",
    "~/.codex/skills"
  ]
}
```

对于项目级 Claude Code 技能，请在 `.pi/settings.json` 中加入：

```json
{
  "skills": ["../.claude/skills"]
}
```

<a id="how-skills-work"></a>

## 技能的工作方式

1. 启动时，Pi 扫描技能位置，并提取名称和说明
2. 系统提示词按照[规范](https://agentskills.io/integrate-skills)，以 XML 格式列出可用技能
3. 当任务与某项技能匹配时，智能体使用 `read` 加载完整的 SKILL.md（模型不一定总会主动执行此步骤；可以通过提示词或 `/skill:name` 强制加载）
4. 智能体遵循技能说明，并使用相对路径引用脚本和资源

这是一种渐进式披露机制：上下文中始终只包含技能说明，完整指令则按需加载。

<a id="skill-commands"></a>

## 技能命令

技能会注册为 `/skill:name` 命令：

```bash
/skill:brave-search           # 加载并执行技能
/skill:pdf-tools extract      # 带参数加载技能
```

命令后的参数会以 `User: <args>` 的形式追加到技能内容中。

可以在交互模式中通过 `/settings`，或在 `settings.json` 中启用/禁用技能命令：

```json
{
  "enableSkillCommands": true
}
```

<a id="skill-structure"></a>

## 技能结构

技能是一个包含 `SKILL.md` 文件的目录，除此之外的结构可以自由组织。

```
my-skill/
├── SKILL.md              # 必需：frontmatter + 指令
├── scripts/              # 辅助脚本
│   └── process.sh
├── references/           # 按需加载的详细文档
│   └── api-reference.md
└── assets/
    └── template.json
```

### SKILL.md 格式

````markdown
---
name: my-skill
description: 该技能的作用以及应在何时使用。请提供具体说明。
---

# 我的技能

## 配置

首次使用前运行一次：
```bash
cd /path/to/skill && npm install
```

## 使用方法

```bash
./scripts/process.sh <input>
```
````

请使用相对于技能目录的路径：

```markdown
详细信息请参阅[参考指南](references/REFERENCE.md)。
```

<a id="frontmatter"></a>

## 前置元数据（Frontmatter）

根据 [Agent Skills 规范](https://agentskills.io/specification#frontmatter-required)：

| 字段 | 必需 | 说明 |
|-------|----------|-------------|
| `name` | 是 | 最多 64 个字符。只能包含小写字母 a-z、数字 0-9 和连字符。与规范不同，Pi 不要求它与父目录同名，因为该规范要求不适合共享技能目录。 |
| `description` | 是 | 最多 1024 个字符。说明技能的作用和适用场景。 |
| `license` | 否 | 许可证名称，或指向包内许可证文件的引用。 |
| `compatibility` | 否 | 最多 500 个字符。说明环境要求。 |
| `metadata` | 否 | 任意键值映射。 |
| `allowed-tools` | 否 | 以空格分隔的预批准工具列表（实验性）。 |
| `disable-model-invocation` | 否 | 设为 `true` 时，系统提示词中不会显示该技能，用户必须使用 `/skill:name`。 |

### 名称规则

- 长度为 1–64 个字符
- 只能包含小写字母、数字和连字符
- 不能以连字符开头或结尾
- 不能包含连续连字符

Pi 不要求名称与父目录一致。Agent Skills 标准有此要求，但对于由多个工具共用的技能目录而言并不理想。

有效名称：`pdf-processing`、`data-analysis`、`code-review`

无效名称：`PDF-Processing`、`-pdf`、`pdf--processing`

### 技能说明的最佳实践

技能说明决定智能体何时加载该技能，因此应当尽量具体。

较好的示例：

```yaml
description: 从 PDF 文件提取文本和表格、填写 PDF 表单，以及合并多个 PDF。处理 PDF 文档时使用。
```

较差的示例：

```yaml
description: 帮助处理 PDF。
```

<a id="validation"></a>

## 验证

Pi 按照 Agent Skills 标准验证技能。大多数问题只会产生警告，技能仍会加载：

- 名称超过 64 个字符或包含无效字符
- 名称以连字符开头/结尾，或包含连续连字符
- 说明超过 1024 个字符

未知的 frontmatter 字段会被忽略。

**例外：** 缺少说明的技能不会加载。

当不同位置存在同名技能时，Pi 会发出警告，并保留最先发现的技能。

<a id="example"></a>

## 示例

```
brave-search/
├── SKILL.md
├── search.js
└── content.js
```

**SKILL.md:**
````markdown
---
name: brave-search
description: 通过 Brave Search API 搜索网页并提取内容。适用于搜索文档、事实或任何网页内容。
---

# Brave Search

## 配置

```bash
cd /path/to/brave-search && npm install
```

## 搜索

```bash
./search.js "query"              # 基础搜索
./search.js "query" --content    # 包含页面内容
```

## 提取页面内容

```bash
./content.js https://example.com
```
````

<a id="skill-repositories"></a>

## 技能仓库

- [Anthropic Skills](https://github.com/anthropics/skills)——文档处理（docx、pdf、pptx、xlsx）和 Web 开发
- [Pi Skills](https://github.com/badlogic/pi-skills)——网页搜索、浏览器自动化、Google API 和语音转写
