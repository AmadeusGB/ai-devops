---
name: container-orchestration
description: 容器编排最佳实践指南。涵盖 Docker、Kubernetes、Docker Compose 等容器技术的使用规范、配置模式和 DevOps 最佳实践。适用于编写、修改和审查容器配置文件。触发关键词包括 docker, kubernetes, k8s, container, deployment, pod, service, 容器, 编排, 部署。
---

# 容器编排（Container Orchestration）

## 目的

为 DevOps 工程师和运维人员提供容器编排的最佳实践指南，确保容器配置的质量、可维护性和安全性。涵盖 Docker、Kubernetes、Docker Compose 等主流容器技术的使用规范和常见模式。

## 何时使用

此技能在以下场景自动激活：

- 编写或修改 Dockerfile
- 创建或更新 Kubernetes 清单文件（deployment.yaml, service.yaml 等）
- 编辑 Docker Compose 配置（docker-compose.yml）
- 提到 Docker、Kubernetes、容器、编排、部署等关键词
- 处理容器化应用、微服务架构相关任务

## 核心原则

1. **最小化镜像** - 构建尽可能小的容器镜像，减少攻击面和部署时间
2. **不可变基础设施** - 容器不可变，配置通过环境变量或 ConfigMap 注入
3. **声明式配置** - 使用 YAML 声明式配置，便于版本控制和审计
4. **健康检查** - 所有容器都应配置健康检查，实现自动故障恢复

## 快速参考

### Docker 基础

```dockerfile
# ✅ 正确的 Dockerfile 示例
FROM node:18-alpine AS builder

# 设置工作目录
WORKDIR /app

# 复制依赖文件（利用缓存）
COPY package*.json ./
RUN npm ci --only=production

# 复制源代码
COPY . .
RUN npm run build

# 生产镜像
FROM node:18-alpine

# 创建非 root 用户
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# 从 builder 复制构建产物
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules

# 使用非 root 用户
USER nodejs

# 暴露端口
EXPOSE 3000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# 启动命令
CMD ["node", "dist/main.js"]
```

**详细指南**: [Docker 详细指南](resources/docker-guide.md)

### Kubernetes 基础

```yaml
# ✅ 正确的 Deployment 配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: v1.0.0
    spec:
      # 安全上下文
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001

      containers:
      - name: web
        image: myregistry/web-app:v1.0.0
        imagePullPolicy: IfNotPresent

        # 资源限制
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"

        # 端口
        ports:
        - containerPort: 3000
          name: http
          protocol: TCP

        # 环境变量
        env:
        - name: NODE_ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password

        # 健康检查
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3

        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3

---
# ✅ 正确的 Service 配置
apiVersion: v1
kind: Service
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
    name: http
  sessionAffinity: None
```

**详细指南**: [Kubernetes 详细指南](resources/kubernetes-guide.md)

### Docker Compose 基础

```yaml
# ✅ 正确的 docker-compose.yml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    image: web-app:latest
    container_name: web-app
    restart: unless-stopped

    # 资源限制
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

    # 端口映射
    ports:
      - "3000:3000"

    # 环境变量
    environment:
      NODE_ENV: production
      DB_HOST: db
      DB_PORT: 5432

    # 敏感信息使用 secrets
    secrets:
      - db_password

    # 健康检查
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s

    # 依赖
    depends_on:
      db:
        condition: service_healthy

    # 网络
    networks:
      - app-network

    # 日志配置
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:15-alpine
    container_name: postgres-db
    restart: unless-stopped

    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password

    # 数据持久化
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro

    # 健康检查
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
      timeout: 5s
      retries: 5

    networks:
      - app-network

# 密钥
secrets:
  db_password:
    file: ./secrets/db_password.txt

# 卷
volumes:
  db-data:
    driver: local

# 网络
networks:
  app-network:
    driver: bridge
```

## 安全最佳实践

### 镜像安全

```dockerfile
# ✅ 正确：使用官方基础镜像和非 root 用户
FROM node:18-alpine

# 安装安全更新
RUN apk update && apk upgrade && rm -rf /var/cache/apk/*

# 创建非 root 用户
RUN addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001

# 切换到非 root 用户
USER appuser

# ❌ 错误：使用 root 用户运行
FROM node:18
WORKDIR /app
COPY . .
CMD ["node", "app.js"]  # 以 root 运行！
```

### Kubernetes 安全

```yaml
# ✅ 正确：Pod 安全配置
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault

  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL

    # 使用只读文件系统 + 临时卷
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/.cache

  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

**详细安全指南**: [容器安全最佳实践](resources/security-guide.md)

## 常见错误和解决方案

### Docker 镜像层过多

```dockerfile
# ❌ 错误：每个命令创建一层
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
RUN apt-get clean

# ✅ 正确：合并命令减少层数
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### Kubernetes Pod 无法启动

```bash
# 诊断步骤
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by='.lastTimestamp'

# 常见原因
# 1. 镜像拉取失败
# 2. 资源限制不足
# 3. 健康检查失败
# 4. ConfigMap/Secret 不存在
```

### Docker Compose 服务依赖

```yaml
# ❌ 错误：仅使用 depends_on
depends_on:
  - db  # 只等待容器启动，不等待服务就绪

# ✅ 正确：使用健康检查条件
depends_on:
  db:
    condition: service_healthy
```

## 项目结构最佳实践

### Docker 项目

```
project/
├── Dockerfile              # 主 Dockerfile
├── .dockerignore          # 忽略文件
├── docker-compose.yml     # 本地开发
├── docker-compose.prod.yml # 生产配置
└── scripts/
    ├── build.sh           # 构建脚本
    └── push.sh            # 推送脚本
```

### Kubernetes 项目

```
k8s/
├── base/                  # 基础配置
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/              # 环境覆盖
│   ├── dev/
│   ├── staging/
│   └── production/
└── charts/                # Helm Charts（可选）
```

## 检查清单

### Docker
- [ ] 使用官方或可信的基础镜像
- [ ] 多阶段构建减少镜像大小
- [ ] 使用 .dockerignore 排除不必要文件
- [ ] 配置健康检查
- [ ] 使用非 root 用户运行
- [ ] 合并 RUN 命令减少层数
- [ ] 固定依赖版本
- [ ] 扫描镜像漏洞（trivy, snyk）

### Kubernetes
- [ ] 配置资源限制（requests/limits）
- [ ] 配置健康检查（liveness/readiness）
- [ ] 使用 Pod 安全上下文
- [ ] 敏感信息使用 Secret
- [ ] 配置滚动更新策略
- [ ] 使用标签和选择器
- [ ] 配置亲和性规则（可选）
- [ ] 运行 kubectl apply --dry-run

### Docker Compose
- [ ] 固定服务镜像版本
- [ ] 配置资源限制
- [ ] 使用健康检查
- [ ] 敏感信息使用 secrets
- [ ] 配置重启策略
- [ ] 数据持久化使用 volumes
- [ ] 配置日志驱动
- [ ] 使用网络隔离

### 通用
- [ ] 配置文件已版本控制
- [ ] 清晰的提交信息
- [ ] 通过 Code Review
- [ ] 更新相关文档
- [ ] 非生产环境测试
- [ ] 有回滚计划

## 相关资源

- [Docker 详细指南](resources/docker-guide.md) - 完整的 Docker 最佳实践
- [Kubernetes 详细指南](resources/kubernetes-guide.md) - 完整的 Kubernetes 最佳实践
- [容器安全最佳实践](resources/security-guide.md) - 容器安全指南

---

**提示**：容器化是现代 DevOps 的基础，确保每次更改都经过充分测试和安全扫描。
