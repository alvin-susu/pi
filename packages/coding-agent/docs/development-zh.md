# 开发

其他开发规范请参阅 [AGENTS.md](https://github.com/earendil-works/pi-mono/blob/main/AGENTS.md)。

## 环境搭建

```bash
git clone https://github.com/earendil-works/pi-mono
cd pi-mono
npm install
npm run build
```

从源码运行：

```bash
/path/to/pi-mono/pi-test.sh
```

该脚本可以从任意目录运行，Pi 会保留调用者的当前工作目录。

## Fork 与品牌重塑

通过 `package.json` 配置：

```json
{
  "piConfig": {
    "name": "pi",
    "configDir": ".pi"
  }
}
```

为自己的 Fork 修改 `name`、`configDir` 和 `bin` 字段。这些字段会影响 CLI 标题、配置路径和环境变量名称。

## 路径解析

项目支持三种执行方式：通过 npm 安装、使用独立二进制文件，以及通过 tsx 从源码运行。

访问包内资源时，**必须始终使用 `src/config.ts`**：

```typescript
import { getPackageDir, getThemeDir } from "./config.js";
```

不要直接使用 `__dirname` 定位包内资源。

## 调试命令

隐藏命令 `/debug` 会将以下内容写入 `~/.pi/agent/pi-debug.log`：

- 包含 ANSI 控制码的 TUI 渲染行
- 最近发送给 LLM 的消息

## 测试

```bash
./test.sh                         # 运行非 LLM 测试（无需 API 密钥）
npm test                          # 运行全部测试
npm test -- test/specific.test.ts # 运行指定测试
```

## 项目结构

```
packages/
  ai/           # LLM 服务提供商抽象
  agent/        # 智能体循环和消息类型
  tui/          # 终端用户界面组件
  coding-agent/ # CLI 和交互模式
```
