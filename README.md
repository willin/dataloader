# DataLoader

<ruby>DataLoader<rp>（</rp><rt>数据加载器</rt><rp>）</rp></ruby> 是一个通用工具，可以用作应用程序数据获取层的一部分，通过批处理和缓存的方式为各种不同的后端和减少后端的请求数量提供一致的 API 接口。

!> 本文翻译自： https://github.com/graphql/dataloader

在 2010 年时由 Facebook 的 [@schrockn](https://github.com/schrockn) 最初开发了 “<ruby>Loader<rp>（</rp><rt>加载器</rt><rp>）</rp></ruby>” 接口的一个端口，在当时后端 API 接口中作为简单的强制合并杂项的 Key-Value 键值对存储使用。后来在 Facebook， “Loader” 变成了一个 “<ruby>Ent<rp>（</rp><rt>异构</rt><rp>）</rp></ruby>” 框架的实现细节，隐私感知数据实体加载和缓存层面的 Web 服务产品代码。 最终成为巩固 Facebook GraphQL （比 REST 更高效、强大和灵活的新一代 API 标准） 服务的实现和类型定义。

DataLoader 是一个用 Javascript 实现 Node.js 服务最初思想的简化版本。DataLoader 通常用于实现 [GraphQL JS][] 服务，但它的适用场景也很广泛。

这种批处理和数据请求缓存的机制并不是 Node.js 或 JavaScript 特有的， 它也是 [Haxl](https://github.com/facebook/Haxl) （Facebook 的 Haskell 数据加载库）的主要动机。 更多关于 Haxl 如何工作的信息可以在这篇[博客文章](ttps://code.facebook.com/posts/302060973291128/open-sourcing-haxl-a-library-for-haskell/)中查看。

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

批处理并不是一种高级特性，但却是 DataLoader 的主要特性。 创建 Loader 需要提供批处理加载方法。

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

批处理方法接受一个数组作为参数，并且返回一个数组结果的 Promise 或者一个 Error 实例。 Loader 本身作为 `this` 上下文。

```js
async function batchFunction(keys) {
  const results = await db.fetchAllKeys(keys);
  return keys.map(key => results[key] || new Error(`No result for ${key}`));
}

const loader = new DataLoader(batchFunction);
```

此方法必须遵守以下约束：

- 返回数组的长度必须和参数数组的长度相同。
- 返回数组中每一个下标必须与参数数组中相对应。

例如，如果你的批处理方法传入的参数为： `[2, 9, 6, 1]`，后端服务加载并返回的结果：

```js
{ id: 9, name: 'Chicago' },
{ id: 1, name: 'New York' },
{ id: 2, name: 'San Francisco' }
```

后端服务返回的结果和我们请求的顺序不同，可能是因为这么做的话效率会更高一些。并且，结果中缺少了参数 `6`，我们可以理解为不存在该参数对应的结果。

为了遵守批处理方法的约束，必须返回一个与参数数组长度相同的返回值数组，并对其进行重新排序，以确保每个下标与原始参数 `[2, 9, 6, 1]`对应。

```js
[
  { id: 2, name: 'San Francisco' },
  { id: 9, name: 'Chicago' },
  null, // or perhaps `new Error()`
  { id: 1, name: 'New York' }
]
```

### 批处理调度

默认情况下，DataLoader 会在调用批处理方法之前合并单框架执行中所有单独的加载。能够确保在多个相关请求转变成一个单一批处理中不会出现额外的延迟。事实上，这与 Facebook 在 2010 年最初的 PHP 实现中表现形式相同。请参阅[源代码](https://github.com/graphql/dataloader/blob/master/src/index.js)中的 `enqueuePostPromiseJob`，了解有关此操作方式的更多详细信息。

然而，有的时候这样的表现形式并不可取，或者并不是最佳。因为你的代码中使用了 `setTimeout`，也许你会希望请求在后续的 Tick 上扩展，或者你希望手动接管而不考虑运行循环。 DataLoader 允许提供自定义的批处理调度来支持这些或其他的场景。

自定义批处理调度在选项参数中称为 `batchScheduleFn`，必须传入一个有回调的方法，并且这个方法在执行批处理请求的时候能够立即执行。

举一个例子，这里是一个采集 100 毫秒内所有请求的批处理调度（所以增加了 100 毫秒的延迟）：

```js
const myLoader = new DataLoader(myBatchFn, {
  batchScheduleFn: callback => setTimeout(callback, 100)
});
```

再来另外一个例子，这是手动分发的批处理调度：

```js
function createScheduler() {
  let callbacks = [];
  return {
    schedule(callback) {
      callbacks.push(callback);
    },
    dispatch() {
      callbacks.forEach(callback => callback());
      callbacks = [];
    }
  };
}

const { schedule, dispatch } = createScheduler();
const myLoader = new DataLoader(myBatchFn, { batchScheduleFn: schedule });

myLoader.load(1);
myLoader.load(2);
dispatch();
```

## 缓存

DataLoader 为应用内每一个单独请求提供了内存缓存。 在 `.load()` 方法执行之后，将排除缓存中的冗余加载。

### 缓存每一个请求

DataLoader 的缓存**并不能**替代 Redis、Memcache 或者其他共享的应用级缓存。 DataLoader 主要是一个数据加载的机制，它的缓存只用于在单个请求的上下文中不再重复加载相同的数据。为了实现这一点，它在内存中维护了一个简单的缓存（更确切应该说：`.load()` 是一个记忆化方法）。

要避免来自不同用户的多个请求使用 DataLoader 实例，因为这可能导致各个请求缓存的数据出现异常。通常，DataLoader 实例在 <ruby>Request<rp>（</rp><rt>请求</rt><rp>）</rp></ruby> 开始时创建，并在请求结束后不再使用。

一个基于 [Express][] 的示例：

```js
function createLoaders(authToken) {
  return {
    users: new DataLoader(ids => genUsers(authToken, ids))
  };
}

const app = express();

app.get('/', function(req, res) {
  const authToken = authenticateUser(req);
  const loaders = createLoaders(authToken);
  res.send(renderPage(req, loaders));
});

app.listen();
```

### 缓存和批处理





[GraphQL JS]: https://github.com/graphql/graphql-js
[Express]: https://expressjs.com/