---
layout: post
title:  "What's JVM: Loading, Linking and Initializing"
date:   2020-06-14 16:51:32 +0800
categories: whats
tags: JVM
---

# 概念
在 JVM 执行 bytecode 之前，其实是要经过题目所说的 `类加载`、`链接`、`初始化` 这 3 个步骤的。那这 3 个步骤都是干嘛的呢？先贴一下 [JVM Spec](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-5.html) 中的介绍：

**Loading**
> Loading is the process of finding the binary representation of a class or interface type with a particular name and creating a class or interface from that binary representation

**Linking**
>  Linking is the process of taking a class or interface and combining it into the run-time state of the Java Virtual Machine so that it can be executed.

其中，Linking 中又包含 3 个小步骤：
1. Verify
> Verification checks that the loaded representation of Class is well-formed, with a proper symbol table. Verification also checks that the code obeys the semantic requirements of the Java programming language and the Java Virtual Machine.

2. Prepare
> Preparation involves allocation of static storage and any data structures that are used internally by the implementation of the Java Virtual Machine, such as method tables. 

3. Resolution
> Resolution is the process of checking symbolic references from loaded class to other classes and interfaces

**Initialization**
> Initialization of a class or interface consists of executing the class or interface initialization method `<clinit> `

贴了一大堆，简单梳理一下：

首先，要执行一个类，比如叫 Test，得先把这个类编译生成得字节码加载到 JVM 中吧，毕竟巧妇难为无米之炊。

好了，通过 classloader 加载进来，初始化一下就能用了吧？

别急，跳步了，中间还得有一步链接。bytecode 都是二进制数据，加载到 JVM 里之后得转换成执行时可用得这么一个状态才行，这其中又得先过一下安检 `verify`，格式结构得符合要求了才行，之后就是给你分配空间，就相当于先给你一个空箱子，告诉你里面会装某个类型得数据，但是不告诉你值是多少，都默认值。Resolution 这步是 Optional 的，就是说如果加载的这个类里有引用了其他类或者接口，那得处理一下他们得符号引用（这块具体做啥还没太懂）。最后才是初始化，给箱子里装具体得东西。

贴张图（引用自[这里](https://www.programcreek.com/2013/04/what-can-you-learn-from-a-java-helloworld-program/)）：
![loading_lingking_initializing](/assets/images/jvm/jvm-load-link-initialize.png)

# 例子
概念上面就算介绍完了，平淡无味，还是来看看几个例子感受一下初始化的顺序。

#### Hello
上代码👇

```java
class Hello {

	//instance variable initializer
	String s = "abc";

	//instance initializer
	{
		System.out.println("instance initializer called, s = " + s);
	}

	//constructor
	public Hello() {
		s = "xyz";
		System.out.println("constructor called, s = "  + s);
	}

	//static initializer
	static {
		System.out.println("static initializer called");
	}

	public static void main(String[] args) {
		System.out.println("Hello World!");
		new Hello();
	}
}

```

猜一下输出会是啥？

```shell
ᐅ javac Hello.java
ᐅ java Hello
static initializer called
Hello World!
instance initializer called, s = abc
constructor called, s = xyz
```

不知道是否和你预期的一样。

首先静态代码段 static 中先输出了.

然后 main 方法执行，打印出 Hello World!

再之后是实例初始化器中调用，变量值 s = abc

最后是构造函数执行，修改变量 s 为 xyz

**可是，实际运行的顺序是否就如同日志输出的一致呢？**

纳尼？！你是说日志还会骗人？

别急，咱们再来看一个例子

#### Hello & Parent

**Parent.java**
```java
class Parent {

	//instance variable initializer
	String s = "p-abc";

	//constructor
	public Parent() {
		System.out.println("parent constructor called");
		doInConstructor();
	}

	protected void doInConstructor() {
		System.out.println("parent doInConstructor");
	}

	//static initializer
	static {
		System.out.println("parent static initializer called");
	}

	//instance initializer
	{
		System.out.println("parent instance initializer called, s = " + s);
	}
}
```

**Hello.java**
```java
class Hello extends Parent {

	//instance variable initializer
	String s = "abc";

	//instance initializer
	{
		System.out.println("instance initializer called");
	}

	//constructor
	public Hello() {
		System.out.println("constructor called, s = " + s);
	}

	protected void doInConstructor() {
		System.out.println("doInConstructor s = " + s);
	}

	//static initializer
	static {
		System.out.println("static initializer called");
	}

	public static void main(String[] args) {
		System.out.println("Hello World!");
		new Hello();
	}
}
```

新加了一个 `Parent` 类，Hello 继承 `Parent`，也都是初始化过程加了一些打印输出，有一点变化的地方在于父类构造函数中调用了 `doInConstructor` 方法，子类重写了这个方法。

再猜一下这次的输出会是什么呢？

```shell
ᐅ javac Hello.java Parent.java
ᐅ java Hello
parent static initializer called
static initializer called
Hello World!
parent instance initializer called, s = p-abc
parent constructor called
doInConstructor s = null
instance initializer called
constructor called, s = abc
```

static 代码块还是在 Hello World 之前输出了，而且 parent 是在 hello 前面。

重点看一下后面，父类的初始化器调用，输出值 `p-abc`，之后是父类的构造函数调用输出，在构造函数中调用 `doInConstructor` 方法，此方法被子类重写，所以父类的实现并没有执行，执行的是子类的。

注意看！这里 s 的值是 `null`，后面才跟了子类的初始化器调用输出，最后是构造函数，此时 s 有值为 `abc`。

哎，按照第一个例子的日志，`Hello` 类的初始化器不是比构造函数早嘛？怎么这里是跟着父类构造函数结束调用，而且值还是 `null` 呢？

#### 真相

聪明的读者们看到这里是不是都已经猜到了？

没错，真相只有一个！

实际的调用顺序其实是这样的：

```
- Hello 构造函数 Start -> 

    - Parent 构造函数 Start -> 

        Parent 实例初始化器完成初始化并打印输出

        构造函数中 System.out.print 输出

        调用 doInConstructor -> 子类重写实际执行，此时子类初始化器还未执行，所以 s 为 null

    - Parent 构造函数 End

    Hello 实例初始化器调用（初始化 s 为 abc）

    Hello System.out.print 输出

- Hello 构造函数 End
```

是说这些 `instance initilizer` 都是隐藏节目？在构造函数开始和我们在里面实际写的第一行代码之间做的？

#### Bytecode

空口无凭，咱们反编译一下字节码看看。

**Hello.class**
字节码比较长，重点看一下 Hello 构造函数：

```
public Hello();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method Parent."<init>":()V
         4: aload_0
         5: ldc           #2                  // String abc
         7: putfield      #3                  // Field s:Ljava/lang/String;
        10: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        13: ldc           #5                  // String instance initializer called
        15: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        18: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        21: new           #7                  // class java/lang/StringBuilder
        24: dup
        25: invokespecial #8                  // Method java/lang/StringBuilder."<init>":()V
        28: ldc           #9                  // String constructor called, s =
        30: invokevirtual #10                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        33: aload_0
        34: getfield      #3                  // Field s:Ljava/lang/String;
        37: invokevirtual #10                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        40: invokevirtual #11                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        43: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        46: return
      LineNumberTable:
        line 12: 0
        line 4: 4
        line 8: 10
        line 13: 18
        line 14: 46
```

下面 `LineNumberTable` 中的行数对应的是 java 源码中的位置，作为 Debug 信息使用，也可以对照着看。

忽略左边的 `opcode`，直接看右侧的注释也看清。

首先调用 Parent 构造函数（1），然后常量 abc 入栈赋值（5, 7），后面是对应这段代码（10, 13, 15）
```
//instance initializer
{
    System.out.println("instance initializer called");
}
```
再后面就是拼接字符串 System out 打印输出了。

至此，这个初始化的顺序算是水落石出了。

# 结语
第一次看到这个结果的时候也是有点意外，因为实际开发中真的不太涉及这种底层细节，而且初始化对象都是构造完成之后用，也不太会涉及这样的问题。

记得看 Swift 初始化时有个 `Two-Phase Initialization` 中也有点 prepare 再 initialize 的意思。不过 Swift 还很不熟悉，留作后续验证。

> Class initialization in Swift is a two-phase process. In the first phase, each stored property is assigned an initial value by the class that introduced it. Once the initial state for every stored property has been determined, the second phase begins, and each class is given the opportunity to customize its stored properties further before the new instance is considered ready for use.

经验：
- 有继承关系类的父类构造函数中如果有 override 方法，需要注意子类成员变量的初始化问题
- 看日志输出顺序也不能太想当然，还是得搞清楚`下面`到底发生了什么。

哎，还是太菜了啊。。

# 参考资料
[What can we learn from Java HelloWorld?](https://www.programcreek.com/2013/04/what-can-you-learn-from-a-java-helloworld-program/)

[When and how a Java class is loaded and initialized?](https://www.programcreek.com/2013/01/when-and-how-a-java-class-is-loaded-and-initialized/)

[What is Instance Initializer in Java?](https://www.programcreek.com/2011/10/java-class-instance-initializers/)

[Java bytecode instruction listings](https://en.wikipedia.org/wiki/Java_bytecode_instruction_listings)

[Java 8 Language Specification - Chapter 12. Execution](https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html#jls-12.1)