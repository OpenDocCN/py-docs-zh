# 处理 ORM 相关对象

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/orm_related_objects.html`](https://docs.sqlalchemy.org/en/20/tutorial/orm_related_objects.html)

在本节中，我们将涵盖另一个重要的 ORM 概念，即 ORM 如何与引用其他对象的映射类交互。在 声明映射类 部分，映射类示例使用了一种称为 `relationship()` 的构造。此构造定义了两个不同映射类之间的链接，或者从一个映射类到它自身，后者称为**自引用**关系。

要描述 `relationship()` 的基本思想，首先我们将以简短形式回顾映射，省略 `mapped_column()` 映射和其他指令。

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "user_account"

    # ... mapped_column() mappings

    addresses: Mapped[List["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "address"

    # ... mapped_column() mappings

    user: Mapped["User"] = relationship(back_populates="addresses")
```

如上，`User` 类现在有一个属性 `User.addresses`，而 `Address` 类有一个属性 `Address.user`。 `relationship()` 构造与 `Mapped` 构造一起指示类型行为，将用于检查与 `User` 和 `Address` 类映射到的 `Table` 对象之间的表关系。由于代表 `address` 表的 `Table` 对象具有指向 `user_account` 表的 `ForeignKeyConstraint`，`relationship()` 可以明确确定从 `User` 类到 `Address` 类的 一对多 关系，沿着 `User.addresses` 关系；`user_account` 表中的一个特定行可能被 `address` 表中的多行引用。

所有一对多关系自然对应于另一个方向的多对一关系，在本例中由`Address.user`指出。如上所示在两个`relationship()`对象上配置的`relationship.back_populates`参数，建立了这两个`relationship()`构造应被视为彼此补充；我们将在下一节中看到这是如何运作的。

## 持久化和加载关系

我们可以首先说明`relationship()`对对象实例做了什么。如果我们创建一个新的`User`对象，我们可以注意到当我们访问`.addresses`元素时有一个 Python 列表：

```py
>>> u1 = User(name="pkrabs", fullname="Pearl Krabs")
>>> u1.addresses
[]
```

此对象是 Python `list`的 SQLAlchemy 特定版本，具有跟踪和响应对其进行的更改的能力。即使我们从未将其分配给对象，当我们访问属性时，集合也会自动出现。这类似于在使用 ORM 工作单元模式插入行中观察到的行为，在那里我们观察到，我们没有明确为其分配值的基于列的属性也会自动显示为`None`，而不是像 Python 通常行为一样引发`AttributeError`。

由于`u1`对象仍然是瞬态，我们从`u1.addresses`获取的`list`尚未发生变异（即未被追加或扩展），因此它实际上还没有与对象关联，但随着我们对其进行更改，它将成为`User`对象状态的一部分。

该集合专用于`Address`类，这是唯一可以在其中持久化的 Python 对象类型。我们可以使用`list.append()`方法添加一个`Address`对象：

```py
>>> a1 = Address(email_address="pearl.krabs@gmail.com")
>>> u1.addresses.append(a1)
```

此时，`u1.addresses`集合如预期中包含新的`Address`对象：

```py
>>> u1.addresses
[Address(id=None, email_address='pearl.krabs@gmail.com')]
```

当我们将`Address`对象与`u1`实例的`User.addresses`集合关联时，还发生了另一个行为，即`User.addresses`关系将自动与`Address.user`关系同步，这样我们不仅可以从`User`对象导航到`Address`对象，还可以从`Address`对象导航回“父”`User`对象：

```py
>>> a1.user
User(id=None, name='pkrabs', fullname='Pearl Krabs')
```

此同步是由我们在两个 `relationship()` 对象之间使用的 `relationship.back_populates` 参数导致的。此参数命名了另一个应该发生补充属性赋值/列表变异的 `relationship()` 。在另一个方向上同样有效，即如果我们创建另一个 `Address` 对象并将其分配给其 `Address.user` 属性，那么该 `Address` 将成为该 `User` 对象上的 `User.addresses` 集合的一部分：

```py
>>> a2 = Address(email_address="pearl@aol.com", user=u1)
>>> u1.addresses
[Address(id=None, email_address='pearl.krabs@gmail.com'), Address(id=None, email_address='pearl@aol.com')]
```

我们实际上在 `Address` 构造函数中使用了 `user` 参数作为关键字参数，它像在 `Address` 类上声明的任何其他映射属性一样被接受。这相当于在事后分配了 `Address.user` 属性：

```py
# equivalent effect as a2 = Address(user=u1)
>>> a2.user = u1
```

### 将对象级联到会话中

我们现在有一个 `User` 和两个 `Address` 对象，它们在内存中以双向结构关联，但正如之前在 使用 ORM 单元工作模式插入行 中所指出的，这些对象被认为处于 瞬时态 ，直到它们与一个 `Session` 对象关联。

我们继续使用正在进行中的 `Session` ，注意当我们对主要的 `User` 对象应用 `Session.add()` 方法时，相关的 `Address` 对象也被添加到同一个 `Session` 中：

```py
>>> session.add(u1)
>>> u1 in session
True
>>> a1 in session
True
>>> a2 in session
True
```

上述行为，`Session` 接收了一个 `User` 对象，并沿着 `User.addresses` 关系找到了相关的 `Address` 对象，被称为 **保存-更新级联**，在 ORM 参考文档的 级联 中详细讨论。

这三个对象现在处于 pending 状态；这意味着它们准备好被用于 INSERT 操作，但这还没有进行；所有三个对象都还没有分配主键，并且`a1`和`a2`对象还有一个名为`user_id`的属性，它指向具有引用`user_account.id`列的`Column`，这些也都是`None`，因为这些对象还没有与真实的数据库行关联：

```py
>>> print(u1.id)
None
>>> print(a1.user_id)
None
```

正是在这个阶段，我们可以看到工作单元过程提供的非常大的实用性；回想在 INSERT 通常会自动生成“values”子句一节中，使用一些复杂的语法将行插入到`user_account`和`address`表中，以便自动将`address.user_id`列与`user_account`行的列关联起来。此外，我们需要先为`user_account`行发出 INSERT，然后再为`address`行发出 INSERT，因为`address`行依赖于其父行`user_account`中`user_id`列的值。

当使用`Session`时，所有这些繁琐的工作都会为我们处理，即使是最顽固的 SQL 纯粹主义者也可以从 INSERT、UPDATE 和 DELETE 语句的自动化中受益。当我们调用`Session.commit()`提交事务时，所有步骤按正确顺序调用，而且`user_account`行的新生成主键也会适当地应用到`address.user_id`列上：

```py
>>> session.commit()
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('pkrabs',  'Pearl Krabs')
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('pearl.krabs@gmail.com',  6)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('pearl@aol.com',  6)
COMMIT 
```  ## 加载关系

在上一步中，我们调用了`Session.commit()`，这会为事务发出一个 COMMIT，然后根据`Session.commit.expire_on_commit`使所有对象过期，以便它们在下一个事务中刷新。

当我们下次访问这些对象的属性时，我们会看到为行的主要属性发出的 SELECT，比如当我们查看`u1`对象的新生成的主键时：

```py
>>> u1.id
BEGIN  (implicit)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,
user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (6,)
6
```

`u1` `User`对象现在有一个持久化集合`User.addresses`，我们也可以访问它。由于这个集合包含了`address`表中的一组额外行，当我们再次访问这个集合时，我们会再次看到一个延迟加载以检索对象：

```py
>>> u1.addresses
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,
address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (6,)
[Address(id=4, email_address='pearl.krabs@gmail.com'), Address(id=5, email_address='pearl@aol.com')]
```

SQLAlchemy ORM 中的集合和相关属性在内存中是持久的；一旦集合或属性被填充，SQL 就不再发出，直到该集合或属性被过期。我们可以再次访问`u1.addresses`，以及添加或删除项目，并且这不会产生任何新的 SQL 调用：

```py
>>> u1.addresses
[Address(id=4, email_address='pearl.krabs@gmail.com'), Address(id=5, email_address='pearl@aol.com')]
```

虽然懒加载所发出的加载请求如果我们不采取明确的优化步骤就很容易变得昂贵，但至少懒加载的网络相当优化，不会执行冗余的工作；由于 u1.addresses 集合被刷新，根据身份映射，这些实际上是我们已经处理过的`a1`和`a2`对象中的相同的`Address`实例，因此我们已经完成了加载这个特定对象图中的所有属性：

```py
>>> a1
Address(id=4, email_address='pearl.krabs@gmail.com')
>>> a2
Address(id=5, email_address='pearl@aol.com')
```

关系如何加载或不加载的问题是一个独立的主题。稍后在本节的加载策略中对这些概念进行了一些补充介绍。 ## 在查询中使用关系

前一节介绍了当使用**映射类的实例**时`relationship()`构造的行为，上文介绍了`User`和`Address`类的`u1`、`a1`和`a2`实例。在本节中，我们介绍了当应用于映射类的**类级行为**时，`relationship()`的行为，它在多个方面帮助自动构建 SQL 查询。

### 使用关系进行连接

显式 FROM 子句和 JOINs 和设置 ON 子句章节介绍了使用`Select.join()`和`Select.join_from()`方法来组合 SQL JOIN 子句。为了描述如何在表之间进行连接，这些方法要么根据表元数据结构中存在的单个明确的`ForeignKeyConstraint`对象推断出 ON 子句，该对象链接了这两个表，要么我们可以提供一个明确的 SQL 表达式构造，指示特定的 ON 子句。

当使用 ORM 实体时，还有一种额外的机制可用于帮助我们设置连接的 ON 子句，这就是利用我们在用户映射中设置的 `relationship()` 对象，就像在 声明映射类 中演示的那样。相应于 `relationship()` 的类绑定属性可以作为 **单个参数** 传递给 `Select.join()`，它既用于指示连接的右侧，又一次性指示 ON 子句：

```py
>>> print(select(Address.email_address).select_from(User).join(User.addresses))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

映射上的 ORM `relationship()` 的存在，如果我们没有指定 ON 子句，将不会被 `Select.join()` 或 `Select.join_from()` 用于推断 ON 子句。这意味着，如果我们从 `User` 连接到 `Address` 而没有 ON 子句，它会工作是因为两个映射的 `Table` 对象之间的 `ForeignKeyConstraint`，而不是 `User` 和 `Address` 类上的 `relationship()` 对象：

```py
>>> print(select(Address.email_address).join_from(User, Address))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

请参阅 连接 在 ORM 查询指南 中，了解如何使用 `Select.join()` 和 `Select.join_from()` 与 `relationship()` 构造的更多示例。

请参见

连接 在 ORM 查询指南 ### 关系 WHERE 运算符

`relationship()` 还配备了一些额外的 SQL 生成辅助工具，当构建语句的 WHERE 子句时通常很有用。请参阅 关系 WHERE 运算符 在 ORM 查询指南 中的部分。

请参见

关系 WHERE 运算符在 ORM 查询指南中  ## 加载策略

在加载关系部分，我们介绍了这样一个概念，当我们使用映射对象的实例时，访问使用`relationship()`映射的属性时，在默认情况下，如果集合未填充，则会发出延迟加载以加载应该存在于此集合中的对象。

延迟加载是最著名的 ORM 模式之一，也是最具争议的模式之一。当内存中有几十个 ORM 对象分别引用少量未加载的属性时，对这些对象的常规操作可能会产生许多额外的查询，这些查询可能会累积（也称为 N 加一问题），更糟糕的是它们是隐式发出的。这些隐式查询可能不会被注意到，在数据库事务不再可用时尝试执行它们时可能会导致错误，或者在使用诸如 asyncio 之类的替代并发模式时，它们实际上根本不起作用。

与此同时，当与正在使用的并发方法兼容且没有引起问题时，延迟加载是一种非常流行和有用的模式。出于这些原因，SQLAlchemy 的 ORM 非常重视能够控制和优化这种加载行为。

首先，有效使用 ORM 延迟加载的第一步是**测试应用程序，打开 SQL 回显，并观察生成的 SQL 语句**。如果看起来有很多冗余的 SELECT 语句，看起来它们可以更有效地合并为一个，如果对象在已经分离的`Session`中不适当地发生加载，那就是使用**加载策略**的时候。

加载策略表示为可以使用`Select.options()`方法与 SELECT 语句关联的对象，例如：

```py
for user_obj in session.execute(
    select(User).options(selectinload(User.addresses))
).scalars():
    user_obj.addresses  # access addresses collection already loaded
```

它们也可以被配置为`relationship()`的默认值，使用`relationship.lazy`选项，例如：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "user_account"

    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", lazy="selectin"
    )
```

每个加载器策略对象都会向语句中添加某种信息，该信息将在以后由`Session`在决定各种属性在访问时应如何加载和/或行为时使用。

下面的部分将介绍一些最常用的加载器策略。

参见

关系加载技术中的两个部分：

+   在映射时配置加载器策略 - 配置在`relationship()`上的策略的详细信息

+   使用加载器选项进行关系加载 - 使用查询时加载策略的详细信息

### Selectin Load

在现代 SQLAlchemy 中最有用的加载器是`selectinload()`加载器选项。该选项解决了最常见形式的“N 加一”问题，即一组对象引用相关集合。`selectinload()`将确保立即使用单个查询加载整个系列对象的特定集合。它使用一种 SELECT 形式，在大多数情况下可以针对相关表单独发出，而不需要引入 JOIN 或子查询，并且仅查询那些集合尚未加载的父对象。下面我们通过加载所有`User`对象及其所有相关的`Address`对象来说明`selectinload()`；虽然我们只调用了一次`Session.execute()`，给定一个`select()`构造，但在访问数据库时，实际上发出了两个 SELECT 语句，第二个语句是用于获取相关的`Address`对象：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(User).options(selectinload(User.addresses)).order_by(User.id)
>>> for row in session.execute(stmt):
...     print(
...         f"{row.User.name}  ({', '.join(a.email_address for a in row.User.addresses)})"
...     )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  ()
SELECT  address.user_id  AS  address_user_id,  address.id  AS  address_id,
address.email_address  AS  address_email_address
FROM  address
WHERE  address.user_id  IN  (?,  ?,  ?,  ?,  ?,  ?)
[...]  (1,  2,  3,  4,  5,  6)
spongebob  (spongebob@sqlalchemy.org)
sandy  (sandy@sqlalchemy.org, sandy@squirrelpower.org)
patrick  ()
squidward  ()
ehkrabs  ()
pkrabs  (pearl.krabs@gmail.com, pearl@aol.com)
```

参见

选择 IN 加载 - 在关系加载技术中

### Joined Load

`joinedload()`预加载策略是 SQLAlchemy 中最古老的预加载器，它通过在传递给数据库的 SELECT 语句中添加一个 JOIN（根据选项可能是外连接或内连接）来增强，然后可以加载相关对象。

`joinedload()`策略最适合加载相关的多对一对象，因为这只需要向主实体行添加额外的列，在任何情况下都会获取这些列。为了提高效率，它还接受一个选项`joinedload.innerjoin`，这样在下面这种情况下可以使用内连接而不是外连接，我们知道所有的`Address`对象都有一个关联的`User`：

```py
>>> from sqlalchemy.orm import joinedload
>>> stmt = (
...     select(Address)
...     .options(joinedload(Address.user, innerjoin=True))
...     .order_by(Address.id)
... )
>>> for row in session.execute(stmt):
...     print(f"{row.Address.email_address} {row.Address.user.name}")
SELECT  address.id,  address.email_address,  address.user_id,  user_account_1.id  AS  id_1,
user_account_1.name,  user_account_1.fullname
FROM  address
JOIN  user_account  AS  user_account_1  ON  user_account_1.id  =  address.user_id
ORDER  BY  address.id
[...]  ()
spongebob@sqlalchemy.org spongebob
sandy@sqlalchemy.org sandy
sandy@squirrelpower.org sandy
pearl.krabs@gmail.com pkrabs
pearl@aol.com pkrabs
```

`joinedload()`也适用于集合，意味着一对多关系，但它会以递归方式将每个相关项乘以主行，从而增加通过结果集发送的数据量，对于嵌套集合和/或较大集合，这会使数据量成倍增长，因此应该根据具体情况评估其与其他选项（例如`selectinload()`）的使用。

需要注意的是，封闭`Select`语句的 WHERE 和 ORDER BY 条件**不会针对 joinedload()生成的表**。上面的例子中，可以看到 SQL 中对`user_account`表应用了一个**匿名别名**，以便在查询中无法直接寻址。这个概念在加入式预加载的禅意一节中有更详细的讨论。

提示

需要注意的是，多对一的预加载通常是不必要的，因为“N 加一”问题在常见情况下要少得多。当许多对象都引用相同的相关对象时，例如每个都引用相同`User`的许多`Address`对象时，SQL 将仅对该`User`对象发出一次，使用普通的惰性加载。惰性加载例程将在当前`Session`中尽可能地通过主键查找相关对象，而不在可能时发出任何 SQL。

另请参阅

加入式预加载 - 在关系加载技术中

### 明确的连接 + 预加载

如果我们在连接到 `user_account` 表时加载 `Address` 行，使用诸如 `Select.join()` 之类的方法来渲染 JOIN，我们也可以利用该 JOIN 来急切地加载每个返回的 `Address` 对象的 `Address.user` 属性的内容。这本质上就是我们正在使用“连接的急切加载”，但是自己渲染 JOIN。这个常见的用例是通过使用 `contains_eager()` 选项实现的。该选项与 `joinedload()` 非常相似，只是它假设我们已经自己设置了 JOIN，并且它仅指示应该将 COLUMNS 子句中的附加列加载到每个返回对象的相关属性中，例如：

```py
>>> from sqlalchemy.orm import contains_eager
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "pkrabs")
...     .options(contains_eager(Address.user))
...     .order_by(Address.id)
... )
>>> for row in session.execute(stmt):
...     print(f"{row.Address.email_address} {row.Address.user.name}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.email_address,  address.user_id
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  ?  ORDER  BY  address.id
[...]  ('pkrabs',)
pearl.krabs@gmail.com pkrabs
pearl@aol.com pkrabs
```

上面，我们同时对 `user_account.name` 进行了筛选，并且将 `user_account` 中的行加载到返回的行的 `Address.user` 属性中。如果我们分别应用了 `joinedload()` ，我们将会得到一个不必要两次连接的 SQL 查询：

```py
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "pkrabs")
...     .options(joinedload(Address.user))
...     .order_by(Address.id)
... )
>>> print(stmt)  # SELECT has a JOIN and LEFT OUTER JOIN unnecessarily
SELECT  address.id,  address.email_address,  address.user_id,
user_account_1.id  AS  id_1,  user_account_1.name,  user_account_1.fullname
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
LEFT  OUTER  JOIN  user_account  AS  user_account_1  ON  user_account_1.id  =  address.user_id
WHERE  user_account.name  =  :name_1  ORDER  BY  address.id 
```

另请参阅

关系加载技术中的两个部分：

+   连接急切加载的禅意 - 详细描述了上述问题

+   将显式连接/语句路由到急切加载的集合 - 使用 `contains_eager()`

### Raiseload

值得一提的另一个加载器策略是 `raiseload()` 。此选项用于通过导致通常将是延迟加载的操作引发错误来完全阻止应用程序遇到 N 加一 问题。它有两个变体，通过 `raiseload.sql_only` 选项进行控制，以阻止需要 SQL 的延迟加载，与所有“加载”操作，包括仅需要查询当前 `Session` 的那些操作。

使用 `raiseload()` 的一种方法是在 `relationship()` 上配置它，通过将 `relationship.lazy` 设置为值 `"raise_on_sql"`，这样对于特定映射，某个关系将永远不会尝试发出 SQL：

```py
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import relationship

>>> class User(Base):
...     __tablename__ = "user_account"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     addresses: Mapped[List["Address"]] = relationship(
...         back_populates="user", lazy="raise_on_sql"
...     )

>>> class Address(Base):
...     __tablename__ = "address"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     user: Mapped["User"] = relationship(back_populates="addresses", lazy="raise_on_sql")
```

使用这样的映射，应用程序被阻止了懒加载，表明特定查询需要指定一个加载策略：

```py
>>> u1 = session.execute(select(User)).scalars().first()
SELECT  user_account.id  FROM  user_account
[...]  ()
>>> u1.addresses
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'User.addresses' is not available due to lazy='raise_on_sql'
```

异常将指示应该预先加载此集合：

```py
>>> u1 = (
...     session.execute(select(User).options(selectinload(User.addresses)))
...     .scalars()
...     .first()
... )
SELECT  user_account.id
FROM  user_account
[...]  ()
SELECT  address.user_id  AS  address_user_id,  address.id  AS  address_id
FROM  address
WHERE  address.user_id  IN  (?,  ?,  ?,  ?,  ?,  ?)
[...]  (1,  2,  3,  4,  5,  6) 
```

`lazy="raise_on_sql"` 选项也会对多对一关系进行智能处理；上面，如果一个 `Address` 对象的 `Address.user` 属性未加载，但是该 `User` 对象在同一个 `Session` 中本地存在，那么“raiseload”策略将不会引发错误。

另请参阅

使用 raiseload 阻止不必要的懒加载 - 在关系加载技术中

## 持久化和加载关系

我们可以先说明 `relationship()` 对象实例的作用。如果我们创建一个新的 `User` 对象，我们可以注意到当我们访问 `.addresses` 元素时会有一个 Python 列表：

```py
>>> u1 = User(name="pkrabs", fullname="Pearl Krabs")
>>> u1.addresses
[]
```

此对象是 Python `list` 的 SQLAlchemy 特定版本，具有跟踪和响应对其进行的更改的能力。当我们访问属性时，集合也会自动出现，即使我们从未将其分配给对象。这类似于在 使用 ORM 工作单元模式插入行 中注意到的行为，即我们没有明确为其分配值的基于列的属性也会自动显示为 `None`，而不是像 Python 的通常行为那样引发 `AttributeError`。

由于 `u1` 对象仍然是 瞬态，并且我们从 `u1.addresses` 得到的 `list` 尚未被改变（即未被添加或扩展），因此实际上尚未与对象关联，但是当我们对其进行更改时，它将成为 `User` 对象状态的一部分。

该集合专用于 `Address` 类，这是唯一可以在其中持久化的 Python 对象类型。使用 `list.append()` 方法，我们可以添加一个 `Address` 对象：

```py
>>> a1 = Address(email_address="pearl.krabs@gmail.com")
>>> u1.addresses.append(a1)
```

此时，`u1.addresses` 集合按预期包含了新的 `Address` 对象：

```py
>>> u1.addresses
[Address(id=None, email_address='pearl.krabs@gmail.com')]
```

当我们将`Address`对象与`u1`实例的`User.addresses`集合关联起来时，还发生了另一个行为，即`User.addresses`关系与`Address.user`关系同步，这样我们不仅可以从`User`对象导航到`Address`对象，还可以从`Address`对象导航回“父”`User`对象：

```py
>>> a1.user
User(id=None, name='pkrabs', fullname='Pearl Krabs')
```

这种同步发生是因为我们在两个`relationship()`对象之间使用了`relationship.back_populates`参数。该参数命名了另一个应进行互补属性赋值/列表变异的`relationship()`。在另一个方向上同样有效，即如果我们创建另一个`Address`对象并将其分配给其`Address.user`属性，该`Address`将成为`User`对象上的`User.addresses`集合的一部分：

```py
>>> a2 = Address(email_address="pearl@aol.com", user=u1)
>>> u1.addresses
[Address(id=None, email_address='pearl.krabs@gmail.com'), Address(id=None, email_address='pearl@aol.com')]
```

我们实际上在`Address`构造函数中使用了`user`参数作为关键字参数，这与在`Address`类上声明的任何其他映射属性一样被接受。这相当于事后对`Address.user`属性进行赋值：

```py
# equivalent effect as a2 = Address(user=u1)
>>> a2.user = u1
```

### 将对象级联到会话中

现在我们有一个`User`和两个`Address`对象，在内存中以双向结构关联，但如前所述，在使用 ORM 工作单元模式插入行中，这些对象被称为处于瞬态状态，直到它们与一个`Session`对象关联为止。

我们利用的是仍在进行中的`Session`，请注意，当我们对主`User`对象应用`Session.add()`方法时，相关的`Address`对象也会被添加到同一个`Session`中：

```py
>>> session.add(u1)
>>> u1 in session
True
>>> a1 in session
True
>>> a2 in session
True
```

上述行为，即`Session`接收到一个`User`对象，并沿着`User.addresses`关系定位相关的`Address`对象的行为，被称为**保存更新级联**，并在 ORM 参考文档中详细讨论，链接地址为 Cascades。

这三个对象现在处于 挂起 状态；这意味着它们已经准备好成为 INSERT 操作的对象，但这还没有进行；所有三个对象目前还没有分配主键，并且此外，`a1` 和 `a2` 对象具有一个名为 `user_id` 的属性，该属性指向具有引用 `user_account.id` 列的 `Column`，这些属性也是 `None`，因为这些对象尚未与真实的数据库行关联：

```py
>>> print(u1.id)
None
>>> print(a1.user_id)
None
```

此时，我们可以看到工作单元流程提供的非常大的实用性；回想一下在 INSERT 通常会自动生成“values”子句 中，行是如何插入到 `user_account` 和 `address` 表中的，使用一些复杂的语法来自动将 `address.user_id` 列与 `user_account` 表中的列关联起来。此外，我们必须首先为 `user_account` 表中的行发出 INSERT，然后是 `address` 表中的行，因为 `address` 中的行**依赖于**其在 `user_account` 表中的父行，以获取其 `user_id` 列中的值。

使用 `Session` 时，所有这些烦琐工作都由我们处理，即使是最铁杆的 SQL 纯粹主义者也可以从 INSERT、UPDATE 和 DELETE 语句的自动化中受益。当我们调用 `Session.commit()` 时，所有步骤都按正确的顺序执行，并且还会将 `user_account` 行的新生成的主键适当地应用到 `address.user_id` 列中：

```py
>>> session.commit()
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('pkrabs',  'Pearl Krabs')
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('pearl.krabs@gmail.com',  6)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('pearl@aol.com',  6)
COMMIT 
```  ### 将对象级联到会话中

现在，我们在内存中有一个双向结构的 `User` 对象和两个 `Address` 对象，但正如之前在 使用 ORM 工作单元模式插入行 中所述，这些对象被认为处于 瞬时 状态，直到它们与一个 `Session` 对象关联为止。

我们利用的是仍在进行中的 `Session`，请注意，当我们将 `Session.add()` 方法应用于主 `User` 对象时，相关的 `Address` 对象也会被添加到同一个 `Session` 中：

```py
>>> session.add(u1)
>>> u1 in session
True
>>> a1 in session
True
>>> a2 in session
True
```

上述行为，其中`Session`接收到一个 `User` 对象，并沿着 `User.addresses` 关系跟踪以找到相关的 `Address` 对象，被称为**save-update cascade**，并且在 ORM 参考文档的 Cascades 中有详细讨论。

这三个对象现在处于 pending 状态；这意味着它们已准备好成为 INSERT 操作的主体，但还没有进行；这三个对象都还没有分配主键，并且此外，`a1` 和 `a2` 对象具有一个名为 `user_id` 的属性，它指向具有引用 `user_account.id` 列的`Column`；由于这些对象尚未与真实的数据库行关联，因此这些值也都是 `None`：

```py
>>> print(u1.id)
None
>>> print(a1.user_id)
None
```

此时，我们可以看到工作单元流程提供的非常大的实用性；回想一下，在 INSERT 通常自动生成“values”子句一节中，我们使用一些复杂的语法将行插入到 `user_account` 和 `address` 表中，以便自动将 `address.user_id` 列与 `user_account` 行的列关联起来。此外，必须先为 `user_account` 行发出 INSERT，然后才能为 `address` 的行发出 INSERT，因为 `address` 中的行依赖于其父行 `user_account` 以在其 `user_id` 列中获得值。

当使用`Session`时，所有这些繁琐的工作都由我们处理，即使是最铁杆的 SQL 纯粹主义者也可以从 INSERT、UPDATE 和 DELETE 语句的自动化中受益。当我们调用`Session.commit()`提交事务时，所有步骤都按正确的顺序执行，并且新生成的 `user_account` 行的主键还会适当地应用到 `address.user_id` 列上：

```py
>>> session.commit()
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)
[...]  ('pkrabs',  'Pearl Krabs')
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...  (insertmanyvalues)  1/2  (ordered;  batch  not  supported)]  ('pearl.krabs@gmail.com',  6)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[insertmanyvalues  2/2  (ordered;  batch  not  supported)]  ('pearl@aol.com',  6)
COMMIT 
```

## 加载关系

在最后一步中，我们调用了`Session.commit()`，它发出了一个 COMMIT 以提交事务，然后根据`Session.commit.expire_on_commit`将所有对象过期，以便它们为下一个事务刷新。

当我们下次访问这些对象的属性时，我们将看到为行的主要属性发出的 SELECT，例如当我们查看 `u1` 对象的新生成的主键时：

```py
>>> u1.id
BEGIN  (implicit)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,
user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (6,)
6
```

现在 `u1` `User` 对象具有一个持久集合 `User.addresses`，我们也可以访问它。由于此集合包含来自 `address` 表的一组额外行，因此当我们再次访问此集合时，我们会再次看到一个懒加载以检索对象：

```py
>>> u1.addresses
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,
address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (6,)
[Address(id=4, email_address='pearl.krabs@gmail.com'), Address(id=5, email_address='pearl@aol.com')]
```

SQLAlchemy ORM 中的集合和相关属性是在内存中持久存在的；一旦集合或属性被填充，SQL 就不再生成，直到该集合或属性被过期。我们可以再次访问 `u1.addresses`，并添加或删除项目，这不会产生任何新的 SQL 调用：

```py
>>> u1.addresses
[Address(id=4, email_address='pearl.krabs@gmail.com'), Address(id=5, email_address='pearl@aol.com')]
```

如果我们不采取显式步骤来优化懒加载，懒加载引发的加载可能会很快变得昂贵，但至少懒加载的网络相对来说已经相当优化，不会执行冗余工作；因为 `u1.addresses` 集合已经刷新，根据标识映射，这些实际上是我们已经处理过的 `a1` 和 `a2` 对象的同一 `Address` 实例，所以我们已经完成了加载此特定对象图中的所有属性：

```py
>>> a1
Address(id=4, email_address='pearl.krabs@gmail.com')
>>> a2
Address(id=5, email_address='pearl@aol.com')
```

关系如何加载或不加载是一个独立的主题。稍后在本节的 加载器策略 中会对这些概念进行一些额外的介绍。

## 在查询中使用关系

前一节介绍了当与**映射类的实例**一起使用时 `relationship()` 构造的行为，上面是 `User` 和 `Address` 类的 `u1`、`a1` 和 `a2` 实例。在本节中，我们将介绍当应用于**映射类的类级行为**时 `relationship()` 的行为，在这里，它以几种方式帮助自动构建 SQL 查询。

### 使用关系进行连接

显式的 FROM 子句和 JOINs 和 设置 ON 子句 章节介绍了使用 `Select.join()` 和 `Select.join_from()` 方法来组合 SQL JOIN 子句。为了描述如何在表之间进行连接，这些方法要么**根据表元数据结构中链接两个表的单个明确的 `ForeignKeyConstraint` 对象推断出 ON 子句，要么我们可以提供一个明确的 SQL 表达式构造，指示特定的 ON 子句。

在使用 ORM 实体时，有一种额外的机制可帮助我们设置连接的 ON 子句，那就是利用我们在用户映射中设置的`relationship()`对象，就像在声明映射类中所演示的那样。相应于`relationship()`的类绑定属性可以作为**单个参数**传递给`Select.join()`，在这里它同时用于指示连接的右侧以及 ON 子句：

```py
>>> print(select(Address.email_address).select_from(User).join(User.addresses))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

如果我们没有指定 ON 子句，则映射上的 ORM `relationship()`对 `Select.join()` 或 `Select.join_from()` 的存在不会用于推断 ON 子句。这意味着，如果我们从 `User` 连接到 `Address` 而没有 ON 子句，这是因为两个映射的 `Table` 对象之间的 `ForeignKeyConstraint`，而不是由于 `User` 和 `Address` 类上的 `relationship()` 对象：

```py
>>> print(select(Address.email_address).join_from(User, Address))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

请参阅 ORM 查询指南中的连接一节，了解如何使用 `Select.join()` 和 `Select.join_from()` 以及 `relationship()` 构造的更多示例。

另请参阅

ORM 查询指南中的连接  ### Relationship WHERE 运算符

还有一些额外的 SQL 生成辅助程序，随着 `relationship()` 一起提供，当构建语句的 WHERE 子句时通常很有用。请参阅 ORM 查询指南中的 Relationship WHERE 运算符一节。

另请参阅

ORM 查询指南中的关系 WHERE 运算符 ### 使用关系进行连接 在 ORM 查询指南

明确的 FROM 子句和 JOIN 和设置 ON 子句部分介绍了使用`Select.join()`和`Select.join_from()`方法组成 SQL JOIN 子句的用法。为了描述如何在表之间进行连接，这些方法根据表元数据结构中链接两个表的单一明确`ForeignKeyConstraint`对象的存在**推断** ON 子句，或者我们可以提供一个明确的 SQL 表达式构造来指示特定的 ON 子句。

在使用 ORM 实体时，有一种额外的机制可帮助我们设置连接的 ON 子句，即利用我们在用户映射中设置的`relationship()`对象，就像在声明映射类中所演示的那样。相应于`relationship()`的类绑定属性可以作为**单个参数**传递给`Select.join()`，在这里它既用于指示连接的右侧，又用于一次性指示 ON 子句：

```py
>>> print(select(Address.email_address).select_from(User).join(User.addresses))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

如果我们不指定，映射中的 ORM `relationship()`的存在不会被`Select.join()`或`Select.join_from()`用于推断 ON 子句。这意味着，如果我们从 `User` 到 `Address` 进行连接而没有 ON 子句，这是因为两个映射的 `Table` 对象之间的 `ForeignKeyConstraint`，而不是 `User` 和 `Address` 类上的 `relationship()` 对象：

```py
>>> print(select(Address.email_address).join_from(User, Address))
SELECT  address.email_address
FROM  user_account  JOIN  address  ON  user_account.id  =  address.user_id 
```

在 ORM 查询指南中查看连接（Joins）部分，了解如何使用`Select.join()`和`Select.join_from()`以及`relationship()`构造的更多示例。

另请参阅

ORM 查询指南中的连接（Joins）

### 关系 WHERE 运算符

在构建语句的 WHERE 子句时，`relationship()`还附带了一些其他类型的 SQL 生成助手，通常在构建过程中非常有用。请查看 ORM 查询指南中的关系 WHERE 运算符部分。

另请参阅

在 ORM 查询指南中的关系 WHERE 运算符部分

## 加载策略

在加载关系部分，我们介绍了一个概念，即当我们处理映射对象的实例时，默认情况下访问使用`relationship()`映射的属性时，如果集合未填充，则会发出惰性加载以加载应该存在于此集合中的对象。

懒加载是最著名的 ORM 模式之一，也是最具争议的模式之一。当内存中有几十个 ORM 对象各自引用了少量未加载的属性时，对这些对象的常规操作可能会产生许多额外的查询，这些查询可能会积累起来（也被称为 N 加一问题），更糟糕的是它们是隐式生成的。这些隐式查询可能不会被注意到，在没有数据库事务可用时尝试使用它们时可能会导致错误，或者当使用诸如 asyncio 等替代并发模式时，它们实际上根本不起作用。

与此同时，当它与正在使用的并发方法兼容并且没有引起问题时，懒加载是一种非常受欢迎和有用的模式。因此，SQLAlchemy 的 ORM 非常强调能够控制和优化此加载行为。

最重要的是，有效使用 ORM 懒加载的第一步是**测试应用程序，打开 SQL 回显，并观察发出的 SQL 语句**。如果看起来有大量的冗余的 SELECT 语句，看起来很像它们可以更有效地合并为一个，如果发生了适用于已从其 `Session` 中分离的对象的不适当的加载，那么就要考虑使用**加载器策略**。

加载器策略表示为对象，可以使用 `Select.options()` 方法将其与 SELECT 语句关联，例如：

```py
for user_obj in session.execute(
    select(User).options(selectinload(User.addresses))
).scalars():
    user_obj.addresses  # access addresses collection already loaded
```

也可以将其配置为 `relationship()` 的默认值，使用 `relationship.lazy` 选项，例如：

```py
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "user_account"

    addresses: Mapped[List["Address"]] = relationship(
        back_populates="user", lazy="selectin"
    )
```

每个加载器策略对象都会向语句添加某种信息，该信息稍后将由 `Session` 在决定在访问属性时应如何加载和/或行为时使用。

下面的章节将介绍一些最常用的加载器策略。

另请参阅

关系加载技术 中的两个部分：

+   在映射时配置加载器策略 - 详细介绍了在 `relationship()` 上配置策略的方法。

+   使用加载器选项进行关系加载 - 详细介绍了使用查询时加载策略的方法。

### Selectin Load

在现代 SQLAlchemy 中最有用的加载器是 `selectinload()` 加载器选项。该选项解决了“N plus one”问题的最常见形式，即一组对象引用相关集合。`selectinload()` 将确保通过单个查询一次性加载一系列对象的特定集合。它使用的 SELECT 形式在大多数情况下可以只针对相关表发出，而不需要引入 JOIN 或子查询，并且仅查询那些尚未加载集合的父对象。下面我们通过加载所有 `User` 对象及其所有相关的 `Address` 对象来说明 `selectinload()`；虽然我们只调用一次 `Session.execute()`，但在访问数据库时实际上发出了两个 SELECT 语句，第二个语句用于获取相关的 `Address` 对象：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(User).options(selectinload(User.addresses)).order_by(User.id)
>>> for row in session.execute(stmt):
...     print(
...         f"{row.User.name}  ({', '.join(a.email_address for a in row.User.addresses)})"
...     )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  ()
SELECT  address.user_id  AS  address_user_id,  address.id  AS  address_id,
address.email_address  AS  address_email_address
FROM  address
WHERE  address.user_id  IN  (?,  ?,  ?,  ?,  ?,  ?)
[...]  (1,  2,  3,  4,  5,  6)
spongebob  (spongebob@sqlalchemy.org)
sandy  (sandy@sqlalchemy.org, sandy@squirrelpower.org)
patrick  ()
squidward  ()
ehkrabs  ()
pkrabs  (pearl.krabs@gmail.com, pearl@aol.com)
```

另请参阅

选择 IN 加载 - 在关系加载技术中

### 联合加载

`joinedload()` 立即加载策略是 SQLAlchemy 中最古老的立即加载器，它通过将传递给数据库的 SELECT 语句与 JOIN（取决于选项可能是外连接或内连接）相结合，从而可以加载相关对象。

`joinedload()` 策略最适合于加载相关的多对一对象，因为这仅需要将额外的列添加到主实体行中，而这些列无论如何都会被获取。为了提高效率，它还接受一个选项 `joinedload.innerjoin`，以便在我们知道所有 `Address` 对象都有关联的 `User` 的情况下使用内连接而不是外连接：

```py
>>> from sqlalchemy.orm import joinedload
>>> stmt = (
...     select(Address)
...     .options(joinedload(Address.user, innerjoin=True))
...     .order_by(Address.id)
... )
>>> for row in session.execute(stmt):
...     print(f"{row.Address.email_address} {row.Address.user.name}")
SELECT  address.id,  address.email_address,  address.user_id,  user_account_1.id  AS  id_1,
user_account_1.name,  user_account_1.fullname
FROM  address
JOIN  user_account  AS  user_account_1  ON  user_account_1.id  =  address.user_id
ORDER  BY  address.id
[...]  ()
spongebob@sqlalchemy.org spongebob
sandy@sqlalchemy.org sandy
sandy@squirrelpower.org sandy
pearl.krabs@gmail.com pkrabs
pearl@aol.com pkrabs
```

`joinedload()` 也适用于集合，即一对多关系，但它会以递归方式将主要行乘以相关项目，从而使结果集发送的数据量呈数量级增长，对于嵌套集合和/或较大集合，因此应该根据具体情况评估其与其他选项（如`selectinload()`）的使用。

需要注意的是，封闭`Select`语句的 WHERE 和 ORDER BY 条件**不针对 joinedload()渲染的表**。在上面的 SQL 中可以看到，`user_account`表被应用了**匿名别名**，因此在查询中无法直接访问。这个概念在连接急切加载的禅意部分中有更详细的讨论。

提示

需要注意的是，很多对一的急切加载通常是不必要的，因为“N 加一”问题在常见情况下不太普遍。当许多对象都引用同一个相关对象时，比如许多`Address`对象都引用同一个`User`时，SQL 只会针对该`User`对象正常使用延迟加载而发出一次。延迟加载程序将在当前`Session`中通过主键查找相关对象，尽可能不发出任何 SQL。

另请参阅

连接急切加载 - 在关系加载技术中

### 显式连接 + 急切加载

如果我们在连接到`user_account`表时加载`Address`行，使用诸如`Select.join()`之类的方法来渲染 JOIN，我们还可以利用该 JOIN 来急切加载每个返回的`Address`对象上的`Address.user`属性的内容。这本质上是我们在使用“连接急切加载”，但自己渲染 JOIN。通过使用`contains_eager()`选项来实现这种常见用例。该选项与`joinedload()`非常相似，只是它假设我们自己设置了 JOIN，并且它只表示应该将 COLUMNS 子句中的附加列加载到每个返回对象的相关属性中，例如：

```py
>>> from sqlalchemy.orm import contains_eager
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "pkrabs")
...     .options(contains_eager(Address.user))
...     .order_by(Address.id)
... )
>>> for row in session.execute(stmt):
...     print(f"{row.Address.email_address} {row.Address.user.name}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.email_address,  address.user_id
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  ?  ORDER  BY  address.id
[...]  ('pkrabs',)
pearl.krabs@gmail.com pkrabs
pearl@aol.com pkrabs
```

在上面的例子中，我们既过滤了`user_account.name`的行，也将`user_account`的行加载到返回行的`Address.user`属性中。如果我们单独应用了`joinedload()`，我们将得到一个不必要两次连接的 SQL 查询：

```py
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "pkrabs")
...     .options(joinedload(Address.user))
...     .order_by(Address.id)
... )
>>> print(stmt)  # SELECT has a JOIN and LEFT OUTER JOIN unnecessarily
SELECT  address.id,  address.email_address,  address.user_id,
user_account_1.id  AS  id_1,  user_account_1.name,  user_account_1.fullname
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
LEFT  OUTER  JOIN  user_account  AS  user_account_1  ON  user_account_1.id  =  address.user_id
WHERE  user_account.name  =  :name_1  ORDER  BY  address.id 
```

请参阅

关系加载技术中的两个部分：

+   连接式预加载的禅意 - 详细描述了上述问题

+   将显式连接/语句路由到已预加载的集合 - 使用`contains_eager()`

### Raiseload

值得一提的一个额外的加载器策略是`raiseload()`。此选项用于通过导致通常是惰性加载的操作引发错误，从而完全阻止应用程序遇到 N 加 1 问题。它有两个变体，通过`raiseload.sql_only`选项进行控制，以阻止仅需要 SQL 的惰性加载，以及所有“加载”操作，包括仅需要查询当前`Session`的操作。

使用`raiseload()`的一种方法是在`relationship()`本身上进行配置，通过将`relationship.lazy`设置为值`"raise_on_sql"`，以便对于特定映射，某个关系永远不会尝试发出 SQL：

```py
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import relationship

>>> class User(Base):
...     __tablename__ = "user_account"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     addresses: Mapped[List["Address"]] = relationship(
...         back_populates="user", lazy="raise_on_sql"
...     )

>>> class Address(Base):
...     __tablename__ = "address"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     user: Mapped["User"] = relationship(back_populates="addresses", lazy="raise_on_sql")
```

使用这样的映射，应用程序被阻止了惰性加载，表示特定查询需要指定加载器策略：

```py
>>> u1 = session.execute(select(User)).scalars().first()
SELECT  user_account.id  FROM  user_account
[...]  ()
>>> u1.addresses
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'User.addresses' is not available due to lazy='raise_on_sql'
```

异常会指示应该预先加载此集合：

```py
>>> u1 = (
...     session.execute(select(User).options(selectinload(User.addresses)))
...     .scalars()
...     .first()
... )
SELECT  user_account.id
FROM  user_account
[...]  ()
SELECT  address.user_id  AS  address_user_id,  address.id  AS  address_id
FROM  address
WHERE  address.user_id  IN  (?,  ?,  ?,  ?,  ?,  ?)
[...]  (1,  2,  3,  4,  5,  6) 
```

`lazy="raise_on_sql"`选项也试图对多对一关系变得更加智能；在上面的例子中，如果`Address`对象的`Address.user`属性未加载，但是`User`对象在同一个`Session`中是本地存在的，那么“raiseload”策略就不会引发错误。

请参阅

使用 raiseload 防止不必要的惰性加载 - 在关系加载技术中

### Selectin Load

在现代 SQLAlchemy 中最有用的加载器是 `selectinload()` 加载器选项。该选项解决了“N 加一”问题的最常见形式，即一组对象引用相关集合的问题。`selectinload()` 将确保一系列对象的特定集合通过单个查询提前加载。它使用一个 SELECT 形式，在大多数情况下可以针对相关表单独发出，而无需引入 JOIN 或子查询，并且仅查询那些集合尚未加载的父对象。下面我们通过加载所有的 `User` 对象及其所有相关的 `Address` 对象来说明 `selectinload()`；虽然我们只调用一次 `Session.execute()`，给定一个 `select()` 构造，在访问数据库时，实际上会发出两个 SELECT 语句，第二个用于获取相关的 `Address` 对象：

```py
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(User).options(selectinload(User.addresses)).order_by(User.id)
>>> for row in session.execute(stmt):
...     print(
...         f"{row.User.name}  ({', '.join(a.email_address for a in row.User.addresses)})"
...     )
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account  ORDER  BY  user_account.id
[...]  ()
SELECT  address.user_id  AS  address_user_id,  address.id  AS  address_id,
address.email_address  AS  address_email_address
FROM  address
WHERE  address.user_id  IN  (?,  ?,  ?,  ?,  ?,  ?)
[...]  (1,  2,  3,  4,  5,  6)
spongebob  (spongebob@sqlalchemy.org)
sandy  (sandy@sqlalchemy.org, sandy@squirrelpower.org)
patrick  ()
squidward  ()
ehkrabs  ()
pkrabs  (pearl.krabs@gmail.com, pearl@aol.com)
```

另见

选择 IN 加载 - 在关系加载技术中

### 加载连接

`joinedload()` 预加载策略是 SQLAlchemy 中最古老的预加载器，它通过在传递给数据库的 SELECT 语句中添加 JOIN（根据选项可能是外连接或内连接）来增强查询，然后可以加载相关联的对象。

`joinedload()` 策略最适合加载相关的一对多对象，因为这只需要向主实体行添加额外的列，这些列无论如何都会被检索。为了提高效率，它还接受一个选项 `joinedload.innerjoin`，以便在下面这种情况下使用内连接而不是外连接，我们知道所有 `Address` 对象都有一个关联的 `User`：

```py
>>> from sqlalchemy.orm import joinedload
>>> stmt = (
...     select(Address)
...     .options(joinedload(Address.user, innerjoin=True))
...     .order_by(Address.id)
... )
>>> for row in session.execute(stmt):
...     print(f"{row.Address.email_address} {row.Address.user.name}")
SELECT  address.id,  address.email_address,  address.user_id,  user_account_1.id  AS  id_1,
user_account_1.name,  user_account_1.fullname
FROM  address
JOIN  user_account  AS  user_account_1  ON  user_account_1.id  =  address.user_id
ORDER  BY  address.id
[...]  ()
spongebob@sqlalchemy.org spongebob
sandy@sqlalchemy.org sandy
sandy@squirrelpower.org sandy
pearl.krabs@gmail.com pkrabs
pearl@aol.com pkrabs
```

`joinedload()`也适用于集合，意味着一对多关系，但是它会以递归方式将主要行乘以相关项目，这样会使结果集发送的数据量呈数量级增长，用于嵌套集合和/或较大集合的情况下，应该根据情况评估其与其他选项（例如`selectinload()`）的使用情况。

重要的是要注意，封闭`Select`语句的 WHERE 和 ORDER BY 条件**不会针对 joinedload()渲染的表**。如上所述，在 SQL 中可以看到对`user_account`表应用了**匿名别名**，因此无法直接在查询中进行地址定位。这个概念在 联接式预加载之禅 部分中有更详细的讨论。

小贴士

重要的是要注意，往往不必要进行多对一的急切加载，因为在常见情况下，“N 加一”问题不太普遍。当许多对象都引用同一个相关对象时，例如每个引用同一个`User`的许多`Address`对象时，SQL 将仅一次对该`User`对象使用正常的延迟加载。延迟加载程序将尽可能地在当前`Session`中通过主键查找相关对象，而不会在可能时发出任何 SQL。

请参见

联接式预加载 - 在 关系加载技术 中

### 显式连接 + 急切加载

如果我们在连接到`user_account`表时加载`Address`行，使用诸如`Select.join()`之类的方法来渲染连接，我们还可以利用该连接以便在每个返回的`Address`对象上急切加载`Address.user`属性的内容。这本质上是我们正在使用“联接式预加载”，但是自己渲染连接。通过使用`contains_eager()`选项实现了这种常见用例。该选项与`joinedload()`非常相似，只是它假设我们已经自己设置了连接，并且它仅指示应该将 COLUMNS 子句中的其他列加载到每个返回对象的相关属性中，例如：

```py
>>> from sqlalchemy.orm import contains_eager
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "pkrabs")
...     .options(contains_eager(Address.user))
...     .order_by(Address.id)
... )
>>> for row in session.execute(stmt):
...     print(f"{row.Address.email_address} {row.Address.user.name}")
SELECT  user_account.id,  user_account.name,  user_account.fullname,
address.id  AS  id_1,  address.email_address,  address.user_id
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  ?  ORDER  BY  address.id
[...]  ('pkrabs',)
pearl.krabs@gmail.com pkrabs
pearl@aol.com pkrabs
```

在上述示例中，我们同时对 `user_account.name` 进行了行过滤，并将 `user_account` 的行加载到返回行的 `Address.user` 属性中。如果我们分别应用了 `joinedload()`，我们会得到一个不必要地两次连接的 SQL 查询：

```py
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "pkrabs")
...     .options(joinedload(Address.user))
...     .order_by(Address.id)
... )
>>> print(stmt)  # SELECT has a JOIN and LEFT OUTER JOIN unnecessarily
SELECT  address.id,  address.email_address,  address.user_id,
user_account_1.id  AS  id_1,  user_account_1.name,  user_account_1.fullname
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
LEFT  OUTER  JOIN  user_account  AS  user_account_1  ON  user_account_1.id  =  address.user_id
WHERE  user_account.name  =  :name_1  ORDER  BY  address.id 
```

另请参阅

关系加载技术 中的两个部分：

+   急切加载的禅意 - 详细描述了上述问题

+   将显式连接/语句路由到急切加载的集合中 - 使用 `contains_eager()`

### Raiseload

还值得一提的一种额外的加载策略是 `raiseload()`。该选项用于通过使通常会产生惰性加载的操作引发错误来完全阻止应用程序出现 N 加一 问题。它有两种变体，通过 `raiseload.sql_only` 选项进行控制，以阻止需要 SQL 的惰性加载，或者包括那些只需查询当前 `Session` 的“加载”操作。

使用 `raiseload()` 的一种方法是在 `relationship()` 上直接配置它，通过将 `relationship.lazy` 设置为值 `"raise_on_sql"`，这样对于特定映射，某个关系将永远不会尝试发出 SQL：

```py
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import relationship

>>> class User(Base):
...     __tablename__ = "user_account"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     addresses: Mapped[List["Address"]] = relationship(
...         back_populates="user", lazy="raise_on_sql"
...     )

>>> class Address(Base):
...     __tablename__ = "address"
...     id: Mapped[int] = mapped_column(primary_key=True)
...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...     user: Mapped["User"] = relationship(back_populates="addresses", lazy="raise_on_sql")
```

使用这样的映射，应用程序被阻止惰性加载，指示特定查询需要指定加载策略：

```py
>>> u1 = session.execute(select(User)).scalars().first()
SELECT  user_account.id  FROM  user_account
[...]  ()
>>> u1.addresses
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'User.addresses' is not available due to lazy='raise_on_sql'
```

异常会指示应该立即加载此集合：

```py
>>> u1 = (
...     session.execute(select(User).options(selectinload(User.addresses)))
...     .scalars()
...     .first()
... )
SELECT  user_account.id
FROM  user_account
[...]  ()
SELECT  address.user_id  AS  address_user_id,  address.id  AS  address_id
FROM  address
WHERE  address.user_id  IN  (?,  ?,  ?,  ?,  ?,  ?)
[...]  (1,  2,  3,  4,  5,  6) 
```

`lazy="raise_on_sql"` 选项还尝试智能处理多对一关系；在上述示例中，如果 `Address` 对象的 `Address.user` 属性没有加载，但是该 `User` 对象在同一个 `Session` 中本地存在，则“raiseload”策略不会引发错误。

另请参阅

使用 raiseload 防止不必要的惰性加载 - 在 关系加载技术 中
