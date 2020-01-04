---
title: NexT主题配置踩坑
categories: 踩坑
date: 2019-08-29 15:13:24
tags:
- next
- valine
- hexo
---

在配置next主题时，遇到了一些问题，解决的过程也很有意思，因此在本文做个记录。

官方网址：[NexT主题官网](https://theme-next.org)（英文），[旧版官网](http://theme-next.iissnan.com/)（中文）。<!-- more -->

## Valine

Valine评论系统还是相当好用的，其使用LeanCloud云存储服务管理评论数据。在这里推荐使用[LeanCloud国际版](https://leancloud.app/)，可以免去实名认证和绑定备案域名的烦恼。在按照官网的教程配置好之后，发现渲染出来的评论框下面有个“powered by valine”的标志。本来使用了人家的东西为人家顺便做个广告也是理所当然，但是总觉得有这个标志太难看了，所以就开始想办法把这个标志去掉。

首先百度出来一个方法，在valine模板文件`next\layout\_third-party\comments\valine.swig`中添加如下代码：

```js
var infoEle = document.querySelector('#comments .info');
if (infoEle && infoEle.childNodes && infoEle.childNodes.length > 0) {
    infoEle.childNodes.forEach(function(item) {
        item.parentNode.removeChild(item);
    });
}
```

{% asset_img 1.png %}

这段js代码做的事情很简单：找到info类的元素，遍历删除其所有子元素。所以这个info类的元素很有可能就是包含“powered by valine”的元素。为了证实这个想法，先不添加上面这段代码，本地启动hexo，随便打开一篇文章，查看其DOM元素结构，发现果然是这样：

{% asset_img 2.png %}

id为comments的div包含的就是valine评论的部分，而info类的div包含的就是我想要去掉的部分。

于是我就把上面的代码添加到swig文件中，重新渲染生成，本地启动hexo。本以为一切ok的我发现“powered by valine”仍然在那，这就让我很纳闷了，于是我就到浏览器上调试一下，看看发生了什么。

找到添加的那段代码，打个断点，发现执行到if语句的时候，infoEle竟然为null。

{% asset_img 3.png %}

这时候再去看看DOM结构，发现现在id为comments的div内没有内容。

{% asset_img 4.png %}

原来在这个时候，valine评论组件还没有插入到DOM树当中去，所以在这个时候想通过js删除相应的div自然会失败。这个时候再回过头来观察一下valine.swig这个文件的截图，发现在添加的代码前面有一段`new Valine({...});`的代码，这显然是在初始化valine评论组件，但是valine并没有在初始化之后立即插入到DOM结构中。这个插入DOM树的时间的控制，我猜应该是valine的开发者故意为之，是让我们无法轻易的去掉“powered by”标志。至于为什么网上其他人可以通过添加上面这段代码去掉这个标志，我猜应该是valine版本更新的缘故。

明白了失败的原因就好解决了，只要使我们添加的这段代码在valine组件插入到DOM树之后执行就好了，这里我使用window.onload方法。window.onload是在页面资源（比如图片和媒体资源，它们的加载速度远慢于DOM的加载速度）加载完成之后才执行。将上面的代码改为这样：

```js
window.onload = function () {
	var infoEle = document.querySelector('#comments .info');
	if (infoEle && infoEle.childNodes && infoEle.childNodes.length > 0) {
        infoEle.childNodes.forEach(function(item) {
            item.parentNode.removeChild(item);
        });
	}
};
```

或者使用css将info类元素隐藏掉：

```js
window.onload = function () {
	var info = document.querySelector('#comments .info');
	if (info) {
		info.style.display = "none";
	}
};
```

重新渲染生成，本地启动hexo，发现大功告成。

当然，没有什么是不可以通过修改源码来解决的，直接修改valine的源码也可以去掉“powered by”的标志。这个也比较简单，找到输出“powered by”标志的相应代码删除，将js文件保存在next主题的`source/js`中，然后在valine.swig中引入就可以了。

## 不蒜子

在使用[不蒜子](http://busuanzi.ibruce.info/)统计访问量的时候，配置好之后在本地启动，发现访问量初始数字很大，又没有找到修改初始化访问量的地方。最后发现部署到线上之后就是正常的了，在本地访问量确实是错误的，遇到这个问题不必惊慌。

