# ORM 映射类配置

> 原文：[`docs.sqlalchemy.org/en/20/orm/mapper_config.html`](https://docs.sqlalchemy.org/en/20/orm/mapper_config.html)

ORM 配置的详细参考，不包括关系，关系详细说明在关系配置。

要快速查看典型的 ORM 配置，请从 ORM 快速入门开始。

要了解 SQLAlchemy 实现的对象关系映射概念，请先查看 SQLAlchemy 统一教程，在使用 ORM 声明形式定义表元数据中介绍。

+   ORM 映射类概述

    +   ORM 映射风格

        +   声明性映射

        +   命令式映射

    +   映射类基本组件

        +   待映射的类

        +   表或其他来自子句对象

        +   属性字典

        +   其他映射器配置参数

    +   映射类行为

        +   默认构造函数

        +   跨加载保持非映射状态

        +   映射类、实例和映射器的运行时内省

            +   映射器对象的检查

            +   映射实例的检查

+   使用声明性映射类

    +   声明性映射风格

        +   使用声明性基类

        +   使用装饰器的声明性映射（无声明性基类）

    +   使用声明性配置表

        +   带有 `mapped_column()` 的声明性表

            +   使用带注释的声明性表（`mapped_column()`的类型注释形式）

            +   访问表和元数据

            +   声明性表配置

            +   使用声明性表的显式模式名称

            +   为声明式映射的列设置加载和持久化选项

            +   显式命名声明式映射列

            +   将额外列添加到现有的声明式映射类

        +   使用命令式表进行声明式（即混合声明式）

            +   映射表列的替代属性名

            +   为命令式表列应用加载、持久化和映射选项

        +   使用反射表进行声明式映射

            +   使用延迟反射

            +   使用自动映射

            +   从反射表自动化列命名方案

            +   映射到显式主键列集合

            +   映射表列的子集

    +   声明式映射器配置

        +   使用声明式定义映射属性

        +   声明式的映射器配置选项

            +   动态构建映射器参数

        +   其他声明式映射指令

            +   `__declare_last__()`

            +   `__declare_first__()`

            +   `metadata`

            +   `__abstract__`

            +   `__table_cls__`

    +   使用混合组合映射层次结构

        +   增强基类

        +   混合使用列

        +   混合使用关系

        +   在 `_orm.column_property()` 和其他 `_orm.MapperProperty` 类中混合使用

        +   使用混合和基类进行映射继承模式

            +   使用 `_orm.declared_attr()` 与继承 `Table` 和 `Mapper` 参数

            +   使用 `_orm.declared_attr()` 生成表特定的继承列

        +   从多个混合类组合表/映射器参数

        +   使用命名约定在混合类上创建索引和约束

+   与 dataclasses 和 attrs 集成

    +   声明式数据类映射

        +   类级别功能配置

        +   属性配置

            +   列默认值

            +   与 Annotated 集成

        +   使用混合类和抽象超类

        +   关系配置

        +   使用未映射的数据类字段

        +   与 Pydantic 等替代数据类提供者集成

    +   将 ORM 映射应用于现有的数据类（传统数据类使用）

        +   使用声明式与命令式表映射映射预先存在的数据类

        +   使用声明式样式字段映射预先存在的数据类

            +   使用预先存在的数据类的声明式混合类

        +   使用命令式映射映射预先存在的数据类

    +   将 ORM 映射应用于现有的 attrs 类

        +   使用声明式“命令式表”映射映射属性

        +   使用命令式映射映射属性

+   SQL 表达式作为映射属性

    +   使用混合类

    +   使用 column_property

        +   将 column_property() 添加到现有的声明式映射类

        +   在映射时从列属性组合

        +   使用 `column_property()` 进行列推迟

    +   使用普通描述符

    +   查询时将 SQL 表达式作为映射属性

+   更改属性行为

    +   简单验证器

        +   `validates()`

    +   在核心级别使用自定义数据类型

    +   使用描述符和混合物

    +   同义词

        +   `synonym()`

    +   操作符定制

+   复合列类型

    +   使用映射的复合列类型

    +   复合体的其他映射形式

        +   直接映射列，然后传递给复合体

        +   直接映射列，将属性名称传递给复合体

        +   命令映射和命令表

    +   使用传统非数据类

    +   跟踪复合体上的原位变化

    +   重新定义复合体的比较操作

    +   嵌套复合体

    +   复合体 API

        +   `composite()`

+   映射类继承层次结构

    +   联接表继承

        +   与联接继承相关的关系

        +   加载联接继承映射

    +   单表继承

        +   使用 `use_existing_column` 解决列冲突

        +   与单表继承相关的关系

        +   使用 `polymorphic_abstract` 构建更深层次的层次结构

        +   加载单表继承映射

    +   具体表继承

        +   具体多态加载配置

        +   抽象具体类

        +   经典和半经典具体多态配置

        +   具体继承关系的关系

        +   加载具体继承映射

+   非传统映射

    +   将类映射到多个表

    +   将类映射到任意子查询

    +   一个类的多个映射器

+   配置版本计数器

    +   简单版本计数

    +   自定义版本计数器/类型

    +   服务器端版本计数器

    +   编程或条件版本计数器

+   类映射 API

    +   `registry`

        +   `registry.__init__()`

        +   `registry.as_declarative_base()`

        +   `registry.configure()`

        +   `registry.dispose()`

        +   `registry.generate_base()`

        +   `registry.map_declaratively()`

        +   `registry.map_imperatively()`

        +   `registry.mapped()`

        +   `registry.mapped_as_dataclass()`

        +   `registry.mappers`

        +   `registry.update_type_annotation_map()`

    +   `add_mapped_attribute()`

    +   `column_property()`

    +   `declarative_base()`

    +   `declarative_mixin()`

    +   `as_declarative()`

    +   `mapped_column()`

    +   `declared_attr`

        +   `declared_attr.cascading`

        +   `declared_attr.directive`

    +   `DeclarativeBase`

        +   `DeclarativeBase.__mapper__`

        +   `DeclarativeBase.__mapper_args__`

        +   `DeclarativeBase.__table__`

        +   `DeclarativeBase.__table_args__`

        +   `DeclarativeBase.__tablename__`

        +   `DeclarativeBase.metadata`

        +   `DeclarativeBase.registry`

    +   `DeclarativeBaseNoMeta`

        +   `DeclarativeBaseNoMeta.__mapper__`

        +   `DeclarativeBaseNoMeta.__mapper_args__`

        +   `DeclarativeBaseNoMeta.__table__`

        +   `DeclarativeBaseNoMeta.__table_args__`

        +   `DeclarativeBaseNoMeta.__tablename__`

        +   `DeclarativeBaseNoMeta.metadata`

        +   `DeclarativeBaseNoMeta.registry`

    +   `has_inherited_table()`

    +   `synonym_for()`

    +   `object_mapper()`

    +   `class_mapper()`

    +   `configure_mappers()`

    +   `clear_mappers()`

    +   `identity_key()`

    +   `polymorphic_union()`

    +   `orm_insert_sentinel()`

    +   `reconstructor()`

    +   `Mapper`

        +   `Mapper.__init__()`

        +   `Mapper.add_properties()`

        +   `Mapper.add_property()`

        +   `Mapper.all_orm_descriptors`

        +   `Mapper.attrs`

        +   `Mapper.base_mapper`

        +   `Mapper.c`

        +   `Mapper.cascade_iterator()`

        +   `Mapper.class_`

        +   `Mapper.class_manager`

        +   `Mapper.column_attrs`

        +   `Mapper.columns`

        +   `Mapper.common_parent()`

        +   `Mapper.composites`

        +   `Mapper.concrete`

        +   `Mapper.configured`

        +   `Mapper.entity`

        +   `Mapper.get_property()`

        +   `Mapper.get_property_by_column()`

        +   `Mapper.identity_key_from_instance()`

        +   `Mapper.identity_key_from_primary_key()`

        +   `Mapper.identity_key_from_row()`

        +   `Mapper.inherits`

        +   `Mapper.is_mapper`

        +   `Mapper.is_sibling()`

        +   `Mapper.isa()`

        +   `Mapper.iterate_properties`

        +   `Mapper.local_table`

        +   `Mapper.mapped_table`

        +   `Mapper.mapper`

        +   `Mapper.non_primary`

        +   `Mapper.persist_selectable`

        +   `Mapper.polymorphic_identity`

        +   `Mapper.polymorphic_iterator()`

        +   `Mapper.polymorphic_map`

        +   `Mapper.polymorphic_on`

        +   `Mapper.primary_key`

        +   `Mapper.primary_key_from_instance()`

        +   `Mapper.primary_mapper()`

        +   `Mapper.relationships`

        +   `Mapper.selectable`

        +   `Mapper.self_and_descendants`

        +   `Mapper.single`

        +   `Mapper.synonyms`

        +   `Mapper.tables`

        +   `Mapper.validators`

        +   `Mapper.with_polymorphic_mappers`

    +   `MappedAsDataclass`

    +   `MappedClassProtocol`
