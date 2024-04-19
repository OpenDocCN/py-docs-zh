# 使用 SELECT 语句

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/data_select.html`](https://docs.sqlalchemy.org/en/20/tutorial/data_select.html)

对于 Core 和 ORM，`select()` 函数生成一个用于所有 SELECT 查询的 `Select` 构造。传递给 Core 中的 `Connection.execute()` 方法和 ORM 中的 `Session.execute()` 方法，在当前事务中发出 SELECT 语句并通过返回的 `Result` 对象获取结果行。

**ORM 读者** - 这里的内容同样适用于 Core 和 ORM 使用，并提到了基本 ORM 变体用例。然而，还有更多的 ORM 特定功能可用；这些在 ORM 查询指南中有文档记录。

## select() SQL 表达式构造

`select()` 构造以与 `insert()` 相同的方式构建语句，使用 生成式 方法，其中每个方法都会将更多的状态添加到对象上。与其他 SQL 构造一样，它可以在原地字符串化：

```py
>>> from sqlalchemy import select
>>> stmt = select(user_table).where(user_table.c.name == "spongebob")
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  :name_1 
```

与所有其他语句级别的 SQL 构造相同，要实际运行语句，我们将其传递给执行方法。由于 SELECT 语句返回行，我们始终可以迭代结果对象以获取 `Row` 对象返回：

```py
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(row)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('spongebob',)
(1, 'spongebob', 'Spongebob Squarepants')
ROLLBACK 
```

当使用 ORM 时，特别是对 ORM 实体组成的 `select()` 结构执行时，我们将希望使用 `Session.execute()` 方法在 `Session` 上执行它；通过这种方法，我们继续从结果中获取 `Row` 对象，但是这些行现在可以包括完整的实体，例如 `User` 类的实例，作为每行中的单独元素：

```py
>>> stmt = select(User).where(User.name == "spongebob")
>>> with Session(engine) as session:
...     for row in session.execute(stmt):
...         print(row)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('spongebob',)
(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
ROLLBACK 
```

以下各节将更详细地讨论 SELECT 构造。

## 设置 COLUMNS 和 FROM 子句

`select()` 函数接受表示任意数量`Column`和/或`Table`表达式的位置元素，以及一系列兼容对象，这些对象将解析为要从中选择的 SQL 表达式列表，这些表达式将作为结果集中的列返回。这些元素在更简单的情况下还用于创建 FROM 子句，该子句是从传递的列和类似表达式中推断出来的：

```py
>>> print(select(user_table))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account 
```

使用核心方法从单独列进行 SELECT 操作时，可以直接访问`Table.c`访问器中的`Column`对象；FROM 子句将被推断为由这些列所代表的所有`Table`和其他`FromClause`对象的集合：

```py
>>> print(select(user_table.c.name, user_table.c.fullname))
SELECT  user_account.name,  user_account.fullname
FROM  user_account 
```

或者，当使用任何`FromClause`的`FromClause.c`集合，例如`Table`时，可以通过使用字符串名称的元组指定多个列进行`select()`操作：

```py
>>> print(select(user_table.c["name", "fullname"]))
SELECT  user_account.name,  user_account.fullname
FROM  user_account 
```

2.0 版本中新增：为`FromClause.c`集合添加了元组访问器功能

### 选择 ORM 实体和列

ORM 实体，例如我们的`User`类以及其上的列映射属性，例如`User.name`，也参与 SQL 表达语言系统，表示表和列。下面举例说明了从`User`实体中进行 SELECT 操作的示例，最终呈现的方式与直接使用`user_table`相同：

```py
>>> print(select(User))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account 
```

当使用 ORM `Session.execute()`方法执行类似上述语句时，当我们从完整实体（例如`User`）选择时，与`user_table`相反，有一个重要的区别，即**实体本身作为每行的单个元素返回**。也就是说，当我们从上述语句中获取行时，因为在要获取的内容列表中只有`User`实体，所以我们会收到仅包含一个元素的`Row`对象，其中包含`User`类的实例：

```py
>>> row = session.execute(select(User)).first()
BEGIN...
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> row
(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
```

上述`Row`只有一个元素，代表`User`实体：

```py
>>> row[0]
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
```

实现与上述相同结果的一种高度推荐的便利方法是使用`Session.scalars()`方法直接执行语句；此方法将返回一个`ScalarResult`对象，该对象一次性返回每行的第一个“列”，在本例中是`User`类的实例：

```py
>>> user = session.scalars(select(User)).first()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> user
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
```

或者，我们可以使用类绑定的属性选择 ORM 实体的各个列作为结果行中的单独元素；当这些属性传递给诸如`select()`之类的构造时，它们会解析为每个属性代表的`Column`或其他 SQL 表达式：

```py
>>> print(select(User.name, User.fullname))
SELECT  user_account.name,  user_account.fullname
FROM  user_account 
```

当我们使用`Session.execute()`调用*此*语句时，我们现在会收到每个值具有单独元素的行，每个元素对应一个单独的列或其他 SQL 表达式：

```py
>>> row = session.execute(select(User.name, User.fullname)).first()
SELECT  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> row
('spongebob', 'Spongebob Squarepants')
```

可以混合使用这些方法，如下所示，我们选择`User`实体的`name`属性作为行的第一个元素，并将其与完整的`Address`实体组合为第二个元素：

```py
>>> session.execute(
...     select(User.name, Address).where(User.id == Address.user_id).order_by(Address.id)
... ).all()
SELECT  user_account.name,  address.id,  address.email_address,  address.user_id
FROM  user_account,  address
WHERE  user_account.id  =  address.user_id  ORDER  BY  address.id
[...]  ()
[('spongebob', Address(id=1, email_address='spongebob@sqlalchemy.org')),
('sandy', Address(id=2, email_address='sandy@sqlalchemy.org')),
('sandy', Address(id=3, email_address='sandy@squirrelpower.org'))]
```

在选择 ORM 实体和列以及将行转换为常见方法方面的方法进一步讨论。

另请参阅

选择 ORM 实体和列 - 在 ORM 查询指南中

### 从带标签的 SQL 表达式中进行选择

`ColumnElement.label()`方法以及可用于 ORM 属性的同名方法提供列或表达式的 SQL 标签，允许它在结果集中具有特定名称。当通过名称引用结果行中的任意 SQL 表达式时，这可能会有所帮助：

```py
>>> from sqlalchemy import func, cast
>>> stmt = select(
...     ("Username: " + user_table.c.name).label("username"),
... ).order_by(user_table.c.name)
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(f"{row.username}")
BEGIN  (implicit)
SELECT  ?  ||  user_account.name  AS  username
FROM  user_account  ORDER  BY  user_account.name
[...]  ('Username: ',)
Username: patrick
Username: sandy
Username: spongebob
ROLLBACK 
```

另请参阅

按标签排序或分组 - 我们创建的标签名称也可以在`Select`的 ORDER BY 或 GROUP BY 子句中引用。

### 使用文本列表达式进行选择

当我们使用`select()`函数构造一个`Select`对象时，通常会向其中传递一系列使用 table metadata 定义的`Table`和`Column`对象，或者在使用 ORM 时，我们可能会发送代表表列的 ORM 映射属性。然而，有时也需要在语句中制造任意 SQL 块，比如常量字符串表达式，或者一些直接编写的任意 SQL。

在 Working with Transactions and the DBAPI 中介绍的`text()`构造实际上可以直接嵌入到`Select`构造中，如下所示，我们制造了一个硬编码的字符串字面量`'some phrase'`并将其嵌入到 SELECT 语句中：

```py
>>> from sqlalchemy import text
>>> stmt = select(text("'some phrase'"), user_table.c.name).order_by(user_table.c.name)
>>> with engine.connect() as conn:
...     print(conn.execute(stmt).all())
BEGIN  (implicit)
SELECT  'some phrase',  user_account.name
FROM  user_account  ORDER  BY  user_account.name
[generated  in  ...]  ()
[('some phrase', 'patrick'), ('some phrase', 'sandy'), ('some phrase', 'spongebob')]
ROLLBACK 
```

虽然`text()`构造可用于大多数位置来插入文字 SQL 短语，但我们实际上更多地处理的是每个代表单个列表达式的文本单元。在这种常见情况下，我们可以使用`literal_column()`构造来获得更多的功能。此对象类似于`text()`，但它不是表示任意形式的任意 SQL，而是明确表示一个“列”，然后可以在子查询和其他表达式中进行标记和引用：

```py
>>> from sqlalchemy import literal_column
>>> stmt = select(literal_column("'some phrase'").label("p"), user_table.c.name).order_by(
...     user_table.c.name
... )
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(f"{row.p}, {row.name}")
BEGIN  (implicit)
SELECT  'some phrase'  AS  p,  user_account.name
FROM  user_account  ORDER  BY  user_account.name
[generated  in  ...]  ()
some phrase, patrick
some phrase, sandy
some phrase, spongebob
ROLLBACK 
```

请注意，在使用 `text()` 或 `literal_column()` 时，我们正在编写一个语法上的 SQL 表达式，而不是一个字面值。因此，我们必须包括所需的任何引号或语法，以便我们想要看到的 SQL 被呈现出来。## WHERE 子句

SQLAlchemy 允许我们通过使用标准 Python 运算符结合 `Column` 和类似对象来组合 SQL 表达式，例如 `name = 'squidward'` 或 `user_id > 10`。对于布尔表达式，大多数 Python 运算符（如 `==`、`!=`、`<`、`>=` 等）生成新的 SQL 表达式对象，而不是纯粹的布尔 `True`/`False` 值：

```py
>>> print(user_table.c.name == "squidward")
user_account.name = :name_1

>>> print(address_table.c.user_id > 10)
address.user_id > :user_id_1
```

我们可以使用这样的表达式来生成 WHERE 子句，方法是将生成的对象传递给 `Select.where()` 方法：

```py
>>> print(select(user_table).where(user_table.c.name == "squidward"))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  :name_1 
```

要生成由 AND 连接的多个表达式，可以多次调用 `Select.where()` 方法：

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

对于具有相同效果的多个表达式，单次调用 `Select.where()` 也可以接受多个表达式：

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

“AND” 和 “OR” 连接词可以直接使用 `and_()` 和 `or_()` 函数，下面是在 ORM 实体方面的示例：

```py
>>> from sqlalchemy import and_, or_
>>> print(
...     select(Address.email_address).where(
...         and_(
...             or_(User.name == "squidward", User.name == "sandy"),
...             Address.user_id == User.id,
...         )
...     )
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  (user_account.name  =  :name_1  OR  user_account.name  =  :name_2)
AND  address.user_id  =  user_account.id 
```

对于针对单个实体的简单“相等性”比较，还有一种称为 `Select.filter_by()` 的流行方法，它接受与列键或 ORM 属性名称匹配的关键字参数。它将针对最左边的 FROM 子句或最后一个连接的实体进行过滤：

```py
>>> print(select(User).filter_by(name="spongebob", fullname="Spongebob Squarepants"))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  :name_1  AND  user_account.fullname  =  :fullname_1 
```

另请参阅

运算符参考 - SQLAlchemy 中大多数 SQL 运算符函数的描述## 明确的 FROM 子句和 JOINs

正如前面提到的，FROM 子句通常是基于我们在列子句中设置的表达式以及 `Select` 的其他元素而 **推断** 的。

如果我们在 COLUMNS 子句中设置了一个特定 `Table` 的单个列，它也会将该 `Table` 放在 FROM 子句中：

```py
>>> print(select(user_table.c.name))
SELECT  user_account.name
FROM  user_account 
```

如果我们从两个表中取列，那么我们得到一个用逗号分隔的 FROM 子句：

```py
>>> print(select(user_table.c.name, address_table.c.email_address))
SELECT  user_account.name,  address.email_address
FROM  user_account,  address 
```

为了将这两个表 JOIN 在一起，我们通常在 `Select` 上使用两种方法之一。第一种是 `Select.join_from()` 方法，它允许我们明确指示 JOIN 的左侧和右侧：

```py
>>> print(
...     select(user_table.c.name, address_table.c.email_address).join_from(
...         user_table, address_table
...     )
... )
SELECT  user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

另一个是 `Select.join()` 方法，它表示 JOIN 的右侧，左侧被推断：

```py
>>> print(select(user_table.c.name, address_table.c.email_address).join(address_table))
SELECT  user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

如果 FROM 子句没有按照我们想要的方式进行推断，我们还可以选择将元素明确添加到 FROM 子句中。我们使用 `Select.select_from()` 方法来实现这一点，如下所示，我们将 `user_table` 设为 FROM 子句中的第一个元素，然后使用 `Select.join()` 将 `address_table` 设为第二个元素：

```py
>>> print(select(address_table.c.email_address).select_from(user_table).join(address_table))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

我们可能想要使用 `Select.select_from()` 的另一个示例是，如果我们的 columns 子句没有足够的信息提供 FROM 子句。例如，要从常见的 SQL 表达式 `count(*)` 中选择，我们使用名为 `sqlalchemy.sql.expression.func` 的 SQLAlchemy 元素来生成 SQL `count()` 函数：

```py
>>> from sqlalchemy import func
>>> print(select(func.count("*")).select_from(user_table))
SELECT  count(:count_2)  AS  count_1
FROM  user_account 
```

另请参阅

在连接中设置最左侧的 FROM 子句 - 在 ORM 查询指南 - 包含有关 `Select.select_from()` 和 `Select.join()` 互动的附加示例和注释。

### 设置 ON 子句

前面 JOIN 的示例说明了 `Select` 结构可以在两个表之间进行 JOIN，并自动生成 ON 子句。这在这些示例中发生，因为 `user_table` 和 `address_table` `Table` 对象包含单个 `ForeignKeyConstraint` 定义，用于形成此 ON 子句。

如果连接的左右目标没有这样的约束，或者存在多个约束，则需要直接指定 ON 子句。 `Select.join()` 和 `Select.join_from()` 都接受用于 ON 子句的额外参数，其使用与我们在 WHERE 子句 中看到的 SQL 表达式机制相同：

```py
>>> print(
...     select(address_table.c.email_address)
...     .select_from(user_table)
...     .join(address_table, user_table.c.id == address_table.c.user_id)
... )
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

**ORM 提示** - 在使用 ORM 实体时，当使用 `relationship()` 构造时，还有另一种生成 ON 子句的方式，就像在 声明映射类 中的前一节设置的映射一样。这是一个单独的主题，详细介绍在 使用关系连接。

### OUTER 和 FULL join

`Select.join()` 和 `Select.join_from()` 方法都接受关键字参数 `Select.join.isouter` 和 `Select.join.full`，分别会渲染 LEFT OUTER JOIN 和 FULL OUTER JOIN：

```py
>>> print(select(user_table).join(address_table, isouter=True))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  LEFT  OUTER  JOIN  address  ON  user_account.id  =  address.user_id
>>> print(select(user_table).join(address_table, full=True))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  FULL  OUTER  JOIN  address  ON  user_account.id  =  address.user_id 
```

还有一个方法 `Select.outerjoin()`，它等同于使用 `.join(..., isouter=True)`。

提示

SQL 还有一个“RIGHT OUTER JOIN”。SQLAlchemy 不会直接呈现这个；相反，反转表的顺序并使用“LEFT OUTER JOIN”。  ## ORDER BY、GROUP BY、HAVING

SELECT SQL 语句包括一个称为 ORDER BY 的子句，用于以给定顺序返回所选行。

GROUP BY 子句的构造方式类似于 ORDER BY 子句，其目的是将所选行细分为特定的分组，从而可以对这些分组调用聚合函数。HAVING 子句通常与 GROUP BY 一起使用，其形式与 WHERE 子句类似，只是它应用于分组内使用的聚合函数。

### ORDER BY

ORDER BY 子句是根据 SQL 表达式构造的，通常基于`Column`或类似对象。`Select.order_by()`方法按位置接受一个或多个这些表达式：

```py
>>> print(select(user_table).order_by(user_table.c.name))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.name 
```

升序/降序可以从`ColumnElement.asc()`和`ColumnElement.desc()`修饰符中获得，这些修饰符也存在于 ORM 绑定的属性中：

```py
>>> print(select(User).order_by(User.fullname.desc()))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.fullname  DESC 
```

上述语句将按照`user_account.fullname`列按降序排序的行。### 带有 GROUP BY / HAVING 的聚合函数

在 SQL 中，聚合函数允许跨多行的列表达式聚合在一起，以产生单个结果。示例包括计数、计算平均值，以及在一组值中定位最大值或最小值。

SQLAlchemy 以一种开放式的方式提供 SQL 函数，使用一个名为`func`的命名空间。这是一个特殊的构造对象，当给定特定 SQL 函数的名称时，它将创建`Function`的新实例，该函数可以有任何名称，以及零个或多个要传递给函数的参数，就像在所有其他情况下一样，都是 SQL 表达式构造。例如，要针对`user_account.id`列渲染 SQL COUNT()函数，我们调用`count()`名称：

```py
>>> from sqlalchemy import func
>>> count_fn = func.count(user_table.c.id)
>>> print(count_fn)
count(user_account.id) 
```

SQL 函数在本教程的后面部分详细描述，链接在使用 SQL 函数。

在 SQL 中使用聚合函数时，GROUP BY 子句是必不可少的，因为它允许将行分成组，其中聚合函数将分别应用于每个组。在 SELECT 语句的 COLUMNS 子句中请求非聚合列时，SQL 要求这些列都受到 GROUP BY 子句的约束，直接或间接地基于主键关联。然后，HAVING 子句类似于 WHERE 子句，不同之处在于它根据聚合值而不是直接行内容来过滤行。

SQLAlchemy 提供了这两个子句的功能，使用 `Select.group_by()` 和 `Select.having()` 方法。下面我们示例选择用户名称字段以及地址计数，对于那些拥有多个地址的用户：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(
...         select(User.name, func.count(Address.id).label("count"))
...         .join(Address)
...         .group_by(User.name)
...         .having(func.count(Address.id) > 1)
...     )
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name,  count(address.id)  AS  count
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id  GROUP  BY  user_account.name
HAVING  count(address.id)  >  ?
[...]  (1,)
[('sandy', 2)]
ROLLBACK 
```  ### 按标签排序或分组

特别重要的一项技术，在某些数据库后端特别是，是有能力按照已在列子句中已经声明的表达式进行 ORDER BY 或 GROUP BY，而无需在 ORDER BY 或 GROUP BY 子句中重新声明表达式，而是使用 COLUMNS 子句中的列名或标记名。可以通过将名称的字符串文本传递给 `Select.order_by()` 或 `Select.group_by()` 方法来使用这种形式。传递的文本**不会直接渲染**；而是在列子句中给定的表达式名称，并在上下文中呈现为该表达式名称，如果找不到匹配项，则会引发错误。这种形式也可以使用一元修饰符 `asc()` 和 `desc()`：

```py
>>> from sqlalchemy import func, desc
>>> stmt = (
...     select(Address.user_id, func.count(Address.id).label("num_addresses"))
...     .group_by("user_id")
...     .order_by("user_id", desc("num_addresses"))
... )
>>> print(stmt)
SELECT  address.user_id,  count(address.id)  AS  num_addresses
FROM  address  GROUP  BY  address.user_id  ORDER  BY  address.user_id,  num_addresses  DESC 
```  ## 使用别名

现在我们正在从多个表中进行选择并使用连接，我们很快就会遇到需要在语句的 FROM 子句中多次引用同一张表的情况。我们使用 SQL **别名** 来实现这一点，这是一种为表或子查询提供替代名称的语法，可以在语句中引用它。

在 SQLAlchemy 表达语言中，这些“名称”代替了 `FromClause` 对象，被称为 `Alias` 构造，在 Core 中使用 `FromClause.alias()` 方法构造。一个 `Alias` 构造就像一个 `Table` 构造一样，它也有一个在 `Alias.c` 集合中的 `Column` 对象的命名空间。例如下面的 SELECT 语句返回所有唯一的用户名对：

```py
>>> user_alias_1 = user_table.alias()
>>> user_alias_2 = user_table.alias()
>>> print(
...     select(user_alias_1.c.name, user_alias_2.c.name).join_from(
...         user_alias_1, user_alias_2, user_alias_1.c.id > user_alias_2.c.id
...     )
... )
SELECT  user_account_1.name,  user_account_2.name  AS  name_1
FROM  user_account  AS  user_account_1
JOIN  user_account  AS  user_account_2  ON  user_account_1.id  >  user_account_2.id 
```

### ORM 实体别名

ORM 中与 `FromClause.alias()` 方法对应的方法是 ORM `aliased()` 函数，可应用于实体，如 `User` 和 `Address`。这将在内部生成一个 `Alias` 对象，针对原始映射的 `Table` 对象，同时保持 ORM 功能。下面的 SELECT 从 `User` 实体中选择包含两个特定电子邮件地址的所有对象：

```py
>>> from sqlalchemy.orm import aliased
>>> address_alias_1 = aliased(Address)
>>> address_alias_2 = aliased(Address)
>>> print(
...     select(User)
...     .join_from(User, address_alias_1)
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join_from(User, address_alias_2)
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

Tip

如在 设置 ON 子句 中提到的，ORM 提供了使用 `relationship()` 结构进行连接的另一种方式。上述使用别名的示例是使用 `relationship()` 在 使用关系在别名目标之间进行连接 中演示的。## 子查询和 CTE

SQL 中的子查询是在括号内呈现并放置在封闭语句上下文中的 SELECT 语句，通常是 SELECT 语句，但不一定。

本节将介绍所谓的“非标量”子查询，通常放置在封闭 SELECT 的 FROM 子句中。我们还将介绍通用表达式（Common Table Expression，CTE），它与子查询的使用方式类似，但包含其他功能。

SQLAlchemy 使用 `Subquery` 对象表示子查询，使用 `CTE` 表示 CTE，通常分别从 `Select.subquery()` 和 `Select.cte()` 方法获取。这两个对象都可以作为较大的 `select()` 结构中的 FROM 元素使用。

我们可以构造一个 `Subquery` ，将从 `address` 表中选择行的聚合计数（聚合函数和 GROUP BY 在 具有 GROUP BY / HAVING 的聚合函数 中已介绍）：

```py
>>> subq = (
...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
...     .group_by(address_table.c.user_id)
...     .subquery()
... )
```

单独将子查询字符串化，而不将其嵌入到另一个`Select`或其他语句中，会生成不带任何封闭括号的普通 SELECT 语句：

```py
>>> print(subq)
SELECT  count(address.id)  AS  count,  address.user_id
FROM  address  GROUP  BY  address.user_id 
```

`Subquery`对象的行为类似于任何其他 FROM 对象，例如`Table`，特别是它包含一个`Subquery.c`列的命名空间，该命名空间选择它。 我们可以使用此命名空间来引用`user_id`列以及我们的自定义标记的`count`表达式：

```py
>>> print(select(subq.c.user_id, subq.c.count))
SELECT  anon_1.user_id,  anon_1.count
FROM  (SELECT  count(address.id)  AS  count,  address.user_id  AS  user_id
FROM  address  GROUP  BY  address.user_id)  AS  anon_1 
```

通过包含在`subq`对象中的一系列行的选择，我们可以将该对象应用于一个更大的`Select`，将数据连接到`user_account`表：

```py
>>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
...     user_table, subq
... )

>>> print(stmt)
SELECT  user_account.name,  user_account.fullname,  anon_1.count
FROM  user_account  JOIN  (SELECT  count(address.id)  AS  count,  address.user_id  AS  user_id
FROM  address  GROUP  BY  address.user_id)  AS  anon_1  ON  user_account.id  =  anon_1.user_id 
```

为了从`user_account`连接到`address`，我们利用了`Select.join_from()`方法。 正如之前所说明的，此连接的 ON 子句再次基于外键约束**推断**。 即使 SQL 子查询本身没有任何约束，SQLAlchemy 也可以根据列上表示的约束来操作列，从而确定`subq.c.user_id`列**派生自**表达外键关系的`address_table.c.user_id`列，该列又表达了与`user_table.c.id`列的外键关系，然后用于生成 ON 子句。

### 公共表达式（CTEs）

在 SQLAlchemy 中使用`CTE`结构的用法与使用`Subquery`结构几乎相同。 通过将`Select.subquery()`方法的调用更改为使用`Select.cte()`而不是，我们可以像以前一样使用结果对象作为 FROM 元素，但是渲染的 SQL 是非常不同的常用表达式语法：

```py
>>> subq = (
...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
...     .group_by(address_table.c.user_id)
...     .cte()
... )

>>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
...     user_table, subq
... )

>>> print(stmt)
WITH  anon_1  AS
(SELECT  count(address.id)  AS  count,  address.user_id  AS  user_id
FROM  address  GROUP  BY  address.user_id)
  SELECT  user_account.name,  user_account.fullname,  anon_1.count
FROM  user_account  JOIN  anon_1  ON  user_account.id  =  anon_1.user_id 
```

`CTE`结构还具有以“递归”方式使用的能力，并且在更复杂的情况下可以由 INSERT、UPDATE 或 DELETE 语句的 RETURNING 子句组成。 `CTE`的文档字符串包含有关这些附加模式的详细信息。

在这两种情况下，子查询和 CTE 在 SQL 层面上都被命名为“匿名”名称。在 Python 代码中，我们根本不需要提供这些名称。当渲染时，`Subquery` 或 `CTE` 实例的对象标识作为对象的语法标识。可以通过将其作为 `Select.subquery()` 或 `Select.cte()` 方法的第一个参数来提供在 SQL 中呈现的名称。  

参见

`Select.subquery()` - 关于子查询的进一步细节

`Select.cte()` - 包括如何使用 RECURSIVE 以及面向 DML 的 CTE 的示例

### ORM 实体子查询/CTEs

在 ORM 中，`aliased()` 构造可用于将 ORM 实体（例如我们的 `User` 或 `Address` 类）与表示行来源的任何 `FromClause` 概念相关联。前一节 ORM 实体别名 演示了如何使用 `aliased()` 将映射类与其映射的 `Table` 的 `Alias` 相关联。在这里，我们演示了 `aliased()` 对一个 `Subquery` 以及对一个由 `Select` 构造生成的 `CTE` 执行相同操作，最终从相同的映射 `Table` 派生。

下面是将`aliased()` 应用到`Subquery` 构造的示例，以便从其行中提取 ORM 实体。结果显示了一系列`User`和`Address`对象，其中每个`Address`对象的数据最终来自于针对`address`表的子查询，而不是直接来自该表：

```py
>>> subq = select(Address).where(~Address.email_address.like("%@aol.com")).subquery()
>>> address_subq = aliased(Address, subq)
>>> stmt = (
...     select(User, address_subq)
...     .join_from(User, address_subq)
...     .order_by(User.id, address_subq.id)
... )
>>> with Session(engine) as session:
...     for user, address in session.execute(stmt):
...         print(f"{user} {address}")
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.email_address,  anon_1.user_id
FROM  user_account  JOIN
(SELECT  address.id  AS  id,  address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  address.email_address  NOT  LIKE  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.id
[...]  ('%@aol.com',)
User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
ROLLBACK 
```

下面是另一个例子，与之前的例子完全相同，只是它使用了`CTE` 构造：

```py
>>> cte_obj = select(Address).where(~Address.email_address.like("%@aol.com")).cte()
>>> address_cte = aliased(Address, cte_obj)
>>> stmt = (
...     select(User, address_cte)
...     .join_from(User, address_cte)
...     .order_by(User.id, address_cte.id)
... )
>>> with Session(engine) as session:
...     for user, address in session.execute(stmt):
...         print(f"{user} {address}")
BEGIN  (implicit)
WITH  anon_1  AS
(SELECT  address.id  AS  id,  address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  address.email_address  NOT  LIKE  ?)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.email_address,  anon_1.user_id
FROM  user_account
JOIN  anon_1  ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.id
[...]  ('%@aol.com',)
User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
ROLLBACK 
```

另请参阅

从子查询中选择实体 - 在 ORM 查询指南 ## 标量和相关子查询

标量子查询是一个返回零行或一行且一列的子查询。然后，该子查询在包含 SELECT 语句的 COLUMNS 或 WHERE 子句中使用，并且与常规子查询不同之处在于它不在 FROM 子句中使用。相关子查询 是指在包含 SELECT 语句中引用表的标量子查询。

SQLAlchemy 使用`ScalarSelect` 构造来表示标量子查询，该构造是`ColumnElement` 表达式层次结构的一部分，与常规子查询不同，常规子查询由`Subquery` 构造表示，该构造位于`FromClause` 层次结构中。

标量子查询通常与聚合函数一起使用，但不一定要这样，之前在带有 GROUP BY / HAVING 的聚合函数中介绍过。标量子查询通过显式使用`Select.scalar_subquery()` 方法来指示。下面是一个示例，其默认的字符串形式在单独字符串化时呈现为从两个表中选择的普通 SELECT 语句：

```py
>>> subq = (
...     select(func.count(address_table.c.id))
...     .where(user_table.c.id == address_table.c.user_id)
...     .scalar_subquery()
... )
>>> print(subq)
(SELECT  count(address.id)  AS  count_1
FROM  address,  user_account
WHERE  user_account.id  =  address.user_id) 
```

上述`subq`对象现在位于`ColumnElement` SQL 表达式层次结构中，因此它可以像任何其他列表达式一样使用：

```py
>>> print(subq == 5)
(SELECT  count(address.id)  AS  count_1
FROM  address,  user_account
WHERE  user_account.id  =  address.user_id)  =  :param_1 
```

虽然标量子查询本身在自身字符串化时在其 FROM 子句中呈现了`user_account`和`address`，但是，当将其嵌入到处理`user_account`表的封闭`select()`构造中时，`user_account`表会自动**相关联**，这意味着它不会在子查询的 FROM 子句中呈现：

```py
>>> stmt = select(user_table.c.name, subq.label("address_count"))
>>> print(stmt)
SELECT  user_account.name,  (SELECT  count(address.id)  AS  count_1
FROM  address
WHERE  user_account.id  =  address.user_id)  AS  address_count
FROM  user_account 
```

简单的相关子查询通常会执行所需的正确操作。但是，在相关性不明确的情况下，SQLAlchemy 将通知我们需要更清晰：

```py
>>> stmt = (
...     select(
...         user_table.c.name,
...         address_table.c.email_address,
...         subq.label("address_count"),
...     )
...     .join_from(user_table, address_table)
...     .order_by(user_table.c.id, address_table.c.id)
... )
>>> print(stmt)
Traceback (most recent call last):
...
InvalidRequestError: Select statement '<... Select object at ...>' returned
no FROM clauses due to auto-correlation; specify correlate(<tables>) to
control correlation manually.
```

要指定`user_table`是我们要关联的表，我们使用`ScalarSelect.correlate()`或`ScalarSelect.correlate_except()`方法来指定：

```py
>>> subq = (
...     select(func.count(address_table.c.id))
...     .where(user_table.c.id == address_table.c.user_id)
...     .scalar_subquery()
...     .correlate(user_table)
... )
```

然后，该语句可以像处理其他列一样返回此列的数据：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(
...         select(
...             user_table.c.name,
...             address_table.c.email_address,
...             subq.label("address_count"),
...         )
...         .join_from(user_table, address_table)
...         .order_by(user_table.c.id, address_table.c.id)
...     )
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name,  address.email_address,  (SELECT  count(address.id)  AS  count_1
FROM  address
WHERE  user_account.id  =  address.user_id)  AS  address_count
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id  ORDER  BY  user_account.id,  address.id
[...]  ()
[('spongebob', 'spongebob@sqlalchemy.org', 1), ('sandy', 'sandy@sqlalchemy.org', 2),
 ('sandy', 'sandy@squirrelpower.org', 2)]
ROLLBACK 
```

### LATERAL 相关性

LATERAL 相关性是 SQL 相关性的一种特殊子类，它允许可选单元在单个 FROM 子句内引用另一个可选单元。这是一个极其特殊的用例，虽然是 SQL 标准的一部分，但只有最近版本的 PostgreSQL 已知支持。

通常，如果 SELECT 语句在其 FROM 子句中引用了`table1 JOIN (SELECT ...) AS subquery`，则右侧的子查询可能不会引用左侧的“table1”表达式；相关联可能只引用完全包围此 SELECT 的另一个 SELECT 的表。LATERAL 关键字允许我们改变这种行为，允许来自右侧 JOIN 的相关联。

SQLAlchemy 支持使用`Select.lateral()`方法实现此功能，该方法创建一个称为`Lateral`的对象。`Lateral`与`Subquery`和`Alias`位于同一家族，但在将构造添加到封闭 SELECT 的 FROM 子句时还包括相关联行为。以下示例说明了使用 LATERAL 的 SQL 查询，选择“用户帐户/电子邮件地址计数”数据，如前一节所讨论的：

```py
>>> subq = (
...     select(
...         func.count(address_table.c.id).label("address_count"),
...         address_table.c.email_address,
...         address_table.c.user_id,
...     )
...     .where(user_table.c.id == address_table.c.user_id)
...     .lateral()
... )
>>> stmt = (
...     select(user_table.c.name, subq.c.address_count, subq.c.email_address)
...     .join_from(user_table, subq)
...     .order_by(user_table.c.id, subq.c.email_address)
... )
>>> print(stmt)
SELECT  user_account.name,  anon_1.address_count,  anon_1.email_address
FROM  user_account
JOIN  LATERAL  (SELECT  count(address.id)  AS  address_count,
address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  user_account.id  =  address.user_id)  AS  anon_1
ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.email_address 
```

在上述示例中，JOIN 的右侧是一个子查询，它与左侧的`user_account`表相关联。

当使用`Select.lateral()`时，`Select.correlate()` 和 `Select.correlate_except()` 方法的行为也会应用于`Lateral` 结构。

另请参阅

`Lateral`

`Select.lateral()`  ## UNION、UNION ALL 和其他集合操作

在 SQL 中，SELECT 语句可以使用 UNION 或 UNION ALL SQL 操作合并在一起，它产生由一个或多个语句一起产生的所有行的集合。还可以进行其他集合操作，例如 INTERSECT [ALL] 和 EXCEPT [ALL]。

SQLAlchemy 的`Select` 结构支持使用像`union()`、`intersect()` 和 `except_()` 这样的函数进行此类组合，以及“all”对应项 `union_all()`、`intersect_all()` 和 `except_all()`。这些函数都接受任意数量的子可选择项，通常是`Select` 结构，但也可以是现有的组合。

这些函数生成的构造物是 `CompoundSelect`，其使用方式与 `Select` 构造物相同，只是方法较少。例如，由 `union_all()` 生成的 `CompoundSelect` 可以直接使用 `Connection.execute()` 调用：

```py
>>> from sqlalchemy import union_all
>>> stmt1 = select(user_table).where(user_table.c.name == "sandy")
>>> stmt2 = select(user_table).where(user_table.c.name == "spongebob")
>>> u = union_all(stmt1, stmt2)
>>> with engine.connect() as conn:
...     result = conn.execute(u)
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[generated  in  ...]  ('sandy',  'spongebob')
[(2, 'sandy', 'Sandy Cheeks'), (1, 'spongebob', 'Spongebob Squarepants')]
ROLLBACK 
```

要将 `CompoundSelect` 用作子查询，就像 `Select` 一样，它提供了一个 `SelectBase.subquery()` 方法，该方法将生成一个带有 `FromClause.c` 集合的 `Subquery` 对象，该集合可以在封闭的 `select()` 中引用：

```py
>>> u_subq = u.subquery()
>>> stmt = (
...     select(u_subq.c.name, address_table.c.email_address)
...     .join_from(address_table, u_subq)
...     .order_by(u_subq.c.name, address_table.c.email_address)
... )
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  anon_1.name,  address.email_address
FROM  address  JOIN
  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
  FROM  user_account
  WHERE  user_account.name  =  ?
UNION  ALL
  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
  FROM  user_account
  WHERE  user_account.name  =  ?)
AS  anon_1  ON  anon_1.id  =  address.user_id
ORDER  BY  anon_1.name,  address.email_address
[generated  in  ...]  ('sandy',  'spongebob')
[('sandy', 'sandy@sqlalchemy.org'), ('sandy', 'sandy@squirrelpower.org'), ('spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

### 从并集中选择 ORM 实体

前面的例子演示了如何构造一个 UNION，给定两个 `Table` 对象，然后返回数据库行。如果我们想要使用 UNION 或其他集合操作来选择行，然后将其作为 ORM 对象接收，有两种方法可以使用。在这两种情况下，我们首先构造一个 `select()` 或 `CompoundSelect` 对象，该对象表示我们想要执行的 SELECT / UNION / 等语句；这个语句应该针对目标 ORM 实体或它们的基础映射 `Table` 对象组成：

```py
>>> stmt1 = select(User).where(User.name == "sandy")
>>> stmt2 = select(User).where(User.name == "spongebob")
>>> u = union_all(stmt1, stmt2)
```

对于一个简单的 SELECT 和 UNION，如果它还没有嵌套在子查询中，那么可以经常在 ORM 对象获取的上下文中使用`Select.from_statement()`方法。通过这种方法，UNION 语句表示整个查询；在使用`Select.from_statement()`之后，不能添加额外的条件：

```py
>>> orm_stmt = select(User).from_statement(u)
>>> with Session(engine) as session:
...     for obj in session.execute(orm_stmt).scalars():
...         print(obj)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?  UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[generated  in  ...]  ('sandy',  'spongebob')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
ROLLBACK 
```

要以更灵活的方式将 UNION 或其他集合相关的构造用作实体相关组件，可以使用`CompoundSelect`构造将其组织到一个子查询中，然后使用`aliased()`函数将其链接到 ORM 对象。这与在 ORM 实体子查询/CTEs 中引入的方式相同，首先创建我们想要的实体到子查询的临时“映射”，然后从新实体中选择，就像它是任何其他映射类一样。在下面的示例中，我们可以添加额外的条件，比如在 UNION 之外进行 ORDER BY，因为我们可以过滤或按子查询导出的列进行排序：

```py
>>> user_alias = aliased(User, u.subquery())
>>> orm_stmt = select(user_alias).order_by(user_alias.id)
>>> with Session(engine) as session:
...     for obj in session.execute(orm_stmt).scalars():
...         print(obj)
BEGIN  (implicit)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.name  =  ?  UNION  ALL  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.name  =  ?)  AS  anon_1  ORDER  BY  anon_1.id
[generated  in  ...]  ('sandy',  'spongebob')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
ROLLBACK 
```

另请参阅

从 UNIONs 和其他集合操作中选择实体 - 在 ORM 查询指南中的 ## EXISTS 子查询

SQL EXISTS 关键字是与标量子查询一起使用的运算符，根据 SELECT 语句是否返回行来返回布尔值 true 或 false。SQLAlchemy 包含一个称为`ScalarSelect`的对象变体，它将生成一个 EXISTS 子查询，并且最方便地使用`SelectBase.exists()`方法生成。下面我们生成一个 EXISTS，以便我们可以返回`user_account`中有多个相关行的行：

```py
>>> subq = (
...     select(func.count(address_table.c.id))
...     .where(user_table.c.id == address_table.c.user_id)
...     .group_by(address_table.c.user_id)
...     .having(func.count(address_table.c.id) > 1)
... ).exists()
>>> with engine.connect() as conn:
...     result = conn.execute(select(user_table.c.name).where(subq))
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name
FROM  user_account
WHERE  EXISTS  (SELECT  count(address.id)  AS  count_1
FROM  address
WHERE  user_account.id  =  address.user_id  GROUP  BY  address.user_id
HAVING  count(address.id)  >  ?)
[...]  (1,)
[('sandy',)]
ROLLBACK 
```

EXISTS 构造更常用于否定，例如 NOT EXISTS，因为它提供了一种 SQL 效率高的形式来定位一个相关表没有行的行。下面我们选择没有电子邮件地址的用户名称；注意第二个 WHERE 子句中使用的二进制否定运算符 (`~`)：

```py
>>> subq = (
...     select(address_table.c.id).where(user_table.c.id == address_table.c.user_id)
... ).exists()
>>> with engine.connect() as conn:
...     result = conn.execute(select(user_table.c.name).where(~subq))
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name
FROM  user_account
WHERE  NOT  (EXISTS  (SELECT  address.id
FROM  address
WHERE  user_account.id  =  address.user_id))
[...]  ()
[('patrick',)]
ROLLBACK 
```  ## 使用 SQL 函数

在本节早些时候介绍的 带 GROUP BY / HAVING 的聚合函数，`func` 对象充当创建新的 `Function` 对象的工厂，当在像 `select()` 这样的结构中使用时，会产生一个 SQL 函数显示，通常由名称、一些括号（虽然不总是），以及可能的一些参数组成。典型的 SQL 函数示例包括：

+   `count()` 函数，计算返回的行数的聚合函数：

    ```py
    >>> print(select(func.count()).select_from(user_table))
    SELECT  count(*)  AS  count_1
    FROM  user_account 
    ```

+   `lower()` 函数，将字符串转换为小写的字符串函数：

    ```py
    >>> print(select(func.lower("A String With Much UPPERCASE")))
    SELECT  lower(:lower_2)  AS  lower_1 
    ```

+   `now()` 函数，提供当前日期和时间；由于这是一个常见的函数，SQLAlchemy 知道如何为每个后端呈现这个函数的不同表现形式，在 SQLite 中使用 CURRENT_TIMESTAMP 函数：

    ```py
    >>> stmt = select(func.now())
    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     print(result.all())
    BEGIN  (implicit)
    SELECT  CURRENT_TIMESTAMP  AS  now_1
    [...]  ()
    [(datetime.datetime(...),)]
    ROLLBACK 
    ```

由于大多数数据库后端都具有几十甚至上百种不同的 SQL 函数，`func` 尽可能宽松地接受任何输入。从这个命名空间访问的任何名称都自动被视为是一个 SQL 函数，将以一种通用的方式呈现：

```py
>>> print(select(func.some_crazy_function(user_table.c.name, 17)))
SELECT  some_crazy_function(user_account.name,  :some_crazy_function_2)  AS  some_crazy_function_1
FROM  user_account 
```

与此同时，一组相对较小的极其常见的 SQL 函数，如`count`、`now`、`max`、`concat` 包括它们自己的预打包版本，这些版本提供了适当的类型信息，并在某些情况下提供特定于后端的 SQL 生成。下面的示例对比了 PostgreSQL 方言和 Oracle 方言对 `now` 函数的 SQL 生成：

```py
>>> from sqlalchemy.dialects import postgresql
>>> print(select(func.now()).compile(dialect=postgresql.dialect()))
SELECT  now()  AS  now_1
>>> from sqlalchemy.dialects import oracle
>>> print(select(func.now()).compile(dialect=oracle.dialect()))
SELECT  CURRENT_TIMESTAMP  AS  now_1  FROM  DUAL 
```

### 函数具有返回类型

由于函数是列表达式，它们还有 SQL 数据类型，描述了生成的 SQL 表达式的数据类型。我们在这里将这些类型称为“SQL 返回类型”，指的是在数据库端 SQL 表达式上下文中由函数返回的 SQL 值的类型，而不是 Python 函数的“返回类型”。

任何 SQL 函数的 SQL 返回类型可以通过引用 `Function.type` 属性来访问，通常用于调试目的：

```py
>>> func.now().type
DateTime()
```

这些 SQL 返回类型在将函数表达式用于更大表达式的上下文中时很重要；也就是说，数学运算符在表达式的数据类型为`Integer`或`Numeric`之类时效果更佳，为了使 JSON 访问器能够工作，需要使用诸如`JSON`之类的类型。某些类别的函数返回整行而不是列值，需要引用特定列；这些函数被称为 table valued functions。

在执行语句并获取行时，函数的 SQL 返回类型也可能很重要，特别是对于那些 SQLAlchemy 必须应用结果集处理的情况。一个典型的例子是 SQLite 上的日期相关函数，其中 SQLAlchemy 的`DateTime`和相关数据类型在收到结果行时扮演了将字符串值转换为 Python `datetime()`对象的角色。

要将特定类型应用于我们创建的函数，我们使用`Function.type_`参数传递它；类型参数可以是`TypeEngine`类或实例。在下面的示例中，我们传递`JSON`类以生成 PostgreSQL 的`json_object()`函数，注意 SQL 返回类型将是 JSON 类型：

```py
>>> from sqlalchemy import JSON
>>> function_expr = func.json_object('{a, 1, b, "def", c, 3.5}', type_=JSON)
```

通过使用带有`JSON`数据类型的 JSON 函数，SQL 表达式对象具有了与 JSON 相关的功能，例如访问元素：

```py
>>> stmt = select(function_expr["def"])
>>> print(stmt)
SELECT  json_object(:json_object_1)[:json_object_2]  AS  anon_1 
```

### 内置函数具有预配置的返回类型

对于像`count`、`max`、`min`等常见的聚合函数，以及非常少数的日期函数，比如`now`和字符串函数，比如`concat`，SQL 返回类型将适当地设置，有时是基于用法。`max`函数和类似的聚合过滤函数将根据给定的参数设置 SQL 返回类型：

```py
>>> m1 = func.max(Column("some_int", Integer))
>>> m1.type
Integer()

>>> m2 = func.max(Column("some_str", String))
>>> m2.type
String()
```

日期和时间函数通常对应于由`DateTime`、`Date`或`Time`描述的 SQL 表达式：

```py
>>> func.now().type
DateTime()
>>> func.current_date().type
Date()
```

已知的字符串函数，如`concat`，将知道 SQL 表达式的类型为`String`：

```py
>>> func.concat("x", "y").type
String()
```

但是，对于绝大多数 SQL 函数，SQLAlchemy 并没有将它们显式地列在已知函数的非常小的列表中。例如，虽然通常使用 SQL 函数 `func.lower()` 和 `func.upper()` 来转换字符串的大小写没有问题，但 SQLAlchemy 实际上并不知道这些函数，因此它们具有“null”SQL 返回类型：

```py
>>> func.upper("lowercase").type
NullType()
```

对于像`upper`和`lower`这样的简单函数，通常情况下问题不是很严重，因为字符串值可以在没有任何特殊类型处理的情况下从数据库接收，而且 SQLAlchemy 的类型转换规则通常也能够正确猜测意图；例如，Python 的`+`操作符会根据表达式的两边正确解释为字符串连接操作符：

```py
>>> print(select(func.upper("lowercase") + " suffix"))
SELECT  upper(:upper_1)  ||  :upper_2  AS  anon_1 
```

总的来说，`Function.type_` 参数可能是必要的情况是：

1.  如果函数不是 SQLAlchemy 的内置函数；这可以通过创建函数并观察`Function.type`属性来证明，即：

    ```py
    >>> func.count().type
    Integer()
    ```

    与：

    ```py
    >>> func.json_object('{"a", "b"}').type
    NullType()
    ```

1.  需要支持函数感知的表达式；这通常是指与诸如`JSON`或`ARRAY`之类的数据类型相关的特殊操作符

1.  需要结果值处理，其中可能包括诸如`DateTime`、`Boolean`、`Enum`或者再次是特殊的数据类型，如`JSON`、`ARRAY`。

### 高级 SQL 函数技术

以下各小节说明了可以使用 SQL 函数做的更多事情。虽然这些技术比基本的 SQL 函数使用更不常见且更高级，但它们仍然非常受欢迎，这在很大程度上是由于 PostgreSQL 强调更复杂的函数形式，包括与 JSON 数据流行的表和列值形式。

#### 使用窗口函数

窗口函数是 SQL 聚合函数的特殊用法，它在处理个别结果行时计算在一组中返回的行上的聚合值。而像 `MAX()` 这样的函数将为你提供一组行中的列的最高值，使用相同函数作为“窗口函数”将为你提供每行的最高值，*截至该行*。

在 SQL 中，窗口函数允许指定应该应用函数的行、一个考虑不同行子集的“分区”值以及一个重要的指示行应该应用到聚合函数的顺序的“order by”表达式。

在 SQLAlchemy 中，由 `func` 命名空间生成的所有 SQL 函数都包括一个 `FunctionElement.over()` 方法，它授予窗口函数或“OVER”语法；生成的结构是 `Over` 结构。

与窗口函数一起常用的函数是 `row_number()` 函数，它简单地计算行数。我们可以根据用户名对此行计数进行分区，以为个别用户的电子邮件地址编号：

```py
>>> stmt = (
...     select(
...         func.row_number().over(partition_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  row_number()  OVER  (PARTITION  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(1, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (1, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

上文中，使用了 `FunctionElement.over.partition_by` 参数，以使 `PARTITION BY` 子句在 OVER 子句中呈现。我们还可以使用 `FunctionElement.over.order_by` 来使用 `ORDER BY` 子句：

```py
>>> stmt = (
...     select(
...         func.count().over(order_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(*)  OVER  (ORDER  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(2, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (3, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

窗口函数的更多选项包括使用范围；请参阅 `over()` 以获取更多示例。

提示

需要注意的是，`FunctionElement.over()` 方法仅适用于那些实际上是聚合函数的 SQL 函数；虽然 `Over` 结构会愉快地为任何给定的 SQL 函数渲染自己，但如果函数本身不是 SQL 聚合函数，数据库将拒绝该表达式。  #### 特殊修饰符 WITHIN GROUP, FILTER

"WITHIN GROUP" SQL 语法与“有序集合”或“假设集合”聚合函数一起使用。常见的“有序集合”函数包括`percentile_cont()`和`rank()`。SQLAlchemy 包含内置实现`rank`, `dense_rank`, `mode`, `percentile_cont` 和 `percentile_disc`，其中包括一个 `FunctionElement.within_group()` 方法：

```py
>>> print(
...     func.unnest(
...         func.percentile_disc([0.25, 0.5, 0.75, 1]).within_group(user_table.c.name)
...     )
... )
unnest(percentile_disc(:percentile_disc_1)  WITHIN  GROUP  (ORDER  BY  user_account.name)) 
```

"FILTER" 受一些后端支持，用于将聚合函数的范围限制为与返回的总行范围相比的特定子集，可使用 `FunctionElement.filter()` 方法：

```py
>>> stmt = (
...     select(
...         func.count(address_table.c.email_address).filter(user_table.c.name == "sandy"),
...         func.count(address_table.c.email_address).filter(
...             user_table.c.name == "spongebob"
...         ),
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_1,
count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_2
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ('sandy',  'spongebob')
[(2, 1)]
ROLLBACK 
```  #### 表值函数

表值 SQL 函数支持包含命名子元素的标量表示形式。通常用于 JSON 和 ARRAY 导向的函数以及像`generate_series()`这样的函数，表值函数在 FROM 子句中指定，然后被引用为表，有时甚至作为列。这种形式的函数在 PostgreSQL 数据库中非常突出，但某些形式的表值函数也受 SQLite、Oracle 和 SQL Server 支持。

另请参阅

表值、表和列值函数、行和元组对象 - 在 PostgreSQL 文档中。

虽然许多数据库支持表值和其他特殊形式，但 PostgreSQL 往往是对这些功能需求最大的地方。请参阅本节，了解 PostgreSQL 语法的附加示例以及其他功能。

SQLAlchemy 提供了 `FunctionElement.table_valued()` 方法作为基本的“表值函数”构造，它将一个 `func` 对象转换为一个包含一系列命名列的 FROM 子句，这些列是基于按位置传递的字符串名称。这将返回一个 `TableValuedAlias` 对象，它是一个启用函数的 `Alias` 构造，可像在 使用别名 中介绍的其他 FROM 子句一样使用。下面我们举例说明 `json_each()` 函数，尽管在 PostgreSQL 上很常见，但也受到现代版本的 SQLite 的支持：

```py
>>> onetwothree = func.json_each('["one", "two", "three"]').table_valued("value")
>>> stmt = select(onetwothree).where(onetwothree.c.value.in_(["two", "three"]))
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     result.all()
BEGIN  (implicit)
SELECT  anon_1.value
FROM  json_each(?)  AS  anon_1
WHERE  anon_1.value  IN  (?,  ?)
[...]  ('["one", "two", "three"]',  'two',  'three')
[('two',), ('three',)]
ROLLBACK 
```

在上面的例子中，我们使用了 SQLite 和 PostgreSQL 支持的 `json_each()` JSON 函数来生成一个具有单列（称为 `value`）的表值表达式，并选择了其三行中的两行。

另请参阅

表值函数 - 在 PostgreSQL 文档中 - 此部分将详细介绍其他语法，例如特殊列派生和“WITH ORDINALITY”，已知可与 PostgreSQL 一起使用。

PostgreSQL 和 Oracle 支持的特殊语法是在 FROM 子句中引用函数，然后将其自身作为 SELECT 语句或其他列表达式上的列传递到列子句中。 PostgreSQL 在 `json_array_elements()`、`json_object_keys()`、`json_each_text()`、`json_each()` 等函数中广泛使用此语法。

SQLAlchemy 将其称为“列值函数”，可通过将 `FunctionElement.column_valued()` 修饰符应用于 `Function` 构造来使用：

```py
>>> from sqlalchemy import select, func
>>> stmt = select(func.json_array_elements('["one", "two"]').column_valued("x"))
>>> print(stmt)
SELECT  x
FROM  json_array_elements(:json_array_elements_1)  AS  x 
```

“列值形式”也受到 Oracle 方言的支持，可以用于自定义 SQL 函数：

```py
>>> from sqlalchemy.dialects import oracle
>>> stmt = select(func.scalar_strings(5).column_valued("s"))
>>> print(stmt.compile(dialect=oracle.dialect()))
SELECT  s.COLUMN_VALUE
FROM  TABLE  (scalar_strings(:scalar_strings_1))  s 
```

另请参阅

列值函数 - 在 PostgreSQL 文档中。 ## 数据转换和类型强制

在 SQL 中，我们经常需要明确指定表达式的数据类型，要么是为了告诉数据库在一个否则模棱两可的表达式中期望的类型是什么，要么是在某些情况下，当我们想要将 SQL 表达式的隐含数据类型转换为其他内容时。SQL CAST 关键字用于此任务，在 SQLAlchemy 中由`cast()`函数提供。该函数接受列表达式和数据类型对象作为参数，如下所示，我们从`user_table.c.id`列对象生成一个 SQL 表达式`CAST(user_account.id AS VARCHAR)`：

```py
>>> from sqlalchemy import cast
>>> stmt = select(cast(user_table.c.id, String))
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     result.all()
BEGIN  (implicit)
SELECT  CAST(user_account.id  AS  VARCHAR)  AS  id
FROM  user_account
[...]  ()
[('1',), ('2',), ('3',)]
ROLLBACK 
```

`cast()`函数不仅会渲染 SQL CAST 语法，还会生成一个 SQLAlchemy 列表达式，在 Python 端也将作为给定的数据类型。将字符串表达式`cast()`到`JSON`将获得 JSON 下标和比较运算符，例如：

```py
>>> from sqlalchemy import JSON
>>> print(cast("{'a': 'b'}", JSON)["a"])
CAST(:param_1  AS  JSON)[:param_2] 
```

### `type_coerce()` - 一个仅限于 Python 的“类型转换”函数

有时需要让 SQLAlchemy 知道表达式的数据类型，出于前述所有原因，但是不要在 SQL 端渲染 CAST 表达式本身，因为它可能会干扰已经正常工作的 SQL 操作。对于这种相当常见的用例，有另一个函数`type_coerce()`，它与`cast()`密切相关，它将设置一个 Python 表达式为具有特定 SQL 数据库类型，但不会在数据库端渲染 CAST 关键字或数据类型。当处理`JSON`数据类型时，`type_coerce()`特别重要，它通常与不同平台上的字符串定向数据类型有着错综复杂的关系，甚至可能不是一个显式的数据类型，例如在 SQLite 和 MariaDB 上。下面，我们使用`type_coerce()`将一个 Python 结构作为 JSON 字符串传递给 MySQL 的一个 JSON 函数：

```py
>>> import json
>>> from sqlalchemy import JSON
>>> from sqlalchemy import type_coerce
>>> from sqlalchemy.dialects import mysql
>>> s = select(type_coerce({"some_key": {"foo": "bar"}}, JSON)["some_key"])
>>> print(s.compile(dialect=mysql.dialect()))
SELECT  JSON_EXTRACT(%s,  %s)  AS  anon_1 
```

在上面的例子中，调用了 MySQL 的 `JSON_EXTRACT` SQL 函数，因为我们使用 `type_coerce()` 指示我们的 Python 字典应该被视为 `JSON`。Python 的 `__getitem__` 运算符，在这种情况下，`['some_key']` 变得可用，并允许一个 `JSON_EXTRACT` 路径表达式（但在本例中没有显示，最终将是 `'$."some_key"'`）被渲染。

## 选择（`select()`）SQL 表达式构造

`select()` 构造以与 `insert()` 相同的方式构建语句，使用一种生成式方法，其中每个方法都向对象添加更多状态。与其他 SQL 构造一样，它可以在原地转换为字符串：

```py
>>> from sqlalchemy import select
>>> stmt = select(user_table).where(user_table.c.name == "spongebob")
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  :name_1 
```

同样，与所有其他语句级 SQL 构造一样，要实际运行语句，我们将其传递给执行方法。由于 SELECT 语句返回行，我们总是可以迭代结果对象以获取 `Row` 对象：

```py
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(row)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('spongebob',)
(1, 'spongebob', 'Spongebob Squarepants')
ROLLBACK 
```

当使用 ORM，特别是对 ORM 实体组成的 `select()` 构造进行操作时，我们将希望使用 `Session.execute()` 方法执行它；使用这种方法，我们仍然从结果中获取 `Row` 对象，但是这些行现在可以包括完整的实体，例如 `User` 类的实例，作为每一行中的单独元素：

```py
>>> stmt = select(User).where(User.name == "spongebob")
>>> with Session(engine) as session:
...     for row in session.execute(stmt):
...         print(row)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('spongebob',)
(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
ROLLBACK 
```

下面的章节将更详细地讨论 SELECT 构造。

## 设置 COLUMNS 和 FROM 子句

`select()` 函数接受表示任意数量的 `Column` 和/或 `Table` 表达式的位置元素，以及一系列兼容的对象，这些对象被解析为要从中选择的 SQL 表达式列表，将作为结果集中的列返回。这些元素还在更简单的情况下用于创建 FROM 子句，该子句从传递的列和类似表达式中推断出：

```py
>>> print(select(user_table))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account 
```

使用核心方法进行按列选择时，从`Table.c`访问器中访问`Column`对象，并且可以直接发送；FROM 子句将被推断为所有由这些列表示的`Table`和其他`FromClause`对象的集合：

```py
>>> print(select(user_table.c.name, user_table.c.fullname))
SELECT  user_account.name,  user_account.fullname
FROM  user_account 
```

或者，当使用任何`FromClause`的`FromClause.c`集合时，可以通过使用字符串名称的元组指定`select()`的多列：

```py
>>> print(select(user_table.c["name", "fullname"]))
SELECT  user_account.name,  user_account.fullname
FROM  user_account 
```

版本 2.0 中的新功能：为`FromClause.c`集合添加了元组访问器功能

### 选择 ORM 实体和列

ORM 实体，例如我们的`User`类以及其上的列映射属性，例如`User.name`，也参与到表示表和列的 SQL 表达式语言系统中。下面举例说明了从`User`实体中进行 SELECT 的示例，这最终的渲染方式与直接使用`user_table`相同：

```py
>>> print(select(User))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account 
```

当使用 ORM `Session.execute()`方法执行类似上面的语句时，当我们从完整实体如`User`中选择时，与`user_table`相反，有一个重要的区别，即**实体本身作为每行中的单个元素返回**。也就是说，当我们从上述语句中提取行时，由于要提取的东西列表中只有`User`实体，我们会得到仅有一个元素的`Row`对象，其中包含`User`类的实例：

```py
>>> row = session.execute(select(User)).first()
BEGIN...
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> row
(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
```

上述`Row`仅有一个元素，代表着`User`实体：

```py
>>> row[0]
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
```

实现与上述相同结果的一个强烈推荐的便利方法是使用 `Session.scalars()` 方法直接执行该语句；该方法将返回一个 `ScalarResult` 对象，该对象一次提供每行的第一个“列”，在本例中为 `User` 类的实例：

```py
>>> user = session.scalars(select(User)).first()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> user
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
```

或者，我们可以选择 ORM 实体的单独列作为结果行中的不同元素，通过使用类绑定的属性；当这些属性传递给 `select()` 这样的构造时，它们会解析为每个属性所表示的 `Column` 或其他 SQL 表达式：

```py
>>> print(select(User.name, User.fullname))
SELECT  user_account.name,  user_account.fullname
FROM  user_account 
```

当我们使用 `Session.execute()` 调用 *此* 语句时，我们现在会收到每个值都有独立元素的行，每个元素对应于一个单独的列或其他 SQL 表达式：

```py
>>> row = session.execute(select(User.name, User.fullname)).first()
SELECT  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> row
('spongebob', 'Spongebob Squarepants')
```

这些方法也可以混合使用，如下所示，我们将 `User` 实体的 `name` 属性选择为行的第一个元素，并将其与完整的 `Address` 实体组合在第二个元素中：

```py
>>> session.execute(
...     select(User.name, Address).where(User.id == Address.user_id).order_by(Address.id)
... ).all()
SELECT  user_account.name,  address.id,  address.email_address,  address.user_id
FROM  user_account,  address
WHERE  user_account.id  =  address.user_id  ORDER  BY  address.id
[...]  ()
[('spongebob', Address(id=1, email_address='spongebob@sqlalchemy.org')),
('sandy', Address(id=2, email_address='sandy@sqlalchemy.org')),
('sandy', Address(id=3, email_address='sandy@squirrelpower.org'))]
```

进一步讨论了选择 ORM 实体和列以及将行转换为常见方法的方法，请参阅 选择 ORM 实体和属性。

另请参阅

选择 ORM 实体和属性 - 在 ORM 查询指南 中

### 从带标签的 SQL 表达式中选择

`ColumnElement.label()` 方法以及可用于 ORM 属性的同名方法提供了列或表达式的 SQL 标签，允许在结果集中使用特定名称引用任意 SQL 表达式。当通过名称引用结果行中的任意 SQL 表达式时，这可能会有所帮助：

```py
>>> from sqlalchemy import func, cast
>>> stmt = select(
...     ("Username: " + user_table.c.name).label("username"),
... ).order_by(user_table.c.name)
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(f"{row.username}")
BEGIN  (implicit)
SELECT  ?  ||  user_account.name  AS  username
FROM  user_account  ORDER  BY  user_account.name
[...]  ('Username: ',)
Username: patrick
Username: sandy
Username: spongebob
ROLLBACK 
```

另请参阅

按标签排序或分组 - 我们创建的标签名称也可以在 `Select` 的 ORDER BY 或 GROUP BY 子句中引用。

### 使用文本列表达式进行选择

当我们使用`select()`函数构造一个`Select`对象时，通常会传递一系列使用表元数据定义的`Table`和`Column`对象，或者在使用 ORM 时，我们可能会发送表示表列的 ORM 映射属性。然而，有时也需要在语句内部制造任意 SQL 块，比如常量字符串表达式，或者一些更容易直接写的任意 SQL。

在处理事务和 DBAPI 中介绍的`text()`构造实际上可以直接嵌入到`Select`构造中，例如下面我们制造一个硬编码的字符串字面值 `'some phrase'` 并将其嵌入到 SELECT 语句中：

```py
>>> from sqlalchemy import text
>>> stmt = select(text("'some phrase'"), user_table.c.name).order_by(user_table.c.name)
>>> with engine.connect() as conn:
...     print(conn.execute(stmt).all())
BEGIN  (implicit)
SELECT  'some phrase',  user_account.name
FROM  user_account  ORDER  BY  user_account.name
[generated  in  ...]  ()
[('some phrase', 'patrick'), ('some phrase', 'sandy'), ('some phrase', 'spongebob')]
ROLLBACK 
```

虽然`text()`构造可以用于大多数地方来注入字面 SQL 词组，但更多时候，我们实际上处理的是每个表示单独列表达式的文本单元。在这种常见情况下，我们可以使用`literal_column()`构造来获得更多文本片段的功能。该对象类似于`text()`，只是它不是表示任意形式的任意 SQL，而是明确表示一个“列”，然后可以在子查询和其他表达式中标记和引用：

```py
>>> from sqlalchemy import literal_column
>>> stmt = select(literal_column("'some phrase'").label("p"), user_table.c.name).order_by(
...     user_table.c.name
... )
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(f"{row.p}, {row.name}")
BEGIN  (implicit)
SELECT  'some phrase'  AS  p,  user_account.name
FROM  user_account  ORDER  BY  user_account.name
[generated  in  ...]  ()
some phrase, patrick
some phrase, sandy
some phrase, spongebob
ROLLBACK 
```

注意，在使用`text()`或`literal_column()`时，我们正在编写一个语法上的 SQL 表达式，而不是一个字面值。因此，我们必须包含所需的引号或语法，以便渲染我们想要看到的 SQL。  ### 选择 ORM 实体和列

ORM 实体，如我们的`User`类以及其上的列映射属性，如`User.name`，也参与 SQL 表达式语言系统，表示表和列。下面演示了从`User`实体中选择的示例，这最终呈现的方式与直接使用`user_table`时相同：

```py
>>> print(select(User))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account 
```

当使用 ORM `Session.execute()`方法执行类似上述的语句时，当我们从完整实体（如`User`）中选择时，与`user_table`相比，有一个重要的区别，即**实体本身作为每行中的单个元素返回**。也就是说，当我们从上述语句中提取行时，由于要提取的内容列表中只有`User`实体，因此我们得到的是仅包含一个元素的`Row`对象，其中包含`User`类的实例：

```py
>>> row = session.execute(select(User)).first()
BEGIN...
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> row
(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),)
```

上述`Row`只有一个元素，表示`User`实体：

```py
>>> row[0]
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
```

实现与上述相同结果的一个强烈推荐的便捷方法是使用`Session.scalars()`方法直接执行语句；此方法将返回一个`ScalarResult`对象，一次传递每行的第一个“列”，在本例中是`User`类的实例：

```py
>>> user = session.scalars(select(User)).first()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> user
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
```

或者，我们可以选择 ORM 实体的各个列作为结果行中的不同元素，方法是使用类绑定的属性；当这些属性传递给诸如`select()`的构造时，它们将解析为每个属性表示的`Column`或其他 SQL 表达式：

```py
>>> print(select(User.name, User.fullname))
SELECT  user_account.name,  user_account.fullname
FROM  user_account 
```

当我们使用`Session.execute()`调用*此*语句时，现在我们会收到每个值对应的单独元素的行，每个元素对应一个单独的列或其他 SQL 表达式：

```py
>>> row = session.execute(select(User.name, User.fullname)).first()
SELECT  user_account.name,  user_account.fullname
FROM  user_account
[...]  ()
>>> row
('spongebob', 'Spongebob Squarepants')
```

这些方法也可以混合使用，如下所示，我们选择`User`实体的`name`属性作为行的第一个元素，并将其与完整的`Address`实体组合成第二个元素：

```py
>>> session.execute(
...     select(User.name, Address).where(User.id == Address.user_id).order_by(Address.id)
... ).all()
SELECT  user_account.name,  address.id,  address.email_address,  address.user_id
FROM  user_account,  address
WHERE  user_account.id  =  address.user_id  ORDER  BY  address.id
[...]  ()
[('spongebob', Address(id=1, email_address='spongebob@sqlalchemy.org')),
('sandy', Address(id=2, email_address='sandy@sqlalchemy.org')),
('sandy', Address(id=3, email_address='sandy@squirrelpower.org'))]
```

关于选择 ORM 实体和列的方法以及将行转换为常见方法的进一步讨论，请参阅选择 ORM 实体和属性。

另请参阅

选择 ORM 实体和属性 - 在 ORM 查询指南中

### 从带标签的 SQL 表达式中进行选择

`ColumnElement.label()`方法以及 ORM 属性上提供的同名方法都提供了列或表达式的 SQL 标签，允许它在结果集中具有特定名称。当通过名称引用结果行中的任意 SQL 表达式时，这可能会有所帮助：

```py
>>> from sqlalchemy import func, cast
>>> stmt = select(
...     ("Username: " + user_table.c.name).label("username"),
... ).order_by(user_table.c.name)
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(f"{row.username}")
BEGIN  (implicit)
SELECT  ?  ||  user_account.name  AS  username
FROM  user_account  ORDER  BY  user_account.name
[...]  ('Username: ',)
Username: patrick
Username: sandy
Username: spongebob
ROLLBACK 
```

另请参阅

按标签排序或分组 - 我们创建的标签名称也可以在`Select`的 ORDER BY 或 GROUP BY 子句中引用。

### 使用文本列表达式进行选择

当我们使用`select()`函数构造一个`Select`对象时，通常会向其传递使用表元数据定义的`Table`和`Column`对象，或者在使用 ORM 时，可能会发送代表表列的 ORM 映射属性。然而，有时也需要在语句中制造任意 SQL 块，例如常量字符串表达式，或者只是一些更快以文字形式编写的任意 SQL。

在处理事务和 DBAPI 中介绍的`text()`构造实际上可以直接嵌入到`Select`构造中，例如下面我们制造了一个硬编码的字符串字面量`'some phrase'`并将其嵌入到 SELECT 语句中：

```py
>>> from sqlalchemy import text
>>> stmt = select(text("'some phrase'"), user_table.c.name).order_by(user_table.c.name)
>>> with engine.connect() as conn:
...     print(conn.execute(stmt).all())
BEGIN  (implicit)
SELECT  'some phrase',  user_account.name
FROM  user_account  ORDER  BY  user_account.name
[generated  in  ...]  ()
[('some phrase', 'patrick'), ('some phrase', 'sandy'), ('some phrase', 'spongebob')]
ROLLBACK 
```

虽然`text()`构造可以在大多数地方用于插入文字 SQL 短语，但更常见的情况是我们实际上正在处理每个代表单独列表达式的文本单元。在这种常见情况下，我们可以使用`literal_column()`构造从我们的文本片段中获得更多功能。这个对象类似于`text()`，只是它不代表任意形式的任意 SQL，而是明确表示一个“列”，然后可以在子查询和其他表达式中进行标记和引用：

```py
>>> from sqlalchemy import literal_column
>>> stmt = select(literal_column("'some phrase'").label("p"), user_table.c.name).order_by(
...     user_table.c.name
... )
>>> with engine.connect() as conn:
...     for row in conn.execute(stmt):
...         print(f"{row.p}, {row.name}")
BEGIN  (implicit)
SELECT  'some phrase'  AS  p,  user_account.name
FROM  user_account  ORDER  BY  user_account.name
[generated  in  ...]  ()
some phrase, patrick
some phrase, sandy
some phrase, spongebob
ROLLBACK 
```

注意，在使用 `text()` 或 `literal_column()` 时，我们正在编写一种句法 SQL 表达式，而不是文字值。因此，我们必须包括所需的引号或语法以获取我们想要看到的 SQL。

## WHERE 子句

SQLAlchemy 允许我们通过使用 `Column` 和类似对象结合标准 Python 运算符来组合 SQL 表达式，例如 `name = 'squidward'` 或 `user_id > 10`。对于布尔表达式，大多数 Python 运算符，如 `==`、`!=`、`<`、`>=` 等，都会生成新的 SQL 表达式对象，而不是纯粹的布尔 `True`/`False` 值：

```py
>>> print(user_table.c.name == "squidward")
user_account.name = :name_1

>>> print(address_table.c.user_id > 10)
address.user_id > :user_id_1
```

我们可以使用这些表达式来通过将生成的对象传递给 `Select.where()` 方法来生成 WHERE 子句：

```py
>>> print(select(user_table).where(user_table.c.name == "squidward"))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  :name_1 
```

要生成由 AND 连接的多个表达式，可以多次调用 `Select.where()` 方法：

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

单次调用 `Select.where()` 也可以接受多个表达式，效果相同：

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

“AND” 和 “OR” 连接词都可以直接使用 `and_()` 和 `or_()` 函数，在 ORM 实体方面的示例如下所示：

```py
>>> from sqlalchemy import and_, or_
>>> print(
...     select(Address.email_address).where(
...         and_(
...             or_(User.name == "squidward", User.name == "sandy"),
...             Address.user_id == User.id,
...         )
...     )
... )
SELECT  address.email_address
FROM  address,  user_account
WHERE  (user_account.name  =  :name_1  OR  user_account.name  =  :name_2)
AND  address.user_id  =  user_account.id 
```

对于针对单个实体的简单“相等性”比较，还有一种常用方法称为 `Select.filter_by()`，它接受与列键或 ORM 属性名称匹配的关键字参数。它将针对最左边的 FROM 子句或最后一个已连接的实体进行过滤：

```py
>>> print(select(User).filter_by(name="spongebob", fullname="Spongebob Squarepants"))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  :name_1  AND  user_account.fullname  =  :fullname_1 
```

另请参阅

操作符参考 - SQLAlchemy 中大多数 SQL 操作符函数的描述

## 明确的 FROM 子句和 JOINs

如前所述，FROM 子句通常是根据我们在列子句中设置的表达式以及 `Select` 的其他元素来 **推断** 的。

如果我们在 COLUMNS 子句中从特定的 `Table` 中设置单个列，则它也将该 `Table` 放入 FROM 子句中：

```py
>>> print(select(user_table.c.name))
SELECT  user_account.name
FROM  user_account 
```

如果我们要将两个表的列放在一起，那么我们会得到一个用逗号分隔的 FROM 子句：

```py
>>> print(select(user_table.c.name, address_table.c.email_address))
SELECT  user_account.name,  address.email_address
FROM  user_account,  address 
```

为了将这两个表 JOIN 在一起，我们通常在 `Select` 上使用以下两种方法之一。第一种是 `Select.join_from()` 方法，它允许我们明确指定 JOIN 的左右两边：

```py
>>> print(
...     select(user_table.c.name, address_table.c.email_address).join_from(
...         user_table, address_table
...     )
... )
SELECT  user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

另一个是 `Select.join()` 方法，它仅指示 JOIN 的右侧，左侧将被推断：

```py
>>> print(select(user_table.c.name, address_table.c.email_address).join(address_table))
SELECT  user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

我们还可以选择显式地向 FROM 子句添加元素，如果它没有从列子句中以我们希望的方式推断出来。我们使用 `Select.select_from()` 方法来实现这一点，如下所示，我们将 `user_table` 建立为 FROM 子句中的第一个元素，使用 `Select.join()` 来建立 `address_table` 作为第二个元素：

```py
>>> print(select(address_table.c.email_address).select_from(user_table).join(address_table))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

另一个我们可能想要使用 `Select.select_from()` 的示例是，如果我们的列子句没有足够的信息来提供 FROM 子句。例如，要从常见的 SQL 表达式 `count(*)` 中选择，我们使用 SQLAlchemy 元素 `sqlalchemy.sql.expression.func` 来生成 SQL `count()` 函数：

```py
>>> from sqlalchemy import func
>>> print(select(func.count("*")).select_from(user_table))
SELECT  count(:count_2)  AS  count_1
FROM  user_account 
```

另请参阅

设置 JOIN 中最左边的 FROM 子句 - 在 ORM 查询指南 中 - 包含有关 `Select.select_from()` 和 `Select.join()` 交互的其他示例和注意事项。

### 设置 ON 子句

之前 JOIN 的示例说明了 `Select` 结构可以在两个表之间进行 JOIN 并自动生成 ON 子句。这是因为 `user_table` 和 `address_table` `Table` 对象包含单个 `ForeignKeyConstraint` 定义，用于形成此 ON 子句。

如果连接的左右目标没有这样的约束，或者有多个约束存在，我们需要直接指定 ON 子句。`Select.join()`和`Select.join_from()`都接受额外的参数用于 ON 子句，这是使用与我们在 WHERE 子句中看到的相同的 SQL 表达式机制来陈述的：

```py
>>> print(
...     select(address_table.c.email_address)
...     .select_from(user_table)
...     .join(address_table, user_table.c.id == address_table.c.user_id)
... )
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

**ORM 提示** - 在使用 ORM 实体生成 ON 子句时，还有另一种方法，这些实体使用了`relationship()`构造，就像在声明映射类的上一节中设置的映射一样。这是一个单独的主题，详细介绍在使用关系连接。

### OUTER 和 FULL 连接

`Select.join()`和`Select.join_from()`方法都接受关键字参数`Select.join.isouter`和`Select.join.full`，分别渲染 LEFT OUTER JOIN 和 FULL OUTER JOIN：

```py
>>> print(select(user_table).join(address_table, isouter=True))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  LEFT  OUTER  JOIN  address  ON  user_account.id  =  address.user_id
>>> print(select(user_table).join(address_table, full=True))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  FULL  OUTER  JOIN  address  ON  user_account.id  =  address.user_id 
```

还有一种方法`Select.outerjoin()`，它等同于使用`.join(..., isouter=True)`。

提示

SQL 也有“RIGHT OUTER JOIN”。SQLAlchemy 不直接渲染这个；相反，倒转表的顺序并使用“LEFT OUTER JOIN”。

### 设置 ON 子句

之前的 JOIN 示例说明了`Select`构造可以在两个表之间进行连接并自动产生 ON 子句。这在那些示例中发生是因为`user_table`和`address_table`的`Table`对象包括一个单一的`ForeignKeyConstraint`定义，该定义用于形成这个 ON 子句。

如果连接的左右目标没有这样的约束，或者存在多个约束，我们需要直接指定 ON 子句。`Select.join()` 和 `Select.join_from()` 都接受 ON 子句的额外参数，该参数使用与我们在 WHERE 子句 中看到的相同的 SQL 表达式机制进行说明：

```py
>>> print(
...     select(address_table.c.email_address)
...     .select_from(user_table)
...     .join(address_table, user_table.c.id == address_table.c.user_id)
... )
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

**ORM 提示** - 当使用 ORM 实体并使用 `relationship()` 构造时，还有另一种生成 ON 子句的方法，就像前一节中在 声明映射类 中设置的映射一样。这是一个整体主题，详细介绍在 使用关系进行连接。

### OUTER 和 FULL 连接

`Select.join()` 和 `Select.join_from()` 方法都接受关键字参数 `Select.join.isouter` 和 `Select.join.full`，分别对应 LEFT OUTER JOIN 和 FULL OUTER JOIN：

```py
>>> print(select(user_table).join(address_table, isouter=True))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  LEFT  OUTER  JOIN  address  ON  user_account.id  =  address.user_id
>>> print(select(user_table).join(address_table, full=True))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  FULL  OUTER  JOIN  address  ON  user_account.id  =  address.user_id 
```

还有一个等同于使用 `.join(..., isouter=True)` 的方法 `Select.outerjoin()`。

提示

SQL 还有一个“RIGHT OUTER JOIN”。SQLAlchemy 不会直接渲染这个，而是将表的顺序反转并使用“LEFT OUTER JOIN”。

## ORDER BY, GROUP BY, HAVING

SELECT SQL 语句包含一个叫做 ORDER BY 的子句，用于按照给定的顺序返回所选行。

GROUP BY 子句的构造方式类似于 ORDER BY 子句，其目的是将所选行分成特定的组，以便对这些组中的聚合函数进行调用。HAVING 子句通常与 GROUP BY 一起使用，其形式与 WHERE 子句类似，只是应用于组内使用的聚合函数。

### ORDER BY

ORDER BY 子句是根据 SQL 表达式构造的，通常基于`Column`或类似对象。`Select.order_by()`方法可以按位置接受一个或多个这些表达式：

```py
>>> print(select(user_table).order_by(user_table.c.name))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.name 
```

升序/降序可以从`ColumnElement.asc()`和`ColumnElement.desc()`修饰符中获得，这些修饰符也存在于 ORM 绑定属性中：

```py
>>> print(select(User).order_by(User.fullname.desc()))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.fullname  DESC 
```

上述语句将按照`user_account.fullname`列的降序排序。### 带有 GROUP BY / HAVING 的聚合函数

在 SQL 中，聚合函数允许将多行的列表达式聚合在一起，以产生单个结果。示例包括计数、计算平均值，以及定位一组值中的最大或最小值。

SQLAlchemy 以一种开放式的方式提供了 SQL 函数，使用了一个名为`func`的命名空间。这是一个特殊的构造对象，当给出特定 SQL 函数的名称时，它将创建`Function`的新实例，该函数可以具有任何名称，以及零个或多个要传递给函数的参数，这些参数像所有其他情况一样是 SQL 表达式构造。例如，要针对`user_account.id`列渲染 SQL COUNT() 函数，我们调用`count()`名称：

```py
>>> from sqlalchemy import func
>>> count_fn = func.count(user_table.c.id)
>>> print(count_fn)
count(user_account.id) 
```

SQL 函数将在本教程后面的使用 SQL 函数中详细描述。

在 SQL 中使用聚合函数时，GROUP BY 子句至关重要，因为它允许将行分成组，其中将对每个组单独应用聚合函数。在 SELECT 语句的 COLUMNS 子句中请求非聚合列时，SQL 要求这些列都受到 GROUP BY 子句的约束，直接或间接地基于主键关联。然后，HAVING 子句类似于 WHERE 子句，但其根据聚合值而不是直接行内容来过滤行。

SQLAlchemy 提供了使用 `Select.group_by()` 和 `Select.having()` 方法的这两个子句。下面我们演示选择用户姓名字段以及地址数量，对于那些拥有多个地址的用户：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(
...         select(User.name, func.count(Address.id).label("count"))
...         .join(Address)
...         .group_by(User.name)
...         .having(func.count(Address.id) > 1)
...     )
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name,  count(address.id)  AS  count
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id  GROUP  BY  user_account.name
HAVING  count(address.id)  >  ?
[...]  (1,)
[('sandy', 2)]
ROLLBACK 
```  ### 按标签分组或排序

一种重要的技术，特别是在某些数据库后端上，是有能力按已在列子句中已经说明的表达式排序或分组，而不需要在 ORDER BY 或 GROUP BY 子句中重新说明该表达式，而是使用 COLUMNS 子句中的列名或标签名。通过将名称的字符串文本传递给`Select.order_by()` 或 `Select.group_by()` 方法来实现此形式。传递的文本**不会直接呈现**；而是在上下文中以该表达式名称的形式呈现，并在没有找到匹配项时引发错误。这种形式还可以使用一元修饰符`asc()` 和 `desc()`。

```py
>>> from sqlalchemy import func, desc
>>> stmt = (
...     select(Address.user_id, func.count(Address.id).label("num_addresses"))
...     .group_by("user_id")
...     .order_by("user_id", desc("num_addresses"))
... )
>>> print(stmt)
SELECT  address.user_id,  count(address.id)  AS  num_addresses
FROM  address  GROUP  BY  address.user_id  ORDER  BY  address.user_id,  num_addresses  DESC 
```  ### 按顺序排列

`ORDER BY` 子句是根据通常基于`Column` 或类似对象的 SQL 表达式构造的。 `Select.order_by()` 方法按位置接受一个或多个这些表达式：

```py
>>> print(select(user_table).order_by(user_table.c.name))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.name 
```

升序 / 降序可以从`ColumnElement.asc()` 和 `ColumnElement.desc()` 修饰符中获得，这些修饰符也存在于 ORM 绑定的属性中：

```py
>>> print(select(User).order_by(User.fullname.desc()))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.fullname  DESC 
```

上述语句将按 `user_account.fullname` 列按降序排列的行。

### 带有 GROUP BY / HAVING 的聚合函数

在 SQL 中，聚合函数允许跨多行的列表达式聚合在一起以产生单个结果。例子包括计数、计算平均值，以及查找一组值中的最大值或最小值。

SQLAlchemy 以一种开放的方式提供 SQL 函数，使用一个名为`func`的命名空间。这是一个特殊的构造对象，当给定特定 SQL 函数的名称时，它将创建`Function`的新实例，该函数可以具有任何名称，以及零个或多个要传递给函数的参数，就像在所有其他情况下一样，是 SQL 表达式构造。例如，要针对`user_account.id`列渲染 SQL COUNT()函数，我们调用`count()`名称：

```py
>>> from sqlalchemy import func
>>> count_fn = func.count(user_table.c.id)
>>> print(count_fn)
count(user_account.id) 
```

SQL 函数在本教程的稍后部分使用 SQL 函数中有更详细的描述。

在 SQL 中使用聚合函数时，GROUP BY 子句是必不可少的，因为它允许将行分成组，其中聚合函数将分别应用于每个组。在 SELECT 语句的 COLUMNS 子句中请求非聚合列时，SQL 要求这些列都受到 GROUP BY 子句的约束，直接或间接地基于主键关联。然后，HAVING 子句类似于 WHERE 子句的使用方式，不同之处在于它根据聚合值而不是直接行内容来过滤行。

SQLAlchemy 提供了这两个子句，使用`Select.group_by()`和`Select.having()`方法。下面我们展示选择用户名称字段以及地址计数，对于那些拥有多个地址的用户：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(
...         select(User.name, func.count(Address.id).label("count"))
...         .join(Address)
...         .group_by(User.name)
...         .having(func.count(Address.id) > 1)
...     )
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name,  count(address.id)  AS  count
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id  GROUP  BY  user_account.name
HAVING  count(address.id)  >  ?
[...]  (1,)
[('sandy', 2)]
ROLLBACK 
```

### 按标签排序或分组

一种重要的技术，特别是在某些数据库后端上，是能够按照已在列子句中声明的表达式进行 ORDER BY 或 GROUP BY，而无需在 ORDER BY 或 GROUP BY 子句中重新声明表达式，而是使用 COLUMNS 子句中的列名或标记名称。通过将名称的字符串文本传递给`Select.order_by()`或`Select.group_by()`方法来实现这种形式。传递的文本**不会直接呈现**；相反，在列子句中给定表达式的名称，并在上下文中呈现为该表达式名称，如果找不到匹配项，则会引发错误。一元修饰符`asc()`和`desc()`也可以在此形式中使用：

```py
>>> from sqlalchemy import func, desc
>>> stmt = (
...     select(Address.user_id, func.count(Address.id).label("num_addresses"))
...     .group_by("user_id")
...     .order_by("user_id", desc("num_addresses"))
... )
>>> print(stmt)
SELECT  address.user_id,  count(address.id)  AS  num_addresses
FROM  address  GROUP  BY  address.user_id  ORDER  BY  address.user_id,  num_addresses  DESC 
```

## 使用别名

现在我们正在从多个表中进行选择并使用连接，我们很快就会遇到需要在语句的 FROM 子句中多次引用同一张表的情况。我们通过使用 SQL **别名** 来实现这一点，别名是一种为表或子查询提供替代名称的语法，可以在语句中引用它。

在 SQLAlchemy 表达式语言中，这些“名称”实际上是由称为`FromClause`的对象表示的，它们构成了 Core 中的`Alias`构造，该构造使用`FromClause.alias()`方法构建。`Alias`构造就像`Table`构造一样，它也有一个`Column`对象的命名空间，位于`Alias.c`集合中。下面的 SELECT 语句例如返回所有唯一的用户名对：

```py
>>> user_alias_1 = user_table.alias()
>>> user_alias_2 = user_table.alias()
>>> print(
...     select(user_alias_1.c.name, user_alias_2.c.name).join_from(
...         user_alias_1, user_alias_2, user_alias_1.c.id > user_alias_2.c.id
...     )
... )
SELECT  user_account_1.name,  user_account_2.name  AS  name_1
FROM  user_account  AS  user_account_1
JOIN  user_account  AS  user_account_2  ON  user_account_1.id  >  user_account_2.id 
```

### ORM 实体别名

`FromClause.alias()`方法的 ORM 等效方法是 ORM `aliased()`函数，它可以应用于诸如 `User` 和 `Address` 等实体。这将在内部生成一个针对原始映射`Table`对象的`Alias`对象，同时保持 ORM 功能。下面的 SELECT 从 `User` 实体中选择所有包含两个特定电子邮件地址的对象：

```py
>>> from sqlalchemy.orm import aliased
>>> address_alias_1 = aliased(Address)
>>> address_alias_2 = aliased(Address)
>>> print(
...     select(User)
...     .join_from(User, address_alias_1)
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join_from(User, address_alias_2)
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

提示

如 设置 ON 子句 中所述，ORM 还提供了另一种使用 `relationship()` 构造进行连接的方法。使用别名的上述示例在使用关系将别名目标连接起来中使用了 `relationship()`。 ### ORM 实体别名

`FromClause.alias()` 方法的 ORM 等效方法是 ORM `aliased()` 函数，可以应用于实体，如 `User` 和 `Address`。这在内部产生一个 `Alias` 对象，针对原始映射的 `Table` 对象，同时保持 ORM 功能。下面的 SELECT 从 `User` 实体中选择包含两个特定电子邮件地址的所有对象：

```py
>>> from sqlalchemy.orm import aliased
>>> address_alias_1 = aliased(Address)
>>> address_alias_2 = aliased(Address)
>>> print(
...     select(User)
...     .join_from(User, address_alias_1)
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join_from(User, address_alias_2)
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

提示

如 设置 ON 子句 中所述，ORM 提供了另一种使用 `relationship()` 构造连接的方式。上面使用别名的示例是在 使用关系连接别名目标 中使用 `relationship()` 进行演示的。

## 子查询和公共表达式

SQL 中的子查询是一个放在括号中并放置在封闭语句上下文中的 SELECT 语句，通常是一个 SELECT 语句，但不一定是这样。

本节将涵盖所谓的“非标量”子查询，通常放置在封闭 SELECT 的 FROM 子句中。我们还将介绍所谓的公共表达式或 CTE，它与子查询类似，但包括其他功能。

SQLAlchemy 使用 `Subquery` 对象来表示子查询，使用 `CTE` 来表示公共表达式，通常可以通过 `Select.subquery()` 和 `Select.cte()` 方法获取。这两种对象都可以作为一个更大的 `select()` 结构中的 FROM 元素。

我们可以构造一个 `Subquery`，它将从 `address` 表中选择行的聚合计数（聚合函数和 GROUP BY 在之前的 带有 GROUP BY / HAVING 的聚合函数 中介绍过）：

```py
>>> subq = (
...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
...     .group_by(address_table.c.user_id)
...     .subquery()
... )
```

仅将子查询字符串化而不将其嵌入到另一个 `Select` 或其他语句中会产生不包含任何括号的普通 SELECT 语句：

```py
>>> print(subq)
SELECT  count(address.id)  AS  count,  address.user_id
FROM  address  GROUP  BY  address.user_id 
```

`Subquery` 对象的行为类似于任何其他 FROM 对象，比如 `Table`，特别是它包含一个 `Subquery.c` 命名空间，其中包括它所选择的列。我们可以使用此命名空间来引用 `user_id` 列以及我们自定义标记的 `count` 表达式：

```py
>>> print(select(subq.c.user_id, subq.c.count))
SELECT  anon_1.user_id,  anon_1.count
FROM  (SELECT  count(address.id)  AS  count,  address.user_id  AS  user_id
FROM  address  GROUP  BY  address.user_id)  AS  anon_1 
```

通过在 `subq` 对象中包含的行的选择，我们可以将对象应用于较大的 `Select` ，该对象将数据连接到 `user_account` 表：

```py
>>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
...     user_table, subq
... )

>>> print(stmt)
SELECT  user_account.name,  user_account.fullname,  anon_1.count
FROM  user_account  JOIN  (SELECT  count(address.id)  AS  count,  address.user_id  AS  user_id
FROM  address  GROUP  BY  address.user_id)  AS  anon_1  ON  user_account.id  =  anon_1.user_id 
```

为了从 `user_account` 连接到 `address`，我们使用了 `Select.join_from()` 方法。正如之前所说明的，此连接的 ON 子句再次是根据外键约束 **推断** 出来的。即使 SQL 子查询本身没有任何约束，SQLAlchemy 也可以根据在列上表示的约束来处理列上的约束，确定 `subq.c.user_id` 列是 **派生** 自 `address_table.c.user_id` 列，后者表示与 `user_table.c.id` 列的外键关系，然后用于生成 ON 子句。

### 通用表达式（CTEs）

SQLAlchemy 中使用 `CTE` 构造的用法与使用 `Subquery` 构造的用法几乎相同。通过将 `Select.subquery()` 方法的调用更改为使用 `Select.cte()` ，我们可以以相同的方式使用生成的对象作为 FROM 元素，但所呈现的 SQL 是非常不同的常规表达式语法：

```py
>>> subq = (
...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
...     .group_by(address_table.c.user_id)
...     .cte()
... )

>>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
...     user_table, subq
... )

>>> print(stmt)
WITH  anon_1  AS
(SELECT  count(address.id)  AS  count,  address.user_id  AS  user_id
FROM  address  GROUP  BY  address.user_id)
  SELECT  user_account.name,  user_account.fullname,  anon_1.count
FROM  user_account  JOIN  anon_1  ON  user_account.id  =  anon_1.user_id 
```

`CTE` 构造还具有以“递归”样式使用的能力，并且在更复杂的情况下可能由 INSERT、UPDATE 或 DELETE 语句的 RETURNING 子句组成。`CTE` 的文档字符串包含有关这些额外模式的详细信息。

在这两种情况下，子查询和 CTE 在 SQL 层面上都使用“匿名”名称命名。在 Python 代码中，我们根本不需要提供这些名称。当渲染时，`Subquery` 或 `CTE` 实例的对象标识充当对象的语法标识。在 SQL 中将要呈现的名称可以通过将其作为 `Select.subquery()` 或 `Select.cte()` 方法的第一个参数传递来提供。

另见

`Select.subquery()` - 关于子查询的进一步细节

`Select.cte()` - 包括如何使用 RECURSIVE 以及面向 DML 的 CTE 的示例

### ORM 实体子查询/CTEs

在 ORM 中，`aliased()` 构造可用于将 ORM 实体（例如我们的 `User` 或 `Address` 类）与任何表示行源的 `FromClause` 概念相关联。前面的部分 ORM 实体别名 演示了如何使用 `aliased()` 将映射类与其映射的 `Table` 的 `Alias` 关联起来。这里我们演示了 `aliased()` 对同一个映射的 `Table` 生成的 `Select` 构造的 `Subquery` 和 `CTE` 进行相同操作。

以下是将`aliased()`应用于`Subquery`构造的示例，以便可以从其行中提取 ORM 实体。结果显示了一系列`User`和`Address`对象，其中每个`Address`对象的数据最终来自对`address`表的子查询，而不是直接来自该表：

```py
>>> subq = select(Address).where(~Address.email_address.like("%@aol.com")).subquery()
>>> address_subq = aliased(Address, subq)
>>> stmt = (
...     select(User, address_subq)
...     .join_from(User, address_subq)
...     .order_by(User.id, address_subq.id)
... )
>>> with Session(engine) as session:
...     for user, address in session.execute(stmt):
...         print(f"{user} {address}")
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.email_address,  anon_1.user_id
FROM  user_account  JOIN
(SELECT  address.id  AS  id,  address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  address.email_address  NOT  LIKE  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.id
[...]  ('%@aol.com',)
User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
ROLLBACK 
```

另一个例子如下，除了它使用了`CTE`构造之外，其他完全相同：

```py
>>> cte_obj = select(Address).where(~Address.email_address.like("%@aol.com")).cte()
>>> address_cte = aliased(Address, cte_obj)
>>> stmt = (
...     select(User, address_cte)
...     .join_from(User, address_cte)
...     .order_by(User.id, address_cte.id)
... )
>>> with Session(engine) as session:
...     for user, address in session.execute(stmt):
...         print(f"{user} {address}")
BEGIN  (implicit)
WITH  anon_1  AS
(SELECT  address.id  AS  id,  address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  address.email_address  NOT  LIKE  ?)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.email_address,  anon_1.user_id
FROM  user_account
JOIN  anon_1  ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.id
[...]  ('%@aol.com',)
User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
ROLLBACK 
```

另请参阅

从子查询中选择实体 - 在 ORM 查询指南

### 公共表达式（CTEs）

使用`CTE`构造在 SQLAlchemy 中的使用方式与`Subquery`构造几乎相同。将`Select.subquery()`方法的调用更改为使用`Select.cte()`，我们可以以相同的方式将生成的对象用作 FROM 元素，但所呈现的 SQL 语法是非常不同的通用表达式语法：

```py
>>> subq = (
...     select(func.count(address_table.c.id).label("count"), address_table.c.user_id)
...     .group_by(address_table.c.user_id)
...     .cte()
... )

>>> stmt = select(user_table.c.name, user_table.c.fullname, subq.c.count).join_from(
...     user_table, subq
... )

>>> print(stmt)
WITH  anon_1  AS
(SELECT  count(address.id)  AS  count,  address.user_id  AS  user_id
FROM  address  GROUP  BY  address.user_id)
  SELECT  user_account.name,  user_account.fullname,  anon_1.count
FROM  user_account  JOIN  anon_1  ON  user_account.id  =  anon_1.user_id 
```

`CTE`构造还具有以“递归”方式使用的能力，并且在更复杂的情况下可以从 INSERT、UPDATE 或 DELETE 语句的 RETURNING 子句组成。`CTE`的文档字符串包含了有关这些附加模式的详细信息。

在这两种情况下，子查询和 CTE 都在 SQL 级别使用“匿名”名称命名。在 Python 代码中，我们根本不需要提供这些名称。当呈现时，`Subquery`或`CTE`实例的对象标识作为对象的句法标识。可以通过将其作为`Select.subquery()`或`Select.cte()`方法的第一个参数传递来提供将在 SQL 中呈现的名称。

另请参阅

`Select.subquery()` - 关于子查询的更多细节

`Select.cte()` - CTE 的示例，包括如何使用 RECURSIVE 以及面向 DML 的 CTE 的示例

### ORM 实体子查询/CTEs

在 ORM 中，`aliased()` 构造可用于将 ORM 实体（例如我们的 `User` 或 `Address` 类）与代表行来源的任何 `FromClause` 概念关联起来。上一节 ORM 实体别名 说明了如何使用 `aliased()` 将映射类与其映射的 `Table` 的 `Alias` 关联起来。这里我们说明了 `aliased()` 如何对一个 `Subquery` 以及一个针对从同一映射的 `Table` 派生的 `Select` 构造的 `CTE` 进行相同的操作。

下面是将 `aliased()` 应用于 `Subquery` 构造的示例，以便从其行中提取 ORM 实体。结果显示了一系列 `User` 和 `Address` 对象，其中每个 `Address` 对象的数据最终来自于对 `address` 表的子查询，而不是直接来自该表：

```py
>>> subq = select(Address).where(~Address.email_address.like("%@aol.com")).subquery()
>>> address_subq = aliased(Address, subq)
>>> stmt = (
...     select(User, address_subq)
...     .join_from(User, address_subq)
...     .order_by(User.id, address_subq.id)
... )
>>> with Session(engine) as session:
...     for user, address in session.execute(stmt):
...         print(f"{user} {address}")
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.email_address,  anon_1.user_id
FROM  user_account  JOIN
(SELECT  address.id  AS  id,  address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  address.email_address  NOT  LIKE  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.id
[...]  ('%@aol.com',)
User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
ROLLBACK 
```

另一个例子如下，与之前的例子完全相同，只是使用了 `CTE` 构造：

```py
>>> cte_obj = select(Address).where(~Address.email_address.like("%@aol.com")).cte()
>>> address_cte = aliased(Address, cte_obj)
>>> stmt = (
...     select(User, address_cte)
...     .join_from(User, address_cte)
...     .order_by(User.id, address_cte.id)
... )
>>> with Session(engine) as session:
...     for user, address in session.execute(stmt):
...         print(f"{user} {address}")
BEGIN  (implicit)
WITH  anon_1  AS
(SELECT  address.id  AS  id,  address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  address.email_address  NOT  LIKE  ?)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.email_address,  anon_1.user_id
FROM  user_account
JOIN  anon_1  ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.id
[...]  ('%@aol.com',)
User(id=1, name='spongebob', fullname='Spongebob Squarepants') Address(id=1, email_address='spongebob@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=2, email_address='sandy@sqlalchemy.org')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='sandy@squirrelpower.org')
ROLLBACK 
```

另请参阅

从子查询中选择实体 - 在 ORM 查询指南 中

## 标量和关联子查询

标量子查询是返回零行或一行以及一列的子查询。然后，在封闭的 SELECT 语句的 COLUMNS 或 WHERE 子句中使用该子查询，它与常规子查询不同，因为它不在 FROM 子句中使用。相关子查询是指涉及封闭 SELECT 语句中的表的标量子查询。

SQLAlchemy 使用`ScalarSelect`结构来表示标量子查询，该结构是`ColumnElement`表达式层次结构的一部分，与常规子查询不同，常规子查询由`Subquery`结构表示，后者属于`FromClause`层次结构。

标量子查询通常与聚合函数一起使用，但不一定要这样做，之前在带有 GROUP BY / HAVING 的聚合函数中介绍过。标量子查询通过显式地使用`Select.scalar_subquery()`方法来表示，如下所示。当单独字符串化时，默认的字符串形式呈现为一个普通的 SELECT 语句，该语句从两个表中选择：

```py
>>> subq = (
...     select(func.count(address_table.c.id))
...     .where(user_table.c.id == address_table.c.user_id)
...     .scalar_subquery()
... )
>>> print(subq)
(SELECT  count(address.id)  AS  count_1
FROM  address,  user_account
WHERE  user_account.id  =  address.user_id) 
```

上述 `subq` 对象现在属于`ColumnElement` SQL 表达式层次结构，因此它可以像任何其他列表达式一样使用：

```py
>>> print(subq == 5)
(SELECT  count(address.id)  AS  count_1
FROM  address,  user_account
WHERE  user_account.id  =  address.user_id)  =  :param_1 
```

尽管单独字符串化时，标量子查询会在其 FROM 子句中同时呈现`user_account`和`address`，但当将其嵌入到处理`user_account`表的封闭`select()`构造中时，`user_account`表会自动**关联**，这意味着它不会出现在子查询的 FROM 子句中：

```py
>>> stmt = select(user_table.c.name, subq.label("address_count"))
>>> print(stmt)
SELECT  user_account.name,  (SELECT  count(address.id)  AS  count_1
FROM  address
WHERE  user_account.id  =  address.user_id)  AS  address_count
FROM  user_account 
```

简单的相关子查询通常会执行所需的正确操作。然而，在相关性不明确的情况下，SQLAlchemy 会提醒我们需要更多的明确性：

```py
>>> stmt = (
...     select(
...         user_table.c.name,
...         address_table.c.email_address,
...         subq.label("address_count"),
...     )
...     .join_from(user_table, address_table)
...     .order_by(user_table.c.id, address_table.c.id)
... )
>>> print(stmt)
Traceback (most recent call last):
...
InvalidRequestError: Select statement '<... Select object at ...>' returned
no FROM clauses due to auto-correlation; specify correlate(<tables>) to
control correlation manually.
```

要指定我们要关联的`user_table`是哪一个，我们使用`ScalarSelect.correlate()`或`ScalarSelect.correlate_except()`方法来指定：

```py
>>> subq = (
...     select(func.count(address_table.c.id))
...     .where(user_table.c.id == address_table.c.user_id)
...     .scalar_subquery()
...     .correlate(user_table)
... )
```

然后语句就可以像对待其他列一样返回该列的数据：

```py
>>> with engine.connect() as conn:
...     result = conn.execute(
...         select(
...             user_table.c.name,
...             address_table.c.email_address,
...             subq.label("address_count"),
...         )
...         .join_from(user_table, address_table)
...         .order_by(user_table.c.id, address_table.c.id)
...     )
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name,  address.email_address,  (SELECT  count(address.id)  AS  count_1
FROM  address
WHERE  user_account.id  =  address.user_id)  AS  address_count
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id  ORDER  BY  user_account.id,  address.id
[...]  ()
[('spongebob', 'spongebob@sqlalchemy.org', 1), ('sandy', 'sandy@sqlalchemy.org', 2),
 ('sandy', 'sandy@squirrelpower.org', 2)]
ROLLBACK 
```

### LATERAL 关联

LATERAL 关联是 SQL 关联的一个特殊子类别，允许可选择的单元引用同一 FROM 子句内的另一个可选择单元。这是一个极其特殊的用例，虽然它是 SQL 标准的一部分，但目前只知道最近的 PostgreSQL 版本支持它。

通常，如果 SELECT 语句在其 FROM 子句中引用 `table1 JOIN (SELECT ...) AS subquery`，右侧的子查询可能无法引用左侧的“table1”表达式；关联只能引用完全包含此 SELECT 的另一个 SELECT 的表。LATERAL 关键字允许我们改变这种行为，并允许来自右侧 JOIN 的关联。

SQLAlchemy 支持使用 `Select.lateral()` 方法来实现此功能，该方法创建一个称为 `Lateral` 的对象。`Lateral` 与 `Subquery` 和 `Alias` 属于同一家族，但在将构造添加到包含 SELECT 的 FROM 子句时还包括关联行为。以下示例说明了使用 LATERAL 的 SQL 查询，选择了在前一节中讨论过的“用户账户/电子邮件地址计数”数据：

```py
>>> subq = (
...     select(
...         func.count(address_table.c.id).label("address_count"),
...         address_table.c.email_address,
...         address_table.c.user_id,
...     )
...     .where(user_table.c.id == address_table.c.user_id)
...     .lateral()
... )
>>> stmt = (
...     select(user_table.c.name, subq.c.address_count, subq.c.email_address)
...     .join_from(user_table, subq)
...     .order_by(user_table.c.id, subq.c.email_address)
... )
>>> print(stmt)
SELECT  user_account.name,  anon_1.address_count,  anon_1.email_address
FROM  user_account
JOIN  LATERAL  (SELECT  count(address.id)  AS  address_count,
address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  user_account.id  =  address.user_id)  AS  anon_1
ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.email_address 
```

在上面的例子中，JOIN 的右侧是一个与左侧连接的 `user_account` 表的子查询。

使用 `Select.lateral()` 时，`Select.correlate()` 和 `Select.correlate_except()` 方法的行为也适用于 `Lateral` 构造。

另请参阅

`Lateral`

`Select.lateral()`  ### LATERAL 关联

横向关联是 SQL 关联的一个特殊子类别，它允许一个可选择的单元在单个 FROM 子句内引用另一个可选择的单元。这是一个非常特殊的用例，虽然是 SQL 标准的一部分，但只有最近版本的 PostgreSQL 已知支持。

通常，如果一个 SELECT 语句在其 FROM 子句中引用了`table1 JOIN (SELECT ...) AS subquery`，则右侧的子查询可能不会引用左侧的“table1”表达式；关联可能仅引用完全包含此 SELECT 的另一个 SELECT 的表。LATERAL 关键字允许我们改变这种行为，允许从右侧 JOIN 进行关联。

SQLAlchemy 通过`Select.lateral()`方法支持此功能，该方法创建一个称为`横向关联`的对象。 `横向关联`与`子查询`和`别名`属于同一系列，但是当将构造添加到包围 SELECT 的 FROM 子句时，还包括关联行为。以下示例说明了使用 LATERAL 的 SQL 查询，选择了前一节中讨论的“用户帐户/电子邮件地址计数”数据：

```py
>>> subq = (
...     select(
...         func.count(address_table.c.id).label("address_count"),
...         address_table.c.email_address,
...         address_table.c.user_id,
...     )
...     .where(user_table.c.id == address_table.c.user_id)
...     .lateral()
... )
>>> stmt = (
...     select(user_table.c.name, subq.c.address_count, subq.c.email_address)
...     .join_from(user_table, subq)
...     .order_by(user_table.c.id, subq.c.email_address)
... )
>>> print(stmt)
SELECT  user_account.name,  anon_1.address_count,  anon_1.email_address
FROM  user_account
JOIN  LATERAL  (SELECT  count(address.id)  AS  address_count,
address.email_address  AS  email_address,  address.user_id  AS  user_id
FROM  address
WHERE  user_account.id  =  address.user_id)  AS  anon_1
ON  user_account.id  =  anon_1.user_id
ORDER  BY  user_account.id,  anon_1.email_address 
```

上述，JOIN 的右侧是一个子查询，它与 JOIN 左侧的`user_account`表相关联。

使用`Select.lateral()`时，`Select.correlate()`和`Select.correlate_except()`方法的行为也适用于`横向关联`构造。

另请参见

`横向关联`

`Select.lateral()`

## UNION、UNION ALL 和其他集合操作

在 SQL 中，SELECT 语句可以使用 UNION 或 UNION ALL SQL 操作合并在一起，该操作生成由一个或多个语句一起生成的所有行的集合。还可以执行其他集合操作，如 INTERSECT [ALL]和 EXCEPT [ALL]。

SQLAlchemy 的`Select`构造支持使用诸如`union()`、`intersect()`和`except_()`之类的函数进行这种性质的组合，以及“all”对应项`union_all()`、`intersect_all()`和`except_all()`。这些函数都接受任意数量的子可选择项，通常是`Select`构造，但也可以是现有的组合。

由这些函数生成的构造是`CompoundSelect`，其使用方式与`Select`构造相同，只是它的方法较少。例如，由`union_all()`产生的`CompoundSelect`可以直接通过`Connection.execute()`调用：

```py
>>> from sqlalchemy import union_all
>>> stmt1 = select(user_table).where(user_table.c.name == "sandy")
>>> stmt2 = select(user_table).where(user_table.c.name == "spongebob")
>>> u = union_all(stmt1, stmt2)
>>> with engine.connect() as conn:
...     result = conn.execute(u)
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[generated  in  ...]  ('sandy',  'spongebob')
[(2, 'sandy', 'Sandy Cheeks'), (1, 'spongebob', 'Spongebob Squarepants')]
ROLLBACK 
```

要将`CompoundSelect`用作子查询，就像`Select`一样，它提供了一个`SelectBase.subquery()`方法，它将生成一个带有`FromClause.c`集合的`Subquery`对象，可以在封闭的`select()`中引用：

```py
>>> u_subq = u.subquery()
>>> stmt = (
...     select(u_subq.c.name, address_table.c.email_address)
...     .join_from(address_table, u_subq)
...     .order_by(u_subq.c.name, address_table.c.email_address)
... )
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  anon_1.name,  address.email_address
FROM  address  JOIN
  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
  FROM  user_account
  WHERE  user_account.name  =  ?
UNION  ALL
  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
  FROM  user_account
  WHERE  user_account.name  =  ?)
AS  anon_1  ON  anon_1.id  =  address.user_id
ORDER  BY  anon_1.name,  address.email_address
[generated  in  ...]  ('sandy',  'spongebob')
[('sandy', 'sandy@sqlalchemy.org'), ('sandy', 'sandy@squirrelpower.org'), ('spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

### 从联合中选择 ORM 实体

前面的示例说明了如何构造一个 UNION，给定两个`Table`对象，然后返回数据库行。如果我们想要使用 UNION 或其他集合操作来选择行，然后将其作为 ORM 对象接收，有两种方法可以使用。在这两种情况下，我们首先构造一个表示我们想要执行的 SELECT / UNION / 等语句的`select()`或`CompoundSelect`对象；这个语句应该针对目标 ORM 实体或它们的底层映射的`Table`对象组成：

```py
>>> stmt1 = select(User).where(User.name == "sandy")
>>> stmt2 = select(User).where(User.name == "spongebob")
>>> u = union_all(stmt1, stmt2)
```

对于一个简单的 SELECT，带有 UNION，它尚未嵌套在子查询内部，通常可以通过使用`Select.from_statement()`方法在 ORM 对象获取上下文中使用。通过这种方法，UNION 语句代表整个查询；在使用`Select.from_statement()`之后，不能添加额外的条件：

```py
>>> orm_stmt = select(User).from_statement(u)
>>> with Session(engine) as session:
...     for obj in session.execute(orm_stmt).scalars():
...         print(obj)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?  UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[generated  in  ...]  ('sandy',  'spongebob')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
ROLLBACK 
```

要以更灵活的方式将 UNION 或其他与实体相关的构造用作实体相关组件，可以使用`CompoundSelect`构造，使用`CompoundSelect.subquery()`将其组织成子查询，然后使用`aliased()`函数将其链接到 ORM 对象。这与在 ORM 实体子查询/CTEs 中介绍的方式相同，首先创建我们所需实体的临时“映射”，然后从该新实体选择，就像它是任何其他映射类一样。在下面的示例中，我们能够添加额外的条件，例如在 UNION 之外的 ORDER BY，因为我们可以过滤或按子查询导出的列排序：

```py
>>> user_alias = aliased(User, u.subquery())
>>> orm_stmt = select(user_alias).order_by(user_alias.id)
>>> with Session(engine) as session:
...     for obj in session.execute(orm_stmt).scalars():
...         print(obj)
BEGIN  (implicit)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.name  =  ?  UNION  ALL  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.name  =  ?)  AS  anon_1  ORDER  BY  anon_1.id
[generated  in  ...]  ('sandy',  'spongebob')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
ROLLBACK 
```

另请参阅

从 UNIONs 和其他集合操作中选择实体 - 在 ORM 查询指南 中的 ORM 实体从联合中选择

前面的示例说明了如何在给定两个`Table`对象的情况下构造一个 UNION，然后返回数据库行。如果我们想要使用 UNION 或其他集合操作来选择行，然后将其作为 ORM 对象接收，有两种方法可以使用。在这两种情况下，我们首先构造一个`select()`或`CompoundSelect`对象，该对象表示我们要执行的 SELECT / UNION /等语句;此语句应针对目标 ORM 实体或其底层映射的`Table`对象组成:

```py
>>> stmt1 = select(User).where(User.name == "sandy")
>>> stmt2 = select(User).where(User.name == "spongebob")
>>> u = union_all(stmt1, stmt2)
```

对于不在子查询内部的简单 SELECT 与 UNION，通常可以使用`Select.from_statement()`方法在 ORM 对象获取上下文中使用。通过这种方法，UNION 语句表示整个查询;在使用`Select.from_statement()`之后，不能添加额外的条件:

```py
>>> orm_stmt = select(User).from_statement(u)
>>> with Session(engine) as session:
...     for obj in session.execute(orm_stmt).scalars():
...         print(obj)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?  UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[generated  in  ...]  ('sandy',  'spongebob')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
ROLLBACK 
```

要以更灵活的方式将 UNION 或其他集相关构造用作实体相关组件，可以使用`CompoundSelect`构造将其组织到子查询中，然后使用`CompoundSelect.subquery()`将其链接到 ORM 对象，然后使用`aliased()`函数。这与 ORM 实体子查询/ CTEs 中介绍的方式相同，首先创建我们所需实体到子查询的临时“映射”，然后从该新实体中选择，就像它是任何其他映射类一样。在下面的示例中，我们能够添加额外的条件，例如在 UNION 本身之外进行 ORDER BY，因为我们可以通过子查询导出的列进行过滤或排序:

```py
>>> user_alias = aliased(User, u.subquery())
>>> orm_stmt = select(user_alias).order_by(user_alias.id)
>>> with Session(engine) as session:
...     for obj in session.execute(orm_stmt).scalars():
...         print(obj)
BEGIN  (implicit)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.name  =  ?  UNION  ALL  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.name  =  ?)  AS  anon_1  ORDER  BY  anon_1.id
[generated  in  ...]  ('sandy',  'spongebob')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
ROLLBACK 
```

另请参阅

从 UNION 和其他集合操作中选择实体 - 在 ORM 查询指南中

## EXISTS 子查询

SQL EXISTS 关键字是一个与标量子查询一起使用的运算符，根据 SELECT 语句是否返回行来返回布尔值 true 或 false。SQLAlchemy 包含一个名为`Exists`的`ScalarSelect`对象的变体，它将生成一个 EXISTS 子查询，并且最方便的方式是使用`SelectBase.exists()`方法生成。下面我们生成一个 EXISTS，以便我们可以返回`user_account`行，其中`address`有多于一个相关行。

```py
>>> subq = (
...     select(func.count(address_table.c.id))
...     .where(user_table.c.id == address_table.c.user_id)
...     .group_by(address_table.c.user_id)
...     .having(func.count(address_table.c.id) > 1)
... ).exists()
>>> with engine.connect() as conn:
...     result = conn.execute(select(user_table.c.name).where(subq))
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name
FROM  user_account
WHERE  EXISTS  (SELECT  count(address.id)  AS  count_1
FROM  address
WHERE  user_account.id  =  address.user_id  GROUP  BY  address.user_id
HAVING  count(address.id)  >  ?)
[...]  (1,)
[('sandy',)]
ROLLBACK 
```

EXISTS 结构更常用于否定，例如 NOT EXISTS，因为它提供了一种 SQL 效率高的方式来定位一个相关表没有行的行。下面我们选择没有电子邮件地址的用户名称；注意在第二个 WHERE 子句中使用的二进制否定运算符（`~`）：

```py
>>> subq = (
...     select(address_table.c.id).where(user_table.c.id == address_table.c.user_id)
... ).exists()
>>> with engine.connect() as conn:
...     result = conn.execute(select(user_table.c.name).where(~subq))
...     print(result.all())
BEGIN  (implicit)
SELECT  user_account.name
FROM  user_account
WHERE  NOT  (EXISTS  (SELECT  address.id
FROM  address
WHERE  user_account.id  =  address.user_id))
[...]  ()
[('patrick',)]
ROLLBACK 
```

## 使用 SQL 函数

此部分较早前在带有 GROUP BY / HAVING 的聚合函数中首次介绍，`func`对象用作创建新的`Function`对象的工厂，在像`select()`这样的构造中使用时，会产生一个 SQL 函数显示，通常包含一个名称、一些括号（尽管不总是），以及可能的一些参数。典型 SQL 函数的示例包括：

+   `count()`函数，一个聚合函数，用于计算返回的行数：

    ```py
    >>> print(select(func.count()).select_from(user_table))
    SELECT  count(*)  AS  count_1
    FROM  user_account 
    ```

+   `lower()`函数，一个字符串函数，用于将字符串转换为小写：

    ```py
    >>> print(select(func.lower("A String With Much UPPERCASE")))
    SELECT  lower(:lower_2)  AS  lower_1 
    ```

+   `now()`函数，提供当前日期和时间；由于这是一个常见的函数，SQLAlchemy 知道如何为每个后端呈现不同的结果，在 SQLite 中使用 CURRENT_TIMESTAMP 函数：

    ```py
    >>> stmt = select(func.now())
    >>> with engine.connect() as conn:
    ...     result = conn.execute(stmt)
    ...     print(result.all())
    BEGIN  (implicit)
    SELECT  CURRENT_TIMESTAMP  AS  now_1
    [...]  ()
    [(datetime.datetime(...),)]
    ROLLBACK 
    ```

由于大多数数据库后端包含数十甚至数百个不同的 SQL 函数，`func`尝试在接受的内容上尽可能宽松。从此命名空间中访问的任何名称都会自动被视为一个 SQL 函数，以一种通用的方式呈现：

```py
>>> print(select(func.some_crazy_function(user_table.c.name, 17)))
SELECT  some_crazy_function(user_account.name,  :some_crazy_function_2)  AS  some_crazy_function_1
FROM  user_account 
```

同时，一组相对较小但极其常见的 SQL 函数，比如 `count`、`now`、`max`、`concat` 等，包含了它们自己的预打包版本，这些版本提供了正确的类型信息以及在某些情况下特定于后端的 SQL 生成。下面的示例对比了 PostgreSQL 方言和 Oracle 方言中 `now` 函数的 SQL 生成：

```py
>>> from sqlalchemy.dialects import postgresql
>>> print(select(func.now()).compile(dialect=postgresql.dialect()))
SELECT  now()  AS  now_1
>>> from sqlalchemy.dialects import oracle
>>> print(select(func.now()).compile(dialect=oracle.dialect()))
SELECT  CURRENT_TIMESTAMP  AS  now_1  FROM  DUAL 
```

### 函数具有返回类型

由于函数是列表达式，它们还有描述生成的 SQL 表达式的数据类型的 SQL 数据类型。我们在这里将这些类型称为“SQL 返回类型”，指的是在数据库端 SQL 表达式的上下文中函数返回的 SQL 值类型，而不是 Python 函数的“返回类型”。

任何 SQL 函数的 SQL 返回类型都可以访问，通常用于调试目的，方法是引用 `Function.type` 属性：

```py
>>> func.now().type
DateTime()
```

这些 SQL 返回类型在使用函数表达式时非常重要，特别是在更大表达式的上下文中；也就是说，当表达式的数据类型是类似 `Integer` 或 `Numeric` 这样的类型时，数学运算符将更有效，为了让 JSON 访问器正常工作，需要使用类似 `JSON` 这样的类型。某些类别的函数返回整行而不是列值，在需要引用特定列的情况下；这些函数被称为表值函数。

当执行语句并获取行时，函数的 SQL 返回类型也可能很重要，对于 SQLAlchemy 需要应用结果集处理的情况来说尤其如此。SQLite 上的日期相关函数是一个典型例子，其中 SQLAlchemy 的 `DateTime` 和相关数据类型在接收到结果行时起到将字符串值转换为 Python `datetime()` 对象的作用。

要将特定类型应用于我们正在创建的函数，我们可以使用 `Function.type_` 参数进行传递；类型参数可以是 `TypeEngine` 类或实例。在下面的示例中，我们将 `JSON` 类传递给生成 PostgreSQL `json_object()` 函数，注意 SQL 返回类型将是 JSON 类型：

```py
>>> from sqlalchemy import JSON
>>> function_expr = func.json_object('{a, 1, b, "def", c, 3.5}', type_=JSON)
```

通过使用具有 `JSON` 数据类型的 JSON 函数，SQL 表达式对象具有与 JSON 相关的功能，例如访问元素：

```py
>>> stmt = select(function_expr["def"])
>>> print(stmt)
SELECT  json_object(:json_object_1)[:json_object_2]  AS  anon_1 
```

### 内置函数具有预配置的返回类型

对于像`count`、`max`和`min`这样的常见聚合函数，以及一些非常少数的日期函数，比如`now`和字符串函数，SQL 返回类型会根据使用情况进行适当设置。`max`函数和类似的聚合过滤函数将根据给定的参数设置 SQL 返回类型：

```py
>>> m1 = func.max(Column("some_int", Integer))
>>> m1.type
Integer()

>>> m2 = func.max(Column("some_str", String))
>>> m2.type
String()
```

日期和时间函数通常对应于由 `DateTime`、`Date` 或 `Time` 描述的 SQL 表达式：

```py
>>> func.now().type
DateTime()
>>> func.current_date().type
Date()
```

已知的字符串函数，如 `concat`，将知道 SQL 表达式的类型将是 `String`：

```py
>>> func.concat("x", "y").type
String()
```

但是，对于绝大多数 SQL 函数，SQLAlchemy 并没有在其极少量的已知函数列表中明确地提供它们。例如，虽然通常使用 SQL 函数 `func.lower()` 和 `func.upper()` 来转换字符串的大小写没有问题，但 SQLAlchemy 实际上并不知道这些函数，因此它们具有“null”SQL 返回类型：

```py
>>> func.upper("lowercase").type
NullType()
```

对于像`upper`和`lower`这样的简单函数，问题通常不是很重要，因为字符串值可能从数据库接收而不需要在 SQLAlchemy 端进行任何特殊类型处理，而且 SQLAlchemy 的类型强制规则通常也可以正确猜测意图；例如，Python 的`+`运算符将根据表达式两侧的内容正确解释为字符串连接运算符：

```py
>>> print(select(func.upper("lowercase") + " suffix"))
SELECT  upper(:upper_1)  ||  :upper_2  AS  anon_1 
```

总的来说，`Function.type_`参数可能是必要的情况是：

1.  函数不是 SQLAlchemy 内置函数；这可以通过创建函数并观察`Function.type`属性来证明，即：

    ```py
    >>> func.count().type
    Integer()
    ```

    vs.：

    ```py
    >>> func.json_object('{"a", "b"}').type
    NullType()
    ```

1.  需要函数感知表达式支持；这通常指的是与数据类型相关的特殊运算符，如`JSON`或`ARRAY`。

1.  需要结果值处理，可能包括诸如`DateTime`、`Boolean`、`Enum`等类型，或者再次特殊数据类型，如`JSON`、`ARRAY`。

### 高级 SQL 函数技术

以下各小节说明了 SQL 函数可以做的更多事情。虽然这些技术比基本 SQL 函数使用更不常见和更高级，但它们仍然非常流行，主要是由于 PostgreSQL 强调更复杂的函数形式，包括对 JSON 数据流行的表和列值形式。

#### 使用窗口函数

窗口函数是 SQL 聚合函数的一种特殊用法，它在处理单个结果行时计算返回组中的行上的聚合值。而像`MAX()`这样的函数会给出一组行中某一列的最高值，使用相同函数作为“窗口函数”将为每一行给出最高值，*截至该行*。

在 SQL 中，窗口函数允许指定应用函数的行，一个“分区”值，考虑窗口在不同子行集上的情况，以及一个“order by”表达式，重要的是指示应用到聚合函数的行的顺序。

在 SQLAlchemy 中，`func`命名空间生成的所有 SQL 函数都包括一个`FunctionElement.over()`方法，该方法授予窗口函数或“OVER”语法；生成的构造是`Over`构造函数。

与窗口函数一起使用的常见函数是`row_number()`函数，它简单地计算行数。我们可以将这个行数按用户名分区，以为每个用户的电子邮件地址编号：

```py
>>> stmt = (
...     select(
...         func.row_number().over(partition_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  row_number()  OVER  (PARTITION  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(1, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (1, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

上述中，`FunctionElement.over.partition_by`参数用于在 OVER 子句内呈现 PARTITION BY 子句。我们还可以使用`FunctionElement.over.order_by`来使用 ORDER BY 子句：

```py
>>> stmt = (
...     select(
...         func.count().over(order_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(*)  OVER  (ORDER  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(2, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (3, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

窗口函数的进一步选项包括使用范围；更多示例请参见`over()`。

提示

需要注意的是，`FunctionElement.over()`方法仅适用于实际上是聚合函数的 SQL 函数；虽然`Over`构造函数将愉快地为任何给定的 SQL 函数呈现自身，但如果函数本身不是 SQL 聚合函数，则数据库将拒绝表达式。#### 特殊修饰符 WITHIN GROUP, FILTER

“WITHIN GROUP” SQL 语法与“ordered set”或“假设性集合”聚合函数一起使用。常见的“ordered set”函数包括`percentile_cont()`和`rank()`。SQLAlchemy 包括内置实现`rank`、`dense_rank`、`mode`、`percentile_cont`和`percentile_disc`等函数，并包括一个`FunctionElement.within_group()`方法：

```py
>>> print(
...     func.unnest(
...         func.percentile_disc([0.25, 0.5, 0.75, 1]).within_group(user_table.c.name)
...     )
... )
unnest(percentile_disc(:percentile_disc_1)  WITHIN  GROUP  (ORDER  BY  user_account.name)) 
```

“FILTER” 受一些后端支持，可以通过使用 `FunctionElement.filter()` 方法将聚合函数的范围限制为与返回的总行范围相比的特定子集：

```py
>>> stmt = (
...     select(
...         func.count(address_table.c.email_address).filter(user_table.c.name == "sandy"),
...         func.count(address_table.c.email_address).filter(
...             user_table.c.name == "spongebob"
...         ),
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_1,
count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_2
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ('sandy',  'spongebob')
[(2, 1)]
ROLLBACK 
```  #### 表值函数

表值 SQL 函数支持包含命名子元素的标量表示。通常用于 JSON 和 ARRAY 导向函数以及诸如 `generate_series()` 等函数，表值函数在 FROM 子句中指定，然后被引用为表，有时甚至被引用为列。这种形式的函数在 PostgreSQL 数据库中非常突出，然而某些形式的表值函数也受到 SQLite、Oracle 和 SQL Server 的支持。

另请参阅

表值、表值和列值函数、行和元组对象 - 在 PostgreSQL 文档中。

虽然许多数据库支持表值函数和其他特殊形式，但 PostgreSQL 往往是对这些功能需求最多的地方。有关 PostgreSQL 语法的其他示例以及其他功能，请参见本节。

SQLAlchemy 提供了 `FunctionElement.table_valued()` 方法作为基本的“表值函数”构造，它将一个 `func` 对象转换为一个包含一系列命名列的 FROM 子句，这些命名是按位置传递的字符串名称。这将返回一个 `TableValuedAlias` 对象，它是一个启用函数的 `Alias` 构造，可以像其他 FROM 子句一样使用，如 Using Aliases 中介绍的那样。下面我们演示了 `json_each()` 函数，虽然它在 PostgreSQL 上很常见，但也被现代版本的 SQLite 支持：

```py
>>> onetwothree = func.json_each('["one", "two", "three"]').table_valued("value")
>>> stmt = select(onetwothree).where(onetwothree.c.value.in_(["two", "three"]))
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     result.all()
BEGIN  (implicit)
SELECT  anon_1.value
FROM  json_each(?)  AS  anon_1
WHERE  anon_1.value  IN  (?,  ?)
[...]  ('["one", "two", "three"]',  'two',  'three')
[('two',), ('three',)]
ROLLBACK 
```

在上面的示例中，我们使用了 SQLite 和 PostgreSQL 支持的 `json_each()` JSON 函数来生成一个包含一个称为 `value` 的单列的表值表达式，然后选择了其中的两行。

另请参阅

表值函数 - 在 PostgreSQL 文档中 - 本节将详细说明其他语法，例如特殊列推导和“WITH ORDINALITY”，这些语法已知适用于 PostgreSQL。 #### 列值函数 - 表值函数作为标量列

PostgreSQL 和 Oracle 支持的一种特殊语法是在 FROM 子句中引用函数，然后在 SELECT 语句或其他列表达式上下文中将其自身作为单个列传递。PostgreSQL 在诸如`json_array_elements()`、`json_object_keys()`、`json_each_text()`、`json_each()`等函数中广泛使用此语法。

SQLAlchemy 将此称为“列值”函数，并通过将`FunctionElement.column_valued()`修饰符应用于`Function`构造来使用：

```py
>>> from sqlalchemy import select, func
>>> stmt = select(func.json_array_elements('["one", "two"]').column_valued("x"))
>>> print(stmt)
SELECT  x
FROM  json_array_elements(:json_array_elements_1)  AS  x 
```

“列值”形式也受 Oracle 方言支持，可用于自定义 SQL 函数：

```py
>>> from sqlalchemy.dialects import oracle
>>> stmt = select(func.scalar_strings(5).column_valued("s"))
>>> print(stmt.compile(dialect=oracle.dialect()))
SELECT  s.COLUMN_VALUE
FROM  TABLE  (scalar_strings(:scalar_strings_1))  s 
```

另请参阅

列值函数 - 在 PostgreSQL 文档中。

### 函数具有返回类型

由于函数是列表达式，它们还具有描述生成的 SQL 表达式的数据类型的 SQL 数据类型。我们在这里将这些类型称为“SQL 返回类型”，指的是在数据库端 SQL 表达式的上下文中函数返回的 SQL 值的类型，而不是 Python 函数的“返回类型”。

任何 SQL 函数的 SQL 返回类型可以通过引用`Function.type`属性来访问，通常用于调试目的：

```py
>>> func.now().type
DateTime()
```

当在更大表达式的上下文中使用函数表达式时，这些 SQL 返回类型很重要；也就是说，数学运算符在表达式的数据类型为`Integer`或`Numeric`时会更好地工作，为了使 JSON 访问器正常工作，需要使用诸如`JSON`之类的类型。某些类别的函数返回整行而不是列值，需要引用特定列；这些函数被称为表值函数。

当执行语句并获取行时，函数的 SQL 返回类型也可能很重要，对于那些 SQLAlchemy 需要应用结果集处理的情况。一个典型的例子是 SQLite 上的日期相关函数，在那里 SQLAlchemy 的`DateTime`和相关数据类型扮演着将字符串值转换为 Python `datetime()`对象的角色，当接收到结果行时。

要将特定类型应用于我们正在创建的函数，我们使用 `Function.type_` 参数传递它；类型参数可以是 `TypeEngine` 类，也可以是一个实例。在下面的示例中，我们传递 `JSON` 类来生成 PostgreSQL `json_object()` 函数，注意 SQL 返回类型将是 JSON 类型：

```py
>>> from sqlalchemy import JSON
>>> function_expr = func.json_object('{a, 1, b, "def", c, 3.5}', type_=JSON)
```

通过使用 `JSON` 数据类型创建我们的 JSON 函数，SQL 表达式对象具有了 JSON 相关的功能，比如访问元素：

```py
>>> stmt = select(function_expr["def"])
>>> print(stmt)
SELECT  json_object(:json_object_1)[:json_object_2]  AS  anon_1 
```

### 内置函数具有预配置的返回类型

对于常见的聚合函数，比如 `count`、`max`、`min` 以及极少数日期函数，比如 `now` 和字符串函数，比如 `concat`，SQL 返回类型将被适当地设置，有时根据使用情况。`max` 函数和类似的聚合过滤函数将根据给定的参数设置 SQL 返回类型：

```py
>>> m1 = func.max(Column("some_int", Integer))
>>> m1.type
Integer()

>>> m2 = func.max(Column("some_str", String))
>>> m2.type
String()
```

日期和时间函数通常对应于由 `DateTime`、`Date` 或 `Time` 描述的 SQL 表达式：

```py
>>> func.now().type
DateTime()
>>> func.current_date().type
Date()
```

一个已知的字符串函数，比如 `concat`，会知道 SQL 表达式的类型将是 `String`：

```py
>>> func.concat("x", "y").type
String()
```

然而，对于绝大多数 SQL 函数，SQLAlchemy 并没有在其非常小的已知函数列表中显式地提供它们。例如，虽然通常可以使用 SQL 函数 `func.lower()` 和 `func.upper()` 来转换字符串的大小写，但 SQLAlchemy 实际上并不知道这些函数，因此它们具有“null”SQL 返回类型：

```py
>>> func.upper("lowercase").type
NullType()
```

对于诸如 `upper` 和 `lower` 这样的简单函数，通常问题并不重要，因为字符串值可以直接从数据库接收，SQLAlchemy 方面不需要进行任何特殊类型处理，而 SQLAlchemy 的类型强制转换规则通常可以正确猜测意图；例如，Python 的 `+` 操作符将根据表达式的两侧正确解释为字符串连接操作符：

```py
>>> print(select(func.upper("lowercase") + " suffix"))
SELECT  upper(:upper_1)  ||  :upper_2  AS  anon_1 
```

总的来说，`Function.type_` 参数可能是必要的场景是：

1.  如果函数不是 SQLAlchemy 内置函数，则需要创建该函数并观察 `Function.type` 属性，即：

    ```py
    >>> func.count().type
    Integer()
    ```

    vs.:

    ```py
    >>> func.json_object('{"a", "b"}').type
    NullType()
    ```

1.  需要函数感知表达式支持；这通常指的是与数据类型相关的特殊运算符，如 `JSON` 或者 `ARRAY`。

1.  需要进行结果值处理，可能涉及到诸如 `DateTime`、`Boolean`、`Enum` 或者特殊的数据类型如 `JSON`、`ARRAY`。

### 高级 SQL 函数技巧

以下各小节说明了可以使用 SQL 函数执行的更多操作。虽然这些技术比基本的 SQL 函数使用更少见、更高级，但它们仍然非常受欢迎，主要是由于 PostgreSQL 对更复杂的函数形式的强调，包括对 JSON 数据非常流行的表值和列值形式。

#### 使用窗口函数

窗口函数是 SQL 聚合函数的一种特殊用法，它在处理个别结果行时计算返回组中的行的聚合值。而像 `MAX()` 这样的函数会给出一组行中的列的最大值，使用同样的函数作为“窗口函数”将为每一行给出最高的值，*截至到那一行*。

在 SQL 中，窗口函数允许指定应应用函数的行，一个“分区”值，它考虑在不同行子集上的窗口，以及一个“order by”表达式，它重要地指示应该将行应用到聚合函数的顺序。

在 SQLAlchemy 中，由 `func` 命名空间生成的所有 SQL 函数都包括一个方法 `FunctionElement.over()`，它授予了窗口函数或“OVER”语法；产生的构造是 `Over` 构造。

与窗口函数一起使用的常见函数是 `row_number()` 函数，它简单地计算行数。我们可以根据用户名对此行计数进行分区，以对各个用户的电子邮件地址进行编号：

```py
>>> stmt = (
...     select(
...         func.row_number().over(partition_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  row_number()  OVER  (PARTITION  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(1, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (1, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

在上述代码中，`FunctionElement.over.partition_by` 参数被使用，以便在 OVER 子句中呈现 `PARTITION BY` 子句。我们还可以使用 `FunctionElement.over.order_by` 使用 `ORDER BY` 子句：

```py
>>> stmt = (
...     select(
...         func.count().over(order_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(*)  OVER  (ORDER  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(2, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (3, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

更多窗口函数的选项包括使用范围；有关更多示例，请参阅`over()`。

提示

需要注意的是，`FunctionElement.over()` 方法仅适用于实际上是聚合函数的 SQL 函数；虽然 `Over` 构造将愉快地为任何给定的 SQL 函数呈现自身，但如果函数本身不是 SQL 聚合函数，数据库将拒绝表达式。  #### 特殊修饰符 WITHIN GROUP, FILTER

“WITHIN GROUP” SQL 语法与“有序集”或“假设集”聚合函数结合使用。常见的“有序集”函数包括 `percentile_cont()` 和 `rank()`。SQLAlchemy 包括内置实现 `rank`、`dense_rank`、`mode`、`percentile_cont` 和 `percentile_disc`，其中包括一个 `FunctionElement.within_group()` 方法：

```py
>>> print(
...     func.unnest(
...         func.percentile_disc([0.25, 0.5, 0.75, 1]).within_group(user_table.c.name)
...     )
... )
unnest(percentile_disc(:percentile_disc_1)  WITHIN  GROUP  (ORDER  BY  user_account.name)) 
```

“FILTER” 受到某些后端的支持，用于将聚合函数的范围限制为与返回的总行数的特定子集相比较，可使用 `FunctionElement.filter()` 方法获得：

```py
>>> stmt = (
...     select(
...         func.count(address_table.c.email_address).filter(user_table.c.name == "sandy"),
...         func.count(address_table.c.email_address).filter(
...             user_table.c.name == "spongebob"
...         ),
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_1,
count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_2
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ('sandy',  'spongebob')
[(2, 1)]
ROLLBACK 
```#### 表值函数

表值 SQL 函数支持包含命名子元素的标量表示。通常用于 JSON 和数组导向的函数以及诸如 `generate_series()` 等函数，表值函数在 FROM 子句中指定，然后被引用为表，有时甚至被引用为列。这种形式的函数在 PostgreSQL 数据库中非常突出，但某些形式的表值函数也受到 SQLite、Oracle 和 SQL Server 的支持。

另请参阅

表值、表和列值函数、行和元组对象 - 在 PostgreSQL 文档中。

虽然许多数据库支持表值和其他特殊形式，但 PostgreSQL 往往是这些特性需求最大的地方。有关 PostgreSQL 语法的其他示例以及其他功能，请参阅本节。

SQLAlchemy 提供了 `FunctionElement.table_valued()` 方法作为基本的“表值函数”构造，它将 `func` 对象转换为包含一系列命名列的 FROM 子句，基于传递的字符串名称位置。这将返回一个 `TableValuedAlias` 对象，这是一个启用了函数的 `Alias` 构造，可以像介绍中的使用别名那样用作任何其他 FROM 子句。下面我们举例说明 `json_each()` 函数，尽管在 PostgreSQL 上很常见，但现代版本的 SQLite 也支持它：

```py
>>> onetwothree = func.json_each('["one", "two", "three"]').table_valued("value")
>>> stmt = select(onetwothree).where(onetwothree.c.value.in_(["two", "three"]))
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     result.all()
BEGIN  (implicit)
SELECT  anon_1.value
FROM  json_each(?)  AS  anon_1
WHERE  anon_1.value  IN  (?,  ?)
[...]  ('["one", "two", "three"]',  'two',  'three')
[('two',), ('three',)]
ROLLBACK 
```

上面，我们使用了 SQLite 和 PostgreSQL 支持的 `json_each()` JSON 函数来生成一个具有单列的表值表达式，该列被称为 `value`，然后选择了它的三行中的两行。

另请参阅

表值函数 - 在 PostgreSQL 文档中 - 本节将详细介绍额外的语法，例如特殊列派生和“WITH ORDINALITY”，这些都是已知与 PostgreSQL 兼容的。#### 列值函数 - 表值函数作为标量列

PostgreSQL 和 Oracle 支持的一种特殊语法是在 FROM 子句中引用函数，然后将其自身作为单个列提供给 SELECT 语句或其他列表达式上下文中。 PostgreSQL 非常善于使用此语法，用于诸如`json_array_elements()`、`json_object_keys()`、`json_each_text()`、`json_each()`等函数。

SQLAlchemy 将此称为“列值”函数，并通过将`FunctionElement.column_valued()`修改器应用于`Function`构造来使用：

```py
>>> from sqlalchemy import select, func
>>> stmt = select(func.json_array_elements('["one", "two"]').column_valued("x"))
>>> print(stmt)
SELECT  x
FROM  json_array_elements(:json_array_elements_1)  AS  x 
```

“列值”形式也受 Oracle 方言支持，可用于自定义 SQL 函数：

```py
>>> from sqlalchemy.dialects import oracle
>>> stmt = select(func.scalar_strings(5).column_valued("s"))
>>> print(stmt.compile(dialect=oracle.dialect()))
SELECT  s.COLUMN_VALUE
FROM  TABLE  (scalar_strings(:scalar_strings_1))  s 
```

另请参阅

列值函数 - 在 PostgreSQL 文档中。#### 使用窗口函数

窗口函数是 SQL 聚合函数的特殊用法，它计算在处理单个结果行时返回的行中的聚合值。而像`MAX()`这样的函数将为一组行中的一列给出最高值，将相同函数用作“窗口函数”将为每一行给出最高值，*截至该行*。

在 SQL 中，窗口函数允许指定应用函数的行，一个“分区”值，该值考虑了对不同行子集的窗口，以及一个“order by”表达式，这个表达式重要地指示应用到聚合函数的行的顺序。

在 SQLAlchemy 中，由`func`命名空间生成的所有 SQL 函数都包含一个方法`FunctionElement.over()`，该方法授予窗口函数或“OVER”语法；产生的构造是`Over`构造。

与窗口函数一起使用的常见函数是`row_number()`函数，它只是计算行数。我们可以将这个行计数按用户名分区，为个别用户的电子邮件地址编号：

```py
>>> stmt = (
...     select(
...         func.row_number().over(partition_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  row_number()  OVER  (PARTITION  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(1, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (1, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

上面，使用`FunctionElement.over.partition_by`参数，以便在 OVER 子句中呈现`PARTITION BY`子句。我们还可以使用`FunctionElement.over.order_by`使用`ORDER BY`子句：

```py
>>> stmt = (
...     select(
...         func.count().over(order_by=user_table.c.name),
...         user_table.c.name,
...         address_table.c.email_address,
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(*)  OVER  (ORDER  BY  user_account.name)  AS  anon_1,
user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
[(2, 'sandy', 'sandy@sqlalchemy.org'), (2, 'sandy', 'sandy@squirrelpower.org'), (3, 'spongebob', 'spongebob@sqlalchemy.org')]
ROLLBACK 
```

窗口函数的进一步选项包括使用范围；更多示例请参见 `over()`。

提示

注意，`FunctionElement.over()` 方法仅适用于那些实际上是聚合函数的 SQL 函数；而 `Over` 构造会为任何给定的 SQL 函数自动渲染自身，但如果函数本身不是 SQL 聚合函数，则数据库将拒绝该表达式。

#### 特殊修饰符 WITHIN GROUP, FILTER

“WITHIN GROUP” SQL 语法与“有序集合”或“假设集合”聚合函数一起使用。常见的“有序集合”函数包括 `percentile_cont()` 和 `rank()`。SQLAlchemy 包括内置的实现 `rank`、`dense_rank`、`mode`、`percentile_cont` 和 `percentile_disc`，其中包括一个 `FunctionElement.within_group()` 方法：

```py
>>> print(
...     func.unnest(
...         func.percentile_disc([0.25, 0.5, 0.75, 1]).within_group(user_table.c.name)
...     )
... )
unnest(percentile_disc(:percentile_disc_1)  WITHIN  GROUP  (ORDER  BY  user_account.name)) 
```

“FILTER” 被一些后端支持，以限制聚合函数的范围仅适用于与返回的行的总范围相比的特定子集，可使用 `FunctionElement.filter()` 方法：

```py
>>> stmt = (
...     select(
...         func.count(address_table.c.email_address).filter(user_table.c.name == "sandy"),
...         func.count(address_table.c.email_address).filter(
...             user_table.c.name == "spongebob"
...         ),
...     )
...     .select_from(user_table)
...     .join(address_table)
... )
>>> with engine.connect() as conn:  
...     result = conn.execute(stmt)
...     print(result.all())
BEGIN  (implicit)
SELECT  count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_1,
count(address.email_address)  FILTER  (WHERE  user_account.name  =  ?)  AS  anon_2
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ('sandy',  'spongebob')
[(2, 1)]
ROLLBACK 
```

#### 表值函数

表值 SQL 函数支持包含命名子元素的标量表示。表值函数通常用于 JSON 和 ARRAY 导向的函数以及像 `generate_series()` 这样的函数，该表值函数在 FROM 子句中指定，然后作为表或有时甚至作为列引用。这种形式的函数在 PostgreSQL 数据库中很突出，然而一些形式的表值函数也受 SQLite、Oracle 和 SQL Server 支持。

另请参见

Table values, Table and Column valued functions, Row and Tuple objects - 在 PostgreSQL 文档中。

虽然许多数据库支持表值函数和其他特殊形式，但 PostgreSQL 往往是最需要这些功能的地方。请参阅此部分，了解 PostgreSQL 语法的其他示例以及其他功能。

SQLAlchemy 提供了 `FunctionElement.table_valued()` 方法作为基本的“表值函数”构造，它将一个 `func` 对象转换为一个包含一系列命名列的 FROM 子句，这些列是基于按位置传递的字符串名称的。这将返回一个 `TableValuedAlias` 对象，这是一个启用函数的 `Alias` 构造，可以像其他 FROM 子句一样使用，如 使用别名 中介绍的。下面我们示例了 `json_each()` 函数，虽然它在 PostgreSQL 上很常见，但也受到了现代版本 SQLite 的支持：

```py
>>> onetwothree = func.json_each('["one", "two", "three"]').table_valued("value")
>>> stmt = select(onetwothree).where(onetwothree.c.value.in_(["two", "three"]))
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     result.all()
BEGIN  (implicit)
SELECT  anon_1.value
FROM  json_each(?)  AS  anon_1
WHERE  anon_1.value  IN  (?,  ?)
[...]  ('["one", "two", "three"]',  'two',  'three')
[('two',), ('three',)]
ROLLBACK 
```

在上面的示例中，我们使用了 SQLite 和 PostgreSQL 支持的 `json_each()` JSON 函数，以生成一个带有一个称为 `value` 的单列的表值表达式，然后选择了其中的两行。

另请参见

表值函数 - 在 PostgreSQL 文档中 - 此部分将详细介绍一些额外的语法，例如特殊的列派生和“WITH ORDINALITY”，这些语法已知可与 PostgreSQL 一起使用。

#### 列值函数 - 表值函数作为标量列

PostgreSQL 和 Oracle 支持的一个特殊语法是在 FROM 子句中引用函数，然后在 SELECT 语句或其他列表达式上下文的列子句中将其自身作为单列传递。PostgreSQL 在诸如 `json_array_elements()`、`json_object_keys()`、`json_each_text()`、`json_each()` 等函数中广泛使用此语法。

SQLAlchemy 将此称为“列值”函数，并可通过将 `FunctionElement.column_valued()` 修饰符应用于 `Function` 构造来使用：

```py
>>> from sqlalchemy import select, func
>>> stmt = select(func.json_array_elements('["one", "two"]').column_valued("x"))
>>> print(stmt)
SELECT  x
FROM  json_array_elements(:json_array_elements_1)  AS  x 
```

Oracle 方言也支持“列值”形式，可用于自定义 SQL 函数：

```py
>>> from sqlalchemy.dialects import oracle
>>> stmt = select(func.scalar_strings(5).column_valued("s"))
>>> print(stmt.compile(dialect=oracle.dialect()))
SELECT  s.COLUMN_VALUE
FROM  TABLE  (scalar_strings(:scalar_strings_1))  s 
```

另请参见

列值函数 - 在 PostgreSQL 文档中。

## 数据类型转换和类型强制转换

在 SQL 中，我们经常需要明确指示表达式的数据类型，要么是告诉数据库在其他情况下模棱两可的表达式中期望的类型，要么在某些情况下，当我们想要将 SQL 表达式的隐含数据类型转换为其他东西时。SQL CAST 关键字用于此任务，在 SQLAlchemy 中由`cast()`函数提供。该函数接受列表达式和数据类型对象作为参数，如下所示，我们从`user_table.c.id`列对象产生一个 SQL 表达式`CAST(user_account.id AS VARCHAR)`：

```py
>>> from sqlalchemy import cast
>>> stmt = select(cast(user_table.c.id, String))
>>> with engine.connect() as conn:
...     result = conn.execute(stmt)
...     result.all()
BEGIN  (implicit)
SELECT  CAST(user_account.id  AS  VARCHAR)  AS  id
FROM  user_account
[...]  ()
[('1',), ('2',), ('3',)]
ROLLBACK 
```

`cast()`函数不仅会渲染 SQL CAST 语法，还会生成一个 SQLAlchemy 列表达式，在 Python 端也将作为给定的数据类型。一个被`cast()`转换为`JSON`的字符串表达式将获得 JSON 下标和比较运算符，例如：

```py
>>> from sqlalchemy import JSON
>>> print(cast("{'a': 'b'}", JSON)["a"])
CAST(:param_1  AS  JSON)[:param_2] 
```

### type_coerce() - 一个仅限 Python 的“转换”

有时候，需要让 SQLAlchemy 知道表达式的数据类型，出于上述所有原因，但不要在 SQL 端渲染 CAST 表达式本身，因为它可能会干扰已经在没有它的情况下正常工作的 SQL 操作。对于这种相当常见的用例，还有另一个函数`type_coerce()`，它与`cast()`密切相关，它设置一个 Python 表达式为具有特定 SQL 数据库类型，但不会在数据库端渲染 CAST 关键字或数据类型。当处理`JSON`数据类型时，`type_coerce()`特别重要，它通常与不同平台上的面向字符串的数据类型有着错综复杂的关系，甚至可能不是一个显式的数据类型，比如在 SQLite 和 MariaDB 上。下面，我们使用`type_coerce()`将 Python 结构传递为 JSON 字符串到 MySQL 的 JSON 函数之一：

```py
>>> import json
>>> from sqlalchemy import JSON
>>> from sqlalchemy import type_coerce
>>> from sqlalchemy.dialects import mysql
>>> s = select(type_coerce({"some_key": {"foo": "bar"}}, JSON)["some_key"])
>>> print(s.compile(dialect=mysql.dialect()))
SELECT  JSON_EXTRACT(%s,  %s)  AS  anon_1 
```

上面，MySQL 的`JSON_EXTRACT` SQL 函数被调用，因为我们使用`type_coerce()`指示我们的 Python 字典应该被视为`JSON`。Python 的`__getitem__`运算符，在这种情况下是`['some_key']`，由此产生并允许一个`JSON_EXTRACT`路径表达式（但在这种情况下没有显示，最终它将是`'$."some_key"'`）被渲染。

### type_coerce() - 一个仅限 Python 的“强制转换”

有时候，需要让 SQLAlchemy 了解表达式的数据类型，出于上述所有原因，但不要在 SQL 端渲染 CAST 表达式本身，因为这可能会干扰已经可以在没有它的情况下工作的 SQL 操作。对于这种相当常见的用例，有另一个函数`type_coerce()`，它与`cast()`密切相关，它设置一个 Python 表达式为具有特定 SQL 数据库类型，但不在数据库端渲染 CAST 关键字或数据类型。当处理`JSON`数据类型时，`type_coerce()`尤为重要，它通常与不同平台上的字符串导向数据类型有复杂的关系，甚至可能不是显式数据类型，例如在 SQLite 和 MariaDB 上。在下面的例子中，我们使用`type_coerce()`将 Python 结构传递为 JSON 字符串到 MySQL 的一个 JSON 函数中：

```py
>>> import json
>>> from sqlalchemy import JSON
>>> from sqlalchemy import type_coerce
>>> from sqlalchemy.dialects import mysql
>>> s = select(type_coerce({"some_key": {"foo": "bar"}}, JSON)["some_key"])
>>> print(s.compile(dialect=mysql.dialect()))
SELECT  JSON_EXTRACT(%s,  %s)  AS  anon_1 
```

上面，MySQL 的`JSON_EXTRACT` SQL 函数被调用，因为我们使用`type_coerce()`指示我们的 Python 字典应该被视为`JSON`。Python 的`__getitem__`运算符，在这种情况下是`['some_key']`，由此产生并允许一个`JSON_EXTRACT`路径表达式（但在这种情况下没有显示，最终它将是`'$."some_key"'`）被渲染。
