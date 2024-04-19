# ORM 快速入门

> 原文：[`docs.sqlalchemy.org/en/20/orm/quickstart.html`](https://docs.sqlalchemy.org/en/20/orm/quickstart.html)

对于想要快速了解基本 ORM 使用情况的新用户，这里提供了 SQLAlchemy 统一教程中使用的映射和示例的缩写形式。这里的代码可以从干净的命令行完全运行。

由于本节中的描述故意**非常简短**，请继续阅读完整的 SQLAlchemy 统一教程以获得对这里所说明的每个概念更深入的描述。

从 2.0 版本开始更改：ORM 快速入门已更新为最新的[**PEP 484**](https://peps.python.org/pep-0484/)兼容功能，使用包括`mapped_column()`在内的新构造。有关迁移信息，请参见 ORM 声明模型部分。

## 声明模型

在这里，我们定义模块级别的构造，这些构造将形成我们将从数据库查询的结构。这个结构被称为声明式映射，它一次定义了 Python 对象模型，以及描述存在或将存在于特定数据库中的真实 SQL 表的数据库元数据：

```py
>>> from typing import List
>>> from typing import Optional
>>> from sqlalchemy import ForeignKey
>>> from sqlalchemy import String
>>> from sqlalchemy.orm import DeclarativeBase
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship

>>> class Base(DeclarativeBase):
...     pass

>>> class User(Base):
...     __tablename__ = "user_account"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str] = mapped_column(String(30))
...     fullname: Mapped[Optional[str]]
...
...     addresses: Mapped[List["Address"]] = relationship(
...         back_populates="user", cascade="all, delete-orphan"
...     )
...
...     def __repr__(self) -> str:
...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

>>> class Address(Base):
...     __tablename__ = "address"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     email_address: Mapped[str]
...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...
...     user: Mapped["User"] = relationship(back_populates="addresses")
...
...     def __repr__(self) -> str:
...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
```

映射始于一个基类，这个基类上面称为`Base`，并且是通过对`DeclarativeBase`类进行简单子类化来创建的。

然后通过对`Base`进行子类化来创建单独的映射类。一个映射类通常指的是单个特定的数据库表，其名称通过使用`__tablename__`类级别属性指示。

接下来，通过添加包含称为`Mapped`的特殊类型注释的属性来声明表的一部分列。每个属性的名称对应于要成为数据库表的一部分的列。每个列的数据类型首先从与每个`Mapped`注释相关联的 Python 数据类型中获取；`int`用于`INTEGER`，`str`用于`VARCHAR`，等等。空值性取决于是否使用了`Optional[]`类型修饰符。可以使用右侧`mapped_column()`指令中的 SQLAlchemy 类型对象指示更具体的类型信息，例如上面在`User.name`列中使用的`String`数据类型。可以使用类型注释映射来自定义 Python 类型和 SQL 类型之间的关联。

`mapped_column()`指令用于所有需要更具体定制的基于列的属性。除了类型信息外，此指令还接受各种参数，指示有关数据库列的特定详细信息，包括服务器默认值和约束信息，例如在主键和外键中的成员资格。`mapped_column()`指令接受的参数是 SQLAlchemy `Column`类所接受的参数的一个超集，该类由 SQLAlchemy 核心用于表示数据库列。

所有 ORM 映射类都要求至少声明一个列作为主键的一部分，通常是通过在那些应该成为主键的`mapped_column()`对象上使用`Column.primary_key`参数来实现的。在上面的示例中，`User.id`和`Address.id`列被标记为主键。

综合考虑，字符串表名称以及列声明列表的组合在 SQLAlchemy 中被称为 table metadata。在 SQLAlchemy 统一教程的处理数据库元数据中介绍了如何使用核心和 ORM 方法设置表元数据。上述映射是所谓的注释声明表配置的示例。

`Mapped` 的其他变体可用，最常见的是上面指示的 `relationship()` 构造。与基于列的属性相比，`relationship()` 表示两个 ORM 类之间的关联。在上面的示例中，`User.addresses` 将 `User` 和 `Address` 连接起来，`Address.user` 将 `Address` 和 `User` 连接起来。`relationship()` 构造介绍于 SQLAlchemy 统一教程 的 处理 ORM 相关对象 部分。

最后，上面的示例类包括一个 `__repr__()` 方法，这并非必需，但对调试很有用。映射类可以使用诸如 `__repr__()` 之类的方法自动生成，使用数据类。有关数据类映射的更多信息，请参阅 声明式数据类映射。

## 创建一个引擎

`Engine` 是一个**工厂**，可以为我们创建新的数据库连接，还在 连接池 中保存连接以便快速重用。出于学习目的，我们通常使用一个 SQLite 内存数据库方便起见：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine("sqlite://", echo=True)
```

小贴士

`echo=True` 参数表示连接发出的 SQL 将被记录到标准输出。

`Engine` 的完整介绍从 建立连接 - 引擎 开始。

## 发出 CREATE TABLE DDL

利用我们的表格元数据和引擎，我们可以一次性在目标 SQLite 数据库中生成我们的模式，使用的方法是 `MetaData.create_all()`：

```py
>>> Base.metadata.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
CREATE  TABLE  user_account  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30)  NOT  NULL,
  fullname  VARCHAR,
  PRIMARY  KEY  (id)
)
...
CREATE  TABLE  address  (
  id  INTEGER  NOT  NULL,
  email_address  VARCHAR  NOT  NULL,
  user_id  INTEGER  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(user_id)  REFERENCES  user_account  (id)
)
...
COMMIT 
```

我们刚刚写的那小段 Python 代码发生了很多事情。要完整了解表格元数据的情况，请在教程中继续阅读 处理数据库元数据 部分。

## 创建对象并持久化

现在我们已经准备好向数据库插入数据了。我们通过创建`User`和`Address`类的实例来实现这一目标，这些类已经通过声明性映射过程自动建立了`__init__()`方法。然后，我们使用一个名为 Session 的对象将它们传递给数据库，该对象利用`Engine`与数据库进行交互。这里使用了`Session.add_all()`方法一次添加多个对象，并且`Session.commit()`方法将被用来提交数据库中的任何挂起更改，然后提交当前的数据库事务，无论何时使用`Session`时，该事务始终处于进行中：

```py
>>> from sqlalchemy.orm import Session

>>> with Session(engine) as session:
...     spongebob = User(
...         name="spongebob",
...         fullname="Spongebob Squarepants",
...         addresses=[Address(email_address="spongebob@sqlalchemy.org")],
...     )
...     sandy = User(
...         name="sandy",
...         fullname="Sandy Cheeks",
...         addresses=[
...             Address(email_address="sandy@sqlalchemy.org"),
...             Address(email_address="sandy@squirrelpower.org"),
...         ],
...     )
...     patrick = User(name="patrick", fullname="Patrick Star")
...
...     session.add_all([spongebob, sandy, patrick])
...
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...]  ('spongebob',  'Spongebob Squarepants')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...]  ('sandy',  'Sandy Cheeks')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...]  ('patrick',  'Patrick Star')
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...]  ('spongebob@sqlalchemy.org',  1)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...]  ('sandy@sqlalchemy.org',  2)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...]  ('sandy@squirrelpower.org',  2)
COMMIT 
```

提示

建议以上述上下文管理器风格使用`Session`，即使用 Python 的 `with:` 语句。`Session` 对象代表了活动的数据库资源，因此确保在完成一系列操作时将其关闭是很好的。在下一节中，我们将保持一个`Session`仅用于说明目的。

关于创建`Session`的基础知识请参见使用 ORM Session 执行，更多内容请查看使用 Session 的基础知识。

然后，在使用 ORM 工作单元模式插入行中介绍了一些基本持久性操作的变体。

## 简单的 SELECT

在数据库中有一些行之后，这是发出 SELECT 语句以加载一些对象的最简单形式。要创建 SELECT 语句，我们使用 `select()` 函数创建一个新的 `Select` 对象，然后使用一个 `Session` 调用它。在查询 ORM 对象时经常有用的方法是 `Session.scalars()` 方法，它将返回一个 `ScalarResult` 对象，该对象将遍历我们已选择的 ORM 对象：

```py
>>> from sqlalchemy import select

>>> session = Session(engine)

>>> stmt = select(User).where(User.name.in_(["spongebob", "sandy"]))

>>> for user in session.scalars(stmt):
...     print(user)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  IN  (?,  ?)
[...]  ('spongebob',  'sandy')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
```

上述查询还使用了 `Select.where()` 方法添加 WHERE 条件，并且还使用了 SQLAlchemy 类似列的构造中的 `ColumnOperators.in_()` 方法来使用 SQL IN 操作符。

有关如何选择对象和单独列的更多细节请参见选择 ORM 实体和列。

## 使用 JOIN 进行 SELECT

在一次性查询多个表格是非常常见的，在 SQL 中，JOIN 关键字是这种情况的主要方式。`Select` 构造使用 `Select.join()` 方法创建连接：

```py
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "sandy")
...     .where(Address.email_address == "sandy@sqlalchemy.org")
... )
>>> sandy_address = session.scalars(stmt).one()
SELECT  address.id,  address.email_address,  address.user_id
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  ?  AND  address.email_address  =  ?
[...]  ('sandy',  'sandy@sqlalchemy.org')
>>> sandy_address
Address(id=2, email_address='sandy@sqlalchemy.org')
```

上述查询演示了多个 WHERE 条件的使用，这些条件会自动使用 AND 进行链接，以及如何使用 SQLAlchemy 类似列对象创建“相等性”比较，这使用了重写的 Python 方法 `ColumnOperators.__eq__()` 来生成 SQL 条件对象。

有关上述概念的更多背景信息在 WHERE 子句和明确的 FROM 子句和 JOIN 处。

## 进行更改

`Session`对象与我们的 ORM 映射类`User`和`Address`结合使用，自动跟踪对对象的更改，这些更改将在下次`Session` flush 时生成 SQL 语句。 在下面，我们更改了与“sandy”关联的一个电子邮件地址，并在发出 SELECT 以检索“patrick”的行后向“patrick”添加了一个新的电子邮件地址：

```py
>>> stmt = select(User).where(User.name == "patrick")
>>> patrick = session.scalars(stmt).one()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('patrick',)
>>> patrick.addresses.append(Address(email_address="patrickstar@sqlalchemy.org"))
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,  address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (3,)
>>> sandy_address.email_address = "sandy_cheeks@sqlalchemy.org"

>>> session.commit()
UPDATE  address  SET  email_address=?  WHERE  address.id  =  ?
[...]  ('sandy_cheeks@sqlalchemy.org',  2)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)
[...]  ('patrickstar@sqlalchemy.org',  3)
COMMIT 
```

注意当我们访问`patrick.addresses`时，会发出一个 SELECT。 这称为延迟加载。 关于使用更多或更少 SQL 访问相关项目的不同方式的背景介绍在加载策略中引入。

有关 ORM 数据操作的详细说明始于使用 ORM 进行数据操作。

## 一些删除

一切都必须有个了结，就像我们的一些数据库行一样 - 这里是两种不同形式的删除的快速演示，这两种删除根据特定用例的不同而重要。

首先，我们将从`sandy`用户中删除一个`Address`对象。 当`Session`下次 flush 时，这将导致该行被删除。 此行为是我们在映射中配置的称为删除级联的东西。 我们可以使用`Session.get()`按主键获取`sandy`对象的句柄，然后使用该对象：

```py
>>> sandy = session.get(User, 2)
BEGIN  (implicit)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,  user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (2,)
>>> sandy.addresses.remove(sandy_address)
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,  address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (2,) 
```

上面的最后一个 SELECT 是延迟加载操作进行，以便加载`sandy.addresses`集合，以便我们可以删除`sandy_address`成员。有其他方法可以完成这一系列操作，这些方法不会生成太多的 SQL。

我们可以选择发出 DELETE SQL，以删除到目前为止已更改的内容，而不提交事务，使用`Session.flush()`方法：

```py
>>> session.flush()
DELETE  FROM  address  WHERE  address.id  =  ?
[...]  (2,) 
```

接下来，我们将完全删除“patrick”用户。 对于对象本身的顶级删除，我们使用`Session.delete()`方法； 此方法实际上不执行删除，而是设置对象将在下次 flush 时被删除。 该操作还将根据我们配置的级联选项级联到相关对象，本例中为相关的`Address`对象：

```py
>>> session.delete(patrick)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,  user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (3,)
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,  address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (3,) 
```

在这种特殊情况下，`Session.delete()`方法发出了两个 SELECT 语句，即使它没有发出 DELETE，这可能看起来令人惊讶。这是因为当该方法去检查对象时，发现`patrick`对象已经过期，这是在我们上次调用`Session.commit()`时发生的，发出的 SQL 是为了重新从新事务加载行。这种过期是可选的，并且在正常使用中，我们经常会在不适用的情况下关闭它。

为了说明被删除的行，这里是提交：

```py
>>> session.commit()
DELETE  FROM  address  WHERE  address.id  =  ?
[...]  (4,)
DELETE  FROM  user_account  WHERE  user_account.id  =  ?
[...]  (3,)
COMMIT 
```

教程讨论了 ORM 删除，详见使用工作单元模式删除 ORM 对象。对象过期的背景信息在过期/刷新；级联在级联中进行了深入讨论。

## 深入学习上述概念

对于新用户来说，上面的部分可能是一个快速浏览。上面的每一步中都有许多重要的概念没有涵盖到。通过快速了解事物的外观，建议通过 SQLAlchemy 统一教程逐步学习，以获得对上面所发生的事物的坚实的工作知识。祝你好运！

## 声明模型

在这里，我们定义了将构成我们从数据库查询的模块级构造。这个结构被称为声明性映射，它一次定义了 Python 对象模型以及描述真实 SQL 表的数据库元数据，这些表存在或将存在于特定数据库中：

```py
>>> from typing import List
>>> from typing import Optional
>>> from sqlalchemy import ForeignKey
>>> from sqlalchemy import String
>>> from sqlalchemy.orm import DeclarativeBase
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship

>>> class Base(DeclarativeBase):
...     pass

>>> class User(Base):
...     __tablename__ = "user_account"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str] = mapped_column(String(30))
...     fullname: Mapped[Optional[str]]
...
...     addresses: Mapped[List["Address"]] = relationship(
...         back_populates="user", cascade="all, delete-orphan"
...     )
...
...     def __repr__(self) -> str:
...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

>>> class Address(Base):
...     __tablename__ = "address"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     email_address: Mapped[str]
...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
...
...     user: Mapped["User"] = relationship(back_populates="addresses")
...
...     def __repr__(self) -> str:
...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
```

映射始于一个基类，上面称为`Base`，通过对`DeclarativeBase`类进行简单的子类化来创建。

通过对`Base`进行子类化，然后创建个体映射类。一个映射类通常指的是一个特定的数据库表，其名称是通过使用`__tablename__`类级属性指示的。

接下来，声明表中的列，通过添加包含一个特殊的类型注释称为`Mapped`的属性来实现。每个属性的名称对应于要成为数据库表的列。每个列的数据类型首先取自与每个`Mapped`注释相关联的 Python 数据类型；对于 `INTEGER` 使用 `int`，对于 `VARCHAR` 使用 `str` 等。可选性取决于是否使用了 `Optional[]` 类型修饰符。可以使用右侧的 SQLAlchemy 类型对象指示更具体的类型信息，例如上面在 `User.name` 列中使用的 `String` 数据类型。Python 类型和 SQL 类型之间的关联可以使用 type annotation map 进行定制。

`mapped_column()` 指令用于所有需要更具体定制的基于列的属性。除了类型信息外，该指令还接受各种参数，指示有关数据库列的特定细节，包括服务器默认值和约束信息，例如主键和外键的成员资格。`mapped_column()` 指令接受了 SQLAlchemy `Column` 类接受的参数的超集，该类由 SQLAlchemy Core 用于表示数据库列。

所有的 ORM 映射类都需要至少声明一个列作为主键的一部分，通常是通过在应该成为键的那些`mapped_column()`对象上使用`Column.primary_key`参数来实现的。在上面的示例中，`User.id` 和 `Address.id` 列被标记为主键。

综合起来，SQLAlchemy 中一个字符串表名和列声明列表的组合被称为 table metadata。在 SQLAlchemy 统一教程中介绍了使用 Core 和 ORM 方法设置表元数据的方法，在 Working with Database Metadata 章节中。上述映射是 Annotated Declarative Table 配置的示例。

还有其他`Mapped`的变体可用，最常见的是上面指示的`relationship()`构造。与基于列的属性相反，`relationship()`表示两个 ORM 类之间的链接。在上面的示例中，`User.addresses` 将`User`链接到`Address`，`Address.user` 将`Address`链接到`User`。`relationship()`构造在 SQLAlchemy 统一教程中的使用 ORM 相关对象中进行介绍。

最后，上面的示例类包括一个 `__repr__()` 方法，虽然不是必需的，但对于调试很有用。映射类可以使用诸如 `__repr__()` 这样的方法自动生成，使用数据类。有关数据类映射的更多信息，请参阅声明性数据类映射。

## 创建引擎

`Engine`是一个能够为我们创建新数据库连接的**工厂**，它还将连接保留在连接池中以供快速重用。出于学习目的，我们通常使用 SQLite 内存数据库以方便起见：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine("sqlite://", echo=True)
```

提示

`echo=True` 参数表示连接发出的 SQL 将被记录到标准输出。

对`Engine`的全面介绍始于建立连接 - 引擎。

## 发出 CREATE TABLE DDL

使用我们的表元数据和引擎，我们可以一次性在目标 SQLite 数据库中生成我们的模式，使用一种叫做`MetaData.create_all()`的方法：

```py
>>> Base.metadata.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
CREATE  TABLE  user_account  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30)  NOT  NULL,
  fullname  VARCHAR,
  PRIMARY  KEY  (id)
)
...
CREATE  TABLE  address  (
  id  INTEGER  NOT  NULL,
  email_address  VARCHAR  NOT  NULL,
  user_id  INTEGER  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(user_id)  REFERENCES  user_account  (id)
)
...
COMMIT 
```

刚才我们编写的那段 Python 代码发生了很多事情。要完整了解表元数据的情况，请参阅使用数据库元数据中的教程。

## 创建对象并持久化

我们现在可以将数据插入到数据库中了。我们通过创建`User`和`Address`类的实例来实现这一点，这些类已经通过声明映射过程自动创建了`__init__()`方法。然后，我们使用一个称为 Session 的对象将它们传递给数据库，该对象使用`Engine`与数据库进行交互。这里使用`Session.add_all()`方法一次添加多个对象，并且将使用`Session.commit()`方法刷新数据库中的任何待处理更改，然后提交当前的数据库事务，该事务始终在使用`Session`时处于进行中：

```py
>>> from sqlalchemy.orm import Session

>>> with Session(engine) as session:
...     spongebob = User(
...         name="spongebob",
...         fullname="Spongebob Squarepants",
...         addresses=[Address(email_address="spongebob@sqlalchemy.org")],
...     )
...     sandy = User(
...         name="sandy",
...         fullname="Sandy Cheeks",
...         addresses=[
...             Address(email_address="sandy@sqlalchemy.org"),
...             Address(email_address="sandy@squirrelpower.org"),
...         ],
...     )
...     patrick = User(name="patrick", fullname="Patrick Star")
...
...     session.add_all([spongebob, sandy, patrick])
...
...     session.commit()
BEGIN  (implicit)
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...]  ('spongebob',  'Spongebob Squarepants')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...]  ('sandy',  'Sandy Cheeks')
INSERT  INTO  user_account  (name,  fullname)  VALUES  (?,  ?)  RETURNING  id
[...]  ('patrick',  'Patrick Star')
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...]  ('spongebob@sqlalchemy.org',  1)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...]  ('sandy@sqlalchemy.org',  2)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)  RETURNING  id
[...]  ('sandy@squirrelpower.org',  2)
COMMIT 
```

提示

建议像上面那样使用 Python 的 `with:` 语句，即使用上下文管理器样式使用`Session`。 `Session` 对象代表着活跃的数据库资源，所以当一系列操作完成时，确保关闭它是很好的。在下一节中，我们将保持`Session`处于打开状态，仅用于说明目的。

创建`Session`的基础知识请参考使用 ORM Session 执行，更多内容请参考使用 Session 的基础知识。

接下来，介绍了一些基本持久化操作的变体，请参阅使用 ORM 工作单元模式插入行。

## 简单的 SELECT

在数据库中有一些行时，这是发出 SELECT 语句以加载一些对象的最简单形式。要创建 SELECT 语句，我们使用`select()` 函数创建一个新的`Select` 对象，然后使用`Session` 调用它。查询 ORM 对象时经常有用的方法是`Session.scalars()` 方法，它将返回一个`ScalarResult` 对象，该对象将迭代我们选择的 ORM 对象：

```py
>>> from sqlalchemy import select

>>> session = Session(engine)

>>> stmt = select(User).where(User.name.in_(["spongebob", "sandy"]))

>>> for user in session.scalars(stmt):
...     print(user)
BEGIN  (implicit)
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  IN  (?,  ?)
[...]  ('spongebob',  'sandy')
User(id=1, name='spongebob', fullname='Spongebob Squarepants')
User(id=2, name='sandy', fullname='Sandy Cheeks')
```

上述查询还使用了`Select.where()` 方法添加 WHERE 条件，并且还使用了所有 SQLAlchemy 列对象的一部分的`ColumnOperators.in_()` 方法来使用 SQL IN 操作符。

如何选择对象和单独列的更多详细信息请参阅选择 ORM 实体和列。

## 使用 JOIN 的 SELECT

在 SQL 中，一次查询多个表是非常常见的，而 JOIN 关键字是实现这一目的的主要方法。`Select` 构造函数使用`Select.join()` 方法创建连接：

```py
>>> stmt = (
...     select(Address)
...     .join(Address.user)
...     .where(User.name == "sandy")
...     .where(Address.email_address == "sandy@sqlalchemy.org")
... )
>>> sandy_address = session.scalars(stmt).one()
SELECT  address.id,  address.email_address,  address.user_id
FROM  address  JOIN  user_account  ON  user_account.id  =  address.user_id
WHERE  user_account.name  =  ?  AND  address.email_address  =  ?
[...]  ('sandy',  'sandy@sqlalchemy.org')
>>> sandy_address
Address(id=2, email_address='sandy@sqlalchemy.org')
```

上述查询示例说明了多个 WHERE 条件如何自动使用 AND 连接，并且展示了如何使用 SQLAlchemy 列对象创建“相等性”比较，该比较使用了重载的 Python 方法`ColumnOperators.__eq__()`来生成 SQL 条件对象。

以上概念的更多背景可在 WHERE 子句和显式 FROM 子句和 JOIN 处找到。

## 进行更改

`Session` 对象与我们的 ORM 映射类 `User` 和 `Address` 一起，会自动跟踪对象的更改，这些更改会导致 SQL 语句在下次 `Session` 刷新时被发出。下面，我们更改了与“sandy”关联的一个电子邮件地址，并在发出 SELECT 以检索“patrick”的行之后，向“patrick”添加了一个新的电子邮件地址：

```py
>>> stmt = select(User).where(User.name == "patrick")
>>> patrick = session.scalars(stmt).one()
SELECT  user_account.id,  user_account.name,  user_account.fullname
FROM  user_account
WHERE  user_account.name  =  ?
[...]  ('patrick',)
>>> patrick.addresses.append(Address(email_address="patrickstar@sqlalchemy.org"))
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,  address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (3,)
>>> sandy_address.email_address = "sandy_cheeks@sqlalchemy.org"

>>> session.commit()
UPDATE  address  SET  email_address=?  WHERE  address.id  =  ?
[...]  ('sandy_cheeks@sqlalchemy.org',  2)
INSERT  INTO  address  (email_address,  user_id)  VALUES  (?,  ?)
[...]  ('patrickstar@sqlalchemy.org',  3)
COMMIT 
```

注意当我们访问`patrick.addresses`时，会发出一个 SELECT。这被称为延迟加载。有关使用更多或更少的 SQL 访问相关项目的不同方法的背景介绍，请参阅加载器策略。

有关使用 ORM 进行数据操作的详细说明，请参阅 ORM 数据操作。

## 一些删除操作

万物都有尽头，就像我们的一些数据库行一样 - 这里快速演示了两种不同形式的删除，根据特定用例的重要性而定。

首先，我们将从`sandy`用户中删除一个`Address`对象。当`Session`下次刷新时，这将导致该行被删除。这种行为是我们在映射中配置的，称为级联删除。我们可以使用 `Session.get()` 按主键获取到`sandy`对象，然后操作该对象：

```py
>>> sandy = session.get(User, 2)
BEGIN  (implicit)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,  user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (2,)
>>> sandy.addresses.remove(sandy_address)
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,  address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (2,) 
```

上面的最后一个 SELECT 是为了进行延迟加载 操作，以便加载`sandy.addresses`集合，以便我们可以删除`sandy_address`成员。还有其他方法可以执行这一系列操作，不会发出太多的 SQL。

我们可以选择发出针对到目前为止被更改的 DELETE SQL，而不提交事务，使用 `Session.flush()` 方法：

```py
>>> session.flush()
DELETE  FROM  address  WHERE  address.id  =  ?
[...]  (2,) 
```

接下来，我们将完全删除“patrick”用户。对于对象的顶级删除，我们使用`Session.delete()`方法；这个方法实际上并不执行删除操作，而是设置对象在下一次刷新时将被删除。该操作还会根据我们配置的级联选项级联到相关对象，本例中是关联的`Address`对象：

```py
>>> session.delete(patrick)
SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name,  user_account.fullname  AS  user_account_fullname
FROM  user_account
WHERE  user_account.id  =  ?
[...]  (3,)
SELECT  address.id  AS  address_id,  address.email_address  AS  address_email_address,  address.user_id  AS  address_user_id
FROM  address
WHERE  ?  =  address.user_id
[...]  (3,) 
```

在这种特殊情况下，`Session.delete()`方法发出了两个 SELECT 语句，即使它没有发出 DELETE，这可能看起来令人惊讶。这是因为当方法检查对象时，发现`patrick`对象已经过期，这是在我们上次调用`Session.commit()`时发生的，发出的 SQL 是为了从新事务重新加载行。这种过期是可选的，在正常使用中，我们通常会在不适用的情况下关闭它。

要说明被删除的行，请看这个提交：

```py
>>> session.commit()
DELETE  FROM  address  WHERE  address.id  =  ?
[...]  (4,)
DELETE  FROM  user_account  WHERE  user_account.id  =  ?
[...]  (3,)
COMMIT 
```

本教程讨论了 ORM 删除操作，详情请见使用工作单元模式删除 ORM 对象。关于对象过期的背景信息请参考过期/刷新；级联操作在 Cascades 中有详细讨论。

## 深入学习上述概念

对于新用户来说，上述部分可能是一场令人眼花缭乱的旅程。每个步骤中都有许多重要概念没有涵盖。快速了解事物的外观后，建议通过 SQLAlchemy 统一教程来深入了解上述内容。祝好运！
