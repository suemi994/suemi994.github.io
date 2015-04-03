---
layout: post
category: WEB开发
title: 锋利的jQuery
tags: jQuery
date: 2014-11-25
---
{% include JB/setup %}

* 目录
{:toc}

#锋利的jQuery
------------------------------------------

###jquery对象

jquery最重要的就是其对象，它的调用基本可以分成方法和函数（方法当然是函数，这里这么说是把由对象调用的和由类调用的区分开来），所以得到jQuery对象就十分重要，第一部分只有一个函数$()

~~~javascript
$(document.getElementById('mydiv'));
$(this)
//从原生对象中生成jQuery对象
var dst=$(CSS-selector);
//利用CSS选择器从文档中得到jquery对象
typeof dst[0]//并非jQuery对象而是原生对象
$('body')[0]==document.body//true
$('.btn-lg',$('body'));
//在body中得到类为btn-lg的元素组装成为对象，$()的第二参数为上下文
$('<img/>',{
    src:"http://www.example.com/img/00.jpg",
    css:{
        width:90px;
        height:45px;
    },
    click:handler
});
//产生一个新对象
//最后一个用法，与上面不同，用来做事件注册
$(func);
//等同于$(document).ready(func); func里的this指向原生对象document
~~~

###jQuery属性方法

jquery的getter和setter不做区分是同一个函数，设定值时就是setter，否则就是getter，但getter得到的是jQuery对象中第一个元素对应的属性值
下面首先介绍有哪些属性方法

- attr，处理html属性
- css，处理css属性
- addClass，removeClass，增添删除对应CSS类
- toggleClass，没有则加，有则删
- hasClass，jQuery对象元素中至少有一个满足就返回true，否则false
- val，操作表单各项，比如文本，复选框等
- 各种坐标属性方法如offset，innerWidth
- data与removeData，将数据绑定到元素上

用法，以下用func指代属性方法

~~~javascript
$('..').func(attribute-name);
$('..').func(name,value);
$('..').func({
    name1:value1,
    name2:value2,
    ...
})
$('..').func(name,function);//function的返回值是name的值
//特例
$('..').val()//get方法
$('..').val(value)//set方法
$('..').is('CSS-selector')//至少有一个满足
~~~

###修改文档结构
基本分成两种形式

~~~javascript
$(target).position($(content));
//上面加$是为了说明均是jQuery对象，position指的是一些位置描述，比如after，before，append等
$(content).actionPosition($(target));
//action代表采取的操作如insert、append和replace等，position代表相对位置如To、All和after等
//比较特殊的例子
$('..').clone()
//clone不会复制事件处理器，要复制的话可以在注册时采用live方法绑定
$('..').wrap(document-element)
//对每一个选中元素外面加一层包装
$('..').wrapall(document-element)
//将选中元素作为一组来包装
$('..').wrapInner(document-element)
//包装每个选中元素的内容
~~~

具体的操作函数太多了就不一一列出了

###事件处理

这一部分在我之前的博客中有详细讲述，还是复制粘贴以下吧
[Web之事件处理](http://www.jianshu.com/p/8d1c0c8ea4c9)

####jquery事件处理器
- jquery的事件处理程序一般只有一个参数（必定是第一个参数），那就是jquery事件对象。
- jquery处理器中的this指向的是对应html元素，需要转成jquery对象，看下面例子

~~~
$("p").click(function(){
    $(this).css({background,#555555});  
});
~~~

- jquery处理器的返回值始终有意义，如果返回false，该事件的默认行为及接下来的冒泡传播均会停止，类似于调用了stopPropagation和preventDefault.
- 要想传入多个参数，需用trigger函数显式触发事件。

####jquery事件对象

包含了jquery事件的详细信息，有诸多NB属性，比如data属性，还定义了preventDefault等方法。

####jquery注册

#####简单注册

~~~
$(".sign").click(handler);
//jquery对象.简单事件类型名(处理器);
//注意，以下为特例
$(selector).hover(f,g);
//为鼠标放上去和离开分别注册
$(selector).toggle(f1,f2,f3...);
//第i次事件调用fi
~~~

#####高级注册

~~~
$('a').on('mouseover mouseleave',handler);
$('a').on({
    mouseover:f,
    mouseleave:g
});
$('a').on('mouseover',{sb:6},handler);
//中间参数会作为jquery事件对象的data属性值
//对于动态元素怎么办呢
$(parent).on(name,childSelector,{sb:6},handler);
//用父元素为它注册
//one方法使用与on一致，但只起一次作用，搞完就注销
~~~

####事件注销

~~~
$(selector).off('EventName');
//移除该事件所有处理器
$(selector).off(jquery对象);
$('a').off({
    mouseover:f,
    mouseleave:g
});//移除单个处理器
~~~

####事件触发

~~~
$('a').click();
//$(selector).SimpleEventName();但这样不能手动触发addEventListener和attachEvent注册的处理器
$(selector).trigger(EventName,[args]);
//triggerHandler方法触发的事件不会冒泡也没有默认操作
$(selector).live(Name,handler)
//为动态元素即在注册后添加的元素也注册，取消用die，用法基本同bind与unbind,在1.7版本后不可用，全部综合为on和off了
~~~

###jQuery的AJAX

在jQuery1.5版之后下面的方法均返回的是jqXHR对象，其回调函数都可以使用链式调用添加，回调函数中的this指向ajax事件对象，需要注意done函数和其他的参数顺序有出入，下面就不一一举例了


####辅助方法

~~~
$.load(url)
//调用http的get方法，并异步加载对应内容,默认得到html
$.load(url,{..});
//当第二个对象为json对象，时采用Post
$.load(url,{..},callback(data,status,req));
//第三个参数，没有json时为第二个参数回调函数，其参数分别是
//返回的数据，状态码如success，请求对象，可以缺省
//回调函数可缺省，其格式如上，不做特殊说明，下面均如此
$.getScript(url,callback(data,status,req));
//得到脚本后执行，再执行回调函数，默认得到javascript
$.getJSON(url,JSON,callback(data,status,req));
//中间参数可以缺省，这样回调函数就变第二个
//中间加数据的话，会变成字符串加在url的'&'和'?'后，也就是说方法依然是get了
~~~

####常用方法

下面介绍两个十分常用的方法get和post

~~~
$.get(url,object,callback(data,status,xhr),type);
$.get(url,object,type).done(function(data,status,xhr){});
//get和post返回的同样是jqXHR对象，是一个deferred对象，所以可以链式调用done、fail、complete等
$.post(url,object,callback(data,status,xhr),type);
$.post(url,object,type).done(function(data,status,xhr){});
//基本同上面一致，但type是希望得到的数据类型，比如json（但要用
//$.parseJSON解析，还有script（传入回调函数前会执行），text等
~~~

####最根本的方法

ajax函数是最复杂也是最根本的方法

~~~
var option={
    type:'GET',
    url:'url',
    data:string or JSON,
    dataType:'JSON',
    done:callback,
    error:errorhandler
};
$.ajax(option);
//ajax可以设置回调选项以在不同阶段做不同的回调处理
//阶段：beforesend,done(success),complete,error

//1.5版本之后可以用更舒服的写法
$.ajax(option).
    success(function(data,status,xhr){..}).
    fail(function(xhr,status,err){..}).
    always(function(xhr,status,err){..});
//注意回调函数参数顺序的不同，回调函数中this指向ajax事件
~~~

###工具函数

~~~
$.each(list,f(index,value));
/*f的参数分别是对象的系列值和对应的值或者元素属性名和对应值，其
this和第二个参数一致，当返回false时循环停止*/
$.extend(dst,src);
//求两个对象的并集并赋给第一个，src中值为undefined和null会被忽略
$.merge(dst,src)
//同上面差不多，只不过对于类数组对象
$.map(list,f(index,value));
//list是类数组对象，其中每个元素被f处理，f返回值组成新数组返回
$.parseJSON(string);
//传入JSON格式字符串得到JSON对象
~~~

工具函数还有许多，这里只是列举了最常用的一些，其余的还是查找文档吧。

###为jQuery写插件

~~~javascript
(function($){
    $.fn.pluginname=function(..){
        ...
    };
})(jQuery);
~~~

###注意事项

- 好的命名规则，对于jQuery对象应该有特殊的命名规则比如在前面一致加上'$'，才能免于与原生对象和一些临时变量混淆
- 使用链式调用，会使得我们对操作的jQuery对象更清楚
