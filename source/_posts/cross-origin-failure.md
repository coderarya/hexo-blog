---
title: Spring Security配置错误导致跨域请求失败
categories: 踩坑
date: 2019-11-29 12:33:56
tags:
- spring security
- 跨域
- CORS
- Jwt
---

前段时间在开发一个前后端分离的项目的时候，遇到了一个问题：已登录的用户带着token请求后端接口时，一直返回401。

整个项目使用vue + spring boot开发，权限控制使用[Jwt](https://jwt.io/) + spring security实现。我在后端过滤的时候如果发现用户未登录，却要访问需要登录权限的接口时就返回401提示未登录。但是我的这个请求显然是已登录并且带着token访问的。Google一番之后终于搞明白了，一切都是因为对跨域和CORS不熟悉所致的，故在此做一个笔记与总结。<!-- more -->

### 跨域与同源策略

> 当一个资源从与该资源本身所在的服务器**不同的域、协议或端口**请求一个资源时，资源会发起一个**跨域HTTP请求**。

例如：站点 http://domain-a.com 的某HTML页面通过\<img\>的 src请求 http://domain-b.com/image.jpg 。网络上的许多页面都会加载来自不同域的CSS样式表，图像和脚本等资源。

跨域请求会引发一系列的安全问题（具体的问题在此不做描述）。所以，为了保障安全，浏览器采用了同源策略。

> 如果两个URL的协议、域名和端口都相同，我们就称这两个URL同源。

同源策略限制了以下行为：

- 同源策略限制了来自不同源的 JavaScript 脚本对当前 DOM 对象读和写的操作。
- 同源策略限制了不同源的站点读取当前站点的 Cookie、IndexDB、LocalStorage 等数据。
- 同源策略限制了通过 XMLHttpRequest 等方式将站点的数据发送给不同源的站点。

那么如何突破同源策略的限制呢？方法较多，我接触过的有3种：JSONP，CORS，Nginx代理。下面重点来介绍主流的跨域解决方案：CORS。

### CORS

CORS是一个W3C标准，全称是"跨域资源共享"（cross-origin resource sharing）。它使用额外的 HTTP头来告诉浏览器，让运行在一个origin(domain)上的Web应用被允许访问来自不同源服务器上的指定的资源。

CORS需要浏览器和服务器同时支持。目前，所有浏览器都支持该功能（IE浏览器不能低于IE 10）。也就是说，现在我们发起**满足CORS条件**的跨域请求时，浏览器会自动使用CORS机制。

> CORS允许在下列场景中使用跨域 HTTP 请求：
> - 由`XMLHttpRequest`或Fetch发起的跨域HTTP请求。
> - Web 字体(CSS 中通过` @font-face `使用跨域字体资源)，因此，网站就可以发布TrueType字体资源，并只允许已授权网站进行跨站调用。
> - WebGL贴图
> - 使用 `drawImage` 将Images/video画面绘制到canvas

CORS新增了一组 HTTP 首部字段，允许服务器声明哪些源站通过浏览器有权限访问哪些资源。例如：Access-Control-Allow-Origin（允许哪些源站），Access-Control-Allow-Methods（允许哪些方法），Access-Control-Allow-Headers（允许哪些请求头）等。

**另外，对那些可能对服务器数据产生副作用的HTTP请求方法，浏览器必须首先使用OPTIONS方法发起一个预检请求（preflight request）**，从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的HTTP请求。

#### 简单请求

某些请求不会触发CORS 预检请求，我们称这样的请求为“简单请求”。若请求满足所有下述条件，则该请求可视为“简单请求”：

1. 使用下列方法之一：GET，HEAD，POST
2. 不得人为设置下列集合之外的其他首部字段：
   - Accept
   - Accept-Language
   - Content-Language
   - Content-Type（需要注意额外的限制）
   - DPR
   - Downlink
   - Save-Data
   - Viewport-Width
   - Width
3. Content-Type的值仅限于下列三者之一：
   - text/plain
   - multipart/form-data
   - application/x-www-form-urlencoded
4. 请求中的任意XMLHttpRequestUpload对象均没有注册任何事件监听器。XMLHttpRequestUpload对象可以使用XMLHttpRequest.upload属性访问。
5. 请求中没有使用ReadableStream对象。

一般来说，我们只用关注前3个条件就好了。

#### 预检请求

相对的，对于不满足简单请求条件的跨域请求，浏览器会先使用OPTIONS方法发起一个预检请求（preflight request）到服务器，以获知服务器是否允许该实际请求。如果允许，则再发送实际请求，完成整个请求过程。使用预检请求的好处是，可以避免跨域请求对服务器的用户数据产生未预期的影响。

### 问题及解决方案

我遇到的这个问题的原因就是我在spring security中的配置出错了，没有放行OPTIONS请求。我在本地开发时，前端vue项目和后端启的是两个服务器，占用不同的端口，所以产生了跨域请求。而我的请求由于带了自定义的头部Authentication: token不满足简单请求的条件，所以会发送OPTIONS预检请求，因此会被spring security拦截返回401。

所以，解决方案就是在spring security的配置中增加以下代码：

```java
http
... //其它配置
.antMatchers(HttpMethod.OPTIONS, "/**")
.permitAll()
... //其它配置
```

部署到服务器上时，使用Nginx统一代理前后端将不会产生跨域问题，故也不需要此配置。

##### 参考

[HTTP访问控制（CORS）](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)

[浏览器的同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)

