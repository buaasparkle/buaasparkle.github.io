---
layout: post
title:  "读书笔记 -《JavaScript DOM 编程艺术》"
date:  2020-12-13 10:32:39 
categories: coding
tags: JavaScript, DOM
---

# 书籍介绍

**英文名**：Web Design with JavaScript and the Document Object Model

**中文名**：JavaScript DOM 编程艺术

**作者**：
- [英] Jeremy Keith
- [加] Jeffrey Sambells

**出版时间**：2011 年

虽然在 2020 年的当下，公司 Web 同事们已在热火朝天地讨论 Angluar/React/Vue 三大框架，作为 Web 开发小白的我去读了下 2011 年的基于 JavaScript DOM 编程的基本知识，还是很有收获的。

# 基本概念

***BOM: Browser Object Model***
> 浏览器对象模型，比如 window

***DOM: Document Object Model***
> 一个与系统平台和编程语言无关的接口，程序和脚本可以通过这个接口动态地访问和修改文档的内容、结构和样式。 -- W3C

DOM 是一套标准，任何遵循这套标准的文档内容，都可以通过其制定的接口对文档进行访问修改，不限定在浏览器 HTML 上下文，对 XML 文件的读写也可以使用。

借助 DOM 就可以给文档增加交互能力，而使用的编程语言就是 -- JavaScript

***[HTML5]***
> 一个整装了结构层、样式层和行为层标准的集合

# 印象点

列出几个读完书印象比较深刻的点：

### 结构分离
- 结构层：HTML，描述了文档的内容的骨架
- 样式层：CSS，给骨架增光添彩
- 行为层：DOM & Javascript，提供了动态交互的能力

### 平稳退化
- 要考虑 JavaScript 禁用的情况下，页面也可以提供基本的可用性
> 思考哪些是页面结构必须的(良好的标记内容)，哪些是动态生成的，以及事件拦截的返回值处理等

- 不同浏览器 HTML-DOM 实现不同的情况下，保证接口的可用性
> check object exist before use

### 渐进增强
> 用一些额外的信息去包裹原始数据，按照“渐进增强”原则创建出来的网页几乎都符合“平稳退化”原则

类似上面的“结构分离”，在不影响你的基础上“额外”去做一些事情，行为可控。

### Ajax

Ajax 的主要优势是对页面的请求以异步方式发送到服务器，根据返回结果局部刷新页面，节省了整个页面刷新的开销。
Ajax 技术的核心就是 XMLHttpRequest 对象。

### JavaScript 库
大名鼎鼎如 jQuery，提供了非常多简化 DOM 操作的接口（还有 CSS 和 Ajax），看过书中基于 DOM 接口的操作和 jQuery 实现的对比之后，会理解这个库的初衷就是提高生产力。

<!-- refs -->
[HTML5]: https://en.wikipedia.org/wiki/HTML5

