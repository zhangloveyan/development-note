# JavaScript - 异步编程

---
tags: [JavaScript, 异步编程, Promise, async/await, 事件循环]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> JavaScript异步编程的核心概念和实现方式

- **事件循环**：JavaScript单线程异步执行机制
- **回调函数**：最基础的异步处理方式
- **Promise**：解决回调地狱的现代方案
- **async/await**：基于Promise的语法糖

## 💡 原理详解

### 1. 事件循环机制

JavaScript是单线程语言，通过事件循环实现异步操作：

```
┌───────────────────────────┐
┌─>│           timers          │  ← setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← 内部使用
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  ← 获取新的I/O事件
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  ← setImmediate
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  ← socket.on('close', ...)
   └───────────────────────────┘
```

### 2. 宏任务和微任务

- **宏任务**：setTimeout、setInterval、I/O、UI渲染
- **微任务**：Promise.then、queueMicrotask、MutationObserver

执行顺序：同步代码 → 微任务 → 宏任务

### 3. Promise状态机

Promise有三种状态：
- **Pending**：初始状态，既不是成功，也不是失败
- **Fulfilled**：操作成功完成
- **Rejected**：操作失败

## 🔧 代码示例

### 基础用法

#### 回调函数
```javascript
// 传统回调方式
function fetchData(callback) {
    setTimeout(() => {
        const data = { id: 1, name: '用户数据' };
        callback(null, data);
    }, 1000);
}

fetchData((error, data) => {
    if (error) {
        console.error('获取数据失败:', error);
    } else {
        console.log('获取数据成功:', data);
    }
});
```

#### Promise基础
```javascript
// 创建Promise
const fetchUserData = () => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            const success = Math.random() > 0.5;
            if (success) {
                resolve({ id: 1, name: '张三' });
            } else {
                reject(new Error('网络错误'));
            }
        }, 1000);
    });
};

// 使用Promise
fetchUserData()
    .then(data => {
        console.log('用户数据:', data);
        return data.id;
    })
    .then(userId => {
        console.log('用户ID:', userId);
    })
    .catch(error => {
        console.error('错误:', error.message);
    })
    .finally(() => {
        console.log('请求完成');
    });
```

### 高级用法

#### async/await
```javascript
// async函数
async function getUserInfo(userId) {
    try {
        const user = await fetchUserData();
        const profile = await fetchUserProfile(user.id);
        const posts = await fetchUserPosts(user.id);

        return {
            user,
            profile,
            posts
        };
    } catch (error) {
        console.error('获取用户信息失败:', error);
        throw error;
    }
}

// 使用async函数
async function main() {
    try {
        const userInfo = await getUserInfo(1);
        console.log('完整用户信息:', userInfo);
    } catch (error) {
        console.error('主函数错误:', error);
    }
}
```

#### Promise并发控制
```javascript
// Promise.all - 并行执行，全部成功才成功
async function fetchAllData() {
    try {
        const [users, posts, comments] = await Promise.all([
            fetchUsers(),
            fetchPosts(),
            fetchComments()
        ]);

        return { users, posts, comments };
    } catch (error) {
        console.error('获取数据失败:', error);
    }
}

// Promise.allSettled - 并行执行，获取所有结果
async function fetchDataWithResults() {
    const results = await Promise.allSettled([
        fetchUsers(),
        fetchPosts(),
        fetchComments()
    ]);

    results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
            console.log(`请求${index}成功:`, result.value);
        } else {
            console.error(`请求${index}失败:`, result.reason);
        }
    });
}

// Promise.race - 竞速执行，第一个完成的决定结果
async function fetchWithTimeout() {
    try {
        const result = await Promise.race([
            fetchUserData(),
            new Promise((_, reject) =>
                setTimeout(() => reject(new Error('请求超时')), 5000)
            )
        ]);

        return result;
    } catch (error) {
        console.error('请求失败或超时:', error);
    }
}
```

#### 自定义Promise工具
```javascript
// 延迟函数
const delay = (ms) => new Promise(resolve => setTimeout(resolve, ms));

// 重试机制
async function retryAsync(fn, maxRetries = 3, delayMs = 1000) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await fn();
        } catch (error) {
            if (i === maxRetries - 1) throw error;
            await delay(delayMs);
            console.log(`重试第${i + 1}次...`);
        }
    }
}

// 并发限制
class ConcurrencyLimit {
    constructor(limit) {
        this.limit = limit;
        this.running = 0;
        this.queue = [];
    }

    async add(fn) {
        return new Promise((resolve, reject) => {
            this.queue.push({ fn, resolve, reject });
            this.process();
        });
    }

    async process() {
        if (this.running >= this.limit || this.queue.length === 0) {
            return;
        }

        this.running++;
        const { fn, resolve, reject } = this.queue.shift();

        try {
            const result = await fn();
            resolve(result);
        } catch (error) {
            reject(error);
        } finally {
            this.running--;
            this.process();
        }
    }
}
```

## ⚡ 性能特点

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 回调函数 | 简单直接 | 回调地狱 | 简单异步操作 |
| Promise | 链式调用，错误处理 | 语法复杂 | 复杂异步流程 |
| async/await | 同步写法 | 需要Promise支持 | 现代异步编程 |

## 🔗 知识关联

- **前置知识**：[[JavaScript核心概念]]
- **相关技术**：[[Node.js异步编程]] [[Web API]]
- **实战应用**：[[React异步处理]] [[Ajax请求]]
- **问题解决**：[[JavaScript问题解决#异步编程问题]]

## 🏷️ 标签
#JavaScript #异步编程 #Promise #async/await #事件循环 #面试重点