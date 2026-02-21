# Nginx - 配置实战

---
tags: [Nginx, Web服务器, 反向代理, 负载均衡, 高性能]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Nginx高性能Web服务器的配置和实战应用

- **高性能**：事件驱动架构，支持高并发连接
- **反向代理**：负载均衡和请求转发功能
- **静态资源**：高效的静态文件服务
- **SSL/TLS**：HTTPS安全连接配置

## 💡 原理详解

### 1. Nginx架构特点

Nginx采用事件驱动的异步非阻塞架构：

- **Master进程**：管理Worker进程，处理信号
- **Worker进程**：处理客户端请求
- **事件驱动**：epoll/kqueue等高效I/O复用
- **内存池**：减少内存分配开销

### 2. 核心功能模块

#### HTTP核心模块
- **静态文件服务**：高效的文件传输
- **目录索引**：自动生成目录列表
- **访问控制**：IP白名单/黑名单
- **URL重写**：灵活的URL转换规则

#### 代理模块
- **反向代理**：隐藏后端服务器
- **负载均衡**：多种负载均衡算法
- **健康检查**：自动检测后端服务状态
- **缓存机制**：提高响应速度

### 3. 配置文件结构

```
nginx.conf
├── main context (全局配置)
├── events context (事件配置)
├── http context (HTTP配置)
│   ├── upstream (上游服务器)
│   └── server context (虚拟主机)
│       └── location context (位置匹配)
```

## 🔧 代码示例

### 基础用法

#### 基础配置文件
```nginx
# /etc/nginx/nginx.conf

# 全局配置
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

# 事件配置
events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

# HTTP配置
http {
    # 基础设置
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # 性能优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip压缩
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/xml+rss
        application/json;

    # 包含其他配置文件
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

#### 静态网站配置
```nginx
# /etc/nginx/sites-available/static-site
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/html;
    index index.html index.htm;

    # 访问日志
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;

    # 静态文件缓存
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;

    # 隐藏Nginx版本
    server_tokens off;

    # 404页面
    error_page 404 /404.html;
    location = /404.html {
        internal;
    }

    # 50x页面
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        internal;
    }
}
```

### 高级用法

#### 反向代理配置
```nginx
# 上游服务器定义
upstream backend {
    # 负载均衡方法
    least_conn;  # 最少连接数
    # ip_hash;   # IP哈希
    # fair;      # 响应时间

    # 后端服务器
    server 192.168.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 weight=1 max_fails=3 fail_timeout=30s backup;

    # 健康检查
    keepalive 32;
}

server {
    listen 80;
    server_name api.example.com;

    # 反向代理配置
    location / {
        proxy_pass http://backend;

        # 代理头设置
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时设置
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        # 缓冲设置
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;

        # 重试设置
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 30s;
    }

    # API路径特殊处理
    location /api/v1/ {
        proxy_pass http://backend/api/v1/;

        # 限流
        limit_req zone=api burst=10 nodelay;

        # CORS处理
        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
        add_header Access-Control-Allow-Headers "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization";

        if ($request_method = 'OPTIONS') {
            return 204;
        }
    }

    # 静态资源直接服务
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public";
    }
}
```

#### HTTPS和SSL配置
```nginx
# HTTP重定向到HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS配置
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL证书配置
    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # SSL安全配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA384;
    ssl_prefer_server_ciphers off;

    # SSL会话缓存
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### 限流和安全配置
```nginx
http {
    # 限流配置
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=global:10m rate=100r/s;

    # 连接限制
    limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;

    # 地理位置限制
    geo $allowed_country {
        default 0;
        include /etc/nginx/allowed_countries.conf;
    }

    # 安全配置
    map $http_user_agent $blocked_agent {
        default 0;
        ~*malicious 1;
        ~*bot 1;
        ~*crawler 1;
    }

    server {
        listen 80;
        server_name secure.example.com;

        # 全局限流
        limit_req zone=global burst=200 nodelay;
        limit_conn conn_limit_per_ip 10;

        # 地理位置限制
        if ($allowed_country = 0) {
            return 403;
        }

        # User-Agent限制
        if ($blocked_agent) {
            return 403;
        }

        # 登录接口限流
        location /login {
            limit_req zone=login burst=5 nodelay;
            proxy_pass http://backend;
        }

        # API接口限流
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            proxy_pass http://backend;
        }

        # 禁止访问敏感文件
        location ~ /\. {
            deny all;
        }

        location ~ \.(sql|bak|backup|log)$ {
            deny all;
        }

        # 防止SQL注入和XSS
        location ~ .*(\;|\||`|>|<|'|"|\$|\&|\*|\?|\\|\{|\}|\[|\]|\(|\)|=).* {
            return 444;
        }
    }
}
```

#### 缓存配置
```nginx
http {
    # 缓存路径配置
    proxy_cache_path /var/cache/nginx/proxy
                     levels=1:2
                     keys_zone=proxy_cache:10m
                     max_size=1g
                     inactive=60m
                     use_temp_path=off;

    # FastCGI缓存
    fastcgi_cache_path /var/cache/nginx/fastcgi
                       levels=1:2
                       keys_zone=fastcgi_cache:10m
                       max_size=1g
                       inactive=60m;

    server {
        listen 80;
        server_name cache.example.com;

        # 代理缓存
        location /api/ {
            proxy_cache proxy_cache;
            proxy_cache_key $scheme$proxy_host$request_uri;
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_lock on;

            # 缓存头
            add_header X-Cache-Status $upstream_cache_status;

            proxy_pass http://backend;
        }

        # 静态文件缓存
        location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|txt)$ {
            expires 1y;
            add_header Cache-Control "public, no-transform";
            add_header Vary Accept-Encoding;

            # 启用gzip
            gzip_static on;
        }

        # 缓存清理接口
        location ~ /purge(/.*) {
            allow 127.0.0.1;
            allow 192.168.1.0/24;
            deny all;

            proxy_cache_purge proxy_cache $scheme$proxy_host$1;
        }
    }
}
```

#### 微服务网关配置
```nginx
# 微服务路由配置
upstream user_service {
    server user-service:8080;
}

upstream order_service {
    server order-service:8080;
}

upstream product_service {
    server product-service:8080;
}

server {
    listen 80;
    server_name gateway.example.com;

    # 全局配置
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 用户服务
    location /api/users/ {
        rewrite ^/api/users/(.*) /$1 break;
        proxy_pass http://user_service;
    }

    # 订单服务
    location /api/orders/ {
        rewrite ^/api/orders/(.*) /$1 break;
        proxy_pass http://order_service;
    }

    # 产品服务
    location /api/products/ {
        rewrite ^/api/products/(.*) /$1 break;
        proxy_pass http://product_service;
    }

    # 健康检查
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # 监控指标
    location /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        allow 192.168.1.0/24;
        deny all;
    }
}
```

## ⚡ 性能特点

| 特性 | 说明 | 适用场景 |
|------|------|----------|
| 高并发 | 支持数万并发连接 | 高流量网站 |
| 低内存 | 内存占用少 | 资源受限环境 |
| 模块化 | 灵活的模块系统 | 定制化需求 |
| 热重载 | 无缝配置更新 | 生产环境 |

## 🔗 知识关联

- **前置知识**：[[Linux基础]] [[HTTP协议]]
- **相关技术**：[[负载均衡]] [[SSL/TLS]]
- **实战应用**：[[微服务网关]] [[CDN配置]]
- **问题解决**：[[Nginx问题解决]]

## 🏷️ 标签
#Nginx #Web服务器 #反向代理 #负载均衡 #高性能 #面试重点