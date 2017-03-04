---
layout: post
title: 谈谈Java的面向对象
category: 学习总结
tags: java
date: 2017-03-03
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

本文主要讲讲在Java的OOP编程里面常常被忽略的一些比较本质的问题，以及给出一些tips。

### 类的拷贝和构造

C++是默认具有拷贝语义的，对于没有拷贝运算符和拷贝构造函数的类，可以直接进行二进制拷贝，但是Java并不天生支持深拷贝，它的拷贝只是拷贝在堆上的地址，不同的变量引用的是堆上的同一个对象，那最初的对象是怎么被构建出来的呢？

#### Java对象的创建过程

关于对象的创建过程一般是从new指令(我说的是JVM的层面)开始的(具体请看图1)，JVM首先对符号引用进行解析，如果找不到对应的符号引用，那么这个类还没有被加载，因此JVM便会进行类加载过程（具体加载过程可参见我的另一篇博文）。符号引用解析完毕之后，JVM会为对象在堆中分配内存，HotSpot虚拟机实现的JAVA对象包括三个部分：对象头、实例字段和对齐填充字段（对齐不一定），其中要注意的是，实例字段包括自身定义的和从父类继承下来的（即使父类的实例字段被子类覆盖或者被private修饰，都照样为其分配内存）。相信很多人在刚接触面向对象语言时，总把继承看成简单的“复制”，这其实是完全错误的。JAVA中的继承仅仅是类之间的一种逻辑关系（具体如何保存记录这种逻辑关系，则设计到Class文件格式的知识），唯有创建对象时的实例字段，可以简单的看成“复制”。

![对象的创建过程](http://images2015.cnblogs.com/blog/592743/201603/592743-20160319235423381-1926278401.png )

为对象分配完堆内存之后，JVM会将该内存（除了对象头区域）进行零值初始化，这也就解释了为什么JAVA的属性字段无需显示初始化就可以被使用，而方法的局部变量却必须要显示初始化后才可以访问。最后，JVM会调用对象的构造函数，当然，调用顺序会一直上溯到Object类。

#### Java对象的初始化

 初始化的顺序是父类的实例变量构造、初始化->父类构造函数->子类的实例变量构造、初始化->子类的构造函数。对于静态变量、静态初始化块、变量、初始化块、构造器，它们的初始化顺序依次是（静态变量、静态初始化块）>（变量、初始化块）>构造器。

 JVM在为一个对象分配完内存之后，会给每一个实例变量赋予默认值，这个时候实例变量被第一次赋值，这个赋值过程是没有办法避免的。如果我们在实例变量初始化器中对某个实例x变量做了初始化操作，那么这个时候，这个实例变量就被第二次赋值了。 如果我们在实例初始化器中，又对变量x做了初始化操作，那么这个时候，这个实例变量就被第三次赋值了。如果我们在类的构造函数中，也对变量x做了初始化操作，那么这个时候，变量x就被第四次赋值。也就是说，一个实例变量，在Java的对象初始化过程中，最多可以被初始化4次。	

下面还是举一个例子吧

```java
class Parent {
        /* 静态变量 */
    public static String p_StaticField = "父类--静态变量";
         /* 变量 */
    public String    p_Field = "父类--变量";
    protected int    i    = 9;
    protected int    j    = 0;
        /* 静态初始化块 */
    static {
        System.out.println( p_StaticField );
        System.out.println( "父类--静态初始化块" );
    }
        /* 初始化块 */
    {
        System.out.println( p_Field );
        System.out.println( "父类--初始化块" );
    }
        /* 构造器 */
    public Parent()
    {
        System.out.println( "父类--构造器" );
        System.out.println( "i=" + i + ", j=" + j );
        j = 20;
    }
}

public class SubClass extends Parent {
         /* 静态变量 */
    public static String s_StaticField = "子类--静态变量";
         /* 变量 */
    public String s_Field = "子类--变量";
        /* 静态初始化块 */
    static {
        System.out.println( s_StaticField );
        System.out.println( "子类--静态初始化块" );
    }
       /* 初始化块 */
    {
        System.out.println( s_Field );
        System.out.println( "子类--初始化块" );
    }
       /* 构造器 */
    public SubClass()
    {
        System.out.println( "子类--构造器" );
        System.out.println( "i=" + i + ",j=" + j );
    }


        /* 程序入口 */
    public static void main( String[] args )
    {
        System.out.println( "子类main方法" );
        new SubClass();
    }
}
```

上面的初始化结果是：

- 父类--静态变量
- 父类--静态初始化块
- 子类--静态变量
- 子类--静态初始化块
- 子类main方法
- 父类--变量
- 父类--初始化块
- 父类--构造器
- i=9, j=0
- 子类--变量
- 子类--初始化块
- 子类--构造器
- i=9,j=20

子类的静态变量和静态初始化块的初始化是在父类的变量、初始化块和构造器初始化之前就完成了。静态变量、静态初始化块，变量、初始化块初始化了顺序取决于它们在类中出现的先后顺序。

分析：

1. 访问SubClass.main(),(这是一个static方法)，于是装载器就会为你寻找已经编译的SubClass类的代码（也就是SubClass.class文件）。在装载的过程中，装载器注意到它有一个基类（也就是extends所要表示的意思），于是它再装载基类。不管你创不创建基类对象，这个过程总会发生。如果基类还有基类，那么第二个基类也会被装载，依此类推。
2. 执行根基类的static初始化，然后是下一个派生类的static初始化，依此类推。这个顺序非常重要，因为派生类的“static初始化”有可能要依赖基类成员的正确初始化。
3. 当所有必要的类都已经装载结束，开始执行main()方法体，并用new SubClass（）创建对象。
4. 类SubClass存在父类，则调用父类的构造函数，你可以使用super来指定调用哪个构造函数。基类的构造过程以及构造顺序，同派生类的相同。首先基类中各个变量按照字面顺序进行初始化，然后执行基类的构造函数的其余部分。
5. 对子类成员数据按照它们声明的顺序初始化，执行子类构造函数的其余部分。


静态变量初始化器和静态初始化器基本同实例变量初始化器和实例初始化器相同，也有相同的限制(按照编码顺序被执行，不能引用后定义和初始化的类变量)。静态变量初始化器和静态初始化器中的代码会被编译器放到一个名为static的方法中(static是Java语言的关键字，因此不能被用作方法名，但是JVM却没有这个限制)，在类被第一次使用时，这个static方法就会被执行。

#### Java对象的引用方式

接下来我们再问一个问题，Java是怎么通过引用找到对象的呢？

至此，一个对象就被创建完毕，此时，一般会有一个引用指向这个对象。在JAVA中，存在两种数据类型，一种就是诸如int、double等基本类型，另一种就是引用类型，比如类、接口、内部类、枚举类、数组类型的引用等。引用的实现方式一般有两种，具体请看图3。此处说一句题外话，经常用人拿C++中的引用和JAVA的引用作对比，其实他们两个只是“名称”一样，本质并没什么关系，C++中的引用只是给现存变量起了一个别名(引用变量只是一个符号引用而已，编译器并不会给引用分配新的内存)，而JAVA中的引用变量却是真真正正的变量，具有自己的内存空间，只是不同的引用变量可以“指向”同一个对象而已。因此，如果要拿C++和JAVA引用对象的方式相对比，C++中的指针倒和JAVA中的引用如出一辙，毕竟，JAVA中的引用其实就是对指针的封装。

![对象的引用方式](http://images2015.cnblogs.com/blog/592743/201603/592743-20160319235423381-1926278401.png )

关于对象引用更深层次的问题，我们将在JVM篇章中详细解释。

### 匿名类、内部类和静态类

这一部分的内容相当宽泛，详细的可以查阅下面的参考文章，我在这里主要强调几个问题：

- 内部类的访问权限（它对外部类的访问权限和外部对它的访问权限）
- 成员内部类为什么不能有静态变量和静态函数（final修饰的除外）
- 内部类和静态内部类(嵌套内部类)的区别
- 局部内部类使用的形参为什么必须是final的
- 匿名内部类无法具有构造函数，怎么做初始化操作
- 内部类的继承问题(由于它必须和外部类实例相关联)

在这里只回答一下最后一个问题，由于成员内部类的实现其实是其构造函数的参数添加了外部类实体，所以内部类的实例化必须有外部类，但就类定义来说，内部类的定义只和外部类定义有关，代码如下

```java
public class Out {
    private static int a;
    private int b;

    public class Inner {
        public void print() {
            System.out.println(a);
            System.out.println(b);
        }
    }
}

// 内部类实例化
Out out = new Out();
Out.Inner inner = out.new Inner();

public class InheritInner extends Out.Inner {
  InheritInner(Out out){
    out.super();
  }
}
```

**_最后关于内部类的实现原理，请阅读参考文章中的《内部类的简单实现原理》，这非常重要_**

### **Java多态的实现原理**

Java的多态主要有以下几种形式：

- 继承
- 覆盖
- 接口


#### 方法调用的原理

多态是面向对象编程语言的重要特性，它允许基类的指针或引用指向派生类的对象，而在具体访问时实现方法的动态绑定。Java 对于方法调用动态绑定的实现主要依赖于方法表，但通过类引用调用(invokevitual)和接口引用调用(invokeinterface)的实现则有所不同。

类引用调用的大致过程为：Java编译器将Java源代码编译成class文件，在编译过程中，会根据静态类型将调用的符号引用写到class文件中。在执行时，JVM根据class文件找到调用方法的符号引用，然后在静态类型的方法表中找到偏移量，然后根据this指针确定对象的实际类型，使用实际类型的方法表，偏移量跟静态类型中方法表的偏移量一样，如果在实际类型的方法表中找到该方法，则直接调用，否则，认为没有重写父类该方法。按照继承关系从下往上搜索。 

方法表是实现动态调用的核心。方法表存放在方法区中的类型信息中。为了优化对象调用方法的速度，方法区的类型信息会增加一个指针，该指针指向一个记录该类方法的方法表，方法表中的每一个项都是对应方法的指针。这些方法中包括从父类继承的所有方法以及自身重写（override）的方法。

Java 的方法调用有两类:
- 动态方法调用：动态方法调用需要有方法调用所作用的对象，是动态绑定的。
- 静态方法调用：静态方法调用是指对于类的静态方法的调用方式，是静态绑定的；
- 类调用 (invokestatic) 是在编译时就已经确定好具体调用方法的情况。
- 实例调用 (invokevirtual)则是在调用的时候才确定具体的调用方法，这就是动态绑定，也是多态要解决的核心问题。

JVM 的方法调用指令有四个，分别是 invokestatic，invokespecial，invokesvirtual 和 invokeinterface。前两个是静态绑定，后两个是动态绑定的。

```java
class Person {   
 public String toString(){   
    return "I'm a person.";   
     }   
 public void eat(){}   
 public void speak(){}   
      
 }   
  
 class Boy extends Person{   
 public String toString(){   
    return "I'm a boy";   
     }   
 public void speak(){}   
 public void fight(){}   
 }   
  
 class Girl extends Person{   
 public String toString(){   
    return "I'm a girl";   
     }   
 public void speak(){}   
 public void sing(){}   
 }  
```

![方法表示意图](http://img.blog.csdn.net/20160812143114843)

如果子类改写了父类的方法，那么子类和父类的那些同名的方法共享一个方法表项。因此，方法表的偏移量总是固定的。所有继承父类的子类的方法表中，其父类所定义的方法的偏移量也总是一个定值。Person 或 Object中的任意一个方法，在它们的方法表和其子类 Girl 和 Boy 的方法表中的位置 (index) 是一样的。这样 JVM 在调用实例方法其实只需要指定调用方法表中的第几个方法即可。

![调用过程示意图](http://img.blog.csdn.net/20160812143114843 "在这里输入图片标题")

1. 在常量池（这里有个错误，上图为ClassReference常量池而非Party的常量池）中找到方法调用的符号引用 。
2. 查看Person的方法表，得到speak方法在该方法表的偏移量（假设为15），这样就得到该方法的直接引用。 
3. 根据this指针得到具体的对象（即 girl 所指向的位于堆中的对象）。
4. 根据对象得到该对象对应的方法表，根据偏移量15查看有无重写（override）该方法，如果重写，则可以直接调用（Girl的方法表的speak项指向自身的方法而非父类）；如果没有重写，则需要拿到按照继承关系从下往上的基类（这里是Person类）的方法表，同样按照这个偏移量15查看有无该方法。

#### 接口方法调用的原理

因为 Java 类是可以同时实现多个接口的，而当用接口引用调用某个方法的时候，情况就有所不同了。
Java 允许一个类实现多个接口，从某种意义上来说相当于多继承，这样同样的方法在基类和派生类的方法表的位置就可能不一样了。

```java
interface IDance{   
   void dance();   
 }   
  
 class Person {   
 public String toString(){   
   return "I'm a person.";   
     }   
 public void eat(){}   
 public void speak(){}   
      
 }   
  
 class Dancer extends Person   
 implements IDance {   
 public String toString(){   
   return "I'm a dancer.";   
     }   
 public void dance(){}   
 }   
  
 class Snake implements IDance{   
 public String toString(){   
   return "A snake.";   
     }   
 public void dance(){   
 //snake dance   
     }   
 }  
```

![接口调用示意](http://img.blog.csdn.net/20160812143114843 "在这里输入图片标题")

#### 方法调用的补充

我们先来看一个示例

```java
public class Test {

  public static class A {
    public void print() {
      System.out.println("A");
    }

    public void invoke() {
      print();
      sprint();
    }

    static void sprint() {
      System.out.println("sA");
    }
  }

  public static class B extends A {
    @Override
    public void print() {
      System.out.println("B");
    }

    static void sprint() {
      System.out.println("sB");
    }
  }

  public static void main(String[] args){
    A a = new B();
    a.invoke(); // B SA
  }
}
```

由于静态方法是静态调用的，在编译期就决定了跳转的符号，所以进入父类的invoke方法调用的sprint在编译期即是A的sprint，A的sprint符号和B的sprint在class中并不相同，这个符号在编译期已经确定了。

但是当在invoke中调用print，和C++有很大的区别，C++即使在父类的虚方法中调用了被override的方法，也依然执行的是父类的方法。这是为什么呢？因为C++的类方法调用实际上是传入了一个对象指针，由于invoke在父类定义，传入的指针类型也是A*，C++的虚方法表指针不是存在类区域的，而是存在对象内存区域中的，它直接在内存中父类对象区域中查找虚方法表，自然只能找到A的print。但是Java不一样，Java是通过传进来的this去找他的类型信息，再从类别信息里去找方法表，所以依然调用的是子类方法表中的print。

我们再看一个例子。

```java
public class Test {

  public static class A {

    public int a = 3;

    public void print() {
      System.out.println(a);
    }
  }

  public static class B extends A {

    public int a = 4;

  }

  public static void main(String[] args){
    B b = new B();
    b.print(); // 3
  }
}
```

多态只适用于父子类同样签名的方法，而属性是不参与多态的。在print里的符号a在编译期就确定是A的a了。同样的还有private的方法，**_私有方法不参与继承， 也不会出现在方法表中，因为私有方法是由invokespecial指令调用的。 _**

- **成员变量的访问只根据静态类型进行选择，不参与多态**
- **私有方法不会发生多态选择，只根据静态类型进选择。**

#### 继承的实现原理

和C++不同，C++的内存布局是非常紧凑的，这也是为了支持它天然的拷贝语义，c++父类对象的内存空间是直接被包含在子类对象的连续内存空间中的，其属性的偏移都取决于声明顺序和对齐。而Java虽然父类的实例变量依然是和子类的放在同一个连续的内存空间，但并非是通过简单的偏移来取成员的。不过在Java对象的内存布局中，依然是先安置父类的再安置子类的，所以讲sizeof(Parent)大小的内容转型成为父类指针，就可以实现super了。具体是在字节码中子类会有个u2类型的父类索引，属于CONSTANT_Class_info类型，通过CONSTANT_Class_info的描述可以找到CONSTANT_Utf8_info,然后可以找到指定的父类。

其实在Java内部，是通过隐式的组合来实现继承的。 子类对象中会保存一个实例对象的引用super，该引用指向其父类。 在实际的方法调用时，java会先在当前类的对象中寻找名称相同的方法，如果没有，就到super引用的父类对象中去寻找该方法，所以，若在子类中存在和 父类方法的签名和返回值类型完全相同的方法（重写）的话，java就会直接调用该对象的方法而不用去父类去寻找调用方法了。而且在子类对象中，可以直接通 过super来调用父类对象中的方法。


#### 重载、覆盖和隐藏

重载：方法名相同，但参数不同的多个同名函数

- 参数不同的意思是参数类型、参数个数、参数顺序至少有一个不同
- 返回值和异常以及访问修饰符，不能作为重载的条件(因为对于匿名调用，会出现歧义，eg:void a ()和int a() ，如果调用a()，出现歧义)
- main方法也是可以被重载的

覆盖：子类重写父类的方法，要求方法名和参数类型完全一样(参数不能是子类)，返回值和异常比父类小或者相同(即为父类的子类)，访问修饰符比父类大或者相同

- 子类实例方法不能覆盖父类的静态方法；子类的静态方法也不能覆盖父类的实例方法(编译时报错)，总结为方法不能交叉覆盖

隐藏：父类和子类拥有相同名字的属性或者方法时，父类的同名的属性或者方法形式上不见了，实际是还是存在的。

- 当发生隐藏的时候，声明类型是什么类，就调用对应类的属性或者方法，而不会发生动态绑定
- 方法隐藏只有一种形式，就是父类和子类存在相同的静态方法
- 属性只能被隐藏，不能被覆盖
- 子类实例变量/静态变量可以隐藏父类的实例/静态变量，总结为变量可以交叉隐藏

隐藏和覆盖的区别：

- 被隐藏的属性，在子类被强制转换成父类后，访问的是父类中的属性
- 被覆盖的方法，在子类被强制转换成父类后，调用的还是子类自身的方法
- 因为覆盖是动态绑定，是受RTTI(run time type identification，运行时类型检查)约束的，隐藏不受RTTI约束，总结为RTTI只针对覆盖，不针对隐藏

### java的对象模型

Java中存在两种类型，原始类型和对象（引用）类型。原始类型，即数据类型，内存布局符合其类型规范，并无其他负载。而对象类型，则由于自定义类型、垃圾回收，对象锁等各种语义与JVM性能原因，需要使用额外空间。

Java对象的内存布局：对象头（Header），实例数据（Instance Data），对齐填充（Padding）。
详细的内容可以查阅参考文章

这里我们主要讲讲在继承和组合两种情形下会对内存布局造成什么变化。

- 类属性按照如下优先级进行排列：长整型和双精度类型；整型和浮点型；字符和短整型；字节类型和布尔类型，最后是引用类型。这些属性都按照各自的单位对齐。
- 不同类继承关系中的成员不能混合排列。首先按照规则2处理父类中的成员，接着才是子类的成员
- 当父类中最后一个成员和子类第一个成员的间隔如果不够4个字节的话，就必须扩展到4个字节的基本单位。
- 如果子类第一个成员是一个双精度或者长整型，并且父类并没有用完8个字节，JVM会破坏规则1，按照整形（int），短整型（short），字节型（byte），引用类型（reference）的顺序，向未填满的空间填充。
- 数组有一个额外的头部成员，用来存放“长度”变量。数组元素以及数组本身，跟其他常规对象同样，都需要遵守8个字节的边界规则。

下面给一个例子

```java
public class Test {

  public static class A {
    public A() {
      System.out.println(this.hashCode());
    }
  }

  public static class B extends A {
    public B(){
      System.out.println(this.hashCode());
      System.out.println(super.equals(this));
    }
  }

  public static void main(String[] args){
    B b = new B();
  }
}
/*
 * 输出如下：
 * 1627674070
 * 1627674070
 * true
 */

```

### 参考文章

- [Java对象的创建](http://www.cnblogs.com/chenyangyao/p/5296807.html)
- [Java类初始化顺序(看这篇文章就够了)](https://segmentfault.com/a/1190000004527951)
- [详解内部类](https://www.cnblogs.com/chenssy/p/3388487.html)
- [详解匿名内部类](http://www.cnblogs.com/chenssy/p/3390871.html)
- [内部类的简单实现原理(时间不够，内部类只看这篇即可)](http://www.jianshu.com/p/e385ce41ca5b)
- [Java技术——多态的实现原理(时间紧多态只看这篇)](http://blog.csdn.net/seu_calvin/article/details/52191321)
- [ java方法调用之重载、重写的调用原理(一)](http://blog.csdn.net/fan2012huan/article/details/50999777)
- [java方法调用之单分派与多分派(二)](http://blog.csdn.net/fan2012huan/article/details/51004615)
- [ java方法调用之动态调用多态（重写override）的实现原理——方法表](http://blog.csdn.net/fan2012huan/article/details/51007517)
- [ java方法调用之多态的补充示例](http://blog.csdn.net/fan2012huan/article/details/51025616)
- [java的重载、覆盖和隐藏的区别](http://www.cnblogs.com/xiaoQLu/archive/2013/01/07/2849869.html)
- [Java 对象内存布局](http://www.nathanyan.com/2015/08/19/Java-%E5%AF%B9%E8%B1%A1%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/)
