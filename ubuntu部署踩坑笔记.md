# Docker & Linux 运维踩坑笔记（生产实战版）

> 适用对象：
> - Ubuntu 云服务器（腾讯云 / 阿里云 / AWS 等）
> - Docker / Docker Compose 部署后端、AI、Java、Python 服务
> - 真实生产环境，长期运行、可维护

---

## 目录

1. Docker Compose 版本问题
2. Linux 用户 / 权限模型（生产实践）
3. Docker daemon 权限（docker.sock）
4. Docker 容器内用户 & 宿主机权限
5. Docker Volume 挂载规则
6. 文件传输 & sudo 的现实问题（FinalShell / SFTP）
7. SSH 密钥 / 登录问题
8. Docker Network 常见问题
9. 最终推荐 Checklist（一次配置，长期不踩坑）
10. 一句话总纲

---

## 1️⃣ Docker Compose 版本问题（必须重点记住）

### ❌ 常见坑

- `KeyError: 'ContainerConfig'`
- build 成功，`up / recreate` 失败
- legacy builder deprecated 提示

### 🎯 根因

- 使用 **docker-compose v1（1.29.x，Python 版，已废弃）**
- Docker Engine / BuildKit 已升级
- v1 与新镜像 metadata **不兼容**

### ✅ 生产唯一推荐

- 使用 **Docker Compose v2（插件版）**
- 命令形式：

```bash
docker compose up -d
```

⚠️ 注意：`docker` 和 `compose` 中间有空格

### 🔧 安装（通用）

```bash
# 使用 ubuntu 用户，不要切 root
mkdir -p ~/.docker/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.25.0/docker-compose-linux-x86_64 \
  -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose
```

### 📌 额外说明

- `version:` 字段在 compose v2 中 **已废弃**
- 不影响运行，但建议删除

---

## 2️⃣ Linux 用户 / 权限模型（真实生产实践）

### ❌ 常见误区

- 登录 ubuntu，但长期 `sudo su`
- 项目目录归 root
- 文件 / volume / compose 权限混乱

### 🎯 Linux 铁律

> **谁执行命令，谁就必须对文件有权限**

### ✅ 推荐用户模型（生产标准）

| 项目 | 推荐 |
|----|----|
| 登录用户 | ubuntu |
| 日常操作 | ubuntu |
| 系统级操作 | sudo |
| 项目目录 owner | ubuntu |
| docker 命令 | 不用 sudo |

### 📁 推荐目录结构

```text
/home/ubuntu/projects/xxx
或
/opt/xxx   （owner 必须是 ubuntu）
```

### ❌ 不推荐

```bash
sudo su
# 用 root 干一整天活
```

---

## 3️⃣ Docker daemon 权限（docker.sock）

### ❌ 坑现象

```text
permission denied while trying to connect to the Docker daemon socket
/var/run/docker.sock
```

### 🎯 根因

- ubuntu 用户 **不在 docker 组**
- docker.sock 权限：

```text
srw-rw---- root docker /var/run/docker.sock
```

### ✅ 正确修复（一次性）

```bash
sudo usermod -aG docker ubuntu
exit  # 必须重新登录
```

验证：

```bash
docker ps
```

### ❌ 错误做法（但很常见）

```bash
sudo docker compose up
```

问题：
- root / ubuntu 使用不同 ~/.docker
- compose 插件路径混乱
- volume 权限更难控

---

## 4️⃣ Docker 容器内用户 & 宿主机权限

### 🎯 核心认知

> **容器在宿主机创建文件，用的是容器内进程的 UID/GID**

与执行 docker 命令的用户（ubuntu / root）**无关**。

### ❌ 默认灾难场景

| 项目 | 结果 |
|----|----|
| 容器用户 | root |
| 使用 volume | 是 |
| 宿主机文件 | root:root |
| ubuntu 操作 | permission denied |

### ✅ 生产推荐方案

#### 方案 A（最佳）：容器使用非 root 用户

```dockerfile
RUN groupadd -g 1000 app && \
    useradd -u 1000 -g app -m app
USER app
```

📌 `1000` 通常就是 ubuntu

#### 方案 B（快捷）：compose 指定 user

```yaml
services:
  app:
    user: "1000:1000"
```

### ❌ 禁忌

```bash
chmod -R 777 data
```

---

## 5️⃣ Docker Volume 挂载规则（必背）

### ❌ 典型报错

```text
not a directory
Are you trying to mount a directory onto a file (or vice-versa)?
```

### 🎯 挂载铁律

| 宿主机 | 容器内 | 是否允许 |
|----|----|----|
| 文件 | 文件 | ✅ |
| 目录 | 目录 | ✅ |
| 文件 | 目录 | ❌ |
| 目录 | 文件 | ❌ |

### ✅ 生产标准

**永远优先挂目录，不挂单文件**

```yaml
volumes:
  - /opt/project/models:/opt/app/models
```

---

## 6️⃣ 文件传输 & sudo 的现实问题（FinalShell / SFTP）

### 🎯 事实

- SFTP 会话身份 = 登录用户（ubuntu）
- `sudo su` **不会改变 SFTP 身份**
- root 无法拖拽文件（正常现象）

### ✅ 正确姿势

1. 拖拽到：

```text
/home/ubuntu/
```

2. 再移动：

```bash
sudo mv xxx /opt/xxx
sudo chown -R ubuntu:ubuntu /opt/xxx
```

---

## 7️⃣ SSH 密钥 / navicat 登录问题

### ❌ 常见坑

- navicat 通过 SSH 连接 数据库，服务器更换密码
- 使用新 SSH 连接，报错异常：

`The server key has changed. Either you are under attack or the administrator changed the key.New server key hash: f3:d6:7e:73:85:e7:b6:9b:be:fe:00:58:21:ff:54:66:0a:1b:0c:00 (9)`

含义是：

- Navicat **曾经保存过这台服务器的 SSH 指纹**
- 现在连接时，服务器返回的 **SSH 主机密钥指纹和之前记录的不一致**
- 为了防止 **中间人攻击（MITM）**，Navicat **拒绝继续连接**

### ✅ 标准配置流程

删掉重新连接，或者删除 Known Hosts 保存的密钥。

---

## 8️⃣ Docker Network 常见问题

### ❌ 常见坑 1：容器间无法通信

```text
Connection refused / Name or service not known
```

### 🎯 原因

- 不在同一个 docker network
- 使用了 localhost

### ✅ 正确做法

```yaml
networks:
  app-net:

services:
  api:
    networks:
      - app-net
  db:
    networks:
      - app-net
```

容器访问方式：

```text
db:3306
api:8080
```

或者使用宿主机内网地址连接

❌ 不要用：

```text
localhost
127.0.0.1
```

---

### ❌ 常见坑 2：端口已映射但访问不了

### 排查顺序

1. 容器是否监听 0.0.0.0
2. compose ports 是否正确
3. 云服务器安全组是否放行

---

## 9️⃣ 最终推荐 Checklist（一次配置，长期不踩坑）

```text
☑ ubuntu 用户登录
☑ ubuntu 加入 docker 组
☑ docker compose v2
☑ 项目目录归 ubuntu
☑ 容器不使用 root
☑ volume 只挂目录
☑ 容器通过 network 名称通信
☑ SSH 权限正确
```

---

## 🔟 一句话总纲（建议贴墙上）

> **不用 sudo 跑 docker**  
> **不用 root 跑容器**  
> **不单挂文件**  
> **不用 compose v1**  
> **容器不连 localhost**



