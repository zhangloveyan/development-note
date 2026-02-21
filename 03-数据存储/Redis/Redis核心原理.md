# Redis - 核心原理

---
tags: [Redis, 缓存, 数据结构, 持久化, 内存数据库, 核心概念]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Redis内存数据库的核心原理和数据结构

- **数据结构**：10大数据类型及其底层实现原理
- **持久化机制**：RDB快照和AOF日志的工作原理
- **内存管理**：过期策略和淘汰机制
- **单线程模型**：Redis高性能的核心架构
- **网络模型**：基于事件驱动的IO多路复用

## 💡 数据结构详解

### 1. 十大数据类型

#### 基础数据类型
- **String**：字符串，最基本的数据类型
- **List**：双向链表，支持头尾操作
- **Hash**：哈希表，类似Java的HashMap
- **Set**：无序集合，元素唯一
- **ZSet**：有序集合，带分数排序

#### 高级数据类型
- **GEO**：地理坐标，支持位置计算
- **HyperLogLog**：基数统计，用于大数据去重计数
- **Bitmap**：位图，节省空间的布尔数组
- **Bitfield**：位域，操作字符串中的位
- **Stream**：消息流，类似Kafka的消息队列

### 2. 底层数据结构

#### String类型实现
```c
// SDS (Simple Dynamic String) 结构
struct sdshdr {
    int len;        // 字符串长度
    int free;       // 剩余空间
    char buf[];     // 字符数组
};
```

**特点**：
- 二进制安全，可存储任意数据
- 预分配空间，减少内存重分配
- 记录长度，O(1)时间获取长度

#### List类型实现
- **Redis 3.2前**：双向链表(linkedlist) + 压缩列表(ziplist)
- **Redis 3.2后**：快速列表(quicklist) = linkedlist + ziplist

```c
// quicklist节点
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;          // ziplist
    unsigned int sz;            // ziplist字节数
} quicklistNode;
```

#### Hash类型实现
- **小数据量**：压缩列表(ziplist)
- **大数据量**：哈希表(hashtable)

```c
// 哈希表结构
typedef struct dict {
    dictht ht[2];              // 两个哈希表，用于rehash
    long rehashidx;            // rehash进度
} dict;
```

#### Set类型实现
- **整数集合**：当元素都是整数且数量较少时
- **哈希表**：其他情况

#### ZSet类型实现
- **压缩列表**：元素数量少且元素长度小
- **跳跃表+哈希表**：其他情况

```c
// 跳跃表节点
typedef struct zskiplistNode {
    robj *obj;                 // 成员对象
    double score;              // 分值
    struct zskiplistNode *backward;  // 后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;  // 前进指针
        unsigned int span;              // 跨度
    } level[];
} zskiplistNode;
```

## 💾 持久化机制

### 1. RDB持久化

#### 工作原理
RDB是Redis数据库在某个时间点的快照，将内存中的数据以二进制形式保存到磁盘。

```bash
# 配置触发条件
save 3600 1     # 3600秒内至少1个key变化
save 300 100    # 300秒内至少100个key变化
save 60 10000   # 60秒内至少10000个key变化
```

#### 触发方式
```bash
# 手动触发
SAVE        # 同步保存，会阻塞Redis
BGSAVE      # 异步保存，fork子进程执行

# 自动触发
# 根据配置的save条件自动触发
```

#### Fork机制
```c
// Redis使用fork创建子进程
pid_t pid = fork();
if (pid == 0) {
    // 子进程执行RDB保存
    rdbSave();
} else {
    // 父进程继续处理客户端请求
}
```

**优点**：
- 文件紧凑，适合备份
- 恢复速度快
- 对性能影响小

**缺点**：
- 可能丢失最后一次快照后的数据
- fork过程可能阻塞

### 2. AOF持久化

#### 工作原理
AOF记录每个写操作命令，通过重新执行命令来恢复数据。

```bash
# 启用AOF
appendonly yes
appendfilename "appendonly.aof"
```

#### 写回策略
```bash
# 三种同步策略
appendfsync always      # 每个命令都同步
appendfsync everysec    # 每秒同步一次(默认)
appendfsync no          # 由操作系统决定
```

#### AOF重写机制
```bash
# 自动重写配置
auto-aof-rewrite-percentage 100    # 文件增长100%时重写
auto-aof-rewrite-min-size 64mb     # 最小64MB时才重写

# 手动重写
BGREWRITEAOF
```

**重写原理**：
1. fork子进程
2. 子进程遍历数据库，生成新的AOF文件
3. 父进程将新命令写入重写缓冲区
4. 子进程完成后，父进程将缓冲区内容追加到新文件
5. 原子性替换旧文件

### 3. 混合持久化

Redis 4.0引入混合持久化，结合RDB和AOF的优点：

```bash
# 启用混合持久化
aof-use-rdb-preamble yes
```

**工作方式**：
- AOF重写时，将当前数据以RDB格式写入AOF文件开头
- 增量数据以AOF格式追加到文件末尾
- 恢复时先加载RDB部分，再重放AOF部分

## 🧠 内存管理

### 1. 过期策略

#### 设置过期时间
```bash
# 设置过期时间
EXPIRE key seconds          # 相对时间
EXPIREAT key timestamp      # 绝对时间
PEXPIRE key milliseconds    # 毫秒级相对时间
TTL key                     # 查看剩余时间
```

#### 过期删除策略
Redis采用**惰性删除 + 定期删除**的组合策略：

**惰性删除**：
```c
// 访问key时检查是否过期
robj *lookupKeyRead(redisDb *db, robj *key) {
    robj *val = lookupKeyWrite(db, key);
    if (val == NULL) return NULL;

    // 检查过期时间
    if (expireIfNeeded(db, key) == 1) {
        return NULL;  // 已过期
    }
    return val;
}
```

**定期删除**：
- 每秒运行10次(100ms一次)
- 随机检查20个带过期时间的key
- 删除其中过期的key
- 如果过期key比例超过25%，重复执行

### 2. 内存淘汰策略

当内存使用达到maxmemory限制时，Redis会根据配置的策略淘汰数据：

```bash
# 配置最大内存
maxmemory 2gb

# 配置淘汰策略
maxmemory-policy allkeys-lru
```

#### 八种淘汰策略

| 策略 | 说明 |
|------|------|
| **noeviction** | 不淘汰，写入时返回错误(默认) |
| **allkeys-lru** | 从所有key中使用LRU算法淘汰 |
| **allkeys-lfu** | 从所有key中使用LFU算法淘汰 |
| **allkeys-random** | 从所有key中随机淘汰 |
| **volatile-lru** | 从设置过期时间的key中使用LRU淘汰 |
| **volatile-lfu** | 从设置过期时间的key中使用LFU淘汰 |
| **volatile-random** | 从设置过期时间的key中随机淘汰 |
| **volatile-ttl** | 淘汰即将过期的key |

#### LRU算法实现
Redis使用近似LRU算法，而非严格的LRU：

```c
// 每个对象都有lru字段记录最后访问时间
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS;  // 最后访问时间
    int refcount;
    void *ptr;
} robj;
```

## ⚡ 单线程模型

### 1. 为什么Redis是单线程的？

**单线程指的是网络IO和键值对读写是由一个线程完成的**，但Redis的其他功能，如持久化、异步删除、集群数据同步等，实际是由额外的线程执行的。

#### 单线程优势
1. **避免线程切换开销**
2. **避免锁竞争**
3. **简化数据结构设计**
4. **易于调试和维护**

#### 性能保证
1. **纯内存操作**：数据存储在内存中
2. **高效数据结构**：针对性优化的数据结构
3. **IO多路复用**：epoll/kqueue等高效IO模型
4. **避免系统调用开销**

### 2. 事件驱动模型

Redis基于Reactor模式实现事件驱动：

```c
// 事件循环伪代码
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        // 处理时间事件
        processTimeEvents(eventLoop);

        // 等待文件事件
        aeApiPoll(eventLoop, tvp);

        // 处理文件事件
        processFileEvents(eventLoop);
    }
}
```

#### 文件事件
- **连接应答处理器**：处理客户端连接
- **命令请求处理器**：处理客户端命令
- **命令回复处理器**：返回命令结果

#### 时间事件
- **定时事件**：如AOF重写、过期key清理
- **周期性事件**：如统计信息更新

### 3. Redis 6.0多线程

Redis 6.0引入了多线程，但仅用于网络IO处理：

```bash
# 启用IO多线程
io-threads 4
io-threads-do-reads yes
```

**多线程范围**：
- ✅ 网络IO读写
- ❌ 键值对操作(仍然单线程)

## 🔗 知识关联
- **实战应用**：[[Redis实战应用.md]]
- **问题解决**：[[Redis问题解决.md]]
- **相关技术**：[[../MySQL/MySQL基础原理.md]]
- **缓存设计**：[[../../02-springboot.md#Redis集成]]

## 🏷️ 标签
#Redis #缓存 #数据结构 #持久化 #内存数据库 #单线程 #事件驱动 #核心原理