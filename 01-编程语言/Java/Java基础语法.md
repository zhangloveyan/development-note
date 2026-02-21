# Java基础语法

---
tags: [Java, 基础语法, 数据类型, 面向对象, 泛型, 反射, 注解]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Java语言的基础语法和核心概念

- **数据类型**：8种基本数据类型和引用数据类型
- **面向对象**：封装、继承、多态三大特性
- **泛型机制**：类型安全和泛型擦除
- **反射机制**：运行时获取类信息和动态调用
- **注解系统**：元数据标记和处理机制

## 💡 原理详解

### 1. 基础定义

#### 数据存储单位
- **bit（比特）**：二进制的1位，0或1
- **byte（字节）**：8位，1byte=8bit，英文1B，中文2B
- **范围**：-2^7 ~ 2^7-1，最前位是符号位

#### 基本数据类型
| 类型 | 位数 | 范围 | 默认值 |
|------|------|------|--------|
| byte | 8位 | -128 ~ 127 | 0 |
| short | 16位 | -32768 ~ 32767 | 0 |
| char | 16位 | 0 ~ 65535 | '\u0000' |
| int | 32位 | -2^31 ~ 2^31-1 | 0 |
| float | 32位 | IEEE 754 | 0.0f |
| long | 64位 | -2^63 ~ 2^63-1 | 0L |
| double | 64位 | IEEE 754 | 0.0d |
| boolean | 1位 | true/false | false |

### 2. 面向对象

#### 三大特性

##### 封装（Encapsulation）
- **定义**：将数据和操作数据的方法绑定在一起
- **实现**：通过访问修饰符控制访问权限
- **好处**：隐藏内部实现，提供稳定接口

```java
public class Person {
    private String name; // 私有属性

    public String getName() { // 公共方法
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

##### 继承（Inheritance）
- **定义**：子类继承父类的属性和方法
- **关键字**：extends（类继承）、implements（接口实现）
- **特点**：单继承、多层继承、多接口实现

```java
public class Animal {
    protected String name;

    public void eat() {
        System.out.println("Animal eating");
    }
}

public class Dog extends Animal {
    @Override
    public void eat() {
        System.out.println("Dog eating");
    }
}
```

##### 多态（Polymorphism）
- **定义**：同一接口的不同实现
- **实现**：方法重写、接口实现
- **条件**：继承关系、方法重写、父类引用指向子类对象

```java
Animal animal = new Dog(); // 父类引用指向子类对象
animal.eat(); // 调用子类重写的方法
```

#### 访问修饰符
| 修饰符 | 同类 | 同包 | 子类 | 其他 |
|--------|------|------|------|------|
| private | ✓ | ✗ | ✗ | ✗ |
| default | ✓ | ✓ | ✗ | ✗ |
| protected | ✓ | ✓ | ✓ | ✗ |
| public | ✓ | ✓ | ✓ | ✓ |

### 3. 泛型机制

#### 泛型擦除
- **编译时**：进行类型检查，确保类型安全
- **运行时**：擦除泛型信息，使用原始类型
- **问题**：运行时无法获取泛型的具体类型

#### 泛型保留
Java在编译时会在字节码里指令集之外的地方保留**部分**泛型信息：
- 泛型接口、类、方法定义上的所有泛型**会被保留**
- 成员变量声明处的泛型**会被保留**
- **其它地方**的泛型信息都会被擦除

#### TypeReference解决方案
```java
// Jackson提供的TypeReference
TypeReference<List<User>> typeRef = new TypeReference<List<User>>() {};
List<User> users = objectMapper.readValue(json, typeRef);
```

#### 泛型通配符
```java
// 上界通配符
List<? extends Number> numbers = new ArrayList<Integer>();

// 下界通配符
List<? super Integer> integers = new ArrayList<Number>();

// 无界通配符
List<?> objects = new ArrayList<String>();
```

### 4. 反射机制

#### 核心类
- **Class**：类的字节码对象
- **Field**：字段对象
- **Method**：方法对象
- **Constructor**：构造器对象

#### 获取Class对象
```java
// 1. 通过对象获取
Class<?> clazz1 = obj.getClass();

// 2. 通过类名获取
Class<?> clazz2 = String.class;

// 3. 通过Class.forName()获取
Class<?> clazz3 = Class.forName("java.lang.String");
```

#### 反射操作
```java
Class<?> clazz = Person.class;

// 获取构造器
Constructor<?> constructor = clazz.getConstructor(String.class);
Object instance = constructor.newInstance("张三");

// 获取字段
Field field = clazz.getDeclaredField("name");
field.setAccessible(true); // 设置可访问私有字段
field.set(instance, "李四");

// 获取方法
Method method = clazz.getMethod("getName");
Object result = method.invoke(instance);
```

### 5. 注解系统

#### 元注解
- **@Target**：指定注解的使用位置
- **@Retention**：指定注解的保留策略
- **@Documented**：指定注解是否包含在JavaDoc中
- **@Inherited**：指定注解是否可以被继承

#### 保留策略
```java
public enum RetentionPolicy {
    SOURCE,    // 源码级别，编译时丢弃
    CLASS,     // 字节码级别，运行时丢弃
    RUNTIME    // 运行时级别，可通过反射获取
}
```

#### 自定义注解
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
    String value() default "";
    int count() default 1;
}

// 使用注解
@MyAnnotation(value = "test", count = 3)
public void testMethod() {
    // 方法实现
}
```

#### 注解处理
```java
Method method = clazz.getMethod("testMethod");
if (method.isAnnotationPresent(MyAnnotation.class)) {
    MyAnnotation annotation = method.getAnnotation(MyAnnotation.class);
    String value = annotation.value();
    int count = annotation.count();
}
```

### 6. 异常处理

#### 异常层次结构
```
Throwable
├── Error (系统错误，不应捕获)
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception (程序异常)
    ├── RuntimeException (运行时异常，非检查异常)
    │   ├── NullPointerException
    │   ├── IndexOutOfBoundsException
    │   └── IllegalArgumentException
    └── 检查异常
        ├── IOException
        ├── SQLException
        └── ClassNotFoundException
```

#### 异常处理机制
```java
try {
    // 可能抛出异常的代码
    riskyOperation();
} catch (SpecificException e) {
    // 处理特定异常
    handleSpecificException(e);
} catch (Exception e) {
    // 处理通用异常
    handleGenericException(e);
} finally {
    // 无论是否异常都会执行
    cleanup();
}
```

### 7. 函数式编程（Java 8+）

#### Lambda表达式
```java
// 传统写法
Comparator<String> comparator = new Comparator<String>() {
    @Override
    public int compare(String s1, String s2) {
        return s1.compareTo(s2);
    }
};

// Lambda写法
Comparator<String> lambdaComparator = (s1, s2) -> s1.compareTo(s2);
```

#### 函数式接口
```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}

@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}

@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}

@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

#### Stream API
```java
List<String> names = Arrays.asList("张三", "李四", "王五");

// 过滤、转换、收集
List<String> result = names.stream()
    .filter(name -> name.length() > 2)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
```

## 🔧 代码示例

### 基本语法示例
```java
public class BasicSyntax {
    // 成员变量
    private int count = 0;
    private static final String CONSTANT = "常量";

    // 构造方法
    public BasicSyntax() {
        this.count = 0;
    }

    public BasicSyntax(int count) {
        this.count = count;
    }

    // 实例方法
    public void increment() {
        this.count++;
    }

    // 静态方法
    public static void staticMethod() {
        System.out.println("静态方法");
    }

    // getter/setter
    public int getCount() {
        return count;
    }

    public void setCount(int count) {
        this.count = count;
    }
}
```

### 继承和多态示例
```java
// 抽象类
abstract class Shape {
    protected String color;

    public Shape(String color) {
        this.color = color;
    }

    // 抽象方法
    public abstract double getArea();

    // 具体方法
    public void display() {
        System.out.println("颜色: " + color + ", 面积: " + getArea());
    }
}

// 具体实现类
class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double getArea() {
        return Math.PI * radius * radius;
    }
}

// 接口
interface Drawable {
    void draw();
}

// 实现接口
class Rectangle extends Shape implements Drawable {
    private double width, height;

    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override
    public double getArea() {
        return width * height;
    }

    @Override
    public void draw() {
        System.out.println("绘制矩形");
    }
}
```

## ⚡ 性能特点

| 特性 | 优势 | 注意事项 |
|------|------|----------|
| 封装 | 代码安全、易维护 | 可能影响性能 |
| 继承 | 代码复用、扩展性好 | 耦合度高 |
| 多态 | 灵活性强、可扩展 | 运行时开销 |
| 泛型 | 类型安全、无需强转 | 泛型擦除限制 |
| 反射 | 动态性强、框架基础 | 性能开销大 |
| 注解 | 元数据丰富、配置简化 | 编译时处理复杂 |

## 🔗 知识关联
- **相关技术**：[[Java集合框架#泛型应用]]
- **实战应用**：[[Java并发编程#面向对象设计]]
- **高级特性**：[[Java虚拟机#对象创建过程]]
- **问题解决**：[[Java问题解决#基础语法问题]]

## 🏷️ 标签
#Java #基础语法 #数据类型 #面向对象 #泛型 #反射 #注解