# MySQL - 基础原理

---
tags: [MySQL, 数据库原理, 存储引擎, 索引, 事务, 核心概念]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> MySQL数据库的基础原理和核心概念

- **数据类型**：整数、浮点数、定点数、日期、字符等基础数据类型
- **逻辑架构**：连接池、解析器、优化器、执行器的完整执行流程
- **物理架构**：页、区、段、表空间的存储结构
- **索引原理**：B+树结构、聚簇索引、二级索引的实现机制
- **存储引擎**：InnoDB与MyISAM的特性对比

## 💡 数据类型详解

### 1. 数值类型

#### 整数类型
- **tinyint**：1字节(8bit) 2^8 0-255(unsigned)
- **int**：4字节
- **bigint**：8字节

> bigint(M) 数据宽度，补零 M取值范围0-255

#### 浮点数类型
- **float**：4字节
- **double**：8字节

> float(M,D) M精度、D标度  M(0-255)=整数位+小数位 D(0-30)=小数位
>
> float(5,2) -999.99~999.99

#### 定点数类型
- **decimal**：底层字符串存储，存储空间=M+2位

> decimal(M,D) M精度、D标度  M(0-65)=整数位+小数位 D(0-30)=小数位
>
> decimal(5,2) -999.99~999.99

### 2. 日期时间类型
- **year**：年份
- **date**：日期
- **time**：时间
- **datetime**：日期时间
- **timestamp**：时间戳

```sql
-- 日期过滤示例
WHERE 日期 > '2023-7-29'  -- 默认 0 点'2023-7-29 00:00:00'
-- 格式化日期
DATE_FORMAT(time,'%Y%m%d %H%i%s')
```

### 3. 字符类型
- **char(M)**：固定长度，M字节
- **varchar(M)**：可变长度，M+1字节（需要额外1个字节记录长度）
- **text**：长文本
- **longtext**：超长文本

### 4. 其他类型
- **枚举**：enum类型
- **二进制**：blob类型，用于存储图片等
- **json**：JSON数据类型

## 🏗️ MySQL逻辑架构

### 1. 客户端请求处理流程

![MySQL架构图](../../../pic/04-mysql/image-20230801095132905.png)

#### 连接池
创建连接池队列存放数据库连接，客户端可直接选取可用连接访问数据库，用完后归还给连接池，避免重复建立连接的开销。

**Druid连接池关键配置**：
```properties
# 基础配置
url=jdbc:mysql://host:port/database?connection_properties
username=数据库用户名
password=数据库密码

# 连接池大小配置
initialSize=0          # 初始连接数
maxActive=8           # 最大连接数
minIdle=0             # 最小空闲连接数
maxWait=-1            # 获取连接最大等待时间(毫秒)

# 连接检测配置
testWhileIdle=false   # 空闲时检测连接
validationQuery=SELECT 1  # 连接检测SQL
testOnBorrow=false    # 获取连接时检测
testOnReturn=false    # 归还连接时检测

# 预处理语句缓存
poolPreparedStatements=false
maxOpenPreparedStatements=-1
```

**MySQL连接参数优化**：
```shell
# 连接相关参数
max_connections=500-1000      # 最大连接数
back_log=512                  # TCP积压请求栈大小
thread_pool_size=CPU核数*2    # 线程池大小

# InnoDB相关参数
innodb_log_file_size=1GB-2GB          # 日志文件大小
innodb_log_buffer_size=32MB-64MB      # 日志缓存大小
innodb_max_dirty_pages_pct=70%-80%    # 最大脏页比例
```

**缓冲参数设置**：
```shell
# 缓冲池配置
innodb_buffer_pool_size=物理内存60%-80%  # InnoDB缓冲池大小
key_buffer_size=物理内存10%              # 索引缓存大小
table_cache=512-1024                    # 同时打开表的个数
query_cache_size=根据实际情况调整         # 查询缓冲区大小(8.0已废弃)
```

### 2. SQL执行流程

![SQL执行流程](../../../pic/04-mysql/d4d2a807dc1d4991af28a565f3dbc1a7.png)

#### 解析器
- **词法分析**：将SQL语句分解为关键字、标识符、操作符等
- **语法分析**：检查SQL语法是否正确，构建语法树

#### 优化器
- **逻辑查询优化**：条件表达式等价谓词重写、条件简化、视图重写、子查询优化、外连接消除等
- **物理查询优化**：计算各种物理路径的代价，选择代价最小的执行计划

#### 执行器
调用存储引擎API对表进行读写操作

### 3. SELECT执行顺序

```sql
(7) SELECT
(8) DISTINCT <select_list>
(1) FROM <left_table>
(3) <join_type> JOIN <right_table>
(2) ON <join_condition>
(4) WHERE <where_condition>
(5) GROUP BY <group_by_list>
(6) HAVING <having_condition>
(9) ORDER BY <order_by_condition>
(10) LIMIT <limit_number>
```

**执行步骤详解**：
1. **FROM**：对FROM子句中的表执行笛卡尔积，生成虚拟表VT1
2. **ON**：对VT1应用ON筛选器，生成VT2
3. **OUTER JOIN**：如果是外连接，添加外部行到VT2，生成VT3
4. **WHERE**：对VT3应用WHERE筛选器，生成VT4
5. **GROUP BY**：按GROUP BY列表对VT4分组，生成VT5
6. **HAVING**：对VT6应用HAVING筛选器，生成VT7
7. **SELECT**：处理SELECT列表，生成VT8
8. **DISTINCT**：去除重复行，生成VT9
9. **ORDER BY**：排序生成游标VC10
10. **LIMIT**：选择指定数量的行，生成最终结果

## 🗄️ MySQL物理架构

### 1. 存储基本单位

#### 页(Page)
- **大小**：16KB
- **作用**：数据存放的基本单位
- **结构**：页中的行数据用链表连接

#### 区(Extent)
- **大小**：64个页 = 1MB
- **作用**：让页连续存储，提高顺序IO性能
- **碎片区**：数据页不足时，其他段的数据可先存入，占用32个区时完成分配

#### 段(Segment)
- **组成**：一个或多个区
- **分类**：数据段、索引段、回滚段
- **功能分类**：叶子节点段、非叶子节点段

#### 表空间(Tablespace)
- **作用**：逻辑容器，存储段

![存储结构图](../../../pic/04-mysql/image-20230805212115848.png)

### 2. 页的内部结构

![页结构图](../../../pic/04-mysql/image-20230805212452085.png)

#### 行格式
- **Compact**：经典格式
- **Dynamic**：默认格式，支持行溢出

![行格式图](../../../pic/04-mysql/image-20230805213936208.png)

#### 页目录
为了解决链表查找慢的问题，MySQL使用页目录对记录分组：
- 每组保存最大记录的位置
- 查找时先在页目录中定位组，再在组内查找
- 大大提高查找效率

![页目录图](../../../pic/04-mysql/image-20230805230118818.png)

#### 隐藏列
除了用户定义的列，MySQL还会添加隐藏列：
- **DB_ROW_ID**：没有主键时自动添加的隐藏主键
- **DB_TRX_ID**：事务ID，记录最后修改该行的事务
- **DB_ROLL_PTR**：回滚指针，指向undo log的指针

## 🔍 索引原理

### 1. 索引分类

#### 按功能逻辑分类
- **普通索引**：最基本的索引
- **唯一索引**：索引列的值必须唯一
- **主键索引**：特殊的唯一索引，不允许NULL
- **全文索引**：用于全文搜索

#### 按物理实现分类
- **聚簇索引**：叶子节点存储完整的行数据
- **非聚簇索引(二级索引)**：叶子节点存储主键值，需要回表查询

#### 按字段个数分类
- **单列索引**：只包含一个列
- **联合索引**：包含多个列

### 2. B+树结构

#### B+树 vs B树
- **B树**：所有节点都存储数据，范围查找效率低
- **B+树**：只有叶子节点存储数据，非叶子节点只存储键值，范围查找效率高

#### B+树优势
- **层级少**：减少磁盘IO次数
- **范围查询高效**：叶子节点形成链表
- **稳定性好**：所有查询都需要到叶子节点

### 3. 索引操作

#### 查看索引
```sql
SHOW INDEX FROM 表名;
-- 示例
SHOW INDEX FROM student_info_1;
```

#### 创建索引
```sql
-- 方式1：ALTER TABLE
ALTER TABLE 表名 ADD [索引类型] INDEX 索引名称(字段名称);

-- 方式2：CREATE INDEX
CREATE [索引类型] INDEX 索引名称 ON 表名(字段名称);

-- 示例
ALTER TABLE student_info_1 ADD UNIQUE INDEX idx_cre_time(create_time);
CREATE INDEX idx_cre_time ON student_info_1(create_time);
```

#### 删除索引
```sql
-- 方式1：ALTER TABLE
ALTER TABLE 表名 DROP INDEX 索引名称;

-- 方式2：DROP INDEX
DROP INDEX 索引名称 ON 表名;

-- 示例
ALTER TABLE student_info_1 DROP INDEX idx_cre_time;
DROP INDEX idx_cre_time ON student_info_1;
```

## ⚙️ 存储引擎

### InnoDB vs MyISAM对比

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持ACID事务 | ❌ 不支持事务 |
| **锁机制** | 行级锁 | 表级锁 |
| **外键** | ✅ 支持外键约束 | ❌ 不支持外键 |
| **索引类型** | 聚簇索引 | 非聚簇索引 |
| **存储空间** | 相对较大 | 占用空间小 |
| **处理速度** | 写入优化 | 读取速度快 |
| **崩溃恢复** | ✅ 支持崩溃恢复 | ❌ 不支持 |
| **适用场景** | 大量写入、更新、删除 | 大量读取、原子性要求不高 |

### 查看存储引擎
```sql
SHOW VARIABLES LIKE '%storage%';
```

## 🔗 知识关联
- **性能优化**：[[MySQL性能优化.md]]
- **问题解决**：[[MySQL问题解决.md]]
- **实战应用**：[[../../../02-springboot.md#数据库操作]]
- **相关技术**：[[../Redis/Redis核心原理.md]]

## 🏷️ 标签
#MySQL #数据库原理 #存储引擎 #索引 #事务 #核心概念 #InnoDB #B+树