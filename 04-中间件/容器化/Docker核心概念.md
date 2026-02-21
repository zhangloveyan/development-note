# Docker - 核心概念

---
tags: [Docker, 容器化, 虚拟化, DevOps, 微服务]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Docker容器化技术的核心概念和实践

- **容器化**：轻量级虚拟化技术，实现应用隔离
- **镜像管理**：构建、存储和分发应用镜像
- **容器编排**：Kubernetes集群管理和服务编排
- **DevOps集成**：CI/CD流水线和自动化部署

## 💡 原理详解

### 1. Docker基础概念

Docker是一个开源的容器化平台，使用Linux内核的cgroup和namespace技术实现进程隔离。

#### 核心组件
- **Docker Engine**：Docker的核心运行时
- **Docker Image**：只读的应用模板
- **Docker Container**：镜像的运行实例
- **Docker Registry**：镜像仓库服务
- **Dockerfile**：构建镜像的脚本文件

#### 架构对比
```
传统虚拟化：
Hardware → Host OS → Hypervisor → Guest OS → App

Docker容器化：
Hardware → Host OS → Docker Engine → Container → App
```

### 2. 容器 vs 虚拟机

| 特性 | 容器 | 虚拟机 |
|------|------|--------|
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | 低 | 高 |
| 隔离程度 | 进程级 | 系统级 |
| 可移植性 | 高 | 中等 |
| 管理复杂度 | 低 | 高 |

### 3. Kubernetes架构

Kubernetes是容器编排平台，提供自动化部署、扩缩容和管理功能。

#### 核心概念
- **Pod**：最小调度单元，包含一个或多个容器
- **Service**：为Pod提供稳定的网络访问
- **Deployment**：管理Pod的副本和更新策略
- **ConfigMap/Secret**：配置和敏感信息管理
- **Ingress**：集群外部访问的入口

## 🔧 代码示例

### 基础用法

#### Dockerfile示例
```dockerfile
# 基础镜像
FROM openjdk:17-jdk-slim

# 设置工作目录
WORKDIR /app

# 复制依赖文件
COPY pom.xml .
COPY src ./src

# 构建应用
RUN ./mvnw clean package -DskipTests

# 复制jar文件
COPY target/app.jar app.jar

# 暴露端口
EXPOSE 8080

# 设置环境变量
ENV JAVA_OPTS="-Xmx512m -Xms256m"
ENV SPRING_PROFILES_ACTIVE=prod

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# 启动命令
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

#### 多阶段构建
```dockerfile
# 构建阶段
FROM maven:3.8.4-openjdk-17 AS builder

WORKDIR /app
COPY pom.xml .
COPY src ./src

RUN mvn clean package -DskipTests

# 运行阶段
FROM openjdk:17-jdk-slim

WORKDIR /app

# 只复制构建产物
COPY --from=builder /app/target/app.jar app.jar

# 创建非root用户
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Docker Compose配置
```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=jdbc:mysql://db:3306/myapp
      - REDIS_HOST=redis
    depends_on:
      - db
      - redis
    volumes:
      - ./logs:/app/logs
    networks:
      - app-network
    restart: unless-stopped

  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=myapp
      - MYSQL_USER=appuser
      - MYSQL_PASSWORD=apppassword
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - app-network
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    networks:
      - app-network
    restart: unless-stopped

volumes:
  mysql_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### 高级用法

#### Kubernetes部署配置
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "k8s"
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database.url
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: database.password
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        - name: log-volume
          mountPath: /app/logs
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: log-volume
        persistentVolumeClaim:
          claimName: app-logs-pvc

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.url: "jdbc:mysql://mysql-service:3306/myapp"
  redis.host: "redis-service"
  application.yml: |
    server:
      port: 8080
    logging:
      level:
        com.myapp: DEBUG

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  database.password: cGFzc3dvcmQ=  # base64 encoded
  redis.password: cmVkaXNwYXNz

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

#### 生产环境部署脚本
```bash
#!/bin/bash

# 生产环境Docker部署脚本

set -e

# 配置变量
APP_NAME="myapp"
VERSION=${1:-latest}
REGISTRY="registry.example.com"
NAMESPACE="production"

echo "开始部署 $APP_NAME:$VERSION 到生产环境..."

# 1. 构建镜像
echo "构建Docker镜像..."
docker build -t $APP_NAME:$VERSION .
docker tag $APP_NAME:$VERSION $REGISTRY/$APP_NAME:$VERSION

# 2. 推送镜像
echo "推送镜像到仓库..."
docker push $REGISTRY/$APP_NAME:$VERSION

# 3. 更新Kubernetes部署
echo "更新Kubernetes部署..."
kubectl set image deployment/$APP_NAME-deployment \
  $APP_NAME=$REGISTRY/$APP_NAME:$VERSION \
  -n $NAMESPACE

# 4. 等待部署完成
echo "等待部署完成..."
kubectl rollout status deployment/$APP_NAME-deployment -n $NAMESPACE

# 5. 验证部署
echo "验证部署状态..."
kubectl get pods -l app=$APP_NAME -n $NAMESPACE

# 6. 健康检查
echo "执行健康检查..."
INGRESS_IP=$(kubectl get ingress $APP_NAME-ingress -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -f http://$INGRESS_IP/actuator/health || {
  echo "健康检查失败，回滚部署..."
  kubectl rollout undo deployment/$APP_NAME-deployment -n $NAMESPACE
  exit 1
}

echo "部署成功完成！"
```

#### 通用镜像启动方式
```bash
# 通用镜像构建
docker build -t device-manager:latest .

# 灵活启动容器
docker run -d \
  --name device-manager-app \
  -p 9090:9090 \
  -m 3g \
  --memory-swap -1 \
  -v /var/www:/var/www \
  -v /wxlogs/device-manager:/wxlogs/device-manager \
  -v /etc/localtime:/etc/localtime:ro \
  -v /etc/timezone:/etc/timezone:ro \
  -v /home/deploy/app:/data \
  --log-opt max-size=100m \
  --log-opt max-file=3 \
  --net host \
  --restart unless-stopped \
  device-manager:latest \
  java -jar /data/app.jar \
  -Xmx2048m -Xms1024m -Xmn512m \
  --spring.config.location=/data/application-prod.yaml \
  --add-opens=java.base/java.lang=ALL-UNNAMED

# 参数说明：
# -d: 后台运行
# -m 3g: 限制内存使用
# --memory-swap -1: 禁用swap
# -v: 挂载卷
# --log-opt: 日志轮转配置
# --net host: 使用主机网络
# --restart: 重启策略
```

## ⚡ 性能特点

| 特性 | 说明 | 适用场景 |
|------|------|----------|
| 轻量级 | 共享内核，资源占用少 | 微服务架构 |
| 快速启动 | 秒级启动时间 | 弹性扩缩容 |
| 一致性 | 环境一致性保证 | 开发测试部署 |
| 可移植性 | 跨平台运行 | 混合云部署 |

## 🔗 知识关联

- **前置知识**：[[Linux基础]] [[网络基础]]
- **相关技术**：[[Kubernetes]] [[Docker Compose]]
- **实战应用**：[[微服务部署]] [[CI/CD流水线]]
- **问题解决**：[[Docker问题解决]]

## 🏷️ 标签
#Docker #容器化 #Kubernetes #DevOps #微服务 #虚拟化 #面试重点