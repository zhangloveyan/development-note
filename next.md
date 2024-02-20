### 受控表单绑定

useState 控制表单状态，视图和数据的绑定

![image-20240220110827075](pic/image-20240220110827075.png)

```react
import { useState } from 'react';

function App() {
  const [value, setValue] = useState('');


  return (
    <div className="App">
      <h1>欢迎</h1>
      <input value={value} onChange={(e) => setValue(e.target.value)}></input>
    </div>
  );
}

export default App;
```

### react 获取 DOM

使用 useRef 钩子函数

1.创建 ref 对象（useRef），并与 jsx 绑定

2.DOM 可用时，通过 inputRef.current 拿到 DOM 对象

```react
import { useState, useRef } from 'react';
import dayjs from 'dayjs';

function App() {
  const inputRef = useRef(null);

  const showDown = () => {
    console.log(inputRef.current);
    console.dir(inputRef.current);
    inputRef.current.focus();
  }
  return (
    <div className='App'>
      <input type='text' ref={inputRef} />
      <button onClick={showDown} >点击</button>
      <p>{dayjs(new Date()).format()}</p>
    </div>
  );
}
export default App;
```

### 组件通信

组件之间的数据传递

#### 父传子

props 可以传递任意类型

![image-20240220154707352](pic/image-20240220154707352.png)

支持开放和封闭标签传递数据，开放标签默认使用 children 名字（多个时会变成数组）

```react
export default Compone;

function Son(props) {
    return (
        <div> 这是封闭子标签 {props.namea}</div>
    );
}

function Child(props) {
    console.log(props)
    return (
        <div> 这是开放子标签 {props.children[1]}</div>
    );
}

function Compone() {
    const value = '这是传递的数据';
    return (
        <div>
            这是父组件
            <Son namea={value} />
            <Child>
                <span>这是span标签</span>
                sldiaj
            </Child>
        </div>
    );
}
```

#### 子传父

父组件写接受方法，然后把方法传递给子组件，子组件调用方法传递数据

```react
import { useState } from "react";

export default Son;

// function Child(props) {
//     props.onGetMsg('传递的数据');
//     return <div>
//         这是子组件
//     </div>
// }

function Child({ onGetMsg }) {
    const msg = '传递的数据';
    return <div>
        这是子组件
        <button onClick={() => onGetMsg(msg)}>传递数据</button>
    </div>
}
function Son() {
    const [msg, setMsg] = useState('');
    const getMsg = (msg) => {
        console.log('接收到的数据' + msg);
        setMsg(msg);
    }
    return (
        <div>
            子传父通信,{msg}
            <Child onGetMsg={getMsg} />
        </div>);
}
```

#### 兄弟组建通信

通过子传父，父在传子进行数据传递，类似 父 view 中转

#### context 机制跨层级组件通信

步骤：（系统封装的对象，可以到处流转binder通信）

1. createContext创建一个上下文对象 ctx
2. 顶层组件（app）通过 ctx.Provider 提供数据
3. 底层组件（B）通过 useContext 钩子函数获取数据

![image-20240220173311391](pic/image-20240220173311391.png)

context -> context1 -> context2 嵌套

context 给 context2 传递数据

context.js

```react
import { createContext } from "react";
import Context1 from "./context1";

export default Context;
export const Ctx = createContext();

function Context() {
    const value = '我是传递的数据';
    return (
        <div>
            我是context
            <Ctx.Provider value={value}>
                <Context1 />
            </Ctx.Provider>
        </div>
    );
}
```

context1.js

```react
import Context2 from "./context2";

export default Context1;

function Context1() {
    return (
        <div>
            我是context1
            <Context2 />
        </div>
    );
}
```

context2.js

```react
import { useContext } from "react";
import { Ctx } from "./context"

function Context2() {
    const ctx = useContext(Ctx);
    return (
        <div>
            我是context2
            <p>
                {ctx}
            </p>
        </div>
    );
}
export default Context2;
```

#### useEffect

创建不是又事件引起而是由渲染本身引起的操作（view 初始化完成，onstart 里面 http 请求），比如：发送 ajax、更改 dom

语法：`useEffect(() => { },[])`

参数 1 是一个函数，可叫做副作用函数，内部放置要执行的操作

参数 2 是一个数组（可选），放置依赖，不同依赖影响第一个参数执行，若空，只执行一次

```react
import { useEffect, useState } from "react";

const URL = '/gzh/web/weChatMiniProgram/v2/faq/list/1';
function App() {
    const [list, setList] = useState([]);
    async function getList() {
        const res = await fetch(URL);
        const resJson = await res.json();
        console.log(resJson);
        setList(resJson.data);
    }

    useEffect(() => {
        getList();
    }, [])

    return (
        <div>
            加载数据。。。
            <ul>
                {list.map(item =><li key={item.id}>{item.faqTitle}</li>)}
            </ul>
        </div>
    );
}

export default App;
```



跨域问题