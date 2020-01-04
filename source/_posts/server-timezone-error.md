---
title: 部署项目至服务器时的时区问题
categories: 
- 踩坑
date: 2019-07-20 16:39:40
tags: 
- docker
- 服务器
- 时区
- jdk
---

前段时间在部署一个项目时，发现部署至服务器之后，系统查询时查出来的数据的时间晚了8个小时，很显然是时区问题，使用了GMT时区而不是CST时区。<!-- more -->

由于在本地运行时是正常的，所以显然是服务器端时区出了问题。整个项目在服务器上使用nginx+docker部署，使用docker拉取jdk镜像并创建jdk容器运行打好的jar包。所以出现问题的地方可能有两处：

- 服务器时区异常
- jdk时区异常

使用`date`命令查看服务器时间正常，所以答案就是jdk时区异常。但是由于是用docker创建的jdk容器，所以最简单解决问题的办法就是jar包运行时加上时区参数。

```java
java -Duser.timezone=GMT+08 -jar app.jar
```

问题成功解决。

最后顺便总结一下常见的时区。

> - GMT(Greenwich Mean Time)，格林威治标准时。英国伦敦郊区的皇家格林尼治天文台的标准时间，也就是本初子午线（0度经线）的当地时间。
>- UTC(Universal Time Coordinated)，协调世界时，又称世界标准时间。比GMT更精准，一般可把两者视为相同。
> - CTT(China Taiwan Time)，中国台湾时间。CTT=GMT+08，和北京时间一致。
>- CST。CST可视为美国、澳大利亚、古巴或中国的标准时间。CST是如下4个不同的时区的缩写：
> 
>美国中部时间：Central Standard Time (USA) UT-6:00
>   
>澳大利亚中部时间：Central Standard Time (Australia) UT+9:30
>   
>中国标准时间：China Standard Time UT+8:00
>   
>古巴标准时间：Cuba Standard Time UT-4:00

所以为了表示中国标准时间，一般使用CTT或者GMT+08，使用CST可能会造成混淆。