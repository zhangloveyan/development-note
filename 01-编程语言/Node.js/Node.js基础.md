# Node.js - 基础

---
tags: [Node.js, 后端开发, JavaScript, 服务器端, npm]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> Node.js运行时环境的基础知识和包管理

- **运行时环境**：基于Chrome V8引擎的JavaScript运行时
- **事件驱动**：非阻塞I/O模型，适合高并发应用
- **包管理**：npm生态系统和依赖管理
- **模块系统**：CommonJS和ES6模块支持

## 💡 原理详解

### 1. Node.js架构

Node.js基于以下核心组件：
- **V8引擎**：Google开发的JavaScript引擎
- **libuv**：跨平台异步I/O库
- **核心模块**：fs、http、path等内置模块
- **C++绑定**：连接JavaScript和底层系统API

### 2. 事件循环

Node.js使用单线程事件循环处理异步操作：

```
   ┌───────────────────────────┐
┌─>│           timers          │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │
   └───────────────────────────┘
```

### 3. 模块系统

#### CommonJS模块
```javascript
// 导出模块
module.exports = {
    add: (a, b) => a + b,
    subtract: (a, b) => a - b
};

// 或者
exports.multiply = (a, b) => a * b;

// 导入模块
const math = require('./math');
const { add, subtract } = require('./math');
```

#### ES6模块
```javascript
// 导出模块
export const add = (a, b) => a + b;
export default class Calculator {
    multiply(a, b) {
        return a * b;
    }
}

// 导入模块
import { add } from './math.js';
import Calculator from './calculator.js';
```

## 🔧 代码示例

### 基础用法

#### HTTP服务器
```javascript
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
    const parsedUrl = url.parse(req.url, true);
    const path = parsedUrl.pathname;
    const method = req.method;

    // 设置响应头
    res.setHeader('Content-Type', 'application/json');
    res.setHeader('Access-Control-Allow-Origin', '*');

    // 路由处理
    if (path === '/api/users' && method === 'GET') {
        res.statusCode = 200;
        res.end(JSON.stringify({
            users: [
                { id: 1, name: '张三' },
                { id: 2, name: '李四' }
            ]
        }));
    } else if (path === '/api/health' && method === 'GET') {
        res.statusCode = 200;
        res.end(JSON.stringify({ status: 'OK' }));
    } else {
        res.statusCode = 404;
        res.end(JSON.stringify({ error: 'Not Found' }));
    }
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`服务器运行在端口 ${PORT}`);
});
```

#### 文件操作
```javascript
const fs = require('fs').promises;
const path = require('path');

// 异步文件操作
async function fileOperations() {
    try {
        // 读取文件
        const data = await fs.readFile('config.json', 'utf8');
        const config = JSON.parse(data);

        // 写入文件
        const newConfig = { ...config, lastUpdated: new Date().toISOString() };
        await fs.writeFile('config.json', JSON.stringify(newConfig, null, 2));

        // 检查文件是否存在
        try {
            await fs.access('backup.json');
            console.log('备份文件存在');
        } catch {
            console.log('备份文件不存在');
        }

        // 创建目录
        await fs.mkdir('logs', { recursive: true });

        // 读取目录
        const files = await fs.readdir('.');
        console.log('当前目录文件:', files);

    } catch (error) {
        console.error('文件操作错误:', error);
    }
}
```

### 高级用法

#### Express框架基础
```javascript
const express = require('express');
const app = express();

// 中间件
app.use(express.json());
app.use(express.static('public'));

// 日志中间件
app.use((req, res, next) => {
    console.log(`${new Date().toISOString()} - ${req.method} ${req.url}`);
    next();
});

// 路由
app.get('/api/users', (req, res) => {
    res.json({
        users: [
            { id: 1, name: '张三', email: 'zhangsan@example.com' },
            { id: 2, name: '李四', email: 'lisi@example.com' }
        ]
    });
});

app.post('/api/users', (req, res) => {
    const { name, email } = req.body;

    if (!name || !email) {
        return res.status(400).json({ error: '姓名和邮箱是必需的' });
    }

    const newUser = {
        id: Date.now(),
        name,
        email
    };

    res.status(201).json(newUser);
});

// 错误处理中间件
app.use((error, req, res, next) => {
    console.error('服务器错误:', error);
    res.status(500).json({ error: '内部服务器错误' });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Express服务器运行在端口 ${PORT}`);
});
```

#### 流处理
```javascript
const fs = require('fs');
const { Transform } = require('stream');

// 创建转换流
class UpperCaseTransform extends Transform {
    _transform(chunk, encoding, callback) {
        this.push(chunk.toString().toUpperCase());
        callback();
    }
}

// 使用流处理大文件
function processLargeFile() {
    const readStream = fs.createReadStream('large-file.txt');
    const writeStream = fs.createWriteStream('output.txt');
    const upperCaseTransform = new UpperCaseTransform();

    readStream
        .pipe(upperCaseTransform)
        .pipe(writeStream)
        .on('finish', () => {
            console.log('文件处理完成');
        })
        .on('error', (error) => {
            console.error('处理错误:', error);
        });
}
```

## 📦 NPM包管理

### 基础命令

#### 项目初始化
```bash
# 初始化新项目
npm init
npm init -y  # 使用默认配置

# 安装依赖
npm install express
npm install --save express          # 生产依赖
npm install --save-dev nodemon      # 开发依赖
npm install -g pm2                  # 全局安装

# 安装特定版本
npm install express@4.18.0
npm install express@^4.18.0         # 兼容版本
npm install express@~4.18.0         # 补丁版本
```

#### 依赖管理
```bash
# 查看依赖
npm list                    # 本地依赖树
npm list -g                 # 全局依赖
npm list --depth=0          # 只显示顶级依赖

# 更新依赖
npm update                  # 更新所有依赖
npm update express          # 更新特定依赖
npm outdated               # 查看过时依赖

# 卸载依赖
npm uninstall express
npm uninstall -g pm2
```

#### 脚本管理
```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest",
    "build": "webpack --mode production",
    "lint": "eslint src/",
    "clean": "rm -rf dist/"
  },
  "dependencies": {
    "express": "^4.18.0",
    "mongoose": "^6.0.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.0",
    "jest": "^28.0.0"
  }
}
```

### 配置管理

#### .npmrc配置
```bash
# 查看配置
npm config list
npm config get registry

# 设置配置
npm config set registry https://registry.npmmirror.com/
npm config set prefix "D:\Program Files\nodejs\node_global"
npm config set cache "D:\Program Files\nodejs\node_cache"

# 删除配置
npm config delete registry
```

#### 多版本管理
```bash
# 使用nvm管理Node.js版本
nvm install 16.14.0
nvm use 16.14.0
nvm list
nvm alias default 16.14.0

# 项目级版本控制
echo "16.14.0" > .nvmrc
nvm use  # 自动使用.nvmrc中的版本
```

## ⚡ 性能特点

| 特性 | 说明 | 适用场景 |
|------|------|----------|
| 单线程事件循环 | 高效处理I/O密集型任务 | Web服务器、API服务 |
| 非阻塞I/O | 高并发处理能力 | 实时应用、聊天系统 |
| 轻量级 | 快速启动和部署 | 微服务架构 |
| 丰富生态 | npm包生态系统 | 快速开发 |

## 🔗 知识关联

- **前置知识**：[[JavaScript核心概念]] [[JavaScript异步编程]]
- **相关技术**：[[Express框架]] [[MongoDB数据库]]
- **实战应用**：[[RESTful API开发]] [[微服务架构]]
- **问题解决**：[[Node.js问题解决]]

## 🏷️ 标签
#Node.js #后端开发 #JavaScript #npm #服务器端 #面试重点