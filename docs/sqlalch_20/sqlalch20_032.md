# 关系配置

> 原文：[`docs.sqlalchemy.org/en/20/orm/relationships.html`](https://docs.sqlalchemy.org/en/20/orm/relationships.html)

本节描述了`relationship()`函数及其用法的深入讨论。关于关系的介绍，请从使用 ORM 相关对象开始，参阅 SQLAlchemy 统一教程。

+   基本关系模式

    +   声明式 vs. 命令式形式

    +   一对多

        +   使用集合、列表或其他集合类型进行一对多

        +   为一对多配置删除行为

    +   多对一

        +   可空多对一

    +   一对一

        +   为非注释配置设置 uselist=False

    +   多对多

        +   设置双向多对多关系

        +   使用延迟评估形式的“次要”参数

        +   使用集合、列表或其他集合类型进行多对多

        +   从多对多表中删除行

    +   关联对象

        +   将关联对象与多对多访问模式相结合

    +   延迟评估关系参数

        +   在声明后为映射类添加关系

        +   使用多对多的“次要”参数进行延迟评估

+   邻接列表关系

    +   复合邻接列表

    +   自引用查询策略

    +   配置自引用急加载

+   配置关系连接方式

    +   处理多个连接路径

    +   指定备用连接条件

    +   创建自定义外键条件

    +   在连接条件中使用自定义运算符

    +   基于 SQL 函数的自定义运算符

    +   重叠的外键

    +   非关系比较 / 材料化路径

    +   自引用多对多关系

    +   复合“次要”连接

    +   与别名类的关系

        +   将别名类映射与类型化集成并避免早期映射器配置

        +   在查询中使用别名类目标

    +   使用窗口函数进行行限制关系

    +   构建支持查询的属性

    +   关于使用 viewonly 关系参数的注意事项

        +   在 Python 中进行突变，包括具有 viewonly=True 的反向引用不适用

        +   viewonly=True 集合 / 属性直到过期才重新查询

+   处理大型集合

    +   只写关系

        +   创建和持久化新的只写集合

        +   向现有集合添加新项目

        +   查询项目

        +   删除项目

        +   批量插入新项目

        +   项目的批量更新和删除

        +   只写集合 - API 文档

    +   动态关系加载器

        +   动态关系加载器 - API

    +   设置 RaiseLoad

    +   使用被动删除

+   集合自定义和 API 详情

    +   自定义集合访问

        +   字典集合

    +   自定义集合实现

        +   通过装饰器注释自定义集合

        +   自定义基于字典的集合

        +   仪器化和自定义类型

    +   集合 API

        +   `attribute_keyed_dict()`

        +   `column_keyed_dict()`

        +   `keyfunc_mapping()`

        +   `attribute_mapped_collection`

        +   `column_mapped_collection`

        +   `mapped_collection`

        +   `KeyFuncDict`

        +   `MappedCollection`

    +   集合内部

        +   `bulk_replace()`

        +   `collection`

        +   `collection_adapter`

        +   `CollectionAdapter`

        +   `InstrumentedDict`

        +   `InstrumentedList`

        +   `InstrumentedSet`

        +   `prepare_instrumentation()`

+   特殊关系持久化模式

    +   指向自身的行/相互依赖的行

    +   可变主键/更新级联

        +   模拟无外键支持的有限 ON UPDATE CASCADE

+   使用传统的 'backref' 关系参数

    +   Backref 默认参数

    +   指定 Backref 参数

+   关系 API

    +   `relationship()`

    +   `backref()`

    +   `dynamic_loader()`

    +   `foreign()`

    +   `remote()`
