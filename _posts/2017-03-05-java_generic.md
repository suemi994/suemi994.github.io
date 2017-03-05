---
layout: post
title: 聊聊Java的泛型及实现
category: 学习总结
tags: java
date: 2017-03-05
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

和C++以模板来实现静多态不同，Java基于运行时支持选择了泛型，两者的实现原理大相庭径。C++可以支持基本类型作为模板参数，Java却只能接受类作为泛型参数；Java可以在泛型类的方法中取得自己泛型参数的Class类型，C++只能由编译器推断在不为人知的地方生成新的类，对于特定的模板参数你只能使用特化。在本文中我主要想聊聊泛型的实现原理和一些高级特性。

### 泛型基础

泛型是对Java语言类型系统的一种扩展，有点类似于C++的模板，可以把类型参数看作是使用参数化类型时指定的类型的一个占位符。引入泛型，是对Java语言一个较大的功能增强，带来了很多的好处：

1. 类型安全。类型错误现在在编译期间就被捕获到了，而不是在运行时当作java.lang.ClassCastException展示出来，将类型检查从运行时挪到编译时有助于开发者更容易找到错误，并提高程序的可靠性

2. 消除了代码中许多的强制类型转换，增强了代码的可读性

3. 为较大的优化带来了可能

泛型是什么并不会对一个对象实例是什么类型的造成影响，所以，通过改变泛型的方式试图定义不同的重载方法是不可以的。剩下的内容我不会对泛型的使用做过多的讲述，泛型的通配符等知识请自行查阅。

在进入下面的论述之前我想先问几个问题：

- 定义一个泛型类最后到底会生成几个类，比如ArrayList<T>到底有几个类
- 定义一个泛型方法最终会有几个方法在class文件中
- 为什么泛型参数不能是基本类型呢
- ArrayList<Integer>是一个类吗
- ArrayList<Integer>和List<Integer>和ArrayList<Number>和List<Number>是什么关系呢，这几个类型的引用能相互赋值吗

### 类型擦除

正确理解泛型概念的首要前提是理解类型擦除（type erasure）。 Java中的泛型基本上都是在编译器这个层次来实现的。在生成的Java字节代码中是不包含泛型中的类型信息的。使用泛型的时候加上的类型参数，会被编译器在编译的时候去掉。这个过程就称为类型擦除。如在代码中定义的List<Object>和List<String>等类型，在编译之后都会变成List。JVM看到的只是List，而由泛型附加的类型信息对JVM来说是不可见的。Java编译器会在编译时尽可能的发现可能出错的地方，但是仍然无法避免在运行时刻出现类型转换异常的情况。类型擦除也是Java的泛型实现方式与C++模板机制实现方式之间的重要区别。

很多泛型的奇怪特性都与这个类型擦除的存在有关，包括：

- 泛型类并没有自己独有的Class类对象。比如并不存在List<String>.class或是List<Integer>.class，而只有List.class。
- 静态变量是被泛型类的所有实例所共享的。对于声明为MyClass<T>的类，访问其中的静态变量的方法仍然是 MyClass.myStaticVar。不管是通过new MyClass<String>还是new MyClass<Integer>创建的对象，都是共享一个静态变量。
- 泛型的类型参数不能用在Java异常处理的catch语句中。因为异常处理是由JVM在运行时刻来进行的。由于类型信息被擦除，JVM是无法区分两个异常类型MyException<String>和MyException<Integer>的。对于JVM来说，它们都是 MyException类型的。也就无法执行与异常对应的catch语句。


类型擦除的基本过程也比较简单，首先是找到用来替换类型参数的具体类。这个具体类一般是Object。如果指定了类型参数的上界的话，则使用这个上界。把代码中的类型参数都替换成具体的类。同时去掉出现的类型声明，即去掉<>的内容。比如T get()方法声明就变成了Object get()；List<String>就变成了List。

### 泛型的实现原理

因为种种原因，Java不能实现真正的泛型，只能使用类型擦除来实现伪泛型，这样虽然不会有类型膨胀（C++模板令人困扰的难题）的问题，但是也引起了许多新的问题。所以，Sun对这些问题作出了许多限制，避免我们犯各种错误。

#### 保证类型安全

首先第一个是泛型所宣称的类型安全，既然类型擦除了，如何保证我们只能使用泛型变量限定的类型呢？java编译器是通过先检查代码中泛型的类型，然后再进行类型擦除，在进行编译的。那类型检查是针对谁的呢，让我们先看一个例子。

```java
ArrayList<String> arrayList1=new ArrayList(); // 正确，只能放入String
ArrayList arrayList2=new ArrayList<String>(); // 可以放入任意Object
```

这样是没有错误的，不过会有个编译时警告。不过在第一种情况，可以实现与 完全使用泛型参数一样的效果，第二种则完全没效果。因为，本来类型检查就是编译时完成的。new ArrayList()只是在内存中开辟一个存储空间，可以存储任何的类型对象。而真正涉及类型检查的是它的引用，因为我们是使用它引用arrayList1 来调用它的方法，比如说调用add()方法。所以arrayList1引用能完成泛型类型的检查。
而引用arrayList2没有使用泛型，所以不行。

**类型检查就是针对引用的，谁是一个引用，用这个引用调用泛型方法，就会对这个引用调用的方法进行类型检测，而无关它真正引用的对象。**

#### 实现自动类型转换

因为类型擦除的问题，所以所有的泛型类型变量最后都会被替换为原始类型。这样就引起了一个问题，既然都被替换为原始类型，那么为什么我们在获取的时候，不需要进行强制类型转换呢？

```java
public class Test {  
    public static void main(String[] args) {  
        ArrayList<Date> list=new ArrayList<Date>();  
        list.add(new Date());  
        Date myDate=list.get(0);
    }      
}  
```

编译器生成的class文件中会在你调用泛型方法完成之后返回调用点之前加上类型转换的操作，比如上文的get函数，就是在get方法完成后，jump回原本的赋值操作的指令位置之前加入了强制转换，转换的类型由编译器推导。

### 泛型中的继承关系

先看一个例子：

```java


class DateInter extends A<Date> {  
    @Override  
    public void setValue(Date value) {  
        super.setValue(value);  
    }  
    @Override  
    public Date getValue() {  
        return super.getValue();  
    }  
}
```

先来分析setValue方法，父类的类型是Object，而子类的类型是Date，参数类型不一样，这如果实在普通的继承关系中，根本就不会是重写，而是重载。

```java
public void setValue(java.util.Date);  //我们重写的setValue方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: invokespecial #16                // invoke A setValue
:(Ljava/lang/Object;)V  
       5: return  
  
  public java.util.Date getValue();    //我们重写的getValue方法  
    Code:  
       0: aload_0  
       1: invokespecial #23                 // A.getValue  
:()Ljava/lang/Object;  
       4: checkcast     #26               
       7: areturn  
  
  public java.lang.Object getValue();     //编译时由编译器生成的方法  
    Code:  
       0: aload_0  
       1: invokevirtual #28                 // Method getValue:() 去调用我们重写的getValue方法  
;  
       4: areturn  
  
  public void setValue(java.lang.Object);   //编译时由编译器生成的方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: checkcast     #26                 
       5: invokevirtual #30                 // Method setValue;   去调用我们重写的setValue方法  
)V  
       8: return  
```

并且，还有一点也许会有疑问，子类中的方法  Object   getValue()和Date getValue()是同 时存在的，可是如果是常规的两个方法，他们的方法签名是一样的，也就是说虚拟机根本不能分别这两个方法。如果是我们自己编写Java代码，这样的代码是无法通过编译器的检查的，但是虚拟机却是允许这样做的，因为虚拟机通过参数类型和返回类型来确定一个方法，所以编译器为了实现泛型的多态允许自己做这个看起来“不合法”的事情，然后交给虚拟器去区别。

我们再看一个经常出现的例子。

```java
class A {
    Object get(){
        return new Object();
    }
}

class B extends A {
    @Override
    Integer get() {
        return new Integer(1);
    }
}

  public static void main(String[] args){
    A a = new B();
    B b = (B) a;
    A c = new A();
    a.get();
    b.get();
    c.get();
  }
```

反编译之后的结果

```java
17: invokespecial #5                  // Method com/suemi/network/test/A."<init>":()V
      20: astore_3
      21: aload_1
      22: invokevirtual #6                  // Method com/suemi/network/test/A.get:()Ljava/lang/Object;
      25: pop
      26: aload_2
      27: invokevirtual #7                  // Method com/suemi/network/test/B.get:()Ljava/lang/Integer;
      30: pop
      31: aload_3
      32: invokevirtual #6                  // Method com/suemi/network/test/A.get:()Ljava/lang/Object;
```

实际上当我们使用父类引用调用子类的get时，先调用的是JVM生成的那个覆盖方法，在桥接方法再调用自己写的方法实现。



#### 泛型参数的继承关系

在Java中，大家比较熟悉的是通过继承机制而产生的类型体系结构。比如String继承自Object。根据Liskov替换原则，子类是可以替换父类的。当需要Object类的引用的时候，如果传入一个String对象是没有任何问题的。但是反过来的话，即用父类的引用替换子类引用的时候，就需要进行强制类型转换。编译器并不能保证运行时刻这种转换一定是合法的。这种自动的子类替换父类的类型转换机制，对于数组也是适用的。 String[]可以替换Object[]。但是泛型的引入，对于这个类型系统产生了一定的影响。正如前面提到的List<String>是不能替换掉List<Object>的。


引入泛型之后的类型系统增加了两个维度：一个是类型参数自身的继承体系结构，另外一个是泛型类或接口自身的继承体系结构。第一个指的是对于 List<String>和List<Object>这样的情况，类型参数String是继承自Object的。而第二种指的是 List接口继承自Collection接口。对于这个类型系统，有如下的一些规则：

相同类型参数的泛型类的关系取决于泛型类自身的继承体系结构。即List<String>可以赋给Collection<String> 类型的引用，List<String>可以替换Collection<String>。这种情况也适用于带有上下界的类型声明。
当泛型类的类型声明中使用了通配符的时候， 这种替换的判断可以在两个维度上分别展开。如对Collection<? extends Number>来说，用来替换他的引用可以在Collection这个维度上展开，即List<? extends Number>和Set<? extends Number>等；也可以在Number这个层次上展开，即Collection<Double>和 Collection<Integer>等。如此循环下去，ArrayList<Long>和 HashSet<Double>等也都可以替换Collection<? extends Number>。

如果泛型类中包含多个类型参数，则对于每个类型参数分别应用上面的规则。理解了上面的规则之后，就可以很容易的修正实例分析中给出的代码了。只需要把List<Object>改成List<?>即可。List<String>可以替换List<?>的子类型，因此传递参数时不会发生错误。

**_个人认为这里对上面这种情形使用子类型这种说法来形容这种关系是不当的，因为List<String>等本质上来说不能算作类型，只是对List类型加上了编译器检查约束，也就不存在子类型这种说法。只能用是否在赋值时能够进行类型转换来说明。_**

### 泛型使用中的注意点

#### 运行时型别查询

```java
// 错误，为类型擦除之后，ArrayList<String>只剩下原始类型，泛型信息String不存在了，无法进行判断
if( arrayList instanceof ArrayList<String>) 

if( arrayList instanceof ArrayList<?>)    // 正确
```

#### 异常中使用泛型的问题

- 不能抛出也不能捕获泛型类的对象。事实上，泛型类扩展Throwable都不合法。为什么不能扩展Throwable，因为异常都是在运行时捕获和抛出的，而在编译的时候，泛型信息全都会被擦除掉。类型信息被擦除后，那么多个使用不同泛型参数地方的catch都变为原始类型Object，那么也就是说，多个地方的catch变的一模一样，这自然不被允许。
- 不能再catch子句中使用泛型变量。

```java

public static <T extends Throwable> void doWork(Class<T> t){  
        try{  
            ...  
        }catch(T e){ //编译错误  T->Throwable，下面的永远不会被捕获，所以不被允许
            ...  
        }catch(IndexOutOfBounds e){  
        }                           
 }  
```

#### 不允许创建泛型类数组

```java
Pair<String,Integer>[] table = new Pair<String,Integer>[10];// 编译错误
Pair[] table = new Pair[10];// 无编译错误
```

由于数组必须携带自己元素的类型信息，在类型擦除之后，Pair<String,Integer>数组就变成了Pair<Object,Object>数组，数组只能携带它的元素是Pair这样的信息，但是并不能携带其泛型参数类型的信息，所以也就无法保证table[i]赋值的类型安全。编译器只能禁用这种操作。

#### 泛型类中的静态方法和静态变量

泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数。

```java
public class Test2<T> {    
    public static T one;   //编译错误    
    public static  T show(T one){ //编译错误    
        return null;    
    }    
}
```

因为泛型类中的泛型参数的实例化是在定义对象的时候指定的，而静态变量和静态方法不需要使用对象来调用。对象都没有创建，如何确定这个泛型参数是何种类型，所以当然是错误的。

#### 类型擦除后的冲突

```java
class Pair<T>   {  
    public boolean equals(T value) {  
        return null;  
    }        
}  
```

方法重定义了，同时存在两个equals(Object o)。

### 参考文章

- [**_Java深度历险（五）——Java泛型_**](http://www.infoq.com/cn/articles/cf-java-generics?utm_source=infoq&utm_campaign=user_page&utm_medium=link)
- [Java语法糖（3）：泛型](http://www.importnew.com/22529.html)
- [**_ java泛型（二）、泛型的内部原理：类型擦除以及类型擦除带来的问题_**](http://blog.csdn.net/lonelyroamer/article/details/7868820)
- [ java泛型:通配符的使用](http://blog.csdn.net/lonelyroamer/article/details/7927212)
