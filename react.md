# React

# 路线图

![image-20240214201003519](pic/image-20240214201003519.png)

# 基础

## 1.创建

1.create-react-app 快速搭建环境（类似 spring-boot interlize）创建项目，底层由 webpack 构建

命令：

```react
npx create-react-app react-basic
// npx node.js 工具命令
// create-react-app 核心包（固定）
// react-basic 项目名称
```

2.目录结构

package.json 包管理文件

index.js 程序入口

渲染路径 App -> index.js -> public/index.html（root）

## 2.JSX

JavaScript 和 XML 缩写，在 js 代码中写 html 模板

![image-20240214205905063](pic/image-20240214205905063.png)

编译过程

![image-20240214210111034](pic/image-20240214210111034.png)

**JSX 中使用 JS 表达式**

大括号 {} 识别 js 表达式，常见

1. 字符串
2. js 变量
3. 函数/方法调用
4. 使用 js 对象

注意：if、switch、变量声明属于语句，不是表达式，不能出现在 {} 中

![image-20240214211503126](pic/image-20240214211503126.png)

**列表渲染**

使用 js map 方法遍历渲染列表

```react
const list = [
  { id: 1001, name: 'vue' },
  { id: 1002, name: 'react' },
  { id: 1003, name: 'angular' }
]
<ul>
    {list.map(item =>
              <li key={item.id}>
                  {item.name}
              </li>)}
</ul>
```

**条件渲染**

![image-20240214212956051](pic/image-20240214212956051.png)

**复杂情况**

通过函数和 if 判断

![image-20240214214645980](pic/image-20240214214645980.png)

## 3.事件绑定

语法：on + 事件名称 = { 事件处理程序 }

```react
function App() {
  const clickHandle = () => {
    console.log('按钮点击')
  }
  return (
    <div className="App">
      <button onClick={clickHandle}>按钮</button>
    </div>
  );
}
```

自定义函数

```react
function App() {
  // e element
  const clickHandle = (name, e) => {
    console.log(name + '按钮点击')
  }
  return (
    <div className="App">
      <button onClick={(e) => clickHandle('abc',e)}>按钮</button>
    </div>
  );
}
```

```react
// 错误写法 直接写 相当于是方法调用 不是绑定方法
<button onClick={clickHandle('abc')}>按钮</button>
```

## 4.组件

抽取通用 view ，引入使用

![image-20240214220603549](pic/image-20240214220603549.png)

```react
function Button() {
  return <button>组件按钮</button>
}
function App() {
  return (
    <div className="App">
      <Button />
    </div>
  );
}
```

## 5.useState

React Hook（函数），添加一个状态变量，影响组件的渲染结果（数据驱动视图）

类似于 mvvm 的刷新

![image-20240214221300897](pic/image-20240214221300897.png)

注意：状态不可改

> 状态被认为是只读的，始终要替换它，而不是修改它，修改不会引起视图更新

## 6.组件样式

两种方式

1.行内样式（不推荐）

```react
<p style={{ color: 'red' ,fontSize: '20px'}}>红色标签</p>
```

2.class 类名控制

index.css 定义样式（样式抽取）

```css
.foo{
    color: rgb(0, 21, 255)
}
```

使用时，直接 className 引入

```react
import './index.css'; 
<p className="foo">蓝色标签</p>
```

































