# Java问题解决

---
tags: [问题解决, Java, 调试, 性能优化, 内存问题, 并发问题]
created: 2026-02-21
updated: 2026-02-21
status: 持续更新
importance: ⭐⭐⭐⭐
---

## 🚨 高频问题速查

### 问题1：内存溢出OutOfMemoryError `#内存问题`
**现象**：程序运行时抛出java.lang.OutOfMemoryError异常
**原因**：
- 堆内存不足：对象创建过多或内存泄漏
- 栈内存不足：递归调用过深
- 方法区不足：类加载过多（JDK 1.7及之前）
- 直接内存不足：NIO操作使用过多堆外内存

**解决**：
1. 增加堆内存：-Xms2g -Xmx4g
2. 分析内存使用：使用MAT工具分析heap dump
3. 检查内存泄漏：重点关注集合类、缓存、监听器
4. 优化代码：及时释放资源，避免大对象

**相关原理**：[[Java虚拟机#内存模型]]

---

### 问题2：线程死锁 `#并发问题`
**现象**：程序卡死，多个线程相互等待对方释放锁
**原因**：两个或多个线程持有各自的锁，同时等待获取对方的锁

**解决**：
1. 使用jstack命令检查线程状态：`jstack <pid>`
2. 统一锁的获取顺序
3. 使用tryLock()设置超时时间
4. 减少锁的持有时间

```java
// 死锁排查命令
jps -l                    // 获取进程ID
jstack <pid>              // 查看线程堆栈
```

**相关原理**：[[Java并发编程#死锁]]

---

### 问题3：CPU使用率过高 `#性能问题`
**现象**：系统CPU使用率持续很高，响应缓慢
**原因**：
- 无限循环或死循环
- 频繁的GC操作
- 大量的线程上下文切换
- 计算密集型操作

**解决**：
1. 使用top命令找到占用CPU高的Java进程
2. 使用jstack分析线程状态
3. 使用jstat监控GC情况
4. 优化算法和数据结构

```bash
# CPU问题排查步骤
top                       # 找到CPU占用高的进程
top -H -p <pid>          # 查看该进程的线程CPU使用情况
printf "%x\n" <thread_id> # 将线程ID转为16进制
jstack <pid> | grep <hex_thread_id> # 查找对应线程的堆栈
```

**相关原理**：[[Java虚拟机#垃圾回收]]

---

### 问题4：频繁Full GC `#GC问题`
**现象**：应用频繁出现Full GC，导致停顿时间长
**原因**：
- 老年代空间不足
- 永久代/元空间不足
- 内存泄漏导致对象无法回收
- 新生代设置过小

**解决**：
1. 调整堆内存分配：增加老年代大小
2. 优化GC参数：选择合适的垃圾收集器
3. 分析内存使用：查找内存泄漏点
4. 调整新生代比例：-XX:NewRatio=2

```bash
# GC监控命令
jstat -gc <pid> 1000     # 每秒输出GC统计信息
jstat -gccapacity <pid>  # 查看各代内存容量
```

**相关原理**：[[Java虚拟机#垃圾回收器]]

---

### 问题5：HashMap在多线程环境下的问题 `#并发问题`
**现象**：
- 数据丢失
- 死循环（JDK 1.7及之前）
- 数据不一致

**原因**：HashMap非线程安全，并发操作可能导致链表环形结构

**解决**：
1. 使用ConcurrentHashMap替代HashMap
2. 使用Collections.synchronizedMap()包装
3. 使用ThreadLocal为每个线程提供独立的Map
4. 在同步块中使用HashMap

```java
// 线程安全的Map选择
Map<String, Object> map = new ConcurrentHashMap<>(); // 推荐
Map<String, Object> syncMap = Collections.synchronizedMap(new HashMap<>()); // 性能较差
```

**相关原理**：[[Java集合框架#ConcurrentHashMap]]

---

### 问题6：类加载器相关问题 `#类加载问题`
**现象**：
- ClassNotFoundException
- NoClassDefFoundError
- ClassCastException

**原因**：
- 类路径配置错误
- 类加载器层次问题
- 版本冲突

**解决**：
1. 检查类路径配置
2. 使用-verbose:class查看类加载过程
3. 解决jar包版本冲突
4. 理解双亲委派模型

**相关原理**：[[Java虚拟机#类加载机制]]

---

### 问题7：字符串常量池问题 `#内存问题`
**现象**：
- 字符串占用内存过多
- 永久代溢出（JDK 1.7之前）

**原因**：
- 大量使用String.intern()
- 动态生成大量字符串

**解决**：
1. 避免不必要的intern()调用
2. 使用StringBuilder进行字符串拼接
3. 合理使用字符串缓存

```java
// 字符串优化
StringBuilder sb = new StringBuilder();
for (String str : strings) {
    sb.append(str); // 避免使用 + 拼接
}
String result = sb.toString();
```

**相关原理**：[[Java虚拟机#运行时常量池]]

## 🔧 调试技巧

### 常用调试方法

#### 1. 日志调试
```java
// 使用日志框架
private static final Logger logger = LoggerFactory.getLogger(MyClass.class);

public void debugMethod() {
    logger.debug("方法开始执行，参数：{}", param);
    try {
        // 业务逻辑
        logger.info("业务逻辑执行成功");
    } catch (Exception e) {
        logger.error("业务逻辑执行失败", e);
    }
}
```

#### 2. 断点调试
- 设置条件断点：只在特定条件下停止
- 设置异常断点：在抛出异常时停止
- 使用表达式求值：在断点处计算表达式值

#### 3. 远程调试
```bash
# JVM启动参数
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005
```

### 性能分析工具

#### 1. JVM自带工具
```bash
# 进程监控
jps -l                    # 列出Java进程
jinfo <pid>               # 查看JVM参数

# 内存分析
jmap -histo <pid>         # 查看对象统计信息
jmap -dump:format=b,file=heap.hprof <pid> # 生成堆转储

# 线程分析
jstack <pid>              # 查看线程堆栈
jstat -gc <pid> 1000      # 监控GC情况
```

#### 2. 第三方工具
- **MAT**：内存分析工具，分析heap dump
- **JProfiler**：商业性能分析工具
- **Arthas**：阿里开源的在线诊断工具
- **VisualVM**：可视化性能分析工具

### 代码审查要点

#### 1. 内存泄漏检查
- 集合类是否及时清理
- 监听器是否正确移除
- 缓存是否有过期机制
- 静态变量是否合理使用

#### 2. 线程安全检查
- 共享变量是否正确同步
- 是否存在竞态条件
- 锁的粒度是否合适
- 是否存在死锁风险

#### 3. 性能优化检查
- 是否存在不必要的对象创建
- 循环中是否有重复计算
- 数据库查询是否优化
- 缓存策略是否合理

## 🔗 相关文档
- **技术原理**：[[Java虚拟机]]、[[Java并发编程]]、[[Java集合框架]]
- **实战应用**：[[Java基础语法]]

## 🏷️ 标签
#问题解决 #Java #调试 #性能优化 #内存问题 #并发问题