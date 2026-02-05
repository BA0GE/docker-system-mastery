# 03. 命令行基础与参数显微镜 (CLI Basics)

## 1. 核心概念 (Concepts)

Docker CLI (Command Line Interface) 是我们与 Docker Daemon 对话的唯一窗口。大多数初学者只会死记硬背 `docker run`，但并不理解命令背后的**动词-名词**逻辑。

Docker 现代命令遵循 `docker <management-command> <sub-command>` 的语法结构（如 `docker container run`），虽然旧版语法（如 `docker run`）依然被支持，但理解新版语法有助于理解 Docker 的资源分类。

## 2. 核心命令显微镜 (Linguistic Breakdown)

我们将最常用的命令进行拆解，深入理解每一个参数的物理含义。

### A. 启动容器 (The "Run" Command)

```bash
docker run -d --name web-server -p 8080:80 -v $(pwd)/html:/usr/share/nginx/html nginx:latest
```

*   `docker`: 调用 Docker 客户端。
*   `run`: 这是一个组合命令，实际上 = `create` (创建容器层) + `start` (启动主进程)。
*   `-d` (`--detach`): **分离模式**。让容器在后台运行。如果不加，容器的标准输出 (stdout) 会直接占用你的终端，一旦你按 Ctrl+C，容器通常也会随之停止。
*   `--name web-server`: **DNS 友好名**。如果不指定，Docker 会生成 `pedantic_blackwell` 这种随机名字。指定名字对于后续管理（停止、重启、日志查看）至关重要。
*   `-p 8080:80` (`--publish`): **端口映射**。
    *   `8080`: Host Port (宿主机端口)。外部访问入口。
    *   `80`: Container Port (容器端口)。容器内部进程监听的端口。
    *   *原理*: Docker 启动了一个 `docker-proxy` 进程或通过 iptables NAT 规则，将宿主机 8080 的流量转发到容器 IP 的 80 端口。
*   `-v` (`--volume`): **挂载卷**。格式 `宿主机路径:容器路径`。
    *   `$(pwd)/html`: 当前目录下的 html 文件夹。
    *   `/usr/share/nginx/html`: Nginx 容器内存放网页的默认位置。
    *   *效果*: 打通了宿主机与容器的文件系统墙。修改宿主机文件，容器内立即生效。
*   `nginx:latest`: **镜像定位**。
    *   `nginx`: 镜像仓库名 (Repository)。
    *   `latest`: 标签 (Tag)。如果不写，默认为 `latest`。*生产环境严禁使用 `latest`，必须锁定版本号！*

### B. 进入容器 (Exec vs Attach)

这是初学者最容易混淆的一对命令。

1.  **`docker exec -it <container> bash`** (推荐)
    *   `exec`: Execute。在**运行中**的容器里启动一个**新进程**。
    *   `-i` (`--interactive`): 保持标准输入 (Stdin) 打开。让你能输入命令。
    *   `-t` (`--tty`): 分配一个伪终端 (Pseudo-TTY)。让你看到命令行提示符，并支持颜色和格式化。
    *   `bash`: 要执行的命令。如果镜像里没有 bash，尝试 `sh`。
    *   *原理*: 就像通过 SSH 登录到了服务器，但实际上只是在同一个 Namespace 下起了一个 Shell 进程。

2.  **`docker attach <container>`** (慎用)
    *   `attach`: 附着。将当前终端连接到容器的**主进程 (PID 1)** 的标准输入/输出/错误。
    *   *风险*: 如果你按 Ctrl+C，你实际上是向 PID 1 发送了 SIGINT 信号，会导致容器主进程退出，容器也就挂了！

### C. 停止与删除 (Stop vs Kill)

1.  **`docker stop <container>`** (优雅)
    *   发送 `SIGTERM` 信号给容器主进程。
    *   给进程 10秒 (默认) 时间来保存状态、处理未完成请求、关闭连接。
    *   如果 10秒后未退出，再发送 `SIGKILL`。

2.  **`docker kill <container>`** (暴力)
    *   直接发送 `SIGKILL` 信号。
    *   进程立即被内核终止，可能导致数据丢失或文件损坏。

## 3. 信息检查 (Inspection)

当你需要查看容器的“体检报告”时：

```bash
docker inspect <container_id>
```
*   返回一个巨大的 JSON 对象。包含 IP 地址、挂载路径、环境变量、启动命令等所有元数据。
*   **高阶用法**: 使用 `-f` (Format) 配合 Go Template 提取特定信息。
    *   获取 IP: `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>`

## 4. 避坑指南 (Troubleshooting)

### 报错 1: `docker: Error response from daemon: Conflict. The container name "/web-server" is already in use`
*   **原因**: 试图创建一个名字叫 `web-server` 的容器，但这个名字已经被另一个容器（可能是已停止的）占用了。
*   **解决**:
    *   删除旧的: `docker rm -f web-server`
    *   或者给新的起个别名。

### 报错 2: `exec: "bash": executable file not found in $PATH`
*   **原因**: 试图进入容器 (`docker exec -it ... bash`)，但该镜像（通常是 Alpine 版）为了瘦身，没有安装 `bash`。
*   **解决**: 尝试使用 `sh`: `docker exec -it <container> sh`。

### 报错 3: 端口冲突 `Bind for 0.0.0.0:8080 failed: port is already allocated`
*   **原因**: 宿主机的 8080 端口已经被其他进程（或其他容器）占用了。
*   **排查**: `lsof -i :8080` 查看谁在用。
*   **解决**: 修改 `-p` 参数，映射到其他端口，如 `-p 8081:80`。

---
[Next: 04_Container_Life.md](../03_Workflow_and_Lifecycle/04_Container_Life.md)
