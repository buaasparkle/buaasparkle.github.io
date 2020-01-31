---
layout: post
title:  "What's Markdown?"
date:   2020-01-31 23:12:58 +0800
categories: whats
tags: markdown
---

# 定义

Markdown 语言是在 2004 年由 [John Gruber][John Gruber] 发明的。

> Markdown is a lightweight markup language with plain-text-formatting syntax.

> from [Wikipedia Markdown][Markdown]

见文之意，两点：
- 是一门标记语言
- 用来给纯文本添加一些格式化的元素

还是太抽象，先来张图看看

![help_md](/assets/images/screenshot_macdown_help_md.png)

上图中的左边部分就是用 Markdown 标记语言编写的文档，右边就是其最终展示的样子。

好了，在有个直观印象的基础上再解释一下 `plain-text-formatting syntax` 是啥意思？

简单来说，就是你要给纯文本排版，比如设置个几级标题，高亮一些文字（加粗，斜体），插入个图片，URL 超链接什么的，还有代码段等等等等，你要怎么做呢？

设想你在用 Word 编辑器，得在软件里一通鼠标键盘操作，才能搞定这些。但是 Markdown 不，你只需要在纯文本中间插入一些**特殊**（比如图中的 `#/*/[link_text](url)`等）符号，就能实现这个事儿啦。

这些特殊的符号，就是 Markdown 的语法啦。

再说一句，你看到的 Markdown 文档，都是一群以 `md` 或者 `markdown` 作为后缀名的小朋友。

# Markdown vs WYSIWYG

问：[WYSIWYG](https://en.wikipedia.org/wiki/WYSIWYG) 什么鬼？

答：所见即所得

就拿上面的 Word 来解释一下，你对文档内容的所有修改，其最终呈现的状态，加粗了字体啦，段落设置为几级标题啦，你立马就能看见。

但是，Markdown 做不到啊～

我擦，既然有 WYSIWYG 这么牛逼的东东，Markdown 是怎么活下来，还混得不错的呢？

下面列一下其过人之处：

- 只要是文本编辑器都可以写 Markdown 文档。没装 Office Word 软件？没关系，抄起一个“记事本”就能撸起来。
- 用途广泛。不止写文档，Email，websites，eBook，presentation 都可以用 Markdown 来写。
- 平台无关。可以在运行着任意操作系统的任意设备上创建 Markdown 文本。

总儿言之，卸下特定软件的束缚，小巧简便的 Markdown 语言就像打不死的小强一样遍地开花啦。

# 原理（浅）

带`奇怪符号`的纯文本文档是怎么变成`格式化`的文档的呢？

不废话直接揭晓答案：

***Markdown file -> processor -> HTML***

是的，并没有特别的神奇，就是通过一个 processor（parser），将 markdown 文档转换成了 [HTML][HTML] 文件，这样就可以在浏览器中展示了。
对于支持 Markdown 的应用来说，其内部就自带了一个这个 processor 和展示 HTML 的引擎。

至于这个 Processor 是怎么实现的，我没有去看过，应该就是一个语法解析器，这里就先不深究了哈。

# 语法

看到这里想必你一定是蠢蠢欲动，想要学习一下这些语法了。

可以浏览 
- [Basic-Syntax][Basic]
- [Extended-Syntax][Extended]
- [Cheet Sheet][Cheet]

来速度学习一下，很快就可以照猫画虎了，本文就不赘述啦。

# 工具

工欲善其事必先利其器，说了这么多，有没有 Markdown Application 可以推荐来写 markdown 的？

笔者只用过几个：
- [MacDown][MacDown]（免费）
- [iA Writer][iA Writer]（收费）
- [Visual Studio Code][VS Code]（免费）

iA Writer 试用了一下（贫穷 😂），有 Focus 模式和语法高亮，营造了一种沉浸打字的感觉，还是挺香的。

MacDown 就比较朴实了，最开始的图例就是截图其 help.md 文档的样子，编辑和展示同步，也算所见即所得了。

VS Code 算是一个编辑器的代表了吧，可以高亮 Markdown 语法，还可以安装 MarkDown Preview Enhanced 的插件，通过 `⌘-k v` 快捷键可以在右侧同步展示 Preview，本文就是在这里敲的，也是非常方便了。

# 参考资料

[Markdown Guide](https://www.markdownguide.org/) 是个很不错的引导文档，这个网站的静态文档就是用 Markdown 写的，很多内容也是摘录自这里，更多信息可以自己逛逛看啦。

<!-- references -->
[John Gruber]: https://en.wikipedia.org/wiki/John_Gruber
[Markdown]: https://en.wikipedia.org/wiki/Markdown
[HTML]: https://zh.wikipedia.org/zh-hans/HTML
[Basic]: https://www.markdownguide.org/basic-syntax/
[Extended]: https://www.markdownguide.org/extended-syntax/
[Cheet]: https://www.markdownguide.org/cheat-sheet/
[MacDown]: https://macdown.uranusjr.com/
[iA Writer]: https://ia.net/writer
[VS Code]: https://code.visualstudio.com/