---
layout: post
title:  "What's AIDL?"
date:   2020-05-31 17:59:35 +0800
categories: whats
tags: ARTS
---

# 前言
AIDL 最开始看 Android 的时候就知道个大概其，但是项目中一直没有用到过，最近看 [RePlugin]() 的源码讲解时有涉及，所以回过头来再看一下，从结果上看既温故又知新。

# What's AIDL?
A - Android

I - Interface

D - Definition

L - Language

# 干嘛用？
直接援引一下 Android 开发者网站 [AIDL](https://developer.android.com/guide/components/aidl) 页面中的定义：
> It allows you to define the programming interface that both the client and service agree upon in order to communicate with each other using interprocess communication (IPC).

OK. IPC - 进程间通信用。

从名字上看，AIDL 是一种接口定义，单单接口定义是怎么跟 IPC 扯上关系的呢？

还有， Android 进程间通信是通过一种叫 Binder 的机制实现的。那这个 AIDL 是不是和 Binder 有什么关系呢？

别急，先接着往下看。

# AIDL 长啥样
直接上代码
```
// IMath.aidl
package me.sure.mylib;

// Declare any non-default types here with import statements

interface IMath {

int add(int a, int b);
}
```

这文件定义在哪儿了？上图
![IMath.aidl](/assets/images/aidl/IMath.aidl.png)

好，解释一下。
定义 AIDL 首先需要先建立一个 aidl 文件，可以直接通过 New->Template(AIDL) 的方式创建。

内容方面，还是得使用 Java 语法定义接口，参数类型也是有要求的，基本数据类型都没问题，String/CharSequence 也都行，List/Map 带着基本数据类型或者 String 这种的也可以。如果要用自定义类型，得单独搞，这个后面说。

OK. 有了这个第一眼认识，AIDL 算是见过面了。跟 IDL 比对了一下，好像也是那么个意思。不过还是要问上面的问题，有了这玩意儿，怎么就能 IPC 了？

# Under The Hood
当你在项目中定义完 AIDL 文件，build 之后，在 generate 目录下就能找到一个同名的 Java 文件。
上图：
![IMath Generated](/assets/images/aidl/IMath.generated.png)

简单说，生成的文件有这么几点需要注意下：
1. IMath 继承了 `android.os.IInterface`
2. 内部定义了一个 `Stub` 不但实现了 IMath 接口，更重要的是它还继承了 `android.os.Binder`
3. 带实现的接口定义 `add(int, int)`

至此，真相大白，最终就是靠的 Binder 啊。关于 Binder，涉及到的内容也很多，不在本文中展开了。总结一下就是，AIDL 通过编译生成的接口定义文件，定义 Stub 继承了 Binder，接口本身又实现了 IInterface， IPC 的框架就这么搭好了。
多说一句参数传递，因为 Android 不同进程间肯定是不能互相访问内存的嘛，所以你要传什么参数进出，都要经过 [marshalling](https://en.wikipedia.org/wiki/Marshalling_(computer_science))，所以上文就列出了一下参数格式要求，AIDL 把这步也帮你做了。

# Client & Server
这里 IPC 通信的两端被赋予了 Client 和 Server 的角色。能力提供着是 Server 端，Client 来请求服务。连接方式是通过 bindService，server 端定义 service，返回 binder 对象（继承 aidl 中的）, client 端在 onServiceConnected 方法中获取到接口以便调用。

下面简单列一下基于上面 IMath.aidl 接口实现的 client/server 端部分代码：

Server 端：
![MathService](/assets/images/aidl/MathService.png)

AndroidManifest.xml 中 Service 定义：
```xml
<service
    android:name=".MathService"
    android:enabled="true"
    android:exported="true"
    android:process=":MathService">
    <intent-filter>
        <action android:name="me.sure.aidl.MathService" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</service>
```
如果跨进程调用的话，exported 需要为 true。

Client 端：
![MainActivity](/assets/images/aidl/MainActivity.png)

更多关于如何 BindService 的内容这里也不赘述了，参加开发者官网 [Bound services overview](https://developer.android.com/guide/components/bound-services)

# 自定义参数类型
在 demo 项目中添加了一个 Point(int x, int y) 数据结构作为 AIDL 接口中传递的自定义参数。

基本上要做 3 件事：
1. 自定义类型实现 `Parcelable` 接口
2. 定义 Point.aidl 文件声明 `parcelable Point;`
3. 在 IMath.aidl 中 import Point （即便在同一个包下也得 import）
```
import me.sure.mylib.Point;
```
代码最后会给出 Demo 地址，可以查看具体实现细节。

# 注意事项
IPC 需要注意执行对应的进程/线程问题，也是对照的官方文档总结如下：

1. 如果是本地调用 (call made from local process)，执行和调用所在线程相同。同一个进程嘛，就不用 IPC 了，如果使用场景都是这种情况，推荐直接在 Service 中定义 IBinder 对象返回，根本都用不着 AIDL。
> Calls made from the local process are executed in the same thread that is making the call. 

2. 如果是远程调用 (call from a remote process)，则是在平台维护在当前进程中的一个线程池中执行。因为不确定是哪个线程中处理，所以 AIDL 接口实现要保证线程安全。
> Calls from a remote process are dispatched from a thread pool the platform maintains inside of your own process.

3. RPC 调用默认情况下是同步的。所以，如果计算可能比较耗时，调用方就别在主线程做了，免得影响 UI。
> By default, RPC calls are synchronous.

还是列出 Demo 中的日志记录说明一下：
Client 在连接成功后触发两个 AIDL 接口调用 `add` 和 `pointAdd`.

```
private fun testMath() {
        Log.d(Tag.me, "[add] client thread: ${Thread.currentThread()}")
        val res = mathInterface?.add(1, 3)
        Log.d(Tag.me, "testMath: $res")

        Thread {
            Log.d(Tag.me, "[point add] client thread: ${Thread.currentThread()}")
            val point = mathInterface?.pointAdd(
                Point().apply { x = 1; y = 2; },
                Point().apply { x = 3; y = 4 }
            )
            Log.d(Tag.me, "test point add: $point")
        }.start()
    }
```

输入日志记录如下（包含 Service 部分）：
```
2020-05-31 17:16:18.350 12138-12138/me.sure.client D/--sure--: click
2020-05-31 17:16:18.351 12045-12045/me.sure.aidl:MathService D/--sure--: MathService onCreate()
2020-05-31 17:16:18.351 12045-12045/me.sure.aidl:MathService D/--sure--: MathService onBind(), Intent { act=me.sure.aidl.MathService pkg=me.sure.aidl }
2020-05-31 17:16:18.356 12138-12138/me.sure.client D/--sure--: onServiceConnected: ComponentInfo{me.sure.aidl/me.sure.aidl.MathService}
2020-05-31 17:16:18.358 12138-12138/me.sure.client D/--sure--: [add] client thread: Thread[main,5,main]
2020-05-31 17:16:18.358 12045-12066/me.sure.aidl:MathService D/--sure--: [add] server thread: Thread[Binder:12045_3,5,main]
2020-05-31 17:16:18.359 12138-12138/me.sure.client D/--sure--: testMath: 4
2020-05-31 17:16:18.360 12138-12184/me.sure.client D/--sure--: [point add] client thread: Thread[Thread-2,5,main]
2020-05-31 17:16:18.363 12045-12066/me.sure.aidl:MathService D/--sure--: [pointAdd] server thread: Thread[Binder:12045_3,5,main]
2020-05-31 17:16:18.363 12138-12184/me.sure.client D/--sure--: test point add: Point(x=4, y=6)

```

注意看：
1. IPC 是同步调用的。虽然两个接口是在不同 thread 调用的，但是从日志上看，都是 调用->返回，再调用->返回
2. Server 端处理 AIDL 接口调用所在线程为 `Thread[Binder:12045_3,5,main]`，不是主线程，在一个线程池里。

# 结语
至此，AIDL 大致就算介绍完了，内容虽然还不够细致，但是流程基本上都串下来了。其实不谈 Binder 的 AIDL 感觉就是刷流氓，没办法，Binder 我也只一知半解的，下次争取写个文章把 Binder 聊清楚。
Demo 代码列到最后了，感兴趣的同学可以 clone 下来跑跑看。

# 参考资料
[Android Developer AIDL](https://developer.android.com/guide/components/aidl)

[Bound services overview](https://developer.android.com/guide/components/bound-services)

[Android AIDL](https://android.jlelse.eu/android-aidl-937daf89e685)

[Android AIDL Deep Dive](https://medium.com/@budhdisharma/aidl-and-its-uses-in-android-e7a2520093e)

# Demo地址
[AIDL Demo](https://github.com/buaasparkle/AIDL)
