# 类映射 API

> 原文：[`docs.sqlalchemy.org/en/20/orm/mapping_api.html`](https://docs.sqlalchemy.org/en/20/orm/mapping_api.html)

| 对象名称 | 描述 |
| --- | --- |
| add_mapped_attribute(target, key, attr) | 向 ORM 映射类添加新的映射属性。 |
| as_declarative(**kw) | 类装饰器，将给定的类适配为`declarative_base()`。 |
| class_mapper(class_[, configure]) | 给定一个类，返回与该键关联的主要`Mapper`。 |
| clear_mappers() | 从所有类中删除所有映射器。 |
| column_property(column, *additional_columns, [group, deferred, raiseload, comparator_factory, init, repr, default, default_factory, compare, kw_only, active_history, expire_on_flush, info, doc]) | 为映射提供列级别属性。 |
| configure_mappers() | 初始化到目前为止已在所有`registry`集合中构造的所有映射器之间的相互关系。 |
| declarative_base(*, [metadata, mapper, cls, name, class_registry, type_annotation_map, constructor, metaclass]) | 构造用于声明性类定义的基类。 |
| declarative_mixin(cls) | 将类标记为提供“声明混入”功能。 |
| DeclarativeBase | 用于声明性类定义的基类。 |
| DeclarativeBaseNoMeta | 与`DeclarativeBase`相同，但不使用元类拦截新属性。 |
| declared_attr | 将类级别方法标记为表示映射属性或声明式指令定义的方法。 |
| has_inherited_table(cls) | 给定一个类，如果它继承的任何类都有映射表，则返回 True，否则返回 False。 |
| identity_key([class_, ident], *, [instance, row, identity_token]) | 生成“标识键”元组，这些元组用作`Session.identity_map` 字典中的键。 |
| mapped_column([__name_pos, __type_pos], *args, [init, repr, default, default_factory, compare, kw_only, nullable, primary_key, deferred, deferred_group, deferred_raiseload, use_existing_column, name, type_, autoincrement, doc, key, index, unique, info, onupdate, insert_default, server_default, server_onupdate, active_history, quote, system, comment, sort_order], **kw) | 在 声明式表 配置中声明一个新的 ORM 映射的 `Column` 构造。 |
| MappedAsDataclass | 用于指示映射此类时，同时将其转换为数据类的混合类。 |
| MappedClassProtocol | 表示 SQLAlchemy 映射类的协议。 |
| Mapper | 定义 Python 类与数据库表或其他关系结构之间的关联，以便对该类进行的 ORM 操作可以继续进行。 |
| object_mapper(instance) | 给定一个对象，返回与对象实例关联的主要 Mapper。 |
| orm_insert_sentinel([name, type_], *, [default, omit_from_statements]) | 提供一个替代 `mapped_column()` 的代理，生成所谓的 sentinel 列，允许在其他情况下没有符合条件的主键配置的表中进行高效的批量插入，并且具有确定性的 RETURNING 排序。 |
| polymorphic_union(table_map, typecolname[, aliasname, cast_nulls]) | 创建多态映射器使用的 `UNION` 语句。 |
| reconstructor(fn) | 将方法装饰为 ‘reconstructor’ 钩子。 |
| 注册表 | 用于映射类的通用注册表。 |
| synonym_for(name[, map_column]) | 与 Python 描述符一起生成一个 `synonym()` 属性的装饰器。 |

```py
class sqlalchemy.orm.registry
```

用于映射类的通用注册表。

`registry` 用作维护映射集合的基础，并提供用于映射类的配置钩子。

支持的三种常规映射类型是声明基类（Declarative Base）、声明装饰器（Declarative Decorator）和命令式映射（Imperative Mapping）。所有这些映射样式都可以互换使用：

+   `registry.generate_base()` 返回一个新的声明基类，是 `declarative_base()` 函数的底层实现。

+   `registry.mapped()` 提供了一个类装饰器，它将为一个类应用声明性映射，而不使用声明性基类。

+   `registry.map_imperatively()` 会为一个类生成一个 `Mapper`，而不会扫描该类以寻找声明性类属性。这种方法适用于历史上由 `sqlalchemy.orm.mapper()` 传统映射函数提供的用例，该函数已在 SQLAlchemy 2.0 中移除。

从版本 1.4 新增。

**成员**

__init__(), as_declarative_base(), configure(), dispose(), generate_base(), map_declaratively(), map_imperatively(), mapped(), mapped_as_dataclass(), mappers, update_type_annotation_map()

参见

ORM 映射类概述 - 类映射样式概述。

```py
method __init__(*, metadata: Optional[MetaData] = None, class_registry: Optional[clsregistry._ClsRegistryType] = None, type_annotation_map: Optional[_TypeAnnotationMapType] = None, constructor: Callable[..., None] = <function _declarative_constructor>)
```

构建一个新的 `registry`

参数：

+   `metadata` – 一个可选的 `MetaData` 实例。使用声明性表映射生成的所有 `Table` 对象将使用此 `MetaData` 集合。如果将此参数保留在默认值 `None`，则会创建一个空白的 `MetaData` 集合。

+   `constructor` – 指定在没有自己的 `__init__` 的映射类上的 `__init__` 函数的实现。默认情况下，为声明的字段和关系分配 **kwargs 的实现分配给一个实例。如果提供 `None`，则不会提供 __init__，并且构造将回退到 cls.__init__ 的普通 Python 语义。

+   `class_registry` – 可选的字典，当使用字符串名称来标识 `relationship()` 等内部类时，将充当类名称->映射类的注册表。允许两个或更多声明性基类共享相同的类名称注册表，以简化基类之间的关系。

+   `type_annotation_map` –

    可选的 Python 类型到 SQLAlchemy `TypeEngine`类或实例的字典。提供的字典将更新默认类型映射。这仅由`MappedColumn`构造在`Mapped`类型内部的注解产生列类型时使用。

    新版本 2.0 中的内容。

    另请参阅

    自定义类型映射

```py
method as_declarative_base(**kw: Any) → Callable[[Type[_T]], Type[_T]]
```

类装饰器，将为给定的基类调用`registry.generate_base()`。

例如：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

@mapper_registry.as_declarative_base()
class Base:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()
    id = Column(Integer, primary_key=True)

class MyMappedClass(Base):
    # ...
```

所有传递给`registry.as_declarative_base()`的关键字参数都会传递给`registry.generate_base()`。

```py
method configure(cascade: bool = False) → None
```

配置此`注册表`中所有尚未配置的映射器。

配置步骤用于调和和初始化`relationship()`链接，以及调用配置事件，如`MapperEvents.before_configured()`和`MapperEvents.after_configured()`，这些事件可以被 ORM 扩展或用户定义的扩展钩子所使用。

如果此注册表中的一个或多个映射器包含指向其他注册表中映射类的`relationship()`构造，则称该注册表为*依赖*于那些注册表。为了自动配置这些依赖注册表，`configure.cascade`标志应设置为`True`。否则，如果它们未配置，则会引发异常。此行为背后的原理是允许应用程序在控制是否隐式到达其他注册表的同时，以编程方式调用注册表的配置。

作为调用`registry.configure()`的替代方案，可以使用 ORM 函数`configure_mappers()`函数确保内存中所有`registry`对象的配置完成。这通常更简单，并且还早于整体使用`registry`对象的用法。但是，此函数将影响运行中的 Python 进程中的所有映射，并且对于具有许多用于不同目的的注册表的应用程序可能更耗费内存/时间，这些注册表可能不会立即需要。

另请参阅

`configure_mappers()`

自版本 1.4.0b2 新增。

```py
method dispose(cascade: bool = False) → None
```

处理此 `registry` 中的所有映射器。

调用后，此注册表中映射的所有类将不再具有与类相关联的类仪器。该方法是每个`registry`的类似于应用程序范围的`clear_mappers()`函数。

如果此注册表包含其他注册表的依赖项映射器，通常通过`relationship()`链接，则必须将这些注册表也处理掉。当这些注册表存在于与此相关的关系中时，如果设置了`dispose.cascade`标志为`True`，则它们的`registry.dispose()`方法也将被调用；否则，如果这些注册表尚未被处理，则会引发错误。

自版本 1.4.0b2 新增。

另请参阅

`clear_mappers()`

```py
method generate_base(mapper: ~typing.Callable[[...], ~sqlalchemy.orm.mapper.Mapper[~typing.Any]] | None = None, cls: ~typing.Type[~typing.Any] = <class 'object'>, name: str = 'Base', metaclass: ~typing.Type[~typing.Any] = <class 'sqlalchemy.orm.decl_api.DeclarativeMeta'>) → Any
```

生成一个声明性基类。

继承自返回的类对象的类将使用声明性映射自动映射。

例如：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

Base = mapper_registry.generate_base()

class MyClass(Base):
    __tablename__ = "my_table"
    id = Column(Integer, primary_key=True)
```

上述动态生成的类等同于下面的非动态示例：

```py
from sqlalchemy.orm import registry
from sqlalchemy.orm.decl_api import DeclarativeMeta

mapper_registry = registry()

class Base(metaclass=DeclarativeMeta):
    __abstract__ = True
    registry = mapper_registry
    metadata = mapper_registry.metadata

    __init__ = mapper_registry.constructor
```

自版本 2.0 变更：请注意，`registry.generate_base()`方法已被新的`DeclarativeBase`类取代，该类使用子类化生成一个新的“基”类，而不是函数的返回值。这样可以与[**PEP 484**](https://peps.python.org/pep-0484/)类型工具兼容的方法。

`registry.generate_base()`方法提供了`declarative_base()`函数的实现，该函数一次性创建了`registry`和基类。

查看声明式映射部分以获取背景和示例。

参数：

+   `mapper` – 可选可调用对象，默认为`Mapper`。此函数用于生成新的`Mapper`对象。

+   `cls` – 默认为`object`。要用作生成的声明性基类的基础的类型。可以是类或类的元组。

+   `name` – 默认为`Base`。生成类的显示名称。虽然不需要自定义此项，但可以提高回溯和调试时的清晰度。

+   `metaclass` – 默认为`DeclarativeMeta`。作为生成的声明性基类的元类型的元类或`__metaclass__`兼容可调用对象。

另请参阅

声明式映射

`declarative_base()`

```py
method map_declaratively(cls: Type[_O]) → Mapper[_O]
```

声明性地映射一个类。

在这种映射形式中，将扫描类以获取映射信息，包括要与表关联的列和/或实际表对象。

返回`Mapper`对象。

例如：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

class Foo:
    __tablename__ = 'some_table'

    id = Column(Integer, primary_key=True)
    name = Column(String)

mapper = mapper_registry.map_declaratively(Foo)
```

此函数更方便地通过`registry.mapped()`类装饰器或通过从`registry.generate_base()`生成的声明性元类的子类间接调用。

查看完整详情和示例，请参阅声明式映射部分。

参数：

**cls** – 要映射的类。

返回：

一个`Mapper`对象。

另请参阅

声明式映射

`registry.mapped()` - 更常见的此函数的装饰器接口。

`registry.map_imperatively()`

```py
method map_imperatively(class_: Type[_O], local_table: FromClause | None = None, **kw: Any) → Mapper[_O]
```

命令式地映射一个类。

在这种映射形式中，不会扫描类以获取任何映射信息。相反，所有映射构造都作为参数传递。

此方法旨在与现在已删除的 SQLAlchemy `mapper()`函数完全等效，只是以特定注册表的术语表示。

例如：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

my_table = Table(
    "my_table",
    mapper_registry.metadata,
    Column('id', Integer, primary_key=True)
)

class MyClass:
    pass

mapper_registry.map_imperatively(MyClass, my_table)
```

查看完整背景和用法示例，请参阅命令式映射部分。

参数：

+   `class_` – 要映射的类。对应于`Mapper.class_`参数。

+   `local_table` – 映射的主题是`Table`或其他`FromClause`对象。对应于`Mapper.local_table`参数。

+   `**kw` – 所有其他关键字参数直接传递给`Mapper`构造函数。

另请参见

命令式映射

声明式映射

```py
method mapped(cls: Type[_O]) → Type[_O]
```

类装饰器，将声明式映射过程应用于给定的类。

例如：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()

@mapper_registry.mapped
class Foo:
    __tablename__ = 'some_table'

    id = Column(Integer, primary_key=True)
    name = Column(String)
```

参见声明式映射部分，获取完整的细节和示例。

参数：

**cls** – 要映射的类。

返回：

传递的类。

另请参见

声明式映射

`registry.generate_base()` - 生成一个基类，将自动对子类应用声明式映射，使用 Python 元类。

另请参见

`registry.mapped_as_dataclass()`

```py
method mapped_as_dataclass(_registry__cls: Type[_O] | None = None, *, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, eq: _NoArg | bool = _NoArg.NO_ARG, order: _NoArg | bool = _NoArg.NO_ARG, unsafe_hash: _NoArg | bool = _NoArg.NO_ARG, match_args: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, dataclass_callable: _NoArg | Callable[..., Type[Any]] = _NoArg.NO_ARG) → Type[_O] | Callable[[Type[_O]], Type[_O]]
```

类装饰器，将声明式映射过程应用于给定的类，并将类转换为 Python 数据类。

另请参见

声明式数据类映射 - SQLAlchemy 原生数据类映射的完整背景

版本 2.0 中的新功能。

```py
attribute mappers
```

所有`Mapper`对象的只读集合。

```py
method update_type_annotation_map(type_annotation_map: _TypeAnnotationMapType) → None
```

使用新值更新`registry.type_annotation_map`。

```py
function sqlalchemy.orm.add_mapped_attribute(target: Type[_O], key: str, attr: MapperProperty[Any]) → None
```

向 ORM 映射类添加新的映射属性。

例如：

```py
add_mapped_attribute(User, "addresses", relationship(Address))
```

这可用于未使用截获属性设置操作的声明性元类的 ORM 映射。

版本 2.0 中的新功能。

```py
function sqlalchemy.orm.column_property(column: _ORMColumnExprArgument[_T], *additional_columns: _ORMColumnExprArgument[Any], group: str | None = None, deferred: bool = False, raiseload: bool = False, comparator_factory: Type[PropComparator[_T]] | None = None, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, active_history: bool = False, expire_on_flush: bool = True, info: _InfoType | None = None, doc: str | None = None) → MappedSQLExpression[_T]
```

为映射提供列级属性。

使用声明式映射时，`column_property()`用于将只读的 SQL 表达式映射到映射类。

使用命令式映射时，`column_property()`还承担了将表列与附加功能进行映射的角色。使用完全声明式映射时，应使用`mapped_column()`构造来实现此目的。

在声明式数据类映射中，`column_property()` 被认为是**只读**的，并且不会包含在数据类的 `__init__()` 构造函数中。

`column_property()` 函数返回 `ColumnProperty` 的实例。

另请参阅

使用 column_property - 通常使用 `column_property()` 来映射 SQL 表达式。

对命令式表列应用加载、持久化和映射选项 - 使用`column_property()`与命令式表映射，将附加选项应用到普通`Column`对象的用法。

参数：

+   `*cols` – 要映射的列对象列表。

+   `active_history=False` – 仅用于命令式表映射，或遗留式声明式映射（即尚未升级为`mapped_column()`的映射），用于预期可写的基于列的属性；对于声明式映射，请使用 `mapped_column()` 与 `mapped_column.active_history`。有关功能细节，请参阅该参数。

+   `comparator_factory` – 一个继承自`Comparator`的类，提供比较操作的自定义 SQL 子句生成。

+   `group` – 当标记为延迟加载时，此属性的组名称。

+   `deferred` – 当为 True 时，列属性是“延迟加载”的，这意味着它不会立即加载，而是在首次在实例上访问属性时加载。另请参阅 `deferred()`。

+   `doc` – 可选字符串，将应用于类绑定的描述符的文档。

+   `expire_on_flush=True` – 禁用刷新时的过期。引用 SQL 表达式（而不是单个表绑定列）的 column_property() 被视为“只读”属性；填充它对数据状态没有影响，它只能返回数据库状态。因此，每当父对象涉及到刷新时，即在刷新中具有任何类型的“脏”状态时，都会过期 column_property() 的值。将此参数设置为 `False` 将导致在刷新继续进行后保留任何现有值。请注意，默认过期设置的 `Session` 仍在 `Session.commit()` 调用后过期所有属性，但是。

+   `info` – 可选数据字典，将填充到此对象的 `MapperProperty.info` 属性中。

+   `raiseload` –

    如果为 True，则表示在未延迟加载列时应引发错误，而不是加载值。可以通过在查询时使用带有 raiseload=False 的 `deferred()` 选项来更改此行为。

    从版本 1.4 开始新增。

    另请参阅

    使用 raiseload 避免延迟列加载

+   `init` –

    自版本 1.4 起弃用：`column_property.init` 参数对于 `column_property()` 已弃用。此参数仅适用于声明性数据类配置中的可写属性，而 `column_property()` 在此上下文中被视为只读属性。

+   `default` –

    自版本 1.4 起弃用：`column_property.default` 参数对于 `column_property()` 已弃用。此参数仅适用于声明性数据类配置中的可写属性，而 `column_property()` 在此上下文中被视为只读属性。

+   `default_factory` –

    自 1.4 版本起弃用：`column_property.default_factory` 参数已弃用于 `column_property()`。此参数仅适用于声明性数据类配置中的可写属性，而在此上下文中，`column_property()` 被视为只读属性。

+   `kw_only` –

    自 1.4 版本起弃用：`column_property.kw_only` 参数已弃用于 `column_property()`。此参数仅适用于声明性数据类配置中的可写属性，而在此上下文中，`column_property()` 被视为只读属性。

```py
function sqlalchemy.orm.declarative_base(*, metadata: Optional[MetaData] = None, mapper: Optional[Callable[..., Mapper[Any]]] = None, cls: Type[Any] = <class 'object'>, name: str = 'Base', class_registry: Optional[clsregistry._ClsRegistryType] = None, type_annotation_map: Optional[_TypeAnnotationMapType] = None, constructor: Callable[..., None] = <function _declarative_constructor>, metaclass: Type[Any] = <class 'sqlalchemy.orm.decl_api.DeclarativeMeta'>) → Any
```

构建用于声明性类定义的基类。

新的基类将被赋予一个元类，该元类生成适当的 `Table` 对象，并根据在类及其任何子类中声明的信息进行适当的 `Mapper` 调用。

在 2.0 版本中更改：注意 `declarative_base()` 函数已被新的 `DeclarativeBase` 类所取代，该类使用子类化生成一个新的“基”类，而不是一个函数的返回值。这允许与 [**PEP 484**](https://peps.python.org/pep-0484/) 类型工具兼容的方法。

`declarative_base()` 函数是使用 `registry.generate_base()` 方法的简写版本。即：

```py
from sqlalchemy.orm import declarative_base

Base = declarative_base()
```

等同于：

```py
from sqlalchemy.orm import registry

mapper_registry = registry()
Base = mapper_registry.generate_base()
```

查看 `registry` 和 `registry.generate_base()` 的文档字符串以获取更多细节。

在 1.4 版本中更改：`declarative_base()` 函数现在是更通用的 `registry` 类的特化版本。该函数还从 `declarative.ext` 包移动到 `sqlalchemy.orm` 包中。

参数：

+   `metadata` – 可选的`MetaData`实例。所有基类的子类隐式声明的所有`Table`对象将共享此 MetaData。如果未提供，则将创建一个 MetaData 实例。 `MetaData` 实例将通过生成的声明性基类的 `metadata` 属性可用。

+   `mapper` – 可选可调用项，默认为`Mapper`。将用于将子类映射到其表格。

+   `cls` – 默认为`object`。要用作生成的声明性基类的基类的类型。可以是一个类或类的元组。

+   `name` – 默认为`Base`。生成类的显示名称。不需要自定义此选项，但可以提高回溯和调试时的清晰度。

+   `constructor` – 指定在没有自己的`__init__`的映射类上实现`__init__`函数的实现。默认为一种实现，将声明的字段和关系的 **kwargs 分配给一个实例。如果提供了`None`，则不会提供`__init__`，并且构造将通过普通的 Python 语义回退到 cls.`__init__`。

+   `class_registry` – 可选字典，将用作当使用字符串名称标识`relationship()`等内部的类时，类名->映射类的注册表。允许两个或更多声明性基类共享相同的类名注册表，以简化基类之间的关系。

+   `type_annotation_map` –

    Python 类型到 SQLAlchemy `TypeEngine` 类或实例的可选字典。这仅由`MappedColumn`构造用于基于`Mapped`类型中的注释生成列类型。

    版本 2.0 中的新功能。

    另请参见

    自定义类型映射

+   `metaclass` – 默认为`DeclarativeMeta`。要用作生成的声明性基类的元类型的元类或`__metaclass__`兼容可调用项。

另请参见

`registry`

```py
function sqlalchemy.orm.declarative_mixin(cls: Type[_T]) → Type[_T]
```

将类标记为提供“声明性混合”的功能。

例如：

```py
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import declarative_mixin

@declarative_mixin
class MyMixin:

    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()

    __table_args__ = {'mysql_engine': 'InnoDB'}
    __mapper_args__= {'always_refresh': True}

    id =  Column(Integer, primary_key=True)

class MyModel(MyMixin, Base):
    name = Column(String(1000))
```

`declarative_mixin()` 装饰器当前不会以任何方式修改给定的类；其当前目的严格来说是帮助 Mypy 插件能够在没有其他上下文的情况下识别 SQLAlchemy 声明性混合类。

版本 1.4.6 中的新功能。

另请参阅

使用 Mixins 组合映射层次结构

使用 @declared_attr 和声明性 Mixins - 在 Mypy 插件文档中

```py
function sqlalchemy.orm.as_declarative(**kw: Any) → Callable[[Type[_T]], Type[_T]]
```

类装饰器，将给定的类适应为`declarative_base()`。

此函数利用了`registry.as_declarative_base()`方法，首先自动创建一个`registry`，然后调用装饰器。

例如：

```py
from sqlalchemy.orm import as_declarative

@as_declarative()
class Base:
    @declared_attr
    def __tablename__(cls):
        return cls.__name__.lower()
    id = Column(Integer, primary_key=True)

class MyMappedClass(Base):
    # ...
```

另请参阅

`registry.as_declarative_base()`

```py
function sqlalchemy.orm.mapped_column(__name_pos: str | _TypeEngineArgument[Any] | SchemaEventTarget | None = None, __type_pos: _TypeEngineArgument[Any] | SchemaEventTarget | None = None, *args: SchemaEventTarget, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: Any | None = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, nullable: bool | Literal[SchemaConst.NULL_UNSPECIFIED] | None = SchemaConst.NULL_UNSPECIFIED, primary_key: bool | None = False, deferred: _NoArg | bool = _NoArg.NO_ARG, deferred_group: str | None = None, deferred_raiseload: bool | None = None, use_existing_column: bool = False, name: str | None = None, type_: _TypeEngineArgument[Any] | None = None, autoincrement: _AutoIncrementType = 'auto', doc: str | None = None, key: str | None = None, index: bool | None = None, unique: bool | None = None, info: _InfoType | None = None, onupdate: Any | None = None, insert_default: Any | None = _NoArg.NO_ARG, server_default: _ServerDefaultArgument | None = None, server_onupdate: FetchedValue | None = None, active_history: bool = False, quote: bool | None = None, system: bool = False, comment: str | None = None, sort_order: _NoArg | int = _NoArg.NO_ARG, **kw: Any) → MappedColumn[Any]
```

为在声明性表配置中使用的新的 ORM 映射的`Column`构造声明。

`mapped_column()`函数提供了一个与 ORM 兼容且与 Python 类型兼容的构造，用于声明性映射，指示映射到 Core `Column` 对象的属性。当使用声明性时，特别是在使用声明性表配置时，它提供了将属性映射到`Column`对象的等效功能。

2.0 版中的新功能。

`mapped_column()`通常与显式类型一起使用，以及`Mapped`注释类型一起使用，它可以根据`Mapped`注释中的内容推导出列的 SQL 类型和可空性。它也可以在不带注释的情况下使用，作为 SQLAlchemy 1.x 风格中声明性映射中使用`Column`的替代品。

对于`mapped_column()`的使用示例，请参阅使用 mapped_column() 的声明性表中的文档。

另请参阅

使用 mapped_column() 的声明性表 - 完整文档

ORM 声明性模型 - 使用 1.x 风格映射的声明性映射的迁移说明

参数：

+   `__name` – 要为 `Column` 指定的字符串名称。这是一个可选的仅位置参数，如果存在，必须是传递的第一个位置参数。如果省略，则将使用 `mapped_column()` 映射到的属性名称作为 SQL 列名。

+   `__type` – `TypeEngine` 类型或实例，指示与 `Column` 关联的数据类型。这是一个可选的仅位置参数，如果存在，则必须紧随 `__name` 参数，否则必须是第一个位置参数。如果省略，则列的最终类型可以从注释类型中推导出，或者如果存在 `ForeignKey`，则可以从引用列的数据类型中推导出。

+   `*args` – 额外的位置参数包括诸如 `ForeignKey`、`CheckConstraint` 和 `Identity` 这样的结构，它们被传递到构造的 `Column` 中。

+   `nullable` – 可选布尔值，指示列是否应为“NULL”或“NOT NULL”。如果省略，nullability 将根据类型注释推导而来，根据 `typing.Optional` 是否存在而定。否则，对于非主键列，`nullable` 默认为 `True`，对于主键列，默认为 `False`。

+   `primary_key` – 可选布尔值，表示 `Column` 是否将成为表的主键。

+   `deferred` –

    可选布尔值 - 此关键字参数由 ORM 声明过程使用，并不是 `Column` 本身的一部分；相反，它表示此列应当被“延迟”加载，就好像被 `deferred()` 映射一样。

    另请参阅

    配置映射中的列延迟

+   `deferred_group` –

    暗示将 `mapped_column.deferred` 设置为 `True`，并设置 `deferred.group` 参数。

    另请参阅

    以组加载延迟列

+   `deferred_raiseload` –

    意味着将 `mapped_column.deferred` 设置为 `True`，并设置 `deferred.raiseload` 参数。

    另请参阅

    使用 raiseload 避免延迟加载列

+   `use_existing_column` –

    如果为 True，则尝试在继承的超类（通常是单一继承的超类）上定位给定列名，如果存在，则不会生成新列，将映射到超类列，就好像该列从此类中省略一样。这用于将新列添加到继承的超类的混合类。

    另请参阅

    使用 use_existing_column 解决列冲突

    从 2.0.0b4 版开始新增。

+   `default` –

    如果 `mapped_column.insert_default` 参数不存在，则直接传递给 `Column.default` 参数。此外，在使用声明式数据类映射时，表示应该应用于生成的 `__init__()` 方法内的关键字构造函数的默认 Python 值。

    请注意，在生成数据类时，当 `mapped_column.insert_default` 不存在时，这意味着 `mapped_column.default` 的值将在 **两个** 地方使用，即 `__init__()` 方法和 `Column.default` 参数。虽然此行为可能在将来的版本中更改，但目前这种情况通常“可以解决”；`None` 的默认值意味着 `Column` 不会得到默认生成器，而引用非`None`的默认值将在调用`__init__()`时提前分配给对象，在任何情况下，核心 `Insert` 构造将使用相同的值，从而导致相同的最终结果。

    注意

    当使用在 Core 级别的列默认值作为可调用对象，由底层`Column`与 ORM 映射的数据类，特别是那些是上下文感知的默认函数时，必须使用**`mapped_column.insert_default`参数**。这是必要的，以消除可调用对象被解释为数据类级别默认值的歧义。

+   `insert_default` – 直接传递给`Column.default`参数；当存在时，将取代`mapped_column.default`的值，但无论何时，`mapped_column.default`都将应用于数据类映射的构造函数默认值。

+   `sort_order` –

    表示当 ORM 创建`Table`时，此映射列应如何与其他列排序的整数。对于具有相同值的映射列，默认使用默认排序，首先放置在主类中定义的映射列，然后放置在超类中的映射列。默认为 0。排序为升序。

    版本 2.0.4 中的新内容。

+   `active_history=False` –

    当`True`时，表示应在替换时加载标量属性的“上一个”值，如果尚未加载。通常，简单非主键标量值的历史跟踪逻辑只需要知道“新”值就能执行刷新。此标志适用于需要使用`get_history()`或`Session.is_modified()`并且还需要知道属性的“上一个”值的应用程序。

    版本 2.0.10 中的新内容。

+   `init` – 特定于声明性数据类映射，指定映射属性是否应作为数据类过程生成的`__init__()`方法的一部分。

+   `repr` – 特定于声明性数据类映射，指定映射属性是否应作为数据类过程生成的`__repr__()`方法的一部分。

+   `default_factory` – 特定于声明性数据类映射，指定作为数据类过程生成的`__init__()`方法的一部分的默认值生成函数。

+   `compare` –

    特定于声明式数据类映射，指示在为映射类生成`__eq__()`和`__ne__()`方法时，是否应包含此字段在比较操作中。

    在版本 2.0.0b4 中新增。

+   `kw_only` – 特定于声明式数据类映射，指示在生成`__init__()`时，是否应将此字段标记为仅关键字。

+   `**kw` – 所有剩余的关键字参数都传递给`Column`的构造函数。

```py
class sqlalchemy.orm.declared_attr
```

将类级方法标记为表示映射属性或声明性指令定义。

`declared_attr`通常作为类级方法的装饰器应用，将属性转换为类似标量的属性，可以从未实例化的类中调用。声明性映射过程在扫描类时寻找这些`declared_attr`可调用对象，并假定任何标记为`declared_attr`的属性将是一个可调用对象，将生成特定于声明性映射或表配置的对象。

`declared_attr`通常适用于混入类，用于定义应用于类的不同实现者的关系。它还可以用于定义动态生成的列表达式和其他声明性属性。

示例：

```py
class ProvidesUserMixin:
    "A mixin that adds a 'user' relationship to classes."

    user_id: Mapped[int] = mapped_column(ForeignKey("user_table.id"))

    @declared_attr
    def user(cls) -> Mapped["User"]:
        return relationship("User")
```

当与`__tablename__`等声明性指令一起使用时，可以使用`declared_attr.directive()`修饰符，指示[**PEP 484**](https://peps.python.org/pep-0484/)类型工具，给定的方法不涉及`Mapped`属性：

```py
class CreateTableName:
    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()
```

`declared_attr`也可以直接应用于映射类，以允许在使用映射继承方案时，属性可以在子类上动态配置自身。下面说明了使用`declared_attr`创建为子类生成`Mapper.polymorphic_identity`参数的动态方案：

```py
class Employee(Base):
    __tablename__ = 'employee'

    id: Mapped[int] = mapped_column(primary_key=True)
    type: Mapped[str] = mapped_column(String(50))

    @declared_attr.directive
    def __mapper_args__(cls) -> Dict[str, Any]:
        if cls.__name__ == 'Employee':
            return {
                    "polymorphic_on":cls.type,
                    "polymorphic_identity":"Employee"
            }
        else:
            return {"polymorphic_identity":cls.__name__}

class Engineer(Employee):
    pass
```

`declared_attr`支持装饰使用`@classmethod`显式装饰的函数。从运行时的角度来看，这从未必要，但可能需要支持不认识已装饰函数具有类级行为的`cls`参数的[**PEP 484**](https://peps.python.org/pep-0484/)类型工具：

```py
class SomethingMixin:
    x: Mapped[int]
    y: Mapped[int]

    @declared_attr
    @classmethod
    def x_plus_y(cls) -> Mapped[int]:
        return column_property(cls.x + cls.y)
```

版本 2.0 中的新功能：- `declared_attr`可以容纳使用`@classmethod`装饰的函数，以帮助需要的[**PEP 484**](https://peps.python.org/pep-0484/)集成。

另见

通过混合组合映射层次结构 - 附带对`declared_attr`使用模式的背景说明的声明性混合文档。

**成员**

级联，指令

**类签名**

类`sqlalchemy.orm.declared_attr` (`sqlalchemy.orm.base._MappedAttribute`, `sqlalchemy.orm.decl_api._declared_attr_common`)

```py
attribute cascading
```

将`declared_attr`标记为级联。

这是一个特殊用途的修饰符，表明在映射继承场景中，列或基于 MapperProperty 的声明属性应该在映射的子类中独立配置。

警告

`declared_attr.cascading`修饰符有几个限制：

+   标志`only`适用于在声明性混合类和`__abstract__`类上使用`declared_attr`；当直接在映射类上使用时，它目前没有任何效果。

+   标志`only`仅适用于通常命名的属性，例如不是任何特殊下划线属性，例如`__tablename__`。在这些属性上它`没有`效果。

+   当前标志`不允许进一步覆盖`类层次结构下游；如果子类尝试覆盖属性，则会发出警告并跳过覆盖的属性。这是一个希望在某些时候解决的限制。

下面，无论是`MyClass`还是`MySubClass`都将建立一个独特的`id`列对象：

```py
class HasIdMixin:
    @declared_attr.cascading
    def id(cls):
        if has_inherited_table(cls):
            return Column(ForeignKey("myclass.id"), primary_key=True)
        else:
            return Column(Integer, primary_key=True)

class MyClass(HasIdMixin, Base):
    __tablename__ = "myclass"
    # ...

class MySubClass(MyClass):
  """ """

    # ...
```

上述配置的行为是，`MySubClass`将引用其自己的`id`列以及`MyClass`下面命名为`some_id`的属性。

另见

声明性继承

使用 _orm.declared_attr() 生成特定表继承列

```py
attribute directive
```

将 `declared_attr` 标记为装饰声明性指令，如 `__tablename__` 或 `__mapper_args__`。

`declared_attr.directive` 的目的严格是支持[**PEP 484**](https://peps.python.org/pep-0484/)类型工具，允许装饰的函数具有不使用 `Mapped` 通用类的返回类型，这在使用 `declared_attr` 用于列和映射属性时通常是不会发生的。在运行时，`declared_attr.directive` 返回未经修改的 `declared_attr` 类。

例如：

```py
class CreateTableName:
    @declared_attr.directive
    def __tablename__(cls) -> str:
        return cls.__name__.lower()
```

2.0 版本中的新功能。

另请参见

使用 Mixins 组合映射层次结构

`declared_attr`

```py
class sqlalchemy.orm.DeclarativeBase
```

用于声明性类定义的基类。

`DeclarativeBase` 允许以与类型检查器兼容的方式创建新的声明性基类：

```py
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass
```

上述 `Base` 类现在可用作新声明性映射的基类。超类利用 `__init_subclass__()` 方法设置新类，而不使用元类。

首次使用时，`DeclarativeBase` 类实例化一个新的 `registry` 用于与基类一起使用，假设未明确提供。`DeclarativeBase` 类支持类级属性，这些属性充当此注册表构建的参数；例如指示特定的 `MetaData` 集合以及 `registry.type_annotation_map` 的特定值：

```py
from typing_extensions import Annotated

from sqlalchemy import BigInteger
from sqlalchemy import MetaData
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase

bigint = Annotated[int, "bigint"]
my_metadata = MetaData()

class Base(DeclarativeBase):
    metadata = my_metadata
    type_annotation_map = {
        str: String().with_variant(String(255), "mysql", "mariadb"),
        bigint: BigInteger()
    }
```

可指定的类级属性包括：

参数：

+   `metadata` – 可选的`MetaData` 集合。如果自动构造了一个`registry`，则将使用该`MetaData` 集合来构造它。否则，本地的`MetaData` 集合将取代通过`DeclarativeBase.registry` 参数传递的现有`registry` 使用的集合。

+   `type_annotation_map` – 可选的类型注释映射，将传递给`registry` 作为`registry.type_annotation_map`.

+   `registry` – 直接提供预先存在的`registry`。

2.0 版本中的新功能：添加了`DeclarativeBase`，以便可以以也被[**PEP 484**](https://peps.python.org/pep-0484/)类型检查器识别的方式构造声明性基类。因此，`DeclarativeBase` 和其他基于子类化的 API 应被视为取代先前的“由函数返回的类” API，即`declarative_base()` 和`registry.generate_base()`，其中返回的基类不能被类型检查器识别，除非使用插件。

**__init__ 行为**

在普通的 Python 类中，类层次结构中基本的 `__init__()` 方法是 `object.__init__()`，不接受任何参数。然而，当首次声明`DeclarativeBase`子类时，如果没有已经存在的 `__init__()` 方法，该类将被赋予一个 `__init__()` 方法，该方法链接到`registry.constructor` 构造函数；这是通常的声明性构造函数，将关键字参数分配给实例的属性，假定这些属性在类级别已经建立（即已映射，或者与描述符链接）。这个构造函数**永远不会被映射类直接访问，除非通过显式调用 super()**，因为映射类本身会直接得到一个 `__init__()` 方法，该方法调用`registry.constructor`，所以在默认情况下独立于基本的 `__init__()` 方法的操作。

从版本 2.0.1 开始发生了变化：`DeclarativeBase` 现在具有默认构造函数，默认链接到 `registry.constructor`，以便调用 `super().__init__()` 可以访问此构造函数。先前，由于一个实现错误，这个默认构造函数丢失了，调用 `super().__init__()` 将会调用 `object.__init__()`。

`DeclarativeBase` 子类也可以声明一个显式的 `__init__()` 方法，该方法将在此级别替代 `registry.constructor` 函数的使用：

```py
class Base(DeclarativeBase):
    def __init__(self, id=None):
        self.id = id
```

映射的类仍然不会隐式调用这个构造函数；只能通过调用 `super().__init__()` 来访问它：

```py
class MyClass(Base):
    def __init__(self, id=None, name=None):
        self.name = name
        super().__init__(id=id)
```

请注意，这与诸如传统的 `declarative_base()` 等函数的行为不同；由这些函数创建的基类将始终为 `__init__()` 安装 `registry.constructor`。

**成员**

__mapper__, __mapper_args__, __table__, __table_args__, __tablename__, metadata, registry

**类签名**

类 `sqlalchemy.orm.DeclarativeBase` (`sqlalchemy.inspection.Inspectable`)

```py
attribute __mapper__: ClassVar[Mapper[Any]]
```

将特定类映射到的 `Mapper` 对象。

也可以使用 `inspect()` 获取，例如 `inspect(klass)`。

```py
attribute __mapper_args__: Any
```

传递给 `Mapper` 构造函数的参数字典。

另见

使用声明式的 Mapper 配置选项

```py
attribute __table__: ClassVar[FromClause]
```

将特定子类映射到的 `FromClause`。

这通常是 `Table` 的一个实例，但根据类的映射方式，也可能是其他类型的 `FromClause`，比如 `Subquery`。

另见

访问表和元数据

```py
attribute __table_args__: Any
```

将传递给`Table`构造函数的参数字典或元组。有关此集合特定结构的背景，请参阅声明式表配置。

另请参阅

声明式表配置

```py
attribute __tablename__: Any
```

将生成的`Table`对象分配的字符串名称，如果没有通过`DeclarativeBase.__table__`直接指定。

另请参阅

使用 mapped_column()的声明式表

```py
attribute metadata: ClassVar[MetaData]
```

指的是将用于新`Table`对象的`MetaData`集合。

另请参阅

访问表和元数据

```py
attribute registry: ClassVar[registry]
```

指的是新`Mapper`对象将关联的正在使用的`registry`。

```py
class sqlalchemy.orm.DeclarativeBaseNoMeta
```

与`DeclarativeBase`相同，但不使用元类拦截新属性。

当希望使用自定义元类时，可以使用`DeclarativeBaseNoMeta`基类。

2.0 版本中的新功能。

**成员**

__mapper__, __mapper_args__, __table__, __table_args__, __tablename__, metadata, registry

**类签名**

类`sqlalchemy.orm.DeclarativeBaseNoMeta` (`sqlalchemy.inspection.Inspectable`)

```py
attribute __mapper__: ClassVar[Mapper[Any]]
```

将特定类映射到的`Mapper`对象。

也可以使用`inspect()`获得，例如`inspect(klass)`。

```py
attribute __mapper_args__: Any
```

将传递给`Mapper`构造函数的参数字典。

另请参阅

使用声明式的 Mapper 配置选项

```py
attribute __table__: FromClause | None
```

将特定子类映射到的`FromClause`。

这通常是`Table`的实例，但根据类的映射方式，也可能引用其他类型的`FromClause`，例如`Subquery`。

另请参阅

访问表和元数据

```py
attribute __table_args__: Any
```

将传递给`Table`构造函数的参数字典或元组。有关此集合特定结构的背景，请参阅声明式表配置。

另请参阅

声明式表配置

```py
attribute __tablename__: Any
```

分配给生成的`Table`对象的字符串名称，如果未直接通过`DeclarativeBase.__table__`指定。

另请参阅

具有 mapped_column()的声明式表

```py
attribute metadata: ClassVar[MetaData]
```

指的是将用于新`Table`对象的`MetaData`集合。

另请参阅

访问表和元数据

```py
attribute registry: ClassVar[registry]
```

指的是新的`Mapper`对象将与之关联的正在使用的`registry`。

```py
function sqlalchemy.orm.has_inherited_table(cls: Type[_O]) → bool
```

给定一个类，如果它继承的任何类都有一个映射表，则返回 True，否则返回 False。

这在声明式混合中用于构建在继承层次结构中的基类和子类之间行为不同的属性。

另请参阅

使用混合和基类进行映射继承模式

```py
function sqlalchemy.orm.synonym_for(name: str, map_column: bool = False) → Callable[[Callable[[...], Any]], Synonym[Any]]
```

在与 Python 描述符一起生成`synonym()`属性的装饰器。

被装饰的函数将被传递给`synonym()`作为`synonym.descriptor`参数：

```py
class MyClass(Base):
    __tablename__ = 'my_table'

    id = Column(Integer, primary_key=True)
    _job_status = Column("job_status", String(50))

    @synonym_for("job_status")
    @property
    def job_status(self):
        return "Status: %s" % self._job_status
```

SQLAlchemy 的混合属性功能通常比同义词更受青睐，后者是一个更传统的功能。

另请参阅

同义词 - 同义词概述

`synonym()` - 映射器级函数

使用描述符和混合属性 - Hybrid Attribute 扩展提供了一种更新的方法来更灵活地增强属性行为，比使用同义词更容易实现。

```py
function sqlalchemy.orm.object_mapper(instance: _T) → Mapper[_T]
```

给定一个对象，返回与该对象实例关联的主要映射器。

如果没有配置映射，则引发`sqlalchemy.orm.exc.UnmappedInstanceError`。

此功能可通过检查系统使用：

```py
inspect(instance).mapper
```

如果实例不是映射的一部分，则使用检查系统将引发`sqlalchemy.exc.NoInspectionAvailable`。

```py
function sqlalchemy.orm.class_mapper(class_: Type[_O], configure: bool = True) → Mapper[_O]
```

给定一个类，返回与该键关联的主要`Mapper`。

如果给定类上没有配置映射，则引发`UnmappedClassError`，或者如果传递了非类对象，则引发`ArgumentError`。

相当的功能可以通过`inspect()`函数实现：

```py
inspect(some_mapped_class)
```

如果类未映射，则使用检查系统将引发`sqlalchemy.exc.NoInspectionAvailable`。

```py
function sqlalchemy.orm.configure_mappers() → None
```

初始化到目前为止在所有`registry`集合中已构建的所有映射器的互映关系。

配置步骤用于协调和初始化映射类之间的`relationship()`链接，以及调用配置事件，如`MapperEvents.before_configured()`和`MapperEvents.after_configured()`，这些事件可能被 ORM 扩展或用户定义的扩展钩子使用。

映射器配置通常是自动调用的，第一次使用特定 `registry` 的映射时，以及每当使用映射并且已经构造了额外的尚未配置的映射器时。然而，自动配置过程仅局限于涉及目标映射器和任何相关的 `registry` 对象的 `registry`；这相当于在特定 `registry` 上调用 `registry.configure()` 方法。

与之相比，`configure_mappers()` 函数将在内存中存在的所有 `registry` 对象上调用配置过程，并且可能对使用许多个体 `registry` 对象但彼此相关的场景有用。

从版本 1.4 开始更改：从 SQLAlchemy 1.4.0b2 开始，此函数按照每个 `registry` 的方式工作，定位所有存在的 `registry` 对象并调用每个对象上的 `registry.configure()` 方法。可能更喜欢使用 `registry.configure()` 方法来限制映射器的配置仅限于特定 `registry` 和/或声明性基类。

自动配置被调用的点包括当映射类被实例化为实例时，以及当使用 `Session.query()` 或 `Session.execute()` 发出 ORM 查询时使用 ORM 启用的语句。

映射器配置过程，无论是由 `configure_mappers()` 还是 `registry.configure()` 调用，都提供了几个可用于增强映射器配置步骤的事件挂钩。这些挂钩包括：

+   `MapperEvents.before_configured()` - 在 `configure_mappers()` 或 `registry.configure()` 执行任何工作之前调用一次；这可用于在操作继续之前建立其他选项、属性或相关映射。

+   `MapperEvents.mapper_configured()` - 在进程中配置每个单独的 `Mapper` 时调用；将包括除其他映射器设置的反向引用之外的所有映射器状态，这些映射器尚未配置。

+   `MapperEvents.after_configured()` - 在 `configure_mappers()` 或 `registry.configure()` 完成后调用一次；在此阶段，所有配置操作范围内的 `Mapper` 对象将被完全配置。请注意，调用应用程序可能仍然有其他尚未生成的映射，例如，如果它们在尚未导入的模块中，还可能有映射尚未配置，如果它们位于当前配置范围之外的其他`registry`集合中。

```py
function sqlalchemy.orm.clear_mappers() → None
```

删除所有类的所有映射器。

从版本 1.4 开始变更：这个函数现在定位所有的`registry`对象，并调用每个对象的 `registry.dispose()` 方法。

这个函数从类中删除所有的仪器，并处置它们的关联映射器。一旦调用，这些类将被取消映射，以后可以用新的映射器重新映射。

`clear_mappers()` *不*是正常使用，因为在非常特定的测试场景之外，它实际上没有任何有效用途。通常，映射器是用户定义类的永久结构组件，绝不会独立于其类被丢弃。如果映射类本身被垃圾回收，其映射器也将被自动处理。因此，`clear_mappers()` 仅用于在测试套件中重复使用相同类的不同映射的情况下，这本身是一个极为罕见的用例 - 唯一的这种用例实际上是 SQLAlchemy 自己的测试套件，可能是其他 ORM 扩展库的测试套件，这些库打算在一组固定的类上测试各种映射构造的组合。

```py
function sqlalchemy.orm.util.identity_key(class_: Type[_T] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: _T | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[_T]
```

生成“标识键”元组，用作 `Session.identity_map` 字典中的键。

此函数有几种调用样式：

+   `identity_key(class, ident, identity_token=token)`

    此形式接收一个映射类和一个主键标量或元组作为参数。

    例如：

    ```py
    >>> identity_key(MyClass, (1, 2))
    (<class '__main__.MyClass'>, (1, 2), None)
    ```

    参数类：

    映射类（必须是一个位置参数）

    参数 ident：

    主键，可以是标量或元组参数。

    参数 identity_token：

    可选的标识令牌

    版本 1.2 中的新功能：添加了 identity_token

+   `identity_key(instance=instance)`

    此形式将为给定实例生成标识键。实例不必是持久的，只需其主键属性被填充（否则键将包含这些缺失值的 `None`）。

    例如：

    ```py
    >>> instance = MyClass(1, 2)
    >>> identity_key(instance=instance)
    (<class '__main__.MyClass'>, (1, 2), None)
    ```

    在此形式中，给定实例最终将通过 `Mapper.identity_key_from_instance()` 运行，如果对象已过期，则将执行相应行的数据库检查。

    参数实例：

    对象实例（必须作为关键字参数给出）

+   `identity_key(class, row=row, identity_token=token)`

    此形式类似于类/元组形式，但是传递了数据库结果行作为 `Row` 或 `RowMapping` 对象。

    例如：

    ```py
    >>> row = engine.execute(\
     text("select * from table where a=1 and b=2")\
     ).first()
    >>> identity_key(MyClass, row=row)
    (<class '__main__.MyClass'>, (1, 2), None)
    ```

    参数类：

    映射类（必须是一个位置参数）

    参数行：

    `Row` 由 `CursorResult` 返回的行（必须作为关键字参数给出）

    参数 identity_token：

    可选的标识令牌

    版本 1.2 中的新功能：添加了 identity_token

```py
function sqlalchemy.orm.polymorphic_union(table_map, typecolname, aliasname='p_union', cast_nulls=True)
```

创建一个多态映射器使用的 `UNION` 语句。

请参见具体表继承以了解如何使用此功能。

参数：

+   `table_map` – 将多态标识映射到 `Table` 对象。

+   `typecolname` – “鉴别器”列的字符串名称，该列将从查询中派生，为每一行产生多态标识。如果为 `None`，则不生成多态鉴别器。

+   `aliasname` – 生成的 `alias()` 构造的名称。

+   `cast_nulls` – 如果为 True，则不存在的列，表示为标记的 NULL 值，将被传递到 CAST 中。这是一种问题的传统行为，对于某些后端（如 Oracle）存在问题 - 在这种情况下，可以将其设置为 False。

```py
function sqlalchemy.orm.orm_insert_sentinel(name: str | None = None, type_: _TypeEngineArgument[Any] | None = None, *, default: Any | None = None, omit_from_statements: bool = True) → MappedColumn[Any]
```

提供了一个虚拟的 `mapped_column()`，它生成所谓的 sentinel 列，允许对于不具有合格的主键配置的表进行具有确定性的 RETURNING 排序的高效批量插入。

使用`orm_insert_sentinel()`类似于在 Core `Table` 构造中使用`insert_sentinel()` 构造的用法。

将此构造添加到声明式映射类的指南与`insert_sentinel()` 构造的相同；数据库表本身也需要具有此名称的列。

关于此对象的使用背景，请参阅 配置 Sentinel 列 作为 “INSERT 语句的“插入多个值”行为 部分的一部分。

另请参见

`insert_sentinel()`

“INSERT 语句的“插入多个值”行为

配置 Sentinel 列

2.0.10 版本中的新增内容。

```py
function sqlalchemy.orm.reconstructor(fn)
```

将一个方法装饰为‘reconstructor’挂钩。

将单个方法指定为“reconstructor”，一个类似于`__init__`方法的方法，ORM 在实例从数据库中加载或者以其他方式重新构建后会调用该方法。

提示

`reconstructor()` 装饰器使用了 `InstanceEvents.load()` 事件挂钩，该事件可以直接使用。

重构器将在没有参数的情况下被调用。实例的标量（非集合）数据库映射属性将在函数内可用。急切加载的集合通常尚不可用，并且通常只包含第一个元素。在这个阶段对对象进行的 ORM 状态更改不会被记录到下一个 flush()操作中，因此重构器内的活动应该保守。

另请参阅

`InstanceEvents.load()`

```py
class sqlalchemy.orm.Mapper
```

定义了 Python 类与数据库表或其他关系结构之间的关联，以便对该类进行 ORM 操作。

`Mapper`对象是使用`registry`对象上存在的映射方法实例化的。有关实例化新`Mapper`对象的信息，请参阅 ORM 映射类概述。

**成员**

__init__(), add_properties(), add_property(), all_orm_descriptors, attrs, base_mapper, c, cascade_iterator(), class_, class_manager, column_attrs, columns, common_parent(), composites, concrete, configured, entity, get_property(), get_property_by_column(), identity_key_from_instance(), identity_key_from_primary_key(), identity_key_from_row(), inherits, is_mapper, is_sibling(), isa(), iterate_properties, local_table, mapped_table, mapper, non_primary, persist_selectable, polymorphic_identity, polymorphic_iterator(), polymorphic_map, polymorphic_on, primary_key, primary_key_from_instance(), primary_mapper(), relationships, selectable, self_and_descendants, single, synonyms, tables, validators, with_polymorphic_mappers

**类签名**

类 `sqlalchemy.orm.Mapper` (`sqlalchemy.orm.ORMFromClauseRole`，`sqlalchemy.orm.ORMEntityColumnsClauseRole`，`sqlalchemy.sql.cache_key.MemoizedHasCacheKey`，`sqlalchemy.orm.base.InspectionAttr`，`sqlalchemy.log.Identified`，`sqlalchemy.inspection.Inspectable`，`sqlalchemy.event.registry.EventTarget`，`typing.Generic`)

```py
method __init__(class_: Type[_O], local_table: FromClause | None = None, properties: Mapping[str, MapperProperty[Any]] | None = None, primary_key: Iterable[_ORMColumnExprArgument[Any]] | None = None, non_primary: bool = False, inherits: Mapper[Any] | Type[Any] | None = None, inherit_condition: _ColumnExpressionArgument[bool] | None = None, inherit_foreign_keys: Sequence[_ORMColumnExprArgument[Any]] | None = None, always_refresh: bool = False, version_id_col: _ORMColumnExprArgument[Any] | None = None, version_id_generator: Literal[False] | Callable[[Any], Any] | None = None, polymorphic_on: _ORMColumnExprArgument[Any] | str | MapperProperty[Any] | None = None, _polymorphic_map: Dict[Any, Mapper[Any]] | None = None, polymorphic_identity: Any | None = None, concrete: bool = False, with_polymorphic: _WithPolymorphicArg | None = None, polymorphic_abstract: bool = False, polymorphic_load: Literal['selectin', 'inline'] | None = None, allow_partial_pks: bool = True, batch: bool = True, column_prefix: str | None = None, include_properties: Sequence[str] | None = None, exclude_properties: Sequence[str] | None = None, passive_updates: bool = True, passive_deletes: bool = False, confirm_deleted_rows: bool = True, eager_defaults: Literal[True, False, 'auto'] = 'auto', legacy_is_orphan: bool = False, _compiled_cache_size: int = 100)
```

一个新的 `Mapper` 对象的直接构造函数。

不直接调用 `Mapper` 构造函数，通常通过使用 `registry` 对象通过声明式或命令式映射样式调用。

在 2.0 版本中进行了更改：公开的 `mapper()` 函数已移除；对于传统的映射配置，请使用 `registry.map_imperatively()` 方法。

下面记录的参数可以传递给 `registry.map_imperatively()` 方法，或者可以在具有声明性的 Mapper 配置选项中描述的 `__mapper_args__` 声明类属性中传递。

参数：

+   `class_` – 要映射的类。在使用声明式时，此参数将自动传递为声明的类本身。

+   `local_table` – 要映射到的 `Table` 或其他 `FromClause`（即可选择的）。如果此映射器使用单表继承从另一个映射器继承，则可以为 `None`。在使用声明式时，此参数由扩展自动传递，根据通过 `DeclarativeBase.__table__` 属性配置的内容或通过 `DeclarativeBase.__tablename__` 属性的结果产生的 `Table`。

+   `polymorphic_abstract` –

    表示此类将在多态层次结构中映射，但不会直接实例化。该类通常被映射，只是在继承层次结构中没有对 `Mapper.polymorphic_identity` 的要求。但是，该类必须是使用基类中的 `Mapper.polymorphic_on` 的多态继承方案的一部分。

    2.0 版中的新功能。

    另请参见

    使用 polymorphic_abstract 构建更深层次的层次结构

+   `always_refresh` – 如果为 True，则为此映射类的所有查询操作将覆盖已存在于会话中的对象实例中的所有数据，用从数据库加载的任何信息擦除任何内存中的更改。强烈不建议使用此标志；作为替代方案，请参见方法 `Query.populate_existing()`。

+   `allow_partial_pks` – 默认为 True。表示具有一些 NULL 值的复合主键应被视为可能存在于数据库中。这会影响映射器是否将传入的行分配给现有标识，以及 `Session.merge()` 是否首先检查数据库中特定主键值。例如，如果已映射到 OUTER JOIN，则可能会出现“部分主键”。

+   `batch` – 默认为 `True`，表示可以将多个实体的保存操作一起批处理以提高效率。将其设置为 False 表示在保存下一个实例之前将完全保存一个实例。这在极为罕见的情况下使用，即 `MapperEvents` 监听器需要在单个行持久性操作之间被调用的情况下。

+   `column_prefix` –

    一个字符串，当将 `Column` 对象自动分配为映射类的属性时，将会在映射属性名称之前添加。不影响在 `Mapper.properties` 字典中显式映射的 `Column` 对象。

    此参数通常与将 `Table` 对象保持分开的命令式映射一起使用。假设 `user_table` `Table` 对象具有名为 `user_id`、`user_name` 和 `password` 的列：

    ```py
    class User(Base):
        __table__ = user_table
        __mapper_args__ = {'column_prefix':'_'}
    ```

    上述映射将 `user_id`、`user_name` 和 `password` 列分配给映射的 `User` 类上名为 `_user_id`、`_user_name` 和 `_password` 的属性。

    `Mapper.column_prefix` 参数在现代用法中不常见。对于处理反射表，更灵活的自动命名方案是拦截反射时的 `Column` 对象；请参阅从反射表自动化列命名方案一节中关于此用法模式的注释。

+   `concrete` –

    如果为 True，则表示此映射器应使用具体表继承与其父映射器。

    请参阅具体表继承中的示例。

+   `confirm_deleted_rows` – 默认为 True；当基于特定主键发生 DELETE 时，如果匹配的行数不等于预期的行数，则会发出警告。可以将此参数设置为 False，以处理数据库 ON DELETE CASCADE 规则可能自动删除某些行的情况。警告可能在将来的版本中更改为异常。

+   `eager_defaults` –

    如果为 True，则 ORM 将在 INSERT 或 UPDATE 后立即获取服务器生成的默认值的值，而不是将其保留为过期以在下次访问时获取。这可以用于需要在 flush 完成之前立即获取服务器生成值的事件方案。

    值的获取可以通过在 `INSERT` 或 `UPDATE` 语句中与 `RETURNING` 一起使用，或者在 `INSERT` 或 `UPDATE` 之后添加额外的 `SELECT` 语句，如果后端不支持 `RETURNING`。

    使用 `RETURNING` 对于 SQLAlchemy 可以利用 insertmanyvalues 特别适用于 `INSERT` 语句，而使用额外的 `SELECT` 相对性能较差，增加了额外的 SQL 往返，如果这些新属性不被访问，则这些往返是不必要的。

    因此，`Mapper.eager_defaults` 默认为字符串值`"auto"`，表示应该使用 `RETURNING` 获取 INSERT 的服务器默认值，如果后端数据库支持的话，并且如果正在使用的方言支持“insertmanyreturning”作为 INSERT 语句。如果后端数据库不支持 `RETURNING` 或者“insertmanyreturning”不可用，则不会获取服务器默认值。

    从版本 2.0.0rc1 开始更改：为 `Mapper.eager_defaults` 添加了“auto”选项

    另请参阅

    获取服务器生成的默认值

    从版本 2.0.0 开始更改：`RETURNING`现在可以同时使用插入多行的 insertmanyvalues 功能，这使得支持的后端上的`Mapper.eager_defaults`特性性能非常高。

+   `exclude_properties` –

    排除映射的字符串列名列表或集合。

    另请参见

    映射表列的子集

+   `include_properties` –

    要映射的字符串列名的包含列表或集合。

    另请参见

    映射表列的子集

+   `inherits` –

    映射类或其中一个的对应`Mapper`，指示此`Mapper`应从中*继承*的超类。此处映射的类必须是另一个映射器类的子类。在使用声明式时，此参数会自动传递，因为已声明类的自然类层次结构。

    另请参见

    映射类继承层次结构

+   `inherit_condition` – 对于联接表继承，定义两个表如何连接的 SQL 表达式；默认为两个表之间的自然连接。

+   `inherit_foreign_keys` – 当使用`inherit_condition`并且存在的列缺少`ForeignKey`配置时，可以使用此参数来指定哪些列是“外键”。在大多数情况下可以保持为`None`。

+   `legacy_is_orphan` –

    布尔值，默认为`False`。当为`True`时，指定对由此映射器映射的对象应用“传统”孤立考虑，这意味着仅当它从指向此映射器的*所有*父级中解除关联时，即将删除孤立级联的挂起（即，非持久性）对象才会自动从所拥有的`Session`中清除。新的默认行为是，当对象与指定了`delete-orphan`级联的*任何*父级之一解除关联时，对象会自动从其父级中清除。此行为与持久性对象的行为更一致，并允许行为在更多的场景中独立于孤立对象是否已刷新。

    有关此更改的详细信息和示例，请参见将“待处理”对象视为“孤立”对象的考虑更为积极。

+   `non_primary` –

    指定此`Mapper`

    除了“主”映射器之外，也就是用于持久化的映射器。在此创建的`Mapper`可用于将类的临时映射到备用可选择的对象上，仅用于加载。

    自版本 1.3 起已弃用：`mapper.non_primary`参数已弃用，并将在将来的发布版本中删除。非主映射器的功能现在更适合使用`AliasedClass`构造，1.3 中也可以作为`relationship()`的目标使用。

    另请参阅

    与别名类的关系 - 新模式，消除了`Mapper.non_primary`标志的需要。

+   `passive_deletes` - 

    指示在删除联合表继承实体时外键列的 DELETE 行为。基本映射器默认为`False`；对于继承映射器，默认为`False`，除非在超类映射器上将值设置为`True`。

    当为`True`时，假定已在将此映射器的表与其超类表链接的外键关系上配置了 ON DELETE CASCADE，以便当工作单元尝试删除实体时，只需为超类表发出 DELETE 语句，而不是为此表发出 DELETE 语句。

    当为`False`时，将为此映射器的表分别发出 DELETE 语句。如果此表的本地主键属性未加载，则必须发出 SELECT 以验证这些属性；请注意，联合表子类的主键列不是对象整体的“主键”部分。

    请注意，`True`的值始终强制应用于子类映射器；也就是说，超类无法指定无主动删除而不对所有子类映射器产生影响。

    另请参阅

    在 ORM 关系中使用外键 ON DELETE 级联 - 描述了与`relationship()`一起使用的类似功能。 

    `mapper.passive_updates` - 支持联合表继承映射的 ON UPDATE CASCADE

+   `passive_updates` - 

    指示联合表继承映射中主键列更改时外键列的 UPDATE 行为。默认为`True`。

    当为 True 时，假定数据库上的外键已配置为 ON UPDATE CASCADE，并且数据库将处理从源列到联合表行上的依赖列的 UPDATE 传播。

    当为 False 时，假定数据库不执行参照完整性，并且不会为更新发出自己的 CASCADE 操作。在主键更改期间，工作单元过程将针对依赖列发出 UPDATE 语句。

    另请参阅

    可变主键 / 更新级联 - 描述与 `relationship()` 一起使用的类似功能的说明

    `mapper.passive_deletes` - 为连接表继承映射器支持 ON DELETE CASCADE

+   `polymorphic_load` –

    在继承层次结构中的子类中指定“多态加载”行为（仅适用于连接和单表继承）。有效值为：

    > +   “‘inline’” - 指定此类应该是“with_polymorphic”映射器的一部分，例如，它的列将包含在针对基础的 SELECT 查询中。
    > +   
    > +   “‘selectin’” - 指定当加载此类的实例时，将发出额外的 SELECT 来检索特定于此子类的列。SELECT 使用 IN 一次性检索多个子类。

    版本 1.2 中的新功能。

    另请参阅

    在映射器上配置 with_polymorphic()

    使用 selectin_polymorphic()

+   `polymorphic_on` –

    指定用于确定传入行的目标类的列、属性或 SQL 表达式，当存在继承类时。

    可以指定为字符串属性名称，也可以指定为 SQL 表达式，例如 `Column` 或在声明性映射中为 `mapped_column()` 对象。通常期望 SQL 表达式对应于基础映射的最底层映射的 `Table` 中的列：

    ```py
    class Employee(Base):
        __tablename__ = 'employee'

        id: Mapped[int] = mapped_column(primary_key=True)
        discriminator: Mapped[str] = mapped_column(String(50))

        __mapper_args__ = {
            "polymorphic_on":discriminator,
            "polymorphic_identity":"employee"
        }
    ```

    它也可以指定为 SQL 表达式，如此示例中我们使用 `case()` 构造来提供条件方法：

    ```py
    class Employee(Base):
        __tablename__ = 'employee'

        id: Mapped[int] = mapped_column(primary_key=True)
        discriminator: Mapped[str] = mapped_column(String(50))

        __mapper_args__ = {
            "polymorphic_on":case(
                (discriminator == "EN", "engineer"),
                (discriminator == "MA", "manager"),
                else_="employee"),
            "polymorphic_identity":"employee"
        }
    ```

    它也可能使用其字符串名称引用任何属性，在使用注释列配置时特别有用：

    ```py
    class Employee(Base):
        __tablename__ = 'employee'

        id: Mapped[int] = mapped_column(primary_key=True)
        discriminator: Mapped[str]

        __mapper_args__ = {
            "polymorphic_on": "discriminator",
            "polymorphic_identity": "employee"
        }
    ```

    当将 `polymorphic_on` 设置为引用不存在于本地映射的 `Table` 中的属性或表达式时，但是鉴别器的值应该持久化到数据库中时，鉴别器的值不会自动设置在新实例上；这必须由用户处理，可以通过手动方式或通过事件监听器来处理。建立这样一个监听器的典型方法如下所示：

    ```py
    from sqlalchemy import event
    from sqlalchemy.orm import object_mapper

    @event.listens_for(Employee, "init", propagate=True)
    def set_identity(instance, *arg, **kw):
        mapper = object_mapper(instance)
        instance.discriminator = mapper.polymorphic_identity
    ```

    在上述情况下，我们将映射类的`polymorphic_identity`值分配给`discriminator`属性，从而将该值持久化到数据库中的`discriminator`列中。

    警告

    目前，**只能设置一个鉴别器列**，通常在层次结构中的最底层类上。尚不支持“级联”多态列。

    参见

    映射类继承层次结构

+   `polymorphic_identity` –

    指定由`Mapper.polymorphic_on`设置引用的列表达式返回的值，用于识别此特定类的值。当接收到行时，与`Mapper.polymorphic_on`列表达式对应的值将与此值进行比较，指示应使用哪个子类来重建新对象。

    参见

    映射类继承层次结构

+   `properties` –

    将对象属性的字符串名称映射到`MapperProperty`实例的字典，这些实例定义了该属性的持久化行为。请注意，在映射到映射`Table`的`Column`对象时，除非被覆盖，否则会自动将其放置到`ColumnProperty`实例中。使用声明时，此参数将根据在声明类体中声明的所有这些`MapperProperty`实例自动传递。 

    参见

    属性字典 - 在 ORM 映射类概述中

+   `primary_key` –

    一组`Column`对象，或者是指向`Column`的属性名称的字符串名称，这些属性定义了要针对此映射器的可选择单元使用的主键。这通常只是`local_table`的主键，但可以在此处进行覆盖。

    从版本 2.0.2 开始更改：`Mapper.primary_key`参数也可以表示为字符串属性名称。

    参见

    映射到一组显式主键列 - 背景和示例用法

+   `version_id_col` –

    用于保持表中行的运行版本 ID 的`Column`。这用于检测并发更新或刷新中存在过时数据的存在。方法是检测如果 UPDATE 语句与最后已知的版本 ID 不匹配，则抛出`StaleDataError`异常。默认情况下，列必须是`Integer`类型，除非`version_id_generator`指定了替代版本生成器。

    另请参阅

    配置版本计数器 - 版本计数和原理的讨论。

+   `version_id_generator` –

    定义如何生成新版本 ID。默认为`None`，表示采用简单的整数计数方案。要提供自定义版本计数方案，请提供一个形如以下的可调用函数：

    ```py
    def generate_version(version):
        return next_version
    ```

    或者，可以使用服务器端版本控制功能，例如触发器，或者在版本 ID 生成器之外的程序化版本控制方案，通过指定值`False`。请参阅服务器端版本计数器以了解在使用此选项时的重要要点的讨论。

    另请参阅

    自定义版本计数器/类型

    服务器端版本计数器

+   `with_polymorphic` –

    一个形如`(<classes>, <selectable>)`的元组，表示“多态”加载的默认样式，即一次查询哪些表。`<classes>`是任何指示一次加载的继承类的单个或列表的映射器和/或类。特殊值`'*'`可用于指示应立即加载所有后代类。第二个元组参数`<selectable>`指示将用于查询多个类的可选择项。

    在现代映射中，`Mapper.polymorphic_load`参数可能比使用`Mapper.with_polymorphic`更可取，以指示多态加载样式的子类技术。

    另请参阅

    在映射器上配置 `with_polymorphic()`

```py
method add_properties(dict_of_properties)
```

将给定的属性字典添加到此映射器中，使用`add_property`。

```py
method add_property(key: str, prop: Column[Any] | MapperProperty[Any]) → None
```

向此映射器添加单个 MapperProperty。

如果尚未配置映射器，则只需将属性添加到发送到构造函数的初始属性字典中。如果此映射器已配置，则立即配置给定的 MapperProperty。

```py
attribute all_orm_descriptors
```

一个包含与映射类关联的所有`InspectionAttr`属性的命名空间。

这些属性在所有情况下都是与映射类或其超类关联的 Python 描述符。

此命名空间包括映射到类的属性以及由扩展模块声明的属性。它包括任何从`InspectionAttr`继承的 Python 描述符类型。这包括`QueryableAttribute`，以及扩展类型，如`hybrid_property`、`hybrid_method`和`AssociationProxy`。

为了区分映射属性和扩展属性，属性`InspectionAttr.extension_type`将引用一个常量，用于区分不同的扩展类型。

属性的排序基于以下规则：

1.  从子类到超类按顺序迭代类及其超类（即通过`cls.__mro__`迭代）

1.  对于每个类，按照它们在`__dict__`中出现的顺序生成属性，但以下步骤除外。在 Python 3.6 及以上版本中，此顺序将与类的构造相同，但有一个例外，即应用程序或映射器后来添加的属性。

1.  如果某个属性键也在超类`__dict__`中，那么它将包含在该类的迭代中，而不是它首次出现的类中。

上述过程产生了一种确定性排序，该排序是根据属性被分配给类的顺序确定的。

自版本 1.3.19 更改：确保对`Mapper.all_orm_descriptors()`的确定性排序。

当处理`QueryableAttribute`时，`QueryableAttribute.property`属性引用了`MapperProperty`属性，当通过`Mapper.attrs`引用映射属性集合时，将得到它。

警告

`Mapper.all_orm_descriptors`访问器命名空间是`OrderedProperties`的一个实例。这是一个类似字典的对象，包括一小部分命名方法，如`OrderedProperties.items()`和`OrderedProperties.values()`。当动态访问属性时，建议使用字典访问方案，例如`mapper.all_orm_descriptors[somename]`，而不是`getattr(mapper.all_orm_descriptors, somename)`，以避免名称冲突。

另请参阅

`Mapper.attrs`

```py
attribute attrs
```

该映射器的所有`MapperProperty`对象的命名空间。

这是一个根据其键名提供每个属性的对象。例如，具有`User.name`属性的`User`类的映射器将提供`mapper.attrs.name`，这将是代表`name`列的`ColumnProperty`。命名空间对象还可以进行迭代，这将产生每个`MapperProperty`。

`Mapper`具有该属性的几个预过滤视图，限制了返回的属性类型，包括`synonyms`、`column_attrs`、`relationships`和`composites`。

警告

`Mapper.attrs`访问器命名空间是`OrderedProperties`的一个实例。这是一个类似字典的对象，包括一小部分命名方法，如`OrderedProperties.items()`和`OrderedProperties.values()`。当动态访问属性时，建议使用字典访问方案，例如`mapper.attrs[somename]`，而不是`getattr(mapper.attrs, somename)`，以避免名称冲突。

另请参阅

`Mapper.all_orm_descriptors`

```py
attribute base_mapper: Mapper[Any]
```

继承链中最基础的`Mapper`。

在非继承场景中，此属性始终为此`Mapper`。在继承场景中，它引用继承链中所有其他`Mapper`对象的父级`Mapper`。

这是在映射器构建期间确定的*只读*属性。如果直接修改，行为未定义。

```py
attribute c: ReadOnlyColumnCollection[str, Column[Any]]
```

`Mapper.columns`的同义词。

```py
method cascade_iterator(type_: str, state: InstanceState[_O], halt_on: Callable[[InstanceState[Any]], bool] | None = None) → Iterator[Tuple[object, Mapper[Any], InstanceState[Any], _InstanceDict]]
```

遍历对象图中的每个元素及其映射器，对于符合给定级联规则的所有关系。

参数：

+   `type_` –

    级联规则的名称（即`"save-update"`，`"delete"`等）。

    注意

    在此处不接受`"all"`级联。有关通用对象遍历函数，请参阅如何遍历与给定对象相关的所有对象？。

+   `state` – 主要的 InstanceState。子项将根据为此对象的映射器定义的关系进行处理。

返回：

该方法产生单个对象实例。

另请参阅

级联

如何遍历与给定对象相关的所有对象？ - 演示了一个通用函数，用于遍历所有对象而不依赖于级联。

```py
attribute class_: Type[_O]
```

此`Mapper`映射到的类。

```py
attribute class_manager: ClassManager[_O]
```

`ClassManager`维护此`Mapper`的事件监听器和类绑定描述符。

这是在映射器构建期间确定的*只读*属性。如果直接修改，行为是未定义的。

```py
attribute column_attrs
```

返回此`Mapper`维护的所有`ColumnProperty`属性的命名空间。

另请参阅

`Mapper.attrs` - 所有`MapperProperty`对象的命名空间。

```py
attribute columns: ReadOnlyColumnCollection[str, Column[Any]]
```

由此`Mapper`维护的`Column`或其他标量表达式对象的集合。

该集合的行为与任何`Table`对象上的`c`属性相同，只是此映射中包含的列，且基于映射中定义的属性名称进行键控，而不一定是`Column`本身的`key`属性。此外，由`column_property()`映射的标量表达式也在此处。

这是在映射器构建期间确定的*只读*属性。如果直接修改，行为是未定义的。

```py
method common_parent(other: Mapper[Any]) → bool
```

如果给定的映射器与此映射器共享一个共同的继承父级，则返回 true。

```py
attribute composites
```

返回此`Mapper`维护的所有`Composite`属性的命名空间。

另请参阅

`Mapper.attrs` - 所有`MapperProperty`对象的命名空间。

```py
attribute concrete: bool
```

如果此`Mapper`是具体继承映射器，则表示`True`。

这是在映射器构建期间确定的*只读*属性。如果直接修改，行为是未定义的。

```py
attribute configured: bool = False
```

如果已配置此`Mapper`，则表示`True`。

这是在映射器构建期间确定的*只读*属性。如果直接修改，行为是未定义的。

另请参阅

`configure_mappers()`。

```py
attribute entity
```

检查 API 的一部分。

返回 self.class_。

```py
method get_property(key: str, _configure_mappers: bool = False) → MapperProperty[Any]
```

返回与给定键关联的 MapperProperty。

```py
method get_property_by_column(column: ColumnElement[_T]) → MapperProperty[_T]
```

给定`Column`对象，返回映射到此列的`MapperProperty`。

```py
method identity_key_from_instance(instance: _O) → _IdentityKeyType[_O]
```

根据其主键属性返回给定实例的标识键。

如果实例的状态已过期，则调用此方法将导致数据库检查以查看对象是否已被删除。如果行不再存在，则引发`ObjectDeletedError`。

此值通常也在实例状态下以属性名称键的形式找到。

```py
method identity_key_from_primary_key(primary_key: Tuple[Any, ...], identity_token: Any | None = None) → _IdentityKeyType[_O]
```

返回一个用于在标识映射中存储/检索项目的标识映射键。

参数：

**primary_key** - 表示标识符的值列表。

```py
method identity_key_from_row(row: Row[Any] | RowMapping | None, identity_token: Any | None = None, adapter: ORMAdapter | None = None) → _IdentityKeyType[_O]
```

返回用于在标识映射中存储/检索项目的标识映射键。

参数：

**行** -

从选择了 ORM 映射的主键列的结果集生成的`Row`或`RowMapping`。

从版本 2.0 开始：`Row`或`RowMapping`被接受作为“row”参数

```py
attribute inherits: Mapper[Any] | None
```

引用此`Mapper`继承自的`Mapper`（如果有）。

```py
attribute is_mapper = True
```

检查 API 的一部分。

```py
method is_sibling(other: Mapper[Any]) → bool
```

如果另一个映射器是此映射器的继承兄弟，则返回 true。共同的父级但不同的分支

```py
method isa(other: Mapper[Any]) → bool
```

如果此映射器从给定的映射器继承，则返回 True。

```py
attribute iterate_properties
```

返回所有 MapperProperty 对象的迭代器。

```py
attribute local_table: FromClause
```

此`Mapper`所引用的直接`FromClause`。

通常是`Table`的一个实例，可以是任何`FromClause`。

“本地”表是`Mapper`直接负责管理的可选择的表，从属性访问和 flush 的角度来看。对于非继承映射器，`Mapper.local_table`将与`Mapper.persist_selectable`相同。对于继承映射器，`Mapper.local_table`指的是包含该`Mapper`正在加载/持久化的列的特定部分，例如加入中的特定`Table`。

另请参阅

`Mapper.persist_selectable`。

`Mapper.selectable`.

```py
attribute mapped_table
```

自版本 1.3 起已弃用：使用 .persist_selectable

```py
attribute mapper
```

是检查 API 的一部分。

返回自身。

```py
attribute non_primary: bool
```

如果此`Mapper`是“非主”映射器，例如仅用于选择行而不用于持久化管理，则表示为 `True`。

这是在映射器构造期间确定的*只读*属性。如果直接修改，则行为未定义。

```py
attribute persist_selectable: FromClause
```

此`Mapper`映射到的`FromClause`。

通常是`Table`的一个实例，可以是任何`FromClause`。

`Mapper.persist_selectable`类似于`Mapper.local_table`，但表示继承方案中整体表示继承类层次结构的`FromClause`。

:attr.`.Mapper.persist_selectable`也与`Mapper.selectable`属性分开，后者可能是用于选择列的替代子查询。:attr.`.Mapper.persist_selectable`针对的是在持久化操作中将被写入的列。

另请参阅

`Mapper.selectable`。

`Mapper.local_table`。

```py
attribute polymorphic_identity: Any | None
```

表示一个标识符，该标识符在结果行加载期间与`Mapper.polymorphic_on`列匹配。

仅在继承时使用，此对象可以是与由`Mapper.polymorphic_on`表示的列的类型可比较的任何类型。

这是在映射器构造期间确定的*只读*属性。如果直接修改，则行为未定义。

```py
method polymorphic_iterator() → Iterator[Mapper[Any]]
```

遍历包括此映射器和所有后代映射器在内的集合。

这不仅包括直接继承的映射器，还包括所有它们的继承映射器。

要遍历整个层次结构，请使用`mapper.base_mapper.polymorphic_iterator()`。

```py
attribute polymorphic_map: Dict[Any, Mapper[Any]]
```

在继承场景中，将“多态身份”标识符映射到`Mapper`实例。

标识符可以是与`Mapper.polymorphic_on`所表示的列的类型可比较的任何类型。

映射器的继承链都将引用相同的多态映射对象。该对象用于将传入的结果行与目标映射器相关联。

这是在映射器构造期间确定的*只读*属性。如果直接修改，则行为未定义。

```py
attribute polymorphic_on: KeyedColumnElement[Any] | None
```

`Mapper`的`polymorphic_on`参数指定的`Column`或 SQL 表达式，在继承场景中。

此属性通常是一个`Column`实例，但也可能是一个表达式，例如从`cast()`派生的表达式。

这是在映射器构造期间确定的*只读*属性。如果直接修改，则行为未定义。

```py
attribute primary_key: Tuple[Column[Any], ...]
```

包含作为此`Mapper`在表映射的‘主键’的一部分的`Column`对象的集合的可迭代对象，从此`Mapper`的角度来看。

这个列表与`Mapper.persist_selectable`中的可选择项相对。在继承映射器的情况下，一些列可能由超类映射器管理。例如，在`Join`的情况下，主键由`Join`引用的所有表的主键列确定。

此列表也不一定与与基础表关联的主键列集合相同；`Mapper`具有可以覆盖`Mapper`认为是主键列的`primary_key`参数。

这是一个*只读*属性，在映射器构造期间确定。如果直接修改，行为是未定义的。

```py
method primary_key_from_instance(instance: _O) → Tuple[Any, ...]
```

返回给定实例的主键值列表。

如果实例的状态已过期，则调用此方法将导致数据库检查以查看对象是否已被删除。如果行不再存在，则会引发`ObjectDeletedError`。

```py
method primary_mapper() → Mapper[Any]
```

返回与此映射器的类键（类）对应的主映射器。

```py
attribute relationships
```

由此`Mapper`维护的所有`Relationship`属性的命名空间。

警告

`Mapper.relationships` 访问器命名空间是`OrderedProperties`的实例。这是一个类似于字典的对象，其中包含少量命名方法，例如`OrderedProperties.items()`和`OrderedProperties.values()`。在动态访问属性时，应优先使用字典访问方案，例如`mapper.relationships[somename]`而不是`getattr(mapper.relationships, somename)`，以避免名称冲突。

另请参阅

`Mapper.attrs` - 所有`MapperProperty`对象的命名空间。

```py
attribute selectable
```

默认情况下，此`Mapper`从中选择的`FromClause`构造。

通常情况下，这等同于`persist_selectable`，除非使用了`with_polymorphic`功能，在这种情况下，将返回完整的“多态”可选择项。

```py
attribute self_and_descendants
```

包括此映射器和所有后代映射器的集合。

这不仅包括直接继承的映射器，还包括所有它们继承的映射器。

```py
attribute single: bool
```

如果此 `Mapper` 是单表继承映射器，则表示 `True`。

如果设置了此标志，`Mapper.local_table` 将为 `None`。

这是在映射器构建期间确定的*只读*属性。如果直接修改，行为未定义。

```py
attribute synonyms
```

返回此 `Mapper` 维护的所有 `Synonym` 属性的命名空间。

另请参阅

`Mapper.attrs` - 所有 `MapperProperty` 对象的命名空间。

```py
attribute tables: Sequence[TableClause]
```

包含此 `Mapper` 意识到的所有 `Table` 或 `TableClause` 对象的序列。

如果映射器被映射到一个 `Join` 或者代表 `Select` 的 `Alias`，构成完整结构的各个 `Table` 对象将在这里表示。

这是在映射器构建期间确定的*只读*属性。如果直接修改，行为未定义。

```py
attribute validators: util.immutabledict[str, Tuple[str, Dict[str, Any]]]
```

一个不可变字典，其中属性已使用 `validates()` 装饰器装饰。

字典包含字符串属性名称作为键，映射到实际验证方法。

```py
attribute with_polymorphic_mappers
```

默认“多态”查询中包含的 `Mapper` 对象列表。

```py
class sqlalchemy.orm.MappedAsDataclass
```

混合类用于指示映射此类时，还将其转换为数据类。

另请参阅

声明性数据类映射 - 完整的 SQLAlchemy 本地数据类映射背景

版本 2.0 中的新功能。

```py
class sqlalchemy.orm.MappedClassProtocol
```

表示 SQLAlchemy 映射类的协议。

协议对类的类型是通用的，使用 `MappedClassProtocol[Any]` 来允许任何映射类。

**类签名**

类 `sqlalchemy.orm.MappedClassProtocol` (`typing_extensions.Protocol`)
