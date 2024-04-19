# SQLAlchemy ORM

> 原文：[`docs.sqlalchemy.org/en/20/orm/index.html`](https://docs.sqlalchemy.org/en/20/orm/index.html)

在这里，介绍并完整描述了对象关系映射器。如果你想要使用为你自动构建的更高级别的 SQL，并且自动持久化 Python 对象，请首先转到教程。

+   ORM 快速入门

    +   声明模型

    +   创建一个引擎

    +   发出 CREATE TABLE DDL

    +   创建对象并持久化

    +   简单 SELECT

    +   带 JOIN 的 SELECT

    +   进行更改

    +   一些删除操作

    +   深入学习上述概念

+   ORM 映射类配置

    +   ORM 映射类概述

    +   使用声明式映射类

    +   与 dataclasses 和 attrs 的集成

    +   将 SQL 表达式作为映射属性

    +   更改属性行为

    +   复合列类型

    +   映射类继承层次结构

    +   非传统映射

    +   配置版本计数器

    +   类映射 API

+   关系配置

    +   基本关系模式

    +   邻接列表关系

    +   配置关系连接的方式

    +   处理大型集合

    +   集合定制和 API 详细信息

    +   特殊关系持久化模式

    +   使用传统的 'backref' 关系参数

    +   关系 API

+   ORM 查询指南

    +   为 ORM 映射类编写 SELECT 语句

    +   编写继承映射的 SELECT 语句

    +   启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

    +   列加载选项

    +   关系加载技术

    +   用于查询的 ORM API 功能

    +   遗留查询 API

+   使用会话

    +   会话基础知识

    +   状态管理

    +   级联操作

    +   事务和连接管理

    +   附加持久化技术

    +   上下文/线程本地会话

    +   使用事件跟踪查询、对象和会话的更改

    +   会话 API

+   事件和内部原理

    +   ORM 事件

    +   ORM 内部

    +   ORM 异常

+   ORM 扩展

    +   异步 I/O（asyncio）

    +   关联代理

    +   自动映射

    +   烘焙查询

    +   声明式扩展

    +   Mypy / Pep-484 支持 ORM 映射

    +   变异跟踪

    +   排序列表

    +   水平分片

    +   混合属性

    +   可索引

    +   替代类仪器

+   ORM 示例

    +   映射示例

    +   继承映射示例

    +   特殊 API

    +   扩展 ORM
