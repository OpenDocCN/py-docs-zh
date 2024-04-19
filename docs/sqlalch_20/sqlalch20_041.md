# ORM 查询指南

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/index.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/index.html)

本节概述了使用 SQLAlchemy ORM 发出查询的 2.0 样式用法。

本节的读者应该熟悉 SQLAlchemy 统一教程中的 SQLAlchemy 概述，特别是这里的大部分内容扩展了使用 SELECT 语句的内容。

对于 SQLAlchemy 1.x 的用户

在 SQLAlchemy 2.x 系列中，ORM 的 SQL SELECT 语句是使用与 Core 中相同的`select()`构造而构建的，然后在`Session`的上下文中使用`Session.execute()`方法调用（就像用于 ORM-Enabled INSERT、UPDATE 和 DELETE 语句功能的现在使用的`update()`和`delete()`构造一样）。然而，遗留的`Query`对象，它执行与这些步骤相同的操作，更像是一个“一体化”的对象，仍然作为对这个新系统的薄外观保持可用，以支持在 1.x 系列上构建的应用程序，而无需对所有查询进行全面替换。有关此对象的参考，请参阅 Legacy Query API 部分。

+   为 ORM 映射类编写 SELECT 语句

    +   选择 ORM 实体和属性

        +   选择 ORM 实体

        +   同时选择多个 ORM 实体

        +   选择单个属性

        +   将选定的属性与包一起分组

        +   选择 ORM 别名

        +   从文本语句中获取 ORM 结果

        +   从子查询中选择实体

        +   从 UNIONs 和其他集合操作中选择实体

    +   连接

        +   简单的关系连接

        +   链接多个连接

        +   连接到目标实体

        +   使用 ON 子句连接到目标的连接（Joins to a Target with an ON Clause）

        +   将关系与自定义 ON 条件组合（Combining Relationship with Custom ON Criteria）

        +   使用 Relationship 在别名目标之间进行连接（Using Relationship to join between aliased targets）

        +   连接到子查询（Joining to Subqueries）

        +   沿关系路径连接到子查询（Joining to Subqueries along Relationship paths）

        +   引用多个实体的子查询（Subqueries that Refer to Multiple Entities）

        +   设置连接中的最左侧 FROM 子句（Setting the leftmost FROM clause in a join）

    +   关系 WHERE 操作符（Relationship WHERE Operators）

        +   EXISTS 表单：has() / any()（EXISTS forms: has() / any()）

        +   关系实例比较操作符（Relationship Instance Comparison Operators）

+   用于继承映射的写入 SELECT 语句（Writing SELECT statements for Inheritance Mappings）

    +   从基类 vs. 特定子类进行 SELECT（SELECTing from the base class vs. specific sub-classes）

    +   使用 `selectin_polymorphic()`（使用 selectin_polymorphic()）

        +   将 selectin_polymorphic() 应用于现有的急切加载（Applying selectin_polymorphic() to an existing eager load）

        +   将加载器选项应用于由 selectin_polymorphic 加载的子类（Applying loader options to the subclasses loaded by selectin_polymorphic）

        +   在映射器上配置 selectin_polymorphic()（Configuring selectin_polymorphic() on mappers）

    +   使用 with_polymorphic()（Using with_polymorphic()）

        +   使用 with_polymorphic() 过滤子类属性（Filtering Subclass Attributes with with_polymorphic()）

        +   使用 with_polymorphic 进行别名处理（Using aliasing with with_polymorphic）

        +   在映射器上配置 with_polymorphic()（Configuring with_polymorphic() on mappers）

    +   连接到特定子类型或 with_polymorphic() 实体（Joining to specific sub-types or with_polymorphic() entities）

        +   多态子类型的急切加载（Eager Loading of Polymorphic Subtypes）

    +   单一继承映射的 SELECT 语句（SELECT Statements for Single Inheritance Mappings）

        +   为单一继承优化属性加载（Optimizing Attribute Loads for Single Inheritance）

    +   继承加载 API（Inheritance Loading API）

        +   `with_polymorphic()`（`with_polymorphic()`）

        +   `selectin_polymorphic()`（`selectin_polymorphic()`）

+   启用 ORM 的 INSERT、UPDATE 和 DELETE 语句（ORM-Enabled INSERT, UPDATE, and DELETE statements）

    +   ORM 批量 INSERT 语句（ORM Bulk INSERT Statements）

        +   使用 RETURNING 获取新对象（Getting new objects with RETURNING）

        +   使用异构参数字典

        +   在 ORM 批量插入语句中发送 NULL 值

        +   连接表继承的批量插入

        +   使用 SQL 表达式的 ORM 批量插入

        +   遗留会话批量插入方法

        +   ORM“upsert”语句

    +   按主键进行 ORM 批量更新

        +   为具有多个参数集的 UPDATE 语句禁用按主键进行 ORM 批量更新

        +   用于连接表继承的按主键进行批量更新

        +   遗留会话批量更新方法

    +   使用自定义 WHERE 条件的 ORM UPDATE 和 DELETE

        +   ORM 启用的更新和删除的重要说明和注意事项

        +   选择同步策略

        +   使用 RETURNING 与 UPDATE/DELETE 和自定义 WHERE 条件

        +   使用自定义 WHERE 条件的 UPDATE/DELETE 用于连接表继承

        +   遗留查询方法

+   列加载选项

    +   限制列延迟加载的列

        +   使用`load_only()`减少加载的列

        +   使用`defer()`省略特定列

        +   使用 raiseload 防止延迟加载列

    +   配置映射上的列延迟

        +   使用`deferred()`为命令式映射器、映射的 SQL 表达式

        +   使用`undefer()`“急切地”加载延迟列

        +   以组加载延迟列

        +   使用`undefer_group()`按组取消延迟加载

        +   使用通配符取消延迟加载

        +   配置映射器级别的“raiseload”行为

    +   将任意 SQL 表达式加载到对象上

        +   使用 `with_expression()` 加载 UNIONs、其他子查询

    +   列加载 API

        +   `defer()`

        +   `deferred()`

        +   `query_expression()`

        +   `load_only()`

        +   `undefer()`

        +   `undefer_group()`

        +   `with_expression()`

+   关系加载技巧

    +   关系加载风格摘要

    +   在映射时配置加载器策略

    +   带有加载器选项的关系加载

        +   向加载器选项添加条件

        +   使用 Load.options() 指定子选项

    +   惰性加载

        +   使用 raiseload 防止不必要的惰性加载

    +   连接式急加载

        +   连接式急加载的禅意

    +   选择 IN 加载

    +   子查询急加载

    +   使用何种加载方式？

    +   多态急加载

    +   通配符加载策略

        +   每个实体的通配符加载策略

    +   将显式连接/语句路由到急加载集合

        +   使用 contains_eager() 加载自定义过滤的集合结果

    +   关系加载器 API

        +   `contains_eager()`

        +   `defaultload()`

        +   `immediateload()`

        +   `joinedload()`

        +   `lazyload()`

        +   `Load`

        +   `noload()`

        +   `raiseload()`

        +   `selectinload()`

        +   `subqueryload()`

+   查询的 ORM API 特性](api.html)

    +   ORM 加载器选项

    +   ORM 执行选项

        +   填充现有内容

        +   自动刷新

        +   使用每个结果生成器获取大型结果集

        +   身份标记

+   旧版查询 API

    +   查询对象

        +   `查询`

    +   ORM 特定的查询构造
