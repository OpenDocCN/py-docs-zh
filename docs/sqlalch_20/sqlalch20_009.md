# 使用 UPDATE 和 DELETE 语句

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/data_update.html`](https://docs.sqlalchemy.org/en/20/tutorial/data_update.html)

到目前为止，我们已经覆盖了 `Insert`，这样我们可以将一些数据放入我们的数据库中，并且花了很多时间在 `Select` 上，该语句处理了从数据库检索数据所使用的各种广泛的使用模式。 在本节中，我们将涵盖 `Update` 和 `Delete` 构造，用于修改现有行以及删除现有行。 本节将从核心的角度讨论这些构造。

**ORM 读者** - 正如在 使用 INSERT 语句 中提到的情况一样，当与 ORM 一起使用时，`Update` 和 `Delete` 操作通常从 `Session` 对象内部作为 工作单元 进程的一部分调用。

然而，与 `Insert` 不同，`Update` 和 `Delete` 构造也可以直接与 ORM 一起使用，使用一种称为“ORM-enabled update and delete”的模式；因此，熟悉这些构造对于 ORM 的使用很有用。 这两种使用方式在以下章节中讨论：使用工作单元模式更新 ORM 对象 和 使用工作单元模式删除 ORM 对象。

## update() SQL 表达式构造

`update()` 函数生成一个 `Update` 的新实例，表示 SQL 中的 UPDATE 语句，该语句将更新表中的现有数据。

与`insert()`构造类似，还有一种“传统”的`update()`形式，它一次只针对一个表发出 UPDATE，不返回任何行。然而，一些后端支持可以一次修改多个表的 UPDATE 语句，并且 UPDATE 语句也支持 RETURNING，使得匹配行中包含的列可以在结果集中返回。

一个基本的 UPDATE 看起来像：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(user_table)
...     .where(user_table.c.name == "patrick")
...     .values(fullname="Patrick the Star")
... )
>>> print(stmt)
UPDATE  user_account  SET  fullname=:fullname  WHERE  user_account.name  =  :name_1 
```

`Update.values()`方法控制 UPDATE 语句的 SET 元素的内容。这是由`Insert`构造共享的相同方法。参数通常可以使用列名称作为关键字参数传递。

UPDATE 支持所有主要的 SQL UPDATE 形式，包括针对表达式的更新，在其中我们可以利用`Column`表达式：

```py
>>> stmt = update(user_table).values(fullname="Username: " + user_table.c.name)
>>> print(stmt)
UPDATE  user_account  SET  fullname=(:name_1  ||  user_account.name) 
```

为了在“executemany”上下文中支持 UPDATE，其中将对同一语句调用许多参数集，可以使用`bindparam()`构造来设置绑定参数；这些参数取代了通常放置文本值的位置：

```py
>>> from sqlalchemy import bindparam
>>> stmt = (
...     update(user_table)
...     .where(user_table.c.name == bindparam("oldname"))
...     .values(name=bindparam("newname"))
... )
>>> with engine.begin() as conn:
...     conn.execute(
...         stmt,
...         [
...             {"oldname": "jack", "newname": "ed"},
...             {"oldname": "wendy", "newname": "mary"},
...             {"oldname": "jim", "newname": "jake"},
...         ],
...     )
BEGIN  (implicit)
UPDATE  user_account  SET  name=?  WHERE  user_account.name  =  ?
[...]  [('ed',  'jack'),  ('mary',  'wendy'),  ('jake',  'jim')]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

可应用于 UPDATE 的其他技术包括：

### 相关更新

UPDATE 语句可以通过使用相关子查询中的其他表中的行来使用。子查询可以用于任何可以放置列表达式的地方：

```py
>>> scalar_subq = (
...     select(address_table.c.email_address)
...     .where(address_table.c.user_id == user_table.c.id)
...     .order_by(address_table.c.id)
...     .limit(1)
...     .scalar_subquery()
... )
>>> update_stmt = update(user_table).values(fullname=scalar_subq)
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=(SELECT  address.email_address
FROM  address
WHERE  address.user_id  =  user_account.id  ORDER  BY  address.id
LIMIT  :param_1) 
```  ### UPDATE..FROM

一些数据库，如 PostgreSQL 和 MySQL，支持一种称为“UPDATE FROM”的语法，在特殊的 FROM 子句中可以直接声明附加表。当其他表位于语句的 WHERE 子句中时，此语法将隐式生成：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
...     .values(fullname="Pat")
... )
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=:fullname  FROM  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  :email_address_1 
```

还有一种 MySQL 特定的语法，可以更新多个表。这要求我们在 VALUES 子句中引用`Table`对象，以便引用其他表：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
...     .values(
...         {
...             user_table.c.fullname: "Pat",
...             address_table.c.email_address: "pat@aol.com",
...         }
...     )
... )
>>> from sqlalchemy.dialects import mysql
>>> print(update_stmt.compile(dialect=mysql.dialect()))
UPDATE  user_account,  address
SET  address.email_address=%s,  user_account.fullname=%s
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  %s 
```  ### 参数有序更新

另一个仅适用于 MySQL 的行为是，UPDATE 的 SET 子句中参数的顺序实际上影响每个表达式的评估。对于这种用例，`Update.ordered_values()`方法接受一个元组序列，以便可以控制此顺序 [[2]](#id2)：

```py
>>> update_stmt = update(some_table).ordered_values(
...     (some_table.c.y, 20), (some_table.c.x, some_table.c.y + 10)
... )
>>> print(update_stmt)
UPDATE  some_table  SET  y=:y,  x=(some_table.y  +  :y_1) 
```  ## delete() SQL 表达式构造

`delete()` 函数生成一个表示 SQL 中 DELETE 语句的新实例 `Delete`，该语句将从表中删除行。

从 API 视角来看，`delete()` 语句与 `update()` 构造非常相似，传统上不返回行，但在一些数据库后端上允许有 RETURNING 变体。

```py
>>> from sqlalchemy import delete
>>> stmt = delete(user_table).where(user_table.c.name == "patrick")
>>> print(stmt)
DELETE  FROM  user_account  WHERE  user_account.name  =  :name_1 
```

### 多表删除

与 `Update` 类似，`Delete` 支持在 WHERE 子句中使用相关子查询以及后端特定的多表语法，例如 MySQL 上的 `DELETE FROM..USING`：

```py
>>> delete_stmt = (
...     delete(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
... )
>>> from sqlalchemy.dialects import mysql
>>> print(delete_stmt.compile(dialect=mysql.dialect()))
DELETE  FROM  user_account  USING  user_account,  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  %s 
```  ## 从 UPDATE、DELETE 中获取受影响的行数

`Update` 和 `Delete` 都支持在语句执行后返回匹配行数的功能，对于使用 Core `Connection` 调用的语句，即 `Connection.execute()`。根据下面提到的注意事项，这个值可以从 `CursorResult.rowcount` 属性中获取：

```py
>>> with engine.begin() as conn:
...     result = conn.execute(
...         update(user_table)
...         .values(fullname="Patrick McStar")
...         .where(user_table.c.name == "patrick")
...     )
...     print(result.rowcount)
BEGIN  (implicit)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Patrick McStar',  'patrick')
1
COMMIT 
```

提示

`CursorResult` 类是 `Result` 的子类，其中包含特定于 DBAPI `cursor` 对象的附加属性。当通过 `Connection.execute()` 方法调用语句时，将返回此子类的实例。在使用 ORM 时，对所有 INSERT、UPDATE 和 DELETE 语句使用 `Session.execute()` 方法会返回此类型的对象。

关于 `CursorResult.rowcount` 的事实：

+   返回的值是由语句的 WHERE 子句匹配的行数。无论实际上是否修改了行都无关紧要。

+   对于使用 RETURNING 的 UPDATE 或 DELETE 语句，或者使用 executemany 执行的 UPDATE 或 DELETE 语句，不一定可以使用 `CursorResult.rowcount`。其可用性取决于正在使用的 DBAPI 模块。

+   在任何 DBAPI 不能确定某种类型语句的行数的情况下，返回值将为 `-1`。

+   SQLAlchemy 在关闭游标之前预先缓存 DBAPI 的 `cursor.rowcount` 值，因为某些 DBAPI 不支持事后访问此属性。为了为不是 UPDATE 或 DELETE 的语句（如 INSERT 或 SELECT）预先缓存 `cursor.rowcount`，可以使用 `Connection.execution_options.preserve_rowcount` 执行选项。

+   一些驱动程序，特别是用于非关系型数据库的第三方方言，可能根本不支持 `CursorResult.rowcount`。`CursorResult.supports_sane_rowcount` 游标属性会指示此情况。

+   “rowcount” 被 ORM 工作单元 过程用于验证 UPDATE 或 DELETE 语句是否匹配了预期数量的行，并且也是 ORM 版本控制功能的重要组成部分，该功能在 配置版本计数器 中有文档说明。

## 使用 UPDATE、DELETE 与 RETURNING

与 `Insert` 构造类似，`Update` 和 `Delete` 也支持 RETURNING 子句，通过使用 `Update.returning()` 和 `Delete.returning()` 方法添加。当这些方法在支持 RETURNING 的后端上使用时，与语句的 WHERE 条件匹配的所有行的选定列将作为可以迭代的行返回到 `Result` 对象中：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.name == "patrick")
...     .values(fullname="Patrick the Star")
...     .returning(user_table.c.id, user_table.c.name)
... )
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=:fullname
WHERE  user_account.name  =  :name_1
RETURNING  user_account.id,  user_account.name
>>> delete_stmt = (
...     delete(user_table)
...     .where(user_table.c.name == "patrick")
...     .returning(user_table.c.id, user_table.c.name)
... )
>>> print(delete_stmt)
DELETE  FROM  user_account
WHERE  user_account.name  =  :name_1
RETURNING  user_account.id,  user_account.name 
```

## 更新、删除的进一步阅读

另请参阅

更新/删除的 API 文档：

+   `更新`

+   `Delete`

ORM 启用的 UPDATE 和 DELETE：

ORM-启用的 INSERT、UPDATE 和 DELETE 语句 - 在 ORM 查询指南 中

## update() SQL 表达式构造

`update()` 函数生成一个新的 `Update` 实例，表示 SQL 中的 UPDATE 语句，将更新表中的现有数据。

像 `insert()` 构造一样，还有一个“传统”形式的 `update()`，它一次针对单个表发出 UPDATE，并且不返回任何行。然而，一些后端支持一种可以一次修改多个表的 UPDATE 语句，并且 UPDATE 语句还支持 RETURNING，以便匹配行中包含的列可以在结果集中返回。

基本的 UPDATE 如下所示：

```py
>>> from sqlalchemy import update
>>> stmt = (
...     update(user_table)
...     .where(user_table.c.name == "patrick")
...     .values(fullname="Patrick the Star")
... )
>>> print(stmt)
UPDATE  user_account  SET  fullname=:fullname  WHERE  user_account.name  =  :name_1 
```

`Update.values()` 方法控制 UPDATE 语句的 SET 元素的内容。这是由 `Insert` 构造共享的相同方法。通常可以使用列名作为关键字参数传递参数。

UPDATE 支持所有主要的 SQL UPDATE 形式，包括针对表达式的更新，我们可以利用 `Column` 表达式：

```py
>>> stmt = update(user_table).values(fullname="Username: " + user_table.c.name)
>>> print(stmt)
UPDATE  user_account  SET  fullname=(:name_1  ||  user_account.name) 
```

为了支持在“executemany”上下文中的 UPDATE，其中将针对同一语句调用许多参数集，可以使用 `bindparam()` 构造来设置绑定参数；这些参数替换了通常放置字面值的位置：

```py
>>> from sqlalchemy import bindparam
>>> stmt = (
...     update(user_table)
...     .where(user_table.c.name == bindparam("oldname"))
...     .values(name=bindparam("newname"))
... )
>>> with engine.begin() as conn:
...     conn.execute(
...         stmt,
...         [
...             {"oldname": "jack", "newname": "ed"},
...             {"oldname": "wendy", "newname": "mary"},
...             {"oldname": "jim", "newname": "jake"},
...         ],
...     )
BEGIN  (implicit)
UPDATE  user_account  SET  name=?  WHERE  user_account.name  =  ?
[...]  [('ed',  'jack'),  ('mary',  'wendy'),  ('jake',  'jim')]
<sqlalchemy.engine.cursor.CursorResult  object  at  0x...>
COMMIT 
```

可应用于 UPDATE 的其他技术包括：

### 相关更新

UPDATE 语句可以通过使用 相关子查询 来使用其他表中的行。子查询可以在任何可以放置列表达式的地方使用：

```py
>>> scalar_subq = (
...     select(address_table.c.email_address)
...     .where(address_table.c.user_id == user_table.c.id)
...     .order_by(address_table.c.id)
...     .limit(1)
...     .scalar_subquery()
... )
>>> update_stmt = update(user_table).values(fullname=scalar_subq)
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=(SELECT  address.email_address
FROM  address
WHERE  address.user_id  =  user_account.id  ORDER  BY  address.id
LIMIT  :param_1) 
```  ### UPDATE..FROM

一些数据库，如 PostgreSQL 和 MySQL，支持“UPDATE FROM”语法，其中额外的表可以直接在特殊的 FROM 子句中声明。当额外的表位于语句的 WHERE 子句中时，将隐式生成此语法：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
...     .values(fullname="Pat")
... )
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=:fullname  FROM  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  :email_address_1 
```

还有一种 MySQL 特定的语法可以更新多个表。这需要在 VALUES 子句中引用`Table`对象，以便引用其他表：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
...     .values(
...         {
...             user_table.c.fullname: "Pat",
...             address_table.c.email_address: "pat@aol.com",
...         }
...     )
... )
>>> from sqlalchemy.dialects import mysql
>>> print(update_stmt.compile(dialect=mysql.dialect()))
UPDATE  user_account,  address
SET  address.email_address=%s,  user_account.fullname=%s
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  %s 
```  ### 参数排序更新

另一个仅适用于 MySQL 的行为是，UPDATE 的 SET 子句中参数的顺序实际上影响每个表达式的评估。对于这种用例，`Update.ordered_values()`方法接受一个元组序列，以便可以控制此顺序 [[2]](#id2)：

```py
>>> update_stmt = update(some_table).ordered_values(
...     (some_table.c.y, 20), (some_table.c.x, some_table.c.y + 10)
... )
>>> print(update_stmt)
UPDATE  some_table  SET  y=:y,  x=(some_table.y  +  :y_1) 
```  ### 相关更新

UPDATE 语句可以通过使用相关子查询中的行来使用其他表中的行。子查询可以在任何可以放置列表达式的地方使用：

```py
>>> scalar_subq = (
...     select(address_table.c.email_address)
...     .where(address_table.c.user_id == user_table.c.id)
...     .order_by(address_table.c.id)
...     .limit(1)
...     .scalar_subquery()
... )
>>> update_stmt = update(user_table).values(fullname=scalar_subq)
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=(SELECT  address.email_address
FROM  address
WHERE  address.user_id  =  user_account.id  ORDER  BY  address.id
LIMIT  :param_1) 
```

### UPDATE..FROM

一些数据库，如 PostgreSQL 和 MySQL，支持“UPDATE FROM”语法，其中额外的表可以直接在特殊的 FROM 子句中声明。当额外的表位于语句的 WHERE 子句中时，此语法将隐式生成：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
...     .values(fullname="Pat")
... )
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=:fullname  FROM  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  :email_address_1 
```

还有一种 MySQL 特定的语法可以更新多个表。这需要在 VALUES 子句中引用`Table`对象，以便引用其他表：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
...     .values(
...         {
...             user_table.c.fullname: "Pat",
...             address_table.c.email_address: "pat@aol.com",
...         }
...     )
... )
>>> from sqlalchemy.dialects import mysql
>>> print(update_stmt.compile(dialect=mysql.dialect()))
UPDATE  user_account,  address
SET  address.email_address=%s,  user_account.fullname=%s
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  %s 
```

### 参数排序更新

另一个仅适用于 MySQL 的行为是，UPDATE 的 SET 子句中参数的顺序实际上影响每个表达式的评估。对于这种用例，`Update.ordered_values()`方法接受一个元组序列，以便可以控制此顺序 [[2]](#id2)：

```py
>>> update_stmt = update(some_table).ordered_values(
...     (some_table.c.y, 20), (some_table.c.x, some_table.c.y + 10)
... )
>>> print(update_stmt)
UPDATE  some_table  SET  y=:y,  x=(some_table.y  +  :y_1) 
```

## delete() SQL 表达式构造

`delete()`函数生成一个新的`Delete`实例，表示 SQL 中的 DELETE 语句，它将从表中删除行。

`delete()`语句从 API 的角度来看与`update()`构造非常相似，传统上不返回任何行，但在一些数据库后端上允许使用 RETURNING 变体。

```py
>>> from sqlalchemy import delete
>>> stmt = delete(user_table).where(user_table.c.name == "patrick")
>>> print(stmt)
DELETE  FROM  user_account  WHERE  user_account.name  =  :name_1 
```

### 多表删除

像`Update`一样，`Delete`支持在 WHERE 子句中使用相关子查询，以及后端特定的多表语法，例如 MySQL 上的`DELETE FROM..USING`：

```py
>>> delete_stmt = (
...     delete(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
... )
>>> from sqlalchemy.dialects import mysql
>>> print(delete_stmt.compile(dialect=mysql.dialect()))
DELETE  FROM  user_account  USING  user_account,  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  %s 
```  ### 多表删除

与`Update`类似，`Delete`也支持在 WHERE 子句中使用相关子查询，以及后端特定的多表语法，例如在 MySQL 上的 `DELETE FROM..USING`：

```py
>>> delete_stmt = (
...     delete(user_table)
...     .where(user_table.c.id == address_table.c.user_id)
...     .where(address_table.c.email_address == "patrick@aol.com")
... )
>>> from sqlalchemy.dialects import mysql
>>> print(delete_stmt.compile(dialect=mysql.dialect()))
DELETE  FROM  user_account  USING  user_account,  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  %s 
```

## 从 UPDATE、DELETE 获取受影响的行数

`Update` 和 `Delete` 都支持在语句执行后返回匹配的行数的功能，对于使用 Core `Connection` 调用的语句，即 `Connection.execute()`。根据下面提到的注意事项，此值可从 `CursorResult.rowcount` 属性中获取：

```py
>>> with engine.begin() as conn:
...     result = conn.execute(
...         update(user_table)
...         .values(fullname="Patrick McStar")
...         .where(user_table.c.name == "patrick")
...     )
...     print(result.rowcount)
BEGIN  (implicit)
UPDATE  user_account  SET  fullname=?  WHERE  user_account.name  =  ?
[...]  ('Patrick McStar',  'patrick')
1
COMMIT 
```

提示

`CursorResult` 类是 `Result` 的子类，它包含特定于 DBAPI `cursor` 对象的其他属性。当通过 `Connection.execute()` 方法调用语句时，将返回此子类的实例。在使用 ORM 时，`Session.execute()` 方法为所有 INSERT、UPDATE 和 DELETE 语句返回此类型的对象。

有关 `CursorResult.rowcount` 的事实：

+   返回的值是由语句的 WHERE 子句**匹配**的行数。无论实际上是否修改了行都无关紧要。

+   `CursorResult.rowcount` 对于使用 RETURNING 的 UPDATE 或 DELETE 语句，或者使用 executemany 执行的语句未必可用。可用性取决于所使用的 DBAPI 模块。

+   在 DBAPI 未确定某种类型语句的行数的任何情况下，返回值都将是 `-1`。

+   SQLAlchemy 在游标关闭之前预先缓存 DBAPIs `cursor.rowcount` 的值，因为某些 DBAPIs 不支持在事后访问此属性。为了为非 UPDATE 或 DELETE 的语句（例如 INSERT 或 SELECT）预先缓存 `cursor.rowcount`，可以使用 `Connection.execution_options.preserve_rowcount` 执行选项。

+   一些驱动程序，特别是非关系数据库的第三方方言，可能根本不支持 `CursorResult.rowcount`。`CursorResult.supports_sane_rowcount` 游标属性将指示这一点。

+   “rowcount” 被 ORM 工作单元 过程用于验证 UPDATE 或 DELETE 语句是否匹配预期的行数，并且还是 ORM 版本控制功能的关键，该功能在 配置版本计数器 中有文档记录。

## 使用 RETURNING 与 UPDATE、DELETE

与 `Insert` 构造相似，`Update` 和 `Delete` 也支持通过使用 `Update.returning()` 和 `Delete.returning()` 方法添加的 RETURNING 子句。当这些方法在支持 RETURNING 的后端上使用时，匹配 WHERE 条件的所有行的选定列将作为可迭代的行返回到 `Result` 对象中：

```py
>>> update_stmt = (
...     update(user_table)
...     .where(user_table.c.name == "patrick")
...     .values(fullname="Patrick the Star")
...     .returning(user_table.c.id, user_table.c.name)
... )
>>> print(update_stmt)
UPDATE  user_account  SET  fullname=:fullname
WHERE  user_account.name  =  :name_1
RETURNING  user_account.id,  user_account.name
>>> delete_stmt = (
...     delete(user_table)
...     .where(user_table.c.name == "patrick")
...     .returning(user_table.c.id, user_table.c.name)
... )
>>> print(delete_stmt)
DELETE  FROM  user_account
WHERE  user_account.name  =  :name_1
RETURNING  user_account.id,  user_account.name 
```

## 关于 UPDATE、DELETE 的进一步阅读

请参阅

UPDATE / DELETE 的 API 文档：

+   `Update`

+   `Delete`

启用 ORM 的 UPDATE 和 DELETE：

ORM 支持的 INSERT、UPDATE 和 DELETE 语句 - 在 ORM 查询指南 中
