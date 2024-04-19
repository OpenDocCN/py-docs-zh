# SQL 和通用函数

> 原文：[`docs.sqlalchemy.org/en/20/core/functions.html`](https://docs.sqlalchemy.org/en/20/core/functions.html)

通过使用 `func` 命名空间来调用 SQL 函数。请参阅 使用 SQL 函数 教程，了解如何使用 `func` 对象在语句中渲染 SQL 函数的背景知识。

另请参阅

使用 SQL 函数 - 在 SQLAlchemy 统一教程 中

## 函数 API

SQL 函数的基本 API，提供了 `func` 命名空间以及可用于可扩展性的类。

| 对象名称 | 描述 |
| --- | --- |
| AnsiFunction | 以“ansi”格式定义函数，不会渲染括号。 |
| Function | 描述一个命名的 SQL 函数。 |
| FunctionElement | SQL 函数导向构造的基类。 |
| GenericFunction | 定义一个‘通用’函数。 |
| register_function(identifier, fn[, package]) | 将可调用对象与特定函数名关联。 |

```py
class sqlalchemy.sql.functions.AnsiFunction
```

以“ansi”格式定义函数，不会渲染括号。

**类签名**

类 `sqlalchemy.sql.functions.AnsiFunction` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.Function
```

描述一个命名的 SQL 函数。

`Function` 对象通常由 `func` 生成对象生成。

参数：

+   `*clauses` – 形成 SQL 函数调用参数的列表达式列表。

+   `type_` – 可选的 `TypeEngine` 数据类型对象，将用作由此函数调用生成的列表达式的返回值。

+   `packagenames` –

    一个字符串，指示在生成 SQL 时要在函数名前添加的包前缀名称。当使用点格式调用 `func` 生成器时会创建这些，例如：

    ```py
    func.mypackage.some_function(col1, col2)
    ```

另请参阅

使用 SQL 函数 - 在 SQLAlchemy 统一教程 中

`func` - 产生注册或特设的 `Function` 实例的命名空间。

`GenericFunction` - 允许创建注册的函数类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.sql.functions.Function` (`sqlalchemy.sql.functions.FunctionElement`)

```py
method __init__(name: str, *clauses: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T] | None = None, packagenames: Tuple[str, ...] | None = None)
```

构造一个 `Function`。

通常使用 `func` 构造函数来构造新的 `Function` 实例。

```py
class sqlalchemy.sql.functions.FunctionElement
```

用于 SQL 函数导向构造的基础。

这是一个 [通用类型](https://peps.python.org/pep-0484/#generics)，意味着类型检查器和 IDE 可以指示在此函数的 `Result` 中期望的类型。参见 `GenericFunction` 以了解如何执行此操作的示例。

另请参阅

使用 SQL 函数 - 在 SQLAlchemy 统一教程 中

`Function` - SQL 函数的命名。

`func` - 产生注册或特设的 `Function` 实例的命名空间。

`GenericFunction` - 允许创建注册的函数类型。

**成员**

__init__(), alias(), as_comparison(), c, clauses, column_valued(), columns, entity_namespace, exported_columns, filter(), over(), scalar_table_valued(), select(), self_group(), table_valued(), within_group(), within_group_type()

**类签名**

类`sqlalchemy.sql.functions.FunctionElement`（`sqlalchemy.sql.expression.Executable`，`sqlalchemy.sql.expression.ColumnElement`，`sqlalchemy.sql.expression.FromClause`，`sqlalchemy.sql.expression.Generative`）

```py
method __init__(*clauses: _ColumnExpressionOrLiteralArgument[Any])
```

构建一个`FunctionElement`。

参数：

+   `*clauses` – 列表，包含形成 SQL 函数调用参数的列表达式。

+   `**kwargs` – 通常由子类消耗的额外 kwargs。

另请参阅

`func`

`Function`

```py
method alias(name: str | None = None, joins_implicitly: bool = False) → TableValuedAlias
```

对这个`FunctionElement`构建一个别名。

提示

`FunctionElement.alias()` 方法是创建“表值”SQL 函数的机制的一部分。但是，大多数用例都通过`FunctionElement`上的更高级方法来处理，包括`FunctionElement.table_valued()`和`FunctionElement.column_valued()`。

此结构将函数包装在适合 FROM 子句的命名别名中，例如 PostgreSQL 所接受的风格。 还提供了使用特殊的 `.column` 属性的列表达式，该属性可用于在列或 where 子句中引用函数的输出，例如 PostgreSQL 等后端的标量值。

对于完整的表值表达式，请先使用 `FunctionElement.table_valued()` 方法来建立命名列。

例如：

```py
>>> from sqlalchemy import func, select, column
>>> data_view = func.unnest([1, 2, 3]).alias("data_view")
>>> print(select(data_view.column))
SELECT  data_view
FROM  unnest(:unnest_1)  AS  data_view 
```

`FunctionElement.column_valued()` 方法为上述模式提供了一种快捷方式：

```py
>>> data_view = func.unnest([1, 2, 3]).column_valued("data_view")
>>> print(select(data_view))
SELECT  data_view
FROM  unnest(:unnest_1)  AS  data_view 
```

新版本 1.4.0b2 中添加了 `.column` 访问器

参数：

+   `name` – 别名，将在 FROM 子句中渲染为 `AS <name>`

+   `joins_implicitly` –

    当为 True 时，可以在 SQL 查询的 FROM 子句中使用表值函数，而无需显式连接到其他表，并且不会生成“笛卡尔积”警告。 对于 `func.json_each()` 等 SQL 函数可能很有用。

    新版本 1.4.33 中添加。

另请参阅

表值函数 - 在 SQLAlchemy Unified Tutorial 中

`FunctionElement.table_valued()`

`FunctionElement.scalar_table_valued()`

`FunctionElement.column_valued()`

```py
method as_comparison(left_index: int, right_index: int) → FunctionAsBinary
```

将此表达式解释为两个值之间的布尔比较。

此方法用于描述 Custom operators based on SQL functions 中的 ORM 用例。

假设的 SQL 函数“is_equal()”，用于比较两个值是否相等，可以用 Core 表达式语言编写为：

```py
expr = func.is_equal("a", "b")
```

如果上面的“is_equal()”是在比较“a”和“b”是否相等，那么`FunctionElement.as_comparison()`方法将被调用为：

```py
expr = func.is_equal("a", "b").as_comparison(1, 2)
```

在上面的例子中，“1”这个整数值是指“is_equal()”函数的第一个参数，“2”这个整数值是指第二个参数。

这将创建一个等效于`BinaryExpression`的表达式：

```py
BinaryExpression("a", "b", operator=op.eq)
```

但是，在 SQL 级别上，它仍然会呈现为“is_equal('a'，'b')”。

当 ORM 加载相关对象或集合时，需要能够操作 JOIN 表达式的 ON 子句的“left”和“right”两边。此方法的目的是在与`relationship.primaryjoin`参数一起使用时，为 ORM 提供也可以向其提供此信息的 SQL 函数构造。返回值是一个名为`FunctionAsBinary`的包含对象。

一个 ORM 示例如下：

```py
class Venue(Base):
    __tablename__ = 'venue'
    id = Column(Integer, primary_key=True)
    name = Column(String)

    descendants = relationship(
        "Venue",
        primaryjoin=func.instr(
            remote(foreign(name)), name + "/"
        ).as_comparison(1, 2) == 1,
        viewonly=True,
        order_by=name
    )
```

上面，“Venue”类可以通过确定父 Venue 的名称是否包含在假设的后代值的名称的开头来加载后代“Venue”对象，例如“parent1”将匹配到“parent1/child1”，但不会匹配到“parent2/child1”。

可能的用例包括上面给出的“材料化路径”示例，以及利用特殊的 SQL 函数（例如几何函数）创建连接条件。

参数：

+   `left_index` - 作为表达式“left”侧的函数参数的整数基于 1 的索引。

+   `right_index` - 作为表达式“right”侧的函数参数的整数基于 1 的索引。

版本 1.3 中的新内容。

另请参见

基于 SQL 函数的自定义运算符 - ORM 中的示例用法

```py
attribute c
```

`joins_implicitly` - `FunctionElement.columns`的同义词。

```py
attribute clauses
```

返回包含此`FunctionElement`参数的基础`ClauseList`。

```py
method column_valued(name: str | None = None, joins_implicitly: bool = False) → TableValuedColumn[_T]
```

将此`FunctionElement`作为选择自身的 FROM 子句的列表达式返回。

例如：

```py
>>> from sqlalchemy import select, func
>>> gs = func.generate_series(1, 5, -1).column_valued()
>>> print(select(gs))
SELECT  anon_1
FROM  generate_series(:generate_series_1,  :generate_series_2,  :generate_series_3)  AS  anon_1 
```

这是的简写形式：

```py
gs = func.generate_series(1, 5, -1).alias().column
```

参数：

+   `name` - 分配给生成的别名的可选名称。如果省略，将使用唯一的匿名名称。

+   `joins_implicitly` –

    当为 True 时，列值函数的“表”部分可以成为 SQL 查询中 FROM 子句的成员，而无需对其他表进行显式 JOIN，并且不会生成“笛卡尔积”警告。可能对诸如 `func.json_array_elements()` 等 SQL 函数有用。

    1.4.46 版本中的新功能。

另请参见

列值函数 - 表值函数作为标量列 - 在 SQLAlchemy 统一教程中

列值函数 - 在 PostgreSQL 文档中

`FunctionElement.table_valued()`

```py
attribute columns
```

此`FunctionElement`导出的一组列。

这是一个占位符集合，允许将函数放置在语句的 FROM 子句中：

```py
>>> from sqlalchemy import column, select, func
>>> stmt = select(column('x'), column('y')).select_from(func.myfunction())
>>> print(stmt)
SELECT  x,  y  FROM  myfunction() 
```

上述形式是一个已过时的功能，现在已被完全功能的`FunctionElement.table_valued()`方法取代；请参阅该方法以获取详情。

另请参见

`FunctionElement.table_valued()` - 生成表值 SQL 函数表达式。

```py
attribute entity_namespace
```

覆盖 FromClause.entity_namespace，因为函数通常是列表达式而不是 FromClauses。

```py
attribute exported_columns
```

```py
method filter(*criterion: _ColumnExpressionArgument[bool]) → Self | FunctionFilter[_T]
```

针对此函数生成一个 FILTER 子句。

用于针对支持“FILTER”子句的聚合和窗口函数的数据库后端。

表达式：

```py
func.count(1).filter(True)
```

是的缩写：

```py
from sqlalchemy import funcfilter
funcfilter(func.count(1), True)
```

另请参见

组内特殊修饰符，过滤器 - 在 SQLAlchemy 统一教程中

`FunctionFilter`

`funcfilter()`

```py
method over(*, partition_by: _ByArgument | None = None, order_by: _ByArgument | None = None, rows: Tuple[int | None, int | None] | None = None, range_: Tuple[int | None, int | None] | None = None) → Over[_T]
```

针对此函数生成一个 OVER 子句。

用于针对聚合或所谓的“窗口”函数，适用于支持窗口函数的数据库后端。

表达式：

```py
func.row_number().over(order_by='x')
```

是的缩写：

```py
from sqlalchemy import over
over(func.row_number(), order_by='x')
```

有关完整描述，请参阅`over()`。

另请参见

`over()`

使用窗口函数 - 在 SQLAlchemy 统一教程中

```py
method scalar_table_valued(name: str, type_: _TypeEngineArgument[_T] | None = None) → ScalarFunctionColumn[_T]
```

返回一个列表达式，作为标量表值表达式针对这个`FunctionElement`。

返回的表达式类似于从`FunctionElement.table_valued()`结构中访问的单个列返回的表达式，只是不生成 FROM 子句；该函数以类似于标量子查询的方式呈现。

例如：

```py
>>> from sqlalchemy import func, select
>>> fn = func.jsonb_each("{'k', 'v'}").scalar_table_valued("key")
>>> print(select(fn))
SELECT  (jsonb_each(:jsonb_each_1)).key 
```

版本 1.4.0b2 中的新功能。

另见

`FunctionElement.table_valued()`

`FunctionElement.alias()`

`FunctionElement.column_valued()`

```py
method select() → Select
```

产生针对这个`FunctionElement`的`select()`构造。

这是的缩写：

```py
s = select(function_element)
```

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

对这个`ClauseElement`应用一个“分组”。

此方法被子类重写为返回“分组”构造，即括号。特别是，它被“二元”表达式使用，当将它们放入较大的表达式中时，提供对自身的分组，以及当将它们放入另一个`select()`构造的 FROM 子句中时，被`select()`构造使用。（请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句具有名称）。

随着表达式组合在一起，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此可能不需要括号，例如，在表达式`x OR (y AND z)`中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本方法`self_group()`只是返回自身。

```py
method table_valued(*expr: _ColumnExpressionOrStrLabelArgument[Any], **kw: Any) → TableValuedAlias
```

返回一个 `FunctionElement` 的 `TableValuedAlias` 表示形式，其中添加了表值表达式。

例如：

```py
>>> fn = (
...     func.generate_series(1, 5).
...     table_valued("value", "start", "stop", "step")
... )

>>> print(select(fn))
SELECT  anon_1.value,  anon_1.start,  anon_1.stop,  anon_1.step
FROM  generate_series(:generate_series_1,  :generate_series_2)  AS  anon_1
>>> print(select(fn.c.value, fn.c.stop).where(fn.c.value > 2))
SELECT  anon_1.value,  anon_1.stop
FROM  generate_series(:generate_series_1,  :generate_series_2)  AS  anon_1
WHERE  anon_1.value  >  :value_1 
```

通过传递关键字参数“with_ordinality”可以生成一个 WITH ORDINALITY 表达式：

```py
>>> fn = func.generate_series(4, 1, -1).table_valued("gen", with_ordinality="ordinality")
>>> print(select(fn))
SELECT  anon_1.gen,  anon_1.ordinality
FROM  generate_series(:generate_series_1,  :generate_series_2,  :generate_series_3)  WITH  ORDINALITY  AS  anon_1 
```

参数：

+   `*expr` - 将添加到结果 `TableValuedAlias` 构造的 `.c` 集合中的一系列字符串列名。也可以使用具有或不具有数据类型的 `column()` 对象。

+   `name` - 分配给生成的别名的可选名称。如果省略，将使用唯一的匿名化名称。

+   `with_ordinality` - 存在时，将 WITH ORDINALITY 子句添加到别名，并将给定的字符串名称添加为结果 `TableValuedAlias` 的 `.c` 集合中的列。

+   `joins_implicitly` -

    当为 True 时，可以在 SQL 查询的 FROM 子句中使用表值函数，而无需对其他表进行显式的 JOIN，并且不会生成“笛卡尔积”警告。对于 SQL 函数（例如 `func.json_each()`）可能很有用。

    新版本 1.4.33 中的新增功能。

新版本 1.4.0b2 中的新增功能。

另请参见

表值函数 - 在 SQLAlchemy 统一教程 中

表值函数 - 在 PostgreSQL 文档中

`FunctionElement.scalar_table_valued()` - `FunctionElement.table_valued()` 的变体，将完整的表值表达式作为标量列表达式传递

`FunctionElement.column_valued()`

`TableValuedAlias.render_derived()` - 使用派生列子句渲染别名，例如 `AS name(col1, col2, ...)`

```py
method within_group(*order_by: _ColumnExpressionArgument[Any]) → WithinGroup[_T]
```

生成针对此函数的 WITHIN GROUP (ORDER BY expr) 子句。

用于所谓的“有序集合聚合”和“假设集合聚合”函数，包括 `percentile_cont`、`rank`、`dense_rank` 等。

有关完整描述，请参阅 `within_group()`。

另请参阅

WITHIN GROUP、FILTER 特殊修饰符 - 在 SQLAlchemy 统一教程 中

```py
method within_group_type(within_group: WithinGroup[_S]) → TypeEngine | None
```

对于将其返回类型定义为基于 WITHIN GROUP (ORDER BY) 表达式中的条件的类型，由 `WithinGroup` 构造调用。

默认情况下返回 None，在这种情况下，函数的正常`.type`被使用。

```py
class sqlalchemy.sql.functions.GenericFunction
```

定义一个‘通用’函数。

泛型函数是预先定义的 `Function` 类，在从 `func` 属性按名称调用时会自动实例化。请注意，从 `func` 调用任何名称的效果是自动创建一个新的 `Function` 实例，给定该名称。定义 `GenericFunction` 类的主要用例是为特定名称的函数指定固定的返回类型。它还可以包括自定义参数解析方案以及其他方法。

`GenericFunction` 的子类会自动注册到类的名称下。例如，用户定义的函数 `as_utc()` 将立即可用：

```py
from sqlalchemy.sql.functions import GenericFunction
from sqlalchemy.types import DateTime

class as_utc(GenericFunction):
    type = DateTime()
    inherit_cache = True

print(select(func.as_utc()))
```

用户定义的通用函数可以通过在定义 `GenericFunction` 时指定“package”属性来组织成包。包含许多函数的第三方库可能希望这样做，以避免与其他系统的名称冲突。例如，如果我们的 `as_utc()` 函数是包 “time” 的一部分：

```py
class as_utc(GenericFunction):
    type = DateTime()
    package = "time"
    inherit_cache = True
```

上述函数可以通过使用包名 `time` 从 `func` 中获得：

```py
print(select(func.time.as_utc()))
```

最后一种选择是允许从`func`中的一个名称访问函数，但呈现为不同的名称。`identifier`属性将覆盖从`func`加载的函数名称，但将保留`name`作为呈现名称的用法：

```py
class GeoBuffer(GenericFunction):
    type = Geometry()
    package = "geo"
    name = "ST_Buffer"
    identifier = "buffer"
    inherit_cache = True
```

以上函数将呈现如下：

```py
>>> print(func.geo.buffer())
ST_Buffer() 
```

名称将原样显示，但不会加引号，除非名称包含需要加引号的特殊字符。要在名称上强制加引号或取消引号，请使用`quoted_name`结构：

```py
from sqlalchemy.sql import quoted_name

class GeoBuffer(GenericFunction):
    type = Geometry()
    package = "geo"
    name = quoted_name("ST_Buffer", True)
    identifier = "buffer"
    inherit_cache = True
```

以上函数将呈现为：

```py
>>> print(func.geo.buffer())
"ST_Buffer"() 
```

可以传递此类作为[泛型类型](https://peps.python.org/pep-0484/#generics)的类的类型参数，并应与`Result`中看到的类型相匹配。例如：

```py
class as_utc(GenericFunction[datetime.datetime]):
    type = DateTime()
    inherit_cache = True
```

以上表明以下表达式返回一个`datetime`对象：

```py
connection.scalar(select(func.as_utc()))
```

从版本 1.3.13 开始：在对象的“name”属性中使用`quoted_name`结构现在被识别为引用，因此可以强制对函数名称进行引用或取消引用。

**类签名**

类`sqlalchemy.sql.functions.GenericFunction`（`sqlalchemy.sql.functions.Function`）

```py
function sqlalchemy.sql.functions.register_function(identifier: str, fn: Type[Function[Any]], package: str = '_default') → None
```

将可调用对象与特定的函数名关联起来。

通常由 GenericFunction 调用，但也可单独使用，以便将非 Function 构造与`func`访问器关联起来（即 CAST，EXTRACT）。

## 选定的“已知”函数

这些是一组常见 SQL 函数的`GenericFunction`实现，为每个函数自动设置了预期的返回类型。它们以与`func`命名空间的任何其他成员相同的方式调用：

```py
select(func.count("*")).select_from(some_table)
```

请注意，任何`func`未知的名称都会按原样生成函数名称 - SQLAlchemy 对可以调用的 SQL 函数没有限制，不管对 SQLAlchemy 已知还是未知，内置还是用户定义。本节仅描述 SQLAlchemy 已知参数和返回类型的函数。

| 对象名称 | 描述 |
| --- | --- |
| aggregate_strings | 实现一个通用的字符串聚合函数。 |
| array_agg | 对 ARRAY_AGG 函数的支持。 |
| char_length | CHAR_LENGTH() SQL 函数。 |
| coalesce |  |
| concat | SQL CONCAT()函数，用于连接字符串。 |
| count | ANSI COUNT 聚合函数。没有参数时，发出 COUNT *。 |
| cube | 实现`CUBE`分组操作。 |
| cume_dist | 实现`cume_dist`假设集合聚合函数。 |
| current_date | CURRENT_DATE() SQL 函数。 |
| current_time | CURRENT_TIME() SQL 函数。 |
| current_timestamp | CURRENT_TIMESTAMP() SQL 函数。 |
| current_user | CURRENT_USER() SQL 函数。 |
| dense_rank | 实现`dense_rank`假设集合聚合函数。 |
| grouping_sets | 实现`GROUPING SETS`分组操作。 |
| localtime | localtime() SQL 函数。 |
| localtimestamp | localtimestamp() SQL 函数。 |
| max | SQL MAX()聚合函数。 |
| min | SQL MIN()聚合函数。 |
| mode | 实现`mode`有序集合聚合函数。 |
| next_value | 代表“下一个值”，给定一个`Sequence`作为其唯一参数。 |
| now | SQL now()日期时间函数。 |
| percent_rank | 实现`percent_rank`假设集合聚合函数。 |
| percentile_cont | 实现`percentile_cont`有序集合聚合函数。 |
| percentile_disc | 实现`percentile_disc`有序集合聚合函数。 |
| random | RANDOM() SQL 函数。 |
| rank | 实现`rank`假设集合聚合函数。 |
| rollup | 实现`ROLLUP`分组操作。 |
| session_user | SESSION_USER() SQL 函数。 |
| sum | SQL SUM()聚合函数。 |
| sysdate | SYSDATE() SQL 函数。 |
| user | USER() SQL 函数。 |

```py
class sqlalchemy.sql.functions.aggregate_strings
```

实现一个通用的字符串聚合函数。

此函数将非空值连接成字符串，并用分隔符分隔值。

此函数根据每个后端编译为`group_concat()`、`string_agg()`或`LISTAGG()`等函数。

例如，使用分隔符‘.’的示例用法：

```py
stmt = select(func.aggregate_strings(table.c.str_col, "."))
```

此函数的返回类型是`String`。

**类签名**

类`sqlalchemy.sql.functions.aggregate_strings`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.array_agg
```

支持 ARRAY_AGG 函数。

`func.array_agg(expr)`构造返回类型为`ARRAY`的表达式。

例如：

```py
stmt = select(func.array_agg(table.c.values)[2:5])
```

参见

`array_agg()` - 返回`ARRAY`的 PostgreSQL 特定版本，其中添加了 PG 特定的运算符。

**类签名**

类`sqlalchemy.sql.functions.array_agg`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.char_length
```

CHAR_LENGTH() SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.char_length`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.coalesce
```

**类签名**

类`sqlalchemy.sql.functions.coalesce`（`sqlalchemy.sql.functions.ReturnTypeFromArgs`）

```py
class sqlalchemy.sql.functions.concat
```

SQL CONCAT()函数，用于连接字符串。

例如：

```py
>>> print(select(func.concat('a', 'b')))
SELECT  concat(:concat_2,  :concat_3)  AS  concat_1 
```

在 SQLAlchemy 中，字符串连接更常见地使用 Python 的`+`运算符与字符串数据类型一起使用，这将呈现特定于后端的连接运算符，例如：

```py
>>> print(select(literal("a") + "b"))
SELECT  :param_1  ||  :param_2  AS  anon_1 
```

**类签名**

类`sqlalchemy.sql.functions.concat`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.count
```

ANSI COUNT 聚合函数。没有参数时，发出 COUNT *。

例如：

```py
from sqlalchemy import func
from sqlalchemy import select
from sqlalchemy import table, column

my_table = table('some_table', column('id'))

stmt = select(func.count()).select_from(my_table)
```

执行`stmt`将发出：

```py
SELECT count(*) AS count_1
FROM some_table
```

**类签名**

类`sqlalchemy.sql.functions.count`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.cube
```

实现`CUBE`分组操作。

此函数用作语句的 GROUP BY 的一部分，例如`Select.group_by()`： 

```py
stmt = select(
    func.sum(table.c.value), table.c.col_1, table.c.col_2
).group_by(func.cube(table.c.col_1, table.c.col_2))
```

新增于版本 1.2。

**类签名**

类`sqlalchemy.sql.functions.cube`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.cume_dist
```

实现`cume_dist`假设集聚合函数。

此函数必须与`FunctionElement.within_group()`修饰符一起使用，以提供要操作的排序表达式。

此函数的返回类型为`Numeric`。

**类签名**

类`sqlalchemy.sql.functions.cume_dist`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.current_date
```

`CURRENT_DATE()` SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.current_date`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.current_time
```

`CURRENT_TIME()` SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.current_time`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.current_timestamp
```

`CURRENT_TIMESTAMP()` SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.current_timestamp`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.current_user
```

`CURRENT_USER()` SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.current_user`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.dense_rank
```

实现`dense_rank`假设集聚合函数。

此函数必须与`FunctionElement.within_group()`修饰符一起使用，以提供要操作的排序表达式。

此函数的返回类型为`Integer`。

**类签名**

类 `sqlalchemy.sql.functions.dense_rank`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.grouping_sets
```

实现 `GROUPING SETS` 分组操作。

此函数用作语句的 GROUP BY 的一部分，例如 `Select.group_by()`:

```py
stmt = select(
    func.sum(table.c.value), table.c.col_1, table.c.col_2
).group_by(func.grouping_sets(table.c.col_1, table.c.col_2))
```

为了按多个集合进行分组，请使用 `tuple_()` 构造：

```py
from sqlalchemy import tuple_

stmt = select(
    func.sum(table.c.value),
    table.c.col_1, table.c.col_2,
    table.c.col_3
).group_by(
    func.grouping_sets(
        tuple_(table.c.col_1, table.c.col_2),
        tuple_(table.c.value, table.c.col_3),
    )
)
```

版本 1.2 中的新增内容。

**类签名**

类 `sqlalchemy.sql.functions.grouping_sets`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.localtime
```

localtime() SQL 函数。

**类签名**

类 `sqlalchemy.sql.functions.localtime`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.localtimestamp
```

localtimestamp() SQL 函数。

**类签名**

类 `sqlalchemy.sql.functions.localtimestamp`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.max
```

SQL MAX() 聚合函数。

**类签名**

类 `sqlalchemy.sql.functions.max`（`sqlalchemy.sql.functions.ReturnTypeFromArgs`）

```py
class sqlalchemy.sql.functions.min
```

SQL MIN() 聚合函数。

**类签名**

类 `sqlalchemy.sql.functions.min`（`sqlalchemy.sql.functions.ReturnTypeFromArgs`）

```py
class sqlalchemy.sql.functions.mode
```

实现 `mode` 排序集合聚合函数。

必须与 `FunctionElement.within_group()` 修改器一起使用，以提供要操作的排序表达式。

此函数的返回类型与排序表达式相同。

**类签名**

类 `sqlalchemy.sql.functions.mode`（`sqlalchemy.sql.functions.OrderedSetAgg`）

```py
class sqlalchemy.sql.functions.next_value
```

表示“下一个值”，给定 `Sequence` 作为其唯一参数。

编译为每个后端的适当函数，或者如果在不提供序列支持的后端上使用，则会引发 NotImplementedError。

**类签名**

类`sqlalchemy.sql.functions.next_value`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.now
```

SQL 的 now()日期时间函数。

SQLAlchemy 方言通常会以特定于后端的方式呈现此特定函数，例如将其呈现为`CURRENT_TIMESTAMP`。

**类签名**

类`sqlalchemy.sql.functions.now`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.percent_rank
```

实现`percent_rank`假设集合聚合函数。

必须使用`FunctionElement.within_group()`修饰符来提供要操作的排序表达式。

这个函数的返回类型是`Numeric`。

**类签名**

类`sqlalchemy.sql.functions.percent_rank`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.percentile_cont
```

实现`percentile_cont`有序集合聚合函数。

必须使用`FunctionElement.within_group()`修饰符来提供要操作的排序表达式。

这个函数的返回类型与排序表达式相同，或者如果参数是一个数组，则返回排序表达式类型的`ARRAY`。

**类签名**

类`sqlalchemy.sql.functions.percentile_cont`（`sqlalchemy.sql.functions.OrderedSetAgg`）

```py
class sqlalchemy.sql.functions.percentile_disc
```

实现`percentile_disc`有序集合聚合函数。

必须使用`FunctionElement.within_group()`修饰符来提供要操作的排序表达式。

这个函数的返回类型与排序表达式相同，或者如果参数是一个数组，则返回排序表达式类型的`ARRAY`。

**类签名**

类`sqlalchemy.sql.functions.percentile_disc`（`sqlalchemy.sql.functions.OrderedSetAgg`）

```py
class sqlalchemy.sql.functions.random
```

RANDOM() SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.random` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.rank
```

实现`rank`虚拟集合聚合函数。

此函数必须与`FunctionElement.within_group()`修饰符一起使用，以提供要操作的排序表达式。

此函数的返回类型为`Integer`。

**类签名**

类`sqlalchemy.sql.functions.rank` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.rollup
```

实现`ROLLUP`分组操作。

此函数用作语句的 GROUP BY 的一部分，例如`Select.group_by()`:

```py
stmt = select(
    func.sum(table.c.value), table.c.col_1, table.c.col_2
).group_by(func.rollup(table.c.col_1, table.c.col_2))
```

新版本 1.2 中添加。

**类签名**

类`sqlalchemy.sql.functions.rollup` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.session_user
```

SESSION_USER() SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.session_user` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.sum
```

SQL 的 SUM()聚合函数。

**类签名**

类`sqlalchemy.sql.functions.sum` (`sqlalchemy.sql.functions.ReturnTypeFromArgs`)

```py
class sqlalchemy.sql.functions.sysdate
```

SYSDATE() SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.sysdate` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.user
```

USER() SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.user` (`sqlalchemy.sql.functions.AnsiFunction`)

## 函数 API

SQL 函数的基本 API，提供了`func`命名空间以及可用于可扩展性的类。

| 对象名称 | 描述 |
| --- | --- |
| AnsiFunction | 定义以“ansi”格式编写的函数，不渲染括号。 |
| Function | 描述一个命名的 SQL 函数。 |
| FunctionElement | 面向 SQL 函数构建的基类。 |
| GenericFunction | 定义一个“通用”函数。 |
| register_function(identifier, fn[, package]) | 将可调用对象与特定的 func. 名称关联起来。 |

```py
class sqlalchemy.sql.functions.AnsiFunction
```

在“ansi”格式中定义一个不渲染括号的函数。

**类签名**

类 `sqlalchemy.sql.functions.AnsiFunction` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.Function
```

描述一个命名的 SQL 函数。

`Function` 对象通常是从 `func` 生成对象生成的。

参数：

+   `*clauses` – 形成 SQL 函数调用参数的列表达式列表。

+   `type_` – 可选的 `TypeEngine` 数据类型对象，将用作由此函数调用生成的列表达式的返回值。

+   `packagenames` –

    一个字符串，指示在生成 SQL 时要在函数名称之前添加的包前缀名称。当以点格式调用 `func` 生成器时，会创建这些内容，例如：

    ```py
    func.mypackage.some_function(col1, col2)
    ```

另请参阅

处理 SQL 函数 - 在 SQLAlchemy 统一教程 中

`func` - 产生注册或临时 `Function` 实例的命名空间。

`GenericFunction` - 允许创建已注册的函数类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.sql.functions.Function` (`sqlalchemy.sql.functions.FunctionElement`)

```py
method __init__(name: str, *clauses: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T] | None = None, packagenames: Tuple[str, ...] | None = None)
```

构建一个 `Function`。

`func` 结构通常用于构建新的 `Function` 实例。

```py
class sqlalchemy.sql.functions.FunctionElement
```

面向 SQL 函数构建的基类。

这是一个[通用类型](https://peps.python.org/pep-0484/#generics)，意味着类型检查器和集成开发环境可以指示在此函数的 `Result` 中期望的类型。查看 `GenericFunction` 以了解如何执行此操作的示例。

另请参阅

使用 SQL 函数 - 在 SQLAlchemy 统一教程 中

`Function` - 命名的 SQL 函数。

`func` - 生成注册或临时的 `Function` 实例的命名空间。

`GenericFunction` - 允许创建注册的函数类型。

**成员**

__init__(), alias(), as_comparison(), c, clauses, column_valued(), columns, entity_namespace, exported_columns, filter(), over(), scalar_table_valued(), select(), self_group(), table_valued(), within_group(), within_group_type()

**类签名**

类 `sqlalchemy.sql.functions.FunctionElement` (`sqlalchemy.sql.expression.Executable`, `sqlalchemy.sql.expression.ColumnElement`, `sqlalchemy.sql.expression.FromClause`, `sqlalchemy.sql.expression.Generative`)

```py
method __init__(*clauses: _ColumnExpressionOrLiteralArgument[Any])
```

构建一个 `FunctionElement`。

参数：

+   `*clauses` – 构成 SQL 函数调用参数的列表达式列表。

+   `**kwargs` – 通常由子类使用的额外 kwargs。

另请参阅

`func`

`Function`

```py
method alias(name: str | None = None, joins_implicitly: bool = False) → TableValuedAlias
```

根据此 `FunctionElement` 创建一个 `Alias` 结构。

提示

`FunctionElement.alias()` 方法是创建“表值”SQL 函数的机制的一部分。 但是，大多数用例都由 `FunctionElement` 上的更高级方法覆盖，包括 `FunctionElement.table_valued()` 和 `FunctionElement.column_valued()`。

此结构将函数包装在一个适合 FROM 子句的命名别名中，其样式符合 PostgreSQL 示例。 还提供了一个列表达式，使用特殊的 `.column` 属性，该属性可用于在列或 WHERE 子句中引用函数的输出，例如 PostgreSQL 这样的后端中的标量值。

对于完整的表值表达式，首先使用 `FunctionElement.table_valued()` 方法来建立具名列。

例如：

```py
>>> from sqlalchemy import func, select, column
>>> data_view = func.unnest([1, 2, 3]).alias("data_view")
>>> print(select(data_view.column))
SELECT  data_view
FROM  unnest(:unnest_1)  AS  data_view 
```

`FunctionElement.column_valued()` 方法提供了上述模式的快捷方式：

```py
>>> data_view = func.unnest([1, 2, 3]).column_valued("data_view")
>>> print(select(data_view))
SELECT  data_view
FROM  unnest(:unnest_1)  AS  data_view 
```

新于版本 1.4.0b2：添加了 `.column` 访问器

参数：

+   `name` – 别名，将在 FROM 子句中呈现为 `AS <name>`

+   `joins_implicitly` –

    当为 True 时，可以在 SQL 查询的 FROM 子句中使用表值函数，而无需对其他表进行显式 JOIN，并且不会生成“笛卡尔积”警告。 对于诸如 `func.json_each()` 之类的 SQL 函数可能很有用。

    新于版本 1.4.33。

另请参阅

表值函数 - 在 SQLAlchemy 统一教程 中

`FunctionElement.table_valued()`

`FunctionElement.scalar_table_valued()`

`FunctionElement.column_valued()`

```py
method as_comparison(left_index: int, right_index: int) → FunctionAsBinary
```

将此表达式解释为两个值之间的布尔比较。

此方法用于描述 ORM 用例的基于 SQL 函数的自定义运算符。

一个假设的比较两个值是否相等的 SQL 函数“is_equal()”将在 Core 表达式语言中编写为：

```py
expr = func.is_equal("a", "b")
```

如果上述的“is_equal()”比较的是“a”和“b”的相等性，那么`FunctionElement.as_comparison()`方法将被调用如下：

```py
expr = func.is_equal("a", "b").as_comparison(1, 2)
```

在上面，整数值“1”指的是“is_equal()”函数的第一个参数，整数值“2”指的是第二个参数。

这将创建一个等同于的`BinaryExpression`：

```py
BinaryExpression("a", "b", operator=op.eq)
```

但是，在 SQL 级别上，它仍然呈现为“is_equal('a', 'b')”。

当 ORM 加载相关对象或集合时，需要能够操作 JOIN 表达式的 ON 子句的“左”和“右”侧。此方法的目的是在使用`relationship.primaryjoin`参数时，为 ORM 提供一个也可以向其提供此信息的 SQL 函数构造，返回值是一个名为`FunctionAsBinary`的包含对象。

一个 ORM 示例如下：

```py
class Venue(Base):
    __tablename__ = 'venue'
    id = Column(Integer, primary_key=True)
    name = Column(String)

    descendants = relationship(
        "Venue",
        primaryjoin=func.instr(
            remote(foreign(name)), name + "/"
        ).as_comparison(1, 2) == 1,
        viewonly=True,
        order_by=name
    )
```

在上面的例子中，“Venue”类可以通过确定父级 Venue 的名称是否包含在假想后代值的名称的开头来加载后代“Venue”对象，例如，“parent1”将匹配到“parent1/child1”，但不会匹配到“parent2/child1”。

可能的用例包括上面给出的“materialized path”示例，以及利用特殊的 SQL 函数来创建连接条件，如几何函数。

参数：

+   `left_index` – 函数参数中作为“左侧”表达式的整数索引（从 1 开始）。

+   `right_index` – 函数参数中作为“右侧”表达式的整数索引（从 1 开始）。

版本 1.3 中的新功能。

另请参阅

基于 SQL 函数的自定义运算符 - 在 ORM 中的示例用法

```py
attribute c
```

`FunctionElement.columns`的同义词。

```py
attribute clauses
```

返回包含此`FunctionElement`参数的`ClauseList`的基础对象。

```py
method column_valued(name: str | None = None, joins_implicitly: bool = False) → TableValuedColumn[_T]
```

将此`FunctionElement`作为从自身选择的列表达式返回。

例如：

```py
>>> from sqlalchemy import select, func
>>> gs = func.generate_series(1, 5, -1).column_valued()
>>> print(select(gs))
SELECT  anon_1
FROM  generate_series(:generate_series_1,  :generate_series_2,  :generate_series_3)  AS  anon_1 
```

这是的简写形式：

```py
gs = func.generate_series(1, 5, -1).alias().column
```

参数：

+   `name` - 可选的名称，用于分配生成的别名名称。如果省略，将使用唯一的匿名名称。

+   `joins_implicitly` -

    当为 True 时，列值函数的“table”部分可以作为 SQL 查询中 FROM 子句的成员，而不需要对其他表进行显式 JOIN，并且不会生成“笛卡尔积”警告。 对于诸如`func.json_array_elements()`之类的 SQL 函数可能有用。

    1.4.46 版中的新功能。

请参阅

列值函数 - 表值函数作为标量列 - 在 SQLAlchemy 统一教程中

列值函数 - 在 PostgreSQL 文档中

`FunctionElement.table_valued()`

```py
attribute columns
```

此`FunctionElement`导出的列的集合。

这是一个占位符集合，允许将函数放置在语句的 FROM 子句中：

```py
>>> from sqlalchemy import column, select, func
>>> stmt = select(column('x'), column('y')).select_from(func.myfunction())
>>> print(stmt)
SELECT  x,  y  FROM  myfunction() 
```

上述形式是一个现在已被完全功能的`FunctionElement.table_valued()`方法所取代的遗留特性；有关详细信息，请参阅该方法。

请参阅

`FunctionElement.table_valued()` - 生成表值 SQL 函数表达式。

```py
attribute entity_namespace
```

覆盖 FromClause.entity_namespace，因为函数通常是列表达式，而不是 FromClauses。

```py
attribute exported_columns
```

```py
method filter(*criterion: _ColumnExpressionArgument[bool]) → Self | FunctionFilter[_T]
```

产生针对此函数的 FILTER 子句。

用于支持“FILTER”子句的数据库后端中的聚合和窗口函数。

表达式：

```py
func.count(1).filter(True)
```

是的简写形式：

```py
from sqlalchemy import funcfilter
funcfilter(func.count(1), True)
```

请参阅

特殊修饰符 WITHIN GROUP，FILTER - 在 SQLAlchemy 统一教程中

`FunctionFilter`

`funcfilter()`

```py
method over(*, partition_by: _ByArgument | None = None, order_by: _ByArgument | None = None, rows: Tuple[int | None, int | None] | None = None, range_: Tuple[int | None, int | None] | None = None) → Over[_T]
```

产生针对此函数的 OVER 子句。

用于支持窗口函数的聚合或所谓的“窗口”函数的数据库后端。

表达式：

```py
func.row_number().over(order_by='x')
```

是的简写形式：

```py
from sqlalchemy import over
over(func.row_number(), order_by='x')
```

请参阅`over()`以获取完整描述。

请参阅

`over()`

使用窗口函数 - 在 SQLAlchemy 统一教程中

```py
method scalar_table_valued(name: str, type_: _TypeEngineArgument[_T] | None = None) → ScalarFunctionColumn[_T]
```

返回一个针对这个`FunctionElement`的列表达式作为标量表值表达式。

返回的表达式类似于从`FunctionElement.table_valued()`构造中访问的单个列返回的表达式，除了不生成 FROM 子句；该函数以标量子查询的方式呈现。

例如：

```py
>>> from sqlalchemy import func, select
>>> fn = func.jsonb_each("{'k', 'v'}").scalar_table_valued("key")
>>> print(select(fn))
SELECT  (jsonb_each(:jsonb_each_1)).key 
```

版本 1.4.0b2 中的新功能。

另请参见

`FunctionElement.table_valued()`

`FunctionElement.alias()`

`FunctionElement.column_valued()`

```py
method select() → Select
```

产生一个针对这个`FunctionElement`的`select()`构造。

这是一个简写：

```py
s = select(function_element)
```

```py
method self_group(against: OperatorType | None = None) → ClauseElement
```

对这个`ClauseElement`应用一个“分组”。

子类重写此方法以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当它们被放置到更大的表达式中时提供一个围绕自身的分组，以及当它们被放置到另一个`select()`的 FROM 子句中时，由`select()`构造使用。（请注意，子查询通常应该使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须被命名）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应该直接使用这个方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在表达式中可能不需要括号，例如，`x OR (y AND z)` - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
method table_valued(*expr: _ColumnExpressionOrStrLabelArgument[Any], **kw: Any) → TableValuedAlias
```

返回此`FunctionElement`的`TableValuedAlias`表示，其中添加了表值表达式。

例如：

```py
>>> fn = (
...     func.generate_series(1, 5).
...     table_valued("value", "start", "stop", "step")
... )

>>> print(select(fn))
SELECT  anon_1.value,  anon_1.start,  anon_1.stop,  anon_1.step
FROM  generate_series(:generate_series_1,  :generate_series_2)  AS  anon_1
>>> print(select(fn.c.value, fn.c.stop).where(fn.c.value > 2))
SELECT  anon_1.value,  anon_1.stop
FROM  generate_series(:generate_series_1,  :generate_series_2)  AS  anon_1
WHERE  anon_1.value  >  :value_1 
```

通过传递关键字参数“with_ordinality”可以生成一个 WITH ORDINALITY 表达式：

```py
>>> fn = func.generate_series(4, 1, -1).table_valued("gen", with_ordinality="ordinality")
>>> print(select(fn))
SELECT  anon_1.gen,  anon_1.ordinality
FROM  generate_series(:generate_series_1,  :generate_series_2,  :generate_series_3)  WITH  ORDINALITY  AS  anon_1 
```

参数：

+   `*expr` – 一系列将作为列添加到结果的`TableValuedAlias`构造中的字符串列名。也可以使用具有或不具有数据类型的`column()`对象。

+   `name` – 分配给生成的别名名称的可选名称。如果省略，将使用唯一的匿名化名称。

+   `with_ordinality` – 当存在时，会将`WITH ORDINALITY`子句添加到别名中，并且给定的字符串名称将作为列添加到结果的`TableValuedAlias`的`.c`集合中。

+   `joins_implicitly` –

    当为 True 时，可以在 SQL 查询的 FROM 子句中使用表值函数，而无需对其他表进行显式 JOIN，并且不会生成“笛卡尔积”警告。对于诸如`func.json_each()`之类的 SQL 函数可能很有用。

    新功能在版本 1.4.33 中引入。

新功能在版本 1.4.0b2 中引入。

另请参阅

表值函数 - 在 SQLAlchemy 统一教程中

表值函数 - 在 PostgreSQL 文档中

`FunctionElement.scalar_table_valued()` - `FunctionElement.table_valued()`的变体，将完整的表值表达式作为标量列表达式传递

`FunctionElement.column_valued()`

`TableValuedAlias.render_derived()` - 使用派生列子句呈现别名，例如`AS name(col1, col2, ...)`

```py
method within_group(*order_by: _ColumnExpressionArgument[Any]) → WithinGroup[_T]
```

生成一个针对此函数的 WITHIN GROUP (ORDER BY expr) 子句。

用于所谓的“有序集合聚合”和“假设集合聚合”函数，包括`percentile_cont`、`rank`、`dense_rank`等。

详细描述请参见`within_group()`。

另请参见

WITHIN GROUP、FILTER 特殊修饰符 - 在 SQLAlchemy 统一教程中

```py
method within_group_type(within_group: WithinGroup[_S]) → TypeEngine | None
```

对于将其返回类型定义为基于 WITHIN GROUP (ORDER BY) 表达式中的条件的类型，通过 `WithinGroup` 构造调用。

默认情况下返回 None，此时使用函数的普通`.type`。

```py
class sqlalchemy.sql.functions.GenericFunction
```

定义一个“通用”函数。

通用函数是预先建立的`Function`类，在从`func`属性中按名称调用时自动实例化。请注意，从`func`调用任何名称都会自动创建一个新的`Function`实例，给定该名称。定义`GenericFunction`类的主要用例是为特定名称的函数指定固定的返回类型。它还可以包括自定义参数解析方案以及其他方法。

`GenericFunction`的子类会自动注册在类的名称下。例如，用户定义的函数`as_utc()`将立即可用：

```py
from sqlalchemy.sql.functions import GenericFunction
from sqlalchemy.types import DateTime

class as_utc(GenericFunction):
    type = DateTime()
    inherit_cache = True

print(select(func.as_utc()))
```

用户定义的通用函数可以通过在定义`GenericFunction`时指定“package”属性来组织到包中。许多函数的第三方库可能想要使用此功能，以避免与其他系统的名称冲突。例如，如果我们的 `as_utc()` 函数是“time”包的一部分：

```py
class as_utc(GenericFunction):
    type = DateTime()
    package = "time"
    inherit_cache = True
```

上述函数可以通过`func`来使用，使用包名`time`：

```py
print(select(func.time.as_utc()))
```

最后一个选项是允许从`func`中的一个名称访问该函数，但呈现为不同的名称。 `identifier` 属性将覆盖从`func`加载时用于访问函数的名称，但将保留使用 `name` 作为呈现名称的用法：

```py
class GeoBuffer(GenericFunction):
    type = Geometry()
    package = "geo"
    name = "ST_Buffer"
    identifier = "buffer"
    inherit_cache = True
```

上述函数将呈现如下：

```py
>>> print(func.geo.buffer())
ST_Buffer() 
```

名称将按原样呈现，但如果名称包含需要引用的特殊字符，则不会引用。要强制对名称进行引用或取消引用，请使用 `quoted_name` 结构：

```py
from sqlalchemy.sql import quoted_name

class GeoBuffer(GenericFunction):
    type = Geometry()
    package = "geo"
    name = quoted_name("ST_Buffer", True)
    identifier = "buffer"
    inherit_cache = True
```

上述函数将呈现为：

```py
>>> print(func.geo.buffer())
"ST_Buffer"() 
```

此类的类型参数作为 [通用类型](https://peps.python.org/pep-0484/#generics) 可以传递，并且应该与 `Result` 中看到的类型匹配。例如：

```py
class as_utc(GenericFunction[datetime.datetime]):
    type = DateTime()
    inherit_cache = True
```

以上表明以下表达式返回一个 `datetime` 对象：

```py
connection.scalar(select(func.as_utc()))
```

从版本 1.3.13 开始：当与对象的“name”属性一起使用时，`quoted_name` 结构现在被识别为引用，因此可以强制对函数名称进行引用。

**类签名**

类`sqlalchemy.sql.functions.GenericFunction` (`sqlalchemy.sql.functions.Function`)

```py
function sqlalchemy.sql.functions.register_function(identifier: str, fn: Type[Function[Any]], package: str = '_default') → None
```

将可调用对象与特定的函数名称关联起来。

通常由 GenericFunction 调用，但也可以单独使用，以便将非 Function 结构与`func`访问器关联起来（例如 CAST、EXTRACT）。

## 选定的“已知”函数

这些是一组选定的常见 SQL 函数的`GenericFunction`实现，为每个函数自动设置了预期的返回类型。它们以与`func`命名空间的任何其他成员相同的方式调用：

```py
select(func.count("*")).select_from(some_table)
```

请注意，任何未知于`func`的名称都会按原样生成函数名称 - 对于可以调用的 SQL 函数，对 SQLAlchemy 有无所谓是否知道它们，内置或用户定义的没有限制。这里的部分仅描述了 SQLAlchemy 已经知道正在使用什么参数和返回类型的函数。

| 对象名称 | 描述 |
| --- | --- |
| aggregate_strings | 实现一个通用的字符串聚合函数。 |
| array_agg | 支持 ARRAY_AGG 函数。 |
| char_length | CHAR_LENGTH() SQL 函数。 |
| coalesce |  |
| concat | SQL CONCAT() 函数，用于连接字符串。 |
| count | ANSI COUNT 聚合函数。没有参数时，发出 COUNT *。 |
| cube | 实现`CUBE`分组操作。 |
| cume_dist | 实现`cume_dist`假设集聚合函数。 |
| current_date | CURRENT_DATE() SQL 函数。 |
| current_time | CURRENT_TIME() SQL 函数。 |
| current_timestamp | CURRENT_TIMESTAMP() SQL 函数。 |
| current_user | CURRENT_USER() SQL 函数。 |
| dense_rank | 实现`dense_rank`假设集聚合函数。 |
| grouping_sets | 实现`GROUPING SETS`分组操作。 |
| localtime | localtime() SQL 函数。 |
| localtimestamp | localtimestamp() SQL 函数。 |
| max | SQL MAX() 聚合函数。 |
| min | SQL MIN() 聚合函数。 |
| mode | 实现`mode`有序集聚合函数。 |
| next_value | 代表“下一个值”，以`Sequence`作为其唯一参数。 |
| now | SQL now() 日期时间函数。 |
| percent_rank | 实现`percent_rank`假设集聚合函数。 |
| percentile_cont | 实现`percentile_cont`有序集聚合函数。 |
| percentile_disc | 实现`percentile_disc`有序集聚合函数。 |
| random | RANDOM() SQL 函数。 |
| rank | 实现`rank`假设集聚合函数。 |
| rollup | 实现`ROLLUP`分组操作。 |
| session_user | SESSION_USER() SQL 函数。 |
| sum | SQL SUM() 聚合函数。 |
| sysdate | SYSDATE() SQL 函数。 |
| user | USER() SQL 函数。 |

```py
class sqlalchemy.sql.functions.aggregate_strings
```

实现一个通用的字符串聚合函数。

此函数将非空值连接为一个字符串，并用分隔符分隔值。

此函数根据每个后端编译为`group_concat()`、`string_agg()`或`LISTAGG()`等函数。

例如，使用分隔符‘.’的示例用法：

```py
stmt = select(func.aggregate_strings(table.c.str_col, "."))
```

此函数的返回类型为`String`。

**类签名**

类`sqlalchemy.sql.functions.aggregate_strings`（`sqlalchemy.sql.functions.GenericFunction`）。

```py
class sqlalchemy.sql.functions.array_agg
```

支持 ARRAY_AGG 函数。

`func.array_agg(expr)`构造返回类型为`ARRAY`的表达式。

例如：

```py
stmt = select(func.array_agg(table.c.values)[2:5])
```

另请参阅

`array_agg()` - 返回`ARRAY`的 PostgreSQL 特定版本，其中添加了 PG 特定运算符。

**类签名**

类`sqlalchemy.sql.functions.array_agg`（`sqlalchemy.sql.functions.GenericFunction`）。

```py
class sqlalchemy.sql.functions.char_length
```

SQL 函数`CHAR_LENGTH()`.

**类签名**

类`sqlalchemy.sql.functions.char_length`（`sqlalchemy.sql.functions.GenericFunction`）。

```py
class sqlalchemy.sql.functions.coalesce
```

**类签名**

类`sqlalchemy.sql.functions.coalesce`（`sqlalchemy.sql.functions.ReturnTypeFromArgs`）。

```py
class sqlalchemy.sql.functions.concat
```

SQL CONCAT()函数，用于连接字符串。

例如：

```py
>>> print(select(func.concat('a', 'b')))
SELECT  concat(:concat_2,  :concat_3)  AS  concat_1 
```

在 SQLAlchemy 中，使用 Python 的`+`运算符与字符串数据类型更常见，这将呈现特定于后端的连接运算符，例如：

```py
>>> print(select(literal("a") + "b"))
SELECT  :param_1  ||  :param_2  AS  anon_1 
```

**类签名**

类`sqlalchemy.sql.functions.concat`（`sqlalchemy.sql.functions.GenericFunction`）。

```py
class sqlalchemy.sql.functions.count
```

ANSI COUNT 聚合函数。没有参数时，发出 COUNT *。

例如：

```py
from sqlalchemy import func
from sqlalchemy import select
from sqlalchemy import table, column

my_table = table('some_table', column('id'))

stmt = select(func.count()).select_from(my_table)
```

执行`stmt`将发出：

```py
SELECT count(*) AS count_1
FROM some_table
```

**类签名**

类`sqlalchemy.sql.functions.count`（`sqlalchemy.sql.functions.GenericFunction`）。

```py
class sqlalchemy.sql.functions.cube
```

实现`CUBE`分组操作。

此函数用作语句的 GROUP BY 的一部分，例如 `Select.group_by()`：

```py
stmt = select(
    func.sum(table.c.value), table.c.col_1, table.c.col_2
).group_by(func.cube(table.c.col_1, table.c.col_2))
```

新版本 1.2 中新增。

**类签名**

class `sqlalchemy.sql.functions.cube` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.cume_dist
```

实现 `cume_dist` 假设集聚合函数。

必须使用 `FunctionElement.within_group()` 修饰符来提供一个排序表达式以进行操作。

该函数的返回类型是 `Numeric`。

**类签名**

class `sqlalchemy.sql.functions.cume_dist` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.current_date
```

CURRENT_DATE() SQL 函数。

**类签名**

class `sqlalchemy.sql.functions.current_date` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.current_time
```

CURRENT_TIME() SQL 函数。

**类签名**

class `sqlalchemy.sql.functions.current_time` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.current_timestamp
```

CURRENT_TIMESTAMP() SQL 函数。

**类签名**

class `sqlalchemy.sql.functions.current_timestamp` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.current_user
```

CURRENT_USER() SQL 函数。

**类签名**

class `sqlalchemy.sql.functions.current_user` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.dense_rank
```

实现 `dense_rank` 假设集聚合函数。

必须使用 `FunctionElement.within_group()` 修饰符来提供一个排序表达式以进行操作。

该函数的返回类型是 `Integer`。

**类签名**

类`sqlalchemy.sql.functions.dense_rank`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.grouping_sets
```

实现 `GROUPING SETS` 分组操作。

此函数用作语句的 GROUP BY 的一部分，例如 `Select.group_by()`：

```py
stmt = select(
    func.sum(table.c.value), table.c.col_1, table.c.col_2
).group_by(func.grouping_sets(table.c.col_1, table.c.col_2))
```

要按多个集合分组，请使用`tuple_()`结构：

```py
from sqlalchemy import tuple_

stmt = select(
    func.sum(table.c.value),
    table.c.col_1, table.c.col_2,
    table.c.col_3
).group_by(
    func.grouping_sets(
        tuple_(table.c.col_1, table.c.col_2),
        tuple_(table.c.value, table.c.col_3),
    )
)
```

版本 1.2 中的新功能。

**类签名**

类`sqlalchemy.sql.functions.grouping_sets`（`sqlalchemy.sql.functions.GenericFunction`）

```py
class sqlalchemy.sql.functions.localtime
```

localtime() SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.localtime`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.localtimestamp
```

localtimestamp() SQL 函数。

**类签名**

类`sqlalchemy.sql.functions.localtimestamp`（`sqlalchemy.sql.functions.AnsiFunction`）

```py
class sqlalchemy.sql.functions.max
```

SQL MAX() 聚合函数。

**类签名**

类`sqlalchemy.sql.functions.max`（`sqlalchemy.sql.functions.ReturnTypeFromArgs`）

```py
class sqlalchemy.sql.functions.min
```

SQL MIN() 聚合函数。

**类签名**

类`sqlalchemy.sql.functions.min`（`sqlalchemy.sql.functions.ReturnTypeFromArgs`）

```py
class sqlalchemy.sql.functions.mode
```

实现 `mode` 有序集合聚合函数。

这个函数必须与`FunctionElement.within_group()`修饰符一起使用，以提供要操作的排序表达式。

此函数的返回类型与排序表达式相同。

**类签名**

类`sqlalchemy.sql.functions.mode`（`sqlalchemy.sql.functions.OrderedSetAgg`）

```py
class sqlalchemy.sql.functions.next_value
```

代表给定`Sequence`作为其唯一参数的‘下一个值’。

在每个后端编译成适当的函数，或者如果在不提供序列支持的后端上使用则会引发 NotImplementedError。

**类签名**

类`sqlalchemy.sql.functions.next_value`(`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.now
```

SQL 现在()日期时间函数。

SQLAlchemy 方言通常以特定于后端的方式呈现此特定函数，例如将其呈现为`CURRENT_TIMESTAMP`。

**类签名**

类`sqlalchemy.sql.functions.now`(`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.percent_rank
```

实现`percent_rank`假设集合聚合函数。

必须使用`FunctionElement.within_group()`修饰符来提供要操作的排序表达式。

此函数的返回类型是`Numeric`。

**类签名**

类`sqlalchemy.sql.functions.percent_rank`(`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.percentile_cont
```

实现`percentile_cont`有序集合聚合函数。

必须使用`FunctionElement.within_group()`修饰符来提供要操作的排序表达式。

此函数的返回类型与排序表达式相同，或者如果参数是数组，则为排序表达式类型的`ARRAY`。

**类签名**

类`sqlalchemy.sql.functions.percentile_cont`(`sqlalchemy.sql.functions.OrderedSetAgg`)

```py
class sqlalchemy.sql.functions.percentile_disc
```

实现`percentile_disc`有序集合聚合函数。

必须使用`FunctionElement.within_group()`修饰符来提供要操作的排序表达式。

此函数的返回类型与排序表达式相同，或者如果参数是数组，则为排序表达式类型的`ARRAY`。

**类签名**

类`sqlalchemy.sql.functions.percentile_disc`(`sqlalchemy.sql.functions.OrderedSetAgg`)

```py
class sqlalchemy.sql.functions.random
```

RANDOM() SQL 函数。

**类签名**

类 `sqlalchemy.sql.functions.random` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.rank
```

实现 `rank` 虚拟集合聚合函数。

此函数必须与 `FunctionElement.within_group()` 修改器一起使用，以提供要操作的排序表达式。

此函数的返回类型是 `Integer`。

**类签名**

类 `sqlalchemy.sql.functions.rank` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.rollup
```

实现 `ROLLUP` 分组操作。

此函数用作语句的 GROUP BY 的一部分，例如 `Select.group_by()`:

```py
stmt = select(
    func.sum(table.c.value), table.c.col_1, table.c.col_2
).group_by(func.rollup(table.c.col_1, table.c.col_2))
```

新版本 1.2 中添加。

**类签名**

类 `sqlalchemy.sql.functions.rollup` (`sqlalchemy.sql.functions.GenericFunction`)

```py
class sqlalchemy.sql.functions.session_user
```

SESSION_USER() SQL 函数。

**类签名**

类 `sqlalchemy.sql.functions.session_user` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.sum
```

SQL SUM() 聚合函数。

**类签名**

类 `sqlalchemy.sql.functions.sum` (`sqlalchemy.sql.functions.ReturnTypeFromArgs`)

```py
class sqlalchemy.sql.functions.sysdate
```

SYSDATE() SQL 函数。

**类签名**

类 `sqlalchemy.sql.functions.sysdate` (`sqlalchemy.sql.functions.AnsiFunction`)

```py
class sqlalchemy.sql.functions.user
```

USER() SQL 函数。

**类签名**

类 `sqlalchemy.sql.functions.user` (`sqlalchemy.sql.functions.AnsiFunction`)
