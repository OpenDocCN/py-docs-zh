# 处理数据

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/data.html`](https://docs.sqlalchemy.org/en/20/tutorial/data.html)

在处理事务和 DBAPI 中，我们学习了如何与 Python DBAPI 及其事务状态进行交互的基础知识。然后，在处理数据库元数据中，我们学习了如何使用`MetaData`和相关对象在 SQLAlchemy 中表示数据库表、列和约束。在本节中，我们将结合上述两个概念来创建、选择和操作关系数据库中的数据。我们与数据库的交互**始终**是在事务的范围内，即使我们已经设置我们的数据库驱动程序在后台使用自动提交。

本节的组成部分如下：

+   使用 INSERT 语句 - 为了将一些数据插入数据库，我们介绍并演示了核心`Insert`构造。从 ORM 的角度来看，INSERT 在下一节使用 ORM 进行数据操作中进行了描述。

+   使用 SELECT 语句 - 本节将详细描述`Select`构造，这是 SQLAlchemy 中最常用的对象。`Select`构造为 Core 和 ORM 中心应用程序发出 SELECT 语句，并且这两种用例将在此处进行描述。在稍后的在查询中使用关系部分以及 ORM 查询指南中还会提到其他 ORM 用例。

+   使用 UPDATE 和 DELETE 语句 - 补充了数据的插入和选择，本节将从核心的角度描述`Update`和`Delete`构造的使用。ORM 特定的 UPDATE 和 DELETE 同样在使用 ORM 进行数据操作部分中进行描述。
