# 运维/脚本规范

## 脚本规则
- 所有脚本必须加 shebang（`#!/bin/bash`）
- 脚本开头写清用途注释
- 关键步骤加 `echo` 日志输出
- 失败必须 `set -e`，遇到错误立即停止
- 不要硬编码路径，用变量定义根目录

## 脚本示例
```bash
#!/bin/bash
# 用途：构建前端并部署到 Nginx

set -e

PROJECT_ROOT="$(cd "$(dirname "$0")/.." && pwd)"
BUILD_DIR="$PROJECT_ROOT/frontend/dist"

echo "开始构建前端..."
cd "$PROJECT_ROOT/frontend"
pnpm build

echo "部署到 Nginx..."
cp -r "$BUILD_DIR"/* /var/www/html/

echo "部署完成"
```

## 部署
- 前端构建产物放 `nginx/html/`
- 后端打包为 jar，用 `java -jar` 启动
- 数据库迁移用 Flyway 或手动执行 SQL 脚本

---

## Docker 规范（Compose，只放中间件）

> 应用本身（前端/后端）在宿主机运行，Docker 只管 MySQL、Redis、Nginx 等中间件。

### 版本与镜像管理

- 指定明确镜像版本标签，禁止使用 `latest`
- 优先官方镜像，其次 Verified Publisher 镜像
- 版本格式：`镜像名:主版本.次版本.补丁`，如 `mysql:8.0.36`

### 目录结构

```
project/
└── docker/
    └── docker-compose.yml   # 中间件编排
```

### docker-compose.yml

```yaml
version: "3.8"

services:
  mysql:
    image: mysql:8.0.36
    container_name: xxx-mysql
    restart: unless-stopped
    ports:
      - "${MYSQL_PORT:-3306}:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: xxx
      TZ: Asia/Shanghai
    volumes:
      - mysql-data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d  # 初始化 SQL 脚本目录
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7.2.4
    container_name: xxx-redis
    restart: unless-stopped
    ports:
      - "${REDIS_PORT:-6379}:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  nginx:
    image: nginx:1.25.4
    container_name: xxx-nginx
    restart: unless-stopped
    ports:
      - "${NGINX_PORT:-80}:80"
    volumes:
      - ../frontend/dist:/usr/share/nginx/html:ro
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - mysql
      - redis

volumes:
  mysql-data:
  redis-data:
```

### 启动与拉取脚本

```bash
#!/bin/bash
# 用途：启动中间件容器并校验

set -e

PROJECT_ROOT="$(cd "$(dirname "$0")/.." && pwd)"
COMPOSE_FILE="$PROJECT_ROOT/docker/docker-compose.yml"
CONTAINER_PASSWORD="${MYSQL_ROOT_PASSWORD}"

# 1. 检查 Docker 是否可用，不可用则尝试修复
ensure_docker_running() {
  if docker info &>/dev/null; then
    echo "Docker 已运行"
    return 0
  fi

  echo "Docker 未运行，尝试启动 Docker Desktop..."
  powershell.exe -Command "Start-Process 'C:\Program Files\Docker\Docker\Docker Desktop.exe'" 2>/dev/null || true
  echo "等待 Docker 就绪（最多 120 秒）..."
  for i in $(seq 1 60); do
    docker info &>/dev/null && echo "Docker 已启动" && return 0
    sleep 2
  done

  # Docker Desktop 启动失败，尝试重启 WSL
  echo "Docker Desktop 启动超时，尝试重启 WSL..."
  wsl --shutdown 2>/dev/null || true
  sleep 5
  powershell.exe -Command "Start-Process 'C:\Program Files\Docker\Docker\Docker Desktop.exe'" 2>/dev/null || true
  for i in $(seq 1 30); do
    docker info &>/dev/null && echo "Docker 已启动" && return 0
    sleep 2
  done

  echo "Docker 启动失败，请手动检查 Docker Desktop 或 WSL 状态"
  exit 1
}

# 2. 检测端口冲突，自动切换
resolve_conflict_ports() {
  local mysql_port=3306 redis_port=6379 nginx_port=80

  while netstat -ano 2>/dev/null | grep -q ":${mysql_port} LISTENING"; do
    echo "[端口冲突] MySQL 端口 $mysql_port 被占用，切换到 $((mysql_port + 1))"
    mysql_port=$((mysql_port + 1))
  done
  while netstat -ano 2>/dev/null | grep -q ":${redis_port} LISTENING"; do
    echo "[端口冲突] Redis 端口 $redis_port 被占用，切换到 $((redis_port + 1))"
    redis_port=$((redis_port + 1))
  done
  while netstat -ano 2>/dev/null | grep -q ":${nginx_port} LISTENING"; do
    echo "[端口冲突] Nginx 端口 $nginx_port 被占用，切换到 $((nginx_port + 1))"
    nginx_port=$((nginx_port + 1))
  done

  export MYSQL_PORT=$mysql_port
  export REDIS_PORT=$redis_port
  export NGINX_PORT=$nginx_port
}

# 3. 主动拉取镜像
pull_images() {
  echo "拉取镜像..."
  docker compose -f "$COMPOSE_FILE" pull
}

# 4. 启动容器
start_services() {
  echo "启动中间件..."
  docker compose -f "$COMPOSE_FILE" up -d
}

# 5. 启动校验
verify_services() {
  echo "=== 启动校验 ==="
  local failed=0

  for name in xxx-mysql xxx-redis xxx-nginx; do
    local status
    status=$(docker inspect --format='{{.State.Status}}' "$name" 2>/dev/null)
    if [ "$status" = "running" ]; then
      echo "✓ $name 运行中"
    else
      echo "✗ $name 状态异常：$status"
      failed=1
    fi
  done

  # 等待 MySQL 就绪
  echo "等待 MySQL 就绪..."
  for i in $(seq 1 15); do
    docker exec xxx-mysql mysqladmin ping -uroot -p"${CONTAINER_PASSWORD}" --silent 2>/dev/null && \
      echo "✓ MySQL 可连接" && break
    sleep 2
  done

  # Redis
  docker exec xxx-redis redis-cli -a "${CONTAINER_PASSWORD}" ping 2>/dev/null | grep -q PONG && \
    echo "✓ Redis 可连接"

  # Nginx
  local http_code
  http_code=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:${NGINX_PORT}" 2>/dev/null || echo "000")
  [ "$http_code" = "200" ] && echo "✓ Nginx 响应正常" || echo "✗ Nginx 响应异常：HTTP $http_code"

  [ $failed -eq 0 ] && echo "=== 校验通过 ===" || echo "=== 存在异常，执行 docker logs <容器名> 排查 ==="
}

# 执行
ensure_docker_running
resolve_conflict_ports
pull_images
start_services
verify_services
```

### 容器密码说明

所有中间件密码统一通过 `${MYSQL_ROOT_PASSWORD}` / `${REDIS_PASSWORD}` 环境变量注入，**禁止**明文硬编码。

| 容器 | 用途 | 配置位置 |
|------|------|---------|
| MySQL | `MYSQL_ROOT_PASSWORD` | `docker-compose.yml` 环境变量 |
| Redis | `--requirepass` | `docker-compose.yml` command 参数 |
| Spring Boot | `spring.datasource.password` | `application.yml` |

> 生产环境应改用 `.env` 文件或 Docker Secrets，禁止明文提交密码。

### 常用运维命令

```bash
# 查看容器状态
docker compose -f docker/docker-compose.yml ps

# 查看日志（最近 50 行）
docker compose -f docker/docker-compose.yml logs --tail 50 mysql

# 进入容器
docker exec -it xxx-mysql bash

# 停止
docker compose -f docker/docker-compose.yml down

# 停止并删除数据卷（慎用，会清空数据库）
docker compose -f docker/docker-compose.yml down -v

# 重启某服务
docker compose -f docker/docker-compose.yml restart redis

# 拉取最新镜像并重建
docker compose -f docker/docker-compose.yml pull && \
docker compose -f docker/docker-compose.yml up -d
```
