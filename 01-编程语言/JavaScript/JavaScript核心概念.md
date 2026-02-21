# JavaScript - 核心概念

---
tags: [JavaScript, 核心概念, 前端开发, 编程语言]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> JavaScript语言的核心概念和基础知识

- **语言特性**：动态类型、解释执行、基于原型的面向对象
- **执行环境**：浏览器和Node.js运行时环境
- **异步编程**：事件循环、Promise、async/await
- **模块系统**：CommonJS、ES6 Modules

## 💡 原理详解

### 1. 基础概念

JavaScript是一种高级的、解释型的编程语言，具有以下特点：

- **动态类型**：变量类型在运行时确定
- **弱类型**：允许隐式类型转换
- **基于原型**：对象继承基于原型链
- **函数式编程**：支持高阶函数、闭包等特性

### 2. 数据类型

#### 基本数据类型
- `undefined`：未定义
- `null`：空值
- `boolean`：布尔值
- `number`：数字
- `string`：字符串
- `symbol`：符号（ES6）
- `bigint`：大整数（ES2020）

#### 引用数据类型
- `Object`：对象
- `Array`：数组
- `Function`：函数
- `Date`：日期
- `RegExp`：正则表达式

### 3. 作用域和闭包

#### 作用域链
```javascript
var globalVar = "全局变量";

function outerFunction() {
    var outerVar = "外部变量";

    function innerFunction() {
        var innerVar = "内部变量";
        console.log(globalVar); // 可访问
        console.log(outerVar);  // 可访问
        console.log(innerVar);  // 可访问
    }

    return innerFunction;
}
```

#### 闭包
```javascript
function createCounter() {
    let count = 0;

    return function() {
        count++;
        return count;
    };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
```

## 🔧 代码示例

### 基础用法

#### 变量声明
```javascript
// var - 函数作用域
var name = "张三";

// let - 块级作用域
let age = 25;

// const - 常量
const PI = 3.14159;
```

#### 函数定义
```javascript
// 函数声明
function greet(name) {
    return `Hello, ${name}!`;
}

// 函数表达式
const greet2 = function(name) {
    return `Hello, ${name}!`;
};

// 箭头函数
const greet3 = (name) => `Hello, ${name}!`;
```

### 高级用法

#### 对象和原型
```javascript
// 构造函数
function Person(name, age) {
    this.name = name;
    this.age = age;
}

Person.prototype.sayHello = function() {
    return `我是${this.name}，今年${this.age}岁`;
};

// ES6 类语法
class PersonES6 {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }

    sayHello() {
        return `我是${this.name}，今年${this.age}岁`;
    }
}
```

#### 数组操作
```javascript
const numbers = [1, 2, 3, 4, 5];

// 高阶函数
const doubled = numbers.map(n => n * 2);
const evens = numbers.filter(n => n % 2 === 0);
const sum = numbers.reduce((acc, n) => acc + n, 0);

// 解构赋值
const [first, second, ...rest] = numbers;
```

## ⚡ 性能特点

| 特性 | 说明 | 适用场景 |
|------|------|----------|
| 解释执行 | 无需编译，直接执行 | 快速开发和调试 |
| 垃圾回收 | 自动内存管理 | 减少内存泄漏风险 |
| 事件驱动 | 非阻塞I/O | 高并发Web应用 |
| JIT编译 | 运行时优化 | 提升执行性能 |

## 🔗 知识关联

- **前置知识**：[[HTML基础]] [[CSS基础]]
- **相关技术**：[[Node.js基础]] [[TypeScript]]
- **实战应用**：[[React核心概念]] [[Vue.js开发]]
- **问题解决**：[[JavaScript问题解决]]

## 🏷️ 标签
#JavaScript #核心概念 #前端开发 #编程语言 #面试重点