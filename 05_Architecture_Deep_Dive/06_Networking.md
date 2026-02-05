# 06. 网络模型详解 (Network Models)

## 1. 核心概念 (Concepts)

网络是 Docker 中最让初学者头疼的部分。容器之间如何通信？容器如何被外界访问？

### CNM (Container Network Model)
Docker 遵循 CNM 标准，包含三个要素：
1.  **Sandbox (沙盒)**: 包含容器的网络栈配置（Ethernet 接口、路由表、DNS）。
2.  **Endpoint (端点)**: 虚拟网络接口，类似于网卡。
3.  **Network (网络)**: 软件定义的网桥，类似于交换机，连接多个 Endpoint。

## 2. 三大核心模式 (Network Drivers)

`[Image of Network Drivers: Bridge vs Host vs None]`

### A. Bridge (网桥模式 - 默认)
当你运行 `docker run` 不加 `--net` 参数时，默认使用此模式。

*   **原理**: Docker 在宿主机上创建一个虚拟网桥 `docker0` (类似于物理交换机)。每个容器分配一个独立的 IP (如 `172.17.0.x`)。容器通过 `veth pair` (虚拟网线) 连接到 `docker0`。
*   **适用场景**: 单机多容器应用。
*   **通信方式**:
    *   **容器间**: 直接通过 IP 通信，或通过 `--name` 解析（仅限自定义 Bridge）。
    *   **外部访问**: 必须使用端口映射 (`-p`)。

### B. Host (主机模式)
```bash
docker run --net=host ...
```
*   **原理**: 容器**不再拥有独立的 Network Namespace**。它直接共享宿主机的网络栈。
*   **特点**:
    *   容器 IP = 宿主机 IP。
    *   **无需 `-p` 映射端口**。容器里监听 80，宿主机 80 就被占用了。
    *   性能最高（没有 NAT 损耗）。
*   **适用场景**: 对网络性能要求极高，或端口数量巨大的服务。

### C. None (无网络模式)
```bash
docker run --net=none ...
```
*   **原理**: 容器有独立的 Network Namespace，但没有任何网卡（只有 loopback）。
*   **适用场景**: 极度安全、不需要联网的批处理任务（如生成密钥、离线计算）。

## 3. 容器互联实操 (Inter-container Communication)

### 为什么 `localhost` 连不上？
在容器 A (Node.js) 中尝试连接 `localhost:3306` (MySQL)，通常会失败。
*   **原因**: 容器内的 `localhost` 指向的是**容器 A 自身**，而不是宿主机，也不是容器 B。

### 解决方案：自定义网络 (User-defined Bridge)

Docker 自带的默认 `bridge` 网络不支持 DNS 自动解析。我们需要创建一个自定义网络。

```bash
# 1. 创建网络
docker network create my-net

# 2. 启动 MySQL，加入网络，并起名为 'db'
docker run -d --name db --net my-net -e MYSQL_ROOT_PASSWORD=secret mysql:5.7

# 3. 启动 Web App，加入同一网络
# 此时，在 Web App 代码中，可以直接使用 'db' 作为主机名连接 MySQL
docker run -d --name web --net my-net my-web-app
```

*   **Linguistic Breakdown**:
    *   `--net my-net`: 指定网络驱动。
    *   **DNS Resolution**: Docker 内部维护了一个 DNS Server。当 Web App 请求 `db` 时，Docker 会自动解析为 MySQL 容器的内部 IP (如 `172.18.0.2`)。

## 4. 避坑指南 (Troubleshooting)

### 报错 1: `Connection refused`
*   **场景**: 容器 A 连容器 B。
*   **排查步骤**:
    1.  确认容器 B 是否已启动 (`docker ps`)。
    2.  确认容器 B 的进程是否监听了 `0.0.0.0` 而不是 `127.0.0.1`。
        *   如果 MySQL 配置为 `bind-address = 127.0.0.1`，它只接受容器 B 内部的连接，外部（包括容器 A）无法连接。
    3.  确认两个容器是否在同一个 Network 下 (`docker inspect`)。

### 误区 2: 在容器里填宿主机 IP
*   **现象**: 代码里写死宿主机局域网 IP (如 `192.168.1.100`)。
*   **问题**: 
    *   IP 可能会变（DHCP）。
    *   如果在云服务器上，可能涉及防火墙规则。
*   **正确做法**: 使用 `host.docker.internal` (Mac/Windows Docker Desktop 特有) 或在 Linux 使用 `--add-host host.docker.internal:host-gateway`。

---
[Next: 07_Storage.md](../05_Architecture_Deep_Dive/07_Storage.md)
