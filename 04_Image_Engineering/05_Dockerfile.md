# 05. Dockerfile 深度剖析与分层原理 (Dockerfile Deep Dive)

## 1. 核心概念 (Concepts)

Dockerfile 是构建镜像的“图纸”。它是一个包含一系列指令的文本文件，Docker 引擎按顺序读取这些指令，一层层地构建出最终的镜像。

### 分层原理 (Layer Caching)
这是 Docker 构建速度快、存储高效的核心秘密。
*   Dockerfile 中的每一行指令（`RUN`, `COPY`, `ADD`）都会创建一个新的**镜像层 (Layer)**。
*   **缓存机制**: 如果你修改了 Dockerfile 的第 10 行，那么前 9 行的构建结果会直接从缓存中读取，只有第 10 行及之后的指令会重新执行。

## 2. 指令显微镜 (Instruction Breakdown)

我们将通过构建一个生产级的 Node.js 应用镜像来剖析关键指令。

```dockerfile
# 1. FROM: 基础镜像
FROM node:18-alpine

# 2. WORKDIR: 工作目录
WORKDIR /app

# 3. COPY (依赖先行): 利用缓存
COPY package.json package-lock.json ./

# 4. RUN: 执行构建命令
RUN npm ci --only=production

# 5. COPY (源码): 经常变动的部分放后面
COPY . .

# 6. ENV: 环境变量
ENV PORT=8080

# 7. EXPOSE: 端口声明 (仅作为文档)
EXPOSE 8080

# 8. CMD: 启动命令
CMD ["node", "src/index.js"]
```

### 深度解析

*   **`FROM`**: 一切的起点。推荐使用 `alpine` 版本（体积小，安全性高）。
*   **`WORKDIR`**: 相当于 `mkdir + cd`。后续命令都在这个目录下执行。**不要**用 `RUN cd /app`，这会导致层级上下文混乱。
*   **`COPY` vs `ADD`**:
    *   `COPY`: 仅仅是复制本地文件。**推荐使用**。
    *   `ADD`: 高级版复制。支持自动解压 `.tar.gz`，支持从 URL 下载文件。**慎用**，除非你明确需要解压功能。
*   **缓存优化技巧 (The "Dependency First" Pattern)**:
    *   为什么先 `COPY package.json` 然后 `RUN npm install`，最后才 `COPY .`？
    *   *原理*: 源代码 (`src/`) 经常变，但依赖 (`package.json`) 不常变。
    *   如果把 `COPY .` 放在最前，每次改代码都会导致缓存失效，导致 `npm install` 重新运行，浪费大量时间。
*   **`RUN`**: 在**构建时**执行。常用于安装软件、编译代码。
*   **`CMD` vs `ENTRYPOINT`**:
    *   `CMD`: 容器启动时的默认命令。**可以被 `docker run` 后的参数覆盖**。
        *   `CMD ["node", "app.js"]` -> `docker run my-img echo hello` -> 实际运行 `echo hello`。
    *   `ENTRYPOINT`: 容器启动时的固定入口。**不可被覆盖**（除非用 `--entrypoint`）。`docker run` 后的参数会作为参数传给它。
        *   常用于制作命令行工具镜像。

## 3. 多阶段构建 (Multi-stage Builds)

这是**镜像瘦身**的终极武器。

### 场景
Go/Java/C++ 需要编译。编译需要 GCC/JDK (几百MB)。运行只需要二进制文件 (几MB)。我们不希望把 GCC 带入生产镜像。

### 解决方案

```dockerfile
# Stage 1: Build (编译环境)
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp main.go

# Stage 2: Production (生产环境)
FROM alpine:latest
WORKDIR /root/
# 关键：只从 builder 阶段复制编译好的二进制文件
COPY --from=builder /app/myapp .
CMD ["./myapp"]
```

*   *结果*: 最终镜像不包含 Go 编译器，只有 Alpine 系统和二进制文件，体积可能只有 10MB。

## 4. 避坑指南 (Troubleshooting)

### 报错 1: `standard_init_linux.go:228: exec user process caused: no such file or directory`
*   **原因**: 通常发生在 Windows 下编写 Shell 脚本，然后 `COPY` 进 Linux 容器。
    *   Windows 换行符是 `CRLF` (`\r\n`)。
    *   Linux 只能识别 `LF` (`\n`)。
    *   导致 Shebang `#!/bin/bash\r` 找不到解释器。
*   **解决**: 在编辑器中将换行符设置为 LF，或使用 `dos2unix` 转换。

### 报错 2: 构建缓存导致代码未更新
*   **现象**: 修改了代码，重新 build，但运行的还是旧代码。
*   **原因**: Docker 误判文件未修改，使用了缓存层。
*   **解决**: `docker build --no-cache ...` 强制不使用缓存。

### 误区 3: 在 `RUN` 中使用 `export` 设置环境变量
*   **错误写法**: `RUN export MY_VAR=123`
*   **后果**: `export` 只在当前 `RUN` 指令的 shell 进程中有效。下一行指令（新的层）就没了。
*   **正确做法**: 使用 `ENV MY_VAR=123` 指令，它会持久化到镜像元数据中。

---
[Next: 06_Networking.md](../05_Architecture_Deep_Dive/06_Networking.md)
