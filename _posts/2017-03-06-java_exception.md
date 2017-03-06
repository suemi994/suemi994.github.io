---
layout: post
title: 聊聊Java的异常机制及实现
category: 学习总结
tags: java
date: 2017-03-06
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

在一些传统的编程语言，如C语言中，并没有专门处理异常的机制，程序员通常用方法的特定返回值来表示异常情况，并且程序的正常流程和异常流程都采用同样的流程控制语句。
Java语言按照面向对象的思想来处理异常，使得程序具有更好的可维护性。Java异常处理机制具有以下优点：

1. 把各种不同类型的异常情况进行分类，用Java类来表示异常情况，这种类被称为异常类。把异常情况表示成异常类，可以充分发挥类的可扩展和可重用的优势。
2. 异常流程的代码和正常流程的代码分离，提高了程序的可读性，简化程序的结构。
3. 可以灵活的处理异常，如果当前方法有能力处理异常，就捕获并处理它，否则只需要抛出异常，由方法调用。

### Java异常基础

关于异常的使用我就不再多说了，在这里还是先提几个问题：

- catch多个异常的时候，按什么规则选择呢
- throws异常是否是函数签名的一部分呢
- 覆盖父类的带throws的函数是否也需要加throws呢
- 同时实现多个接口中同名抛出异常的函数最后抛出异常的集合是什么呢

接下来我们回答其中的部分问题，先看一个例子

![异常捕获示例](https://static.oschina.net/uploads/img/201703/06140743_L79R.png)

可以看到Java是按照catch声明的顺序来捕获异常的，且编译器不允许将父类异常声明在子类之前。

throws异常显然不是函数的一部分，因为两个throws不同的同名同参数的函数不允许重载。

![覆盖时的异常声明示例](https://static.oschina.net/uploads/img/201703/06141430_xvWP.png)

从上图我们可以看出覆盖对抛出异常的声明并没有要求。

![throws的接口继承](https://static.oschina.net/uploads/img/201703/06142637_kIbP.png )

**上图可以看出编译器对接口的方法实现也并无什么要求，重点在于try-catch块的检查，你不能catch一个你在throw块里不可能抛出的检查类型异常，而这种判断是通过你调用方法声明的抛出异常，即使你在方法实现里不可能抛出该异常，你加在throws里，一样可以蒙骗编译器。对于方法声明的抛出异常，只有一个条件需要满足，那就是你的实现中可能抛出的检查类型异常要么处理要么声明抛出，不需要考虑继承和实现关系给throws带来的影响，这是参考文章中的一点小错误，特此更正。**

### Java异常类的架构

![Java异常架构图](http://images.cnitblog.com/blog/497634/201402/111228085926220.jpg )

1. Throwable

- Throwable是 Java 语言中所有错误或异常的超类。
- Throwable包含两个子类: Error 和 Exception。它们通常用于指示发生了异常情况。
- Throwable包含了其线程创建时线程执行堆栈的快照，它提供了printStackTrace()等接口用于获取堆栈跟踪数据等信息。

2. Exception

- Exception及其子类是 Throwable 的一种形式，它指出了合理的应用程序想要捕获的条件。

3. RuntimeException 

- RuntimeException是那些可能在 Java 虚拟机正常运行期间抛出的异常的超类。
- 编译器不会检查RuntimeException异常。例如，除数为零时，抛出ArithmeticException异常。RuntimeException是ArithmeticException的超类。当代码发生除数为零的情况时，倘若既"没有通过throws声明抛出ArithmeticException异常"，也"没有通过try...catch...处理该异常"，也能通过编译。这就是我们所说的"编译器不会检查RuntimeException异常"！
- 如果代码会产生RuntimeException异常，则需要通过修改代码进行避免。例如，若会发生除数为零的情况，则需要通过代码避免该情况的发生！

4. Error

- 和Exception一样，Error也是Throwable的子类。它用于指示合理的应用程序不应该试图捕获的严重问题，大多数这样的错误都是异常条件。
- 和RuntimeException一样，编译器也不会检查Error。

 

Java将可抛出(Throwable)的结构分为三种类型：被检查的异常(Checked Exception)，运行时异常(RuntimeException)和错误(Error)。

(01) 运行时异常

- 定义: RuntimeException及其子类都被称为运行时异常。
- 特点: Java编译器不会检查它。也就是说，当程序中可能出现这类异常时，倘若既"没有通过throws声明抛出它"，也"没有用try-catch语句捕获它"，还是会编译通过。例如，除数为零时产生的ArithmeticException异常，数组越界时产生的IndexOutOfBoundsException异常，fail-fail机制产生的ConcurrentModificationException异常等，都属于运行时异常。
- 虽然Java编译器不会检查运行时异常，但是我们也可以通过throws进行声明抛出，也可以通过try-catch对它进行捕获处理。
- 如果产生运行时异常，则需要通过修改代码来进行避免。例如，若会发生除数为零的情况，则需要通过代码避免该情况的发生！

(02) 被检查的异常

- 定义: Exception类本身，以及Exception的子类中除了"运行时异常"之外的其它子类都属于被检查异常。
- 特点: Java编译器会检查它。此类异常，要么通过throws进行声明抛出，要么通过try-catch进行捕获处理，否则不能通过编译。例如，CloneNotSupportedException就属于被检查异常。当通过clone()接口去克隆一个对象，而该对象对应的类没有实现Cloneable接口，就会抛出CloneNotSupportedException异常。
- 被检查异常通常都是可以恢复的。

(03) 错误

- 定义: Error类及其子类。
- 特点: 和运行时异常一样，编译器也不会对错误进行检查。
- 当资源不足、约束失败、或是其它程序无法继续运行的条件发生时，就产生错误。程序本身无法修复这些错误的。例如，VirtualMachineError就属于错误。
- 按照Java惯例，我们是不应该是实现任何新的Error子类的！

![常用异常举例](http://images.cnitblog.com/blog/497634/201402/111228085926220.jpg)

### Java异常的实现原理

#### 异常的捕获原理

首先介绍下java的异常表（Exception table），异常表是JVM处理异常的关键点，在java类中的每个方法中，会为所有的try-catch语句，生成一张异常表，存放在字节码的最后，该表记录了该方法内每个异常发生的起止指令和处理指令。

接下来看一个例子：

```java
public void catchException() {  
    long l = System.nanoTime();  
    for (int i = 0; i < testTimes; i++) { 
        try {  
            throw new Exception();  
        } catch (Exception e) { 
            //nothing to do
        }  
    }
    System.out.println("抛出并捕获异常：" + (System.nanoTime() - l));  
}
```

字节码如下

![字节码示例](http://img3.tbcdn.cn/L1/461/1/655b57165d2c17361f64e1e9b87364207dc95c9f)

面请结合java代码和生成的字节码来看下面的指令分析：
0-4号： 执行try前面的语句
5号： 执行try语句前保存现场
6号： 执行try语句后跳转指令行，图中表示跳转到22
9-17号： try-catch代码生成指令，结合红色框图异常表，表示9-17号指令若有Exception异常抛出就执行17行指令. 
16号： athrow 表示抛出异常
17号： astore 表示jvm将该异常实例存储到局部变量表中方便一旦出方法栈调用方可以找到
22号： 恢复try语句执行前保存的现场
对比指令分析，再结合使用try-catch代码分析：

- 若try没有抛出异常，则继续执行完try语句，跳过catch语句，此时就是从指令6跳转到指令22.
- 若try语句抛出异常则执行指令17，将异常保存起来，若异常被方法抛出，调用方拿到异常可用于异常层次索引。


通过以上的分析，可以知道JVM是怎么捕获并处理异常，其实就是使用goto指令来做上下文切换。


#### 异常的处理机制

上面大致介绍了异常是如何产生并捕获的，接下来我们详细讲讲athrow指令抛出异常后的故事，也就是如何处理异常的问题。

athrow指令，这个指令运作过程大致是首先检查操作栈顶，这时栈顶必须存在一个reference类型的值，并且是java.lang.Throwable的子类（虚拟机规范中要求如果遇到null则当作NPE异常使用），然后暂时先把这个引用出栈，接着搜索本方法的异常表，找一下本方法中是否有能处理这个异常的handler，如果能找到合适的handler就会重新初始化PC寄存器指针指向此异常handler的第一个指令的偏移地址。接着把当前栈帧的操作栈清空，再把刚刚出栈的引用重新入栈。如果在当前方法中很悲剧的找不到handler，那只好把当前方法的栈帧出栈（这个栈是VM栈，不要和前面的操作栈搞混了，栈帧出栈就意味着当前方法退出），这个方法的调用者的栈帧就自然在这条线程VM栈的栈顶了，然后再对这个新的当前方法再做一次刚才做过的异常handler搜索，如果还是找不到，继续把这个栈帧踢掉，这样一直到找，要么找到一个能使用的handler，转到这个handler的第一条指令开始继续执行，要么把VM栈的栈帧抛光了都没有找到期望的handler，这样的话这条线程就只好被迫终止、退出了。 

对于Java语言中的关键字catch和finally，虚拟机中并没有特殊的字节码指令去支持它们，都是通过编译器生成字节码片段以及不同的异常处理器来实现。 

我们总结一下athrow指令中虚拟机可能做的事情：

- 检查栈顶异常对象类型（只检查是不是null，是否referance类型，是否Throwable的子类一般在类验证阶段的数据流分析中做，或者索性不做靠编译器保证了，编译时写到Code属性的StackMapTable中，在加载时仅做类型验证）
- 把异常对象的引用出栈
- 搜索异常表，找到匹配的异常handler
- 重置PC寄存器状态
- 清理操作栈
- 把异常对象的引用入栈
- 把异常方法的栈帧逐个出栈（这里的栈是VM栈）
- 残忍地终止掉当前线程。


#### 异常到底慢不慢

这里直接给出一些结论吧：

新建一个异常对象比新建一个普通对象在耗时上多一个数量级，抛出并捕获异常的耗时比新建一个异常在耗时上也要多一个数量级。创建一个异常对象却是要比一个普通对象耗时多，捕获一个异常耗时更甚。捕获的过程我们上面已经简要介绍了，为什么新建一个异常对象这么耗时？且看源码：

在java中，所有的异常都继承自Throwable类，Throwable的构造函数

```java
public Throwable() {
    ...
    fillInStackTrace();
    ...
}
```

有个nativ方法public synchronized native Throwable fillInStackTrace();这个方法会存入当前线程的堆栈信息。也就是说每次创建一个异常实例都会把堆栈信息存一遍。这就是时间开销的主要来源了。

这个时候我们可以下一个结论：**新建异常对象比创建一个普通对象是要更加的耗时。**

能避开创建异常的这个耗时吗？答案是可以的，如果在程序中我们不关心异常抛出的异常占信息，我们可以自己定义一个异常继承自已有的异常类型，并写一个方法覆盖掉fillInStackTrace方法就行了。

### 参考文章

- [**_Java异常简介及其架构_**](http://www.cnblogs.com/skywang12345/p/3544168.html)
- [关于异常处理的几条建议](http://www.cnblogs.com/skywang12345/p/3544287.html)
- [**_关于异常的几个谜题_**(必看)](http://www.cnblogs.com/skywang12345/p/3544353.html#a7)
- [**_异常分析初探_**](https://yq.aliyun.com/articles/4202)
- [**透过JVM看Exception本质**](http://icyfenix.iteye.com/blog/857722)
- [三言两语：JVM 字节码执行实例分析](http://blog.xiaohansong.com/2016/04/26/java-bytecode/#try-catch-finally__u5B57_u8282_u7801_u5206_u6790)
