---
layout: post
title:  "What's npm & yarn?"
date:   2020-02-08 12:39:46 +0800
categories: whats
tags: npm, yarn
---

# 0 前言
最近憋家里学习一下 React Native，尝试用它写一个小 App 用来管理一下密码（拯救一下自己经常忘记密码的老毛病🤦‍♂️），其中用到了 react-navigation 这个 module 来实现页面之间的导航，参照说明文档 install, 用的就是 npm 命令:

``` bash
$npm install @react-navigation/native
$npm install react-native-reanimated react-native-gesture-handler react-native-screens react-native-safe-area-context @react-native-community/masked-view
```

当然，如果一帆风顺可能就没有本文了，在执行到第二个命令的时候卡住了，确认不是网络问题后尝试分别执行第二个命令中的这些 module，发现是卡在 `react-native-gesture-handler` 这里了，没办法，开始 Google 有什么解决方法。

于是本文的另一个主角 yarn 就出现了，在病急乱投医的情况下，通过：
``` bash
$yarn add react-native-gesture-handler
```
居然很快安装上了，于是准备查一下 npm & yarn 到底是何方神圣。

各自的版本信息如下：
```bash
$npm -v
6.13.1

$yarn -v
1.19.2
```

# 1 npm

[npm](https://zh.wikipedia.org/wiki/npm) 的 WIKI 定义如下：
> npm（全称 Node Package Manager，即“node 包管理器”）是 Node.js 预设的、以 JavaScript 编写的软件套件管理系統。

这里有两个概念 `Node` 和 `Package` 都不太清楚，看来要深度优先搜索了(如果你已经熟悉可以跳过)。

#### 1.1 Node

在 [nodejs 主页](https://nodejs.org/en/) 就有其定义：

> Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine.

渣渣的世界就是痛苦啊，这个 `javascript runtime` 和 `javascript engine` 又是什么鬼🤦‍♂️。

- Javascript engine

Javascript 是一种解释型语言，也就是说它没有像 C 语言这种先通过编译器转换成本地机器码的过程，而是直接运行时解释执行。那好了，谁用来解释执行 Javascript 代码呢？就是这个 JavaScript Engine 啦。那这个 Chrome‘s V8 又是啥呢？Javascript 语言之前一个重要的应用场景就是在浏览器中做动态交互，那么浏览器中肯定得有一个 Javascript engine 来解释执行啊，对于 Chrome 浏览器，默认用的就是一个叫 V8 的引擎。

同理，还有其他浏览器也有各自的引擎，列一下这些引擎：
- **V8**: Chrome，NodeJS，Opera 使用，C++ 实现的开源项目。
- **SpinderMonkey**： FireFox 使用，Mozilla 维护，也是 C++ 实现，开源。
- **Nitro**：苹果爸爸的 Safari 中使用。
- **Chakra**：微软为 Edge 浏览器开发的引擎。

说到这里可能会有同学担心，各家都有自己的实现，会不会用起来接口不一致啊，适配多个浏览器岂不是蛋疼。

别急，各 Javascript engine 都是依照 ECMAScript 提供的[说明文档](https://www.ecma-international.org/publications/standards/Ecma-262.htm)来实现的，所以基本上都是一致的。不过由于 ECMAScript 最近的改动较之前要多些，所以各端的实现节奏可能略有不同。

- Javascript Runtime

好了，说完了 Javascript engine，再来看看 Javascript runtime。还是以浏览器为背景介绍，Web 开发同学可以通过 Javascript 来操作 DOM（Document Object Model），发 HTTP 请求，AJAX 等等来实现一些客户端的交互行为。但是，**敲黑板**，这些能力都不是 Javascript 本身自带的，而是浏览器的 JavaScript runtime 环境提供的，可能不太准确的说法是有点像一个 SDK，可以给开发同学提供可编程的库和接口。

总结一下，NodeJs 就是一个基于 Chrome V8 引擎开发的一个 JavaScript 编程环境（runtime），不同于传统浏览器中的 runtime，Nodejs 可以用 Javascript 来方便高效的搭建 Web Application (一个比较有名的模块就是 Expressjs)，支持服务端的开发。
至此，Javascript 语言可以说是前后端通吃了，有一个叫 [MEAN](https://en.wikipedia.org/wiki/MEAN_(software_bundle)) 的全家桶，感兴趣的同学可以去研究研究。

没说清楚的还可以移步读读👇这几篇文章，我也是参考其中的内容总结的。

[The JavaScript runtime environment](http://dolszewski.com/javascript/javascript-runtime-environment/)

[The Javascript Runtime Environment](https://medium.com/@olinations/the-javascript-runtime-environment-d58fa2e60dd0)

[What is Node.js? The JavaScript runtime explained](https://www.infoworld.com/article/3210589/what-is-nodejs-javascript-runtime-explained.html)

#### 1.2 Packages and Modules

在 [npmjs 网页](https://docs.npmjs.com/about-packages-and-modules) 找到了 `Package` 和 `Module` 的释义辨析，npm 注册中心包含的都是 package，很多这些 package 本身就是 module，或者包含 module，快一起来看一下。

- packages

> A package is a file or directory that is described by a package.json file. A package must contain a package.json file in order to be published to the npm registry.

还有一个关于 package 的定义也挺有意思的

> A package is any of the following:
> 
> a) A folder containing a program described by a package.json file.

> b) A gzipped tarball containing (a).

> c) A URL that resolves to (b).

> d) A <name>@<version> that is published on the registry with (c).

> e) A <name>@<tag> that points to (d).

> f) A <name> that has a latest tag satisfying (e).

> g) A git url that, when cloned, results in (a). 

- modules

> A module is any file or directory in the node_modules directory that can be loaded by the Node.js require() function.

> To be loaded by the Node.js require() function, a module must be one of the following:
> 
> - A folder with a package.json file containing a "main" field.

> - A folder with an index.js file in it.

> - A JavaScript file.

简单总结一下： `package` 是符合 package.json 描述的文件或文件夹，而 `module` 主要是用来满足可被 Node.js require() 方法引用，形式上可能是个 package 也可能不是（文档中 NOTE 处有说明不要求有 package.json), 除此之外还有满足 **"main" field** 和  **index.js** 的要求。

#### 1.3 npm 基础

绕了这么一大圈，总算明白了 npm 就是一个围绕着 nodejs runtime 开发的一套包管理中心，全球各地的开发者可以将自己开发的功能打成 package 发布到 npm 中心供其他人下载使用，也就此成就了 npm 为全球最大的软件注册表的地位（包含超过 600000 个 包）。

npm有 3 个组成部分：
- the website
- the command line interface (CLI)
- the registry

前言中用到的就是 npm 的 CLI 去下载其他开发者添加到 registry 中的包，那这个 website 又是啥呢？

接着上面 60,000 多个 package 的上下文说，既然有这么多包，我该怎么查找呢？

[npmjs](https://www.npmjs.com/) 就是这个 `website`，有点像一个搜索引擎，输入你想要查询的关键字，就可以找到你想要的包了。

不过，有的同学说这个不好用，检索匹配不好，而且还没有筛选(比如希望一些 Github 中高星项目排前面），于是就有了 [npms](https://npms.io/)。

哪家强我也不好说，毕竟之前不是 Web 开发，等用的时候大家就都试试好了。

👇想再解释几个概念。

- [`scope`](https://docs.npmjs.com/about-scopes)

当你注册好一个 npm 的账号后，你就被授权了一个 `scope` 来表示你的用户名或者组织的名字，这个 scope 就像一个 namespace，这样即使大家的 package 重名了也没有关系，因为 scope 是不同的，如果你见过类型这样的命令：
```bash
$npm install @react-navigation/native
```
这里的 react-navigation 就是 scope 名称啦。

scope 和 package 的可见性有以下关系：
> Unscoped packages are always public.

> Private packages are always scoped.

> Scoped packages are private by default; you must pass a command-line flag when publishing to make them public.

- [`Package.json`](https://docs.npmjs.com/creating-a-package-json-file)

如前文所说，要想发布到 npm registry，项目中必须要有一个 package.json 文件。
package.json 文件中有两个 field 是必不可少的:

`name`: package 的名称，必须是小写的一个词（可以用 - 或者 _ 连接）

`version`: 必须是 x.x.x 的形式，要符合 [semantic versioning guidelines](https://docs.npmjs.com/about-semantic-versioning)

更多规则细节可以看[这里](https://docs.npmjs.com/files/package.json)

举两个依赖版本号中 `~（Tilde）` 和 `^（Caret）` 符号的例子:
1. ~1.2.0 up to < 1.3.0
1. ^1.2.0 up to < 2.0.0

文档可以手动创建，或者通过命令回答一系列问题后生成，如果图省事想直接用默认值，可以在 init 后面加上 `--yes` 参数
```bash
$npm init --yes
```

关于如何发布一个 package 到 registry，可以参考官方的文档 [Creating and publishing unscoped public packages](https://docs.npmjs.com/creating-and-publishing-unscoped-public-packages)，这里只挑了发布 unscoped 的公共包，鉴于我自己也没有试过，就列一个最简单的在这里，如有需要可以再查看其他官方 Guide 文档。

- Cheet Sheet

CLI 命令这里就不赘述了，找了个 [cheet sheet](https://devhints.io/npm) 的链接，有需要的时候再查也来得及。

# 2 yarn

说完了 npm 现在来看看 yarn。

#### 2.1 介绍

首先，别惊讶，yarn 本身就是 npm 的一个 package。[链接在此](https://www.npmjs.com/package/yarn)

好，看看介绍是怎么说的：

> Fast, reliable, and secure dependency management.

嗯嗯，也是一个包依赖管理系统，它自己介绍说有这几个优点：

- Fast

yarn caches every package it has downloaded, so it never needs to download the same package again. It also does almost everything concurrently to maximize resource utilization. This means even faster installs.

- Reliable

Using a detailed but concise lockfile format and a deterministic algorithm for install operations, yarn is able to guarantee that any installation that works on one system will work exactly the same on another system.

- Secure

yarn uses checksums to verify the integrity of every installed package before its code is executed.

`Feature` 我就不赘述了，大家去说明文档里看吧，有趣的是 yarn 还有很多 emoji，下载的时候也不会觉得无趣。

#### 2.2 安装

安装请直接参考链接 [Installation](https://classic.yarnpkg.com/en/docs/install#mac-stable)

#### 2.3 yarn workflow

[yarn workflow](https://classic.yarnpkg.com/en/docs/yarn-workflow) 这个文档介绍了使用 yarn 的基本流程，从创建项目，添加/删除依赖，安装/卸载，版本管理到持续集成，简洁清晰，强烈推荐！

- 想从 npm 迁移到 yarn？贼简单：

```bash
$yarn init
```
更多信息请参看👉：[Migrating from npm](https://classic.yarnpkg.com/en/docs/migrating-from-npm)

- 使用命令更是相当简洁，不赘述，参考👉：[Usage](https://classic.yarnpkg.com/en/docs/usage)

- 更多 CLI 命令见👉：[CLI Introduction](https://classic.yarnpkg.com/en/docs/cli/)

- 依赖版本的表示方式👉：[dependency-versions](https://classic.yarnpkg.com/en/docs/dependency-versions) 这个文档说的很清楚！

- 关于创建 package.json 及发布看👉：[create/publish](https://classic.yarnpkg.com/en/docs/creating-a-package)

- 关于 yarn.lock 见👉：[yarn.lock](https://classic.yarnpkg.com/en/docs/yarn-lock)，重要说明几点👇：
1. 自动生成的，不要手动修改
1. 类似于 npm 的 `npm-shinkwrap.json`
1. install 期间只用 top-level 的  yarn.lock，子项目中的 lock 文件将被忽略
1. 需要添加到版本控制里去

#### 2.4 包搜索

[yarn](https://yarnpkg.com/) 的官方网站首页就有个搜索栏可以直接查询

# 3 npm vs yarn

本来写到这里才应该是重头戏，结果我看 yarn 的自我介绍的时候感觉全都说完了，毕竟嘛，如果 npm 在这几方面做的够好的话，也不会横空出世个 yarn 了，所以再啰嗦一下：

1. npm 是 node package 管理系统，yarn 本身也是一个 package 添加到 npm registry 中的，用来解决其依赖管理的不足。
1. yarn 并行下载有缓存，支持离线，快！
1. 有 yarn.lock 文件可以锁死版本，结合一致性算法保证不同系统上安装版本是统一的。

完！

# Reference
[npm 官方网址](https://www.npmjs.com/)

[yarn 官方网址](https://yarnpkg.com/)

[nodejs 官方网址](https://nodejs.org/en/)
