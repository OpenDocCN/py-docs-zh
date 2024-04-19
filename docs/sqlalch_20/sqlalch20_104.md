# 运行时检查 API

> 原文：[`docs.sqlalchemy.org/en/20/core/inspection.html`](https://docs.sqlalchemy.org/en/20/core/inspection.html)

检查模块提供了`inspect()`函数，该函数提供了关于 SQLAlchemy 对象的运行时信息，包括核心部分和 ORM 部分。

`inspect()`函数是访问 SQLAlchemy 公共 API 以查看内存对象配置和构建的入口点。根据传递给`inspect()`的对象类型，返回值将是提供已知接口的相关对象，或者在许多情况下将返回对象本身。

`inspect()`的理由是双重的。一是它取代了需要了解 SQLAlchemy 中大量“获取信息”的函数的需求，例如`Inspector.from_engine()`（在 1.4 中已弃用）、`instance_state()`、`class_mapper()`等。另一个是`inspect()`的返回值保证遵守文档化的 API，从而允许基于 SQLAlchemy 配置构建的第三方工具以向前兼容的方式构建。

| 对象名称 | 描述 |
| --- | --- |
| inspect(subject[, raiseerr]) | 为给定目标生成一个检查对象。 |

```py
function sqlalchemy.inspect(subject: Any, raiseerr: bool = True) → Any
```

为给定目标生成一个检查对象。

在某些情况下，返回的值可能与给定的对象相同，例如如果传递了一个`Mapper`对象。在其他情况下，它将是给定对象的注册检查类型的实例，例如如果传递了一个`Engine`，则返回一个`Inspector`对象。

参数：

+   `subject` – 要检查的主题。

+   `raiseerr` – 当为`True`时，如果给定主题不对应于已知的 SQLAlchemy 检查类型，则引发`sqlalchemy.exc.NoInspectionAvailable`。如果为`False`，则返回`None`。

## 可检查的目标

下面是许多常见检查目标的列表。

+   `Connectable`（即 `Engine`、`Connection`） - 返回一个 `Inspector` 对象。

+   `ClauseElement` - 所有 SQL 表达式组件，包括 `Table`、`Column`，都作为自己的检查对象，这意味着传递给 `inspect()` 的这些对象返回它们自身。

+   `object` - 给定一个对象，ORM 将检查其映射 - 如果是这样，将返回一个表示对象的映射状态的 `InstanceState`。`InstanceState` 还通过 `AttributeState` 接口提供对每个属性状态的访问，以及通过 `History` 对象提供对任何属性的每次 flush 的“历史”访问。

    请参见

    映射实例的检查

+   `type`（即类） - 由 ORM 给定的类将被检查映射 - 如果是这样，将返回该类的 `Mapper`。

    请参见

    Mapper 对象的检查

+   映射属性 - 将映射属性传递给 `inspect()`，例如 `inspect(MyClass.some_attribute)`，将返回一个 `QueryableAttribute` 对象，它是与映射类关联的 描述符。该描述符指的是一个 `MapperProperty`，通常是 `ColumnProperty` 或 `RelationshipProperty` 的实例，通过其 `QueryableAttribute.property` 属性。

+   `AliasedClass` - 返回一个 `AliasedInsp` 对象。

## 可用的检查目标

以下是许多常见检查目标的列表。

+   `Connectable`（即`Engine`，`Connection`） - 返回一个`Inspector`对象。

+   `ClauseElement` - 所有 SQL 表达式组件，包括`Table`，`Column`，都作为自己的检查对象，这意味着任何这些对象传递给`inspect()`都会返回它们自己。

+   `object` - 给定的对象将由 ORM 检查映射 - 如果是这样，将返回一个表示对象映射状态的`InstanceState`。`InstanceState`还通过`AttributeState`接口提供对每个属性状态的访问，以及通过`History`对象提供对任何属性的每次刷新“历史”的访问。

    另请参见

    映射实例的检查

+   `type`（即一个类） - 将给定的类检查 ORM 是否有映射 - 如果有，将返回该类的`Mapper`。

    另请参见

    Mapper 对象的检查

+   传递映射属性 - 将映射属性传递给`inspect()`，例如`inspect(MyClass.some_attribute)`，将返回一个`QueryableAttribute`对象，该对象是与映射类相关联的描述符。此描述符指向一个`MapperProperty`，通常是`ColumnProperty`或`RelationshipProperty`的实例，通过其`QueryableAttribute.property`属性。

+   `AliasedClass` - 返回一个 `AliasedInsp` 对象。
