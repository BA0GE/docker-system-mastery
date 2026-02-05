# 07. 存储与数据持久化 (Storage & Persistence)

## 1. 核心概念 (Concepts)

Docker 容器的设计原则是 **"Ephemeral" (瞬时的)**。当容器被删除 (`docker rm`)，其内部文件系统（读写层）中的所有数据都会永久丢失。

为了保存数据库文件、日志或配置文件，我们需要将数据“外包”给宿主机。

## 2. 三大挂载方式 (Mount Types)

`[Image of Mount Types: Volume vs Bind Mount vs tmpfs]`

### A. Volumes (卷 - 推荐)
由 Docker 进程全权管理，存储在宿主机的特定区域（Linux 下通常是 `/var/lib/docker/volumes/`）。

*   **优点**:
    *   与宿主机文件系统解耦（你不需要关心具体路径）。
    *   易于备份和迁移。
    *   跨平台兼容性好。
*   **语法**: `-v <卷名>:<容器路径>`
    ```bash
    docker run -v my-db-data:/var/lib/mysql mysql
    ```

### B. Bind Mounts (绑定挂载)
将宿主机的**任意文件或目录**直接映射到容器中。

*   **优点**: 开发环境神器。修改宿主机代码，容器内立即生效。
*   **缺点**: 依赖宿主机特定路径，移植性差。
*   **语法**: `-v <宿主机绝对路径>:<容器路径>`
    ```bash
    docker run -v $(pwd)/src:/app/src node
    ```

### C. tmpfs Mounts (内存挂载)
数据仅存储在宿主机的内存中，不写入磁盘。

*   **场景**: 存储敏感信息（密钥）或追求极致读写速度的临时缓存。

## 3. 权限与所有权 (Permissions & Ownership)

这是存储挂载中最令人头秃的问题。

### 场景
宿主机当前用户是 `weiguang` (uid=1000)，容器内运行的是 `postgres` (uid=999)。
当你把宿主机目录挂载进去时，可能会出现 `Permission denied`。

### 显微镜分析
*   **Linux 文件权限机制**: 内核只认 UID，不认用户名。
*   如果宿主机文件夹权限是 `drwx------ weiguang:weiguang` (700)，即只有 UID 1000 能读写。
*   容器内进程以 UID 999 运行，试图访问该目录 -> 内核拒绝 -> 崩溃。

### 解决方案
1.  **Chown (暴力法)**: 在宿主机上 `sudo chown -R 999:999 data_dir`。但这样宿主机用户就很难查看了。
2.  **User Remapping (高级)**: 启动容器时指定 User。
    ```bash
    docker run -u $(id -u):$(id -g) -v $(pwd):/app node
    ```
    *   这会让容器内进程以宿主机用户的 UID 运行，从而拥有读写权限。

## 4. 常用命令 (Management)

*   `docker volume create <name>`: 手动建卷。
*   `docker volume ls`: 列出所有卷。
*   `docker volume inspect <name>`: 查看卷在宿主机的真实物理路径。
*   `docker volume prune`: **危险操作**。删除所有未被任何容器使用的卷。

## 5. 避坑指南 (Troubleshooting)

### 报错 1: `Is a directory`
*   **命令**: `docker run -v /path/to/folder:/etc/nginx/nginx.conf ...`
*   **原因**: 试图把宿主机的**文件夹**挂载到容器的**文件**上。
    *   如果宿主机路径不存在，Docker 会自动创建一个**文件夹**。
*   **解决**: 确保宿主机文件存在，或者挂载整个目录。

### 误区 2: 多个容器写同一个卷
*   **现象**: 两个 MySQL 容器挂载同一个数据卷。
*   **后果**: 数据损坏。标准文件系统（ext4/xfs）不支持并发写。
*   **例外**: 除非是只读挂载 (`:ro`) 或应用层支持（如分布式文件系统）。

---
[Next: 08_Compose.md](../06_Orchestration/08_Compose.md)
