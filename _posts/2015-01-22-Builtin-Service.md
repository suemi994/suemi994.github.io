---
layout: post
title: Zeta.JS之内置服务
date: 2015-01-22
tag: zeta
category: Web开发
---

##前言
Zeta.js 是一款为node打造的轻量级后端框架，引入了许多angular的概念，可以让你以一种不同于express的更有层次的方式编写后端代码。这里是Zeta的中文文档。

[Source On GitHub](https://github.com/BenBBear/Zeta)

[Website For Zeta](http://zetajs.io/)

##概览
Zeta.js提供了一些十分常见的服务给后端web开发者们使得工作更加简单方便。在接下来的内容里，我们将分别介绍这些内置服务。

##渲染
渲染服务既可以用于字符串也可以用于文件，目前还只支持html文本和swig模板，但是你可以十分方便地扩展这一服务因为它是使用Provder实现的。

###渲染字符串
你需要显式地使用Provider $render来实现字符串的渲染，字符串里也可以包含变量。

```js
demo.get('/',function($scope,$render){
    $scope.end($render.text('<p></p>'),{
        msg:"hello,world"
    });
});
//demo is a module
```

###渲染文件
我们暴露出一个简易的函数接口供html或swig文件的渲染。

```js
demo.get('/',function($scope){
    $scope.render('/index.swig',{
        title:'Welcome'
    })
});
```

##Cookie
Cookie服务在章节工厂里已经提到过，而实际上它也确实是用Factory实现的。

###提取cookie

```js
var user=$cookie.val('user');
//return a cookie named user
```

###设置cookie

```js
//reset the value of cookie
$cookie.val('user',Json.stringfy({name:'bevis'}));
//initialize the cookie
$cookie.val('user','bevis',{
    path:'/',
    maxAge:1000
});
//set value of cookie & cookie 
$cookie.val('user','bevis','maxAge',10000);

//write the cookie to client
$cookie.write($scope);

//a complete example
demo.get('/',function($scope,$cookie){
    $cookie.val('user','bevis',{
        path:'/user',
        maxAge:10000
    });
    $cookie.write($scope);
    $scope.send('').end();
});
```

##表单
我们使用大名鼎鼎的formidable插件来处理表单提交，无论是post方法提交的json对象还是文件上传的form-data，都可以处理。工厂$form会返回一个formidable中的IncomingForm对象，其余的你可以参照formidable的文档。

```js
demo.post('/',function($scope,$form){
    .....
});
//$form is a IncomingForm object of formidable
```

##静态服务器

静态服务器功能返回与请求路径一致的静态文件如果你并没有为这些路径指定处理函数。

###开启服务

```js
demo.config('public',__dirname+'/public');
demo.load();
demo.any('static');
```

举个例子，当你发起对于路径'/img/avatar.jpg'的请求时，客户端将得到文件路径为'/public/img/avatar.jpg'的图像。如果文件不存在的话，如果你设置了的话，你应该会得到一个404。对于子目录的请求也支持，同时你可以省略路径最末端的'/'字符。

###Index的情况
静态服务器会自动查找由请求指定的目录下名为index有特定后缀的文件，后缀可以由你设定。目前支持的后缀有html，htm及md三种。设置方法如下。

```js
demo.config.of('built-in').of('static-server').val('indexFile',['.html','.md']);
//allow index file with suffix as html or md
//you get /views/index.html when request for path /views
```

另外，后缀名的优先级同他们在配置数组里的顺序一致，以上文为例，html文件优先于md文件，二者同时存在时会返回html文件。
