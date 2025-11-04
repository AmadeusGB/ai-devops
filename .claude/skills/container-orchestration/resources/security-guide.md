# 容器安全最佳实践

## Docker 安全

### 镜像安全

```dockerfile
# ✅ 正确：使用官方镜像和固定版本
FROM node:18.19.0-alpine3.19

# 安装安全更新
RUN apk update && \
    apk upgrade && \
    rm -rf /var/cache/apk/*

# 创建非 root 用户
RUN addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001 -G appuser

# 设置工作目录和权限
WORKDIR /app
COPY --chown=appuser:appuser . .

# 切换到非 root 用户
USER appuser

# ❌ 错误：使用 latest 标签和 root 用户
FROM node:latest
WORKDIR /app
COPY . .
CMD ["node", "app.js"]
```

### 最小权限原则

```dockerfile
# ✅ 正确：只读根文件系统
FROM node:18-alpine

RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app
COPY --chown=nodejs:nodejs . .
USER nodejs

# 运行时配置只读根文件系统
# docker run --read-only --tmpfs /tmp myapp
```

### 密钥管理

```dockerfile
# ✅ 正确：使用构建时密钥（BuildKit）
# syntax=docker/dockerfile:1.4
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./

# 使用密钥挂载
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --only=production

# ❌ 错误：硬编码密钥或复制密钥文件
COPY .npmrc /root/.npmrc  # 密钥会留在镜像层中！
RUN npm ci
RUN rm /root/.npmrc  # 删除无效，密钥已在前面的层中
```

### 扫描漏洞

```bash
# 使用 Trivy 扫描镜像
trivy image --severity HIGH,CRITICAL myapp:latest

# 使用 Grype 扫描
grype myapp:latest

# 使用 Snyk 扫描
snyk container test myapp:latest --file=Dockerfile

# Docker Scout 扫描
docker scout cves myapp:latest
docker scout quickview myapp:latest

# 在 CI 中集成扫描
# .github/workflows/security.yml
- name: Run Trivy scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'myapp:${{ github.sha }}'
    exit-code: '1'
    severity: 'HIGH,CRITICAL'
```

## Kubernetes 安全

### Pod 安全标准

```yaml
# ✅ 正确：完整的安全配置
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  # Pod 级别安全上下文
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
    fsGroupChangePolicy: "OnRootMismatch"
    seccompProfile:
      type: RuntimeDefault
    supplementalGroups: [1001]

  containers:
  - name: app
    image: myapp:v1.0.0

    # 容器级别安全上下文
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1001
      capabilities:
        drop:
          - ALL
        add:
          - NET_BIND_SERVICE  # 仅在需要时添加特定能力

    # 资源限制
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"

    # 只读根文件系统 + 临时卷
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

  # 自动挂载服务账号令牌（默认启用，按需禁用）
  automountServiceAccountToken: false
```

### Secret 管理

```yaml
# ✅ 正确：使用外部密钥管理（推荐）
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  annotations:
    # 使用 External Secrets Operator
    external-secrets.io/backend: vault
    external-secrets.io/vault-path: secret/data/db
type: Opaque

---
# ✅ 正确：使用 Sealed Secrets（加密存储在 Git）
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...

---
# 环境变量使用 Secret
spec:
  containers:
  - name: app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # 或作为文件挂载
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400  # 只读权限
```

### 网络策略

```yaml
# ✅ 正确：严格的网络策略
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-app

  policyTypes:
  - Ingress
  - Egress

  # 入站规则
  ingress:
  - from:
    # 仅允许来自 ingress-nginx 的流量
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
      podSelector:
        matchLabels:
          app: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080

  # 出站规则
  egress:
  # 允许访问 DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53

  # 允许访问数据库
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432

  # 允许访问外部 API
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

### RBAC（基于角色的访问控制）

```yaml
# ✅ 正确：最小权限 ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: production

---
# Role（命名空间级别）
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: web-app-role
  namespace: production
rules:
# 仅允许读取 ConfigMap
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
# 仅允许读取 Secret
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: web-app-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: web-app-sa
  namespace: production
roleRef:
  kind: Role
  name: web-app-role
  apiGroup: rbac.authorization.k8s.io
```

### Pod Security Standards（PSS）

```yaml
# ✅ 正确：在命名空间级别强制执行 Pod 安全标准
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Restricted 是最严格的标准
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## 镜像签名和验证

### Cosign 签名

```bash
# 安装 Cosign
curl -sL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o cosign
chmod +x cosign && sudo mv cosign /usr/local/bin/

# 生成密钥对
cosign generate-key-pair

# 签名镜像
cosign sign --key cosign.key myregistry/myapp:v1.0.0

# 验证镜像
cosign verify --key cosign.pub myregistry/myapp:v1.0.0
```

### Kubernetes 镜像签名验证（Kyverno）

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: enforce
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "myregistry/*"
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
```

## 运行时安全

### Falco 规则

```yaml
# 检测容器中的异常行为
- rule: Unexpected outbound connection
  desc: Detect unexpected outbound connections
  condition: >
    outbound and container and
    not fd.sip in (allowed_ips)
  output: >
    Unexpected outbound connection
    (user=%user.name command=%proc.cmdline connection=%fd.name)
  priority: WARNING

- rule: Write below root
  desc: Detect writes to root filesystem
  condition: >
    open_write and container and
    fd.name startswith "/" and
    not fd.name startswith "/tmp"
  output: >
    File write below root
    (user=%user.name command=%proc.cmdline file=%fd.name)
  priority: WARNING
```

## 供应链安全

### SBOM（软件物料清单）

```bash
# 使用 Syft 生成 SBOM
syft packages myapp:v1.0.0 -o spdx-json > sbom.json

# 使用 Grype 扫描 SBOM
grype sbom:sbom.json

# 在 Dockerfile 中集成
FROM node:18-alpine AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# 生成 SBOM
FROM anchore/syft:latest AS sbom
COPY --from=builder /app /app
RUN syft /app -o spdx-json > /sbom.json

# 最终镜像
FROM node:18-alpine
COPY --from=builder /app/dist /app/dist
COPY --from=sbom /sbom.json /sbom.json
```

### 依赖扫描

```bash
# 扫描 package.json 依赖
npm audit
npm audit fix

# 使用 Snyk 扫描
snyk test

# 使用 OWASP Dependency Check
dependency-check --scan . --format HTML --out report.html
```

## 安全检查清单

### Docker 镜像
- [ ] 使用官方或可信的基础镜像
- [ ] 固定镜像版本（避免使用 latest）
- [ ] 定期更新基础镜像和依赖
- [ ] 使用多阶段构建减少攻击面
- [ ] 创建并使用非 root 用户
- [ ] 扫描镜像漏洞（Trivy、Snyk）
- [ ] 不在镜像中硬编码密钥
- [ ] 签名镜像（Cosign）
- [ ] 生成和发布 SBOM
- [ ] 使用 .dockerignore 排除敏感文件

### Kubernetes 配置
- [ ] 配置 Pod 安全上下文（runAsNonRoot, readOnlyRootFilesystem）
- [ ] 删除所有 Linux 能力（drop: ALL）
- [ ] 配置资源限制（防止资源耗尽攻击）
- [ ] 敏感信息使用 Secret（或外部密钥管理）
- [ ] 配置网络策略（NetworkPolicy）
- [ ] 使用 RBAC 最小权限
- [ ] 禁用自动挂载服务账号令牌（如不需要）
- [ ] 启用 Pod Security Standards
- [ ] 配置准入控制器（OPA/Kyverno）
- [ ] 定期轮换密钥

### 运行时安全
- [ ] 部署运行时安全工具（Falco）
- [ ] 配置日志和审计
- [ ] 监控异常行为
- [ ] 定期安全扫描
- [ ] 配置告警和响应流程
- [ ] 定期进行渗透测试
- [ ] 实施零信任网络架构
- [ ] 使用服务网格（Istio/Linkerd）加密流量

### 供应链安全
- [ ] 使用私有镜像仓库
- [ ] 扫描依赖漏洞
- [ ] 生成和验证 SBOM
- [ ] 签名和验证镜像
- [ ] 使用可信的基础镜像
- [ ] 定期审查依赖
- [ ] 自动化安全扫描（CI/CD）
- [ ] 实施镜像准入策略
