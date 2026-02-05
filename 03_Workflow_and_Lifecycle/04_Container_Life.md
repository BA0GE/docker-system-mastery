# 04. 容器生命周期与状态流转 (Container Lifecycle)

## 1. 核心概念 (Concepts)

容器不仅仅是“运行”或“停止”那么简单。它拥有一个完整的生命周期状态机。理解这个状态机，是编写可靠的编排脚本（如 Docker Compose, Kubernetes）的基础。

### 核心状态机 (State Machine)

`[Image of Container Lifecycle: Created -> Running -> Paused/Exited -> Dead]`

1.  **Created (已创建)**: 容器层已分配，元数据已生成，但主进程尚未启动。
2.  **Running (运行中)**: 主进程 (PID 1) 正在运行。
3.  **Paused (已暂停)**: 所有进程被 `cgroup` 冻结 (frozen)，CPU 占用降为 0，但内存仍保留。
4.  **Exited (已退出)**: 主进程结束。容器文件系统保留，可以查看日志，但不再消耗 CPU/内存。
5.  **Dead/Removal (死亡/移除)**: 彻底从 Docker 引擎中移除。

## 2. 状态流转实操 (Workflow)

我们将演示一个容器从出生到死亡的完整旅程。

### Step 1: Create (只生不养)
```bash
docker create --name life-demo nginx:alpine
```
*   *状态*: **Created**
*   *解释*: 此时容器已存在，但未启动。适用于需要先行配置网络或挂载卷，稍后再启动的场景。

### Step 2: Start (启动)
```bash
docker start life-demo
```
*   *状态*: **Running**
*   *解释*: 触发 PID 1 进程启动。

### Step 3: Pause/Unpause (冷冻解冻)
```bash
docker pause life-demo
# 此时访问 Nginx 会无响应，但连接不会立即断开（直到超时）
docker unpause life-demo
```
*   *状态*: **Running** <-> **Paused**
*   *原理*: 利用 Linux cgroup 的 `freezer` 子系统。

### Step 4: Stop (寿终正寝)
```bash
docker stop life-demo
```
*   *状态*: **Exited (0)**
*   *解释*: 正常退出。使用 `docker ps -a` 可以看到它，状态栏显示 `Exited (0) ...`。

### Step 5: Restart (起死回生)
```bash
docker restart life-demo
```
*   *状态*: **Running**
*   *解释*: 先 stop 再 start。原有的文件系统修改（如日志文件）依然存在。

### Step 6: OOM Kill (非正常死亡)
如果容器内存超限，会被内核 OOM Killer 杀掉。
*   *状态*: **Exited (137)**
*   *解释*: 退出码 137 = 128 + 9 (SIGKILL)。这是典型的内存溢出信号。

## 3. 自动重启策略 (Restart Policies)

在生产环境中，我们不能手动盯着容器去重启。Docker 提供了 `--restart` 参数来自动管理生命周期。

```bash
docker run -d --restart always nginx
```

| 策略 | 解释 | 适用场景 |
| :--- | :--- | :--- |
| `no` | 默认值。挂了就挂了，不重启。 | 一次性任务 (Job) |
| `on-failure[:max-retries]` | 只有当退出码非 0 (报错退出) 时才重启。可限制重试次数。 | 也是 Job，但容忍偶尔失败 |
| `always` | 无论如何都重启。即使手动 stop，Docker 服务重启后它也会自动拉起。 | **生产 Web 服务首选** |
| `unless-stopped` | 类似 always，但如果你手动 stop 了它，Docker 重启后**不会**自动拉起它。 | 开发环境服务 |

## 4. 避坑指南 (Troubleshooting)

### 现象 1: 容器一启动就退出 (Exited immediately)
*   **命令**: `docker run -d ubuntu`
*   **原因**: Docker 容器的生命周期绑定在 **PID 1 (主进程)** 上。
    *   `ubuntu` 镜像默认命令是 `bash`。
    *   在 `-d` 后台模式下，`bash` 没有连接终端，发现无事可做，立即退出。
    *   PID 1 退出 -> 容器退出。
*   **解决**:
    *   如果是服务类容器，确保前台运行（如 `nginx -g "daemon off;"`）。
    *   如果是测试容器，加 `-it` 或命令 `tail -f /dev/null` 让它卡住。

### 现象 2: `docker stop` 极慢 (卡住 10 秒)
*   **原因**: 容器内的主进程忽略了 `SIGTERM` 信号。
    *   通常因为 PID 1 是 shell 脚本 (`/bin/sh -c ...`)，而 shell 不会转发信号给子进程。
*   **解决**:
    *   在 Dockerfile 中使用 `exec` 启动命令。
    *   或者使用 `tini` 作为 init 进程 (`docker run --init ...`)。

---
[Next: 05_Dockerfile.md](../04_Image_Engineering/05_Dockerfile.md)
