# 模式定义语言

> 原文：[`docs.sqlalchemy.org/en/20/core/schema.html`](https://docs.sqlalchemy.org/en/20/core/schema.html)

本节涉及 SQLAlchemy **模式元数据**，这是一种全面描述和检查数据库模式的系统。

SQLAlchemy 查询和对象映射操作的核心由 *数据库元数据* 支持，它由描述表和其他模式级对象的 Python 对象组成。这些对象是三种主要类型操作的核心 - 发出 CREATE 和 DROP 语句（称为 *DDL*）、构造 SQL 查询以及表达有关已存在于数据库中的结构的信息。

数据库元数据可以通过显式命名各种组件及其属性来表示，使用诸如 `Table`、`Column`、`ForeignKey` 和 `Sequence` 等构造，所有这些都从 `sqlalchemy.schema` 包中导入。它也可以由 SQLAlchemy 使用称为 *反射* 的过程生成，这意味着您从一个单一对象（例如 `Table`）开始，为其指定一个名称，然后指示 SQLAlchemy 从特定的引擎源加载与该名称相关的所有附加信息。

SQLAlchemy 数据库元数据构造的一个关键特性是它们设计成以 *声明式* 风格使用，这与真实的 DDL 非常相似。因此，对于那些有一定创建真实模式生成脚本背景的人来说，它们是最直观的。

+   使用 MetaData 描述数据库

    +   访问表和列

    +   创建和删除数据库表

    +   通过迁移修改数据库对象

    +   指定模式名称

        +   使用 MetaData 指定默认模式名称

        +   应用动态模式命名约定

        +   为新连接设置默认模式

        +   模式和反射

    +   特定于后端的选项

    +   列、表、MetaData API

        +   `Column`

        +   `MetaData`

        +   `SchemaConst`

        +   `SchemaItem`

        +   `insert_sentinel()`

        +   `Table`

+   反射数据库对象

    +   覆盖反射列

    +   反射视图

    +   一次性反射所有表

    +   从其他模式反射表

        +   模式限定的反射与默认模式的交互

    +   使用检查器进行精细反射

        +   `Inspector`

        +   `ReflectedColumn`

        +   `ReflectedComputed`

        +   `ReflectedCheckConstraint`

        +   `ReflectedForeignKeyConstraint`

        +   `ReflectedIdentity`

        +   `ReflectedIndex`

        +   `ReflectedPrimaryKeyConstraint`

        +   `ReflectedUniqueConstraint`

        +   `ReflectedTableComment`

    +   使用数据库无关类型进行反射

    +   反射的限制

+   列的插入/更新默认值

    +   标量默认值

    +   Python 执行函数

        +   上下文敏感的默认函数

    +   客户端调用的 SQL 表达式

    +   服务器调用的 DDL 显式默认表达式

    +   标记隐式生成的值、时间戳和触发列

    +   定义序列

        +   将序列关联到 SERIAL 列

        +   独立执行序列

        +   将序列与 MetaData 关联

        +   将序列关联为服务器端默认值

    +   计算列（GENERATED ALWAYS AS）

    +   标识列（GENERATED { ALWAYS | BY DEFAULT } AS IDENTITY）

    +   默认对象 API

        +   `Computed`

        +   `ColumnDefault`

        +   `DefaultClause`

        +   `DefaultGenerator`

        +   `FetchedValue`

        +   `Sequence`

        +   `Identity`

+   定义约束和索引

    +   定义外键

        +   通过 ALTER 创建/删除外键约束

        +   ON UPDATE 和 ON DELETE

    +   唯一约束

    +   CHECK 约束

    +   主键约束

    +   在使用声明性 ORM 扩展时设置约束

    +   配置约束命名约定

        +   为 MetaData 集合配置���名约定

        +   默认命名约定

        +   截断长名称

        +   为命名约定创建自定义标记

        +   命名 CHECK 约束

        +   为布尔值、枚举和其他模式类型配置命名

        +   在 ORM 声明性混合中使用命名约定

    +   约束 API

        +   `Constraint`

        +   `ColumnCollectionMixin`

        +   `ColumnCollectionConstraint`

        +   `CheckConstraint`

        +   `ForeignKey`

        +   `ForeignKeyConstraint`

        +   `HasConditionalDDL`

        +   `PrimaryKeyConstraint`

        +   `UniqueConstraint`

        +   `conv()`

    +   索引

        +   函数索引

    +   索引 API

        +   `Index`

+   自定义 DDL

    +   自定义 DDL

    +   控制 DDL 序列

    +   使用内置的 DDLElement 类

    +   控制约束和索引的 DDL 生成

    +   DDL 表达式构造 API

        +   `sort_tables()`

        +   `sort_tables_and_constraints()`

        +   `BaseDDLElement`

        +   `ExecutableDDLElement`

        +   `DDL`

        +   `_CreateDropBase`

        +   `CreateTable`

        +   `DropTable`

        +   `CreateColumn`

        +   `CreateSequence`

        +   `DropSequence`

        +   `CreateIndex`

        +   `DropIndex`

        +   `AddConstraint`

        +   `DropConstraint`

        +   `CreateSchema`

        +   `DropSchema`
