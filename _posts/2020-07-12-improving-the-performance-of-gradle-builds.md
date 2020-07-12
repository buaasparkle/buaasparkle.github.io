---
layout: post
title:  "Improving the Performance of Gradle Builds"
date:   2020-07-12 14:09:02 +0800
categories: how
tags: Gradle
---

# 前言

最近在尝试对项目进行插件化改造，遇到一个比较蛋疼的问题就是项目编译龟速，插件编译一次 4m，主项目 3m，同时因为在实验阶段，所以经常要频繁修改，导致“带薪编译”时间过长，影响进展。

所以找了下 Gradle 构想优化的文章来看看，简单做一些记录，本文主要还是资料的搬运，缺少实践经验，有待后面应用之后再做更新。

# 要点记录

#### 用新的版本
因为版本更新通常都会做一些优化，同时 Gradle 又是兼容旧版本的，所以尽量用新版本，包括：
- Android Studio
- Android Gradle plugin
- Gradle version
- Android Build Tools

#### Build Variant
开发时针对不同的配置进行打包，可以避免编译一些不必要的依赖，避免“大水漫灌”。

#### 避免编译不必要的资源
如果有多个语言包，或者针对不同屏幕的图片，开发时指定一个就好嘛。

```groovy
android {
  ...
  productFlavors {
    dev {
      ...
      // The following configuration limits the "dev" flavor to using
      // English stringresources and xxhdpi screen-density resources.
      resConfigs "en", "xxhdpi"
    }
    ...
  }
}
```

#### Debug 构建时使用静态参数值

> Always use static/hard-coded values for properties that go in the manifest file or resource files for your debug build type.

比如使用动态版本号，或者改了版本明以及其他会影响到 Manifest 变化的改动，都会导致 `full apk build`!

```groovy
int MILLIS_IN_MINUTE = 1000 * 60
int minutesSinceEpoch = System.currentTimeMillis() / MILLIS_IN_MINUTE

android {
    ...
    defaultConfig {
        // Making either of these two values dynamic in the defaultConfig will
        // require a full APK build and reinstallation because the AndroidManifest.xml
        // must be updated.
        versionCode 1
        versionName "1.0"
        ...
    }

    // The defaultConfig values above are fixed, so your incremental builds don't
    // need to rebuild the manifest (and therefore the whole APK, slowing build times).
    // But for release builds, it's okay. So the following script iterates through
    // all the known variants, finds those that are "release" build types, and
    // changes those properties to something dynamic.
    applicationVariants.all { variant ->
        if (variant.buildType.name == "release") {
            variant.outputs.each { output ->
                output.versionCodeOverride = minutesSinceEpoch
                output.versionNameOverride = minutesSinceEpoch + "-" + variant.flavorName
            }
        }
    }
}
```

#### 使用静态依赖版本号
别用 `x.+` 这种，一个可能会引入异常的依赖，再有会增加 dependency resolve 的时间开销。

#### 创建 lib module
模块化允许构建系统只编译修改的模块，同时也会让并行编译更有效。

#### 图片优化
图片都转为 WebP，因为良好的压缩率，可以减少编译时的图片压缩开销。

Android Plugin 3.0 及以上，默认是在 debug 模式下关闭 `PNG crunching` 的。

#### 启用 Build Cache

Android Plugin 2.3.0 及以上是默认开启的。

#### 使用增量的注解处理

> Android Gradle plugin 3.3.0 and higher improve support for incremental annotation processing

给出了一些开源库的例子，还没有看。
- Data Binding Library with Android Gradle plugin 3.5.0 and higher
- Glide 4.9.0 and higher
- Dagger 2.18 and higher
- Auto Service and Auto Value 1.6.3 and higher

<!-- 待确认
#### Java Compiler daemon

> The Gradle Java plugin allows you to run the compiler as a separate process by using the following configuration for any JavaCompile task:

build.gradle
```groovy
<task>.options.fork = true
```

> or, more commonly, to apply the configuration to all Java compilation tasks:

build.gradle
```groovy
tasks.withType(JavaCompile) {
    options.fork = true
}
```

新创建的进程会在编译期间被复用，所以开销还好。同时对于 `memory-intensive` 编译，使用不同进程就会减轻主 Gradle daemon 的 GC 开销。

使用自己的一个项目测试，在已经 clean build 的情况下只修改版本号，配置 fork 之后大幅缩短了时间，如下面 2 图可以看一下并行任务，后者明显更优：

**未开启 Fork**
![只修改版本号](/assets/images/gradle_build/gradle_build_only_change_version.png)

**开启 Fork**
![只修改版本号且开启Fork](/assets/images/gradle_build/gradle_build_only_change_version_with_fork.png)

 -->

# 构建 Profile

#### 离线

```shell
gradlew --profile --offline --rerun-tasks assembleFlavorDebug
```

执行完毕后会在 `project-root/build/reports/profile/` 中生成报告，可以用浏览器打开。

图见 [Optimize your build speed]


#### Build Scan

Gradle 4.3+ 版本以上可以通过以下命令获取构建过程的全部信息

```groovy
gradle build --scan
```

build 完成后会先问你是否接受 [Gradle Terms of Service](https://gradle.com/terms-of-service)，同意后就会生成一个报告链接，如果链接 3 个月内都没有访问，到期后失效，符合会永远保留。

Gradle github 提供了一个 [demo](https://github.com/gradle/gradle-build-scan-quickstart)，执行后得到的链接是： https://scans.gradle.com/s/5ftv3j5czcq4i

进入链接就可以查看很多构建信息：
- task 数量 & 各阶段耗时
- test 运行情况
- 依赖数量
- 引用了哪些 plugin
- 一些功能开关开启情况
- 设备信息

目前很多信息还没太明白可以怎么使用以便优化，有待继续学习。


# 参考资料

[Improving the Performance of Gradle Builds]

[Optimize your build speed]

[Accelerate clean builds with the build cache]

[Get started with build scans]

<!-- refs -->
[Improving the Performance of Gradle Builds]: https://guides.gradle.org/performance/#apply_plugins_judiciously

[Optimize your build speed]: https://developer.android.com/studio/build/optimize-your-build.html

[Accelerate clean builds with the build cache]: https://developer.android.com/studio/build/build-cache

[Get started with build scans]: https://scans.gradle.com/?_ga=2.244911431.312171665.1594432203-462338920.1580176149#gradle