# web开发

js

合约开发、部署、调用

vue+web3+solidity

## 区块链

区块 + 链

**特征**

去中心化：对等网络每个节点即是服务器又是客户端

共识机制：PoW（工作量证明，算题）、PoS

不可篡改：修改数据会影响整个链的其他数据

**类型**

公链、私链、联盟链、混合链

![image-20230903210752137](pic/image-20230903210752137.png)

**记账挖矿**

![image-20230903213415615](pic/image-20230903213415615.png)

**共识机制**

- PoW：工作量证明
- PoS：权益证明机制
- PoA：权威证明机制
- PoC：容量证明机制
- CPoC：有条件的容量证明机制

## ETH

智能合约 Solidity语言

### 编写一个智能钱包

**环境准备**

- 工具：vscode
- 环境：node.js 

**创建项目**

1.安装 vue cli `npm install -g @vue/cli` 地址： (https://cli.vuejs.org/zh/#%E8%B5%B7%E6%AD%A5)

2.生成项目 `vue create web-wallet`，执行结果如下：

![image-20230904165855034](pic/image-20230904165855034.png)

3.选择自定义操作，选择如下：

![image-20230904170035418](pic/image-20230904170035418.png)

4.版本选择：

![image-20230904170106480](pic/image-20230904170106480.png)

5.CSS 样式选择：

![image-20230904170422548](pic/image-20230904170422548.png)

6.单独配置文件：

![image-20230904170200625](pic/image-20230904170200625.png)

7.是否保存配置：**选No**（否则后面没法选了）

![image-20230904170245459](pic/image-20230904170245459.png)

创建完成后可使用 `npm run serve` 运行，运行完成可点击链接打开

![image-20230904171543924](pic/image-20230904171543924.png)

**web3相关包引入** 

1.安装 web3.js： ``npm install web3`

2.实例化：`var web3 = new Web3(Web3.givenProvider || "ws://localhost:8545");`

3.获取以太坊网络的节点地址

infura 提供公开的猪王和测试网节点，infura.io 注册即可获取地址

如：`wss://goerli.infura.io/ws/v3/cb7e63cf28244e4499b4b6fb6162e746`





### 其他

#### 1. 配置 vant-ui ui 组件库 

地址：https://vant-contrib.gitee.io/vant/#/zh-CN

1.安装：npm=node package manager i=install

```vue
npm install vant
npm install unplugin-vue-components -D
```

2.在 vue.config.js 配置插件

```vue
const { defineConfig } = require('@vue/cli-service')
++ const { VantResolver } = require('unplugin-vue-components/resolvers')
++ const CompentsPlugin = require('unplugin-vue-components/webpack')
module.exports = defineConfig({
  transpileDependencies: true,
++  configureWebpack: {
++    plugins: [CompentsPlugin({ resolvers: [VantResolver()] })]
++  }
})
```

3.测试 在 app.vue 文件添加代码

```vue
<button>创建钱包</button>
// 替换为
<van-button type="primary">创建钱包</van-button>
```

#### 2. web3 简介

官方提供的一个连接以太坊区块链的模块，使用 http、ipc 与本地或远程以太坊节点进行交互，包含以太坊生态的所有功能。

连接以太坊暴露出来的 PRC 层，可以连接任何暴露的节点，从而与区块链交互。

JavaScript API 叫 web3.js，还有 web3.py 等

- web3.eth：用于以太坊区块链和智能合约之间的交互
- web3.utils：一些辅助方法
- web3.shh：用于协议进行通信的 P2P 和广播
- web3.bzz：用于与群网络交互的 Bzz 模块

**相关地址**：

github地址：https://github.com/web3/web3.js

开发文档：https://web3js.readthedocs.io/en/v1.10.0/

中文文档：https://learnblockchain.cn/docs/web3.js/

**常用 api**：

需要先实例化 web3 相当于 java new 对象

```vue
// 实例化
var web3 = new Web3(Web3.givenProvider || "wss://goerli.infura.io/ws/v3/cb7e63cf28244e4499b4b6fb6162e746");
```

1.创建账户

```vue
// 创建账户 参数-密码 可不填 
const account = web3.eth.accounts.create('123456');
// 账户对象 
console.log(account);
// 地址 "0x4C2a92E7CC53Ea722FC9B816b108D87ccB701aDc"
console.log(account.address);
// 私钥 "0xc4c81dd272f1b73682da85b93ba4bd0318dff02f2e2d8f35bb4672ef97ee4c62"
console.log(account.privateKey);
```

