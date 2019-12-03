# DataLoader

DataLoader 是一个通用工具，可以用作应用程序数据获取层的一部分，通过批处理和缓存的方式为各种不同的后端和减少后端的请求数量提供一致的 API 接口。

!> 本文翻译自： https://github.com/graphql/dataloader

在 2010 年时由 Facebook 的 [@schrockn](https://github.com/schrockn) 最初开发了 “Loader” （加载器）接口的一个端口，在当时后端 API 接口中作为简单的强制合并杂项的 Key-Value 键值对存储使用。后来在 Facebook， “Loader” 变成了一个 “Ent”（异构） 框架的实现细节，隐私感知数据实体加载和缓存层面的 Web 服务产品代码。 最终成为巩固 Facebook GraphQL （比 REST 更高效、强大和灵活的新一代 API 标准） 服务的实现和类型定义。

DataLoader 是一个用 Javascript 实现 Node.js 服务最初思想的简化版本。DataLoader 通常用于实现 [GraphQL JS][] 服务，但它的适用场景也很广泛。

这种批处理和数据请求缓存的机制并不是 Node.js 或 JavaScript 特有的， 它也是 [Haxl](https://github.com/facebook/Haxl) （Facebook 的 Haskell 数据加载库）的主要动机。 更多关于 H阿修罗 如何工作的信息可以在这篇[博客文章](ttps://code.facebook.com/posts/302060973291128/open-sourcing-haxl-a-library-for-haskell/)中查看。

Dataloader 不仅可以用于构建 Node.js 下的 GraphQL 服务，还可以作为一个公共参考实现的概念移植到其他开发语言。 如果您将 DataLoader 移植到了其他的语言平台，请在源码项目中提交一个带有您项目链接的 Issue。

## 起步

首先，使用 npm 安装 DataLoader。

```bash
npm install --save dataloader
```

或者使用 yarn 进行安装：

```bash
yarn add dataloader
```

起步阶段，创建一个 `DataLoader`。每一个 `DataLoader` 实例代表一个独立缓存。如果不同的用户可以看到不同的东西，`DataLoader` 实例通常可以配合在类似 [Express][] 之类的 Web 服务器每个请求中使用。

!> 注意： DataLoader 需要 ES6 `Promise` 和 `Map` 类的 JavaScript 环境，或仅在具备该环境支持的 Node.js 版本中可以使用。

## 批处理

批处理并不是一种高级特性，但却是 DataLoader 的主要特性。 创建 Loaders （加载器）需要提供批处理加载方法。

```js
const DataLoader = require('dataloader');

const userLoader = new DataLoader(keys => myBatchGetUsers(keys));
```

一个批处理加载方法接受一个数组的参数，并且返回包裹一个数组的值的 Promise 对象[<sup>*</sup>](#batch-function)。

然后，从 Loader 中加载独立的返回值。 DataLoader 会合并单框架执行（时间循环中的一个 Tick）所有独立加载，然后执行所有请求参数的批处理方法。

```js
const user = await userLoader.load(1);
const invitedBy = await userLoader.load(user.invitedByID);
console.log(`User 1 was invited by ${invitedBy}`);

// 在你应用中的其他地方
const user = await userLoader.load(2);
const lastInvited = await userLoader.load(user.lastInvitedID);
console.log(`User 2 last invited ${lastInvited}`);
```

一个“天真”的应用可能会被4轮的后端信息请求问题困扰，但使用了 DataLoader 之后，这个应用最多只会产生两次请求。

DataLoader 允许你在不牺牲批量数据加载性能的情况下分离程序的无关部分。虽然 Loader 提供了加载单个值的 API，但所有并发请求都合并一起装载到你的批处理加载方法中执行。这就意味着可以在你的整个应用中安全分发数据获取需求，并维持最小的数据传出请求。

<a id="batch-function"></a>

### 批处理方法



[GraphQL JS]: https://github.com/graphql/graphql-js
[Express]: https://expressjs.com/