# SQL 表达语言基础构造

> 原文：[`docs.sqlalchemy.org/en/20/core/foundation.html`](https://docs.sqlalchemy.org/en/20/core/foundation.html)

用于组成 SQL 表达语言元素的基类和混合类。

| 对象名称 | 描述 |
| --- | --- |
| CacheKey | 用于在 SQL 编译缓存中标识 SQL 语句构造的键。 |
| ClauseElement | 用于程序化构建 SQL 表达式的元素的基类。 |
| DialectKWArgs | 建立一个类具有特定方言参数的能力，带有默认值和构造函数验证。 |
| HasCacheKey | 用于能够生成缓存键的对象的混合类。 |
| LambdaElement | 一个 SQL 构造，其中状态存储为未调用的 lambda。 |
| StatementLambdaElement | 代表一个可组合的 SQL 语句，作为`LambdaElement`。 |

```py
class sqlalchemy.sql.expression.CacheKey
```

用于在 SQL 编译缓存中标识 SQL 语句构造的键。

另请参阅

SQL 编译缓存

**成员**

bindparams, key, to_offline_string()

**类签名**

类`sqlalchemy.sql.expression.CacheKey` (`builtins.tuple`)

```py
attribute bindparams: Sequence[BindParameter[Any]]
```

字段编号 1 的别名

```py
attribute key: Tuple[Any, ...]
```

字段编号 0 的别名

```py
method to_offline_string(statement_cache: MutableMapping[Any, str], statement: ClauseElement, parameters: _CoreSingleExecuteParams) → str
```

生成这个`CacheKey`的“离线字符串”形式

“离线字符串”基本上是语句的字符串 SQL 加上一系列绑定参数值的 repr。而`CacheKey`对象依赖于内存中的标识以便作为缓存键工作，“离线”版本适用于其他进程也能工作的缓存。

给定的`statement_cache`是一个类似字典的对象，其中语句本身的字符串形式将被缓存。为了减少字符串化语句所花费的时间，这个字典应该在一个更长寿命的范围内。

```py
class sqlalchemy.sql.expression.ClauseElement
```

用于程序化构建 SQL 表达式的元素的基类。

**成员**

compare(), compile(), get_children(), inherit_cache, params(), self_group(), unique_params()

**类签名**

类`sqlalchemy.sql.expression.ClauseElement`（`sqlalchemy.sql.annotation.SupportsWrappingAnnotations`、`sqlalchemy.sql.cache_key.MemoizedHasCacheKey`、`sqlalchemy.sql.traversals.HasCopyInternals`、`sqlalchemy.sql.visitors.ExternallyTraversible`、`sqlalchemy.sql.expression.CompilerElement`）

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

将此`ClauseElement`与给定的`ClauseElement`进行比较。

子类应该覆盖默认行为，即直接进行身份比较。

**kw 是子类`compare()`方法消耗的参数，可用于修改比较的标准（参见`ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*从* `CompilerElement` *的* `CompilerElement.compile()` *方法继承*

编译此 SQL 表达式。

返回值是一个`Compiled`对象。对返回值调用`str()`或`unicode()`将产生结果的字符串表示。`Compiled`对象还可以使用`params`访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 一个`Connection`或`Engine`，它可以提供一个`Dialect`以生成一个`Compiled`对象。如果`bind`和`dialect`参数都被省略，将使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，一个应该存在于编译后语句的 VALUES 子句中的列名列表。如果为`None`，则从目标表对象中渲染所有列。

+   `dialect` – 一个`Dialect`实例，可以生成一个`Compiled`对象。此参数优先于`bind`参数。

+   `compile_kwargs` –

    额外参数的可选字典，这些参数将通过所有“访问”方法传递给编译器。这允许通过到自定义编译结构的任何自定义标志进行传递。它还用于通过以下方式传递 `literal_binds` 标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

我如何将 SQL 表达式呈现为字符串，可能还包含内联的绑定参数？

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*从* `HasTraverseInternals.get_children()` *方法继承* `HasTraverseInternals`

返回此 `HasTraverseInternals` 的直接子元素 `HasTraverseInternals`。

这用于访问遍历。

**kw 可包含更改返回集合的标志，例如返回子集以减少较大的遍历，或从不同上下文（例如模式级集合而不是子句级）返回子项的标志。

```py
attribute inherit_cache: bool | None = None
```

*从* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性继承*

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

属性默认为 `None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为 `False`，只是还会发出警告。

如果对应于对象的 SQL 不根据此类的本地属性（而不是其超类）更改，则可以在特定类上将此标志设置为 `True`。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method params(_ClauseElement__optionaldict: Mapping[str, Any] | None = None, **kwargs: Any) → Self
```

返回一个副本，其中 `bindparam()` 元素已被替换。

返回此 ClauseElement 的副本，并用从给定字典中取出的值替换其中的 `bindparam()` 元素：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

对此 `ClauseElement` 应用‘分组’。

子类重写此方法以返回一个“分组”构造，即括号。特别是当“二进制”表达式被放置到更大的表达式中时，它们会提供一个围绕自身的分组，以及当 `select()` 构造被放置到另一个 `select()` 的 FROM 子句中时。 （请注意，子查询通常应该使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句必须被命名）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在像 `x OR (y AND z)` 这样的表达式中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基础 `self_group()` 方法只返回自身。

```py
method unique_params(_ClauseElement__optionaldict: Dict[str, Any] | None = None, **kwargs: Any) → Self
```

返回一个将 `bindparam()` 元素替换的副本。

与 `ClauseElement.params()` 相同的功能，只是对影响到的绑定参数添加了 unique=True，以便可以使用多个语句。

```py
class sqlalchemy.sql.base.DialectKWArgs
```

建立类具有方言特定参数的能力，并具有默认值和构造函数验证。

`DialectKWArgs` 与方言上的 `DefaultDialect.construct_arguments` 交互。

**成员**

argument_for(), dialect_kwargs, dialect_options, kwargs

另请参阅

`DefaultDialect.construct_arguments`

```py
classmethod argument_for(dialect_name, argument_name, default)
```

为这个类添加一种新的方言特定的关键字参数。

例如：

```py
Index.argument_for("mydialect", "length", None)

some_index = Index('a', 'b', mydialect_length=5)
```

`DialectKWArgs.argument_for()`方法是一种逐个参数地向`DefaultDialect.construct_arguments`字典添加额外参数的方式。该字典提供了各种模式级构造的方言接受的参数名称列表。

新方言通常应一次性指定该字典作为方言类的数据成员。通常情况下，用于临时添加参数名称的用例是为了终端用户代码，该代码还使用了消耗额外参数的自定义编译方案。

参数：

+   `dialect_name` – 方言的名称。方言必须是可定位的，否则会引发`NoSuchModuleError`。方言还必须包含一个现有的`DefaultDialect.construct_arguments`集合，指示其参与关键字参数验证和默认系统，否则会引发`ArgumentError`。如果方言不包含此集合，则已经可以为该方言指定任何关键字参数。SQLAlchemy 内置的所有方言都包含此集合，但对于第三方方言，支持可能有所不同。

+   `argument_name` – 参数的名称。

+   `default` – 参数的默认值。

```py
attribute dialect_kwargs
```

作为特定方言选项指定的关键字参数集合。

这里的参数以其原始的`<dialect>_<kwarg>`格式呈现。只包括实际传递的参数；不同于`DialectKWArgs.dialect_options`集合，后者包含了该方言已知的所有选项，包括默认值。

该集合也是可写的；接受形式为`<dialect>_<kwarg>`的键，其值将被组装到选项列表中。

另请参阅

`DialectKWArgs.dialect_options` - 嵌套字典形式

```py
attribute dialect_options
```

作为特定方言选项指定的关键字参数集合。

这是一个两级嵌套的注册表，以`<dialect_name>`和`<argument_name>`为键。例如，`postgresql_where`参数可以定位为：

```py
arg = my_object.dialect_options['postgresql']['where']
```

0.9.2 版本中新增。

另请参阅

`DialectKWArgs.dialect_kwargs` - 扁平字典形式

```py
attribute kwargs
```

`DialectKWArgs.dialect_kwargs` 的别名。

```py
class sqlalchemy.sql.traversals.HasCacheKey
```

用于可以生成缓存键的对象的混合类。

此类通常位于以 `HasTraverseInternals` 为基础的层次结构中，但这是可选的。目前，该类应该能够在不包括 `HasTraverseInternals` 的情况下独立工作。

**成员**

inherit_cache

另请参阅

`CacheKey`

SQL 编译缓存

```py
attribute inherit_cache: bool | None = None
```

表明此 `HasCacheKey` 实例是否应该使用其直接超类使用的缓存键生成方案。

该属性默认为 `None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为 `False`，除了还会发出警告。

如果对象对应的 SQL 不基于仅限于此类而非其超类的属性发生变化，则可以在特定类上将此标志设置为 `True`。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的通用指南。

```py
class sqlalchemy.sql.expression.LambdaElement
```

一个 SQL 构造，其中状态被存储为未调用的 lambda。

`LambdaElement` 在将 lambda 表达式传递给 SQL 构造时会透明地生成，例如：

```py
stmt = select(table).where(lambda: table.c.col == parameter)
```

`LambdaElement` 是 `StatementLambdaElement` 的基础，它代表了 lambda 中的完整语句。

新版本 1.4 中新增。

另请参阅

使用 Lambdas 为语句生成带来显著的速度提升

**类签名**

类 `sqlalchemy.sql.expression.LambdaElement` (`sqlalchemy.sql.expression.ClauseElement`)

```py
class sqlalchemy.sql.expression.StatementLambdaElement
```

将可组合的 SQL 语句表示为 `LambdaElement`。

使用 `lambda_stmt()` 函数构建 `StatementLambdaElement`：

```py
from sqlalchemy import lambda_stmt

stmt = lambda_stmt(lambda: select(table))
```

构建完成后，可以通过添加后续 lambda 将额外条件添加到语句中，这些 lambda 将现有语句对象作为单个参数接受：

```py
stmt += lambda s: s.where(table.c.col == parameter)
```

版本 1.4 中的新功能。

另请参阅

使用 Lambda 添加显著的语句生成速度提升

**成员**

add_criteria(), is_delete, is_dml, is_insert, is_select, is_text, is_update, spoil()

**类签名**

类 `sqlalchemy.sql.expression.StatementLambdaElement` (`sqlalchemy.sql.roles.AllowsLambdaRole`, `sqlalchemy.sql.expression.LambdaElement`, `sqlalchemy.sql.expression.Executable`)

```py
method add_criteria(other: Callable[[Any], Any], enable_tracking: bool = True, track_on: Any | None = None, track_closure_variables: bool = True, track_bound_values: bool = True) → StatementLambdaElement
```

向此 `StatementLambdaElement` 添加新条件。

例如：

```py
>>> def my_stmt(parameter):
...     stmt = lambda_stmt(
...         lambda: select(table.c.x, table.c.y),
...     )
...     stmt = stmt.add_criteria(
...         lambda: table.c.x > parameter
...     )
...     return stmt
```

`StatementLambdaElement.add_criteria()` 方法等同于使用 Python 加法运算符添加新的 lambda，不过可以添加额外的参数，包括 `track_closure_values` 和 `track_on`：

```py
>>> def my_stmt(self, foo):
...     stmt = lambda_stmt(
...         lambda: select(func.max(foo.x, foo.y)),
...         track_closure_variables=False
...     )
...     stmt = stmt.add_criteria(
...         lambda: self.where_criteria,
...         track_on=[self]
...     )
...     return stmt
```

有关可接受参数的说明，请参阅 `lambda_stmt()`。

```py
attribute is_delete
```

```py
attribute is_dml
```

```py
attribute is_insert
```

```py
attribute is_select
```

```py
attribute is_text
```

```py
attribute is_update
```

```py
method spoil() → NullLambdaStatement
```

返回一个新的 `StatementLambdaElement`，每次运行所有 lambda 时都会无条件地运行。
