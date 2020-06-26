---
layout: post
title:  "What's RePlugin (1)?"
date:   2020-06-26 10:18:25 +0800
categories: whats
tags: RePlugin
---

# 简介

[RePlugin-GitHub]

> RePlugin是一套完整的、稳定的、适合全面使用的，占坑类插件化方案，由 360 手机卫士的 RePlugin Team 研发，也是业内首个提出”全面插件化“（全面特性、全面兼容、全面使用）的方案。

### 卖点

- 仅有一处针对 ClassLoader 的 Hook 点
- 借助 Gradle 插件完成 Manifest 中组件预埋
- 易集成
- 有成熟稳定的“插件管理方案”
- 线上庞大用户验证，稳定

### 特性

这里还是援引 [RePlugin-GitHub] 中的介绍：

| 特性 | 描述 |
| --- | :---:  |
| 组件 | 四大组件（含静态Receiver）|
|升级无需改主程序Manifest |	完美支持 |
|Android特性 |	支持近乎所有（包括SO库等）|
|TaskAffinity & 多进程	| 支持（坑位方案）|
|插件类型 |	支持自带插件（自识别）、外置插件|
|插件间耦合 |	支持Binder、Class Loader、资源等|
|进程间通讯 |	支持同步、异步、Binder、广播等|
|自定义Theme & AppComat | 支持|
|DataBinding |	支持|
|安全校验 |	支持|
|资源方案 |	独立资源 + Context传递（相对稳定）|
|Android 版本 |	API Level 9+ （2.3及以上）|

### 项目结构

当你 Clone 完 RePlugin 项目后，其目录内容如下：

![项目结构](/assets/images/replugin/project_structure.jpg)

红色框住的即为项目的核心实现部分。

如果按适配项目分：`host` 和 `plugin` 表示分别应用到宿主和插件的部分。

如果按实现方式分： `gradle` 和 `library` 分别表示的是 gradle 插件以及 java 库实现部分。

这里简单提一下 Gradle 插件部分的作用，也算是为后面要讲的内容做一点铺垫。简而言之，插件肯定是想默默帮你搞定一些事情的，即：

- `host-gradle`: 主要作用就是默默将“占坑”组件预埋到宿主的 AndroidManifest 文件中备用。

- `plugin-gradle`: 主要作用就是替换插件中的 Activity/Fragment 为 RePluginActivity/RePluginFragment。

先有个印象就行。

### 内容提纲

RePlugin 的特性和作用可能看一看介绍就能了解个大概了，不过深入到代码阅读的时候还是有些一头雾水，毕竟 README 不会告诉你整个架构的设计以及其中的诸多细节。

所以笔者也想分多篇文章来介绍 RePlugin 的源码，也算是遵循个人的疑问/学习过程，目前想到的是分为 4 步：

1. 仅一处 Hook 点在哪里？
1. 如何实现插件 Activity 启动的？
1. 插件是如何管理的？
1. Gradle 插件都做了什么？

本文先试试看能不能把 1 说清楚。

# 唯一 Hook 点

RePlugin 只有一处 Hook 点 —— 那就是 hook 住了宿主的 classloader。

如果你之前没有了解过插件化 Hook 技术，一定会觉得这玩意真点黑科技，往下看完你可能会觉得说实际上也没那么复杂。

如果要 Hook 宿主的 classloader，这个时机肯定是越早越好，不然一堆类都陆陆续续加载出来了，还 hook 个屁啊。众所周知，对于一个应用来说，最早的启动时机肯定是在 `Application` 里。没错，RePlugin 挑选的下手地点就在 Application 的 `attachBaseContext(Context)` 方法里。

***文件：RePlugin.App***
```java
/**
  * （推荐）当Application的attachBaseContext调用时需调用此方法 <p>
  * 可自定义插件框架的行为。参见RePluginConfig类的说明
  *
  * @param app Application对象
  * @see Application#attachBaseContext(Context)
  * @see RePluginConfig
  * @since 1.2.0
  */
public static void attachBaseContext(Application app, RePluginConfig config) {
    if (sAttached) {
        if (LogDebug.LOG) {
            LogDebug.d(TAG, "attachBaseContext: Already called");
        }
        return;
    }

    RePluginInternal.init(app);
    sConfig = config;
    sConfig.initDefaults(app);

    IPC.init(app);

    // 打印当前内存占用情况
    // 只有开启“详细日志”才会输出，防止“消耗性能”
    if (LOG && RePlugin.getConfig().isPrintDetailLog()) {
        LogDebug.printMemoryStatus(LogDebug.TAG, "act=, init, flag=, Start, pn=, framework, func=, attachBaseContext, lib=, RePlugin");
    }

    // 初始化HostConfigHelper（通过反射HostConfig来实现）
    // NOTE 一定要在IPC类初始化之后才使用
    HostConfigHelper.init();

    // FIXME 此处需要优化掉
    AppVar.sAppContext = app;

    // Plugin Status Controller
    PluginStatusController.setAppContext(app);

    PMF.init(app); // 👈 注意看这里!
    PMF.callAttach();

    sAttached = true;
}
```

👆这段代码就是 RePlugin 启动的主流程

- `RePluginConfig` 是你构造好的配置

- `IPC` 用于“进程间通信”的类。
    - 插件和宿主可使用此类来做一些跨进程发送广播、判断进程等工作

- `PMF` 框架的主程序接口代码。
    - PMF 我理解是 `Plugin Manager Framework` 的缩写

`PMF.init(app)` 主要做的是插件管理相关的初始化工作，默认是启动一个独立进程负责维护所有插件的信息的，这里就包括借助 Binder 完成进程间通信的桥梁，具体后面再讲。

***文件: PMF***
```java
public static final void init(Application application) {
    setApplicationContext(application);

    PluginManager.init(application);

    sPluginMgr = new PmBase(application);
    sPluginMgr.init();

    Factory.sPluginManager = PMF.getLocal();
    Factory2.sPLProxy = PMF.getInternal();

    PatchClassLoaderUtils.patch(application); // 👈 唯一 Hook 点！
}
```

初始化的最后一步就是激动人心的 hook classloader 了！

***文件: PatchClassLoaderUtils***
```java
public static boolean patch(Application application) {
    try {
        // 获取Application的BaseContext （来自ContextWrapper）
        Context oBase = application.getBaseContext();
        // 省略 oBase null check

        // 获取mBase.mPackageInfo
        // 1. ApplicationContext - Android 2.1
        // 2. ContextImpl - Android 2.2 and higher
        // 3. AppContextImpl - Android 2.2 and higher
        Object oPackageInfo = ReflectUtils.readField(oBase, "mPackageInfo");
        // 省略 oPackageInfo null check

        // mPackageInfo的类型主要有两种：
        // 1. android.app.ActivityThread$PackageInfo - Android 2.1 - 2.3
        // 2. android.app.LoadedApk - Android 2.3.3 and higher
        // 获取mPackageInfo.mClassLoader
        ClassLoader oClassLoader = (ClassLoader) ReflectUtils.readField(oPackageInfo, "mClassLoader");
        // 省略 oClassLoader null check

        // 外界可自定义ClassLoader的实现，但一定要基于RePluginClassLoader类
        ClassLoader cl = RePlugin.getConfig().getCallbacks().createClassLoader(oClassLoader.getParent(), oClassLoader);

        // 将新的ClassLoader写入mPackageInfo.mClassLoader
        ReflectUtils.writeField(oPackageInfo, "mClassLoader", cl);

        // 设置线程上下文中的ClassLoader为RePluginClassLoader
        // 防止在个别Java库用到了Thread.currentThread().getContextClassLoader()时，“用了原来的PathClassLoader”，或为空指针
        Thread.currentThread().setContextClassLoader(cl);
    } catch (Throwable e) {
        e.printStackTrace();
        return false;
    }
    return true;
}
```

代码中省略了部分 null check 和 日志输出，你会发现，原来要 hook 应用的 classloader 原来这么几行就够了！

代码中的注释已经比较清楚了，我这里就不赘述流程了。需要补充的有两个地方：

### 1. LoadedApk

*提问：LoadedApk 是啥？为啥要 hook 它的 classloader？*

先贴一下 `LoadedApk` 代码中的注释说明：

> Local state maintained about a currently loaded .apk.

对于 Framework 这部分的代码我也不熟，个人理解就是读 apk 文件到内存后保存其基本信息的数据结构。这里面的 classloader 默认是通过
`ClassLoader.getSystemClassLoader()` 初始化的，如果跟进去看，创建的就是 `PathClassLoader`。

OK，那为什么 Hook 住这里的 classloader 就 hook 到 “根” 上了呢？

Android 程序执行都离不开 `Context`，Context 中的 classloader 其实就是构造 `ContextImpl` 时传进来的 `LoadedApk` 的 classloader。

***文件: ContextImpl***
```java
@Override
public ClassLoader getClassLoader() {
    return mPackageInfo != null ? // 👈 mPackageInfo 的类型就是 LoadedApk
            mPackageInfo.getClassLoader() : ClassLoader.getSystemClassLoader();
}
```

那 Thread 的 classloader 呢？

看到 `Thread.setContextClassLoader` 源码中有这么一段注释：
> The context ClassLoader can be set when a thread is created, and allows the creator of the thread to provide the appropriate class loader, through {@code getContextClassLoader}, to code running in the thread when loading classes and resources.

搜索 `Thread.setContextClassLoader` 方法被调用的地方发现在 `LoadedApk.initializeJavaContextClassLoader()` 有调用，再往上追溯是在 `LoadedApk.makeApplication()` 方法中调用的。

***文件: LoadedApk***
```java
public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }

    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        java.lang.ClassLoader cl = getClassLoader();
        if (!mPackageName.equals("android")) {
            initializeJavaContextClassLoader(); // 👈 初始化 classloader，内部调用 Thread.setClassLoader
        }
        //  创建 ContextImpl, this 就是 LoadedApk
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext); // 👈 创建 Application
        appContext.setOuterContext(app);
    } catch (Exception e) {
        // 省略...
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app); // 👈 调用 Application 的 onCreate
        } catch (Exception e) {
            // 省略...
        }
    }
    // 省略部分代码

    return app;
}
```

我滴乖乖，原来 Application 也是在 LoadedApk 中造出来的啊！那是够早了～

跑偏一点哈，可能有同学跟我有一样的疑问，怎么创建 Application 后面没见 attach，就 `callApplicationOnCreate` 了呢？

跟进去看 `Instrumentation.newApplication()` 其实里面 new 完就紧跟着调 `Application.attach（）` 了，然后才是返回 App 对象。(`attachBaseContext` 是在 Application 的 attach 方法中)

嗯嗯，这样就跟之前的认知一样了。

兜回来，正如 RePlugin 注释中所说，处理 Thread 的 classloader 是为了：

> 防止在个别 Java 库用到了 Thread.currentThread().getContextClassLoader() 时，“用了原来的 PathClassLoader”，或为空指针

虽然我看代码这里的 classloader 一般情况下也是 LoadedApk 中的 classloader，但是不排除有的地方重新做了 setClassLoader 处理，所以再保护一下。


### 2 RePlugin.getConfig().getCallbacks().createClassLoader

上面说完了 Hook 点的选择，这里再看一下构造的 classloader。

`RePlugin.getConfig().getCallbacks().createClassLoader（）` 方法是有一个默认实现的，当然在配置 `config` 时也可以自定义，反正留了个口子在这儿。

***文件: RePluginCallbacks***
```java
/**
  * 创建【宿主用的】 RePluginClassLoader 对象以支持大多数插件化特征。默认为：RePluginClassLoader的实例
  * <p>
  * 子类可复写此方法，来创建自己的ClassLoader，做相应的事情（如Hook宿主的ClassLoader做一些特殊的事）
  *
  * @param parent   该ClassLoader的父亲，通常为BootClassLoader
  * @param original 宿主的原ClassLoader，通常为PathClassLoader
  * @return 支持插件化方案的ClassLoader对象，可直接返回RePluginClassLoader
  * @see RePluginClassLoader
  */
public RePluginClassLoader createClassLoader(ClassLoader parent, ClassLoader original) {
    return new RePluginClassLoader(parent, original);
}
```

下面来看一下 `RePluginClassLoader` 的初始化：

```java
// 文件: RePluginClassLoader
public RePluginClassLoader(ClassLoader parent, ClassLoader orig) {

    // 由于PathClassLoader在初始化时会做一些Dir的处理，所以这里必须要传一些内容进来
    // 但我们最终不用它，而是拷贝所有的Fields
    super("", "", parent);
    mOrig = orig;

    // 将原来宿主里的关键字段，拷贝到这个对象上，这样骗系统以为用的还是以前的东西（尤其是DexPathList）
    // 注意，这里用的是“浅拷贝”
    // Added by Jiongxuan Zhang
    copyFromOriginal(orig);

    // 👇 通过反射的方式获取 classloader 中的方法，以便调用原 classloder 中的方法
    initMethods(orig);
}
```

重点 Hook 部分当然是 `loadClass()` 方法的处理了，还是看源码：

```java
// 文件: RePluginClassLoader
@Override
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    //
    Class<?> c = null;
    c = PMF.loadClass(className, resolve); // 👈 先从插件里找
    if (c != null) {
        return c;
    }
    //
    try {
        c = mOrig.loadClass(className);
        // 只有开启“详细日志”才会输出，防止“刷屏”现象
        if (LogDebug.LOG && RePlugin.getConfig().isPrintDetailLog()) {
            LogDebug.d(TAG, "loadClass: load other class, cn=" + className);
        }
        return c;
    } catch (Throwable e) {
        //
    }
    //
    return super.loadClass(className, resolve);
}
```
处理的顺序为：先从 PMF 中找，找到就返回，如果没有继续用 Orig classloader 加载。

简单说就是：先从插件中找，再走默认。

从插件中找的内容大致可以分为两类：

1. 四大组件
1. 自定义 Hook 类

因为组件替换还是通过占坑的方式，所以真正启动时要把坑位组件替换为插件中的组件，这是其一。

另外，可以通过 `RePlugin.registerHookingClass` 方法注册一个跳转类，一旦系统或自身想调用指定类时，将自动跳转到插件里的另一个类。

关于组件替换的内容会在下一篇中介绍。 

# 小结

至此，RePlugin 框架中唯一一处 Hook 系统类私有成员的场景已经介绍完了。相对于其他 Hook Android 框架的实现，只 hook classloader 的兼容风险还是比较小的，越少 hook，越简单，越稳定。当然配合这仅有的 Hook 点所需要做的周边工作就会多些，比如 gradle 插件埋坑，处理坑位替换等，这个在下一篇介绍 Activity 如何启动中会再说明。

知道大概原理和动手实现还是相隔十万八千里的，`RTFSC`。

# 参考资料
[RePlugin][RePlugin-GitHub]

[RePlugin #高级话题](https://github.com/Qihoo360/RePlugin/wiki/%E9%AB%98%E7%BA%A7%E8%AF%9D%E9%A2%98)


<!-- reference -->
[RePlugin-GitHub]: https://github.com/Qihoo360/RePlugin/blob/dev/README_CN.md