# Nginx - 问题解决

---
tags: [问题解决, Nginx, Web服务器, 性能优化, 故障排查]
created: 2026-02-21
updated: 2026-02-21
status: 持续更新
importance: ⭐⭐⭐⭐
---

## 🚨 高频问题速查

### 问题1：502 Bad Gateway错误 `#502错误`
**现象**：客户端收到502错误响应
**原因**：后端服务不可用、连接超时、配置错误
**解决**：
1. 检查后端服务状态
2. 验证代理配置
3. 调整超时参数
4. 检查防火墙设置

```bash
# 检查后端服务
curl -I http://backend-server:8080/health
telnet backend-server 8080

# 检查Nginx配置
nginx -t
nginx -T | grep proxy_pass

# 查看错误日志
tail -f /var/log/nginx/error.log

# 测试配置
location /test {
    proxy_pass http://backend;
    proxy_connect_timeout 5s;
    proxy_send_timeout 5s;
    proxy_read_timeout 5s;

    # 调试信息
    add_header X-Debug-Backend $upstream_addr;
    add_header X-Debug-Status $upstream_status;
}
```

**相关原理**：[[Nginx配置实战#反向代理配置]]

---

### 问题2：高并发下性能下降 `#性能问题`
**现象**：高并发时响应变慢，连接被拒绝
**原因**：worker进程数不足、连接数限制、缓冲区设置不当
**解决**：
1. 调整worker进程数
2. 增加连接数限制
3. 优化缓冲区配置
4. 启用keepalive

```nginx
# 性能优化配置
worker_processes auto;  # 自动设置为CPU核心数
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;  # 增加连接数
    use epoll;
    multi_accept on;
}

http {
    # 连接优化
    keepalive_timeout 65;
    keepalive_requests 1000;

    # 缓冲区优化
    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 4k;

    # 代理缓冲优化
    proxy_buffering on;
    proxy_buffer_size 4k;
    proxy_buffers 8 4k;
    proxy_busy_buffers_size 8k;

    # 文件传输优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # 上游连接池
    upstream backend {
        server 192.168.1.10:8080;
        keepalive 32;  # 保持连接池
    }
}
```

---

### 问题3：SSL证书问题 `#SSL问题`
**现象**：HTTPS访问失败，证书错误
**原因**：证书过期、配置错误、证书链不完整
**解决**：
1. 检查证书有效期
2. 验证证书配置
3. 完善证书链
4. 测试SSL配置

```bash
# 检查证书信息
openssl x509 -in /etc/ssl/certs/example.com.crt -text -noout
openssl x509 -in /etc/ssl/certs/example.com.crt -dates -noout

# 检查证书链
openssl verify -CAfile /etc/ssl/certs/ca-bundle.crt /etc/ssl/certs/example.com.crt

# 测试SSL连接
openssl s_client -connect example.com:443 -servername example.com

# SSL配置测试
curl -I https://example.com
curl -k -I https://example.com  # 忽略证书验证
```

```nginx
# SSL配置优化
server {
    listen 443 ssl http2;
    server_name example.com;

    # 证书配置
    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # 中间证书
    ssl_trusted_certificate /etc/ssl/certs/ca-bundle.crt;

    # SSL优化
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
}
```

---

### 问题4：负载均衡不均匀 `#负载均衡`
**现象**：后端服务器负载分布不均
**原因**：算法选择不当、权重配置错误、会话粘性问题
**解决**：
1. 选择合适的负载均衡算法
2. 调整服务器权重
3. 配置健康检查
4. 处理会话粘性

```nginx
# 负载均衡配置
upstream backend {
    # 负载均衡算法
    least_conn;  # 最少连接数
    # ip_hash;   # IP哈希（会话粘性）
    # hash $request_uri;  # URL哈希

    # 服务器配置
    server 192.168.1.10:8080 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 weight=2 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 weight=1 max_fails=3 fail_timeout=30s backup;

    # 健康检查
    keepalive 32;
}

# 监控负载均衡状态
location /upstream_status {
    # 需要nginx-module-vts模块
    vhost_traffic_status_display;
    vhost_traffic_status_display_format html;
    allow 127.0.0.1;
    deny all;
}
```

---

### 问题5：缓存命中率低 `#缓存问题`
**现象**：缓存效果不佳，命中率低
**原因**：缓存键设置不当、缓存时间过短、缓存大小不足
**解决**：
1. 优化缓存键
2. 调整缓存时间
3. 增加缓存空间
4. 监控缓存状态

```nginx
http {
    # 缓存配置优化
    proxy_cache_path /var/cache/nginx/proxy
                     levels=1:2
                     keys_zone=proxy_cache:100m
                     max_size=10g
                     inactive=60m
                     use_temp_path=off;

    # 缓存键优化
    proxy_cache_key "$scheme$request_method$host$request_uri$is_args$args";

    server {
        location /api/ {
            proxy_cache proxy_cache;

            # 缓存时间设置
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 301 1h;
            proxy_cache_valid 404 1m;
            proxy_cache_valid any 5m;

            # 缓存控制
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
            proxy_cache_lock on;
            proxy_cache_lock_timeout 5s;

            # 缓存头信息
            add_header X-Cache-Status $upstream_cache_status;
            add_header X-Cache-Key $proxy_cache_key;

            proxy_pass http://backend;
        }

        # 缓存统计
        location /cache_status {
            proxy_cache_purge proxy_cache "$scheme$request_method$host$request_uri";
            allow 127.0.0.1;
            deny all;
        }
    }
}
```

## 🔧 调试技巧

### 常用调试方法

#### 1. 日志分析
```bash
# 实时查看错误日志
tail -f /var/log/nginx/error.log

# 分析访问日志
tail -f /var/log/nginx/access.log | grep "POST"

# 统计状态码
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr

# 统计IP访问量
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10

# 分析响应时间
awk '{print $NF}' /var/log/nginx/access.log | sort -n | tail -10
```

#### 2. 配置测试
```bash
# 测试配置文件语法
nginx -t

# 显示完整配置
nginx -T

# 重新加载配置
nginx -s reload

# 检查配置文件包含关系
nginx -T | grep -E "include|server_name"

# 测试特定配置
nginx -t -c /etc/nginx/test.conf
```

#### 3. 性能监控
```bash
# 查看Nginx状态
curl http://localhost/nginx_status

# 监控连接数
ss -tuln | grep :80
netstat -an | grep :80 | wc -l

# 监控进程状态
ps aux | grep nginx
top -p $(pgrep nginx | tr '\n' ',' | sed 's/,$//')

# 监控文件描述符
lsof -p $(pgrep nginx)
cat /proc/$(pgrep nginx | head -1)/limits
```

### 性能分析工具

#### 1. 日志分析脚本
```bash
#!/bin/bash
# nginx_log_analyzer.sh

LOG_FILE=${1:-/var/log/nginx/access.log}

echo "=== Nginx日志分析报告 ==="
echo "日志文件: $LOG_FILE"
echo "分析时间: $(date)"
echo ""

echo "=== 状态码统计 ==="
awk '{print $9}' $LOG_FILE | sort | uniq -c | sort -nr

echo ""
echo "=== 访问量TOP 10 IP ==="
awk '{print $1}' $LOG_FILE | sort | uniq -c | sort -nr | head -10

echo ""
echo "=== 访问量TOP 10 URL ==="
awk '{print $7}' $LOG_FILE | sort | uniq -c | sort -nr | head -10

echo ""
echo "=== User-Agent统计 ==="
awk -F'"' '{print $6}' $LOG_FILE | sort | uniq -c | sort -nr | head -10

echo ""
echo "=== 响应时间分析 ==="
awk '{print $NF}' $LOG_FILE | awk '{
    if ($1 < 0.1) fast++
    else if ($1 < 0.5) normal++
    else if ($1 < 1.0) slow++
    else very_slow++
    total++
}
END {
    printf "快速(<0.1s): %d (%.2f%%)\n", fast, fast/total*100
    printf "正常(0.1-0.5s): %d (%.2f%%)\n", normal, normal/total*100
    printf "较慢(0.5-1.0s): %d (%.2f%%)\n", slow, slow/total*100
    printf "很慢(>1.0s): %d (%.2f%%)\n", very_slow, very_slow/total*100
}'
```

#### 2. 性能监控脚本
```bash
#!/bin/bash
# nginx_monitor.sh

while true; do
    echo "=== $(date) ==="

    # Nginx进程状态
    echo "Nginx进程数: $(pgrep nginx | wc -l)"

    # 连接统计
    CONNECTIONS=$(ss -tuln | grep :80 | wc -l)
    echo "当前连接数: $CONNECTIONS"

    # 内存使用
    MEMORY=$(ps aux | grep nginx | awk '{sum+=$6} END {print sum/1024 "MB"}')
    echo "内存使用: $MEMORY"

    # CPU使用
    CPU=$(ps aux | grep nginx | awk '{sum+=$3} END {print sum "%"}')
    echo "CPU使用: $CPU"

    # 状态统计
    if curl -s http://localhost/nginx_status > /dev/null; then
        curl -s http://localhost/nginx_status
    fi

    echo "========================"
    sleep 30
done
```

#### 3. 自动化健康检查
```bash
#!/bin/bash
# nginx_health_check.sh

NGINX_URL="http://localhost"
EMAIL="admin@example.com"
LOG_FILE="/var/log/nginx_health.log"

check_nginx() {
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # 检查Nginx进程
    if ! pgrep nginx > /dev/null; then
        echo "[$timestamp] ERROR: Nginx进程未运行" >> $LOG_FILE
        systemctl start nginx
        return 1
    fi

    # 检查HTTP响应
    local http_code=$(curl -s -o /dev/null -w "%{http_code}" $NGINX_URL)
    if [ "$http_code" != "200" ]; then
        echo "[$timestamp] ERROR: HTTP响应异常 ($http_code)" >> $LOG_FILE
        return 1
    fi

    # 检查响应时间
    local response_time=$(curl -s -o /dev/null -w "%{time_total}" $NGINX_URL)
    if (( $(echo "$response_time > 5.0" | bc -l) )); then
        echo "[$timestamp] WARNING: 响应时间过长 ($response_time s)" >> $LOG_FILE
    fi

    # 检查错误日志
    local error_count=$(tail -100 /var/log/nginx/error.log | grep "$(date '+%Y/%m/%d %H:%M')" | wc -l)
    if [ "$error_count" -gt 10 ]; then
        echo "[$timestamp] WARNING: 错误日志过多 ($error_count)" >> $LOG_FILE
    fi

    echo "[$timestamp] INFO: 健康检查通过" >> $LOG_FILE
    return 0
}

# 执行检查
if ! check_nginx; then
    # 发送告警邮件
    echo "Nginx健康检查失败，请及时处理" | mail -s "Nginx告警" $EMAIL
fi
```

## 🔗 相关文档

- **技术原理**：[[Nginx配置实战]]
- **实战应用**：[[负载均衡实战]] [[Web性能优化]]

## 🏷️ 标签
#问题解决 #Nginx #Web服务器 #性能优化 #故障排查