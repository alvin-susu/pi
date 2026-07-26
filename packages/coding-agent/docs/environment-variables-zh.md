# 环境变量

Pi 通过三种方式使用环境变量：

- 使用 `PI_OFFLINE` 等变量配置 Pi 进程。
- Pi 会设置 `PI_CODING_AGENT`，以便子进程判断自己是否在 Pi 内运行。
- 由 LLM 调用的 bash 工具所执行的命令，会收到描述当前会话的 `PI_*` 变量。

服务提供商的 API 密钥变量在[服务提供商](providers-zh.md#environment-variables-or-auth-file)文档中单独说明。

## 进程标记

CLI 和 RPC 入口会设置 `PI_CODING_AGENT=true`。子进程会继承该变量，并可据此判断自己是否在 Pi 内运行。它不属于某个特定会话；通过 SDK 嵌入 Pi 时，也不会自动设置该变量。

<a id="bash-tool-session-environment"></a>

## Bash 工具的会话环境

Bash 工具执行的命令会收到当前 Pi 会话状态：

| 变量 | 说明 |
|------|------|
| `PI_SESSION_ID` | 当前会话 ID |
| `PI_SESSION_FILE` | 当前会话 JSONL 文件的绝对路径；临时会话中不设置 |
| `PI_PROVIDER` | 当前选择的模型服务提供商 |
| `PI_MODEL` | 当前选择的模型 ID |
| `PI_REASONING_LEVEL` | 当前实际生效的推理级别：`off`、`minimal`、`low`、`medium`、`high`、`xhigh` 或 `max` |

每条命令启动时才会解析这些值。因此，切换模型或更改推理级别后，无需重启 Pi，下一条 bash 命令就会获得新值。`PI_PROVIDER` 和 `PI_MODEL` 标识的是 Pi 中选定的模型，而不是路由器内部可能另行选择的上游模型。

需要确认当前运行的是哪个模型或服务提供商时，应检查这些变量，而不要根据系统提示词推断：

```bash
printf '%s/%s\n' "$PI_PROVIDER" "$PI_MODEL"
printf 'reasoning=%s session=%s\n' "$PI_REASONING_LEVEL" "$PI_SESSION_ID"
```

如果是持久化会话，可以直接检查会话文件：

```bash
if [ -n "$PI_SESSION_FILE" ]; then
  tail -n 1 "$PI_SESSION_FILE"
fi
```

这些变量只会注入可由 LLM 调用的 bash 工具，不会注入用户手动输入的 `!` 或 `!!` 命令。

### 自定义 Bash 工具

使用 `createBashTool()` 创建的 bash 工具注册到 Pi 后，默认会公开会话环境。变量注入发生在 `spawnHook` 之前，因此钩子可以通过 `ctx.env` 收到这些变量：

```typescript
const bashTool = createBashTool(cwd, {
  spawnHook: (ctx) => ({
    ...ctx,
    env: { ...ctx.env, CI: "1" },
  }),
});
```

可以独立于生成进程钩子禁用会话元数据：

```typescript
const bashTool = createBashTool(cwd, {
  exposeSessionEnvironment: false,
  spawnHook: (ctx) => ctx,
});
```

禁用后，Pi 会移除这些变量中继承而来的值，避免嵌套的 Pi 进程暴露已经失效的父会话元数据。

## Pi 进程配置

以下变量由 Pi 自身读取：

| 变量 | 说明 |
|------|------|
| `PI_CODING_AGENT_DIR` | 覆盖配置目录；默认为 `~/.pi/agent` |
| `PI_CODING_AGENT_SESSION_DIR` | 覆盖会话存储目录；优先级低于 `--session-dir` |
| `PI_PACKAGE_DIR` | 覆盖包目录，适用于 Nix/Guix 存储路径 |
| `PI_OFFLINE` | 禁用启动阶段的网络操作，包括更新检查、包更新和安装/更新遥测 |
| `PI_SKIP_VERSION_CHECK` | 禁止向 `pi.dev` 请求最新版本 |
| `PI_TELEMETRY` | 覆盖安装/更新遥测和服务提供商归因请求头：`1`/`true`/`yes` 或 `0`/`false`/`no` |
| `PI_CACHE_RETENTION` | 在服务提供商支持时，设为 `long` 可延长提示词缓存的保留时间 |
| `PI_SHARE_VIEWER_URL` | 覆盖 `/share` 使用的基础 URL |
| `PI_HARDWARE_CURSOR` | 设为 `1` 以显示硬件光标；参阅[终端配置](terminal-setup-zh.md) |
| `VISUAL`、`EDITOR` | 未设置 `externalEditor` 时使用的外部编辑器后备配置 |
| `HTTP_PROXY`、`HTTPS_PROXY` | 代理出站 HTTP 请求 |

`ANTHROPIC_API_KEY`、`OPENAI_API_KEY` 等服务提供商凭据以及云服务配置，列在[服务提供商](providers-zh.md#environment-variables-or-auth-file)文档中。
