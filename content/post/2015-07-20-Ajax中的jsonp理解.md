---
date: 2015-07-20T15:00:48+08:00
title: Ajax中的jsonp理解
tags: ["json"]
---

公司自有服务都是后台api来实现,即使是website也是通过调用api获取来渲染网页,正常的Ajax中请求都是调用浏览器的XMLHttpRequest接口来异步请求数据,但是无论是Chrome,还是IE都有一个限制,就是不能跨域通过XMLHttpRequest来发请求.

话说你有过墙梯我有张良计,聪明的开发者总是会有办法来绕过浏览器跨域的限制来请求数据,在HTML中具有`src`属性的标签例如`script`,`img`都是可用跨域来请求`src`指向的资源数据的,既然`script`标签可以跨域,那就利用这个来实现跨域.但是`script`标签返回的数据不能直接是json格式,需要是标准的javascript函数调用,所以需要返回的数据中包括一个在已有js函数的本地调用方式,传入我们需要的数据,这里的数据才是json格式的.  

发送请求:

```html
<script type="text/javascript" src="http://server2.example.com/RetrieveUser?UserId=1823&jsonp=parseResponse"> </script>
```

<!--more-->
响应内容:

```javascript
parseResponse({"Name": "Cheeso", "Id" : 1823, "Rank": 7})
```

在本地一个parseResponse的函数可用用来处理服务端传入的参数,而一般这个回调函数是作为参数来传递给服务端api的就是`src`属性的jsonp参数.  
最近也是因为客户有跨域请求我们api数据的需求所有了解了一下jsonp,不得不说开发者前辈们的各种机智,在jQuery中使用jsonp更是方便,不需要自己写`script`标签.  

```javascript
<script type="text/javascript">
jQuery(document).ready(function(){
 
    $.ajax({
        type: "get",
        async: false,
        url: "http://flightQuery.com/jsonp/flightResult.aspx?code=CA1998",
        dataType: "jsonp",
        jsonp: "callback",//传递给请求处理程序或页面的，用以获得jsonp回调函数名的参数名(一般默认为:callback)
        jsonpCallback:"flightHandler",//自定义的jsonp回调函数名称，默认为jQuery自动生成的随机函数名，也可以写"?"，jQuery会自动为你处理数据
        success: function(json){
            alert('您查询到航班信息：票价： ' + json.price + ' 元，余票： ' + json.tickets + ' 张。');
        },
        error: function(){
            alert('fail');
        }
    });
});
</script>
```
