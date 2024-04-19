# 处理事务和 DBAPI

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/dbapi_transactions.html`](https://docs.sqlalchemy.org/en/20/tutorial/dbapi_transactions.html)

准备好的`Engine` 对象后，我们现在可以继续深入探讨 `Engine` 的基本操作及其主要交互端点，即 `Connection` 和 `Result`。我们还将介绍 ORM 对这些对象的门面，称为 `Session`。

**ORM 读者注意**

使用 ORM 时，`Engine` 由另一个称为 `Session` 的对象管理。现代 SQLAlchemy 中的 `Session` 强调的是一种事务性和 SQL 执行模式，它与下面讨论的 `Connection` 的模式基本相同，因此，虽然本小节是以核心为中心的，但这里的所有概念基本上都与 ORM 使用相关，并且建议所有 ORM 学习者阅读。`Connection` 使用的执行模式将在本节末尾与 `Session` 的模式进行对比。

由于我们尚未介绍 SQLAlchemy 表达语言，这是 SQLAlchemy 的主要特性，我们将利用该软件包中的一个简单构造，称为`text()` 构造，它允许我们以**文本 SQL**的形式编写 SQL 语句。请放心，在日常使用 SQLAlchemy 时，文本 SQL 绝大多数情况下都是例外而不是规则，即使如此，它仍然始终完全可用。

## 获取连接

`Engine`对象从用户角度看唯一的目的是提供称为`Connection`的数据库连接单元。当直接使用核心时，与数据库的所有交互都是通过`Connection`对象完成的。由于`Connection`代表着针对数据库的一个开放资源，我们希望始终将对此对象的使用范围限制在特定的上下文中，而使用 Python 上下文管理器形式，也称为[with 语句](https://docs.python.org/3/reference/compound_stmts.html#with)是这样做的最佳方式。下面我们使用文本 SQL 语句说明“Hello World”。文本 SQL 使用一个叫做`text()`的构造发出，稍后将更详细地讨论：

```py
>>> from sqlalchemy import text

>>> with engine.connect() as conn:
...     result = conn.execute(text("select 'hello world'"))
...     print(result.all())
BEGIN  (implicit)
select  'hello world'
[...]  ()
[('hello world',)]
ROLLBACK 
```

在上面的示例中，为数据库连接提供了上下文管理器，并将操作放在事务内。Python DBAPI 的默认行为包括事务始终处于进行中；当连接的范围被释放时，会发出 ROLLBACK 以结束事务。事务**不会自动提交**；当我们想要提交数据时，通常需要调用`Connection.commit()`，我们将在下一节中看到。

提示

“自动提交”模式适用于特殊情况。章节设置事务隔离级别，包括 DBAPI 自动提交讨论了这一点。

我们的 SELECT 的结果也以一个叫做`Result`的对象返回，稍后将讨论，但是暂时我们将添加这样一句，最好确保在“connect”块内消耗此对象，并且不要在连接范围之外传递。## 提交更改

我们刚刚学到 DBAPI 连接是非自动提交的。如果我们想提交一些数据怎么办？我们可以修改我们上面的示例来创建一个表并插入一些数据，然后使用`Connection.commit()`方法在我们获取`Connection`对象的块内调用进行事务提交：

```py
# "commit as you go"
>>> with engine.connect() as conn:
...     conn.execute(text("CREATE TABLE some_table (x int, y int)"))
...     conn.execute(
...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
...         [{"x": 1, "y": 1}, {"x": 2, "y": 4}],
...     )
...     conn.commit()
BEGIN  (implicit)
CREATE  TABLE  some_table  (x  int,  y  int)
[...]  ()
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
INSERT  INTO  some_table  (x,  y)  VALUES  (?,  ?)
[...]  [(1,  1),  (2,  4)]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

在上面，我们发出了两个通常是事务性的 SQL 语句，“CREATE TABLE”语句[[1]](#id2)和一个参数化的“INSERT”语句（上面的参数化语法在发送多个参数中讨论）。由于我们希望我们所做的工作在我们的块内被提交，我们调用`Connection.commit()`方法来提交事务。在块内调用此方法后，我们可以继续运行更多的 SQL 语句，如果选择的话，我们可以再次调用`Connection.commit()`来进行后续语句的提交。SQLAlchemy 将这种风格称为**边提交边进行**。

还有另一种提交数据的风格，即我们可以事先将我们的“connect”块声明为事务块。在这种操作模式下，我们使用`Engine.begin()`方法来获取连接，而不是使用`Engine.connect()`方法。这种方法既管理了`Connection`的范围，也在事务结束时包含了 COMMIT，假设块成功，或者在出现异常时回滚。这种风格被称为**一次性开始**：

```py
# "begin once"
>>> with engine.begin() as conn:
...     conn.execute(
...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
...         [{"x": 6, "y": 8}, {"x": 9, "y": 10}],
...     )
BEGIN  (implicit)
INSERT  INTO  some_table  (x,  y)  VALUES  (?,  ?)
[...]  [(6,  8),  (9,  10)]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

“一次性开始”风格通常更受青睐，因为它更简洁，并且事先指示整个块的意图。然而，在本教程中，我们通常会使用“边提交边进行”风格，因为这样更灵活，适合演示目的。## 语句执行的基础知识

我们已经看到了一些例子，针对数据库运行 SQL 语句，利用了一个叫做`Connection.execute()`的方法，结合一个叫做`text()`的对象，并返回一个叫做`Result`的对象。在本节中，我们将更详细地说明这些组件的机制和交互。

当使用 `Session.execute()` 方法时，本节大部分内容同样适用于现代 ORM 的使用，其工作原理与 `Connection.execute()` 非常相似，包括 ORM 结果行使用的是与 Core 相同的 `Result` 接口来传递。

### 获取行

我们首先通过利用之前插入的行来更仔细地说明 `Result` 对象，运行一个对我们创建的表进行文本选择的 SELECT 语句：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(text("SELECT x, y FROM some_table"))
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table
[...]  ()
x: 1  y: 1
x: 2  y: 4
x: 6  y: 8
x: 9  y: 10
ROLLBACK 
```

上面，我们执行的“SELECT”字符串选择了我们表中的所有行。返回的对象称为 `Result`，表示结果行的可迭代对象。

`Result` 有许多用于获取和转换行的方法，例如之前介绍的 `Result.all()` 方法，它返回所有 `Row` 对象的列表。它还实现了 Python 迭代器接口，以便我们可以直接对 `Row` 对象的集合进行迭代。

`Row` 对象本身旨在像 Python 的[命名元组](https://docs.python.org/3/library/collections.html#collections.namedtuple)一样运作。下面我们展示了访问行的各种方式。

+   **元组赋值** - 这是最具 Python 风格的方式，即按位置分配变量，就像它们被接收到的那样：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for x, y in result:
        ...
    ```

+   **整数索引** - 元组是 Python 序列，因此也可以使用常规整数访问：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for row in result:
        x = row[0]
    ```

+   **属性名称** - 由于这些是 Python 命名元组，元组具有与每个列的名称匹配的动态属性名称。这些名称通常是 SQL 语句分配给每行中的列的名称。虽然它们通常是相当可预测的，并且也可以由标签控制，在 less 定义的情况下，它们可能受到特定于数据库的行为的影响：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for row in result:
        y = row.y

        # illustrate use with Python f-strings
        print(f"Row: {row.x}  {y}")
    ```

+   **映射访问** - 为了将行作为 Python **映射**对象接收，这本质上是 Python 对普通`dict`对象的只读版本的接口，`Result`可以通过`Result.mappings()`修改器转换为`MappingResult`对象；这是一个生成类似于字典的`RowMapping`对象而不是`Row`对象的结果对象：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for dict_row in result.mappings():
        x = dict_row["x"]
        y = dict_row["y"]
    ```  ### 发送参数

SQL 语句通常会伴随着要与语句本身一起传递的数据，就像我们之前在 INSERT 示例中看到的那样。因此，`Connection.execute()`方法也接受参数，这些参数被称为绑定参数。一个基本的例子可能是，如果我们想要将 SELECT 语句限制为只选择满足某些条件的行，比如“y”值大于通过函数传递的某个值的行。

为了达到这样的效果，使得 SQL 语句保持固定，同时驱动程序可以正确地清理值，我们在语句中添加了一个名为“y”的 WHERE 条件；`text()`构造函数使用冒号格式“`:y`”接受这些参数。然后，“`:y`”的实际值作为字典形式的第二个参数传递给`Connection.execute()`：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(text("SELECT x, y FROM some_table WHERE y > :y"), {"y": 2})
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table  WHERE  y  >  ?
[...]  (2,)
x: 2  y: 4
x: 6  y: 8
x: 9  y: 10
ROLLBACK 
```

在记录的 SQL 输出中，我们可以看到绑定参数`:y`在发送到 SQLite 数据库时被转换成了一个问号。这是因为 SQLite 数据库驱动程序使用了一种称为“问号参数风格”的格式，这是 DBAPI 规范允许的六种不同格式之一。SQLAlchemy 将这些格式抽象为一种，即使用冒号的“命名”格式。  ### 发送多个参数

在提交更改的示例中，我们执行了一个 INSERT 语句，似乎我们能够一次将多行插入到数据库中。对于 DML 语句，如“INSERT”，“UPDATE”和“DELETE”，我们可以通过传递一个字典列表而不是单个字典给`Connection.execute()`方法，从而发送**多个参数集**，这表明单个 SQL 语句应该被多次调用，每次为一个参数集。这种执行方式称为 executemany：

```py
>>> with engine.connect() as conn:
...     conn.execute(
...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
...         [{"x": 11, "y": 12}, {"x": 13, "y": 14}],
...     )
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  some_table  (x,  y)  VALUES  (?,  ?)
[...]  [(11,  12),  (13,  14)]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

以上操作等同于针对每个参数集一次运行给定的 INSERT 语句，但该操作将被优化以在许多行上获得更好的性能。

“execute”和“executemany”之间的一个关键行为差异是，后者不支持返回结果行，即使语句包含 RETURNING 子句也是如此。唯一的例外是使用 Core `insert()`构造时，稍后在本教程的使用 INSERT 语句中介绍，该构造还使用`Insert.returning()`方法指示 RETURNING。在这种情况下，SQLAlchemy 利用特殊逻辑重新组织 INSERT 语句，以便在支持 RETURNING 的同时可以为多行调用它。

另请参阅

executemany - 在 Glossary 中，描述了 DBAPI 级别的[cursor.executemany()](https://peps.python.org/pep-0249/#executemany)方法，用于大多数“executemany”执行。

INSERT 语句的“插入多个值”行为 - 在引擎和连接中，描述了`Insert.returning()`使用的专门逻辑，以便通过“executemany”执行传递结果集。## 使用 ORM 会话执行

正如前面提到的，上面的大多数模式和示例也适用于与 ORM 一起使用，因此我们在这里介绍这种用法，以便在教程进行时，我们能够将每个模式以 Core 和 ORM 一起使用的方式进行说明。

当使用 ORM 时，与数据库交互的基本事务对象称为`Session`。在现代 SQLAlchemy 中，此对象的使用方式与`Connection`非常相似，实际上，当使用`Session`时，它会内部引用一个`Connection`，然后使用它来发出 SQL。

当与非 ORM 构造一起使用`Session`时，它会通过我们给它的 SQL 语句并且通常不会与`Connection`做什么不同的事情，因此我们可以在这里以我们已经学到的简单文本 SQL 操作来说明它。

`Session`具有几种不同的创建模式，但在这里我们将展示最基本的一种，它与`Connection`的使用方式完全一致，即在上下文管理器中构造它：

```py
>>> from sqlalchemy.orm import Session

>>> stmt = text("SELECT x, y FROM some_table WHERE y > :y ORDER BY x, y")
>>> with Session(engine) as session:
...     result = session.execute(stmt, {"y": 6})
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table  WHERE  y  >  ?  ORDER  BY  x,  y
[...]  (6,)
x: 6  y: 8
x: 9  y: 10
x: 11  y: 12
x: 13  y: 14
ROLLBACK 
```

上面的示例可以与发送参数中的示例进行比较 - 我们直接将对`with engine.connect() as conn`的调用替换为`with Session(engine) as session`，然后像使用`Connection.execute()`方法一样使用`Session.execute()`方法。

与`Connection`类似，`Session`通过使用`Session.commit()`方法具有“边提交边执行”的行为，如下所示，使用文本 UPDATE 语句来修改部分数据：

```py
>>> with Session(engine) as session:
...     result = session.execute(
...         text("UPDATE some_table SET y=:y WHERE x=:x"),
...         [{"x": 9, "y": 11}, {"x": 13, "y": 15}],
...     )
...     session.commit()
BEGIN  (implicit)
UPDATE  some_table  SET  y=?  WHERE  x=?
[...]  [(11,  9),  (15,  13)]
COMMIT 
```

在上面的示例中，我们使用绑定参数，“executemany”样式的执行来调用 UPDATE 语句，在发送多个参数中介绍了这种方式，并以“边提交边执行”的方式结束了该块。

提示

`Session`在事务结束后实际上不会保留`Connection`对象。它会在下一次执行数据库 SQL 时从`Engine`中获取一个新的`Connection`。

`Session`显然有比那更多的技巧，但是了解它有一个`Session.execute()`方法，与`Connection.execute()`的使用方式相同，将使我们能够开始后面的示例。

参见

使用会话的基础知识 - 提供了与`Session`对象的基本创建和使用模式。## 获取连接

从用户的角度来看，`Engine`对象的唯一目的是提供与数据库的连接单元`Connection`。当直接使用核心时，与数据库的所有交互都是通过`Connection`对象完成的。由于`Connection`表示对数据库的打开资源，我们希望始终将此对象的使用范围限制在特定的上下文中，而最好的方法是使用 Python 上下文管理器形式，也称为[with 语句](https://docs.python.org/3/reference/compound_stmts.html#with)。下面我们以一个使用文本 SQL 语句的“Hello World”为例进行说明。文本 SQL 是使用称为`text()`的构造发出的，稍后将对其进行更详细的讨论。

```py
>>> from sqlalchemy import text

>>> with engine.connect() as conn:
...     result = conn.execute(text("select 'hello world'"))
...     print(result.all())
BEGIN  (implicit)
select  'hello world'
[...]  ()
[('hello world',)]
ROLLBACK 
```

在上面的示例中，为数据库连接提供了上下文管理器，并将操作框定在事务内。Python DBAPI 的默认行为包括事务始终在进行中；当连接的范围被释放时，会发出 ROLLBACK 来结束事务。事务**不会自动提交**；当我们想要提交数据时，通常需要调用`Connection.commit()`，正如我们将在下一节中看到的。

提示

“自动提交”模式适用于特殊情况。本节设置事务隔离级别，包括 DBAPI 自动提交对此进行了讨论。

我们的 SELECT 的结果也以一个叫做`Result`的对象返回，稍后将对其进行讨论，但目前我们将补充说明最好确保在“connect”块内消耗此对象，并且不要在连接范围之外传递。

## 提交更改

我们刚刚学到 DBAPI 连接是非自动提交的。如果我们想提交一些数据怎么办？我们可以修改上面的示例来创建一个表并插入一些数据，然后使用`Connection.commit()`方法来提交事务，在我们获取`Connection`对象的块内调用：

```py
# "commit as you go"
>>> with engine.connect() as conn:
...     conn.execute(text("CREATE TABLE some_table (x int, y int)"))
...     conn.execute(
...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
...         [{"x": 1, "y": 1}, {"x": 2, "y": 4}],
...     )
...     conn.commit()
BEGIN  (implicit)
CREATE  TABLE  some_table  (x  int,  y  int)
[...]  ()
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
INSERT  INTO  some_table  (x,  y)  VALUES  (?,  ?)
[...]  [(1,  1),  (2,  4)]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

在上面，我们发出了两个通常是事务性的 SQL 语句，一个是“CREATE TABLE”语句[[1]](#id2)，另一个是参数化的“INSERT”语句（上面的参数化语法在发送多个参数一节中讨论）。由于我们希望我们所做的工作在我们的块内被提交，我们调用`Connection.commit()`方法来提交事务。在我们在块内调用这个方法之后，我们可以继续运行更多的 SQL 语句，如果选择的话，我们可以再次调用`Connection.commit()`来提交后续的语句。SQLAlchemy 将这种风格称为**边做边提交**。

还有另一种提交数据的风格，即我们可以事先声明我们的“connect”块是一个事务块。对于这种操作模式，我们使用`Engine.begin()`方法来获取连接，而不是`Engine.connect()`方法。该方法将管理`Connection`的范围，并在成功块的情况下封闭事务内的所有内容，或者在出现异常时回滚。这种风格称为**一次性开始**：

```py
# "begin once"
>>> with engine.begin() as conn:
...     conn.execute(
...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
...         [{"x": 6, "y": 8}, {"x": 9, "y": 10}],
...     )
BEGIN  (implicit)
INSERT  INTO  some_table  (x,  y)  VALUES  (?,  ?)
[...]  [(6,  8),  (9,  10)]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

“一次性开始”风格通常更受欢迎，因为它更简洁，并且在前面指示了整个块的意图。然而，在本教程中，我们通常会使用“随时提交”风格，因为它对于演示目的更灵活。

## 语句执行基础

我们已经看到了一些示例，通过一种称为`Connection.execute()`的方法来执行 SQL 语句，结合一个称为`text()`的对象，并返回一个称为`Result`的对象。在本节中，我们将更详细地说明这些组件的机制和交互。

本节中的大部分内容同样适用于在使用`Session.execute()`方法时的现代 ORM 使用，该方法与`Connection.execute()`非常相似，包括 ORM 结果行都使用与 Core 相同的`Result`接口传递。

### 获取行

我们将首先通过使用先前插入的行来更详细地说明`Result`对象，对我们创建的表运行一个文本 SELECT 语句：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(text("SELECT x, y FROM some_table"))
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table
[...]  ()
x: 1  y: 1
x: 2  y: 4
x: 6  y: 8
x: 9  y: 10
ROLLBACK 
```

在上面，我们执行的“SELECT”字符串选择了我们表中的所有行。返回的对象称为`Result`，表示一个结果行的可迭代对象。

`Result` 有很多用于获取和转换行的方法，例如之前示例中说明的 `Result.all()` 方法，它返回所有 `Row` 对象的列表。它还实现了 Python 迭代器接口，以便我们可以直接迭代 `Row` 对象的集合。

`Row` 对象本身旨在像 Python [named tuples](https://docs.python.org/3/library/collections.html#collections.namedtuple) 一样行事。下面我们展示了访问行的各种方法。

+   **Tuple Assignment** - 这是最 Python 特有的风格，即按位置分配变量，就像它们被接收到的那样：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for x, y in result:
        ...
    ```

+   **整数索引** - 元组是 Python 序列，因此也可以进行常规整数访问：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for row in result:
        x = row[0]
    ```

+   **属性名称** - 由于这些是 Python 的命名元组，元组具有与每列名称匹配的动态属性名称。这些名称通常是 SQL 语句为每行的列分配的名称。虽然它们通常是相当可预测的，也可以通过标签进行控制，在定义较少的情况下，它们可能受到特定于数据库的行为的影响：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for row in result:
        y = row.y

        # illustrate use with Python f-strings
        print(f"Row: {row.x}  {y}")
    ```

+   **映射访问** - 要将行作为 Python **mapping** 对象接收，这本质上是 Python 对通用 `dict` 对象的只读版本的接口，可以使用 `Result.mappings()` 修改器将 `Result` 转换为 `MappingResult` 对象；这是一个产生类似于字典的 `RowMapping` 对象而不是 `Row` 对象的结果对象：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for dict_row in result.mappings():
        x = dict_row["x"]
        y = dict_row["y"]
    ```  ### 发送参数

SQL 语句通常伴随着要与语句本身一起传递的数据，就像我们之前在 INSERT 示例中看到的那样。因此，`Connection.execute()` 方法还接受参数，这些参数称为 bound parameters。一个简单的示例可能是，如果我们想要将 SELECT 语句限制为仅符合某个条件的行，例如“y”值大于通过函数传入的某个特定值的行。

为了实现这一点，使得 SQL 语句保持不变并且驱动程序可以正确地清理值，我们在语句中添加了一个名为“y”的 WHERE 条件；`text()`构造使用冒号格式“`:y`”接受这些参数。然后，“`:y`”的实际值以字典的形式作为第二个参数传递给`Connection.execute()`：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(text("SELECT x, y FROM some_table WHERE y > :y"), {"y": 2})
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table  WHERE  y  >  ?
[...]  (2,)
x: 2  y: 4
x: 6  y: 8
x: 9  y: 10
ROLLBACK 
```

在记录的 SQL 输出中，我们可以看到当绑定参数`:y`发送到 SQLite 数据库时，它被转换为问号。这是因为 SQLite 数据库驱动程序使用一种称为“qmark 参数样式”的格式，这是 DBAPI 规范允许的六种不同格式之一。SQLAlchemy 将这些格式抽象成了一个，即使用冒号的“named”格式。### 发送多个参数

在 提交更改 的示例中，我们执行了一个 INSERT 语句，其中看起来我们能够一次将多行插入到数据库中。对于“INSERT”、“UPDATE”和“DELETE”等 DML 语句，我们可以通过传递一个字典列表而不是单个字典给`Connection.execute()`方法来发送**多个参数集**，这表明应该针对每个参数集调用单个 SQL 语句多次。这种执行方式称为 executemany：

```py
>>> with engine.connect() as conn:
...     conn.execute(
...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
...         [{"x": 11, "y": 12}, {"x": 13, "y": 14}],
...     )
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  some_table  (x,  y)  VALUES  (?,  ?)
[...]  [(11,  12),  (13,  14)]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

上述操作相当于针对每个参数集合运行给定的 INSERT 语句一次，但该操作将被优化以在多行上获得更好的性能。

“execute”和“executemany”之间的一个关键行为差异是，后者不支持返回结果行，即使语句包含 RETURNING 子句也是如此。唯一的例外是当使用 Core 的`insert()`构造时，稍后在本教程的使用 INSERT 语句中介绍，该构造还使用`Insert.returning()`方法指示 RETURNING。在这种情况下，SQLAlchemy 使用特殊逻辑重新组织 INSERT 语句，以便在支持 RETURNING 的同时可以为多行调用它。

另请参阅

executemany - 在 Glossary 中描述了 DBAPI 级别的[cursor.executemany()](https://peps.python.org/pep-0249/#executemany) 方法，该方法用于大多数“executemany”执行。

INSERT 语句的“插入多个值”行为 - 在与引擎和连接一起工作中，描述了`Insert.returning()`用于提供带有“executemany”执行的结果集的专用逻辑。### 获取行

首先，我们将通过利用之前插入的行，对我们创建的表运行文本 SELECT 语句，更详细地说明`Result`对象：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(text("SELECT x, y FROM some_table"))
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table
[...]  ()
x: 1  y: 1
x: 2  y: 4
x: 6  y: 8
x: 9  y: 10
ROLLBACK 
```

上面，我们执行的“SELECT”字符串选择了我们表中的所有行。返回的对象称为`Result`，表示结果行的可迭代对象。

`Result`有很多用于获取和转换行的方法，例如之前演示的`Result.all()`方法，它返回所有`Row`对象的列表。它还实现了 Python 迭代器接口，以便我们可以直接迭代`Row`对象的集合。

`Row`对象本身旨在像 Python 的[命名元组](https://docs.python.org/3/library/collections.html#collections.namedtuple)一样操作。下面我们演示了多种访问行的方式。

+   **元组赋值** - 这是最符合 Python 习惯的样式，即按位置将变量分配给每行接收到的值：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for x, y in result:
        ...
    ```

+   **整数索引** - 元组是 Python 序列，因此也可以使用常规整数访问：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for row in result:
        x = row[0]
    ```

+   **属性名称** - 由于这些是 Python 命名元组，元组具有与每列名称匹配的动态属性名称。这些名称通常是 SQL 语句为每行分配的列名称。虽然它们通常相当可预测，并且也可以由标签控制，在定义不太明确的情况下，它们可能受到数据库特定的行为的影响：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for row in result:
        y = row.y

        # illustrate use with Python f-strings
        print(f"Row: {row.x}  {y}")
    ```

+   **映射访问** - 要将行作为 Python **映射** 对象接收，这实质上是 Python 对普通`dict`对象的只读接口，可以使用`Result`通过`Result.mappings()`修饰符将其**转换**为`MappingResult`对象；这是一个产生类似于字典的`RowMapping`对象而不是`Row`对象的结果对象：

    ```py
    result = conn.execute(text("select x, y from some_table"))

    for dict_row in result.mappings():
        x = dict_row["x"]
        y = dict_row["y"]
    ```

### 发送参数

SQL 语句通常伴随着要与语句一起传递的数据，就像我们之前在 INSERT 示例中看到的那样。因此，`Connection.execute()`方法也接受参数，这些参数被称为绑定参数。一个简单的例子可能是，如果我们想要将 SELECT 语句限制仅适用于满足某些条件的行，例如行中的“y”值大于传入函数的某个值。

为了使 SQL 语句保持不变，以便驱动程序可以正确地对值进行处理，我们在语句中添加了一个名为“y”的 WHERE 条件；`text()`构造函数接受这些参数，使用冒号格式“`:y`”。然后，“`:y`”的实际值作为字典的第二个参数传递给`Connection.execute()`：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(text("SELECT x, y FROM some_table WHERE y > :y"), {"y": 2})
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table  WHERE  y  >  ?
[...]  (2,)
x: 2  y: 4
x: 6  y: 8
x: 9  y: 10
ROLLBACK 
```

在记录的 SQL 输出中，我们可以看到绑定参数`:y`在发送到 SQLite 数据库时被转换为问号。这是因为 SQLite 数据库驱动程序使用一种称为“问号参数样式”的格式，这是 DBAPI 规范允许的六种不同格式之一。SQLAlchemy 将这些格式抽象成了一种格式，即使用冒号的“命名”格式。

### 发送多个参数

在提交更改的示例中，我们执行了一个 INSERT 语句，看起来我们能够一次性向数据库中插入多行数据。对于 DML 语句，如“INSERT”、“UPDATE”和“DELETE”，我们可以通过传递一个字典列表而不是单个字典给`Connection.execute()`方法，从而发送**多个参数集**，这表明单个 SQL 语句应该被多次调用，每次为一个参数集。这种执行方式被称为 executemany：

```py
>>> with engine.connect() as conn:
...     conn.execute(
...         text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
...         [{"x": 11, "y": 12}, {"x": 13, "y": 14}],
...     )
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  some_table  (x,  y)  VALUES  (?,  ?)
[...]  [(11,  12),  (13,  14)]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

上述操作等同于为每个参数集运行给定的 INSERT 语句一次，只是该操作将被优化以在许多行上获得更好的性能。

“execute”和“executemany”之间的一个关键行为差异是，后者不支持返回结果行，即使语句包含 RETURNING 子句也是如此。唯一的例外是在使用 Core `insert()`构造时，稍后在本教程的使用 INSERT 语句部分介绍，该构造还使用`Insert.returning()`方法指示 RETURNING。在这种情况下，SQLAlchemy 利用特殊逻辑重新组织 INSERT 语句，以便可以为多行调用它，同时仍支持 RETURNING。

另请参阅

executemany - 在 Glossary 中描述了用于大多数“executemany”执行的 DBAPI 级别[cursor.executemany()](https://peps.python.org/pep-0249/#executemany)方法。

“INSERT 语句的 Insert Many Values”行为 - 在使用引擎和连接中，描述了`Insert.returning()`使用的专门逻辑，以便通过“executemany”执行交付结果集。

## 使用 ORM 会话执行

正如之前提到的，上面的大多数模式和示例也适用于与 ORM 一起使用，因此在这里我们将介绍这种用法，以便随着教程的进行，我们将能够以 Core 和 ORM 一起的方式来说明每个模式。

当使用 ORM 时，基本的事务/数据库交互对象称为`Session`。在现代 SQLAlchemy 中，这个对象的使用方式与`Connection`非常相似，实际上，当使用`Session`时，它在内部引用一个`Connection`，用于发出 SQL。

当`Session`与非 ORM 构造一起使用时，它会通过我们提供的 SQL 语句，并且通常不会与`Connection`直接执行有太大不同，因此我们可以根据我们已经学过的简单文本 SQL 操作来说明它。

`Session`有几种不同的创建模式，但在这里我们将说明最基本的一种，它与使用`Connection`的方式完全一致，即在上下文管理器中构造它：

```py
>>> from sqlalchemy.orm import Session

>>> stmt = text("SELECT x, y FROM some_table WHERE y > :y ORDER BY x, y")
>>> with Session(engine) as session:
...     result = session.execute(stmt, {"y": 6})
...     for row in result:
...         print(f"x: {row.x}  y: {row.y}")
BEGIN  (implicit)
SELECT  x,  y  FROM  some_table  WHERE  y  >  ?  ORDER  BY  x,  y
[...]  (6,)
x: 6  y: 8
x: 9  y: 10
x: 11  y: 12
x: 13  y: 14
ROLLBACK 
```

上面的示例可以与前一节中发送参数中的示例进行比较 - 我们直接将`with engine.connect() as conn`的调用替换为`with Session(engine) as session`，然后像使用`Connection.execute()`方法一样使用`Session.execute()`方法。

同样，像`Connection`一样，`Session`也具有“边提交边执行”的行为，使用`Session.commit()`方法，下面通过一个文本 UPDATE 语句来修改一些数据进行说明：

```py
>>> with Session(engine) as session:
...     result = session.execute(
...         text("UPDATE some_table SET y=:y WHERE x=:x"),
...         [{"x": 9, "y": 11}, {"x": 13, "y": 15}],
...     )
...     session.commit()
BEGIN  (implicit)
UPDATE  some_table  SET  y=?  WHERE  x=?
[...]  [(11,  9),  (15,  13)]
COMMIT 
```

在上面，我们使用绑定参数“executemany”风格的执行方式调用了一个 UPDATE 语句，该语句介绍在发送多个参数中，以“边提交边执行”方式结束了该块。

提示

`Session` 实际上在结束事务后并不保留 `Connection` 对象。下次需要对数据库执行 SQL 时，它会从 `Engine` 获取一个新的 `Connection`。

`Session` 很显然比那个拥有更多的技巧，然而理解它有一个 `Session.execute()` 方法，该方法的使用方式与 `Connection.execute()` 相同，将使我们能够开始后面的示例。

另请参阅

使用会话的基础知识 - 展示了与 `Session` 对象的基本创建和使用模式。
