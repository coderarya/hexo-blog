---
title: JPA自动建表错误-Table doesn't exist
categories: 踩坑
date: 2020-01-04 20:33:36
tags:
- JPA
- Hibernate
---

这几天在开发一个新项目，项目启动时会抛出一大堆数据库相关的异常。这些异常当中除了生成外键的错误之外，只剩下一个是：`Caused by: java.sql.SQLSyntaxErrorException: Table 'npc_platform.performance' doesn't exist`。持久层使用Spring Data JPA，自动根据实体类生成数据库表。很显然，这是自动建表时出了问题，没有生成performance表。<!-- more -->

### 两个解决方案

经过一番百度，总体得出了两个原因及其相应的解决办法。

1. MySQL驱动的版本问题

因为MySQL驱动8以上的版本将参数`nullCatalogMeansCurrent`的默认值由true改为了false（详见[这里](https://segmentfault.com/a/1190000017264370)），所以建表时会遍历当前数据库连接的所有数据库的表，如果有同名的表存在则不创建当前表，故最后导致表不存在的问题。解决办法有两种：

- 降低MySQL数据库驱动的版本
- 为数据源的url添加参数`nullCatalogMeansCurrent=true`

2. 实体的主键自增长策略问题

将实体类的主键id的生成策略改成`@GeneratedValue(strategy = GenerationType.IDENTITY)`。因为MySQL不支持`GenerationType.AUTO`的生成策略。

由于我的主键生成策略本来就是正确的，于是我尝试了第一个问题的解决方案，然而，很遗憾，并没有解决我的问题。

### 真正的原因

后来有人提示我将sql语句打印出来看看，于是我在配置中将`spring.jpa.show-sql`设置为true，重新启动项目。我检查了一下抛出异常时的那一条sql语句，没有发现什么问题，看起来一切正常。然后我将该sql语句复制到Navicat中执行，这时数据库报错了。

SQL语句：

> alter table performance add constraint FKfb1hwmhggn54m4krusff5e0fu foreign key (group) references npc_member_group (id);

错误提示：

> 1064 - You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near **'group) references npc_member_group (id)'** at line 1

显然是表performance的'group'字段导致了语法问题，一查之下发现果然如此：**group是mysql的关键字**。在将Performance实体类中的group字段改了以后所有的异常消失的干干净净。

“命名不规范，调试两行泪”。这次的调试过程也给了我一条很有用的经验：使用JPA自动建表时，如果遇到数据库相关的错误，一定要打印sql语句看一看，或者复制到数据库中实际运行一下，一定有用！