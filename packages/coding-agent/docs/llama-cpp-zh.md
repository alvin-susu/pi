# llama.cpp

Pi 支持 [llama.cpp](https://github.com/ggml-org/llama.cpp) 路由服务器。路由服务器可以发现多个 GGUF 模型，并按需加载或卸载。

请使用支持路由功能的较新 llama.cpp 版本。可以按照[构建说明](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md)自行构建，也可以为当前平台安装[预构建版本](https://github.com/ggml-org/llama.cpp/releases)。

## 启动路由服务器

启动 `llama-server` 时不要指定 `--model` 或 `-m`。如果传入模型参数，服务器会进入单模型模式，而不是路由模式。

```bash
llama-server \
  --models-dir ~/models \
  --no-models-autoload \
  --jinja \
  --host 127.0.0.1 \
  --port 8080 \
  -ngl 999 \
  -c 32768
```

重要选项：

- `--models-dir ~/models`：发现本地 GGUF 文件。
- `--no-models-autoload`：不自动加载模型，改为通过 `/llama` 显式加载。
- `--jinja`：启用兼容的聊天模板和工具调用。
- `-ngl 999`：尽可能多地将模型层交给 GPU 处理。
- `-c 32768`：将每个已加载模型的上下文窗口设为 32768。省略该选项则使用模型原生上下文长度，但这可能需要多得多的内存。

单文件模型可以直接放在模型目录中。多模态模型和多分片模型应分别放入独立的子目录：

```text
~/models/
├── llama-3.2-1b-Q4_K_M.gguf
├── gemma-3-4b-it-Q4_K_M/
│   ├── gemma-3-4b-it-Q4_K_M.gguf
│   └── mmproj-F16.gguf
└── large-model-Q4_K_M/
    ├── large-model-Q4_K_M-00001-of-00003.gguf
    ├── large-model-Q4_K_M-00002-of-00003.gguf
    └── large-model-Q4_K_M-00003-of-00003.gguf
```

手动添加文件后，需要重启路由服务器。要为不同模型设置各自的上下文长度或其他选项，请使用 [llama.cpp 模型预设](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md#model-presets)。

## 配置 Pi

启动 Pi 并配置服务提供商：

```text
/login llama.cpp
```

输入路由服务器 URL，并按需填写 API 密钥。默认 URL 为 `http://127.0.0.1:8080`。

也可以通过环境变量配置相同的值，而无需执行 `/login`：

```bash
export LLAMA_BASE_URL=http://127.0.0.1:8080
export LLAMA_API_KEY=optional-secret
pi
```

如果服务器使用 API 密钥，请使用相同的 `--api-key` 值启动 `llama-server`。如果只允许本机访问，请保留 `--host 127.0.0.1`。

## 管理模型

运行：

```text
/llama
```

- 选择尚未加载的模型可以加载它。
- 选择已经加载的模型可以卸载它。
- 选择 **Download model…（下载模型……）**，搜索 Hugging Face，然后选择仓库和量化版本。也可以直接输入完整的 `owner/repository[:quant]`。
- 加载或下载期间按 Escape，可以确认取消操作。

搜索 Hugging Face 时，Pi 会优先使用已设置的 `HF_TOKEN`，然后依次检查 `$HF_TOKEN_PATH`、`$HF_HOME/token`、`$XDG_CACHE_HOME/huggingface/token` 和 `~/.cache/huggingface/token`。未认证时也可以搜索，但速率限制会更严格。下载需要授权访问的仓库前，Pi 会显示警告并提供其访问申请页面链接。实际下载由 llama.cpp 服务器执行，因此如果所选仓库需要访问权限，服务器进程也必须拥有 `HF_TOKEN`。

如果已经加载了其他模型，Pi 会询问是先卸载它们，还是继续保留。Pi 不会静默卸载模型，也绝不会删除模型文件。路由服务器可能由多个客户端共享，因此 `/llama` 始终显示路由服务器的当前状态。

只有已加载的模型才会出现在 `/model` 中。加载模型后，请运行 `/model`，将它选为当前 Pi 会话使用的模型。

如果路由服务器断开连接，`/llama` 会显示 **Retry（重试）** 和 **Close（关闭）**。重试会重新连接并刷新模型状态，但不会重复执行被中断的操作。

## 故障排查

检查路由服务器是否可以访问：

```bash
curl http://127.0.0.1:8080/health
curl http://127.0.0.1:8080/models
```

- **`/llama` 中没有模型：** 检查 `--models-dir` 和目录结构，然后重启路由服务器。
- **`/model` 中缺少模型：** 先使用 `/llama` 加载该模型。
- **加载失败或占用内存过多：** 降低 `-c`，或卸载其他模型。
- **服务器未处于路由模式：** 启动时不要使用 `--model`、`-m` 或 `-hf`。
