# JavaScript - 问题解决

---
tags: [问题解决, JavaScript, 调试, 性能优化, 最佳实践]
created: 2026-02-21
updated: 2026-02-21
status: 持续更新
importance: ⭐⭐⭐⭐
---

## 🚨 高频问题速查

### 问题1：变量提升导致的意外行为 `#变量提升`
**现象**：使用var声明的变量在声明前就能访问，值为undefined
**原因**：JavaScript引擎在执行前会将var声明提升到作用域顶部
**解决**：
1. 使用let/const替代var
2. 在使用前声明变量
3. 启用严格模式

```javascript
// 问题代码
console.log(name); // undefined，而不是报错
var name = "张三";

// 解决方案
const name = "张三";
console.log(name); // "张三"
```

**相关原理**：[[JavaScript核心概念#作用域和闭包]]

---

### 问题2：this指向混乱 `#this绑定`
**现象**：函数中的this指向不是预期的对象
**原因**：this的值取决于函数的调用方式，不是定义方式
**解决**：
1. 使用箭头函数
2. 使用bind/call/apply显式绑定
3. 保存this引用

```javascript
// 问题代码
const obj = {
    name: "对象",
    getName: function() {
        return this.name;
    }
};

const getName = obj.getName;
console.log(getName()); // undefined

// 解决方案1：箭头函数
const obj2 = {
    name: "对象",
    getName: () => this.name // 继承外层this
};

// 解决方案2：bind绑定
const boundGetName = obj.getName.bind(obj);
console.log(boundGetName()); // "对象"
```

**相关原理**：[[JavaScript核心概念#函数和this]]

---

### 问题3：异步操作中的闭包陷阱 `#闭包陷阱`
**现象**：循环中的异步操作都使用了最后一个循环变量的值
**原因**：闭包捕获的是变量的引用，不是值
**解决**：
1. 使用let替代var
2. 使用立即执行函数
3. 使用bind传参

```javascript
// 问题代码
for (var i = 0; i < 3; i++) {
    setTimeout(() => {
        console.log(i); // 输出三次3
    }, 100);
}

// 解决方案1：使用let
for (let i = 0; i < 3; i++) {
    setTimeout(() => {
        console.log(i); // 输出0, 1, 2
    }, 100);
}

// 解决方案2：立即执行函数
for (var i = 0; i < 3; i++) {
    (function(index) {
        setTimeout(() => {
            console.log(index); // 输出0, 1, 2
        }, 100);
    })(i);
}
```

**相关原理**：[[JavaScript异步编程#事件循环机制]]

---

### 问题4：内存泄漏 `#内存泄漏`
**现象**：应用运行时间长后内存占用持续增长
**原因**：未正确清理事件监听器、定时器、闭包引用等
**解决**：
1. 及时清理事件监听器
2. 清除定时器
3. 避免循环引用

```javascript
// 问题代码
function createHandler() {
    const largeData = new Array(1000000).fill('data');

    document.addEventListener('click', function() {
        // 闭包引用了largeData，导致无法回收
        console.log(largeData.length);
    });
}

// 解决方案
function createHandler() {
    const largeData = new Array(1000000).fill('data');
    const dataLength = largeData.length; // 只保存需要的值

    const handler = function() {
        console.log(dataLength);
    };

    document.addEventListener('click', handler);

    // 清理函数
    return function cleanup() {
        document.removeEventListener('click', handler);
    };
}
```

---

### 问题5：类型转换陷阱 `#类型转换`
**现象**：比较操作或运算结果不符合预期
**原因**：JavaScript的隐式类型转换规则复杂
**解决**：
1. 使用严格相等（===）
2. 显式类型转换
3. 使用类型检查

```javascript
// 问题代码
console.log(0 == ''); // true
console.log(0 == '0'); // true
console.log('' == '0'); // false
console.log([] + [] === ''); // true
console.log([] + {} === '[object Object]'); // true

// 解决方案
console.log(0 === ''); // false
console.log(Number('0') === 0); // true
console.log(String([]) === ''); // true

// 类型检查函数
function isString(value) {
    return typeof value === 'string';
}

function isArray(value) {
    return Array.isArray(value);
}
```

## 🔧 调试技巧

### 常用调试方法

#### 1. Console调试
```javascript
// 基础日志
console.log('普通日志');
console.warn('警告信息');
console.error('错误信息');

// 对象调试
const obj = { name: '张三', age: 25 };
console.table(obj); // 表格形式显示
console.dir(obj); // 详细对象信息

// 性能调试
console.time('操作耗时');
// 执行一些操作
console.timeEnd('操作耗时');

// 堆栈跟踪
console.trace('调用堆栈');
```

#### 2. 断点调试
```javascript
// 代码断点
function complexFunction(data) {
    debugger; // 浏览器会在此处暂停

    const result = data.map(item => {
        return processItem(item);
    });

    return result;
}

// 条件断点（在开发者工具中设置）
for (let i = 0; i < 100; i++) {
    if (i === 50) {
        // 只在i等于50时暂停
        console.log('到达断点');
    }
}
```

#### 3. 错误处理
```javascript
// 全局错误捕获
window.addEventListener('error', (event) => {
    console.error('全局错误:', event.error);
    // 发送错误报告到服务器
    reportError(event.error);
});

// Promise错误捕获
window.addEventListener('unhandledrejection', (event) => {
    console.error('未处理的Promise拒绝:', event.reason);
    event.preventDefault(); // 阻止默认的错误处理
});

// 自定义错误类
class CustomError extends Error {
    constructor(message, code) {
        super(message);
        this.name = 'CustomError';
        this.code = code;
    }
}

try {
    throw new CustomError('自定义错误', 'E001');
} catch (error) {
    if (error instanceof CustomError) {
        console.error(`错误代码: ${error.code}, 消息: ${error.message}`);
    }
}
```

### 性能分析工具

#### 1. Performance API
```javascript
// 测量函数执行时间
function measurePerformance(fn, name) {
    const start = performance.now();
    const result = fn();
    const end = performance.now();

    console.log(`${name} 执行时间: ${end - start} 毫秒`);
    return result;
}

// 内存使用情况
if (performance.memory) {
    console.log('已使用内存:', performance.memory.usedJSHeapSize);
    console.log('总内存限制:', performance.memory.totalJSHeapSize);
}
```

#### 2. 性能优化技巧
```javascript
// 防抖函数
function debounce(func, wait) {
    let timeout;
    return function executedFunction(...args) {
        const later = () => {
            clearTimeout(timeout);
            func(...args);
        };
        clearTimeout(timeout);
        timeout = setTimeout(later, wait);
    };
}

// 节流函数
function throttle(func, limit) {
    let inThrottle;
    return function(...args) {
        if (!inThrottle) {
            func.apply(this, args);
            inThrottle = true;
            setTimeout(() => inThrottle = false, limit);
        }
    };
}

// 使用示例
const debouncedSearch = debounce((query) => {
    console.log('搜索:', query);
}, 300);

const throttledScroll = throttle(() => {
    console.log('滚动事件');
}, 100);
```

### 日志分析技巧

#### 1. 结构化日志
```javascript
class Logger {
    constructor(level = 'info') {
        this.level = level;
        this.levels = {
            error: 0,
            warn: 1,
            info: 2,
            debug: 3
        };
    }

    log(level, message, data = {}) {
        if (this.levels[level] <= this.levels[this.level]) {
            const logEntry = {
                timestamp: new Date().toISOString(),
                level: level.toUpperCase(),
                message,
                data,
                stack: level === 'error' ? new Error().stack : undefined
            };

            console.log(JSON.stringify(logEntry, null, 2));
        }
    }

    error(message, data) { this.log('error', message, data); }
    warn(message, data) { this.log('warn', message, data); }
    info(message, data) { this.log('info', message, data); }
    debug(message, data) { this.log('debug', message, data); }
}

const logger = new Logger('debug');
logger.info('用户登录', { userId: 123, ip: '192.168.1.1' });
```

## 🔗 相关文档

- **技术原理**：[[JavaScript核心概念]] [[JavaScript异步编程]]
- **实战应用**：[[React问题解决]] [[Node.js问题解决]]

## 🏷️ 标签
#问题解决 #JavaScript #调试 #性能优化 #最佳实践