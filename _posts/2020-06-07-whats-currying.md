---
layout: post
title:  "What's Currying?"
date:   2020-06-07 13:20:01 +0800
categories: whats
tags: Currying
---

# 什么是柯里化？
本文要说的 curry 既不是咖喱，也不是 NBA 神射手，而是函数式编程中绕不开的一个概念：柯里化 (Currying)。

好，那什么是柯里化呢？先看 [Wiki 的定义](https://zh.wikipedia.org/zh-cn/%E6%9F%AF%E9%87%8C%E5%8C%96)：
> 在计算机科学中，柯里化（英语：Currying），又译为卡瑞化或加里化，是把接受多个参数的函数变换成接受一个单一参数（最初函数的第一个参数）的函数，并且返回接受余下的参数而且返回结果的新函数的技术。
这个技术由克里斯托弗·斯特雷奇以逻辑学家哈斯凯尔·加里命名的，尽管它是Moses Schönfinkel和戈特洛布·弗雷格发明的。

最后一句有点酸，咱们不管，看概念还是略显抽象，先来看一个几乎所有相关文章都会提到的一个例子：

```swift
// MARK: Normal

func add(_ x: Int, _ y: Int) -> Int {
    return x + y
}

add(1, 2) // (1) output: 3

// MARK: Currying

typealias I2I = (Int) -> Int
func add(_ x: Int) -> I2I {
    return { (y: Int) in
        return x + y
    }
}

let add1 = add(1)
add1(2) // (2) output: 3
add(1)(2) // (3) output: 3
```
为了方便说明，这里直接就用 swift 的 playground 来作为 REPL 了。

好，简单说明一下这个例子。定义了一个 `add` 方法，接收两个输入 `x`, `y`，返回它们的和，贼简单。

调用形式上就如注释 (1) 中，`add(1, 2)` 无须多言。

好了，那如果把 add 方法柯里化是一个什么样子的呢？

先看结果，调用形式就如同注释 (3) 中对应的样子了：`add(1)(2)`。即：分别调用参数 1 和 2，其中 add(1) 等同于 add1，也是一个函数，即固定 x 为 1，根据输入 y 返回 1 + y。这个转化定义在上面的代码中实现了，`add(_ x: Int) -> I2I`，借助闭包持有了上下文 x，返回一个接收 y 输入并求和的函数。

#### Currying vs Partial Application
几乎看到所有讲 Currying 的文章里都提到了一个叫 `Partial Application`（部分应用） 的辨析。本文也不免俗地简单说一下。

还是举例说明：假如上面 add 方法扩充为需要 3 个参数
`func add(x, y, z)`
柯里化的结果就是：`add(x)(y)(z)`

那部分应用是什么意思呢？
> Partial application: A function returning another function that might return another function, but each returned function can take several parameters.

 这个说明引用自文章 《[Functional Programming: Currying vs. Partial Application](https://medium.com/better-programming/functional-programming-currying-vs-partial-application-53b8b05c73e3)》

什么意思？大体上也是和 柯里化 一样，对于多个参数输入的函数，可以固定部分参数，返回其他参数作为输入的函数，而柯里化必须是严格要求：输入参数一个，返回的函数也得是只有一个输入参数。

继续拿上面 add(x,y,z) 的例子说就是，add(x)(y, z) 就算是一个部分应用了 x 的例子： add(x) 表示一个函数，这个函数的输入是 y,z 两个参数，返回 x + y + z。

# 柯里化作用
其实很多时候我们不理解一个概念主要还是搞不清这个概念到底能应用在什么地方。我在查阅文章的时候也是一直带着这个疑问，下面尝试结合学习的文章，总结一下自己的肤浅理解，欢迎指正。

通过上面 `add(x)(y)` 的例子我们算是认识了柯里化，但是对于这么写的好处其实并不是很能理解，我调用 `add(x, y)` 不是更加简单明了吗？干嘛还得中间再搞一层函数折腾得这么麻烦。

这里其实是有两个疑问的：
1. 为什么要把参数拆开？
2. 为什么要求拆成必须只有一个输入参数

下面尝试分别来解释一下：

#### 函数组合

还是先来看一个例子，意义上比上面的 add 要直观些。

代码如下：

```swift
// MARK: Composition

func decorate(pre: String, post: String, data: String) -> String {
    return "\(pre)\(data)\(post)"
}
decorate(pre: "A", post: "Z", data: "[123]") // (1) "A[123]Z"
```
 
定义了一个 `decorate` 函数，接收 3 个 String 参数，实现的效果就是给 data 加上前缀和后缀组成新的 String。
通常的调用方式就如同注释 (1) 所示

那好，如果要把这个函数柯里化，就分别按照加前缀，后缀的方式实现了 `head` 和 `tail` 两个函数，函数定义都是接收一个 String 返回一个重命名的 `Trans` 类型，也即一个函数类型。

```swift
typealias Trans = (String) -> String
func head(head: String) -> Trans {
    return { (data: String) in
        return "\(head)\(data)"
    }
}
func tail(tail: String) -> Trans {
    return { (data: String) in
        return "\(data)\(tail)"
    }
}

let headA = head(head: "A")
let tailZ = tail(tail: "Z")

headA(tailZ("[123]")) // (2) "A[123]Z"
```

基于此，就可以定义固定前缀和后缀的函数 `headA` 和 `tailZ`.
最终的调用方式如注释 (2)

有的同学说这个写法看起来也不是很方便嘛，如果再多加几个前缀后缀，岂不是要嵌套一堆？
没关系，借助 swift 的操作符重载，再优化一下代码看：
```swift
// MARK: infix
infix operator >>: FunctionArrowPrecedence // 可以读作 runAfter
func >>(left: @escaping Trans, right: @escaping Trans) -> Trans {
    return { (data: String) in
        return left(right(data))
    }
}

let dec = headA >> tailZ
dec("[123]") // (3) "A[123]Z"
```

借助中缀操作符 `>>` 处理后，可以通过 `headA >> tailZ` 的方式构造出新的转换函数 `dec`，然后如注释 (3) 处所示调用即可。

看到这里，相信`函数组合`的这个优点就已经很明显了，通过柯里化转换为中间函数，每个参数其实都对应了一个“函数族”，不同参数对应的函数通过组合的方式完成调用，大大增加了基本函数功能的可重用性，以及组合产生的多样性。

假设需要在 headA 之后再加一个 headB，只需要类似 headA 的方式定义 headB，然后通过 `>>` 装配为 `headA >> headB >> tailZ` 即可实现输出为 `“AB[123]Z”` 的效果。

代码读起来是不是也更加清晰直观呢。
至此，第一个疑问 `为什么要把参数拆开？` 就算是解释完了，函数组合链式调用绝对是主要作用。

#### Point-Free Style
那第二个疑问 `为什么要求拆成必须只有一个输入参数？` 是怎么回事呢？

说到这个，其实还要一个概念 `point-free` 也得提一下。

> Point-free style is a style of programming where function definitions do not make reference to the function’s arguments.

想干嘛呢？就是追求函数定义不引用函数的参数！

为啥要有这个洁癖？

其实看一下上节的例子也能有所感受。

headA, tailB 其实都是符合 `point-free` 风格的函数定义啊，好处是啥呢？看看组装的时候就知道了，`headA >> headB >> tailZ` 这还不够清清爽爽，清清楚楚嘛！
想象一下如果中间拆成了多个(非 1 个) 的函数形式，headA 可能得变成 `headA(post: String)` 这种了，那么拼装的时候还得去配置这个多余的参数，不好串联了嘛。

再抄一下[这篇文章](https://medium.com/javascript-scene/curry-and-function-composition-2c208d774983)里给出的示意：
```
f: a => b
g:      b => c
h: a    =>   c
```
h = f · g 很容易达到。但如果 g 输入需要 2 个参数：
```
f: a => b
g:     (x, b) => c
h: a    =>   c
```
这时候就串联不起来了，如果要搞，就得继续 柯里化 g 才行。

# 总结
好了，写到这里 `柯里化` 相关的事情基本上也就讲完了，因为实际工作中没有用过，所以开始看和理解起来都比较费劲。这里只是记录了我当下的理解，以便后续可以再来 review。

其实函数可以方便进行组装还有一个前提是其为 `纯函数`，有输入输出，没有副作用。本来打算试试看能否泛泛写一下函数式编程的内容，后来看得时候比较懵，所以先挑了`柯里化`这部分来学习。

其实不论面向对象还是函数式，都是对问题的抽象，只不过一个是通过 `对象` 之间交互的方式，一个是通过 `函数` 调用的方式。有一些基本原则都是通用的，比如单一职责，DRY(Don't Repeat Yourself)，高内聚低耦合，方便组合和扩展。但同时思考问题的方式和角度确是有很大差异，我看文章都有提到说一旦习惯函数式，感觉就是从低维度上到了高纬度，再想下来反而难了。

希望有一天我也能有这种体会。

# 参考资料
[柯里化 wiki](https://zh.wikipedia.org/zh-cn/%E6%9F%AF%E9%87%8C%E5%8C%96)

[Functional Programming: Currying vs. Partial Application](https://medium.com/better-programming/functional-programming-currying-vs-partial-application-53b8b05c73e3)

[Curry and Function Composition](https://medium.com/javascript-scene/curry-and-function-composition-2c208d774983)

[《函数式编程指北》第 4 章: 柯里化（curry）](https://llh911001.gitbooks.io/mostly-adequate-guide-chinese/content/ch4.html#%E4%B8%8D%E4%BB%85%E4%BB%85%E6%98%AF%E5%8F%8C%E5%85%B3%E8%AF%AD%E5%92%96%E5%96%B1)

[So You Want to be a Functional Programmer Part.1 ~ 6](https://medium.com/@cscalfani/so-you-want-to-be-a-functional-programmer-part-6-db502830403)

