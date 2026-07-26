# Windows 配置

Pi 在 Windows 上需要使用 bash。程序会按以下顺序查找：

1. `~/.pi/agent/settings.json` 中配置的自定义路径
2. Git Bash (`C:\Program Files\Git\bin\bash.exe`)
3. PATH 中的 `bash.exe`（Cygwin、MSYS2 或 WSL）

对大多数用户而言，安装 [Git for Windows](https://git-scm.com/download/win) 即可满足要求。

## 自定义 Shell 路径

```json
{
  "shellPath": "C:\\cygwin64\\bin\\bash.exe"
}
```
