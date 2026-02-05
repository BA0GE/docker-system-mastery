# 08. Docker Compose 实战与逻辑 (Docker Compose)

## 1. 核心概念 (Concepts)

`docker run` 适合管理单个容器。但在微服务架构中，一个应用可能包含 Web 服务、数据库、Redis 缓存、消息队列等多个容器。
手动敲 5 个 `docker run` 命令不仅累，还容易出错（忘记网络连接、环境变量写错）。

### Docker Compose
*   **定义**: 一个用于定义和运行多容器 Docker 应用程序的工具。
*   **核心思想**: **Infrastructure as Code (IaC)**。使用 YAML 文件声明应用的最终状态，一键启动。

## 2. YAML 语法显微镜 (Syntax Breakdown)

创建一个 `docker-compose.yml` 文件。

```yaml
version: '3.8'  # 声明语法版本

services:       # 定义服务集合
  web:          # 服务名 (也是 DNS 主机名)
    build: .    # 使用当前目录的 Dockerfile 构建
    ports:
      - "8080:80" # 宿主机:容器
    environment:
      - NODE_ENV=production
    depends_on: # 依赖顺序
      - db
      - redis
    networks:
      - backend

  db:
    image: postgres:14-alpine
    volumes:
      - db-data:/var/lib/postgresql/data # 使用命名卷
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend

  redis:
    image: redis:alpine
    networks:
      - backend

volumes:        # 声明顶层资源：卷
  db-data:      # 自动创建名为 project_db-data 的卷

networks:       # 声明顶层资源：网络
  backend:      # 自动创建名为 project_backend 的网络
```

### 关键指令解析
*   **`services`**: 每一个 Key (如 `web`, `db`) 都是一个服务，对应一个或多个容器。
*   **`build` vs `image`**:
    *   `build`: 现场根据 Dockerfile 构建镜像。
    *   `image`: 直接拉取现成镜像。
*   **`depends_on`**:
    *   **作用**: 控制启动顺序。保证 `db` 和 `redis` 在 `web` 之前启动。
    *   **局限性**: 它只保证容器启动了，**不保证**容器内的数据库服务已经准备好接受连接（Ready）。生产环境需要配合 `healthcheck`。

## 3. 常用命令 (Workflow)

注意：新版 Docker 已集成 Compose，命令是 `docker compose` (无连字符)，旧版是 `docker-compose`。

*   `docker compose up -d`: **构建** (如果需要) -> **创建**网络/卷 -> **启动**容器 (后台模式)。
*   `docker compose down`: 停止并**移除**容器、网络。**保留**卷数据。
*   `docker compose down -v`: 停止并移除一切，**包括卷** (数据全删)。
*   `docker compose logs -f`: 查看所有服务的聚合日志。
*   `docker compose ps`: 查看当前 Compose 项目下的容器状态。

## 4. 生产级技巧 (Production Tips)

### A. 环境变量文件 (.env)
不要把密码直接写在 YAML 里。Docker Compose 会自动读取同级目录下的 `.env` 文件。

**`docker-compose.yml`**:
```yaml
environment:
  POSTGRES_PASSWORD: ${DB_PASSWORD}
```

**`.env`**:
```ini
DB_PASSWORD=SuperSecretPass123!
```

### B. 扩展与覆盖 (Override)
开发环境和生产环境配置不同？
*   基础配置: `docker-compose.yml`
*   开发覆盖: `docker-compose.override.yml` (默认自动加载)
*   生产覆盖: `docker-compose.prod.yml`

运行生产环境:
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## 5. 避坑指南 (Troubleshooting)

### 报错 1: `Version in "./docker-compose.yml" is unsupported`
*   **原因**: Docker Engine 版本太旧，不支持 YAML 文件里声明的 `version: '3.8'`。
*   **解决**: 升级 Docker，或者降低 version 号。

### 报错 2: 容器名冲突
*   **原因**: Compose 默认使用 `目录名_服务名_序号` 作为容器名。如果你在两个不同目录下有同样的 `docker-compose.yml` 且目录名相同，会冲突。
*   **解决**: 使用 `-p <project_name>` 指定项目名，或修改目录名。

---
[Next: 09_Optimization.md](../07_Production_and_Debug/09_Optimization.md)
