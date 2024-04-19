# 处理数据库元数据

> 原文：[`docs.sqlalchemy.org/en/20/tutorial/metadata.html`](https://docs.sqlalchemy.org/en/20/tutorial/metadata.html)

随着引擎和 SQL 执行完成，我们准备开始一些 Alchemy。SQLAlchemy Core 和 ORM 的核心元素是 SQL 表达语言，它允许流畅、可组合地构建 SQL 查询。这些查询的基础是代表数据库概念（如表和列）的 Python 对象。这些对象被统称为数据库元数据。

SQLAlchemy 中数据库元数据的最常见基础对象称为`MetaData`、`Table`和`Column`。下面的部分将说明这些对象在 Core 导向风格和 ORM 导向风格中的使用方式。

**ORM 读者，请继续关注！**

与其他部分一样，Core 用户可以跳过 ORM 部分，但 ORM 用户最好从两个角度熟悉这些对象。这里讨论的`Table`对象在使用 ORM 时以一种更间接的方式（也是完全 Python 类型化的方式）声明，然而，在 ORM 的配置中仍然有一个`Table`对象。

## 使用表对象设置元数据

当我们使用关系型数据库时，数据库中的基本数据保存结构，我们从中查询的结构称为**表**。在 SQLAlchemy 中，数据库“表”最终由一个名为`Table`的 Python 对象表示。

要开始使用 SQLAlchemy 表达语言，我们需要构建`Table`对象，这些对象表示我们有兴趣使用的所有数据库表。 `Table`是通过编程方式构建的，可以直接使用`Table`构造函数，也可以间接地使用 ORM 映射类（稍后在使用 ORM 声明形式定义表元数据中描述）。还有一种选项可以从现有数据库加载一些或全部表信息，称为反射。

无论使用哪种方法，我们始终从一个集合开始，这个集合将是我们放置表的地方，称为 `MetaData` 对象。这个对象本质上是一个围绕 Python 字典的 外观，该字典存储了一系列以其字符串名称为键的 `Table` 对象。虽然 ORM 在获取这个集合的位置上提供了一些选项，但我们始终可以选择直接创建一个，看起来像这样：

```py
>>> from sqlalchemy import MetaData
>>> metadata_obj = MetaData()
```

一旦我们有了 `MetaData` 对象，我们可以声明一些 `Table` 对象。本教程将从经典的 SQLAlchemy 教程模型开始，其中有一个名为 `user_account` 的表，存储着网站的用户，以及一个相关的 `address` 表，存储着与 `user_account` 表中的行相关联的电子邮件地址。当完全不使用 ORM Declarative 模型时，我们直接构造每个 `Table` 对象，通常将每个对象分配给一个变量，这将是我们在应用程序代码中引用表的方式：

```py
>>> from sqlalchemy import Table, Column, Integer, String
>>> user_table = Table(
...     "user_account",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30)),
...     Column("fullname", String),
... )
```

有了上面的例子，当我们希望编写引用数据库中 `user_account` 表的代码时，我们将使用 `user_table` Python 变量来引用它。

### `Table` 的组件

我们可以观察到，Python 中的 `Table` 构造与 SQL CREATE TABLE 语句相似；从表名开始，然后列出每个列，其中每个列都有一个名称和一个数据类型。我们上面使用的对象是：

+   `Table` - 表示数据库表并将自己分配给 `MetaData` 集合。

+   `Column` - 表示数据库表中的列，并将自己分配给 `Table` 对象。`Column` 通常包括一个字符串名称和一个类型对象。以父 `Table` 的 `Column` 对象的集合通常通过位于 `Table.c` 的关联数组来访问：

    ```py
    >>> user_table.c.name
    Column('name', String(length=30), table=<user_account>)

    >>> user_table.c.keys()
    ['id', 'name', 'fullname']
    ```

+   `Integer`，`String` - 这些类表示 SQL 数据类型，并且可以被传递给具有或没有必要被实例化的`Column`。在上面的例子中，我们想要给“name”列一个长度为“30”，因此我们实例化了`String(30)`。但对于“id”和“fullname”，我们没有指定这些，所以我们可以发送类本身。

另请参阅

`MetaData`，`Table`和`Column`的参考和 API 文档位于用 MetaData 描述数据库。数据类型的参考文档位于 SQL 数据类型对象。

在接下来的一节中，我们将说明`Table`的一个基本功能，即在特定数据库连接上生成 DDL。但首先，我们将声明第二个`Table`。

### 声明简单约束

示例中的第一个`Column`在`user_table`中包含`Column.primary_key`参数，这是一种简写技术，表示这个`Column`应该是这个表的主键的一部分。主键本身通常是隐式声明的，并且由`PrimaryKeyConstraint`构造表示，我们可以在`Table`对象的`Table.primary_key`属性上看到它：

```py
>>> user_table.primary_key
PrimaryKeyConstraint(Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False))
```

最常显式声明的约束是对应于数据库外键约束的`ForeignKeyConstraint`对象。当我们声明相互关联的表时，SQLAlchemy 不仅使用这些外键约束声明在向数据库发送 CREATE 语句时将其发送出去，而且还用于帮助构造 SQL 表达式。

一个涉及目标表上仅一个列的`ForeignKeyConstraint`通常使用列级别的速记符号通过`ForeignKey`对象声明。下面我们声明了一个将具有引用`user`表的外键约束的第二个表`address`：

```py
>>> from sqlalchemy import ForeignKey
>>> address_table = Table(
...     "address",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("user_id", ForeignKey("user_account.id"), nullable=False),
...     Column("email_address", String, nullable=False),
... )
```

上面的表还包含了第三种约束类型，在 SQL 中是“NOT NULL”约束，在上面使用`Column.nullable`参数进行指示。

提示

在`Column`定义中使用`ForeignKey`对象时，我们可以省略该`Column`的数据类型；它会自动从相关列的数据类型中推断出来，在上面的示例中是`user_account.id`列的`Integer`数据类型。

在下一节中，我们将发出`user`和`address`表的完整 DDL 以查看完成的结果。

### 发出 DDL 到数据库

我们已经构建了一个对象结构，代表数据库中的两个数据库表，在根`MetaData`对象开始，然后进入两个`Table`对象，每个对象都包含一组`Column`和`Constraint`对象。这个对象结构将成为我们今后使用 Core 和 ORM 执行的大多数操作的中心。

我们可以对此结构进行的第一个有用的操作是发出 CREATE TABLE 语句，或者 DDL 到我们的 SQLite 数据库，以便我们可以从中插入和查询数据。我们已经拥有完成此操作所需的所有工具，通过在我们的`MetaData`上调用`MetaData.create_all()`方法，将目标数据库的`Engine`传递给它：

```py
>>> metadata_obj.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
CREATE  TABLE  user_account  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30),
  fullname  VARCHAR,
  PRIMARY  KEY  (id)
)
...
CREATE  TABLE  address  (
  id  INTEGER  NOT  NULL,
  user_id  INTEGER  NOT  NULL,
  email_address  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(user_id)  REFERENCES  user_account  (id)
)
...
COMMIT 
```

以上的 DDL 创建过程包括一些 SQLite 特定的 PRAGMA 语句，用于在发出 CREATE 之前测试每个表的存在性。所有步骤也包含在 BEGIN/COMMIT 对中，以适应事务性 DDL。

create 进程还负责按正确顺序发出 CREATE 语句；以上，FOREIGN KEY 约束依赖于 `user` 表的存在，因此 `address` 表在第二创建。在更复杂的依赖情况下，FOREIGN KEY 约束也可能使用 ALTER 在表创建后进行应用。

`MetaData` 对象还具有一个 `MetaData.drop_all()` 方法，它将按照与发出 CREATE 相反的顺序发出 DROP 语句以删除模式元素。## 使用 ORM 声明式表单定义表元数据

在使用 ORM 时，声明 `Table` 元数据的过程通常与声明 映射 类的过程结合在一起。映射类是我们想要创建的任何 Python 类，然后它将具有链接到数据库表中列的属性。虽然有几种实现方式，但最常见的风格称为 声明式，它允许我们一次声明我们的用户定义类和 `Table` 元数据。

### 建立声明性基础

在使用 ORM 时，`MetaData` 集合仍然存在，但它本身与一个仅用于 ORM 的构造关联，通常称为 **声明式基础**。获取新的声明式基础的最简便方法是创建一个继承 SQLAlchemy `DeclarativeBase` 类的新类：

```py
>>> from sqlalchemy.orm import DeclarativeBase
>>> class Base(DeclarativeBase):
...     pass
```

以上，`Base` 类就是我们将称为声明式基础的类。当我们创建新的类作为 `Base` 的子类时，并结合适当的类级指令，它们将在类创建时各自作为一个新的 **ORM 映射类** 建立，每个类通常（但不一定）引用一个特定的 `Table` 对象。

Declarative Base 指的是一个`MetaData`集合，它会自动为我们创建，假设我们没有从外部提供。这个`MetaData`集合可以通过`DeclarativeBase.metadata`类级别属性访问。当我们创建新的映射类时，它们将分别引用此`MetaData`集合内的一个`Table`：

```py
>>> Base.metadata
MetaData()
```

Declarative Base 还指的是一个称为`registry`的集合，它是 SQLAlchemy ORM 中的中央“映射器配置”单元。虽然很少直接访问，但该对象在映射器配置过程中是至关重要的，因为一组 ORM 映射类将通过此注册表相互协调。与`MetaData`的情况一样，我们的 Declarative Base 也为我们创建了一个`registry`（再次提供自己的`registry`的选项），我们可以通过`DeclarativeBase.registry`类变量访问它：

```py
>>> Base.registry
<sqlalchemy.orm.decl_api.registry object at 0x...>
```  ### 声明映射类

有了`Base`类的设立，我们现在可以根据新类`User`和`Address`定义`user_account`和`address`表的 ORM 映射类。我们下面展示了最现代化的 Declarative 形式，它是从[**PEP 484**](https://peps.python.org/pep-0484/)类型注解中驱动的，使用了一个特殊类型`Mapped`，它指示要映射为特定类型的属性：

```py
>>> from typing import List
>>> from typing import Optional
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship

>>> class User(Base):
...     __tablename__ = "user_account"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str] = mapped_column(String(30))
...     fullname: Mapped[Optional[str]]
...
...     addresses: Mapped[List["Address"]] = relationship(back_populates="user")
...
...     def __repr__(self) -> str:
...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

>>> class Address(Base):
...     __tablename__ = "address"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     email_address: Mapped[str]
...     user_id = mapped_column(ForeignKey("user_account.id"))
...
...     user: Mapped[User] = relationship(back_populates="addresses")
...
...     def __repr__(self) -> str:
...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
```

上面的两个类`User`和`Address`现在被称为**ORM 映射类**，并且可以在 ORM 持久性和查询操作中使用，稍后将对这些类的详细信息进行描述：

+   每个类都指向一个`Table`对象，该对象是作为声明性映射过程的一部分生成的，并通过将字符串赋值给`DeclarativeBase.__tablename__`属性命名。一旦类被创建，这个生成的`Table`可以通过`DeclarativeBase.__table__`属性进行访问。

+   如前所述，这种形式被称为声明性表配置。数种替代声明风格之一会让我们直接构建`Table`对象，并直接将其分配给`DeclarativeBase.__table__`。这种风格被称为声明性与命令式表配置。

+   为了指示`Table`中的列，我们使用`mapped_column()`构造，结合基于`Mapped`类型的类型注释。此对象将生成应用于`Table`构造的`Column`对象。

+   对于简单数据类型且没有其他选项的列，我们可以单独指定`Mapped`类型注释，使用简单的 Python 类型如`int`和`str`表示`Integer`和`String`。在声明性映射过程中，如何解释 Python 类型的定制化是非常开放的；请参阅使用注释的声明性表（用于`mapped_column()`的类型注释形式）和自定义类型映射部分了解背景知识。

+   根据`Optional[<typ>]`类型注释（或其等效形式，`<typ> | None`或`Union[<typ>, None]`）的存在，可以将列声明为“可空”或“非空”。还可以显式使用`mapped_column.nullable`参数（不必与注释的可选性匹配）。

+   使用显式类型注释是**完全可选的**。我们也可以在没有注释的情况下使用`mapped_column()`。在使用这种形式时，我们会根据需要在每个`mapped_column()`构造内使用更明确的类型对象，如`Integer`和`String`以及`nullable=False`。

+   另外两个属性，`User.addresses`和`Address.user`，定义了一种不同类型的属性，称为`relationship()`，它具有与注释相似的配置样式。`relationship()`构造在使用 ORM 相关对象中有更详细的讨论。

+   如果我们没有声明自己的`__init__()`方法，则会自动为类添加一个`__init__()`方法。该方法的默认形式接受所有属性名称作为可选关键字参数：

    ```py
    >>> sandy = User(name="sandy", fullname="Sandy Cheeks")
    ```

    要自动生成一个全功能的`__init__()`方法，既提供位置参数又提供具有默认关键字值的参数，可以使用在声明式数据类映射中引入的数据类功能。当然，始终可以选择使用显式的`__init__()`方法。

+   添加`__repr__()`方法是为了获得可读的字符串输出；这些方法不需要存在的要求。与`__init__()`一样，可以使用 dataclasses 功能自动生成`__repr__()`方法。

另请参阅

ORM 映射风格 - 不同 ORM 配置风格的完整背景。

声明式映射 - 声明式类映射概述

使用`mapped_column()`的声明式表 - 详细说明如何使用`mapped_column()`和`Mapped`来定义在使用声明式时要映射的`Table`中的列。

### 从 ORM 映射向数据库发出 DDL

由于我们的 ORM 映射类引用包含在`MetaData`集合中的`Table`对象，所以根据声明式基类发出 DDL 与在 Emitting DDL to the Database 中描述的过程相同。在我们的情况下，我们已经在我们的 SQLite 数据库中生成了`user`和`address`表。如果我们之前没有这样做，我们可以自由地利用与我们的 ORM 声明基类相关联的`MetaData`来做到这一点，方法是通过访问`DeclarativeBase.metadata`属性中的集合，然后像以前一样使用`MetaData.create_all()`。在这种情况下，会运行 PRAGMA 语句，但不会生成新表，因为已经发现它们已经存在：

```py
>>> Base.metadata.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
COMMIT 
```  ## 表反射

为了完成与表元数据一起工作的部分，我们将说明在该部分开头提到的另一个操作，即**表反射**。表反射是指通过读取数据库的当前状态来生成`Table`和相关对象的过程。而在之前的部分中，我们一直在 Python 中声明`Table`对象，然后有选择地将 DDL 发出到数据库以生成这样的模式，反射过程将这两个步骤反向执行，从现有数据库开始，并生成用于表示该数据库中模式的 Python 数据结构。

提示

并非要求必须使用反射才能将 SQLAlchemy 与现有数据库一起使用。完全可以在 Python 中显式声明所有元数据，使其结构与现有数据库相对应，这是很典型的。元数据结构也不必包含表、列或其他在本地应用程序中不需要的预先存在数据库中的约束和构造。

作为反射的示例，我们将创建一个新的`Table`对象，该对象表示我们在本文档的前几节中手动创建的`some_table`对象。这样做的方式有很多种，但最基本的方式是构建一个`Table`对象，给定表的名称和它将属于的`MetaData`集合，然后不是指示单独的`Column`和`Constraint`对象，而是传递目标`Engine`使用`Table.autoload_with`参数：

```py
>>> some_table = Table("some_table", metadata_obj, autoload_with=engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("some_table")
[raw  sql]  ()
SELECT  sql  FROM  (SELECT  *  FROM  sqlite_master  UNION  ALL  SELECT  *  FROM  sqlite_temp_master)  WHERE  name  =  ?  AND  type  in  ('table',  'view')
[raw  sql]  ('some_table',)
PRAGMA  main.foreign_key_list("some_table")
...
PRAGMA  main.index_list("some_table")
...
ROLLBACK 
```

在这个过程的结尾，`some_table`对象现在包含了表中存在的`Column`对象的信息，该对象可与我们明确声明的`Table`完全相同的方式使用：

```py
>>> some_table
Table('some_table', MetaData(),
 Column('x', INTEGER(), table=<some_table>),
 Column('y', INTEGER(), table=<some_table>),
 schema=None)
```

另请参阅

了解有关表和模式反射的更多信息，请参阅反射数据库对象。

对于 ORM 相关的表反射变体，在使用反射表声明映射一节中包含了可用选项的概述。

## 下一步

现在我们有一个 SQLite 数据库准备好使用，其中有两个表存在，并且我们可以使用`Connection`和/或 ORM `Session`通过 Core 和 ORM 表导向的构造与这些表进行交互。在接下来的章节中，我们将说明如何使用这些结构创建、操作和选择数据。

## 使用 Table 对象设置 MetaData

当我们使用关系型数据库时，数据库中我们查询的基本数据持有结构被称为**表**。在 SQLAlchemy 中，数据库中的“表”最终由一个名为`Table`的 Python 对象表示。

要开始使用 SQLAlchemy 表达式语言，我们将希望构建`Table`对象，这些对象代表我们有兴趣使用的所有数据库表。`Table`是以编程方式构建的，可以直接使用`Table`构造函数，也可以间接地使用 ORM 映射的类（稍后在使用 ORM 声明形式定义表元数据中描述）。还可以选择从现有数据库加载一些或所有表信息，称为反射。

无论采用哪种方法，我们始终从一个集合开始，这个集合将是我们放置表的地方，称为`MetaData`对象。这个对象本质上是一个 Python 字典的外观，它存储了一系列以它们的字符串名称为键的`Table`对象。虽然 ORM 提供了一些选项来获取此集合，但我们始终有直接制作一个的选择，看起来像这样：

```py
>>> from sqlalchemy import MetaData
>>> metadata_obj = MetaData()
```

一旦我们有了`MetaData`对象，我们就可以声明一些`Table`对象。本教程将从经典的 SQLAlchemy 教程模型开始，其中有一个名为`user_account`的表，该表存储网站的用户，以及一个相关的`address`表，该表存储与`user_account`表中的行关联的电子邮件地址。当完全不使用 ORM 声明模型时，我们直接构建每个`Table`对象，通常将每个分配给将在应用程序代码中引用表的变量： 

```py
>>> from sqlalchemy import Table, Column, Integer, String
>>> user_table = Table(
...     "user_account",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("name", String(30)),
...     Column("fullname", String),
... )
```

有了上面的示例，当我们希望编写引用数据库中`user_account`表的代码时，我们将使用`user_table` Python 变量来引用它。

### `Table`的组成部分

我们可以观察到，用 Python 编写的`Table`构造与 SQL CREATE TABLE 语句相似；从表名开始，然后列出每个列，每个列都有一个名称和一个数据类型。我们使用的对象包括：

+   `Table` - 表示数据库表并将自己分配给`MetaData`集合。

+   `Column` - 表示数据库表中的列，并将自身分配给`Table`对象。`Column`通常包括一个字符串名称和一个类型对象。关于父`Table`的`Column`对象的集合通常通过位于`Table.c`的关联数组进行访问：

    ```py
    >>> user_table.c.name
    Column('name', String(length=30), table=<user_account>)

    >>> user_table.c.keys()
    ['id', 'name', 'fullname']
    ```

+   `Integer`、`String` - 这些类表示 SQL 数据类型，可以带着或者不带着实例化传递给`Column`。在上面的例子中，我们想给“name”列指定长度为“30”，所以我们实例化了`String(30)`。但对于“id”和“fullname”，我们没有指定长度，所以我们可以直接发送类本身。

另请参阅

`MetaData`、`Table`和`Column`的参考文档和 API 文档在使用 MetaData 描述数据库。数据类型的参考文档在 SQL 数据类型对象。

在接下来的一节中，我们将说明`Table`的一个基本功能，即在特定的数据库连接上生成 DDL。但首先我们将声明第二个`Table`。

### 声明简单的约束

示例中`user_table`的第一个`Column`包括`Column.primary_key`参数，这是一种指示此`Column`应该成为该表主键的简写技术。主键本身通常是隐式声明的，并由`PrimaryKeyConstraint`构造表示，我们可以在`Table`对象的`Table.primary_key`属性中看到：

```py
>>> user_table.primary_key
PrimaryKeyConstraint(Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False))
```

最常明确声明的约束是与数据库外键约束对应的`ForeignKeyConstraint`对象。当我们声明相互关联的表时，SQLAlchemy 使用这些外键约束声明的存在，不仅在将它们发射到数据库的 CREATE 语句中，还用于辅助构建 SQL 表达式。

只涉及目标表上的单个列的`ForeignKeyConstraint`通常使用列级别的简写表示法通过`ForeignKey`对象声明。下面我们声明一个将引用`user`表的外键约束的第二个表`address`：

```py
>>> from sqlalchemy import ForeignKey
>>> address_table = Table(
...     "address",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("user_id", ForeignKey("user_account.id"), nullable=False),
...     Column("email_address", String, nullable=False),
... )
```

上述表还包含第三种约束，这在 SQL 中是“NOT NULL”约束，上面使用`Column.nullable`参数指示。

提示

在`Column`定义中使用`ForeignKey`对象时，我们可以省略该`Column`的数据类型；它将自动从相关列的数据类型推断出来，在上面的示例中为`user_account.id`列的`Integer`数据类型。

在下一节中，我们将发射完成的 DDL 到`user`和`address`表，以查看完成的结果。

### 发射 DDL 到数据库

我们构建了一个对象结构，表示数据库中的两个数据库表，从根`MetaData`对象开始，然后进入两个`Table`对象，每个对象都包含一组`Column`和`Constraint`对象。这个对象结构将成为我们今后在 Core 和 ORM 中执行大多数操作的核心。

我们可以使用这个结构的第一项有用的事情是发出 CREATE TABLE 语句，或者 DDL 到我们的 SQLite 数据库中，以便我们可以向其中插入和查询数据。我们已经拥有完成这样做所需的所有工具，只需在我们的`MetaData`上调用`MetaData.create_all()`方法，发送给它引用目标数据库的`Engine`：

```py
>>> metadata_obj.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
CREATE  TABLE  user_account  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30),
  fullname  VARCHAR,
  PRIMARY  KEY  (id)
)
...
CREATE  TABLE  address  (
  id  INTEGER  NOT  NULL,
  user_id  INTEGER  NOT  NULL,
  email_address  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(user_id)  REFERENCES  user_account  (id)
)
...
COMMIT 
```

上面的 DDL 创建过程包括一些特定于 SQLite 的 PRAGMA 语句，用于测试每个表的存在性，然后发出 CREATE。全部步骤也包括在一个 BEGIN/COMMIT 对中，以适应事务性 DDL。

创建过程还负责按正确顺序发出 CREATE 语句；上面，FOREIGN KEY 约束依赖于`user`表的存在，因此`address`表第二个创建。在更复杂的依赖场景中，FOREIGN KEY 约束也可以在创建后使用 ALTER 应用于表。

`MetaData`对象还具有一个`MetaData.drop_all()`方法，它将按相反顺序发出 DROP 语句，以便删除模式元素，就像发出 CREATE 语句一样。

### `Table` 的组件

我们可以观察到，Python 中的`Table`构造与 SQL CREATE TABLE 语句有些相似；从表名开始，然后列出每个列，其中每个列都有一个名称和一个数据类型。我们上面使用的对象是：

+   `Table` - 表示数据库表，并将自己分配给`MetaData`集合。

+   `Column` - 表示数据库表中的列，并将自身分配给一个`Table`对象。`Column`通常包括一个字符串名称和一个类型对象。关于父`Table`的`Column`对象的集合通常通过位于`Table.c`的关联数组来访问：

    ```py
    >>> user_table.c.name
    Column('name', String(length=30), table=<user_account>)

    >>> user_table.c.keys()
    ['id', 'name', 'fullname']
    ```

+   `Integer`，`String` - 这些类表示 SQL 数据类型，并且可以被传递给一个`Column`，无论是否被实例化。在上面的例子中，我们想给“name”列设置长度为“30”，所以我们实例化了`String(30)`。但是对于“id”和“fullname”，我们没有指定长度，所以我们可以直接发送类本身。

另请参阅

关于`MetaData`，`Table`和`Column`的参考文档和 API 文档位于用 MetaData 描述数据库。数据类型的参考文档位于 SQL 数据类型对象。

在接下来的章节中，我们将说明`Table`的一个基本功能，即在特定数据库连接上生成 DDL。但首先，我们将声明第二个`Table`。

### 声明简单约束

示例中的第一个`Column`包含了`Column.primary_key`参数，这是一种简写技术，表示这个`Column`应该是这个表的主键的一部分。主键本身通常是隐式声明的，并且由`PrimaryKeyConstraint`构造表示，我们可以在`Table`对象的`Table.primary_key`属性上看到：

```py
>>> user_table.primary_key
PrimaryKeyConstraint(Column('id', Integer(), table=<user_account>, primary_key=True, nullable=False))
```

最常明确声明的约束是对应于数据库外键约束的`ForeignKeyConstraint`对象。当我们声明彼此相关的表时，SQLAlchemy 使用这些外键约束声明的存在不仅使它们在向数据库发送 CREATE 语句时被发射，而且还有助于构建 SQL 表达式。

只涉及目标表中的单个列的`ForeignKeyConstraint`通常使用列级别的简写符号通过`ForeignKey`对象声明。下面我们声明了一个第二个表`address`，它将有一个外键约束，指向`user`表：

```py
>>> from sqlalchemy import ForeignKey
>>> address_table = Table(
...     "address",
...     metadata_obj,
...     Column("id", Integer, primary_key=True),
...     Column("user_id", ForeignKey("user_account.id"), nullable=False),
...     Column("email_address", String, nullable=False),
... )
```

上面的表还具有第三种约束，这在 SQL 中是“NOT NULL”约束，在上面使用`Column.nullable`参数指示。

提示

在`Column`定义中使用`ForeignKey`对象时，我们可以省略该`Column`的数据类型；它会自动从相关列的数据类型中推断出来，在上面的例子中是`user_account.id`列的`Integer`数据类型。

在下一节中，我们将发送完成的`user`和`address`表的 DDL 以查看完成的结果。

### 发送 DDL 到数据库

我们已经构建了一个对象结构，表示数据库中的两个数据库表，从根`MetaData`对象开始，然后进入两个`Table`对象，每个对象都持有一组`Column`和`Constraint`对象的集合。这个对象结构将成为我们使用 Core 和 ORM 执行的大多数操作的中心。

我们可以对这个结构进行的第一项有用的事情是发出 CREATE TABLE 语句，或者 DDL 到我们的 SQLite 数据库，这样我们就可以向其中插入和查询数据。我们已经拥有所有必要的工具来做到这一点，通过在我们的`MetaData`上调用`MetaData.create_all()`方法，将指向目标数据库的`Engine`传递给它：

```py
>>> metadata_obj.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
CREATE  TABLE  user_account  (
  id  INTEGER  NOT  NULL,
  name  VARCHAR(30),
  fullname  VARCHAR,
  PRIMARY  KEY  (id)
)
...
CREATE  TABLE  address  (
  id  INTEGER  NOT  NULL,
  user_id  INTEGER  NOT  NULL,
  email_address  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(user_id)  REFERENCES  user_account  (id)
)
...
COMMIT 
```

上面的 DDL 创建过程包括一些 SQLite 特定的 PRAGMA 语句，用于在发出 CREATE 之前测试每个表的存在。完整的步骤系列也包含在 BEGIN/COMMIT 对中，以适应事务性 DDL。

创建过程还负责按正确顺序发出 CREATE 语句；上面，FOREIGN KEY 约束依赖于`user`表的存在，因此`address`表是第二个被创建的。在更复杂的依赖场景中，FOREIGN KEY 约束也可以在创建后针对表使用 ALTER 来应用。

`MetaData`对象还具有一个`MetaData.drop_all()`方法，它将按相反顺序发出 DROP 语句，以删除模式元素，与发出 CREATE 语句的顺序相反。

## 使用 ORM 声明形式定义表元数据

当使用 ORM 时，声明`Table`元数据的过程通常与声明映射类的过程结合在一起。映射类是我们想要创建的任何 Python 类，然后该类上将具有与数据库表中的列相关联的属性。虽然有几种实现这一目标的方式，但最常见的风格被称为声明式，它允许我们一次声明我们的用户定义类和`Table`元数据。

### 建立声明式基础

在使用 ORM 时，`MetaData` 集合仍然存在，但它本身与一个仅在 ORM 中使用的构造关联，通常称为**声明式基础**。获得新的声明式基础的最快捷方式是创建一个新类，它是 SQLAlchemy `DeclarativeBase` 类的子类：

```py
>>> from sqlalchemy.orm import DeclarativeBase
>>> class Base(DeclarativeBase):
...     pass
```

在上文中，`Base` 类是我们所称的声明式基础。当我们创建的新类是 `Base` 的子类，并且结合适当的类级指令时，它们将在类创建时作为一个新的**ORM 映射类**建立，每个类通常（但不仅限于）引用一个特定的`Table`对象。

声明式基础指的是一个自动为我们创建的 `MetaData` 集合，假设我们没有从外部提供。这个 `MetaData` 集合可通过类级属性 `DeclarativeBase.metadata` 访问。当我们创建新的映射类时，它们每个都会引用这个 `MetaData` 集合中的一个 `Table`：

```py
>>> Base.metadata
MetaData()
```

声明式基础还指一个称为 `registry` 的集合，这是 SQLAlchemy ORM 中的中心“映射器配置”单元。虽然很少直接访问，但该对象是映射器配置过程的核心，因为一组 ORM 映射类将通过该注册表相互协调。就像 `MetaData` 一样，我们的声明式基础也为我们创建了一个 `registry`（再次有选项传递我们自己的 `registry`），我们可以通过类变量 `DeclarativeBase.registry` 访问：

```py
>>> Base.registry
<sqlalchemy.orm.decl_api.registry object at 0x...>
```  ### 声明映射类

有了`Base`类，我们现在可以根据新类`User`和`Address`为`user_account`和`address`表定义 ORM 映射类。我们下面展示了声明性的最现代形式，它是由[**PEP 484**](https://peps.python.org/pep-0484/)类型注释驱动的，使用了一种特殊类型`Mapped`，指示要映射为特定类型的属性：

```py
>>> from typing import List
>>> from typing import Optional
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship

>>> class User(Base):
...     __tablename__ = "user_account"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str] = mapped_column(String(30))
...     fullname: Mapped[Optional[str]]
...
...     addresses: Mapped[List["Address"]] = relationship(back_populates="user")
...
...     def __repr__(self) -> str:
...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

>>> class Address(Base):
...     __tablename__ = "address"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     email_address: Mapped[str]
...     user_id = mapped_column(ForeignKey("user_account.id"))
...
...     user: Mapped[User] = relationship(back_populates="addresses")
...
...     def __repr__(self) -> str:
...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
```

上面的两个类，`User`和`Address`，现在被称为**ORM 映射类**，并可用于 ORM 持久性和查询操作，稍后将描述。关于这些类的详细信息包括：

+   每个类都指向一个由声明性映射过程生成的`Table`对象，通过将字符串赋值给`DeclarativeBase.__tablename__`属性来命名。一旦类被创建，这个生成的`Table`可以通过`DeclarativeBase.__table__`属性访问。

+   如前所述，此形式被称为声明性表配置。几种替代的声明样式之一将直接构建`Table`对象，并将其直接分配给`DeclarativeBase.__table__`。这种风格被称为声明性与命令式表配置。

+   要指示`Table`中的列，我们使用`mapped_column()`结构，结合基于`Mapped`类型的类型注释。这个对象将生成应用于`Table`构造的`Column`对象。

+   对于具有简单数据类型且没有其他选项的列，我们可以单独指定`Mapped`类型注释，使用简单的 Python 类型如`int`和`str`来表示`Integer`和`String`。在 Declarative 映射过程中解释 Python 类型的方式非常开放；请参阅使用注释的 Declarative 表（对 mapped_column() 的类型注释形式）和自定义类型映射部分了解背景信息。

+   可以根据存在`Optional[<typ>]`类型注释（或其等效项`<typ> | None`或`Union[<typ>, None]`）来声明列是否“可空”或“非空”。还可以显式使用`mapped_column.nullable`参数（不必与注释的可选性匹配）。

+   显式类型注释的使用**完全是可选的**。我们还可以在没有注释的情况下使用`mapped_column()`。在使用这种形式时，我们会根据每个`mapped_column()`构造中的需要使用更明确的类型对象，例如`Integer`和`String`，以及`nullable=False`。

+   两个额外的属性，`User.addresses`和`Address.user`，定义了一种称为`relationship()`的不同类型属性，它具有与上述类似的注释感知配置样式。`relationship()`构造更详细地讨论在处理 ORM 相关对象中。

+   如果我们没有声明自己的`__init__()`方法，类会自动获得一个。默认情况下，这个方法接受所有属性名称作为可选关键字参数：

    ```py
    >>> sandy = User(name="sandy", fullname="Sandy Cheeks")
    ```

    要自动生成一个支持位置参数以及具有默认关键字值的全功能`__init__()`方法，可以使用在声明性数据类映射中介绍的 dataclasses 功能。当然，始终可以选择使用显式的`__init__()`方法。

+   `__repr__()` 方法被添加以便我们获得可读的字符串输出；这些方法不要求必须在这里。与 `__init__()` 一样，可以通过使用 dataclasses 功能来自动生成 `__repr__()` 方法。

请参阅

ORM 映射样式 - 不同 ORM 配置样式的完整背景。

声明式映射 - 声明类映射的概述

带有 mapped_column() 的声明式表 - 如何使用 `mapped_column()` 和 `Mapped` 来定义在使用声明式时映射到 `Table` 中的列的详细信息。

### 从 ORM 映射向数据库发出 DDL

由于我们的 ORM 映射类引用了包含在 `MetaData` 集合中的 `Table` 对象，因此，使用声明基类发出 DDL 与之前在 将 DDL 发送到数据库 中描述的过程相同。在我们的情况下，我们已经在我们的 SQLite 数据库中生成了 `user` 和 `address` 表。如果我们还没有这样做，我们可以自由地利用与我们 ORM 声明基类关联的 `MetaData` 来执行此操作，方法是从 `DeclarativeBase.metadata` 属性访问集合，然后像之前一样使用 `MetaData.create_all()`。在这种情况下，将运行 PRAGMA 语句，但不会生成新表，因为已经发现它们已经存在：

```py
>>> Base.metadata.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
COMMIT 
```

### 建立声明基类

当使用 ORM 时，`MetaData` 集合仍然存在，但它本身与一个仅与 ORM 关联的构造相关联，通常称为**声明基类**。获取新的声明基类的最方便的方法是创建一个新类，该类是 SQLAlchemy `DeclarativeBase` 类的子类：

```py
>>> from sqlalchemy.orm import DeclarativeBase
>>> class Base(DeclarativeBase):
...     pass
```

在上面，`Base` 类是我们将称之为声明性基类的内容。当我们创建新的类作为 `Base` 的子类时，结合适当的类级指令，它们将在类创建时分别被确立为新的**ORM 映射类**，每个类通常（但不是唯一地）引用一个特定的`Table`对象。

声明性基类引用了一个`MetaData`集合，如果我们没有从外部提供，将会自动创建。这个`MetaData`集合可通过`DeclarativeBase.metadata`类级属性访问。当我们创建新的映射类时，它们每个都将引用此`MetaData`集合内的一个`Table`：

```py
>>> Base.metadata
MetaData()
```

声明性基类还引用了一个称为`registry`的集合，它是 SQLAlchemy ORM 中的中心“映射器配置”单元。虽然很少直接访问，但该对象对映射器配置过程至关重要，因为一组 ORM 映射类将通过此注册表相互协调。与`MetaData`一样，我们的声明性基类也为我们创建了一个`registry`（再次有选项传递我们自己的`registry`），我们可以通过`DeclarativeBase.registry`类变量访问它：

```py
>>> Base.registry
<sqlalchemy.orm.decl_api.registry object at 0x...>
```

### 声明映射类

有了 `Base` 类的确立，我们现在可以根据 `User` 和 `Address` 的新类分别为 `user_account` 和 `address` 表定义 ORM 映射类。我们下面展示了声明性的最现代形式，它是从[**PEP 484**](https://peps.python.org/pep-0484/)类型注释驱动的，使用特殊类型`Mapped`，它指示要映射为特定类型的属性：

```py
>>> from typing import List
>>> from typing import Optional
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship

>>> class User(Base):
...     __tablename__ = "user_account"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     name: Mapped[str] = mapped_column(String(30))
...     fullname: Mapped[Optional[str]]
...
...     addresses: Mapped[List["Address"]] = relationship(back_populates="user")
...
...     def __repr__(self) -> str:
...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

>>> class Address(Base):
...     __tablename__ = "address"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     email_address: Mapped[str]
...     user_id = mapped_column(ForeignKey("user_account.id"))
...
...     user: Mapped[User] = relationship(back_populates="addresses")
...
...     def __repr__(self) -> str:
...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
```

上面的两个类，`User` 和 `Address`，现在被称为**ORM 映射类**，并可用于 ORM 持久性和查询操作，这将在后面进行描述。关于这些类的详细信息包括：

+   每个类都引用了作为声明性映射过程的一部分生成的`Table`对象，该对象通过将字符串分配给`DeclarativeBase.__tablename__`属性而命名。一旦类被创建，此生成的`Table`可从`DeclarativeBase.__table__`属性中获得。

+   如前所述，这种形式称为声明性表配置。几种备选声明样式之一是直接构建`Table`对象，并将其直接分配给`DeclarativeBase.__table__`。这种样式称为具有命令式表的声明性。

+   为了指示`Table`中的列，我们使用`mapped_column()`构造，结合基于`Mapped`类型的类型注释。这个对象将生成应用于`Table`构造的`Column`对象。

+   对于具有简单数据类型且没有其他选项的列，我们可以单独指示`Mapped`类型注释，使用简单的 Python 类型，如`int`和`str`，表示`Integer`和`String`。如何在声明性映射过程中解释 Python 类型的定制非常开放；请参阅使用带注释的声明性表（对 mapped_column()的类型注释形式）和自定义类型映射章节了解背景信息。

+   根据存在的`Optional[<typ>]`类型注释（或其等效形式`<typ> | None`或`Union[<typ>, None]`），可以将列声明为“可为空”或“非空”。还可以显式使用`mapped_column.nullable`参数（不一定要与注释的可选性匹配）。

+   显式类型注释的使用**完全是可选的**。我们也可以在不带注释的情况下使用 `mapped_column()`。在使用这种形式时，我们将根据每个`mapped_column()`构造中所需的更明确的类型对象，如`Integer` 和 `String`，以及 `nullable=False`。

+   两个额外的属性，`User.addresses` 和 `Address.user`，定义了一种称为`relationship()`的不同类型的属性，该属性具有与示例相似的注释感知配置样式。更多关于 `relationship()` 构造的讨论请见使用 ORM 相关对象。

+   如果我们没有声明自己的 `__init__()` 方法，这些类将自动获得一个 `__init__()` 方法。该方法的默认形式接受所有属性名称作为可选关键字参数：

    ```py
    >>> sandy = User(name="sandy", fullname="Sandy Cheeks")
    ```

    要自动生成一个提供位置参数以及带有默认关键字值的全功能 `__init__()` 方法，可以使用在声明性数据类映射中介绍的数据类功能。当然，始终可以选择使用显式的 `__init__()` 方法。

+   添加了 `__repr__()` 方法以获得可读的字符串输出；这些方法没有必须存在的要求。与 `__init__()` 类似，可以使用 dataclasses 功能自动生成 `__repr__()` 方法。

另见

ORM 映射样式 - 不同 ORM 配置样式的完整背景。

声明性映射 - 声明性类映射概述

使用 `mapped_column()` 的声明式表 - 关于如何使用`mapped_column()`和`Mapped`来定义在声明式使用时要映射的`Table`中的列的详细信息。

### 从 ORM 映射向数据库发出 DDL

因为我们的 ORM 映射类引用了包含在`MetaData`集合中的`Table`对象，所以给定声明性基类发出 DDL 与先前描述的 Emitting DDL to the Database 相同。在我们的情况下，我们已经在我们的 SQLite 数据库中生成了`user`和`address`表。如果我们还没有这样做，我们可以自由地利用与我们的 ORM 声明性基类关联的`MetaData`，通过访问从`DeclarativeBase.metadata`属性获取的集合，然后像以前一样使用`MetaData.create_all()`。在这种情况下，会运行 PRAGMA 语句，但是不会生成新的表，因为它们已经存在：

```py
>>> Base.metadata.create_all(engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("user_account")
...
PRAGMA  main.table_...info("address")
...
COMMIT 
```

## 表反射

为了补充对工作中的表元数据的部分说明，我们将说明一种在部分开始时提到的操作，即**表反射**。表反射是指通过读取数据库的当前状态生成`Table`和相关对象的过程。在以前的部分中，我们在 Python 中声明了`Table`对象，然后我们有选择地将 DDL 发出到数据库以生成这样的模式，反射过程将这两个步骤倒置，从现有数据库开始，并生成 Python 中的数据结构以表示该数据库中的模式。

提示

并非要求使用反射来与预先存在的数据库一起使用 SQLAlchemy。完全可以将 SQLAlchemy 应用程序中的所有元数据都在 Python 中显式声明，以使其结构与现有数据库相对应。元数据结构也不必包括表、列或其他在预先存在的数据库中不需要的约束和结构，在本地应用程序中不需要。

作为反射的示例，我们将创建一个新的`Table`对象，该对象表示我们在本文档早期部分手动创建的`some_table`对象。这又有一些执行方式的变体，但最基本的是构建一个`Table`对象，给出表的名称和它将属于的`MetaData`集合，然后不是指示单独的`Column`和`Constraint`对象，而是使用`Table.autoload_with`参数传递目标`Engine`：

```py
>>> some_table = Table("some_table", metadata_obj, autoload_with=engine)
BEGIN  (implicit)
PRAGMA  main.table_...info("some_table")
[raw  sql]  ()
SELECT  sql  FROM  (SELECT  *  FROM  sqlite_master  UNION  ALL  SELECT  *  FROM  sqlite_temp_master)  WHERE  name  =  ?  AND  type  in  ('table',  'view')
[raw  sql]  ('some_table',)
PRAGMA  main.foreign_key_list("some_table")
...
PRAGMA  main.index_list("some_table")
...
ROLLBACK 
```

在流程结束时，`some_table`对象现在包含了表中存在的`Column`对象的信息，并且该对象可像我们显式声明的`Table`一样使用：

```py
>>> some_table
Table('some_table', MetaData(),
 Column('x', INTEGER(), table=<some_table>),
 Column('y', INTEGER(), table=<some_table>),
 schema=None)
```

另请参阅

阅读有关表和模式反射的更多信息，请访问反射数据库对象。

对于 ORM 相关的表反射变体，本节使用反射表声明性映射包括了可用选项的概述。

## 下一步

我们现在有一个准备好的 SQLite 数据库，其中包含两个表，以及我们可以使用它们与这些表进行交互的 Core 和 ORM 表导向结构，通过`Connection`和/或 ORM `Session`。在接下来的章节中，我们将说明如何使用这些结构来创建、操作和选择数据。
