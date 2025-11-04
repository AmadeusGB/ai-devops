# Kubernetes 详细指南

## 项目结构

```
k8s/
├── base/                  # 基础配置（Kustomize）
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── kustomization.yaml
├── overlays/              # 环境覆盖
│   ├── dev/
│   │   └── kustomization.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
└── charts/                # Helm Charts（可选）
    └── myapp/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

## Deployment 最佳实践

### 完整的 Deployment 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    version: v1.0.0
    component: backend
  annotations:
    description: "Web application backend"
spec:
  replicas: 3

  # 滚动更新策略
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 更新时最多多出的 Pod 数
      maxUnavailable: 0    # 更新时最多不可用的 Pod 数

  # 选择器
  selector:
    matchLabels:
      app: web-app

  # Pod 模板
  template:
    metadata:
      labels:
        app: web-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"

    spec:
      # 服务账号
      serviceAccountName: web-app-sa

      # 安全上下文（Pod 级别）
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
        seccompProfile:
          type: RuntimeDefault

      # 初始化容器
      initContainers:
      - name: init-db
        image: busybox:1.35
        command: ['sh', '-c', 'until nc -z db 5432; do echo waiting for db; sleep 2; done']

      # 主容器
      containers:
      - name: web
        image: myregistry/web-app:v1.0.0
        imagePullPolicy: IfNotPresent

        # 安全上下文（容器级别）
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
              - ALL

        # 端口
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 9090
          name: metrics
          protocol: TCP

        # 环境变量
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "8080"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db.host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password

        # 资源限制
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        # 存活探针
        livenessProbe:
          httpGet:
            path: /health
            port: http
            httpHeaders:
            - name: X-Custom-Header
              value: Awesome
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3

        # 就绪探针
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3

        # 启动探针（适用于慢启动应用）
        startupProbe:
          httpGet:
            path: /startup
            port: http
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 30

        # 卷挂载
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/.cache
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: logs
          mountPath: /app/logs

      # 卷
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      - name: config
        configMap:
          name: app-config
      - name: logs
        persistentVolumeClaim:
          claimName: app-logs-pvc

      # 亲和性规则
      affinity:
        # Pod 反亲和性（不同节点）
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-app
              topologyKey: kubernetes.io/hostname

        # 节点亲和性
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-type
                operator: In
                values:
                - compute

      # 容忍污点
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "compute"
        effect: "NoSchedule"
```

## Service 配置

### ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
  namespace: production
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
  sessionAffinity: ClientIP  # 会话亲和性
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

### LoadBalancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 443
    targetPort: 8080
    protocol: TCP
```

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-headless
spec:
  clusterIP: None  # Headless
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
```

## ConfigMap 和 Secret

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  # 键值对
  db.host: "postgres.database.svc.cluster.local"
  db.port: "5432"
  log.level: "info"

  # 文件内容
  app.conf: |
    server {
      listen 8080;
      location / {
        proxy_pass http://backend;
      }
    }
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
data:
  # Base64 编码
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
stringData:
  # 明文（自动编码）
  api-key: "my-secret-api-key"
```

```bash
# 创建 Secret（命令行）
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=password \
  --namespace=production

# 从文件创建
kubectl create secret generic tls-secret \
  --from-file=tls.crt=cert.pem \
  --from-file=tls.key=key.pem
```

## Ingress 配置

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret

  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

## 资源配额和限制

### ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    persistentvolumeclaims: "10"
    pods: "50"
```

### LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
  - max:
      cpu: "2"
      memory: "4Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    type: Container
```

## 常用命令

### 部署管理

```bash
# 应用配置
kubectl apply -f deployment.yaml
kubectl apply -k overlays/production  # Kustomize

# 查看部署
kubectl get deployments
kubectl get pods -l app=web-app
kubectl describe deployment web-app

# 滚动更新
kubectl set image deployment/web-app web=myapp:v2.0.0
kubectl rollout status deployment/web-app

# 回滚
kubectl rollout undo deployment/web-app
kubectl rollout history deployment/web-app
kubectl rollout undo deployment/web-app --to-revision=2

# 扩缩容
kubectl scale deployment web-app --replicas=5
kubectl autoscale deployment web-app --min=3 --max=10 --cpu-percent=80
```

### 调试

```bash
# 查看日志
kubectl logs -f pod-name
kubectl logs -f pod-name -c container-name
kubectl logs --tail=100 pod-name

# 进入容器
kubectl exec -it pod-name -- sh
kubectl exec -it pod-name -c container-name -- sh

# 端口转发
kubectl port-forward pod-name 8080:8080
kubectl port-forward service/web-app 8080:80

# 查看事件
kubectl get events --sort-by='.lastTimestamp'
kubectl describe pod pod-name

# 查看资源使用
kubectl top nodes
kubectl top pods
```

### 配置管理

```bash
# ConfigMap
kubectl create configmap app-config --from-file=config.yaml
kubectl get configmap app-config -o yaml
kubectl edit configmap app-config

# Secret
kubectl create secret generic db-secret --from-literal=password=secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d

# 查看配置
kubectl get cm,secret
```

## HPA（水平 Pod 自动扩缩容）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

## 检查清单

- [ ] 配置资源限制（requests/limits）
- [ ] 配置健康检查（liveness/readiness/startup）
- [ ] 使用 Pod 安全上下文（非 root、只读根文件系统）
- [ ] 敏感信息使用 Secret
- [ ] 配置使用 ConfigMap
- [ ] 使用标签和选择器
- [ ] 配置滚动更新策略
- [ ] 配置 PodDisruptionBudget
- [ ] 使用 Kustomize 或 Helm 管理配置
- [ ] 配置 HPA 自动扩缩容
- [ ] 配置亲和性和反亲和性规则
- [ ] 使用命名空间隔离
- [ ] 运行 kubectl apply --dry-run
- [ ] 配置网络策略（NetworkPolicy）
