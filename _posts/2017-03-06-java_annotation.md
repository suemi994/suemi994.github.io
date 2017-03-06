---
layout: post
title: 聊聊Java的注解及实现
category: 学习总结
tags: java
date: 2017-03-06
---
{% include JB/setup %}


* 目录
{:toc}

---

### 摘要

Annotation（注解）就是Java提供了一种元程序中的元素关联任何信息和着任何元数据（metadata）的途径和方法。Annotion(注解)是一个接口，程序可以通过反射来获取指定程序元素的Annotion对象，然后通过Annotion对象来获取注解里面的元数据。Annotation(注解)是JDK5.0及以后版本引入的。它可以用于创建文档，跟踪代码中的依赖性，甚至执行基本编译时检查。从某些方面看，annotation就像修饰符一样被使用，并应用于包、类 型、构造方法、方法、成员变量、参数、本地变量的声明中。这些信息被存储在Annotation的“name=value”结构对中。

### 注解基础

　Annotation的成员在Annotation类型中以无参数的方法的形式被声明。其方法名和返回值定义了该成员的名字和类型。在此有一个特定的默认语法：允许声明任何Annotation成员的默认值：一个Annotation可以将name=value对作为没有定义默认值的Annotation成员的值，当然也可以使用name=value对来覆盖其它成员默认值。这一点有些近似类的继承特性，父类的构造函数可以作为子类的默认构造函数，但是也可以被子类覆盖。

　　Annotation能被用来为某个程序元素（类、方法、成员变量等）关联任何的信息。需要注意的是，这里存在着一个基本的规则：Annotation不能影响程序代码的执行，无论增加、删除 Annotation，代码都始终如一的执行。另外，尽管一些annotation通过java的反射api方法在运行时被访问，而java语言解释器在工作时忽略了这些annotation。正是由于java虚拟机忽略了Annotation，导致了annotation类型在代码中是“不起作用”的； 只有通过某种配套的工具才会对annotation类型中的信息进行访问和处理。本文中将涵盖标准的Annotation和meta-annotation类型，陪伴这些annotation类型的工具是java编译器（当然要以某种特殊的方式处理它们）。

元数据从metadata一词译来，就是“关于数据的数据”的意思。
　　元数据的功能作用有很多，比如：你可能用过Javadoc的注释自动生成文档。这就是元数据功能的一种。总的来说，元数据可以用来创建文档，跟踪代码的依赖性，执行编译时格式检查，代替已有的配置文件。如果要对于元数据的作用进行分类，目前还没有明确的定义，不过我们可以根据它所起的作用，大致可分为三类： 

1. 编写文档：通过代码里标识的元数据生成文档
2. 代码分析：通过代码里标识的元数据对代码进行分析
3. 编译检查：通过代码里标识的元数据让编译器能实现基本的编译检查
　　
在Java中元数据以标签的形式存在于Java代码中，元数据标签的存在并不影响程序代码的编译和执行，它只是被用来生成其它的文件或针在运行时知道被运行代码的描述信息。
综上所述：
　　　　
- 元数据以标签的形式存在于Java代码中。
- 元数据描述的信息是类型安全的，即元数据内部的字段都是有明确类型的。
- 元数据需要编译器之外的工具额外的处理用来生成其它的程序部件。
- 元数据可以只存在于Java源代码级别，也可以存在于编译之后的Class文件内部。

### 常用注解

根据注解参数的个数，我们可以将注解分为三类：

- 标记注解:一个没有成员定义的Annotation类型被称为标记注解。这种Annotation类型仅使用自身的存在与否来为我们提供信息。比如后面的系统注解@Override;
- 单值注解
- 完整注解　　

根据注解使用方法和用途，我们可以将Annotation分为三类：
- JDK内置系统注解
- 元注解
- 自定义注解

![Java注解架构图](http://images.cnitblog.com/blog/34483/201304/25200814-475cf2f3a8d24e0bb3b4c442a4b44734.jpg )


### 自定义注解

在这里就只给个例子，请务必查阅参考文章的第二篇

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FruitColor {
    /**
     * 颜色枚举
     * @author peida
     *
     */
    public enum Color{ BULE,RED,GREEN};
    
    /**
     * 颜色属性
     * @return
     */
    Color fruitColor() default Color.GREEN;

}
```

顺便再强调几个重要的点：

- 注解元素必须有确定的值，要么在定义注解的默认值中指定，要么在使用注解时指定，非基本类型的注解元素的值不可为null。因此, 使用空字符串或0作为默认值是一种常用的做法。这个约束使得处理器很难表现一个元素的存在或缺失的状态，因为每个注解的声明中，所有元素都存在，并且都具有相应的值，为了绕开这个约束，我们只能定义一些特殊的值，例如空字符串或者负数，一次表示某个元素不存在，在定义注解时，这已经成为一个习惯用法。
- Annotation类型里面的参数只能用public或默认(default)这两个访问权修饰.例如,String value();这里把方法设为defaul默认类型。
- 参数成员只能用基本类型byte,short,char,int,long,float,double,boolean八种基本数据类型和 String,Enum,Class,annotations等数据类型,以及这一些类型的数组.例如,String value();这里的参数成员就为String。
- 如果只有一个参数成员,最好把参数名称设为"value",后加小括号.例:下面的例子FruitName注解就只有一个参数成员。

### 处理注解

　　Java使用Annotation接口来代表程序元素前面的注解，该接口是所有Annotation类型的父接口。除此之外，Java在java.lang.reflect 包下新增了AnnotatedElement接口，该接口代表程序中可以接受注解的程序元素，该接口主要有如下几个实现类：

- Class：类定义
- Constructor：构造器定义
- Field：累的成员变量定义
- Method：类的方法定义
- Package：类的包定义

java.lang.reflect 包下主要包含一些实现反射功能的工具类，实际上，java.lang.reflect 包所有提供的反射API扩充了读取运行时Annotation信息的能力。当一个Annotation类型被定义为运行时的Annotation后，该注解才能是运行时可见，当class文件被装载时被保存在class文件中的Annotation才会被虚拟机读取。

AnnotatedElement 接口是所有程序元素（Class、Method和Constructor）的父接口，所以程序通过反射获取了某个类的AnnotatedElement对象之后，程序就可以调用该对象的如下四个个方法来访问Annotation信息：

- 方法1：<T extends Annotation> T getAnnotation(Class<T> annotationClass): 返回改程序元素上存在的、指定类型的注解，如果该类型注解不存在，则返回null。
- 方法2：Annotation[] getAnnotations():返回该程序元素上存在的所有注解。
- 方法3：boolean is AnnotationPresent(Class<?extends Annotation> annotationClass):判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false.
- 方法4：Annotation[] getDeclaredAnnotations()：返回直接存在于此元素上的所有注释。与此接口中的其他方法不同，该方法将忽略继承的注释。（如果没有注释直接存在于此元素上，则返回长度为零的一个数组。）该方法的调用者可以随意修改返回的数组；这不会对其他调用者返回的数组产生任何影响。

下面给一个处理的例子

```java
@Target(value = { ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface UserAnnotation {
	public int id() default 0;
	public String name() default "";
	public int age() default 18;
	public String gender() default "M";
}

public class TestMain
{
  @UserAnnotation(age=20,gender="F",id=2014,name="zhangsan")//注解的使用
  private Object obj;
  
  public static void main(String[] args) throws Exception
  {
     Filed objField = TestMain.class.getField("obj");
     UserAnnotation ua = objField.getAnnotation(UserAnnotation.class);//得到注解,起到了标记的作用
     
    System.out.println(ua.age()+","+ua.gender()+","+ua.id()+","+ua.name());
    //***进一步操作的话，假设Object要指向一个User类，那么可以讲注解的值给他
    TestMain tm = new TestMain();
    objFiled.set(tm,new User(ua.age(),ua.gender(),ua.id(),ua.name())); //不错吧，将自己的信息送给obj，起到了附加信息的作用
    
    //-----------请自由遐想吧~~，下面来说说注解怎么能获得注解自己的注解-------------
   Target t = ua.annotationType().getAnnotation(Target.class)
   ElementType[] values = t.value();
   //~~~~~~~~~~~~~~完了，再一次自由遐想吧~~~~~~~~~~~~~~
   
   Sysout.out.println("注意：是遐想，不是瞎想！！");
  }
}
```

### 注解实现原理

前面介绍了如何使用Java内置的注解以及如何自定义一个注解，接下去看看注解实现的原理，看看在Java的大体系下面是如何对注解的支持的。还是回到上面自定义注解的例子，对于注解Test，如下，如果对AnnotationTest类进行注解，则运行时可以通过AnnotationTest.class.getAnnotation(Test.class)获取注解声明的值，从上面的句子就可以看出，它是从class结构中获取出Test注解的，所以肯定是在某个时候注解被加入到class结构中去了。

```java
@Test("test")
public class AnnotationTest {
    public void test(){
    }
}
```

从java源码到class字节码是由编译器完成的，编译器会对java源码进行解析并生成class文件，而注解也是在编译时由编译器进行处理，编译器会对注解符号处理并附加到class结构中，根据jvm规范，class文件结构是严格有序的格式，唯一可以附加信息到class结构中的方式就是保存到class结构的attributes属性中。我们知道对于类、字段、方法，在class结构中都有自己特定的表结构，而且各自都有自己的属性，而对于注解，作用的范围也可以不同，可以作用在类上，也可以作用在字段或方法上，这时编译器会对应将注解信息存放到类、字段、方法自己的属性上。

在我们的AnnotationTest类被编译后，在对应的AnnotationTest.class文件中会包含一个RuntimeVisibleAnnotations属性，由于这个注解是作用在类上，所以此属性被添加到类的属性集上。即Test注解的键值对value=test会被记录起来。而当JVM加载AnnotationTest.class文件字节码时，就会将RuntimeVisibleAnnotations属性值保存到AnnotationTest的Class对象中，于是就可以通过AnnotationTest.class.getAnnotation(Test.class)获取到Test注解对象，进而再通过Test注解对象获取到Test里面的属性值。

这里可能会有疑问，Test注解对象是什么？其实注解被编译后的本质就是一个继承Annotation接口的接口，所以@Test其实就是“public interface Test extends Annotation”，当我们通过AnnotationTest.class.getAnnotation(Test.class)调用时，JDK会通过动态代理生成一个实现了Test接口的对象，并把将RuntimeVisibleAnnotations属性值设置进此对象中，此对象即为Test注解对象，通过它的value()方法就可以获取到注解值。

Java注解实现机制的整个过程如上面所示，它的实现需要编译器和JVM一起配合。

### 参考文章

- [**注解基本概念**](http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html)
- [**_自定义注解入门_**](http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html)
- [注解的基本原理和编程实现](https://my.oschina.net/ososchina/blog/345288)
- [**注解机制及其原理**](http://blog.csdn.net/wangyangzhizhou/article/details/51698638)
