# Docker 详细指南

## Dockerfile 最佳实践

### 多阶段构建

```dockerfile
# ✅ 正确：多阶段构建
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/main.js"]

# ❌ 错误：单阶段构建，镜像包含不必要文件
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/main.js"]
```

### 层优化

```dockerfile
# ✅ 正确：合并命令减少层数
RUN apt-get update && \
    apt-get install -y \
        curl \
        git \
        vim && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ❌ 错误：每个命令创建新层
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get install -y vim
RUN apt-get clean
```

### 缓存优化

```dockerfile
# ✅ 正确：利用构建缓存
WORKDIR /app

# 先复制依赖文件（变化少）
COPY package*.json ./
RUN npm ci

# 后复制源代码（变化多）
COPY . .
RUN npm run build

# ❌ 错误：一次性复制所有文件
COPY . .
RUN npm ci && npm run build
```

### 使用 .dockerignore

```
# .dockerignore 示例
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.DS_Store
*.md
dist
coverage
.vscode
```

## 镜像安全

### 非 root 用户

```dockerfile
# ✅ 正确：创建并使用非 root 用户
FROM node:18-alpine

# 创建用户组和用户
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# 设置工作目录权限
WORKDIR /app
COPY --chown=nodejs:nodejs . .

# 切换到非 root 用户
USER nodejs

CMD ["node", "app.js"]
```

### 只读根文件系统

```dockerfile
# Dockerfile
FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app
COPY --chown=nodejs:nodejs . .
USER nodejs

# 使用临时卷作为可写目录
VOLUME ["/tmp", "/app/logs"]

CMD ["node", "app.js"]
```

### 扫描漏洞

```bash
# 使用 Trivy 扫描镜像
trivy image myapp:latest

# 使用 Snyk 扫描
snyk container test myapp:latest

# 使用 Docker Scout
docker scout cves myapp:latest
```

## Docker Compose 进阶

### 多环境配置

```yaml
# docker-compose.yml (基础)
version: '3.8'
services:
  web:
    build: .
    ports:
      - "3000:3000"

# docker-compose.override.yml (开发环境)
version: '3.8'
services:
  web:
    environment:
      NODE_ENV: development
    volumes:
      - ./src:/app/src

# docker-compose.prod.yml (生产环境)
version: '3.8'
services:
  web:
    environment:
      NODE_ENV: production
    restart: always
    deploy:
      replicas: 3
```

```bash
# 开发环境（自动加载 override）
docker-compose up

# 生产环境
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

### 健康检查最佳实践

```yaml
services:
  web:
    image: myapp:latest
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s      # 检查间隔
      timeout: 10s       # 超时时间
      retries: 3         # 失败重试次数
      start_period: 40s  # 启动缓冲期

  db:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

### 日志管理

```yaml
services:
  web:
    image: myapp:latest
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "service=web"

  # 使用 syslog
  api:
    image: api:latest
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://192.168.1.100:514"
        tag: "api"
```

## 常用命令

### 镜像管理

```bash
# 构建镜像
docker build -t myapp:v1.0.0 .
docker build -t myapp:latest --target production .

# 列出镜像
docker images

# 删除未使用镜像
docker image prune -a

# 导出导入镜像
docker save myapp:v1.0.0 -o myapp.tar
docker load -i myapp.tar

# 查看镜像历史
docker history myapp:v1.0.0

# 查看镜像详情
docker inspect myapp:v1.0.0
```

### 容器管理

```bash
# 运行容器
docker run -d --name myapp -p 3000:3000 myapp:latest

# 查看容器日志
docker logs -f myapp
docker logs --tail 100 myapp

# 进入容器
docker exec -it myapp sh

# 查看容器资源使用
docker stats myapp

# 查看容器详情
docker inspect myapp

# 停止和删除
docker stop myapp
docker rm myapp

# 清理停止的容器
docker container prune
```

### 网络管理

```bash
# 创建网络
docker network create myapp-network

# 列出网络
docker network ls

# 查看网络详情
docker network inspect myapp-network

# 连接容器到网络
docker network connect myapp-network mycontainer
```

### 卷管理

```bash
# 创建卷
docker volume create mydata

# 列出卷
docker volume ls

# 查看卷详情
docker volume inspect mydata

# 删除未使用卷
docker volume prune
```

## 性能优化

### 构建缓存

```bash
# 使用 BuildKit（更快的构建）
export DOCKER_BUILDKIT=1
docker build -t myapp:latest .

# 使用缓存从仓库
docker build --cache-from myregistry/myapp:latest -t myapp:latest .
```

### 镜像大小优化

```dockerfile
# ✅ 使用 Alpine 基础镜像
FROM node:18-alpine

# ✅ 清理安装缓存
RUN apk add --no-cache python3 make g++ && \
    npm ci --only=production && \
    apk del python3 make g++

# ✅ 使用 distroless 镜像（更小更安全）
FROM gcr.io/distroless/nodejs18-debian11
COPY --from=builder /app /app
CMD ["app/main.js"]
```

## 检查清单

- [ ] 使用官方或可信的基础镜像
- [ ] 固定基础镜像版本（避免使用 latest）
- [ ] 多阶段构建减少镜像大小
- [ ] 使用 .dockerignore 排除不必要文件
- [ ] 合并 RUN 命令减少层数
- [ ] 创建并使用非 root 用户
- [ ] 配置健康检查
- [ ] 固定依赖版本
- [ ] 设置资源限制
- [ ] 扫描镜像漏洞
- [ ] 使用标签管理镜像版本
- [ ] 配置日志驱动
- [ ] 定期更新基础镜像
