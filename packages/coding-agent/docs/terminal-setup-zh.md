# 终端配置

Pi 使用 [Kitty 键盘协议](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)可靠地识别修饰键。大多数现代终端都支持该协议，但部分终端需要额外配置。

## Kitty, iTerm2

无需配置即可使用。

## Apple 终端

条件允许时，Pi 会启用增强按键上报。如果按下 `Shift+Enter` 时 Terminal.app 仍然发送普通 Return，Pi 会使用 macOS 本地修饰键后备机制，将该 Return 识别为 `Shift+Enter`。

该后备机制仅在 Pi 与 Terminal.app 运行于同一台 Mac 时有效，无法通过远程 SSH 检测本地键盘。

## Ghostty

在 Ghostty 配置中加入以下内容（macOS：`~/Library/Application Support/com.mitchellh.ghostty/config`；Linux：`~/.config/ghostty/config`）：

```
keybind = alt+backspace=text:\x1b\x7f
```

较早版本的 Claude Code 可能添加过以下 Ghostty 映射：

```
keybind = shift+enter=text:\n
```

该映射会发送一个原始换行（LF）字节。在 Pi 内部，它与 `Ctrl+J` 无法区分，因此 tmux 和 Pi 都无法再收到真正的 `shift+enter` 按键事件。

如果添加该映射只是为了使用 Claude Code 2.x 或更高版本，可以将其删除；但如果要在 tmux 中使用 Claude Code，则仍然需要保留该 Ghostty 映射。

Pi 默认将 `Ctrl+J` 绑定为换行的别名，因此通过该重映射，`Shift+Enter` 在 tmux 中仍然有效，无需额外配置 Pi。

## WezTerm

WezTerm 通常可以通过 xterm `modifyOtherKeys` 直接支持 `Shift+Enter`。如果要显式使用 Kitty 键盘协议，请创建 `~/.wezterm.lua`：

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.enable_kitty_keyboard = true
return config
```

在 macOS 上，WezTerm 默认将 `Option+Enter` 绑定为全屏。要使用 `Option+Enter` 将 Pi 的后续消息加入队列，请添加以下按键覆盖：

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.keys = {
  {
    key = 'Enter',
    mods = 'ALT',
    action = wezterm.action.SendString('\x1b[13;3u'),
  },
}
return config
```

如果已经存在 `config.keys` 表，请将该条目加入其中。

在 WSL 上，WezTerm 可能需要显示硬件光标，以便输入法正确定位候选词窗口。如果中日韩输入法的候选词窗口没有跟随文本光标，请在运行 Pi 前设置 `PI_HARDWARE_CURSOR=1`，或在设置中将 `showHardwareCursor` 设为 `true`。

## Alacritty

Alacritty 通常无需配置即可支持 `Shift+Enter`。在 macOS 上，`Option+Enter` 可能会被识别为普通 `Enter`。要使用 `Option+Enter` 将 Pi 的后续消息加入队列，请在 `~/.config/alacritty/alacritty.toml` 中加入：

```toml
[[keyboard.bindings]]
key = "Enter"
mods = "Alt"
chars = "\u001b[13;3u"
```

修改配置后重启 Alacritty。

## VS Code（集成终端）

VS Code 1.109.5 及更高版本默认在集成终端中启用 Kitty 键盘协议，因此 `Shift+Enter` 应该无需配置即可使用。

低于 1.109.5 的 VS Code 版本需要为 `Shift+Enter` 显式配置终端键位绑定。

`keybindings.json` 的位置：

- macOS: `~/Library/Application Support/Code/User/keybindings.json`
- Linux: `~/.config/Code/User/keybindings.json`
- Windows: `%APPDATA%\\Code\\User\\keybindings.json`

在 `keybindings.json` 中加入：

```json
{
  "key": "shift+enter",
  "command": "workbench.action.terminal.sendSequence",
  "args": { "text": "\u001b[13;2u" },
  "when": "terminalFocus"
}
```

## Windows Terminal

在 `settings.json` 中加入以下内容（按 Ctrl+Shift+,，或依次选择“设置 → 打开 JSON 文件”），以转发 Pi 使用的带修饰键 Enter：

```json
{
  "actions": [
    {
      "command": { "action": "sendInput", "input": "\u001b[13;2u" },
      "keys": "shift+enter"
    },
    {
      "command": { "action": "sendInput", "input": "\u001b[13;3u" },
      "keys": "alt+enter"
    }
  ]
}
```

- `Shift+Enter` 插入新行。
- Windows Terminal 默认将 `Alt+Enter` 绑定为全屏，导致 Pi 无法收到用于后续消息入队的 `Alt+Enter`。
- 将 `Alt+Enter` 重新映射到 `sendInput` 后，真实的组合键会转发给 Pi。

如果已经存在 `actions` 数组，请将这些对象加入其中。如果仍然触发原来的全屏行为，请彻底关闭并重新打开 Windows Terminal。

## xfce4-terminal, terminator

这些终端对转义序列的支持有限，无法区分 `Ctrl+Enter`、`Shift+Enter` 等组合键与普通 `Enter`，因此 `submit: ["ctrl+enter"]` 之类的自定义键位绑定无法生效。

为获得最佳体验，请使用支持 Kitty 键盘协议的终端：

- [Kitty](https://sw.kovidgoyal.net/kitty/)
- [Ghostty](https://ghostty.org/)
- [WezTerm](https://wezfurlong.org/wezterm/)
- [iTerm2](https://iterm2.com/)
- [Alacritty](https://github.com/alacritty/alacritty)（编译时需要启用 Kitty 协议支持）

## IntelliJ IDEA（集成终端）

内置终端对转义序列的支持有限。在 IntelliJ 终端中，Shift+Enter 与 Enter 无法区分。

如果希望显示硬件光标，请在运行 Pi 前设置 `PI_HARDWARE_CURSOR=1`（出于兼容性考虑，该功能默认禁用）。

建议使用独立的终端模拟器，以获得最佳体验。
