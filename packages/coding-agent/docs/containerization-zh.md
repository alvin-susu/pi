# 容器化

默认情况下，Pi 拥有启动它的用户所具备的全部权限。但在某些场景中，你可能希望更严格地控制 Pi 可以写入哪些目录，以及可以访问哪些资源。

通常有两种方案：

1. 在隔离环境中运行整个 `pi` 进程；或者
2. 在宿主机上运行 `pi`，但将工具执行转发到隔离环境。

## 选择隔离方案

| 方案 | 隔离对象 | 适用场景 | 备注 |
| --- | --- | --- | --- |
| Gondolin 扩展 | 内置工具和 `!` 命令 | 将认证保留在宿主机，同时通过本地微型虚拟机实现隔离 | 参阅 [`examples/extensions/gondolin/`](../examples/extensions/gondolin/)。 |
| 普通 Docker | 本地容器中的整个 `pi` 进程 | 简单的本地隔离 | 服务提供商 API 密钥会进入容器。 |
| OpenShell | 受策略控制的沙箱中的整个 `pi` 进程 | 本地或远程托管沙箱 | 需要 OpenShell 网关 |

扩展始终在 `pi` 进程所在的环境中运行。如果 Pi 运行在宿主机上，并使用工具路由扩展，那么其他自定义扩展工具仍会在宿主机上执行，除非它们也将操作委托给隔离环境。

## Gondolin

[Gondolin](https://github.com/earendil-works/gondolin) 是一个本地 Linux 微型虚拟机。
如果希望 `pi` 保持运行在宿主机上，但将所有内置工具都转发到虚拟机中，请使用[示例扩展](../examples/extensions/gondolin)。

安装：

```bash
cp -R packages/coding-agent/examples/extensions/gondolin ~/.pi/agent/extensions/gondolin
cd ~/.pi/agent/extensions/gondolin
npm install --ignore-scripts
```

在需要挂载的项目中运行：

```bash
cd /path/to/project
pi -e ~/.pi/agent/extensions/gondolin
```

该扩展会把宿主机的当前工作目录挂载到虚拟机的 `/workspace`，并覆盖 `read`、`write`、`edit`、`bash`、`grep`、`find` 和 `ls`。
用户输入的 `!` 命令也会转发到虚拟机。
对 `/workspace` 中的文件所做的修改会直接写回宿主机。

环境要求：`@earendil-works/gondolin` 需要 Node.js >= 23.6.0，此外还需要通过系统包管理器安装 QEMU。

## 普通 Docker

如果需要最简单的本地容器边界，可以在 Docker 中运行整个 `pi` 进程。

`Dockerfile.pi`:

```dockerfile
FROM node:24-bookworm-slim

RUN apt-get update \
  && apt-get install -y --no-install-recommends bash ca-certificates git ripgrep \
  && rm -rf /var/lib/apt/lists/*
RUN npm install -g --ignore-scripts @earendil-works/pi-coding-agent

WORKDIR /workspace
ENTRYPOINT ["pi"]
```

构建并运行：

```bash
docker build -t pi-sandbox -f Dockerfile.pi .

docker run --rm -it \
  -e ANTHROPIC_API_KEY \
  -v "$PWD:/workspace" \
  -v pi-agent-home:/root/.pi/agent \
  pi-sandbox
```

`-v "$PWD:/workspace"` 会将当前目录挂载到容器内的 `/workspace`。与 Gondolin 示例相同，在 Docker 中读写 `/workspace` 会直接影响宿主机文件。

如果希望设置和会话只保存在容器环境中，请为 `/root/.pi/agent` 使用命名卷。挂载宿主机的 `~/.pi/agent` 会使容器能够访问宿主机的认证信息和会话文件。

## OpenShell

如果需要通过策略控制沙箱中的文件系统、进程、网络、凭据和模型推理，请使用 [NVIDIA OpenShell](https://docs.nvidia.com/openshell/about/overview)。
OpenShell 可以通过由 Docker、Podman 或虚拟机运行时支持的本地网关运行沙箱，也可以使用远程 Kubernetes 网关。

每个沙箱都需要一个处于活动状态的网关。
创建沙箱之前，请先注册并选择网关：

```bash
openshell gateway add <gateway-url> --name <name>
openshell gateway select <name>
```

在 OpenShell 沙箱中启动 `pi`：

```bash
openshell sandbox create --name pi-sandbox --from pi -- pi
```

在该方案中，整个 `pi` 进程都运行在沙箱内。
内置工具、`!` 命令和扩展工具均在 OpenShell 边界内执行。

如果使用远程网关，宿主机的项目文件不会绑定挂载到沙箱，因此沙箱中的写入不会反映到本机。
可以在沙箱内克隆仓库，也可以使用 OpenShell 文件传输命令：

```bash
openshell sandbox upload pi-sandbox ./repo /workspace
openshell sandbox download pi-sandbox /workspace/repo ./repo-out
```

OpenShell 的服务提供商配置可以将原始模型 API 密钥保留在沙箱之外。
配置推理路由后，沙箱内的代码可以调用 `https://inference.local`，网关会在向上游转发请求时注入已配置的服务提供商凭据。
如果希望模型流量经过该路由，请将 Pi 配置为使用相应的 OpenAI 兼容或 Anthropic 兼容端点。
