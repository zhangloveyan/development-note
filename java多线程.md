# JUC 并发编程

## 回顾

### 0. java8 函数式编程

runnable、function、consumer（BiConsumer）、supplier

参数、返回，用一个接口代替（接口只能有一个方法）

**作为参数：**

```java
interface DemoI {
    void eat();
}

public static void anyMeth(DemoI demoI) {
    demoI.eat();
}

public static void main(String[] args) {
    anyMeth(new DemoI() {
        @Override
        public void eat() {
		
        }
    });
    // 作为参数
    anyMeth(() -> {

    });
}
```

**作为返回：**

```java
public static DemoI anyMeth2(){
    return new DemoI() {
        @Override
        public void eat() {

        }
    };
}

public static DemoI anyMeth3(){
    // 作为返回
    return () -> {

    };
}
```

#### 1. Supplier

用来获取一个泛型参数指定类型的对象数据

该接口被称为生产型接口，指定接口的泛型是什么类型，那么get方法就会产生什么类型的数据

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

作用入参时，可以是一个方法，方法中返回泛型类

```java
public static void main(String[] args) {
    getS(() -> {
        String s = "adb";
        return s + "234";
    });
}

public static String getS(Supplier<String> supplier) {
    return supplier.get();
}
```

#### 2. Consumer

它不是生产一个数据，而是消费一个数据，其数据类型由指定的泛型决定

该接口包含抽象方法`void accept(T t)`，作用是消费一个指定泛型的数据，消费是自定义的(输出、计算....)

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

#### 3. Predicate

当我们需要对某种类型的数据进行判断，从而得到一个boolean值结果，这时可以使用`java.util.function.Predicate<T>` 接口

```java
@FunctionalInterface
public interface Predicate<T> {

    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }

    @SuppressWarnings("unchecked")
    static <T> Predicate<T> not(Predicate<? super T> target) {
        Objects.requireNonNull(target);
        return (Predicate<T>)target.negate();
    }
}
```

#### 4. Function

`java.util.function.Function<T,R>` 接口为一个转换类型的接口，用来根据一个类型的数据得到另一个类型的数据，T 称为前置条件，R 称为后置条件

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```



### 1. thread.start() 解析

**最基本的开启线程**

```java
Thread thread = new Thread(() -> {
    // do something...
});
thread.start();
```

start 方法中，最终调用 native 方法开启线程

```java
private native void start0();
```

**native 方法实现**

openjdk 源码地址：https://hg.openjdk.org/

涉及到的类：

![image-20231016111632085](pic/image-20231016111632085.png)

在线地址：

https://hg.openjdk.org/jdk8u/jdk8u/jdk/file/7fcf35286d52/src/share/native/java/lang/Thread.c

https://hg.openjdk.org/jdk8u/jdk8u/hotspot/file/69087d08d473/src/share/vm/prims/jvm.cpp

https://hg.openjdk.org/jdk8u/jdk8u/hotspot/file/69087d08d473/src/share/vm/runtime/thread.cpp

JNI 一般与 java 文件一一对应，Thread.java 对应的就是 Thread.c，在 Thread.c 中找打对应方法 JVM_StartThread

![image-20231016114114561](pic/image-20231016114114561.png)

JVM_StartThread具体实现在 jvm.cpp 中

![image-20231016114615838](pic/image-20231016114615838.png)

方法最后调用 thread.cpp 中 的 Thread::start 方法

![image-20231016114655950](pic/image-20231016114655950.png)

thread.cpp 中 的 Thread::start 方法，最终调用系统开启线程

![image-20231016114811675](pic/image-20231016114811675.png)

整体图：

![image-20231016115656439](pic/image-20231016115656439.png)

### 2. 常问

1. 核心线程、最大线程、队列
2. 任务满了的拒绝模式
3. 创建线程池的方法

## 线程方法

### 1. Runnable



### 2. FutureTask

```java
// 创建线程池
ExecutorService threadPool = Executors.newFixedThreadPool(3);

FutureTask<String> task1 = new FutureTask<>(() -> {
    TimeUnit.MILLISECONDS.sleep(200);
    return "task1 over";
});
// 提交
threadPool.submit(task1);
System.out.println(task1.get());
// 销毁
threadPool.shutdown();
```

**缺点**

**1.get() 会阻塞其他线程**

```java
// 第一种
threadPool.submit(task1);
System.out.println(task1.get());

threadPool.submit(task2);
System.out.println(task2.get());

vs
// 第二种
threadPool.submit(task1);
threadPool.submit(task2);

System.out.println(task1.get());
System.out.println(task2.get());    
```

第一种：时间是 task1+task2

第二种：时间是 max（task1，task2）

如果不愿意等待，可以添加超时设置，到时直接抛异常（TimeoutException）

```java
task3.get(3,TimeUnit.SECONDS)
```

![image-20231020115106599](pic/image-20231020115106599.png)

**2.容易造成 cpu 空转**

实际中，并不会直接会用 get()，会轮询判断 task 是否完成（task.isDone()），完成后在调用 get() 获取值。

但这种情况需要定时去查询是否完成，造成 cpu 性能浪费

```
// task3 5s 执行完成
// 每 0.5s 查看是否完成
while (true) {
    if (task3.isDone()) {
        System.out.println(task3.get());
        break;
    } else {
        TimeUnit.MILLISECONDS.sleep(500);
        System.out.println("正在处理中");
    }
}
```

![image-20231020115320464](pic/image-20231020115320464.png)

### 3. CompletableFuture

基于 FutureTask 缺点，以及无法处理复杂组合任务（有先后依赖关系），JDK8 设计出 CompletableFuture。提供观察者模式，完成后可以通知监听的一方。

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> 
```

CompletionStage：

- 代表异步计算的某一个阶段，一个完成后触发另一个
- 一个阶段的计算执行可以是一个 Function、Consumer、Runnable
- 可以单个触发，也可以多个阶段一起触发

#### 3.1 四大静态方法调用

1. runAsync 无返回值（是否传入线程池）
   - public static CompletableFuture<Void> runAsync(Runnable runnable)
   - public static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor)
2. supplyAsync 有返回值（是否传入线程池）
   - public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
   - public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor)

不传入线程池时，默认使用 ForkJoinPool 线程池。默认创建的线程数是 CPU 的核数（可通过 JVM option:-Djava.util.concurrent.ForkJoinPool.common.parallelism 设置线程数）。

**main结束 会自动关闭，切记！**

```
import java.util.concurrent.*;

ExecutorService threadPool = Executors.newFixedThreadPool(3);
CompletableFuture<Void> runAsync = CompletableFuture.runAsync(() -> {
    System.out.println(Thread.currentThread().getName());
    try {
        TimeUnit.MILLISECONDS.sleep(200);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}, threadPool);
System.out.println(runAsync.get());

CompletableFuture<String> supplyAsync = CompletableFuture.supplyAsync(() -> {
    System.out.println(Thread.currentThread().getName());
    try {
        TimeUnit.MILLISECONDS.sleep(200);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "supplyAsync";
});
System.out.println(supplyAsync.get());
```

#### 3.2 任务异步回调

![CompletableFuture异步回调](pic/CompletableFuture异步回调.png)

**不传、无返 thenRun/thenRunAsync**

**传参、无返 thenAccept/thenAcceptAsync**

**传参、有返 thenApply/thenApplyAsync**

**额外处理、有返 whenComplete**

**额外处理、无返 handle**

**异常处理 exceptionally**

**有无 async 区别：**

**无 async 共用同一个线程池**

**有 async 第一个用传入，第二个用ForkJoin**

#### 3.3 任务异步组合

![CompletableFuture多任务组合](pic/CompletableFuture多任务组合.png)

#### 参考：

[异步编程利器：CompletableFuture详解 ｜Java 开发实战](https://juejin.cn/post/6970558076642394142)

#### 3.4 常用方法

1. 获得结果和触发计算
   - 获取结果：
     - T get：阻塞，可选超时方法
     - T join：
     - T getNow（T valueIfAbsent）：立即获取结果，不阻塞，没算完给传入值
   - 计算：
     - boolean complete（T value）：告诉是否计算完，没算完给传入值
2. 对计算结果进行处理，串行化
   - thenApply
   - handle
3. 对计算结果进行消费，无返回
4. 对计算结果进行选用
5. 对计算结果进行合并

## 锁

### 1. synchronized

修饰 方法、静态方法、代码块

锁当前对象(object)、锁类模板(.class文件)、锁指定对象(object)

.class 只有一份，锁的最高，不管你 new 多少对象，都是可以锁

object 同一个对象是锁一个，但是 new 多个时，不同对象无法相互锁

```
// -c 代码反汇编
javap -c *.class
// -v verbose 输出附加信息
javap -p *.class
```

### 2. 对象头

![object_header.png](pic/5c081fb4576641eaa63e4966703845eftplv-k3u1fbpfcp-zoom-in-crop-mark1512000.awebp)

### 3. monitor 对象

每一个实例都会关联一个 monitor 对象，这个对象可以与对象一起创建销毁，也可以在线程获取锁的时候生成。monitor 对象被持有的时候，便处于锁定状态。

在HotSpot虚拟机中，Monitor是由 [ObjectMonitor](https://link.juejin.cn/?target=https%3A%2F%2Fhg.openjdk.java.net%2Fjdk8u%2Fjdk8u%2Fhotspot%2Ffile%2F782f3b88b5ba%2Fsrc%2Fshare%2Fvm%2Fruntime%2FobjectMonitor.hpp) 实现的,它是一个使用C++实现的类，主要数据结构如下：

```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;  // 线程重入次数
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; // 调用wait方法后的线程会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; // 阻塞队列，线程被唤醒后根据决策判读是放入cxq还是EntryList
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 没有抢到锁的线程会被放到这个队列
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

ObjectMonitor 中有五个重要部分，分别为 _ower, _WaitSet, _cxq, _EntryList 和 count。

- **_ower** 用来指向持有monitor的线程，它的初始值为NULL,表示当前没有任何线程持有monitor。当一个线程成功持有该锁之后会保存线程的ID标识，等到线程释放锁后_ower又会被重置为NULL;
- **_WaitSet** 调用了锁对象的wait方法后的线程会被加入到这个队列中；
- **_cxq**  是一个阻塞队列，线程被唤醒后根据决策判读是放入cxq还是EntryList;
- **_EntryList** 没有抢到锁的线程会被放到这个队列；
- **count** 用于记录线程获取锁的次数，成功获取到锁后count会加1，释放锁时count减1。

![image-20231121194958179](pic/image-20231121194958179.png)

> 如果线程获取到对象的monitor后，就会将monitor中的ower设置为该线程的ID，同时monitor中的count进行加1. 如果调用锁对象的wait()方法，线程会释放当前持有的monitor，并将owner变量重置为NULL，且count减1,同时该线程会进入到_WaitSet集合中等待被唤醒。
>
> 另外_WaitSet，_cxq与_EntryList都是链表结构的队列，存放的是封装了线程的ObjectWaiter对象。如果不深入虚拟机查看相关源码很难理解这几个队列的作用，关于源码会在后边系列文章中分析。这里我简单说下它们之间的关系，如下：
>
> 在多条线程竞争monitor锁的时候，所有没有竞争到锁的线程会被封装成ObjectWaiter并加入_EntryList队列。 当一个已经获取到锁的线程，调用锁对象的wait方法后，线程也会被封装成一个ObjectWaiter并加入到_WaitSet队列中。 当调用锁对象的notify方法后，会根据不同的情况来决定是将_WaitSet集合中的元素转移到_cxq队列还是_EntryList队列。 等到获得锁的线程释放锁后，又会根据条件来执行_EntryList中的线程或者将_cxq转移到_EntryList中再执行_EntryList中的线程。
>
> 所以，可以看得出来，_WaitSet存放的是处于WAITING状态等待被唤醒的线程。而_EntryList队列中存放的是等待锁的BLOCKED状态。_cxq队列仅仅是临时存放，最终还是会被转移到_EntryList中等待获取锁。
>
> 链接：https://juejin.cn/post/6973571891915128846

#### 底层原理

**同步代码块**

```
public void add() {
    synchronized (this) {
        i++;
    }
}
```

反编译字节码：

```
public class com.zhangpan.text.TestSync {
  public com.zhangpan.text.TestSync();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public void add();
    Code:
       0: aload_0
       1: dup
       2: astore_1
       3: monitorenter    // synchronized关键字的入口
       4: getstatic     #2                  // Field i:I
       7: iconst_1
       8: iadd
       9: putstatic     #2                  // Field i:I
      12: aload_1
      13: monitorexit  // synchronized关键字的出口
      14: goto          22
      17: astore_2
      18: aload_1
      19: monitorexit // synchronized关键字的出口
      20: aload_2
      21: athrow
      22: return
    Exception table:
       from    to  target type
           4    14    17   any
          17    20    17   any
}
```

monitorenter 和 moniterexit 两条指令

当执行到monitorenter指令时，线程就会去尝试获取该对象对应的Monitor的所有权，即尝试获得该对象的锁。

当该对象的 monitor 的计数器count为0时，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有该对象monitor的持有权，那它可以重入这个 monitor ，计数器的值也会加 1。而当执行monitorexit指令时，锁的计数器会减1。

倘若其他线程已经拥有monitor 的所有权，那么当前线程获取锁失败将被阻塞并进入到_EntryList中，直到等待的锁被释放为止。也就是说，当所有相应的monitorexit指令都被执行，计数器的值减为0，执行线程将释放 monitor(锁)，其他线程才有机会持有 monitor 。

**同步方法**

```
public synchronized void add(){
       i++;
}
```

反编译字节码：

```
public synchronized void add();
    descriptor: ()V
    flags: (0x0021) ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 5: 0
        line 6: 10
```

方法的 flag 上加入了 ACC_SYNCHRONIZED 的标记位。如果有的话则会尝试获取 monitor 对象锁。

### 4. ReentrantLock 非公平锁

```
// 默认非公平锁 true 公平
ReentrantLock lock = new ReentrantLock(true);
```

为什么默认非公平锁？（核心 减少切换，省cpu时间，减少其他开销）

1. 线程恢复挂起有时间差，不公平减少恢复挂起，可以较少 cpu 空闲
2. 线程切换也需要开销，不公平不需要来回切换，减少开销

**可重入锁**

synchronized 隐式重入，又计数器计数

reentrantLock 显示重入，需要自己加解锁

### 5. synchronized 和 Lock的区别？

- Lock是显示锁，需要手动开启和关闭。synchronized是隐士锁，可以自动释放锁。
- Lock是一个接口，是JDK实现的。synchronized是一个关键字，是依赖JVM实现的。
- Lock是可中断锁，synchronized是不可中断锁，需要线程执行完才能释放锁。
- 发生异常时，Lock不会主动释放占有的锁，必须通过unlock进行手动释放，因此可能引发死锁。synchronized在发生异常时会自动释放占有的锁，不会出现死锁的情况。
- Lock可以判断锁的状态，synchronized不可以判断锁的状态。
- Lock实现锁的类型是可重入锁、公平锁。synchronized 实现锁的类型是可重入锁，非公平锁。
- Lock适用于大量同步代码块的场景，synchronized适用于少量同步代码块的场景。

### 6. 死锁

本质上是两个线程持有各自的锁，在没有释放的时候都想要获取对方的锁，进入等待状态。

手写一个死锁的例子：

```java
public class DeathLock {
    public static void main(String[] args) {
        // 锁1
        Object object1 = new Object();
        // 锁2
        Object object2 = new Object();

        new Thread(() -> {
            synchronized (object1) {
                System.out.println(Thread.currentThread().getName() + " 持有 obj1 希望获取 obj2");
                synchronized (object2) {
                    System.out.println(Thread.currentThread().getName() + "成功获取 obj2");
                }
            }
        }, "t1").start();
        new Thread(() -> {
            synchronized (object2) {
                System.out.println(Thread.currentThread().getName() + " 持有 obj2 希望获取 obj1");
                synchronized (object1) {
                    System.out.println(Thread.currentThread().getName() + "成功获取 obj1");
                }
            }
        }, "t2").start();
    }
}
```

当发生死锁时，排查方法：

```
// JVM Process Status Tool 获取当前所有 java 进程 pid 命令
jps -l
```

其他方式获取

```
top | grep java
ps -ef | grep java
```

![image-20231122094602663](pic/image-20231122094602663.png)

```
//jstack pid 查看堆栈
jstack 15180
```

![image-20231122094724132](pic/image-20231122094724132.png)



### 5. 锁优化

因为Java虚拟机是通过进入和退出Monitor对象来实现代码块同步和方法同步的，而Monitor是依靠底层操作系统的`Mutex Lock`来实现的，操作系统实现线程之间的切换需要从用户态转换到内核态，这个切换成本比较高，对性能影响较大
