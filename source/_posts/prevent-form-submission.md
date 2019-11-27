---
title: 无法阻止表单提交
categories: 日常
date: 2019-11-27 16:14:18
tags:
- 表单默认提交
- js
---

最近在写一个登录页面的时候，在提交表单事件的处理函数中先判断必填字段是否完整，不完整则提示用户并且不发送登录请求。然而，结果却是当用户输入不完整时，提示之后依旧会提交，并且提交的表单信息自动追加在当前地址栏的url之后。<!-- more -->

在我百度“无法阻止表单提交”之后，经过一番研究终于搞懂了原因。都是对前端基础知识不熟导致的，故在此做一下总结。

## 原生表单提交

为了演示，写了一个模拟登录的demo。

后端代码如下，当使用arya，123456登录时才会成功。

```java
@PostMapping("/login")
public @ResponseBody String login(String username, String password) {
    if (username.equals("arya")) {
        if (password.equals("123456")) {
            return "login success";
        } else {
            return "password error";
        }
    } else {
        return "username invalid";
    }
}
```

前端使用原生的写法，没有引入任何框架或组件。

登录表单如下：

```html
<form action="/login" method="post">
    <div>
        <label>username</label>
        <input type="text" name="username" id="username">
    </div>

    <div>
        <label>password</label>
        <input type="password" name="password" id="password">
    </div>

    <div>
        <!-- 两种登录按钮写法都可以 -->
        <!-- <button type="submit">提交</button> -->
        <input type="submit">
        <input type="reset">
    </div>
</form>
```

这个时候一切工作正常，点击type为submit的按钮，会自动将表单数据以post方式提交到/login。但是有2点需要注意：

- 如果form标签不写action和method属性，则会默认以get方式提交到当前地址栏的url。
- button标签的type取值可以为：button，submit，reset，默认为submit。

而如果我们想在提交表单数据之前对输入域做一个判断，限制不完整的表单无法提交时，可以在form标签中添加一个提交表单事件的处理函数`<form action="/login" method="post" onSubmit="login()">`。js代码如下：

```js
function login() {
    var username = document.getElementById("username");
    var password = document.getElementById("password");
            
    if (username.value === null || username.value === '') {
        alert("username cannot be empty");
        return false;
    }

    if (password.value === null || password.value === '') {
        alert("password cannot be empty");
        return false;
    }

    return true;
}
```

但是这样写会发现虽然有输入域的提示，但是依然会提交请求，这是没有阻止掉表单的默认提交导致的。

## 阻止表单默认提交

想要阻止表单的默认提交，有3种方法。

1. 使用return false

将form标签改为`<form action="/login" method="post" onSubmit="return login()">`。

说到这种方法，就必须说一下，在js的事件中调用函数时加不加return的区别。引用网上的解释：

> js在事件中调用函数时用return来返回值实际上是对window.event.returnvalue进行设置。而该值决定了当前事件的默认行为是否执行。
>
> 当返回的是true时，将执行事件的默认行为。当返回是false时，将阻止事件的默认行为。
>
> 而不加return时。将不会对window.event.returnvalue进行设置，所以会默认执行事件。

所以login函数判断输入域为空时return false，会阻止表单的默认提交。只有返回true时才会提交表单。

2. event.preventDefault()

form表单写为`<form action="/login" method="post" onSubmit="login(event)">`。login函数增加一个事件event参数，并添加一行代码`event.preventDefault();`。

使用这个方法可以阻止事件的默认行为，所以添加这行代码后，无论返回true还是false都不会再自动提交表单，所以需要我们手动写ajax请求来提交表单数据。这时可以删除form标签的action和method等属性了。

3. type="button"

将提交表单按钮的type改为button，这时它变为一个“普通”的按钮，不会再触发表单的默认提交事件。我们可以删除form标签的action和method等属性，为提交按钮添加一个onclick事件处理函数，来完成判断输入域和提交表单数据等一系列操作。

## 总结

现在一般的表单提交写法就是直接使用“普通”的提交按钮，在该按钮的点击事件中处理登录逻辑。所以在使用button按钮时千万不要忘了`type="button"`，否则就会因为没有阻止表单的默认提交行为而使自己的函数处理逻辑失效。