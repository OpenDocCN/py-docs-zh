# 水平分片

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/horizontal_shard.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/horizontal_shard.html)

水平分片支持。

定义了一个基本的“水平分片”系统，允许会话在多个数据库之间分发查询和持久化操作。

有关用法示例，请参见源分发中包含的水平分片示例。

深度炼金术

水平分片扩展是一个高级功能，涉及复杂的语句 -> 数据库交互以及对非平凡情况使用半公共 API。在使用这种更复杂且 less-production-tested 系统之前，应始终首先考虑更简单的引用多个数据库“分片”的方法，最常见的是每个“分片”使用一个独立的`会话`。

## API 文档

| 对象名称 | 描述 |
| --- | --- |
| set_shard_id | 一个加载器选项，用于为语句应用特定的分片 ID 到主查询，以及为其他关系和列加载器。 |
| 分片查询 | 与`分片会话`一起使用的查询类。 |
| 分片会话 |  |

```py
class sqlalchemy.ext.horizontal_shard.ShardedSession
```

**成员**

__init__(), connection_callable(), get_bind()

**类签名**

类`sqlalchemy.ext.horizontal_shard.ShardedSession` (`sqlalchemy.orm.session.Session`)

```py
method __init__(shard_chooser: ShardChooser, identity_chooser: Optional[IdentityChooser] = None, execute_chooser: Optional[Callable[[ORMExecuteState], Iterable[Any]]] = None, shards: Optional[Dict[str, Any]] = None, query_cls: Type[Query[_T]] = <class 'sqlalchemy.ext.horizontal_shard.ShardedQuery'>, *, id_chooser: Optional[Callable[[Query[_T], Iterable[_T]], Iterable[Any]]] = None, query_chooser: Optional[Callable[[Executable], Iterable[Any]]] = None, **kwargs: Any) → None
```

构造一个 ShardedSession。

参数：

+   `shard_chooser` – 一个可调用对象，传入一个 Mapper、一个映射实例，可能还有一个 SQL 子句，返回一个分片 ID。该 ID 可能基于对象中存在的属性，或者基于某种轮询方案。如果方案基于选择，则应在实例上设置任何状态，以标记它在未来参与该分片。

+   `identity_chooser` –

    一个可调用对象，传入一个 Mapper 和主键参数，应返回一个主键可能存在的分片 ID 列表。

    > 在 2.0 版本中更改：`identity_chooser` 参数取代了 `id_chooser` 参数。

+   `execute_chooser` –

    对于给定的`ORMExecuteState`，返回应发出查询的 shard_ids 列表。从所有返回的 shards 中返回的结果将合并到一个列表中。

    在 1.4 版本中更改：`execute_chooser`参数取代了`query_chooser`参数。

+   `shards` – 一个字符串分片名称到`Engine`对象的字典。

```py
method connection_callable(mapper: Mapper[_T] | None = None, instance: Any | None = None, shard_id: ShardIdentifier | None = None, **kw: Any) → Connection
```

提供一个用于工作单元刷新过程中使用的`Connection`。

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, *, shard_id: ShardIdentifier | None = None, instance: Any | None = None, clause: ClauseElement | None = None, **kw: Any) → _SessionBind
```

返回此`Session`绑定的“bind”。

“bind”通常是`Engine`的实例，除非`Session`已经明确地直接绑定到`Connection`的情况。

对于多重绑定或未绑定的`Session`，使用`mapper`或`clause`参数确定要返回的适当绑定。

请注意，当通过 ORM 操作调用`Session.get_bind()`时，通常会出现“映射器”参数，例如`Session.query()`中的每个单独的 INSERT/UPDATE/DELETE 操作，在`Session.flush()`调用中等。

解析顺序为：

1.  如果提供了映射器并且`Session.binds`存在，则首先基于正在使用的映射器，然后基于正在使用的映射类，然后基于映射类的`__mro__`中存在的任何基类，从更具体的超类到更一般的超类进行绑定定位。

1.  如果提供了子句并且`Session.binds`存在，则基于在`Session.binds`中找到的给定子句中存在的`Table`对象进行绑定定位。

1.  如果`Session.binds`存在，则返回该绑定。

1.  如果提供了子句，则尝试返回一个与最终与该子句关联的`MetaData`绑定。

1.  如果提供了映射器，尝试返回一个与最终与该映射器映射的`Table`或其他可选择对象关联的`MetaData`绑定。

1.  找不到绑定时，会引发`UnboundExecutionError`。

请注意，`Session.get_bind()` 方法可以在用户定义的 `Session` 子类上被重写，以提供任何类型的绑定解析方案。请参阅 自定义垂直分区 中的示例。

参数：

+   `mapper` – 可选的映射类或相应的 `Mapper` 实例。绑定可以首先从与此 `Session` 关联的“binds”映射中查询 `Mapper`，其次从 `Mapper` 映射到的 `Table` 的 `MetaData` 中查询绑定。

+   `clause` – 一个 `ClauseElement`（即 `select()`、`text()` 等）。如果 `mapper` 参数不存在或无法生成绑定，则将搜索给定的表达式构造，通常是与绑定的 `MetaData` 关联的 `Table`。

另请参阅

分区策略（例如每个会话多个数据库后端）

`Session.binds`

`Session.bind_mapper()`

`Session.bind_table()`

```py
class sqlalchemy.ext.horizontal_shard.set_shard_id
```

语句的加载器选项，用于将特定的分片 id 应用于主查询，以及额外的关系和列加载器。

`set_shard_id` 选项可以使用任何可执行语句的 `Executable.options()` 方法应用：

```py
stmt = (
    select(MyObject).
    where(MyObject.name == 'some name').
    options(set_shard_id("shard1"))
)
```

上面的语句在调用时将限制为主查询的“shard1”分片标识符，以及所有关系和列加载策略，包括像`selectinload()`这样的急切加载器，像`defer()`这样的延迟列加载器，以及惰性关系加载器`lazyload()`。

这样，`set_shard_id`选项的范围比在`Session.execute.bind_arguments`字典中使用“shard_id”参数要广泛得多。

2.0.0 版本中的新功能。

**成员**

__init__(), propagate_to_loaders

**类签名**

类`sqlalchemy.ext.horizontal_shard.set_shard_id` (`sqlalchemy.orm.ORMOption`)

```py
method __init__(shard_id: str, propagate_to_loaders: bool = True)
```

构造一个`set_shard_id`选项。

参数：

+   `shard_id` – 分片标识符

+   `propagate_to_loaders` – 如果保持默认值`True`，则分片选项将适用于诸如`lazyload()`和`defer()`之类的惰性加载器；如果为 False，则该选项不会传播到加载的对象。请注意，`defer()`在任何情况下始终限制为父行的 shard_id，因此该参数仅对`lazyload()`策略的行为产生净效果。

```py
attribute propagate_to_loaders
```

如果为 True，则表示此选项应该在“次要”SELECT 语句中传递，这些语句发生在关系惰性加载器以及属性加载/刷新操作中。

```py
class sqlalchemy.ext.horizontal_shard.ShardedQuery
```

与`ShardedSession`一起使用的查询类。

遗留特性

`ShardedQuery`是遗留`Query`类的子类。 `ShardedSession`现在通过`ShardedSession.execute()`方法支持 2.0 风格的执行。

**成员**

set_shard()

**类签名**

类`sqlalchemy.ext.horizontal_shard.ShardedQuery` (`sqlalchemy.orm.Query`)的构造函数

```py
method set_shard(shard_id: str) → Self
```

返回一个新的查询，限制为单个分片 ID。

返回的查询的所有后续操作将针对单个分片执行，而不考虑其他状态。

分片 ID 可以传递给 `Session.execute()` 的 bind_arguments 字典，以进行 2.0 样式的执行：

```py
results = session.execute(
    stmt,
    bind_arguments={"shard_id": "my_shard"}
)
```

## API 文档

| 对象名称 | 描述 |
| --- | --- |
| set_shard_id | 用于语句的加载器选项，以将特定的分片 ID 应用于主查询，以及额外的关系和列加载器。 |
| ShardedQuery | 与 `ShardedSession` 一起使用的查询类。 |
| ShardedSession |  |

```py
class sqlalchemy.ext.horizontal_shard.ShardedSession
```

**成员**

__init__(), connection_callable(), get_bind()

**类签名**

类`sqlalchemy.ext.horizontal_shard.ShardedSession` (`sqlalchemy.orm.session.Session`)的构造函数

```py
method __init__(shard_chooser: ShardChooser, identity_chooser: Optional[IdentityChooser] = None, execute_chooser: Optional[Callable[[ORMExecuteState], Iterable[Any]]] = None, shards: Optional[Dict[str, Any]] = None, query_cls: Type[Query[_T]] = <class 'sqlalchemy.ext.horizontal_shard.ShardedQuery'>, *, id_chooser: Optional[Callable[[Query[_T], Iterable[_T]], Iterable[Any]]] = None, query_chooser: Optional[Callable[[Executable], Iterable[Any]]] = None, **kwargs: Any) → None
```

构造一个 ShardedSession。

参数:

+   `shard_chooser` – 一个可调用对象，传入 Mapper、映射实例和可能的 SQL 子句，返回一个分片 ID。此 ID 可能基于对象中存在的属性，或者基于某种循环选择方案。如果方案基于选择，则应在实例上设置将来标记其参与该分片的任何状态。

+   `identity_chooser` –

    一个可调用对象，传入 Mapper 和主键参数，应返回此主键可能存在的分片 ID 列表。

    > 在 2.0 版本中更改：`identity_chooser` 参数取代了 `id_chooser` 参数。

+   `execute_chooser` –

    对于给定的`ORMExecuteState`，返回应发出查询的分片 ID 列表。返回的所有分片的结果将合并为单个列表。

    在 1.4 版本中更改：`execute_chooser` 参数取代了 `query_chooser` 参数。

+   `shards` – 一个字符串分片名称到`Engine` 对象的字典。

```py
method connection_callable(mapper: Mapper[_T] | None = None, instance: Any | None = None, shard_id: ShardIdentifier | None = None, **kw: Any) → Connection
```

提供一个用于工作单元刷新过程中使用的`Connection`。

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, *, shard_id: ShardIdentifier | None = None, instance: Any | None = None, clause: ClauseElement | None = None, **kw: Any) → _SessionBind
```

返回此`Session`绑定的“绑定”。

“绑定”通常是`Engine`的一个实例，除非`Session`已经被显式地直接绑定到`Connection`。

对于多重绑定或未绑定的`Session`，将使用`mapper`或`clause`参数来确定要返回的适当绑定。

请注意，“mapper”参数通常在通过 ORM 操作调用`Session.get_bind()`时存在，例如`Session.query()`中的每个单独的 INSERT/UPDATE/DELETE 操作，在`Session.flush()`调用等。

解析的顺序如下：

1.  如果给定了映射器并且存在`Session.binds`，则首先基于正在使用的映射器，然后基于正在使用的映射类，然后基于映射类的`__mro__`中存在的任何基类，从更具体的超类到更一般的超类。

1.  如果给定了子句并且存在`Session.binds`，则基于在`Session.binds`中找到的给定子句中的`Table`对象定位绑定。

1.  如果存在`Session.binds`，则返回该绑定。

1.  如果给定了子句，则尝试返回与该子句最终相关联的`MetaData`相关的绑定。

1.  如果给定了映射器，则尝试返回与最终与该映射器映射的`Table`或其他可选择对象相关联的`MetaData`相关的绑定。

1.  找不到绑定时，会引发`UnboundExecutionError`。

注意，`Session.get_bind()`方法可以在`Session`的用户定义子类上被覆盖，以提供任何类型的绑定解析方案。 请参阅自定义垂直分区中的示例。

参数：

+   `mapper` – 可选的映射类或对应的`Mapper`实例。 绑定可以首先通过查阅与此`Session`关联的“binds”映射，其次通过查阅`Mapper`映射到的`Table`关联的`MetaData`进行绑定。

+   `clause` – 一个`ClauseElement`（例如`select()`，`text()`等）。 如果未提供`mapper`参数或无法生成绑定，则将搜索给定的表达式构造以查找绑定元素，通常是与绑定的`MetaData`关联的`Table`。

另请参阅

分区策略（例如每个 Session 的多个数据库后端）

`Session.binds`

`Session.bind_mapper()`

`Session.bind_table()`

```py
class sqlalchemy.ext.horizontal_shard.set_shard_id
```

语句的加载器选项，可将特定的 shard id 应用于主查询以及用于额外的关系和列加载器。

可以使用任何可执行语句的`Executable.options()`方法应用`set_shard_id`选项：

```py
stmt = (
    select(MyObject).
    where(MyObject.name == 'some name').
    options(set_shard_id("shard1"))
)
```

在上述情况下，当调用语句时，主查询以及所有关系和列加载策略，包括诸如`selectinload()`、延迟列加载器`defer()`和惰性关系加载器`lazyload()`等都将限制为“shard1”分片标识符。

这样，`set_shard_id`选项的范围比在`Session.execute.bind_arguments`字典中使用“shard_id”参数要广泛得多。

2.0.0 版本中的新功能。

**成员**

__init__(), propagate_to_loaders

**类签名**

类`sqlalchemy.ext.horizontal_shard.set_shard_id`（`sqlalchemy.orm.ORMOption`）

```py
method __init__(shard_id: str, propagate_to_loaders: bool = True)
```

构造一个`set_shard_id`选项。

参数：

+   `shard_id` – 分片标识符

+   `propagate_to_loaders` – 如果保持默认值`True`，则`shard`选项将对懒加载器（如`lazyload()`和`defer()`）生效；如果为`False`，则该选项将不会传播到已加载的对象。请注意，`defer()`无论如何都会限制到父行的`shard_id`，因此该参数仅对`lazyload()`策略的行为产生净效果。

```py
attribute propagate_to_loaders
```

如果为`True`，表示此选项应传递到关系懒加载器以及属性加载/刷新操作中发生的“次要”SELECT 语句。

```py
class sqlalchemy.ext.horizontal_shard.ShardedQuery
```

与`ShardedSession`一起使用的查询类。

遗留特性

`ShardedQuery`是遗留`Query`类的子类。`ShardedSession`现在通过`ShardedSession.execute()`方法支持 2.0 风格的执行。

**成员**

set_shard()

**类签名**

类 `sqlalchemy.ext.horizontal_shard.ShardedQuery`（`sqlalchemy.orm.Query`）

```py
method set_shard(shard_id: str) → Self
```

返回一个新的查询，限定在单个分片 ID。

返回的查询的所有后续操作都将针对单个分片进行，而不考虑其他状态。

shard_id 可以传递给 `Session.execute()` 的 bind_arguments 字典，用于执行 2.0 风格的操作：

```py
results = session.execute(
    stmt,
    bind_arguments={"shard_id": "my_shard"}
)
```
