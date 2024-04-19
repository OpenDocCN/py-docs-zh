# 关系 API

> 原文：[`docs.sqlalchemy.org/en/20/orm/relationship_api.html`](https://docs.sqlalchemy.org/en/20/orm/relationship_api.html)

| 对象名称 | 描述 |
| --- | --- |
| backref(name, **kwargs) | 在使用`relationship.backref`参数时，提供在生成新的`relationship()`时使用的特定参数。 |
| dynamic_loader([argument], **kw) | 构造一个动态加载的映射器属性。 |
| foreign(expr) | 使用‘foreign’注释对主要连接表达式的部分进行注释。 |
| relationship([argument, secondary], *, [uselist, collection_class, primaryjoin, secondaryjoin, back_populates, order_by, backref, overlaps, post_update, cascade, viewonly, init, repr, default, default_factory, compare, kw_only, lazy, passive_deletes, passive_updates, active_history, enable_typechecks, foreign_keys, remote_side, join_depth, comparator_factory, single_parent, innerjoin, distinct_target_key, load_on_pending, query_class, info, omit_join, sync_backref], **kw) | 提供两个映射类之间的关联。 |
| remote(expr) | 使用‘remote’注释对主要连接表达式的部分进行注释。 |

```py
function sqlalchemy.orm.relationship(argument: _RelationshipArgumentType[Any] | None = None, secondary: _RelationshipSecondaryArgument | None = None, *, uselist: bool | None = None, collection_class: Type[Collection[Any]] | Callable[[], Collection[Any]] | None = None, primaryjoin: _RelationshipJoinConditionArgument | None = None, secondaryjoin: _RelationshipJoinConditionArgument | None = None, back_populates: str | None = None, order_by: _ORMOrderByArgument = False, backref: ORMBackrefArgument | None = None, overlaps: str | None = None, post_update: bool = False, cascade: str = 'save-update, merge', viewonly: bool = False, init: _NoArg | bool = _NoArg.NO_ARG, repr: _NoArg | bool = _NoArg.NO_ARG, default: _NoArg | _T = _NoArg.NO_ARG, default_factory: _NoArg | Callable[[], _T] = _NoArg.NO_ARG, compare: _NoArg | bool = _NoArg.NO_ARG, kw_only: _NoArg | bool = _NoArg.NO_ARG, lazy: _LazyLoadArgumentType = 'select', passive_deletes: Literal['all'] | bool = False, passive_updates: bool = True, active_history: bool = False, enable_typechecks: bool = True, foreign_keys: _ORMColCollectionArgument | None = None, remote_side: _ORMColCollectionArgument | None = None, join_depth: int | None = None, comparator_factory: Type[RelationshipProperty.Comparator[Any]] | None = None, single_parent: bool = False, innerjoin: bool = False, distinct_target_key: bool | None = None, load_on_pending: bool = False, query_class: Type[Query[Any]] | None = None, info: _InfoType | None = None, omit_join: Literal[None, False] = None, sync_backref: bool | None = None, **kw: Any) → _RelationshipDeclared[Any]
```

提供两个映射类之间的关联。

这对应于父子或关联表关系。构造的类是`Relationship`的一个实例。

另请参阅

使用 ORM 相关对象 - 在 SQLAlchemy 统一教程中对`relationship()`进行教程介绍

关系配置 - 叙述性文档

参数：

+   `argument` –

    此参数指的是要相关联的类。它接受几种形式，包括对目标类本身的直接引用，目标类的`Mapper`实例，将在调用时返回对类或`Mapper`的引用的 Python 可调用/ lambda，以及类的字符串名称，这将从正在使用的`registry`中解析类，以便找到该类，例如：

    ```py
    class SomeClass(Base):
        # ...

        related = relationship("RelatedClass")
    ```

    `relationship.argument` 也可以完全省略不在 `relationship()` 构造中传递，而是放置在左侧的 `Mapped` 注释中，如果关系预期为集合，则应包含 Python 集合类型，例如：

    ```py
    class SomeClass(Base):
        # ...

        related_items: Mapped[List["RelatedItem"]] = relationship()
    ```

    或者对于多对一或一对一关系：

    ```py
    class SomeClass(Base):
        # ...

        related_item: Mapped["RelatedItem"] = relationship()
    ```

    另请参阅

    使用声明性定义映射属性 - 在使用声明性时关系配置的更多细节。

+   `secondary` –

    对于多对多关系，指定中间表，通常是 `Table` 的一个实例。在较不常见的情况下，参数也可以指定为 `Alias` 构造，甚至是 `Join` 构造。

    `relationship.secondary` 可以作为一个可调用函数传递，该函数在映射初始化时进行评估。使用声明性时，它也可以是一个字符串参数，指示存在于与父映射的 `Table` 关联的 `MetaData` 集合中的 `Table` 的名称。

    警告

    当作为 Python 可评估字符串传递时，使用 Python 的 `eval()` 函数解释该参数。**不要将不受信任的输入传递给该字符串**。有关声明性评估 `relationship()` 参数的详细信息，请参阅 关系参数的评估 。

    `relationship.secondary`关键字参数通常适用于中间`Table`在任何直接类映射中没有其他表达的情况。如果“secondary”表也在其他地方明确映射（例如在关联对象中），则应考虑应用`relationship.viewonly`标志，以便这个`relationship()`不用于可能与关联对象模式冲突的持久化操作。

    另请参阅

    多对多 - “多对多”关系的参考示例。

    自引用多对多关系 - 在自引用情况下使用多对多的具体细节。

    配置多对多关系 - 在使用声明式时的附加选项。

    关联对象 - 在组合关联表关系时的一种替代`relationship.secondary`的方法，允许在关联表上指定附加属性。

    复合“次要”连接 - 一种较少使用的模式，在某些情况下可以使复杂的`relationship()` SQL 条件得以使用。

+   `active_history=False` – 当为`True`时，表示当替换时应加载多对一引用的“先前”值，如果尚未加载。通常，对于简单的多对一引用，历史跟踪逻辑只需要了解“新”值即可执行刷新。此标志适用于使用`get_history()`并且还需要知道属性的“先前”值的应用程序。

+   `backref` –

    引用一个字符串关系名称，或者一个`backref()`构造，将被用来自动生成一个新的`relationship()`在相关类上，然后使用双向`relationship.back_populates`配置来引用这个类。

    在现代 Python 中，应优先使用`relationship()`和`relationship.back_populates`的显式用法，因为在映射器配置和概念上更为健壮直观。它还与 SQLAlchemy 2.0 中引入的新的[**PEP 484**](https://peps.python.org/pep-0484/)类型特性集成，而动态生成属性则不支持此特性。

    另请参阅

    使用传统的 ‘backref’ 关系参数 - 关于使用`relationship.backref`的注意事项

    与 ORM 相关对象的工作 - 在 SQLAlchemy 统一教程中，使用`relationship.back_populates`提供了双向关系配置和行为的概述

    `backref()` - 在使用`relationship.backref`时允许控制`relationship()`的配置。

+   `back_populates` –

    表示与此类同步的相关类上的`relationship()`的名称。通常期望相关类上的`relationship()`也参考了这个。这允许每个`relationship()`两侧的对象同步 Python 状态变化，并为工作单元刷新过程提供指令，指导沿着这些关系的更改如何持久化。

    另请参阅

    与 ORM 相关对象的工作 - 在 SQLAlchemy 统一教程中，提供了双向关系配置和行为的概述。

    基本关系模式 - 包含了许多`relationship.back_populates`的示例。

    `relationship.backref` - 旧形式，允许更简洁的配置，但不支持显式类型化

+   `overlaps` –

    字符串名称或以逗号分隔的名称集，位于此映射器、后代映射器或目标映射器上，此关系可以与之同时写入相同的外键。此唯一的效果是消除此关系将在持久化时与另一个关系发生冲突的警告。这用于真正可能在写入时与彼此冲突的关系，但应用程序将确保不会发生此类冲突。

    新版本 1.4 中新增。

    另请参阅

    关系 X 将列 Q 复制到列 P，与关系‘Y’冲突 - 用法示例

+   `cascade` –

    一个逗号分隔的级联规则列表，确定 Session 操作应该如何从父级到子级进行“级联”。默认为 `False`，表示应该使用默认级联 - 此默认级联为 `"save-update, merge"`。

    可用的级联包括 `save-update`、`merge`、`expunge`、`delete`、`delete-orphan` 和 `refresh-expire`。另一个选项 `all` 表示 `"save-update, merge, refresh-expire, expunge, delete"` 的简写，通常用于指示相关对象应在所有情况下跟随父对象，并在取消关联时删除。

    另请参阅

    级联 - 每个可用级联选项的详细信息。

+   `cascade_backrefs=False` –

    旧版本；此标志始终为 False。

    在版本 2.0 中更改：“cascade_backrefs” 功能已被移除。

+   `collection_class` –

    一个类或可调用对象，返回一个新的列表持有对象。将用于代替普通列表存储元素。

    另请参阅

    自定义集合访问 - 入门文档和示例。

+   `comparator_factory` –

    一个扩展了 `Comparator` 的类，为比较操作提供自定义 SQL 子句生成。

    另请参阅

    `PropComparator` - 在此级别重新定义比较器的一些详细信息。

    操作符自定义 - 关于这一特性的简要介绍。

+   `distinct_target_key=None` –

    指示“子查询”预加载是否应将 DISTINCT 关键字应用于内层 SELECT 语句。当留空时，当目标列不包括目标表的完整主键时，将应用 DISTINCT 关键字。当设置为 `True` 时，DISTINCT 关键字将无条件地应用于内层 SELECT。

    当 DISTINCT 降低内层子查询的性能超出重复的内层行可能导致的性能时，将此标志设置为 False 可能是合适的。

    另请参阅

    关系加载技术 - 包括对子查询预加载的介绍。

+   `doc` – 将应用于生成描述符的文档字符串。

+   `foreign_keys` –

    要在此 `relationship()` 对象的 `relationship.primaryjoin` 条件的上下文中用作“外键”列或引用远程列中的值的列的列表。也就是说，如果此 `relationship()` 的 `relationship.primaryjoin` 条件是 `a.id == b.a_id`，并且要求 `b.a_id` 中的值在 `a.id` 中存在，则此 `relationship()` 的“外键”列是 `b.a_id`。

    在正常情况下，**不需要** `relationship.foreign_keys` 参数。`relationship()` 将根据那些指定了 `ForeignKey` 的 `Column` 对象或以其他方式列在 `ForeignKeyConstraint` 构造中的引用列的那些列自动确定在 `relationship.primaryjoin` 条件中应被视为“外键”列。只有在以下情况下才需要 `relationship.foreign_keys` 参数：

    > 1.  从本地表到远程表的连接可以有多种构造方式，因为存在多个外键引用。设置 `foreign_keys` 将限制 `relationship()` 仅考虑此处指定的列作为“外键”。
    > 1.  
    > 1.  被映射的 `Table` 实际上没有 `ForeignKey` 或 `ForeignKeyConstraint` 构造存在，通常是因为该表是从不支持外键反射的数据库（MySQL MyISAM）反射而来。
    > 1.  
    > 1.  `relationship.primaryjoin` 参数用于构建非标准的连接条件，该条件使用通常不会引用其“父”列的列或表达式，例如使用 SQL 函数进行的复杂比较表达的连接条件。

    当`relationship()`构造引发信息性错误消息时，建议使用`relationship.foreign_keys`参数，以处理模棱两可的情况。在典型情况下，如果`relationship()`没有引发任何异常，则通常不需要`relationship.foreign_keys`参数。

    `relationship.foreign_keys` 也可以传递为一个在映射器初始化时求值的可调用函数，并且在使用声明性时可以传递为 Python 可评估的字符串。

    警告

    当作为 Python 可评估的字符串传递时，该参数将使用 Python 的`eval()`函数进行解释。**不要将不受信任的输入传递给此字符串**。有关使用`relationship()`参数的声明性评估的详细信息，请参阅关系参数的评估。

    另请参阅

    处理多个连接路径

    创建自定义外键条件

    `foreign()` - 允许在`relationship.primaryjoin`条件中直接注释“外键”列。

+   `info` – 可选数据字典，将被填充到此对象的`MapperProperty.info`属性中。

+   `innerjoin=False` –

    当为`True`时，连接式急加载将使用内连接而不是外连接来与相关表连接。该选项的目的通常是性能之一，因为内连接通常比外连接执行得更好。

    当关系引用通过不可为空的本地外键引用对象时，或者引用为一对一或保证具有一个或至少一个条目的集合时，可以将此标志设置为`True`。

    该选项支持与`joinedload.innerjoin`相同的“嵌套”和“未嵌套”选项。有关嵌套/未嵌套行为的详细信息，请参阅该标志。

    另请参阅

    `joinedload.innerjoin` - 由加载器选项指定的选项，包括嵌套行为的详细信息。

    应该使用什么类型的加载？ - 讨论各种加载器选项的一些细节。

+   `join_depth` –

    当非`None`时，表示“急切”加载器应该在自引用或循环关系上连接多少级深度的整数值。该数字计算相同 Mapper 在加载条件中沿着特定连接分支出现的次数。当保持默认值`None`时，急切加载器在遇到已经在链中较高位置的相同目标映射器时将停止链接。此选项适用于连接和子查询急切加载器。

    另请参见

    配置自引用急切加载 - 入门文档和示例。

+   `lazy='select'` –

    指定相关项目应该如何加载。默认值为`select`。值包括：

    +   `select` - 当首次访问属性时，应该懒加载项目，使用一个单独的 SELECT 语句，或者对于简单的多对一引用，使用标识映射获取。

    +   `immediate` - 项目应该在父项加载时加载，使用一个单独的 SELECT 语句，或者对于简单的多对一引用，使用标识映射获取。

    +   `joined` - 项目应该在与父项相同的查询中“急切”加载，使用 JOIN 或 LEFT OUTER JOIN。JOIN 是“外部”的还是不是由`relationship.innerjoin`参数确定。

    +   `subquery` - 项目应该在父项加载时“急切”加载，使用一个额外的 SQL 语句，为每个请求的集合发出一个 JOIN 到原始语句的子查询。

    +   `selectin` - 项目应该在父项加载时“急切”加载，使用一个或多个额外的 SQL 语句，发出一个 JOIN 到直接父对象，使用 IN 子句指定主键标识符。

    +   `noload` - 任何时候都不应发生加载。相关集合将保持为空。不建议一般使用`noload`策略。对于一般的“永不加载”方法，请参见仅写关系。

    +   `raise` - 禁止惰性加载；如果属性的值尚未通过急切加载加载，则访问该属性将引发`InvalidRequestError`。当对象在加载后要从其附加的`Session`中分离时，可以使用此策略。

    +   `raise_on_sql` - 禁止发出 SQL 的延迟加载；如果该属性的值尚未通过急加载加载，则访问该属性将引发`InvalidRequestError`，“如果延迟加载需要发出 SQL”。如果延迟加载可以从标识映射中提取相关值或确定它应该是 None，则加载该值。当对象将保持与附加的`Session`关联时，可以使用此策略，但应阻止附加的额外 SELECT 语句。

    +   `write_only` - 该属性将配置为具有特殊的“虚拟集合”，该集合可能接收`WriteOnlyCollection.add()`和`WriteOnlyCollection.remove()`命令以添加或删除单个对象，但绝不会直接从数据库加载或迭代完整对象集。而是提供了诸如`WriteOnlyCollection.select()`、`WriteOnlyCollection.insert()`、`WriteOnlyCollection.update()`和`WriteOnlyCollection.delete()`等方法，生成可用于批量加载和修改行的 SQL 构造。用于从不适合一次加载到内存中的大型集合。

        当在声明性映射的左侧提供`WriteOnlyMapped`注释时，将自动配置`write_only`加载程序样式。有关示例，请参阅仅写关系部分。

        在版本 2.0 中新增。

        另请参阅

        仅写关系 - 在 ORM 查询指南中

    +   `dynamic` - 属性将为所有读操作返回预配置的`Query`对象，可以在迭代结果之前应用进一步的过滤操作。

        当在声明式映射中的左侧提供了`DynamicMapped`注释时，将自动配置`dynamic`加载程序样式。有关示例，请参见动态关系加载器一节。

        传统功能

        “动态”懒加载策略是现在描述的“只写”策略的传统形式，详见仅写关系一节。

        另请参见

        动态关系加载器 - 在 ORM 查询指南中

        仅写关系 - 用于大型集合的更普遍有用的方法，不应完全加载到内存中。

    +   True - ‘select’的同义词

    +   False - ‘joined’的同义词

    +   None - ‘noload’的同义词

    另请参见

    关系加载技术 - 在 ORM 查询指南中关于关系加载程序配置的完整文档。

+   `load_on_pending=False` –

    指示暂态或挂起父对象的加载行为。

    当设置为`True`时，会导致惰性加载程序对尚未持久的父对象发出查询，即从未刷新过的父对象。当自动刷新被禁用时，这可能会对挂起对象产生影响，或者对已“附加”到`Session`但不属于其挂起集合的暂态对象产生影响。

    `relationship.load_on_pending`标志在 ORM 正常使用时不会改善行为 - 对象引用应在对象级别构造，而不是在外键级别构造，以便它们在刷新进行之前以普通方式存在。此标志不打算供常规使用。

    另请参见

    `Session.enable_relationship_loading()` - 此方法为整个对象建立了“在挂起时加载”的行为，还允许在保持为暂态或游离状态的对象上加载。

+   `order_by` –

    指示加载这些项时应应用的排序。`relationship.order_by`预期引用目标类映射到的一个`Column`对象之一，或者绑定到引用列的目标类的属性本身。

    `relationship.order_by` 还可以作为可调用函数传递，该函数在映射器初始化时进行评估，并且在使用 Declarative 时可以作为 Python 可评估字符串进行传递。

    警告

    当作为 Python 可评估字符串传递时，该参数将使用 Python 的 `eval()` 函数进行解释。**不要将不受信任的输入传递给此字符串**。有关`relationship()`参数的声明性评估的详细信息，请参阅关系参数的评估。

+   `passive_deletes=False` -

    指示删除操作期间的加载行为。

    True 的值表示在父对象的删除操作期间不应加载未加载的子项目。通常，当删除父项目时，所有子项目都会加载，以便可以将它们标记为已删除，或者将它们的外键设置为 NULL。将此标志标记为 True 通常意味着已经存在一个 ON DELETE <CASCADE|SET NULL> 规则，该规则将处理数据库端的更新/删除子行。

    此外，将标志设置为字符串值“all”将禁用在父对象被删除且未启用删除或删除-孤儿级联时的“空值”子外键。当数据库端存在触发或错误提升方案时，通常会使用此选项。请注意，在刷新后，会话中的子对象上的外键属性不会更改，因此这是一个非常特殊的用例设置。此外，如果子对象与父对象解除关联，则“nulling out”仍会发生。

    另请参阅

    使用 ORM 关系的外键 ON DELETE 级联 - 入门文档和示例。

+   `passive_updates=True` -

    指示当引用的主键值在原位更改时要采取的持久性行为，这表示引用的外键列也需要更改其值。

    当为 True 时，假定数据库上的外键已配置为 `ON UPDATE CASCADE`，并且数据库将处理从源列到依赖行的 UPDATE 传播。当为 False 时，SQLAlchemy `relationship()` 构造将尝试发出自己的 UPDATE 语句以修改相关目标。但请注意，SQLAlchemy **无法** 对超过一级的级联发出 UPDATE。此外，将此标志设置为 False 在数据库实际强制执行引用完整性的情况下不兼容，除非这些约束明确为“延迟”，如果目标后端支持。

    强烈建议使用可变主键的应用程序将 `passive_updates` 设置为 True，并且使用数据库本身的引用完整性功能来高效完全处理更改。

    另请参阅

    可变主键 / 更新级联 - 介绍文档和示例。

    `mapper.passive_updates` - 类似的标志也适用于连接表继承映射。

+   `post_update` –

    这表示关系应该在插入后或删除前通过第二个 UPDATE 语句进行处理。该标志用于处理两个单独行之间的双向依赖关系（即每行引用另一行），否则将无法完全插入或删除两行，因为一行在另一行之前存在。当特定的映射安排将导致两行彼此依赖时，请使用此标志，例如，一个表与一组子行之间存在一对多关系，并且还有一个列引用该列表中的单个子行（即两个表相互包含对方的外键）。如果刷新操作返回检测到“循环依赖”错误，这表明您可能希望使用 `relationship.post_update` 来“打破”循环。

    另请参阅

    指向自身的行 / 相互依赖行 - 介绍文档和示例。

+   `primaryjoin` –

    将用作子对象与父对象之间的主要连接的 SQL 表达式，或者在多对多关系中将父对象连接到关联表。默认情况下，此值基于父表和子表（或关联表）的外键关系计算。

    `relationship.primaryjoin` 也可以作为一个可调用函数传递，该函数在映射器初始化时进行评估，并且在使用声明性时可以作为一个可评估的 Python 字符串进行传递。

    警告

    当作为一个可评估的 Python 字符串传递时，该参数将使用 Python 的 `eval()` 函数进行解释。**不要传递不受信任的输入给此字符串**。有关声明性评估 `relationship()` 参数的详细信息，请参阅关系参数的评估。

    另请参阅

    指定替代连接条件

+   `remote_side` –

    用于自引用关系，指示形成关系的“远端”的列或列列表。

    `relationship.remote_side` 还可以作为可调用函数传递，在映射器初始化时进行评估，并且在使用声明性时可以作为 Python 可评估字符串传递。

    警告

    当作为 Python 可评估字符串传递时，该参数将使用 Python 的 `eval()` 函数进行解释。**不要将不受信任的输入传递给此字符串**。有关使用 `relationship()` 参数的声明性评估的详细信息，请参阅关系参数的评估。

    另请参阅

    邻接列表关系 - 如何配置自引用关系的详细说明，`relationship.remote_side` 的使用。

    `remote()` - 完成与 `relationship.remote_side` 相同目的的注释函数，通常在使用自定义 `relationship.primaryjoin` 条件时使用。

+   `query_class` –

    `Query` 的子类，将在由“动态”关系返回的 `AppenderQuery` 内部使用，即指定了 `lazy="dynamic"` 的关系或以其他方式使用了 `dynamic_loader()` 函数构造的关系。

    另请参阅

    动态关联加载器 - “动态”关联加载器的介绍。

+   `secondaryjoin` –

    将用作关联表与子对象的连接的 SQL 表达式。默认情况下，此值根据关联和子表的外键关系计算而来。

    `relationship.secondaryjoin` 还可以作为可调用函数传递，在映射器初始化时进行评估，并且在使用声明性时可以作为 Python 可评估字符串传递。

    警告

    当作为 Python 可评估字符串传递时，该参数将使用 Python 的 `eval()` 函数进行解释。**不要将不受信任的输入传递给此字符串**。有关使用 `relationship()` 参数的声明性评估的详细信息，请参阅关系参数的评估。

    另请参阅

    指定替代连接条件

+   `single_parent` –

    当为 True 时，安装一个验证器，该验证器将阻止对象同时与多个父对象关联。这用于应将多对一或多对多关系视为一对一或一对多的情况。除了指定`delete-orphan`级联选项的多对一或多对多关系外，其使用是可选的。当要求此选项时，`relationship()`构造本身将引发错误指示。

    另请参阅

    级联操作 - 包括有关何时适合使用`relationship.single_parent`标志的详细信息。

+   `uselist` –

    一个布尔值，指示此属性是否应加载为列表或标量。在大多数情况下，此值由`relationship()`在映射配置时自动确定。当使用显式的`Mapped`注解时，`relationship.uselist`可以根据`Mapped`中的注解是否包含集合类来推导出。否则，`relationship.uselist`可以从关系的类型和方向推导出 - 一对多形成一个列表，多对一形成一个标量，多对多是一个列表。如果希望在通常存在列表的地方使用标量，例如双向一对一关系，请使用适当的`Mapped`注解或将`relationship.uselist`设置为 False。

    `relationship.uselist`标志也可用于现有的`relationship()`构造，作为一个只读属性，可用于确定此`relationship()`是否处理集合或标量属性：

    ```py
    >>> User.addresses.property.uselist
    True
    ```

    另请参阅

    一对一关系 - 介绍了“一对一”关系模式，通常涉及`relationship.uselist`的备用设置。

+   `viewonly=False` –

    当设置为`True`时，该关系仅用于加载对象，而不用于任何持久性操作。指定了`relationship.viewonly`的`relationship()`可以在`relationship.primaryjoin`条件内与更广泛的 SQL 操作一起使用，其中包括使用各种比较运算符以及 SQL 函数，如`cast()`。`relationship.viewonly`标志在定义任何不代表完整相关对象集的`relationship()`时也是一般用途，以防止对集合的修改导致持久性操作。

    另见

    关于使用视图关系参数的注意事项 - 使用`relationship.viewonly`时的最佳实践的更多细节。

+   `sync_backref` -

    一个布尔值，用于在此关系是`relationship.backref`或`relationship.back_populates`的目标时启用用于同步 Python 属性的事件。

    默认为`None`，表示应根据`relationship.viewonly`标志的值选择自动值。在其默认状态下，只有在关系的任一方都不是视图时状态变化才会被回填。

    版本 1.3.17 中新增。

    从版本 1.4 开始：- 指定了`relationship.viewonly`的关系自动意味着`relationship.sync_backref`为`False`。

    另见

    `relationship.viewonly`

+   `omit_join` -

    允许手动控制“selectin”自动连接优化。将其设置为`False`以禁用 SQLAlchemy 1.3 中添加的“omit join”功能；或者将其保留为`None`以保留自动优化。

    注意

    此标志只能设置为`False`。不需要将其设置为`True`，因为“omit_join”优化会自动检测到；如果未检测到，则不支持优化。

    在版本 1.3.11 中更改：设置`omit_join`为 True 现在会发出警告，因为这不是此标志的预期使用方式。

    从版本 1.3 开始新添加。

+   `init` – 专门针对声明性数据类映射，指定映射属性是否应作为 dataclass 流程生成的`__init__()`方法的一部分。

+   `repr` – 专门针对声明性数据类映射，指定映射属性是否应作为 dataclass 流程生成的`__repr__()`方法的一部分。

+   `default_factory` – 专门针对声明性数据类映射，指定一个默认值生成函数，该函数将作为 dataclass 流程生成的`__init__()`方法的一部分进行处理。

+   `compare` –

    专门针对声明性数据类映射，表示在生成映射类的`__eq__()`和`__ne__()`方法时，此字段是否应包含在比较操作中。

    从版本 2.0.0b4 开始新添加。

+   `kw_only` – 专门针对声明性数据类映射，表示在生成`__init__()`时此字段是否应标记为关键字参数。

```py
function sqlalchemy.orm.backref(name: str, **kwargs: Any) → ORMBackrefArgument
```

使用`relationship.backref`参数时，提供要在生成新的`relationship()`时使用的特定参数。

例如：

```py
'items':relationship(
    SomeItem, backref=backref('parent', lazy='subquery'))
```

一般认为`relationship.backref`参数是遗留的；对于现代应用程序，应优先使用显式的`relationship()`构造，使用`relationship.back_populates`参数进行链接。

另请参阅

使用传统的‘backref’关系参数的背景信息，请参阅使用传统的‘backref’关系参数。

```py
function sqlalchemy.orm.dynamic_loader(argument: _RelationshipArgumentType[Any] | None = None, **kw: Any) → RelationshipProperty[Any]
```

构造一个动态加载的映射器属性。

这与使用`relationship()`的`lazy='dynamic'`参数基本相同：

```py
dynamic_loader(SomeClass)

# is the same as

relationship(SomeClass, lazy="dynamic")
```

更多关于动态加载的详细信息，请参阅动态关系加载器一节。

```py
function sqlalchemy.orm.foreign(expr: _CEA) → _CEA
```

使用“foreign”注解注释主要联接表达式的一部分。

请参阅创建自定义外键条件一节，了解其用法描述。

另请参阅

创建自定义外键条件

`remote()`

```py
function sqlalchemy.orm.remote(expr: _CEA) → _CEA
```

使用“remote”注解注释主要联接表达式的一部分。

参见章节创建自定义外键条件以了解其用法描述。

请参阅也

创建自定义外键条件

`foreign()`
