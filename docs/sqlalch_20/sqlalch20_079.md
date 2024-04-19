# 运算符参考

> 原文：[`docs.sqlalchemy.org/en/20/core/operators.html`](https://docs.sqlalchemy.org/en/20/core/operators.html)

本节详细介绍了用于构建 SQL 表达式的运算符的用法。

这些方法按照 `Operators` 和 `ColumnOperators` 基类的方式呈现。然后这些方法可用于这些类的后代，包括：

+   `Column` 对象

+   `ColumnElement` 对象更一般地，这些对象是所有 Core SQL 表达式语言列级表达式的根

+   `InstrumentedAttribute` 对象，这些对象是 ORM 级别的映射属性。

运算符首先在教程部分中介绍，包括：

+   SQLAlchemy 统一教程 - 以 2.0 风格 呈现的统一教程

+   对象关系教程 - ORM 教程以 1.x 风格 呈现

+   SQL 表达式语言教程 - 以 1.x 风格 呈现的核心教程

## 比较运算符

基本比较，适用于许多数据类型，包括数值、字符串、日期等：

+   `ColumnOperators.__eq__()` （Python “`==`” 运算符）：

    ```py
    >>> print(column("x") == 5)
    x  =  :x_1 
    ```

+   `ColumnOperators.__ne__()` （Python “`!=`” 运算符）：

    ```py
    >>> print(column("x") != 5)
    x  !=  :x_1 
    ```

+   `ColumnOperators.__gt__()` （Python “`>`” 运算符）：

    ```py
    >>> print(column("x") > 5)
    x  >  :x_1 
    ```

+   `ColumnOperators.__lt__()` （Python “`<`” 运算符）：

    ```py
    >>> print(column("x") < 5)
    x  <  :x_1 
    ```

+   `ColumnOperators.__ge__()` （Python “`>=`” 运算符）：

    ```py
    >>> print(column("x") >= 5)
    x  >=  :x_1 
    ```

+   `ColumnOperators.__le__()` （Python “`<=`” 运算符）：

    ```py
    >>> print(column("x") <= 5)
    x  <=  :x_1 
    ```

+   `ColumnOperators.between()`：

    ```py
    >>> print(column("x").between(5, 10))
    x  BETWEEN  :x_1  AND  :x_2 
    ```

## IN 比较

SQL IN 运算符是 SQLAlchemy 中的一个主题。 由于 IN 运算符通常针对一组固定值使用，因此 SQLAlchemy 的绑定参数 coercion 功能利用了特殊形式的 SQL 编译，将中间 SQL 字符串渲染为最终绑定参数列表，然后在第二步形成。 换句话说，“它只是工作”。

### 针对一组值的 IN

通常通过将值列表传递给 `ColumnOperators.in_()` 方法来实现 IN：

```py
>>> print(column("x").in_([1, 2, 3]))
x  IN  (__[POSTCOMPILE_x_1]) 
```

特殊的绑定形式`__[POSTCOMPILE` 在执行时被渲染为单独的参数，如下所示：

```py
>>> stmt = select(User.id).where(User.id.in_([1, 2, 3]))
>>> result = conn.execute(stmt)
SELECT  user_account.id
FROM  user_account
WHERE  user_account.id  IN  (?,  ?,  ?)
[...]  (1,  2,  3) 
```

### 空 IN 表达式

SQLAlchemy 通过渲染一个返回零行的特定于后端的子查询来为空 IN 表达式产生数学上有效的结果。 再换句话说，“它只是工作”：

```py
>>> stmt = select(User.id).where(User.id.in_([]))
>>> result = conn.execute(stmt)
SELECT  user_account.id
FROM  user_account
WHERE  user_account.id  IN  (SELECT  1  FROM  (SELECT  1)  WHERE  1!=1)
[...]  () 
```

上面的“空集”子查询正确地进行了概括，并且还以保持不变的 IN 运算符形式呈现。

### NOT IN

“NOT IN”可通过`ColumnOperators.not_in()` 运算符获得：

```py
>>> print(column("x").not_in([1, 2, 3]))
(x  NOT  IN  (__[POSTCOMPILE_x_1])) 
```

通常，通过`~`运算符否定更容易实现：

```py
>>> print(~column("x").in_([1, 2, 3]))
(x  NOT  IN  (__[POSTCOMPILE_x_1])) 
```

### 元组 IN 表达式

使用 IN 与元组进行元组比较是常见的，因为在其他用例中，它可以适应将行匹配到一组潜在的复合主键值的情况。 `tuple_()` 构造提供了元组比较的基本构建块。 `Tuple.in_()` 运算符然后接收元组列表：

```py
>>> from sqlalchemy import tuple_
>>> tup = tuple_(column("x", Integer), column("y", Integer))
>>> expr = tup.in_([(1, 2), (3, 4)])
>>> print(expr)
(x,  y)  IN  (__[POSTCOMPILE_param_1]) 
```

为了说明渲染的参数：

```py
>>> tup = tuple_(User.id, Address.id)
>>> stmt = select(User.name).join(Address).where(tup.in_([(1, 1), (2, 2)]))
>>> conn.execute(stmt).all()
SELECT  user_account.name
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  (user_account.id,  address.id)  IN  (VALUES  (?,  ?),  (?,  ?))
[...]  (1,  1,  2,  2)
[('spongebob',), ('sandy',)]
```

### 子查询 IN

最后，`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符与子查询一起使用。 这种形式提供了直接传递 `Select` 构造的方法，无需任何显式转换为命名子查询：

```py
>>> print(column("x").in_(select(user_table.c.id)))
x  IN  (SELECT  user_account.id
FROM  user_account) 
```

元组按预期工作：

```py
>>> print(
...     tuple_(column("x"), column("y")).in_(
...         select(user_table.c.id, address_table.c.id).join(address_table)
...     )
... )
(x,  y)  IN  (SELECT  user_account.id,  address.id
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id) 
```

## 身份比较

这些运算符涉及测试特殊的 SQL 值，如`NULL`，一些数据库支持的布尔常量，如`true`或`false`：

+   `ColumnOperators.is_()`：

    此运算符将为“x IS y”提供确切的 SQL，最常见的是“<expr> IS NULL”。 使用常规 Python `None` 最容易获得`NULL` 常量：

    ```py
    >>> print(column("x").is_(None))
    x  IS  NULL 
    ```

    如果需要，SQL NULL 也可以显式使用`null()` 构造：

    ```py
    >>> from sqlalchemy import null
    >>> print(column("x").is_(null()))
    x  IS  NULL 
    ```

    当使用`ColumnOperators.__eq__()`重载运算符，即`==`，与`None`或`null()`值一起使用时，`ColumnOperators.is_()`运算符会自动调用。因此，通常不需要显式使用`ColumnOperators.is_()`，特别是在与动态值一起使用时：

    ```py
    >>> a = None
    >>> print(column("x") == a)
    x  IS  NULL 
    ```

    注意，Python 的`is`运算符**没有被重载**。尽管 Python 提供了重载运算符如`==`和`!=`的钩子，但它**没有**提供任何重新定义`is`的方法。

+   `ColumnOperators.is_not()`：

    类似于`ColumnOperators.is_()`，生成“IS NOT”：

    ```py
    >>> print(column("x").is_not(None))
    x  IS  NOT  NULL 
    ```

    等同于`!= None`：

    ```py
    >>> print(column("x") != None)
    x  IS  NOT  NULL 
    ```

+   `ColumnOperators.is_distinct_from()`：

    生成 SQL IS DISTINCT FROM：

    ```py
    >>> print(column("x").is_distinct_from("some value"))
    x  IS  DISTINCT  FROM  :x_1 
    ```

+   `ColumnOperators.isnot_distinct_from()`：

    生成 SQL IS NOT DISTINCT FROM：

    ```py
    >>> print(column("x").isnot_distinct_from("some value"))
    x  IS  NOT  DISTINCT  FROM  :x_1 
    ```

## 字符串比较：

+   `ColumnOperators.like()`：

    ```py
    >>> print(column("x").like("word"))
    x  LIKE  :x_1 
    ```

+   `ColumnOperators.ilike()`：

    大小写不敏感的 LIKE 使用通用后端上的 SQL `lower()`函数。在 PostgreSQL 后端上，它将使用`ILIKE`：

    ```py
    >>> print(column("x").ilike("word"))
    lower(x)  LIKE  lower(:x_1) 
    ```

+   `ColumnOperators.notlike()`：

    ```py
    >>> print(column("x").notlike("word"))
    x  NOT  LIKE  :x_1 
    ```

+   `ColumnOperators.notilike()`：

    ```py
    >>> print(column("x").notilike("word"))
    lower(x)  NOT  LIKE  lower(:x_1) 
    ```

## 字符串包含：

字符串包含运算符基本上是由 LIKE 和字符串连接运算符组合而成，大多数后端使用`||`，有时候也会使用像`concat()`这样的函数：

+   `ColumnOperators.startswith()`：

    ```py
    >>> print(column("x").startswith("word"))
    x  LIKE  :x_1  ||  '%' 
    ```

+   `ColumnOperators.endswith()`：

    ```py
    >>> print(column("x").endswith("word"))
    x  LIKE  '%'  ||  :x_1 
    ```

+   `ColumnOperators.contains()`：

    ```py
    >>> print(column("x").contains("word"))
    x  LIKE  '%'  ||  :x_1  ||  '%' 
    ```

## 字符串匹配

匹配运算符始终是特定于后端的，并且在不同数据库上可能提供不同的行为和结果：

+   `ColumnOperators.match()`：

    这是一种方言特定的运算符，如果底层数据库支持 MATCH 功能，则使用它：

    ```py
    >>> print(column("x").match("word"))
    x  MATCH  :x_1 
    ```

+   `ColumnOperators.regexp_match()`：

    此运算符是方言特定的。我们可以用 PostgreSQL 方言举例说明：

    ```py
    >>> from sqlalchemy.dialects import postgresql
    >>> print(column("x").regexp_match("word").compile(dialect=postgresql.dialect()))
    x  ~  %(x_1)s 
    ```

    或 MySQL：

    ```py
    >>> from sqlalchemy.dialects import mysql
    >>> print(column("x").regexp_match("word").compile(dialect=mysql.dialect()))
    x  REGEXP  %s 
    ```

## 字符串改动

+   `ColumnOperators.concat()`：

    字符串连接：

    ```py
    >>> print(column("x").concat("some string"))
    x  ||  :x_1 
    ```

    此运算符通过 `ColumnOperators.__add__()` 可用，即，当使用从 `String` 派生的列表达式时，Python `+` 运算符：

    ```py
    >>> print(column("x", String) + "some string")
    x  ||  :x_1 
    ```

    此运算符将产生适当的特定于数据库的构造，例如在 MySQL 上，它历来是 `concat()` SQL 函数：

    ```py
    >>> print((column("x", String) + "some string").compile(dialect=mysql.dialect()))
    concat(x,  %s) 
    ```

+   `ColumnOperators.regexp_replace()`：

    与 `ColumnOperators.regexp()` 互补，这产生了支持它的后端的 REGEXP REPLACE 等效项：

    ```py
    >>> print(column("x").regexp_replace("foo", "bar").compile(dialect=postgresql.dialect()))
    REGEXP_REPLACE(x,  %(x_1)s,  %(x_2)s) 
    ```

+   `ColumnOperators.collate()`：

    产生 COLLATE SQL 运算符，为表达式提供特定的排序规则：

    ```py
    >>> print(
    ...     (column("x").collate("latin1_german2_ci") == "Müller").compile(
    ...         dialect=mysql.dialect()
    ...     )
    ... )
    (x  COLLATE  latin1_german2_ci)  =  %s 
    ```

    要对字面值使用 COLLATE，使用 `literal()` 构造：

    ```py
    >>> from sqlalchemy import literal
    >>> print(
    ...     (literal("Müller").collate("latin1_german2_ci") == column("x")).compile(
    ...         dialect=mysql.dialect()
    ...     )
    ... )
    (%s  COLLATE  latin1_german2_ci)  =  x 
    ```

## 算术运算符

+   `ColumnOperators.__add__()`，`ColumnOperators.__radd__()`（Python “`+`” 运算符）：

    ```py
    >>> print(column("x") + 5)
    x  +  :x_1
    >>> print(5 + column("x"))
    :x_1  +  x 
    ```

    请注意，当表达式的数据类型是 `String` 或类似类型时，`ColumnOperators.__add__()` 运算符会产生 字符串连接：

+   `ColumnOperators.__sub__()`, `ColumnOperators.__rsub__()` （Python “`-`” 运算符）：

    ```py
    >>> print(column("x") - 5)
    x  -  :x_1
    >>> print(5 - column("x"))
    :x_1  -  x 
    ```

+   `ColumnOperators.__mul__()`, `ColumnOperators.__rmul__()` （Python “`*`” 运算符）：

    ```py
    >>> print(column("x") * 5)
    x  *  :x_1
    >>> print(5 * column("x"))
    :x_1  *  x 
    ```

+   `ColumnOperators.__truediv__()`, `ColumnOperators.__rtruediv__()` （Python “`/`” 运算符）。这是 Python 的 `truediv` 运算符，它将确保进行整数真除法：

    ```py
    >>> print(column("x") / 5)
    x  /  CAST(:x_1  AS  NUMERIC)
    >>> print(5 / column("x"))
    :x_1  /  CAST(x  AS  NUMERIC) 
    ```

    从版本 2.0 起更改：Python 的 `/` 运算符现在确保进行整数真除法。

+   `ColumnOperators.__floordiv__()`, `ColumnOperators.__rfloordiv__()` （Python “`//`” 运算符）。这是 Python 的 `floordiv` 运算符，它将确保进行地板除法。对于默认后端以及诸如 PostgreSQL 之类的后端，SQL `/` 运算符通常以这种方式处理整数值：

    ```py
    >>> print(column("x") // 5)
    x  /  :x_1
    >>> print(5 // column("x", Integer))
    :x_1  /  x 
    ```

    对于默认不使用地板除法的后端，或者当与数字值一起使用时，使用 FLOOR() 函数以确保地板除法：

    ```py
    >>> print(column("x") // 5.5)
    FLOOR(x  /  :x_1)
    >>> print(5 // column("x", Numeric))
    FLOOR(:x_1  /  x) 
    ```

    版本 2.0 中新增：支持地板除法

+   `ColumnOperators.__mod__()`, `ColumnOperators.__rmod__()` （Python “`%`” 运算符）：

    ```py
    >>> print(column("x") % 5)
    x  %  :x_1
    >>> print(5 % column("x"))
    :x_1  %  x 
    ```

## 位运算符

位运算符函数提供了对不同后端的位运算符的统一访问，这些后端预期在兼容值（例如整数和位字符串，如 PostgreSQL `BIT` 等）上进行操作。请注意，这些**不是**通用布尔运算符。

版本 2.0.2 中新增：添加了专用于位运算的运算符。

+   `ColumnOperators.bitwise_not()`，`bitwise_not()`。作为一个列级方法可用，针对父对象产生按位 NOT 子句：

    ```py
    >>> print(column("x").bitwise_not())
    ~x
    ```

    这个操作符也作为列表达式级方法可用，将按位 NOT 应用于单个列表达式：

    ```py
    >>> from sqlalchemy import bitwise_not
    >>> print(bitwise_not(column("x")))
    ~x
    ```

+   `ColumnOperators.bitwise_and()` 产生按位与：

    ```py
    >>> print(column("x").bitwise_and(5))
    x & :x_1
    ```

+   `ColumnOperators.bitwise_or()` 产生按位或：

    ```py
    >>> print(column("x").bitwise_or(5))
    x | :x_1
    ```

+   `ColumnOperators.bitwise_xor()` 产生按位异或：

    ```py
    >>> print(column("x").bitwise_xor(5))
    x ^ :x_1
    ```

    对于 PostgreSQL 方言，“#” 被用来表示按位异或；当使用这些后端之一时，它会自动发出：

    ```py
    >>> from sqlalchemy.dialects import postgresql
    >>> print(column("x").bitwise_xor(5).compile(dialect=postgresql.dialect()))
    x # %(x_1)s
    ```

+   `ColumnOperators.bitwise_rshift()`，`ColumnOperators.bitwise_lshift()` 产生按位移动操作符：

    ```py
    >>> print(column("x").bitwise_rshift(5))
    x >> :x_1
    >>> print(column("x").bitwise_lshift(5))
    x << :x_1
    ```

## 使用连接词和否定词

最常见的连接词 “AND”，如果我们重复使用 `Select.where()` 方法，以及类似的方法如 `Update.where()` 和 `Delete.where()`，则会自动应用：

```py
>>> print(
...     select(address_table.c.email_address)
...     .where(user_table.c.name == "squidward")
...     .where(address_table.c.user_id == user_table.c.id)
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  user_account.name  =  :name_1  AND  address.user_id  =  user_account.id 
```

`Select.where()`，`Update.where()` 和 `Delete.where()` 也接受具有相同效果的多个表达式：

```py
>>> print(
...     select(address_table.c.email_address).where(
...         user_table.c.name == "squidward",
...         address_table.c.user_id == user_table.c.id,
...     )
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  user_account.name  =  :name_1  AND  address.user_id  =  user_account.id 
```

“AND” 连接词以及其伴侣 “OR”，都可以直接使用 `and_()` 和 `or_()` 函数获得：

```py
>>> from sqlalchemy import and_, or_
>>> print(
...     select(address_table.c.email_address).where(
...         and_(
...             or_(user_table.c.name == "squidward", user_table.c.name == "sandy"),
...             address_table.c.user_id == user_table.c.id,
...         )
...     )
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  (user_account.name  =  :name_1  OR  user_account.name  =  :name_2)
AND  address.user_id  =  user_account.id 
```

使用 `not_()` 函数可得到否定。这通常会反转布尔表达式中的操作符：

```py
>>> from sqlalchemy import not_
>>> print(not_(column("x") == 5))
x  !=  :x_1 
```

当适用时，也可以应用关键词如 `NOT`：

```py
>>> from sqlalchemy import Boolean
>>> print(not_(column("x", Boolean)))
NOT  x 
```

## 逻辑与运算符

上述逻辑连接函数`and_()`, `or_()`, `not_()` 也可以作为 Python 运算符进行重载：

注意

Python 中的`&`、`|`和`~`运算符在语言中具有高优先级；因此，通常必须为包含表达式的操作数应用括号，如下面的示例所示。

+   `Operators.__and__()` (Python “`&`” operator):

    Python 二进制`&`运算符被重载，以与`and_()` 相同（请注意两个操作数周围的括号）：

    ```py
    >>> print((column("x") == 5) & (column("y") == 10))
    x  =  :x_1  AND  y  =  :y_1 
    ```

+   `Operators.__or__()` (Python “`|`” operator):

    Python 二进制`|`运算符被重载，以与`or_()` 相同（请注意两个操作数周围的括号）：

    ```py
    >>> print((column("x") == 5) | (column("y") == 10))
    x  =  :x_1  OR  y  =  :y_1 
    ```

+   `Operators.__invert__()` (Python “`~`” operator):

    Python 二进制`~`运算符被重载，以与`not_()` 相同，可以反转现有运算符，或将`NOT`关键字应用于整个表达式：

    ```py
    >>> print(~(column("x") == 5))
    x  !=  :x_1
    >>> from sqlalchemy import Boolean
    >>> print(~column("x", Boolean))
    NOT  x 
    ```

## 比较运算符

基本比较适用于许多数据类型，包括数字、字符串、日期等：

+   `ColumnOperators.__eq__()` (Python “`==`” operator):

    ```py
    >>> print(column("x") == 5)
    x  =  :x_1 
    ```

+   `ColumnOperators.__ne__()` (Python “`!=`” operator):

    ```py
    >>> print(column("x") != 5)
    x  !=  :x_1 
    ```

+   `ColumnOperators.__gt__()` (Python “`>`” operator):

    ```py
    >>> print(column("x") > 5)
    x  >  :x_1 
    ```

+   `ColumnOperators.__lt__()` (Python “`<`” operator):

    ```py
    >>> print(column("x") < 5)
    x  <  :x_1 
    ```

+   `ColumnOperators.__ge__()` (Python “`>=`” operator):

    ```py
    >>> print(column("x") >= 5)
    x  >=  :x_1 
    ```

+   `ColumnOperators.__le__()` (Python “`<=`” operator):

    ```py
    >>> print(column("x") <= 5)
    x  <=  :x_1 
    ```

+   `ColumnOperators.between()`:

    ```py
    >>> print(column("x").between(5, 10))
    x  BETWEEN  :x_1  AND  :x_2 
    ```

## IN 比较

SQL 的 IN 操作符在 SQLAlchemy 中是一个单独的主题。由于 IN 操作符通常针对一组固定值使用，SQLAlchemy 的绑定参数强制转换特性利用一种特殊形式的 SQL 编译，在第二步形成最终的绑定参数列表。换句话说，"它就是这样工作"。

### 针对值列表的 IN

通过将值列表传递给 `ColumnOperators.in_()` 方法，通常可以使用 IN：

```py
>>> print(column("x").in_([1, 2, 3]))
x  IN  (__[POSTCOMPILE_x_1]) 
```

特殊的绑定形式 `__[POSTCOMPILE` 在执行时被渲染为单独的参数，如下所示：

```py
>>> stmt = select(User.id).where(User.id.in_([1, 2, 3]))
>>> result = conn.execute(stmt)
SELECT  user_account.id
FROM  user_account
WHERE  user_account.id  IN  (?,  ?,  ?)
[...]  (1,  2,  3) 
```

### 空的 IN 表达式

SQLAlchemy 通过渲染一个特定于后端的子查询返回没有行的有效数学结果，从而为空的 IN 表达式生成一个中间值。换句话说，"它就是这样工作"：

```py
>>> stmt = select(User.id).where(User.id.in_([]))
>>> result = conn.execute(stmt)
SELECT  user_account.id
FROM  user_account
WHERE  user_account.id  IN  (SELECT  1  FROM  (SELECT  1)  WHERE  1!=1)
[...]  () 
```

上述的“空集合”子查询正确地泛化，并且也是以 IN 操作符的形式呈现的，该操作符保持不变。

### NOT IN

“NOT IN” 可通过 `ColumnOperators.not_in()` 运算符使用：

```py
>>> print(column("x").not_in([1, 2, 3]))
(x  NOT  IN  (__[POSTCOMPILE_x_1])) 
```

通常通过使用 `~` 运算符进行否定更容易：

```py
>>> print(~column("x").in_([1, 2, 3]))
(x  NOT  IN  (__[POSTCOMPILE_x_1])) 
```

### 元组 IN 表达式

使用 IN 与元组比较是常见的，除了其他用例外，它适应了将行匹配到一组潜在的复合主键值的情况。`tuple_()` 构造提供了元组比较的基本构建块。然后 `Tuple.in_()` 运算符接收一个元组列表：

```py
>>> from sqlalchemy import tuple_
>>> tup = tuple_(column("x", Integer), column("y", Integer))
>>> expr = tup.in_([(1, 2), (3, 4)])
>>> print(expr)
(x,  y)  IN  (__[POSTCOMPILE_param_1]) 
```

为了说明渲染的参数： 

```py
>>> tup = tuple_(User.id, Address.id)
>>> stmt = select(User.name).join(Address).where(tup.in_([(1, 1), (2, 2)]))
>>> conn.execute(stmt).all()
SELECT  user_account.name
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  (user_account.id,  address.id)  IN  (VALUES  (?,  ?),  (?,  ?))
[...]  (1,  1,  2,  2)
[('spongebob',), ('sandy',)]
```

### 子查询 IN

最后，`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 运算符与子查询一起使用。该形式允许直接传递 `Select` 构造，无需明确转换为命名子查询：

```py
>>> print(column("x").in_(select(user_table.c.id)))
x  IN  (SELECT  user_account.id
FROM  user_account) 
```

元组按预期工作：

```py
>>> print(
...     tuple_(column("x"), column("y")).in_(
...         select(user_table.c.id, address_table.c.id).join(address_table)
...     )
... )
(x,  y)  IN  (SELECT  user_account.id,  address.id
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id) 
```

### 针对值列表的 IN

通过将值列表传递给 `ColumnOperators.in_()` 方法，通常可以使用 IN：

```py
>>> print(column("x").in_([1, 2, 3]))
x  IN  (__[POSTCOMPILE_x_1]) 
```

特殊的绑定形式 `__[POSTCOMPILE` 在执行时被渲染为单独的参数，如下所示：

```py
>>> stmt = select(User.id).where(User.id.in_([1, 2, 3]))
>>> result = conn.execute(stmt)
SELECT  user_account.id
FROM  user_account
WHERE  user_account.id  IN  (?,  ?,  ?)
[...]  (1,  2,  3) 
```

### 空的 IN 表达式

SQLAlchemy 通过渲染一个特定于后端的子查询来为空 IN 表达式生成一个数学上有效的结果，该子查询不返回任何行。换句话说，“它只是有效地工作”：

```py
>>> stmt = select(User.id).where(User.id.in_([]))
>>> result = conn.execute(stmt)
SELECT  user_account.id
FROM  user_account
WHERE  user_account.id  IN  (SELECT  1  FROM  (SELECT  1)  WHERE  1!=1)
[...]  () 
```

上述“空集”子查询正确地进行了泛化，并且也以 IN 操作符的形式呈现在原位。

### NOT IN

“NOT IN” 可通过`ColumnOperators.not_in()` 操作符获得：

```py
>>> print(column("x").not_in([1, 2, 3]))
(x  NOT  IN  (__[POSTCOMPILE_x_1])) 
```

通常通过使用`~`操作符进行否定更容易获得：

```py
>>> print(~column("x").in_([1, 2, 3]))
(x  NOT  IN  (__[POSTCOMPILE_x_1])) 
```

### 元组 IN 表达式

使用 IN 进行元组与元组的比较很常见，因为除了其他用例外，还适用于将匹配行与一组潜在的复合主键值进行匹配的情况。`tuple_()` 构造提供了元组比较的基本构建块。然后`Tuple.in_()` 操作符接收一个元组列表：

```py
>>> from sqlalchemy import tuple_
>>> tup = tuple_(column("x", Integer), column("y", Integer))
>>> expr = tup.in_([(1, 2), (3, 4)])
>>> print(expr)
(x,  y)  IN  (__[POSTCOMPILE_param_1]) 
```

渲染参数的示例：

```py
>>> tup = tuple_(User.id, Address.id)
>>> stmt = select(User.name).join(Address).where(tup.in_([(1, 1), (2, 2)]))
>>> conn.execute(stmt).all()
SELECT  user_account.name
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  (user_account.id,  address.id)  IN  (VALUES  (?,  ?),  (?,  ?))
[...]  (1,  1,  2,  2)
[('spongebob',), ('sandy',)]
```

### 子查询 IN

最后，`ColumnOperators.in_()` 和 `ColumnOperators.not_in()` 操作符与子查询一起使用。该形式提供了直接传入`Select` 构造的方式，无需明确转换为命名子查询：

```py
>>> print(column("x").in_(select(user_table.c.id)))
x  IN  (SELECT  user_account.id
FROM  user_account) 
```

元组按预期工作：

```py
>>> print(
...     tuple_(column("x"), column("y")).in_(
...         select(user_table.c.id, address_table.c.id).join(address_table)
...     )
... )
(x,  y)  IN  (SELECT  user_account.id,  address.id
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id) 
```

## 身份比较

这些操作符涉及测试特殊的 SQL 值，如`NULL`，一些数据库支持的布尔常量，如`true`或`false`：

+   `ColumnOperators.is_()`:

    该操作符将提供“x IS y”的确切 SQL，通常表示为“<expr> IS NULL”。使用常规的 Python `None` 最容易获得`NULL` 常量：

    ```py
    >>> print(column("x").is_(None))
    x  IS  NULL 
    ```

    如果需要，SQL NULL 也可以明确使用`null()` 构造获得：

    ```py
    >>> from sqlalchemy import null
    >>> print(column("x").is_(null()))
    x  IS  NULL 
    ```

    当使用`ColumnOperators.__eq__()` 重载的操作符，即`==`，与`None`或`null()`值一起使用时，`ColumnOperators.is_()` 操作符会自动调用。因此，通常不需要显式使用`ColumnOperators.is_()`，特别是在与动态值一起使用时：

    ```py
    >>> a = None
    >>> print(column("x") == a)
    x  IS  NULL 
    ```

    请注意，Python 的 `is` 运算符**未被重载**。即使 Python 提供了像 `==` 和 `!=` 这样的运算符重载钩子，它也**没有**提供任何重新定义 `is` 的方法。

+   `ColumnOperators.is_not()`:

    类似于 `ColumnOperators.is_()`，生成“IS NOT”：

    ```py
    >>> print(column("x").is_not(None))
    x  IS  NOT  NULL 
    ```

    同样等价于 `!= None`：

    ```py
    >>> print(column("x") != None)
    x  IS  NOT  NULL 
    ```

+   `ColumnOperators.is_distinct_from()`:

    生成 SQL IS DISTINCT FROM：

    ```py
    >>> print(column("x").is_distinct_from("some value"))
    x  IS  DISTINCT  FROM  :x_1 
    ```

+   `ColumnOperators.isnot_distinct_from()`:

    生成 SQL IS NOT DISTINCT FROM：

    ```py
    >>> print(column("x").isnot_distinct_from("some value"))
    x  IS  NOT  DISTINCT  FROM  :x_1 
    ```

## 字符串比较

+   `ColumnOperators.like()`:

    ```py
    >>> print(column("x").like("word"))
    x  LIKE  :x_1 
    ```

+   `ColumnOperators.ilike()`:

    不区分大小写的 LIKE 在通用后端上使用 SQL `lower()` 函数。在 PostgreSQL 后端上，它将使用 `ILIKE`：

    ```py
    >>> print(column("x").ilike("word"))
    lower(x)  LIKE  lower(:x_1) 
    ```

+   `ColumnOperators.notlike()`:

    ```py
    >>> print(column("x").notlike("word"))
    x  NOT  LIKE  :x_1 
    ```

+   `ColumnOperators.notilike()`:

    ```py
    >>> print(column("x").notilike("word"))
    lower(x)  NOT  LIKE  lower(:x_1) 
    ```

## 字符串包含

字符串包含运算符基本上是由 LIKE 和字符串连接运算符构建的，后端上为 `||` 或者有时是像 `concat()` 这样的函数：

+   `ColumnOperators.startswith()`:

    ```py
    >>> print(column("x").startswith("word"))
    x  LIKE  :x_1  ||  '%' 
    ```

+   `ColumnOperators.endswith()`:

    ```py
    >>> print(column("x").endswith("word"))
    x  LIKE  '%'  ||  :x_1 
    ```

+   `ColumnOperators.contains()`:

    ```py
    >>> print(column("x").contains("word"))
    x  LIKE  '%'  ||  :x_1  ||  '%' 
    ```

## 字符串匹配

匹配运算符始终是特定于后端的，可能在不同的数据库上提供不同的行为和结果：

+   `ColumnOperators.match()`:

    这是一个特定于方言的运算符，如果底层数据库支持 MATCH 特性，则使用它：

    ```py
    >>> print(column("x").match("word"))
    x  MATCH  :x_1 
    ```

+   `ColumnOperators.regexp_match()`:

    此运算符是方言特定的。我们可以用 PostgreSQL 方言举例说明：

    ```py
    >>> from sqlalchemy.dialects import postgresql
    >>> print(column("x").regexp_match("word").compile(dialect=postgresql.dialect()))
    x  ~  %(x_1)s 
    ```

    或者 MySQL：

    ```py
    >>> from sqlalchemy.dialects import mysql
    >>> print(column("x").regexp_match("word").compile(dialect=mysql.dialect()))
    x  REGEXP  %s 
    ```

## 字符串修改

+   `ColumnOperators.concat()`：

    字符串连接：

    ```py
    >>> print(column("x").concat("some string"))
    x  ||  :x_1 
    ```

    当使用从 `String` 派生的列表达式时，此运算符可通过 `ColumnOperators.__add__()` 提供，即 Python 的 `+` 运算符：

    ```py
    >>> print(column("x", String) + "some string")
    x  ||  :x_1 
    ```

    该运算符将产生适当的数据库特定结构，例如在 MySQL 上，它历史上一直是 `concat()` SQL 函数：

    ```py
    >>> print((column("x", String) + "some string").compile(dialect=mysql.dialect()))
    concat(x,  %s) 
    ```

+   `ColumnOperators.regexp_replace()`：

    与 `ColumnOperators.regexp()` 互补，对于支持的后端，会产生等效的 REGEXP REPLACE：

    ```py
    >>> print(column("x").regexp_replace("foo", "bar").compile(dialect=postgresql.dialect()))
    REGEXP_REPLACE(x,  %(x_1)s,  %(x_2)s) 
    ```

+   `ColumnOperators.collate()`：

    生成 COLLATE SQL 运算符，可在表达式时间为特定排序提供支持：

    ```py
    >>> print(
    ...     (column("x").collate("latin1_german2_ci") == "Müller").compile(
    ...         dialect=mysql.dialect()
    ...     )
    ... )
    (x  COLLATE  latin1_german2_ci)  =  %s 
    ```

    若要针对字面值使用 COLLATE，请使用 `literal()` 构造：

    ```py
    >>> from sqlalchemy import literal
    >>> print(
    ...     (literal("Müller").collate("latin1_german2_ci") == column("x")).compile(
    ...         dialect=mysql.dialect()
    ...     )
    ... )
    (%s  COLLATE  latin1_german2_ci)  =  x 
    ```

## 算术运算符

+   `ColumnOperators.__add__()`，`ColumnOperators.__radd__()`（Python “`+`” 运算符）：

    ```py
    >>> print(column("x") + 5)
    x  +  :x_1
    >>> print(5 + column("x"))
    :x_1  +  x 
    ```

    请注意，当表达式的数据类型是 `String` 或类似时，`ColumnOperators.__add__()` 运算符会产生字符串连接而不是 字符串连接：

+   `ColumnOperators.__sub__()`，`ColumnOperators.__rsub__()`（Python “`-`” 运算符）：

    ```py
    >>> print(column("x") - 5)
    x  -  :x_1
    >>> print(5 - column("x"))
    :x_1  -  x 
    ```

+   `ColumnOperators.__mul__()`，`ColumnOperators.__rmul__()`（Python “`*`” 运算符）：

    ```py
    >>> print(column("x") * 5)
    x  *  :x_1
    >>> print(5 * column("x"))
    :x_1  *  x 
    ```

+   `ColumnOperators.__truediv__()`, `ColumnOperators.__rtruediv__()`（Python “`/`” 运算符）。这是 Python 的 `truediv` 运算符，它确保进行整数真除法：

    ```py
    >>> print(column("x") / 5)
    x  /  CAST(:x_1  AS  NUMERIC)
    >>> print(5 / column("x"))
    :x_1  /  CAST(x  AS  NUMERIC) 
    ```

    在 2.0 版本中更改：Python `/` 运算符现在确保进行整数真除法

+   `ColumnOperators.__floordiv__()`, `ColumnOperators.__rfloordiv__()`（Python “`//`” 运算符）。这是 Python 的 `floordiv` 运算符，它确保进行 floor division。对于默认后端以及诸如 PostgreSQL 这样的后端，SQL `/` 运算符通常以这种方式对整数值进行操作：

    ```py
    >>> print(column("x") // 5)
    x  /  :x_1
    >>> print(5 // column("x", Integer))
    :x_1  /  x 
    ```

    对于默认不使用 floor division 的后端，或者与数值一起使用时，使用 FLOOR() 函数来确保进行 floor division：

    ```py
    >>> print(column("x") // 5.5)
    FLOOR(x  /  :x_1)
    >>> print(5 // column("x", Numeric))
    FLOOR(:x_1  /  x) 
    ```

    2.0 版本中的新功能：支持 FLOOR division

+   `ColumnOperators.__mod__()`, `ColumnOperators.__rmod__()`（Python “`%`” 运算符）：

    ```py
    >>> print(column("x") % 5)
    x  %  :x_1
    >>> print(5 % column("x"))
    :x_1  %  x 
    ```

## 按位运算符

按位运算符函数提供了对不同后端的位运算符的统一访问，这些后端预计将对兼容的值（例如整数和位字符串，例如 PostgreSQL `BIT` 和类似的）进行操作。请注意，这些**不是**通用布尔运算符。

2.0.2 版本中的新功能：添加了专用位运算符。

+   `ColumnOperators.bitwise_not()`, `bitwise_not()`。作为列级方法可用，对父对象生成按位 NOT 子句：

    ```py
    >>> print(column("x").bitwise_not())
    ~x
    ```

    这个运算符也作为列表达式级别的方法可用，对单个列表达式应用按位 NOT：

    ```py
    >>> from sqlalchemy import bitwise_not
    >>> print(bitwise_not(column("x")))
    ~x
    ```

+   `ColumnOperators.bitwise_and()` 产生按位与：

    ```py
    >>> print(column("x").bitwise_and(5))
    x & :x_1
    ```

+   `ColumnOperators.bitwise_or()` 产生按位 OR：

    ```py
    >>> print(column("x").bitwise_or(5))
    x | :x_1
    ```

+   `ColumnOperators.bitwise_xor()` 产生按位异或操作符：

    ```py
    >>> print(column("x").bitwise_xor(5))
    x ^ :x_1
    ```

    对于 PostgreSQL 方言，“#” 用于表示按位异或；在使用其中一个后端时会自动发出：

    ```py
    >>> from sqlalchemy.dialects import postgresql
    >>> print(column("x").bitwise_xor(5).compile(dialect=postgresql.dialect()))
    x # %(x_1)s
    ```

+   `ColumnOperators.bitwise_rshift()`，`ColumnOperators.bitwise_lshift()` 产生按位移动操作符：

    ```py
    >>> print(column("x").bitwise_rshift(5))
    x >> :x_1
    >>> print(column("x").bitwise_lshift(5))
    x << :x_1
    ```

## 使用连接词和否定

如果我们反复使用 `Select.where()` 方法，以及类似的方法如 `Update.where()` 和 `Delete.where()`，最常见的连接词“AND”会自动应用：

```py
>>> print(
...     select(address_table.c.email_address)
...     .where(user_table.c.name == "squidward")
...     .where(address_table.c.user_id == user_table.c.id)
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  user_account.name  =  :name_1  AND  address.user_id  =  user_account.id 
```

`Select.where()`、`Update.where()` 和 `Delete.where()` 也接受具有相同效果的多个表达式：

```py
>>> print(
...     select(address_table.c.email_address).where(
...         user_table.c.name == "squidward",
...         address_table.c.user_id == user_table.c.id,
...     )
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  user_account.name  =  :name_1  AND  address.user_id  =  user_account.id 
```

“AND” 连接词以及其伴侣“OR”可以直接使用 `and_()` 和 `or_()` 函数：

```py
>>> from sqlalchemy import and_, or_
>>> print(
...     select(address_table.c.email_address).where(
...         and_(
...             or_(user_table.c.name == "squidward", user_table.c.name == "sandy"),
...             address_table.c.user_id == user_table.c.id,
...         )
...     )
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  (user_account.name  =  :name_1  OR  user_account.name  =  :name_2)
AND  address.user_id  =  user_account.id 
```

使用 `not_()` 函数可进行否定。这通常会反转布尔表达式中的运算符：

```py
>>> from sqlalchemy import not_
>>> print(not_(column("x") == 5))
x  !=  :x_1 
```

在适当时还可以应��关键字，如 `NOT`：

```py
>>> from sqlalchemy import Boolean
>>> print(not_(column("x", Boolean)))
NOT  x 
```

## 连接操作符

上述连接函数 `and_()`、`or_()`、`not_()` 也可作为 Python 中的重载运算符使用：

注意

Python 中的 `&`、`|` 和 `~` 操作符在语言中具有高优先级；因此，通常需要为包含表达式的操作数应用括号，如下面的示例所示。

+   `Operators.__and__()`（Python “`&`” 操作符）：

    Python 的二进制`&`运算符被重载，以与`and_()`（注意两个操作数周围的括号）行为相同：

    ```py
    >>> print((column("x") == 5) & (column("y") == 10))
    x  =  :x_1  AND  y  =  :y_1 
    ```

+   `Operators.__or__()` (Python “`|`” operator):

    Python 的二进制`|`运算符被重载，以与`or_()`（注意两个操作数周围的括号）行为相同：

    ```py
    >>> print((column("x") == 5) | (column("y") == 10))
    x  =  :x_1  OR  y  =  :y_1 
    ```

+   `Operators.__invert__()` (Python “`~`” operator):

    Python 的二进制`~`运算符被重载，以与`not_()`行为相同，可以反转现有运算符，或将`NOT`关键字应用于整个表达式：

    ```py
    >>> print(~(column("x") == 5))
    x  !=  :x_1
    >>> from sqlalchemy import Boolean
    >>> print(~column("x", Boolean))
    NOT  x 
    ```
