# 使用插入语句

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/data_insert.html`](https://docs.sqlalchemy.org/en/20/tutorial/data_insert.html)

在使用 Core 以及在使用 ORM 进行批量操作时，可以直接使用`insert()`函数生成 SQL INSERT 语句 - 此函数生成`Insert`的新实例，表示将新数据添加到表中的 INSERT 语句。

**ORM 读者** -

本节详细介绍了在表中添加新行时生成单个 SQL INSERT 语句的核心方法。在使用 ORM 时，我们通常会使用另一个称为 unit of work 的工具，它会自动化一次性生成许多 INSERT 语句。但是，即使 ORM 为我们运行它，了解核心如何处理数据创建和操作也非常有用。此外，ORM 还支持使用称为批量/多行插入、更新和删除的功能直接使用 INSERT。

要直接跳转到使用 ORM 使用正常工作单元模式插入行的方法，请参阅使用 ORM 工作单元模式插入行。

## 插入（insert()）SQL 表达式构造

一种简单的`Insert`示例，同时说明了目标表和 VALUES 子句：

```py
>>> from sqlalchemy import insert
>>> stmt = insert(user_table).values(name="spongebob", fullname="Spongebob Squarepants")
```

上述`stmt`变量是`Insert`的一个实例。大多数 SQL 表达式都可以直接转换为字符串形式，以便查看生成的通用形式：

```py
>>> print(stmt)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (:name,  :fullname) 
```

字符串形式是通过生成对象的`Compiled`形式来创建的，该对象包括语句的数据库特定字符串 SQL 表示；我们可以直接使用`ClauseElement.compile()`方法获取此对象：

```py
>>> compiled = stmt.compile()
```

我们的`Insert`构造是“参数化”构造的一个例子，前面在发送参数已经有过示例；要查看`name`和`fullname`绑定参数，这些也可以从`Compiled`构造中获取：

```py
>>> compiled.params
{'name': 'spongebob', 'fullname': 'Spongebob Squarepants'}
```

## 执行语句

调用该语句，我们可以将一行插入到`user_table`中。 可以在 SQL 日志中看到 INSERT SQL 和捆绑参数：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('spongebob',  'Spongebob Squarepants')
COMMIT 
```

在上面的简单形式中，INSERT 语句不会返回任何行，如果只插入了一行，则通常会包括返回有关插入该行期间生成的列级默认值的信息的能力，最常见的是整数主键值。 在上述情况下，SQLite 数据库中的第一行通常会为第一个整数主键值返回 `1`，我们可以使用`CursorResult.inserted_primary_key` 访问器获取它：

```py
>>> result.inserted_primary_key
(1,)
```

提示

`CursorResult.inserted_primary_key` 返回一个元组，因为主键可能包含多列。 这称为复合主键。 `CursorResult.inserted_primary_key` 旨在始终包含刚刚插入的记录的完整主键，而不仅仅是“cursor.lastrowid”类型的值，并且旨在无论是否使用了“autoincrement”，都将其填充，因此为了表示完整的主键，它是一个元组。

从版本 1.4.8 中更改：由 `CursorResult.inserted_primary_key` 返回的元组现在是通过将其作为`Row` 对象来实现的命名元组。

## INSERT 通常会自动生成“values”子句

上面的示例使用了 `Insert.values()` 方法来显式创建 SQL INSERT 语句的 VALUES 子句。 如果我们实际上不使用 `Insert.values()` 而只打印出一个“空”语句，我们会得到一个插入表中每一列的 INSERT：

```py
>>> print(insert(user_table))
INSERT  INTO  user_account  (id,  name,  fullname)  VALUES  (:id,  :name,  :fullname) 
```

如果我们拿一个尚未调用`Insert.values()`的 `Insert` 构造，并执行它而不是打印它，语句将根据我们传递给`Connection.execute()` 方法的参数编译为一个字符串，而且只包含与传递的参数相关的列。实际上，这是通常使用`Insert` 插入行的方式，而无需输入显式的 VALUES 子句。下面的示例说明了执行具有一次性参数列表的两列 INSERT 语句：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(
...         insert(user_table),
...         [
...             {"name": "sandy", "fullname": "Sandy Cheeks"},
...             {"name": "patrick", "fullname": "Patrick Star"},
...         ],
...     )
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  [('sandy',  'Sandy Cheeks'),  ('patrick',  'Patrick Star')]
COMMIT 
```

上述执行首次展示了发送多个参数中介绍的“executemany”形式，但与使用`text()` 构造时不同，我们不必拼写任何 SQL。通过将字典或字典列表传递给`Connection.execute()` 方法与 `Insert` 构造一起使用，`Connection` 确保传递的列名将自动在 `Insert` 构造的 VALUES 子句中表达。

深度炼金

嗨，欢迎来到**深度炼金**的第一版。左边的人被称为**炼金师**，你会注意到他们**并不**是巫师，因为尖尖的帽子并没有竖起来。炼金师会描述通常**更加高级和/或棘手**的事物，而且通常**不是**必需的，但出于某种原因，他们觉得你应该了解 SQLAlchemy 能做的这件事情。

在这个版本中，为了在 `address_table` 中拥有一些有趣的数据，下面是一个更高级的示例，说明了如何在明确使用 `Insert.values()` 方法的同时，包含从参数生成的额外 VALUES。一个 标量子查询 被构建，利用了下一节中介绍的 `select()` 结构，子查询中使用的参数使用明确的绑定参数名设置，使用了 `bindparam()` 结构。

这是一些稍微**深入**的炼金术，这样我们就可以在不将主键标识符从 `user_table` 操作中提取到应用程序中的情况下添加相关行。大多数炼金术师会简单地使用 ORM 来处理这类事情。

```py
>>> from sqlalchemy import select, bindparam
>>> scalar_subq = (
...     select(user_table.c.id)
...     .where(user_table.c.name == bindparam("username"))
...     .scalar_subquery()
... )

>>> with engine.connect() as conn:
...     result = conn.execute(
...         insert(address_table).values(user_id=scalar_subq),
...         [
...             {
...                 "username": "spongebob",
...                 "email_address": "spongebob@sqlalchemy.org",
...             },
...             {"username": "sandy", "email_address": "sandy@sqlalchemy.org"},
...             {"username": "sandy", "email_address": "sandy@squirrelpower.org"},
...         ],
...     )
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  address  (user_id,  email_address)  VALUES  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?)
[...]  [('spongebob',  'spongebob@sqlalchemy.org'),  ('sandy',  'sandy@sqlalchemy.org'),
('sandy',  'sandy@squirrelpower.org')]
COMMIT 
```

有了这个，我们的表中有一些更有趣的数据，我们将在接下来的章节中加以利用。

提示

如果我们在 `Insert.values()` 中不带参数地指定，将生成一个真正的“空”INSERT，它仅插入表的“默认值”，而不包括任何明确的值；并非每个数据库后端都支持这个功能，但下面是 SQLite 生成的内容：

```py
>>> print(insert(user_table).values().compile(engine))
INSERT  INTO  user_account  DEFAULT  VALUES 
```  ## INSERT…RETURNING

对于支持的后端，RETURNING 子句会自动被用来检索最后插入的主键值以及服务器默认值。但是 RETURNING 子句也可以使用 `Insert.returning()` 方法来明确指定；在这种情况下，执行语句时返回的 `Result` 对象具有可提取的行：

```py
>>> insert_stmt = insert(address_table).returning(
...     address_table.c.id, address_table.c.email_address
... )
>>> print(insert_stmt)
INSERT  INTO  address  (id,  user_id,  email_address)
VALUES  (:id,  :user_id,  :email_address)
RETURNING  address.id,  address.email_address 
```

它也可以与 `Insert.from_select()` 结合使用，就像下面的例子一样，它建立在 INSERT…FROM SELECT 中所述的例子之上：

```py
>>> select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
>>> insert_stmt = insert(address_table).from_select(
...     ["user_id", "email_address"], select_stmt
... )
>>> print(insert_stmt.returning(address_table.c.id, address_table.c.email_address))
INSERT  INTO  address  (user_id,  email_address)
SELECT  user_account.id,  user_account.name  ||  :name_1  AS  anon_1
FROM  user_account  RETURNING  address.id,  address.email_address 
```

提示

RETURNING 特性也被 UPDATE 和 DELETE 语句所支持，这将在本教程的后续部分介绍。

对于 INSERT 语句，RETURNING 功能可用于单行语句以及一次插入多行的语句。对于支持 RETURNING 的 SQLAlchemy 中包含的所有方言，多行 INSERT 支持是特定于方言的。请参阅“INSERT 语句的插入多个值”行为部分了解此功能的背景。

另请参阅

ORM 也支持带有或不带有 RETURNING 的批量 INSERT。请参阅 ORM 批量 INSERT 语句以获取参考文档。## INSERT…FROM SELECT

`Insert`的一个较少使用的特性，但为了完整性，在这里，`Insert`构造可以使用`Insert.from_select()`方法直接从 SELECT 中获取行进行插入。此方法接受一个`select()`构造，下一节将讨论此构造，以及要在实际 INSERT 中定位的列名列表。在下面的示例中，从`user_account`表中派生的行被添加到`address`表中，为每个用户提供`aol.com`的免费电子邮件地址：

```py
>>> select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
>>> insert_stmt = insert(address_table).from_select(
...     ["user_id", "email_address"], select_stmt
... )
>>> print(insert_stmt)
INSERT  INTO  address  (user_id,  email_address)
SELECT  user_account.id,  user_account.name  ||  :name_1  AS  anon_1
FROM  user_account 
```

当希望直接将数据从数据库的其他部分复制到新的行集时使用此构造，而无需实际从客户端获取和重新发送数据。

另请参阅

`Insert` - 在 SQL 表达式 API 文档中

## insert() SQL 表达式构造

一个简单的`Insert`示例，同时说明目标表和 VALUES 子句：

```py
>>> from sqlalchemy import insert
>>> stmt = insert(user_table).values(name="spongebob", fullname="Spongebob Squarepants")
```

上述`stmt`变量是`Insert`的一个实例。大多数 SQL 表达式可以直接转换为字符串形式，以查看正在生成的一般形式：

```py
>>> print(stmt)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (:name,  :fullname) 
```

字符串形式是通过生成对象的`Compiled`形式创建的，其中包括语句的特定于数据库的字符串 SQL 表示；我们可以直接使用`ClauseElement.compile()`方法获取此对象：

```py
>>> compiled = stmt.compile()
```

我们的`Insert`构造是“参数化”构造的一个示例，在之前的发送参数中已经说明过；要查看`name`和`fullname` 绑定参数，这些都可以从`Compiled`构造中获取：

```py
>>> compiled.params
{'name': 'spongebob', 'fullname': 'Spongebob Squarepants'}
```

## 执行该语句

调用该语句，我们可以将一行插入到`user_table`中。可以在 SQL 日志中看到 INSERT SQL 以及捆绑的参数：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('spongebob',  'Spongebob Squarepants')
COMMIT 
```

在上面的简单形式中，INSERT 语句不会返回任何行，如果只插入了一行，则通常会包含返回有关在插入该行期间生成的列级默认值信息的功能，最常见的是整数主键值。在上述情况下，SQLite 数据库中的第一行通常将为第一个整数主键值返回`1`，我们可以使用`CursorResult.inserted_primary_key`访问器来获取：

```py
>>> result.inserted_primary_key
(1,)
```

提示

`CursorResult.inserted_primary_key`返回一个元组，因为主键可能包含多个列。这称为复合主键。`CursorResult.inserted_primary_key`旨在始终包含刚刚插入的记录的完整主键，而不仅仅是“cursor.lastrowid”类型的值，并且旨在无论是否使用“autoincrement”，都会填充，因此为了表达完整的主键，它是一个元组。

从版本 1.4.8 中更改：`CursorResult.inserted_primary_key`返回的元组现在是通过将其返回为`Row`对象来履行的命名元组。

## INSERT 通常会自动生成“values”子句

上面的示例使用了`Insert.values()`方法来显式创建 SQL INSERT 语句的 VALUES 子句。如果我们实际上不使用`Insert.values()`，只打印出一个“空”的语句，我们会得到一个对表中每一列进行插入的 INSERT：

```py
>>> print(insert(user_table))
INSERT  INTO  user_account  (id,  name,  fullname)  VALUES  (:id,  :name,  :fullname) 
```

如果我们对一个尚未调用`Insert.values()`的`Insert`构造进行执行而不是打印它，该语句将根据我们传递给`Connection.execute()`方法的参数编译为一个字符串，并且仅包括与传递的参数相关的列。这实际上是使用`Insert`插入行的常用方式，而无需编写明确的 VALUES 子句。下面的示例说明了如何一次执行具有参数列表的两列 INSERT 语句：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(
...         insert(user_table),
...         [
...             {"name": "sandy", "fullname": "Sandy Cheeks"},
...             {"name": "patrick", "fullname": "Patrick Star"},
...         ],
...     )
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  [('sandy',  'Sandy Cheeks'),  ('patrick',  'Patrick Star')]
COMMIT 
```

上述执行首先展示了“executemany”形式，如发送多个参数中所示，但与使用`text()`构造时不同，我们不必拼写任何 SQL。通过将字典或字典列表传递给`Connection.execute()`方法，与`Insert`构造一起使用，`Connection`确保传递的列名将自动在`Insert`构造的 VALUES 子句中表示。

深度魔法

嗨，欢迎来到第一版的**深度魔法**。左边的人被称为**炼金术士**，你会注意到他们**不是**巫师，因为尖顶帽没有竖起来。炼金术士会来描述一些通常**更高级和/或棘手**的事情，此外通常**不需要**，但出于某种原因他们觉得你应该知道 SQLAlchemy 能做这件事。

在这个版本中，为了使`address_table`中有一些有趣的数据，下面是一个更高级的示例，演示了如何在同时包含来自参数的附加 VALUES 的情况下，可以显式使用`Insert.values()`方法。构造了一个标量子查询，利用了下一节中介绍的`select()`构造，并且在子查询中使用的参数使用了显式绑定参数名称，使用`bindparam()`构造建立。

这是一些稍微**深入**的炼金术，这样我们就可以在不从`user_table`操作中获取主键标识符的情况下添加相关行到应用程序中。大多数炼金术师将简单地使用 ORM 来处理这类事情。

```py
>>> from sqlalchemy import select, bindparam
>>> scalar_subq = (
...     select(user_table.c.id)
...     .where(user_table.c.name == bindparam("username"))
...     .scalar_subquery()
... )

>>> with engine.connect() as conn:
...     result = conn.execute(
...         insert(address_table).values(user_id=scalar_subq),
...         [
...             {
...                 "username": "spongebob",
...                 "email_address": "spongebob@sqlalchemy.org",
...             },
...             {"username": "sandy", "email_address": "sandy@sqlalchemy.org"},
...             {"username": "sandy", "email_address": "sandy@squirrelpower.org"},
...         ],
...     )
...     conn.commit()
BEGIN  (implicit)
INSERT  INTO  address  (user_id,  email_address)  VALUES  ((SELECT  user_account.id
FROM  user_account
WHERE  user_account.name  =  ?),  ?)
[...]  [('spongebob',  'spongebob@sqlalchemy.org'),  ('sandy',  'sandy@sqlalchemy.org'),
('sandy',  'sandy@squirrelpower.org')]
COMMIT 
```

有了这个，我们的表中有了一些更有趣的数据，我们将在接下来的章节中使用它们。

提示

如果我们指示不带任何参数的`Insert.values()`，则生成一个真正的“空”INSERT，仅为表中的“默认值”插入，但并不包括任何显式值；并非所有的数据库后端都支持此功能，但是这是 SQLite 生成的内容：

```py
>>> print(insert(user_table).values().compile(engine))
INSERT  INTO  user_account  DEFAULT  VALUES 
```

## INSERT…RETURNING

支持的后端自动使用 RETURNING 子句以检索最后插入的主键值以及服务器默认值的值。但是，也可以使用`Insert.returning()`方法显式指定 RETURNING 子句；在这种情况下，执行该语句时返回的`Result`对象具有可以获取的行：

```py
>>> insert_stmt = insert(address_table).returning(
...     address_table.c.id, address_table.c.email_address
... )
>>> print(insert_stmt)
INSERT  INTO  address  (id,  user_id,  email_address)
VALUES  (:id,  :user_id,  :email_address)
RETURNING  address.id,  address.email_address 
```

它也可以与`Insert.from_select()`结合使用，就像下面的示例一样，该示例建立在 INSERT…FROM SELECT 中所述示例的基础上：

```py
>>> select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
>>> insert_stmt = insert(address_table).from_select(
...     ["user_id", "email_address"], select_stmt
... )
>>> print(insert_stmt.returning(address_table.c.id, address_table.c.email_address))
INSERT  INTO  address  (user_id,  email_address)
SELECT  user_account.id,  user_account.name  ||  :name_1  AS  anon_1
FROM  user_account  RETURNING  address.id,  address.email_address 
```

提示

RETURNING 特性也被 UPDATE 和 DELETE 语句支持，这将在本教程的后续部分中介绍。

对于 INSERT 语句，RETURNING 功能可用于单行语句以及一次插入多行的语句。对于具有 RETURNING 功能的多行 INSERT 的支持是方言特定的，但是对于 SQLAlchemy 中支持 RETURNING 的所有方言都是支持的。有关此功能的背景，请参阅 “Insert Many Values” Behavior for INSERT statements 部分。

请参阅

ORM 也支持带有或不带有 RETURNING 的批量插入。请参阅 ORM 批量插入语句 进行参考文档。

## INSERT…FROM SELECT

`Insert` 的一个不太常用的特性，但出于完整性考虑，在这里，`Insert` 结构可以使用 `Insert.from_select()` 方法组合一个直接从 SELECT 中获取行的 INSERT。该方法接受一个 `select()` 结构，下一节将讨论它，以及一个要在实际 INSERT 中定位的列名列表。在下面的示例中，从 `user_account` 表中的行派生出添加到 `address` 表中的行，为每个用户提供 `aol.com` 的免费电子邮件地址：

```py
>>> select_stmt = select(user_table.c.id, user_table.c.name + "@aol.com")
>>> insert_stmt = insert(address_table).from_select(
...     ["user_id", "email_address"], select_stmt
... )
>>> print(insert_stmt)
INSERT  INTO  address  (user_id,  email_address)
SELECT  user_account.id,  user_account.name  ||  :name_1  AS  anon_1
FROM  user_account 
```

当一个人想要直接将数据从数据库的某个其他部分复制到一组新的行中时，可以使用这个结构，而不需要从客户端获取和重新发送数据。

请参阅

`插入` - SQL Expression API 文档中的 INSERT
