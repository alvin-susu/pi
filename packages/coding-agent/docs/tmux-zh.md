# tmux 配置

Pi 可以在 tmux 中运行，但 tmux 默认会移除部分按键的修饰键信息。如果不做配置，`Shift+Enter`、`Ctrl+Enter` 通常与普通 `Enter` 无法区分。

## 推荐配置

在 `~/.tmux.conf` 中加入：

```tmux
set -g extended-keys on
set -g extended-keys-format csi-u
```

然后彻底重启 tmux：

```bash
tmux kill-server
tmux
```

当 Kitty 键盘协议不可用时，Pi 会自动请求扩展按键上报。设置 `extended-keys-format csi-u` 后，tmux 会以 CSI-u 格式转发带修饰键的按键；这是最可靠的配置。`extended-keys-format` 选项需要 tmux 3.5 或更高版本。

## 为什么推荐 `csi-u`

如果只配置：

```tmux
set -g extended-keys on
```

tmux 默认使用 `extended-keys-format xterm`。应用程序请求扩展按键上报时，带修饰键的按键会以 xterm `modifyOtherKeys` 格式转发，例如：

- `Ctrl+C` → `\x1b[27;5;99~`
- `Ctrl+D` → `\x1b[27;5;100~`
- `Ctrl+Enter` → `\x1b[27;5;13~`

使用 `extended-keys-format csi-u` 时，相同按键会按以下形式转发：

- `Ctrl+C` → `\x1b[99;5u`
- `Ctrl+D` → `\x1b[100;5u`
- `Ctrl+Enter` → `\x1b[13;5u`

Pi 同时支持这两种格式，但建议在 tmux 中使用 `csi-u`。

## 该配置解决的问题

如果不启用 tmux 扩展按键，带修饰键的 Enter 会退化为传统转义序列：

| 按键 | 未启用扩展按键 | 启用 `csi-u` |
|------|----------------|--------------|
| Enter | `\r` | `\r` |
| Shift+Enter | `\r` | `\x1b[13;2u` |
| Ctrl+Enter | `\r` | `\x1b[13;5u` |
| Alt/Option+Enter | `\x1b\r` | `\x1b[13;3u` |

这会影响默认键位绑定（`Enter` 提交、`Shift+Enter` 换行），也会影响所有使用带修饰键 Enter 的自定义键位绑定。

## 环境要求

- 使用 `extended-keys-format csi-u` 需要 tmux 3.5 或更高版本（运行 `tmux -V` 检查）
- 需要支持扩展按键的终端模拟器（Ghostty、Kitty、iTerm2、WezTerm 或 Windows Terminal）

使用 tmux 3.2 至 3.4 时，请省略 `extended-keys-format csi-u`；Pi 仍然支持 tmux 默认的 xterm `modifyOtherKeys` 格式。
