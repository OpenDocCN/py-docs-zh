# 使用声明性的映射类

> 原文：[`docs.sqlalchemy.org/en/20/orm/declarative_mapping.html`](https://docs.sqlalchemy.org/en/20/orm/declarative_mapping.html)

声明性映射风格是 SQLAlchemy 中主要使用的映射风格。请参阅 声明性映射 部分进行顶层介绍。

+   声明性映射风格

    +   使用声明性基类

    +   使用装饰器进行声明性映射（无声明性基类）

+   使用声明性的表配置

    +   具有`mapped_column()` 的声明性表

        +   使用带注释的声明性表（`mapped_column()` 的类型注释形式）

        +   访问表和元数据

        +   声明性表配置

        +   使用声明性表的显式模式名称

        +   为声明性映射的列设置加载和持久化选项

        +   显式命名声明性映射列

        +   向现有的声明性映射类添加附加列

    +   声明性与命令式表（又名混合声明性）

        +   映射表列的备用属性名

        +   为命令式表列应用加载、持久化和映射选项

    +   使用反射表进行声明性映射

        +   使用延迟反射

        +   使用 Automap

        +   从反射表自动化列命名方案

        +   映射到明确一组主键列

        +   映射表列的子集

+   使用声明性的映射器配置

    +   使用声明性定义映射属性

    +   使用声明性配置的 Mapper 配置选项

        +   动态构建映射器参数

    +   其他声明性映射指令

        +   `__declare_last__()`

        +   `__declare_first__()`

        +   `metadata`

        +   `__abstract__`

        +   `__table_cls__`

+   使用 Mixins 构建映射的层次结构

    +   增强基类

    +   混合列

    +   混合关联

    +   混合 `_orm.column_property()` 和其他 `_orm.MapperProperty` 类

    +   使用 Mixins 和基类与映射继承模式

        +   在继承 `Table` 和 `Mapper` 参数中使用 `_orm.declared_attr()`

        +   使用 `_orm.declared_attr()` 生成特定于表的继承列

    +   结合多个 Mixins 的表/映射器参数

    +   使用 Mixins 在 Mixins 上创建索引和约束
