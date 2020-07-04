---
layout: post
title:  "What's RePlugin (2)?"
date:   2020-07-04 16:50:38 +0800
categories: whats
tags: RePlugin
---

# 内容提纲

在 [What's RePlugin (1)] 主要介绍了框架中的唯一一处 Hook 点的实现，这篇继续看一下 RePlugin 是怎么实现四大组件中的 Activity 是如何启动的。

# Activity 启动流程

看到一篇 [文章](https://juejin.im/post/5c6d0161f265da2dbc598603) 中总结的 Activity 启动的流程图很好，引用过来：

![Activity 启动流程](/assets/images/replugin/activity_start_workflow.jpg)

说来惭愧，我也是通过阅读“二手”资料大致学习 Activity 的启动流程，没有实际上对着源码看过，下面介绍有不对的地方，还请多多指正。

上图中的流程还有点长，我大概总结一下：

1. Activity 都归 ActivityManagerService 管，即 AMS，这个相当于服务器，我们要求他办事得借助 Binder，通过本地 Proxy 代理，属于跨进程通信。

1. 我们在 AndroidManifest 中声明的组件信息也是 AMS 来处理维护的，所以要启动没有声明的组件，AMS 是不答应的，而且 AMS 是系统维护，我们插不了手。

1. 流程上简述就是本地请求 AMS 启动指定 Activity，AMS 先检查，比如有没有事先声明过，权限等，然后处理任务栈，都没问题了告诉要启动的线程说你创建吧。

1. 可以关注的类一个是 `Instrumentation`，一个是`ActivityThread`，Instrumentation 是真正调用 Activity 初始化的地方，所以很多 hook 的方法都是拿它开刀；ActivityThread 是应用的主线程，你会发现 `main` 函数就定义在这里面，其中有个叫 `H` 的 Handler，处理一系列接收到的事件，其中 `LAUNCH_ACTIVITY` 事件就会最终让 Instrumentation 创建 Activity 并启动。

# RePlugin 启动 Activity

## 狸猫换太子

前面说了，没有在 AndroidManifest 中声明的 Activity 是过不了 AMS 这一关的，所以就有预埋“桩”位这种 “狸猫换太子” 的搞法。

大致流程如下：

1. 启动 “太子 A”（插件中 Activity）

1. 但是宿主中 Manifest 没有声明啊，这里只有“狸猫 X”，没关系，那就把 “太子 A” 和 “狸猫 X” 的对应关系记一下，告诉 AMS 启动 “狸猫 X”

1. AMS 一看要启动 “狸猫 X”，查了一下“清单”，发现里面是有的，批准启动

1. ActivityThread 收到启动命令，一看目标是 “狸猫 X”，查了一下之前对应的实际对象是 “太子 A”，好，那咱就创建 “太子 A” 并启动。

不同的框架在”换“这一步的实现略有不同，有 Hook `Instrumentation` 的，有 Hook `H` 的，还有 “寄人篱下”型，就是最后真启动了一个 `狸猫 X`, `太子 A`就委身于 狸猫 X 中，然后狸猫收到什么生命周期调用就告诉太子一声，这种情况下 太子 虽然号称是 Activity，实际上是个冒牌货，只能吃点 狸猫 的嗟来之食。

RePlugin 由于只有 Hook 系统 classloader 这一处黑科技，所以 “换” 太子自然也是通过这个 classloader 了。

## 桩位

先来速度看一下 RePlugin 的狸猫们，它们是借助宿主 的 gradle 插件来生成的，还可以通过 replugin 的 `repluginHostConfig` extension 来配置。

Gradle 插件这块还么看，后面再写，就简单看一下生成的 AndroidManifest 吧：

```xml
<activity
    android:theme="@ref/0x01030010"
    android:name="com.qihoo360.replugin.sample.host.loader.a.ActivityP1TA0STTS1"
    android:exported="false"
    android:process=":p1"
    android:taskAffinity=":t0"
    android:launchMode="2"
    android:screenOrientation="1"
    android:configChanges="0x4b0" />
```

上面只截取了 RePlugin Sample 项目中 Host 最终生成的 AndroidManifest.xml 中一个预埋 Activity，实际上有好几十个 Activity，有 theme/launchMode/orientation/透明度 等等的区别，命名中后缀的一堆 “PTAS01” 是什么意思还没有细看，大致理解一下是方便区分不同特性就行，后面介绍 Gradle 插件时再看。

## 启动

还是以 RePlugin Sample 为例，启动一个插件 Activity 的方式有：

```java
Intent intent = new Intent();
intent.setComponent(new ComponentName("demo1", "com.qihoo360.replugin.sample.demo1.activity.for_result.ForResultActivity"));
RePlugin.startActivityForResult(MainActivity.this, intent, REQUEST_CODE_DEMO1, null);
```

demo1 是插件名，ForResultActivity 是插件中定义的 Activity。

启动中涉及的类即方法见下图：

![RePlugin 启动 Activity 时序图](/assets/images/replugin/replugin_start_activity.png)

这里面主要部分在 `PluginCommImpl` 中获取插件的 ActivityInfo，并分配坑位，建立对应关系。

至于这里为什么跳来跳去的，以及这几个类分别对应什么作用，还没有搞清楚，留一个疑问后面继续学习。

`Activity.startActivityForResult` 方法最终会执行上面提到的 `Instrumentation` 中的方法来创建 Activity，即：

```java
// Instrumentation.java
public Activity newActivity(ClassLoader cl, String className, Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
    String pkg = intent != null && intent.getComponent() != null
            ? intent.getComponent().getPackageName() : null;
    // getFactory 对应 AppComponentFactory
    return getFactory(pkg).instantiateActivity(cl, className, intent);
}

// AppComponentFactory
public @NonNull Activity instantiateActivity(@NonNull ClassLoader cl, @NonNull String className, @Nullable Intent intent)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    return (Activity) cl.loadClass(className).newInstance();
}
```

而这里面的 classloader 正是上一篇中 Hook 系统的 `RePluginClassLoader`，继续看 cl 中具体怎么加载类的.

```java
// RePluginClassLoader.java
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    Class<?> c = null;
    c = PMF.loadClass(className, resolve);
    if (c != null) {
        return c;
    }
    try {
        c = mOrig.loadClass(className);
        return c;
    } catch (Throwable e) {
    }
    return super.loadClass(className, resolve);
}
```

`mOrig` 是原先的 classloader，所以在使用默认 classloader 加载前，先通过 PMF.loadClass 截上一道，后面的代码就不贴了，还是画个时序图来看：

![RePlugin ClassLoader 类加载时序图](/assets/images/replugin/replugin_classloader_load.png)

这一步就相当于根据之前建立的“太子-狸猫”对应关系，把“太子”再换回来，然后创建启动。

## Context

换是换过来了，不过还有一个问题没有解决，就是 Context!

这涉及到 2 个问题：
1. 插件内资源加载，需要用包含插件资源的 context
2. 再跳转到其他 activity 怎么办

这个就要看一下插件这边了，也是通过 Gradle 插件将插件中的 Activity 都替换为插件 library 中的 PluginActivity。

对比一下 demo1 中编译前后的 MainActivity，会发现将 Activity 都替换为了 `PluginActivity`:

```java
// 编译前
public class MainActivity extends Activity {
    // 省略实现
}

// 编译后
public class MainActivity extends PluginActivity {
    // 省略实现
}
```

那这个 `PluginActivity` 到底做了啥呢？代码不算多，我都贴过来

```java
/**
 * 插件内的BaseActivity，建议插件内所有的Activity都要继承此类
 * 此类通过override方式来完成插件框架的相关工作
 *
 * @author RePlugin Team
 */
public abstract class PluginActivity extends Activity {

    @Override
    protected void attachBaseContext(Context newBase) {
        newBase = RePluginInternal.createActivityContext(this, newBase);
        super.attachBaseContext(newBase);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        RePluginInternal.handleActivityCreateBefore(this, savedInstanceState);

        super.onCreate(savedInstanceState);

        RePluginInternal.handleActivityCreate(this, savedInstanceState);
    }

    @Override
    protected void onDestroy() {
        RePluginInternal.handleActivityDestroy(this);

        super.onDestroy();
    }

    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        RePluginInternal.handleRestoreInstanceState(this, savedInstanceState);

        try {
            super.onRestoreInstanceState(savedInstanceState);
        } catch (Throwable e) {
        }
    }

    @Override
    public void startActivity(Intent intent) {
        if (RePluginInternal.startActivity(this, intent)) {
            return;
        }

        super.startActivity(intent);
    }

    @Override
    public void startActivityForResult(Intent intent, int requestCode) {
        if (RePluginInternal.startActivityForResult(this, intent, requestCode)) {
            return;
        }

        super.startActivityForResult(intent, requestCode);
    }
}
```

代码都是一个套路，在调用 super 方法前先截一道，跟进去看的话会发现 `RePluginInternal` 其实是宿主和插件的一个桥梁，这里面通过反射的方式保存了宿主 `Factory2` 类中的同名方法，而宿主中因为负责加载插件，所以包含了插件需要的全部信息（包括 Plugin, Context, Resource 等），所以在 `attachBaseContext` 方法中创建插件的 Context (`PluginContext`), 在启动其他 activity 时还是通过宿主搞定。

那有的同学又会问，插件和宿主是怎么“勾搭”上的呢？

答案是：反射。

插件中的入口类：`com.qihoo360.replugin.Entry`，会传入宿主的 classloader 进行初始化。

关联时机：插件加载时会调用 `Plugin` 类中的 `loadEntryLocked` 方法，因为兼容的关系还会看到调用函数名如：`loadEntryMethod2`, `loadEntryMethod3` 之类的，具体代码就不贴了，感兴趣的同学可以去给出的类中看细节。

# 小结

至此，RePlugin 怎么启动 Activity 的过程就算是介绍完了，主要过程都是围绕着“狸猫换太子”在进行，其实对于很多实现细节来说，我也只能算是了解个大概其，也深感实现一个框架的不易，还需要多多学习～


<!-- references -->
[What's RePlugin (1)]: https://buaasparkle.github.io/whats/2020/06/26/whats-replugin-(1).html

[PlantUML 时序图]: https://plantuml.com/zh/sequence-diagram