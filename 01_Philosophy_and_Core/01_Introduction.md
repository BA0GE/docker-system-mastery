# 01. 容器化起源与核心思想 (Introduction)

## 1. 核心概念 (Concepts)

在深入 Docker 命令之前，我们需要理解它诞生的背景。在“前容器时代”，软件交付面临着著名的 **"It works on my machine" (在我的机器上没问题)** 的困境。

### 痛点：环境漂移 (Environment Drift)
开发环境、测试环境和生产环境的操作系统版本、依赖库版本、配置文件往往存在微小差异。这些差异如同蝴蝶效应，导致代码在生产环境中崩溃。

### 解决方案：集装箱隐喻 (The Shipping Container Metaphor)
Docker 的 Logo 是一只驮着集装箱的鲸鱼。这正是其核心概念的直观体现：
*   **货物 (Code & Deps)**: 无论是 Java 代码、Python 脚本还是 MySQL 数据库。
*   **集装箱 (Container)**: 一个标准化的封装单元。它将代码及其所有依赖（库、运行时、配置、OS文件）打包在一起。
*   **码头/船 (OS/Infrastructure)**: 无论底层是 AWS、阿里云，还是你的笔记本，只要有 Docker 引擎（码头工人），集装箱就能原样运行。

> **核心定义**: Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器或 Windows 机器上。

## 2. 底层思想 (Philosophy)

Docker 的设计哲学建立在以下几个关键支柱之上：

### A. 不可变基础设施 (Immutable Infrastructure)
一旦镜像 (Image) 被构建，它就是只读的。你不会“修补”一个正在运行的容器（例如 SSH 进去升级软件），而是修改 Dockerfile，重新构建镜像，并替换旧容器。这确保了部署的一致性和可回滚性。

### B. 进程级隔离 (Process Isolation)
虚拟机 (VM) 模拟整个硬件并运行完整的操作系统内核，沉重且启动慢。
Docker 容器**共享宿主机的内核**，但通过 Linux 内核特性实现隔离。容器本质上只是宿主机上的一个**被隔离的进程**。

`[Image of VM vs Container Architecture]`

### C. 分层存储 (Layered File System)
Docker 镜像不是一个大文件，而是一组只读层的叠加。
*   **复用性**: 多个镜像可以共享基础层（如 Ubuntu Base Layer）。
*   **写时复制 (CoW)**: 当容器需要修改文件时，它不会修改只读层，而是将文件复制到最上层的“读写层”进行修改。

## 3. 关键术语拆解 (Terminology Breakdown)

| 术语 | 英文全称 | 解释 | 比喻 |
| :--- | :--- | :--- | :--- |
| **镜像** | **Image** | 应用程序的静态打包文件，包含代码、运行时、库、环境变量和配置文件。**只读**。 | 类的定义 (Class) / 游戏光盘 |
| **容器** | **Container** | 镜像的运行实例。它在镜像之上增加了一个可写的容器层。**可读写**。 | 类的对象 (Object) / 正在运行的游戏存档 |
| **仓库** | **Repository** | 存放镜像的地方。 | 代码仓库 (GitHub) / 应用商店 |
| **引擎** | **Engine** | 包含守护进程 (Daemon)、REST API 和 客户端 (CLI) 的 C/S 架构程序。 | 游戏机主机 |

## 4. 标准化流程 (Workflow)

一个典型的 Docker 工作流遵循 **Build - Ship - Run** 的模式：

1.  **Build (构建)**: 编写 `Dockerfile`，定义应用环境。运行 `docker build` 生成镜像。
2.  **Ship (运输)**: 运行 `docker push` 将镜像推送到镜像仓库 (Registry)。
3.  **Run (运行)**: 在生产服务器上运行 `docker run`，拉取镜像并启动容器。

## 5. 避坑指南 (Troubleshooting)

### 误区 1：容器是轻量级虚拟机
*   **现象**: 试图在容器里运行 systemd、SSH 服务，或者在一个容器里塞进 MySQL + PHP + Nginx。
*   **纠正**: **一容器一进程 (One process per container)**。容器的设计初衷是运行单一应用进程。虽然可以运行多进程，但这违背了设计哲学，会导致日志管理和扩缩容变得困难。

### 误区 2：数据写在容器里
*   **现象**: 删除了容器，发现数据库里的数据全丢了。
*   **原因**: 容器的读写层是临时的。容器被删除时，读写层也会被销毁。
*   **解决方案**: 使用 **Volumes (卷)** 或 **Bind Mounts (挂载)** 将数据持久化到宿主机。

### 误区 3：镜像臃肿
*   **现象**: 一个简单的 Python 脚本镜像高达 1GB。
*   **原因**: 包含了不必要的构建工具（GCC, Make）、缓存文件或使用了过大的基础镜像（如完整的 Ubuntu）。
*   **解决方案**: 使用多阶段构建 (Multi-stage Builds) 和精简基础镜像 (Alpine, Distroless)。

---
[Next: 02_Installation.md](../02_Foundation_and_Setup/02_Installation.md)
