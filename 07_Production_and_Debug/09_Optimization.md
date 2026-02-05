# 09. 镜像瘦身与安全加固 (Optimization & Security)

## 1. 核心概念 (Concepts)

在生产环境中，"It works" 只是起点。一个优秀的 Docker 镜像必须具备两个特质：**Small (小)** 和 **Secure (安全)**。

*   **小**: 传输更快，部署更快，攻击面更小。
*   **安全**: 最小权限原则，防止容器逃逸。

## 2. 镜像瘦身工程 (Slimming Strategy)

### A. 基础镜像选择 (Base Image Selection)

| 镜像类型 | 示例 (`node`) | 大小 (Approx) | 适用场景 |
| :--- | :--- | :--- | :--- |
| **Full (Debian)** | `node:18` | ~1 GB | 开发环境，需要完整工具链 |
| **Slim** | `node:18-slim` | ~200 MB | 生产环境，删除了 man pages 等 |
| **Alpine** | `node:18-alpine` | ~50 MB | **推荐**。基于 musl libc，极小 |
| **Distroless** | `gcr.io/distroless/nodejs` | ~30 MB | **极致**。不包含 Shell，无法 exec 进去 |

### B. .dockerignore (构建上下文优化)

这不仅是为了瘦身，更是为了安全。不要把 `.git` 目录、`node_modules`、`.env` 文件发送给 Docker Daemon。

**`.dockerignore` 示例**:
```text
.git
.env
node_modules
Dockerfile
README.md
docs/
test/
```

### C. 减少层数 (Layer Consolidation)

在 Docker 旧版本中，建议将多个 `RUN` 合并为一行。但在新版本中，重点是**清理缓存**。

**Bad**:
```dockerfile
RUN apt-get update
RUN apt-get install -y vim
```

**Good**:
```dockerfile
RUN apt-get update && apt-get install -y vim \
    && rm -rf /var/lib/apt/lists/*
```
*   *原理*: 必须在同一层中安装并清理。如果在下一层清理，文件虽然不可见，但仍然存在于底层镜像中（类似 Git 历史）。

## 3. 安全加固 (Security Hardening)

### A. 非 Root 用户运行 (Non-root User)
默认情况下，容器内进程以 root 运行。一旦逃逸，攻击者就拥有宿主机 root 权限。

**Dockerfile**:
```dockerfile
# 创建用户和组 (Alpine 语法)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 切换用户
USER appuser

# 启动应用
CMD ["node", "app.js"]
```

### B. 剥夺能力 (Drop Capabilities)
Linux Root 拥有一系列“能力” (Capabilities)，如 `CAP_NET_ADMIN` (管理网络)。容器其实不需要这么多。

**运行期限制**:
```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```
*   `--cap-drop=ALL`: 先剥夺所有能力。
*   `--cap-add=...`: 只按需赋予特定能力。

### C. 资源限制 (Resource Quotas)
防止某个容器吃光宿主机 CPU/内存 (DoS 攻击)。

```bash
docker run --cpus=0.5 --memory=512m nginx
```
*   `--cpus=0.5`: 限制使用最多 50% 的单核 CPU 时间。
*   `--memory=512m`: 内存硬限制。超过即 OOM Kill。

## 4. 避坑指南 (Troubleshooting)

### 报错 1: `Permission denied` (在非 Root 容器中)
*   **场景**: 切换到 `USER appuser` 后，应用试图写日志到 `/var/log/app.log`。
*   **原因**: `/var/log` 属于 root。
*   **解决**: 在切换 `USER` 之前，先创建目录并移交权限。
    ```dockerfile
    RUN mkdir -p /var/log/app && chown appuser:appgroup /var/log/app
    USER appuser
    ```

### 误区 2: Alpine 的 DNS 问题
*   **现象**: Alpine 使用 `musl libc`，其 DNS 解析逻辑与标准 `glibc` 不同。在某些旧版 Kubernetes 或特定网络环境下可能解析超时。
*   **解决**: 如果遇到诡异网络问题，尝试换回 `slim` (Debian) 版本验证。

---
[Next: 10_Troubleshooting.md](../07_Production_and_Debug/10_Troubleshooting.md)
