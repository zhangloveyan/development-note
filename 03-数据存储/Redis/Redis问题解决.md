# Redis - 问题解决

---
tags: [Redis, 问题解决, 调试, 故障排查, 性能问题, 内存问题]
created: 2026-02-21
updated: 2026-02-21
status: 持续更新
importance: ⭐⭐⭐⭐
---

## 🚨 高频问题速查

### 问题1：Redis连接超时 `#连接问题`
**现象**：客户端连接Redis时出现超时错误
**原因**：网络延迟、Redis服务器负载过高、连接池配置不当
**解决**：
1. 检查网络连通性和延迟
2. 调整连接超时时间配置
3. 优化连接池参数
4. 检查Redis服务器资源使用情况

```bash
# 检查Redis连接
redis-cli -h host -p port ping

# 检查网络延迟
ping redis-server-ip

# 查看Redis连接数
redis-cli info clients
```

**相关原理**：[[Redis核心原理.md#网络模型]]

---

### 问题2：内存使用过高 `#内存问题`
**现象**：Redis内存使用率持续上升，接近或超过配置的最大内存
**原因**：大key存储、内存泄漏、过期策略配置不当、数据结构选择不合理
**解决**：
1. 分析内存使用情况
2. 清理大key和过期数据
3. 优化数据结构选择
4. 调整内存淘汰策略

```bash
# 查看内存使用情况
redis-cli info memory

# 查看大key
redis-cli --bigkeys

# 查看内存使用详情
redis-cli memory usage key_name

# 设置内存淘汰策略
redis-cli config set maxmemory-policy allkeys-lru
```

**相关原理**：[[Redis核心原理.md#内存管理]]

---

### 问题3：缓存穿透 `#缓存问题`
**现象**：大量请求查询不存在的数据，导致请求直接打到数据库
**原因**：恶意攻击、业务逻辑问题、缓存策略设计不当
**解决**：
1. 实现布隆过滤器
2. 缓存空值
3. 参数校验
4. 限流保护

```java
// 布隆过滤器解决方案
@Component
public class BloomFilterService {

    private final BloomFilter<String> bloomFilter;

    public BloomFilterService() {
        this.bloomFilter = BloomFilter.create(
            Funnels.stringFunnel(Charset.defaultCharset()),
            1000000, 0.01);
        initBloomFilter();
    }

    public boolean mightContain(String key) {
        return bloomFilter.mightContain(key);
    }
}

// 缓存空值解决方案
public Object getWithNullCache(String key) {
    Object value = redisTemplate.opsForValue().get(key);
    if (value != null) {
        return "NULL".equals(value) ? null : value;
    }

    Object dbValue = getFromDatabase(key);
    if (dbValue != null) {
        redisTemplate.opsForValue().set(key, dbValue, Duration.ofMinutes(30));
    } else {
        redisTemplate.opsForValue().set(key, "NULL", Duration.ofMinutes(5));
    }

    return dbValue;
}
```

**相关原理**：[[Redis实战应用.md#缓存问题解决方案]]

---

### 问题4：缓存雪崩 `#缓存问题`
**现象**：大量缓存同时失效，导致请求全部打到数据库
**原因**：缓存同时过期、Redis服务器宕机、重启导致缓存丢失
**解决**：
1. 设置随机过期时间
2. 实现多级缓存
3. 使用互斥锁
4. 熔断降级机制

```java
// 随机过期时间
public void setWithRandomExpire(String key, Object value) {
    int baseExpire = 30; // 30分钟
    int randomExpire = new Random().nextInt(10); // 0-10分钟随机

    redisTemplate.opsForValue().set(key, value,
        Duration.ofMinutes(baseExpire + randomExpire));
}

// 多级缓存
@Component
public class MultiLevelCache {

    private final Cache<String, Object> localCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .build();

    public Object get(String key) {
        // 1. 本地缓存
        Object value = localCache.getIfPresent(key);
        if (value != null) return value;

        // 2. Redis缓存
        value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            localCache.put(key, value);
            return value;
        }

        // 3. 数据库
        value = getFromDatabase(key);
        if (value != null) {
            localCache.put(key, value);
            setWithRandomExpire(key, value);
        }

        return value;
    }
}
```

---

### 问题5：热key问题 `#性能问题`
**现象**：某个key被大量访问，导致单个Redis节点负载过高
**原因**：热点数据访问、业务设计不合理、缓存策略问题
**解决**：
1. 本地缓存热key
2. 热key分散存储
3. 使用多副本
4. 限流保护

```java
// 热key本地缓存
@Component
public class HotKeyCache {

    private final Cache<String, Object> hotKeyCache = Caffeine.newBuilder()
        .maximumSize(100)
        .expireAfterWrite(Duration.ofMinutes(1))
        .build();

    private final Set<String> hotKeys = Set.of("hot_key_1", "hot_key_2");

    public Object get(String key) {
        if (hotKeys.contains(key)) {
            Object value = hotKeyCache.getIfPresent(key);
            if (value != null) return value;

            value = redisTemplate.opsForValue().get(key);
            if (value != null) {
                hotKeyCache.put(key, value);
            }
            return value;
        }

        return redisTemplate.opsForValue().get(key);
    }
}

// 热key分散存储
public Object getHotKeyWithSharding(String key) {
    // 随机选择一个副本
    int replica = new Random().nextInt(3);
    String shardKey = key + "_replica_" + replica;

    Object value = redisTemplate.opsForValue().get(shardKey);
    if (value == null) {
        // 如果副本没有数据，从主key获取并复制
        value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            redisTemplate.opsForValue().set(shardKey, value, Duration.ofMinutes(30));
        }
    }

    return value;
}
```

---

### 问题6：主从同步延迟 `#主从复制`
**现象**：从库数据滞后于主库，读取到旧数据
**原因**：网络延迟、从库性能不足、大事务阻塞、配置不当
**解决**：
1. 检查网络连接
2. 优化从库配置
3. 避免大事务
4. 监控同步状态

```bash
# 查看主从同步状态
redis-cli info replication

# 查看同步延迟
redis-cli --latency-history -i 1

# 从库配置优化
# redis.conf
repl-diskless-sync yes
repl-diskless-sync-delay 5
client-output-buffer-limit replica 256mb 64mb 60
```

```java
// 读写分离时的一致性处理
@Component
public class ReadWriteConsistency {

    @Autowired
    private RedisTemplate<String, Object> masterRedis;

    @Autowired
    private RedisTemplate<String, Object> slaveRedis;

    public void write(String key, Object value) {
        masterRedis.opsForValue().set(key, value);

        // 写入后短时间内从主库读取
        String flagKey = "write_flag:" + key;
        masterRedis.opsForValue().set(flagKey, "1", Duration.ofSeconds(5));
    }

    public Object read(String key) {
        String flagKey = "write_flag:" + key;

        // 检查是否刚写入
        if (masterRedis.hasKey(flagKey)) {
            return masterRedis.opsForValue().get(key);
        }

        return slaveRedis.opsForValue().get(key);
    }
}
```

---

### 问题7：Redis集群脑裂 `#集群问题`
**现象**：网络分区导致集群出现多个主节点
**原因**：网络分区、配置不当、节点故障
**解决**：
1. 配置合适的超时时间
2. 设置最小主节点数
3. 监控网络状态
4. 自动故障恢复

```bash
# 集群配置优化
# redis.conf
cluster-node-timeout 15000
cluster-require-full-coverage no

# 哨兵配置
# sentinel.conf
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

---

### 问题8：慢查询问题 `#性能问题`
**现象**：Redis响应时间变慢，影响应用性能
**原因**：复杂命令、大key操作、内存不足、网络问题
**解决**：
1. 启用慢查询日志
2. 分析慢查询命令
3. 优化数据结构
4. 避免阻塞命令

```bash
# 配置慢查询日志
redis-cli config set slowlog-log-slower-than 10000  # 10ms
redis-cli config set slowlog-max-len 128

# 查看慢查询日志
redis-cli slowlog get 10

# 重置慢查询日志
redis-cli slowlog reset
```

```java
// 避免阻塞操作
@Component
public class NonBlockingRedisOperations {

    // 使用SCAN替代KEYS
    public Set<String> scanKeys(String pattern) {
        Set<String> keys = new HashSet<>();
        ScanOptions options = ScanOptions.scanOptions()
            .match(pattern)
            .count(100)
            .build();

        Cursor<String> cursor = redisTemplate.scan(options);
        while (cursor.hasNext()) {
            keys.add(cursor.next());
        }

        return keys;
    }

    // 分批删除大集合
    public void deleteLargeSet(String key) {
        while (redisTemplate.opsForSet().size(key) > 0) {
            Set<Object> members = redisTemplate.opsForSet().pop(key, 100);
            if (members.isEmpty()) break;
        }
        redisTemplate.delete(key);
    }
}
```

---

### 问题9：持久化问题 `#持久化`
**现象**：数据丢失、持久化文件损坏、恢复失败
**原因**：磁盘空间不足、配置错误、硬件故障
**解决**：
1. 检查磁盘空间
2. 验证配置文件
3. 修复损坏文件
4. 优化持久化策略

```bash
# 检查RDB文件
redis-check-rdb dump.rdb

# 检查AOF文件
redis-check-aof appendonly.aof

# 修复AOF文件
redis-check-aof --fix appendonly.aof

# 持久化配置优化
# redis.conf
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

---

### 问题10：连接池耗尽 `#连接问题`
**现象**：获取Redis连接超时，连接池无可用连接
**原因**：连接泄漏、连接池配置过小、并发量过大
**解决**：
1. 检查连接泄漏
2. 调整连接池配置
3. 优化连接使用
4. 监控连接状态

```yaml
# 连接池配置优化
spring:
  redis:
    lettuce:
      pool:
        max-active: 50      # 增加最大连接数
        max-idle: 20        # 增加最大空闲连接数
        min-idle: 10        # 设置最小空闲连接数
        max-wait: 5000ms    # 增加等待时间
      shutdown-timeout: 100ms
```

```java
// 连接泄漏检测
@Component
public class RedisConnectionMonitor {

    @Autowired
    private LettuceConnectionFactory connectionFactory;

    @Scheduled(fixedRate = 60000) // 每分钟检查一次
    public void monitorConnections() {
        GenericObjectPool<?> pool = (GenericObjectPool<?>)
            connectionFactory.getConnection().getNativeConnection();

        log.info("Redis连接池状态 - 活跃: {}, 空闲: {}, 等待: {}",
            pool.getNumActive(),
            pool.getNumIdle(),
            pool.getNumWaiters());

        if (pool.getNumWaiters() > 0) {
            log.warn("Redis连接池有等待连接的请求: {}", pool.getNumWaiters());
        }
    }
}
```

## 🔧 调试技巧

### 性能监控命令
```bash
# 实时监控Redis命令
redis-cli monitor

# 查看Redis统计信息
redis-cli info all

# 查看客户端连接信息
redis-cli client list

# 查看内存使用详情
redis-cli info memory

# 查看慢查询日志
redis-cli slowlog get 10

# 查看大key
redis-cli --bigkeys

# 延迟监控
redis-cli --latency
redis-cli --latency-history
redis-cli --latency-dist
```

### 内存分析工具
```bash
# 分析内存使用
redis-cli --memkeys

# 查看key的内存使用
redis-cli memory usage key_name

# 内存使用报告
redis-cli --memkeys-samples 1000
```

### 集群诊断命令
```bash
# 检查集群状态
redis-cli --cluster check 127.0.0.1:7000

# 查看集群信息
redis-cli cluster info
redis-cli cluster nodes

# 修复集群
redis-cli --cluster fix 127.0.0.1:7000

# 重新分片
redis-cli --cluster reshard 127.0.0.1:7000
```

## 🛠️ 故障排查工具

### 日志分析
```bash
# Redis日志位置
tail -f /var/log/redis/redis-server.log

# 分析错误日志
grep -i error /var/log/redis/redis-server.log

# 分析连接日志
grep -i "connection" /var/log/redis/redis-server.log
```

### 系统资源监控
```bash
# 查看Redis进程资源使用
top -p $(pgrep redis-server)

# 查看内存使用
free -h

# 查看磁盘IO
iostat -x 1

# 查看网络连接
netstat -an | grep :6379
```

### 性能测试工具
```bash
# Redis基准测试
redis-benchmark -h localhost -p 6379 -c 100 -n 10000

# 指定命令测试
redis-benchmark -h localhost -p 6379 -t set,get -n 10000 -q

# 管道测试
redis-benchmark -h localhost -p 6379 -n 10000 -P 16
```

## 📋 故障预防措施

### 监控告警设置
```yaml
# Prometheus监控配置示例
- alert: RedisDown
  expr: redis_up == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Redis实例宕机"

- alert: RedisMemoryHigh
  expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Redis内存使用率过高"

- alert: RedisSlowQueries
  expr: increase(redis_slowlog_length[5m]) > 10
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "Redis慢查询增多"
```

### 定期维护任务
```bash
#!/bin/bash
# Redis维护脚本

# 1. 检查内存使用
MEMORY_USAGE=$(redis-cli info memory | grep used_memory_human | cut -d: -f2)
echo "当前内存使用: $MEMORY_USAGE"

# 2. 检查慢查询
SLOW_QUERIES=$(redis-cli slowlog len)
if [ $SLOW_QUERIES -gt 10 ]; then
    echo "警告: 慢查询数量过多 ($SLOW_QUERIES)"
    redis-cli slowlog get 5
fi

# 3. 检查大key
redis-cli --bigkeys

# 4. 清理过期key
redis-cli --scan --pattern "*" | xargs -I {} redis-cli ttl {} | grep -c "^-1$"

# 5. 备份数据
redis-cli bgsave
```

### 容量规划
```java
@Component
public class RedisCapacityPlanning {

    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
    public void analyzeCapacity() {
        // 分析内存使用趋势
        long usedMemory = getUsedMemory();
        long maxMemory = getMaxMemory();
        double usageRatio = (double) usedMemory / maxMemory;

        if (usageRatio > 0.8) {
            log.warn("Redis内存使用率过高: {}%", usageRatio * 100);
            // 发送告警
            sendAlert("Redis内存使用率过高", usageRatio);
        }

        // 分析key数量增长
        long keyCount = getKeyCount();
        recordMetric("redis.key.count", keyCount);

        // 分析连接数使用
        int connectedClients = getConnectedClients();
        recordMetric("redis.clients.connected", connectedClients);
    }
}
```

## 🔗 相关文档
- **核心原理**：[[Redis核心原理.md]]
- **实战应用**：[[Redis实战应用.md]]
- **相关技术**：[[../MySQL/MySQL问题解决.md]]

## 🏷️ 标签
#Redis #问题解决 #调试 #故障排查 #性能问题 #内存问题 #缓存问题 #集群问题 #持久化