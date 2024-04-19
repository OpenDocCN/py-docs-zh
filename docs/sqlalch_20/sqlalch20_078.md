# 列元素和表达式

> 原文：[`docs.sqlalchemy.org/en/20/core/sqlelement.html`](https://docs.sqlalchemy.org/en/20/core/sqlelement.html)

表达式 API 由一系列类组成，每个类代表 SQL 字符串中的特定词法元素。将它们组合成一个更大的结构，形成一个语句构造，可以*编译*成一个字符串表示，可以传递给数据库。这些类被组织成一个从最基本的`ClauseElement`类开始的层次结构。关键子类包括`ColumnElement`，它代表 SQL 语句中任何基于列的表达式的角色，例如在列子句、WHERE 子句和 ORDER BY 子句中，以及`FromClause`，它代表放置在 SELECT 语句的 FROM 子句中的令牌的角色。

## 列元素基础构造函数

从`sqlalchemy`命名空间导入的独立函数，用于构建 SQLAlchemy 表达语言构造时使用。

| 对象名称 | 描述 |
| --- | --- |
| and_(*clauses) | 生成一个由`AND`连接的表达式的合取。 |
| bindparam(key[, value, type_, unique, ...]) | 生成一个“绑定表达式”。 |
| bitwise_not(expr) | 生成一个一元按位取反子句，通常通过`~`运算符实现。 |
| case(*whens, [value, else_]) | 生成一个`CASE`表达式。 |
| cast(expression, type_) | 生成一个`CAST`表达式。 |
| column(text[, type_, is_literal, _selectable]) | 生成一个`ColumnClause`对象。 |
| custom_op | 表示一个‘自定义’运算符。 |
| distinct(expr) | 生成一个列表达级别的一元`DISTINCT`子句。 |
| extract(field, expr) | 返回一个`Extract`构造。 |
| false() | 返回一个`False_`构造。 |
| func | 生成 SQL 函数表达式。 |
| lambda_stmt(lmb[, enable_tracking, track_closure_variables, track_on, ...]) | 生成一个被缓存为 lambda 的 SQL 语句。 |
| literal(value[, type_, literal_execute]) | 返回一个文字子句，绑定到绑定参数。 |
| literal_column(text[, type_]) | 生成一个`ColumnClause`对象，其`column.is_literal`标志设置为 True。 |
| not_(clause) | 返回给定子句的否定，即`NOT(clause)`。 |
| null() | 返回一个常量`Null`构造。 |
| or_(*clauses) | 生成由`OR`连接的表达式的合取。 |
| outparam(key[, type_]) | 为可以支持的数据库中的函数（存储过程）创建一个‘OUT’参数。 |
| quoted_name | 表示与引号偏好组合的 SQL 标识符。 |
| text(text) | 构造一个新的`TextClause`子句，直接表示文本型的 SQL 字符串。 |
| true() | 返回一个常量`True_`构造。 |
| try_cast(expression, type_) | 为支持它的后端生成一个`TRY_CAST`表达式；这是一个返回 NULL 的`CAST`，用于不可转换的转换。 |
| tuple_(*clauses, [types]) | 返回一个`Tuple`。 |
| type_coerce(expression, type_) | 将 SQL 表达式与特定类型关联，而不会渲染`CAST`。 |

```py
function sqlalchemy.sql.expression.and_(*clauses)
```

生成由`AND`连接的表达式的合取。

例如：

```py
from sqlalchemy import and_

stmt = select(users_table).where(
                and_(
                    users_table.c.name == 'wendy',
                    users_table.c.enrolled == True
                )
            )
```

`and_()`合取也可使用 Python 的`&`运算符（但请注意，为了与 Python 运算符优先级行为配合使用，复合表达式需要加括号）：

```py
stmt = select(users_table).where(
                (users_table.c.name == 'wendy') &
                (users_table.c.enrolled == True)
            )
```

`and_()`操作在某些情况下也是隐式的；例如，`Select.where()`方法可以对语句多次调用，每个子句都会使用`and_()`结合：

```py
stmt = select(users_table).\
        where(users_table.c.name == 'wendy').\
        where(users_table.c.enrolled == True)
```

`and_()` 构造必须至少给定一个位置参数才能有效；没有参数的`and_()` 构造是模棱两可的。为了生成一个“空”或动态生成的`and_()` 表达式，从给定的表达式列表中，应指定一个“默认”元素`true()`（或只是`True`）：

```py
from sqlalchemy import true
criteria = and_(true(), *expressions)
```

上述表达式将编译为 SQL 表达式`true`或`1 = 1`，取决于后端，如果没有其他表达式存在。如果存在表达式，则`true()`值将被忽略，因为它不影响具有其他元素的 AND 表达式的结果。

自版本 1.4 弃用：`and_()` 元素现在要求至少传递一个参数；创建没有参数的`and_()` 构造已被弃用，并将发出弃用警告，同时继续生成空白的 SQL 字符串。

另请参阅

`or_()`

```py
function sqlalchemy.sql.expression.bindparam(key: str | None, value: Any = _NoArg.NO_ARG, type_: _TypeEngineArgument[_T] | None = None, unique: bool = False, required: bool | Literal[_NoArg.NO_ARG] = _NoArg.NO_ARG, quote: bool | None = None, callable_: Callable[[], Any] | None = None, expanding: bool = False, isoutparam: bool = False, literal_execute: bool = False) → BindParameter[_T]
```

生成一个“绑定表达式”。

返回值是`BindParameter`的一个实例；这是`ColumnElement`的子类，表示 SQL 表达式中的所谓“占位符”值，在执行语句时会根据数据库连接提供的值。

在 SQLAlchemy 中，`bindparam()` 构造具有携带最终在表达式时间使用的实际值的能力。通过这种方式，它不仅作为最终填充的“占位符”，还作为表示所谓“不安全”值的一种方式，这些值不应直接呈现在 SQL 语句中，而应作为需要正确转义和可能处理类型安全性的值传递给 DBAPI。

在显式使用`bindparam()`时，典型用例通常是传统参数的延迟；`bindparam()` 构造接受一个名称，然后可以在执行时引用：

```py
from sqlalchemy import bindparam

stmt = select(users_table).where(
    users_table.c.name == bindparam("username")
)
```

上述语句在渲染时将生成类似于 SQL 的内容：

```py
SELECT id, name FROM user WHERE name = :username
```

为了填充上面的`:username`的值，通常会在执行时将该值应用到`Connection.execute()`方法中：

```py
result = connection.execute(stmt, {"username": "wendy"})
```

当产生要多次调用的 UPDATE 或 DELETE 语句时，明确使用`bindparam()`也很常见，其中语句的 WHERE 条件在每次调用时都会更改，例如：

```py
stmt = (
    users_table.update()
    .where(user_table.c.name == bindparam("username"))
    .values(fullname=bindparam("fullname"))
)

connection.execute(
    stmt,
    [
        {"username": "wendy", "fullname": "Wendy Smith"},
        {"username": "jack", "fullname": "Jack Jones"},
    ],
)
```

SQLAlchemy 的核心表达式系统在隐式意义上广泛使用`bindparam()`。几乎所有 SQL 表达式函数传递给 Python 字面值都会被强制转换为固定的`bindparam()`构造。例如，给定一个比较操作，如下所示：

```py
expr = users_table.c.name == 'Wendy'
```

上述表达式将生成一个`BinaryExpression`结构，其中左侧是代表`name`列的`Column`对象，右侧是代表字面值的`BindParameter`：

```py
print(repr(expr.right))
BindParameter('%(4327771088 name)s', 'Wendy', type_=String())
```

上述表达式将渲染出如下的 SQL：

```py
user.name = :name_1
```

`:name_1`参数名是匿名的。实际字符串`Wendy`不在渲染的字符串中，但在稍后在语句执行中使用时会一起传递。如果我们调用以下语句：

```py
stmt = select(users_table).where(users_table.c.name == 'Wendy')
result = connection.execute(stmt)
```

我们会看到 SQL 日志输出如下：

```py
SELECT "user".id, "user".name
FROM "user"
WHERE "user".name = %(name_1)s
{'name_1': 'Wendy'}
```

上面，我们看到`Wendy`作为参数传递给数据库，而占位符`:name_1`在适当形式下被渲染到目标数据库中，这里是 PostgreSQL 数据库。

类似地，当处理 CRUD 语句的“VALUES”部分时，`bindparam()`会在自动调用时进行处理。`insert()`构造生成一个`INSERT`表达式，在语句执行时，将根据传递的参数生成绑定占位符，如下所示：

```py
stmt = users_table.insert()
result = connection.execute(stmt, {"name": "Wendy"})
```

上述操作将产生如下的 SQL 输出：

```py
INSERT INTO "user" (name) VALUES (%(name)s)
{'name': 'Wendy'}
```

`Insert`构造在编译/执行时，根据我们传递给`Connection.execute()`方法的单个`name`参数，生成一个单个的`bindparam()`镜像列名`name`。

参数：

+   `key` –

    此绑定参数的键（例如名称）。对于使用命名参数的方言，将在生成的 SQL 语句中使用此键。在编译操作的一部分时，如果存在具有相同键的其他 `BindParameter` 对象，或者如果其长度太长并且需要截断，则可能会修改此值。

    如果省略，则为绑定参数生成一个“匿名”名称；当给定一个要绑定的值时，最终结果相当于使用一个值来绑定调用 `literal()` 函数，特别是如果还提供了 `bindparam.unique` 参数。

+   `value` – 此绑定参数的初始值。如果没有为此特定参数名称指示语句执行方法的其他值，则在语句执行时将用作传递给 DBAPI 的此参数的值。默认为 `None`。

+   `callable_` – 一个可调用函数，用于取代“value”。该函数将在语句执行时被调用，以确定最终值。用于无法在创建子句构造时确定实际绑定值，但仍希望使用嵌入式绑定值的情况。

+   `type_` –

    表示此 `bindparam()` 的可选数据类型的 `TypeEngine` 类或实例。如果未传递，则可以根据给定值自动确定绑定的类型；例如，trivial Python 类型，如 `str`、`int`、`bool` 可能会自动选择 `String`、`Integer` 或 `Boolean` 类型。

    `bindparam()` 的类型尤其重要，因为该类型将在将值传递到数据库之前对值进行预处理。例如，引用 datetime 值的 `bindparam()`，并且被指定为持有 `DateTime` 类型，可能会在将值传递到数据库之前对值进行所需的转换（例如在 SQLite 上进行字符串化）。

+   `unique` – 如果为 True，则此 `BindParameter` 的键名将被修改，如果同名的另一个 `BindParameter` 已经存在于包含表达式中。这个标志通常由内部使用，当产生所谓的“匿名”绑定表达式时，它通常不适用于明确命名的 `bindparam()` 构造。

+   `required` – 如果为`True`，则在执行时需要一个值。如果未传递，且既未传递 `bindparam.value` 也未传递 `bindparam.callable`，则默认为`True`。如果存在这些参数中的任何一个，则 `bindparam.required` 默认为`False`。

+   `quote` – 如果此参数名需要引用，并且当前不被认为是 SQLAlchemy 保留字，则为 True；目前仅适用于 Oracle 后端，其中绑定名称有时必须被引用。

+   `isoutparam` – 如果为 True，则应将参数视为存储过程的“OUT”参数。这适用于诸如 Oracle 之类支持 OUT 参数的后端。

+   `expanding` –

    如果为 True，则此参数在执行时将被视为“扩展”参数；参数值应为序列，而不是标量值，并且字符串 SQL 语句将在每次执行时进行转换，以适应具有可变数量参数槽的序列传递给 DBAPI。这是为了允许语句缓存与 IN 子句一起使用。

    另请参阅

    `ColumnOperators.in_()`

    使用 IN 表达式 - 使用烘焙查询

    注意

    “扩展”功能不支持“executemany”式参数集。

    自版本 1.2 新增。

    自版本 1.3 更改：现在，“扩展”绑定参数功能支持空列表。

+   `literal_execute` –

    如果为 True，则绑定参数将在编译阶段呈现为特殊的“POSTCOMPILE”令牌，并且 SQLAlchemy 编译器将在语句执行时将参数的最终值呈现为 SQL 语句，省略了传递给 DBAPI `cursor.execute()` 的参数字典/列表中的值。这产生了与使用 `literal_binds` 编译标志类似的效果，但是发生在将语句发送到 DBAPI `cursor.execute()` 方法时，而不是在语句编译时。此功能的主要用途是在数据库驱动程序无法在这些上下文中容纳绑定参数的情况下呈现 LIMIT/OFFSET 子句，同时允许 SQL 结构在编译级别可缓存。

    版本 1.4 中的新功能：添加了“编译后”绑定参数

    另请参阅

    Oracle、SQL Server 中用于 LIMIT/OFFSET 的新的“编译后”绑定参数。

另请参阅

发送参数 - 在 SQLAlchemy 统一教程 中

```py
function sqlalchemy.sql.expression.bitwise_not(expr: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

生成一元位取反子句，通常通过 `~` 运算符实现。

不要与布尔取反 `not_()` 混淆。

版本 2.0.2 中的新功能。

另请参阅

位运算符

```py
function sqlalchemy.sql.expression.case(*whens: typing_Tuple[_ColumnExpressionArgument[bool], Any] | Mapping[Any, Any], value: Any | None = None, else_: Any | None = None) → Case[Any]
```

生成一个 `CASE` 表达式。

SQL 中的 `CASE` 结构是一个条件对象，其作用类似于其他语言中的“if/then”结构。它返回一个 `Case` 的实例。

`case()` 通常形式下传递了一系列“when”构造，即一系列条件和结果的元组：

```py
from sqlalchemy import case

stmt = select(users_table).\
            where(
                case(
                    (users_table.c.name == 'wendy', 'W'),
                    (users_table.c.name == 'jack', 'J'),
                    else_='E'
                )
            )
```

上述语句将生成类似于以下的 SQL：

```py
SELECT id, name FROM user
WHERE CASE
    WHEN (name = :name_1) THEN :param_1
    WHEN (name = :name_2) THEN :param_2
    ELSE :param_3
END
```

当需要针对单个父列的几个值进行简单等式表达式时，`case()` 也有一种“简写”格式，通过 `case.value` 参数传递，该参数传递了要比较的列表达式。在这种形式下，`case.whens` 参数作为包含要与键入结果表达式比较的表达式的字典传递。下面的语句与前面的语句等效：

```py
stmt = select(users_table).\
            where(
                case(
                    {"wendy": "W", "jack": "J"},
                    value=users_table.c.name,
                    else_='E'
                )
            )
```

在 `case.whens` 中作为结果值接受的值以及与 `case.else_` 相同的值从 Python 文字转换为 `bindparam()` 构造。 SQL 表达式，例如 `ColumnElement` 构造，也被接受。要将文字字符串表达式强制转换为内联渲染的常量表达式，请使用 `literal_column()` 构造，例如：

```py
from sqlalchemy import case, literal_column

case(
    (
        orderline.c.qty > 100,
        literal_column("'greaterthan100'")
    ),
    (
        orderline.c.qty > 10,
        literal_column("'greaterthan10'")
    ),
    else_=literal_column("'lessthan10'")
)
```

以上将渲染给定的常量，而不是使用绑定参数作为结果值（但仍然适用于比较值），例如：

```py
CASE
    WHEN (orderline.qty > :qty_1) THEN 'greaterthan100'
    WHEN (orderline.qty > :qty_2) THEN 'greaterthan10'
    ELSE 'lessthan10'
END
```

参数：

+   `*whens` –

    要进行比较的条件，`case.whens` 接受两种不同形式，基于是否使用了 `case.value`。

    1.4 版更改：`case()` 函数现在按位置接受 WHEN 条件的系列。

    在第一种形式中，它接受多个以位置参数传递的 2 元组；每个 2 元组由 `(<sql expression>, <value>)` 组成，其中 SQL 表达式是布尔表达式，而“value”是一个结果值，例如：

    ```py
    case(
        (users_table.c.name == 'wendy', 'W'),
        (users_table.c.name == 'jack', 'J')
    )
    ```

    在第二种形式中，它接受一个将比较值映射到结果值的 Python 字典；此形式需要 `case.value` 存在，并且值将使用 `==` 运算符进行比较，例如：

    ```py
    case(
        {"wendy": "W", "jack": "J"},
        value=users_table.c.name
    )
    ```

+   `value` – 一个可选的 SQL 表达式，它将作为传递给 `case.whens` 的字典中的候选值的固定“比较点”。

+   `else_` – 一个可选的 SQL 表达式，如果 `case.whens` 中的所有表达式都为 false，则将其作为 `CASE` 构造的评估结果。如果省略，大多数数据库在所有“when”表达式均未评估为 true 时会产生 NULL 结果。

```py
function sqlalchemy.sql.expression.cast(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → Cast[_T]
```

生成一个 `CAST` 表达式。

`cast()` 返回 `Cast` 的实例。

例如：

```py
from sqlalchemy import cast, Numeric

stmt = select(cast(product_table.c.unit_price, Numeric(10, 4)))
```

上述语句将产生类似的 SQL：

```py
SELECT CAST(unit_price AS NUMERIC(10, 4)) FROM product
```

当使用时，`cast()` 函数执行两个不同的功能。第一个是它在结果 SQL 字符串中呈现 `CAST` 表达式。第二个是它将给定类型（例如`TypeEngine`类或实例）与 Python 端的列表达式关联起来，这意味着表达式将采用与该类型关联的表达式运算符行为，以及该类型的绑定值处理和结果行处理行为。

`cast()`的替代方法是`type_coerce()`函数。该函数执行了将表达式与特定类型关联的第二个任务，但不会在 SQL 中呈现 `CAST` 表达式。

参数：

+   `expression` – 一个 SQL 表达式，比如一个 `ColumnElement` 表达式或一个将被强制转换为绑定文字值的 Python 字符串。

+   `type_` – 一个指示 `CAST` 应用的类型的`TypeEngine`类或实例。

另请参阅

数据转换和类型强制

`try_cast()` - CAST 的一种替代方法，当转换失败时返回 NULL，而不是引发错误。仅受一些方言支持。

`type_coerce()` - CAST 的替代方法，仅在 Python 端强制转换类型，这通常足以生成正确的 SQL 和数据强制转换。

```py
function sqlalchemy.sql.expression.column(text: str, type_: _TypeEngineArgument[_T] | None = None, is_literal: bool = False, _selectable: FromClause | None = None) → ColumnClause[_T]
```

生成一个 `ColumnClause` 对象。

`ColumnClause` 是 `Column` 类的轻量级模拟。`column()` 函数可以仅使用名称调用，如下所示：

```py
from sqlalchemy import column

id, name = column("id"), column("name")
stmt = select(id, name).select_from("user")
```

上述语句将生成类似于以下的 SQL：

```py
SELECT id, name FROM user
```

构造完毕后，`column()` 可以像其他 SQL 表达式元素一样使用，比如在 `select()` 构造中：

```py
from sqlalchemy.sql import column

id, name = column("id"), column("name")
stmt = select(id, name).select_from("user")
```

由 `column()` 处理的文本被假定处理为数据库列的名称；如果字符串包含混合大小写、特殊字符，或与目标后端的已知保留字匹配，则列表达式将使用由后端确定的引用行为渲染。若要生成一个确切的文本 SQL 表达式而不应用任何引用，请改用 `literal_column()`，或将 `column.is_literal` 的值传递为 `True`。此外，完整的 SQL 语句最好使用 `text()` 构造处理。

`column()` 可以通过与 `table()` 函数组合（它是 `Table` 的轻量级类比）以表格方式使用，以产生一个带有最小样板的可用表构造：

```py
from sqlalchemy import table, column, select

user = table("user",
        column("id"),
        column("name"),
        column("description"),
)

stmt = select(user.c.description).where(user.c.name == 'wendy')
```

像上面示例的 `column()` / `table()` 构造可以以临时方式创建，并且与任何 `MetaData`、DDL 或事件无关，不像它的 `Table` 对应物。

参数：

+   `text` – 元素的文本。

+   `type` – `TypeEngine` 对象，可以将此 `ColumnClause` 与类型关联。

+   `is_literal` – 如果为 True，则假定 `ColumnClause` 是一个精确的表达式，将以不考虑大小写设置的情况下，无论如何都不会应用任何引用规则传递到输出。 `literal_column()` 函数本质上调用 `column()`，同时传递 `is_literal=True`。

另请参阅

`Column`

`literal_column()`

`table()`

`text()`

使用文本列表达式进行选择

```py
class sqlalchemy.sql.expression.custom_op
```

表示“自定义”运算符。

当使用`Operators.op()`或`Operators.bool_op()`方法创建自定义运算符可调用时，通常会实例化`custom_op`。该类也可以直接在编程构建表达式时使用。例如，表示“阶乘”操作：

```py
from sqlalchemy.sql import UnaryExpression
from sqlalchemy.sql import operators
from sqlalchemy import Numeric

unary = UnaryExpression(table.c.somecolumn,
        modifier=operators.custom_op("!"),
        type_=Numeric)
```

另请参阅

`Operators.op()`

`Operators.bool_op()`

**类签名**

类`sqlalchemy.sql.expression.custom_op`（`sqlalchemy.sql.expression.OperatorType`，`typing.Generic`）

```py
function sqlalchemy.sql.expression.distinct(expr: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

生成一个列表达式级别的一元`DISTINCT`子句。

这将在**单独的列表达式**上应用`DISTINCT`关键字（例如，不是整个语句），并**特定地在该列位置呈现**；这用于在聚合函数内部进行包含，如：

```py
from sqlalchemy import distinct, func
stmt = select(users_table.c.id, func.count(distinct(users_table.c.name)))
```

上述将产生类似于以下语句：

```py
SELECT user.id, count(DISTINCT user.name) FROM user
```

提示

`distinct()`函数**不会**将 DISTINCT 应用于完整的 SELECT 语句，而是将 DISTINCT 修饰符应用于**单独的列表达式**。要获得一般的`SELECT DISTINCT`支持，请在`Select`上使用`Select.distinct()`方法。

`distinct()`函数也可作为列级方法使用，例如`ColumnElement.distinct()`，如下所示：

```py
stmt = select(func.count(users_table.c.name.distinct()))
```

`distinct()`运算符与`Select.distinct()`方法不同，后者会在整个结果集上应用`DISTINCT`，例如`SELECT DISTINCT`表达式。有关更多信息，请参阅该方法。

另请参阅

`ColumnElement.distinct()`

`Select.distinct()`

`func`

```py
function sqlalchemy.sql.expression.extract(field: str, expr: _ColumnExpressionArgument[Any]) → Extract
```

返回一个 `Extract` 结构。

通常可从 `func` 命名空间中的 `extract()` 或 `func.extract` 中获得此功能。

参数：

+   `field` – 要提取的字段。

+   `expr` – 作为 `EXTRACT` 表达式右侧的列或 Python 标量表达式。

例如：

```py
from sqlalchemy import extract
from sqlalchemy import table, column

logged_table = table("user",
        column("id"),
        column("date_created"),
)

stmt = select(logged_table.c.id).where(
    extract("YEAR", logged_table.c.date_created) == 2021
)
```

在上面的示例中，该语句用于从数据库中选择 `YEAR` 组件与特定值匹配的 id。

同样，也可以选择提取的组件：

```py
stmt = select(
    extract("YEAR", logged_table.c.date_created)
).where(logged_table.c.id == 1)
```

`EXTRACT` 的实现可能因数据库后端而异。提醒用户查阅其数据库文档。

```py
function sqlalchemy.sql.expression.false() → False_
```

返回一个 `False_` 结构。

例如：

```py
>>> from sqlalchemy import false
>>> print(select(t.c.x).where(false()))
SELECT  x  FROM  t  WHERE  false 
```

如果后端不支持真/假常量，将呈现为针对 1 或 0 的表达式：

```py
>>> print(select(t.c.x).where(false()))
SELECT  x  FROM  t  WHERE  0  =  1 
```

`true()` 和 `false()` 常量也在 `and_()` 或 `or_()` 连接中具有“短路”操作：

```py
>>> print(select(t.c.x).where(or_(t.c.x > 5, true())))
SELECT  x  FROM  t  WHERE  true
>>> print(select(t.c.x).where(and_(t.c.x > 5, false())))
SELECT  x  FROM  t  WHERE  false 
```

另请参阅

`true()`

```py
sqlalchemy.sql.expression.func = <sqlalchemy.sql.functions._FunctionGenerator object>
```

生成 SQL 函数表达式。

`func` 是一个特殊的对象实例，根据基于名称的属性生成 SQL 函数，例如：

```py
>>> print(func.count(1))
count(:param_1) 
```

返回的对象是 `Function` 的实例，并且是一个面向列的 SQL 元素，可以以这种方式使用：

```py
>>> print(select(func.count(table.c.id)))
SELECT  count(sometable.id)  FROM  sometable 
```

`func` 可以指定任何名称。如果 SQLAlchemy 不认识函数名称，则会按原样呈现。对于 SQLAlchemy 认识的常见 SQL 函数，名称可能被解释为*通用函数*，将根据目标数据库适当编译：

```py
>>> print(func.current_timestamp())
CURRENT_TIMESTAMP 
```

要调用点分包中存在的函数，以相同的方式指定它们：

```py
>>> print(func.stats.yield_curve(5, 10))
stats.yield_curve(:yield_curve_1,  :yield_curve_2) 
```

SQLAlchemy 可以了解函数的返回类型，以启用特定类型的词法和基于结果的行为。例如，要确保基于字符串的函数返回 Unicode 值，并且在表达式中类似地对待为字符串，请将 `Unicode` 指定为类型：

```py
>>> print(func.my_string(u'hi', type_=Unicode) + ' ' +
...       func.my_string(u'there', type_=Unicode))
my_string(:my_string_1)  ||  :my_string_2  ||  my_string(:my_string_3) 
```

通过 `func` 调用返回的对象通常是 `Function` 的实例。该对象符合“列”接口，包括比较和标记函数。该对象也可以传递给 `Connection` 或 `Engine` 的 `Connectable.execute()` 方法，它首先将被包装在 SELECT 语句中：

```py
print(connection.execute(func.current_timestamp()).scalar())
```

在一些例外情况下，`func` 访问器将名称重定向到内置表达式，如 `cast()` 或 `extract()` ，因为这些名称具有众所周知的含义，但从 SQLAlchemy 的角度来看并不完全相同于“函数”。

被解释为“通用”函数的函数知道如何自动计算它们的返回类型。有关已知通用函数的列表，请参阅 SQL 和通用函数。

注意

`func` 结构对于调用独立的“存储过程”仅有有限的支持，特别是那些具有特殊参数化关注的情况。

有关如何使用 DBAPI 级别的 `callproc()` 方法来完全传统的存储过程的详细信息，请参阅 调用存储过程和用户定义的函数 部分。

另请参阅

使用 SQL 函数 - 在 SQLAlchemy 统一教程 中

`Function`

```py
function sqlalchemy.sql.expression.lambda_stmt(lmb: Callable[[], Any], enable_tracking: bool = True, track_closure_variables: bool = True, track_on: object | None = None, global_track_bound_values: bool = True, track_bound_values: bool = True, lambda_cache: MutableMapping[Tuple[Any, ...], NonAnalyzedFunction | AnalyzedFunction] | None = None) → StatementLambdaElement
```

将一个 SQL 语句制作成一个缓存为 lambda。

lambda 中的 Python 代码对象会被扫描，其中既包括将变为绑定参数的 Python 字面量，也包括引用可能不同的 Core 或 ORM 构造的闭包变量。 lambda 本身仅在检测到特定一组构造时才会被调用一次。

例如：

```py
from sqlalchemy import lambda_stmt

stmt = lambda_stmt(lambda: table.select())
stmt += lambda s: s.where(table.c.id == 5)

result = connection.execute(stmt)
```

返回的对象是 `StatementLambdaElement` 的实例。

在版本 1.4 中新增。

参数：

+   `lmb` – 一个 Python 函数，通常是一个 lambda，不接受任何参数并返回一个 SQL 表达式构造

+   `enable_tracking` – 当为 False 时，所有对给定 lambda 的扫描，以检测闭包变量或绑定参数的更改都将被禁用。对于在所有情况下都产生相同结果且没有参数化的 lambda 使用。

+   `track_closure_variables` – 当为 False 时，将不会扫描 lambda 中闭包变量的更改。用于闭包变量的状态永远不会改变 lambda 返回的 SQL 结构的 lambda。

+   `track_bound_values` – 当为 False 时，将禁用给定 lambda 的绑定参数跟踪。用于不生成任何绑定值的 lambda，或者初始绑定值永远不会更改的 lambda。

+   `global_track_bound_values` – 当为 False 时，整个语句的绑定参数跟踪将被禁用，包括通过`StatementLambdaElement.add_criteria()`方法添加的其他链接。

+   `lambda_cache` – 一个字典或其他类似映射的对象，其中将存储有关 lambda 的 Python 代码以及 lambda 本身中跟踪的闭包变量的信息。默认为全局 LRU 缓存。此缓存独立于 `Connection` 对象使用的“compiled_cache”。

另请参阅

使用 Lambdas 为语句生成添加显著的速度增益

```py
function sqlalchemy.sql.expression.literal(value: Any, type_: _TypeEngineArgument[Any] | None = None, literal_execute: bool = False) → BindParameter[Any]
```

返回一个文字子句，绑定到一个绑定参数。

当使用非`ClauseElement`对象（例如字符串、整数、日期等）与`ColumnElement`子类（如 `Column` 对象）进行比较操作时，会自动创建文字子句。使用此函数强制生成文字子句，它将作为具有绑定值的 `BindParameter` 创建。

参数：

+   `value` – 要绑定的值。可以是底层 DB-API 支持的任何 Python 对象，或者可以通过给定的类型参数进行翻译。

+   `type_` – 一个可选的`TypeEngine`，它将为此文字提供绑定参数翻译。

+   `literal_execute` –

    可选布尔值，当为 True 时，SQL 引擎将尝试在执行时直接在 SQL 语句中渲染绑定值，而不是提供作为参数值。

    新版本 2.0 中新增。

```py
function sqlalchemy.sql.expression.literal_column(text: str, type_: _TypeEngineArgument[_T] | None = None) → ColumnClause[_T]
```

生成一个具有 `column.is_literal` 标志设置为 True 的 `ColumnClause` 对象。

`literal_column()` 类似于 `column()`，但更常用作“独立的”列表达式，其呈现方式与所述完全相同；而 `column()` 存储一个假设为表的一部分并可能被引用为此类的字符串名称， `literal_column()` 可以是那个，或任何其他任意的面向列的表达式。

参数：

+   `text` – 表达式的文本；可以是任何 SQL 表达式。引号规则不会应用。要指定应受引号规则约束的列名表达式，请使用 `column()` 函数。

+   `type_` – 一个可选的 `TypeEngine` 对象，将为该列提供结果集转换和附加表达语义。如果保持为 `None`，类型将是 `NullType`。

另请参阅

`column()`

`text()`

使用文本列表达式进行选择

```py
function sqlalchemy.sql.expression.not_(clause: _ColumnExpressionArgument[_T]) → ColumnElement[_T]
```

返回给定子句的否定，即 `NOT(clause)`。

`~` 操作符也对所有 `ColumnElement` 子类进行了重载，以产生相同的结果。

```py
function sqlalchemy.sql.expression.null() → Null
```

返回一个常量 `Null` 结构。

```py
function sqlalchemy.sql.expression.or_(*clauses)
```

生成由 `OR` 连接的表达式。

例如：

```py
from sqlalchemy import or_

stmt = select(users_table).where(
                or_(
                    users_table.c.name == 'wendy',
                    users_table.c.name == 'jack'
                )
            )
```

`or_()` 连接词也可使用 Python 的 `|` 操作符（但请注意，复合表达式需要用括号括起来以与 Python 运算符优先级行为一起使用）：

```py
stmt = select(users_table).where(
                (users_table.c.name == 'wendy') |
                (users_table.c.name == 'jack')
            )
```

`or_()` 结构必须至少给定一个位置参数才能有效；没有参数的 `or_()` 结构是模棱两可的。为了生成一个“空”的或动态生成的 `or_()` 表达式，从给定的表达式列表中，应指定一个“默认”元素 `false()` （或只是 `False`）：

```py
from sqlalchemy import false
or_criteria = or_(false(), *expressions)
```

如果没有其他表达式存在，则上述表达式将编译为 SQL 表达式`false`或`0 = 1`，取决于后端。 如果存在表达式，则`false()`值将被忽略，因为它不影响具有其他元素的 OR 表达式的结果。

自版本 1.4 起弃用：`or_()`元素现在要求至少传递一个参数；创建没有参数的`or_()`结构已被弃用，并将发出弃用警告，同时继续生成空白 SQL 字符串。

另请参阅

`and_()`

```py
function sqlalchemy.sql.expression.outparam(key: str, type_: TypeEngine[_T] | None = None) → BindParameter[_T]
```

为在支持它们的数据库中使用的函数（存储过程）创建一个“OUT”参数。

`outparam`可以像常规函数参数一样使用。 “输出”值将通过其`out_parameters`属性从`CursorResult`对象中获取，该属性返回一个包含值的字典。

```py
function sqlalchemy.sql.expression.text(text: str) → TextClause
```

构建一个新的`TextClause`子句，直接表示文本 SQL 字符串。

例如：

```py
from sqlalchemy import text

t = text("SELECT * FROM users")
result = connection.execute(t)
```

`text()`相对于普通字符串提供的优势是，后端中立地支持绑定参数，每个语句的执行选项，以及绑定参数和结果列类型行为，允许 SQLAlchemy 类型构造在执行字面上指定的语句时发挥作用。该构造还可以提供一个`.c`列元素集合，允许将其嵌入到其他 SQL 表达式构造中作为子查询。

通过名称指定绑定参数，使用格式`:name`。 例如：

```py
t = text("SELECT * FROM users WHERE id=:user_id")
result = connection.execute(t, {"user_id": 12})
```

对于需要直接使用冒号的 SQL 语句，如在内联字符串中，使用反斜杠进行转义：

```py
t = text(r"SELECT * FROM users WHERE name='\:username'")
```

`TextClause`构造包括可以提供有关绑定参数的信息以及从文本语句返回的列值的方法，假设它是可执行的 SELECT 类型语句。 `TextClause.bindparams()`方法用于提供绑定参数详细信息，`TextClause.columns()`方法允许指定返回列，包括名称和类型：

```py
t = text("SELECT * FROM users WHERE id=:user_id").\
        bindparams(user_id=7).\
        columns(id=Integer, name=String)

for id, name in connection.execute(t):
    print(id, name)
```

在指定较大查询的一部分（例如 SELECT 语句的 WHERE 子句）时，使用`text()`构造表示文本字符串 SQL 片段的情况：

```py
s = select(users.c.id, users.c.name).where(text("id=:user_id"))
result = connection.execute(s, {"user_id": 12})
```

`text()`还用于构建完整的、独立的文本语句。因此，SQLAlchemy 将其称为`Executable`对象，并且可以像传递给`.execute()`方法的任何其他语句一样使用。

参数：

**text** –

要创建的 SQL 语句的文本。使用`:<param>`指定绑定参数；它们将被编译为特定于引擎的格式。

警告

`text()`函数的`text.text`参数可以作为 Python 字符串参数传递，它将被视为**受信任的 SQL 文本**并按照给定的方式呈现。**不要将不受信任的输入传递给此参数**。

亦可参见

使用文本列表达式进行选择

```py
function sqlalchemy.sql.expression.true() → True_
```

返回一个常量`True_`构造。

例如：

```py
>>> from sqlalchemy import true
>>> print(select(t.c.x).where(true()))
SELECT  x  FROM  t  WHERE  true 
```

不支持 true/false 常量的后端将呈现为针对 1 或 0 的表达式：

```py
>>> print(select(t.c.x).where(true()))
SELECT  x  FROM  t  WHERE  1  =  1 
```

`true()`和`false()`常量在`and_()`或`or_()`连接中也具有“短路”操作：

```py
>>> print(select(t.c.x).where(or_(t.c.x > 5, true())))
SELECT  x  FROM  t  WHERE  true
>>> print(select(t.c.x).where(and_(t.c.x > 5, false())))
SELECT  x  FROM  t  WHERE  false 
```

亦可参见

`false()`

```py
function sqlalchemy.sql.expression.try_cast(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → TryCast[_T]
```

为支持它的后端生成一个`TRY_CAST`表达式；这是一个返回不可转换的转换为 NULL 的`CAST`。

在 SQLAlchemy 中，这种构造仅由 SQL Server 方言支持，如果在其他包含的后端使用将引发`CompileError`。但是，第三方后端也可能支持此构造。

提示

由于`try_cast()`源自 SQL Server 方言，因此可以从`sqlalchemy.`以及`sqlalchemy.dialects.mssql`导入它。

`try_cast()` 返回 `TryCast` 的实例，并且通常行为类似于 `Cast` 构造；在 SQL 级别上，`CAST` 和 `TRY_CAST` 之间的区别是 `TRY_CAST` 对于无法转换的表达式（例如尝试将字符串`"hi"`转换为整数值）返回 NULL。

例如：

```py
from sqlalchemy import select, try_cast, Numeric

stmt = select(
    try_cast(product_table.c.unit_price, Numeric(10, 4))
)
```

在 Microsoft SQL Server 上，上述内容将呈现为：

```py
SELECT TRY_CAST (product_table.unit_price AS NUMERIC(10, 4))
FROM product_table
```

从版本 2.0.14 开始：`try_cast()` 已从 SQL Server 方言泛化为一个通用构造，可能会受到其他方言的支持。

```py
function sqlalchemy.sql.expression.tuple_(*clauses: _ColumnExpressionArgument[Any], types: Sequence[_TypeEngineArgument[Any]] | None = None) → Tuple
```

返回一个 `Tuple`。

主要用途是使用`ColumnOperators.in_()`生成复合的 IN 构造。

```py
from sqlalchemy import tuple_

tuple_(table.c.col1, table.c.col2).in_(
    [(1, 2), (5, 12), (10, 19)]
)
```

在版本 1.3.6 中更改：添加对 SQLite IN 元组的支持。

警告

不是所有后端都支持复合的 IN 构造，目前已知的仅支持 PostgreSQL、MySQL 和 SQLite。当调用这样的表达式时，不支持的后端将引发 `DBAPIError` 的子类。

```py
function sqlalchemy.sql.expression.type_coerce(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → TypeCoerce[_T]
```

将 SQL 表达式与特定类型关联，而不会渲染`CAST`。

例如：

```py
from sqlalchemy import type_coerce

stmt = select(type_coerce(log_table.date_string, StringDateTime()))
```

上述构造将生成一个 `TypeCoerce` 对象，在 SQL 方面不会以任何方式修改呈现，可能的例外情况是在列子句上下文中使用时生成的标签：

```py
SELECT  date_string  AS  date_string  FROM  log
```

当提取结果行时，`StringDateTime` 类型处理器将代表`date_string`列应用于结果行。

注意

`type_coerce()` 构造不会渲染任何自己的 SQL 语法，包括不会暗示添加括号。如果需要显式添加括号，请使用 `TypeCoerce.self_group()`。

为了为表达式提供一个命名标签，请使用 `ColumnElement.label()`：

```py
stmt = select(
    type_coerce(log_table.date_string, StringDateTime()).label('date')
)
```

具有绑定值处理功能的类型在将文字值或`bindparam()`构造传递给`type_coerce()`作为目标时也会产生该行为。例如，如果一个类型实现了`TypeEngine.bind_expression()`方法或`TypeEngine.bind_processor()`方法或等效方法，这些函数将在语句编译/执行时生效，当传递文字值时，如：

```py
# bound-value handling of MyStringType will be applied to the
# literal value "some string"
stmt = select(type_coerce("some string", MyStringType))
```

在使用`type_coerce()`与组合表达式时，请注意**不要应用括号**。如果在需要 CAST 中通常存在的括号的运算符上下文中使用`type_coerce()`，请使用`TypeCoerce.self_group()`方法：

```py
>>> some_integer = column("someint", Integer)
>>> some_string = column("somestr", String)
>>> expr = type_coerce(some_integer + 5, String) + some_string
>>> print(expr)
someint  +  :someint_1  ||  somestr
>>> expr = type_coerce(some_integer + 5, String).self_group() + some_string
>>> print(expr)
(someint  +  :someint_1)  ||  somestr 
```

参数：

+   `expression` – 一个 SQL 表达式，例如一个`ColumnElement`表达式或一个将被强制转换为绑定文字值的 Python 字符串。

+   `type_` – 一个指示表达式被强制转换为的类型的`TypeEngine`类或实例。

另请参阅

数据转换和类型强制转换

`cast()`

```py
class sqlalchemy.sql.expression.quoted_name
```

表示与引号偏好结合的 SQL 标识符。

`quoted_name`是一个表示特定标识符名称的 Python unicode/str 子类，以及一个`quote`标志。当将此`quote`标志设置为`True`或`False`时，将覆盖此标识符的自动引号行为，以便无条件引用或不引用名称。如果保持默认值`None`，则引号行为将根据对标记本身的检查在每个后端基础上应用于标识符。

当 `quoted_name` 对象的 `quote=True` 时，还会防止出现所谓的“名称规范化”选项。某些数据库后端（如 Oracle、Firebird 和 DB2）将不区分大小写的名称规范化为大写。这些后端的 SQLAlchemy 方言将从 SQLAlchemy 的小写表示不敏感约定转换为这些后端的大写表示不敏感约定。在此处的 `quote=True` 标志将阻止此转换发生，以支持针对此类后端引用的标识符为全部小写。

`quoted_name` 对象通常在指定键架构构造的名称时自动生成，如 `Table`、`Column` 等。该类也可以显式地作为任何接收可引用名称的函数的名称传递。例如，要在未经条件引用的名称上使用 `Engine.has_table()` 方法：

```py
from sqlalchemy import create_engine
from sqlalchemy import inspect
from sqlalchemy.sql import quoted_name

engine = create_engine("oracle+cx_oracle://some_dsn")
print(inspect(engine).has_table(quoted_name("some_table", True)))
```

上述逻辑将针对 Oracle 后端运行“has table”逻辑，将名称传递为 `"some_table"`，而不会转换为大写。

从版本 1.2 开始更改：`quoted_name` 构造现在可以从 `sqlalchemy.sql` 导入，而不仅仅是以前的位置 `sqlalchemy.sql.elements`。

**成员**

quote

**类签名**

类 `sqlalchemy.sql.expression.quoted_name` (`sqlalchemy.util.langhelpers.MemoizedSlots`, `builtins.str`)

```py
attribute quote
```

字符串是否应该被无条件引用  ## Column Element Modifier Constructors

此处列出的函数通常作为任何 `ColumnElement` 构造的方法更常见。例如，`label()` 函数通常通过 `ColumnElement.label()` 方法调用。

| 对象名称 | 描述 |
| --- | --- |
| all_(expr) | 生成一个 ALL 表达式。 |
| any_(expr) | 生成一个 ANY 表达式。 |
| asc(column) | 生成一个升序的 `ORDER BY` 子句元素。 |
| between(expr, lower_bound, upper_bound[, symmetric]) | 生成一个 `BETWEEN` 谓词子句。 |
| collate(expression, collation) | 返回子句 `expression COLLATE collation`。 |
| desc(column) | 生成一个降序的 `ORDER BY` 子句元素。 |
| funcfilter(func, *criterion) | 生成一个 `FunctionFilter` 对象来对函数进行操作。 |
| label(name, element[, type_]) | 返回给定 `ColumnElement` 的 `Label` 对象。 |
| nulls_first(column) | 生成一个用于 `ORDER BY` 表达式的 `NULLS FIRST` 修饰符。 |
| nulls_last(column) | 生成一个用于 `ORDER BY` 表达式的 `NULLS LAST` 修饰符。 |
| nullsfirst | `nulls_first()` 函数的同义词。 |
| nullslast | `nulls_last()` 函数的传统同义词。 |
| over(element[, partition_by, order_by, range_, ...]) | 生成一个 `Over` 对象来对函数进行操作。 |
| within_group(element, *order_by) | 生成一个 `WithinGroup` 对象来对函数进行操作。 |

```py
function sqlalchemy.sql.expression.all_(expr: _ColumnExpressionArgument[_T]) → CollectionAggregate[bool]
```

生成一个 ALL 表达式。

对于像 PostgreSQL 的方言，这个运算符适用于 `ARRAY` 数据类型的用法，对于 MySQL，它可能适用于子查询。例如：

```py
# renders on PostgreSQL:
# '5 = ALL (somearray)'
expr = 5 == all_(mytable.c.somearray)

# renders on MySQL:
# '5 = ALL (SELECT value FROM table)'
expr = 5 == all_(select(table.c.value))
```

与 NULL 的比较可以使用 `None`：

```py
None == all_(mytable.c.somearray)
```

`any_()` / `all_()` 运算符还具有特殊的“操作数翻转”行为，即如果 `any_()` / `all_()` 用于比较的左侧是独立运算符，如 `==`、`!=` 等（不包括操作符方法，比如 `ColumnOperators.is_()`），则渲染的表达式会被翻转：

```py
# would render '5 = ALL (column)`
all_(mytable.c.column) == 5
```

或者使用 `None`，需要注意的是，通常情况下不会像对待 NULL 一样渲染 “IS”：

```py
# would render 'NULL = ALL(somearray)'
all_(mytable.c.somearray) == None
```

从版本 1.4.26 开始更改：修复了 any_() / all_() 将 NULL 与右侧比较时翻转为左侧的问题。

列级别的 `ColumnElement.all_()` 方法（不要与 `ARRAY` 级别的 `Comparator.all()` 混淆）是 `all_(col)` 的速记形式：

```py
5 == mytable.c.somearray.all_()
```

参见

`ColumnOperators.all_()`

`any_()`

```py
function sqlalchemy.sql.expression.any_(expr: _ColumnExpressionArgument[_T]) → CollectionAggregate[bool]
```

生成一个 ANY 表达式。

对于像 PostgreSQL 这样的方言，此运算符适用于 `ARRAY` 数据类型的使用，对于 MySQL，它可能适用于子查询。例如：

```py
# renders on PostgreSQL:
# '5 = ANY (somearray)'
expr = 5 == any_(mytable.c.somearray)

# renders on MySQL:
# '5 = ANY (SELECT value FROM table)'
expr = 5 == any_(select(table.c.value))
```

使用 `None` 或 `null()` 可能会与 NULL 进行比较：

```py
None == any_(mytable.c.somearray)
```

any_() / all_() 运算符还具有特殊的“操作数翻转”行为，即如果 any_() / all_() 用于比较的左侧，使用独立运算符如 `==`、`!=` 等（不包括操作符方法如 `ColumnOperators.is_()`）则渲染的表达式会被翻转：

```py
# would render '5 = ANY (column)`
any_(mytable.c.column) == 5
```

或者使用 `None`，这个注释不会执行通常用于 NULL 的“IS”渲染步骤：

```py
# would render 'NULL = ANY(somearray)'
any_(mytable.c.somearray) == None
```

在版本 1.4.26 中更改：修复了 any_() / all_() 与右侧 NULL 进行比较时翻转到左侧的问题。

列级别的 `ColumnElement.any_()` 方法（不要与 `ARRAY` 级别的 `Comparator.any()` 混淆）是 `any_(col)` 的简写：

```py
5 = mytable.c.somearray.any_()
```

另请参见

`ColumnOperators.any_()`

`all_()`

```py
function sqlalchemy.sql.expression.asc(column: _ColumnExpressionOrStrLabelArgument[_T]) → UnaryExpression[_T]
```

生成一个升序的 `ORDER BY` 子句元素。

例如：

```py
from sqlalchemy import asc
stmt = select(users_table).order_by(asc(users_table.c.name))
```

将产生 SQL 如下：

```py
SELECT id, name FROM user ORDER BY name ASC
```

`asc()` 函数是所有 SQL 表达式上可用的 `ColumnElement.asc()` 方法的独立版本，例如：

```py
stmt = select(users_table).order_by(users_table.c.name.asc())
```

参数：

**column** – 一个 `ColumnElement`（例如标量 SQL 表达式），用于应用 `asc()` 操作。

另请参见

`desc()`

`nulls_first()`

`nulls_last()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.between(expr: _ColumnExpressionOrLiteralArgument[_T], lower_bound: Any, upper_bound: Any, symmetric: bool = False) → BinaryExpression[bool]
```

生成一个 `BETWEEN` 谓词子句。

例如：

```py
from sqlalchemy import between
stmt = select(users_table).where(between(users_table.c.id, 5, 7))
```

会产生类似的 SQL：

```py
SELECT id, name FROM user WHERE id BETWEEN :id_1 AND :id_2
```

`between()`函数是所有 SQL 表达式上都可用的`ColumnElement.between()`方法的独立版本，如下所示：

```py
stmt = select(users_table).where(users_table.c.id.between(5, 7))
```

所有传递给`between()`的参数，包括左侧列表达式，如果值不是`ColumnElement`子类，则从 Python 标量值转换。例如，可以像这样比较三个固定值：

```py
print(between(5, 3, 7))
```

这将产生：

```py
:param_1 BETWEEN :param_2 AND :param_3
```

参数：

+   `expr` – 一个列表达式，通常是一个`ColumnElement`实例，或者作为要强制转换为列表达式的 Python 标量表达式，用作`BETWEEN`表达式的左侧。

+   `lower_bound` – 作为`BETWEEN`表达式右侧下限的列或 Python 标量表达式。

+   `upper_bound` – 作为`BETWEEN`表达式右侧上限的列或 Python 标量表达式。

+   `symmetric` – 如果为 True，则渲染“ BETWEEN SYMMETRIC ”。请注意，并非所有数据库都支持此语法。

另请参阅

`ColumnElement.between()`

```py
function sqlalchemy.sql.expression.collate(expression: _ColumnExpressionArgument[str], collation: str) → BinaryExpression[str]
```

返回子句`expression COLLATE collation`。

例如：

```py
collate(mycolumn, 'utf8_bin')
```

产生：

```py
mycolumn COLLATE utf8_bin
```

如果排序表达式是大小写敏感的标识符，例如包含大写字符，则排序表达式也会被引用。

从版本 1.2 开始更改：如果 COLLATE 表达式是大小写敏感的，则会自动应用引用。

```py
function sqlalchemy.sql.expression.desc(column: _ColumnExpressionOrStrLabelArgument[_T]) → UnaryExpression[_T]
```

产生一个降序的`ORDER BY`子句元素。

例如：

```py
from sqlalchemy import desc

stmt = select(users_table).order_by(desc(users_table.c.name))
```

将生成 SQL 如下：

```py
SELECT id, name FROM user ORDER BY name DESC
```

`desc()`函数是所有 SQL 表达式上可用的`ColumnElement.desc()`方法的独立版本，例如：

```py
stmt = select(users_table).order_by(users_table.c.name.desc())
```

参数：

**column** – 一个`ColumnElement`（例如标量 SQL 表达式），用于应用`desc()`操作。

另请参阅

`asc()`

`nulls_first()`

`nulls_last()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.funcfilter(func: FunctionElement[_T], *criterion: _ColumnExpressionArgument[bool]) → FunctionFilter[_T]
```

对函数产生一个`FunctionFilter`对象。

对支持“FILTER”子句的数据库后端使用于聚合和窗口函数。

例如：

```py
from sqlalchemy import funcfilter
funcfilter(func.count(1), MyClass.name == 'some name')
```

会产生“COUNT(1) FILTER (WHERE myclass.name = ‘some name’)”。

此函数也可通过`func`构造本身通过`FunctionElement.filter()`方法使用。

另请参见

组内特殊修改器，过滤器 - 在 SQLAlchemy 统一教程中

`FunctionElement.filter()`

```py
function sqlalchemy.sql.expression.label(name: str, element: _ColumnExpressionArgument[_T], type_: _TypeEngineArgument[_T] | None = None) → Label[_T]
```

为给定的`ColumnElement`返回一个`Label`对象。

标签通过`AS` SQL 关键字通常在`SELECT`语句的列子句中更改元素的名称。

通过`ColumnElement.label()`方法更方便地使用`ColumnElement`。

参数：

+   `name` – 标签名称

+   `obj` – 一个`ColumnElement`。

```py
function sqlalchemy.sql.expression.nulls_first(column: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

为`ORDER BY`表达式生成`NULLS FIRST`修饰符。

`nulls_first()`旨在修改由`asc()`或`desc()`产生的表达式，并指示在排序过程中遇到 NULL 值时应如何处理：

```py
from sqlalchemy import desc, nulls_first

stmt = select(users_table).order_by(
    nulls_first(desc(users_table.c.name)))
```

上面的 SQL 表达式类似于：

```py
SELECT id, name FROM user ORDER BY name DESC NULLS FIRST
```

类似于`asc()`和`desc()`，`nulls_first()`通常是从列表达式本身使用`ColumnElement.nulls_first()`调用的，而不是作为其独立的函数版本，如下所示：

```py
stmt = select(users_table).order_by(
    users_table.c.name.desc().nulls_first())
```

从版本 1.4 开始更改：`nulls_first()`从以前的版本中的`nullsfirst()`重命名。以前的名称仍然可用于向后兼容性。

另请参见

`asc()`

`desc()`

`nulls_last()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.nullsfirst()
```

`nulls_first()` 函数的同义词。

2.0.5 版本更改：恢复了丢失的遗留符号 `nullsfirst()`。

```py
function sqlalchemy.sql.expression.nulls_last(column: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

生成 `ORDER BY` 表达式的 `NULLS LAST` 修饰符。

`nulls_last()` 旨在修改由 `asc()` 或 `desc()` 生成的表达式，并指示在排序过程中遇到 NULL 值时应如何处理：

```py
from sqlalchemy import desc, nulls_last

stmt = select(users_table).order_by(
    nulls_last(desc(users_table.c.name)))
```

上述 SQL 表达式将类似于：

```py
SELECT id, name FROM user ORDER BY name DESC NULLS LAST
```

像 `asc()` 和 `desc()` 一样，`nulls_last()` 通常是通过列表达式本身使用 `ColumnElement.nulls_last()` 而不是作为其独立函数版本调用的，如下所示：

```py
stmt = select(users_table).order_by(
    users_table.c.name.desc().nulls_last())
```

1.4 版本更改：`nulls_last()` 在之前的版本中被重命名为 `nullslast()`。 以前的名称仍然可用于向后兼容。

另请参阅

`asc()`

`desc()`

`nulls_first()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.nullslast()
```

`nulls_last()` 函数的旧词语同义词。

2.0.5 版本更改：恢复了丢失的遗留符号 `nullslast()`。

```py
function sqlalchemy.sql.expression.over(element: FunctionElement[_T], partition_by: _ByArgument | None = None, order_by: _ByArgument | None = None, range_: typing_Tuple[int | None, int | None] | None = None, rows: typing_Tuple[int | None, int | None] | None = None) → Over[_T]
```

对函数生成一个 `Over` 对象。

用于针对聚合或所谓的“窗口”函数，在支持窗口函数的数据库后端使用。

`over()`通常使用`FunctionElement.over()`方法调用，例如：

```py
func.row_number().over(order_by=mytable.c.some_column)
```

会产生：

```py
ROW_NUMBER() OVER(ORDER BY some_column)
```

还可以使用`over.range_`和`over.rows`参数进行范围设置。这两个互斥参数各自接受一个二元组，其中包含整数和 None 的组合：

```py
func.row_number().over(
    order_by=my_table.c.some_column, range_=(None, 0))
```

上述将产生：

```py
ROW_NUMBER() OVER(ORDER BY some_column
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

`None`的值表示“无限制”，零的值表示“当前行”，负/正整数表示“前面”和“后面”：

+   RANGE BETWEEN 5 PRECEDING AND 10 FOLLOWING:

    ```py
    func.row_number().over(order_by='x', range_=(-5, 10))
    ```

+   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW:

    ```py
    func.row_number().over(order_by='x', rows=(None, 0))
    ```

+   RANGE BETWEEN 2 PRECEDING AND UNBOUNDED FOLLOWING:

    ```py
    func.row_number().over(order_by='x', range_=(-2, None))
    ```

+   RANGE BETWEEN 1 FOLLOWING AND 3 FOLLOWING:

    ```py
    func.row_number().over(order_by='x', range_=(1, 3))
    ```

参数：

+   `element` – 一个`FunctionElement`、`WithinGroup`或其他兼容的构造。

+   `partition_by` – 作为 OVER 构造的 PARTITION BY 子句使用的列元素或字符串，或这些列元素或字符串的列表。

+   `order_by` – 作为 OVER 构造的 ORDER BY 子句使用的列元素或字符串，或这些列元素或字符串的列表。

+   `range_` – 窗口的可选范围子句。这是一个元组值，可以包含整数值或`None`，并将呈现为一个 RANGE BETWEEN PRECEDING / FOLLOWING 子句。

+   `rows` – 窗口的可选行子句。这是一个元组值，可以包含整数值或 None，并将呈现为一个 ROWS BETWEEN PRECEDING / FOLLOWING 子句。

此函数也可以通过`func`构造本身使用`FunctionElement.over()`方法。

另见

使用窗口函数 - 在 SQLAlchemy 统一教程中

`func`

`within_group()`

```py
function sqlalchemy.sql.expression.within_group(element: FunctionElement[_T], *order_by: _ColumnExpressionArgument[Any]) → WithinGroup[_T]
```

产生一个`WithinGroup`对象与一个函数。

用于所谓的“有序集聚合”和“假设集聚合”函数，包括 `percentile_cont`、`rank`、`dense_rank` 等。

`within_group()` 通常使用 `FunctionElement.within_group()` 方法来调用，例如：

```py
from sqlalchemy import within_group
stmt = select(
    department.c.id,
    func.percentile_cont(0.5).within_group(
        department.c.salary.desc()
    )
)
```

上述语句将产生类似于 `SELECT department.id, percentile_cont(0.5) WITHIN GROUP (ORDER BY department.salary DESC)` 的 SQL。

参数:

+   `element` – 一个 `FunctionElement` 结构，通常由 `func` 生成。

+   `*order_by` – 一个或多个列元素，将用作 WITHIN GROUP 构造的 ORDER BY 子句。

另请参阅

特殊修饰符 WITHIN GROUP, FILTER - 在 SQLAlchemy 统一教程 中

`func`

`over()`

## 列元素类文档

这里的类是使用 列元素基础构造函数 和 列元素修饰符构造函数 列出的构造函数生成的。

| 对象名称 | 描述 |
| --- | --- |
| BinaryExpression | 表示 `LEFT <operator> RIGHT` 的表达式。 |
| BindParameter | 表示“绑定表达式”。 |
| Case | 表示 `CASE` 表达式。 |
| Cast | 表示 `CAST` 表达式。 |
| ClauseList | 描述由操作符分隔的子句列表。 |
| ColumnClause | 表示来自任何文本字符串的列表达式。 |
| ColumnCollection | `FromClause` 对象的 `ColumnElement` 实例的集合。 |
| ColumnElement | 代表适用于语句的“列”子句、WHERE 子句等的面向列的 SQL 表达式。 |
| ColumnExpressionArgument | 通用的“列表达式”参数。 |
| ColumnOperators | 为`ColumnElement`表达式定义布尔、比较和其他运算符。 |
| Extract | 代表一个 SQL EXTRACT 子句，`extract(field FROM expr)`。 |
| False_ | 代表 SQL 语句中的`false`关键字或等效关键字。 |
| FunctionFilter | 代表一个函数 FILTER 子句。 |
| Label | 代表列标签（AS）。 |
| Null | 代表 SQL 语句中的 NULL 关键字。 |
| Operators | 比较和逻辑运算符的基类。 |
| Over | 代表一个 OVER 子句。 |
| SQLColumnExpression | 可用于指示任何 SQL 列元素或充当其替代品的对象的类型。 |
| TextClause | 代表一个字面的 SQL 文本片段。 |
| True_ | 代表 SQL 语句中的`true`关键字或等效关键字。 |
| TryCast | 代表一个 TRY_CAST 表达式。 |
| Tuple | 代表一个 SQL 元组。 |
| TypeCoerce | 代表一个 Python 端的类型强制转换包装器。 |
| UnaryExpression | 定义一个“一元”表达式。 |
| WithinGroup | 代表一个 WITHIN GROUP（ORDER BY）子句。 |
| WrapsColumnExpression | 定义一个具有特殊标签行为的`ColumnElement`包装器的混合类，用于已经具有名称的表达式。 |

```py
class sqlalchemy.sql.expression.BinaryExpression
```

代表一个`LEFT <operator> RIGHT`的表达式。

当两个列表达式在 Python 二元表达式中使用时，会自动生成一个`BinaryExpression`：

```py
>>> from sqlalchemy.sql import column
>>> column('a') + column('b')
<sqlalchemy.sql.expression.BinaryExpression object at 0x101029dd0>
>>> print(column('a') + column('b'))
a  +  b 
```

**类签名**

类`sqlalchemy.sql.expression.BinaryExpression`（`sqlalchemy.sql.expression.OperatorExpression`）

```py
class sqlalchemy.sql.expression.BindParameter
```

代表一个“绑定表达式”。

`BindParameter`是通过使用`bindparam()`函数显式调用的，如下所示：

```py
from sqlalchemy import bindparam

stmt = select(users_table).where(
    users_table.c.name == bindparam("username")
)
```

详细讨论了`BindParameter`的使用方法在`bindparam()`中。

另请参阅

`bindparam()`

**成员**

effective_value, inherit_cache, render_literal_execute()

**类签名**

类`sqlalchemy.sql.expression.BindParameter` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.KeyedColumnElement`)

```py
attribute effective_value
```

返回此绑定参数的值，考虑是否设置了`callable`参数。

如果存在`callable`值，则将对其进行评估并返回，否则返回`value`。

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存; 这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与对象对应的 SQL 不基于仅属于此类而不是其超类的属性更改，则可以在特定类上将此标志设置为`True`。

另请参阅

为自定义构造启用缓存支持 - 有关为第三方或用户定义的 SQL 构造设置`HasCacheKey.inherit_cache`属性的一般指南。

```py
method render_literal_execute() → BindParameter[_T]
```

生成此绑定参数的副本，将启用`BindParameter.literal_execute`标志。

`BindParameter.literal_execute` 标志将使参数在编译后的 SQL 字符串中以`[POSTCOMPILE]`形式呈现，这是一种特殊形式，会在 SQL 执行时转换为参数的字面值。其目的是支持缓存可以嵌入每个语句字面值的 SQL 语句字符串，例如 LIMIT 和 OFFSET 参数，在传递给 DBAPI 的最终 SQL 字符串中。特别是方言可能希望在自定义编译方案中使用此方法。

在 1.4.5 版中新增。

另请参阅

第三方方言的缓存

```py
class sqlalchemy.sql.expression.Case
```

表示一个 `CASE` 表达式。

`Case` 是使用 `case()` 工厂函数生成的，如下所示：

```py
from sqlalchemy import case

stmt = select(users_table).                    where(
                case(
                    (users_table.c.name == 'wendy', 'W'),
                    (users_table.c.name == 'jack', 'J'),
                    else_='E'
                )
            )
```

有关 `Case` 的详细用法请参阅 `case()`。

另请参阅

`case()`

**类签名**

类 `sqlalchemy.sql.expression.Case` (`sqlalchemy.sql.expression.ColumnElement`)

```py
class sqlalchemy.sql.expression.Cast
```

表示一个 `CAST` 表达式。

`Cast` 是使用 `cast()` 工厂函数生成的，如下所示：

```py
from sqlalchemy import cast, Numeric

stmt = select(cast(product_table.c.unit_price, Numeric(10, 4)))
```

有关 `Cast` 的详细用法请参阅 `cast()`。

另请参阅

数据转换和类型强制转换

`cast()`

`try_cast()`

`type_coerce()` - 一种仅在 Python 端强制类型的 CAST 替代方法，通常足以生成正确的 SQL 和数据强制转换。

**类签名**

类 `sqlalchemy.sql.expression.Cast` (`sqlalchemy.sql.expression.WrapsColumnExpression`)

```py
class sqlalchemy.sql.expression.ClauseList
```

描述由运算符分隔的子句列表。

默认情况下，以逗号分隔，例如列清单。

**成员**

self_group()

**类签名**

类`sqlalchemy.sql.expression.ClauseList`（`sqlalchemy.sql.roles.InElementRole`，`sqlalchemy.sql.roles.OrderByRole`，`sqlalchemy.sql.roles.ColumnsClauseRole`，`sqlalchemy.sql.roles.DMLColumnRole`，`sqlalchemy.sql.expression.DQLDMLClauseElement`）。

```py
method self_group(against=None)
```

对此`ClauseElement`应用“分组”。

此方法被子类重写以返回一个“分组”构造，即括号。特别是当“二元”表达式被放置到较大表达式中时，它们会提供自己周围的分组，以及当`select()`构造被放置到另一个`select()`的 FROM 子句中时。（注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套 SELECT 语句必须命名）。

随着表达式的组合，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 所以可能不需要括号，例如，在像`x OR (y AND z)`这样的表达式中 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
class sqlalchemy.sql.expression.ColumnClause
```

表示来自任何文本字符串的列表达式。

`ColumnClause`，类似于`Column`类的轻量级模拟，通常使用`column()`函数调用，如下所示：

```py
from sqlalchemy import column

id, name = column("id"), column("name")
stmt = select(id, name).select_from("user")
```

上述语句将生成类似于的 SQL：

```py
SELECT id, name FROM user
```

`ColumnClause`是模式特定的`Column`对象的直接超类。虽然`Column`类具有与`ColumnClause`相同的所有功能，但`ColumnClause`类可单独使用，用于那些行为要求仅限于简单的 SQL 表达式生成的情况。该对象没有与模式级元数据或执行时行为相关的关联，因此在这个意义上是`Column`的“轻量级”版本。

`ColumnClause`用法的完整详细信息在`column()`。

另请参阅

`column()`

`Column`

**成员**

get_children()

**类签名**

类`sqlalchemy.sql.expression.ColumnClause` (`sqlalchemy.sql.roles.DDLReferredColumnRole`, `sqlalchemy.sql.roles.LabeledColumnExprRole`, `sqlalchemy.sql.roles.StrAsPlainColumnRole`, `sqlalchemy.sql.expression.Immutable`, `sqlalchemy.sql.expression.NamedColumn`)

```py
method get_children(*, column_tables=False, **kw)
```

返回此`HasTraverseInternals`的直接子`HasTraverseInternals`元素。

这用于访问遍历。

**kw 可能包含更改返回集合的标志，例如返回子项的子集以减少更大的遍历，或者从不同上下文返回子项（例如模式级集合而不是子句级）。

```py
class sqlalchemy.sql.expression.ColumnCollection
```

`ColumnElement`实例的集合，通常用于`FromClause`对象。

`ColumnCollection`对象通常作为`Table.c`或`Table.columns`集合在`Table`对象上最常用，介绍在访问表和列。

`ColumnCollection` 具有类似映射和序列的行为。 `ColumnCollection` 通常存储 `Column` 对象，然后可以通过映射样式访问以及属性访问样式访问。

要使用普通属性样式访问方式访问 `Column` 对象，请像访问任何其他对象属性一样指定名称，例如下面访问了一个名为 `employee_name` 的列：

```py
>>> employee_table.c.employee_name
```

要访问具有特殊字符或空格名称的列，使用索引样式访问，例如下面演示了如何访问名为 `employee ' payment` 的列：

```py
>>> employee_table.c["employee ' payment"]
```

由于 `ColumnCollection` 对象提供了 Python 字典接口，因此常见的字典方法名称如 `ColumnCollection.keys()`、`ColumnCollection.values()` 和 `ColumnCollection.items()` 都可用，这意味着键入这些名称的数据库列也需要使用索引访问：

```py
>>> employee_table.c["values"]
```

通常情况下，`Column`存在的名称是 `Column.key` 参数的值。在某些上下文中，例如使用 `Select.set_label_style()` 方法设置标签样式的 `Select` 对象，某个键的列可能被表示为特定标签名称，如`tablename_columnname`：

```py
>>> from sqlalchemy import select, column, table
>>> from sqlalchemy import LABEL_STYLE_TABLENAME_PLUS_COL
>>> t = table("t", column("c"))
>>> stmt = select(t).set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)
>>> subq = stmt.subquery()
>>> subq.c.t_c
<sqlalchemy.sql.elements.ColumnClause at 0x7f59dcf04fa0; t_c>
```

`ColumnCollection` 还按顺序索引列，并允许通过它们的整数位置访问它们：

```py
>>> cc[0]
Column('x', Integer(), table=None)
>>> cc[1]
Column('y', Integer(), table=None)
```

从 1.4 版开始：`ColumnCollection` 允许基于整数的索引访问集合。

遍历集合按顺序产生列表达式：

```py
>>> list(cc)
[Column('x', Integer(), table=None),
 Column('y', Integer(), table=None)]
```

基本的`ColumnCollection`对象可以存储重复项，这可能意味着具有相同键的两个列，此时通过键访问返回的列是**任意的**：

```py
>>> x1, x2 = Column('x', Integer), Column('x', Integer)
>>> cc = ColumnCollection(columns=[(x1.name, x1), (x2.name, x2)])
>>> list(cc)
[Column('x', Integer(), table=None),
 Column('x', Integer(), table=None)]
>>> cc['x'] is x1
False
>>> cc['x'] is x2
True
```

或者它也可以多次表示相同的列。这些情况是支持的，因为`ColumnCollection`用于表示 SELECT 语句中的列，该语句可能包含重复项。

存在一个特殊的子类`DedupeColumnCollection`，它维护的是 SQLAlchemy 的旧行为，即不允许重复项；这个集合用于模式级对象，如`Table`和`PrimaryKeyConstraint`，在这些情况下去重是有帮助的。`DedupeColumnCollection`类还具有额外的变异方法，因为模式构造具有更多需要移除和替换列的用例。

自版本 1.4 更改：`ColumnCollection`现在也存储重复的列键以及同一列在多个位置的情况。在这些情况下，添加了`DedupeColumnCollection`类以保持以前的行为，其中需要去重以及额外的替换/移除操作。

**成员**

add(), as_readonly(), clear(), compare(), contains_column(), corresponding_column(), get(), items(), keys(), update(), values()

**类签名**

类`sqlalchemy.sql.expression.ColumnCollection` (`typing.Generic`)

```py
method add(column: ColumnElement[Any], key: _COLKEY | None = None) → None
```

向这个`ColumnCollection`添加一列。

注意

此方法**通常不会被用户界面代码使用**，因为`ColumnCollection`通常是现有对象的一部分，例如`Table`。要向现有的`Table`对象添加`Column`，请使用`Table.append_column()`方法。

```py
method as_readonly() → ReadOnlyColumnCollection[_COLKEY, _COL_co]
```

返回此`ColumnCollection`的“只读”形式。

```py
method clear() → NoReturn
```

对于`ColumnCollection`，未实现字典清除()。

```py
method compare(other: ColumnCollection[Any, Any]) → bool
```

根据键的名称将此`ColumnCollection`与另一个进行比较

```py
method contains_column(col: ColumnElement[Any]) → bool
```

检查此集合中是否存在列对象

```py
method corresponding_column(column: _COL, require_embedded: bool = False) → _COL | _COL_co | None
```

给定一个`ColumnElement`，从此`ColumnCollection`中返回导出的`ColumnElement`对象，该对象通过共同祖先列与原始`ColumnElement`相对应。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 仅在给定的`ColumnElement`实际存在于此`Selectable`的子元素中时返回相应的列。通常，如果列仅与此`Selectable`的导出列之一共享共同的祖先，则列将匹配。

另请参阅

`Selectable.corresponding_column()` - 调用此方法以针对`Selectable.exported_columns`返回的集合。

在 1.4 版本中更改：`corresponding_column`的实现已移至`ColumnCollection`本身。

```py
method get(key: str, default: _COL_co | None = None) → _COL_co | None
```

基于此`ColumnCollection`中的字符串键名获取一个`ColumnClause`或`Column`对象。

```py
method items() → List[Tuple[_COLKEY, _COL_co]]
```

为此集合中的所有列返回一个(key, column)元组序列，每个元组由一个字符串键名和`ColumnClause`或`Column`对象组成。

```py
method keys() → List[_COLKEY]
```

返回此集合中所有列的字符串键名序列。

```py
method update(iter_: Any) → NoReturn
```

对于`ColumnCollection`，未实现字典的 update()方法。

```py
method values() → List[_COL_co]
```

为此集合中的所有列返回一个`ColumnClause`或`Column`对象序列。

```py
class sqlalchemy.sql.expression.ColumnElement
```

表示适用于语句的“columns”子句、WHERE 子句等的面向列的 SQL 表达式。

虽然最熟悉的`ColumnElement`类型是`Column`对象，但`ColumnElement`作为可能出现在 SQL 表达式中的任何单元的基础，包括表达式本身、SQL 函数、绑定参数、文字表达式、关键字如`NULL`等。`ColumnElement`是所有这些元素的最终基类。

一系列广泛的 SQLAlchemy 核心函数在 SQL 表达式级别工作，并旨在接受`ColumnElement`的实例作为参数。这些函数通常会记录它们接受一个“SQL 表达式”作为参数。在 SQLAlchemy 中，这通常指的是一个已经是`ColumnElement`对象形式的输入，或者可以**强制转换**为其中一个的值。大多数 SQLAlchemy 核心函数关于 SQL 表达式的强制转换规则如下：

> +   一个字面的 Python 值，比如字符串、整数或浮点值、布尔值、日期时间、`Decimal`对象，或者几乎任何其他 Python 对象，都将被强制转换为“字面绑定值”。这通常意味着将生成一个包含给定值嵌入到结构中的`bindparam()`；最终产生的`BindParameter`对象是`ColumnElement`的一个实例。Python 值最终将在执行时作为参数化参数传递给`execute()`或`executemany()`方法，之后会应用 SQLAlchemy 类型特定的转换器（例如任何相关的`TypeEngine`对象提供的转换器）。
> +   
> +   任何特殊对象值，通常是 ORM 级别的构造，其中包含一个名为`__clause_element__()`的访问器。当将一个否则未知类型的对象传递给一个希望将参数强制转换为`ColumnElement`和有时是`SelectBase`表达式的函数时，核心表达式系统会查找这个方法。在 ORM 中，它用于将 ORM 特定的对象（如映射类和映射属性）转换为核心表达式对象。
> +   
> +   Python 的`None`值通常被解释为`NULL`，在 SQLAlchemy Core 中会产生一个`null()`的实例。

`ColumnElement` 提供了使用 Python 表达式生成新的 `ColumnElement` 对象的能力。这意味着 Python 操作符如`==`、`!=`和`<`被重载以模仿 SQL 操作，并允许实例化更多由其他更基本的`ColumnElement` 对象组成的`ColumnElement` 实例。例如，两个`ColumnClause` 对象可以使用加法运算符`+`相加，生成一个`BinaryExpression`。`ColumnClause` 和`BinaryExpression` 都是 `ColumnElement` 的子类。

```py
>>> from sqlalchemy.sql import column
>>> column('a') + column('b')
<sqlalchemy.sql.expression.BinaryExpression object at 0x101029dd0>
>>> print(column('a') + column('b'))
a  +  b 
```

另请参阅

`Column`

`column()`

**成员**

__eq__(), __le__(), __lt__(), __ne__(), all_(), allows_lambda, anon_key_label, anon_label, any_(), asc(), base_columns, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), cast(), collate(), comparator, compare(), compile(), concat(), contains(), desc(), description, distinct(), endswith(), entity_namespace, expression, foreign_keys, get_children(), icontains(), iendswith(), ilike(), in_(), inherit_cache, is_(), is_clause_element, is_distinct_from(), is_dml, is_not(), is_not_distinct_from(), is_selectable, isnot(), isnot_distinct_from(), istartswith(), key, label(), like(), match(), negation_clause, not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), params(), primary_key, proxy_set, regexp_match(), regexp_replace(), reverse_operate(), self_group(), shares_lineage(), startswith(), stringify_dialect, supports_execution, timetuple, type, unique_params(), uses_inspection

**类签名**

类`sqlalchemy.sql.expression.ColumnElement`（`sqlalchemy.sql.roles.ColumnArgumentOrKeyRole`、`sqlalchemy.sql.roles.StatementOptionRole`、`sqlalchemy.sql.roles.WhereHavingRole`、`sqlalchemy.sql.roles.BinaryElementRole`、`sqlalchemy.sql.roles.OrderByRole`、`sqlalchemy.sql.roles.ColumnsClauseRole`、`sqlalchemy.sql.roles.LimitOffsetRole`、`sqlalchemy.sql.roles.DMLColumnRole`、`sqlalchemy.sql.roles.DDLConstraintColumnRole`、`sqlalchemy.sql.roles.DDLExpressionRole`、`sqlalchemy.sql.expression.SQLColumnExpression`、`sqlalchemy.sql.expression.DQLDMLClauseElement`）

```py
method __eq__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__eq__` *方法的* `ColumnOperators`

实现`==`运算符。

在列上下文中，生成子句`a = b`。如果目标是`None`，则生成`a IS NULL`。

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法的* `ColumnOperators`

实现`<=`运算符。

在列上下文中，生成子句`a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法的* `ColumnOperators`

实现`<`运算符。

在列上下文中，生成子句`a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法的* `ColumnOperators`

实现`!=`运算符。

在列上下文中，生成子句`a != b`。如果目标是`None`，则生成`a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators.all_()` *方法的* `ColumnOperators`

对父对象生成一个`all_()`子句。

请参阅`all_()`的文档以获取示例。

注意

一定要注意不要将新的`ColumnOperators.all_()`方法与**旧版**方法混淆，旧版方法是`Comparator.all()`方法，该方法专用于`ARRAY`，其使用不同的调用风格。

```py
attribute allows_lambda = True
```

```py
attribute anon_key_label
```

自版本 1.4 起弃用：`ColumnElement.anon_key_label` 属性现在是私有的，公共访问器已弃用。

```py
attribute anon_label
```

自版本 1.4 起弃用：`ColumnElement.anon_label` 属性现在是私有的，公共访问器已弃用。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators`

产生一个针对父对象的`any_()`子句。

查看 `any_()` 的文档以获取示例。

注意

请确保不要混淆较新的 `ColumnOperators.any_()` 方法与这个方法的**传统**版本，即特定于 `ARRAY` 的 `Comparator.any()` 方法，它使用不同的调用风格。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

产生一个针对父对象的`asc()`子句。

```py
attribute base_columns
```

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

产生一个针对父对象的`between()`子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

产生一个按位与操作，通常通过`&`运算符实现。

新版本 2.0.2 中新增。

另请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

执行按位 LSHIFT 操作，通常通过 `<<` 运算符。

版本 2.0.2 中的新内容。

另请参阅

按位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

执行按位 NOT 操作，通常通过 `~` 运算符。

版本 2.0.2 中的新内容。

另请参阅

按位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

执行按位 OR 操作，通常通过 `|` 运算符。

版本 2.0.2 中的新内容。

另请参阅

按位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

执行按位 RSHIFT 操作，通常通过 `>>` 运算符。

版本 2.0.2 中的新内容。

另请参阅

按位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

执行按位异或操作，通常通过 `^` 运算符，或者对于 PostgreSQL 是 `#`。

版本 2.0.2 中的新内容。

另请参阅

按位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

这个方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的快捷方式。 使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在 [**PEP 484**](https://peps.python.org/pep-0484/) 目的上。

另请参阅

`Operators.op()`

```py
method cast(type_: _TypeEngineArgument[_OPT]) → Cast[_OPT]
```

产生一个类型转换，即 `CAST(<expression> AS <type>)`。

这是 `cast()` 函数的快捷方式。

另请参阅

数据转换和类型强制转换

`cast()`

`type_coerce()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

产生一个针对父对象的 `collate()` 子句，给定排序字符串。

另请参阅

`collate()`

```py
attribute comparator
```

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

将这个 `ClauseElement` 与给定的 `ClauseElement` 进行比较。

子类应该覆盖默认行为，即直接的身份比较。

**kw 是子类 `compare()` 方法消耗的参数，可以用来修改比较的标准（参见 `ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译这个 SQL 表达式。

返回值是一个 `Compiled` 对象。对返回值调用 `str()` 或 `unicode()` 将产生结果的字符串表示。`Compiled` 对象还可以使用 `params` 访问器返回一个绑定参数名称和值的字典。

参数：

+   `bind` – 一个 `Connection` 或 `Engine`，可以提供一个 `Dialect` 以生成一个 `Compiled` 对象。如果 `bind` 和 `dialect` 参数都被省略，则使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，一个列名的列表，应该在编译后的语句的 VALUES 子句中出现。如果为 `None`，则从目标表对象中渲染所有列。

+   `dialect` – 一个 `Dialect` 实例，可以生成一个 `Compiled` 对象。此参数优先于 `bind` 参数。

+   `compile_kwargs` –

    可选的字典，包含将传递给所有“visit”方法中的编译器的附加参数。这允许传递任何自定义标志到自定义编译构造中，例如。它还用于通过以下方式传递 `literal_binds` 标志：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参阅

如何将 SQL 表达式渲染为字符串，可能会内联绑定参数？

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现 'concat' 运算符。

在列上下文中，生成子句 `a || b`，或在 MySQL 上使用 `concat()` 运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法的* `ColumnOperators`

实现 'contains' 运算符。

产生一个 LIKE 表达式，用于测试字符串值中间的匹配：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于操作符使用 `LIKE`，因此 `<other>` 表达式内部存在的通配符字符 `"%"` 和 `"_"` 也会像通配符一样行为。对于字面字符串值，可以将 `ColumnOperators.contains.autoescape` 标志设置为 `True`，以将转义应用于字符串值中这些字符的出现次数，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.contains.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 待比较的表达式。通常是纯字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不会转义，除非设置了 `ColumnOperators.contains.autoescape` 标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的 `"%"`、`"_"` 和转义字符本身的出现次数，假定比较值为字面字符串而不是 SQL 表达式。

    诸如以下表达式：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    值为 `:param` 的情况下为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将呈现为带有 `ESCAPE` 关键字以将该字符设定为转义字符。然后，可以将该字符放置在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    诸如以下表达式：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.contains.autoescape` 结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

对父对象生成一个`desc()`子句。

```py
attribute description
```

*继承自* `ClauseElement` *的* `ClauseElement.description` *属性*

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators.distinct()` *方法的* `ColumnOperators`

对父对象生成一个`distinct()`子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.endswith()` *方法的* `ColumnOperators`

实现‘endswith’操作符。

生成一个 LIKE 表达式，用于测试字符串值的结尾是否匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于文字字符串值，可能会将`ColumnOperators.endswith.autoescape`标志设置为`True`，以将这些字符出现在字符串值中的转义，使它们匹配为自己而不是通配符字符。或者，`ColumnOperators.endswith.escape`参数将建立一个给定字符作为转义字符，这在目标表达式不是文字字符串时可能有用。

参数：

+   `other` – 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.endswith.autoescape`标志设置为 True，否则不会转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是文字字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    使用值`：param`作为`"foo/%bar"`。

+   `escape` –

    给定的字符，当使用时将呈现为`ESCAPE`关键字，以将该字符作为转义字符。然后，此字符可以放置在`%`和`_`的前面，以允许它们像自己一样工作，而不是通配符字符。

    诸如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute entity_namespace
```

*继承自* `ClauseElement` *的* `ClauseElement.entity_namespace` *属性*

```py
attribute expression
```

返回列表达式。

检查界面的一部分；返回自身。

```py
attribute foreign_keys: AbstractSet[ForeignKey] = frozenset({})
```

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals` *的* `HasTraverseInternals.get_children()` *方法*

返回此`HasTraverseInternals`的即时子元素。

这用于访问遍历。

**kw 可能包含更改返回的集合的标志，例如返回子集以减少较大的遍历，或者返回不同上下文的子项（例如模式级别的集合而不是子句级别）。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.icontains()` *的方法* `ColumnOperators`

实现`icontains`运算符，例如`ColumnOperators.contains()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值中间的不区分大小写匹配进行测试：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于运算符使用了`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样行事。对于字面字符串值，可以将`ColumnOperators.icontains.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.icontains.escape`参数将建立给定的字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个普通字符串值，但也可以是任意 SQL 表达式。除非将`ColumnOperators.icontains.autoescape`标志设置为 True，否则不会转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    将值`:param`设为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将该字符建立为转义字符。然后可以将该字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数在传递给数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.iendswith()` *方法的* `ColumnOperators`

实现`iendswith`操作符，例如`ColumnOperators.endswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的末尾进行不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该操作符使用`LIKE`，因此在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.iendswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符 `%` 和 `_` 不会被转义，除非将 `ColumnOperators.iendswith.autoescape` 标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内的所有 `"%"`、`"_"` 和转义字符本身的出现，假设比较值是一个文字字符串而不是一个 SQL 表达式。

    诸如：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其中，`:param` 的值为 `"foo/%bar"`。

+   `escape` –

    给定的字符会与 `ESCAPE` 关键字一起渲染，以将该字符确定为转义字符。然后可以将此字符放置在 `%` 和 `_` 的前面，以使它们能够充当它们自己而不是通配符字符。

    诸如：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.iendswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    上述情况下，给定的文字参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参见

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *的方法* `ColumnOperators`

实现 `ilike` 运算符，例如大小写不敏感的 LIKE。

在列上下文中，生成形式为以下之一的表达式：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，渲染 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参见

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *的方法* `ColumnOperators`

实现 `in` 运算符。

在列上下文中，生成子句 `column IN <other>`。

给定参数 `other` 可能是：

+   字面值的列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表被转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的 `tuple_()`，则可以提供元组的列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在此调用形式中，该表达式呈现一个“空集”表达式。这些表达式针对各个后端进行了定制，并且通常试图将空 SELECT 语句作为子查询。例如在 SQLite 上，该表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    在 1.4 版本中更改：所有情况下，空 IN 表达式现在都使用执行时生成的 SELECT 子查询。

+   如果包括 `bindparam.expanding` 标志，则可以使用绑定参数，例如 `bindparam()`：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在此调用形式中，该表达式呈现一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，以转换为前面示例中的变量数量的绑定参数形式。如果执行语句为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    对于每个值，数据库将传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    1.2 版本中的新功能：添加了“expanding”绑定参数

    如果传递了空列表，则会呈现一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    1.3 版本中的新功能：现在“expanding”绑定参数支持空列表

+   一个 `select()` 构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在此调用形式中，`ColumnOperators.in_()`如下呈现：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**其他** – 一组文字，`select()` 构造，或者包括 `bindparam()` 构造的 `bindparam()` 构造，并将 `bindparam.expanding` 标志设置为 True。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey` 的 `HasCacheKey.inherit_cache` *属性*

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为`None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等效于将值设置为`False`，只是还会发出警告。

如果对象对于本类是本地的，而不是其超类，则可以在特定类上将此标志设置为`True`，如果 SQL 不基于本类的属性而变化。

另请参见

为自定义结构启用缓存支持 - 设置第三方或用户定义的 SQL 结构的 `HasCacheKey.inherit_cache` 属性的通用指南。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现 `IS` 运算符。

通常情况下，与 `None` 的值进行比较时会自动生成 `IS`，这会解析为 `NULL`。但是，如果在某些平台上与布尔值进行比较时，可能希望显式使用 `IS`。

另请参见

`ColumnOperators.is_not()`

```py
attribute is_clause_element = True
```

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现 `IS DISTINCT FROM` 运算符。

在大多数平台上呈现为 “a IS DISTINCT FROM b”；在某些平台上（例如 SQLite）可能呈现为 “a IS NOT b”。

```py
attribute is_dml = False
```

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现 `IS NOT` 运算符。

通常情况下，与 `None` 的值进行比较时会自动生成 `IS NOT`，这会解析为 `NULL`。但是，如果在某些平台上与布尔值进行比较时，可能希望显式使用 `IS NOT`。

在版本 1.4 中更改：`is_not()` 运算符从先前的发布中重命名为 `isnot()`。以前的名称仍然可用以保持向后兼容性。

另请参见

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现 `IS NOT DISTINCT FROM` 运算符。

在大多数平台上呈现为 “a IS NOT DISTINCT FROM b”；在某些平台上（例如 SQLite）可能呈现为 “a IS b”。

在版本 1.4 中更改：`is_not_distinct_from()` 运算符从先前的发布中重命名为 `isnot_distinct_from()`。以前的名称仍然可用以保持向后兼容性。

```py
attribute is_selectable = False
```

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS NOT`，其解析为`NULL`。然而，如果在某些平台上与布尔值进行比较时，显式使用`IS NOT`可能是可取的。

在 1.4 版本中更改：`is_not()`运算符从以前的版本中的`isnot()`更名。以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *方法的* `ColumnOperators`

实现`IS NOT DISTINCT FROM`运算符。

在大多数平台上渲染为“a IS NOT DISTINCT FROM b”；在某些平台（如 SQLite）上可能渲染为“a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()`运算符从以前的版本中的`isnot_distinct_from()`更名。以前的名称仍然可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *方法的* `ColumnOperators`

实现`istartswith`运算符，例如`ColumnOperators.startswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的开头进行不区分大小写匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该运算符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样行为。对于字面字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，以使它们与自身匹配而不是作为通配符字符。或者，`ColumnOperators.istartswith.escape`参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个普通的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符`%`和`_`不会被转义，除非`ColumnOperators.istartswith.autoescape`标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值为字面字符串而不是 SQL 表达式。

    诸如下面的表达式：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    值为`:param`的`"foo/%bar"`。

+   `escape` –

    一个字符，给定时将使用`ESCAPE`关键字来建立该字符作为转义字符。然后，可以将此字符放置在`%`和`_`之前，以允许它们充当自身而不是通配符字符。

    诸如下面的表达式：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

```py
attribute key: str | None = None
```

在某些情况下指的是 Python 命名空间中的对象的“键”。

这通常是指在可选择的`.c`集合中出现的列的“键”，例如`sometable.c["somekey"]`将返回具有“somekey”`.key`的`Column`。

```py
method label(name: str | None) → Label[_T]
```

生成列标签，即`<columnname> AS <name>`。

这是`label()`函数的快捷方式。

如果‘name’为 `None`，将生成一个匿名标签名称。

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *方法的* `ColumnOperators`

实现`like`操作符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *方法的* `ColumnOperators`

实现了特定于数据库的‘match’运算符。

`ColumnOperators.match()` 尝试解析为后端提供的类似 MATCH 的函数或运算符。例如：

+   PostgreSQL - 渲染 `x @@ plainto_tsquery(y)`

    > 在版本 2.0 中更改：现在对于 PostgreSQL，使用`plainto_tsquery()`代替`to_tsquery()`；为了与其他形式兼容，请参见全文搜索。

+   MySQL - 渲染 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 渲染 `CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将运算符发出为“MATCH”。例如，这与 SQLite 兼容。

```py
attribute negation_clause: ColumnElement[bool]
```

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这等同于使用否定与`ColumnOperators.ilike()`，即 `~x.ilike(y)`。

在版本 1.4 中更改：`not_ilike()`运算符从先前版本的`notilike()`重命名。以前的名称仍可用于向后兼容。

另请参��

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这等同于使用`ColumnOperators.in_()`进行否定，即 `~x.in_(y)`。

如果`other`是一个空序列，编译器会生成一个“空 not in”表达式。默认情况下，这会产生表达式“1 = 1”，以在所有情况下产生 true。可以使用`create_engine.empty_in_strategy`来更改这种行为。

在版本 1.4 中更改：`not_in()`运算符从先前版本的`notin_()`重命名。以前的名称仍可用于向后兼容。

从 1.2 版本开始更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认情况下为一个空的 IN 序列生成“静态”表达式。

请参见

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *的方法* `ColumnOperators`

实现`NOT LIKE`运算符。

这相当于使用带有`ColumnOperators.like()`的否定，即`~x.like(y)`。

从 1.4 版本开始更改：`not_like()`运算符从先前版本的`notlike()`重命名。以确保向后兼容性，先前的名称仍然可用。

请参见

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *的方法* `ColumnOperators`

实现`NOT ILIKE`运算符。

这相当于使用带有`ColumnOperators.ilike()`的否定，即`~x.ilike(y)`。

从 1.4 版本开始更改：`not_ilike()`运算符从先前版本的`notilike()`重命名。以确保向后兼容性，先前的名称仍然可用。

请参见

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *的方法* `ColumnOperators`

实现`NOT IN`运算符。

这相当于使用带有`ColumnOperators.in_()`的否定，即`~x.in_(y)`。

在`other`为空序列的情况下，编译器生成一个“空 not in”表达式。这默认为表达式“1 = 1”，以在所有情况下产生 true。`create_engine.empty_in_strategy` 可用于更改此行为。

在版本 1.4 中发生了变化：`not_in()` 运算符从之前的版本中的 `notin_()` 重新命名。以前的名称仍然可用于向后兼容。

在版本 1.2 中发生了变化：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认情况下会针对空的 IN 序列生成一个“静态”表达式。

另请参见

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*从* `ColumnOperators.notlike()` *方法继承* *自* `ColumnOperators`。

实现 `NOT LIKE` 运算符。

这等同于使用 `ColumnOperators.like()` 进行否定，即 `~x.like(y)`。

在版本 1.4 中发生了变化：`not_like()` 运算符从之前的版本中的 `notlike()` 重新命名。以前的名称仍然可用于向后兼容。

另请参见

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*从* `ColumnOperators.nulls_first()` *方法继承* *自* `ColumnOperators`。

针对父对象产生一个 `nulls_first()` 子句。

在版本 1.4 中发生了变化：`nulls_first()` 运算符从之前的版本中的 `nullsfirst()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*从* `ColumnOperators.nulls_last()` *方法继承* *自* `ColumnOperators`。

针对父对象产生一个 `nulls_last()` 子句。

在版本 1.4 中发生了变化：`nulls_last()` 运算符从之前的版本中的 `nullslast()` 重新命名。以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*从* `ColumnOperators.nullsfirst()` *方法继承* *自* `ColumnOperators`。

生成针对父对象的`nulls_first()`子句。

在 1.4 版本中更改：`nulls_first()`运算符从先前的版本中的`nullsfirst()`重命名。之前的名称仍然可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法* `ColumnOperators`

生成针对父对象的`nulls_last()`子句。

在 1.4 版本中更改：`nulls_last()`运算符从先前的版本中的`nullslast()`重命名。之前的名称仍然可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法* `Operators`

生成通用运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

此函数也可用于使位运算符明确化。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中的值的按位与。

参数：

+   `opstring` – 将作为此元素与传递给生成函数的表达式之间的中缀运算符输出的字符串。

+   `precedence` –

    数据库预期应用于 SQL 表达式中的运算符的优先级。这个整数值作为一个提示，让 SQL 编译器知道何时应该在特定操作周围渲染明确的括号。较低的数字将导致在与具有更高优先级的另一个运算符应用时对表达式进行括号化。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，-100 将低于或等于所有运算符。

    另见

    我使用 op()生成自定义运算符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` –

    传统上；如果为 True，则该运算符将被视为“比较”运算符，即评估为布尔值 true/false 的运算符，例如`==`、`>`等。提供此标志是为了让 ORM 关系在自定义连接条件中使用时确认该运算符是一个比较运算符。

    使用`is_comparison`参数已经被使用`Operators.bool_op()`方法取代；这个更简洁的运算符会自动设置该参数，但也会提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表达“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine` 类或对象，将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定了`Operators.op.is_comparison`的运算符将解析为`Boolean`，而没有指定的运算符将与左操作数的类型相同。

+   `python_impl` –

    一个可选的 Python 函数，可以以与数据库服务器上运行此运算符时相同的方式评估两个 Python 值。用于在 Python 中的 SQL 表达式评估函数，例如 ORM 混合属性，以及在多行更新或删除后用于匹配会话中的对象的 ORM“评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也适用于非 SQL 的左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版中的新功能。

另请参阅

`Operators.bool_op()`

重新定义和创建新的运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数进行操作。

这是最低级别的操作，默认情况下会引发`NotImplementedError`。

在子类上覆盖这个可以允许将通用行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的“另一侧”。对于大多数操作，将是一个单一的标量。

+   `**kwargs` – 修饰符。这些可以通过特殊的运算符传递，比如`ColumnOperators.contains()`。

```py
method params(_ClauseElement__optionaldict: Mapping[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement.params()` *方法的* `ClauseElement`

返回一个副本，其中包含替换的`bindparam()`元素。

返回一个将 `bindparam()` 元素替换为给定字典中的值的此 ClauseElement 的副本：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute primary_key: bool = False
```

```py
attribute proxy_set: util.generic_fn_descriptor[FrozenSet[Any]]
```

我们正在代理的所有列的集合

从 2.0 开始，这是显式的去注释列。之前它实际上是去注释的列，但并没有被强制执行。注释列应该尽可能不要进入集合，因为它们的哈希行为非常低效。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现了特定于数据库的“正则表达式匹配”操作符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 试图解析为后端提供的类似于 REGEXP 的函数或操作符，但是具体的正则表达式语法和可用的标志并**不是后端无关的**。

示例包括：

+   PostgreSQL - 在否定时渲染 `x ~ y` 或 `x !~ y`。

+   Oracle - 渲染 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符运算符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊的实现。

+   没有任何特殊实现的后端将作为“REGEXP”或“NOT REGEXP”发出该操作符。例如，这与 SQLite 和 MySQL 兼容。

当前实现了 Oracle、PostgreSQL、MySQL 和 MariaDB 的正则表达式支持。对 SQLite 提供了部分支持。第三方方言之间的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为纯 Python 字符串传递。这些标志是特定于后端的。某些后端，如 PostgreSQL 和 MariaDB，也可以将标志作为模式的一部分指定。在 PostgreSQL 中使用忽略大小写标志 ‘i’ 时，将使用忽略大小写的正则表达式匹配运算符 `~*` 或 `!~*`。

1.4 版本中的新功能。

在版本 1.4.48 改变：2.0.18 注意，由于一个实现错误，“flags”参数先前接受了 SQL 表达式对象，如列表达式，除了普通的 Python 字符串。这种实现与缓存一起使用时不能正常工作，已被删除；应该只传递字符串作为“flags”参数，因为这些标志在 SQL 表达式中被呈现为文字内联值。

另请参见

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.regexp_replace()` *方法*

实现特定于数据库的‘正则表达式替换’运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 试图解析后端提供的类似 REGEXP_REPLACE 的函数，通常会生成函数 `REGEXP_REPLACE()`。但是，具体的正则表达式语法和可用标志**不是跨后端的**。

目前 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 已实现正则表达式替换支持。第三方方言的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅传递为普通的 Python 字符串。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分来指定。

新版版本 1.4。

在版本 1.4.48 改动：2.0.18 请注意，由于实现错误，之前“flags”参数接受 SQL 表达式对象，例如列表达式，而不仅限于普通的 Python 字符串。这种实现与缓存不兼容，并已删除；应仅传递字符串作为“flags”参数，因为这些标志会作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_match()`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[Any]
```

在参数上执行反向操作。

使用方法与 `operate()` 相同。

```py
method self_group(against: OperatorType | None = None) → ColumnElement[Any]
```

对此 `ClauseElement` 应用一个‘分组’。

此方法由子类重写，以返回一个“分组”构造，即括号。特别是当“二元”表达式放置到更大表达式中时，它被“二元”表达式使用以提供环绕自己的分组，以及当 `select()` 构造放置到另一个 `select()` 的 FROM 子句中时。（请注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句具有名称）。

当表达式组合在一起时，`self_group()` 的应用是自动的 - 最终用户代码不应该直接使用这个方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此在诸如 `x OR (y AND z)` 这样的表达式中，可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
method shares_lineage(othercolumn: ColumnElement[Any]) → bool
```

如果给定的 `ColumnElement` 有一个与这个 `ColumnElement` 共同的祖先，则返回 True。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.startswith()` *方法*

实现 `startswith` 操作符。

生成一个 LIKE 表达式，用于测试字符串值的开头是否匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于操作符使用 `LIKE`，存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.startswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现应用转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.startswith.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个普通字符串值，但也可以是任意 SQL 表达式。除非将 `ColumnOperators.startswith.autoescape` 标志设置为 True，否则不会对 LIKE 通配符字符 `%` 和 `_` 进行转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    诸如以下表达式：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    使用值 `:param` 作为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来将该字符确定为转义字符。然后可以将此字符放在`%`和`_`之前，以使它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.startswith.autoescape`结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute stringify_dialect = 'default'
```

```py
attribute supports_execution = False
```

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *属性的* `ColumnOperators`

Hack，允许将日期时间对象与左侧进行比较。

```py
attribute type: TypeEngine[_T]
```

```py
method unique_params(_ClauseElement__optionaldict: Dict[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement.unique_params()` *方法的* `ClauseElement`

返回一个副本，其中`bindparam()`元素被替换。

与`ClauseElement.params()`具有相同功能，只是将 unique=True 添加到受影响的绑定参数中，以便可以使用多个语句。

```py
attribute uses_inspection = True
```

```py
sqlalchemy.sql.expression.ColumnExpressionArgument
```

通用“列表达式”参数。

新版本 2.0.13 中新增。

此类型用于通常表示单个 SQL 列表达式的“列”类型表达式，包括`ColumnElement`，以及将具有`__clause_element__()`方法的 ORM 映射属性。

```py
class sqlalchemy.sql.expression.ColumnOperators
```

为`ColumnElement`表达式定义布尔、比较和其他运算符。

默认情况下，所有方法都调用 `operate()` 或 `reverse_operate()`，传入 Python 内置 `operator` 模块的适当运算符函数或来自 `sqlalchemy.expression.operators` 的特定于 SQLAlchemy 的运算符函数。例如 `__eq__` 函数：

```py
def __eq__(self, other):
    return self.operate(operators.eq, other)
```

当 `operators.eq` 实际上是：

```py
def eq(a, b):
    return a == b
```

核心列表达单元 `ColumnElement` 覆盖 `Operators.operate()` 和其他方法以返回更多的 `ColumnElement` 构造，因此上述的 `==` 操作被替换为一个子句构造。

另请参阅

重新定义和创建新的运算符

`TypeEngine.comparator_factory`

`ColumnOperators`

`PropComparator`

**成员**

__add__(), __and__(), __eq__(), __floordiv__(), __ge__(), __getitem__(), __gt__(), __hash__(), __invert__(), __le__(), __lshift__(), __lt__(), __mod__(), __mul__(), __ne__(), __neg__(), __or__(), __radd__(), __rfloordiv__(), __rmod__(), __rmul__(), __rshift__(), __rsub__(), __rtruediv__(), __sa_operate__(), __sub__(), __truediv__(), all_(), any_(), asc(), between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), regexp_match(), regexp_replace(), reverse_operate(), startswith(), timetuple

**类签名**

类`sqlalchemy.sql.expression.ColumnOperators`（`sqlalchemy.sql.expression.Operators`）

```py
method __add__(other: Any) → ColumnOperators
```

实现`+`运算符。

在列上下文中，如果父对象具有非字符串亲和性，则生成子句`a + b`。如果父对象具有字符串亲和性，则生成连接运算符`a || b` - 请参阅`ColumnOperators.concat()`。

```py
method __and__(other: Any) → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__and__` *方法*。

实现`&`运算符。

当与 SQL 表达式一起使用时，会导致 AND 操作，等同于 `and_()`，即：

```py
a & b
```

等同于：

```py
from sqlalchemy import and_
and_(a, b)
```

使用`&`时应注意运算符优先级；`&`运算符具有最高优先级。如果操作数包含更多子表达式，则应将其括在括号中：

```py
(a == 2) & (b == 4)
```

```py
method __eq__(other: Any) → ColumnOperators
```

实现`==`运算符。

在列上下文中，生成子句`a = b`。如果目标为`None`，则生成`a IS NULL`。

```py
method __floordiv__(other: Any) → ColumnOperators
```

实现`//`运算符。

在列上下文中，生成子句`a / b`，这与“truediv”相同，但考虑结果类型为整数。

版本 2.0 中的新功能。

```py
method __ge__(other: Any) → ColumnOperators
```

实现`>=`运算符。

在列上下文中，生成子句`a >= b`。

```py
method __getitem__(index: Any) → ColumnOperators
```

实现`[]`运算符。

这可以被某些特定于数据库的类型使用，例如 PostgreSQL ARRAY 和 HSTORE。

```py
method __gt__(other: Any) → ColumnOperators
```

实现`>`运算符。

在列上下文中，生成子句`a > b`。

```py
method __hash__()
```

返回`hash(self)`。

```py
method __invert__() → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__invert__` *方法*。

实现`~`运算符。

当与 SQL 表达式一起使用时，会导致 NOT 操作，等同于 `not_()`，即：

```py
~a
```

等同于：

```py
from sqlalchemy import not_
not_(a)
```

```py
method __le__(other: Any) → ColumnOperators
```

实现`<=`运算符。

在列上下文中，生成子句`a <= b`。

```py
method __lshift__(other: Any) → ColumnOperators
```

实现`<<`运算符。

不被 SQLAlchemy 核心使用，这是为想要使用 << 作为扩展点的自定义运算符系统提供的。

```py
method __lt__(other: Any) → ColumnOperators
```

实现`<`运算符。

在列上下文中，生成子句`a < b`。

```py
method __mod__(other: Any) → ColumnOperators
```

实现`%`运算符。

在列上下文中，生成子句`a % b`。

```py
method __mul__(other: Any) → ColumnOperators
```

实现`*`运算符。

在列上下文中，生成子句`a * b`。

```py
method __ne__(other: Any) → ColumnOperators
```

实现`!=`运算符。

在列上下文中，生成子句`a != b`。如果目标为`None`，则生成`a IS NOT NULL`。

```py
method __neg__() → ColumnOperators
```

实现`-`运算符。

在列上下文中，生成子句`-a`。

```py
method __or__(other: Any) → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__or__` *方法*

实现 `|` 运算符。

在与 SQL 表达式一起使用时，导致 OR 操作的结果，等效于 `or_()`，即：

```py
a | b
```

等价于：

```py
from sqlalchemy import or_
or_(a, b)
```

使用 `|` 时应注意运算符优先级；`|` 运算符具有最高优先级。如果操作数包含更多子表达式，则应将其括在括号中：

```py
(a == 2) | (b == 4)
```

```py
method __radd__(other: Any) → ColumnOperators
```

以反向方式实现 `+` 运算符。

参见 `ColumnOperators.__add__()`。

```py
method __rfloordiv__(other: Any) → ColumnOperators
```

以反向方式实现 `//` 运算符。

参见 `ColumnOperators.__floordiv__()`。

```py
method __rmod__(other: Any) → ColumnOperators
```

以反向方式实现 `%` 运算符。

参见 `ColumnOperators.__mod__()`。

```py
method __rmul__(other: Any) → ColumnOperators
```

以反向方式实现 `*` 运算符。

参见 `ColumnOperators.__mul__()`。

```py
method __rshift__(other: Any) → ColumnOperators
```

实现 `>>` 运算符。

SQLAlchemy 核心未使用此运算符，而是为希望使用 `>>` 作为扩展点的自定义运算符系统提供的。

```py
method __rsub__(other: Any) → ColumnOperators
```

以反向方式实现 `-` 运算符。

参见 `ColumnOperators.__sub__()`。

```py
method __rtruediv__(other: Any) → ColumnOperators
```

以反向方式实现 `/` 运算符。

参见 `ColumnOperators.__truediv__()`。

```py
method __sa_operate__(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__sa_operate__` *方法*

对参数进行操作。

这是操作的最低级别，默认引发 `NotImplementedError`。

在子类上覆盖此方法可以使常见行为适用于所有操作。例如，覆盖 `ColumnOperators` 以将 `func.lower()` 应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的“other”一侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可以通过特殊运算符传递，如 `ColumnOperators.contains()`。

```py
method __sub__(other: Any) → ColumnOperators
```

实现 `-` 运算符。

在列上下文中，生成子句 `a - b`。

```py
method __truediv__(other: Any) → ColumnOperators
```

实现 `/` 运算符。

在列上下文中，生成子句 `a / b`，并将结果类型视为数字。

2.0 版本中的更改：针对两个整数的 truediv 运算现在被认为返回数值。在特定后端上的行为可能会有所不同。

```py
method all_() → ColumnOperators
```

生成针对父对象的 `all_()` 子句。

请参阅 `all_()` 的文档以获取示例。

注意

请务必不要将较新的 `ColumnOperators.all_()` 方法与此方法的**传统**版本混淆，即 `Comparator.all()` 方法，该方法专用于 `ARRAY`，其使用不同的调用样式。

```py
method any_() → ColumnOperators
```

生成针对父对象的 `any_()` 子句。

请参阅 `any_()` 的文档以获取示例。

注意

请务必不要将较新的 `ColumnOperators.any_()` 方法与此方法的**传统**版本混淆，即 `Comparator.any()` 方法，该方法专用于 `ARRAY`，其使用不同的调用样式。

```py
method asc() → ColumnOperators
```

生成针对父对象的 `asc()` 子句。

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

生成针对父对象的 `between()` 子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

执行按位 AND 运算，通常通过 `&` 运算符执行。

新版本 2.0.2 中的更新。

另请参阅

按位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

执行按位 LSHIFT 运算，通常通过 `<<` 运算符执行。

新版本 2.0.2 中的更新。

另请参阅

按位运算符

```py
method bitwise_not() → ColumnOperators
```

执行按位 NOT 运算，通常通过 `~` 运算符执行。

新版本 2.0.2 中的更新。

另请参阅

按位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

执行按位 OR 运算，通常通过 `|` 运算符执行。

新版本 2.0.2 中的更新。

另请参阅

按位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

执行按位 RSHIFT 运算，通常通过 `>>` 运算符执行。

新版本 2.0.2 中的更新。

另请参阅

按位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

执行按位 XOR 运算，通常通过 `^` 运算符执行，或 PostgreSQL 中使用 `#`。

新版本 2.0.2 中的更新。

另请参阅

按位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

此方法是调用`Operators.op()`并传递`Operators.op.is_comparison`标志为 True 的简写。使用`Operators.bool_op()`的一个关键优势是，在使用列构造时，返回的表达式的“布尔”性质将出现在[**PEP 484**](https://peps.python.org/pep-0484/)中。

另见

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

生成针对父对象的`collate()`子句，给定排序规则字符串。

另见

`collate()`

```py
method concat(other: Any) → ColumnOperators
```

实现‘concat’运算符。

在列上下文中，生成子句`a || b`，或在 MySQL 上使用`concat()`运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

实现‘contains’运算符。

生成针对字符串值中间的匹配的 LIKE 表达式：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于操作符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样行为。对于字面字符串值，可以将`ColumnOperators.contains.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，以便它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.contains.escape`参数将建立给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.contains.autoescape`标志设置为 True，否则 LIKE 通配符字符`%`和`_`不会被转义。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值为字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    以值 `:param` 为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用 `ESCAPE` 关键字来将该字符建立为转义字符。然后，可以将该字符放在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况中，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method desc() → ColumnOperators
```

产生一个针对父对象的 `desc()` 子句。

```py
method distinct() → ColumnOperators
```

产生一个针对父对象的 `distinct()` 子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现 'endswith' 运算符。

生成一个针对字符串值结尾的匹配的 LIKE 表达式：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于该运算符使用 `LIKE`，因此在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.endswith.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使它们与自身匹配而不是作为通配符字符。或者，`ColumnOperators.endswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时，这可能是有用的。

参数：

+   `other` – 待比较的表达式。通常是普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不会被转义，除非 `ColumnOperators.endswith.autoescape` 标志被设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有 `"%"`、`"_"` 和转义字符本身的出现，假定比较值是字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    使用值 `:param` 作为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用 `ESCAPE` 关键字来将该字符设定为转义字符。然后可以将该字符放在 `%` 和 `_` 的前面，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.endswith.autoescape` 结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

实现 `icontains` 操作符，例如 `ColumnOperators.contains()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值中间的大小写不敏感匹配进行测试：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于该操作符使用 `LIKE`，在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也将像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.icontains.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现进行转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.icontains.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符 `%` 和 `_` 默认情况下不会被转义，除非设置了 `ColumnOperators.icontains.autoescape` 标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的 `"%"`、`"_"` 和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    其中`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字呈现，以将该字符作为转义字符。然后，可以将此字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    该参数还可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现`iendswith`运算符，例如，`ColumnOperators.endswith()`的不区分大小写版本。

生成一个对字符串值末尾进行不区分大小写匹配的 LIKE 表达式：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该运算符使用`LIKE`，因此在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.iendswith.escape`参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。除非将`ColumnOperators.iendswith.autoescape`标志设置为 True，否则 LIKE 通配符字符`%`和`_`默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有的`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其中`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字呈现，以将该字符作为转义字符。然后，可以将此字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.iendswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的例子中，给定的文字参数将在传递给数据库之前转换为`"foo^%bar^^bat"`。

参见

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

实现`ilike`运算符，例如，大小写不敏感的 LIKE。

在列上下文中，生成的表达式形式为：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

参见

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

实现`in`运算符。

在列上下文中，生成子句`column IN <other>`。

给定的参数`other`可以是：

+   一个文字值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表将转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的`tuple_()`的元组，则可以提供一个元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，表达式呈现出一个“空集”表达式。这些表达式针对各个后端进行了定制，通常试图得到一个空的 SELECT 语句作为子查询。例如，在 SQLite 上，表达式是：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    从版本 1.4 开始更改：在所有情况下，空 IN 表达式现在都使用运行时生成的 SELECT 子查询。

+   如果包含了`bindparam.expanding`标志，则可以使用绑定的参数，例如`bindparam()`：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，表达式呈现出一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，以转换为前面所示的可变数量的绑定参数形式。如果执行语句为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    新版本 1.2 中：添加了“expanding”绑定参数

    如果传递了空列表，则会呈现一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    新版本 1.3 中：现在“expanding”绑定参数支持空列表

+   一个`select()`构造，通常是相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()`呈现如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一系列文字、一个 `select()` 构造，或者一个包含设置为 True 的 `bindparam.expanding` 标志的 `bindparam()` 构造。

```py
method is_(other: Any) → ColumnOperators
```

实现 `IS` 操作符。

通常情况下，与 `None` 值进行比较时会自动生成 `IS`，其解析为 `NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用 `IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

实现 `IS DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS DISTINCT FROM b”；在某些平台上，例如 SQLite 可以呈现“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

实现 `IS NOT` 操作符。

通常情况下，与 `None` 值进行比较时会自动生成 `IS NOT`，其解析为 `NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用 `IS NOT`。

在版本 1.4 中更改：`is_not()` 操作符从以前的版本中重命名为 `isnot()`。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

实现 `IS NOT DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台上，例如 SQLite 可以呈现“a IS b”。

在版本 1.4 中更改：`is_not_distinct_from()` 操作符从以前的版本中重命名为 `isnot_distinct_from()`。以前的名称仍可用于向后兼容。

```py
method isnot(other: Any) → ColumnOperators
```

实现 `IS NOT` 操作符。

通常情况下，与 `None` 值进行比较时会自动生成 `IS NOT`，其解析为 `NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用 `IS NOT`。

在版本 1.4 中更改：`is_not()` 操作符从以前的版本中重命名为 `isnot()`。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

实现 `IS NOT DISTINCT FROM` 操作符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台上，例如 SQLite 可以呈现“a IS b”。

在版本 1.4 中更改：`is_not_distinct_from()` 操作符从以前的版本中重命名为 `isnot_distinct_from()`。以前的名称仍可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现 `istartswith` 操作符，例如 `ColumnOperators.startswith()` 的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的开头进行不区分大小写的匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于运算符使用了`LIKE`，所以存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于字面字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为`True`，以将这些字符在字符串值中的出现进行转义，使它们作为自身匹配，而不是作为通配符字符。另外，`ColumnOperators.istartswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 待比较的表达式。通常是普通字符串值，但也可以是任意的 SQL 表达式。除非设置了`ColumnOperators.istartswith.autoescape`标志为 True，否则 LIKE 通配符字符`%`和`_`不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值是字面字符串而不是 SQL 表达式。

    例如以下表达式：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    值为`:param`，为`"foo/%bar"`。

+   `escape` –

    当给定时，将呈现一个字符与`ESCAPE`关键字以建立该字符作为转义字符。然后可以将此字符放在`%`和`_`的前面以允许它们作为自身而不是通配符字符。

    例如以下表达式：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    此参数也可以与`ColumnOperators.istartswith.autoescape`组合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另见

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

实现`like`运算符。

在列上下文中，产生表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 待比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另见

`ColumnOperators.ilike()`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

实现特定于数据库的‘match’运算符。

`ColumnOperators.match()` 尝试解析为后端提供的类似 MATCH 的函数或操作符。 例如：

+   PostgreSQL - 渲染 `x @@ plainto_tsquery(y)`

    > 自版本 2.0 起：现在对于 PostgreSQL 使用 `plainto_tsquery()` 而不是 `to_tsquery()`；为了与其他形式兼容，请参阅全文搜索。

+   MySQL - 渲染 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参见

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 渲染 `CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将操作符输出为“MATCH”。 例如，这与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT ILIKE` 操作符。

这相当于对 `ColumnOperators.ilike()` 使用否定，即 `~x.ilike(y)`。

自版本 1.4 起：`not_ilike()` 操作符从先前版本的 `notilike()` 重命名。 以前的名称仍可用于向后兼容。

另请参见

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

实现 `NOT IN` 操作符。

这相当于对 `ColumnOperators.in_()` 使用否定，即 `~x.in_(y)`。

如果 `other` 是一个空序列，则编译器会生成一个“空 not in” 表达式。 默认情况下，这将产生一个“1 = 1” 表达式，以在所有情况下产生 true。 可以使用 `create_engine.empty_in_strategy` 来更改此行为。

自版本 1.4 起：`not_in()` 操作符从先前版本的 `notin_()` 重命名。 以前的名称仍可用于向后兼容。

自版本 1.2 起：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认为一个空的 IN 序列生成一个“静态” 表达式。

另请参见

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT LIKE` 操作符。

这相当于对 `ColumnOperators.like()` 使用否定，即 `~x.like(y)`。

从版本 1.4 开始更改：`not_like()` 操作符在先前版本中从 `notlike()` 重命名。 以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT ILIKE` 操作符。

这相当于对 `ColumnOperators.ilike()` 使用否定，即 `~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()` 操作符在先前版本中从 `notilike()` 重命名。 以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

实现 `NOT IN` 操作符。

这相当于对 `ColumnOperators.in_()` 使用否定，即 `~x.in_(y)`。

在 `other` 是空序列的情况下，编译器生成一个“空的 not in” 表达式。 这默认为表达式 “1 = 1”，以在所有情况下生成 true。 `create_engine.empty_in_strategy` 可用于更改此行为。

从版本 1.4 开始更改：`not_in()` 操作符在先前版本中从 `notin_()` 重命名。 以前的名称仍然可用于向后兼容。

从版本 1.2 开始更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认情况下会为空的 IN 序列生成一个“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT LIKE` 操作符。

这相当于对 `ColumnOperators.like()` 使用否定，即 `~x.like(y)`。

从版本 1.4 开始更改：`not_like()` 操作符在先前版本中从 `notlike()` 重命名。 以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

生成针对父对象的 `nulls_first()` 子句。

从版本 1.4 开始更改：`nulls_first()` 操作符在先前版本中从 `nullsfirst()` 重命名。 以前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

对父对象生成一个`nulls_last()`子句。

从版本 1.4 开始：`nulls_last()`运算符在以前的版本中从`nullslast()`重命名。以前的名称仍然可用以实现向后兼容。

```py
method nullsfirst() → ColumnOperators
```

对父对象生成一个`nulls_first()`子句。

从版本 1.4 开始：`nulls_first()`运算符在以前的版本中从`nullsfirst()`重命名。以前的名称仍然可用以实现向后兼容。

```py
method nullslast() → ColumnOperators
```

对父对象生成一个`nulls_last()`子句。

从版本 1.4 开始：`nulls_last()`运算符在以前的版本中从`nullslast()`重命名。以前的名称仍然可用以实现向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用的操作符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

此函数还可用于明确表示位运算符。例如：

```py
somecolumn.op('&')(0xff)
```

是在`somecolumn`中的值的按位与。

参数：

+   `opstring` – 一个字符串，将在此元素和传递给生成函数的表达式之间输出为中缀操作符。

+   `precedence` –

    预期数据库在 SQL 表达式中应用于操作符的优先级。此整数值作为 SQL 编译器的提示，用于确定何时应在特定操作周围呈现显式括号。当应用于具有更高优先级的另一个操作符时，较低的数字将导致表达式被括在括号中。默认值`0`低于所有操作符，除了逗号（`,`）和`AS`操作符外。值为 100 将高于或等于所有操作符，-100 将低于或等于所有操作符。

    另请参见

    我正在使用 op()生成自定义操作符，但是我的括号不正确 - SQLAlchemy SQL 编译器如何渲染括号的详细描述

+   `is_comparison` –

    legacy；如果为 True，则将操作符视为“比较”操作符，即评估为布尔值 true/false 的操作符，如`==`、`>`等。提供此标志是为了让 ORM 关系能够确定当在自定义连接条件中使用操作符时，该操作符是比较操作符。

    使用`is_comparison`参数已经被使用`Operators.bool_op()`方法取代；这个更简洁的操作符会自动设置这个参数，但也会提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，它将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的运算符将解析为`Boolean`，而不指定的运算符将与左操作数的类型相同。

+   `python_impl` –

    可选的 Python 函数，可以以与在数据库服务器上运行此运算符时运行相同的方式评估两个 Python 值。用于在 Python 中进行 SQL 表达式评估函数，例如用于 ORM 混合属性的函数，以及用于在多行更新或删除后匹配会话中的对象的 ORM “评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也将适用于非 SQL 左侧和右侧对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    新版本 2.0 中新增。

请参见

`Operators.bool_op()`

重新定义和创建新运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*从* `Operators.operate()` *方法继承*

对参数执行操作。

这是操作的最低级别，默认情况下引发`NotImplementedError`。

在子类中重写这一点可以使常见行为应用于所有操作。例如，重写`ColumnOperators`以将`func.lower()`应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的“其他”一侧。对于大多数操作，它将是一个单个标量。

+   `**kwargs` – 修饰符。这些可能由特殊运算符传递，如`ColumnOperators.contains()`。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

实现数据库特定的‘正则表达式匹配’运算符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 试图解析为后端提供的类似 REGEXP 的函数或操作符，然而具体的正则表达式语法和可用标志**与后端无关**。

示例包括：

+   PostgreSQL - 渲染为 `x ~ y` 或当否定时 `x !~ y`。

+   Oracle - 渲染为 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符操作符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将生成操作符为 “REGEXP” 或 “NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

正则表达式支持当前已经在 Oracle、PostgreSQL、MySQL 和 MariaDB 中实现。对于 SQLite，提供了部分支持。第三方方言的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通的 Python 字符串传递。这些标志是后端特定的。一些后端，如 PostgreSQL 和 MariaDB，可能也会将标志作为模式的一部分来指定。在 PostgreSQL 中使用忽略大小写标志 ‘i’ 时，将使用忽略大小写的正则表达式匹配操作符 `~*` 或 `!~*`。

新版本 1.4 中新增。

从版本 1.4.48 改变，：2.0.18 请注意，由于实现错误，“flags” 参数先前接受了 SQL 表达式对象，如列表达式，除了普通的 Python 字符串。这个实现与缓存一起无法正确工作，因此被删除；应该只传递字符串给 “flags” 参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

实现了特定于数据库的“正则表达式替换”操作符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 试图解析为由后端提供的类似 REGEXP_REPLACE 的函数，通常会生成函数 `REGEXP_REPLACE()`。然而，具体的正则表达式语法和可用标志**与后端相关**。

正则表达式替换支持当前已在 Oracle、PostgreSQL、MySQL 8 或更高版本以及 MariaDB 中实现。第三方方言的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通的 Python 字符串传递。这些标志是后端特定的。一些后端，如 PostgreSQL 和 MariaDB，可能也会将标志作为模式的一部分来指定。

新版本 1.4 中新增。

自版本 1.4.48 更改，: 2.0.18 请注意，由于实现错误，"flags"参数先前接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。该实现与缓存一起不正确地工作，并已被删除；"flags"参数应仅传递字符串，因为这些标志被呈现为 SQL 表达式中的文字内联值。

另请参见

`ColumnOperators.regexp_match()`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*从* `Operators` *的* `Operators.reverse_operate()` *方法继承的*

对参数进行反向操作。

使用方法与`operate()`相同。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现`startswith`操作符。

生成一个 LIKE 表达式，用于测试字符串值开头的匹配：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该操作符使用`LIKE`，因此存在于<other>表达式内部的通配符字符`"%"`和`"_"`也将像通配符一样运行。对于文字字符串值，可以将`ColumnOperators.startswith.autoescape`标志设置为`True`，以将这些字符的出现转义为字符串值内部的这些字符，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.startswith.escape`参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个纯字符串值，但也可以是任意的 SQL 表达式。除非设置了`ColumnOperators.startswith.autoescape`标志为 True，否则`LIKE`通配符字符`%`和`_`默认不会被转义。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值是一个文字字符串而不是 SQL 表达式。

    一个类似于：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    其值为`:param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来建立该字符作为转义字符。然后可以将该字符放在`%`和`_`之前的位置，以允许它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.startswith.autoescape` 结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另见

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute timetuple: Literal[None] = None
```

黑客，允许在左边比较日期时间对象。

```py
class sqlalchemy.sql.expression.Extract
```

表示一个 SQL EXTRACT 子句，`extract(field FROM expr)`。

**类签名**

类 `sqlalchemy.sql.expression.Extract` (`sqlalchemy.sql.expression.ColumnElement`)

```py
class sqlalchemy.sql.expression.False_
```

表示 SQL 语句中的 `false` 关键字，或者等效的。

`False_` 通过 `false()` 函数被访问为一个常量。

**类签名**

类 `sqlalchemy.sql.expression.False_` (`sqlalchemy.sql.expression.SingletonConstant`, `sqlalchemy.sql.roles.ConstExprRole`, `sqlalchemy.sql.expression.ColumnElement`)

```py
class sqlalchemy.sql.expression.FunctionFilter
```

表示一个函数 FILTER 子句。

这是针对聚合和窗口函数的特殊操作符，用于控制传递给它的行。仅受某些数据库后端支持。

调用 `FunctionFilter` 是通过 `FunctionElement.filter()` 进行的：

```py
func.count(1).filter(True)
```

另见

`FunctionElement.filter()`

**成员**

filter(), over(), self_group()

**类签名**

类 `sqlalchemy.sql.expression.FunctionFilter` (`sqlalchemy.sql.expression.ColumnElement`)

```py
method filter(*criterion: _ColumnExpressionArgument[bool]) → Self
```

对该函数进行额外的过滤。

这个方法向由`FunctionElement.filter()`设置的初始条件添加额外的条件。

多个条件在 SQL 渲染时通过`AND`连接在一起。

```py
method over(partition_by: Iterable[_ColumnExpressionArgument[Any]] | _ColumnExpressionArgument[Any] | None = None, order_by: Iterable[_ColumnExpressionArgument[Any]] | _ColumnExpressionArgument[Any] | None = None, range_: typing_Tuple[int | None, int | None] | None = None, rows: typing_Tuple[int | None, int | None] | None = None) → Over[_T]
```

对这个过滤函数产生一个 OVER 子句。

用于聚合或所谓的“窗口”函数，适用于支持窗口函数的数据库后端。

表达式：

```py
func.rank().filter(MyClass.y > 5).over(order_by='x')
```

是以下内容的简写：

```py
from sqlalchemy import over, funcfilter
over(funcfilter(func.rank(), MyClass.y > 5), order_by='x')
```

查看`over()`获取完整描述。

```py
method self_group(against: OperatorType | None = None) → Self | Grouping[_T]
```

对这个`ClauseElement`应用一个“分组”。

这个方法被子类重写以返回一个“分组”构造，即括号。特别是它被“二进制”表达式使用，当它们被放置到更大的表达式中时提供一个围绕自身的分组，以及当它们被放置到另一个`select()`的 FROM 子句中时，由`select()`构造使用。（请注意，子查询通常应该使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

当表达式组合在一起时，`self_group()`的应用是自动的 - 最终用户代码不应该直接使用这个方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在表达式中可能不需要括号，例如，`x OR (y AND z)` - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
class sqlalchemy.sql.expression.Label
```

代表一个列标签（AS）。

代表一个标签，通常使用`AS` sql 关键字应用于任何列级元素。

**成员**

foreign_keys, primary_key, self_group()

**类签名**

类`sqlalchemy.sql.expression.Label` (`sqlalchemy.sql.roles.LabeledColumnExprRole`, `sqlalchemy.sql.expression.NamedColumn`)

```py
attribute foreign_keys: AbstractSet[ForeignKey]
```

```py
attribute primary_key: bool
```

```py
method self_group(against=None)
```

对这个`ClauseElement`应用一个“分组”。

子类会重写此方法以返回“分组”结构，即括号。特别是，当“二进制”表达式放置到更大的表达式中时，它们会提供围绕自己的分组，以及当`select()`构造放置到另一个`select()`的 FROM 子句中时。 （请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句具有名称）。

当表达式组合在一起时，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了操作符优先级 - 因此，可能不需要括号，例如，在表达式`x OR (y AND z)`中 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
class sqlalchemy.sql.expression.Null
```

在 SQL 语句中表示 NULL 关键字。

`Null`通过`null()`函数作为常量访问。

**类签名**

类`sqlalchemy.sql.expression.Null` (`sqlalchemy.sql.expression.SingletonConstant`，`sqlalchemy.sql.roles.ConstExprRole`，`sqlalchemy.sql.expression.ColumnElement`)

```py
class sqlalchemy.sql.expression.Operators
```

比较基础和逻辑运算符。

实现基本方法`Operators.operate()`和`Operators.reverse_operate()`，以及`Operators.__and__()`，`Operators.__or__()`，`Operators.__invert__()`。

**成员**

__and__(), __invert__(), __or__(), __sa_operate__(), bool_op(), op(), operate(), reverse_operate()

通常通过其最常见的子类`ColumnOperators`使用。

```py
method __and__(other: Any) → Operators
```

实现`&`运算符。

当与 SQL 表达式一起使用时，会导致一个 AND 操作，相当于`and_()`，即：

```py
a & b
```

相当于：

```py
from sqlalchemy import and_
and_(a, b)
```

在使用`&`时应注意运算符优先级；`&`运算符具有最高优先级。如果操作数包含进一步的子表达式，则应将其括在括号中：

```py
(a == 2) & (b == 4)
```

```py
method __invert__() → Operators
```

实现`~`运算符。

当与 SQL 表达式一起使用时，会导致一个 NOT 操作，相当于`not_()`，即：

```py
~a
```

相当于：

```py
from sqlalchemy import not_
not_(a)
```

```py
method __or__(other: Any) → Operators
```

实现`|`运算符。

当与 SQL 表达式一起使用时，会导致一个 OR 操作，相当于`or_()`，即：

```py
a | b
```

相当于：

```py
from sqlalchemy import or_
or_(a, b)
```

在使用`|`时应注意运算符优先级；`|`运算符具有最高优先级。如果操作数包含进一步的子表达式，则应将其括在括号中：

```py
(a == 2) | (b == 4)
```

```py
method __sa_operate__(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

对参数进行操作。

这是操作的最低级别，通常默认引发`NotImplementedError`。

在子类上覆盖这个可以允许将常见行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的‘另一’侧。对于大多数操作，将是一个单一标量。

+   `**kwargs` – 修饰符。这些可以通过特殊运算符传递，比如`ColumnOperators.contains()`。

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

返回一个自定义布尔运算符。

此方法是调用`Operators.op()`并传递带有 True 的`Operators.op.is_comparison`标志的简写。使用`Operators.bool_op()`的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在[**PEP 484**](https://peps.python.org/pep-0484/)中。

另请参阅

`Operators.op()`

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

生成一个通用运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

此函数还可用于使位运算符明确。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位 AND。

参数：

+   `opstring` – 一个字符串，将作为此元素与传递给生成函数的表达式之间的中缀运算符输出。

+   `precedence` –

    操作符在 SQL 表达式中预期数据库应用的优先级。此整数值作为 SQL 编译器的提示，用于了解何时应在特定操作周围呈现显式括号。较低的数字将导致在与具有较高优先级的另一个操作符应用时对表达式进行括号化。默认值为`0`，低于所有操作符，除了逗号（`,`）和 `AS` 操作符。值为 100 将高于或等于所有操作符，-100 将低于或等于所有操作符。

    另请参阅

    我正在使用 op() 生成自定义操作符，但我的括号不正确 - SQLAlchemy SQL 编译器如何呈现括号的详细描述

+   `is_comparison` –

    传统的；如果为 True，则将该操作符视为“比较”操作符，即评估为布尔真/假值的操作符，如 `==`、`>` 等。提供此标志是为了当在自定义连接条件中使用时，ORM 关系可以确认该操作符是比较操作符。

    使用 `is_comparison` 参数已被使用 `Operators.bool_op()` 方法替代；此更简洁的操作符会自动设置此参数，但也会提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即 `BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，将强制此操作符生成的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的操作符将解析为`Boolean`，而未指定的操作符将与左操作数相同类型。

+   `python_impl` –

    一个可选的 Python 函数，可以在数据库服务器上运行此操作符时以与此操作符相同的方式评估两个 Python 值。对于在 Python 中的 SQL 表达式评估函数非常有用，例如 ORM 混合属性，以及用于在多行更新或删除后匹配会话中的对象的 ORM“评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的操作符也将适用于非 SQL 左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    新版本 2.0 中新增。

另请参阅

`Operators.bool_op()`

重新定义和创建新的操作符

在连接条件中使用自定义操作符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

对参数进行操作。

这是操作的最低级别，默认情况下引发 `NotImplementedError`。

在子类上重写此方法可以使常见行为适用于所有操作。例如，重写 `ColumnOperators` 以将 `func.lower()` 应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的“other”一侧。对于大多数操作，将是一个单一标量。

+   `**kwargs` – 修饰符。这些可能会被特殊操作符（例如 `ColumnOperators.contains()`）传递。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

对参数执行反向操作。

用法与 `operate()` 相同。

```py
class sqlalchemy.sql.expression.Over
```

表示一个 OVER 子句。

这是针对所谓的“窗口”函数以及任何聚合函数的特殊操作符，它产生相对于结果集本身的结果。大多数现代 SQL 后端现在支持窗口函数。

**成员**

element

**类签名**

class `sqlalchemy.sql.expression.Over` (`sqlalchemy.sql.expression.ColumnElement`)

```py
attribute element: ColumnElement[_T]
```

此 `Over` 对象引用的基础表达式对象。

```py
class sqlalchemy.sql.expression.SQLColumnExpression
```

可用于指示任何 SQL 列元素或充当其替代物的对象的类型。

`SQLColumnExpression` 是 `ColumnElement` 的基类，并且在 ORM 元素的基类中也是 `InstrumentedAttribute` 中的一部分，可以在 [**PEP 484**](https://peps.python.org/pep-0484/) 类型提示中用于指示应该像列表达式一样行为的参数或返回值。

在版本 2.0.0b4 中新增。

**类签名**

class `sqlalchemy.sql.expression.SQLColumnExpression` (`sqlalchemy.sql.expression.SQLCoreOperations`, `sqlalchemy.sql.roles.ExpressionElementRole`, `sqlalchemy.util.langhelpers.TypingOnly`)

```py
class sqlalchemy.sql.expression.TextClause
```

表示一个文字 SQL 文本片段。

例如：

```py
from sqlalchemy import text

t = text("SELECT * FROM users")
result = connection.execute(t)
```

使用 `text()` 函数生成 `TextClause` 构造; 参见该函数以获取完整文档。

另请参见

`text()`

**成员**

bindparams(), columns(), self_group()

**类签名**

`sqlalchemy.sql.expression.TextClause`类（`sqlalchemy.sql.roles.DDLConstraintColumnRole`、`sqlalchemy.sql.roles.DDLExpressionRole`、`sqlalchemy.sql.roles.StatementOptionRole`、`sqlalchemy.sql.roles.WhereHavingRole`、`sqlalchemy.sql.roles.OrderByRole`、`sqlalchemy.sql.roles.FromClauseRole`、`sqlalchemy.sql.roles.SelectStatementRole`、`sqlalchemy.sql.roles.InElementRole`、`sqlalchemy.sql.expression.Generative`、`sqlalchemy.sql.expression.Executable`、`sqlalchemy.sql.expression.DQLDMLClauseElement`、`sqlalchemy.sql.roles.BinaryElementRole`、`sqlalchemy.inspection.Inspectable`）

```py
method bindparams(*binds: BindParameter[Any], **names_to_values: Any) → Self
```

在这个`TextClause`结构中确定绑定参数的值和/或类型。

给定文本构造如下：

```py
from sqlalchemy import text
stmt = text("SELECT id, name FROM user WHERE name=:name "
            "AND timestamp=:timestamp")
```

`TextClause.bindparams()`方法可用于使用简单的关键字参数来确定`:name`和`:timestamp`的初始值：

```py
stmt = stmt.bindparams(name='jack',
            timestamp=datetime.datetime(2012, 10, 8, 15, 12, 5))
```

在上面的代码中，将生成新的`BindParameter`对象，名称分别为`name`和`timestamp`，值分别为`jack`和`datetime.datetime(2012, 10, 8, 15, 12, 5)`。类型将根据给定的值推断，在这种情况下为`String`和`DateTime`。

当需要特定的类型行为时，可以使用位置参数`*binds`来直接指定`bindparam()`构造。这些构造必须至少包括`key`参数，然后是一个可选的值和类型：

```py
from sqlalchemy import bindparam
stmt = stmt.bindparams(
                bindparam('name', value='jack', type_=String),
                bindparam('timestamp', type_=DateTime)
            )
```

在上面，我们为`timestamp`绑定指定了`DateTime`类型，并为`name`绑定指定了`String`类型。对于`name`，我们还设置了默认值为`"jack"`。

可以在语句执行时提供额外的绑定参数，例如：

```py
result = connection.execute(stmt,
            timestamp=datetime.datetime(2012, 10, 8, 15, 12, 5))
```

`TextClause.bindparams()`方法可以重复调用，在这里它将重用现有的`BindParameter`对象来添加新的信息。例如，我们可以首先调用`TextClause.bindparams()`来传递类型信息，然后第二次传递值信息，它将被合并：

```py
stmt = text("SELECT id, name FROM user WHERE name=:name "
            "AND timestamp=:timestamp")
stmt = stmt.bindparams(
    bindparam('name', type_=String),
    bindparam('timestamp', type_=DateTime)
)
stmt = stmt.bindparams(
    name='jack',
    timestamp=datetime.datetime(2012, 10, 8, 15, 12, 5)
)
```

`TextClause.bindparams()`方法还支持**unique**绑定参数的概念。这些是在语句编译时按名称“唯一化”的参数，因此多个`text()`构造可以合并在一起而不会冲突。要使用此功能，请在每个`bindparam()`对象上指定`BindParameter.unique`标志：

```py
stmt1 = text("select id from table where name=:name").bindparams(
    bindparam("name", value='name1', unique=True)
)
stmt2 = text("select id from table where name=:name").bindparams(
    bindparam("name", value='name2', unique=True)
)

union = union_all(
    stmt1.columns(column("id")),
    stmt2.columns(column("id"))
)
```

上述语句将呈现为：

```py
select id from table where name=:name_1
UNION ALL select id from table where name=:name_2
```

新版本 1.3.11 中新增：支持`BindParameter.unique`标志与`text()`构造一起使用。

```py
method columns(*cols: _ColumnExpressionArgument[Any], **types: TypeEngine[Any]) → TextualSelect
```

将这个`TextClause`对象转换为一个`TextualSelect`对象，它扮演了与 SELECT 语句相同的角色。

`TextualSelect`是`SelectBase`层次结构的一部分，可以通过使用`TextualSelect.subquery()`方法嵌入到另一个语句中，以生成一个`Subquery`对象，然后可以从中进行 SELECT 操作。

此函数本质上填补了纯文本 SELECT 语句与 SQL 表达式语言中“可选择”的概念之间的差距：

```py
from sqlalchemy.sql import column, text

stmt = text("SELECT id, name FROM some_table")
stmt = stmt.columns(column('id'), column('name')).subquery('st')

stmt = select(mytable).\
        select_from(
            mytable.join(stmt, mytable.c.name == stmt.c.name)
        ).where(stmt.c.id > 5)
```

在上面的示例中，我们按位置传递了一系列 `column()` 元素给 `TextClause.columns()` 方法。这些 `column()` 元素现在成为 `TextualSelect.selected_columns` 列集合的一部分，之后在调用 `TextualSelect.subquery()` 后成为 `Subquery.c` 集合的一部分。

我们传递给 `TextClause.columns()` 的列表达式也可以具有类型；当我们这样做时，这些 `TypeEngine` 对象将成为列的有效返回类型，因此 SQLAlchemy 的结果集处理系统可以用于返回值。对于诸如日期或布尔类型以及在某些方言配置中进行 Unicode 处理等类型，这通常是必需的：

```py
stmt = text("SELECT id, name, timestamp FROM some_table")
stmt = stmt.columns(
            column('id', Integer),
            column('name', Unicode),
            column('timestamp', DateTime)
        )

for id, name, timestamp in connection.execute(stmt):
    print(id, name, timestamp)
```

作为上述语法的一种快捷方式，如果只需要类型转换，则可以使用仅指向类型的关键字参数：

```py
stmt = text("SELECT id, name, timestamp FROM some_table")
stmt = stmt.columns(
            id=Integer,
            name=Unicode,
            timestamp=DateTime
        )

for id, name, timestamp in connection.execute(stmt):
    print(id, name, timestamp)
```

`TextClause.columns()` 的位置形式还提供了**位置列定位**的独特功能，当使用 ORM 处理复杂的文本查询时，这一点尤其有用。如果我们将模型中的列指定给 `TextClause.columns()`，则结果集将按位置与这些列匹配，这意味着文本 SQL 中列的名称或来源并不重要：

```py
stmt = text("SELECT users.id, addresses.id, users.id, "
     "users.name, addresses.email_address AS email "
     "FROM users JOIN addresses ON users.id=addresses.user_id "
     "WHERE users.id = 1").columns(
        User.id,
        Address.id,
        Address.user_id,
        User.name,
        Address.email_address
     )

query = session.query(User).from_statement(stmt).options(
    contains_eager(User.addresses))
```

`TextClause.columns()` 方法提供了直接调用 `FromClause.subquery()` 和 `SelectBase.cte()` 对象的途径，用于针对文本 SELECT 语句：

```py
stmt = stmt.columns(id=Integer, name=String).cte('st')

stmt = select(sometable).where(sometable.c.id == stmt.c.id)
```

参数：

+   `*cols` – 一系列 `ColumnElement` 对象，通常是来自 `Table` 或 ORM 级列映射属性的 `Column` 对象，代表此文本字符串将从中进行选择的列集合。

+   `**types` – 一个将字符串名称映射到`TypeEngine`类型对象的映射，指示从文本字符串中选择的名称使用的数据类型。最好使用`*cols`参数，因为它还指示位置顺序。

```py
method self_group(against=None)
```

对这个`ClauseElement`应用一个“分组”。

子类会重写此方法以返回一个“分组”构造，即括号。特别是当“二进制”表达式被放置到更大的表达式中时，它们会提供一个围绕自身的分组，以及当`select()`构造被放置到另一个`select()`的 FROM 子句中时。 （请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

当表达式组合在一起时，`self_group()`的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此在表达式中可能不需要括号，例如，`x OR (y AND z)` - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
class sqlalchemy.sql.expression.TryCast
```

表示一个 TRY_CAST 表达式。

`TryCast`用法的详细信息在`try_cast()`处。

另请参阅

`try_cast()`

数据转换和类型强制转换

**成员**

inherit_cache

**类签名**

类`sqlalchemy.sql.expression.TryCast`（`sqlalchemy.sql.expression.Cast`)

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

此属性默认为`None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

这个标志可以在特定类上设置为`True`，如果与对象对应的 SQL 不基于此类的局部属性而改变，而不是其超类。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的`HasCacheKey.inherit_cache` 属性的一般指南。

```py
class sqlalchemy.sql.expression.Tuple
```

表示 SQL 元组。

**成员**

`self_group()`

**类签名**

类`sqlalchemy.sql.expression.Tuple`（`sqlalchemy.sql.expression.ClauseList`，`sqlalchemy.sql.expression.ColumnElement`)

```py
method self_group(against=None)
```

将“分组”应用于此 `ClauseElement`。

此方法被子类重写以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当它们被放置到较大的表达式中时，提供一个围绕自身的分组，以及被 `select()` 构造在另一个 `select()` 的 FROM 子句中时。 （注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句具有名称）。

当表达式组合在一起时，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此可能不需要括号，例如，在表达式中像`x OR (y AND z)` - AND 的优先级高于 OR，可能不需要括号。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
class sqlalchemy.sql.expression.WithinGroup
```

表示 WITHIN GROUP (ORDER BY) 子句。

这是针对所谓的“有序集合聚合”和“假设集合聚合”函数的特殊运算符，包括 `percentile_cont()`、`rank()`、`dense_rank()` 等。

仅受某些数据库后端支持，如 PostgreSQL、Oracle 和 MS SQL Server。

`WithinGroup` 构造从方法 `FunctionElement.within_group_type()` 中提取其类型。如果此返回 `None`，则使用函数的 `.type`。

**成员**

over()

**类签名**

类 `sqlalchemy.sql.expression.WithinGroup`（`sqlalchemy.sql.expression.ColumnElement`）

```py
method over(partition_by=None, order_by=None, range_=None, rows=None)
```

对此 `WithinGroup` 构造生成一个 OVER 子句。

此函数具有与 `FunctionElement.over()` 相同的签名。

```py
class sqlalchemy.sql.elements.WrapsColumnExpression
```

定义一个 `ColumnElement` 作为具有特殊标签行为的包装器的混合体，用于已经具有名称的表达式。

从版本 1.4 新增。

另请参见

使用 CAST 或类似方法改进简单列表达式的列标签

**类签名**

类 `sqlalchemy.sql.expression.WrapsColumnExpression`（`sqlalchemy.sql.expression.ColumnElement`）

```py
class sqlalchemy.sql.expression.True_
```

代表 SQL 语句中的 `true` 关键字或等效物。

通过 `true()` 函数访问 `True_` 作为常量。

**类签名**

类 `sqlalchemy.sql.expression.True_`（`sqlalchemy.sql.expression.SingletonConstant`、`sqlalchemy.sql.roles.ConstExprRole`、`sqlalchemy.sql.expression.ColumnElement`）

```py
class sqlalchemy.sql.expression.TypeCoerce
```

表示 Python 端的类型强制转换包装器。

`TypeCoerce` 提供了 `type_coerce()` 函数；请参阅该函数以获取使用详情。

另请参见

`type_coerce()`

`cast()`

**成员**

self_group()

**类签名**

类`sqlalchemy.sql.expression.TypeCoerce`（`sqlalchemy.sql.expression.WrapsColumnExpression`）

```py
method self_group(against=None)
```

对此`ClauseElement`应用“分组”。

此方法被子类重写以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当它们放置到更大的表达式中时提供一个围绕自身的分组，以及当它们被放置到另一个`select()`构造的 FROM 子句中时，也被`select()`构造使用。（请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

当表达式组合在一起时，自动应用`self_group()` - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在表达式中可能不需要括号，例如`x OR (y AND z)` - AND 优先于 OR。

基本的`self_group()`方法在`ClauseElement`中只返回自身。

```py
class sqlalchemy.sql.expression.UnaryExpression
```

定义一个“一元”表达式。

一元表达式有一个单独的列表达式和一个运算符。运算符可以放在列表达式的左侧（称为“运算符”）或右侧（称为“修饰符”）。

`UnaryExpression`是几个一元运算符的基础，包括`desc()`、`asc()`、`distinct()`、`nulls_first()`和`nulls_last()`。

**成员**

self_group()

**类签名**

类 `sqlalchemy.sql.expression.UnaryExpression`（`sqlalchemy.sql.expression.ColumnElement`）

```py
method self_group(against=None)
```

对此 `ClauseElement` 应用“分组”。

此方法被子类重写以返回一个“分组”构造，即括号。特别是它被“二元”表达式使用，当放置到更大的表达式中时提供一个围绕自身的分组，以及当放置到另一个 `select()` 构造的 FROM 子句中时，由 `select()` 构造使用。（请注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在表达式中可能不需要括号，例如在表达式 `x OR (y AND z)` 中 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

## 列元素类型化实用程序

从 `sqlalchemy` 命名空间导入的独立实用函数，以提高类型检查器的支持。

| 对象名称 | 描述 |
| --- | --- |
| NotNullable(val) | 将列或 ORM 类型标记为非空。 |
| Nullable(val) | 将列或 ORM 类型标记为可空。 |

```py
function sqlalchemy.NotNullable(val: _TypedColumnClauseArgument[_T | None] | Type[_T] | None) → _TypedColumnClauseArgument[_T]
```

将列或 ORM 类型标记为非空。

这可以在选择和其他上下文中使用，以表达列的值不能为 null，例如由于可为空列上的 where 条件：

```py
stmt = select(NotNullable(A.value)).where(A.value.is_not(None))
```

在运行时，此方法返回未更改的输入。

新版本 2.0.20 中新增。

```py
function sqlalchemy.Nullable(val: _TypedColumnClauseArgument[_T]) → _TypedColumnClauseArgument[_T | None]
```

将列或 ORM 类型标记为可空。

这可以在选择和其他上下文中使用，以表达列的值可以为 null，例如由于外连接：

```py
stmt1 = select(A, Nullable(B)).outerjoin(A.bs)
stmt2 = select(A.data, Nullable(B.data)).outerjoin(A.bs)
```

在运行时，此方法返回未更改的输入。

新版本 2.0.20 中新增。

## 列元素基础构造函数

从 `sqlalchemy` 命名空间导入的独立函数，用于构建 SQLAlchemy 表达语言构造时使用。

| 对象名称 | 描述 |
| --- | --- |
| and_(*clauses) | 生成由 `AND` 连接的表达式的合取。 |
| bindparam(key[, value, type_, unique, ...]) | 生成一个“绑定表达式”。 |
| bitwise_not(expr) | 生成一个一元位非子句，通常通过 `~` 运算符。 |
| case(*whens, [value, else_]) | 生成一个 `CASE` 表达式。 |
| cast(expression, type_) | 生成一个 `CAST` 表达式。 |
| column(text[, type_, is_literal, _selectable]) | 生成一个 `ColumnClause` 对象。 |
| custom_op | 代表一个“自定义”操作符。 |
| distinct(expr) | 生成一个列表达式级的一元 `DISTINCT` 子句。 |
| extract(field, expr) | 返回一个 `Extract` 构造。 |
| false() | 返回一个 `False_` 构造。 |
| func | 生成 SQL 函数表达式。 |
| lambda_stmt(lmb[, enable_tracking, track_closure_variables, track_on, ...]) | 生成一个作为 lambda 缓存的 SQL 语句。 |
| literal(value[, type_, literal_execute]) | 返回一个与绑定参数绑定的文字子句。 |
| literal_column(text[, type_]) | 生成一个具有 `column.is_literal` 标志设置为 True 的 `ColumnClause` 对象。 |
| not_(clause) | 返回给定子句的否定，即 `NOT(clause)`。 |
| null() | 返回一个常量 `Null` 构造。 |
| or_(*clauses) | 生成由 `OR` 连接的表达式的合取。 |
| outparam(key[, type_]) | 为在支持它们的数据库中的函数（存储过程）使用而创建一个“OUT”参数。 |
| quoted_name | 表示与引用偏好结合的 SQL 标识符。 |
| text(text) | 构造一个新的`TextClause`子句，直接表示文本型的 SQL 字符串。 |
| true() | 返回一个常量 `True_` 构造。 |
| try_cast(expression, type_) | 为支持的后端生成一个 `TRY_CAST` 表达式；这是一个返回不可转换为 NULL 的 `CAST`。 |
| tuple_(*clauses, [types]) | 返回一个 `Tuple`。 |
| type_coerce(expression, type_) | 将 SQL 表达式与特定类型关联，而不会渲染 `CAST`。 |

```py
function sqlalchemy.sql.expression.and_(*clauses)
```

生成由 `AND` 连接的表达式的合取。

例如：

```py
from sqlalchemy import and_

stmt = select(users_table).where(
                and_(
                    users_table.c.name == 'wendy',
                    users_table.c.enrolled == True
                )
            )
```

使用 Python `&` 运算符也可以获得 `and_()` 合取（注意，复合表达式需要用括号括起来，以便与 Python 运算符优先级行为一起使用）：

```py
stmt = select(users_table).where(
                (users_table.c.name == 'wendy') &
                (users_table.c.enrolled == True)
            )
```

`and_()` 操作在某些情况下也是隐式的；例如，`Select.where()` 方法可以针对一个语句多次调用，这将导致每个子句使用 `and_()` 进行组合：

```py
stmt = select(users_table).\
        where(users_table.c.name == 'wendy').\
        where(users_table.c.enrolled == True)
```

`and_()` 构造必须至少给定一个位置参数才能有效；没有参数的 `and_()` 构造是模棱两可的。要从给定的表达式列表生成一个“空”或动态生成的 `and_()` 表达式，应指定一个“默认”元素为 `true()`（或只是 `True`）：

```py
from sqlalchemy import true
criteria = and_(true(), *expressions)
```

如果没有其他表达式存在，上述表达式将编译为 SQL 表达式 `true` 或 `1 = 1`，取决于后端。如果存在其他表达式，则 `true()` 值将被忽略，因为它不会影响具有其他元素的 AND 表达式的结果。

自版本 1.4 起已弃用：现在 `and_()` 元素要求至少传递一个参数；创建没有参数的 `and_()` 构造已被弃用，并将发出弃用警告，同时继续生成空白的 SQL 字符串。

另���参阅

`or_()`

```py
function sqlalchemy.sql.expression.bindparam(key: str | None, value: Any = _NoArg.NO_ARG, type_: _TypeEngineArgument[_T] | None = None, unique: bool = False, required: bool | Literal[_NoArg.NO_ARG] = _NoArg.NO_ARG, quote: bool | None = None, callable_: Callable[[], Any] | None = None, expanding: bool = False, isoutparam: bool = False, literal_execute: bool = False) → BindParameter[_T]
```

生成一个“绑定表达式”。

返回值是`BindParameter`的一个实例；这是一个`ColumnElement`子类，代表了 SQL 表达式中的所谓“占位符”值，其值在执行语句针对数据库连接时提供。

在 SQLAlchemy 中，`bindparam()`构造具有在表达式时间最终使用的实际值的能力。通过这种方式，它不仅作为最终填充的“占位符”，还作为表示所谓“不安全”值的一种方式，这些值不应直接呈现在 SQL 语句中，而应作为需要正确转义并可能处理类型安全性的值传递给 DBAPI。

明确使用`bindparam()`时，典型用例通常是传统参数的延迟；`bindparam()`构造接受一个名称，然后可以在执行时引用：

```py
from sqlalchemy import bindparam

stmt = select(users_table).where(
    users_table.c.name == bindparam("username")
)
```

上述语句在渲染时将生成类似以下的 SQL：

```py
SELECT id, name FROM user WHERE name = :username
```

为了填充上述`:username`的值，该值通常会在执行时应用到类似`Connection.execute()`的方法中：

```py
result = connection.execute(stmt, {"username": "wendy"})
```

明确使用`bindparam()`在多次调用的情况下生成 UPDATE 或 DELETE 语句时也很常见，其中语句的 WHERE 条件在每次调用时都会更改，例如：

```py
stmt = (
    users_table.update()
    .where(user_table.c.name == bindparam("username"))
    .values(fullname=bindparam("fullname"))
)

connection.execute(
    stmt,
    [
        {"username": "wendy", "fullname": "Wendy Smith"},
        {"username": "jack", "fullname": "Jack Jones"},
    ],
)
```

SQLAlchemy 的核心表达式系统在隐式意义上广泛使用`bindparam()`。通常，传递给几乎所有 SQL 表达式函数的 Python 字面值都会被强制转换为固定的`bindparam()`构造。例如，给定一个比较操作，如下所示：

```py
expr = users_table.c.name == 'Wendy'
```

上述表达式将产生一个`BinaryExpression`构造，其中左侧是代表`name`列的`Column`对象，右侧是代表字面值的`BindParameter`：

```py
print(repr(expr.right))
BindParameter('%(4327771088 name)s', 'Wendy', type_=String())
```

上述表达式将生成类似以下的 SQL：

```py
user.name = :name_1
```

其中 `:name_1` 参数名是匿名的。实际字符串 `Wendy` 不在生成的字符串中，但在稍后在语句执行中使用时一直保留。如果我们调用如下语句：

```py
stmt = select(users_table).where(users_table.c.name == 'Wendy')
result = connection.execute(stmt)
```

我们将看到 SQL 日志输出为：

```py
SELECT "user".id, "user".name
FROM "user"
WHERE "user".name = %(name_1)s
{'name_1': 'Wendy'}
```

如上所示，`Wendy` 被传递为参数到数据库，而占位符 `:name_1` 在适当形式上呈现给目标数据库，在本例中是 PostgreSQL 数据库。

类似地，在处理 CRUD 语句的“VALUES”部分时，当自动调用 `bindparam()`。`insert()` 结构产生一个 `INSERT` 表达式，在语句执行时，基于传递的参数生成绑定的占位符，如下所示：

```py
stmt = users_table.insert()
result = connection.execute(stmt, {"name": "Wendy"})
```

上述将产生以下 SQL 输出：

```py
INSERT INTO "user" (name) VALUES (%(name)s)
{'name': 'Wendy'}
```

编译/执行时，`Insert` 结构会生成一个 `bindparam()`，镜像了列名 `name`，这是由于我们传递给 `Connection.execute()` 方法的单个 `name` 参数。

参数：

+   `key` –

    此绑定参数的键（例如名称）。将用于使用命名参数的方言生成的 SQL 语句中。如果存在具有相同键的其他 `BindParameter` 对象，或者如果其长度太长并且需要截断，则此值在编译操作的一部分时可能会被修改。

    如果省略，将为绑定参数生成一个“匿名”名称；在给定要绑定的值时，最终结果等同于调用 `literal()` 函数与要绑定的值，特别是如果还提供了 `bindparam.unique` 参数时。

+   `value` – 此绑定参数的初始值。如果在为此特定参数名的语句执行方法中未指示其他值，则将在语句执行时作为传递给 DBAPI 的此参数的值使用。默认为 `None`。

+   `callable_` – 一个可调用函数，代替“value”。该函数将在语句执行时被调用，以确定最终值。用于无法在创建子句构造时确定实际绑定值的情况，但仍希望使用嵌入式绑定值的情况。

+   `type_` –

    表示此 `bindparam()` 的可选数据类型的 `TypeEngine` 类或实例。如果未传递，类型可能会根据给定的值自动确定绑定；例如，trivial Python 类型，如 `str`、`int`、`bool`，可能会导致自动选择 `String`、`Integer` 或 `Boolean` 类型。

    `bindparam()` 的类型非常重要，特别是该类型将在将值传递给数据库之前对值进行预处理。例如，引用 datetime 值的 `bindparam()`，并且指定为持有 `DateTime` 类型，可能会在传递值之前对值进行所需的转换（例如，在 SQLite 上进行字符串化）。

+   `unique` – 如果为 True，则此 `BindParameter` 的键名将被修改，如果已经在包含表达式中找到具有相同名称的另一个 `BindParameter`。此标志通常由内部使用，用于生成所谓的“匿名”绑定表达式，通常不适用于显式命名的 `bindparam()` 结构。

+   `required` – 如果为 `True`，则在执行时需要一个值。如果未传递，则默认为 `True`，如果没有传递 `bindparam.value` 或 `bindparam.callable`，则为 `True`。如果这些参数中的任何一个存在，则 `bindparam.required` 默认为 `False`。

+   `quote` – 如果此参数名需要引号，并且当前不被认为是 SQLAlchemy 的保留字，则为 True；目前仅适用于 Oracle 后端，在那里绑定的名称有时必须用引号括起来。

+   `isoutparam` – 如果为 True，则应该将该参数视为存储过程的“OUT”参数。这适用于支持 OUT 参数的后端，如 Oracle。

+   `expanding` –

    如果为 True，则此参数将在执行时被视为“扩展”参数；参数值应为序列，而不是标量值，并且字符串 SQL 语句将在每次执行时进行转换，以适应具有可变数量参数槽的序列传递给 DBAPI。这是为了允许语句缓存与 IN 子句结合使用。

    另请参阅

    `ColumnOperators.in_()`

    使用 IN 表达式 - 使用烘焙查询

    注意

    “扩展”功能不支持“executemany”样式的参数集。

    在版本 1.2 中新增。

    在版本 1.3 中更改：现在“扩展”边界参数功能支持空列表。

+   `literal_execute` –

    如果为 True，则绑定参数将在编译阶段以特殊的“POSTCOMPILE”标记呈现，并且 SQLAlchemy 编译器将在语句执行时将参数的最终值呈现到 SQL 语句中，省略了参数字典/列表中传递给 DBAPI `cursor.execute()` 的值。这产生了类似于使用 `literal_binds` 编译标志的效果，但是发生在语句发送到 DBAPI `cursor.execute()` 方法时，而不是在语句编译时。此功能的主要用途是为无法在这些上下文中适应绑定参数的数据库驱动程序渲染 LIMIT / OFFSET 子句，同时允许 SQL 构造在编译级别可缓存。

    在版本 1.4 中新增：“编译后”边界参数。

    另请参阅

    Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”边界参数。

另请参阅

发送参数 - 在 SQLAlchemy 统一教程中

```py
function sqlalchemy.sql.expression.bitwise_not(expr: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

产生一个一元位取反子句，通常通过 `~` 运算符。

请勿与布尔取反 `not_()` 混淆。

在版本 2.0.2 中新增。

另请参阅

按位运算符

```py
function sqlalchemy.sql.expression.case(*whens: typing_Tuple[_ColumnExpressionArgument[bool], Any] | Mapping[Any, Any], value: Any | None = None, else_: Any | None = None) → Case[Any]
```

产生一个 `CASE` 表达式。

SQL 中的 `CASE` 构造是一个条件对象，其行为在某种程度上类似于其他语言中的“if/then”构造。它返回 `Case` 的实例。

`case()` 通常形式下被传递一系列“when”构造，即条件和结果作为元组的列表：

```py
from sqlalchemy import case

stmt = select(users_table).\
            where(
                case(
                    (users_table.c.name == 'wendy', 'W'),
                    (users_table.c.name == 'jack', 'J'),
                    else_='E'
                )
            )
```

上述语句将产生类似于：

```py
SELECT id, name FROM user
WHERE CASE
    WHEN (name = :name_1) THEN :param_1
    WHEN (name = :name_2) THEN :param_2
    ELSE :param_3
END
```

当需要针对单个父列的多个值的简单相等表达式时，`case()`还具有通过`case.value`参数使用的“简写”格式，该参数传递一个要比较的列表达式。在这种形式中，通过包含要与键控结果表达式进行比较的表达式的字典传递`case.whens`参数。下面的语句等效于前面的语句：

```py
stmt = select(users_table).\
            where(
                case(
                    {"wendy": "W", "jack": "J"},
                    value=users_table.c.name,
                    else_='E'
                )
            )
```

在`case.whens`中接受的结果值以及在`case.else_`中接受的值都从 Python 文字转换为`bindparam()`构造。也接受 SQL 表达式，例如`ColumnElement`构造。要将文字字符串表达式转换为内联呈现的常量表达式，请使用`literal_column()`构造，如下所示：

```py
from sqlalchemy import case, literal_column

case(
    (
        orderline.c.qty > 100,
        literal_column("'greaterthan100'")
    ),
    (
        orderline.c.qty > 10,
        literal_column("'greaterthan10'")
    ),
    else_=literal_column("'lessthan10'")
)
```

以上将呈现给定的常量，而不使用绑定参数作为结果值（但仍然用于比较值），如下所示：

```py
CASE
    WHEN (orderline.qty > :qty_1) THEN 'greaterthan100'
    WHEN (orderline.qty > :qty_2) THEN 'greaterthan10'
    ELSE 'lessthan10'
END
```

参数：

+   `*whens` –

    要进行比较的标准，`case.whens`接受两种不同形式，取决于是否使用`case.value`。

    从版本 1.4 开始更改：`case()`函数现在按位置接受 WHEN 条件的系列

    在第一种形式中，它接受多个作为位置参数传递的 2 元组；每个 2 元组由`(<sql 表达式>, <值>)`组成，其中 SQL 表达式是布尔表达式，“值”是一个结果值，例如：

    ```py
    case(
        (users_table.c.name == 'wendy', 'W'),
        (users_table.c.name == 'jack', 'J')
    )
    ```

    在第二种形式中，它接受一个 Python 字典，将比较值映射到一个结果值；这种形式需要`case.value`存在，并且值将使用`==`运算符进行比较，例如：

    ```py
    case(
        {"wendy": "W", "jack": "J"},
        value=users_table.c.name
    )
    ```

+   `value` – 一个可选的 SQL 表达式，将用作传递给`case.whens`的字典中的候选值的固定“比较点”。

+   `else_` – 如果`case.whens`中的所有表达式求值结果都为 false，则将是`CASE`构造中的可选 SQL 表达式的评估结果。如果省略，则大多数数据库将在“when”表达式没有一个求值结果为 true 时产生一个 NULL 的结果。

```py
function sqlalchemy.sql.expression.cast(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → Cast[_T]
```

生成一个`CAST`表达式。

`cast()`返回一个`Cast`的实例。

例如：

```py
from sqlalchemy import cast, Numeric

stmt = select(cast(product_table.c.unit_price, Numeric(10, 4)))
```

上述语句将生成类似于：

```py
SELECT CAST(unit_price AS NUMERIC(10, 4)) FROM product
```

当使用时，`cast()`函数执行两个不同的功能。首先，它在生成的 SQL 字符串中呈现`CAST`表达式。其次，它将给定类型（例如`TypeEngine`类或实例）与 Python 端的列表达式关联，这意味着表达式将具有与该类型关联的表达式运算符行为，以及该类型的绑定值处理和结果行处理行为。

一个替代`cast()`的函数是`type_coerce()`。此函数执行关联表达式与特定类型的第二个任务，但不会在 SQL 中渲染`CAST`表达式。

参数：

+   `expression` – 一个 SQL 表达式，例如`ColumnElement`表达式或将被强制转换为绑定字面值的 Python 字符串。

+   `type_` – 一个`TypeEngine`类或实例，指示`CAST`应用的类型。

另请参阅

数据转换和类型强制转换

`try_cast()` - 一个替代`CAST`的函数，当转换失败时会产生 NULL，而不是引发错误。只有一些方言支持。

`type_coerce()` - 一个替代`CAST`的函数，仅在 Python 端强制转换类型，通常足以生成正确的 SQL 和数据强制转换。

```py
function sqlalchemy.sql.expression.column(text: str, type_: _TypeEngineArgument[_T] | None = None, is_literal: bool = False, _selectable: FromClause | None = None) → ColumnClause[_T]
```

生成一个`ColumnClause`对象。

`ColumnClause`是`Column`类的轻量级类比。可以仅使用名称调用`column()`函数，如下所示：

```py
from sqlalchemy import column

id, name = column("id"), column("name")
stmt = select(id, name).select_from("user")
```

上述语句将产生如下 SQL：

```py
SELECT id, name FROM user
```

一旦构造完成，`column()` 可以像其他 SQL 表达式元素一样在 `select()` 构造中使用：

```py
from sqlalchemy.sql import column

id, name = column("id"), column("name")
stmt = select(id, name).select_from("user")
```

`column()` 处理的文本被假定为像处理数据库列名一样；如果字符串包含混合大小写、特殊字符或与目标后端的已知保留字匹配，列表达式将使用后端确定的引用行为呈现。要生成一个完全不带引用的文本 SQL 表达式，请使用 `literal_column()` ，或者将 `column.is_literal` 的值传递为 `True`。此外，最好使用 `text()` 构造来处理完整的 SQL 语句。

`column()` 可以通过与 `table()` 函数（它是 `Table` 的轻量级类比）结合使用，以生成具有最小样板的工作表构造：

```py
from sqlalchemy import table, column, select

user = table("user",
        column("id"),
        column("name"),
        column("description"),
)

stmt = select(user.c.description).where(user.c.name == 'wendy')
```

像上面示例的 `column()` / `table()` 构造可以以临时方式创建，并且不与任何 `MetaData`、DDL 或事件关联，不像它的 `Table` 对应物。

参数：

+   `text` – 元素的文本。

+   `type` – `TypeEngine` 对象，它可以将此 `ColumnClause` 与一个类型关联。

+   `is_literal` – 如果为 True，则假定 `ColumnClause` 是一个精确的表达式，无论大小写设置如何，都将以不应用引用规则的方式传递到输出中。 `literal_column()` 函数本质上调用 `column()` ，同时传递 `is_literal=True`。

另请参阅

`Column`

`literal_column()`

`table()`

`text()`

使用文本列表达式进行选择

```py
class sqlalchemy.sql.expression.custom_op
```

表示一个“自定义”操作符。

当使用 `Operators.op()` 或 `Operators.bool_op()` 方法创建自定义操作符可调用时，通常会实例化 `custom_op`。该类也可在以编程方式构建表达式时直接使用。例如，表示“阶乘”操作：

```py
from sqlalchemy.sql import UnaryExpression
from sqlalchemy.sql import operators
from sqlalchemy import Numeric

unary = UnaryExpression(table.c.somecolumn,
        modifier=operators.custom_op("!"),
        type_=Numeric)
```

另请参阅

`Operators.op()`

`Operators.bool_op()`

**类签名**

类 `sqlalchemy.sql.expression.custom_op` (`sqlalchemy.sql.expression.OperatorType`, `typing.Generic`)

```py
function sqlalchemy.sql.expression.distinct(expr: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

生成一个基于列表达式的一元 `DISTINCT` 子句。

这将向 **单个列表达式** 应用 `DISTINCT` 关键字（例如，不是整个语句），并且**具体在该列位置上**呈现；这用于在聚合函数中的包含，例如：

```py
from sqlalchemy import distinct, func
stmt = select(users_table.c.id, func.count(distinct(users_table.c.name)))
```

上述将产生类似于以下语句：

```py
SELECT user.id, count(DISTINCT user.name) FROM user
```

提示

`distinct()` 函数 **不会** 将 DISTINCT 应用于完整的 SELECT 语句，而是将 DISTINCT 修饰符应用于 **单个列表达式**。对于一般的 `SELECT DISTINCT` 支持，请在 `Select.distinct()` 上使用方法 `Select`。

`distinct()` 函数也可以作为列级方法使用，例如 `ColumnElement.distinct()`，如下所示：

```py
stmt = select(func.count(users_table.c.name.distinct()))
```

`distinct()` 操作符与 `Select.distinct()` 方法不同，后者会将 `DISTINCT` 应用于整个结果集，例如 `SELECT DISTINCT` 表达式。有关该方法的更多信息，请参见该方法。

请参见

`ColumnElement.distinct()`

`Select.distinct()`

`func`

```py
function sqlalchemy.sql.expression.extract(field: str, expr: _ColumnExpressionArgument[Any]) → Extract
```

返回一个 `Extract` 构造。

这通常也可以通过 `extract()` 或 `func.extract` 从 `func` 命名空间中获取。

参数：

+   `field` – 要提取的字段。

+   `expr` – 作为 `EXTRACT` 表达式右侧的列或 Python 标量表达式。

例如：

```py
from sqlalchemy import extract
from sqlalchemy import table, column

logged_table = table("user",
        column("id"),
        column("date_created"),
)

stmt = select(logged_table.c.id).where(
    extract("YEAR", logged_table.c.date_created) == 2021
)
```

在上面的示例中，该语句用于从数据库中选择 `YEAR` 组件与特定值匹配的 ids。

类似地，也可以选择提取的组件：

```py
stmt = select(
    extract("YEAR", logged_table.c.date_created)
).where(logged_table.c.id == 1)
```

`EXTRACT` 的实现在不同的数据库后端可能会有所不同。用户被提醒要查阅其数据库文档。

```py
function sqlalchemy.sql.expression.false() → False_
```

返回一个 `False_` 构造。

例如：

```py
>>> from sqlalchemy import false
>>> print(select(t.c.x).where(false()))
SELECT  x  FROM  t  WHERE  false 
```

一个不支持真/假常量的后端将以对 1 或 0 的表达式形式呈现：

```py
>>> print(select(t.c.x).where(false()))
SELECT  x  FROM  t  WHERE  0  =  1 
```

`true()` 和 `false()` 常量还在 `and_()` 或 `or_()` 连接中具有“短路”操作：

```py
>>> print(select(t.c.x).where(or_(t.c.x > 5, true())))
SELECT  x  FROM  t  WHERE  true
>>> print(select(t.c.x).where(and_(t.c.x > 5, false())))
SELECT  x  FROM  t  WHERE  false 
```

请参见

`true()`

```py
sqlalchemy.sql.expression.func = <sqlalchemy.sql.functions._FunctionGenerator object>
```

生成 SQL 函数表达式。

`func` 是一个特殊的对象实例，它基于基于名称的属性生成 SQL 函数，例如：

```py
>>> print(func.count(1))
count(:param_1) 
```

返回的对象是 `Function` 的一个实例，与任何其他列导向的 SQL 元素一样，并以那种方式使用：

```py
>>> print(select(func.count(table.c.id)))
SELECT  count(sometable.id)  FROM  sometable 
```

可以给`func`任何名称。如果 SQLAlchemy 不知道函数名称，则将按原样呈现。对于 SQLAlchemy 知道的常见 SQL 函数，该名称可能被解释为*通用函数*，将被适当地编译到目标数据库：

```py
>>> print(func.current_timestamp())
CURRENT_TIMESTAMP 
```

要调用位于点分隔包中的函数，请以相同的方式指定它们：

```py
>>> print(func.stats.yield_curve(5, 10))
stats.yield_curve(:yield_curve_1,  :yield_curve_2) 
```

SQLAlchemy 可以意识到函数的返回类型，以启用特定类型的词法和基于结果的行为。例如，要确保基于字符串的函数返回 Unicode 值，并在表达式中类似地对待为字符串，请指定`Unicode`作为类型：

```py
>>> print(func.my_string(u'hi', type_=Unicode) + ' ' +
...       func.my_string(u'there', type_=Unicode))
my_string(:my_string_1)  ||  :my_string_2  ||  my_string(:my_string_3) 
```

由`func`调用返回的对象通常是`Function`的实例。此对象符合“列”接口，包括比较和标记函数。该对象还可以传递给`Connection`或`Engine`的`Connectable.execute()`方法，在那里它首先将被包装在 SELECT 语句中：

```py
print(connection.execute(func.current_timestamp()).scalar())
```

在一些特殊情况下，`func`访问器将将名称重定向到内置表达式，例如`cast()`或`extract()`，因为这些名称具有众所周知的含义，但从 SQLAlchemy 的角度来看并不完全相同于“函数”。

被解释为“通用”函数的函数知道如何自动计算其返回类型。有关已知通用函数的列表，请参见 SQL 和通用函数。

注意

`func`构造仅对调用独立的“存储过程”提供有限支持，特别是那些具有特殊参数化问题的存储过程。

有关如何使用 DBAPI 级别的`callproc()`方法完全传统存储过程的详细信息，请参见调用存储过程和用户定义函数部分。

另请参见

使用 SQL 函数 - 在 SQLAlchemy 统一教程中

`Function`

```py
function sqlalchemy.sql.expression.lambda_stmt(lmb: Callable[[], Any], enable_tracking: bool = True, track_closure_variables: bool = True, track_on: object | None = None, global_track_bound_values: bool = True, track_bound_values: bool = True, lambda_cache: MutableMapping[Tuple[Any, ...], NonAnalyzedFunction | AnalyzedFunction] | None = None) → StatementLambdaElement
```

生成一个作为 lambda 缓存的 SQL 语句。

lambda 内部的 Python 代码对象将被扫描，其中包括将成为绑定参数的 Python 字面值，以及引用可能变化的 Core 或 ORM 构造的闭包变量。lambda 本身将仅在检测到特定构造集的情况下调用一次。

例如：

```py
from sqlalchemy import lambda_stmt

stmt = lambda_stmt(lambda: table.select())
stmt += lambda s: s.where(table.c.id == 5)

result = connection.execute(stmt)
```

返回的对象是 `StatementLambdaElement` 的实例。

新版本中的新增内容 1.4。

参数：

+   `lmb` – 一个 Python 函数，通常是 lambda，它不带参数并返回一个 SQL 表达式构造

+   `enable_tracking` – 当为 False 时，将禁用对给定 lambda 进行闭包变量或绑定参数更改的所有扫描。用于在所有情况下产生相同结果且不进行参数化的 lambda。

+   `track_closure_variables` – 当为 False 时，将不会扫描 lambda 内部闭包变量的更改。用于一个 lambda，其闭包变量的状态永远不会改变 lambda 返回的 SQL 结构。

+   `track_bound_values` – 当为 False 时，将禁用对给定 lambda 的绑定参数跟踪。用于要么不产生任何绑定值的 lambda，要么初始绑定值永远不会更改的 lambda。

+   `global_track_bound_values` – 当为 False 时，将禁用整个语句的参数跟踪，包括通过 `StatementLambdaElement.add_criteria()` 方法添加的额外链接。

+   `lambda_cache` – 一个字典或其他类似映射的对象，其中将存储关于 lambda 的 Python 代码以及 lambda 本身中跟踪的闭包变量的信息。默认为全局 LRU 缓存。此缓存独立于 `Connection` 对象使用的“compiled_cache”。

另请参见

使用 Lambdas 加快语句生成速度

```py
function sqlalchemy.sql.expression.literal(value: Any, type_: _TypeEngineArgument[Any] | None = None, literal_execute: bool = False) → BindParameter[Any]
```

返回一个字面值子句，绑定到一个绑定参数。

当非 `ClauseElement` 对象（如字符串、整数、日期等）与 `ColumnElement` 子类进行比较操作时，将自动创建字面值子句，例如 `Column` 对象。使用此函数强制生成字面值子句，将其创建为具有绑定值的 `BindParameter`。

参数：

+   `value` – 要绑定的值。可以是底层 DB-API 支持的任何 Python 对象，或者可以通过给定类型参数进行转换。

+   `type_` – 一个可选的 `TypeEngine`，将为此文字提供绑定参数转换。

+   `literal_execute` –

    可选的布尔值，当为 True 时，SQL 引擎将尝试在执行时直接将绑定值呈现在 SQL 语句中，而不是作为参数值提供。

    2.0 版本中的新功能。

```py
function sqlalchemy.sql.expression.literal_column(text: str, type_: _TypeEngineArgument[_T] | None = None) → ColumnClause[_T]
```

生成一个具有`column.is_literal`标志设置为 True 的 `ColumnClause` 对象。

`literal_column()`类似于`column()`，只是更常用作“独立”的列表达式，以确切的方式呈现；而`column()`存储一个字符串名称，将被假定为表的一部分，并可能被引用为这样，`literal_column()`可以是那样，或者任何其他任意的面向列的表达式。

参数：

+   `text` – 表达式的文本；可以是任何 SQL 表达式。不会应用引用规则。要指定应该受引用规则约束的列名表达式，请使用 `column()` 函数。

+   `type_` – 一个可选的 `TypeEngine` 对象，它将为此列提供结果集转换和其他表达式语义。如果留空，类型将是 `NullType`。

另请参阅

`column()`

`text()`

使用文本列表达式进行选择

```py
function sqlalchemy.sql.expression.not_(clause: _ColumnExpressionArgument[_T]) → ColumnElement[_T]
```

返回给定子句的否定，即`NOT(clause)`。

`~`运算符也对所有`ColumnElement`子类进行了重载，以产生相同的结果。

```py
function sqlalchemy.sql.expression.null() → Null
```

返回一个常量`Null`构造。

```py
function sqlalchemy.sql.expression.or_(*clauses)
```

生成由`OR`连接的表达式的合取。

例如：

```py
from sqlalchemy import or_

stmt = select(users_table).where(
                or_(
                    users_table.c.name == 'wendy',
                    users_table.c.name == 'jack'
                )
            )
```

逻辑或`or_()`也可使用 Python 的`|`运算符（尽管请注意，为了与 Python 运算符优先级行为相匹配，复合表达式需要括号化）：

```py
stmt = select(users_table).where(
                (users_table.c.name == 'wendy') |
                (users_table.c.name == 'jack')
            )
```

`or_()`构造必须至少给出一个位置参数才能有效；没有参数的`or_()`构造是含糊的。 为了生成一个“空”或动态生成的`or_()`表达式，从给定的表达式列表中，应指定一个`false()`（或仅`False`）的“默认”元素：

```py
from sqlalchemy import false
or_criteria = or_(false(), *expressions)
```

如果没有其他表达式存在，上述表达式将编译为 SQL 作为表达式`false`或`0 = 1`，具体取决于后端。 如果存在表达式，则`false()`值将被忽略，因为它不影响具有其他元素的 OR 表达式的结果。

从版本 1.4 开始弃用：`or_()`元素现在要求至少传递一个参数； 创建没有参数的`or_()`构造已经被弃用，并且将发出弃用警告，同时继续生成空的 SQL 字符串。

另请参阅

`and_()`

```py
function sqlalchemy.sql.expression.outparam(key: str, type_: TypeEngine[_T] | None = None) → BindParameter[_T]
```

为在支持它们的数据库中使用的函数（存储过程）创建一个“OUT”参数。

`outparam`可以像普通函数参数一样使用。 “输出”值将通过其`out_parameters`属性从`CursorResult`对象中获得，该属性返回一个包含值的字典。

```py
function sqlalchemy.sql.expression.text(text: str) → TextClause
```

构造一个新的`TextClause`子句，表示直接的文本 SQL 字符串。

例如：

```py
from sqlalchemy import text

t = text("SELECT * FROM users")
result = connection.execute(t)
```

`text()`相比于纯字符串提供的优势是，它提供了对绑定参数的后端中立支持，每个语句的执行选项，以及绑定参数和结果列类型化行为，允许在字面上指定的语句执行时使用 SQLAlchemy 类型构造。 该构造还可以提供一个`.c`列元素的集合，允许它作为子查询嵌入到其他 SQL 表达式构造中。

绑定参数通过名称指定，使用格式`:name`。 例如：

```py
t = text("SELECT * FROM users WHERE id=:user_id")
result = connection.execute(t, {"user_id": 12})
```

对于需要直接输入冒号的 SQL 语句，例如内联字符串中，请使用反斜杠进行转义：

```py
t = text(r"SELECT * FROM users WHERE name='\:username'")
```

`TextClause` 结构包括可以提供有关绑定参数的信息以及假定它是可执行 SELECT 类型语句时将从文本语句返回的列值的方法。使用 `TextClause.bindparams()` 方法来提供绑定参数的详细信息，而 `TextClause.columns()` 方法允许指定返回列，包括名称和类型：

```py
t = text("SELECT * FROM users WHERE id=:user_id").\
        bindparams(user_id=7).\
        columns(id=Integer, name=String)

for id, name in connection.execute(t):
    print(id, name)
```

当在较大查询的一部分，如 SELECT 语句的 WHERE 子句中指定了文字字符串 SQL 片段时，使用 `text()` 结构：

```py
s = select(users.c.id, users.c.name).where(text("id=:user_id"))
result = connection.execute(s, {"user_id": 12})
```

`text()` 也可用于使用纯文本构建完整的、独立的语句。因此，SQLAlchemy 将其称为一个 `Executable` 对象，并可以像传递给 `.execute()` 方法的任何其他语句一样使用。

参数：

**text** –

要创建的 SQL 语句的文本。使用 `:<param>` 来指定绑定参数；它们将编译为其引擎特定的格式。

警告

`text.text` 参数可以作为 Python 字符串参数传递，它将被视为**受信任的 SQL 文本**并按照给定的方式呈现。**不要传递不受信任的输入给此参数**。

请参见

使用文本列表达式进行选择

```py
function sqlalchemy.sql.expression.true() → True_
```

返回一个常量`True_`构造。

例如：

```py
>>> from sqlalchemy import true
>>> print(select(t.c.x).where(true()))
SELECT  x  FROM  t  WHERE  true 
```

一个不支持 true/false 常量的后端将呈现为针对 1 或 0 的表达式：

```py
>>> print(select(t.c.x).where(true()))
SELECT  x  FROM  t  WHERE  1  =  1 
```

`true()` 和 `false()` 常量还在 `and_()` 或 `or_()` 连接中具有“短路”操作：

```py
>>> print(select(t.c.x).where(or_(t.c.x > 5, true())))
SELECT  x  FROM  t  WHERE  true
>>> print(select(t.c.x).where(and_(t.c.x > 5, false())))
SELECT  x  FROM  t  WHERE  false 
```

请参见

`false()`

```py
function sqlalchemy.sql.expression.try_cast(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → TryCast[_T]
```

为支持的后端生成一个 `TRY_CAST` 表达式；这是一个返回不可转换为 NULL 的 `CAST`。

在 SQLAlchemy 中，这个结构**仅**被 SQL Server 方言支持，如果在其他包含的后端上使用，将会引发`CompileError`。但是，第三方后端也可能支持这个结构。

提示

由于`try_cast()`源自 SQL Server 方言，因此可以从`sqlalchemy.`以及`sqlalchemy.dialects.mssql`中导入。

`try_cast()`返回一个`TryCast`的实例，并且通常类似于`Cast`结构；在 SQL 级别，`CAST`和`TRY_CAST`之间的区别是`TRY_CAST`对于无法转换的表达式返回 NULL，例如尝试将字符串`"hi"`转换为整数值。

例如：

```py
from sqlalchemy import select, try_cast, Numeric

stmt = select(
    try_cast(product_table.c.unit_price, Numeric(10, 4))
)
```

在 Microsoft SQL Server 上，上述内容将呈现为：

```py
SELECT TRY_CAST (product_table.unit_price AS NUMERIC(10, 4))
FROM product_table
```

在版本 2.0.14 中新增：`try_cast()`已经从 SQL Server 方言泛化为一个通用的构造，可能会被其他方言支持。

```py
function sqlalchemy.sql.expression.tuple_(*clauses: _ColumnExpressionArgument[Any], types: Sequence[_TypeEngineArgument[Any]] | None = None) → Tuple
```

返回一个`Tuple`。

主要用途是使用`ColumnOperators.in_()`生成一个复合 IN 结构

```py
from sqlalchemy import tuple_

tuple_(table.c.col1, table.c.col2).in_(
    [(1, 2), (5, 12), (10, 19)]
)
```

在版本 1.3.6 中更改：增加了对 SQLite 中 IN 元组的支持。

警告

复合 IN 结构并不被所有后端支持，目前已知可以在 PostgreSQL、MySQL 和 SQLite 上工作。当调用这样的表达式时，不支持的后端将引发`DBAPIError`的子类。

```py
function sqlalchemy.sql.expression.type_coerce(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → TypeCoerce[_T]
```

将 SQL 表达式与特定类型关联，而不渲染`CAST`。

例如：

```py
from sqlalchemy import type_coerce

stmt = select(type_coerce(log_table.date_string, StringDateTime()))
```

上述结构将生成一个`TypeCoerce`对象，在 SQL 端不会以任何方式修改渲染，可能的例外是在列子句上下文中使用时生成的标签：

```py
SELECT  date_string  AS  date_string  FROM  log
```

当获取结果行时，`StringDateTime`类型处理器将代表`date_string`列应用于结果行。

注意

`type_coerce()`结构不会渲染任何自己的 SQL 语法，包括不意味着括号化。如果需要显式括号化，请使用`TypeCoerce.self_group()`。

为了为表达式提供一个命名标签，使用 `ColumnElement.label()`:

```py
stmt = select(
    type_coerce(log_table.date_string, StringDateTime()).label('date')
)
```

一个具有绑定值处理功能的类型在将字面值或 `bindparam()` 构造传递给 `type_coerce()` 作为目标时也会生效。例如，如果一个类型实现了 `TypeEngine.bind_expression()` 方法或 `TypeEngine.bind_processor()` 方法或等效方法，当传递字面值时，这些函数将在语句编译/执行时生效，如下所示：

```py
# bound-value handling of MyStringType will be applied to the
# literal value "some string"
stmt = select(type_coerce("some string", MyStringType))
```

当在组合表达式中使用 `type_coerce()` 时，请注意**不会应用括号**。如果 `type_coerce()` 被用在一个操作符上下文中，通常来自 CAST 的括号是必要的，那么可以使用 `TypeCoerce.self_group()` 方法：

```py
>>> some_integer = column("someint", Integer)
>>> some_string = column("somestr", String)
>>> expr = type_coerce(some_integer + 5, String) + some_string
>>> print(expr)
someint  +  :someint_1  ||  somestr
>>> expr = type_coerce(some_integer + 5, String).self_group() + some_string
>>> print(expr)
(someint  +  :someint_1)  ||  somestr 
```

参数:

+   `expression` – 一个 SQL 表达式，比如一个 `ColumnElement` 表达式或一个将被强制转换为绑定字面值的 Python 字符串。

+   `type_` – 一个指示表达式被强制转换为的类型的 `TypeEngine` 类或实例。

另请参阅

数据转换和类型转换

`cast()`

```py
class sqlalchemy.sql.expression.quoted_name
```

表示一个与引号偏好结合的 SQL 标识符。

`quoted_name` 是一个 Python unicode/str 子类，表示特定的标识符名称以及一个 `quote` 标志。当 `quote` 标志设置为 `True` 或 `False` 时，将覆盖此标识符的自动引用行为，以便无条件引用或不引用名称。如果保持默认值 `None`，则引用行为将根据标记本身的检查在每个后端基础上应用到标识符上。

具有`quote=True`的`quoted_name`对象也在所谓的“名称规范化”选项的情况下被阻止修改。某些数据库后端，如 Oracle、Firebird 和 DB2，将大小写不敏感的名称“规范化”为大写。这些后端的 SQLAlchemy 方言将从 SQLAlchemy 的小写表示不敏感约定转换为这些后端的大写表示不敏感约定。这里的`quote=True`标志将阻止此转换发生，以支持针对此类后端作为全小写引用的标识符。

当为键模式构造指定名称时，`quoted_name`对象通常会自动创建，例如`Table`、`Column`等的构造。该类还可以显式传递为任何接收可引用名称的函数的名称。例如，使用`Engine.has_table()`方法时使用无条件引用名称：

```py
from sqlalchemy import create_engine
from sqlalchemy import inspect
from sqlalchemy.sql import quoted_name

engine = create_engine("oracle+cx_oracle://some_dsn")
print(inspect(engine).has_table(quoted_name("some_table", True)))
```

上述逻辑��针对 Oracle 后端运行“has table”逻辑，将名称传递为`"some_table"`而不转换为大写。

从版本 1.2 开始更改：`quoted_name`构造现在可以从`sqlalchemy.sql`导入，而不仅仅是以前的位置`sqlalchemy.sql.elements`。

**成员**

quote

**类签名**

类`sqlalchemy.sql.expression.quoted_name` (`sqlalchemy.util.langhelpers.MemoizedSlots`, `builtins.str`)

```py
attribute quote
```

字符串是否应该被无条件引用

## 列元素修饰符构造函数

此处列出的函数通常作为任何`ColumnElement`构造的方法更常见，例如，`label()`函数通常通过`ColumnElement.label()`方法调用。

| 对象名称 | 描述 |
| --- | --- |
| all_(expr) | 生成一个 ALL 表达式。 |
| any_(expr) | 生成一个 ANY 表达式。 |
| asc(column) | 生成升序`ORDER BY`子句元素。 |
| between(expr, lower_bound, upper_bound[, symmetric]) | 生成一个`BETWEEN`谓词子句。 |
| collate(expression, collation) | 返回子句`expression COLLATE collation`。 |
| desc(column) | 生成一个降序 `ORDER BY` 子句元素。 |
| funcfilter(func, *criterion) | 为函数生成一个 `FunctionFilter` 对象。 |
| label(name, element[, type_]) | 返回给定 `ColumnElement` 的 `Label` 对象。 |
| nulls_first(column) | 为 `ORDER BY` 表达式生成 `NULLS FIRST` 修饰符。 |
| nulls_last(column) | 为 `ORDER BY` 表达式生成 `NULLS LAST` 修饰符。 |
| nullsfirst | 同义词，用于 `nulls_first()` 函数。 |
| nullslast | `nulls_last()` 函数的传统同义词。 |
| over(element[, partition_by, order_by, range_, ...]) | 为函数生成一个 `Over` 对象。 |
| within_group(element, *order_by) | 为函数生成一个 `WithinGroup` 对象。 |

```py
function sqlalchemy.sql.expression.all_(expr: _ColumnExpressionArgument[_T]) → CollectionAggregate[bool]
```

生成一个 ALL 表达式。

对于诸如 PostgreSQL 的方言，该运算符适用于 `ARRAY` 数据类型的使用，对于 MySQL 的方言，它可能适用于子查询。例如：

```py
# renders on PostgreSQL:
# '5 = ALL (somearray)'
expr = 5 == all_(mytable.c.somearray)

# renders on MySQL:
# '5 = ALL (SELECT value FROM table)'
expr = 5 == all_(select(table.c.value))
```

与 NULL 的比较可能使用 `None` 进行：

```py
None == all_(mytable.c.somearray)
```

使用 any_() / all_() 运算符还具有特殊的“操作数翻转”行为，以便如果 any_() / all_() 用于比较操作的左侧，使用独立运算符（例如 `==`，`!=` 等）（不包括操作符方法，如 `ColumnOperators.is_()`），则渲染的表达式将被翻转：

```py
# would render '5 = ALL (column)`
all_(mytable.c.column) == 5
```

或使用 `None`，请注意，这不会执行通常情况下针对 NULL 渲染 “IS” 的常规步骤：

```py
# would render 'NULL = ALL(somearray)'
all_(mytable.c.somearray) == None
```

版本 1.4.26 中的更改：修复了 any_() / all_() 与右侧 NULL 比较时翻转到左侧的使用。

列级别的 `ColumnElement.all_()` 方法（不要与 `ARRAY` 级别的 `Comparator.all()` 混淆）是 `all_(col)` 的速记形式：

```py
5 == mytable.c.somearray.all_()
```

另请参阅

`ColumnOperators.all_()`

`any_()`

```py
function sqlalchemy.sql.expression.any_(expr: _ColumnExpressionArgument[_T]) → CollectionAggregate[bool]
```

生成一个 ANY 表达式。

对于诸如 PostgreSQL 的方言，此运算符适用于 `ARRAY` 数据类型的使用，对于 MySQL，它可能适用于子查询。例如：

```py
# renders on PostgreSQL:
# '5 = ANY (somearray)'
expr = 5 == any_(mytable.c.somearray)

# renders on MySQL:
# '5 = ANY (SELECT value FROM table)'
expr = 5 == any_(select(table.c.value))
```

使用 `None` 或 `null()` 可能会与 NULL 进行比较：

```py
None == any_(mytable.c.somearray)
```

`any_()` / `all_()` 运算符还具有特殊的“操作数翻转”行为，即如果 `any_()` / `all_()` 用于比较的左侧使用独立运算符（如 `==`、`!=` 等）（不包括操作符方法，如 `ColumnOperators.is_()`），则渲染的表达式会被翻转：

```py
# would render '5 = ANY (column)`
any_(mytable.c.column) == 5
```

或者使用 `None`，请注意，这不会像通常情况下对 NULL 渲染“IS”那样执行：

```py
# would render 'NULL = ANY(somearray)'
any_(mytable.c.somearray) == None
```

从版本 1.4.26 中更改：修复了将 any_() / all_() 与 NULL 比较在右侧时翻转到左侧的问题。

列级别的 `ColumnElement.any_()` 方法（不要与 `ARRAY` 级别的 `Comparator.any()` 混淆）是 `any_(col)` 的简写：

```py
5 = mytable.c.somearray.any_()
```

另请参阅

`ColumnOperators.any_()`

`all_()`

```py
function sqlalchemy.sql.expression.asc(column: _ColumnExpressionOrStrLabelArgument[_T]) → UnaryExpression[_T]
```

生成一个升序的 `ORDER BY` 子句元素。

例如：

```py
from sqlalchemy import asc
stmt = select(users_table).order_by(asc(users_table.c.name))
```

将生成以下 SQL：

```py
SELECT id, name FROM user ORDER BY name ASC
```

`asc()` 函数是所有 SQL 表达式上都可用的 `ColumnElement.asc()` 方法的独立版本，例如：

```py
stmt = select(users_table).order_by(users_table.c.name.asc())
```

参数：

**column** – 一个 `ColumnElement`（例如，标量 SQL 表达式），用于应用 `asc()` 操作。

另请参阅

`desc()`

`nulls_first()`

`nulls_last()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.between(expr: _ColumnExpressionOrLiteralArgument[_T], lower_bound: Any, upper_bound: Any, symmetric: bool = False) → BinaryExpression[bool]
```

生成一个 `BETWEEN` 谓词子句。

例如：

```py
from sqlalchemy import between
stmt = select(users_table).where(between(users_table.c.id, 5, 7))
```

将生成类似于以下 SQL：

```py
SELECT id, name FROM user WHERE id BETWEEN :id_1 AND :id_2
```

`between()`函数是所有 SQL 表达式上都可用的`ColumnElement.between()`方法的独立版本，例如：

```py
stmt = select(users_table).where(users_table.c.id.between(5, 7))
```

传递给`between()`的所有参数（包括左侧列表达式）如果该值不是`ColumnElement`子类，则将从 Python 标量值强制转换。例如，可以比较三个固定值，如下所示：

```py
print(between(5, 3, 7))
```

这将产生：

```py
:param_1 BETWEEN :param_2 AND :param_3
```

参数：

+   `expr` – 列表达式，通常是一个`ColumnElement`实例，或者是要强制转换为列表达式的 Python 标量表达式，用作`BETWEEN`表达式的左侧。

+   `lower_bound` – 作为`BETWEEN`表达式右侧下限的列或 Python 标量表达式。

+   `upper_bound` – 作为`BETWEEN`表达式右侧上限的列或 Python 标量表达式。

+   `symmetric` – 如果为 True，则渲染“ BETWEEN SYMMETRIC ”。请注意，并非所有数据库都支持此语法。

另请参阅

`ColumnElement.between()`

```py
function sqlalchemy.sql.expression.collate(expression: _ColumnExpressionArgument[str], collation: str) → BinaryExpression[str]
```

返回子句`expression COLLATE collation`。

例如：

```py
collate(mycolumn, 'utf8_bin')
```

产生：

```py
mycolumn COLLATE utf8_bin
```

如果它是区分大小写的标识符，例如包含大写字符，则也会引用排序表达式。

在版本 1.2 中更改：如果它们是区分大小写的，则对 COLLATE 表达式自动应用引用。

```py
function sqlalchemy.sql.expression.desc(column: _ColumnExpressionOrStrLabelArgument[_T]) → UnaryExpression[_T]
```

生成一个降序的`ORDER BY`子句元素。

例如：

```py
from sqlalchemy import desc

stmt = select(users_table).order_by(desc(users_table.c.name))
```

将产生 SQL 如下：

```py
SELECT id, name FROM user ORDER BY name DESC
```

`desc()`函数是所有 SQL 表达式上可用的`ColumnElement.desc()`方法的独立版本，例如：

```py
stmt = select(users_table).order_by(users_table.c.name.desc())
```

参数：

**column** – 一个`ColumnElement`（例如标量 SQL 表达式），用于应用`desc()`操作。

另请参阅

`asc()`

`nulls_first()`

`nulls_last()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.funcfilter(func: FunctionElement[_T], *criterion: _ColumnExpressionArgument[bool]) → FunctionFilter[_T]
```

对函数生成一个`FunctionFilter`对象。

用于支持“FILTER”子句的聚合和窗口函数。

例如：

```py
from sqlalchemy import funcfilter
funcfilter(func.count(1), MyClass.name == 'some name')
```

会产生“COUNT(1) FILTER (WHERE myclass.name = ‘some name’)”。

这个函数也可以通过`func`构造本身通过`FunctionElement.filter()`方法获得。

另请参阅

特殊修饰符 WITHIN GROUP, FILTER - 在 SQLAlchemy 统一教程中

`FunctionElement.filter()`

```py
function sqlalchemy.sql.expression.label(name: str, element: _ColumnExpressionArgument[_T], type_: _TypeEngineArgument[_T] | None = None) → Label[_T]
```

为给定的`ColumnElement`对象返回一个`Label`对象。

标签通过`AS` SQL 关键字通常修改`SELECT`语句中列子句中元素的名称。

此功能更方便地通过`ColumnElement.label()`方法在`ColumnElement`上使用。

参数：

+   `name` – 标签名

+   `obj` – 一个`ColumnElement`。

```py
function sqlalchemy.sql.expression.nulls_first(column: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

为`ORDER BY`表达式生成`NULLS FIRST`修饰符。

`nulls_first()`用于修改由`asc()`或`desc()`产生的表达式，并指示在排序过程中遇到 NULL 值时应如何处理：

```py
from sqlalchemy import desc, nulls_first

stmt = select(users_table).order_by(
    nulls_first(desc(users_table.c.name)))
```

上述 SQL 表达式将类似于：

```py
SELECT id, name FROM user ORDER BY name DESC NULLS FIRST
```

类似于`asc()`和`desc()`，`nulls_first()`通常从列表达式本身使用`ColumnElement.nulls_first()`调用，而不是作为其独立的函数版本，如下所示：

```py
stmt = select(users_table).order_by(
    users_table.c.name.desc().nulls_first())
```

从版本 1.4 开始更改：`nulls_first()`从以前的版本中的`nullsfirst()`重命名。 以前的名称仍可供向后兼容使用。

另请参阅

`asc()`

`desc()`

`nulls_last()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.nullsfirst()
```

`nulls_first()`函数的同义词。

在版本 2.0.5 中更改：恢复了缺失的传统符号`nullsfirst()`。

```py
function sqlalchemy.sql.expression.nulls_last(column: _ColumnExpressionArgument[_T]) → UnaryExpression[_T]
```

生成`ORDER BY`表达式的`NULLS LAST`修饰符。

`nulls_last()`旨在修改由`asc()`或`desc()`生成的表达式，并指示在排序过程中遇到 NULL 值时应如何处理：

```py
from sqlalchemy import desc, nulls_last

stmt = select(users_table).order_by(
    nulls_last(desc(users_table.c.name)))
```

上述 SQL 表达式类似于：

```py
SELECT id, name FROM user ORDER BY name DESC NULLS LAST
```

与`asc()`和`desc()`类似，`nulls_last()`通常是从列表达式本身使用`ColumnElement.nulls_last()`调用的，而不是作为独立的函数版本，如下所示：

```py
stmt = select(users_table).order_by(
    users_table.c.name.desc().nulls_last())
```

在版本 1.4 中更改：`nulls_last()`从先前版本的`nullslast()`重命名。以前的名称仍可用于向后兼容。

另请参见

`asc()`

`desc()`

`nulls_first()`

`Select.order_by()`

```py
function sqlalchemy.sql.expression.nullslast()
```

`nulls_last()`函数的传统同义词。

在版本 2.0.5 中更改：恢复了缺失的传统符号`nullslast()`。

```py
function sqlalchemy.sql.expression.over(element: FunctionElement[_T], partition_by: _ByArgument | None = None, order_by: _ByArgument | None = None, range_: typing_Tuple[int | None, int | None] | None = None, rows: typing_Tuple[int | None, int | None] | None = None) → Over[_T]
```

生成针对函数的`Over`对象。

用于聚合或所谓的“窗口”函数，适用于支持窗口函数的数据库后端。

`over()`通常使用`FunctionElement.over()`方法调用，例如：

```py
func.row_number().over(order_by=mytable.c.some_column)
```

将产生：

```py
ROW_NUMBER() OVER(ORDER BY some_column)
```

使用`over.range_`和`over.rows`参数也可以实现范围。这些互斥参数每个都接受一个 2 元组，其中包含整数和 None 的组合：

```py
func.row_number().over(
    order_by=my_table.c.some_column, range_=(None, 0))
```

以上将产生：

```py
ROW_NUMBER() OVER(ORDER BY some_column
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

`None`值表示“无界”，零值表示“当前行”，负/正整数表示“前面”和“后面”：

+   RANGE BETWEEN 5 PRECEDING AND 10 FOLLOWING:

    ```py
    func.row_number().over(order_by='x', range_=(-5, 10))
    ```

+   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW:

    ```py
    func.row_number().over(order_by='x', rows=(None, 0))
    ```

+   RANGE BETWEEN 2 PRECEDING AND UNBOUNDED FOLLOWING:

    ```py
    func.row_number().over(order_by='x', range_=(-2, None))
    ```

+   RANGE BETWEEN 1 FOLLOWING AND 3 FOLLOWING:

    ```py
    func.row_number().over(order_by='x', range_=(1, 3))
    ```

参数：

+   `element` – 一个`FunctionElement`、`WithinGroup`或其他兼容的构造。

+   `partition_by` – 一个列元素或字符串，或者一个这样的列表，将用作 OVER 构造的 PARTITION BY 子句。

+   `order_by` – 一个列元素或字符串，或者一个这样的列表，将用作 OVER 构造的 ORDER BY 子句。

+   `range_` – 可选的窗口范围子句。这是一个元组值，可以包含整数值或`None`，并将呈现一个 RANGE BETWEEN PRECEDING / FOLLOWING 子句。

+   `rows` – 可选的行子句，用于窗口。这是一个元组值，可以包含整数值或 None，并将呈现一个 ROWS BETWEEN PRECEDING / FOLLOWING 子句。

该函数也可以通过`func`构造本身的`FunctionElement.over()`方法来调用。

另请参阅

使用窗口函数 - 在 SQLAlchemy 统一教程中

`func`

`within_group()`

```py
function sqlalchemy.sql.expression.within_group(element: FunctionElement[_T], *order_by: _ColumnExpressionArgument[Any]) → WithinGroup[_T]
```

产生一个`WithinGroup`对象针对一个函数。

用于所谓的“有序集合聚合”和“假设集合聚合”函数，包括`percentile_cont`，`rank`，`dense_rank`等。

`within_group()`通常使用`FunctionElement.within_group()`方法调用，例如：

```py
from sqlalchemy import within_group
stmt = select(
    department.c.id,
    func.percentile_cont(0.5).within_group(
        department.c.salary.desc()
    )
)
```

上述语句将生成类似于`SELECT department.id, percentile_cont(0.5) WITHIN GROUP (ORDER BY department.salary DESC)`的 SQL。

参数：

+   `element` – 一个`FunctionElement`构造，通常由`func`生成。

+   `*order_by` – 一个或多个列元素，将用作 WITHIN GROUP 构造的 ORDER BY 子句。

另请参阅

特殊修饰符 WITHIN GROUP, FILTER - 在 SQLAlchemy 统一教程中

`func`

`over()`

## 列元素类文档

这里的类是使用列元素基础构造函数和列元素修饰符构造函数列出的构造函数生成的。

| 对象名称 | 描述 |
| --- | --- |
| BinaryExpression | 代表一个`LEFT <operator> RIGHT`的表达式。 |
| BindParameter | 代表一个“绑定表达式”。 |
| Case | 代表一个`CASE`表达式。 |
| Cast | 代表一个`CAST`表达式。 |
| ClauseList | 描述由运算符分隔的子句列表。 |
| ColumnClause | 代表来自任何文本字符串的列表达式。 |
| ColumnCollection | 包含`ColumnElement`实例的集合，通常用于`FromClause`对象。 |
| ColumnElement | 代表一个以列为导向的 SQL 表达式，适用于语句的“columns”子句、WHERE 子句等。 |
| ColumnExpressionArgument | 通用的“列表达式”参数。 |
| ColumnOperators | 为 `ColumnElement` 表达式定义布尔、比较和其他运算符。 |
| Extract | 代表一个 SQL EXTRACT 子句，`extract(field FROM expr)`。 |
| False_ | 代表 SQL 语句中的 `false` 关键字或等效项。 |
| FunctionFilter | 代表一个函数 FILTER 子句。 |
| Label | 表示列标签 (AS)。 |
| Null | 代表 SQL 语句中的 NULL 关键字。 |
| Operators | 比较和逻辑运算符的基础。 |
| Over | 代表一个 OVER 子句。 |
| SQLColumnExpression | 可以用来表示任何 SQL 列元素或充当之一的对象的类型。 |
| TextClause | 代表一个文字 SQL 文本片段。 |
| True_ | 代表 SQL 语句中的 `true` 关键字或等效项。 |
| TryCast | 代表一个 TRY_CAST 表达式。 |
| Tuple | 代表一个 SQL 元组。 |
| TypeCoerce | 代表一个 Python 端的类型强制转换包装器。 |
| UnaryExpression | 定义一个‘一元’表达式。 |
| WithinGroup | 代表一个 WITHIN GROUP (ORDER BY) 子句。 |
| WrapsColumnExpression | 定义一个 `ColumnElement` 作为具有特殊标签行为的包装器的混合。 |

```py
class sqlalchemy.sql.expression.BinaryExpression
```

代表一个 `LEFT <operator> RIGHT` 的表达式。

当两个列表达式在 Python 二进制表达式中使用时，会自动生成一个 `BinaryExpression`：

```py
>>> from sqlalchemy.sql import column
>>> column('a') + column('b')
<sqlalchemy.sql.expression.BinaryExpression object at 0x101029dd0>
>>> print(column('a') + column('b'))
a  +  b 
```

**类签名**

类 `sqlalchemy.sql.expression.BinaryExpression` (`sqlalchemy.sql.expression.OperatorExpression`)

```py
class sqlalchemy.sql.expression.BindParameter
```

代表一个“绑定表达式”。

`BindParameter`是通过`bindparam()`函数显式调用的，如下所示：

```py
from sqlalchemy import bindparam

stmt = select(users_table).where(
    users_table.c.name == bindparam("username")
)
```

如何使用`BindParameter`的详细讨论在`bindparam()`处。

另请参见

`bindparam()`

**成员**

effective_value，inherit_cache，render_literal_execute()

**类签名**

类`sqlalchemy.sql.expression.BindParameter`（`sqlalchemy.sql.roles.InElementRole`，`sqlalchemy.sql.expression.KeyedColumnElement`）

```py
attribute effective_value
```

返回此绑定参数的值，考虑到是否设置了`callable`参数。

如果存在`callable`值，则将其计算并返回，否则返回`value`。

```py
attribute inherit_cache: bool | None = True
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑是否应该参与缓存；这在功能上等同于将值设置为`False`，除了还会发出警告。

如果与此类本地属性相关而不是其超类的属性的 SQL 不会更改，则可以在特定类上将此标志设置为`True`。

另请参见

为自定义结构启用缓存支持 - 为第三方或用户定义的 SQL 结构设置`HasCacheKey.inherit_cache`属性的一般指南。

```py
method render_literal_execute() → BindParameter[_T]
```

生成此绑定参数的副本，该副本将启用`BindParameter.literal_execute`标志。

`BindParameter.literal_execute` 标志会使参数在编译后的 SQL 字符串中以 `[POSTCOMPILE]` 形式呈现，这是一种特殊形式，会在 SQL 执行时转换为参数的字面值。其目的是支持缓存 SQL 语句字符串，其中可以嵌入每个语句的字面值，如 LIMIT 和 OFFSET 参数，在传递给 DBAPI 的最终 SQL 字符串中。特别是方言可能希望在自定义编译方案中使用此方法。

1.4.5 版中的新内容。

另请参阅

第三方方言的缓存

```py
class sqlalchemy.sql.expression.Case
```

表示一个 `CASE` 表达式。

`Case` 是使用 `case()` 工厂函数生成的，如下所示：

```py
from sqlalchemy import case

stmt = select(users_table).                    where(
                case(
                    (users_table.c.name == 'wendy', 'W'),
                    (users_table.c.name == 'jack', 'J'),
                    else_='E'
                )
            )
```

`Case` 用法的详细信息位于 `case()`。

另请参阅

`case()`

**类签名**

类 `sqlalchemy.sql.expression.Case` (`sqlalchemy.sql.expression.ColumnElement`)

```py
class sqlalchemy.sql.expression.Cast
```

表示一个 `CAST` 表达式。

`Cast` 是使用 `cast()` 工厂函数生成的，如下所示：

```py
from sqlalchemy import cast, Numeric

stmt = select(cast(product_table.c.unit_price, Numeric(10, 4)))
```

`Cast` 的用法详见 `cast()`。

另请参阅

数据转换和类型强制转换

`cast()`

`try_cast()`

`type_coerce()` - 一种在 Python 端仅强制类型的替代方法，通常足以生成正确的 SQL 和数据强制转换。

**类签名**

类 `sqlalchemy.sql.expression.Cast` (`sqlalchemy.sql.expression.WrapsColumnExpression`)

```py
class sqlalchemy.sql.expression.ClauseList
```

描述一系列由运算符分隔的子句。

默认情况下，以逗号分隔，例如列列表。

**成员**

self_group()

**类签名**

类`sqlalchemy.sql.expression.ClauseList` (`sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.roles.OrderByRole`, `sqlalchemy.sql.roles.ColumnsClauseRole`, `sqlalchemy.sql.roles.DMLColumnRole`, `sqlalchemy.sql.expression.DQLDMLClauseElement`)

```py
method self_group(against=None)
```

对这个`ClauseElement`应用“分组”。

此方法被子类重写以返回一个“分组”构造，即括号。特别是在“二元”表达式中被用来在放置到较大表达式中时提供自身周围的分组，以及在放置到另一个`select()`的 FROM 子句中时被`select()`构造使用。（请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

当表达式组合在一起时，会自动应用`self_group()` - 最终用户代码通常不需要直接使用这个方法。请注意，SQLAlchemy 的子句构造考虑了操作符优先级 - 因此在像`x OR (y AND z)`这样的表达式中可能不需要括号 - AND 的优先级高于 OR。

`ClauseElement`的基本`self_group()`方法只是返回自身。

```py
class sqlalchemy.sql.expression.ColumnClause
```

表示来自任何文本字符串的列表达式。

`ColumnClause`，是`Column`类的一个轻量级类似物，通常使用`column()`函数来调用，如下所示：

```py
from sqlalchemy import column

id, name = column("id"), column("name")
stmt = select(id, name).select_from("user")
```

上述语句将产生类似的 SQL：

```py
SELECT id, name FROM user
```

`ColumnClause` 是模式特定的 `Column` 对象的直接超类。虽然 `Column` 类具有与 `ColumnClause` 相同的功能，但 `ColumnClause` 类本身可在那些行为要求仅限于简单的 SQL 表达式生成的情况下使用。该对象没有与模式级元数据或执行时行为的关联，因此在这个意义上是 `Column` 的“轻量级”版本。

`ColumnClause` 的完整用法详情请参阅 `column()`。

另请参阅

`column()`

`Column`

**成员**

get_children()

**类签名**

类 `sqlalchemy.sql.expression.ColumnClause` (`sqlalchemy.sql.roles.DDLReferredColumnRole`, `sqlalchemy.sql.roles.LabeledColumnExprRole`, `sqlalchemy.sql.roles.StrAsPlainColumnRole`, `sqlalchemy.sql.expression.Immutable`, `sqlalchemy.sql.expression.NamedColumn`)

```py
method get_children(*, column_tables=False, **kw)
```

返回此 `HasTraverseInternals` 的即时子 `HasTraverseInternals` 元素。

用于访问遍历。

**kw 可能包含改变返回集合的标志，例如返回子集以减少更大的遍历，或从不同的上下文返回子项（例如模式级别集合而不是子句级别）。**

```py
class sqlalchemy.sql.expression.ColumnCollection
```

`ColumnElement` 实例的集合，通常用于 `FromClause` 对象。

`ColumnCollection` 对象最常见的形式是作为 `Table.c` 或 `Table.columns` 在 `Table` 对象上的集合，引入自 访问表和列。

`ColumnCollection`具有映射和序列类似的行为。`ColumnCollection`通常存储`Column`对象，然后可以通过映射样式访问以及属性访问样式访问：

要使用普通的属性样式访问来访问`Column`对象，请像访问任何其他对象属性一样指定名称，例如下面访问了一个名为`employee_name`的列：

```py
>>> employee_table.c.employee_name
```

要访问具有特殊字符或空格名称的列，需要使用索引样式访问，例如下面演示了如何访问名为`employee ' payment`的列：

```py
>>> employee_table.c["employee ' payment"]
```

由于`ColumnCollection`对象提供了 Python 字典接口，因此可用常见的字典方法名称如`ColumnCollection.keys()`、`ColumnCollection.values()`和`ColumnCollection.items()`，这意味着以这些名称为键的数据库列也需要使用索引访问：

```py
>>> employee_table.c["values"]
```

`Column`存在的名称通常是`Column.key`参数的名称。在某些上下文中，例如使用`Select.set_label_style()`方法设置标签样式的`Select`对象中，某个键的列可能会以特定的标签名称表示，例如`tablename_columnname`：

```py
>>> from sqlalchemy import select, column, table
>>> from sqlalchemy import LABEL_STYLE_TABLENAME_PLUS_COL
>>> t = table("t", column("c"))
>>> stmt = select(t).set_label_style(LABEL_STYLE_TABLENAME_PLUS_COL)
>>> subq = stmt.subquery()
>>> subq.c.t_c
<sqlalchemy.sql.elements.ColumnClause at 0x7f59dcf04fa0; t_c>
```

`ColumnCollection`还按顺序索引列，并允许通过它们的整数位置访问它们：

```py
>>> cc[0]
Column('x', Integer(), table=None)
>>> cc[1]
Column('y', Integer(), table=None)
```

从版本 1.4 开始：`ColumnCollection`允许对集合进行基于整数的索引访问。

迭代集合以按顺序生成列表达式：

```py
>>> list(cc)
[Column('x', Integer(), table=None),
 Column('y', Integer(), table=None)]
```

基础 `ColumnCollection` 对象可以存储重复项，这可能意味着两个具有相同键的列，此时通过键访问的列是**任意的**：

```py
>>> x1, x2 = Column('x', Integer), Column('x', Integer)
>>> cc = ColumnCollection(columns=[(x1.name, x1), (x2.name, x2)])
>>> list(cc)
[Column('x', Integer(), table=None),
 Column('x', Integer(), table=None)]
>>> cc['x'] is x1
False
>>> cc['x'] is x2
True
```

或者也可能意味着同一列多次出现。这些情况都受到支持，因为 `ColumnCollection` 用于表示 SELECT 语句中的列，其中可能包括重复项。

还存在一个特殊的子类 `DedupeColumnCollection`，它保留了 SQLAlchemy 的旧行为，不允许重复项；此集合用于模式级对象，如 `Table` 和 `PrimaryKeyConstraint`，其中这种去重是有帮助的。`DedupeColumnCollection` 类还具有额外的变异方法，因为模式构造具有更多需要删除和替换列的用例。

自版本 1.4 更改：`ColumnCollection` 现在存储重复列键以及同一列在多个位置。添加了 `DedupeColumnCollection` 类以维护以前的行为，在这些情况下需要去重以及额外的替换/删除操作。

**成员**

add(), as_readonly(), clear(), compare(), contains_column(), corresponding_column(), get(), items(), keys(), update(), values()

**类签名**

类 `sqlalchemy.sql.expression.ColumnCollection` (`typing.Generic`)

```py
method add(column: ColumnElement[Any], key: _COLKEY | None = None) → None
```

向此 `ColumnCollection` 添加一列。

注意

此方法通常**不由用户界面代码**使用，因为`ColumnCollection`通常是现有对象的一部分，例如`Table`。要将`Column`添加到现有的`Table`对象中，请使用`Table.append_column()`方法。

```py
method as_readonly() → ReadOnlyColumnCollection[_COLKEY, _COL_co]
```

返回此`ColumnCollection`的“只读”形式。

```py
method clear() → NoReturn
```

对于`ColumnCollection`，字典清除()方法未实现。

```py
method compare(other: ColumnCollection[Any, Any]) → bool
```

根据键的名称将此`ColumnCollection`与另一个进行比较

```py
method contains_column(col: ColumnElement[Any]) → bool
```

检查此集合中是否存在列对象

```py
method corresponding_column(column: _COL, require_embedded: bool = False) → _COL | _COL_co | None
```

给定一个`ColumnElement`，从此`ColumnCollection`返回对应于该原始`ColumnElement`的导出`ColumnElement`对象，通过一个公共祖先列。

参数：

+   `column` – 要匹配的目标`ColumnElement`。

+   `require_embedded` – 仅当给定的`ColumnElement`存在于此`Selectable`的子元素中时，返回相应的列。通常情况下，如果列仅与此`Selectable`的导出列共享一个公共祖先，则列将匹配。

另请参阅

`Selectable.corresponding_column()` - 对由`Selectable.exported_columns`返回的集合调用此方法。

在 1.4 版本中更改：`corresponding_column`的实现已移至`ColumnCollection`本身。

```py
method get(key: str, default: _COL_co | None = None) → _COL_co | None
```

基于此`ColumnCollection`中的字符串键名获取一个`ColumnClause`或`Column`对象。

```py
method items() → List[Tuple[_COLKEY, _COL_co]]
```

返回此集合中所有列的(key, column)元组序列，每个元组由字符串键名和`ColumnClause`或`Column`对象组成。

```py
method keys() → List[_COLKEY]
```

返回此集合中所有列的字符串键名序列。

```py
method update(iter_: Any) → NoReturn
```

对于`ColumnCollection`，字典的 update()方法未实现。

```py
method values() → List[_COL_co]
```

返回此集合中所有列的`ColumnClause`或`Column`对象序列。

```py
class sqlalchemy.sql.expression.ColumnElement
```

表示适用于语句的“columns”子句、WHERE 子句等的面向列的 SQL 表达式。

最熟悉的`ColumnElement`类型是`Column`对象，`ColumnElement`作为 SQL 表达式中可能存在的任何单元的基础，包括表达式本身、SQL 函数、绑定参数、文字表达式、`NULL`等关键字。`ColumnElement`是所有这些元素的最终基类。

大量的 SQLAlchemy 核心函数在 SQL 表达式级别工作，并且旨在接受`ColumnElement`实例作为参数。这些函数通常会记录它们接受一个“SQL 表达式”作为参数。在 SQLAlchemy 中，这通常指的是一个已经是`ColumnElement`对象形式的输入，或者可以*强制转换*为其中一个的值。大多数但不是所有 SQLAlchemy 核心函数关于 SQL 表达式的强制转换规则如下：

> +   一个字面的 Python 值，比如字符串、整数或浮点值、布尔值、日期时间、`Decimal`对象，或几乎任何其他 Python 对象，将被强制转换为“字面绑定值”。这通常意味着将生成一个包含给定值嵌入到构造中的`bindparam()`；生成的`BindParameter`对象是`ColumnElement`的一个实例。Python 值最终将在执行时作为参数化参数传递给 DBAPI，作为`execute()`或`executemany()`方法的参数，之前会应用 SQLAlchemy 类型特定的转换器（例如任何相关的`TypeEngine`对象提供的转换器）。
> +   
> +   任何特殊对象值，通常是 ORM 级别的构造，具有名为`__clause_element__()`的访问器。当将一个否则未知类型的对象传递给一个希望将参数强制转换为`ColumnElement`和有时是`SelectBase`表达式的函数时，核心表达式系统会查找此方法。它在 ORM 中用于将 ORM 特定对象（如映射类和映射属性）转换为核心表达式对象。
> +   
> +   Python 的`None`值通常被解释为`NULL`，在 SQLAlchemy Core 中会产生一个`null()`的实例。

一个`ColumnElement`提供了使用 Python 表达式生成新的`ColumnElement`对象的能力。这意味着 Python 运算符如`==`、`!=`和`<`被重载以模仿 SQL 操作，并允许从其他更基本的`ColumnElement`对象实例化进一步的`ColumnElement`实例。例如，两个`ColumnClause`对象可以使用加法运算符`+`相加，以生成一个`BinaryExpression`。`ColumnClause`和`BinaryExpression`都是`ColumnElement`的子类：

```py
>>> from sqlalchemy.sql import column
>>> column('a') + column('b')
<sqlalchemy.sql.expression.BinaryExpression object at 0x101029dd0>
>>> print(column('a') + column('b'))
a  +  b 
```

另请参阅

`Column`

`column()`

**成员**

__eq__(), __le__(), __lt__(), __ne__(), all_(), allows_lambda, anon_key_label, anon_label, any_(), asc(), base_columns, between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), cast(), collate(), comparator, compare(), compile(), concat(), contains(), desc(), description, distinct(), endswith(), entity_namespace, expression, foreign_keys, get_children(), icontains(), iendswith(), ilike(), in_(), inherit_cache, is_(), is_clause_element, is_distinct_from(), is_dml, is_not(), is_not_distinct_from(), is_selectable, isnot(), isnot_distinct_from(), istartswith(), key, label(), like(), match(), negation_clause, not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), params(), primary_key, proxy_set, regexp_match(), regexp_replace(), reverse_operate(), self_group(), shares_lineage(), startswith(), stringify_dialect, supports_execution, timetuple, type, unique_params(), uses_inspection

**类签名**

类`sqlalchemy.sql.expression.ColumnElement`（`sqlalchemy.sql.roles.ColumnArgumentOrKeyRole`、`sqlalchemy.sql.roles.StatementOptionRole`、`sqlalchemy.sql.roles.WhereHavingRole`、`sqlalchemy.sql.roles.BinaryElementRole`、`sqlalchemy.sql.roles.OrderByRole`、`sqlalchemy.sql.roles.ColumnsClauseRole`、`sqlalchemy.sql.roles.LimitOffsetRole`、`sqlalchemy.sql.roles.DMLColumnRole`、`sqlalchemy.sql.roles.DDLConstraintColumnRole`、`sqlalchemy.sql.roles.DDLExpressionRole`、`sqlalchemy.sql.expression.SQLColumnExpression`、`sqlalchemy.sql.expression.DQLDMLClauseElement`）

```py
method __eq__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__eq__` *方法。

实现`==`运算符。

在列上下文中，生成子句`a = b`。如果目标是`None`，则生成`a IS NULL`。

```py
method __le__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__le__` *方法。

实现`<=`运算符。

在列上下文中，生成子句`a <= b`。

```py
method __lt__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__lt__` *方法。

实现`<`运算符。

在列上下文中，生成子句`a < b`。

```py
method __ne__(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `sqlalchemy.sql.expression.ColumnOperators.__ne__` *方法。

实现`!=`运算符。

在列上下文中，生成子句`a != b`。如果目标是`None`，则生成`a IS NOT NULL`。

```py
method all_() → ColumnOperators
```

*继承自* `ColumnOperators` 的 `ColumnOperators.all_()` *方法。

生成针对父对象的`all_()` 子句。

请参阅`all_()` 的文档以获取示例。

注意

请确保不要混淆较新的`ColumnOperators.all_()` 方法与此方法的**传统**版本，即特定于`ARRAY` 的 `Comparator.all()` 方法，其使用不同的调用风格。

```py
attribute allows_lambda = True
```

```py
attribute anon_key_label
```

自 1.4 版起弃用：`ColumnElement.anon_key_label` 属性现在是私有的，公共访问器已被弃用。

```py
attribute anon_label
```

自 1.4 版起弃用：`ColumnElement.anon_label` 属性现在是私有的，公共访问器已被弃用。

```py
method any_() → ColumnOperators
```

*继承自* `ColumnOperators.any_()` *方法的* `ColumnOperators`

生成针对父对象的 `any_()` 子句。

请参阅 `any_()` 的文档以获取示例。

注意

请确保不要将新版 `ColumnOperators.any_()` 方法与这个方法的 **旧版**，即 `Comparator.any()` 方法混淆，后者是专门针对 `ARRAY` 的，并且使用了不同的调用样式。

```py
method asc() → ColumnOperators
```

*继承自* `ColumnOperators.asc()` *方法的* `ColumnOperators`

生成针对父对象的 `asc()` 子句。

```py
attribute base_columns
```

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.between()` *方法的* `ColumnOperators`

生成针对父对象的 `between()` 子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_and()` *方法的* `ColumnOperators`

生成按位与操作，通常通过 `&` 运算符进行。

2.0.2 版中新增。

请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_lshift()` *方法的* `ColumnOperators`

执行按位左移操作，通常通过`<<`运算符。

新版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_not() → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_not()` *方法的* `ColumnOperators`

执行按位非操作，通常通过`~`运算符。

新版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_or()` *方法的* `ColumnOperators`

执行按位或操作，通常通过`|`运算符。

新版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_rshift()` *方法的* `ColumnOperators`

执行按位右移操作，通常通过`>>`运算符。

新版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.bitwise_xor()` *方法的* `ColumnOperators`

执行按位异或操作，通常通过`^`运算符，或者对于 PostgreSQL 使用`#`。

新版本 2.0.2 中新增。

另请参阅

按位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

该方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。 使用 `Operators.bool_op()` 的一个关键优势是，当使用列构造时，返回的表达式的“布尔”性质将出现在[**PEP 484**](https://peps.python.org/pep-0484/)的目的。

另请参阅

`Operators.op()`

```py
method cast(type_: _TypeEngineArgument[_OPT]) → Cast[_OPT]
```

生成类型转换，即 `CAST(<expression> AS <type>)`。

这是对 `cast()` 函数的快捷方式。

另请参阅

数据类型转换和强制类型转换

`cast()`

`type_coerce()`

```py
method collate(collation: str) → ColumnOperators
```

*继承自* `ColumnOperators.collate()` *方法的* `ColumnOperators`

对父对象生成一个 `collate()` 子句，给定排序规则字符串。

另请参阅

`collate()`

```py
attribute comparator
```

```py
method compare(other: ClauseElement, **kw: Any) → bool
```

*继承自* `ClauseElement.compare()` *方法的* `ClauseElement`

比较此 `ClauseElement` 与给定的 `ClauseElement`。

子类应该重写默认行为，即简单的标识比较。

**kw 是子类 `compare()` 方法消耗的参数，可以用来修改比较的条件（参见 `ColumnElement`）。

```py
method compile(bind: _HasDialect | None = None, dialect: Dialect | None = None, **kw: Any) → Compiled
```

*继承自* `CompilerElement.compile()` *方法的* `CompilerElement`

编译此 SQL 表达式。

返回值是一个`Compiled`对象。调用`str()`或`unicode()`返回的值将得到结果的字符串表示。`Compiled`对象还可以使用`params`访问器返回绑定参数名称和值的字典。

参数：

+   `bind` – 一个`Connection`或`Engine`，它可以提供一个`Dialect`以生成一个`Compiled`对象。如果`bind`和`dialect`参数都被省略，则使用默认的 SQL 编译器。

+   `column_keys` – 用于 INSERT 和 UPDATE 语句，应该出现在编译语句的 VALUES 子句中的列名列表。如果是`None`，则渲染目标表对象的所有列。

+   `dialect` – 一个`Dialect`实例，它可以生成一个`Compiled`对象。此参数优先于`bind`参数。

+   `compile_kwargs` –

    可选的额外参数字典，将在所有“visit”方法中传递给编译器。这允许将任何自定义标志传递给自定义编译构造，例如。它还用于传递`literal_binds`标志的情况：

    ```py
    from sqlalchemy.sql import table, column, select

    t = table('t', column('x'))

    s = select(t).where(t.c.x == 5)

    print(s.compile(compile_kwargs={"literal_binds": True}))
    ```

另请参见

如何将 SQL 表达式渲染为字符串，可能包含内联的绑定参数？

```py
method concat(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.concat()` *方法的* `ColumnOperators`

实现‘concat’运算符。

在列上下文中，生成子句`a || b`，或者在 MySQL 上使用`concat()`运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators.contains()` *方法的* `ColumnOperators`

实现‘contains’运算符。

生成一个 LIKE 表达式，用于测试字符串值的中间匹配：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于该操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.contains.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们作为自身而不是通配符字符。或者，`ColumnOperators.contains.escape`参数将建立一个给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。通常是一个普通字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会被转义，除非设置了`ColumnOperators.contains.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    其中参数的值为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来将该字符作为转义字符。然后可以将该字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method desc() → ColumnOperators
```

*继承自* `ColumnOperators.desc()` *方法的* `ColumnOperators`

产生针对父对象的 `desc()` 子句。

```py
attribute description
```

*继承自* `ClauseElement` *的* `ClauseElement.description` *属性*

```py
method distinct() → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.distinct()` *方法*

产生一个针对父对象的 `distinct()` 子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.endswith()` *方法*

实现‘endswith’操作符。

产生一个 LIKE 表达式，测试是否匹配字符串值的结尾：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于操作符使用了`LIKE`，所以存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以将`ColumnOperators.endswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现进行转义，使它们匹配为它们自己而不是通配符字符。或者，`ColumnOperators.endswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` - 要比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非设置了`ColumnOperators.endswith.autoescape`标志为 True，否则 LIKE 通配符字符`%`和`_`默认不会被转义。

+   `autoescape` -

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值是文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    其中`：param`的值为`"foo/%bar"`。

+   `escape` -

    一个字符，当给定时，将以`ESCAPE`关键字呈现，以建立该字符作为转义字符。然后可以将该字符放置在`%`和`_`之前，以使它们能够作为它们自己而不是通配符字符。

    例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在此之上，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute entity_namespace
```

*继承自* `ClauseElement` *的* `ClauseElement.entity_namespace` *属性

```py
attribute expression
```

返回一个列表达式。

检查接口的一部分；返回自身。

```py
attribute foreign_keys: AbstractSet[ForeignKey] = frozenset({})
```

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此`HasTraverseInternals`的直接子`HasTraverseInternals`元素。

用于访问遍历。

**kw 可能包含更改返回的集合的标志，例如返回子项的子集以减少更大的遍历，或者返回来自不同上下文的子项（例如模式级集合而不是子句级集合）。

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.icontains()` *方法

实现`icontains`操作符，例如`ColumnOperators.contains()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的中间进行不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于操作符使用`LIKE`，在<other>表达式中存在的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于文字字符串值，可以设置`ColumnOperators.icontains.autoescape`标志为`True`，以对字符串值中这些字符的出现应用转义，使它们作为自身而不是通配符字符匹配。或者，`ColumnOperators.icontains.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个普通的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符 `%` 和 `_` 不会被转义，除非设置了 `ColumnOperators.icontains.autoescape` 标志为 True。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内的所有 `"%"`、`"_"` 和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    一个表达式，如下所示：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    具有值 `:param` 为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时将以 `ESCAPE` 关键字进行渲染，以将该字符建立为转义字符。然后，可以将此字符放置在 `%` 和 `_` 的前面，以使它们作为它们自身而不是通配符字符。

    一个表达式，如下所示：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    此参数还可以与 `ColumnOperators.contains.autoescape` 结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators` *的* `ColumnOperators.iendswith()` *方法*

实现 `iendswith` 运算符，例如 `ColumnOperators.endswith()` 的不区分大小写版本。

生成一个针对字符串值末尾的不区分大小写匹配的 LIKE 表达式：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于运算符使用了 `LIKE`，存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.iendswith.autoescape` 标志设置为 `True`，以对字符串值内这些字符的出现应用转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.iendswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时可以派上用场。

参数：

+   `other` – 要比较的表达式。这通常是一个普通字符串值，但也可以是任意 SQL 表达式。默认情况下，LIKE 通配符`%`和`_`不会被转义，除非设置了`ColumnOperators.iendswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假定比较值是一个文字字符串而不是 SQL 表达式。

    一个表达式，例如：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    其值为`:param`，为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来确定该字符作为转义字符。然后可以将此字符放在`%`和`_`之前，以使它们可以作为它们自己而不是通配符字符。

    一个表达式，例如：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与`ColumnOperators.iendswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.ilike()` *方法的* `ColumnOperators`

实现`ilike`运算符，例如，不区分大小写的 LIKE。

在列上下文中，生成一个形式为：

```py
lower(a) LIKE lower(other)
```

或在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.in_()` *方法的* `ColumnOperators`

实现`in`运算符。

在列上下文中，生成子句`column IN <other>`。

给定的参数`other`可以是：

+   一个文字值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在这种调用形式中，项目列表将转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的`tuple_()`的元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   一个空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在这种调用形式中，该表达式渲染为一个“空集合”表达式。这些表达式针对各个后端进行了定制，并且通常试图将一个空的 SELECT 语句作为子查询。例如在 SQLite 上，该表达式为：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    1.4 版本中的更改：空的 IN 表达式现在在所有情况下都使用执行时生成的 SELECT 子查询。

+   可以使用绑定参数，例如 `bindparam()`，如果它包含了 `bindparam.expanding` 标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在这种调用形式中，该表达式渲染为一个特殊的非 SQL 占位符表达式，看起来像：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时被拦截，转换为前面示例中所示的可变数量的绑定参数形式。如果执行的语句为：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    1.2 版本中的新功能：添加了“扩展”绑定参数

    如果传递了一个空列表，则会渲染一个特殊的“空列表”表达式，该表达式特定于正在使用的数据库。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    1.3 版本中的新功能：“扩展”绑定参数现在支持空列表

+   一个 `select()` 构造，通常是一个相关的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在这种调用形式中，`ColumnOperators.in_()` 渲染如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个字面值列表，一个 `select()` 构造，或者一个包含设置了 `bindparam.expanding` 标志的 `bindparam()` 构造。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性*

指示此 `HasCacheKey` 实例是否应该使用其直接父类所使用的缓存键生成方案。

该属性默认为 `None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为 `False`，只是还会发出警告。

如果对于特定的类设置了此标志为 `True`，则表示该对象对应的 SQL 不会根据本类而不是其超类的属性而改变。

另请参阅

为自定义构造启用缓存支持 - 为第三方或用户定义的 SQL 构造设置`HasCacheKey.inherit_cache`属性的一般指南。

```py
method is_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_()` *方法的* `ColumnOperators`

实现`IS`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS`，这会解析为`NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
attribute is_clause_element = True
```

```py
method is_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_distinct_from()` *方法的* `ColumnOperators`

实现`IS DISTINCT FROM`运算符。

在大多数平台上呈现“a IS DISTINCT FROM b”；在某些平台上，如 SQLite，可能呈现“a IS NOT b”。

```py
attribute is_dml = False
```

```py
method is_not(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not()` *方法的* `ColumnOperators`

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS NOT`，这会解析为`NULL`。但是，在某些平台上，如果要与布尔值进行比较，则可能希望显式使用`IS NOT`。

从版本 1.4 开始更改：`is_not()`运算符从先前版本的`isnot()`重命名。以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.is_not_distinct_from()` *方法的* `ColumnOperators`

实现`IS NOT DISTINCT FROM`运算符。

在大多数平台上呈现“a IS NOT DISTINCT FROM b”；在某些平台上，如 SQLite，可能呈现“a IS b”。

从版本 1.4 开始更改：`is_not_distinct_from()`运算符从先前版本的`isnot_distinct_from()`重命名。以确保向后兼容性，先前的名称仍然可用。

```py
attribute is_selectable = False
```

```py
method isnot(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot()` *的方法* `ColumnOperators`

实现`IS NOT`操作符。

通常，当与`None`的值进行比较时，将自动生成`IS NOT`，这会解析为`NULL`。但是，如果在某些平台上与布尔值进行比较，则可能希望显式使用`IS NOT`。

版本 1.4 中的更改：`is_not()`操作符从先前的发布中的`isnot()`重新命名。以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.isnot_distinct_from()` *的方法* `ColumnOperators`

实现`IS NOT DISTINCT FROM`操作符。

在大多数平台上渲染为“a IS NOT DISTINCT FROM b”；在某些平台上（如 SQLite）可能渲染为“a IS b”。

版本 1.4 中的更改：`is_not_distinct_from()`操作符从先前的发布中的`isnot_distinct_from()`重新命名。以前的名称仍然可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.istartswith()` *的方法* `ColumnOperators`

实现`istartswith`操作符，例如，不区分大小写版本的`ColumnOperators.startswith()`。

生成一个 LIKE 表达式，用于对字符串值的起始部分进行不区分大小写的匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该操作符使用`LIKE`，因此存在于<other>表达式内部的通配符字符`"%"`和`"_"`也将像通配符一样运行。对于文字字符串值，可以将`ColumnOperators.istartswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，以便它们匹配为它们自己而不是通配符字符。或者，`ColumnOperators.istartswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符`%`和`_`默认情况下不会被转义，除非设置了`ColumnOperators.istartswith.autoescape`标志为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值内的所有`"%"`、`"_"`和转义字符本身的出现，假设比较值是一个文字字符串而不是 SQL 表达式。

    比如这样一个表达式：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    其中`param`的值为`"foo/%bar"`。

+   `escape` –

    一个字符，给定时将以`ESCAPE`关键字呈现，以建立该字符作为转义字符。然后可以在`%`和`_`之前放置此字符，以允许它们作为自身而不是通配符字符。

    比如这样一个表达式：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.istartswith.autoescape`结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况中，给定的文字参数在传递给数据库之前将转换为`"foo^%bar^^bat"`。

另请参见

`ColumnOperators.startswith()`

```py
attribute key: str | None = None
```

在某些情况下引用此对象的‘键’在 Python 命名空间中。

这通常是指在可选择项的`.c`集合中出现的列的“键”，例如，`sometable.c["somekey"]`会返回一个具有“somekey”`.key`的`Column`。

```py
method label(name: str | None) → Label[_T]
```

生成列标签，即 `<columnname> AS <name>`。

这是一个`label()`函数的快捷方式。

如果‘name’是`None`，将生成一个匿名标签名称。

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.like()` *的方法* `ColumnOperators`

实现`like`运算符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现`ESCAPE`关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参见

`ColumnOperators.ilike()`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

*继承自* `ColumnOperators.match()` *的方法* `ColumnOperators`

实现了特定于数据库的‘match’运算符。

`ColumnOperators.match()`尝试解析为后端提供的类似 MATCH 的函数或运算符。例如：

+   PostgreSQL - 渲染`x @@ plainto_tsquery(y)`

    > 从版本 2.0 开始：现在对于 PostgreSQL 使用`plainto_tsquery()`而不是`to_tsquery()`；为了与其他形式兼容，请参阅全文搜索。

+   MySQL - 渲染`MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参见

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 渲染`CONTAINS(x, y)`

+   其他后端可能提供特殊的实现。

+   没有任何特殊实现的后端将将运算符发出为“MATCH”。例如，这与 SQLite 兼容。

```py
attribute negation_clause: ColumnElement[bool]
```

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_ilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这等效于使用否定与`ColumnOperators.ilike()`，即`~x.ilike(y)`。

从版本 1.4 开始更改：`not_ilike()`运算符从先前版本的`notilike()`重命名。以保持向后兼容性，先前的名称仍然可用。

另请参见

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.not_in()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这等效于使用`ColumnOperators.in_()`进行否定，即`~x.in_(y)`。

在`other`为空序列的情况下，编译器会生成一个“空 not in”表达式。这默认为表达式“1 = 1”，以在所有情况下产生 true。`create_engine.empty_in_strategy`可用于更改此行为。

从版本 1.4 开始更改：`not_in()`运算符从先前版本的`notin_()`重命名。以保持向后兼容性，先前的名称仍然可用。

从版本 1.2 开始变更：`ColumnOperators.in_()`和`ColumnOperators.not_in()`操作现在默认为一个空的 IN 序列生成一个“静态”表达式。

另见

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.not_like()` *方法的* `ColumnOperators`

实现`NOT LIKE`运算符。

这相当于使用`ColumnOperators.like()`的否定，即`~x.like(y)`。

从版本 1.4 开始变更：`not_like()`运算符从先前的版本中的`notlike()`重命名。以前的名称仍然可用以实现向后兼容。

另见

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notilike()` *方法的* `ColumnOperators`

实现`NOT ILIKE`运算符。

这相当于使用`ColumnOperators.ilike()`的否定，即`~x.ilike(y)`。

从版本 1.4 开始变更：`not_ilike()`运算符从先前的版本中的`notilike()`重命名。以前的名称仍然可用以实现向后兼容。

另见

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

*继承自* `ColumnOperators.notin_()` *方法的* `ColumnOperators`

实现`NOT IN`运算符。

这相当于使用`ColumnOperators.in_()`的否定，即`~x.in_(y)`。

如果`other`是一个空序列，编译器会生成一个“空不在其中”的表达式。这会默认为表达式“1 = 1”，在所有情况下都返回 true。`create_engine.empty_in_strategy`可以用来改变这种行为。

自 1.4 版本开始：`not_in()` 操作符在之前的版本中从 `notin_()` 重命名。以前的名称仍然可用于向后兼容。

自 1.2 版本开始：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作现在默认生成空的 IN 序列的“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.notlike()` *方法的* `ColumnOperators`

实现 `NOT LIKE` 操作符。

这相当于在 `ColumnOperators.like()` 中使用否定，即 `~x.like(y)`。

自 1.4 版本开始：`not_like()` 操作符在之前的版本中从 `notlike()` 重命名。以前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_first()` *方法的* `ColumnOperators`

生成针对父对象的 `nulls_first()` 子句。

自 1.4 版本开始：`nulls_first()` 操作符在之前的版本中从 `nullsfirst()` 重命名。以前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

*继承自* `ColumnOperators.nulls_last()` *方法的* `ColumnOperators`

生成针对父对象的 `nulls_last()` 子句。

自 1.4 版本开始：`nulls_last()` 操作符在之前的版本中从 `nullslast()` 重命名。以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

*继承自* `ColumnOperators.nullsfirst()` *方法的* `ColumnOperators`

产生一个针对父对象的 `nulls_first()` 子句。

版本 1.4 中的变更：`nulls_first()` 操作符从先前版本的 `nullsfirst()` 重命名。以前的名称仍可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

*继承自* `ColumnOperators.nullslast()` *方法的* `ColumnOperators`

产生一个针对父对象的 `nulls_last()` 子句。

版本 1.4 中的变更：`nulls_last()` 操作符从先前版本的 `nullslast()` 重命名。以前的名称仍可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.op()` *方法的* `Operators`

生成一个通用的操作符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

此函数还可用于显式地指定按位运算符。例如：

```py
somecolumn.op('&')(0xff)
```

是 `somecolumn` 中值的按位 AND。

参数：

+   `opstring` – 将作为中缀操作符输出在此元素和传递给生成函数的表达式之间的字符串。

+   `precedence` –

    数据库在 SQL 表达式中预期应用于操作符的优先级。此整数值充当 SQL 编译器的提示，以便知道何时应在特定操作周围呈现显式括号。较低的数字将导致在应用于具有较高优先级的另一个操作符时，表达式被括起来。默认值 `0` 低于所有操作符，除了逗号 (`,`) 和 `AS` 操作符。值为 100 将高于或等于所有操作符，而 -100 将低于或等于所有操作符。

    另请参见

    我正在使用 op() 生成自定义操作符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何呈现括号的详细描述

+   `is_comparison` –

    遗留; 如果为 True，则该操作符将被视为“比较”操作符，即评估为布尔值 true/false 的操作符，类似于 `==`、`>` 等。此标志提供了当在自定义连接条件中使用操作符时，ORM 关系可以建立该操作符是比较操作符的信息。

    使用 `is_comparison` 参数被使用 `Operators.bool_op()` 方法所取代；这个更简洁的运算符会自动设置这个参数，但也提供了正确的 [**PEP 484**](https://peps.python.org/pep-0484/) 类型支持，因为返回的对象将表达一个“布尔”数据类型，即 `BinaryExpression[bool]`。

+   `return_type` – 一个 `TypeEngine` 类或对象，将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定 `Operators.op.is_comparison` 的运算符将解析为 `Boolean`，而那些不指定的则与左操作数具有相同的类型。

+   `python_impl` –

    一个可选的 Python 函数，可以在与数据库服务器上运行此运算符时以相同方式评估两个 Python 值。用于在 Python 中进行 SQL 表达式评估函数，例如 ORM 混合属性，以及 ORM “评估器”用于在多行更新或删除后匹配会话中的对象。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也适用于非 SQL 左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    新版本 2.0 中新增。

另请参阅

`Operators.bool_op()`

重新定义和创建新运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数进行操作。

这是操作的最低级别，默认情况下会引发 `NotImplementedError`。

在子类上覆盖此方法可以使常见行为应用于所有操作。例如，覆盖 `ColumnOperators` 以将 `func.lower()` 应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用对象。

+   `*other` – 操作的‘另一’侧。对于大多数操作来说，将是一个单一标量。

+   `**kwargs` – 修饰符。这些可以通过特殊运算符传递，比如`ColumnOperators.contains()`。

```py
method params(_ClauseElement__optionaldict: Mapping[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement` *的* `ClauseElement.params()` *方法*

返回一个带有替换了 `bindparam()` 元素的副本。

返回此 ClauseElement 的副本，其中包含从给定字典中取出的 `bindparam()` 元素的值：

```py
>>> clause = column('x') + bindparam('foo')
>>> print(clause.compile().params)
{'foo':None}
>>> print(clause.params({'foo':7}).compile().params)
{'foo':7}
```

```py
attribute primary_key: bool = False
```

```py
attribute proxy_set: util.generic_fn_descriptor[FrozenSet[Any]]
```

我们正在代理的所有列的集合

从 2.0 版本开始，这是显式取消注释的列。以前它实际上是取消注释的列，但没有强制执行。如果可能的话，注释的列基本上不应该进入集合，因为它们的哈希行为非常低效。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_match()` *方法的* `ColumnOperators`

实现了特定于数据库的 ‘regexp match’ 运算符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为由后端提供的类似 REGEXP 的函数或运算符，但是具体的正则表达式语法和可用标志 **不是后端无关的**。

例如：

+   PostgreSQL - 渲染为 `x ~ y` 或当否定时 `x !~ y`。

+   Oracle - 渲染为 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符运算符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将该运算符呈现为 “REGEXP” 或 “NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

目前已为 Oracle、PostgreSQL、MySQL 和 MariaDB 实现了正则表达式支持。SQLite 有部分支持。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅以普通的 Python 字符串形式传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，也可以将标志作为模式的一部分指定。在 PostgreSQL 中使用忽略大小写标志 ‘i’ 时，将使用忽略大小写的正则表达式匹配运算符 `~*` 或 `!~*`。

版本 1.4 中的新功能。

自版本 1.4.48 更改为：2.0.18 请注意，由于实现错误，以前的 “flags” 参数接受了 SQL 表达式对象，例如列表达式，除了普通的 Python 字符串。这种实现与缓存一起使用时无法正常工作，并已删除；应仅传递字符串以用于 “flags” 参数，因为这些标志被渲染为 SQL 表达式中的文字内联值。

另请参阅

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

*继承自* `ColumnOperators.regexp_replace()` *方法的* `ColumnOperators`

实现了数据库特定的“正则替换”运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 尝试解析为后端提供的类似 REGEXP_REPLACE 的函数，通常会发出函数 `REGEXP_REPLACE()`。但是，可用的特定于后端的正则表达式语法和标志是 **不可后端通用的**。

目前已为 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB 实现了正则表达式替换支持。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通的 Python 字符串传递。这些标志是后端特定的。一些后端，如 PostgreSQL 和 MariaDB，也可以将标志作为模式的一部分来指定。

新版本 1.4 中的新功能。

从版本 1.4.48 改变为：2.0.18 请注意，由于实现错误，先前的 “flags” 参数接受了 SQL 表达式对象，如列表达式，而不仅仅是普通的 Python 字符串。这种实现与缓存不兼容，已被删除；只应传递字符串以用于 “flags” 参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参阅

`ColumnOperators.regexp_match()`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → ColumnElement[Any]
```

对参数进行反向操作。

用法与 `operate()` 相同。

```py
method self_group(against: OperatorType | None = None) → ColumnElement[Any]
```

对此 `ClauseElement` 应用“分组”。

此方法被子类覆盖以返回一个“分组”构造，即括号。特别是当“二元”表达式被放置到较大表达式中时，它们会提供一个围绕自己的分组，以及当 `select()` 构造被放置到另一个 `select()` 的 FROM 子句中时。 (请注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句必须具有名称)。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此，例如，在表达式`x OR (y AND z)`中可能不需要括号 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
method shares_lineage(othercolumn: ColumnElement[Any]) → bool
```

如果给定的`ColumnElement`具有与此`ColumnElement`的共同祖先，则返回 True。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

*继承自* `ColumnOperators.startswith()` *方法的* `ColumnOperators`

实现`startswith`操作符。

产生一个 LIKE 表达式，用于测试字符串值的开头是否有匹配项：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该操作符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于文字字符串值，可以将`ColumnOperators.startswith.autoescape`标志设置为`True`，以将这些字符在字符串值中的出现情况应用转义，使它们与自身匹配而不是作为通配符字符。或者，`ColumnOperators.startswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` - 要比较的表达式。这通常是一个纯字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符`%`和`_`不会被转义，除非将`ColumnOperators.startswith.autoescape`标志设置为 True。

+   `autoescape` -

    布尔值；当为 True 时，在 LIKE 表达式中建立转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现次数，假定比较值是一个文字字符串而不是一个 SQL 表达式。

    例如，表达式如下：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    使用值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字来将该字符设为转义字符。然后可以将此字符放在`%`和`_`的前面，以使它们作为自身而不是通配符字符。

    例如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    该参数也可以与`ColumnOperators.startswith.autoescape`结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的文字参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute stringify_dialect = 'default'
```

```py
attribute supports_execution = False
```

```py
attribute timetuple: Literal[None] = None
```

*继承自* `ColumnOperators.timetuple` *属性的* `ColumnOperators`

Hack，允许将日期时间对象与左侧进行比较。

```py
attribute type: TypeEngine[_T]
```

```py
method unique_params(_ClauseElement__optionaldict: Dict[str, Any] | None = None, **kwargs: Any) → Self
```

*继承自* `ClauseElement.unique_params()` *方法的* `ClauseElement`

返回一个副本，其中`bindparam()`元素被替换。

与`ClauseElement.params()`具有相同功能，只是将 unique=True 添加到受影响的绑定参数中，以便可以使用多个语句。

```py
attribute uses_inspection = True
```

```py
sqlalchemy.sql.expression.ColumnExpressionArgument
```

通用的“列表达式”参数。

版本 2.0.13 中新增。

此类型用于通常表示单个 SQL 列表达式的“列”类型表达式，包括`ColumnElement`，以及将具有`__clause_element__()`方法的 ORM 映射属性。

```py
class sqlalchemy.sql.expression.ColumnOperators
```

为`ColumnElement`表达式定义布尔、比较和其他运算符。

默认情况下，所有方法都会调用`operate()` 或 `reverse_operate()`，传入适当的 Python 内置 `operator` 模块中的操作函数或来自 `sqlalchemy.expression.operators` 的 SQLAlchemy 特定操作函数。例如 `__eq__` 函数：

```py
def __eq__(self, other):
    return self.operate(operators.eq, other)
```

`operators.eq` 本质上是：

```py
def eq(a, b):
    return a == b
```

核心列表达式单元`ColumnElement` 重写 `Operators.operate()` 等方法，以返回进一步的 `ColumnElement` 构造，因此上述的 `==` 操作被替换为一个子句构造。

另请参阅

重新定义和创建新的运算符

`TypeEngine.comparator_factory`

`ColumnOperators`

`PropComparator`

**成员**

__add__(), __and__(), __eq__(), __floordiv__(), __ge__(), __getitem__(), __gt__(), __hash__(), __invert__(), __le__(), __lshift__(), __lt__(), __mod__(), __mul__(), __ne__(), __neg__(), __or__(), __radd__(), __rfloordiv__(), __rmod__(), __rmul__(), __rshift__(), __rsub__(), __rtruediv__(), __sa_operate__(), __sub__(), __truediv__(), all_(), any_(), asc(), between(), bitwise_and(), bitwise_lshift(), bitwise_not(), bitwise_or(), bitwise_rshift(), bitwise_xor(), bool_op(), collate(), concat(), contains(), desc(), distinct(), endswith(), icontains(), iendswith(), ilike(), in_(), is_(), is_distinct_from(), is_not(), is_not_distinct_from(), isnot(), isnot_distinct_from(), istartswith(), like(), match(), not_ilike(), not_in(), not_like(), notilike(), notin_(), notlike(), nulls_first(), nulls_last(), nullsfirst(), nullslast(), op(), operate(), regexp_match(), regexp_replace(), reverse_operate(), startswith(), timetuple

**类签名**

类`sqlalchemy.sql.expression.ColumnOperators`（`sqlalchemy.sql.expression.Operators`）

```py
method __add__(other: Any) → ColumnOperators
```

实现 `+` 运算符。

在列上下文中，如果父对象具有非字符串亲和性，则生成子句 `a + b`。如果父对象具有字符串亲和性，则生成连接运算符 `a || b` - 参见`ColumnOperators.concat()`。

```py
method __and__(other: Any) → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__and__` *方法*

实现 `&` 运算符。

当与 SQL 表达式一起使用时，导致一个 AND 操作，相当于`and_()`，即：

```py
a & b
```

相当于：

```py
from sqlalchemy import and_
and_(a, b)
```

在使用 `&` 时应注意运算符的优先级；`&` 运算符具有最高优先级。如果操作数包含进一步的子表达式，则应将其括在括号中：

```py
(a == 2) & (b == 4)
```

```py
method __eq__(other: Any) → ColumnOperators
```

实现 `==` 运算符。

在列上下文中，生成子句 `a = b`。如果目标为 `None`，则生成 `a IS NULL`。

```py
method __floordiv__(other: Any) → ColumnOperators
```

实现 `//` 运算符。

在列上下文中，生成子句 `a / b`，这与“truediv”相同，但考虑结果类型为整数。

新版本 2.0 中新增。

```py
method __ge__(other: Any) → ColumnOperators
```

实现 `>=` 运算符。

在列上下文中，生成子句 `a >= b`。

```py
method __getitem__(index: Any) → ColumnOperators
```

实现 [] 运算符。

这可被一些特定于数据库的类型使用，例如 PostgreSQL ARRAY 和 HSTORE。

```py
method __gt__(other: Any) → ColumnOperators
```

实现 `>` 运算符。

在列上下文中，生成子句 `a > b`。

```py
method __hash__()
```

返回 hash(self)。

```py
method __invert__() → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__invert__` *方法*

实现 `~` 运算符。

当与 SQL 表达式一起使用时，导致一个 NOT 操作，相当于`not_()`，即：

```py
~a
```

相当于：

```py
from sqlalchemy import not_
not_(a)
```

```py
method __le__(other: Any) → ColumnOperators
```

实现 `<=` 运算符。

在列上下文中，生成子句 `a <= b`。

```py
method __lshift__(other: Any) → ColumnOperators
```

实现 << 运算符。

SQLAlchemy 核心不使用此功能，这是为希望使用 << 作为扩展点的自定义运算符系统提供的。

```py
method __lt__(other: Any) → ColumnOperators
```

实现 `<` 运算符。

在列上下文中，生成子句 `a < b`。

```py
method __mod__(other: Any) → ColumnOperators
```

实现 `%` 运算符。

在列上下文中，生成子句 `a % b`。

```py
method __mul__(other: Any) → ColumnOperators
```

实现 `*` 运算符。

在列上下文中，生成子句 `a * b`。

```py
method __ne__(other: Any) → ColumnOperators
```

实现 `!=` 运算符。

在列上下文中，生成子句 `a != b`。如果目标为 `None`，则生成 `a IS NOT NULL`。

```py
method __neg__() → ColumnOperators
```

实现 `-` 运算符。

在列上下文中，生成子句 `-a`。

```py
method __or__(other: Any) → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__or__` *方法*

实现`|`操作符。

与 SQL 表达式一起使用时，结果为 OR 操作，等同于`or_()`，即：

```py
a | b
```

等同于：

```py
from sqlalchemy import or_
or_(a, b)
```

在使用`|`时应注意运算符优先级；`|`运算符具有最高优先级。如果操作数包含进一步的子表达式，则应将其括在括号中：

```py
(a == 2) | (b == 4)
```

```py
method __radd__(other: Any) → ColumnOperators
```

反向实现`+`操作符。

参见`ColumnOperators.__add__()`。

```py
method __rfloordiv__(other: Any) → ColumnOperators
```

反向实现`//`操作符。

参见`ColumnOperators.__floordiv__()`。

```py
method __rmod__(other: Any) → ColumnOperators
```

反向实现`%`操作符。

参见`ColumnOperators.__mod__()`。

```py
method __rmul__(other: Any) → ColumnOperators
```

反向实现`*`操作符。

参见`ColumnOperators.__mul__()`。

```py
method __rshift__(other: Any) → ColumnOperators
```

实现`>>`操作符。

SQLAlchemy 核心未使用此操作符，这是为想要使用`>>`作为扩展点的自定义操作符系统提供的。

```py
method __rsub__(other: Any) → ColumnOperators
```

反向实现`-`操作符。

参见`ColumnOperators.__sub__()`。

```py
method __rtruediv__(other: Any) → ColumnOperators
```

反向实现`/`操作符。

参见`ColumnOperators.__truediv__()`。

```py
method __sa_operate__(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators` *的* `sqlalchemy.sql.expression.Operators.__sa_operate__` *方法*

对参数进行操作。

这是操作的最低级别，默认情况下引发`NotImplementedError`。

在子类中覆盖此方法可以使常见行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左右两侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用对象。

+   `*other` – 操作的‘other’一侧。对于大多数操作，将是单个标量。

+   `**kwargs` – 修饰符。这些可能会被特殊操作符传递，如`ColumnOperators.contains()`。

```py
method __sub__(other: Any) → ColumnOperators
```

实现`-`操作符。

在列上下文中，生成子句`a - b`。

```py
method __truediv__(other: Any) → ColumnOperators
```

实现`/`操作符。

在列上下文中，生成子句`a / b`，并将结果类型视为数值型。

在 2.0 版本中更改：两个整数之间的 truediv 运算符现在被认为返回一个数值。特定后端的行为可能有所不同。

```py
method all_() → ColumnOperators
```

针对父对象生成一个`all_()`子句。

查看`all_()`的文档以获取示例。

注意

请确保不要混淆较新的`ColumnOperators.all_()`方法与这个方法的**旧版**，即特定于`ARRAY`的`Comparator.all()`方法，它使用不同的调用风格。

```py
method any_() → ColumnOperators
```

针对父对象生成一个`any_()`子句。

查看`any_()`的文档以获取示例。

注意

请确保不要混淆较新的`ColumnOperators.any_()`方法与这个方法的**旧版**，即特定于`ARRAY`的`Comparator.any()`方法，它使用不同的调用风格。

```py
method asc() → ColumnOperators
```

针对父对象生成一个`asc()`子句。

```py
method between(cleft: Any, cright: Any, symmetric: bool = False) → ColumnOperators
```

针对父对象生成一个`between()`子句，给定下限和上限范围。

```py
method bitwise_and(other: Any) → ColumnOperators
```

执行按位与操作，通常通过`&`运算符。

新功能在版本 2.0.2 中。

另请参阅

位运算符

```py
method bitwise_lshift(other: Any) → ColumnOperators
```

执行按位左移操作，通常通过`<<`运算符。

新功能在版本 2.0.2 中。

另请参阅

位运算符

```py
method bitwise_not() → ColumnOperators
```

执行按位非操作，通常通过`~`运算符。

新功能在版本 2.0.2 中。

另请参阅

位运算符

```py
method bitwise_or(other: Any) → ColumnOperators
```

执行按位或操作，通常通过`|`运算符。

新功能在版本 2.0.2 中。

另请参阅

位运算符

```py
method bitwise_rshift(other: Any) → ColumnOperators
```

执行按位右移操作，通常通过`>>`运算符。

新功能在版本 2.0.2 中。

另请参阅

位运算符

```py
method bitwise_xor(other: Any) → ColumnOperators
```

执行按位异或操作，通常通过`^`运算符，或者对于 PostgreSQL 使用`#`。

新功能在版本 2.0.2 中。

另请参阅

位运算符

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

*继承自* `Operators.bool_op()` *方法的* `Operators`

返回自定义布尔运算符。

这个方法是调用 `Operators.op()` 并传递 `Operators.op.is_comparison` 标志为 True 的简写。使用 `Operators.bool_op()` 的一个关键优势是，在使用列构造时，返回的表达式的“布尔”性质将用于 [**PEP 484**](https://peps.python.org/pep-0484/)。

另请参阅

`Operators.op()`

```py
method collate(collation: str) → ColumnOperators
```

根据给定的排序字符串生成针对父对象的 `collate()` 子句。

另请参阅

`collate()`

```py
method concat(other: Any) → ColumnOperators
```

实现‘concat’运算符。

在列上下文中，产生子句 `a || b`，或在 MySQL 上使用 `concat()` 运算符。

```py
method contains(other: Any, **kw: Any) → ColumnOperators
```

实现‘包含’运算符。

生成一个 LIKE 表达式，测试字符串值的中间是否存在匹配项：

```py
column LIKE '%' || <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.contains("foobar"))
```

由于该运算符使用 `LIKE`，在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也会像通配符一样起作用。对于字面字符串值，可以将 `ColumnOperators.contains.autoescape` 标志设置为 `True`，以对字符串值中这些字符的出现应用转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.contains.escape` 参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时，这可能会有用。

参数：

+   `other` – 要比较的表达式。这通常是一个普通字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认不会被转义，除非将 `ColumnOperators.contains.autoescape` 标志设置为 True。

+   `autoescape` –

    布尔；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有 `"%"`、`"_"` 和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.contains("foo%bar", autoescape=True)
    ```

    将渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '/'
    ```

    其中 `:param` 的值为 `"foo/%bar"`。

+   `escape` –

    当给出一个字符时，将使用 `ESCAPE` 关键字将其渲染为转义字符。然后可以将此字符放在 `%` 和 `_` 的前面，以允许它们作为自己而不是通配符字符。

    诸如：

    ```py
    somecolumn.contains("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    somecolumn LIKE '%' || :param || '%' ESCAPE '^'
    ```

    此参数也可以与 `ColumnOperators.contains.autoescape` 结合使用：

    ```py
    somecolumn.contains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前被转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.endswith()`

`ColumnOperators.like()`

```py
method desc() → ColumnOperators
```

产生一个针对父对象的 `desc()` 子句。

```py
method distinct() → ColumnOperators
```

产生一个针对父对象的 `distinct()` 子句。

```py
method endswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现 ‘endswith’ 操作符。

产生一个 LIKE 表达式，用于测试字符串值的结尾匹配：

```py
column LIKE '%' || <other>
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.endswith("foobar"))
```

由于操作符使用了 `LIKE`，所以在 <other> 表达式中存在的通配符字符 `"%"` 和 `"_"` 也会像通配符一样运作。对于字面字符串值，可以将 `ColumnOperators.endswith.autoescape` 标志设置为 `True`，以将这些字符在字符串值中的出现转义，使它们匹配为它们自身而不是通配符字符。另外，参数 `ColumnOperators.endswith.escape` 可以将给定字符确定为转义字符，这在目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。这通常是一个简单的字符串值，但也可以是任意的 SQL 表达式。除非将 `ColumnOperators.endswith.autoescape` 标志设置为 True，否则 LIKE 通配符字符 `%` 和 `_` 默认不会被转义。

+   `autoescape` –

    boolean；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有 `"%"`、`"_"` 和转义字符本身的出现，假定比较值是一个字面字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.endswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '/'
    ```

    使用参数的值`"foo/%bar"`。

+   `escape` –

    一个字符，当给出时，将使用`ESCAPE`关键字将该字符确定为转义字符。然后，该字符可以放置在`%`和`_`之前，使它们能够按其自身方式而不是通配符字符进行操作。

    例如：

    ```py
    somecolumn.endswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    somecolumn LIKE '%' || :param ESCAPE '^'
    ```

    该参数还可以与`ColumnOperators.endswith.autoescape`结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况中，给定的文字参数在传递到数据库之前将被转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
method icontains(other: Any, **kw: Any) → ColumnOperators
```

实现`icontains`运算符，例如，`ColumnOperators.contains()`的不区分大小写版本。

产生一个 LIKE 表达式，用于对字符串值的中间进行不区分大小写的匹配：

```py
lower(column) LIKE '%' || lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.icontains("foobar"))
```

由于该运算符使用`LIKE`，存在于<other>表达式内部的通配符字符`"%"`和`"_"`也将像通配符一样工作。对于文字字符串值，可以将`ColumnOperators.icontains.autoescape`标志设置为`True`，以对字符串值内部的这些字符的出现应用转义，使它们作为自身而不是通配符字符进行匹配。或者，`ColumnOperators.icontains.escape`参数将确定给定的字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要进行比较的表达式。通常这是一个简单的字符串值，但也可以是任意的 SQL 表达式。默认情况下，LIKE 通配符字符`"%"`和`"_"`不会被转义，除非将`ColumnOperators.icontains.autoescape`标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有`"%"`、`"_"`和转义字符本身的出现，假设比较值是一个文字字符串而不是 SQL 表达式。

    例如：

    ```py
    somecolumn.icontains("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '/'
    ```

    使用值为`:param`的`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将与`ESCAPE`关键字一起渲染，将该字符建立为转义字符。然后可以将此字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.icontains("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) || '%' ESCAPE '^'
    ```

    参数还可以与`ColumnOperators.contains.autoescape`结合使用：

    ```py
    somecolumn.icontains("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为`"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.contains()`

```py
method iendswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现`iendswith`运算符，例如`ColumnOperators.endswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于测试字符串值末尾的不区分大小写匹配：

```py
lower(column) LIKE '%' || lower(<other>)
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.iendswith("foobar"))
```

由于该运算符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样起作用。对于字面字符串值，可以将`ColumnOperators.iendswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，使它们匹配为自身而不是通配符字符。或者，`ColumnOperators.iendswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常是一个普通字符串值，但也可以是任意 SQL 表达式。除非将`ColumnOperators.iendswith.autoescape`标志设置为 True，否则默认情况下不会转义 LIKE 通配符字符`%`和`_`。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中的所有出现的`"%"`、`"_"`和转义字符本身，假定比较值为字面字符串而不是 SQL 表达式。

    例如表达式：

    ```py
    somecolumn.iendswith("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '/'
    ```

    使用值为`:param`的`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将与`ESCAPE`关键字一起渲染，将该字符建立为转义字符。然后可以将此字符放在`%`和`_`的前面，以允许它们作为自身而不是通配符字符。

    例如表达式：

    ```py
    somecolumn.iendswith("foo/%bar", escape="^")
    ```

    将呈现为：

    ```py
    lower(somecolumn) LIKE '%' || lower(:param) ESCAPE '^'
    ```

    参数也可以与 `ColumnOperators.iendswith.autoescape` 结合使用：

    ```py
    somecolumn.endswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上面的情况下，给定的字面参数在传递到数据库之前将被转换为 `"foo^%bar^^bat"`。

请参见

`ColumnOperators.endswith()`

```py
method ilike(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `ilike` 运算符，例如不区分大小写的 LIKE。

在列上下文中，生成一个表达式，形式为：

```py
lower(a) LIKE lower(other)
```

或者在支持 ILIKE 运算符的后端上：

```py
a ILIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.ilike("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，呈现 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.ilike("foo/%bar", escape="/")
    ```

请参见

`ColumnOperators.like()`

```py
method in_(other: Any) → ColumnOperators
```

实现 `in` 运算符。

在列上下文中，生成子句 `column IN <other>`。

给定参数 `other` 可能是：

+   字面值列表，例如：

    ```py
    stmt.where(column.in_([1, 2, 3]))
    ```

    在此调用形式中，项目列表将转换为与给定列表长度相同的一组绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

+   如果比较是针对包含多个表达式的 `tuple_()`，则可以提供元组列表：

    ```py
    from sqlalchemy import tuple_
    stmt.where(tuple_(col1, col2).in_([(1, 10), (2, 20), (3, 30)]))
    ```

+   空列表，例如：

    ```py
    stmt.where(column.in_([]))
    ```

    在此调用形式中，表达式呈现为空集合表达式。这些表达式针对各个后端进行了定制，并且通常尝试将一个空的 SELECT 语句作为子查询。例如在 SQLite 上，表达式为：

    ```py
    WHERE col IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    1.4 版本中的变更：在所有情况下，空 IN 表达式现在都使用执行时生成的 SELECT 子查询。

+   可以使用绑定参数，例如 `bindparam()`，如果包含 `bindparam.expanding` 标志：

    ```py
    stmt.where(column.in_(bindparam('value', expanding=True)))
    ```

    在此调用形式中，表达式呈现为一个特殊的非 SQL 占位符表达式，如下所示：

    ```py
    WHERE COL IN ([EXPANDING_value])
    ```

    此占位符表达式在语句执行时拦截，以转换为前面所示的变量数目的绑定参数形式。如果语句执行如下所示：

    ```py
    connection.execute(stmt, {"value": [1, 2, 3]})
    ```

    数据库将为每个值传递一个绑定参数：

    ```py
    WHERE COL IN (?, ?, ?)
    ```

    1.2 版本中新增：“expanding” 绑定参数

    如果传递空列表，则呈现特殊的“空列表”表达式，该表达式针对使用的数据库特定。在 SQLite 上，这将是：

    ```py
    WHERE COL IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    ```

    1.3 版本中新增：“expanding” 绑定参数现在支持空列表

+   一个 `select()` 构造，通常是一个关联的标量选择：

    ```py
    stmt.where(
        column.in_(
            select(othertable.c.y).
            where(table.c.x == othertable.c.x)
        )
    )
    ```

    在此调用形式中，`ColumnOperators.in_()` 的呈现如下：

    ```py
    WHERE COL IN (SELECT othertable.y
    FROM othertable WHERE othertable.x = table.x)
    ```

参数：

**other** – 一个字面量列表，一个`select()`构造，或一个包含设置为 True 的`bindparam()`构造，其中包括`bindparam.expanding`标志。

```py
method is_(other: Any) → ColumnOperators
```

实现`IS`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS`，这会解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS`。

另请参阅

`ColumnOperators.is_not()`

```py
method is_distinct_from(other: Any) → ColumnOperators
```

实现`IS DISTINCT FROM`运算符。

大多数平台上呈现“a IS DISTINCT FROM b”; 在某些平台上，如 SQLite，可能呈现“a IS NOT b”。

```py
method is_not(other: Any) → ColumnOperators
```

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS NOT`，这会解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS NOT`。

在 1.4 版本中更改：`is_not()`运算符从先前版本的`isnot()`重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method is_not_distinct_from(other: Any) → ColumnOperators
```

实现`IS NOT DISTINCT FROM`运算符。

大多数平台上呈现“a IS NOT DISTINCT FROM b”; 在某些平台上���如 SQLite，可能呈现“a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()`运算符从先前版本的`isnot_distinct_from()`重命名。以前的名称仍可用于向后兼容。

```py
method isnot(other: Any) → ColumnOperators
```

实现`IS NOT`运算符。

通常，当与`None`的值进行比较时，会自动生成`IS NOT`，这会解析为`NULL`。然而，在某些平台上，如果与布尔值进行比较，则可能希望显式使用`IS NOT`。

在 1.4 版本中更改：`is_not()`运算符从先前版本的`isnot()`重命名。以前的名称仍可用于向后兼容。

另请参阅

`ColumnOperators.is_()`

```py
method isnot_distinct_from(other: Any) → ColumnOperators
```

实现`IS NOT DISTINCT FROM`运算符。

大多数平台上呈现“a IS NOT DISTINCT FROM b”; 在某些平台上，如 SQLite，可能呈现“a IS b”。

在 1.4 版本中更改：`is_not_distinct_from()`运算符从先前版本的`isnot_distinct_from()`重命名。以前的名称仍可用于向后兼容。

```py
method istartswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现`istartswith`运算符，例如`ColumnOperators.startswith()`的不区分大小写版本。

生成一个 LIKE 表达式，用于对字符串值的开头进行不区分大小写匹配：

```py
lower(column) LIKE lower(<other>) || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.istartswith("foobar"))
```

由于该运算符使用 `LIKE`，存在于 <other> 表达式中的通配符字符 `"%"` 和 `"_"` 也将像通配符一样行为。对于字面字符串值，可以设置 `ColumnOperators.istartswith.autoescape` 标志为 `True`，以对字符串值中这些字符的出现应用转义，使它们匹配为它们自身而不是通配符字符。或者，`ColumnOperators.istartswith.escape` 参数将建立一个给定的字符作为转义字符，当目标表达式不是字面字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。通常这是一个普通的字符串值，但也可以是任意的 SQL 表达式。LIKE 通配符字符 `%` 和 `_` 默认情况下不会转义，除非 `ColumnOperators.istartswith.autoescape` 标志设置为 True。

+   `autoescape` –

    布尔值；当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的 `"%"`、`"_"` 和转义字符本身，假定比较值是一个字面字符串而不是 SQL 表达式。

    诸如以下表达式：

    ```py
    somecolumn.istartswith("foo%bar", autoescape=True)
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '/'
    ```

    其值为 `:param` 的值为 `"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用 `ESCAPE` 关键字将其渲染为转义字符。然后，可以将此字符放在 `%` 和 `_` 的前面，以使它们像自己一样而不是通配符字符。

    诸如以下表达式：

    ```py
    somecolumn.istartswith("foo/%bar", escape="^")
    ```

    渲染为：

    ```py
    lower(somecolumn) LIKE lower(:param) || '%' ESCAPE '^'
    ```

    该参数还可以与 `ColumnOperators.istartswith.autoescape` 结合使用：

    ```py
    somecolumn.istartswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.startswith()`

```py
method like(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `like` 运算符。

在列上下文中，生成表达式：

```py
a LIKE other
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.like("%foobar%"))
```

参数：

+   `other` – 要比较的表达式

+   `escape` –

    可选的转义字符，渲染 `ESCAPE` 关键字，例如：

    ```py
    somecolumn.like("foo/%bar", escape="/")
    ```

另请参阅

`ColumnOperators.ilike()`

```py
method match(other: Any, **kwargs: Any) → ColumnOperators
```

实现了数据库特定的 ‘match’ 运算符。

`ColumnOperators.match()` 尝试解析为后端提供的类似 MATCH 的函数或运算符。例如：

+   PostgreSQL - 渲染 `x @@ plainto_tsquery(y)`

    > 自版本 2.0 更改：现在对于 PostgreSQL，使用 `plainto_tsquery()` 而不是 `to_tsquery()`；为了与其他形式兼容，请参阅全文搜索。

+   MySQL - 渲染 `MATCH (x) AGAINST (y IN BOOLEAN MODE)`

    另请参阅

    `match` - 具有附加功能的 MySQL 特定构造。

+   Oracle - 渲染 `CONTAINS(x, y)`

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将将运算符发出为“MATCH”。例如，��与 SQLite 兼容。

```py
method not_ilike(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT ILIKE` 运算符。

这相当于使用 `ColumnOperators.ilike()` 的否定，即 `~x.ilike(y)`。

自版本 1.4 更改：`not_ilike()` 运算符从先前版本的 `notilike()` 重命名。以确保向后兼容性，先前的名称仍然可用。

另请参阅

`ColumnOperators.ilike()`

```py
method not_in(other: Any) → ColumnOperators
```

实现 `NOT IN` 运算符。

这相当于使用 `ColumnOperators.in_()` 的否定，即 `~x.in_(y)`。

如果`other`是一个空序列，编译器将生成一个“空 not in”表达式。这默认为表达式“1 = 1”，在所有情况下产生 true。可以使用 `create_engine.empty_in_strategy` 来更改此行为。

自版本 1.4 更改：`not_in()` 运算符从先前版本的 `notin_()` 重命名。以确保向后兼容性，先前的名称仍然可用。

自版本 1.2 更改：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符现在默认生成一个空 IN 序列的“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method not_like(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT LIKE` 运算符。

这相当于使用 `ColumnOperators.like()` 的否定，即 `~x.like(y)`。

自版本 1.4 起：`not_like()` 操作符从先前版本的 `notlike()` 重命名。先前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method notilike(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT ILIKE` 操作符。

这相当于使用 `ColumnOperators.ilike()` 的否定，即 `~x.ilike(y)`。

自版本 1.4 起：`not_ilike()` 操作符从先前版本的 `notilike()` 重命名。先前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.ilike()`

```py
method notin_(other: Any) → ColumnOperators
```

实现 `NOT IN` 操作符。

这相当于使用 `ColumnOperators.in_()` 的否定，即 `~x.in_(y)`。

如果 `other` 是一个空序列，则编译器会生成一个“空 not in” 表达式。这默认为表达式“1 = 1”，以在所有情况下生成 true。可以使用 `create_engine.empty_in_strategy` 来更改此行为。

自版本 1.4 起：`not_in()` 操作符从先前版本的 `notin_()` 重命名。先前的名称仍然可用于向后兼容。

自版本 1.2 起：`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符现在默认情况下为一个空 IN 序列生成一个“静态”表达式。

另请参阅

`ColumnOperators.in_()`

```py
method notlike(other: Any, escape: str | None = None) → ColumnOperators
```

实现 `NOT LIKE` 操作符。

这相当于使用 `ColumnOperators.like()` 的否定，即 `~x.like(y)`。

自版本 1.4 起：`not_like()` 操作符从先前版本的 `notlike()` 重命名。先前的名称仍然可用于向后兼容。

另请参阅

`ColumnOperators.like()`

```py
method nulls_first() → ColumnOperators
```

生成一个 `nulls_first()` 从句对父对象。

自版本 1.4 起：`nulls_first()` 操作符从先前版本的 `nullsfirst()` 重命名。先前的名称仍然可用于向后兼容。

```py
method nulls_last() → ColumnOperators
```

生成一个针对父对象的`nulls_last()`子句。

在 1.4 版本中更改：`nulls_last()`运算符从之前的版本中的`nullslast()`重新命名。以前的名称仍然可用于向后兼容。

```py
method nullsfirst() → ColumnOperators
```

生成一个针对父对象的`nulls_first()`子句。

在 1.4 版本中更改：`nulls_first()`运算符从之前的版本中的`nullsfirst()`重新命名。以前的名称仍然可用于向后兼容。

```py
method nullslast() → ColumnOperators
```

生成一个针对父对象的`nulls_last()`子句。

在 1.4 版本中更改：`nulls_last()`运算符从之前的版本中的`nullslast()`重新命名。以前的名称仍然可用于向后兼容。

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

*从* `Operators.op()` *方法继承*

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

这个函数也可以用来使位运算符明确。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位 AND。

参数：

+   `opstring` – 一个字符串，将作为中缀运算符输出在此元素和传递给生成函数的表达式之间。

+   `precedence` –

    数据库预期在 SQL 表达式中应用于运算符的优先级。这个整数值充当 SQL 编译器的提示，以便知道何时应该在特定操作周围渲染显式括号。当应用于具有更高优先级的另一个运算符时，较低的数字将导致表达式被括在括号中。默认值为`0`，低于除逗号（`,`）和`AS`运算符之外的所有运算符。值为 100 将高于或等于所有运算符，而-100 将低于或等于所有运算符。

    另请参阅

    我正在使用 op()生成自定义运算符，但是我的括号没有正确出现 - SQLAlchemy SQL 编译器如何呈现括号的详细描述

+   `is_comparison` –

    legacy；如果为 True，则运算符将被视为“比较”运算符，即评估为布尔真/假值的运算符，例如`==`，`>`等。提供此标志是为了让 ORM 关系能够在自定义连接条件中使用时确定该运算符是比较运算符。

    使用 `is_comparison` 参数被 `Operators.bool_op()` 方法取代；这个更简洁的运算符会自动设置这个参数，但也会提供正确的 [**PEP 484**](https://peps.python.org/pep-0484/) 类型支持，因为返回的对象将表示“布尔”数据类型，即 `BinaryExpression[bool]`。

+   `return_type` – 一个 `TypeEngine` 类或对象，将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定了 `Operators.op.is_comparison` 的运算符将解析为 `Boolean`，而未指定的将与左操作数具有相同的类型。

+   `python_impl` –

    一个可选的 Python 函数，可以像在数据库服务器上运行此运算符时一样评估两个 Python 值。适用于在 Python 中进行 SQL 表达式评估函数，例如用于 ORM 混合属性的，以及 ORM “评估器” 用于在多行更新或删除后匹配会话中的对象。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也将适用于非 SQL 左对象和右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版中的新功能。

另请参阅

`Operators.bool_op()`

重新定义和创建新运算符

在联接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.operate()` *方法的* `Operators`

对参数进行操作。

这是操作的最低级别， 默认情况下会引发 `NotImplementedError`。

在子类上覆盖这个可以让常见的行为应用于所有操作。例如，覆盖 `ColumnOperators` 来将 `func.lower()` 应用于左边和右边：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的‘其他’一侧。对于大多数操作来说，它将是一个单个标量。

+   `**kwargs` – 修改器。这些可以通过特殊运算符传递，例如 `ColumnOperators.contains()`。

```py
method regexp_match(pattern: Any, flags: str | None = None) → ColumnOperators
```

实现了特定于数据库的‘regexp 匹配’运算符。

例如：

```py
stmt = select(table.c.some_column).where(
    table.c.some_column.regexp_match('^(b|c)')
)
```

`ColumnOperators.regexp_match()` 尝试解析为后端提供的类似于 REGEXP 的函数或运算符，但是特定的正则表达式语法和可用标志**不是与后端无关**。

示例包括：

+   PostgreSQL - 在否定时呈现 `x ~ y` 或 `x !~ y`。

+   Oracle - 在 Oracle 中呈现 `REGEXP_LIKE(x, y)`

+   SQLite - 使用 SQLite 的 `REGEXP` 占位符运算符，并调用 Python 的 `re.match()` 内置函数。

+   其他后端可能提供特殊实现。

+   没有任何特殊实现的后端将生成运算符为“REGEXP”或“NOT REGEXP”。例如，这与 SQLite 和 MySQL 兼容。

目前为止，正则表达式支持已经实现了 Oracle、PostgreSQL、MySQL 和 MariaDB。对 SQLite 的支持是部分的。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。在 PostgreSQL 中使用忽略大小写标志 ‘i’ 时，将使用忽略大小写的正则表达式匹配运算符 `~*` 或 `!~*`。

1.4 版本中新增。

从 1.4.48 版更改，: 2.0.18 请注意，由于实现错误，以前的 “flags” 参数接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。这种实现不能正确地与缓存一起使用，并且已被移除；应该仅传递字符串作为 “flags” 参数，因为这些标志将作为 SQL 表达式中的字面内联值呈现。

另请参阅

`ColumnOperators.regexp_replace()`

```py
method regexp_replace(pattern: Any, replacement: Any, flags: str | None = None) → ColumnOperators
```

实现了一个特定于数据库的‘正则表达式替换’运算符。

例如：

```py
stmt = select(
    table.c.some_column.regexp_replace(
        'b(..)',
        'XY',
        flags='g'
    )
)
```

`ColumnOperators.regexp_replace()` 尝试解析为由后端提供的类似于 REGEXP_REPLACE 的函数，通常会生成函数 `REGEXP_REPLACE()`。然而，特定的正则表达式语法和可用标志**不是与后端无关**。

目前为止，正则表达式替换支持的数据库包括 Oracle、PostgreSQL、MySQL 8 或更高版本和 MariaDB。第三方方言中的支持可能有所不同。

参数：

+   `pattern` – 正则表达式模式字符串或列子句。

+   `pattern` – 替换字符串或列子句。

+   `flags` – 要应用的任何正则表达式字符串标志，仅作为普通 Python 字符串传递。这些标志是特定于后端的。一些后端，如 PostgreSQL 和 MariaDB，可能会将标志作为模式的一部分指定。

1.4 版本中新增。

在版本 1.4.48 中更改为：2.0.18 请注意，由于实现错误，“flags”参数先前接受了 SQL 表达式对象，例如列表达式，而不仅仅是普通的 Python 字符串。 这种实现与缓存不兼容，并已删除； 仅应传递字符串作为“flags”参数，因为这些标志将作为 SQL 表达式中的文字内联值呈现。

另请参见

`ColumnOperators.regexp_match()`

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

*继承自* `Operators.reverse_operate()` *方法的* `Operators`

对参数执行反向操作。

用法与`operate()`相同。

```py
method startswith(other: Any, escape: str | None = None, autoescape: bool = False) → ColumnOperators
```

实现`startswith`运算符。

生成一个 LIKE 表达式，用于测试字符串值开头的匹配项：

```py
column LIKE <other> || '%'
```

例如：

```py
stmt = select(sometable).\
    where(sometable.c.column.startswith("foobar"))
```

由于该运算符使用`LIKE`，因此存在于<other>表达式中的通配符字符`"%"`和`"_"`也将像通配符一样起作用。 对于文字字符串值，可以将`ColumnOperators.startswith.autoescape`标志设置为`True`，以对字符串值中这些字符的出现应用转义，以便它们作为自身而不是通配符字符进行匹配。 或者，`ColumnOperators.startswith.escape`参数将建立一个给定字符作为转义字符，当目标表达式不是文字字符串时可能会有用。

参数：

+   `other` – 要比较的表达式。 这通常是一个普通字符串值，但也可以是任意 SQL 表达式。 除非将`ColumnOperators.startswith.autoescape`标志设置为 True，否则不会转义 LIKE 通配符`%`和`_`。

+   `autoescape` –

    布尔值； 当为 True 时，在 LIKE 表达式中建立一个转义字符，然后将其应用于比较值中所有出现的`"%"`，`"_"`和转义字符本身，假定比较值是文字字符串而不是 SQL 表达式。

    诸如：

    ```py
    somecolumn.startswith("foo%bar", autoescape=True)
    ```

    将呈现为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '/'
    ```

    具有值`:param`为`"foo/%bar"`。

+   `escape` –

    一个字符，当给定时，将使用`ESCAPE`关键字将该字符建立���转义字符。 然后，可以将此字符放在`%`和`_`之前，以允许它们作为自身而不是通配符字符。

    诸如：

    ```py
    somecolumn.startswith("foo/%bar", escape="^")
    ```

    将渲染为：

    ```py
    somecolumn LIKE :param || '%' ESCAPE '^'
    ```

    参数还可以与 `ColumnOperators.startswith.autoescape` 结合使用：

    ```py
    somecolumn.startswith("foo%bar^bat", escape="^", autoescape=True)
    ```

    在上述情况下，给定的字面参数将在传递到数据库之前转换为 `"foo^%bar^^bat"`。

另请参阅

`ColumnOperators.endswith()`

`ColumnOperators.contains()`

`ColumnOperators.like()`

```py
attribute timetuple: Literal[None] = None
```

Hack，允许在左手边比较 datetime 对象。

```py
class sqlalchemy.sql.expression.Extract
```

表示 SQL EXTRACT 子句，`extract(field FROM expr)`。

**类签名**

类 `sqlalchemy.sql.expression.Extract`（`sqlalchemy.sql.expression.ColumnElement`）

```py
class sqlalchemy.sql.expression.False_
```

表示 SQL 语句中的 `false` 关键字，或等效的内容。

`False_` 通过 `false()` 函数作为常量访问。

**类签名**

类 `sqlalchemy.sql.expression.False_`（`sqlalchemy.sql.expression.SingletonConstant`、`sqlalchemy.sql.roles.ConstExprRole`、`sqlalchemy.sql.expression.ColumnElement`）

```py
class sqlalchemy.sql.expression.FunctionFilter
```

表示一个函数 FILTER 子句。

这是针对聚合和窗口函数的特殊操作符，用于控制传递给它的行。仅受某些数据库后端支持。

`FunctionFilter` 的调用通过 `FunctionElement.filter()` 完成：

```py
func.count(1).filter(True)
```

另请参阅

`FunctionElement.filter()`

**成员**

filter(), over(), self_group()

**类签名**

类 `sqlalchemy.sql.expression.FunctionFilter`（`sqlalchemy.sql.expression.ColumnElement`）

```py
method filter(*criterion: _ColumnExpressionArgument[bool]) → Self
```

对函数执行额外的 FILTER。 

此方法在`FunctionElement.filter()`设置的初始条件之上添加了额外的条件。

多个条件在 SQL 渲染时通过`AND`连接在一起。

```py
method over(partition_by: Iterable[_ColumnExpressionArgument[Any]] | _ColumnExpressionArgument[Any] | None = None, order_by: Iterable[_ColumnExpressionArgument[Any]] | _ColumnExpressionArgument[Any] | None = None, range_: typing_Tuple[int | None, int | None] | None = None, rows: typing_Tuple[int | None, int | None] | None = None) → Over[_T]
```

对此过滤函数生成一个 OVER 子句。

用于对聚合或所谓的“窗口”函数进行操作，适用于支持窗口函数的数据库后端。

表达式：

```py
func.rank().filter(MyClass.y > 5).over(order_by='x')
```

是的速记为：

```py
from sqlalchemy import over, funcfilter
over(funcfilter(func.rank(), MyClass.y > 5), order_by='x')
```

有关详细说明，请参见`over()`。

```py
method self_group(against: OperatorType | None = None) → Self | Grouping[_T]
```

对此`ClauseElement`应用一个‘分组’。

子类重写此方法以返回“分组”结构，即括号。特别是它被“二进制”表达式使用时，在放置到较大表达式中时提供了一个围绕自身的分组，以及当放置到另一个`select()`的 FROM 子句中时，由`select()`结构使用。 （请注意，通常应使用`Select.alias()`方法创建子查询，因为许多平台要求嵌套 SELECT 语句被命名）。

随着表达式的组合，对`self_group()`的应用是自动的 - 最终用户代码不应该直接使用这个方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此可能不需要括号，例如，在`x OR (y AND z)`这样的表达式中 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
class sqlalchemy.sql.expression.Label
```

表示列标签（AS）。

表示一个标签，通常使用`AS` SQL 关键字应用到任何列级元素上。

**成员**

foreign_keys, primary_key, self_group()

**类签名**

类`sqlalchemy.sql.expression.Label` (`sqlalchemy.sql.roles.LabeledColumnExprRole`, `sqlalchemy.sql.expression.NamedColumn`)

```py
attribute foreign_keys: AbstractSet[ForeignKey]
```

```py
attribute primary_key: bool
```

```py
method self_group(against=None)
```

对此`ClauseElement`应用一个‘分组’。

此方法被子类重写为返回“分组”构造，即括号。特别是它被“二进制”表达式用于在放置到更大的表达式中时提供自己周围的分组，以及当放置到另一个`select()`的 FROM 子句中时，被`select()`构造使用。 （请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须具有名称）。

当表达式组合在一起时，自动应用`self_group()` - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造会考虑运算符优先级 - 因此在表达式中可能不需要括号，例如`x OR (y AND z)` - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

```py
class sqlalchemy.sql.expression.Null
```

表示 SQL 语句中的 NULL 关键字。

`Null` 通过`null()`函数作为常量访问。

**类签名**

类`sqlalchemy.sql.expression.Null` (`sqlalchemy.sql.expression.SingletonConstant`, `sqlalchemy.sql.roles.ConstExprRole`, `sqlalchemy.sql.expression.ColumnElement`)。

```py
class sqlalchemy.sql.expression.Operators
```

比较和逻辑运算符的基础。

实现基本方法`Operators.operate()`和`Operators.reverse_operate()`，以及`Operators.__and__()`、`Operators.__or__()`、`Operators.__invert__()`。

**成员**

__and__(), __invert__(), __or__(), __sa_operate__(), bool_op(), op(), operate(), reverse_operate()

通常是通过其最常见的子类`ColumnOperators`来使用。

```py
method __and__(other: Any) → Operators
```

实现`&`运算符。

在与 SQL 表达式一起使用时，会导致 AND 操作，等同于`and_()`，即：

```py
a & b
```

等同于：

```py
from sqlalchemy import and_
and_(a, b)
```

在使用`&`时应注意运算符优先级；`&`运算符具有最高优先级。如果操作数包含进一步的子表达式，则应将其括在括号中：

```py
(a == 2) & (b == 4)
```

```py
method __invert__() → Operators
```

实现`~`运算符。

在与 SQL 表达式一起使用时，会导致 NOT 操作，等同于`not_()`，即：

```py
~a
```

等同于：

```py
from sqlalchemy import not_
not_(a)
```

```py
method __or__(other: Any) → Operators
```

实现`|`运算符。

在与 SQL 表达式一起使用时，会导致 OR 操作，等同于`or_()`，即：

```py
a | b
```

等同于：

```py
from sqlalchemy import or_
or_(a, b)
```

在使用`|`时应注意运算符优先级；`|`运算符具有最高优先级。如果操作数包含进一步的子表达式，则应将其括在括号中：

```py
(a == 2) | (b == 4)
```

```py
method __sa_operate__(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

对参数进行操作。

这是最低级别的操作， 默认情况下会引发`NotImplementedError`。

在子类上覆盖此操作可以使常见行为应用于所有操作。例如，覆盖`ColumnOperators`以将`func.lower()`应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 运算符可调用。

+   `*other` – 操作的‘other’一侧。对于大多数操作，将是一个单一标量。

+   `**kwargs` – 修饰符。这些可以通过特殊运算符（如`ColumnOperators.contains()`）传递。

```py
method bool_op(opstring: str, precedence: int = 0, python_impl: Callable[[...], Any] | None = None) → Callable[[Any], Operators]
```

返回一个自定义布尔运算符。

此方法是调用`Operators.op()`并传递带有 True 的`Operators.op.is_comparison`标志的简写。使用`Operators.bool_op()`的一个关键优势是，在使用列构造时，返回表达式的“布尔”性质将出现在[**PEP 484**](https://peps.python.org/pep-0484/)中。

另请参阅

`Operators.op()`

```py
method op(opstring: str, precedence: int = 0, is_comparison: bool = False, return_type: Type[TypeEngine[Any]] | TypeEngine[Any] | None = None, python_impl: Callable[..., Any] | None = None) → Callable[[Any], Operators]
```

生成一个通用的运算符函数。

例如：

```py
somecolumn.op("*")(5)
```

产生：

```py
somecolumn * 5
```

此函数还可用于使位运算符明确。例如：

```py
somecolumn.op('&')(0xff)
```

是`somecolumn`中值的按位 AND。

参数：

+   `opstring` – 一个字符串，将作为中缀运算符输出在此元素和传递给生成函数的表达式之间。

+   `precedence` –

    数据库预期在 SQL 表达式中应用的运算符的优先级。这个整数值作为 SQL 编译器的提示，以便知道何时应该在特定操作周围呈现显式括号。较低的数字将导致在应用于具有更高优先级的另一个运算符时表达式被括号括起来。默认值为`0`，低于所有运算符，除了逗号（`,`）和`AS`运算符。值为 100 将高于或等于所有运算符，-100 将低于或等于所有运算符。

    另请参阅

    我正在使用 op()生成自定义运算符，但我的括号没有正确显示 - SQLAlchemy SQL 编译器如何呈现括号的详细描述

+   `is_comparison` –

    传统的；如果为 True，则将运算符视为“比较”运算符，即评估为布尔值的运算符，如`==`，`>`等。提供此标志是为了 ORM 关系可以在自定义连接条件中使用时建立该运算符是比较运算符。

    使用`is_comparison`参数已被使用`Operators.bool_op()`方法取代；这个更简洁的运算符会自动设置此参数，但同时提供正确的[**PEP 484**](https://peps.python.org/pep-0484/)类型支持，因为返回的对象将表示“布尔”数据类型，即`BinaryExpression[bool]`。

+   `return_type` – 一个`TypeEngine`类或对象，将强制此运算符生成的表达式的返回类型为该类型。默认情况下，指定`Operators.op.is_comparison`的运算符将解析为`Boolean`，而未指定的运算符将与左操作数的类型相同。

+   `python_impl` –

    一个可选的 Python 函数，可以在数据库服务器上运行时以与此运算符相同的方式评估两个 Python 值。对于在 Python 中的 SQL 表达式评估函数非常有用，例如用于 ORM 混合属性的函数，以及用于在多行更新或删除后匹配会话中的对象的 ORM“评估器”。

    例如：

    ```py
    >>> expr = column('x').op('+', python_impl=lambda a, b: a + b)('y')
    ```

    上述表达式的运算符也适用于非 SQL 左右对象：

    ```py
    >>> expr.operator(5, 10)
    15
    ```

    2.0 版本中的新功能。

另请参阅

`Operators.bool_op()`

重新定义和创建新运算符

在连接条件中使用自定义运算符

```py
method operate(op: OperatorType, *other: Any, **kwargs: Any) → Operators
```

对参数进行操作。

这是操作的最低级别，默认情况下会引发`NotImplementedError`。

在子类上重写这个可以允许将通用行为应用于所有操作。例如，重写 `ColumnOperators` 来将 `func.lower()` 应用于左侧和右侧：

```py
class MyComparator(ColumnOperators):
    def operate(self, op, other, **kwargs):
        return op(func.lower(self), func.lower(other), **kwargs)
```

参数：

+   `op` – 操作符可调用。

+   `*other` – 操作的“其他”方。对于大多数操作，将是一个单个标量。

+   `**kwargs` – 修饰符。这些可以由特殊操作符（如 `ColumnOperators.contains()`）传递。

```py
method reverse_operate(op: OperatorType, other: Any, **kwargs: Any) → Operators
```

对参数进行反向操作。

用法与 `operate()` 相同。

```py
class sqlalchemy.sql.expression.Over
```

表示一个 OVER 子句。

这是针对所谓的“窗口”函数以及任何聚合函数的特殊操作符，它生成相对于结果集本身的结果。现代大多数 SQL 后端现在都支持窗口函数。

**成员**

element

**类签名**

类 `sqlalchemy.sql.expression.Over` （`sqlalchemy.sql.expression.ColumnElement`）

```py
attribute element: ColumnElement[_T]
```

此 `Over` 对象所引用的底层表达式对象。

```py
class sqlalchemy.sql.expression.SQLColumnExpression
```

可用于表示任何 SQL 列元素或充当其中之一的对象的类型。

`SQLColumnExpression` 是 `ColumnElement` 的一个基类，也是 ORM 元素的基类，如 `InstrumentedAttribute` 中所述，并且可以在 [**PEP 484**](https://peps.python.org/pep-0484/) 中用于指示应该作为列表达式行为的参数或返回值。

新版本 2.0.0b4 中引入。

**类签名**

类 `sqlalchemy.sql.expression.SQLColumnExpression` （`sqlalchemy.sql.expression.SQLCoreOperations`、`sqlalchemy.sql.roles.ExpressionElementRole`、`sqlalchemy.util.langhelpers.TypingOnly`）

```py
class sqlalchemy.sql.expression.TextClause
```

表示一个文字 SQL 文本片段。

例如：

```py
from sqlalchemy import text

t = text("SELECT * FROM users")
result = connection.execute(t)
```

使用 `text()` 函数生成 `TextClause` 构造; 请参阅该函数以获取完整文档。

另请参见

`text()`

**成员**

bindparams(), columns(), self_group()

**类签名**

class `sqlalchemy.sql.expression.TextClause` (`sqlalchemy.sql.roles.DDLConstraintColumnRole`, `sqlalchemy.sql.roles.DDLExpressionRole`, `sqlalchemy.sql.roles.StatementOptionRole`, `sqlalchemy.sql.roles.WhereHavingRole`, `sqlalchemy.sql.roles.OrderByRole`, `sqlalchemy.sql.roles.FromClauseRole`, `sqlalchemy.sql.roles.SelectStatementRole`, `sqlalchemy.sql.roles.InElementRole`, `sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.Executable`, `sqlalchemy.sql.expression.DQLDMLClauseElement`, `sqlalchemy.sql.roles.BinaryElementRole`, `sqlalchemy.inspection.Inspectable`)

```py
method bindparams(*binds: BindParameter[Any], **names_to_values: Any) → Self
```

在此 `TextClause` 构造中建立绑定参数的值和/或类型。

给定文本构造，例如：

```py
from sqlalchemy import text
stmt = text("SELECT id, name FROM user WHERE name=:name "
            "AND timestamp=:timestamp")
```

`TextClause.bindparams()` 方法可用于使用简单的关键字参数来建立 `:name` 和 `:timestamp` 的初始值：

```py
stmt = stmt.bindparams(name='jack',
            timestamp=datetime.datetime(2012, 10, 8, 15, 12, 5))
```

在上面的示例中，将生成新的 `BindParameter` 对象，其名称分别为 `name` 和 `timestamp`，值分别为 `jack` 和 `datetime.datetime(2012, 10, 8, 15, 12, 5)`。类型将根据给定的值推断，本例中分别为 `String` 和 `DateTime`。

当需要特定的类型行为时，可以使用位置参数 `*binds` 来直接指定 `bindparam()` 构造。这些构造必须至少包括 `key` 参数，然后是可选的值和类型：

```py
from sqlalchemy import bindparam
stmt = stmt.bindparams(
                bindparam('name', value='jack', type_=String),
                bindparam('timestamp', type_=DateTime)
            )
```

在上面的示例中，我们为 `timestamp` 绑定指定了 `DateTime` 类型，并为 `name` 绑定指定了 `String` 类型。对于 `name`，我们还设置了默认值为 `"jack"`。

额外的绑定参数可以在语句执行时提供，例如：

```py
result = connection.execute(stmt,
            timestamp=datetime.datetime(2012, 10, 8, 15, 12, 5))
```

`TextClause.bindparams()`方法可以重复调用，在其中将重用现有的`BindParameter`对象以添加新信息。例如，我们可以首先使用类型信息调用`TextClause.bindparams()`，然后再次使用值信息调用，它们将被合并：

```py
stmt = text("SELECT id, name FROM user WHERE name=:name "
            "AND timestamp=:timestamp")
stmt = stmt.bindparams(
    bindparam('name', type_=String),
    bindparam('timestamp', type_=DateTime)
)
stmt = stmt.bindparams(
    name='jack',
    timestamp=datetime.datetime(2012, 10, 8, 15, 12, 5)
)
```

`TextClause.bindparams()`方法还支持**唯一**绑定参数的概念。这些参数在语句编译时通过名称“唯一化”，以便多个`text()`构造可以组合在一起而不发生名称冲突。要使用此功能，请在每个`bindparam()`对象上指定`BindParameter.unique`标志：

```py
stmt1 = text("select id from table where name=:name").bindparams(
    bindparam("name", value='name1', unique=True)
)
stmt2 = text("select id from table where name=:name").bindparams(
    bindparam("name", value='name2', unique=True)
)

union = union_all(
    stmt1.columns(column("id")),
    stmt2.columns(column("id"))
)
```

上述语句将呈现为：

```py
select id from table where name=:name_1
UNION ALL select id from table where name=:name_2
```

版本 1.3.11 中的新功能：添加了对与`text()`构造一起使用的`BindParameter.unique`标志的支持。

```py
method columns(*cols: _ColumnExpressionArgument[Any], **types: TypeEngine[Any]) → TextualSelect
```

将此`TextClause`对象转换为一个`TextualSelect`对象，其作用与 SELECT 语句相同。

`TextualSelect`是`SelectBase`层次结构的一部分，可以通过使用`TextualSelect.subquery()`方法将其嵌入到另一个语句中，从而产生一个`Subquery`对象，然后可以从中进行 SELECT。

此函数本质上是在完全文本的 SELECT 语句与 SQL 表达式语言概念“可选择”的之间架起了桥梁：

```py
from sqlalchemy.sql import column, text

stmt = text("SELECT id, name FROM some_table")
stmt = stmt.columns(column('id'), column('name')).subquery('st')

stmt = select(mytable).\
        select_from(
            mytable.join(stmt, mytable.c.name == stmt.c.name)
        ).where(stmt.c.id > 5)
```

在上面的例子中，我们按位置传递了一系列 `column()` 元素给 `TextClause.columns()` 方法。这些 `column()` 元素现在成为 `TextualSelect.selected_columns` 列集合的一等元素，然后在调用 `TextualSelect.subquery()` 之后成为 `Subquery.c` 集合的一部分。

我们传递给 `TextClause.columns()` 的列表达式也可以被类型化；当我们这样做时，这些 `TypeEngine` 对象成为列的有效返回类型，以便 SQLAlchemy 的结果集处理系统可以用于返回值。这通常对于诸如日期或布尔类型以及某些方言配置上的 Unicode 处理是必要的：

```py
stmt = text("SELECT id, name, timestamp FROM some_table")
stmt = stmt.columns(
            column('id', Integer),
            column('name', Unicode),
            column('timestamp', DateTime)
        )

for id, name, timestamp in connection.execute(stmt):
    print(id, name, timestamp)
```

作为上述语法的快捷方式，如果只需要类型转换，则可以使用仅引用类型的关键字参数：

```py
stmt = text("SELECT id, name, timestamp FROM some_table")
stmt = stmt.columns(
            id=Integer,
            name=Unicode,
            timestamp=DateTime
        )

for id, name, timestamp in connection.execute(stmt):
    print(id, name, timestamp)
```

`TextClause.columns()` 的位置形式还提供了**位置列定位**的独特功能，当使用 ORM 处理复杂的文本查询时特别有用。如果我们将模型中的列指定给 `TextClause.columns()`，则结果集将按位置匹配到这些列，这意味着文本 SQL 中列的名称或来源并不重要：

```py
stmt = text("SELECT users.id, addresses.id, users.id, "
     "users.name, addresses.email_address AS email "
     "FROM users JOIN addresses ON users.id=addresses.user_id "
     "WHERE users.id = 1").columns(
        User.id,
        Address.id,
        Address.user_id,
        User.name,
        Address.email_address
     )

query = session.query(User).from_statement(stmt).options(
    contains_eager(User.addresses))
```

`TextClause.columns()` 方法提供了直接调用 `FromClause.subquery()` 以及针对文本 SELECT 语句调用 `SelectBase.cte()` 的途径：

```py
stmt = stmt.columns(id=Integer, name=String).cte('st')

stmt = select(sometable).where(sometable.c.id == stmt.c.id)
```

参数：

+   `*cols` – 一系列 `ColumnElement` 对象，通常是从 `Table` 或 ORM 级别的列映射属性中获取的 `Column` 对象，表示此文本字符串将从中选择的列集。

+   `**types` – 一个将字符串名称映射到 `TypeEngine` 类型对象的映射，指示从文本字符串中选择的名称要使用的数据类型。最好使用 `*cols` 参数，因为它还指示了位置顺序。

```py
method self_group(against=None)
```

对此 `ClauseElement` 应用“分组”。

子类会重写此方法以返回一个“分组”构造，即括号。特别是它被“二进制”表达式使用，当它们被放置到更大的表达式中时，提供了一个围绕自身的分组，以及当 `select()` 构造被放置到另一个 `select()` 的 FROM 子句时。 （请注意，子查询通常应使用 `Select.alias()` 方法创建，因为许多平台要求嵌套的 SELECT 语句被命名）。

表达式组合在一起时，自动应用 `self_group()` 方法 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在表达式中可能不需要括号，例如在 `x OR (y AND z)` 这样的表达式中 - AND 优先于 OR。

`ClauseElement` 的基本 `self_group()` 方法只返回自身。

```py
class sqlalchemy.sql.expression.TryCast
```

表示一个 TRY_CAST 表达式。

关于 `TryCast` 使用的详细信息请参见 `try_cast()`。

另请参阅

`try_cast()`

数据转换和类型强制转换

**成员**

inherit_cache

**类签名**

类 `sqlalchemy.sql.expression.TryCast` (`sqlalchemy.sql.expression.Cast`)

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑它是否适合参与缓存；这在功能上等同于将值设置为`False`，除了还会发出警告。

如果与本类局部属性无关，而不是它的超类，则可以在特定类上将此标志设置为`True`，则与对象对应的 SQL 不会根据本类的属性更改。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
class sqlalchemy.sql.expression.Tuple
```

表示 SQL 元组。

**成员**

self_group()

**类签名**

类`sqlalchemy.sql.expression.Tuple`（`sqlalchemy.sql.expression.ClauseList`，`sqlalchemy.sql.expression.ColumnElement`）

```py
method self_group(against=None)
```

将一个“分组”应用于这个`ClauseElement`。

这个方法被子类重写，以返回一个“分组”结构，即括号。特别是它被“二进制”表达式使用，当它们被放置到更大的表达式中时，提供了一个围绕自身的分组，以及当它们被放置到另一个`select()`构造的 FROM 子句中时。（注意，子查询应该通常使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句被命名）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。注意，SQLAlchemy 的子句构造考虑了操作符的优先级 - 因此在表达式中可能不需要括号，例如在表达式`x OR (y AND z)`中 - AND 优先于 OR。

`ClauseElement` 的基本`self_group()`方法只返回自身。

```py
class sqlalchemy.sql.expression.WithinGroup
```

表示 WITHIN GROUP (ORDER BY) 子句。

这是针对所谓的“有序集合聚合”和“假设集合聚合”函数的特殊运算符，包括`percentile_cont()`、`rank()`、`dense_rank()`等。

仅受某些数据库后端支持，如 PostgreSQL、Oracle 和 MS SQL Server。

`WithinGroup` 构造从方法 `FunctionElement.within_group_type()` 中提取其类型。如果返回`None`，则使用函数的`.type`。

**成员**

over()

**类签名**

类 `sqlalchemy.sql.expression.WithinGroup` (`sqlalchemy.sql.expression.ColumnElement`)

```py
method over(partition_by=None, order_by=None, range_=None, rows=None)
```

产生针对此 `WithinGroup` 构造的 OVER 子句。

此函数具有与 `FunctionElement.over()` 相同的签名���

```py
class sqlalchemy.sql.elements.WrapsColumnExpression
```

定义一个 `ColumnElement` 作为一个包装器，具有对已经具有名称的表达式的特殊标签行为。

版本 1.4 中的新功能。

另请参阅

改进的列标签，用于使用 CAST 或类似方法的简单列表达式

**类签名**

类 `sqlalchemy.sql.expression.WrapsColumnExpression` (`sqlalchemy.sql.expression.ColumnElement`)

```py
class sqlalchemy.sql.expression.True_
```

表示 SQL 语句中的 `true` 关键字或等效关键字。

`True_` 通过 `true()` 函数作为常量访问。

**类签名**

类 `sqlalchemy.sql.expression.True_` (`sqlalchemy.sql.expression.SingletonConstant`, `sqlalchemy.sql.roles.ConstExprRole`, `sqlalchemy.sql.expression.ColumnElement`)

```py
class sqlalchemy.sql.expression.TypeCoerce
```

表示一个 Python 端的类型强制转换包装器。

`TypeCoerce` 提供 `type_coerce()` 函数；请参阅该函数以获取使用详细信息。

另请参阅

`type_coerce()`

`cast()`

**成员**

self_group()

**类签名**

类`sqlalchemy.sql.expression.TypeCoerce` (`sqlalchemy.sql.expression.WrapsColumnExpression`)

```py
method self_group(against=None)
```

对这个`ClauseElement` 应用‘分组’。

此方法被子类重写以返回一个“分组”构造，即括号。特别是当“二元”表达式放入更大的表达式中时，它被“二元”表达式用于提供围绕自身的分组，以及当`select()`构造放入另一个`select()`的 FROM 子句中时。 （请注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句必须命名）。

随着表达式的组合，`self_group()` 的应用是自动的 - 最终用户代码不应直接使用此方法。请注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此在诸如 `x OR (y AND z)` 这样的表达式中可能不需要括号 - AND 优先于 OR。

`ClauseElement` 的基本`self_group()` 方法只返回自身。

```py
class sqlalchemy.sql.expression.UnaryExpression
```

定义一个‘一元’表达式。

一元表达式有一个单列表达式和一个运算符。 运算符可以放在列表达式的左侧（称为‘运算符’）或右侧（称为‘修饰符’）。

`UnaryExpression` 是几个一元运算符的基础，包括`desc()`、`asc()`、`distinct()`、`nulls_first()` 和`nulls_last()`。

**成员**

self_group()

**类签名**

类`sqlalchemy.sql.expression.UnaryExpression`（`sqlalchemy.sql.expression.ColumnElement`）

```py
method self_group(against=None)
```

对这个`ClauseElement`应用一个“分组”。

此方法被子类重写为返回一个“分组”构造，即括号。特别是它被“二元”表达式用于在放置到更大表达式中时提供自身周围的分组，以及被`select()`构造用于放置到另一个`select()`的 FROM 子句中。（注意，子查询通常应使用`Select.alias()`方法创建，因为许多平台要求嵌套的 SELECT 语句有名称）。

当表达式组合在一起时，自动应用`self_group()` - 最终用户代码不应直接使用此方法。注意，SQLAlchemy 的子句构造考虑了运算符优先级 - 因此可能不需要括号，例如，在表达式`x OR (y AND z)`中 - AND 优先于 OR。

`ClauseElement`的基本`self_group()`方法只返回自身。

## 列元素类型工具

从`sqlalchemy`命名空间导入的独立实用函数，以提高类型检查器的支持。

| 对象名称 | 描述 |
| --- | --- |
| NotNullable(val) | 将列或 ORM 类的类型定义为非空。 |
| Nullable(val) | 将列或 ORM 类的类型定义为可空。 |

```py
function sqlalchemy.NotNullable(val: _TypedColumnClauseArgument[_T | None] | Type[_T] | None) → _TypedColumnClauseArgument[_T]
```

将列或 ORM 类的类型定义为非空。

这可用于选择和其他上下文中，以表达列的值不能为 null，例如由于对可空列的 where 条件：

```py
stmt = select(NotNullable(A.value)).where(A.value.is_not(None))
```

在运行时，此方法返回未更改的输入。

版本 2.0.20 中的新功能。

```py
function sqlalchemy.Nullable(val: _TypedColumnClauseArgument[_T]) → _TypedColumnClauseArgument[_T | None]
```

将列或 ORM 类的类型定义为可空。

这可用于选择和其他上下文中，以表达列的值可以为 null，例如由于外连接：

```py
stmt1 = select(A, Nullable(B)).outerjoin(A.bs)
stmt2 = select(A.data, Nullable(B.data)).outerjoin(A.bs)
```

在运行时，此方法返回未更改的输入。

版本 2.0.20 中的新功能。
