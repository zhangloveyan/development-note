# Kafka - 实战

---
tags: [Kafka, 消息队列, 大数据, 流处理, 分布式]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Apache Kafka分布式流处理平台的实战应用

- **高吞吐量**：支持百万级消息处理能力
- **分布式架构**：多节点集群部署和管理
- **持久化存储**：消息持久化到磁盘，支持数据回放
- **流处理**：实时数据流处理和分析

## 💡 原理详解

### 1. Kafka架构组件

#### 核心概念
- **Broker**：Kafka服务器节点
- **Topic**：消息主题，逻辑上的消息分类
- **Partition**：主题的物理分区，提高并行度
- **Producer**：消息生产者
- **Consumer**：消息消费者
- **Consumer Group**：消费者组，实现负载均衡

#### 架构演进
```
单点Broker → 多节点集群 → 分区机制 → 副本机制 → Controller管理
```

### 2. 分区和副本机制

#### 分区策略
- **轮询分区**：消息均匀分布到各分区
- **随机分区**：随机选择分区
- **按键分区**：根据消息key的hash值分区
- **自定义分区**：实现自定义分区逻辑

#### 副本机制
- **Leader副本**：负责读写操作
- **Follower副本**：同步Leader数据，提供容错
- **ISR（In-Sync Replicas）**：同步副本集合

### 3. 消费者组和分区分配

#### 分配策略
1. **Range分配**：按分区范围分配
2. **RoundRobin分配**：轮询分配
3. **Sticky分配**：粘性分配，减少重平衡
4. **CooperativeSticky分配**：协作式粘性分配

## 🔧 代码示例

### 基础用法

#### 环境搭建
```bash
# 启动Zookeeper
bin/zookeeper-server-start.sh config/zookeeper.properties

# 启动Kafka
bin/kafka-server-start.sh config/server.properties

# 创建Topic
bin/kafka-topics.sh --create --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1

# 查看Topic
bin/kafka-topics.sh --list --bootstrap-server localhost:9092

# 查看Topic详情
bin/kafka-topics.sh --describe --topic test-topic \
  --bootstrap-server localhost:9092
```

#### Java生产者
```java
@Component
public class KafkaProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public KafkaProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // 简单发送
    public void sendMessage(String topic, Object message) {
        kafkaTemplate.send(topic, message);
    }

    // 带key发送
    public void sendMessage(String topic, String key, Object message) {
        kafkaTemplate.send(topic, key, message);
    }

    // 异步发送带回调
    public void sendMessageAsync(String topic, Object message) {
        ListenableFuture<SendResult<String, Object>> future =
            kafkaTemplate.send(topic, message);

        future.addCallback(new ListenableFutureCallback<SendResult<String, Object>>() {
            @Override
            public void onSuccess(SendResult<String, Object> result) {
                System.out.println("消息发送成功: " + result.getRecordMetadata());
            }

            @Override
            public void onFailure(Throwable ex) {
                System.err.println("消息发送失败: " + ex.getMessage());
            }
        });
    }

    // 同步发送
    public void sendMessageSync(String topic, Object message) {
        try {
            SendResult<String, Object> result = kafkaTemplate.send(topic, message).get();
            System.out.println("消息发送成功: " + result.getRecordMetadata());
        } catch (Exception e) {
            System.err.println("消息发送失败: " + e.getMessage());
        }
    }
}
```

#### Java消费者
```java
@Component
public class KafkaConsumer {

    // 简单消费
    @KafkaListener(topics = "test-topic", groupId = "test-group")
    public void consume(String message) {
        System.out.println("收到消息: " + message);
    }

    // 消费带元数据
    @KafkaListener(topics = "user-events", groupId = "user-group")
    public void consumeWithMetadata(
            @Payload String message,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {

        System.out.printf("收到消息: %s, Topic: %s, Partition: %d, Offset: %d%n",
                message, topic, partition, offset);
    }

    // 批量消费
    @KafkaListener(topics = "batch-topic", groupId = "batch-group",
                   containerFactory = "batchKafkaListenerContainerFactory")
    public void consumeBatch(List<String> messages) {
        System.out.println("批量处理消息数量: " + messages.size());
        messages.forEach(message -> {
            // 处理单条消息
            processMessage(message);
        });
    }

    // 手动确认消费
    @KafkaListener(topics = "manual-topic", groupId = "manual-group")
    public void consumeManual(String message, Acknowledgment ack) {
        try {
            // 处理消息
            processMessage(message);
            // 手动确认
            ack.acknowledge();
        } catch (Exception e) {
            System.err.println("消息处理失败: " + e.getMessage());
            // 不确认，消息会重新消费
        }
    }

    private void processMessage(String message) {
        // 业务逻辑处理
        System.out.println("处理消息: " + message);
    }
}
```

### 高级用法

#### 生产者配置
```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();

        // 基础配置
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // 性能配置
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384); // 批次大小
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 5); // 等待时间
        configProps.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432); // 缓冲区大小

        // 可靠性配置
        configProps.put(ProducerConfig.ACKS_CONFIG, "all"); // 确认机制
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3); // 重试次数
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // 幂等性

        // 压缩配置
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

        return new DefaultKafkaProducerFactory<>(configProps);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

#### 消费者配置
```java
@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();

        // 基础配置
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "default-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        // 消费配置
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // 手动提交
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500); // 每次拉取记录数

        // 会话配置
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3000);

        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());

        // 并发配置
        factory.setConcurrency(3);

        // 确认模式
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);

        // 错误处理
        factory.setErrorHandler(new SeekToCurrentErrorHandler());

        return factory;
    }

    // 批量消费配置
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            batchKafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setBatchListener(true); // 启用批量监听

        return factory;
    }
}
```

#### 自定义分区器
```java
public class CustomPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {

        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (key == null) {
            // 无key时使用轮询
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }

        // 根据key的hash值分区
        if (key instanceof String) {
            String stringKey = (String) key;
            if (stringKey.startsWith("VIP")) {
                // VIP用户使用特定分区
                return 0;
            }
        }

        // 默认hash分区
        return Math.abs(key.hashCode()) % numPartitions;
    }

    @Override
    public void close() {
        // 清理资源
    }

    @Override
    public void configure(Map<String, ?> configs) {
        // 配置初始化
    }
}
```

#### 事务支持
```java
@Service
@Transactional
public class TransactionalKafkaService {

    @Autowired
    private KafkaTransactionManager transactionManager;

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaTransactional
    public void sendTransactionalMessages() {
        try {
            // 发送多条消息，要么全部成功，要么全部失败
            kafkaTemplate.send("topic1", "message1");
            kafkaTemplate.send("topic2", "message2");
            kafkaTemplate.send("topic3", "message3");

            // 模拟业务异常
            if (someBusinessCondition()) {
                throw new RuntimeException("业务异常，回滚事务");
            }

        } catch (Exception e) {
            // 事务会自动回滚
            throw e;
        }
    }

    private boolean someBusinessCondition() {
        return false;
    }
}

// 事务配置
@Configuration
public class KafkaTransactionConfig {

    @Bean
    public KafkaTransactionManager kafkaTransactionManager(
            ProducerFactory<String, Object> producerFactory) {
        return new KafkaTransactionManager<>(producerFactory);
    }

    @Bean
    public ProducerFactory<String, Object> transactionalProducerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // 事务配置
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "tx-producer");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

## ⚡ 性能特点

| 特性 | 说明 | 适用场景 |
|------|------|----------|
| 高吞吐量 | 百万级消息/秒 | 大数据处理 |
| 低延迟 | 毫秒级延迟 | 实时系统 |
| 持久化 | 消息持久化存储 | 数据不丢失 |
| 水平扩展 | 支持集群扩展 | 大规模部署 |

## 🔗 知识关联

- **前置知识**：[[消息队列原理]] [[分布式系统基础]]
- **相关技术**：[[Zookeeper]] [[Kafka Streams]]
- **实战应用**：[[实时数据处理]] [[日志收集系统]]
- **问题解决**：[[消息队列问题解决]]

## 🏷️ 标签
#Kafka #消息队列 #大数据 #流处理 #分布式 #实时系统 #面试重点