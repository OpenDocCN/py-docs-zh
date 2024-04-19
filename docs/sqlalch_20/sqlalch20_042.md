# 为 ORM 映射类编写 SELECT 语句

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/select.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/select.html)

关于本文档

本节利用了首次在 SQLAlchemy 统一教程中展示的 ORM 映射，显示在声明映射类一节中。

查看此页面的 ORM 设置。

SELECT 语句由 `select()` 函数生成，该函数返回一个 `Select` 对象。要返回的实体和/或 SQL 表达式（即“columns”子句）按位置传递给该函数。然后，使用其他方法生成完整的语句，例如下面所示的 `Select.where()` 方法：

```py
>>> from sqlalchemy import select
>>> stmt = select(User).where(User.name == "spongebob")
```

给定一个完成的 `Select` 对象，为了在 ORM 中执行并获取行，对象被传递给 `Session.execute()`，然后返回一个 `Result` 对象：

```py
>>> result = session.execute(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('spongebob',)
>>> for user_obj in result.scalars():
...     print(f"{user_obj.name} {user_obj.fullname}")
spongebob Spongebob Squarepants
```

## 选择 ORM 实体和属性

`select()` 构造函数接受 ORM 实体，包括映射类以及表示映射列的类级别属性，这些属性在构造时转换为 ORM 注解的 `FromClause` 和 `ColumnElement` 元素。

包含 ORM 注解实体的 `Select` 对象通常使用 `Session` 对象执行，而不是 `Connection` 对象，以便 ORM 相关功能生效，包括可以返回 ORM 映射对象的实例。直接使用 `Connection` 时，结果行仅包含列级数据。

### 选择 ORM 实体

下面我们从`User`实体中进行选择，生成一个从`User`映射到的`Table`中进行选择的`Select`：

```py
>>> result = session.execute(select(User).order_by(User.id))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  () 
```

当从 ORM 实体中进行选择时，实体本身作为具有单个元素的行返回结果，而不是一系列单独的列；例如上面，`Result`返回仅在每行具有单个元素的`Row`对象，该元素保留着一个`User`对象：

```py
>>> result.all()
[(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),),
 (User(id=2, name='sandy', fullname='Sandy Cheeks'),),
 (User(id=3, name='patrick', fullname='Patrick Star'),),
 (User(id=4, name='squidward', fullname='Squidward Tentacles'),),
 (User(id=5, name='ehkrabs', fullname='Eugene H. Krabs'),)]
```

当选择包含 ORM 实体的单元素行列表时，通常会跳过生成`Row`对象，而是直接接收 ORM 实体。最简单的方法是使用`Session.scalars()`方法来执行，而不是`Session.execute()`方法，这样就会返回一个`ScalarResult`对象，该对象产生单个元素而不是行：

```py
>>> session.scalars(select(User).order_by(User.id)).all()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  ()
[User(id=1, name='spongebob', fullname='Spongebob Squarepants'),
 User(id=2, name='sandy', fullname='Sandy Cheeks'),
 User(id=3, name='patrick', fullname='Patrick Star'),
 User(id=4, name='squidward', fullname='Squidward Tentacles'),
 User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]
```

调用`Session.scalars()`方法相当于调用`Session.execute()`来接收一个`Result`对象，然后调用`Result.scalars()`来接收一个`ScalarResult`对象。###同时选择多个 ORM 实体

`select()`函数一次接受任意数量的 ORM 类和/或列表达式，包括可以请求多个 ORM 类的情况。当从多个 ORM 类中选择时，它们在每个结果行中根据其类名命名。在下面的示例中，针对`User`和`Address`进行 SELECT 的结果行将以`User`和`Address`的名称引用它们：

```py
>>> stmt = select(User, Address).join(User.addresses).order_by(User.id, Address.id)
>>> for row in session.execute(stmt):
...     print(f"{row.User.name} {row.Address.email_address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id
[...]  ()
spongebob spongebob@sqlalchemy.org
sandy sandy@sqlalchemy.org
sandy squirrel@squirrelpower.org
patrick pat999@aol.com
squidward stentcl@sqlalchemy.org
```

如果我们想要为这些实体在行中分配不同的名称，我们将使用`aliased()`构造，使用`aliased.name`参数将它们别名为显式名称：

```py
>>> from sqlalchemy.orm import aliased
>>> user_cls = aliased(User, name="user_cls")
>>> email_cls = aliased(Address, name="email")
>>> stmt = (
...     select(user_cls, email_cls)
...     .join(user_cls.addresses.of_type(email_cls))
...     .order_by(user_cls.id, email_cls.id)
... )
>>> row = session.execute(stmt).first()
SELECT  user_cls.id,  user_cls.name,  user_cls.fullname,
email.id  AS  id_1,  email.user_id,  email.email_address
FROM  user_account  AS  user_cls  JOIN  address  AS  email
ON  user_cls.id  =  email.user_id  ORDER  BY  user_cls.id,  email.id
[...]  ()
>>> print(f"{row.user_cls.name}  {row.email.email_address}")
spongebob spongebob@sqlalchemy.org
```

上述别名形式在使用关系连接别名目标中进一步讨论。

可以使用`Select`构造来向其列子句添加 ORM 类和/或列表达式，方法是使用`Select.add_columns()`方法。我们也可以使用这种形式来生成上述语句：

```py
>>> stmt = (
...     select(User).join(User.addresses).add_columns(Address).order_by(User.id, Address.id)
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id 
```

### 选择单个属性

映射类上的属性，如`User.name`和`Address.email_address`，当传递给`select()`时，可以像`Column`或其他 SQL 表达式对象一样使用。针对特定列创建一个`select()`将返回`Row`对象，而**不是**像`User`或`Address`对象那样的实体。每个`Row`将单独表示每一列：

```py
>>> result = session.execute(
...     select(User.name, Address.email_address)
...     .join(User.addresses)
...     .order_by(User.id, Address.id)
... )
SELECT  user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id
[...]  () 
```

上述语句返回具有`name`和`email_address`列的`Row`对象，如下所示的运行时演示：

```py
>>> for row in result:
...     print(f"{row.name}  {row.email_address}")
spongebob  spongebob@sqlalchemy.org
sandy  sandy@sqlalchemy.org
sandy  squirrel@squirrelpower.org
patrick  pat999@aol.com
squidward  stentcl@sqlalchemy.org
```

### 使用 Bundle 对选定属性进行分组

`Bundle` 构造是一个可扩展的仅限 ORM 的构造，允许将列表达式集合分组在结果行中：

```py
>>> from sqlalchemy.orm import Bundle
>>> stmt = select(
...     Bundle("user", User.name, User.fullname),
...     Bundle("email", Address.email_address),
... ).join_from(User, Address)
>>> for row in session.execute(stmt):
...     print(f"{row.user.name} {row.user.fullname} {row.email.email_address}")
SELECT  user_account.name,  user_account.fullname,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
spongebob Spongebob Squarepants spongebob@sqlalchemy.org
sandy Sandy Cheeks sandy@sqlalchemy.org
sandy Sandy Cheeks squirrel@squirrelpower.org
patrick Patrick Star pat999@aol.com
squidward Squidward Tentacles stentcl@sqlalchemy.org
```

`Bundle` 可能对创建轻量级视图和自定义列分组很有用。`Bundle` 也可以被子类化以返回替代数据结构；参见`Bundle.create_row_processor()` 以获取示例。

另请参见

`Bundle`

`Bundle.create_row_processor()`  ### 选择 ORM 别名

如在使用别名的教程中讨论的那样，要创建 ORM 实体的 SQL 别名，可以使用针对映射类的`aliased()`构造实现：

```py
>>> from sqlalchemy.orm import aliased
>>> u1 = aliased(User)
>>> print(select(u1).order_by(u1.id))
SELECT  user_account_1.id,  user_account_1.name,  user_account_1.fullname
FROM  user_account  AS  user_account_1  ORDER  BY  user_account_1.id 
```

与使用`Table.alias()`时的情况一样，SQL 别名是匿名命名的。对于从具有显式名称的行中选择实体的情况，也可以传递`aliased.name`参数：

```py
>>> from sqlalchemy.orm import aliased
>>> u1 = aliased(User, name="u1")
>>> stmt = select(u1).order_by(u1.id)
>>> row = session.execute(stmt).first()
SELECT  u1.id,  u1.name,  u1.fullname
FROM  user_account  AS  u1  ORDER  BY  u1.id
[...]  ()
>>> print(f"{row.u1.name}")
spongebob
```

另见

`aliased` 构造在几种情况下都很重要，包括：

+   使用 ORM 的子查询；章节从子查询中选择实体和加入子查询进一步讨论了这一点。

+   控制结果集中实体的名称；参见同时选择多个 ORM 实体以查看示例

+   多次连接到相同的 ORM 实体；参见使用关系连接到别名目标以查看示例。###从文本语句获取 ORM 结果

ORM 支持从其他来源的 SELECT 语句加载实体。典型的用例是文本 SELECT 语句，在 SQLAlchemy 中使用`text()`构造表示。可以使用`text()`构造增强关于该语句将加载的 ORM 映射列的信息；然后可以将其与 ORM 实体本身关联，以便基于此语句加载 ORM 对象。

给定一个文本 SQL 语句，我们希望从中加载：

```py
>>> from sqlalchemy import text
>>> textual_sql = text("SELECT id, name, fullname FROM user_account ORDER BY id")
```

通过使用`TextClause.columns()`方法，我们可以为语句添加列信息；当调用此方法时，`TextClause`对象被转换为一个`TextualSelect`对象，该对象扮演的角色类似于`Select`构造。`TextClause.columns()`方法通常传递`Column`对象或等效对象，在这种情况下，我们可以直接使用`User`类上映射的属性：

```py
>>> textual_sql = textual_sql.columns(User.id, User.name, User.fullname)
```

现在我们有了一个经过 ORM 配置的 SQL 构造，按照给定的方式，可以单独加载“id”、“name”和“fullname”列。要将此 SELECT 语句用作完整`User`实体的源，则可以使用`Select.from_statement()`方法将这些列链接到常规的 ORM 启用的`Select`构造中：

```py
>>> orm_sql = select(User).from_statement(textual_sql)
>>> for user_obj in session.execute(orm_sql).scalars():
...     print(user_obj)
SELECT  id,  name,  fullname  FROM  user_account  ORDER  BY  id
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

同一个`TextualSelect`对象也可以使用`TextualSelect.subquery()`方法转换为子查询，并使用`aliased()`构造将其链接到`User`实体，方式与下面讨论的从子查询中选择实体类似：

```py
>>> orm_subquery = aliased(User, textual_sql.subquery())
>>> stmt = select(orm_subquery)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  id,  name,  fullname  FROM  user_account  ORDER  BY  id)  AS  anon_1
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

直接使用`TextualSelect`与`Select.from_statement()`相比，使用`aliased()`的区别在于，在前一种情况下，生成的 SQL 中不会产生子查询。在某些情景下，这样做从性能或复杂性的角度来看可能是有利的。  ### 从子查询中选择实体

在前一节讨论的`aliased()`构造中，可以与任何来自诸如`Select.subquery()`之类的方法的`Subuqery`构造一起使用，以将 ORM 实体链接到该子查询返回的列；子查询返回的列与实体映射的列之间必须存在**列对应**关系，这意味着子查询最终需要来自这些实体，就像下面的示例中一样：

```py
>>> inner_stmt = select(User).where(User.id < 7).order_by(User.id)
>>> subq = inner_stmt.subquery()
>>> aliased_user = aliased(User, subq)
>>> stmt = select(aliased_user)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
  SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  <  ?  ORDER  BY  user_account.id)  AS  anon_1
[generated  in  ...]  (7,)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

另请参见

ORM 实体子查询/CTEs - 在 SQLAlchemy 统一教程中

加入到子查询  ### 从 UNION 和其他集合操作中选择实体

`union()`和`union_all()` 函数是最常见的集合操作之一，与`except_()`、`intersect()`等其他集合操作一起，它们生成一个称为`CompoundSelect`的对象，该对象由多个使用集合操作关键字连接的`Select`构造组成。ORM 实体可以使用`Select.from_statement()`方法从简单的复合选择中选择，该方法如在从文本语句中获取 ORM 结果中所示。在这种方法中，UNION 语句是将呈现的完整语句，不能在使用`Select.from_statement()`之后添加额外的条件：

```py
>>> from sqlalchemy import union_all
>>> u = union_all(
...     select(User).where(User.id < 2), select(User).where(User.id == 3)
... ).order_by(User.id)
>>> stmt = select(User).from_statement(u)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.id  <  ?  UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.id  =  ?  ORDER  BY  id
[generated  in  ...]  (2,  3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
```

`CompoundSelect`构造可以更灵活地在查询中使用，可以通过将其组织成子查询并使用`aliased()`将其链接到 ORM 实体来进一步修改，如在从子查询中选择实体中所示。在下面的示例中，我们首先使用`CompoundSelect.subquery()`创建 UNION ALL 语句的子查询，然后将其打包到`aliased()`构造中，在其中可以像其他映射实体一样在`select()`构造中使用，包括我们可以基于其导出的列添加过滤和排序条件：

```py
>>> subq = union_all(
...     select(User).where(User.id < 2), select(User).where(User.id == 3)
... ).subquery()
>>> user_alias = aliased(User, subq)
>>> stmt = select(user_alias).order_by(user_alias.id)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  <  ?  UNION  ALL  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  =  ?)  AS  anon_1  ORDER  BY  anon_1.id
[generated  in  ...]  (2,  3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
```

另请参阅

从联合中选择 ORM 实体 - 在 SQLAlchemy 统一教程中##连接

`Select.join()`和`Select.join_from()`方法用于构建针对 SELECT 语句的 SQL JOINs。

本节将详细介绍这些方法的 ORM 用例。有关从核心角度使用它们的通用概述，请参阅明确的 FROM 子句和 JOINs 中的 SQLAlchemy 统一教程。

在 ORM 上下文中使用`Select.join()`进行 2.0 风格查询的用法大致相同，除了遗留用例外，与 1.x 风格查询中的`Query.join()`方法的用法相似。

### 简单的关系连接

考虑两个类`User`和`Address`之间的映射，其中关系`User.addresses`表示与每个`User`关联的`Address`对象的集合。 `Select.join()`的最常见用法是沿着这个关系创建 JOIN，使用`User.addresses`属性作为指示器来指示这应该如何发生：

```py
>>> stmt = select(User).join(User.addresses)
```

在上文中，对`User.addresses`的`Select.join()`调用将导致 SQL 大致等效于：

```py
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

在上面的示例中，我们将`User.addresses`称为传递给`Select.join()`的“on clause”，即它指示如何构造 JOIN 语句中的“ON”部分。

Tip

注意，使用`Select.join()`从一个实体连接到另一个实体会影响 SELECT 语句的 FROM 子句，但不会影响列子句；此示例中的 SELECT 语句将继续只返回`User`实体的行。要同时从`User`和`Address`选择列/实体，必须在`select()`函数中命名`Address`实体，或者使用`Select.add_columns()`方法在之后将其添加到`Select`构造中。请参阅 同时选择多个 ORM 实体 部分以了解这两种形式的示例。

### 链式多重连接

要构建一系列连接，可以使用多个`Select.join()`调用。关系绑定属性一次暗示了连接的左侧和右侧。考虑额外的实体`Order`和`Item`，其中`User.orders`关系引用了`Order`实体，而`Order.items`关系通过关联表`order_items`引用了`Item`实体。两个`Select.join()`调用将首先从`User`到`Order`进行连接，然后从`Order`到`Item`进行第二次连接。但是，由于`Order.items`是多对多关系，它导致两个单独的 JOIN 元素，总共在生成的 SQL 中有三个 JOIN 元素：

```py
>>> stmt = select(User).join(User.orders).join(Order.items)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  user_order  ON  user_account.id  =  user_order.user_id
JOIN  order_items  AS  order_items_1  ON  user_order.id  =  order_items_1.order_id
JOIN  item  ON  item.id  =  order_items_1.item_id 
```

每次调用`Select.join()`方法的顺序只有在我们想要连接的“左”侧需要在 FROM 列表中出现时才有意义；如果我们指定`select(User).join(Order.items).join(User.orders)`，则`Select.join()`将不知道如何正确连接，并引发错误。在正确的做法中，应以使 JOIN 子句在 SQL 中呈现方式对齐的方式调用`Select.join()`方法，并且每次调用应表示从之前的内容清晰地链接过来。

我们在 FROM 子句中目标的所有元素仍然可以作为继续连接 FROM 的潜在点。例如，我们可以继续添加其他元素来连接 FROM 上面的`User`实体，例如在连接链中添加`User.addresses`关系：

```py
>>> stmt = select(User).join(User.orders).join(Order.items).join(User.addresses)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  user_order  ON  user_account.id  =  user_order.user_id
JOIN  order_items  AS  order_items_1  ON  user_order.id  =  order_items_1.order_id
JOIN  item  ON  item.id  =  order_items_1.item_id
JOIN  address  ON  user_account.id  =  address.user_id 
```

### 连接到目标实体

第二种形式的`Select.join()`允许任何映射实体或核心可选择的构造作为目标。在这种用法中，`Select.join()`将尝试**推断**JOIN 的 ON 子句，使用两个实体之间的自然外键关系：

```py
>>> stmt = select(User).join(Address)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

在上述调用形式中，`Select.join()`被调用以自动推断“on 子句”。如果两个映射的`Table`构造之间没有`ForeignKeyConstraint`设置，或者存在多个使适当约束使用变得模糊的 `ForeignKeyConstraint` 链接时，此调用形式最终会引发错误。

注意

当使用 `Select.join()`或 `Select.join_from()` 而不指示 ON 子句时，ORM 配置的`relationship()`构造不会被考虑。只有在尝试推断 JOIN 的 ON 子句时，才会查阅映射的`Table`对象级别上的实体之间配置的`ForeignKeyConstraint`关系。

### 连接到具有 ON 子句的目标

第三种调用形式允许目标实体以及 ON 子句都明确传递。包含 SQL 表达式作为 ON 子句的示例如下：

```py
>>> stmt = select(User).join(Address, User.id == Address.user_id)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

基于表达式的 ON 子句也可以是一个`relationship()`-绑定属性，就像在简单关系连接中使用的那样：

```py
>>> stmt = select(User).join(Address, User.addresses)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

上面的例子看起来是多余的，因为它以两种不同的方式指示了 `Address` 的目标；然而，当连接到别名实体时，这种形式的效用就变得明显了；请参阅 Using Relationship to join between aliased targets 部分以查看示例。### 结合 Relationship 与自定义 ON 条件

`relationship()` 构造生成的 ON 子句可能会通过附加条件进行增强。这对于快速限制特定连接的范围以及配置加载器策略（如 `joinedload()` 和 `selectinload()`）等情况非常有用。`PropComparator.and_()` 方法按位置接受一系列 SQL 表达式，这些表达式将通过 AND 连接到 JOIN 的 ON 子句。例如，如果我们想要从 `User` 连接到 `Address`，但也只限制 ON 条件为特定的电子邮件地址：

```py
>>> stmt = select(User.fullname).join(
...     User.addresses.and_(Address.email_address == "squirrel@squirrelpower.org")
... )
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
JOIN  address  ON  user_account.id  =  address.user_id  AND  address.email_address  =  ?
[...]  ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
```

另见

`PropComparator.and_()` 方法也适用于加载器策略，如 `joinedload()` 和 `selectinload()`。请参阅 Adding Criteria to loader options 部分。### 使用 Relationship 在别名目标之间进行连接

在使用 `relationship()`-绑定属性指示 ON 子句构建连接时，可以将 Joins to a Target with an ON Clause 中说明的两个参数语法扩展到与 `aliased()` 构造一起使用，以指示 SQL 别名作为连接的目标，同时仍然利用 `relationship()`-绑定属性指示 ON 子句，如下例所示，其中 `User` 实体两次与两个不同的 `aliased()` 构造连接到 `Address` 实体：

```py
>>> address_alias_1 = aliased(Address)
>>> address_alias_2 = aliased(Address)
>>> stmt = (
...     select(User)
...     .join(address_alias_1, User.addresses)
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join(address_alias_2, User.addresses)
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

可以使用修饰符 `PropComparator.of_type()` 来更简洁地表达相同的模式，该修饰符可应用于与 `relationship()` 绑定的属性，一次性传递目标实体以指示一步中的目标。下面的示例使用 `PropComparator.of_type()` 来生成与刚刚展示的相同的 SQL 语句：

```py
>>> print(
...     select(User)
...     .join(User.addresses.of_type(address_alias_1))
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join(User.addresses.of_type(address_alias_2))
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

要利用 `relationship()` 来构建来自别名实体的连接，直接从 `aliased()` 构造中获取属性即可：

```py
>>> user_alias_1 = aliased(User)
>>> print(select(user_alias_1.name).join(user_alias_1.addresses))
SELECT  user_account_1.name
FROM  user_account  AS  user_account_1
JOIN  address  ON  user_account_1.id  =  address.user_id 
```  ### 加入到子查询

连接的目标可以是任何“可选择”的实体，包括子查询。在使用 ORM 时，通常将这些目标陈述为 `aliased()` 构造的术语，但这不是严格要求的，特别是如果连接的实体不在结果中返回。例如，要从 `User` 实体连接到 `Address` 实体，其中 `Address` 实体表示为行限制的子查询，我们首先使用 `Select.subquery()` 构造了一个 `Subquery` 对象，然后可以将其用作 `Select.join()` 方法的目标：

```py
>>> subq = select(Address).where(Address.email_address == "pat999@aol.com").subquery()
>>> stmt = select(User).join(subq, User.id == subq.c.user_id)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  :email_address_1)  AS  anon_1
ON  user_account.id  =  anon_1.user_id 
```

上述 SELECT 语句在通过 `Session.execute()` 调用时，将返回包含 `User` 实体但不包含 `Address` 实体的行。为了将 `Address` 实体包含到将在结果集中返回的实体集合中，我们对 `Address` 实体和 `Subquery` 对象构造了一个 `aliased()` 对象。我们还可能希望对 `aliased()` 构造应用一个名称，如下面使用的 `"address"`，这样我们就可以在结果行中按名称引用它：

```py
>>> address_subq = aliased(Address, subq, name="address")
>>> stmt = select(User, address_subq).join(address_subq)
>>> for row in session.execute(stmt):
...     print(f"{row.User} {row.address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.user_id,  anon_1.email_address
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
[...]  ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
```

### 加入到子查询的关联路径

在上一节中说明的子查询形式可以使用`relationship()`绑定属性更具体地表示，使用使用 Relationship 在别名目标之间进行连接中指示的形式之一。例如，要创建相同的连接，同时确保连接是沿着特定`relationship()`进行的，我们可以使用`PropComparator.of_type()`方法，传递包含连接目标的`aliased()` 构造，该目标是`Subquery`对象的。

```py
>>> address_subq = aliased(Address, subq, name="address")
>>> stmt = select(User, address_subq).join(User.addresses.of_type(address_subq))
>>> for row in session.execute(stmt):
...     print(f"{row.User} {row.address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.user_id,  anon_1.email_address
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
[...]  ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
```

### 引用多个实体的子查询

包含跨越多个 ORM 实体列的子查询可以一次应用于多个`aliased()` 构造，并在同一`Select`构造中针对每个实体分别使用。然而，从 ORM / Python 的角度来看，渲染的 SQL 将继续将所有这些`aliased()` 构造视为相同的子查询，但可以通过使用适当的`aliased()` 构造引用不同的返回值和对象属性。

例如，给定同时引用`User`和`Address`的子查询：

```py
>>> user_address_subq = (
...     select(User.id, User.name, User.fullname, Address.id, Address.email_address)
...     .join_from(User, Address)
...     .where(Address.email_address.in_(["pat999@aol.com", "squirrel@squirrelpower.org"]))
...     .subquery()
... )
```

我们可以针对`User`和`Address`分别创建对同一对象的`aliased()` 构造：

```py
>>> user_alias = aliased(User, user_address_subq, name="user")
>>> address_alias = aliased(Address, user_address_subq, name="address")
```

从两个实体中选择的`Select`构造将一次渲染子查询，但在结果行上下文中可以同时返回`User`和`Address`类的对象：

```py
>>> stmt = select(user_alias, address_alias).where(user_alias.name == "sandy")
>>> for row in session.execute(stmt):
...     print(f"{row.user} {row.address}")
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname,  anon_1.id_1,  anon_1.email_address
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,
user_account.fullname  AS  fullname,  address.id  AS  id_1,
address.email_address  AS  email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  address.email_address  IN  (?,  ?))  AS  anon_1
WHERE  anon_1.name  =  ?
[...]  ('pat999@aol.com',  'squirrel@squirrelpower.org',  'sandy')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='squirrel@squirrelpower.org')
```

### 设置连接中最左侧的 FROM 子句

在当前`Select`状态的左侧与我们要连接的内容不一致的情况下，可以使用`Select.join_from()` 方法：

```py
>>> stmt = select(Address).join_from(User, User.addresses).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

`Select.join_from()` 方法接受两个或三个参数，形式可以是 `(<join from>, <onclause>)`，或者 `(<join from>, <join to>, [<onclause>])`：

```py
>>> stmt = select(Address).join_from(User, Address).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

为了为 SELECT 设置初始的 FROM 子句，以便随后可以使用`Select.join()`，也可以使用`Select.select_from()`方法：

```py
>>> stmt = select(Address).select_from(User).join(Address).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

提示

`Select.select_from()`方法实际上并不决定 FROM 子句中表的顺序。如果语句还引用了引用不同顺序的现有表的`Join`构造，那么`Join`构造将优先。当我们使用`Select.join()`和`Select.join_from()`等方法时，这些方法最终会创建这样一个`Join`对象。因此，在这种情况下，我们可以看到`Select.select_from()`的内容被覆盖：

```py
>>> stmt = select(Address).select_from(User).join(Address.user).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

在上面的例子中，我们看到 FROM 子句是`address JOIN user_account`，尽管我们首先声明了`select_from(User)`。由于`.join(Address.user)`方法调用，该语句最终等同于以下内容：

```py
>>> from sqlalchemy.sql import join
>>>
>>> user_table = User.__table__
>>> address_table = Address.__table__
>>>
>>> j = address_table.join(user_table, user_table.c.id == address_table.c.user_id)
>>> stmt = (
...     select(address_table)
...     .select_from(user_table)
...     .select_from(j)
...     .where(user_table.c.name == "sandy")
... )
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

上面的`Join`构造被添加为`Select.select_from()`列表中的另一个条目，它取代了之前的条目。## 关系 WHERE 运算符

除了在`Select.join()`和`Select.join_from()`方法中使用`relationship()`构造之外，`relationship()`还在帮助构建通常用于 WHERE 子句的 SQL 表达式，使用`Select.where()`方法。

### EXISTS 形式：has() / any()

`Exists` 构造首次出现在 SQLAlchemy 统一教程 的 EXISTS 子查询 部分。此对象用于在标量子查询中与 SQL EXISTS 关键字一起呈现。`relationship()` 构造提供了一些辅助方法，可用于生成一些常见的 EXISTS 样式的查询，这些查询涉及关系。

对于一对多关系，例如 `User.addresses`，可以使用与 `user_account` 表相关联的 `address` 表的 EXISTS 来产生一个 `PropComparator.any()`。此方法接受一个可选的 WHERE 条件来限制子查询匹配的行数：

```py
>>> stmt = select(User.fullname).where(
...     User.addresses.any(Address.email_address == "squirrel@squirrelpower.org")
... )
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
WHERE  EXISTS  (SELECT  1
FROM  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  ?)
[...]  ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
```

由于 EXISTS 对于负查找更有效，因此一个常见的查询是定位不存在相关实体的实体。这可以通过短语 `~User.addresses.any()` 来简洁地实现，以选择没有相关 `Address` 行的 `User` 实体：

```py
>>> stmt = select(User.fullname).where(~User.addresses.any())
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
WHERE  NOT  (EXISTS  (SELECT  1
FROM  address
WHERE  user_account.id  =  address.user_id))
[...]  ()
[('Eugene H. Krabs',)]
```

`PropComparator.has()` 方法的工作方式基本与 `PropComparator.any()` 相同，不同之处在于它用于多对一关系，例如，如果我们想要定位所有属于 “sandy” 的 `Address` 对象。

```py
>>> stmt = select(Address.email_address).where(Address.user.has(User.name == "sandy"))
>>> session.execute(stmt).all()
SELECT  address.email_address
FROM  address
WHERE  EXISTS  (SELECT  1
FROM  user_account
WHERE  user_account.id  =  address.user_id  AND  user_account.name  =  ?)
[...]  ('sandy',)
[('sandy@sqlalchemy.org',), ('squirrel@squirrelpower.org',)]
```  ### 关系实例比较运算符

`relationship()` 绑定属性还提供了一些 SQL 构造实现，这些实现旨在根据相关对象的特定实例来过滤 `relationship()` 绑定属性，该实例可以从给定的 持久化（或不太常见的 分离）对象实例中拆解适当的属性值，并按照目标 `relationship()` 构造 WHERE 条件。

+   **多对一等于比较** - 可以将特定对象实例与多对一关系进行比较，以选择目标实体的外键与给定对象的主键值匹配的行：

    ```py
    >>> user_obj = session.get(User, 1)
    SELECT  ...
    >>> print(select(Address).where(Address.user == user_obj))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  :param_1  =  address.user_id 
    ```

+   **多对一不等于比较** - 也可以使用不等于运算符：

    ```py
    >>> print(select(Address).where(Address.user != user_obj))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  address.user_id  !=  :user_id_1  OR  address.user_id  IS  NULL 
    ```

+   **对象包含在一对多集合中** - 这本质上是“等于”比较的一对多版本，选择主键等于相关对象中外键值的行：

    ```py
    >>> address_obj = session.get(Address, 1)
    SELECT  ...
    >>> print(select(User).where(User.addresses.contains(address_obj)))
    SELECT  user_account.id,  user_account.name,  user_account.fullname
    FROM  user_account
    WHERE  user_account.id  =  :param_1 
    ```

+   **从一对多的角度看，对象有一个特定的父对象** - `with_parent()` 函数生成一个比较，返回被给定父对象引用的行，这本质上与在多对一侧使用 `==` 操作符相同：

    ```py
    >>> from sqlalchemy.orm import with_parent
    >>> print(select(Address).where(with_parent(user_obj, User.addresses)))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  :param_1  =  address.user_id 
    ```  ## 选择 ORM 实体和属性

`select()` 构造接受 ORM 实体，包括映射类以及表示映射列的类级属性，这些在构建时转换为 ORM 注释 的 `FromClause` 和 `ColumnElement` 元素。

包含 ORM 注释实体的 `Select` 对象通常使用 `Session` 对象执行，而不是使用 `Connection` 对象，以便 ORM 相关功能生效，包括可以返回 ORM 映射对象的实例。直接使用 `Connection` 时，结果行将仅包含列级数据。

### 选择 ORM 实体

下面我们从 `User` 实体中选择，生成一个从 `User` 映射到的映射 `Table` 中选择的 `Select`：

```py
>>> result = session.execute(select(User).order_by(User.id))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  () 
```

在选择 ORM 实体时，实体本身作为具有单个元素的行返回结果，而不是一系列单独的列；例如上面，`Result` 返回仅具有每行单个元素的 `Row` 对象，该元素保持一个 `User` 对象：

```py
>>> result.all()
[(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),),
 (User(id=2, name='sandy', fullname='Sandy Cheeks'),),
 (User(id=3, name='patrick', fullname='Patrick Star'),),
 (User(id=4, name='squidward', fullname='Squidward Tentacles'),),
 (User(id=5, name='ehkrabs', fullname='Eugene H. Krabs'),)]
```

当选择包含 ORM 实体的单元素行列表时，通常会跳过生成`Row`对象，并直接接收 ORM 实体。这最容易通过使用`Session.scalars()`方法执行，而不是使用`Session.execute()`方法来实现，因此返回一个`ScalarResult`对象，该对象产生单个元素而不是行：

```py
>>> session.scalars(select(User).order_by(User.id)).all()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  ()
[User(id=1, name='spongebob', fullname='Spongebob Squarepants'),
 User(id=2, name='sandy', fullname='Sandy Cheeks'),
 User(id=3, name='patrick', fullname='Patrick Star'),
 User(id=4, name='squidward', fullname='Squidward Tentacles'),
 User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]
```

调用`Session.scalars()`方法相当于调用`Session.execute()`来接收一个`Result`对象，然后调用`Result.scalars()`来接收一个`ScalarResult`对象。 ### 同时选择多个 ORM 实体

`select()`函数一次接受任意数量的 ORM 类和/或列表达式，包括可以请求多个 ORM 类。当从多个 ORM 类中选择时，它们在每个结果行中根据其类名命名。在下面的示例中，对`User`和`Address`进行 SELECT 的结果行将以`User`和`Address`的名称引用它们：

```py
>>> stmt = select(User, Address).join(User.addresses).order_by(User.id, Address.id)
>>> for row in session.execute(stmt):
...     print(f"{row.User.name} {row.Address.email_address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id
[...]  ()
spongebob spongebob@sqlalchemy.org
sandy sandy@sqlalchemy.org
sandy squirrel@squirrelpower.org
patrick pat999@aol.com
squidward stentcl@sqlalchemy.org
```

如果我们想要在这些实体中的行上分配不同的名称，我们将使用`aliased()`构造，使用`aliased.name`参数将它们别名为一个明确的名称：

```py
>>> from sqlalchemy.orm import aliased
>>> user_cls = aliased(User, name="user_cls")
>>> email_cls = aliased(Address, name="email")
>>> stmt = (
...     select(user_cls, email_cls)
...     .join(user_cls.addresses.of_type(email_cls))
...     .order_by(user_cls.id, email_cls.id)
... )
>>> row = session.execute(stmt).first()
SELECT  user_cls.id,  user_cls.name,  user_cls.fullname,
email.id  AS  id_1,  email.user_id,  email.email_address
FROM  user_account  AS  user_cls  JOIN  address  AS  email
ON  user_cls.id  =  email.user_id  ORDER  BY  user_cls.id,  email.id
[...]  ()
>>> print(f"{row.user_cls.name}  {row.email.email_address}")
spongebob spongebob@sqlalchemy.org
```

上述的别名形式在使用关系连接别名目标之间有进一步讨论。

一个现有的`Select`构造也可以使用`Select.add_columns()`方法将 ORM 类和/或列表达式添加到其列子句中。我们也可以使用这种形式生成与上述相同的语句：

```py
>>> stmt = (
...     select(User).join(User.addresses).add_columns(Address).order_by(User.id, Address.id)
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id 
```

### 选择单个属性

映射类上的属性，如`User.name`和`Address.email_address`，可以像传递给`select()`的`Column`或其他 SQL 表达式对象一样使用。创建针对特定列的`select()`将返回`Row`对象，而**不是**像`User`或`Address`对象那样的实体。每个`Row`将分别表示每个列：

```py
>>> result = session.execute(
...     select(User.name, Address.email_address)
...     .join(User.addresses)
...     .order_by(User.id, Address.id)
... )
SELECT  user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id
[...]  () 
```

上述语句返回`Row`对象，具有`name`和`email_address`列，如下所示的运行时演示：

```py
>>> for row in result:
...     print(f"{row.name}  {row.email_address}")
spongebob  spongebob@sqlalchemy.org
sandy  sandy@sqlalchemy.org
sandy  squirrel@squirrelpower.org
patrick  pat999@aol.com
squidward  stentcl@sqlalchemy.org
```

### 使用 Bundles 分组选择的属性

`Bundle`构造是一个可扩展的仅 ORM 构造，允许将列表达式集合分组在结果行中：

```py
>>> from sqlalchemy.orm import Bundle
>>> stmt = select(
...     Bundle("user", User.name, User.fullname),
...     Bundle("email", Address.email_address),
... ).join_from(User, Address)
>>> for row in session.execute(stmt):
...     print(f"{row.user.name} {row.user.fullname} {row.email.email_address}")
SELECT  user_account.name,  user_account.fullname,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
spongebob Spongebob Squarepants spongebob@sqlalchemy.org
sandy Sandy Cheeks sandy@sqlalchemy.org
sandy Sandy Cheeks squirrel@squirrelpower.org
patrick Patrick Star pat999@aol.com
squidward Squidward Tentacles stentcl@sqlalchemy.org
```

`Bundle`可能对创建轻量级视图和自定义列分组有用。`Bundle`也可以被子类化以返回替代数据结构；请参阅`Bundle.create_row_processor()`获取示例。

另请参阅

`Bundle`

`Bundle.create_row_processor()`  ### 选择 ORM 别名

如在使用别名的教程中所讨论的，要创建 ORM 实体的 SQL 别名是使用针对映射类的`aliased()`构造实现的：

```py
>>> from sqlalchemy.orm import aliased
>>> u1 = aliased(User)
>>> print(select(u1).order_by(u1.id))
SELECT  user_account_1.id,  user_account_1.name,  user_account_1.fullname
FROM  user_account  AS  user_account_1  ORDER  BY  user_account_1.id 
```

与使用`Table.alias()`时一样，SQL 别名是匿名命名的。对于从具有显式名称的行中选择实体的情况，还可以传递`aliased.name`参数：

```py
>>> from sqlalchemy.orm import aliased
>>> u1 = aliased(User, name="u1")
>>> stmt = select(u1).order_by(u1.id)
>>> row = session.execute(stmt).first()
SELECT  u1.id,  u1.name,  u1.fullname
FROM  user_account  AS  u1  ORDER  BY  u1.id
[...]  ()
>>> print(f"{row.u1.name}")
spongebob
```

另请参阅

`aliased`构造在几个用例中都很重要，包括：

+   利用 ORM 进行子查询；章节从子查询中选择实体和加入子查询进一步讨论了这一点。

+   控制结果集中实体的名称；参见同时选择多个 ORM 实体的示例。

+   加入到同一个 ORM 实体多次；参见使用关系连接别名目标之间的示例。### 从文本语句中获取 ORM 结果

ORM 支持从来自其他来源的 SELECT 语句加载实体。典型用例是文本 SELECT 语句，在 SQLAlchemy 中使用`text()`构造表示。`text()`构造可以通过有关将加载该语句的 ORM 映射列的信息进行增强；然后可以将其与 ORM 实体本身关联，以便基于此语句加载 ORM 对象。

给定一个文本 SQL 语句，我们希望从中加载：

```py
>>> from sqlalchemy import text
>>> textual_sql = text("SELECT id, name, fullname FROM user_account ORDER BY id")
```

我们可以通过使用`TextClause.columns()`方法向语句添加列信息；当调用此方法时，`TextClause`对象转换为`TextualSelect`对象，其扮演与`Select`构造类似的角色。`TextClause.columns()`方法通常传递`Column`对象或等效对象，在这种情况下，我们可以直接使用`User`类上的 ORM 映射属性：

```py
>>> textual_sql = textual_sql.columns(User.id, User.name, User.fullname)
```

现在我们有一个经过 ORM 配置的 SQL 构造，可以分别加载“id”、“name”和“fullname”列。要将此 SELECT 语句作为完整`User`实体的来源，我们可以使用`Select.from_statement()`方法将这些列链接到常规的 ORM 启用的`Select`构造：

```py
>>> orm_sql = select(User).from_statement(textual_sql)
>>> for user_obj in session.execute(orm_sql).scalars():
...     print(user_obj)
SELECT  id,  name,  fullname  FROM  user_account  ORDER  BY  id
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

相同的`TextualSelect`对象也可以使用`TextualSelect.subquery()`方法转换为子查询，并使用`aliased()`构造将其链接到`User`实体中，方式与下文中从子查询中选择实体中所讨论的类似：

```py
>>> orm_subquery = aliased(User, textual_sql.subquery())
>>> stmt = select(orm_subquery)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  id,  name,  fullname  FROM  user_account  ORDER  BY  id)  AS  anon_1
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

直接使用`TextualSelect`和`Select.from_statement()`与使用`aliased()`之间的区别在于，在前一种情况下，结果 SQL 中不会生成子查询。在某些情况下，从性能或复杂性的角度来看，这可能是有利的。### 从子查询中选择实体

前一节讨论的`aliased()`构造可以与任何`Subquery`构造一起使用，该构造来自诸如`Select.subquery()`之类的方法，以将 ORM 实体链接到该子查询返回的列；子查询返回的列与实体映射的列之间必须存在**列对应**关系，这意味着子查询最终需要源自这些实体，就像下面的示例中所示：

```py
>>> inner_stmt = select(User).where(User.id < 7).order_by(User.id)
>>> subq = inner_stmt.subquery()
>>> aliased_user = aliased(User, subq)
>>> stmt = select(aliased_user)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
  SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  <  ?  ORDER  BY  user_account.id)  AS  anon_1
[generated  in  ...]  (7,)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

也请参见

ORM 实体子查询/CTEs - 在 SQLAlchemy 统一教程中

连接到子查询 ### 从 UNION 和其他集合操作中选择实体

`union()` 和 `union_all()` 函数是最常见的集合操作，与其他集合操作（例如 `except_()`、`intersect()` 等）一起提供了一个称为 `CompoundSelect` 的对象，该对象由多个由集合操作关键字连接的 `Select` 构造组成。ORM 实体可以使用 `Select.from_statement()` 方法从简单的复合选择中选择，如前面在从文本语句中获取 ORM 结果中所示。在此方法中，UNION 语句是将呈现的完整语句，不能在使用 `Select.from_statement()` 后添加额外的条件：

```py
>>> from sqlalchemy import union_all
>>> u = union_all(
...     select(User).where(User.id < 2), select(User).where(User.id == 3)
... ).order_by(User.id)
>>> stmt = select(User).from_statement(u)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.id  <  ?  UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.id  =  ?  ORDER  BY  id
[generated  in  ...]  (2,  3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
```

在查询中，`CompoundSelect` 构造可以更灵活地使用，可以通过将其组织成子查询并使用 `aliased()` 连接到 ORM 实体来进一步修改，如前面在从子查询中选择实体中所示。在下面的示例中，我们首先使用 `CompoundSelect.subquery()` 创建 UNION ALL 语句的子查询，然后将其打包到 `aliased()` 构造中，在这里它可以像任何其他映射实体一样在 `select()` 构造中使用，包括我们可以基于其导出列添加过滤和排序条件：

```py
>>> subq = union_all(
...     select(User).where(User.id < 2), select(User).where(User.id == 3)
... ).subquery()
>>> user_alias = aliased(User, subq)
>>> stmt = select(user_alias).order_by(user_alias.id)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  <  ?  UNION  ALL  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  =  ?)  AS  anon_1  ORDER  BY  anon_1.id
[generated  in  ...]  (2,  3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
```

请参阅

从联合中选择 ORM 实体 - 在 SQLAlchemy 统一教程中 ### 选择 ORM 实体

下面我们从 `User` 实体中进行选择，生成一个从 `User` 映射到的映射 `Table` 中进行选择的 `Select`：

```py
>>> result = session.execute(select(User).order_by(User.id))
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  () 
```

当从 ORM 实体中进行选择时，实体本身作为包含单个元素的行返回结果，而不是一系列单独的列；例如上面的例子，`Result` 返回仅具有每行单个元素的 `Row` 对象，该元素保存一个 `User` 对象：

```py
>>> result.all()
[(User(id=1, name='spongebob', fullname='Spongebob Squarepants'),),
 (User(id=2, name='sandy', fullname='Sandy Cheeks'),),
 (User(id=3, name='patrick', fullname='Patrick Star'),),
 (User(id=4, name='squidward', fullname='Squidward Tentacles'),),
 (User(id=5, name='ehkrabs', fullname='Eugene H. Krabs'),)]
```

当选择包含 ORM 实体的单元素行列表时，通常会跳过生成 `Row` 对象，而是直接接收 ORM 实体。这最容易通过使用 `Session.scalars()` 方法执行，而不是 `Session.execute()` 方法来实现，以便返回一个 `ScalarResult` 对象，该对象产生单个元素而不是行：

```py
>>> session.scalars(select(User).order_by(User.id)).all()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  ()
[User(id=1, name='spongebob', fullname='Spongebob Squarepants'),
 User(id=2, name='sandy', fullname='Sandy Cheeks'),
 User(id=3, name='patrick', fullname='Patrick Star'),
 User(id=4, name='squidward', fullname='Squidward Tentacles'),
 User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')]
```

调用 `Session.scalars()` 方法相当于调用 `Session.execute()` 来接收一个 `Result` 对象，然后调用 `Result.scalars()` 来接收一个 `ScalarResult` 对象。

### 同时选择多个 ORM 实体

`select()` 函数一次接受任意数量的 ORM 类和/或列表达式，包括可以请求多个 ORM 类的情况。当从多个 ORM 类中进行 SELECT 时，它们在每个结果行中基于其类名命名。在下面的示例中，对 `User` 和 `Address` 进行 SELECT 的结果行将以 `User` 和 `Address` 为名称进行引用：

```py
>>> stmt = select(User, Address).join(User.addresses).order_by(User.id, Address.id)
>>> for row in session.execute(stmt):
...     print(f"{row.User.name} {row.Address.email_address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id
[...]  ()
spongebob spongebob@sqlalchemy.org
sandy sandy@sqlalchemy.org
sandy squirrel@squirrelpower.org
patrick pat999@aol.com
squidward stentcl@sqlalchemy.org
```

如果我们想要为这些实体在行中分配不同的名称，我们将使用 `aliased()` 构造，并使用 `aliased.name` 参数将它们别名为具有显式名称的实体：

```py
>>> from sqlalchemy.orm import aliased
>>> user_cls = aliased(User, name="user_cls")
>>> email_cls = aliased(Address, name="email")
>>> stmt = (
...     select(user_cls, email_cls)
...     .join(user_cls.addresses.of_type(email_cls))
...     .order_by(user_cls.id, email_cls.id)
... )
>>> row = session.execute(stmt).first()
SELECT  user_cls.id,  user_cls.name,  user_cls.fullname,
email.id  AS  id_1,  email.user_id,  email.email_address
FROM  user_account  AS  user_cls  JOIN  address  AS  email
ON  user_cls.id  =  email.user_id  ORDER  BY  user_cls.id,  email.id
[...]  ()
>>> print(f"{row.user_cls.name}  {row.email.email_address}")
spongebob spongebob@sqlalchemy.org
```

上面的别名形式在使用关系来在别名目标之间进行连接中进一步讨论。

现有的 `Select` 结构也可以使用 `Select.add_columns()` 方法将 ORM 类和/或列表达式添加到其列子句中。我们也可以使用这种形式来生成与上面相同的语句：

```py
>>> stmt = (
...     select(User).join(User.addresses).add_columns(Address).order_by(User.id, Address.id)
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id 
```

### 选择单个属性

映射类上的属性，如 `User.name` 和 `Address.email_address`，当传递给 `select()` 时，可以像 `Column` 或其他 SQL 表达式对象一样使用。创建针对特定列的 `select()` 将返回 `Row` 对象，而不是像 `User` 或 `Address` 对象那样的实体。每个 `Row` 将分别表示每个列：

```py
>>> result = session.execute(
...     select(User.name, Address.email_address)
...     .join(User.addresses)
...     .order_by(User.id, Address.id)
... )
SELECT  user_account.name,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
ORDER  BY  user_account.id,  address.id
[...]  () 
```

上面的语句返回具有 `name` 和 `email_address` 列的 `Row` 对象，如下运行时演示所示：

```py
>>> for row in result:
...     print(f"{row.name}  {row.email_address}")
spongebob  spongebob@sqlalchemy.org
sandy  sandy@sqlalchemy.org
sandy  squirrel@squirrelpower.org
patrick  pat999@aol.com
squidward  stentcl@sqlalchemy.org
```

### 使用 Bundle 分组选择的属性

`Bundle` 构造是一个可扩展的仅限 ORM 的构造，允许将列表达式集合分组在结果行中：

```py
>>> from sqlalchemy.orm import Bundle
>>> stmt = select(
...     Bundle("user", User.name, User.fullname),
...     Bundle("email", Address.email_address),
... ).join_from(User, Address)
>>> for row in session.execute(stmt):
...     print(f"{row.user.name} {row.user.fullname} {row.email.email_address}")
SELECT  user_account.name,  user_account.fullname,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
[...]  ()
spongebob Spongebob Squarepants spongebob@sqlalchemy.org
sandy Sandy Cheeks sandy@sqlalchemy.org
sandy Sandy Cheeks squirrel@squirrelpower.org
patrick Patrick Star pat999@aol.com
squidward Squidward Tentacles stentcl@sqlalchemy.org
```

`Bundle` 可能对创建轻量级视图和自定义列分组很有用。`Bundle` 也可以被子类化以返回替代数据结构；请参见 `Bundle.create_row_processor()` 以获取示例。

另请参阅

`Bundle`

`Bundle.create_row_processor()`

### 选择 ORM 别名

如使用别名教程中所述，创建 ORM 实体的 SQL 别名是通过对映射类使用 `aliased()` 构造完成的：

```py
>>> from sqlalchemy.orm import aliased
>>> u1 = aliased(User)
>>> print(select(u1).order_by(u1.id))
SELECT  user_account_1.id,  user_account_1.name,  user_account_1.fullname
FROM  user_account  AS  user_account_1  ORDER  BY  user_account_1.id 
```

就像使用`Table.alias()`时一样，SQL 别名是匿名命名的。对于从具有显式名称的行中选择实体的情况，也可以传递`aliased.name`参数：

```py
>>> from sqlalchemy.orm import aliased
>>> u1 = aliased(User, name="u1")
>>> stmt = select(u1).order_by(u1.id)
>>> row = session.execute(stmt).first()
SELECT  u1.id,  u1.name,  u1.fullname
FROM  user_account  AS  u1  ORDER  BY  u1.id
[...]  ()
>>> print(f"{row.u1.name}")
spongebob
```

另请参阅

`aliased`结构对于多种用例至关重要，包括：

+   利用 ORM 的子查询；章节从子查询中选择实体和与子查询连接进一步讨论了这一点。

+   控制结果集中实体的名称；参见同时选择多个 ORM 实体以获取示例

+   多次连接到同一 ORM 实体；参见使用关系在别名目标之间连接以获取示例。

### 从文本语句中获取 ORM 结果

对象关系映射（ORM）支持从其他来源的 SELECT 语句加载实体。典型用例是文本 SELECT 语句，在 SQLAlchemy 中使用`text()`结构表示。`text()`结构可以附加有关语句将加载的 ORM 映射列的信息；然后可以将其与 ORM 实体本身关联，以便基于此语句加载 ORM 对象。

给定要加载的文本 SQL 语句：

```py
>>> from sqlalchemy import text
>>> textual_sql = text("SELECT id, name, fullname FROM user_account ORDER BY id")
```

我们可以使用`TextClause.columns()`方法向语句添加列信息；当调用此方法时，`TextClause`对象将转换为`TextualSelect`对象，其承担的角色可与`Select`构造类似。`TextClause.columns()`方法通常传递`Column`对象或等效对象，在这种情况下，我们可以直接使用`User`类上的 ORM 映射属性：

```py
>>> textual_sql = textual_sql.columns(User.id, User.name, User.fullname)
```

现在，我们有一个经过 ORM 配置的 SQL 构造，可以分别加载“id”、“name”和“fullname”列。要将此 SELECT 语句用作完整`User`实体的来源，我们可以使用`Select.from_statement()`方法将这些列链接到常规的 ORM 启用的`Select`构造中：

```py
>>> orm_sql = select(User).from_statement(textual_sql)
>>> for user_obj in session.execute(orm_sql).scalars():
...     print(user_obj)
SELECT  id,  name,  fullname  FROM  user_account  ORDER  BY  id
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

相同的`TextualSelect`对象也可以使用`TextualSelect.subquery()`方法转换为子查询，并使用`aliased()`构造将其链接到`User`实体，方式与下面讨论的从子查询中选择实体类似：

```py
>>> orm_subquery = aliased(User, textual_sql.subquery())
>>> stmt = select(orm_subquery)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  id,  name,  fullname  FROM  user_account  ORDER  BY  id)  AS  anon_1
[...]  ()
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

直接使用`TextualSelect`与`Select.from_statement()`相比，使用`aliased()`的区别在于，在前一种情况下，生成的 SQL 中不会产生子查询。在某些情况下，这可能有利于性能或复杂性方面。

### 从子查询中选择实体

在前一节讨论的`aliased()`构造中，可以与任何`Subuqery`构造一起使用，该构造来自诸如`Select.subquery()`之类的方法，以将 ORM 实体链接到该子查询返回的列；子查询返回的列与实体映射的列之间必须存在**列对应**关系，这意味着子查询最终需要源自这些实体，例如下面的示例：

```py
>>> inner_stmt = select(User).where(User.id < 7).order_by(User.id)
>>> subq = inner_stmt.subquery()
>>> aliased_user = aliased(User, subq)
>>> stmt = select(aliased_user)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
  SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  <  ?  ORDER  BY  user_account.id)  AS  anon_1
[generated  in  ...]  (7,)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
User(id=3, name='patrick', fullname='Patrick Star')
User(id=4, name='squidward', fullname='Squidward Tentacles')
User(id=5, name='ehkrabs', fullname='Eugene H. Krabs')
```

另请参见

ORM 实体子查询/CTEs - 在 SQLAlchemy 统一教程中

加入子查询

### 从 UNION 和其他集合操作中选择实体

`union()` 和 `union_all()` 函数是最常见的集合操作，与其他集合操作（如 `except_()`、`intersect()` 等）一起，提供了一个称为 `CompoundSelect` 的对象，它由多个通过集合操作关键字连接的 `Select` 构造组成。ORM 实体可以通过简单的复合选择使用 `Select.from_statement()` 方法进行选择，该方法在 从文本语句中获取 ORM 结果 中已经说明。在这种方法中，UNION 语句是将被渲染的完整语句，不能在使用 `Select.from_statement()` 后添加额外的条件：

```py
>>> from sqlalchemy import union_all
>>> u = union_all(
...     select(User).where(User.id < 2), select(User).where(User.id == 3)
... ).order_by(User.id)
>>> stmt = select(User).from_statement(u)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.id  <  ?  UNION  ALL  SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.id  =  ?  ORDER  BY  id
[generated  in  ...]  (2,  3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
```

`CompoundSelect` 构造可以在更灵活的查询中更灵活地使用，该查询可以通过将其组织成子查询并使用 `aliased()` 将其链接到 ORM 实体来进一步修改，如 从子查询中选择实体 中已说明。在下面的示例中，我们首先使用 `CompoundSelect.subquery()` 创建 UNION ALL 语句的子查询，然后将其打包到 `aliased()` 构造中，在这里它可以像其他映射实体一样用于 `select()` 构造中，包括我们可以根据其导出的列添加过滤和排序条件：

```py
>>> subq = union_all(
...     select(User).where(User.id < 2), select(User).where(User.id == 3)
... ).subquery()
>>> user_alias = aliased(User, subq)
>>> stmt = select(user_alias).order_by(user_alias.id)
>>> for user_obj in session.execute(stmt).scalars():
...     print(user_obj)
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  <  ?  UNION  ALL  SELECT  user_account.id  AS  id,  user_account.name  AS  name,  user_account.fullname  AS  fullname
FROM  user_account
WHERE  user_account.id  =  ?)  AS  anon_1  ORDER  BY  anon_1.id
[generated  in  ...]  (2,  3)
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=3, name='patrick', fullname='Patrick Star')
```

另请参阅

从 Union 中选择 ORM 实体 - 在 SQLAlchemy 统一教程 中

## 连接

`Select.join()`和`Select.join_from()`方法用于构建针对 SELECT 语句的 SQL JOINs。

本节将详细介绍这些方法在 ORM 中的用例。有关从 Core 视角的使用的一般概述，请参阅显式 FROM 子句和 JOINs 中的 SQLAlchemy 统一教程。

在 ORM 上下文中使用`Select.join()`进行 2.0 风格查询的用法基本上等同于除了遗留用例之外，在 1.x 风格查询中使用`Query.join()`方法的用法。

### 简单的关系连接

考虑两个类`User`和`Address`之间的映射，其中关系`User.addresses`表示与每个`User`关联的`Address`对象的集合。`Select.join()`最常见的用法是沿着这种关系创建一个 JOIN，使用`User.addresses`属性作为指示器来指示应该如何发生这种情况：

```py
>>> stmt = select(User).join(User.addresses)
```

在上面的例子中，对`Select.join()`和`User.addresses`的调用将导致大致等效的 SQL 语句：

```py
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

在上面的例子中，我们将`User.addresses`称为传递给`Select.join()`的“on clause”，即它指示如何构建 JOIN 的“ON”部分。

提示

请注意，使用`Select.join()`从一个实体连接到另一个实体会影响 SELECT 语句的 FROM 子句，但不会影响列子句；在这个示例中，SELECT 语句将继续仅返回`User`实体的行。要同时从`User`和`Address`中选择列/实体，必须在`select()`函数中也命名`Address`实体，或者在使用`Select.add_columns()`方法后将其添加到`Select`构造中。有关这两种形式的示例，请参阅同时选择多个 ORM 实体部分。

### 链式多重连接

要构建连接链，可以使用多个`Select.join()`调用。关联属性同时涵盖连接的左侧和右侧。考虑额外的实体`Order`和`Item`，其中`User.orders`关系指向`Order`实体，而`Order.items`关系指向`Item`实体，通过一个关联表`order_items`。两个`Select.join()`调用将导致第一个 JOIN 从`User`到`Order`，第二个从`Order`到`Item`。然而，由于`Order.items`是多对多关系，它会导致两个独立的 JOIN 元素，总共有三个 JOIN 元素在结果 SQL 中：

```py
>>> stmt = select(User).join(User.orders).join(Order.items)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  user_order  ON  user_account.id  =  user_order.user_id
JOIN  order_items  AS  order_items_1  ON  user_order.id  =  order_items_1.order_id
JOIN  item  ON  item.id  =  order_items_1.item_id 
```

每次调用`Select.join()`方法的顺序只有在我们想要从中连接的“左”侧需要出现在 FROM 列表中时才重要，然后我们才能指示一个新的目标。例如，如果我们指定`select(User).join(Order.items).join(User.orders)`，`Select.join()`就不会知道如何正确地进行连接，它会引发错误。在正确的实践中，应该以与我们希望在 SQL 中呈现 JOIN 子句相匹配的方式调用`Select.join()`方法，并且每次调用都应该表示从前面的内容清晰链接。

我们在 FROM 子句中定位的所有元素仍然可用作继续连接 FROM 的潜在点。例如，我们可以继续将其他元素添加到上述`User`实体的 FROM 连接中，例如在我们的连接链中添加`User.addresses`关系：

```py
>>> stmt = select(User).join(User.orders).join(Order.items).join(User.addresses)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  user_order  ON  user_account.id  =  user_order.user_id
JOIN  order_items  AS  order_items_1  ON  user_order.id  =  order_items_1.order_id
JOIN  item  ON  item.id  =  order_items_1.item_id
JOIN  address  ON  user_account.id  =  address.user_id 
```

### 连接到目标实体

第二种形式的`Select.join()`允许将任何映射实体或核心可选择的构造作为目标。在此用法中，`Select.join()` 将尝试**推断**JOIN 的 ON 子句，使用两个实体之间的自然外键关系：

```py
>>> stmt = select(User).join(Address)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

在上述调用形式中，`Select.join()` 被调用以自动推断“on clause”。如果两个映射的`Table`构造之间没有设置任何`ForeignKeyConstraint`，或者如果它们之间有多个`ForeignKeyConstraint`链接，使得要使用的适当约束不明确，此调用形式最终将引发错误。

注意

在使用`Select.join()`或`Select.join_from()`而不指定 ON 子句时，ORM 配置的`relationship()`构造**不会被考虑**。仅在尝试为 JOIN 推断 ON 子句时，才会在映射的`Table`对象级别上查阅配置的`ForeignKeyConstraint`关系。

### 到具有 ON 子句的目标的连接

第三种调用形式允许显式传递目标实体以及 ON 子句。包含 SQL 表达式作为 ON 子句的示例如下：

```py
>>> stmt = select(User).join(Address, User.id == Address.user_id)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

基于表达式的 ON 子句也可以是一个`relationship()`绑定属性，就像在简单关系连接中使用的方式一样：

```py
>>> stmt = select(User).join(Address, User.addresses)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

上述示例似乎多余，因为它以两种不同的方式指示了`Address`的目标；然而，当加入到别名实体时，这种形式的实用性变得明显；请参见使用关系连接别名目标中的示例。### 将关系与自定义 ON 条件结合使用

由`relationship()`构造生成的 ON 子句可以通过附加的额外条件进行增强。这对于快速限制特定关系路径上连接范围的方式以及配置加载策略（如`joinedload()`和`selectinload()`）非常有用。`PropComparator.and_()`方法按位置接受一系列 SQL 表达式，这些表达式将通过 AND 连接到 JOIN 的 ON 子句。例如，如果我们想要从`User` JOIN 到`Address`，但也限制 ON 条件仅适用于某些电子邮件地址：

```py
>>> stmt = select(User.fullname).join(
...     User.addresses.and_(Address.email_address == "squirrel@squirrelpower.org")
... )
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
JOIN  address  ON  user_account.id  =  address.user_id  AND  address.email_address  =  ?
[...]  ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
```

另请参阅

`PropComparator.and_()`方法也适用于加载策略，如`joinedload()`和`selectinload()`。请参见向加载选项添加条件部分。### 使用关系连接别名目标

当使用`relationship()`绑定属性构建连接以指示 ON 子句时，使用带有 ON 子句的目标的连接中说明的两参数语法可以扩展为与`aliased()`构造一起使用，以指示 SQL 别名作为连接的目标，同时仍然利用`relationship()`绑定属性来指示 ON 子句，如下例所示，其中`User`实体两次与两个不同的`aliased()`构造连接到`Address`实体：

```py
>>> address_alias_1 = aliased(Address)
>>> address_alias_2 = aliased(Address)
>>> stmt = (
...     select(User)
...     .join(address_alias_1, User.addresses)
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join(address_alias_2, User.addresses)
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

同样的模式可以更简洁地使用修饰符 `PropComparator.of_type()` 表达，该修饰符可以应用于 `relationship()` 绑定的属性，传递目标实体以一步指示目标。下面的示例使用 `PropComparator.of_type()` 来生成与刚刚示例相同的 SQL 语句：

```py
>>> print(
...     select(User)
...     .join(User.addresses.of_type(address_alias_1))
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join(User.addresses.of_type(address_alias_2))
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

要利用一个 `relationship()` 来构建一个从别名实体连接的连接，该属性直接从 `aliased()` 构造中可用：

```py
>>> user_alias_1 = aliased(User)
>>> print(select(user_alias_1.name).join(user_alias_1.addresses))
SELECT  user_account_1.name
FROM  user_account  AS  user_account_1
JOIN  address  ON  user_account_1.id  =  address.user_id 
```  ### 连接到子查询

连接的目标可以是任何可选择的实体，包括子查询。在使用 ORM 时，通常会以 `aliased()` 构造来表示这些目标，但这并不是严格要求的，特别是如果连接的实体不会在结果中返回时。例如，要从 `User` 实体连接到 `Address` 实体，在这里 `Address` 实体被表示为一行限制的子查询，我们首先使用 `Select.subquery()` 构造一个 `Subquery` 对象，然后可以将其用作 `Select.join()` 方法的目标：

```py
>>> subq = select(Address).where(Address.email_address == "pat999@aol.com").subquery()
>>> stmt = select(User).join(subq, User.id == subq.c.user_id)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  :email_address_1)  AS  anon_1
ON  user_account.id  =  anon_1.user_id 
```

上述的 SELECT 语句在通过 `Session.execute()` 调用时将返回包含 `User` 实体的行，但不包含 `Address` 实体。为了将 `Address` 实体包含到将在结果集中返回的实体集合中，我们构造了一个针对 `Address` 实体和 `Subquery` 对象的 `aliased()` 对象。我们还可以希望对 `aliased()` 构造应用一个名称，例如下面使用的 `"address"`，这样我们就可以在结果行中通过名称引用它：

```py
>>> address_subq = aliased(Address, subq, name="address")
>>> stmt = select(User, address_subq).join(address_subq)
>>> for row in session.execute(stmt):
...     print(f"{row.User} {row.address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.user_id,  anon_1.email_address
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
[...]  ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
```

### 沿关系路径连接到子查询

在前一节中示例的子查询形式可以使用更具体的方式来表达，使用一个`relationship()`绑定的属性，使用使用关系在别名目标之间进行连接中指示的形式之一。例如，要创建相同的连接，并确保连接沿着特定`relationship()`进行，我们可以使用`PropComparator.of_type()`方法，传递包含要连接的`Subquery`对象的`aliased()`构造：

```py
>>> address_subq = aliased(Address, subq, name="address")
>>> stmt = select(User, address_subq).join(User.addresses.of_type(address_subq))
>>> for row in session.execute(stmt):
...     print(f"{row.User} {row.address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.user_id,  anon_1.email_address
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
[...]  ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
```

### 引用多个实体的子查询

包含跨越多个 ORM 实体的列的子查询可以同时应用于多个`aliased()`构造，并在相同的`Select`构造中按照每个实体分别处理。然而，生成的 SQL 仍将所有这些`aliased()`构造视为相同的子查询，但是从 ORM / Python 的角度来看，可以使用适当的`aliased()`构造来引用不同的返回值和对象属性。

例如，给定一个同时引用`User`和`Address`的子查询：

```py
>>> user_address_subq = (
...     select(User.id, User.name, User.fullname, Address.id, Address.email_address)
...     .join_from(User, Address)
...     .where(Address.email_address.in_(["pat999@aol.com", "squirrel@squirrelpower.org"]))
...     .subquery()
... )
```

我们可以针对`User`和`Address`分别创建`aliased()`构造，它们各自指向相同的对象：

```py
>>> user_alias = aliased(User, user_address_subq, name="user")
>>> address_alias = aliased(Address, user_address_subq, name="address")
```

从两个实体中进行选择的`Select`构造将会渲染子查询一次，但在结果行上下文中可以同时返回`User`和`Address`类的对象：

```py
>>> stmt = select(user_alias, address_alias).where(user_alias.name == "sandy")
>>> for row in session.execute(stmt):
...     print(f"{row.user} {row.address}")
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname,  anon_1.id_1,  anon_1.email_address
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,
user_account.fullname  AS  fullname,  address.id  AS  id_1,
address.email_address  AS  email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  address.email_address  IN  (?,  ?))  AS  anon_1
WHERE  anon_1.name  =  ?
[...]  ('pat999@aol.com',  'squirrel@squirrelpower.org',  'sandy')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='squirrel@squirrelpower.org')
```

### 设置连接中最左侧的 FROM 子句

在当前`Select`状态的左侧与我们想要连接的内容不符合的情况下，可以使用`Select.join_from()`方法：

```py
>>> stmt = select(Address).join_from(User, User.addresses).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

`Select.join_from()`方法接受两个或三个参数，可以是`(<join from>, <onclause>)`形式，也可以是`(<join from>, <join to>, [<onclause>])`形式：

```py
>>> stmt = select(Address).join_from(User, Address).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

为了设置初始的 FROM 子句，以便之后可以使用 `Select.join()`，可以使用 `Select.select_from()` 方法：

```py
>>> stmt = select(Address).select_from(User).join(Address).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

提示

`Select.select_from()` 方法实际上并不决定 FROM 子句中表的顺序。如果语句还引用了指向不同顺序的现有表的 `Join` 结构，那么 `Join` 结构将优先。当我们使用 `Select.join()` 和 `Select.join_from()` 等方法时，这些方法最终创建了这样一个 `Join` 对象。因此，在这种情况下，我们可以看到 `Select.select_from()` 的内容被覆盖了：

```py
>>> stmt = select(Address).select_from(User).join(Address.user).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

在上面的例子中，我们看到 FROM 子句是 `address JOIN user_account`，尽管我们首先声明了 `select_from(User)`。由于 `.join(Address.user)` 方法调用，该语句最终等同于以下内容：

```py
>>> from sqlalchemy.sql import join
>>>
>>> user_table = User.__table__
>>> address_table = Address.__table__
>>>
>>> j = address_table.join(user_table, user_table.c.id == address_table.c.user_id)
>>> stmt = (
...     select(address_table)
...     .select_from(user_table)
...     .select_from(j)
...     .where(user_table.c.name == "sandy")
... )
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

上述的 `Join` 结构被添加为 `Select.select_from()` 列表中的另一个条目，它取代了先前的条目。 ### 简单的关系连接

考虑两个类 `User` 和 `Address` 之间的映射，其中关系 `User.addresses` 表示与每个 `User` 关联的 `Address` 对象的集合。`Select.join()` 的最常见用法是沿着这种关系创建 JOIN，使用 `User.addresses` 属性作为指示器指示应该如何进行连接：

```py
>>> stmt = select(User).join(User.addresses)
```

在上面的例子中，对 `User.addresses` 使用 `Select.join()` 的调用将导致大致等效于以下 SQL：

```py
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

在上面的示例中，我们将`User.addresses`称为传递给`Select.join()`的“on clause”，即指示如何构建 JOIN 的“ON”部分。

提示

请注意，使用`Select.join()`从一个实体 JOIN 到另一个实体会影响 SELECT 语句的 FROM 子句，但不影响列子句；在这个示例中，SELECT 语句仍将只返回来自`User`实体的行。要同时从`User`和`Address`中 SELECT 列/实体，必须在`select()`函数中也命名`Address`实体，或者使用`Select.add_columns()`方法在之后将其添加到`Select`构造中。有关这两种形式的示例，请参见同时选择多个 ORM 实体部分。

### 链接多个表

要构建一系列 JOIN，可以使用多个`Select.join()`调用。关系绑定属性同时暗示 JOIN 的左侧和右侧。考虑额外的实体`Order`和`Item`，其中`User.orders`关系指向`Order`实体，而`Order.items`关系通过关联表`order_items`指向`Item`实体。两个`Select.join()`调用将导致从`User`到`Order`的第一个 JOIN，以及从`Order`到`Item`的第二个 JOIN。然而，由于`Order.items`是多对多关系，它会导致两个独立的 JOIN 元素，总共在生成的 SQL 中有三个 JOIN 元素：

```py
>>> stmt = select(User).join(User.orders).join(Order.items)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  user_order  ON  user_account.id  =  user_order.user_id
JOIN  order_items  AS  order_items_1  ON  user_order.id  =  order_items_1.order_id
JOIN  item  ON  item.id  =  order_items_1.item_id 
```

每次调用`Select.join()`方法的顺序只有在我们希望连接的“左”侧需要在 FROM 列表中存在之前才会产生影响。例如，如果我们指定了`select(User).join(Order.items).join(User.orders)`，那么`Select.join()`将无法正确连接，并且会引发错误。在正确的实践中，应以类似于 SQL 中 JOIN 子句应该呈现的方式调用`Select.join()`方法，并且每次调用应该代表与其前面的内容之间的清晰链接。

我们在 FROM 子句中定位的所有元素仍然可以作为继续连接 FROM 的潜在点。例如，我们可以在上面的`User`实体上继续添加其他元素以连接 FROM，例如在我们的连接链中添加`User.addresses`关系：

```py
>>> stmt = select(User).join(User.orders).join(Order.items).join(User.addresses)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  user_order  ON  user_account.id  =  user_order.user_id
JOIN  order_items  AS  order_items_1  ON  user_order.id  =  order_items_1.order_id
JOIN  item  ON  item.id  =  order_items_1.item_id
JOIN  address  ON  user_account.id  =  address.user_id 
```

### 连接到目标实体

`Select.join()`的第二种形式允许任何映射实体或核心可选择的构造作为目标。在这种用法中，`Select.join()`将尝试**推断**连接的 ON 子句，使用两个实体之间的自然外键关系：

```py
>>> stmt = select(User).join(Address)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

在上述调用形式中，`Select.join()`被调用来自动推断“on 子句”。如果两个映射的`Table`构造之间没有设置任何`ForeignKeyConstraint`，或者如果它们之间存在多个`ForeignKeyConstraint`链接，使得要使用的适当约束不明确，这种调用形式最终将引发错误。

注意

当使用 `Select.join()` 或 `Select.join_from()` 而没有指定 ON 子句时，ORM 配置的 `relationship()` 构建**不会考虑**。只有在尝试推断 JOIN 的 ON 子句时，才会查询映射的 `Table` 对象级别的实体之间配置的 `ForeignKeyConstraint` 关系。

### 加入带有 ON 子句的目标

第三种调用形式允许同时显式传递目标实体和 ON 子句。一个包含 SQL 表达式作为 ON 子句的示例如下：

```py
>>> stmt = select(User).join(Address, User.id == Address.user_id)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

表达式基于的 ON 子句也可以是 `relationship()` 绑定的属性，就像在 简单 Relationship 加入 中使用的方式一样：

```py
>>> stmt = select(User).join(Address, User.addresses)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

上述示例似乎冗余，因为它以两种不同的方式指示了 `Address` 的目标；然而，当加入别名实体时，这种形式的实用性就变得明显了；请参见 使用 Relationship 在别名目标之间加入 部分的示例。

### 将 Relationship 与自定义 ON 条件相结合

`relationship()` 构建生成的 ON 子句可以通过附加的条件进行增强。这对于快速限制特定关系路径上连接的范围的方法以及配置加载器策略（例如 `joinedload()` 和 `selectinload()`）等情况都很有用。`PropComparator.and_()` 方法接受一系列 SQL 表达式，这些表达式将通过 AND 连接到 JOIN 的 ON 子句中。例如，如果我们想要从 `User` 加入到 `Address`，但也只限制 ON 条件到某些电子邮件地址：

```py
>>> stmt = select(User.fullname).join(
...     User.addresses.and_(Address.email_address == "squirrel@squirrelpower.org")
... )
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
JOIN  address  ON  user_account.id  =  address.user_id  AND  address.email_address  =  ?
[...]  ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
```

另请参见

`PropComparator.and_()`方法也适用于加载策略，如`joinedload()`和`selectinload()`。参见向加载选项添加条件一节。

### 使用关系连接别名目标

当使用`relationship()`绑定的属性来指示 ON 子句构建连接时，可以将具有 ON 子句的目标的连接中示例的二参数语法扩展到与`aliased()`构造一起工作，以指示 SQL 别名作为连接的目标，同时仍然利用`relationship()`绑定的属性来指示 ON 子句，如下例所示，其中`User`实体两次与两个不同的`aliased()`构造连接到`Address`实体：

```py
>>> address_alias_1 = aliased(Address)
>>> address_alias_2 = aliased(Address)
>>> stmt = (
...     select(User)
...     .join(address_alias_1, User.addresses)
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join(address_alias_2, User.addresses)
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

使用修饰符`PropComparator.of_type()`可以更简洁地表达相同的模式，该修饰符可以应用于`relationship()`绑定的属性，通过传递目标实体以一步指示目标。下面的示例使用`PropComparator.of_type()`来生成与刚刚示例相同的 SQL 语句：

```py
>>> print(
...     select(User)
...     .join(User.addresses.of_type(address_alias_1))
...     .where(address_alias_1.email_address == "patrick@aol.com")
...     .join(User.addresses.of_type(address_alias_2))
...     .where(address_alias_2.email_address == "patrick@gmail.com")
... )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  address  AS  address_1  ON  user_account.id  =  address_1.user_id
JOIN  address  AS  address_2  ON  user_account.id  =  address_2.user_id
WHERE  address_1.email_address  =  :email_address_1
AND  address_2.email_address  =  :email_address_2 
```

要利用`relationship()`构建从别名实体的连接，可以直接从`aliased()`构造中使用属性：

```py
>>> user_alias_1 = aliased(User)
>>> print(select(user_alias_1.name).join(user_alias_1.addresses))
SELECT  user_account_1.name
FROM  user_account  AS  user_account_1
JOIN  address  ON  user_account_1.id  =  address.user_id 
```

### 连接到子查询

加入的目标可以是任何“可选择”的实体，包括子查询。在使用 ORM 时，通常会使用 `aliased()` 构造来表示这些目标，但这不是严格要求的，特别是如果加入的实体不会在结果中返回的情况下。例如，要从 `User` 实体加入到 `Address` 实体，其中 `Address` 实体被表示为一行限制的子查询，我们首先使用 `Select.subquery()` 构造一个 `Subquery` 对象，然后可以将其用作 `Select.join()` 方法的目标：

```py
>>> subq = select(Address).where(Address.email_address == "pat999@aol.com").subquery()
>>> stmt = select(User).join(subq, User.id == subq.c.user_id)
>>> print(stmt)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  :email_address_1)  AS  anon_1
ON  user_account.id  =  anon_1.user_id 
```

上述 SELECT 语句在通过 `Session.execute()` 调用时将返回包含 `User` 实体但不包含 `Address` 实体的行。为了将 `Address` 实体包含到将在结果集中返回的实体集中，我们针对 `Address` 实体和 `Subquery` 对象构造了一个 `aliased()` 对象。我们可能还希望对 `aliased()` 构造应用一个名称，例如下面使用的 `"address"`，以便我们可以通过名称在结果行中引用它：

```py
>>> address_subq = aliased(Address, subq, name="address")
>>> stmt = select(User, address_subq).join(address_subq)
>>> for row in session.execute(stmt):
...     print(f"{row.User} {row.address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.user_id,  anon_1.email_address
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
[...]  ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
```

### 沿着关系路径加入子查询

前面部分中示例的子查询形式可以使用一个 `relationship()` 绑定属性更具体地表示，使用 使用 Relationship 在别名目标之间加入 中指示的其中一种形式。例如，为了创建相同的加入并确保加入是沿着特定 `relationship()` 进行的，我们可以使用 `PropComparator.of_type()` 方法，传递包含加入目标的 `aliased()` 构造的 `Subquery` 对象：

```py
>>> address_subq = aliased(Address, subq, name="address")
>>> stmt = select(User, address_subq).join(User.addresses.of_type(address_subq))
>>> for row in session.execute(stmt):
...     print(f"{row.User} {row.address}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
anon_1.id  AS  id_1,  anon_1.user_id,  anon_1.email_address
FROM  user_account
JOIN  (SELECT  address.id  AS  id,
address.user_id  AS  user_id,  address.email_address  AS  email_address
FROM  address
WHERE  address.email_address  =  ?)  AS  anon_1  ON  user_account.id  =  anon_1.user_id
[...]  ('pat999@aol.com',)
User(id=3, name='patrick', fullname='Patrick Star') Address(id=4, email_address='pat999@aol.com')
```

### 引用多个实体的子查询

包含跨越多个 ORM 实体的列的子查询可以同时应用于多个`aliased()`构造，并且在每个实体的情况下都可以在相同的`Select`构造中使用。生成的 SQL 将继续将所有这样的`aliased()`构造视为相同的子查询，但是从 ORM / Python 的角度来看，可以通过使用适当的`aliased()`构造来引用不同的返回值和对象属性。

给定一个同时引用`User`和`Address`的子查询，例如：

```py
>>> user_address_subq = (
...     select(User.id, User.name, User.fullname, Address.id, Address.email_address)
...     .join_from(User, Address)
...     .where(Address.email_address.in_(["pat999@aol.com", "squirrel@squirrelpower.org"]))
...     .subquery()
... )
```

我们可以创建针对`User`和`Address`的`aliased()`构造，它们各自都引用相同的对象：

```py
>>> user_alias = aliased(User, user_address_subq, name="user")
>>> address_alias = aliased(Address, user_address_subq, name="address")
```

从两个实体中选择的`Select`构造将只渲染子查询一次，但在结果行上下文中可以同时返回`User`和`Address`类的对象：

```py
>>> stmt = select(user_alias, address_alias).where(user_alias.name == "sandy")
>>> for row in session.execute(stmt):
...     print(f"{row.user} {row.address}")
SELECT  anon_1.id,  anon_1.name,  anon_1.fullname,  anon_1.id_1,  anon_1.email_address
FROM  (SELECT  user_account.id  AS  id,  user_account.name  AS  name,
user_account.fullname  AS  fullname,  address.id  AS  id_1,
address.email_address  AS  email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  address.email_address  IN  (?,  ?))  AS  anon_1
WHERE  anon_1.name  =  ?
[...]  ('pat999@aol.com',  'squirrel@squirrelpower.org',  'sandy')
User(id=2, name='sandy', fullname='Sandy Cheeks') Address(id=3, email_address='squirrel@squirrelpower.org')
```

### 设置连接中最左边的 FROM 子句

在当前`Select`状态的左侧与我们要连接的内容不一致的情况下，可以使用`Select.join_from()`方法：

```py
>>> stmt = select(Address).join_from(User, User.addresses).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

`Select.join_from()`方法接受两个或三个参数，形式可以是`(<join from>, <onclause>)`，或者是`(<join from>, <join to>, [<onclause>])`：

```py
>>> stmt = select(Address).join_from(User, Address).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

为了为 SELECT 设置初始 FROM 子句，以便之后可以使用`Select.join()`，也可以使用`Select.select_from()`方法：

```py
>>> stmt = select(Address).select_from(User).join(Address).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

提示

`Select.select_from()`方法实际上并没有最终决定 FROM 子句中表的顺序。如果语句还引用了一个`Join`构造，该构造引用了不同顺序的现有表，则`Join`构造优先。当我们使用像`Select.join()`和`Select.join_from()`这样的方法时，这些方法最终会创建这样一个`Join`对象。因此，在这种情况下，我们可以看到`Select.select_from()`的内容被覆盖：

```py
>>> stmt = select(Address).select_from(User).join(Address.user).where(User.name == "sandy")
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

在上述例子中，我们看到 FROM 子句是`address JOIN user_account`，尽管我们首先声明了`select_from(User)`。由于`.join(Address.user)`方法调用，语句最终等效于以下内容：

```py
>>> from sqlalchemy.sql import join
>>>
>>> user_table = User.__table__
>>> address_table = Address.__table__
>>>
>>> j = address_table.join(user_table, user_table.c.id == address_table.c.user_id)
>>> stmt = (
...     select(address_table)
...     .select_from(user_table)
...     .select_from(j)
...     .where(user_table.c.name == "sandy")
... )
>>> print(stmt)
SELECT  address.id,  address.user_id,  address.email_address
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  :name_1 
```

上述`Join`构造是作为`Select.select_from()`列表中的另一个条目添加的，它取代了先前的条目。

## 关系 WHERE 运算符

除了在`Select.join()`和`Select.join_from()`方法中使用`relationship()`构造之外，`relationship()`还在帮助构造通常用于 WHERE 子句的 SQL 表达式，使用`Select.where()`方法。

### EXISTS 形式：has() / any()

`Exists`构造首次在 SQLAlchemy 统一教程的 EXISTS 子查询部分中介绍。该对象用于在标量子查询与 SQL EXISTS 关键字一起呈现。`relationship()`构造提供了一些辅助方法，可以用于以关系的方式生成一些常见的 EXISTS 风格的查询。

对于像`User.addresses`这样的一对多关系，可以使用`PropComparator.any()`针对与`user_account`表相关联的`address`表进行 EXISTS 查询。此方法接受一个可选的 WHERE 条件来限制子查询匹配的行：

```py
>>> stmt = select(User.fullname).where(
...     User.addresses.any(Address.email_address == "squirrel@squirrelpower.org")
... )
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
WHERE  EXISTS  (SELECT  1
FROM  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  ?)
[...]  ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
```

由于 EXISTS 倾向于更有效地进行负查找，一个常见的查询是定位没有相关实体的实体。这可以简洁地使用短语`~User.addresses.any()`来实现，以选择没有相关`Address`行的`User`实体：

```py
>>> stmt = select(User.fullname).where(~User.addresses.any())
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
WHERE  NOT  (EXISTS  (SELECT  1
FROM  address
WHERE  user_account.id  =  address.user_id))
[...]  ()
[('Eugene H. Krabs',)]
```

`PropComparator.has()`方法的工作方式与`PropComparator.any()`基本相同，只是它用于一对多关系，例如，如果我们想要定位所有属于“sandy”的`Address`对象：

```py
>>> stmt = select(Address.email_address).where(Address.user.has(User.name == "sandy"))
>>> session.execute(stmt).all()
SELECT  address.email_address
FROM  address
WHERE  EXISTS  (SELECT  1
FROM  user_account
WHERE  user_account.id  =  address.user_id  AND  user_account.name  =  ?)
[...]  ('sandy',)
[('sandy@sqlalchemy.org',), ('squirrel@squirrelpower.org',)]
```  ### 关系实例比较运算符

`relationship()`-绑定的属性还提供了一些 SQL 构造实现，这些实现旨在根据相关对象的特定实例来过滤`relationship()`-绑定的属性，它可以从给定的持久（或较少见的分离）对象实例中解包适当的属性值，并构造 WHERE 条件，以便针对目标`relationship()`。

+   **一对多等于比较** - 可以将特定对象实例与一对多关系进行比较，以选择外键与给定对象的主键值匹配的行：

    ```py
    >>> user_obj = session.get(User, 1)
    SELECT  ...
    >>> print(select(Address).where(Address.user == user_obj))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  :param_1  =  address.user_id 
    ```

+   **一对多不等于比较** - 也可以使用不等于运算符：

    ```py
    >>> print(select(Address).where(Address.user != user_obj))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  address.user_id  !=  :user_id_1  OR  address.user_id  IS  NULL 
    ```

+   **对象包含在一对多集合中** - 这基本上是“等于”比较的一对多版本，选择主键等于相关对象中的外键值的行：

    ```py
    >>> address_obj = session.get(Address, 1)
    SELECT  ...
    >>> print(select(User).where(User.addresses.contains(address_obj)))
    SELECT  user_account.id,  user_account.name,  user_account.fullname
    FROM  user_account
    WHERE  user_account.id  =  :param_1 
    ```

+   **从一对多的角度来看，对象有一个特定的父对象** - `with_parent()`函数生成一个比较，返回被给定父对象引用的行，这本质上与使用一对多方的`==`运算符相同：

    ```py
    >>> from sqlalchemy.orm import with_parent
    >>> print(select(Address).where(with_parent(user_obj, User.addresses)))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  :param_1  =  address.user_id 
    ```  ### EXISTS forms: has() / any()

`Exists`构造首次在 SQLAlchemy 统一教程中的 EXISTS 子查询一节中引入。该对象用于在标量子查询与 SQL EXISTS 关键字一起渲染。`relationship()`构造提供了一些辅助方法，可以用于根据关系生成一些常见的 EXISTS 风格的查询。

对于像`User.addresses`这样的一对多关系，可以使用与`user_account`表关联的`address`表的 EXISTS 来生成 `PropComparator.any()`。此方法接受一个可选的 WHERE 条件来限制子查询匹配的行：

```py
>>> stmt = select(User.fullname).where(
...     User.addresses.any(Address.email_address == "squirrel@squirrelpower.org")
... )
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
WHERE  EXISTS  (SELECT  1
FROM  address
WHERE  user_account.id  =  address.user_id  AND  address.email_address  =  ?)
[...]  ('squirrel@squirrelpower.org',)
[('Sandy Cheeks',)]
```

由于 EXISTS 倾向于对负查询更有效，一个常见的查询是定位那些不存在相关实体的实体。这可以用如`~User.addresses.any()`这样的短语来简洁地实现，以选择没有相关`Address`行的`User`实体：

```py
>>> stmt = select(User.fullname).where(~User.addresses.any())
>>> session.execute(stmt).all()
SELECT  user_account.fullname
FROM  user_account
WHERE  NOT  (EXISTS  (SELECT  1
FROM  address
WHERE  user_account.id  =  address.user_id))
[...]  ()
[('Eugene H. Krabs',)]
```

`PropComparator.has()`方法的工作方式基本与`PropComparator.any()`相同，只是它用于多对一的关系，比如我们想要定位所有属于“sandy”的`Address`对象：

```py
>>> stmt = select(Address.email_address).where(Address.user.has(User.name == "sandy"))
>>> session.execute(stmt).all()
SELECT  address.email_address
FROM  address
WHERE  EXISTS  (SELECT  1
FROM  user_account
WHERE  user_account.id  =  address.user_id  AND  user_account.name  =  ?)
[...]  ('sandy',)
[('sandy@sqlalchemy.org',), ('squirrel@squirrelpower.org',)]
```

### Relationship Instance Comparison Operators

`relationship()`绑定属性还提供了一些 SQL 构建实现，用于基于特定相关对象的实例来过滤`relationship()`绑定属性，这可以从给定的持久（或更少见的分离）对象实例中解包适当的属性值，并根据目标`relationship()`构造 WHERE 条件。

+   **多对一等于比较** - 一个特定的对象实例可以与多对一关系进行比较，以选择外键与目标实体的主键值匹配的行：

    ```py
    >>> user_obj = session.get(User, 1)
    SELECT  ...
    >>> print(select(Address).where(Address.user == user_obj))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  :param_1  =  address.user_id 
    ```

+   **多对一不等于比较** - 也可以使用不等于运算符：

    ```py
    >>> print(select(Address).where(Address.user != user_obj))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  address.user_id  !=  :user_id_1  OR  address.user_id  IS  NULL 
    ```

+   **对象包含在一对多集合中** - 这基本上是“等于”比较的一对多版本，选择主键等于相关对象中外键值的行：

    ```py
    >>> address_obj = session.get(Address, 1)
    SELECT  ...
    >>> print(select(User).where(User.addresses.contains(address_obj)))
    SELECT  user_account.id,  user_account.name,  user_account.fullname
    FROM  user_account
    WHERE  user_account.id  =  :param_1 
    ```

+   **从一对多的角度看，对象有一个特定的父对象** - `with_parent()` 函数生成一个比较，返回由给定父对象引用的行，这与使用`==`运算符与多对一方面基本相同：

    ```py
    >>> from sqlalchemy.orm import with_parent
    >>> print(select(Address).where(with_parent(user_obj, User.addresses)))
    SELECT  address.id,  address.user_id,  address.email_address
    FROM  address
    WHERE  :param_1  =  address.user_id 
    ```
