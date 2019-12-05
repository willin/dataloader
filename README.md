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

一个批处理加载方法接受一个数组的键名，并且返回包裹一个数组的值的 Promise 对象[<sup>*</sup>](#batch-function)。

然后，从 Loader 中加载独立的返回值。 DataLoader 会合并单框架执行（时间循环中的一个 Tick）所有独立加载，然后执行所有请求键名的批处理方法。

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

批处理方法接受一个数组作为键名，并且返回一个数组结果的 Promise 或者一个 Error 实例。 Loader 本身作为 `this` 上下文。

```js
async function batchFunction(keys) {
  const results = await db.fetchAllKeys(keys);
  return keys.map(key => results[key] || new Error(`No result for ${key}`));
}

const loader = new DataLoader(batchFunction);
```

此方法必须遵守以下约束：

- 返回数组的长度必须和键名数组的长度相同。
- 返回数组中每一个下标必须与键名数组中相对应。

例如，如果你的批处理方法传入的键名为： `[2, 9, 6, 1]`，后端服务加载并返回的结果：

```js
{ id: 9, name: 'Chicago' },
{ id: 1, name: 'New York' },
{ id: 2, name: 'San Francisco' }
```

后端服务返回的结果和我们请求的顺序不同，可能是因为这么做的话效率会更高一些。并且，结果中缺少了键名 `6`，我们可以理解为不存在该键名对应的结果。

为了遵守批处理方法的约束，必须返回一个与键名数组长度相同的返回值数组，并对其进行重新排序，以确保每个下标与原始键名 `[2, 9, 6, 1]`对应。

```js
[
  { id: 2, name: 'San Francisco' },
  { id: 9, name: 'Chicago' },
  null, // or perhaps `new Error()`
  { id: 1, name: 'New York' }
]
```

<a id="batch-scheduling"></a>

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

后续调用相同键名的  `.load()` 方法时，该键名将不会再被添加到批处理方法中。 *然而*，返回的 Promise 将仍然等待当前批处理完成。这样，缓存和未缓存的请求将会同时 <ruby>Resolve<rp>（</rp><rt>解决</rt><rp>）</rp></ruby>，允许 DataLoader 对后续依赖的加载优化。

在下面的例子里，<ruby>User<rp>（</rp><rt>用户</rt><rp>）</rp></ruby> `1` 恰巧是缓存的。 然而，因为 User `1` 和 `2` 在同一个 Tick 中加载，它们将会被同时 Resolve。这就意味着 `user.bestFrientID` 加载也会在同一 Tick 下发生，导致了产生 2 次总请求数（与 User `1` 未缓存情况相同）。

```js
userLoader.prime(1, { bestFriend: 3 });

async function getBestFriend(userID) {
  const user = await userLoader.load(userID);
  return await userLoader.load(user.bestFriendID);
}

// 应用程序的某处
getBestFriend(1);

// 其他地方
getBestFriend(2);
```

如果没有这种优化，缓存的 User `1` 立即 Resolve，这可能会导致 3 次请求总数，因为每次 `user.bestFriendID` 加载并非同时发生。

### 清除缓存

在某些不确定因素下，可能需要清除请求缓存。

最常见的例子是在同一请求的修改或更新后，清除 Loader 缓存是必要的。因为缓存的值可能已经过时了，而未来的加载不应该加载任何之前缓存的值。

这里是个简单的 SQL 更新示例。

```js
// 请求开始...
const userLoader = new DataLoader(...);

// 一个加载并且被缓存
const user = await userLoader.load(4)

// 修改操作，导致缓存内容失效
await sqlRun('UPDATE users WHERE id=4 SET username="zuck"')
userLoader.clear(4)

// 后续使用 Loader 应该重新加载一遍
const user = await userLoader.load(4)

// 请求结束
```

### 缓存报错

如果一次批处理加载失败（即批处理方法抛出 Promise  `reject`），那么和请求的值将不会缓存。但是，如果批处理方法针对单个值返回的是 `Error` 实例，那么 `Error` 将会缓存以防止频繁加载出现相同的报错。

在某些情况下，你可能会希望清除这种针对单个错误的缓存：

```js
try {
  const user = await userLoader.load(1);
} catch (error) {
  if (/* determine if the error should not be cached */) {
    userLoader.clear(1);
  }
  throw error;
}
```

### 禁用缓存

在某些非常见情况下， 可能希望*不缓存* DataLoader。执行 `new DataLoader(myBatchFn, { cache: false })` 将会确保每次执行 `.load()` 方法时产生一个*新* Promise，请求的键名将不会存储到内存中。

然而，当内存缓存禁用时，你的批处理方法可能会接受含有重复值数组的键名！每个键名值各自调用 `.load()`。你的批处理加载器将会为每个请求键名的实例提供返回值。

例如：

```js
const myLoader = new DataLoader(keys => {
  console.log(keys);
  return someBatchLoadFn(keys);
}, { cache: false });

myLoader.load('A');
myLoader.load('B');
myLoader.load('A');

// > [ 'A', 'B', 'A' ]
```

通过调用  `.clear()` 或 `.clearAll()` 而不是完全禁用缓存，可以实现更复杂的缓存行为。例如，让 DataLoader 在启用内存缓存时为批处理方法提供唯一的 Key，但是当调用批处理方法时，立即清理掉它的缓存，这样后续的请求将加载出新的值。

```js
const myLoader = new DataLoader(keys => {
  identityLoader.clearAll();
  return someBatchLoadFn(keys);
})
```

### 自定义缓存

如上所述， DataLoader 的目的是用于缓存每次请求。由于请求是短暂的， DataLoader 使用一个无限增长的 [Map][] 作为内存缓存。这应该不至于造成问题，因为大多数请求都是短暂的，整个缓存可以在请求结束后丢弃掉。

然而对于使用长期 DataLoader 时，这种内存缓存的策略是不安全的，因为它会消耗太多的内存了。如果在这样的场景中使用 DataLoader，你可以提供一个自定义的缓存实例，不管你喜欢怎么样实现，只要能够遵循 [Map][] 相同的 API 即可。

下面这个示例使用了 <ruby>LRU<rp>（</rp><rt>least recently used</rt><rp>）</rp></ruby> 缓存来限制总内存，通过使用 [lru_map](https://github.com/rsms/js-lru) 的 npm 包控制最多存储 100 个缓存值。

```js
import { LRUMap } from 'lru_map';

const myLoader = new DataLoader(someBatchLoadFn, {
  cacheMap: new LRUMap(100)
});
```

更具体地说，任意实现了 `get()`、`set()`、`delete()` 和 `clear()` 方法的实现方式都可以。这就应运而生了大量的[缓存算法](https://en.wikipedia.org/wiki/Cache_algorithms)。

## API

### DataLoader <ruby>Class<rp>（</rp><rt>类</rt><rp>）</rp></ruby>

DataLoader 创建了一个从特定后端加载指定唯一键名（例如 SQL 表的 `id` 字段或者 MongoDB 数据库的文档名称）数据的批处理加载方法的公开 API。

每一个 `DataLoader` 实例包含了唯一的内存缓存。在长期应用或者服务于多个不同访问权限的用户时需要谨慎使用，并考虑为每一个 Web 请求创建一个新的实例。

### `new DataLoader(batchLoadFn [, options])`

使用一个批处理加载方法和参数来创建一个新的 `DataLoader`。

- *batchLoadFn*：一个加载方法，用一个键名数组作为参数，返回一个 Promise 对象来解决一个返回值数组。
- *options*： 一个可选的参数对象：

| 参数名 | 类型 | 默认值 | 描述 |
| ---------- | ---- | ------- | ----------- |
| *batch*  | Boolean | `true` | 设置为 `false` 禁用批处理，每次加载均触发 `batchLoadFn` 方法执行。相当于设置 `maxBatchSize` 为 `1`。 |
| *maxBatchSize* | Number | `Infinity` | 限制传入 `batchLoadFn` 方法的条目数量。 可以设置为 `1` 来禁用批处理。 |
| *batchScheduleFn* | Function | 见 [批处理调度](#batch-scheduling) | 批处理调度的后期执行方法。这个方法需要立即执行回调。 |
| *cache* | Boolean | `true` | 设置为 `false` 禁用缓存，为每一个相同参数的  `batchLoadFn` 创建新的 Promise。相当于设置 `cacheMap` 为 `null`。 |
| *cacheKeyFn* | Function | `key => key` | 为缓存提供键名。当用对象作为参数时，并且两个对象可以视为相同的情况下使用。 |
| *cacheMap* | Object | `new Map()` | [Map][] 对象（或相似 API 的对象）用作缓存。可以设置为 `null` 来禁用缓存。 |

### `.load(key)`

加载一个键名，返回一个对应该键名的值的 Promise。

- `key`： 一个需要加载的键名

###  `.loadMany(keys)`

加载多个键名，返回一个值的数组：

```js
const [ a, b ] = await myLoader.loadMany([ 'a', 'b' ]);
```

这个就类似于冗长的写法：

```js
const [ a, b ] = await Promise.all([
  myLoader.load('a'),
  myLoader.load('b')
]);
```

然而，还是会在加载失败的时候有一些区别。 `Promise.all()` 会抛出 reject 异常， loadMany() 永远返回 resolve，只是在返回的结果中可能会是一个值或者一个 Error 实例。

```js
var [ a, b, c ] = await myLoader.loadMany([ 'a', 'b', 'badkey' ]);
// c 是个 Error 实例
```

* `keys`： 一个需要加载的键名数组

### `.clear(key)`

清除某个键名对应的缓存值（如果存在）。返回自身供链式调用。

* `key`： 一个需要清除的键名

### `.clearAll()`

清除所有缓存。在某些未知非法结果时使用。返回自身供链式调用。

### `.prime(key, value)`

使用键值对初始化缓存。如果键名已经存在，则不发生改变。（如果需要强制重新初始化缓存，可以先清除 `loader.clear(key).prime(key, value)` 。）返回自身供链式调用。

可以用一个错误实例来初始化缓存。

## 与 GraphQL 一起使用

DataLoader 与 [GraphQL][GraphQL JS] 可以完美搭配使用。GraphQL 的字段被设计为独立的方法。 没有缓存或者批处理机制的话， GraphQL 服务器很容易就会被数据请求给撑爆。

例如以下的 GraphQL 请求：

```gql
{
  me {
    name
    bestFriend {
      name
    }
    friends(first: 5) {
      name
      bestFriend {
        name
      }
    }
  }
}
```

如果 `me` 、`bestFriend` 和 `friends` 需要向服务端请求，那么这里可能会有多达 13 条数据查询。

在使用了 DataLoader 后，我们可以用清晰的代码和至多 4 次数据库查询（以 [SQLite](https://github.com/graphql/dataloader/blob/master/examples/SQL.md) 定义 `User` 类型来示意）甚至更少（如果命中缓存）。

```js
const UserType = new GraphQLObjectType({
  name: 'User',
  fields: () => ({
    name: { type: GraphQLString },
    bestFriend: {
      type: UserType,
      resolve: user => userLoader.load(user.bestFriendID)
    },
    friends: {
      args: {
        first: { type: GraphQLInt }
      },
      type: new GraphQLList(UserType),
      resolve: async (user, { first }) => {
        const rows = await queryLoader.load([
          'SELECT toID FROM friends WHERE fromID=? LIMIT ?', user.id, first
        ]);
        return rows.map(row => userLoader.load(row.toID));
      }
    }
  })
});
```

## 常用场景

### 为每个请求创建 DataLoader

在很多应用中，Web 服务器可能会使用 DataLoader 服务于很多不同的用户、并且区分不同的访问权限。如果很多用户共用一个缓存，会是非常危险的，所以鼓励为不同的请求创建新的 DataLoader：

```js
function createLoaders(authToken) {
  return {
    users: new DataLoader(ids => genUsers(authToken, ids)),
    cdnUrls: new DataLoader(rawUrls => genCdnUrls(authToken, rawUrls)),
    stories: new DataLoader(keys => genStories(authToken, keys)),
  }
}

// 当处理一个流入的Web请求时
const loaders = createLoaders(request.query.authToken);

// 然后在应用程序中使用逻辑：
const user = await loaders.users.load(4)
const pic = await loaders.cdnUrls.load(user.rawPicUrl)
```

创建一个对象，每一个不同的键名来区分 DataLoader 是一种常见的使用方式，这可以提供一个单一的值传给需要执行数据加载的代码，例如 [GraphQL JS][] 请求中 `rootValue` 的一部分。

### 通过可替换键名加载

有时，某些值可以通过不同的方式被获取。例如，<ruby>User<rp>（</rp><rt>用户</rt><rp>）</rp></ruby> 类型的数据可以通过 `id` 或者 `username` 字段来获取结果。如果相同的用户被两个键名加载的话，同时缓存所有键名会很有效：

```js
const userByIDLoader = new DataLoader(async ids => {
  const users = await genUsersByID(ids);
  for (let user of users) {
    usernameLoader.prime(user.username, user);
  }
  return users;
})

const usernameLoader = new DataLoader(async names => {
  const users = await genUsernames(names);
  for (let user of users) {
    userByIDLoader.prime(user.id, user);
  }
  return users;
})
```

### 冻结结果强制不可篡改

DataLoader 的缓存值一般情况下是应该视为不可修改的。 然而 DataLoader 本身并不会强制这样，你可以使用 `Object.freeze()` 创建一个高阶函数来实现不可篡改：

```js
function freezeResults(batchLoader) {
  return keys => batchLoader(keys).then(values => values.map(Object.freeze));
}

const myLoader = new DataLoader(freezeResults(myBatchLoader));
```

### 返回对象（而不是数组）的批处理方法

DataLoader 期望的批处理方法需要返回一个与提高键名数组等长度的值数组。但这并不是一种其他第三方库的常见返回格式。 可以使用 DataLoader 高阶方法来转换类型。下面的例子转换成键值对的结果。

```js
function objResults(batchLoader) {
  return keys => batchLoader(keys).then(objValues => keys.map(
    key => objValues[key] || new Error(`No value for ${key}`)
  ));
}

const myLoader = new DataLoader(objResults(myBatchLoader));
```

## 常见后端数据库

想通过一个特定的后端数据库来起步？试试 [DataLoader 官方提供的示例](https://github.com/graphql/dataloader/tree/master/examples)。

## 其他语言实现

按照字母顺序排列

* Elixir
  * [dataloader](https://github.com/absinthe-graphql/dataloader)
* Golang
  * [Dataloader](https://github.com/nicksrandall/dataloader)
* Java
  * [java-dataloader](https://github.com/graphql-java/java-dataloader)
* .Net
  * [GraphQL .NET DataLoader](https://graphql-dotnet.github.io/docs/guides/dataloader/)
  * [GreenDonut](https://github.com/ChilliCream/greendonut)
* Perl
  * [perl-DataLoader](https://github.com/richardjharris/perl-DataLoader)
* PHP
  * [DataLoaderPHP](https://github.com/overblog/dataloader-php)
* Python
  * [aiodataloader](https://github.com/syrusakbary/aiodataloader)
* ReasonML
  * [bs-dataloader](https://github.com/ulrikstrid/bs-dataloader)
* Ruby
  * [BatchLoader](https://github.com/exaspark/batch-loader)
  * [Dataloader](https://github.com/sheerun/dataloader)
  * [GraphQL Batch](https://github.com/Shopify/graphql-batch)
* Rust
  * [Dataloader](https://github.com/cksac/dataloader-rs)
* Swift
  * [SwiftDataLoader](https://github.com/kimdv/SwiftDataLoader)

## 视频教程

**DataLoader 视频教程 （YouTube）：**

一个 DataLoader v1 的视频教程。虽然源码已经重构了，但这个视频依然是个很好的介绍概述，来帮助你了解 DataLoader 如何运作。

<a href="https://youtu.be/OQTnXNCDywA" target="_blank" alt="DataLoader Source Code Walkthrough"><img src="https://img.youtube.com/vi/OQTnXNCDywA/0.jpg" /></a>



## 版权信息

译者： [Willin Wang](https://github.com/willin)

团队： [@视网么](https://github.com/shiwangme)

MIT License

首次翻译完成：2019.12.05

最后更新时间： 2019.12.05

[GraphQL JS]: https://github.com/graphql/graphql-js
[Express]: https://expressjs.com/
[Map]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map

