# Docker - 问题解决

---
tags: [问题解决, Docker, 容器化, Kubernetes, 性能优化, 故障排查]
created: 2026-02-21
updated: 2026-02-21
status: 持续更新
importance: ⭐⭐⭐⭐
---

## 🚨 高频问题速查

### 问题1：容器启动失败 `#启动失败`
**现象**：容器无法正常启动或立即退出
**原因**：镜像问题、端口冲突、资源不足、配置错误
**解决**：
1. 检查容器日志
2. 验证镜像完整性
3. 检查端口占用
4. 调整资源限制

```bash
# 查看容器日志
docker logs container_name
docker logs -f --tail 100 container_name

# 检查容器状态
docker ps -a
docker inspect container_name

# 进入容器调试
docker exec -it container_name /bin/bash
docker run -it --entrypoint /bin/bash image_name

# 检查端口占用
netstat -tulpn | grep :8080
lsof -i :8080

# 资源使用情况
docker stats
docker system df
```

**相关原理**：[[Docker核心概念#容器生命周期]]

---

### 问题2：镜像构建缓慢 `#构建优化`
**现象**：Docker镜像构建时间过长
**原因**：层缓存失效、依赖下载慢、构建上下文过大
**解决**：
1. 优化Dockerfile层顺序
2. 使用多阶段构建
3. 减小构建上下文
4. 使用镜像缓存

```dockerfile
# 优化前：每次都重新下载依赖
FROM node:16
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

# 优化后：利用层缓存
FROM node:16-alpine AS builder
WORKDIR /app

# 先复制依赖文件，利用缓存
COPY package*.json ./
RUN npm ci --only=production

# 再复制源码
COPY src ./src
COPY public ./public
RUN npm run build

# 生产镜像
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

# .dockerignore 文件
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
coverage
.nyc_output
```

---

### 问题3：容器内存溢出 `#内存问题`
**现象**：容器因内存不足被系统杀死（OOMKilled）
**原因**：内存限制设置不当、应用内存泄漏、JVM参数配置错误
**解决**：
1. 调整内存限制
2. 优化JVM参数
3. 监控内存使用
4. 修复内存泄漏

```bash
# 设置内存限制
docker run -m 2g --memory-swap 2g myapp

# JVM内存优化
docker run -e JAVA_OPTS="-Xmx1536m -Xms1024m -XX:+UseG1GC" myapp

# 监控内存使用
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"

# 查看系统内存
free -h
cat /proc/meminfo
```

```yaml
# Kubernetes资源限制
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "500m"
    env:
    - name: JAVA_OPTS
      value: "-Xmx768m -Xms512m -XX:+UseContainerSupport"
```

---

### 问题4：网络连接问题 `#网络问题`
**现象**：容器间无法通信或外部无法访问容器服务
**原因**：网络配置错误、防火墙阻拦、DNS解析问题
**解决**：
1. 检查网络配置
2. 验证端口映射
3. 测试网络连通性
4. 配置DNS解析

```bash
# 查看网络配置
docker network ls
docker network inspect bridge

# 创建自定义网络
docker network create --driver bridge mynetwork

# 容器加入网络
docker run --network mynetwork --name app1 myapp
docker run --network mynetwork --name app2 myapp

# 测试网络连通性
docker exec app1 ping app2
docker exec app1 nslookup app2
docker exec app1 telnet app2 8080

# 端口映射检查
docker port container_name
netstat -tulpn | grep docker-proxy
```

```yaml
# Docker Compose网络配置
version: '3.8'
services:
  app:
    image: myapp
    networks:
      - frontend
      - backend
    ports:
      - "8080:8080"

  db:
    image: mysql:8.0
    networks:
      - backend
    environment:
      MYSQL_ROOT_PASSWORD: password

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # 内部网络，不能访问外网
```

---

### 问题5：存储卷问题 `#存储问题`
**现象**：数据丢失、权限错误、磁盘空间不足
**原因**：卷挂载配置错误、文件权限问题、存储空间不足
**解决**：
1. 正确配置卷挂载
2. 设置合适的文件权限
3. 监控磁盘使用
4. 清理无用数据

```bash
# 查看卷信息
docker volume ls
docker volume inspect volume_name

# 创建命名卷
docker volume create mydata

# 挂载卷
docker run -v mydata:/data myapp
docker run -v /host/path:/container/path myapp

# 权限问题解决
# 在Dockerfile中设置用户
FROM ubuntu:20.04
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
RUN chown -R appuser:appgroup /app
USER appuser

# 或在运行时指定用户
docker run --user $(id -u):$(id -g) -v /host/path:/data myapp

# 清理无用数据
docker system prune -a
docker volume prune
docker image prune -a
```

```yaml
# Kubernetes持久化存储
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd

---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        volumeMounts:
        - name: data-volume
          mountPath: /app/data
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: app-data-pvc
```

## 🔧 调试技巧

### 常用调试方法

#### 1. 容器调试
```bash
# 进入运行中的容器
docker exec -it container_name /bin/bash

# 以root用户进入
docker exec -it --user root container_name /bin/bash

# 查看容器进程
docker exec container_name ps aux

# 查看容器文件系统
docker exec container_name ls -la /app

# 复制文件到容器
docker cp local_file container_name:/path/to/destination

# 从容器复制文件
docker cp container_name:/path/to/file local_destination

# 查看容器变更
docker diff container_name

# 提交容器为新镜像（调试用）
docker commit container_name debug_image:latest
```

#### 2. 日志分析
```bash
# 实时查看日志
docker logs -f container_name

# 查看最近的日志
docker logs --tail 100 container_name

# 查看特定时间段的日志
docker logs --since "2024-01-01T00:00:00" --until "2024-01-01T23:59:59" container_name

# 搜索日志内容
docker logs container_name 2>&1 | grep "ERROR"

# 导出日志到文件
docker logs container_name > container.log 2>&1
```

#### 3. 性能监控
```bash
# 实时监控容器资源使用
docker stats

# 查看特定容器的资源使用
docker stats container_name

# 监控脚本
#!/bin/bash
while true; do
    echo "=== $(date) ==="
    docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
    echo ""
    sleep 30
done
```

### 性能分析工具

#### 1. 容器性能分析
```bash
# 使用cAdvisor监控
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  gcr.io/cadvisor/cadvisor:latest

# 使用Prometheus监控
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

#### 2. 应用性能监控
```yaml
# 应用监控配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: monitoring-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-monitoring
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE
          value: "health,info,prometheus"
```

### 故障排查流程

#### 1. 系统性故障排查
```bash
#!/bin/bash
# Docker故障排查脚本

echo "=== Docker系统信息 ==="
docker version
docker info

echo "=== 系统资源使用 ==="
df -h
free -h
top -bn1 | head -20

echo "=== Docker资源使用 ==="
docker system df
docker stats --no-stream

echo "=== 容器状态 ==="
docker ps -a

echo "=== 网络状态 ==="
docker network ls
netstat -tulpn | grep docker

echo "=== 卷状态 ==="
docker volume ls

echo "=== 最近的系统日志 ==="
journalctl -u docker.service --since "1 hour ago" --no-pager
```

#### 2. Kubernetes故障排查
```bash
#!/bin/bash
# K8s故障排查脚本

NAMESPACE=${1:-default}
APP_NAME=${2:-myapp}

echo "=== 集群状态 ==="
kubectl cluster-info
kubectl get nodes

echo "=== 应用状态 ==="
kubectl get pods -n $NAMESPACE -l app=$APP_NAME
kubectl get svc -n $NAMESPACE -l app=$APP_NAME
kubectl get ingress -n $NAMESPACE

echo "=== 事件信息 ==="
kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp'

echo "=== Pod详细信息 ==="
kubectl describe pods -n $NAMESPACE -l app=$APP_NAME

echo "=== 应用日志 ==="
kubectl logs -n $NAMESPACE -l app=$APP_NAME --tail=100

echo "=== 资源使用 ==="
kubectl top nodes
kubectl top pods -n $NAMESPACE
```

## 🔗 相关文档

- **技术原理**：[[Docker核心概念]]
- **实战应用**：[[Kubernetes实战]] [[CI/CD流水线]]

## 🏷️ 标签
#问题解决 #Docker #容器化 #Kubernetes #性能优化 #故障排查