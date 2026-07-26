# Shell 别名

Pi 以非交互模式运行 bash（`bash -c`），而该模式默认不会展开别名。

要启用 Shell 别名，请在 `~/.pi/agent/settings.json` 中加入：

```json
{
  "shellCommandPrefix": "shopt -s expand_aliases\neval \"$(grep '^alias ' ~/.zshrc)\""
}
```

请根据自己的 Shell 配置调整路径（如 `~/.zshrc`、`~/.bashrc` 等）。
