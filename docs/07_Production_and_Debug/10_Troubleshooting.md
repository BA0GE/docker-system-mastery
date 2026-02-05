# 10. 避坑指南与错误字典 (Troubleshooting Dictionary)

## 1. 核心方法论 (Methodology)

遇到 Docker 报错时，不要慌。遵循 **"L.I.S.T"** 流程：

1.  **L (Logs)**: `docker logs <container>`。看应用本身吐出了什么堆栈信息。
2.  **I (Inspect)**: `docker inspect <container>`。看状态 (OOMKilled?)、IP 地址、挂载路径。
3.  **S (Shell)**: `docker exec -it <container> sh`。进去看看文件在不在，网络通不通。
4.  **T (Test)**: 另外起一个 debug 容器 (`docker run -it --net container:<target> nicolaka/netshoot`) 进行网络抓包或测试。

## 2. 常见报错字典 (Common Errors)

### A. 启动类 (Startup Failures)

#### `exec user process caused: no such file or directory`
*   **人话**: 找不到启动命令对应的文件。
*   **嫌疑人**:
    1.  Windows 换行符 (`CRLF`) 污染了 Shell 脚本。
    2.  二进制文件是动态链接的，但镜像里缺少对应的 `.so` 库 (常发生于 Go 程序跑在 Alpine 上)。
*   **处方**:
    1.  `dos2unix entrypoint.sh`。
    2.  编译 Go 时加 `CGO_ENABLED=0`。

#### `Bind for 0.0.0.0:80 failed: port is already allocated`
*   **人话**: 端口被占了。
*   **处方**: `sudo lsof -i :80` 找出谁在用，杀掉它，或者换个端口映射 (`-p 8080:80`)。

### B. 运行类 (Runtime Errors)

#### `OOMKilled` (Exit Code 137)
*   **人话**: 内存溢出，被内核杀了。
*   **嫌疑人**: Java 应用未配置 Heap 限制，或者应用有内存泄漏。
*   **处方**:
    1.  `docker stats` 观察内存。
    2.  调整应用配置 (如 `-Xmx`)。
    3.  增加容器内存限制 (`--memory`)。

#### `Connection refused`
*   **人话**: 找到了主机，但端口没开。
*   **嫌疑人**:
    1.  应用挂了。
    2.  应用只监听了 `127.0.0.1` (Localhost)，没监听 `0.0.0.0` (All Interfaces)。
*   **处方**: 检查应用配置文件，确保 `bind_address` 为 `0.0.0.0`。

#### `No space left on device`
*   **人话**: 磁盘满了。
*   **嫌疑人**:
    1.  Docker 镜像/容器太多。
    2.  容器日志 (`*-json.log`) 爆了。
*   **处方**:
    1.  `docker system prune -a` (清理所有未使用镜像)。
    2.  配置 Docker Daemon 的日志轮转 (log-rotation)。

### C. 网络类 (Network Issues)

#### `Temporary failure in name resolution`
*   **人话**: DNS 解析失败。
*   **处方**:
    1.  检查容器内 `/etc/resolv.conf`。
    2.  尝试强行指定 DNS: `docker run --dns 8.8.8.8 ...`。

## 3. 调试神器 (Debug Tools)

在极简的镜像 (Alpine/Distroless) 中，往往没有 `ping`, `curl`。推荐使用以下**瑞士军刀镜像**进行排错：

```bash
# 附着到目标容器的网络命名空间中
docker run -it --rm --net container:<target_container_name> nicolaka/netshoot
```
*   `nicolaka/netshoot`: 集成了 tcpdump, curl, dig, iperf, mtr 等几乎所有网络调试工具。

---
**[End of Guide]**
恭喜！你已经掌握了从 Docker 原理到生产排错的完整知识体系。
Go forth and containerize! 🐳
