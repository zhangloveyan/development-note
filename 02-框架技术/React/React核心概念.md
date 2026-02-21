# React - 核心概念

---
tags: [React, 前端框架, 组件化, JSX, Hooks]
created: 2026-02-21
updated: 2026-02-21
status: 已掌握
importance: ⭐⭐⭐⭐⭐
---

## 🎯 核心要点
> React框架的核心概念和开发模式

- **组件化**：基于组件的UI构建方式
- **JSX语法**：JavaScript和XML的结合语法
- **状态管理**：useState Hook管理组件状态
- **生命周期**：useEffect Hook处理副作用

## 💡 原理详解

### 1. React基础概念

React是一个用于构建用户界面的JavaScript库，具有以下特点：

- **声明式**：描述UI应该是什么样子，而不是如何实现
- **组件化**：将UI拆分为独立、可复用的组件
- **虚拟DOM**：通过虚拟DOM提高性能
- **单向数据流**：数据从父组件流向子组件

### 2. JSX语法

JSX是JavaScript的语法扩展，允许在JavaScript中写HTML：

```jsx
// JSX编译前
const element = <h1>Hello, {name}!</h1>;

// JSX编译后
const element = React.createElement(
  'h1',
  null,
  'Hello, ',
  name,
  '!'
);
```

### 3. 组件类型

#### 函数组件（推荐）
```jsx
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}

// 箭头函数形式
const Welcome = (props) => {
  return <h1>Hello, {props.name}</h1>;
};
```

#### 类组件（传统方式）
```jsx
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

## 🔧 代码示例

### 基础用法

#### 创建React应用
```bash
# 使用Create React App
npx create-react-app my-app
cd my-app
npm start
```

#### 基础组件
```jsx
import React from 'react';

// 简单组件
function Greeting({ name }) {
  return <h1>你好, {name}!</h1>;
}

// 带默认props的组件
function Button({ children, onClick, type = 'button' }) {
  return (
    <button type={type} onClick={onClick}>
      {children}
    </button>
  );
}

// 使用组件
function App() {
  const handleClick = () => {
    alert('按钮被点击了!');
  };

  return (
    <div className="App">
      <Greeting name="张三" />
      <Button onClick={handleClick}>
        点击我
      </Button>
    </div>
  );
}

export default App;
```

#### 状态管理 - useState
```jsx
import React, { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(count + 1);
  };

  const decrement = () => {
    setCount(prevCount => prevCount - 1);
  };

  return (
    <div>
      <h2>计数器: {count}</h2>
      <button onClick={increment}>+1</button>
      <button onClick={decrement}>-1</button>
    </div>
  );
}
```

#### 表单处理
```jsx
import React, { useState } from 'react';

function ContactForm() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    message: ''
  });

  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prevData => ({
      ...prevData,
      [name]: value
    }));
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log('提交的数据:', formData);
    // 处理表单提交
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="name">姓名:</label>
        <input
          type="text"
          id="name"
          name="name"
          value={formData.name}
          onChange={handleChange}
          required
        />
      </div>

      <div>
        <label htmlFor="email">邮箱:</label>
        <input
          type="email"
          id="email"
          name="email"
          value={formData.email}
          onChange={handleChange}
          required
        />
      </div>

      <div>
        <label htmlFor="message">消息:</label>
        <textarea
          id="message"
          name="message"
          value={formData.message}
          onChange={handleChange}
          rows="4"
        />
      </div>

      <button type="submit">提交</button>
    </form>
  );
}
```

### 高级用法

#### useEffect Hook
```jsx
import React, { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // 组件挂载和userId变化时获取用户数据
  useEffect(() => {
    const fetchUser = async () => {
      try {
        setLoading(true);
        setError(null);

        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) {
          throw new Error('获取用户信息失败');
        }

        const userData = await response.json();
        setUser(userData);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    if (userId) {
      fetchUser();
    }
  }, [userId]);

  // 清理副作用
  useEffect(() => {
    const timer = setInterval(() => {
      console.log('定时器执行');
    }, 1000);

    // 清理函数
    return () => {
      clearInterval(timer);
    };
  }, []);

  if (loading) return <div>加载中...</div>;
  if (error) return <div>错误: {error}</div>;
  if (!user) return <div>未找到用户</div>;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>邮箱: {user.email}</p>
      <p>注册时间: {user.createdAt}</p>
    </div>
  );
}
```

#### 组件通信
```jsx
import React, { useState } from 'react';

// 父传子
function ParentComponent() {
  const [message, setMessage] = useState('来自父组件的消息');

  return (
    <div>
      <h2>父组件</h2>
      <ChildComponent message={message} />
    </div>
  );
}

function ChildComponent({ message }) {
  return (
    <div>
      <h3>子组件</h3>
      <p>{message}</p>
    </div>
  );
}

// 子传父
function ParentWithCallback() {
  const [childData, setChildData] = useState('');

  const handleChildData = (data) => {
    setChildData(data);
  };

  return (
    <div>
      <h2>父组件接收到: {childData}</h2>
      <ChildWithCallback onDataChange={handleChildData} />
    </div>
  );
}

function ChildWithCallback({ onDataChange }) {
  const [inputValue, setInputValue] = useState('');

  const handleSubmit = () => {
    onDataChange(inputValue);
  };

  return (
    <div>
      <input
        type="text"
        value={inputValue}
        onChange={(e) => setInputValue(e.target.value)}
        placeholder="输入数据"
      />
      <button onClick={handleSubmit}>发送给父组件</button>
    </div>
  );
}
```

#### Context跨层级通信
```jsx
import React, { createContext, useContext, useState } from 'react';

// 创建Context
const ThemeContext = createContext();

// Provider组件
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  const toggleTheme = () => {
    setTheme(prevTheme => prevTheme === 'light' ? 'dark' : 'light');
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// 使用Context的组件
function ThemedButton() {
  const { theme, toggleTheme } = useContext(ThemeContext);

  return (
    <button
      onClick={toggleTheme}
      style={{
        backgroundColor: theme === 'light' ? '#fff' : '#333',
        color: theme === 'light' ? '#333' : '#fff'
      }}
    >
      切换主题 (当前: {theme})
    </button>
  );
}

// 应用根组件
function App() {
  return (
    <ThemeProvider>
      <div>
        <h1>主题切换示例</h1>
        <ThemedButton />
      </div>
    </ThemeProvider>
  );
}
```

#### 列表渲染和条件渲染
```jsx
import React, { useState } from 'react';

function TodoList() {
  const [todos, setTodos] = useState([
    { id: 1, text: '学习React', completed: false },
    { id: 2, text: '写代码', completed: true },
    { id: 3, text: '休息', completed: false }
  ]);

  const [filter, setFilter] = useState('all'); // all, active, completed

  const toggleTodo = (id) => {
    setTodos(prevTodos =>
      prevTodos.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  };

  const filteredTodos = todos.filter(todo => {
    if (filter === 'active') return !todo.completed;
    if (filter === 'completed') return todo.completed;
    return true;
  });

  return (
    <div>
      <h2>待办事项</h2>

      {/* 过滤按钮 */}
      <div>
        <button
          onClick={() => setFilter('all')}
          className={filter === 'all' ? 'active' : ''}
        >
          全部
        </button>
        <button
          onClick={() => setFilter('active')}
          className={filter === 'active' ? 'active' : ''}
        >
          未完成
        </button>
        <button
          onClick={() => setFilter('completed')}
          className={filter === 'completed' ? 'active' : ''}
        >
          已完成
        </button>
      </div>

      {/* 待办列表 */}
      {filteredTodos.length === 0 ? (
        <p>没有待办事项</p>
      ) : (
        <ul>
          {filteredTodos.map(todo => (
            <li key={todo.id}>
              <label>
                <input
                  type="checkbox"
                  checked={todo.completed}
                  onChange={() => toggleTodo(todo.id)}
                />
                <span
                  style={{
                    textDecoration: todo.completed ? 'line-through' : 'none'
                  }}
                >
                  {todo.text}
                </span>
              </label>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## ⚡ 性能特点

| 特性 | 说明 | 适用场景 |
|------|------|----------|
| 虚拟DOM | 减少真实DOM操作 | 复杂UI更新 |
| 组件化 | 代码复用和维护 | 大型应用开发 |
| 单向数据流 | 数据流向清晰 | 状态管理 |
| Hooks | 函数组件状态管理 | 现代React开发 |

## 🔗 知识关联

- **前置知识**：[[JavaScript核心概念]] [[JavaScript异步编程]]
- **相关技术**：[[React Router]] [[Redux状态管理]]
- **实战应用**：[[React项目实战]] [[React性能优化]]
- **问题解决**：[[React问题解决]]

## 🏷️ 标签
#React #前端框架 #组件化 #JSX #Hooks #面试重点