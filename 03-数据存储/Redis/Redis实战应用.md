# Redis - 实战应用

---
tags: [Redis, 实战应用, 缓存策略, 分布式锁, 集群部署, 高可用]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Redis在实际项目中的应用场景和最佳实践

- **缓存应用**：缓存策略、缓存一致性、缓存穿透/击穿/雪崩解决方案
- **分布式锁**：基于Redis实现分布式锁的多种方案
- **集群部署**：主从复制、哨兵模式、集群模式的部署和配置
- **高级应用**：消息队列、限流、统计分析等场景应用
- **性能优化**：连接池、批量操作、内存优化等实践

## 🚀 缓存应用实战

### 1. 缓存策略模式

#### Cache-Aside模式(旁路缓存)
```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // 读取数据
    public User getUser(Long userId) {
        String key = "user:" + userId;

        // 1. 先查缓存
        User user = (User) redisTemplate.opsForValue().get(key);
        if (user != null) {
            return user;
        }

        // 2. 缓存未命中，查数据库
        user = userRepository.findById(userId);
        if (user != null) {
            // 3. 写入缓存
            redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
        }

        return user;
    }

    // 更新数据
    public void updateUser(User user) {
        // 1. 先更新数据库
        userRepository.save(user);

        // 2. 删除缓存
        String key = "user:" + user.getId();
        redisTemplate.delete(key);
    }
}
```

#### Write-Through模式(写穿透)
```java
@Service
public class CacheWriteThroughService {

    public void updateUser(User user) {
        // 同时更新缓存和数据库
        userRepository.save(user);

        String key = "user:" + user.getId();
        redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
    }
}
```

#### Write-Behind模式(写回)
```java
@Service
public class CacheWriteBehindService {

    private final Queue<User> writeQueue = new ConcurrentLinkedQueue<>();

    public void updateUser(User user) {
        // 1. 立即更新缓存
        String key = "user:" + user.getId();
        redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));

        // 2. 异步更新数据库
        writeQueue.offer(user);
    }

    @Scheduled(fixedDelay = 5000)
    public void flushToDatabase() {
        User user;
        while ((user = writeQueue.poll()) != null) {
            userRepository.save(user);
        }
    }
}
```

### 2. 缓存一致性解决方案

#### 双检加锁机制
```java
public User getUserWithDoubleCheck(Long userId) {
    String key = "user:" + userId;
    String lockKey = "lock:user:" + userId;

    // 第一次检查缓存
    User user = (User) redisTemplate.opsForValue().get(key);
    if (user != null) {
        return user;
    }

    // 获取分布式锁
    Boolean lockAcquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", Duration.ofSeconds(10));

    if (lockAcquired) {
        try {
            // 第二次检查缓存
            user = (User) redisTemplate.opsForValue().get(key);
            if (user != null) {
                return user;
            }

            // 查询数据库并缓存
            user = userRepository.findById(userId);
            if (user != null) {
                redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
            }

            return user;
        } finally {
            redisTemplate.delete(lockKey);
        }
    } else {
        // 等待其他线程完成
        Thread.sleep(100);
        return getUserWithDoubleCheck(userId);
    }
}
```

#### 延时双删策略
```java
public void updateUserWithDelayedDoubleDelete(User user) {
    String key = "user:" + user.getId();

    // 1. 删除缓存
    redisTemplate.delete(key);

    // 2. 更新数据库
    userRepository.save(user);

    // 3. 延时删除缓存
    CompletableFuture.runAsync(() -> {
        try {
            Thread.sleep(1000); // 延时1秒
            redisTemplate.delete(key);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
}
```

### 3. 缓存问题解决方案

#### 缓存穿透解决方案
```java
// 1. 布隆过滤器
@Component
public class BloomFilterService {

    private final BloomFilter<String> bloomFilter;

    public BloomFilterService() {
        // 预期插入100万数据，误判率0.01%
        this.bloomFilter = BloomFilter.create(
            Funnels.stringFunnel(Charset.defaultCharset()),
            1000000,
            0.0001
        );

        // 初始化布隆过滤器
        initBloomFilter();
    }

    public boolean mightContain(String key) {
        return bloomFilter.mightContain(key);
    }

    private void initBloomFilter() {
        // 将所有有效的key加入布隆过滤器
        List<String> allKeys = userRepository.findAllUserIds();
        allKeys.forEach(bloomFilter::put);
    }
}

// 2. 缓存空值
public User getUserWithNullCache(Long userId) {
    String key = "user:" + userId;

    Object cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        return cached.equals("NULL") ? null : (User) cached;
    }

    User user = userRepository.findById(userId);
    if (user != null) {
        redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
    } else {
        // 缓存空值，设置较短过期时间
        redisTemplate.opsForValue().set(key, "NULL", Duration.ofMinutes(5));
    }

    return user;
}
```

#### 缓存击穿解决方案
```java
// 互斥锁方案
public User getUserWithMutex(Long userId) {
    String key = "user:" + userId;
    String lockKey = "mutex:user:" + userId;

    User user = (User) redisTemplate.opsForValue().get(key);
    if (user != null) {
        return user;
    }

    // 尝试获取互斥锁
    String lockValue = UUID.randomUUID().toString();
    Boolean lockAcquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(10));

    if (lockAcquired) {
        try {
            // 再次检查缓存
            user = (User) redisTemplate.opsForValue().get(key);
            if (user != null) {
                return user;
            }

            // 查询数据库
            user = userRepository.findById(userId);
            if (user != null) {
                // 设置随机过期时间，避免同时过期
                int randomExpire = 30 + new Random().nextInt(10);
                redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(randomExpire));
            }

            return user;
        } finally {
            // 释放锁
            releaseLock(lockKey, lockValue);
        }
    } else {
        // 等待并重试
        Thread.sleep(50);
        return getUserWithMutex(userId);
    }
}

// 逻辑过期方案
public User getUserWithLogicalExpire(Long userId) {
    String key = "user:" + userId;

    CacheData cacheData = (CacheData) redisTemplate.opsForValue().get(key);
    if (cacheData == null) {
        return null;
    }

    // 检查逻辑过期时间
    if (cacheData.getExpireTime().isAfter(LocalDateTime.now())) {
        return cacheData.getUser();
    }

    // 逻辑过期，异步更新
    String lockKey = "rebuild:user:" + userId;
    Boolean lockAcquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", Duration.ofSeconds(10));

    if (lockAcquired) {
        CompletableFuture.runAsync(() -> {
            try {
                User user = userRepository.findById(userId);
                if (user != null) {
                    CacheData newCacheData = new CacheData(user, LocalDateTime.now().plusMinutes(30));
                    redisTemplate.opsForValue().set(key, newCacheData);
                }
            } finally {
                redisTemplate.delete(lockKey);
            }
        });
    }

    // 返回过期数据
    return cacheData.getUser();
}
```

#### 缓存雪崩解决方案
```java
// 1. 随机过期时间
public void setCacheWithRandomExpire(String key, Object value) {
    int baseExpire = 30; // 基础过期时间30分钟
    int randomExpire = new Random().nextInt(10); // 随机0-10分钟

    redisTemplate.opsForValue().set(key, value,
        Duration.ofMinutes(baseExpire + randomExpire));
}

// 2. 多级缓存
@Service
public class MultiLevelCacheService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    private final Cache<String, Object> localCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(Duration.ofMinutes(5))
        .build();

    public User getUser(Long userId) {
        String key = "user:" + userId;

        // 1. 本地缓存
        User user = (User) localCache.getIfPresent(key);
        if (user != null) {
            return user;
        }

        // 2. Redis缓存
        user = (User) redisTemplate.opsForValue().get(key);
        if (user != null) {
            localCache.put(key, user);
            return user;
        }

        // 3. 数据库
        user = userRepository.findById(userId);
        if (user != null) {
            localCache.put(key, user);
            setCacheWithRandomExpire(key, user);
        }

        return user;
    }
}
```

## 🔒 分布式锁实战

### 1. 基础分布式锁
```java
@Component
public class RedisDistributedLock {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final String LOCK_PREFIX = "lock:";
    private static final int DEFAULT_EXPIRE_TIME = 30; // 秒

    public boolean tryLock(String key, String value, int expireTime) {
        String lockKey = LOCK_PREFIX + key;
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, value, Duration.ofSeconds(expireTime));
        return Boolean.TRUE.equals(result);
    }

    public boolean releaseLock(String key, String value) {
        String lockKey = LOCK_PREFIX + key;

        // Lua脚本保证原子性
        String luaScript =
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";

        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(luaScript);
        redisScript.setResultType(Long.class);

        Long result = redisTemplate.execute(redisScript,
            Collections.singletonList(lockKey), value);

        return result != null && result == 1L;
    }
}
```

### 2. 可重入分布式锁
```java
@Component
public class ReentrantRedisLock {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final String LOCK_PREFIX = "reentrant_lock:";
    private final ThreadLocal<Map<String, Integer>> lockCount = new ThreadLocal<>();

    public boolean lock(String key, int expireTime) {
        String lockKey = LOCK_PREFIX + key;
        String threadId = Thread.currentThread().getId() + "";

        Map<String, Integer> counts = lockCount.get();
        if (counts == null) {
            counts = new HashMap<>();
            lockCount.set(counts);
        }

        Integer count = counts.get(lockKey);
        if (count != null && count > 0) {
            // 重入
            counts.put(lockKey, count + 1);
            return true;
        }

        // 尝试获取锁
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, threadId, Duration.ofSeconds(expireTime));

        if (Boolean.TRUE.equals(acquired)) {
            counts.put(lockKey, 1);
            return true;
        }

        return false;
    }

    public void unlock(String key) {
        String lockKey = LOCK_PREFIX + key;
        String threadId = Thread.currentThread().getId() + "";

        Map<String, Integer> counts = lockCount.get();
        if (counts == null) {
            return;
        }

        Integer count = counts.get(lockKey);
        if (count == null || count <= 0) {
            return;
        }

        if (count == 1) {
            // 释放锁
            String luaScript =
                "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                "    return redis.call('del', KEYS[1]) " +
                "else " +
                "    return 0 " +
                "end";

            DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
            redisScript.setScriptText(luaScript);
            redisScript.setResultType(Long.class);

            redisTemplate.execute(redisScript,
                Collections.singletonList(lockKey), threadId);

            counts.remove(lockKey);
        } else {
            counts.put(lockKey, count - 1);
        }
    }
}
```

### 3. Redisson分布式锁
```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
            .setAddress("redis://localhost:6379")
            .setPassword("password")
            .setConnectionPoolSize(10)
            .setConnectionMinimumIdleSize(2);

        return Redisson.create(config);
    }
}

@Service
public class RedissonLockService {

    @Autowired
    private RedissonClient redissonClient;

    public void executeWithLock(String lockKey, Runnable task) {
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // 尝试获取锁，最多等待10秒，锁自动释放时间30秒
            boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
            if (acquired) {
                task.run();
            } else {
                throw new RuntimeException("获取锁失败");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("获取锁被中断", e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

## 🏗️ 集群部署实战

### 1. 主从复制配置

#### 主库配置(redis-master.conf)
```bash
# 基础配置
port 6379
bind 0.0.0.0
protected-mode no

# 持久化配置
save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec

# 主从复制配置
repl-diskless-sync yes
repl-diskless-sync-delay 5
```

#### 从库配置(redis-slave.conf)
```bash
# 基础配置
port 6380
bind 0.0.0.0
protected-mode no

# 主从复制配置
replicaof 192.168.1.100 6379
masterauth password

# 只读配置
replica-read-only yes
replica-serve-stale-data yes
```

#### Docker Compose部署
```yaml
version: '3.8'

services:
  redis-master:
    image: redis:7-alpine
    container_name: redis-master
    ports:
      - "6379:6379"
    volumes:
      - ./redis-master.conf:/usr/local/etc/redis/redis.conf
      - redis-master-data:/data
    command: redis-server /usr/local/etc/redis/redis.conf

  redis-slave1:
    image: redis:7-alpine
    container_name: redis-slave1
    ports:
      - "6380:6379"
    volumes:
      - ./redis-slave.conf:/usr/local/etc/redis/redis.conf
      - redis-slave1-data:/data
    command: redis-server /usr/local/etc/redis/redis.conf
    depends_on:
      - redis-master

  redis-slave2:
    image: redis:7-alpine
    container_name: redis-slave2
    ports:
      - "6381:6379"
    volumes:
      - ./redis-slave.conf:/usr/local/etc/redis/redis.conf
      - redis-slave2-data:/data
    command: redis-server /usr/local/etc/redis/redis.conf
    depends_on:
      - redis-master

volumes:
  redis-master-data:
  redis-slave1-data:
  redis-slave2-data:
```

### 2. 哨兵模式配置

#### 哨兵配置(sentinel.conf)
```bash
# 哨兵端口
port 26379

# 监控主节点
sentinel monitor mymaster 192.168.1.100 6379 2

# 主节点密码
sentinel auth-pass mymaster password

# 主观下线时间(毫秒)
sentinel down-after-milliseconds mymaster 5000

# 故障转移超时时间
sentinel failover-timeout mymaster 10000

# 并行同步的从节点数量
sentinel parallel-syncs mymaster 1

# 日志配置
logfile "/var/log/sentinel.log"
```

#### 哨兵集群部署
```yaml
version: '3.8'

services:
  sentinel1:
    image: redis:7-alpine
    container_name: sentinel1
    ports:
      - "26379:26379"
    volumes:
      - ./sentinel1.conf:/usr/local/etc/redis/sentinel.conf
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf

  sentinel2:
    image: redis:7-alpine
    container_name: sentinel2
    ports:
      - "26380:26379"
    volumes:
      - ./sentinel2.conf:/usr/local/etc/redis/sentinel.conf
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf

  sentinel3:
    image: redis:7-alpine
    container_name: sentinel3
    ports:
      - "26381:26379"
    volumes:
      - ./sentinel3.conf:/usr/local/etc/redis/sentinel.conf
    command: redis-sentinel /usr/local/etc/redis/sentinel.conf
```

### 3. 集群模式配置

#### 集群节点配置(redis-cluster.conf)
```bash
# 基础配置
port 7000
bind 0.0.0.0
protected-mode no

# 集群配置
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 5000

# 持久化配置
appendonly yes
appendfsync everysec
```

#### 集群创建脚本
```bash
#!/bin/bash

# 创建集群
redis-cli --cluster create \
  192.168.1.100:7000 \
  192.168.1.100:7001 \
  192.168.1.100:7002 \
  192.168.1.101:7000 \
  192.168.1.101:7001 \
  192.168.1.101:7002 \
  --cluster-replicas 1

# 检查集群状态
redis-cli --cluster check 192.168.1.100:7000
```

#### Spring Boot集群配置
```yaml
spring:
  redis:
    cluster:
      nodes:
        - 192.168.1.100:7000
        - 192.168.1.100:7001
        - 192.168.1.100:7002
        - 192.168.1.101:7000
        - 192.168.1.101:7001
        - 192.168.1.101:7002
      max-redirects: 3
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 2000ms
```

## 📊 高级应用场景

### 1. 消息队列实现

#### 基于List的简单队列
```java
@Component
public class RedisMessageQueue {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 生产者
    public void sendMessage(String queue, String message) {
        redisTemplate.opsForList().leftPush(queue, message);
    }

    // 消费者(阻塞)
    public String receiveMessage(String queue, int timeout) {
        List<String> messages = redisTemplate.opsForList()
            .rightPop(queue, Duration.ofSeconds(timeout));
        return messages != null && !messages.isEmpty() ? messages.get(0) : null;
    }

    // 批量消费
    public List<String> receiveMessages(String queue, int count) {
        List<String> messages = new ArrayList<>();
        for (int i = 0; i < count; i++) {
            String message = redisTemplate.opsForList().rightPop(queue);
            if (message == null) {
                break;
            }
            messages.add(message);
        }
        return messages;
    }
}
```

#### 基于Stream的消息队列
```java
@Component
public class RedisStreamQueue {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 发送消息
    public String sendMessage(String stream, Map<String, String> message) {
        RecordId recordId = redisTemplate.opsForStream()
            .add(stream, message);
        return recordId.getValue();
    }

    // 消费消息
    public List<MapRecord<String, Object, Object>> readMessages(
            String stream, String consumerGroup, String consumer) {

        try {
            // 创建消费者组
            redisTemplate.opsForStream()
                .createGroup(stream, consumerGroup, ReadOffset.from("0"));
        } catch (Exception e) {
            // 消费者组已存在
        }

        // 读取消息
        List<MapRecord<String, Object, Object>> messages =
            redisTemplate.opsForStream().read(
                Consumer.from(consumerGroup, consumer),
                StreamReadOptions.empty().count(10).block(Duration.ofSeconds(5)),
                StreamOffset.create(stream, ReadOffset.lastConsumed())
            );

        return messages;
    }

    // 确认消息
    public void ackMessage(String stream, String consumerGroup, String... messageIds) {
        redisTemplate.opsForStream().acknowledge(stream, consumerGroup, messageIds);
    }
}
```

### 2. 限流器实现

#### 滑动窗口限流
```java
@Component
public class SlidingWindowRateLimiter {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public boolean isAllowed(String key, int limit, int windowSize) {
        long now = System.currentTimeMillis();
        long windowStart = now - windowSize * 1000L;

        String luaScript =
            "local key = KEYS[1] " +
            "local window_start = ARGV[1] " +
            "local now = ARGV[2] " +
            "local limit = tonumber(ARGV[3]) " +

            // 删除窗口外的记录
            "redis.call('zremrangebyscore', key, 0, window_start) " +

            // 获取当前窗口内的请求数
            "local current = redis.call('zcard', key) " +

            "if current < limit then " +
            "    redis.call('zadd', key, now, now) " +
            "    redis.call('expire', key, 60) " +
            "    return 1 " +
            "else " +
            "    return 0 " +
            "end";

        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(luaScript);
        redisScript.setResultType(Long.class);

        Long result = redisTemplate.execute(redisScript,
            Collections.singletonList(key),
            String.valueOf(windowStart),
            String.valueOf(now),
            String.valueOf(limit)
        );

        return result != null && result == 1L;
    }
}
```

#### 令牌桶限流
```java
@Component
public class TokenBucketRateLimiter {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    public boolean tryAcquire(String key, int capacity, int refillRate, int tokens) {
        String luaScript =
            "local key = KEYS[1] " +
            "local capacity = tonumber(ARGV[1]) " +
            "local refill_rate = tonumber(ARGV[2]) " +
            "local requested_tokens = tonumber(ARGV[3]) " +
            "local now = tonumber(ARGV[4]) " +

            "local bucket = redis.call('hmget', key, 'tokens', 'last_refill') " +
            "local tokens = tonumber(bucket[1]) or capacity " +
            "local last_refill = tonumber(bucket[2]) or now " +

            // 计算需要添加的令牌数
            "local elapsed = math.max(0, now - last_refill) " +
            "local tokens_to_add = math.floor(elapsed * refill_rate / 1000) " +
            "tokens = math.min(capacity, tokens + tokens_to_add) " +

            "if tokens >= requested_tokens then " +
            "    tokens = tokens - requested_tokens " +
            "    redis.call('hmset', key, 'tokens', tokens, 'last_refill', now) " +
            "    redis.call('expire', key, 3600) " +
            "    return 1 " +
            "else " +
            "    redis.call('hmset', key, 'tokens', tokens, 'last_refill', now) " +
            "    redis.call('expire', key, 3600) " +
            "    return 0 " +
            "end";

        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(luaScript);
        redisScript.setResultType(Long.class);

        Long result = redisTemplate.execute(redisScript,
            Collections.singletonList(key),
            String.valueOf(capacity),
            String.valueOf(refillRate),
            String.valueOf(tokens),
            String.valueOf(System.currentTimeMillis())
        );

        return result != null && result == 1L;
    }
}
```

### 3. 统计分析应用

#### HyperLogLog基数统计
```java
@Component
public class RedisStatisticsService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 添加用户访问记录
    public void addUserVisit(String date, String userId) {
        String key = "uv:" + date;
        redisTemplate.opsForHyperLogLog().add(key, userId);
        redisTemplate.expire(key, Duration.ofDays(7));
    }

    // 获取独立访客数
    public long getUniqueVisitors(String date) {
        String key = "uv:" + date;
        return redisTemplate.opsForHyperLogLog().size(key);
    }

    // 获取多天的独立访客数
    public long getUniqueVisitors(String... dates) {
        String[] keys = Arrays.stream(dates)
            .map(date -> "uv:" + date)
            .toArray(String[]::new);

        return redisTemplate.opsForHyperLogLog().size(keys);
    }
}
```

#### Bitmap用户行为分析
```java
@Component
public class UserBehaviorAnalysis {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 记录用户签到
    public void userSignIn(String userId, LocalDate date) {
        String key = "signin:" + userId + ":" + date.getYear();
        int dayOfYear = date.getDayOfYear() - 1; // 从0开始

        redisTemplate.opsForValue().setBit(key, dayOfYear, true);
        redisTemplate.expire(key, Duration.ofDays(366));
    }

    // 检查用户是否签到
    public boolean hasSignedIn(String userId, LocalDate date) {
        String key = "signin:" + userId + ":" + date.getYear();
        int dayOfYear = date.getDayOfYear() - 1;

        return Boolean.TRUE.equals(
            redisTemplate.opsForValue().getBit(key, dayOfYear)
        );
    }

    // 统计用户签到天数
    public long getSignInDays(String userId, int year) {
        String key = "signin:" + userId + ":" + year;

        String luaScript =
            "local key = KEYS[1] " +
            "local bits = redis.call('get', key) " +
            "if not bits then return 0 end " +

            "local count = 0 " +
            "for i = 1, #bits do " +
            "    local byte = string.byte(bits, i) " +
            "    while byte > 0 do " +
            "        if byte % 2 == 1 then " +
            "            count = count + 1 " +
            "        end " +
            "        byte = math.floor(byte / 2) " +
            "    end " +
            "end " +
            "return count";

        DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(luaScript);
        redisScript.setResultType(Long.class);

        Long result = redisTemplate.execute(redisScript,
            Collections.singletonList(key));

        return result != null ? result : 0L;
    }
}
```

## ⚡ 性能优化实践

### 1. 连接池优化

#### Lettuce连接池配置
```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: password
    database: 0
    timeout: 2000ms

    lettuce:
      pool:
        max-active: 20      # 最大连接数
        max-idle: 10        # 最大空闲连接数
        min-idle: 5         # 最小空闲连接数
        max-wait: 2000ms    # 最大等待时间
      shutdown-timeout: 100ms
```

#### Jedis连接池配置
```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: password
    database: 0
    timeout: 2000ms

    jedis:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 2000ms
```

### 2. 批量操作优化

#### Pipeline批量操作
```java
@Component
public class RedisBatchOperations {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 批量设置
    public void batchSet(Map<String, String> keyValues) {
        redisTemplate.executePipelined(new RedisCallback<Object>() {
            @Override
            public Object doInRedis(RedisConnection connection) throws DataAccessException {
                for (Map.Entry<String, String> entry : keyValues.entrySet()) {
                    connection.set(
                        entry.getKey().getBytes(),
                        entry.getValue().getBytes()
                    );
                }
                return null;
            }
        });
    }

    // 批量获取
    public List<String> batchGet(List<String> keys) {
        List<Object> results = redisTemplate.executePipelined(
            new RedisCallback<Object>() {
                @Override
                public Object doInRedis(RedisConnection connection) throws DataAccessException {
                    for (String key : keys) {
                        connection.get(key.getBytes());
                    }
                    return null;
                }
            }
        );

        return results.stream()
            .map(obj -> obj != null ? obj.toString() : null)
            .collect(Collectors.toList());
    }
}
```

#### Lua脚本批量操作
```java
public class RedisLuaBatchOperations {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    // 批量设置带过期时间
    public void batchSetWithExpire(Map<String, String> keyValues, int expireSeconds) {
        StringBuilder luaScript = new StringBuilder();
        luaScript.append("local expire = ARGV[1] ");

        List<String> keys = new ArrayList<>();
        List<String> values = new ArrayList<>();

        int i = 2;
        for (Map.Entry<String, String> entry : keyValues.entrySet()) {
            keys.add(entry.getKey());
            values.add(entry.getValue());

            luaScript.append("redis.call('setex', KEYS[")
                    .append(keys.size())
                    .append("], expire, ARGV[")
                    .append(i++)
                    .append("]) ");
        }

        values.add(0, String.valueOf(expireSeconds));

        DefaultRedisScript<Void> redisScript = new DefaultRedisScript<>();
        redisScript.setScriptText(luaScript.toString());

        redisTemplate.execute(redisScript, keys, values.toArray());
    }
}
```

### 3. 内存优化策略

#### 数据结构选择优化
```java
@Component
public class RedisMemoryOptimization {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // 使用Hash存储对象，节省内存
    public void saveUserAsHash(User user) {
        String key = "user:hash:" + user.getId();

        Map<String, String> userMap = new HashMap<>();
        userMap.put("name", user.getName());
        userMap.put("age", String.valueOf(user.getAge()));
        userMap.put("email", user.getEmail());

        redisTemplate.opsForHash().putAll(key, userMap);
        redisTemplate.expire(key, Duration.ofMinutes(30));
    }

    // 使用压缩序列化
    public void saveWithCompression(String key, Object value) {
        try {
            // 序列化
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(value);
            oos.close();

            // 压缩
            ByteArrayOutputStream compressedBaos = new ByteArrayOutputStream();
            GZIPOutputStream gzipOut = new GZIPOutputStream(compressedBaos);
            gzipOut.write(baos.toByteArray());
            gzipOut.close();

            // 存储
            redisTemplate.opsForValue().set(key, compressedBaos.toByteArray());

        } catch (IOException e) {
            throw new RuntimeException("压缩存储失败", e);
        }
    }
}
```

## 🔗 知识关联
- **核心原理**：[[Redis核心原理.md]]
- **问题解决**：[[Redis问题解决.md]]
- **相关技术**：[[../MySQL/MySQL性能优化.md]]
- **架构设计**：[[../../02-springboot.md#Redis集成]]

## 🏷️ 标签
#Redis #实战应用 #缓存策略 #分布式锁 #集群部署 #高可用 #性能优化 #消息队列 #限流器