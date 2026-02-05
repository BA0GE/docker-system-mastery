# 02. 多平台安装与架构解析 (Installation & Architecture)

## 1. 核心概念 (Concepts)

在安装 Docker 之前，必须理解其 **Client-Server (C/S)** 架构。当你执行 `docker run` 时，这背后的链路并非仅在本地发生。

### Docker Engine 架构
Docker Engine 是一个 C/S 架构的应用程序，主要由三部分组成：
1.  **Server (Daemon)**: `dockerd` 守护进程。它是长期运行的后台进程，负责创建和管理 Docker 对象（镜像、容器、网络、卷）。它是实际干活的“苦力”。
2.  **REST API**: 守护进程和客户端之间通信的接口规范。
3.  **Client (CLI)**: `docker` 命令行工具。这是用户与 Docker 交互的主要方式。它通过 REST API 向守护进程发送指令。

`[Image of Docker Architecture: Client -> REST API -> Daemon]`

## 2. 安装 (Installation)

我们将重点介绍在 **Linux (Ubuntu)** 和 **MacOS** 上的生产级/开发级安装。

### A. Linux (Ubuntu) - 生产环境首选
不要使用 `apt-get install docker.io` (这是旧版本)。官方推荐从 Docker 仓库安装。

#### 标准化流程 (Workflow)

1.  **卸载旧版本 (Clean up)**:
    ```bash
    sudo apt-get remove docker docker-engine docker.io containerd runc
    ```
    *   *语言拆解*: `apt-get remove` 移除软件包；列出的包名是 Docker 历史上不同时期的名称。

2.  **设置仓库 (Set up repository)**:
    ```bash
    # 更新索引并安装依赖
    sudo apt-get update
    sudo apt-get install ca-certificates curl gnupg

    # 添加 Docker 官方 GPG 密钥 (用于验证包的完整性)
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    # 写入软件源列表
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
    *   *语言拆解*: `curl -fsSL`: Fail silently (不显示进度条但显示错误), Follow redirects (跟随重定向)。`tee`: 读取标准输入并写入文件。

3.  **安装 Docker Engine**:
    ```bash
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```
    *   *术语拆解*:
        *   `docker-ce`: Community Edition (社区版) 守护进程。
        *   `docker-ce-cli`: 命令行客户端。
        *   `containerd.io`: 行业标准的容器运行时（实际管理容器生命周期的组件，Docker 调用它）。
        *   `docker-compose-plugin`: V2 版本的 Compose 插件 (提供 `docker compose` 命令)。

4.  **验证**:
    ```bash
    sudo docker run hello-world
    ```

### B. MacOS - 开发环境
推荐使用 **Docker Desktop for Mac**。它在 macOS 上运行一个轻量级 Linux 虚拟机（因为 macOS 内核本身不支持 Linux 容器特性），并在其中运行 Docker Daemon。

*   **下载**: 访问 [Docker Hub](https://www.docker.com/products/docker-desktop/) 下载 `.dmg` 文件安装。
*   **注意**: Docker Desktop 是收费软件（对于大型企业）。开源替代方案推荐 **OrbStack** 或 **Colima**。

## 3. 权限管理 (Post-installation steps)

在 Linux 上，默认情况下 `docker` 命令需要 `sudo` 权限，因为守护进程以 `root` 运行。为了方便开发，我们通常将用户加入 `docker` 用户组。

```bash
sudo usermod -aG docker $USER
newgrp docker
```
*   *语言拆解*:
    *   `usermod`: User Modify (修改用户属性)。
    *   `-aG`: Append to Group (追加到组)。
    *   `docker`: 组名。
    *   `$USER`: 环境变量，代表当前登录用户名。
    *   `newgrp`: 刷新当前 shell 的组 ID，使其立即生效。

## 4. 避坑指南 (Troubleshooting)

### 报错 1: `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?`
*   **原因**: Docker 守护进程未启动，或者当前用户没有权限访问 Docker 的 Unix Socket。
*   **排查**:
    1.  检查服务状态: `sudo systemctl status docker`
    2.  如果服务未运行: `sudo systemctl start docker`
    3.  如果服务运行中但仍报错: 检查权限（参考第 3 节“权限管理”），或尝试加 `sudo` 执行。

### 报错 2: `WARNING: No swap limit support`
*   **原因**: Linux 内核未开启 cgroup 的内存交换限制功能。
*   **影响**: Docker 无法限制容器的 Swap 使用，可能导致容器耗尽宿主机 Swap。
*   **解决**: 修改 `/etc/default/grub`，在 `GRUB_CMDLINE_LINUX` 中添加 `cgroup_enable=memory swapaccount=1`，然后运行 `sudo update-grub` 并重启。

### 报错 3: 镜像拉取极慢 (Timeout)
*   **原因**: Docker Hub 服务器在国外，国内访问受限。
*   **解决**: 配置镜像加速器 (Registry Mirror)。
    *   编辑 `/etc/docker/daemon.json`:
        ```json
        {
          "registry-mirrors": [
            "https://docker.m.daocloud.io",
            "https://mirror.ccs.tencentyun.com"
          ]
        }
        ```
    *   重启服务: `sudo systemctl daemon-reload && sudo systemctl restart docker`

---
[Next: 03_CLI_Basics.md](03_CLI_Basics.md)
