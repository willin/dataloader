# DataLoader 

DataLoader 是一个通用工具，可以用作应用程序数据获取层的一部分，通过批处理和缓存的方式为各种不同的后端和减少后端的请求数量提供一致的 API 接口。

!> 本文翻译自： https://github.com/graphql/dataloader

在 2010 年时由 Facebook 的 [@schrockn](https://github.com/schrockn) 最初开发了 “Loader” （加载器）接口的一个端口，在当时后端 API 接口中作为简单的强制合并杂项的 Key-Value 键值对存储使用。后来在 Facebook， “Loader” 变成了一个 “Ent”（异构） 框架的实现细节，隐私感知数据实体加载和缓存层面的 Web 服务产品代码。 最终成为巩固 Facebook GraphQL （比 REST 更高效、强大和灵活的新一代 API 标准） 服务的实现和类型定义。

DataLoader 是一个用 Javascript 实现 Node.js 服务最初思想的简化版本。DataLoader 通常用于实现 [GraphQL JS][] 服务，但它的适用场景也很广泛。

这种批处理和数据请求缓存的机制并不是 Node.js 或 JavaScript 特有的， 它也是 [Haxl](https://github.com/facebook/Haxl) （Facebook 的 Haskell 数据加载库）的主要动机。 更多关于 H阿修罗 如何工作的信息可以在这篇[博客文章](ttps://code.facebook.com/posts/302060973291128/open-sourcing-haxl-a-library-for-haskell/)中查看。

Dataloader 不仅可以用于构建 Node.js 下的 GraphQL 服务，还可以作为一个公共参考实现的概念移植到其他开发语言。 如果您将 DataLoader 移植到了其他的语言平台，请在源码项目中提交一个带有您项目链接的 Issue。

## 起步



[GraphQL JS]: https://github.com/graphql/graphql-js