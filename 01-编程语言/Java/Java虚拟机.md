# Java虚拟机

---
tags: [Java, JVM, 垃圾回收, 内存模型, GC调优, 类加载, 字节码]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Java虚拟机的内存管理和垃圾回收机制

- **内存模型**：堆、栈、方法区、程序计数器、本地方法栈五大区域
- **垃圾回收**：标记清除、复制、标记整理三大算法
- **垃圾回收器**：Serial、Parallel、CMS、G1各有特点
- **内存调优**：通过JVM参数和监控工具进行性能优化
- **对象生命周期**：从创建到回收的完整过程

## 💡 原理详解

### 1. JVM内存模型演变

#### JDK 1.6内存模型
- **堆**：新生代（Eden + Survivor） + 老年代
- **方法区**：永久代（PermGen）
- **栈**：虚拟机栈 + 本地方法栈
- **程序计数器**：当前线程执行字节码位置

#### JDK 1.7内存模型
- 字符串常量池从永久代移到堆中
- 符号引用从永久代移到堆中

#### JDK 1.8内存模型
- **移除永久代**：使用元空间（Metaspace）替代
- **元空间**：使用本地内存，不再受JVM堆大小限制
- **直接内存**：NIO操作使用的堆外内存

### 2. 运行时常量池

#### 什么是运行时常量池？
- Class文件中的常量池表在运行时的表示
- 存放编译期生成的各种字面量和符号引用
- 具备动态性，运行期间可以将新的常量放入池中

#### 字符串常量池
- JDK 1.7之前在永久代
- JDK 1.7及之后在堆中
- String.intern()方法的行为变化

### 3. 垃圾回收

#### 垃圾判断算法

##### 引用计数法
- 原理：对象被引用时计数+1，引用失效时计数-1
- 问题：无法解决循环引用问题
- 应用：Python、COM、ActionScript等

##### 可达性分析算法
- 原理：从GC Roots开始向下搜索，不可达对象即为垃圾
- GC Roots包括：
  - 虚拟机栈中引用的对象
  - 方法区中类静态属性引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中JNI引用的对象

#### 垃圾回收算法

##### 标记-清除算法
- **过程**：标记垃圾对象 → 清除垃圾对象
- **优点**：简单直接
- **缺点**：产生内存碎片，效率不高

##### 复制算法
- **过程**：将内存分为两块，将存活对象复制到另一块
- **优点**：效率高，无内存碎片
- **缺点**：内存利用率只有50%
- **应用**：新生代回收

##### 标记-整理算法
- **过程**：标记垃圾对象 → 将存活对象向一端移动 → 清理边界外内存
- **优点**：无内存碎片，内存利用率高
- **缺点**：效率较低
- **应用**：老年代回收

##### 分代收集算法
- **新生代**：使用复制算法（Eden:Survivor1:Survivor2 = 8:1:1）
- **老年代**：使用标记-清除或标记-整理算法

#### 垃圾回收器分类

##### Serial收集器
- **特点**：单线程，Stop The World
- **适用**：客户端模式，小内存应用
- **参数**：-XX:+UseSerialGC

##### Parallel收集器
- **特点**：多线程并行收集
- **适用**：服务器模式，注重吞吐量
- **参数**：-XX:+UseParallelGC

##### CMS收集器
- **特点**：并发收集，低延迟
- **过程**：初始标记 → 并发标记 → 重新标记 → 并发清除
- **问题**：内存碎片、浮动垃圾、CPU敏感
- **参数**：-XX:+UseConcMarkSweepGC

##### G1收集器
- **特点**：低延迟，可预测停顿时间
- **原理**：将堆分为多个Region，优先回收价值最大的Region
- **适用**：大内存应用，要求低延迟
- **参数**：-XX:+UseG1GC

### 4. 对象内存布局

#### 对象组成
1. **对象头**：Mark Word + 类型指针
2. **实例数据**：对象真正存储的有效信息
3. **对齐填充**：保证对象大小是8字节的整数倍

#### 对象头详解

##### Mark Word（64位）
- **无锁状态**：hashcode(31) + age(4) + biased_lock(1) + lock(2)
- **偏向锁**：thread_id(54) + epoch(2) + age(4) + biased_lock(1) + lock(2)
- **轻量锁**：ptr_to_lock_record(62) + lock(2)
- **重量锁**：ptr_to_heavyweight_monitor(62) + lock(2)
- **GC标记**：空(62) + lock(2)

##### 类型指针
- 指向对象的类元数据的指针
- 虚拟机通过这个指针确定对象是哪个类的实例

### 5. 锁升级过程

#### 偏向锁
- **条件**：只有一个线程访问同步块
- **原理**：在对象头记录线程ID，后续该线程无需CAS操作

#### 轻量锁
- **条件**：多个线程交替访问同步块
- **原理**：使用CAS操作避免重量锁的开销

#### 重量锁
- **条件**：多个线程同时访问同步块
- **原理**：使用操作系统的互斥量实现

### 6. 引用类型

#### 强引用（Strong Reference）
- 普通的对象引用
- 只要强引用存在，垃圾收集器永远不会回收

#### 软引用（Soft Reference）
- 内存不足时会被回收
- 适用于内存敏感的缓存

#### 弱引用（Weak Reference）
- 下次垃圾收集时会被回收
- 适用于规范化映射

#### 虚引用（Phantom Reference）
- 无法通过虚引用获取对象实例
- 用于跟踪对象被垃圾收集的活动

## 🔧 代码示例

### 内存溢出示例
```java
// 堆内存溢出
List<Object> list = new ArrayList<>();
while(true) {
    list.add(new Object());
}

// 栈溢出
public void stackOverflow() {
    stackOverflow(); // 无限递归
}

// 方法区溢出（JDK 1.7及之前）
while(true) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(Object.class);
    enhancer.setUseCache(false);
    enhancer.setCallback(new MethodInterceptor() {
        public Object intercept(Object obj, Method method,
                              Object[] args, MethodProxy proxy) {
            return proxy.invokeSuper(obj, args);
        }
    });
    enhancer.create();
}
```

### GC参数配置
```bash
# 堆大小设置
-Xms2g -Xmx2g

# 新生代大小
-Xmn512m

# 垃圾收集器选择
-XX:+UseG1GC

# GC日志
-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps

# 内存溢出时dump
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path/to/dump
```

### 引用类型使用
```java
// 软引用缓存
Map<String, SoftReference<Object>> cache = new HashMap<>();
cache.put("key", new SoftReference<>(new Object()));

// 弱引用
WeakReference<Object> weakRef = new WeakReference<>(new Object());

// 虚引用
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), queue);
```

## ⚡ 性能调优

### 常用JVM参数

#### 内存设置
| 参数 | 说明 | 示例 |
|------|------|------|
| -Xms | 初始堆大小 | -Xms2g |
| -Xmx | 最大堆大小 | -Xmx4g |
| -Xmn | 新生代大小 | -Xmn1g |
| -XX:MetaspaceSize | 元空间初始大小 | -XX:MetaspaceSize=256m |

#### 垃圾收集器
| 参数 | 说明 |
|------|------|
| -XX:+UseSerialGC | 使用Serial收集器 |
| -XX:+UseParallelGC | 使用Parallel收集器 |
| -XX:+UseConcMarkSweepGC | 使用CMS收集器 |
| -XX:+UseG1GC | 使用G1收集器 |

### 监控工具

#### 命令行工具
- **jps**：查看Java进程
- **jstat**：查看GC统计信息
- **jmap**：查看内存使用情况
- **jstack**：查看线程堆栈信息

#### 图形化工具
- **JConsole**：JVM监控工具
- **JVisualVM**：性能分析工具
- **MAT**：内存分析工具
- **Arthas**：在线诊断工具

## 🔗 知识关联
- **前置知识**：[[Java基础语法#内存管理]]
- **相关技术**：[[Java并发编程#内存模型]]
- **实战应用**：[[Java问题解决#性能调优]]
- **监控工具**：[[Java问题解决#JVM监控]]

## 🏷️ 标签
#Java #JVM #垃圾回收 #内存模型 #GC调优 #类加载 #字节码