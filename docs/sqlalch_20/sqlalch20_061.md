# ORM 异常

> 原文：[`docs.sqlalchemy.org/en/20/orm/exceptions.html`](https://docs.sqlalchemy.org/en/20/orm/exceptions.html)

SQLAlchemy ORM 异常。

| 对象名称 | 描述 |
| --- | --- |
| ConcurrentModificationError | `StaleDataError`的别名 |
| NO_STATE | 可能由仪器实现引发的异常类型。 |

```py
attribute sqlalchemy.orm.exc.ConcurrentModificationError
```

`StaleDataError`的别名

```py
exception sqlalchemy.orm.exc.DetachedInstanceError
```

尝试访问已分离的映射实例上未加载的属性。

**类签名**

类`sqlalchemy.orm.exc.DetachedInstanceError` (`sqlalchemy.exc.SQLAlchemyError`)

```py
exception sqlalchemy.orm.exc.FlushError
```

在 flush() 过程中检测到无效条件。

**类签名**

类`sqlalchemy.orm.exc.FlushError` (`sqlalchemy.exc.SQLAlchemyError`)

```py
exception sqlalchemy.orm.exc.LoaderStrategyException
```

一个属性的加载策略不存在。

**类签名**

类`sqlalchemy.orm.exc.LoaderStrategyException` (`sqlalchemy.exc.InvalidRequestError`)

```py
method __init__(applied_to_property_type: Type[Any], requesting_property: MapperProperty[Any], applies_to: Type[MapperProperty[Any]] | None, actual_strategy_type: Type[LoaderStrategy] | None, strategy_key: Tuple[Any, ...])
```

```py
sqlalchemy.orm.exc.NO_STATE = (<class 'AttributeError'>, <class 'KeyError'>)
```

可能由仪器实现引发的异常类型。

```py
exception sqlalchemy.orm.exc.ObjectDeletedError
```

刷新操作未能检索到与对象已知主键标识符对应的数据库行。

当在对象上访问过期属性或使用`Query.get()`检索到被检测为过期的对象时，刷新操作会进行。基于主键发出目标行的 SELECT；如果没有返回行，则引发此异常。

这个异常的真正含义只是与持久对象关联的主键标识符对应的行不存在。该行可能已被删除，或在某些情况下，主键已更新为新值，超出了 ORM 对目标对象的管理。

**类签名**

类`sqlalchemy.orm.exc.ObjectDeletedError` (`sqlalchemy.exc.InvalidRequestError`)

```py
method __init__(state: InstanceState[Any], msg: str | None = None)
```

```py
exception sqlalchemy.orm.exc.ObjectDereferencedError
```

由于对象被垃圾回收而无法完成操作。

**类签名**

类`sqlalchemy.orm.exc.ObjectDereferencedError`（`sqlalchemy.exc.SQLAlchemyError`）

```py
exception sqlalchemy.orm.exc.StaleDataError
```

遇到了未被考虑到的数据库状态操作。

导致此情况发生的条件包括：

+   一个刷新可能已尝试更新或删除行，并且在 UPDATE 或 DELETE 语句期间匹配到了意外数量的行。请注意，当使用 version_id_col 时，UPDATE 或 DELETE 语句中的行也将与当前已知的版本标识符匹配。

+   一个带有 version_id_col 的映射对象被刷新，而从数据库返回的版本号与对象本身的版本号不匹配。

+   一个对象从其父对象中分离出来，然而该对象以前附加到了另一个父标识，该父标识已被垃圾收集，并且无法确定新的父标识是否真的是最新的“父”。

**类签名**

类`sqlalchemy.orm.exc.StaleDataError`（`sqlalchemy.exc.SQLAlchemyError`）

```py
exception sqlalchemy.orm.exc.UnmappedClassError
```

请求了一个未知类的映射操作。

**类签名**

类`sqlalchemy.orm.exc.UnmappedClassError`（`sqlalchemy.orm.exc.UnmappedError`）

```py
method __init__(cls: Type[_T], msg: str | None = None)
```

```py
exception sqlalchemy.orm.exc.UnmappedColumnError
```

请求了一个未知列的映射操作。

**类签名**

类`sqlalchemy.orm.exc.UnmappedColumnError`（`sqlalchemy.exc.InvalidRequestError`）

```py
exception sqlalchemy.orm.exc.UnmappedError
```

引发涉及未出现预期映射的异常的基类。

**类签名**

类`sqlalchemy.orm.exc.UnmappedError`（`sqlalchemy.exc.InvalidRequestError`）

```py
exception sqlalchemy.orm.exc.UnmappedInstanceError
```

请求了一个未知实例的映射操作。

**类签名**

类`sqlalchemy.orm.exc.UnmappedInstanceError`（`sqlalchemy.orm.exc.UnmappedError`）

```py
method __init__(obj: object, msg: str | None = None)
```
