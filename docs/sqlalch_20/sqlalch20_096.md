# 基本类型 API

> 原文：[`docs.sqlalchemy.org/en/20/core/type_api.html`](https://docs.sqlalchemy.org/en/20/core/type_api.html)

| 对象名称 | 描述 |
| --- | --- |
| Concatenable | 标记类型支持“串联”的混合类型，通常用于字符串。 |
| ExternalType | 定义特定于第三方数据类型的属性和行为的混合类型。 |
| Indexable | 标记类型支持索引操作的混合类型，例如数组或 JSON 结构。 |
| NullType | 未知类型。 |
| TypeEngine | 所有 SQL 数据类型的最终基类。 |
| Variant | 不推荐使用。此符号用于向后兼容性，但实际上不应使用此类型。 |

```py
class sqlalchemy.types.TypeEngine
```

所有 SQL 数据类型的最终基类。

`TypeEngine` 的常见子类包括 `String`、`Integer` 和 `Boolean`。

有关 SQLAlchemy 类型系统的概述，请参见 SQL 数据类型对象。

另请参阅

SQL 数据类型对象

**成员**

operate(), reverse_operate(), adapt(), as_generic(), bind_expression(), bind_processor(), coerce_compared_value(), column_expression(), comparator_factory, compare_values(), compile(), dialect_impl(), evaluates_none(), get_dbapi_type(), hashable, literal_processor(), python_type, render_bind_cast, render_literal_cast, result_processor(), should_evaluate_none, sort_key_function, with_variant()

**类签名**

类`sqlalchemy.types.TypeEngine` (`sqlalchemy.sql.visitors.Visitable`, `typing.Generic`)

```py
class Comparator
```

自定义比较操作在类型级别定义的基类。请参见`TypeEngine.comparator_factory`。

**类签名**

类`sqlalchemy.types.TypeEngine.Comparator` (`sqlalchemy.sql.expression.ColumnOperators`, `typing.Generic`)

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[_CT]
```

对参数进行操作。

这是操作的最低级别，默认情况下引发`NotImplementedError`。

在子类上重写此方法可以使常见行为适用于所有操作。例如，重写`ColumnOperators`以对左侧和右侧应用`func.lower()`：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的“其他”一侧。对于大多数操作，它将是一个单一的标量。

+   `**kwargs` – 修饰符。这些可能由特殊运算符传递，如`ColumnOperators.contains()`。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[_CT]
```

对参数进行反向操作。

使用方法与`operate()`相同。

```py
method adapt(cls: Type[TypeEngine | TypeEngineMixin], **kw: Any) → TypeEngine
```

给定一个“impl”类来处理，产生此类型的“适配”形式。

此方法在内部用于将通用类型与特定于特定方言的“实现”类型关联起来。

```py
method as_generic(allow_nulltype: bool = False) → TypeEngine
```

使用启发式规则返回此类型对应的通用类型的实例。如果此启发式规则不足以满足需求，则可以重写此方法。

```py
>>> from sqlalchemy.dialects.mysql import INTEGER
>>> INTEGER(display_width=4).as_generic()
Integer()
```

```py
>>> from sqlalchemy.dialects.mysql import NVARCHAR
>>> NVARCHAR(length=100).as_generic()
Unicode(length=100)
```

在 1.4.0b2 版本中新增。

另请参见

使用与数据库无关的类型反射 - 描述了`TypeEngine.as_generic()`与`DDLEvents.column_reflect()`事件结合使用的情况，这是其预期的用法。

```py
method bind_expression(bindvalue: BindParameter[_T]) → ColumnElement[_T] | None
```

给定一个绑定值（即一个`BindParameter`实例），返回其位置的 SQL 表达式。

这通常是一个 SQL 函数，用于在语句中包装现有的绑定参数。它用于特殊的数据类型，这些类型需要将文本在某些特殊数据库函数中包装，以便将应用程序级值强制转换为数据库特定格式。它是`TypeEngine.bind_processor()`方法的 SQL 模拟。 

此方法在语句的**SQL 编译**阶段调用，用于呈现 SQL 字符串。它**不**针对特定值调用。

请注意，当实现此方法时，应始终返回完全相同的结构，不带任何条件逻辑，因为它可能在针对任意数量的绑定参数集的 executemany()调用中使用。

注意

此方法仅针对**特定方言类型对象**，通常**私有于正在使用的方言**，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_expression()`方法，除非明确地子类化`UserDefinedType`类。

要为`TypeEngine.bind_expression()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.bind_expression()`的实现。

另请参见

增强现有类型

另请参见

应用 SQL 级别的绑定/结果处理

```py
method bind_processor(dialect: Dialect) → _BindProcessorType[_T] | None
```

返回一个转换函数以处理绑定值。

返回一个可调用对象，该对象将接收一个绑定参数值作为唯一的位置参数，并返回一个要发送到 DB-API 的值。

如果不需要处理，则该方法应返回`None`。

注意

此方法仅针对**特定方言类型对象**，通常**私有于正在使用的方言**，并且不是公共类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.bind_processor()`方法，除非明确地子类化`UserDefinedType`类。

要为`TypeEngine.bind_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_bind_param()`的实现。

另请参见

增强现有类型

参数：

**方言** – 正在使用的方言实例。

```py
method coerce_compared_value(op: OperatorType | None, value: Any) → TypeEngine[Any]
```

建议为表达式中的‘强制’Python 值提供一种类型。

给定运算符和值，让类型有机会返回一个应该将值强制转换为的类型。

这里的默认行为是保守的；如果右侧已经根据其 Python 类型被强制转换为 SQL 类型，则通常会被保留。

这里的最终用户功能扩展通常应通过`TypeDecorator`完成，它提供更自由的行为，因为它默认将表达式的另一侧强制转换为此类型，从而应用特殊的 Python 转换，超出了 DBAPI 所需的范围。它还提供了公共方法`TypeDecorator.coerce_compared_value()`，用于最终用户定制此行为。

```py
method column_expression(colexpr: ColumnElement[_T]) → ColumnElement[_T] | None
```

给定 SELECT 列表达式，返回包装的 SQL 表达式。

这通常是一个 SQL 函数，它将列表达式包装为在 SELECT 语句的 columns 子句中呈现的形式。它用于特殊数据类型，这些数据类型要求列在发送回应用程序之前必须被包装在某些特殊的数据库函数中以强制转换值。它是 SQL 中`TypeEngine.result_processor()`方法的类比。

此方法在语句的**SQL 编译**阶段调用，当呈现 SQL 字符串时。它**不会**针对特定值调用。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常是当前正在使用的方言的**私有类型**，并且不是公共类型对象，这意味着不可行通过子类化`TypeEngine`类来提供替代`TypeEngine.column_expression()`方法，除非显式子类化`UserDefinedType`类。

要为`TypeEngine.column_expression()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.column_expression()`的实现。

另见

增强现有类型

另见

应用 SQL 级别的绑定/结果处理

```py
attribute comparator_factory
```

`Comparator` 的别名

```py
method compare_values(x: Any, y: Any) → bool
```

比较两个值是否相等。

```py
method compile(dialect: Dialect | None = None) → str
```

生成此 `TypeEngine` 的字符串编译形式。

当不带参数调用时，使用“默认”方言来生成字符串结果。

参数：

**dialect** – 一个 `Dialect` 实例。

```py
method dialect_impl(dialect: Dialect) → TypeEngine[_T]
```

返回此 `TypeEngine` 的特定于方言的实现。

```py
method evaluates_none() → Self
```

返回具有 `should_evaluate_none` 标志设置为 True 的此类型的副本。

例如：

```py
Table(
    'some_table', metadata,
    Column(
        String(50).evaluates_none(),
        nullable=True,
        server_default='no value')
)
```

ORM 使用此标志来指示在 INSERT 语句中传递 `None` 的正值到列中，而不是省略列从 INSERT 语句中，这将触发列级默认值的效果。它还允许具有与 Python None 值关联的特殊行为的类型来指示该值不一定转换为 SQL NULL；一个典型的例子是可能希望持久化 JSON 值 `'null'` 的 JSON 类型。

在所有情况下，实际的 NULL SQL 值都可以通过在 INSERT 语句中使用 `null` SQL 构造或与 ORM 映射的属性相关联来始终持久化在任何列中。

注

“evaluates none” 标志 **不** 适用于传递给 `Column.default` 或 `Column.server_default` 的 `None` 值；在这些情况下，`None` 仍然表示“无默认值”。

另请参阅

强制具有默认值的列为 NULL - 在 ORM 文档中

`JSON.none_as_null` - 具有此标志的 PostgreSQL JSON 交互。

`TypeEngine.should_evaluate_none` - 类级标志

```py
method get_dbapi_type(dbapi: module) → Any | None
```

返回底层 DB-API 的相应类型对象，如果有的话。

例如，这对于调用 `setinputsizes()` 可能很有用。

```py
attribute hashable = True
```

如果 False，则表示来自此类型的值不可哈希。

在 ORM 列表结果去重时使用。

```py
method literal_processor(dialect: Dialect) → _LiteralProcessorType[_T] | None
```

返回一个转换函数，用于处理直接渲染而不使用绑定的文字值。

当编译器使用“literal_binds”标志时使用此函数，通常用于 DDL 生成以及在某些后端不接受绑定参数的情况下。

返回一个可调用对象，该对象将接收一个字面的 Python 值作为唯一的位置参数，并返回一个字符串表示以在 SQL 语句中呈现。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常是**正在使用的方言的私有对象**，并不是公共类型对象，这意味着不太可能为了提供替代的`TypeEngine.literal_processor()`方法而对`TypeEngine`类进行子类化，除非显式地对`UserDefinedType`类进行子类化。

要为`TypeEngine.literal_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_literal_param()`的实现。

另请参阅

增强现有类型

```py
attribute python_type
```

返回此类型实例预计返回的 Python 类型对象，如果已知的话。

基本上，对于那些强制指定返回类型的类型，或者已知在所有常见的 DBAPI 中都会对所有类型进行这样的操作的类型（例如`int`），将返回该类型。

如果未定义返回类型，则引发`NotImplementedError`。

注意，在 SQL 中，任何类型也可以容纳 NULL，这意味着你在实践中也可以从任何类型中获得`None`。

```py
attribute render_bind_cast = False
```

渲染绑定转换以适应`BindTyping.RENDER_CASTS`模式。

如果为 True，则此类型（通常是一个方言级别的实现类型）向编译器发出信号，表示应该在此类型的绑定参数周围呈现一个转换。

2.0 版本中的新功能。

另请参阅

`BindTyping`

```py
attribute render_literal_cast = False
```

渲染转换当渲染一个值作为内联字面值时，例如通过`TypeEngine.literal_processor()`。

2.0 版本中的新功能。

```py
method result_processor(dialect: Dialect, coltype: object) → _ResultProcessorType[_T] | None
```

返回一个用于处理结果行值的转换函数。

返回一个可调用对象，该对象将接收一个结果行列值作为唯一的位置参数，并返回一个要返回给用户的值。

如果不需要处理，则方法应返回`None`。

注意

此方法仅相对于**特定方言类型对象**调用，该对象通常是**正在使用的方言私有的**，并且不是与公共类型对象相同的类型对象，这意味着无法通过子类化`TypeEngine`类来提供替代的`TypeEngine.result_processor()`方法，除非明确子类化`UserDefinedType`类。

要为`TypeEngine.result_processor()`提供替代行为，请实现一个`TypeDecorator`类，并提供一个`TypeDecorator.process_result_value()`的实现。

另请参见

增强现有类型

参数：

+   `dialect` – 正在使用的方言实例。

+   `coltype` – 在`cursor.description`中接收到的 DBAPI coltype 参数。

```py
attribute should_evaluate_none: bool = False
```

如果为 True，则 Python 常量`None`被认为是此类型明确处理的。

ORM 使用此标志表示在 INSERT 语句中将正值的`None`传递给列，而不是从 INSERT 语句中省略列，这会触发列级默认值。它还允许具有 Python None 的特殊行为的类型（例如 JSON 类型）明确指示它们希望显式处理 None 值。

要在现有类型上设置此标志，请使用`TypeEngine.evaluates_none()`方法。

另请参见

`TypeEngine.evaluates_none()`

```py
attribute sort_key_function: Callable[[Any], Any] | None = None
```

作为传递给 sorted 的键的排序函数。

默认值为`None`表示此类型存储的值是自动排序的。

1.3.8 版中的新功能。

```py
method with_variant(type_: _TypeEngineArgument[Any], *dialect_names: str) → Self
```

生成此类型对象的副本，该副本将在应用于给定名称的方言时使用给定的类型。

例如：

```py
from sqlalchemy.types import String
from sqlalchemy.dialects import mysql

string_type = String()

string_type = string_type.with_variant(
    mysql.VARCHAR(collation='foo'), 'mysql', 'mariadb'
)
```

变体映射表示当特定方言解释此类型时，它将被转换为给定类型，而不是使用主类型。

从版本 2.0 开始更改：`TypeEngine.with_variant()`方法现在在“就地”操作`TypeEngine`对象时工作，返回原始类型的副本，而不是返回包装对象；不再使用`Variant`类。

参数：

+   `type_` – 一个`TypeEngine`，当使用给定名称的方言时，将从原始类型中选择作为变体。

+   `*dialect_names` –

    使用此类型的方言的一个或多个基本名称（即`'postgresql'`，`'mysql'`等）

    从 2.0 版本开始更改：可以为一个变体指定多个方言名称。

另请参见

使用“大写字母”和后端特定类型的多后端 - 演示了`TypeEngine.with_variant()`的使用。

```py
class sqlalchemy.types.Concatenable
```

一个将类型标记为支持“连接”的 mixin，通常是字符串。

**成员**

comparator_factory

**类签名**

类`sqlalchemy.types.Concatenable`（`sqlalchemy.types.TypeEngineMixin`）

```py
class Comparator
```

**类签名**

类`sqlalchemy.types.Concatenable.Comparator`（`sqlalchemy.types.Comparator`）

```py
attribute comparator_factory
```

`Comparator`的别名

```py
class sqlalchemy.types.Indexable
```

一个将类型标记为支持索引操作的 mixin，例如数组或 JSON 结构。

**成员**

comparator_factory

**类签名**

类`sqlalchemy.types.Indexable`（`sqlalchemy.types.TypeEngineMixin`）

```py
class Comparator
```

**类签名**

类`sqlalchemy.types.Indexable.Comparator`（`sqlalchemy.types.Comparator`）

```py
attribute comparator_factory
```

`Comparator`的别名

```py
class sqlalchemy.types.NullType
```

一个未知类型。

`NullType`用作那些无法确定类型的情况的默认类型，包括：

+   在表反射期间，当列的类型未被`Dialect`识别时

+   当使用未知类型的纯 Python 对象构建 SQL 表达式时（例如`somecolumn == my_special_object`）

+   当创建新的`Column`时，并且给定类型传递为`None`或根本不传递时。

`NullType` 可以在 SQL 表达式调用中使用，没有问题，只是在表达式构造级别或绑定参数/结果处理级别上没有行为。

**类签名**

class `sqlalchemy.types.NullType` (`sqlalchemy.types.TypeEngine`)

```py
class sqlalchemy.types.ExternalType
```

定义特定于第三方数据类型的属性和行为的混合项。

“第三方”指的是在 SQLAlchemy 范围之外定义的数据类型，在最终用户应用代码中或在 SQLAlchemy 的外部扩展中定义。

当前的子类包括 `TypeDecorator` 和 `UserDefinedType`。

版本 1.4.28 中的新功能。

**成员**

cache_ok

**类签名**

class `sqlalchemy.types.ExternalType` (`sqlalchemy.types.TypeEngineMixin`)

```py
attribute cache_ok: bool | None = None
```

表示使用此 `ExternalType` 的语句是否“可缓存”。

默认值 `None` 将发出警告，然后不允许包含此类型的语句进行缓存。设置为 `False` 以完全禁用使用此类型的语句的缓存而不发出警告。当设置为 `True` 时，对象的类和其状态的选定元素将用作缓存键的一部分。例如，使用 `TypeDecorator`：

```py
class MyType(TypeDecorator):
    impl = String

    cache_ok = True

    def __init__(self, choices):
        self.choices = tuple(choices)
        self.internal_only = True
```

上述类型的缓存键将等效于：

```py
>>> MyType(["a", "b", "c"])._static_cache_key
(<class '__main__.MyType'>, ('choices', ('a', 'b', 'c')))
```

缓存方案将从与 `__init__()` 方法中的参数名称对应的类型中提取属性。上面的 “choices” 属性成为缓存键的一部分，但 “internal_only” 不是，因为没有名为 “internal_only” 的参数。

可缓存元素的要求是它们是可哈希的，并且它们指示对于给定缓存值的表达式每次使用相同的 SQL 渲染。

为了适应引用不可哈希结构（如字典、集合和列表）的数据类型，可以通过将可哈希结构分配给其属性来使这些对象“可缓存”，其名称与参数的名称对应。例如，接受查找值字典的数据类型可以将其发布为排序的元组序列。给定先前不可缓��的类型为：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 this is the non-cacheable version, as "self.lookup" is not
 hashable.

 '''

    def __init__(self, lookup):
        self.lookup = lookup

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self.lookup" ...
```

其中“lookup”是一个字典。该类型将无法生成缓存键：

```py
>>> type_ = LookupType({"a": 10, "b": 20})
>>> type_._static_cache_key
<stdin>:1: SAWarning: UserDefinedType LookupType({'a': 10, 'b': 20}) will not
produce a cache key because the ``cache_ok`` flag is not set to True.
Set this flag to True if this type object's state is safe to use
in a cache key, or False to disable this warning.
symbol('no_cache')
```

如果我们**设置**了这样一个缓存键，它将无法使用。我们将得到一个包含字典的元组结构，该字典本身不能作为“缓存字典”中的键使用，因为 Python 字典不可哈希：

```py
>>> # set cache_ok = True
>>> type_.cache_ok = True

>>> # this is the cache key it would generate
>>> key = type_._static_cache_key
>>> key
(<class '__main__.LookupType'>, ('lookup', {'a': 10, 'b': 20}))

>>> # however this key is not hashable, will fail when used with
>>> # SQLAlchemy statement cache
>>> some_cache = {key: "some sql value"}
Traceback (most recent call last): File "<stdin>", line 1,
in <module> TypeError: unhashable type: 'dict'
```

通过将排序的元组元组分配给“.lookup”属性，可以使类型可缓存：

```py
class LookupType(UserDefinedType):
  '''a custom type that accepts a dictionary as a parameter.

 The dictionary is stored both as itself in a private variable,
 and published in a public variable as a sorted tuple of tuples,
 which is hashable and will also return the same value for any
 two equivalent dictionaries.  Note it assumes the keys and
 values of the dictionary are themselves hashable.

 '''

    cache_ok = True

    def __init__(self, lookup):
        self._lookup = lookup

        # assume keys/values of "lookup" are hashable; otherwise
        # they would also need to be converted in some way here
        self.lookup = tuple(
            (key, lookup[key]) for key in sorted(lookup)
        )

    def get_col_spec(self, **kw):
        return "VARCHAR(255)"

    def bind_processor(self, dialect):
        # ...  works with "self._lookup" ...
```

在上述情况下，`LookupType({"a": 10, "b": 20})`的缓存键将是：

```py
>>> LookupType({"a": 10, "b": 20})._static_cache_key
(<class '__main__.LookupType'>, ('lookup', (('a', 10), ('b', 20))))
```

新版本 1.4.14 中：- 添加了`cache_ok`标志，允许对`TypeDecorator`类进行一些缓存配置。

新版本 1.4.28 中：- 添加了`ExternalType`混合类型，将`cache_ok`标志泛化到`TypeDecorator`和`UserDefinedType`类。

另请参阅

SQL 编译缓存

```py
class sqlalchemy.types.Variant
```

已弃用。符号用于向后兼容解决方案配方，但不应使用此实际类型。

**成员**

with_variant()

**类签名**

类`sqlalchemy.types.Variant`（`sqlalchemy.types.TypeDecorator`）

```py
method with_variant(type_: _TypeEngineArgument[Any], *dialect_names: str) → Self
```

*继承自* `TypeEngine.with_variant()` *方法的* `TypeEngine`

生成将在应用于给定名称的方言时利用给定类型的此类型对象的副本。

例如：

```py
from sqlalchemy.types import String
from sqlalchemy.dialects import mysql

string_type = String()

string_type = string_type.with_variant(
    mysql.VARCHAR(collation='foo'), 'mysql', 'mariadb'
)
```

变体映射表示，当特定方言解释此类型时，它将被转换为给定类型，而不是使用主要类型。

版本 2.0 中更改：`TypeEngine.with_variant()`方法现在与`TypeEngine`对象“原地”工作，返回原始类型的副本，而不是返回包装对象；不再使用`Variant`类。

参数：

+   `type_` – 一个 `TypeEngine`，当使用给定名称的方言时，将从原始类型中选择作为变体的类型。

+   `*dialect_names*` –

    使用此类型的方言的一个或多个基本名称。（即 `'postgresql'`，`'mysql'` 等）

    版本 2.0 中的变更：一个变体可以指定多个方言名称。

另请参阅

使用“大写”和后端特定类型进行多后端处理 - 演示了 `TypeEngine.with_variant()` 的使用。
