# 非传统映射

> 原文：[`docs.sqlalchemy.org/en/20/orm/nonstandard_mappings.html`](https://docs.sqlalchemy.org/en/20/orm/nonstandard_mappings.html)

## 将类映射到多个表

映射器可以构造与任意关系单元（称为 *selectables*）相对应的类，除了普通表之外。例如，`join()` 函数创建了一个包含多个表的可选择单元，具有自己的复合主键，可以与 `Table` 相同的方式映射：

```py
from sqlalchemy import Table, Column, Integer, String, MetaData, join, ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import column_property

metadata_obj = MetaData()

# define two Table objects
user_table = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String),
)

address_table = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String),
)

# define a join between them.  This
# takes place across the user.id and address.user_id
# columns.
user_address_join = join(user_table, address_table)

class Base(DeclarativeBase):
    metadata = metadata_obj

# map to it
class AddressUser(Base):
    __table__ = user_address_join

    id = column_property(user_table.c.id, address_table.c.user_id)
    address_id = address_table.c.id
```

在上面的示例中，连接表示了 `user` 表和 `address` 表的列。`user.id` 和 `address.user_id` 列通过外键相等，因此在映射中它们被定义为一个属性，即 `AddressUser.id`，使用 `column_property()` 来指示一个特殊的列映射。基于这部分配置，当发生 flush 时，映射将把新的主键值从 `user.id` 复制到 `address.user_id` 列。

另外，`address.id` 列被显式映射到一个名为 `address_id` 的属性。这是为了**消除歧义**，将 `address.id` 列的映射与同名的 `AddressUser.id` 属性区分开来，这里已经被分配为引用 `user` 表与 `address.user_id` 外键结合的表。

上面映射的自然主键是 `(user.id, address.id)` 的组合，因为这些是 `user` 和 `address` 表的联合主键列。`AddressUser` 对象的标识将根据这两个值，并且从 `AddressUser` 对象表示为 `(AddressUser.id, AddressUser.address_id)`。

当涉及到 `AddressUser.id` 列时，大多数 SQL 表达式将仅使用映射列列表中的第一列，因为这两列是同义的。然而，对于特殊用例，比如 GROUP BY 表达式，在这种情况下需要同时引用两列，并且在使用正确的上下文时，即考虑到别名和类似情况时，可以使用访问器 `Comparator.expressions`：

```py
stmt = select(AddressUser).group_by(*AddressUser.id.expressions)
```

新功能在版本 1.3.17 中添加：增加了 `Comparator.expressions` 访问器。

注意

如上所示的针对多个表的映射支持持久化，即对目标表中的行进行 INSERT、UPDATE 和 DELETE。然而，它不支持一次为一个记录在一个表上执行 UPDATE 并在其他表上同时执行 INSERT 或 DELETE 的操作。也就是说，如果一个记录 PtoQ 被映射到“p”和“q”表，其中它基于“p”和“q”的 LEFT OUTER JOIN 有一行，如果进行一个 UPDATE 来修改现有记录中“q”表中的数据，那么“q”中的行必须存在；如果主键标识已经存在，它不会发出 INSERT。如果行不存在，对于大多数支持报告 UPDATE 受影响行数的 DBAPI 驱动程序，ORM 将无法检测到更新的行并引发错误；否则，数据将被静默忽略。

一个允许在相关行上“即时”插入的方法可能会使用.MapperEvents.before_update 事件，并且看起来像：

```py
from sqlalchemy import event

@event.listens_for(PtoQ, "before_update")
def receive_before_update(mapper, connection, target):
    if target.some_required_attr_on_q is None:
        connection.execute(q_table.insert(), {"id": target.id})
```

在上面的例子中，通过使用`Table.insert()`创建一个 INSERT 构造，然后使用给定的`Connection`执行它，将一行 INSERT 到`q_table`表中，这个 Connection 与用于发出 flush 过程中的其他 SQL 的 Connection 相同。用户提供的逻辑必须检测到从“p”到“q”的 LEFT OUTER JOIN 没有“q”侧的条目。## 对任意子查询映射类

类似于针对连接的映射，也可以将一个普通的`select()`对象与映射器一起使用。下面的示例片段说明了将一个名为`Customer`的类映射到一个包含与子查询连接的`select()`中：

```py
from sqlalchemy import select, func

subq = (
    select(
        func.count(orders.c.id).label("order_count"),
        func.max(orders.c.price).label("highest_order"),
        orders.c.customer_id,
    )
    .group_by(orders.c.customer_id)
    .subquery()
)

customer_select = (
    select(customers, subq)
    .join_from(customers, subq, customers.c.id == subq.c.customer_id)
    .subquery()
)

class Customer(Base):
    __table__ = customer_select
```

在上面，由`customer_select`表示的完整行将是`customers`表的所有列，以及`subq`子查询暴露的那些列，即`order_count`、`highest_order`和`customer_id`。将`Customer`类映射到这个可选择的内容，然后创建一个包含这些属性的类。

当 ORM 持久化`Customer`的新实例时，实际上只有`customers`表会收到 INSERT。这是因为`orders`表的主键没有在映射中表示；ORM 只会对已经映射了主键的表发出 INSERT。

注意

映射到任意 SELECT 语句的做法，特别是像上面这样复杂的语句，几乎从不需要；它必然倾向于生成复杂的查询，这些查询通常比直接构造查询要低效。这种做法在某种程度上基于 SQLAlchemy 的非常早期历史，其中`Mapper`构造被认为是主要的查询接口；在现代用法中，`Query`对象可以用于构造几乎任何 SELECT 语句，包括复杂的复合语句，并且应优先于“映射到可选”的方法。

## 为一个类映射多个映射器

在现代的 SQLAlchemy 中，一个特定的类一次只能由一个所谓的**主要**映射器（mapper）映射。这个映射器涉及三个主要功能领域：查询、持久性和对映射类的仪器化。主要映射器的理论基础与以下事实相关：`Mapper`修改了类本身，不仅将其持久化到特定的`Table`中，还对类上的属性进行了仪器化，这些属性根据表元数据特别结构化。不能有多个映射器与一个类同等相关，因为只有一个映射器可以实际仪器化该类。

“非主要”映射器的概念已经存在多个 SQLAlchemy 版本，但从 1.3 版本开始，此功能已被弃用。其中一个非主要映射器有用的情况是构建与备用可选择类之间的关系时。现在可以使用`aliased`构造来满足此用例，并在 Relationship to Aliased Class 中进行了描述。

就一个类可以在不同情境下被完全持久化到不同表中的用例而言，早期版本的 SQLAlchemy 提供了一个来自 Hibernate 的功能，称为“实体名称”功能。然而，在 SQLAlchemy 中，一旦映射类本身成为 SQL 表达式构造的来源，即类的属性直接链接到映射表列，这个用例就变得不可行了。该功能被移除，并被一个简单的面向配方的方法取代，以完成此任务而不产生任何仪器化的歧义——创建新的子类，每个类都被单独映射。该模式现在作为一种配方在[Entity Name](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/EntityName)中提供。

## 将一个类映射到多个表

Mappers 可以针对任意关系单元（称为*selectables*）进行构建，而不仅仅是普通的表。例如，`join()` 函数创建了一个包含多个表的可选单元，其中包括其自己的复合主键，可以与`Table` 以相同的方式映射：

```py
from sqlalchemy import Table, Column, Integer, String, MetaData, join, ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import column_property

metadata_obj = MetaData()

# define two Table objects
user_table = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String),
)

address_table = Table(
    "address",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("user_id", Integer, ForeignKey("user.id")),
    Column("email_address", String),
)

# define a join between them.  This
# takes place across the user.id and address.user_id
# columns.
user_address_join = join(user_table, address_table)

class Base(DeclarativeBase):
    metadata = metadata_obj

# map to it
class AddressUser(Base):
    __table__ = user_address_join

    id = column_property(user_table.c.id, address_table.c.user_id)
    address_id = address_table.c.id
```

在上面的示例中，连接表示了`user`和`address`表的列。 `user.id`和`address.user_id`列由外键等于，因此在映射中它们被定义为一个属性`AddressUser.id`，使用`column_property()`指示专门的列映射。根据配置的这一部分，当发生刷新时，映射将新的主键值从`user.id`复制到`address.user_id`列。

另外，`address.id`列显式映射到名为`address_id`的属性。这是为了**消除歧义**，将`address.id`列的映射与同名的`AddressUser.id`属性分开，这里已经被分配为引用`user`表与`address.user_id`外键的属性。

上述映射的自然主键是`(user.id, address.id)`的组合，因为这些是`user`和`address`表的主键列合并在一起。 `AddressUser`对象的标识将根据这两个值，并且从`AddressUser`对象表示为`(AddressUser.id, AddressUser.address_id)`。

在引用`AddressUser.id`列时，大多数 SQL 表达式将仅使用映射列列表中的第一列，因为这两列是同义的。但是，对于特殊用例，例如必须同时引用两列的 GROUP BY 表达式，同时考虑到适当的上下文，即适应别名等，可以使用访问器`Comparator.expressions`：

```py
stmt = select(AddressUser).group_by(*AddressUser.id.expressions)
```

1.3.17 版本中的新内容：添加了`Comparator.expressions` 访问器。

注意

如上所示的对多个表的映射支持持久性，即对目标表中的行进行 INSERT、UPDATE 和 DELETE 操作。然而，它不支持在一条记录中同时对一个表进行 UPDATE 并在其他表上执行 INSERT 或 DELETE 的操作。也就是说，如果将记录 PtoQ 映射到“p”和“q”表，其中它基于“p”和“q”的 LEFT OUTER JOIN 的行，如果进行更新以更改现有记录中“q”表中的数据，则“q”中的行必须存在；如果主键标识已经存在，它不会发出 INSERT。如果行不存在，对于大多数支持报告 UPDATE 受影响行数的 DBAPI 驱动程序，ORM 将无法检测到更新的行并引发错误；否则，数据将被静默忽略。

允许在“插入”相关行时使用的配方可能利用`.MapperEvents.before_update`事件，并且看起来像：

```py
from sqlalchemy import event

@event.listens_for(PtoQ, "before_update")
def receive_before_update(mapper, connection, target):
    if target.some_required_attr_on_q is None:
        connection.execute(q_table.insert(), {"id": target.id})
```

在上述情况下，通过使用`Table.insert()`创建一个 INSERT 构造将一行插入`q_table`表，然后使用给定的`Connection`执行它，这与用于发出刷新过程中的其他 SQL 的相同连接。用户提供的逻辑必须检测从“p”到“q”的 LEFT OUTER JOIN 是否没有“q”方面的条目。

## 将类映射到任意子查询

类似于对连接进行映射，也可以将一个普通的`select()`对象与映射器一起使用。下面的示例片段说明了将名为`Customer`的类映射到包含与子查询连接的`select()`的过程：

```py
from sqlalchemy import select, func

subq = (
    select(
        func.count(orders.c.id).label("order_count"),
        func.max(orders.c.price).label("highest_order"),
        orders.c.customer_id,
    )
    .group_by(orders.c.customer_id)
    .subquery()
)

customer_select = (
    select(customers, subq)
    .join_from(customers, subq, customers.c.id == subq.c.customer_id)
    .subquery()
)

class Customer(Base):
    __table__ = customer_select
```

在上面，由`customer_select`表示的完整行将是`customers`表的所有列，以及`subq`子查询暴露的那些列，即`order_count`、`highest_order`和`customer_id`。将`Customer`类映射到这个可选择的类，然后创建一个包含这些属性的类。

当 ORM 持久化`Customer`的新实例时，实际上只有`customers`表会收到 INSERT。这是因为`orders`表的主键没有在映射中表示；ORM 只会对已映射主键的表发出 INSERT。

注意

对任意 SELECT 语句进行映射的实践，特别是上面那种复杂的情况，几乎从不需要；这必然会产生复杂的查询，通常比直接构造的查询效率低。这种做法在某种程度上基于 SQLAlchemy 的早期历史，其中`Mapper`构造旨在代表主要的查询接口；在现代用法中，`Query`对象可用于构造几乎任何 SELECT 语句，包括复杂的复合语句，并且应优先使用“映射到可选择”方法。

## 一个类对应多个映射器

在现代的 SQLAlchemy 中，一个特定的类在任何时候只被一个所谓的**主要**映射器所映射。这个映射器涉及三个主要功能领域：查询、持久化和对映射类的仪器化。主要映射器的理念与以下事实相关：`Mapper`不仅修改类本身，而且将其持久化到特定的`Table`，还会根据表元数据结构化地仪器化类上的属性。不可能有多个映射器与一个类一样平等地关联，因为只有一个映射器实际上可以仪器化这个类。

“非主要”映射器的概念在许多版本的 SQLAlchemy 中一直存在，但自版本 1.3 起，此功能已不建议使用。唯一需要非主要映射器的情况是在构造与另一个可选择的类的关系时。现在，可以使用`aliased`构造来满足这个用例，并在关系到别名类中进行描述。

就一个类在不同情境下可以完全持久化到不同表的用例而言，SQLAlchemy 的早期版本提供了一个从 Hibernate 改编而来的功能，称为“实体名称”功能。然而，在 SQLAlchemy 中，一旦映射的类本身成为 SQL 表达式构造的源，即类的属性直接链接到映射表的列，这个用例就变得不可行了。该功能被移除，并用一个简单的基于配方的方法来完成这个任务，而不会有任何仪器化的歧义 - 即创建新的子类，每个类都单独映射。这种模式现在作为[实体名称](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/EntityName)的配方可用。
