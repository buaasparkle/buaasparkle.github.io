---
layout: post
title:  "Java 范型：协变，逆变"
date:   2020-11-14 15:35:18 +0800
categories: coding
tags: Java, Generic, Covariant, Contravariant
---

# 为什么有范型？

Java 中，`Object` 是类型结构的金字塔尖，在 Java 1.5 之前，定义一个 List 是这样：

```java
List list = new ArrayList();
list.add(10);
list.add("hello");

Integer o1 = (Integer) list.get(0);
String o2 = (String) list.get(1);
```

有什么问题？

1. 缺少类型检查：如果只想定义一个 Integer List，是无法阻止误入一个 String 的
1. get 时需要强制类型转换：如果 cast 错了，会抛 `ClassCastException`

Java 1.5 提出`范型`来解决这个问题

```java
List<String> list = new ArrayList<String>();
list.add("hello");
list.add(10); // 编译报错

String o1 = list.get(0);
```

因为处于`向后兼容`的考虑，所以需要兼容 1.5 之前的实现方式，所以这些类型检查/转换都是在编译时做的，而最终的实现还是走到 `Object` 这个塔尖来实现，这个也是所谓的`类型擦除`。

# 不变

定义如下的类结构：

```
         Vehicle
            |
      ----------------
      |              |
     Car            Bus
      |                  
   --------
   |      |
  Bens   BMW
```

```java
List<Car> carList = new ArrayList<>();
carList.add(new Bens());
carList.add(new BMW());

Car c1 = carList.get(0);
Car c2 = carList.get(1);
```

定义一个 List<Car>，根据类本身的继承关系，可以写子类（Bens/BMW）,可以读出父类(Car)，这个就可以认为是`不变`.

但是，下面的赋值是不行的：

```java
List<Car> cars = new ArrayList<>();
List<Vehicle> garage = cars;        // 编译报错
```

虽然 Car 是 Vehicle 的子类，但是 `List<Car>` 和 `List<Vehicle>` 并没有继承关系。

# 协变 & 逆变

定义：

> 协变：如果 A 是 B 的子类型，那么 `Generic<A>` 也是 `Generic<B>` 的子类型

> 逆变：如果 A 是 B 的子类型，但是 `Generic<B>` 却是 `Generic<A>` 的子类型

太费解，没事儿，先以上面定义的继承关系为例，看一下类型的`上界`和`下界`：

```
   null ... Bens ... Car ... Vehicle ... Object
                      |
     extends   <-    锚点    ->   super
```

如果选中 Car 类型作为锚点定义一个 List：

- 定义一个数组内元素类型全为 **Car** 的列表：`List<Car>`

- 定义一个数组内元素类型为 **Car 子类** 的列表：`List<? extends Car>` (这里 Car 就是上界)

- 定义一个数组内元素类型全为 **Car 父类** 的列表：`List<? super Car>` （这里 Car 就是下界）

## 编译器视角

开头说了，范型检查都是编译时做的，还存在`类型擦除`，那定义一个通配符(?)加上 extends/super 描述的范型会怎么样呢？

### List<? extends Car> （上界，只读）

在编译器看来，我只知道这个 List 中的元素都继承自 Car，但是具体是什么类型，也不知道！

那定义这样一个数组有什么作用呢？

`赋值！！！`

试想，一个 `元素类型是 Bens 的数组` 和 `元素类型都是继承自 Car 的数组`，谁表示的范围大呢？

再换句话说：一个 `元素类型是 Bens 的数组` **就是一个(is a)** `元素类型都是继承自 Car 的数组`，没错吧！

即：Bens 是 Car 的子类型，同时 `List<Bens>` 也是 List<? extends Car> 的子类型，这就叫 TMD `协变`！

所以也就有：

```java
List<? extends Car> list = new ArrayList<Bens>() // 赋值成立
```

> 赋值成立，就可以作为参数定义，愉快地传参啦～～～

那 list 定义成 List<? extends Car> 之后有什么限制呢？

由于编译器不知道 List 中具体类型的什么，所以也就不敢往里面随便 `add`，因为不知道类型嘛（不能做类型检查范型就失职了），所以一不做二不休，干脆`不能改`!

虽然不能改了，但是由于我知道存的数据类型上界是 Car，所以可以读！但是必须都当成最高的 Car 类型来读。

所以可以理解为：定义为 `? extends` 范型的 List 就相当于一个 -> Producer，只能往外 get (car 提供给别人)，不能往里 set（消耗点什么进来）。

### List<? super Car> （下界，可写）

List<? super Car> 从语义上看就是定义了一个 `元素类型都是 Car 的父类的数组`，从编译器的视角看，它也只是知道数组中的元素类型都是 Car 的父类，但是具体是哪个咱也不知道，咱也不敢问。

还是举一个具体的例子看：

一个 `元素类型是 Vehicle 的数组` 和 `元素类型都是 Car 父类的数组`，谁表示的范围大呢？

显然满足：

`元素类型是 Vehicle 的数组` **is a** `元素类型都是 Car 父类的数组` 

但是由于 Car 是 Vehicle 的子类，而 `List<Vehicle>` 反倒成了 List<? super Car> 的子类，继承关系正好反过来了，因而称这种情况为 `逆变`。

同样，定义为 List<? super Car> 的列表也是有一些限制的：

编译器：List 中存的都是 Car 或者其父类型，那我把一个 Car 的子类（Bens/BMW）对象赋值给 Car 的父类这样肯定是没问题的嘛，所以支持添加(add) `Car 及其子类`，也就是下界（Car）再往下的类型是可以的，往上就不行了。

如果要读取 List 中的元素，因为只知道下界，不知道上界，所以只能取`最上界 Object`作为类型输出了。

从消费者的角度看，`? super T` 就是一个可以吃 T 及其以下类型的消费者，对外输出的话，只能是最上线 Object 了。

> PECS -> **P**roducer **E**xtends, **C**onsumer **S**uper

# 总结

困扰我多年的逆变/协变（? extends/? super）总算是搞明白了。

其实就是一个语义的概念，我是要定义一个 Car 及其以下类型 (? extends Car) 组成的数组，还是要取 Car 及其以上类型 (? super Car) 组成的数组，有了这个范围，就可以将在对应范围内定义的具体类型的数组 `赋值` 过来了，因为明确了一个范围的 List 语义范围上一定是大于某个具体类型的 List 的，所以一定有具体 List<?> 是 List<? extends T> 或 List<? super T> 的子类型，对于在 T 下面的类型，因为 **is a** 关系一致，就是`协变`，而对于 T 上面的类型，**is a** 关系正好调过来了，也就对应了`逆变`。

范型归根结底就是方便做`类型检查和转换`的，逆变协变感觉更像是一个语义场景下反推出来的原结构继承关系，和范型结构(List<>) 继承关系变化的对应关系，有了范围大小，就可以把小的类型作为参数`赋值`给大的（函数参数定义），同时基于语义的实际情况（上/下界），从编译器的视角看能提供的读/写操作和界限，我的理解大致就是这个样子了。

最后说明以下，以上针对的都是 List 范型，对于数组 Car[], Vehicle[]，由于其本身就是系统已知的类型，天然满足协变关系.


# 参考资料

[Java: Producer Extends, Consumer Super](https://medium.com/@isuru89/java-producer-extends-consumer-super-9fbb0e7dd268)