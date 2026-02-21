# MySQL - 性能优化

---
tags: [MySQL, 性能优化, 索引优化, 查询优化, 调优, 最佳实践]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> MySQL数据库性能优化的关键技术和最佳实践

- **索引优化**：合理设计和使用索引，避免索引失效
- **查询优化**：SQL语句优化，执行计划分析
- **参数调优**：MySQL配置参数优化
- **架构优化**：读写分离、分库分表等架构设计
- **监控分析**：性能监控和瓶颈分析

## 🚀 索引优化策略

### 1. 索引设计原则

#### 适合创建索引的场景
1. **字段的值有唯一性限制**
2. **WHERE查询条件的字段**
3. **GROUP BY和ORDER BY的列**
4. **UPDATE、DELETE的WHERE条件列**
5. **DISTINCT去重的字段**
6. **多表JOIN的连接字段**

#### 索引设计最佳实践
```sql
-- 1. 使用小数据类型
-- 推荐：tinyint > int > bigint
CREATE INDEX idx_status ON orders(status); -- tinyint类型

-- 2. 字符串前缀索引
-- 计算选择度
SELECT COUNT(DISTINCT LEFT(address, 12)) / COUNT(*) FROM shop;
-- 创建前缀索引
ALTER TABLE shop ADD INDEX idx_address(address(12));

-- 3. 联合索引优于单值索引
-- 推荐：创建联合索引
CREATE INDEX idx_user_status_time ON orders(user_id, status, create_time);
-- 不推荐：创建多个单列索引
```

#### 联合索引使用规则
```sql
-- 索引：(a, b, c)
-- ✅ 可以使用索引的查询
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 AND c = 3  -- 只能用到a的索引

-- ❌ 无法使用索引的查询
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3
```

### 2. 索引失效场景

#### 常见索引失效情况
```sql
-- 1. 计算、函数、类型转换
-- ❌ 索引失效
WHERE LEFT(name, 3) = 'abc'
WHERE age + 1 = 25
WHERE phone = 13800138000  -- phone是varchar类型

-- ✅ 正确写法
WHERE name LIKE 'abc%'
WHERE age = 24
WHERE phone = '13800138000'

-- 2. 范围条件右边的列索引失效
-- 索引：(a, b, c)
-- ❌ b和c的索引失效
WHERE a = 1 AND b > 2 AND c = 3

-- 3. 不等于(!= 或 <>)索引失效
-- ❌ 索引失效
WHERE status != 1

-- ✅ 使用IN或EXISTS
WHERE status IN (0, 2, 3)

-- 4. LIKE以通配符开头
-- ❌ 索引失效
WHERE name LIKE '%abc'

-- ✅ 可以使用索引
WHERE name LIKE 'abc%'

-- 5. OR条件索引失效
-- ❌ 如果OR前后有一个字段没有索引，整个查询不走索引
WHERE name = 'abc' OR age = 25

-- ✅ 使用UNION
SELECT * FROM user WHERE name = 'abc'
UNION
SELECT * FROM user WHERE age = 25;
```

### 3. 覆盖索引优化

```sql
-- 创建覆盖索引，避免回表
CREATE INDEX idx_cover ON user(name, age, email);

-- ✅ 覆盖索引查询，无需回表
SELECT name, age, email FROM user WHERE name = 'abc';

-- ❌ 需要回表查询
SELECT * FROM user WHERE name = 'abc';
```

## 📊 查询优化技术

### 1. EXPLAIN执行计划分析

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1001;
```

#### 关键字段解读
| 字段 | 说明 | 重要值 |
|------|------|--------|
| **type** | 连接类型 | system > const > eq_ref > ref > range > index > ALL |
| **key** | 实际使用的索引 | NULL表示未使用索引 |
| **rows** | 扫描的行数 | 越小越好 |
| **Extra** | 额外信息 | Using index(覆盖索引)、Using filesort(需要排序) |

#### 性能等级
```sql
-- 🟢 优秀：const、eq_ref
EXPLAIN SELECT * FROM user WHERE id = 1;

-- 🟡 良好：ref、range
EXPLAIN SELECT * FROM user WHERE name = 'abc';
EXPLAIN SELECT * FROM user WHERE age BETWEEN 20 AND 30;

-- 🔴 需要优化：index、ALL
EXPLAIN SELECT * FROM user WHERE phone LIKE '%123%';
```

### 2. 查询优化技巧

#### 分页查询优化
```sql
-- ❌ 深分页性能差
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- ✅ 使用子查询优化
SELECT * FROM orders
WHERE id >= (SELECT id FROM orders ORDER BY id LIMIT 100000, 1)
ORDER BY id LIMIT 20;

-- ✅ 使用游标分页
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 20;
```

#### COUNT查询优化
```sql
-- ❌ 性能差
SELECT COUNT(*) FROM orders WHERE status = 1;

-- ✅ 使用近似值
SELECT table_rows FROM information_schema.tables
WHERE table_name = 'orders';

-- ✅ 维护计数表
CREATE TABLE order_count (
    status TINYINT,
    count INT,
    PRIMARY KEY (status)
);
```

#### 子查询优化
```sql
-- ❌ 相关子查询性能差
SELECT * FROM user u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- ✅ 使用JOIN替代
SELECT DISTINCT u.* FROM user u
INNER JOIN orders o ON u.id = o.user_id;
```

### 3. 批量操作优化

#### 批量插入优化
```sql
-- ❌ 逐条插入
INSERT INTO user (name, age) VALUES ('user1', 20);
INSERT INTO user (name, age) VALUES ('user2', 21);

-- ✅ 批量插入
INSERT INTO user (name, age) VALUES
('user1', 20), ('user2', 21), ('user3', 22);

-- ✅ 使用LOAD DATA
LOAD DATA INFILE '/path/to/data.csv'
INTO TABLE user
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n';
```

#### 批量更新优化
```sql
-- ❌ 逐条更新
UPDATE user SET age = 25 WHERE id = 1;
UPDATE user SET age = 26 WHERE id = 2;

-- ✅ 批量更新
UPDATE user SET age = CASE id
    WHEN 1 THEN 25
    WHEN 2 THEN 26
    END
WHERE id IN (1, 2);
```

## ⚙️ 参数调优

### 1. 连接相关参数

```ini
# 连接数配置
max_connections = 1000              # 最大连接数
max_connect_errors = 100000         # 最大连接错误数
connect_timeout = 10                # 连接超时时间
wait_timeout = 28800               # 等待超时时间
interactive_timeout = 28800         # 交互超时时间

# 线程池配置
thread_cache_size = 16             # 线程缓存大小
thread_pool_size = 16              # 线程池大小(CPU核数)
```

### 2. InnoDB参数优化

```ini
# 缓冲池配置
innodb_buffer_pool_size = 8G       # 缓冲池大小(物理内存的60-80%)
innodb_buffer_pool_instances = 8   # 缓冲池实例数

# 日志配置
innodb_log_file_size = 1G          # 日志文件大小
innodb_log_buffer_size = 64M       # 日志缓冲区大小
innodb_flush_log_at_trx_commit = 1 # 事务提交时刷新日志

# 刷新配置
innodb_flush_method = O_DIRECT     # 刷新方法
innodb_io_capacity = 2000          # IO容量
innodb_io_capacity_max = 4000      # 最大IO容量

# 其他配置
innodb_file_per_table = ON         # 每表一个文件
innodb_open_files = 4000           # 打开文件数
```

### 3. 查询缓存配置(MySQL 5.7及以下)

```ini
# 查询缓存(MySQL 8.0已移除)
query_cache_type = 1               # 启用查询缓存
query_cache_size = 256M            # 查询缓存大小
query_cache_limit = 2M             # 单个查询缓存限制
```

## 🏗️ 架构优化

### 1. 读写分离

#### 主从复制配置
```ini
# 主库配置
server-id = 1
log-bin = mysql-bin
binlog-format = ROW

# 从库配置
server-id = 2
relay-log = mysql-relay-bin
read-only = 1
```

#### 应用层读写分离
```java
// Spring Boot配置示例
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource writeDataSource() {
        // 主库配置
    }

    @Bean
    public DataSource readDataSource() {
        // 从库配置
    }

    @Bean
    public DataSource routingDataSource() {
        return new RoutingDataSource();
    }
}
```

### 2. 分库分表

#### 垂直分库
```sql
-- 按业务模块分库
-- 用户库
CREATE DATABASE user_db;
-- 订单库
CREATE DATABASE order_db;
-- 商品库
CREATE DATABASE product_db;
```

#### 水平分表
```sql
-- 按时间分表
CREATE TABLE orders_2024_01 LIKE orders;
CREATE TABLE orders_2024_02 LIKE orders;

-- 按哈希分表
CREATE TABLE user_0 LIKE user;
CREATE TABLE user_1 LIKE user;
-- 路由规则：user_id % 2
```

### 3. 连接池优化

#### HikariCP配置
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # 最大连接数
      minimum-idle: 5                # 最小空闲连接数
      connection-timeout: 30000      # 连接超时时间
      idle-timeout: 600000           # 空闲超时时间
      max-lifetime: 1800000          # 连接最大生存时间
      leak-detection-threshold: 60000 # 连接泄漏检测阈值
```

## 📈 性能监控

### 1. 关键性能指标

#### 查询性能监控
```sql
-- 慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 2;

-- 查看慢查询
SHOW VARIABLES LIKE 'slow_query%';

-- 查询统计
SHOW GLOBAL STATUS LIKE 'Com_select';
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

#### 连接状态监控
```sql
-- 连接数统计
SHOW GLOBAL STATUS LIKE 'Connections';
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Threads_running';

-- 查看当前连接
SHOW PROCESSLIST;
```

#### 缓冲池监控
```sql
-- InnoDB缓冲池状态
SHOW ENGINE INNODB STATUS;

-- 缓冲池命中率
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
```

### 2. 性能分析工具

#### MySQL自带工具
```sql
-- Performance Schema
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY sum_timer_wait DESC LIMIT 10;

-- 查看表统计信息
SHOW TABLE STATUS LIKE 'orders';

-- 分析表
ANALYZE TABLE orders;
```

#### 第三方工具
- **pt-query-digest**：慢查询日志分析
- **MySQLTuner**：MySQL配置优化建议
- **Percona Monitoring**：性能监控
- **Grafana + Prometheus**：可视化监控

## 🔧 优化检查清单

### 索引优化检查
- [ ] 是否为WHERE条件创建了合适的索引
- [ ] 是否使用了联合索引替代多个单列索引
- [ ] 是否避免了索引失效的情况
- [ ] 是否使用了覆盖索引减少回表

### 查询优化检查
- [ ] 是否使用EXPLAIN分析了执行计划
- [ ] 是否优化了深分页查询
- [ ] 是否避免了SELECT *
- [ ] 是否使用了合适的JOIN类型

### 配置优化检查
- [ ] 是否根据硬件配置调整了缓冲池大小
- [ ] 是否配置了合适的连接数
- [ ] 是否启用了慢查询日志
- [ ] 是否配置了合适的超时参数

## 🔗 知识关联
- **基础原理**：[[MySQL基础原理.md]]
- **问题解决**：[[MySQL问题解决.md]]
- **相关技术**：[[../Redis/Redis实战应用.md#缓存优化]]
- **架构设计**：[[../../08-docker.md#数据库容器化]]

## 🏷️ 标签
#MySQL #性能优化 #索引优化 #查询优化 #参数调优 #架构优化 #监控分析 #最佳实践