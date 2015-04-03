---
layout: post
title: 设计模式之装饰者
category: 编程技巧
tags: 设计模式
date: 2015-02-15
---
{% include JB/setup %}

* 目录
{:toc}

### 前言
这是本人的设计模式学习笔记，把自己学习过程中的一些总结和认识记录下来，与诸君共勉。

### 基本概念
所谓装饰者，就是为之前的对象添加行为的存在，允许行为可以被修改而无需修改现有代码。要想了解装饰者，首先要介绍几个重要的概念：

- 组件：被装饰的对象
- 装饰者：本身既是对象，以组件为参，返回装饰后的对象
- 接口：功能的组合，在oo语言中是要求类必须实现的方法

我们首先要了解以下特性：

- 装饰者和组件有共同的超类
- 组件可以被任意多次装饰
- 装饰前和装饰后应该暴露出同样的接口
- 装饰 使用继承，但扩展的方式和继承不同
- 行为可以添加在方法前和方法后

我们用下面这幅图来揭示装饰者的结构：
![装饰者结构示意图](/assets/img/2015-02-15-0.png)

下面给出一段代码的例子：

~~~js
//共有超类
var org=function(){this.cost=0;};
org.prototype.getCost=function(){};

//组件
var component=function(num){this.cost=num;};
component.prototype=new org();
component.prototype.getCost=function(){return this.cost;};

//装饰者
var decorator=function(component){
	this.base=component;
};
decorator.prototype=new org();
decorator.prototype.getCost=function(){
	return this.base.getCost()+1.0;
};

//装饰过程

var exp=new component(1.0);
var exp2=new decorator(exp);
exp2.getCost();//2.0
~~~

这里用js其实不大恰当，并不能很好地解释装饰者模式，之所以用js来说明，是想说装饰者模式并非仅仅限于oo语言的场合，对于动态语言，它依然大有用处。

### 应用场合
装饰者适用各种行为可以大量组合起来，这种时候不使用装饰者的话就不得不为所有的组合情况都定义一个类，这无疑是一个灾难。使用装饰者的话，就只用预先定义组合的元素，对于不同的组合方式，动态地实现要求的接口，显然更加优越。

举一个例子，卖电脑器材，以每一次交易为一个对象，为了计算出交易金额和记录交易配件，如果使用继承的话，需要列出所有器材的组合情况，而使用装饰者模式的话，就是把每一种器件都做成一个装饰者。当然在这个例子里面好像这也不是什么好的解决方式，你可以用一个列表来记录所卖的所有器件，然后写个函数遍历列表计算。写出这个例子也是为了告诉我们尽管很多地方可以使用设计模式，但不要做设计模式驱动的开发，而是做设计原则驱动的开发。

### 设计原则
- 对扩展开发，对修改关闭
- 多用组合，少用继承

