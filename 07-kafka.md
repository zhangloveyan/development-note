## 基础

### 1.需求背景

1.java 线程之间的数据交互

![image-20240503154928368](pic/07-kafka/image-20240503154928368.png)

2.进程之间的数据交互

![image-20240503155056213](pic/07-kafka/image-20240503155056213.png)

由于各种原因，消费速度、资源、耦合等问题，引入消息中间件的概念

![image-20240503155416490](pic/07-kafka/image-20240503155416490.png)

JMS 模型（java message service）

![image-20240503155723290](pic/07-kafka/image-20240503155723290.png)

消费模式：

1. P2P（点对点）只有一个接收成功
2. PS（发布订阅）订阅就可以收到

#### 常用中间件对比

![image-20240503160051051](pic/07-kafka/image-20240503160051051.png)

![image-20240503160242760](pic/07-kafka/image-20240503160242760.png)

大数据场景适用 kafka，javaee 适用 mq，可结合适用

#### kafka 组件

![image-20240503160915173](pic/07-kafka/image-20240503160915173.png)

### 2.安装

#### 安装

依赖 zookeeper，也可以适用 raft。

需要修改配置文件

window 写命令：

zk.cmd

```shell
call bin/windows/zookeeper-server-start.bat config/zookeeper.properties
```

kfk.cmd

```shell
call bin/windows/kafka-server-start.bat config/server.properties
```

#### 简单指令

```shell
# 创建 topic
kafka-topics.bat --bootsrtap-server localhost:9092 --topic test --create
# 查看
kafka-topics.bat --bootsrtap-server localhost:9092 --list
# 查看详细
kafka-topics.bat --bootsrtap-server localhost:9092 --topic test --describe
# 修改
kafka-topics.bat --bootsrtap-server localhost:9092 --topic test --alter --partitions 2
# 删除
kafka-topics.bat --bootsrtap-server localhost:9092 --topic test --delete
```

![image-20240503162647631](pic/07-kafka/image-20240503162647631.png)

```shell
# 启动消费者
kafka-console-consumer.bat --bootsrtap-server localhost:9092 --topic test 

# 发送测试消息
kafka-console-producer.bat --bootsrtap-server localhost:9092 --topic test 
```

![image-20240503163315275](pic/07-kafka/image-20240503163315275.png)

### 3.引入

pom文件引入

![image-20240503191209779](pic/07-kafka/image-20240503191209779.png)

构建生产者

![image-20240503191301734](pic/07-kafka/image-20240503191301734.png)

![image-20240503191343960](pic/07-kafka/image-20240503191343960.png)

构建消费者

![image-20240503191621019](pic/07-kafka/image-20240503191621019.png)

![image-20240503191729271](pic/07-kafka/image-20240503191729271.png)

### 4.客户端工具

kafkatool_64bit.exe

添加客户端连接

![image-20240503192010302](pic/07-kafka/image-20240503192010302.png)

使用

![image-20240503192044620](pic/07-kafka/image-20240503192044620.png)

## 原理

### 架构推演

单点 broker ，如果挂掉，则无法使用，那么可以有横向扩展（加多实例）纵向扩展（加内存）

横向扩展演进

1.多节点、分区（数据分散在各个broker）、

2.消费者组（防止单一节点请求）

![image-20240503194635609](pic/07-kafka/image-20240503194635609.png)

3.单节点log文件副本

可以通过备份数据文件，保证可靠性。但是 kafka 没有备份概念，这里叫副本

多个副本同时只能有一个提供数据的读写操作

具有读写能力的副本称之为 Leader 副本，作为备份的副本称之为 Follower 副本

![image-20240503200516678](pic/07-kafka/image-20240503200516678.png)

4.master 节点，controller 控制器

管理节点

![image-20240503200949178](pic/07-kafka/image-20240503200949178.png)

但如果节点挂了怎么办？

1.管理者备份

![image-20240503201202875](pic/07-kafka/image-20240503201202875.png)

2.选举 controller

使用 zookeeper 进行选举

![image-20240503201419205](pic/07-kafka/image-20240503201419205.png)

### 最终架构

![image-20240503201747526](pic/07-kafka/image-20240503201747526.png)  

关于选举：

比如三个节点 1、2、3，当他们连接 zookeeper 时，第一个连上的做节点（controller），其他的监视这个节点，比如 1 当选，2、3 监视。当 1 挂掉时，2、3 发送请求，谁先建立连接谁就当选 controller。

### 底层通讯

多节点连接

![image-20240504113532362](pic/07-kafka/image-20240504113532362.png)

通信

![image-20240504114024083](pic/07-kafka/image-20240504114024083.png)

P19 集 底层深入分析

## 使用

### 主题

![image-20240504120146811](pic/07-kafka/image-20240504120146811.png)

多个broker，放多个分区，放不同的副本

![image-20240504120723307](pic/07-kafka/image-20240504120723307.png)

例子：三个topic，一个1分区、一个2分区、一个3分区

![image-20240504121516596](pic/07-kafka/image-20240504121516596.png)

每个颜色的框是一个 topic，⭐️为 leader，这样每个节点更均衡。

### 数据发送

![image-20240504145412196](pic/07-kafka/image-20240504145412196.png)

![image-20240504145525579](pic/07-kafka/image-20240504145525579.png)

#### 自定义拦截器

1.在map中配置

![image-20240504150513784](pic/07-kafka/image-20240504150513784.png)

2.创建类，实现拦截器接口

![image-20240504150440718](pic/07-kafka/image-20240504150440718.png)

#### 分区计算

通过元数据拿到集群相关 数据，各个集群、分区等信息

通过计算，要发到那个分区

发送时有缓冲区，一批一批发送。默认缓存 32m，批次 16k。

![image-20240504152053540](pic/07-kafka/image-20240504152053540.png)

数据通过 sender 线程发送。发送后，会有发送应答，通过回调方法

![image-20240504152427019](pic/07-kafka/image-20240504152427019.png)

同步发送，等待成功在发下一条，逐条发送

![image-20240504152924823](pic/07-kafka/image-20240504152924823.png)

#### 可配置应答模式 acks

acks = 0 （优先满足数据传输，传输快，但数据可能丢失）

![image-20240504153222295](pic/07-kafka/image-20240504153222295.png)

acks = 1 leader 保存好了，就行了

![image-20240504153432778](pic/07-kafka/image-20240504153432778.png)

acks = -1（all）最可靠，需要等待多副本同步数据

![image-20240504153333316](pic/07-kafka/image-20240504153333316.png)

#### 失败重试

可能造成问题：

1.副本写入成功，acks 响应失败时，数据会重复。

2.数据乱序问题，重复推送时，造成的数据错乱。

#### 幂等性操作

开启幂等性配置（默认开启），幂等性只对同一个分区起作用

acks 必须为 -1，且开启重试。

基于生产者的produceid和序列号，比对数据、比对重复。

![image-20240504170323606](pic/07-kafka/image-20240504170323606.png)

#### 生产者事务

当使用事务时，produceId和kafka实例是绑定的

添加事务id

![image-20240504172918307](pic/07-kafka/image-20240504172918307.png)

使用事务

![image-20240504172013393](pic/07-kafka/image-20240504172013393.png)

### 数据存储

数据保存在 .log 文件中，可以设置刷写时间，log 文件会根据策略生成小数据文件

kafka 工具类可以查看文件内容

执行 DumpLogSegments 类的方法

```shell
kafka-run-class.bat kafka.tools.DumpLogSegments --files D:/data/kafka/test-0/00000000.log --print-data-log
```

![image-20240504191905130](pic/07-kafka/image-20240504191905130.png)

通过索引、偏移量定位数据。offset、position

#### 日志删除

index 文件是 kafka 的内存映射文件，kafka 启动时，无法删除。其他文件 log timestamp 可以删除。默认保存时间如下：

![image-20240504194514847](pic/07-kafka/image-20240504194514847.png)

### 数据消费

#### 过程

![image-20240504200456721](pic/07-kafka/image-20240504200456721.png)

根据主题和偏移量进行数据获取。

#### 偏移量配置：

![image-20240504195227138](pic/07-kafka/image-20240504195227138.png)

消费者自动保存偏移量，防止重启时，不知道从什么地方开始（默认 5s 保存一次），但是也会有一些问题，重复消费。

**手动保存偏移量**

1.关闭自动提交

![image-20240504195640059](pic/07-kafka/image-20240504195640059.png)

2.手动保存偏移量

![image-20240504195708179](pic/07-kafka/image-20240504195708179.png)

#### 主题 topic 消费组 group

消费者一定是 < = 分区数，如果大于，则消费者相当于备用。

消费者构成组去组合消费，不用每一个都去消费。且拉取的数据不会重复。

![image-20240504201424993](pic/07-kafka/image-20240504201424993.png)

分区分配策略

消费者中的每个消费者到底消费哪一个主题分区，策略由消费者的 Leader 决定，称为群主。

第一个加入组中的消费者就是群主，其他为 Follower。

![image-20240505094700056](pic/07-kafka/image-20240505094700056.png)

加群时会发 JoinGroup 请求，群主负责给消费者分配分区。

分配策略

1.轮询分配

![image-20240505094840338](pic/07-kafka/image-20240505094840338.png)

2.范围分配

![image-20240505095129628](pic/07-kafka/image-20240505095129628.png)

3.粘性分配

![image-20240505095753680](pic/07-kafka/image-20240505095753680.png)

4.优化后的分配

![image-20240505095917769](pic/07-kafka/image-20240505095917769.png)

## 扩展

### 脑裂问题

集群中，有多个管理者，那么 broker 到底听谁的呢？

epoch纪元，会存储第几届管理者（相当于给管理者按顺序编号），如果发现某一个管理者不是最新编号，那么相当于他下台了，就不用听他的就好了

### 零拷贝

数据流转，如果是这种，cpu 会经常切换 -> 用户态、内核态，消耗资源。

![image-20240505110945251](pic/07-kafka/image-20240505110945251.png)

优化后，直接走内核发送数据，不在通过 io 走到 broker

![image-20240505111127342](pic/07-kafka/image-20240505111127342.png)

### 顺写日志

数据追加进去，类似于时序数据库，提高效率

顺序写入对机械和固态差别不大

### 集群

2.8 版本提供 KRaft，摆脱对 zookeeper 的依赖

## 优化

磁盘空间预估：

![image-20240505114909517](pic/07-kafka/image-20240505114909517.png)

网络估算

![image-20240505115035212](pic/07-kafka/image-20240505115035212.png)

![image-20240505115100611](pic/07-kafka/image-20240505115100611.png)

![image-20240505115137741](pic/07-kafka/image-20240505115137741.png)内存配置

![image-20240505115155721](pic/07-kafka/image-20240505115155721.png)

cpu 选择

![image-20240505115239981](pic/07-kafka/image-20240505115239981.png)

配置参数

![image-20240505115626868](pic/07-kafka/image-20240505115626868.png)

数据压缩

![image-20240505115744572](pic/07-kafka/image-20240505115744572.png)

## 常见问题

### 1.LSO LEO HW 含义

都是 kafka 中的偏移量

![image-20240505120005317](pic/07-kafka/image-20240505120005317.png)

![image-20240505120020688](pic/07-kafka/image-20240505120020688.png)

### 2.Controller 选举怎么实现

![image-20240505120117810](pic/07-kafka/image-20240505120117810.png)

### 3.分区副本 AR ISR OSR 含义

![image-20240505120201036](pic/07-kafka/image-20240505120201036.png)

### 4.Producer ACK 应答策略

生产者发送数据后的 kafka 接收确认方式。有 三 种

![image-20240505120351994](pic/07-kafka/image-20240505120351994.png)

![image-20240505120412971](pic/07-kafka/image-20240505120412971.png)

### 5.Producer消息重复或丢失原因？

![image-20240505120539429](pic/07-kafka/image-20240505120539429.png)

### 6.Consumer 消息重复或丢失的原因？

![image-20240505120742257](pic/07-kafka/image-20240505120742257.png)

### 7.如何保证有序

![image-20240505120823913](pic/07-kafka/image-20240505120823913.png)

![image-20240505120859783](pic/07-kafka/image-20240505120859783.png)

![image-20240505120912163](pic/07-kafka/image-20240505120912163.png)

 