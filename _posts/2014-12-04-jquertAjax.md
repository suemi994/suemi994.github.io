---
layout: post
title: Jquery的异步机制
category: WEB开发
tags: jQuery
date: 2014-12-04
---
{% include JB/setup %}

* 目录
{:toc}

#Jquery的异步机制
----------------------------------------------------------

###异步方式的概述

- 通过事件达到异步操作
- 通过我们最熟悉的回调
- 类promise的方式

首先我们注意到1.5版本前后的jquery有一个重要的不同

~~~javascript
//before 1.5
var option={
    type:'GET',
    url:'...',
    success:function(){..},
    fail:function(){..},
    complete:function(){..}
}；
$.ajax(option);
//after 1.5
$.ajax(option)
    .done(function(){..})
    .fail(function(){..})
    .progress(function(){..})
    .always(function(){..})
            
~~~

1.5版本之后变成了优美的链式调用，并且可以对同一事件增加多个回调函数，原因是原生的xhr对象换成了jqxhr对象，里面有什么魔法呢？

###从promise讲起

promise在javascript编程世界里可以说是大名鼎鼎，下面这段代码给出了它的简单用法。

~~~javascript
promise(function(){..}).
then([successhandler1,successhandler2,..],
     [failhandler1,failhandler2,...]);
~~~
promise为我们呈现出了异步编程的一种新模式，但它还不够漂亮，接下来我们看一看jQuery引入的deferred。

###jQuery的Deferred

####创建Deferred对象

~~~javascript
var de=$.Deferred();//空的Deferred对象
$.Deferred(function(){..}).
  done(function(){..});
//会直接执行里面的function并返回一个Deferred对象
function tmp(defer){
    ....
    return defer;
};//这个写法的含义我们留在后面再讲
~~~

####Deferred的机理
我们可以这样想，每个deferred对象内部有一个隐藏着的状态变量，有成功、失败、执行中三种状态，当程序执行成功则会把它设为成功，否则设为失败。这里的成功失败指的是满足某种设定条件，并非一般意义上的出错，即使抛出异常也不认为其失败。我们可以在程序的运行过程中根据检测到的状态来决定之后哪些回调函数会被执行。

设定状态实例

~~~javascript
$(function(){
    ....
    if(..) this.resolve(arg1,arg2,..);
    //会执行done和always回调，args是传入回调的参数
    //同类函数this.resolveWith(content,[args..]);content作为回调函数中的this
    else this.reject(arg1,arg2,..);
    //执行fail和always回调，rejectWith基本同上
    //需要注意的是always也必须状态改变才能调用，若一直处于执行状态，也不会调用
    ....
    this.notify(args);//调用progress回调，但必须在状态变为成功或失败前发起，这里progress不会被调用
    //notifyWith基本同上
}).fail(..).done(..).progress(..);
~~~

还需要注意的是，即使状态发生改变了，程序还会继续运行。这里所说的成功失败指的是满足我们设定的条件与否，和程序真的运行状态无关，done、fail、always返回的依然是原本的deferred对象。

**请记住无论什么回调都是在函数执行完后调用，函数执行不会被中断**

最后还要注意一点未防止状态在程序外部被更改，应该加上一句
`$.Deferred(function(){..}).promise().done(..).fail(..);`
但promise返回的不再是原本的Deferred对象了，也不能在外部更改其状态了。

###强劲的when
看到这里我们可能有一个疑问，如果我们有一系列任务来决定后来的回调怎么办，不用担心我们有when。

~~~javascript
$.when(defer1,defer2...);
//这时候如果每个defer还要$(func(){..})就很难写了，所以就回到我们开始介绍的最后一种，利用函数返回
var defer1=$.Deferred();
var defer2=$.Deferred();
function func1(defer){...return defer;}
function func2(defer){...return defer;}
$.when(func1(defer1),func2(defer2)).done(..).fail(..);
//当全部defer成功done才会被调用，有一个defer失败fail就会调用，但无论怎样一个回调函数最多被调用一次，且不会影响所有函数的执行
var tmp=9;
$.when(tmp).done(..).fail(..);
//当when链里面全部不是defer变量时会直接执行done回调，注意是全部，有defer变量会跳过非defer，回调取决于defer
~~~

