# 关系加载技术

> 原文：[`docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html`](https://docs.sqlalchemy.org/en/20/orm/queryguide/relationships.html)

关于本文档

本节详细介绍了如何加载相关对象。读者应熟悉关系配置和基本用法。

大多数示例假定“用户/地址”映射设置类似于在选择设置中所示的设置。

SQLAlchemy 的一个重要部分是在查询时提供对相关对象加载方式的广泛控制。所谓“相关对象”是指在映射器上使用`relationship()`配置的集合或标量关联。这种行为可以在映射器构造时使用`relationship()`函数的`relationship.lazy`参数进行配置，以及通过使用**ORM 加载选项**与`Select`构造函数一起使用。

关系加载分为三类：**延迟**加载、**急切**加载和**无**加载。延迟加载指的是从查询返回的对象，相关对象一开始并未加载。当在特定对象上首次访问给定集合或引用时，会发出额外的 SELECT 语句，以加载请求的集合。

急切加载是指从查询返回的对象中，相关集合或标量引用已经提前加载。ORM 实现这一点要么通过增加通常会发出的 SELECT 语句以加载相关行，要么通过在主要语句之后发出额外的 SELECT 语句以一次性加载集合或标量引用。

“无”加载指的是在给定关系上禁用加载，要么属性为空且从不加载，要么在访问时引发错误，以防止不必要的延迟加载。

## 关系加载样式总结

关系加载的主要形式包括：

+   **惰性加载** - 可通过`lazy='select'`或`lazyload()`选项使用，这是一种加载形式，它在属性访问时发出 SELECT 语句，以惰性加载单个对象上的相关引用。惰性加载是所有未明示`relationship.lazy`选项的`relationship()`构造的**默认加载方式**。惰性加载详见惰性加载。

+   **选择 IN 加载** - 可通过`lazy='selectin'`或`selectinload()`选项使用，这种加载形式会发出第二个（或更多）SELECT 语句，将父对象的主键标识符组装到一个 IN 子句中，以便通过主键一次加载所有相关集合/标量引用。选择 IN 加载详见选择 IN 加载。

+   **关联加载** - 可通过`lazy='joined'`或`joinedload()`选项使用，这种加载方式会在给定的 SELECT 语句上应用 JOIN，以便相关行在同一结果集中加载。关联的及时加载详见关联的及时加载。

+   **引发加载** - 可通过`lazy='raise'`、`lazy='raise_on_sql'`或`raiseload()`选项使用，这种加载形式在通常会发生惰性加载的时候被触发，但它会引发一个 ORM 异常，以防止应用程序进行不必要的惰性加载。引发加载的简介详见使用 raiseload 防止不想要的惰性加载。

+   **子查询加载** - 可通过`lazy='subquery'`或`subqueryload()`选项使用，这种加载方式会发出第二个 SELECT 语句，该语句重新陈述了原始查询嵌入到子查询中，然后将该子查询与要加载的相关表 JOIN 起来，以一次加载所有相关集合/标量引用的成员。子查询的及时加载详见子查询的及时加载。

+   **仅写入加载** - 可通过`lazy='write_only'`获得，或通过在`Relationship`对象的左侧使用`WriteOnlyMapped`注解来标注。这种仅限集合的加载方式产生了一种替代的属性装载机制，从不隐式地从数据库加载记录，而是仅允许使用`WriteOnlyCollection.add()`、`WriteOnlyCollection.add_all()`和`WriteOnlyCollection.remove()`方法。查询集合是通过调用使用`WriteOnlyCollection.select()`方法构建的 SELECT 语句来执行的。仅写入加载在仅写入关系中进行了讨论。

+   **动态加载** - 可通过`lazy='dynamic'`获得，或通过在`Relationship`对象的左侧使用`DynamicMapped`注解来标注。这是一种传统的仅限集合的加载样式，当访问集合时会产生一个`Query`对象，允许针对集合的内容发出自定义 SQL。然而，动态加载器在各种情况下将隐式迭代底层集合，这使它们对管理真正大型集合不太有用。动态加载器已被“仅写入”集合取代，后者将阻止在任何情况下隐式加载底层集合。动态加载器在动态关系加载器中进行了讨论。

## 在映射时配置加载策略

特定关系的加载策略可以在映射时配置，以在加载映射类型的对象的所有情况下发生，即使没有修改它的任何查询级别选项。这是使用`relationship()`的`relationship.lazy`参数进行配置的；此参数的常见值包括`select`、`selectin`和`joined`。

下面的示例说明了在一对多关系模式下的关系示例，配置了`Parent.children`关系以在发出`Parent`对象的 SELECT 语句时使用选择 IN 加载：

```py
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(lazy="selectin")

class Child(Base):
    __tablename__ = "child"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
```

在上面的示例中，每当加载`Parent`对象集合时，每个`Parent`也将其`children`集合填充，使用`"selectin"`加载策略发出第二个查询。

`relationship.lazy`参数的默认值是`"select"`，表示延迟加载。## 使用加载器选项进行关系加载

另一种，可能更常见的配置加载策略的方式是在针对特定属性的每个查询上设置它们，使用`Select.options()`方法。使用加载器选项可以对关系加载进行非常详细的控制；最常见的是`joinedload()`、`selectinload()`和`lazyload()`。该选项接受一个类绑定的属性，指示应针对特定类/属性进行定位：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

# set children to load lazily
stmt = select(Parent).options(lazyload(Parent.children))

from sqlalchemy.orm import joinedload

# set children to load eagerly with a join
stmt = select(Parent).options(joinedload(Parent.children))
```

加载器选项也可以使用**方法链接**进行“链接”，以指定进一步深层次的加载方式：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload

stmt = select(Parent).options(
    joinedload(Parent.children).subqueryload(Child.subelements)
)
```

链接的加载器选项可以应用于“延迟”加载的集合。这意味着当在访问时惰性加载集合或关联时，指定的选项将生效：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))
```

在上面的查询中，将返回未加载`children`集合的`Parent`对象。当首次访问特定`Parent`对象上的`children`集合时，它将延迟加载相关对象，并且还将对每个`children`成员的`subelements`集合应用急加载。

### 向加载器选项添加条件

用于指示加载器选项的关系属性包括向创建的连接的 ON 子句添加额外的过滤条件，或者根据加载器策略涉及到的 WHERE 条件添加过滤条件的能力。这可以通过使用`PropComparator.and_()`方法来实现，该方法将通过一个选项传递，以便加载的结果被限制为给定的过滤条件：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(A).options(lazyload(A.bs.and_(B.id > 5)))
```

当使用限制条件时，如果特定的集合已经加载，则不会刷新；为了确保新的条件生效，请应用现有填充执行选项：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = (
    select(A)
    .options(lazyload(A.bs.and_(B.id > 5)))
    .execution_options(populate_existing=True)
)
```

为了在查询中添加对实体的所有出现的过滤条件，无论加载策略如何或它在加载过程中的位置如何，参见 `with_loader_criteria()` 函数。

新版本 1.4 中新增。### 使用 Load.options() 指定子选项

使用方法链，路径中每个链接的加载器样式都明确说明。要沿着路径导航而不更改特定属性的现有加载器样式，请使用 `defaultload()` 方法/函数：

```py
from sqlalchemy import select
from sqlalchemy.orm import defaultload

stmt = select(A).options(defaultload(A.atob).joinedload(B.btoc))
```

类似的方法可以用来一次指定多个子选项，使用 `Load.options()` 方法：

```py
from sqlalchemy import select
from sqlalchemy.orm import defaultload
from sqlalchemy.orm import joinedload

stmt = select(A).options(
    defaultload(A.atob).options(joinedload(B.btoc), joinedload(B.btod))
)
```

另请参见

在相关对象和集合上使用 load_only() - 演示了结合关系和基于列的加载器选项的示例。

注意

应用于对象的延迟加载集合的加载器选项对于特定对象实例是**“粘性的”**，这意味着它们将在内存中存在的对象加载的集合上持续存在。例如，给定前面的示例：

```py
stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))
```

如果上述查询加载的特定 `Parent` 对象上的 `children` 集合过期（例如当 `Session` 对象的事务提交或回滚时，或使用了 `Session.expire_all()`），当下次访问 `Parent.children` 集合以重新加载时，`Child.subelements` 集合将再次使用子查询急加载。即使上述 `Parent` 对象是从指定了不同选项集的后续查询中访问的，这种情况仍然存在。要在不清除对象并重新加载的情况下更改现有对象上的选项，必须使用 Populate Existing 执行选项显式设置它们：

```py
# change the options on Parent objects that were already loaded
stmt = (
    select(Parent)
    .execution_options(populate_existing=True)
    .options(lazyload(Parent.children).lazyload(Child.subelements))
    .all()
)
```

如果上面加载的对象被 `Session` 完全清除，例如由于垃圾回收或使用了 `Session.expunge_all()`，那么“粘性”选项也将消失，如果再次加载新创建的对象，则会使用新选项。

未来的 SQLAlchemy 发布版本可能会添加更多的选择来操作已加载对象上的加载器选项。## 延迟加载

默认情况下，所有对象之间的关系都是**延迟加载**的。与`relationship()`关联的标量或集合属性包含一个触发器，在第一次访问属性时触发。这个触发器通常在访问点发出 SQL 调用，以加载相关的对象或对象：

```py
>>> spongebob.addresses
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id
FROM  addresses
WHERE  ?  =  addresses.user_id
[5]
[<Address(u'spongebob@google.com')>, <Address(u'j25@yahoo.com')>]
```

唯一的情况是不发出 SQL 的情况是简单的多对一关系，当相关对象仅可以通过其主键单独标识，并且该对象已经存在于当前`Session`中时。因此，虽然对相关集合进行延迟加载可能很昂贵，但在加载许多对象与相对较小的可能目标对象集合的情况下，延迟加载可能能够在本地引用这些对象，而无需发出与父对象数量相同数量的 SELECT 语句。

这种“根据属性访问加载”的默认行为称为“延迟”或“选择”加载 - 名称“选择”是因为在首次访问属性时通常会发出“SELECT”语句。

可以使用`lazyload()`加载器选项来启用通常以其他方式配置的给定属性的延迟加载：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

# force lazy loading for an attribute that is set to
# load some other way normally
stmt = select(User).options(lazyload(User.addresses))
```

### 使用 raiseload 防止不需要的延迟加载

`lazyload()`策略产生的效果是对象关系映射中最常见的问题之一；N 加一问题，它指出对于任何加载的 N 个对象，访问其惰性加载的属性意味着会发出 N+1 个 SELECT 语句。在 SQLAlchemy 中，解决 N+1 问题的通常方法是利用其非常强大的急加载系统。然而，急加载要求事先使用`Select`指定要加载的属性。可能访问其他未急加载的属性的代码问题，不希望延迟加载，可以使用`raiseload()`策略来解决；这个加载器策略用引发一个具有信息性错误的方式替换了惰性加载的行为：

```py
from sqlalchemy import select
from sqlalchemy.orm import raiseload

stmt = select(User).options(raiseload(User.addresses))
```

以上，从上述查询加载的`User`对象将不会加载`.addresses`集合；如果稍后的一些代码尝试访问此属性，则会引发 ORM 异常。

可以使用所谓的“通配符”指示符将`raiseload()`用于表示所有关系都应该使用这种策略。例如，设置仅一个属性为急加载，并将其余全部设置为 raise：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import raiseload

stmt = select(Order).options(joinedload(Order.items), raiseload("*"))
```

上述通配符将适用于**所有**关系，而不仅适用于`items`，还适用于`Item`对象上的所有关系。要仅为`Order`对象设置`raiseload()`，请指定具有`Load`的完整路径：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import Load

stmt = select(Order).options(joinedload(Order.items), Load(Order).raiseload("*"))
```

相反，要为仅`Item`对象设置提升：

```py
stmt = select(Order).options(joinedload(Order.items).raiseload("*"))
```

`raiseload()`选项仅适用于关系属性。对于基于列的属性，`defer()`选项支持`defer.raiseload`选项，其工作方式相同。

提示

“raiseload”策略在工作单元刷新过程中**不适用**。这意味着如果`Session.flush()`过程需要加载集合以完成其工作，则会在绕过任何`raiseload()`指令的情况下执行此操作。

另请参见

通配符加载策略

使用 raiseload 防止延迟列加载  ## 连接预加载

连接预加载是包含在 SQLAlchemy ORM 中的最古老的预加载样式。它通过将 JOIN（默认为 LEFT OUTER join）连接到发出的 SELECT 语句，并且从与父级相同的结果集中填充目标标量/集合来工作。

在映射级别，看起来像这样：

```py
class Address(Base):
    # ...

    user: Mapped[User] = relationship(lazy="joined")
```

连接预加载通常作为查询的选项应用，而不是作为映射的默认加载选项，特别是在用于集合而不是多对一引用时。可以使用`joinedload()`加载器选项来实现这一点：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import joinedload
>>> stmt = select(User).options(joinedload(User.addresses)).filter_by(name="spongebob")
>>> spongebob = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
['spongebob'] 
```

提示

当参考一对多或多对多集合时包括`joinedload()`时，必须对返回的结果应用`Result.unique()`方法，该方法将通过否则由连接相乘出的主键使传入的行不重复。如果未提供此项，则 ORM 将引发错误。

这在现代 SQLAlchemy 中不是自动的，因为它会更改结果集的行为，使其返回的 ORM 对象比语句通常返回的行数少。因此，SQLAlchemy 保持了对`Result.unique()`的使用是显式的，因此返回的对象是在主键上唯一的，没有任何歧义。

默认情况下发出的 JOIN 是 LEFT OUTER JOIN，以允许不引用相关行的主对象。对于保证具有元素的属性，例如引用相关对象的多对一引用，其中引用外键不为 NULL，可以通过使用内连接使查询更有效；这在映射级别通过 `relationship.innerjoin` 标志可用：

```py
class Address(Base):
    # ...

    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    user: Mapped[User] = relationship(lazy="joined", innerjoin=True)
```

在查询选项级别，通过 `joinedload.innerjoin` 标志：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload

stmt = select(Address).options(joinedload(Address.user, innerjoin=True))
```

当应用于包含 OUTER JOIN 的链时，JOIN 会向右嵌套自身：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import joinedload
>>> stmt = select(User).options(
...     joinedload(User.addresses).joinedload(Address.widgets, innerjoin=True)
... )
>>> results = session.scalars(stmt).unique().all()
SELECT
  widgets_1.id  AS  widgets_1_id,
  widgets_1.name  AS  widgets_1_name,
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
LEFT  OUTER  JOIN  (
  addresses  AS  addresses_1  JOIN  widgets  AS  widgets_1  ON
  addresses_1.widget_id  =  widgets_1.id
)  ON  users.id  =  addresses_1.user_id 
```

提示

如果在发出 `SELECT` 时使用数据库行锁定技术，即意味着正在使用 `Select.with_for_update()` 方法发出 `SELECT..FOR UPDATE`，则根据使用的后端的行为，可能会锁定连接的表。出于这个原因，不建议同时使用连接的急切加载和 `SELECT..FOR UPDATE`。

### 连接急切加载的禅意

由于连接的急切加载似乎与使用 `Select.join()` 的方式有很多相似之处，因此人们经常困惑于何时以及如何使用它。至关重要的是要理解这样的区别，即虽然 `Select.join()` 用于更改查询的结果，但 `joinedload()` 费尽心思地**不**更改查询的结果，而是隐藏所渲染的连接的效果，仅允许相关对象存在。

加载器策略背后的理念是，任何一组加载方案都可以应用于特定查询，*结果不会改变* - 只有用于完全加载相关对象和集合所需的 SQL 语句数量会改变。一个特定的查询可能首先使用所有惰性加载。在上下文中使用后，可能会发现总是访问特定属性或集合，并且更改这些属性的加载策略更有效。该策略可以更改而不必对查询进行其他修改，结果将保持不变，但会发出更少的 SQL 语句。理论上（实际上基本如此），对 `Select` 所做的任何操作都不会使其根据加载器策略的更改而加载不同的主对象或相关对象集。

特别地，`joinedload()`是如何实现不以任何方式影响返回的实体行的结果的，这是因为它为添加到查询中的连接创建了一个匿名别名，因此它们不能被查询的其他部分引用。例如，下面的查询使用`joinedload()`来创建从`users`到`addresses`的 LEFT OUTER JOIN，但是针对`Address.email_address`添加的`ORDER BY`是无效的 - 查询中未命名实体`Address`：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import joinedload
>>> stmt = (
...     select(User)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address  <-- this part is wrong !
['spongebob'] 
```

上面，`ORDER BY addresses.email_address` 是无效的，因为`addresses`不在 FROM 列表中。加载`User`记录并按电子邮件地址排序的正确方法是使用`Select.join()`：

```py
>>> from sqlalchemy import select
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
JOIN  addresses  ON  users.id  =  addresses.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address
['spongebob'] 
```

上述语句当然与先前的语句不同，因为从`addresses`中提取的列根本没有包含在结果中。我们可以重新添加`joinedload()`，这样就有两个连接 - 一个是我们正在排序的连接，另一个是匿名使用的连接，用于加载`User.addresses`集合的内容：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users  JOIN  addresses
  ON  users.id  =  addresses.user_id
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address
['spongebob'] 
```

我们上面看到的是我们对`Select.join()`的用法是提供我们希望在随后的查询条件中使用的 JOIN 子句，而我们对`joinedload()`的用法只涉及加载结果中每个`User`的`User.addresses`集合。在这种情况下，这两个连接很可能看起来是多余的 - 它们就是。如果我们只想使用一个 JOIN 来加载集合并进行排序，我们可以使用`contains_eager()`选项，下面将介绍如何将显式的 JOIN/语句路由到急加载的集合中。但要了解`joinedload()`为什么会产生这样的结果，考虑一下如果我们在特定的`Address`上进行**过滤**：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .filter(Address.email_address == "someaddress@foo.com")
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users  JOIN  addresses
  ON  users.id  =  addresses.user_id
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?  AND  addresses.email_address  =  ?
['spongebob',  'someaddress@foo.com'] 
```

上面，我们可以看到这两个 JOIN 的作用非常不同。一个将完全匹配一个行，即`User`和`Address`的连接，其中`Address.email_address=='someaddress@foo.com'`。另一个 LEFT OUTER JOIN 将匹配与`User`相关的所有`Address`行，并且仅用于为返回的那些`User`对象填充`User.addresses`集合。

通过改变 `joinedload()` 的使用方式到另一种加载样式，我们可以完全独立于用于检索实际想要的 `User` 行的 SQL 改变集合的加载方式。以下我们将 `joinedload()` 改变为 `selectinload()`：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(selectinload(User.addresses))
...     .filter(User.name == "spongebob")
...     .filter(Address.email_address == "someaddress@foo.com")
... )
>>> result = session.scalars(stmt).all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
JOIN  addresses  ON  users.id  =  addresses.user_id
WHERE
  users.name  =  ?
  AND  addresses.email_address  =  ?
['spongebob',  'someaddress@foo.com']
#  ...  selectinload()  emits  a  SELECT  in  order
#  to  load  all  address  records  ... 
```

当使用联接式的急加载时，如果查询包含影响联接外返回的行的修改器，比如使用 DISTINCT、LIMIT、OFFSET 或等效的修改器时，完成的语句首先被包裹在一个子查询中，并且专门用于联接式的急加载的联接应用于子查询。SQLAlchemy 的联接式急加载会走出额外的一步，然后再走出额外的十步，绝对确保它不会影响查询的最终结果，只会影响集合和相关对象的加载方式，无论查询的格式如何。

另请参阅

将显式联接/语句路由到急加载集合 - 使用 `contains_eager()`  ## 选择 IN 加载

在大多数情况下，选择 IN 加载是急加载对象集合的最简单和最有效的方式。唯一一个不可行的情况是当模型使用复合主键，并且后端数据库不支持具有 IN 的元组时，这目前包括 SQL Server。

使用 `"selectin"` 参数提供了“选择 IN”急加载，或者通过使用 `selectinload()` 加载器选项。这种加载样式发出一个 SELECT，该 SELECT 引用父对象的主键值，或者在一对多关系的情况下引用子对象的主键值，以便在 IN 子句中加载相关联的关系：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import selectinload
>>> stmt = (
...     select(User)
...     .options(selectinload(User.addresses))
...     .filter(or_(User.name == "spongebob", User.name == "ed"))
... )
>>> result = session.scalars(stmt).all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
WHERE  users.name  =  ?  OR  users.name  =  ?
('spongebob',  'ed')
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id
FROM  addresses
WHERE  addresses.user_id  IN  (?,  ?)
(5,  7) 
```

上面，第二个 SELECT 是指 `addresses.user_id IN (5, 7)`，其中的 “5” 和 “7” 是之前加载的前两个 `User` 对象的主键值；在一批对象完全加载后，它们的主键值被注入到第二个 SELECT 的 `IN` 子句中。因为 `User` 和 `Address` 之间的关系有一个简单的主键连接条件，并且提供了 `User` 的主键值可以从 `Address.user_id` 派生出来，所以语句根本没有联接或子查询。

对于简单的一对多加载，也不需要使用 JOIN，因为会使用父对象的外键值：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Address).options(selectinload(Address.user))
>>> result = session.scalars(stmt).all()
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id
  FROM  addresses
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
WHERE  users.id  IN  (?,  ?)
(1,  2) 
```

提示

通过“简单”我们指的是 `relationship.primaryjoin` 条件表达了“一”侧的主键与“多”侧的直接外键之间的相等比较，没有任何额外的条件。

Select IN 加载还支持多对多关系，其中它当前将跨越所有三个表进行 JOIN，以将一侧的行与另一侧的行匹配。

关于这种加载方式需要知道的事情包括：

+   该策略每次会发出一个 SELECT 查询，最多查询 500 个父主键值，因为这些主键值会被渲染成 SQL 语句中的一个大型 IN 表达式。一些数据库，比如 Oracle，在 IN 表达式的大小上有一个硬限制，总体上 SQL 字符串的大小不应该是任意大的。

+   由于“selectin”加载依赖于 IN，在具有复合主键的映射中，它必须使用“元组”形式的 IN，看起来像 `WHERE (table.column_a, table.column_b) IN ((?, ?), (?, ?), (?, ?))`。这种语法目前不受 SQL Server 支持，对于 SQLite 需要至少版本 3.15。SQLAlchemy 中没有特殊逻辑来提前检查哪些平台支持这种语法，如果运行在不支持的平台上，数据库将立即返回错误。SQLAlchemy 只需运行 SQL 语句以使其失败的一个优点是，如果某个特定数据库开始支持这种语法，它将无需对 SQLAlchemy 进行任何更改即可工作（就像 SQLite 的情况一样）。## 子查询急加载

传统特性

`subqueryload()` 预加载器在这一点上主要是传统的，已被 `selectinload()` 策略取代，后者设计更简单，更灵活，具有诸如 Yield Per 等功能，并在大多数情况下发出更有效的 SQL 语句。由于 `subqueryload()` 依赖于重新解释原始的 SELECT 语句，当给定非常复杂的源查询时，可能无法有效地工作。

对于使用复合主键的对象的急加载集合的特定情况，`subqueryload()` 可能仍然有用，因为 Microsoft SQL Server 后端仍然不支持“元组 IN”语法。

子查询加载在操作上类似于 selectin 急加载，但发出的 SELECT 语句是从原始语句派生的，并且具有更复杂的查询结构，类似于 selectin 急加载。

通过在 `relationship.lazy` 中使用 `"subquery"` 参数或使用 `subqueryload()` 加载器选项提供子查询急加载。

子查询急加载的操作是为要加载的每个关系发出第二个 SELECT 语句，跨所有结果对象一次性加载。此 SELECT 语句引用原始 SELECT 语句，该语句包装在子查询中，以便我们检索要返回的主对象的相同主键列表，然后将其与要一次性加载的所有集合成员的总和链接起来：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import subqueryload
>>> stmt = select(User).options(subqueryload(User.addresses)).filter_by(name="spongebob")
>>> results = session.scalars(stmt).all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
WHERE  users.name  =  ?
('spongebob',)
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id,
  anon_1.users_id  AS  anon_1_users_id
FROM  (
  SELECT  users.id  AS  users_id
  FROM  users
  WHERE  users.name  =  ?)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id,  addresses.id
('spongebob',) 
```

关于这种加载方式需要知道的事项包括：

+   “子查询”加载策略发出的 SELECT 语句（与“selectin”的不同之处在于）需要一个子查询，并将继承原始查询中存在的任何性能限制。子查询本身也可能因为所使用的数据库的具体情况而产生性能惩罚。

+   “子查询”加载在正确工作时会施加一些特殊的排序要求。使用 `subqueryload()` 与诸如 `Select.limit()` 或 `Select.offset()` 这样的限定修饰符的查询应该**始终**包含针对唯一列（如主键）的 `Select.order_by()`，以便由 `subqueryload()` 发出的附加查询包含与父查询使用的相同排序。否则，内部查询可能会返回错误的行：

    ```py
    # incorrect, no ORDER BY
    stmt = select(User).options(subqueryload(User.addresses).limit(1))

    # incorrect if User.name is not unique
    stmt = select(User).options(subqueryload(User.addresses)).order_by(User.name).limit(1)

    # correct
    stmt = (
        select(User)
        .options(subqueryload(User.addresses))
        .order_by(User.name, User.id)
        .limit(1)
    )
    ```

    另请参阅

    为什么建议在 LIMIT 中使用 ORDER BY（特别是在 subqueryload() 中）？ - 详细示例

+   当在许多级别的深层急加载上使用时，“子查询”加载还会带来额外的性能/复杂性问题，因为子查询将被重复嵌套。

+   “子查询”加载与由 Yield Per 提供的“批量”加载（对集合和标量关系均适用）不兼容。

出于上述原因，“selectin”策略应优先于“子查询”。

另请参阅

选择 IN 加载  ## 何种加载方式？

使用哪种加载方式通常涉及到优化 SQL 执行次数、所发出 SQL 的复杂度以及获取的数据量之间的权衡。

**一对多 / 多对多集合** - `selectinload()` 通常是最佳的加载策略。它会发出额外的 SELECT 查询，尽可能使用少量的表，不影响原始语句，并且对于任何类型的原始查询都非常灵活。它唯一的主要限制是在使用不支持“tuple IN”的后端的复合主键表时，目前包括 SQL Server 和非常旧的 SQLite 版本；所有其他包含的后端都支持它。

**多对一** - `joinedload()` 策略是最通用的策略。在特殊情况下，如果潜在相关值数量很少，也可以使用`immediateload()` 策略，因为如果相关对象已经存在，则该策略将从本地`Session`获取对象，而不发出任何 SQL。

## 多态急加载

支持按急加载基础上的每个急加载选项进行多态选项的指定。参见 Eager Loading of Polymorphic Subtypes 部分中与`with_polymorphic()`函数配合使用的`PropComparator.of_type()`方法的示例。

## 通配符加载策略

`joinedload()`、`subqueryload()`、`lazyload()`、`selectinload()`、`noload()` 和`raiseload()` 中的每一个都可以用于为特定查询设置`relationship()`加载的默认样式，影响除了在语句中另有规定的所有 `relationship()` -映射属性。通过将字符串 `'*'` 作为这些选项中的任何一个的参数传递，可以使用此功能：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(MyClass).options(lazyload("*"))
```

在上面的例子中，`lazyload('*')` 选项将取代该查询中所有正在使用的所有 `relationship()` 构造的 `lazy` 设置，但不包括那些使用 `lazy='write_only'` 或 `lazy='dynamic'` 的构造。

如果某些关系指定了`lazy='joined'`或`lazy='selectin'`，例如，使用`lazyload('*')`将单方面地导致所有这些关系使用`'select'`加载，例如，当访问每个属性时发出 SELECT 语句。

该选项不会取代查询中声明的加载选项，例如`joinedload()`，`selectinload()`等。下面的查询仍将使用`widget`关系的 joined 加载：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload
from sqlalchemy.orm import joinedload

stmt = select(MyClass).options(lazyload("*"), joinedload(MyClass.widget))
```

尽管`joinedload()`的指令会发生，无论它出现在`lazyload()`选项之前还是之后，但如果传递了每个包含`"*"`的多个选项，则最后一个将生效。

### 按实体的通配符加载策略

通配符加载策略的变体是能够根据每个实体设置策略的能力。例如，如果查询`User`和`Address`，我们可以指示`Address`上的所有关系使用延迟加载，同时通过首先应用`Load`对象，然后指定`*`作为链接选项，保持对`User`的加载策略不受影响：

```py
from sqlalchemy import select
from sqlalchemy.orm import Load

stmt = select(User, Address).options(Load(Address).lazyload("*"))
```

上面，`Address`上的所有关系都将设置为延迟加载。  ## 将显式连接/语句路由到急加载集合

`joinedload()`的行为是自动创建连接，使用匿名别名作为目标，其结果被路由到加载对象上的集合和标量引用中。通常情况下，查询已经包括表示特定集合或标量引用的必要连接，并且 joinedload 功能添加的连接是多余的 - 但您仍希望填充集合/引用。

对此，SQLAlchemy 提供了`contains_eager()`选项。此选项的使用方式与`joinedload()`选项相同，不同之处在于假定`Select`对象将明确包含适当的连接，通常使用`Select.join()`等方法。下面，我们指定了`User`和`Address`之间的连接，并将其另外建立为`User.addresses`的急加载的基础：

```py
from sqlalchemy.orm import contains_eager

stmt = select(User).join(User.addresses).options(contains_eager(User.addresses))
```

如果语句的“eager”部分是“aliased”，则应使用`PropComparator.of_type()` 指定路径，这允许传递特定的`aliased()` 构造：

```py
# use an alias of the Address entity
adalias = aliased(Address)

# construct a statement which expects the "addresses" results

stmt = (
    select(User)
    .outerjoin(User.addresses.of_type(adalias))
    .options(contains_eager(User.addresses.of_type(adalias)))
)

# get results normally
r = session.scalars(stmt).unique().all()
SELECT
  users.user_id  AS  users_user_id,
  users.user_name  AS  users_user_name,
  adalias.address_id  AS  adalias_address_id,
  adalias.user_id  AS  adalias_user_id,
  adalias.email_address  AS  adalias_email_address,
  (...other  columns...)
FROM  users
LEFT  OUTER  JOIN  email_addresses  AS  email_addresses_1
ON  users.user_id  =  email_addresses_1.user_id 
```

作为参数给出的路径`contains_eager()` 需要是从起始实体开始的完整路径。例如，如果我们正在加载 `Users->orders->Order->items->Item` ，则选项将如下使用：

```py
stmt = select(User).options(contains_eager(User.orders).contains_eager(Order.items))
```

### 使用 contains_eager() 加载自定义过滤的集合结果

当我们使用`contains_eager()` 时，*我们*正在构造用于填充集合的 SQL。由此自然地可以选择**修改**要存储在集合中的值，通过编写 SQL 来加载集合或标量属性的子集。

提示

SQLAlchemy 现在有一种**更简单的方法**来做到这一点，它允许将 WHERE 条件直接添加到加载器选项中，例如`joinedload()` 和 `selectinload()` ，使用`PropComparator.and_()`。参见添加条件到加载器选项部分的示例。

此处描述的技术仍然适用于使用 SQL 条件或修饰符查询相关集合，而不仅仅是简单的 WHERE 子句。

例如，我们可以加载一个 `User` 对象，并通过过滤连接数据来将只特定地址急切地加载到其 `.addresses` 集合中，使用`contains_eager()` 路由，还使用 Populate Existing 确保任何已加载的集合都被覆盖：

```py
stmt = (
    select(User)
    .join(User.addresses)
    .filter(Address.email_address.like("%@aol.com"))
    .options(contains_eager(User.addresses))
    .execution_options(populate_existing=True)
)
```

上述查询将仅加载包含至少一个在其 `email` 字段中包含子字符串`'aol.com'`的 `Address` 对象的 `User` 对象；`User.addresses` 集合将仅包含这些 `Address` 条目，并且不包含实际与集合关联的任何其他 `Address` 条目。

提示

在所有情况下，SQLAlchemy ORM **不会覆盖已加载的属性和集合**，除非有指示要这样做。由于正在使用一个身份映射，通常情况下，ORM 查询返回的对象实际上已经存在并加载到内存中。因此，当使用`contains_eager()`以另一种方式填充集合时，通常最好像上面示例中所示那样使用填充现有，以便已加载的集合使用新数据进行刷新。`populate_existing` 选项将重置已经存在的**所有**属性，包括待处理的更改，因此在使用它之前确保所有数据都已刷新。使用带有其默认行为的`Session`，默认行为为自动刷新，已足够。

注意

我们使用`contains_eager()`加载的定制集合不是“粘性”的；也就是说，下次加载此集合时，它将使用其通常的默认内容加载。如果对象过期，则该集合可能会重新加载，这在默认会话设置下发生，即每当使用 `Session.commit()`、`Session.rollback()` 方法时，或者使用 `Session.expire_all()` 或 `Session.expire()` 方法。

参见

将条件添加到加载器选项 - 现代 API 允许在任何关系加载器选项中直接添加 WHERE 条件

## 关系加载器 API

| 对象名称 | 描述 |
| --- | --- |
| contains_eager(*keys, **kw) | 指示给定属性应通过手动在查询中声明的列进行急加载。 |
| defaultload(*keys) | 指示属性应使用其预定义的加载器样式加载。 |
| immediateload(*keys, [recursion_depth]) | 指示给定属性应使用立即加载，并使用每个属性的 SELECT 语句。 |
| joinedload(*keys, **kw) | 指示给定属性应使用连接的急加载。 |
| lazyload(*keys) | 指示给定属性应使用“懒”加载。 |
| Load | 表示加载器选项，可修改 ORM 启用的`Select`或传统`Query`的状态，以影响如何加载各种映射属性。 |
| noload(*keys) | 表示给定关系属性应该保持未加载状态。 |
| raiseload(*keys, **kw) | 表示访问给定属性时应引发错误。 |
| selectinload(*keys, [recursion_depth]) | 表示给定属性应该使用 SELECT IN 急加载加载。 |
| subqueryload(*keys) | 表示给定属性应该使用子查询急加载加载。 |

```py
function sqlalchemy.orm.contains_eager(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) → _AbstractLoad
```

表示应该从查询中手动声明的列急加载给定属性。

此函数是`Load`接口的一部分，支持方法链和独立操作。

此选项与加载所需行的显式连接一起使用，即：

```py
sess.query(Order).join(Order.user).options(
    contains_eager(Order.user)
)
```

上述查询将从`Order`实体连接到其相关的`User`实体，并且返回的`Order`对象将预先填充`Order.user`属性。

它还可用于自定义急加载集合中的条目；查询通常会希望使用 Populate Existing 执行选项，假设父对象的主要集合可能已经被加载：

```py
sess.query(User).join(User.addresses).filter(
    Address.email_address.like("%@aol.com")
).options(contains_eager(User.addresses)).populate_existing()
```

有关完整使用详细信息，请参阅将显式连接/语句路由到急加载集合部分。

另请参见

关系加载技术

将显式连接/语句路由到急加载集合

```py
function sqlalchemy.orm.defaultload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

表示属性应该使用其预定义的加载器样式加载。

此加载选项的行为是不更改属性的当前加载样式，这意味着将使用先前配置的样式，或者如果没有选择先前的样式，则将使用默认加载。

此方法用于进一步链接到属性链中的其他加载器选项，而不更改沿链的链接的加载器样式。例如，要为元素的元素设置连接的急加载：

```py
session.query(MyClass).options(
    defaultload(MyClass.someattribute).joinedload(
        MyOtherClass.someotherattribute
    )
)
```

`defaultload()` 也对于在相关类上设置列级选项很有用，即`defer()`和`undefer()`的选项：

```py
session.scalars(
    select(MyClass).options(
        defaultload(MyClass.someattribute)
        .defer("some_column")
        .undefer("some_other_column")
    )
)
```

另请参见

使用 Load.options()指定子选项

`Load.options()`

```py
function sqlalchemy.orm.immediateload(*keys: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → _AbstractLoad
```

表示给定属性应使用具有每个属性 SELECT 语句的立即加载进行加载。

加载是使用“lazyloader”策略实现的，并且不会触发任何其他急加载器。

`immediateload()` 选项一般被`selectinload()` 选项取代，后者通过为所有加载的对象发出一个 SELECT 更有效地执行相同的任务。

此函数是`Load`接口的一部分，支持方法链接和独立操作。

参数：

**递归深度** –

可选的整数；当与自引用关系一起设置为正整数时，表示“选择加载”将自动继续到没有找到项目为止的那么多级别深度。

注意

`immediateload.recursion_depth` 选项当前仅支持自引用关系。目前还没有选项可以自动遍历具有多个涉及的关系的递归结构。

警告

此参数是新的实验性参数，应视为“alpha”状态

新版 2.0 中新增 `immediateload.recursion_depth`

另请参阅

关联加载技术

选择 IN 加载

```py
function sqlalchemy.orm.joinedload(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) → _AbstractLoad
```

表示给定属性应使用连接式急加载进行加载。

此函数是`Load`接口的一部分，支持方法链接和独立操作。

例子：

```py
# joined-load the "orders" collection on "User"
select(User).options(joinedload(User.orders))

# joined-load Order.items and then Item.keywords
select(Order).options(
    joinedload(Order.items).joinedload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# joined-load the keywords collection
select(Order).options(
    lazyload(Order.items).joinedload(Item.keywords)
)
```

参数：

**innerjoin** –

如果为 `True`，则表示连接式急加载应使用内部连接而不是默认的左外连接：

```py
select(Order).options(joinedload(Order.user, innerjoin=True))
```

为了将多个急加载连接在一起，其中一些可能是 OUTER 而其他是 INNER，使用右嵌套连接将它们链接起来：

```py
select(A).options(
    joinedload(A.bs, innerjoin=False).joinedload(
        B.cs, innerjoin=True
    )
)
```

上述查询，通过“outer”连接链接 A.bs 和通过“inner”连接链接 B.cs，会呈现为“a LEFT OUTER JOIN (b JOIN c)” 的连接。当使用较旧版本的 SQLite (< 3.7.16) 时，此 JOIN 形式将转换为使用完整子查询，因为否则不直接支持此语法。

`innerjoin` 标志也可以用术语 `"unnested"` 表示。这表示应使用 INNER JOIN，*除非*连接链接到左边的 LEFT OUTER JOIN，此时它将呈现为 LEFT OUTER JOIN。例如，假设 `A.bs` 是一个 outerjoin：

```py
select(A).options(
    joinedload(A.bs).joinedload(B.cs, innerjoin="unnested")
)
```

上述连接将呈现为“a LEFT OUTER JOIN b LEFT OUTER JOIN c”，而不是“a LEFT OUTER JOIN (b JOIN c)”。

注意

“unnested”标志**不会**影响从多对多关联表（例如，配置为`relationship.secondary`的表）到目标表的 JOIN；为了结果的正确性，这些 JOIN 始终是 INNER JOIN，因此如果与 OUTER JOIN 相关联，则为右嵌套。

注意

`joinedload()`生成的 JOIN 是**匿名别名**的。 JOIN 进行的标准无法修改，也无法通过 ORM 启用的`Select`或传统的`Query`以任何方式引用这些 JOIN，包括排序。有关详细信息，请参阅关联及时加载的禅意。

要生成明确可用的特定 SQL JOIN，请使用`Select.join()`和`Query.join()`。要将明确的 JOIN 与集合的及时加载结合使用，请使用`contains_eager()`；参见将明确的 JOIN/语句路由到及时加载的集合中。

另请参阅

关系加载技术

关联及时加载

```py
function sqlalchemy.orm.lazyload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

表明给定的属性应该使用“懒加载”。

此函数是`Load`接口的一部分，支持方法链接和独立操作。

另请参阅

关系加载技术

懒加载

```py
class sqlalchemy.orm.Load
```

代表修改 ORM 启用的`Select`或传统的`Query`状态以影响加载各种映射属性的加载器选项。

当使用查询选项如`joinedload()`、`defer()`或类似选项时，`Load`对象在大多数情况下会在幕后隐式使用。除了一些非常特殊的情况外，通常不会直接实例化它。

另请参阅

每个实体通配符加载策略 - 演示了直接使用`Load`可能有用的一个示例

**成员**

contains_eager()，defaultload()，defer()，get_children()，immediateload()，inherit_cache，joinedload()，lazyload()，load_only()，noload()，options()，process_compile_state()，process_compile_state_replaced_entities()，propagate_to_loaders，raiseload()，selectin_polymorphic()，selectinload()，subqueryload()，undefer()，undefer_group()，with_expression()

**类签名**

类`sqlalchemy.orm.Load` (`sqlalchemy.orm.strategy_options._AbstractLoad`)

```py
method contains_eager(attr: _AttrType, alias: _FromClauseArgument | None = None, _is_chain: bool = False) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.contains_eager` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

生成一个应用了`contains_eager()`选项的新`Load`对象。

有关用法示例，请参见`contains_eager()`。

```py
method defaultload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.defaultload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

生成一个应用了`defaultload()`选项的新`Load`对象。

有关用法示例，请参见`defaultload()`。

```py
method defer(key: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.defer` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

生成一个应用了`defer()`选项的新`Load`对象。

有关用法示例，请参见`defer()`。

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*继承自* `HasTraverseInternals.get_children()` *方法的* `HasTraverseInternals`

返回此`HasTraverseInternals`的直接子`HasTraverseInternals`元素。

用于访问遍历。

**kw 可能包含更改返回集合的标志，例如返回子集以减少更大的遍历，或者返回来自不同上下文的子项（例如模式级集合而不是子句级集合）。

```py
method immediateload(attr: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.immediateload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`immediateload()`选项应用于生成新的`Load`对象。

查看`immediateload()`以获取用法示例。

```py
attribute inherit_cache: bool | None = None
```

*继承自* `HasCacheKey`的`HasCacheKey.inherit_cache` *属性*

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与对象对应的 SQL 不基于本类的本地属性而是基于其超类，则可以在特定类上将此标志设置为`True`。

另请参阅

为自定义构造启用缓存支持 - 为第三方或用户定义的 SQL 构造设置`HasCacheKey.inherit_cache`属性的一般指南。

```py
method joinedload(attr: Literal['*'] | QueryableAttribute[Any], innerjoin: bool | None = None) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.joinedload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`joinedload()`选项应用于生成新的`Load`对象。

查看`joinedload()`以获取用法示例。

```py
method lazyload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.lazyload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`lazyload()`选项应用于生成新的`Load`对象。

查看`lazyload()`以获取用法示例。

```py
method load_only(*attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.load_only` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用已应用了 `load_only()` 选项的新 `Load` 对象。

有关使用示例，请参见 `load_only()`。

```py
method noload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.noload` 方法 `sqlalchemy.orm.strategy_options._AbstractLoad`

使用已应用了 `noload()` 选项的新 `Load` 对象。

有关使用示例，请参见 `noload()`。

```py
method options(*opts: _AbstractLoad) → Self
```

将一系列选项应用为此 `Load` 对象的子选项。

例如：

```py
query = session.query(Author)
query = query.options(
            joinedload(Author.book).options(
                load_only(Book.summary, Book.excerpt),
                joinedload(Book.citations).options(
                    joinedload(Citation.author)
                )
            )
        )
```

参数：

***opts** – 应用于此 `Load` 对象指定路径的一系列加载器选项对象（最终为 `Load` 对象）。

自版本 1.3.6 新增。

另请参阅

`defaultload()`

使用 Load.options() 指定子选项

```py
method process_compile_state(compile_state: ORMCompileState) → None
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.process_compile_state` 方法 `sqlalchemy.orm.strategy_options._AbstractLoad`

对给定的 `ORMCompileState` 应用修改。

此方法是特定 `CompileStateOption` 实现的一部分，并且仅在编译 ORM 查询时内部调用。

```py
method process_compile_state_replaced_entities(compile_state: ORMCompileState, mapper_entities: Sequence[_MapperEntity]) → None
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.process_compile_state_replaced_entities` 方法 `sqlalchemy.orm.strategy_options._AbstractLoad`

对给定的 `ORMCompileState` 应用修改，给定了由 with_only_columns() 或 with_entities() 替换的实体。

此方法是特定 `CompileStateOption` 实现的一部分，并且仅在编译 ORM 查询时内部调用。

自版本 1.4.19 新增。

```py
attribute propagate_to_loaders: bool
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.propagate_to_loaders` 的属性 `sqlalchemy.orm.strategy_options._AbstractLoad`

如果为 True，则指示此选项应该随着“次要”SELECT 语句一起传递，这些语句会出现在关系惰性加载器以及属性加载/刷新操作中。

```py
method raiseload(attr: Literal['*'] | QueryableAttribute[Any], sql_only: bool = False) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.raiseload` 方法 `sqlalchemy.orm.strategy_options._AbstractLoad`

使用已应用了 `raiseload()` 选项的新 `Load` 对象。

有关使用示例，请参见 `raiseload()`。

```py
method selectin_polymorphic(classes: Iterable[Type[Any]]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.selectin_polymorphic` 方法 `sqlalchemy.orm.strategy_options._AbstractLoad`

使用 `Load` 对象，并应用 `selectin_polymorphic()` 选项。

查看 `selectin_polymorphic()` 以获取使用示例。

```py
method selectinload(attr: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.selectinload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用 `Load` 对象，并应用 `selectinload()` 选项。

查看 `selectinload()` 以获取使用示例。

```py
method subqueryload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.subqueryload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用 `Load` 对象，并应用 `subqueryload()` 选项。

查看 `subqueryload()` 以获取使用示例。

```py
method undefer(key: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.undefer` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用 `Load` 对象，并应用 `undefer()` 选项。

查看 `undefer()` 以获取使用示例。

```py
method undefer_group(name: str) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.undefer_group` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用 `Load` 对象，并应用 `undefer_group()` 选项。

查看 `undefer_group()` 以获取使用示例。

```py
method with_expression(key: _AttrType, expression: _ColumnExpressionArgument[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.with_expression` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用 `Load` 对象，并应用 `with_expression()` 选项。

查看 `with_expression()` 以获取使用示例。

```py
function sqlalchemy.orm.noload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

指示给定的关系属性应保持未加载状态。

当访问关系属性而不产生任何加载效果时，关系属性将返回 `None`。

此函数是 `Load` 接口的一部分，并支持方法链式和独立操作。

`noload()` 仅适用于 `relationship()` 属性。

注意

将此加载策略设置为使用 `relationship.lazy` 参数的默认策略可能会导致刷新时出现问题，比如删除操作需要加载相关对象，而返回的却是 `None`。

另请参阅

关系加载技术

```py
function sqlalchemy.orm.raiseload(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) → _AbstractLoad
```

表示如果访问将引发错误的给定属性。

使用 `raiseload()` 配置的关系属性在访问时会引发 `InvalidRequestError`。此方法通常有用的方式是，当应用程序试图确保在特定上下文中访问的所有关系属性都已通过激进加载加载时。而不是必须阅读 SQL 日志以确保不发生懒加载，此策略将立即导致它们引发。

`raiseload()` 仅适用于 `relationship()` 属性。要将 raise-on-SQL 行为应用于基于列的属性，请在 `defer()` 加载选项的 `defer.raiseload` 参数上使用。

参数：

**sql_only** – 如果为 True，则仅在懒加载将发出 SQL 时引发，但如果仅检查标识映射或确定由于缺少键而相关值应为 None，则不会引发。当为 False 时，该策略将引发所有类型的关系加载。

此函数是 `Load` 接口的一部分，并支持方法链和独立操作。

另请参阅

关系加载技术

使用 raiseload 避免不必要的懒加载

使用 raiseload 避免延迟列加载

```py
function sqlalchemy.orm.selectinload(*keys: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → _AbstractLoad
```

表示给定属性应使用 SELECT IN 激进加载。

此函数是 `Load` 接口的一部分，并支持方法链和独立操作。

示例：

```py
# selectin-load the "orders" collection on "User"
select(User).options(selectinload(User.orders))

# selectin-load Order.items and then Item.keywords
select(Order).options(
    selectinload(Order.items).selectinload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# selectin-load the keywords collection
select(Order).options(
    lazyload(Order.items).selectinload(Item.keywords)
)
```

参数：

**recursion_depth** –

可选整数；与自引用关系结合设置为正整数时，指示“选择加载”将自动继续加载到没有找到项目为止的那么多级。

注意

`selectinload.recursion_depth` 选项目前仅支持自引用关系。目前还没有自动遍历涉及多个关系的递归结构的选项。

此外，`selectinload.recursion_depth` 参数是新的实验性参数，应该在 2.0 系列中视为“alpha”状态。

2.0 版本新增：添加了`selectinload.recursion_depth`

另请参阅

关系加载技术

选择 IN 加载

```py
function sqlalchemy.orm.subqueryload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

表示应使用子查询急加载加载给定属性。

此函数是`Load`接口的一部分，支持方法链接和独立操作。

示例：

```py
# subquery-load the "orders" collection on "User"
select(User).options(subqueryload(User.orders))

# subquery-load Order.items and then Item.keywords
select(Order).options(
    subqueryload(Order.items).subqueryload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# subquery-load the keywords collection
select(Order).options(
    lazyload(Order.items).subqueryload(Item.keywords)
)
```

另请参阅

关系加载技术

子查询急加载

## 关系加载样式总结

关系加载的主要形式包括：

+   **延迟加载** - 通过`lazy='select'`或`lazyload()` 选项可用，这是在属性访问时发出 SELECT 语句以延迟加载单个对象上的相关引用的加载形式。延迟加载是所有未指示`relationship.lazy` 选项的所有`relationship()` 构造的**默认加载样式**。延迟加载详细信息请参阅延迟加载。

+   **选择 IN 加载** - 通过`lazy='selectin'`或`selectinload()` 选项可用，此加载形式发出第二个（或更多）SELECT 语句，将父对象的主键标识符组装成一个 IN 子句，以便通过主键一次加载所有相关集合/标量引用的成员。选择 IN 加载详细信息请参阅选择 IN 加载。

+   **连接加载** - 通过`lazy='joined'`或`joinedload()` 选项可用，此加载形式将 JOIN 应用于给定的 SELECT 语句，以便相关行在同一结果集中加载。连接急加载详细信息请参阅连接急加载。

+   **抛出加载** - 可通过`lazy='raise'`、`lazy='raise_on_sql'`或`raiseload()`选项使用，这种加载方式在通常发生惰性加载的同时触发，但会引发 ORM 异常，以防止应用程序进行不必要的惰性加载。关于抛出加载的介绍请参阅使用 raiseload 防止不必要的惰性加载。

+   **子查询加载** - 可通过`lazy='subquery'`或`subqueryload()`选项使用，这种加载方式会发出第二个 SELECT 语句，该语句重新陈述原始查询嵌入到子查询中，然后将该子查询与相关表进行 JOIN，以一次加载所有相关集合/标量引用的成员。子查询急切加载的详细信息请参阅子查询急切加载。

+   **仅写加载** - 可通过`lazy='write_only'`使用，或者通过使用`Relationship`对象的左侧进行注释，并使用`WriteOnlyMapped`注解。这种仅限集合加载样式会产生一种替代的属性仪器，从不从数据库隐式加载记录，而是仅允许`WriteOnlyCollection.add()`、`WriteOnlyCollection.add_all()`和`WriteOnlyCollection.remove()`方法。通过调用使用`WriteOnlyCollection.select()`方法构造的 SELECT 语句来查询集合。关于仅写加载的讨论请参阅仅写关系。

+   **动态加载** - 通过`lazy='dynamic'`可用，或者通过使用`DynamicMapped`注释来注释`Relationship`对象的左侧。这是一种传统的仅集合加载器样式，当访问集合时会生成一个`Query`对象，允许针对集合内容发出自定义 SQL。然而，动态加载器在各种情况下会隐式迭代底层集合，这使得它们对于管理真正大型集合不太有用。动态加载器被“仅写入”集合取代，这将阻止在任何情况下隐式加载底层集合。动态加载器在动态关系加载器中讨论。

## 在映射时间配置加载策略

特定关系的加载策略可以在映射时间配置，以在加载映射类型的对象的所有情况下发生，没有任何修改它的查询级选项的情况下。这是使用`relationship()`的`relationship.lazy`参数进行配置的；该参数的常见值包括`select`、`selectin`和`joined`。

下面的示例说明了在一对多关系示例中，配置`Parent.children`关系以在发出`Parent`对象的 SELECT 语句时使用 Select IN loading：

```py
from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Parent(Base):
    __tablename__ = "parent"

    id: Mapped[int] = mapped_column(primary_key=True)
    children: Mapped[List["Child"]] = relationship(lazy="selectin")

class Child(Base):
    __tablename__ = "child"

    id: Mapped[int] = mapped_column(primary_key=True)
    parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))
```

在上述情况中，每当加载一组`Parent`对象时，每个`Parent`对象的`children`集合也会被填充，使用`"selectin"`加载策略会发出第二个查询。

`relationship.lazy`参数的默认值是`"select"`，表示懒加载。

## 使用加载器选项加载关系

配置加载策略的另一种，可能更常见的方式是针对特定属性在每个查询上设置它们，使用 `Select.options()` 方法。使用加载器选项可以对关系加载进行非常详细的控制；最常见的是 `joinedload()`、`selectinload()` 和 `lazyload()`。该选项接受一个类绑定的属性，引用应该被定位的特定类/属性：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

# set children to load lazily
stmt = select(Parent).options(lazyload(Parent.children))

from sqlalchemy.orm import joinedload

# set children to load eagerly with a join
stmt = select(Parent).options(joinedload(Parent.children))
```

加载器选项也可以使用 **方法链** 进行“链接”，以指定加载应如何进行更深层次的操作：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload

stmt = select(Parent).options(
    joinedload(Parent.children).subqueryload(Child.subelements)
)
```

可以将链接的加载器选项应用于“惰性”加载的集合。这意味着当在访问时惰性加载集合或关联时，指定的选项将立即生效：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))
```

上面的查询将返回未加载 `Parent` 对象的 `children` 集合。当首次访问特定 `Parent` 对象上的 `children` 集合时，它将惰性加载相关对象，但还将对 `children` 中的每个成员的 `subelements` 集合应用急切加载。

### 向加载器选项添加条件

用于指示加载器选项的关系属性包括向创建的联接的 ON 子句或涉及的 WHERE 条件添加额外的筛选条件的能力，具体取决于加载器策略。这可以通过使用 `PropComparator.and_()` 方法来实现，该方法将通过一个选项，使加载的结果限制为给定的筛选条件：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(A).options(lazyload(A.bs.and_(B.id > 5)))
```

在使用限制条件时，如果特定集合已加载，则不会刷新；为了确保新条件生效，请应用 Populate Existing 执行选项：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = (
    select(A)
    .options(lazyload(A.bs.and_(B.id > 5)))
    .execution_options(populate_existing=True)
)
```

为了对查询中的所有实体的所有出现都添加筛选条件，无论加载策略如何或它在加载过程中的位置如何，请参阅 `with_loader_criteria()` 函数。

新版本 1.4 特性。### 使用 Load.options() 指定子选项

使用方法链，显式声明路径中每个链接的加载器样式。要沿着路径导航而不更改特定属性的现有加载器样式，可以使用 `defaultload()` 方法/函数：

```py
from sqlalchemy import select
from sqlalchemy.orm import defaultload

stmt = select(A).options(defaultload(A.atob).joinedload(B.btoc))
```

可以使用 `Load.options()` 方法一次指定多个子选项的类似方法：

```py
from sqlalchemy import select
from sqlalchemy.orm import defaultload
from sqlalchemy.orm import joinedload

stmt = select(A).options(
    defaultload(A.atob).options(joinedload(B.btoc), joinedload(B.btod))
)
```

另见

在相关对象和集合上使用 `load_only()` - 举例说明了如何结合关系和基于列的加载器选项。

注意

应用于对象的惰性加载集合的加载器选项对于特定对象实例是 **“粘性的”**，这意味着它们将在内存中存在的时间内持续存在于由该特定对象加载的集合上。例如，考虑到上一个例子：

```py
stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))
```

如果上述查询加载的特定 `Parent` 对象上的 `children` 集合已过期（例如当 `Session` 对象的事务提交或回滚时，或者使用了 `Session.expire_all()`），当下次访问 `Parent.children` 集合以重新加载时，`Child.subelements` 集合将再次使用子查询急加载。即使上述 `Parent` 对象是从指定了不同选项集的后续查询中访问的，情况仍然如此。要在不清除并重新加载现有对象的情况下更改选项，必须使用 Populate Existing 执行选项显式设置它们：

```py
# change the options on Parent objects that were already loaded
stmt = (
    select(Parent)
    .execution_options(populate_existing=True)
    .options(lazyload(Parent.children).lazyload(Child.subelements))
    .all()
)
```

如果上面加载的对象被完全从 `Session` 中清除，例如由于垃圾回收或使用了 `Session.expunge_all()`，则 “粘性” 选项也将消失，并且新创建的对象将在再次加载时使用新选项。

未来的 SQLAlchemy 版本可能会增加更多操作已加载对象的加载器选项的替代方法。### 向加载器选项添加条件

用于指示加载器选项的关系属性包括在创建的联接的 ON 子句或涉及的 WHERE 条件中添加附加过滤条件的能力，具体取决于加载器策略。这可以通过使用 `PropComparator.and_()` 方法来实现，该方法将通过选项传递，从而将加载的结果限制为给定的过滤条件：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(A).options(lazyload(A.bs.and_(B.id > 5)))
```

在使用限制条件时，如果特定集合已经加载，它将不会被刷新；为了确保新的条件生效，应用 Populate Existing 执行选项：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = (
    select(A)
    .options(lazyload(A.bs.and_(B.id > 5)))
    .execution_options(populate_existing=True)
)
```

为了向查询中的实体的所有出现添加过滤条件，无论加载策略如何或出现在加载过程中的位置如何，请参阅 `with_loader_criteria()` 函数。

1.4 版本中的新功能。

### 使用 Load.options() 指定子选项

使用方法链时，路径中每个链接的加载器样式都会明确说明。要沿着路径导航而不更改特定属性的现有加载器样式，可以使用`defaultload()`方法/函数：

```py
from sqlalchemy import select
from sqlalchemy.orm import defaultload

stmt = select(A).options(defaultload(A.atob).joinedload(B.btoc))
```

可以使用`Load.options()`方法一次性指定多个子选项的类似方法：

```py
from sqlalchemy import select
from sqlalchemy.orm import defaultload
from sqlalchemy.orm import joinedload

stmt = select(A).options(
    defaultload(A.atob).options(joinedload(B.btoc), joinedload(B.btod))
)
```

另请参阅

在相关对象和集合上使用 load_only() - 展示了结合关系和基于列的加载器选项的示例。

注意

对象的延迟加载集合上应用的加载器选项是**“粘性”的**，即它们将持续存在于内存中的特定对象实例上加载的集合上。例如，给定上面的例子：

```py
stmt = select(Parent).options(lazyload(Parent.children).subqueryload(Child.subelements))
```

如果上述查询加载的特定`Parent`对象上的`children`集合过期（例如，当`Session`对象的事务被提交或回滚，或使用`Session.expire_all()`时），当下次访问`Parent.children`集合以重新加载时，`Child.subelements`集合将再次使用子查询的急加载。即使从指定了不同选项集的后续查询中访问了上述`Parent`对象，这种情况也会保持不变。要在不清除并重新加载对象的情况下更改现有对象上的选项，必须结合使用填充现有执行选项显式设置它们：

```py
# change the options on Parent objects that were already loaded
stmt = (
    select(Parent)
    .execution_options(populate_existing=True)
    .options(lazyload(Parent.children).lazyload(Child.subelements))
    .all()
)
```

如果上面加载的对象完全从`Session`中清除，例如由于垃圾回收或使用`Session.expunge_all()`，那么“粘性”选项也将消失，如果再次加载，则新创建的对象将使用新选项。

未来的 SQLAlchemy 版本可能会添加更多的选择来操作已加载对象的加载器选项。

## 延迟加载

默认情况下，所有对象间的关系都是**延迟加载**的。与`relationship()`相关的标量或集合属性包含一个触发器，第一次访问属性时触发。这个触发器通常在访问点发出 SQL 调用，以加载相关的对象：

```py
>>> spongebob.addresses
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id
FROM  addresses
WHERE  ?  =  addresses.user_id
[5]
[<Address(u'spongebob@google.com')>, <Address(u'j25@yahoo.com')>]
```

唯一一个不发出 SQL 的情况是简单的多对一关系，当相关对象仅可通过其主键标识且该对象已经存在于当前 `Session` 中时。因此，尽管延迟加载对于相关集合可能很昂贵，在加载许多对象与相对较小的可能目标对象集合相比，如果一个对象加载了很多对象，则可能可以在本地引用这些对象，而不像有多少父对象就发出多少 SELECT 语句。

“在属性访问时加载”的默认行为称为“延迟”或“选择”加载 - 名称“选择”是因为通常在首次访问属性时会发出“SELECT”语句。

可以使用 `lazyload()` 加载器选项为通常以其他方式配置的给定属性启用延迟加载：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

# force lazy loading for an attribute that is set to
# load some other way normally
stmt = select(User).options(lazyload(User.addresses))
```

### 使用 raiseload 防止不必要的延迟加载

`lazyload()` 策略产生的效果是对象关系映射中最常见的问题之一；N 加一问题，即对于加载的任何 N 个对象，访问它们的延迟加载属性意味着将会发出 N+1 个 SELECT 语句。在 SQLAlchemy 中，解决 N 加一问题的常规方法是利用其非常强大的急切加载系统。然而，急切加载要求提前使用 `Select` 指定要加载的属性。对于可能访问未急切加载的其他属性的代码，不希望进行延迟加载，可以使用 `raiseload()` 策略来解决；此加载器策略将延迟加载的行为替换为引发信息性错误：

```py
from sqlalchemy import select
from sqlalchemy.orm import raiseload

stmt = select(User).options(raiseload(User.addresses))
```

上面，从上述查询加载的 `User` 对象不会加载 `.addresses` 集合；如果稍后的某些代码尝试访问此属性，则会引发 ORM 异常。

`raiseload()` 可以与所谓的“通配符”说明符一起使用，以指示所有关系应使用此策略。例如，只设置一个属性为急切加载，而所有其他属性都为提升：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import raiseload

stmt = select(Order).options(joinedload(Order.items), raiseload("*"))
```

上述通配符将适用于**所有**关系，不仅限于除 `items` 外的 `Order`，还包括 `Item` 对象上的所有关系。要为仅 `Order` 对象设置 `raiseload()`，请使用 `Load` 指定完整路径：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import Load

stmt = select(Order).options(joinedload(Order.items), Load(Order).raiseload("*"))
```

相反，要为 `Item` 对象设置提升：

```py
stmt = select(Order).options(joinedload(Order.items).raiseload("*"))
```

`raiseload()` 选项仅适用于关系属性。对于面向列的属性，`defer()` 选项支持 `defer.raiseload` 选项，其工作方式相同。

提示

“raiseload”策略**不适用**于工作单元刷新过程中。这意味着如果 `Session.flush()` 过程需要加载集合以完成其工作，它将在绕过任何 `raiseload()` 指令的情况下执行此操作。

另请参阅

通配符加载策略

使用 raiseload 防止延迟列加载  ### 使用 raiseload 防止不需要的延迟加载

`lazyload()` 策略产生了一个最常见的对象关系映射中提到的问题之一的效果；N 加一问题，它说明对于加载的任何 N 个对象，访问它们的延迟加载属性意味着会有 N+1 个 SELECT 语句被发送。在 SQLAlchemy 中，对 N+1 问题的常规缓解方法是利用其非常强大的急切加载系统。然而，急切加载要求在前面指定要加载的属性。对于不希望进行延迟加载的其他属性的代码问题，可以使用 `raiseload()` 策略来解决；此加载器策略用具有信息性错误引发替换了延迟加载的行为：

```py
from sqlalchemy import select
from sqlalchemy.orm import raiseload

stmt = select(User).options(raiseload(User.addresses))
```

以上，从上述查询中加载的`User`对象不会加载`.addresses`集合；如果稍后的一些代码尝试访问此属性，则会引发 ORM 异常。

可以使用 `raiseload()` 与所谓的“通配符”指示符一起使用，以指示所有关系都应使用此策略。例如，要设置仅一个属性作为急切加载，而其他所有属性都作为 raise：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import raiseload

stmt = select(Order).options(joinedload(Order.items), raiseload("*"))
```

上述通配符将适用于**所有**关系，而不仅限于 `Order` 除 `items` 之外，还包括 `Item` 对象上的所有关系。要仅为 `Order` 对象设置 `raiseload()`，请指定带有 `Load` 的完整路径：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload
from sqlalchemy.orm import Load

stmt = select(Order).options(joinedload(Order.items), Load(Order).raiseload("*"))
```

相反，要仅为 `Item` 对象设置 raise：

```py
stmt = select(Order).options(joinedload(Order.items).raiseload("*"))
```

`raiseload()`选项仅适用于关系属性。对于面向列的属性，`defer()`选项支持`defer.raiseload`选项，其工作方式相同。

提示

“raiseload”策略**不适用**于 unit of work 提交过程中。这意味着如果`Session.flush()`过程需要加载一个集合以完成其工作，它将通过任何`raiseload()`指令绕过这样做。

另请参阅

通配符加载策略

使用 raiseload 防止延迟列加载

## 连接式急加载

连接式急加载是 SQLAlchemy ORM 包含的最古老的急加载样式。它通过将 JOIN（默认为 LEFT OUTER join）连接到发出的 SELECT 语句，并从与父级相同的结果集填充目标标量/集合来工作。

在映射级别，这看起来像是：

```py
class Address(Base):
    # ...

    user: Mapped[User] = relationship(lazy="joined")
```

连接式急加载通常作为查询的选项应用，而不是作为映射的默认加载选项，特别是当用于集合而不是多对一引用时。这通过使用`joinedload()`加载器选项来实现：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import joinedload
>>> stmt = select(User).options(joinedload(User.addresses)).filter_by(name="spongebob")
>>> spongebob = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
['spongebob'] 
```

提示

当涉及到一对多或多对多集合时，包括`joinedload()`时，必须对返回的结果应用`Result.unique()`方法，该方法将通过主键使传入的行唯一化，否则会被联接乘以。如果没有这个方法，ORM 将引发错误。

这在现代 SQLAlchemy 中不是自动的，因为它改变了结果集的行为，以返回比语句通常返回的 ORM 对象少的行数。因此，SQLAlchemy 保持了对`Result.unique()`的使用明确，这样就不会产生返回的对象在主键上的唯一性。

默认情况下发出的 JOIN 是一个 LEFT OUTER JOIN，以允许引用一个不存在相关行的主对象。对于保证具有元素的属性，例如对一个相关对象的多对一引用，其中引用的外键不为 NULL，通过使用内连接可以使查询更有效率；这可以通过映射级别的`relationship.innerjoin`标志来实现：

```py
class Address(Base):
    # ...

    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    user: Mapped[User] = relationship(lazy="joined", innerjoin=True)
```

在查询选项级别，通过`joinedload.innerjoin`标志：

```py
from sqlalchemy import select
from sqlalchemy.orm import joinedload

stmt = select(Address).options(joinedload(Address.user, innerjoin=True))
```

当在包含 OUTER JOIN 的链中应用时，JOIN 将会右嵌套自身：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import joinedload
>>> stmt = select(User).options(
...     joinedload(User.addresses).joinedload(Address.widgets, innerjoin=True)
... )
>>> results = session.scalars(stmt).unique().all()
SELECT
  widgets_1.id  AS  widgets_1_id,
  widgets_1.name  AS  widgets_1_name,
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
LEFT  OUTER  JOIN  (
  addresses  AS  addresses_1  JOIN  widgets  AS  widgets_1  ON
  addresses_1.widget_id  =  widgets_1.id
)  ON  users.id  =  addresses_1.user_id 
```

提示

如果在发出 SELECT 时使用数据库行锁定技术，这意味着使用`Select.with_for_update()`方法来发出 SELECT..FOR UPDATE，那么根据所使用的后端的行为，连接表也可能被锁定。基于这个原因，不建议同时使用联接式急加载和 SELECT..FOR UPDATE。

### 联接式急加载的禅意

由于联接式急加载似乎与`Select.join()`的使用有很多相似之处，因此经常会产生何时以及如何使用它的混淆。重要的是要理解这样一个区别，即虽然`Select.join()`用于修改查询的结果，但`joinedload()`竭尽全力**不**修改查询的结果，而是隐藏渲染联接的效果，以便仅允许相关对象存在。

加载策略背后的哲学是，任何一组加载方案都可以应用于特定的查询，并且*结果不会改变* - 只有用于完全加载相关对象和集合的 SQL 语句数量会改变。一个特定的查询可能起初使用了所有的延迟加载。在上下文中使用后，可能会发现特定的属性或集合总是被访问，更改这些属性的加载器策略将更有效率。策略可以更改而不影响查询的其他部分，结果将保持不变，但 SQL 语句数量会减少。理论上（而且在实践中几乎是如此），对`Select`所做的任何操作都不会因为加载器策略的改变而使其基于不同的一组主对象或相关对象加载不同的集合。

特别是 `joinedload()` 如何实现这一结果不以任何方式影响返回的实体行，它创建了查询中添加的连接的匿名别名，以便它们不能被查询的其他部分引用。例如，下面的查询使用 `joinedload()` 创建了从 `users` 到 `addresses` 的 LEFT OUTER JOIN，然而对 `Address.email_address` 添加的 `ORDER BY` 是无效的 - 查询中没有命名 `Address` 实体：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import joinedload
>>> stmt = (
...     select(User)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address  <-- this part is wrong !
['spongebob'] 
```

上面，`ORDER BY addresses.email_address` 是无效的，因为 `addresses` 不在 FROM 列表中。加载 `User` 记录并按电子邮件地址排序的正确方法是使用 `Select.join()`：

```py
>>> from sqlalchemy import select
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
JOIN  addresses  ON  users.id  =  addresses.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address
['spongebob'] 
```

当然，上面的语句与之前的语句不同，因为根本没有包含来自 `addresses` 的列在结果中。我们可以添加 `joinedload()` 回来，这样就有了两个连接 - 一个是我们正在排序的连接，另一个是匿名使用的，用于加载 `User.addresses` 集合的内容：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users  JOIN  addresses
  ON  users.id  =  addresses.user_id
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address
['spongebob'] 
```

我们在上面看到 `Select.join()` 的用法是提供我们希望在后续查询条件中使用的 JOIN 子句，而我们对 `joinedload()` 的用法只关注于为结果中的每个 `User` 加载 `User.addresses` 集合。在这种情况下，这两个连接很可能是多余的 - 而事实上它们确实是。如果我们只想使用一个 JOIN 来加载集合并排序，我们可以使用 `contains_eager()` 选项，下面描述了 将明确的 JOIN/语句路由到急切加载的集合。但要了解为什么 `joinedload()` 所做的事情，请考虑如果我们**过滤**某个特定的 `Address`：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .filter(Address.email_address == "someaddress@foo.com")
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users  JOIN  addresses
  ON  users.id  =  addresses.user_id
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?  AND  addresses.email_address  =  ?
['spongebob',  'someaddress@foo.com'] 
```

上面我们可以看到，这两个 JOIN 扮演着非常不同的角色。其中一个将精确匹配一行，即 `User` 和 `Address` 的连接，其中 `Address.email_address=='someaddress@foo.com'`。另一个 LEFT OUTER JOIN 将匹配与 `User` 相关的*所有* `Address` 行，并且仅用于填充返回的那些 `User` 对象的 `User.addresses` 集合。

通过将 `joinedload()` 的使用方式更改为另一种加载方式，我们可以完全独立于用于检索实际所需的 `User` 行的 SQL，改变集合的加载方式。以下我们将 `joinedload()` 改为 `selectinload()`：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(selectinload(User.addresses))
...     .filter(User.name == "spongebob")
...     .filter(Address.email_address == "someaddress@foo.com")
... )
>>> result = session.scalars(stmt).all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
JOIN  addresses  ON  users.id  =  addresses.user_id
WHERE
  users.name  =  ?
  AND  addresses.email_address  =  ?
['spongebob',  'someaddress@foo.com']
#  ...  selectinload()  emits  a  SELECT  in  order
#  to  load  all  address  records  ... 
```

当使用连接式贪婪加载时，如果查询包含影响外部连接返回行的修饰符，例如使用 DISTINCT、LIMIT、OFFSET 或等效操作，完成的语句首先被包装在一个子查询中，连接专门用于连接式贪婪加载被应用于子查询。SQLAlchemy 的连接式贪婪加载额外努力，然后再努力十英里，绝对确保它不会影响查询的最终结果，只影响集合和相关对象的加载方式，无论查询的格式如何。

另请参阅

将显式连接/语句路由到贪婪加载的集合 - 使用 `contains_eager()`  ### 连接式贪婪加载的禅意

由于连接式贪婪加载似乎与 `Select.join()` 的使用方式有很多相似之处，因此在何时以及如何使用它经常会产生困惑。重要的是要理解，虽然 `Select.join()` 用于更改查询的结果，但 `joinedload()` 却极力避免更改查询的结果，而是隐藏渲染连接的效果，以允许相关对象存在。

装载策略背后的哲学是，任何一组装载方案都可以应用于特定的查询，并且*结果不会改变*——只有完全加载相关对象和集合所需的 SQL 语句数量会改变。一个特定的查询可能首先使用所有的延迟加载。在上下文中使用后，可能会发现特定属性或集合总是被访问，并且更改这些的加载策略会更有效。该策略可以在不修改查询的其他部分的情况下更改，结果将保持相同，但会发出更少的 SQL 语句。理论上（实际上基本如此），无论你对 `Select` 做什么修改，都不会使其根据加载策略的变化加载不同的主要或相关对象集合。

如何使用`joinedload()`来实现不影响返回的实体行的结果，它的特点是创建查询中添加的连接的匿名别名，以便其他查询的部分不能引用它们。例如，下面的查询使用`joinedload()`创建了一个从`users`到`addresses`的左外连接，但是针对`Address.email_address`添加的`ORDER BY`是无效的 - 查询中未命名`Address`实体：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import joinedload
>>> stmt = (
...     select(User)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address  <-- this part is wrong !
['spongebob'] 
```

上述中，`ORDER BY addresses.email_address`是无效的，因为`addresses`不在 FROM 列表中。加载`User`记录并按电子邮件地址排序的正确方法是使用`Select.join()`：

```py
>>> from sqlalchemy import select
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
JOIN  addresses  ON  users.id  =  addresses.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address
['spongebob'] 
```

当然，上面的语句与前面的语句不同，因为根本没有包含来自`addresses`的列在结果中。我们可以重新添加`joinedload()`，以便有两个连接 - 一个是我们正在排序的连接，另一个是匿名使用的，用于加载`User.addresses`集合的内容：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .order_by(Address.email_address)
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users  JOIN  addresses
  ON  users.id  =  addresses.user_id
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?
ORDER  BY  addresses.email_address
['spongebob'] 
```

以上我们看到，我们使用`Select.join()`来提供我们希望在随后的查询条件中使用的 JOIN 子句，而我们使用`joinedload()`只关心加载每个结果中的`User.addresses`集合。在这种情况下，这两个连接很可能是多余的 - 它们确实是。如果我们只想使用一个 JOIN 来加载集合并排序，我们可以使用`contains_eager()`选项，下面描述了将显式的连接/语句路由到急加载的集合。但要看看为什么`joinedload()`会做它的工作，考虑一下如果我们**过滤**特定的`Address`：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(joinedload(User.addresses))
...     .filter(User.name == "spongebob")
...     .filter(Address.email_address == "someaddress@foo.com")
... )
>>> result = session.scalars(stmt).unique().all()
SELECT
  addresses_1.id  AS  addresses_1_id,
  addresses_1.email_address  AS  addresses_1_email_address,
  addresses_1.user_id  AS  addresses_1_user_id,
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users  JOIN  addresses
  ON  users.id  =  addresses.user_id
LEFT  OUTER  JOIN  addresses  AS  addresses_1
  ON  users.id  =  addresses_1.user_id
WHERE  users.name  =  ?  AND  addresses.email_address  =  ?
['spongebob',  'someaddress@foo.com'] 
```

上面，我们可以看到这两个 JOIN 有非常不同的角色。一个将精确匹配一个行，即`User`和`Address`的连接，其中`Address.email_address=='someaddress@foo.com'`。另一个左外连接将匹配与`User`相关的*所有*`Address`行，并且仅用于为返回的`User`对象填充`User.addresses`集合。

通过改变`joinedload()`的使用方式为另一种加载样式，我们可以完全独立于用于检索实际所需`User`行的 SQL，改变集合的加载方式。以下我们将`joinedload()`改为`selectinload()`：

```py
>>> stmt = (
...     select(User)
...     .join(User.addresses)
...     .options(selectinload(User.addresses))
...     .filter(User.name == "spongebob")
...     .filter(Address.email_address == "someaddress@foo.com")
... )
>>> result = session.scalars(stmt).all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
JOIN  addresses  ON  users.id  =  addresses.user_id
WHERE
  users.name  =  ?
  AND  addresses.email_address  =  ?
['spongebob',  'someaddress@foo.com']
#  ...  selectinload()  emits  a  SELECT  in  order
#  to  load  all  address  records  ... 
```

当使用连接式急切加载时，如果查询包含影响联接外部返回的行的修饰符，例如使用 DISTINCT、LIMIT、OFFSET 或等效的修饰符，完成的语句首先包装在一个子查询中，并且专门用于连接式急切加载的联接应用于子查询。SQLAlchemy 的连接式急切加载努力工作，然后再走十英里，绝对确保它不会影响查询的最终结果，只影响加载集合和相关对象的方式，无论查询的格式是什么。

另请参阅

将显式联接/语句路由到急切加载的集合 - 使用`contains_eager()`

## 选择性加载

在大多数情况下，选择性加载是急切加载对象集合的最简单和最有效的方法。唯一不可行的选择性急切加载的情况是当模型使用复合主键，并且后端数据库不支持具有 IN 的元组时，这种情况目前包括 SQL Server。

使用`"selectin"`参数或使用`selectinload()`加载器选项提供了“选择 IN”急切加载。这种加载样式发出一个 SELECT，该 SELECT 引用父对象的主键值，或者在一对多关系的情况下引用子对象的主键值，位于 IN 子句中，以加载相关联的关系：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import selectinload
>>> stmt = (
...     select(User)
...     .options(selectinload(User.addresses))
...     .filter(or_(User.name == "spongebob", User.name == "ed"))
... )
>>> result = session.scalars(stmt).all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
WHERE  users.name  =  ?  OR  users.name  =  ?
('spongebob',  'ed')
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id
FROM  addresses
WHERE  addresses.user_id  IN  (?,  ?)
(5,  7) 
```

在上面，第二个 SELECT 引用了`addresses.user_id IN (5, 7)`，其中的“5”和“7”是前两个加载的`User`对象的主键值；在一批对象完全加载后，它们的主键值被注入到第二个 SELECT 的`IN`子句中。因为`User`和`Address`之间的关系具有简单的主键连接条件，并且提供了`User`的主键值可以从`Address.user_id`派生，所以该语句根本没有联接或子查询。

对于简单的一对多加载，也不需要 JOIN，因为使用父对象的外键值即可：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import selectinload
>>> stmt = select(Address).options(selectinload(Address.user))
>>> result = session.scalars(stmt).all()
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id
  FROM  addresses
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
WHERE  users.id  IN  (?,  ?)
(1,  2) 
```

提示

“简单”是指 `relationship.primaryjoin` 条件表达了“一”侧的主键和“多”侧的直接外键之间的相等比较，没有任何其他条件。

选择 IN 加载还支持多对多的关系，在目前的情况下，它会跨越所有三个表进行 JOIN，以匹配一边到另一边的行。

关于这种加载方式需要知道的事情包括：

+   此策略每次会发出一个 SELECT，最多为 500 个父主键值，因为主键被渲染为 SQL 语句中的大型 IN 表达式。一些数据库，如 Oracle，对 IN 表达式的大小有硬限制，总体上，SQL 字符串的大小不应该是任意大的。

+   由于“选择加载”依赖于 IN，对于具有复合主键的映射，它必须使用 IN 的“元组”形式，看起来像 `WHERE (table.column_a, table.column_b) IN ((?, ?), (?, ?), (?, ?))`。这种语法目前不受 SQL Server 支持，对于 SQLite，需要至少 3.15 版本。SQLAlchemy 中没有特殊的逻辑来提前检查哪些平台支持此语法；如果运行在不支持的平台上，数据库将立即返回错误。SQLAlchemy 之所以仅运行 SQL 以使其失败的优点是，如果特定的数据库确实开始支持此语法，则无需对 SQLAlchemy 进行任何更改（就像 SQLite 的情况一样）。

## 子查询预加载

旧特性

`subqueryload()` 预加载器在大多数情况下已经过时，被设计更简单、更灵活，例如 Yield Per 等功能的 `selectinload()` 策略取代，并在大多数情况下发出更有效的 SQL 语句。由于 `subqueryload()` 依赖于重新解释原始的 SELECT 语句，当给出非常复杂的源查询时，它可能无法有效地工作。

对于具有复合主键的对象的预加载集合的特定情况，`subqueryload()` 在 Microsoft SQL Server 后端上继续没有支持“元组 IN”语法的情况下仍可能有用。

子查询加载在操作上类似于选择加载，但是发出的 SELECT 语句是从原始语句派生的，并且查询结构比选择加载更复杂。

使用`relationship.lazy`中的`"subquery"`参数提供子查询即时加载，或者使用`subqueryload()`加载器选项。

子查询即时加载的操作是为要加载的每个关系发出第二个 SELECT 语句，在所有结果对象中一次完成加载。该 SELECT 语句引用原始 SELECT 语句，包装在一个子查询中，以便我们检索返回的主对象的相同主键列表，然后将其链接到加载所有集合成员的总和：

```py
>>> from sqlalchemy import select
>>> from sqlalchemy.orm import subqueryload
>>> stmt = select(User).options(subqueryload(User.addresses)).filter_by(name="spongebob")
>>> results = session.scalars(stmt).all()
SELECT
  users.id  AS  users_id,
  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  users
WHERE  users.name  =  ?
('spongebob',)
SELECT
  addresses.id  AS  addresses_id,
  addresses.email_address  AS  addresses_email_address,
  addresses.user_id  AS  addresses_user_id,
  anon_1.users_id  AS  anon_1_users_id
FROM  (
  SELECT  users.id  AS  users_id
  FROM  users
  WHERE  users.name  =  ?)  AS  anon_1
JOIN  addresses  ON  anon_1.users_id  =  addresses.user_id
ORDER  BY  anon_1.users_id,  addresses.id
('spongebob',) 
```

关于这种加载方式需要了解的事项包括：

+   “子查询”加载策略发出的 SELECT 语句，与“selectin”不同，需要一个子查询，并将继承原始查询中存在的任何性能限制。子查询本身也可能因使用的数据库的具体情况而产生性能损失。

+   “子查询”加载会对正确工作施加一些特殊的排序要求。使用`subqueryload()`的查询，结合使用诸如`Select.limit()`或`Select.offset()`之类的限定修饰符，应**始终**包括针对唯一列（如主键）的`Select.order_by()`，以便由`subqueryload()`发出的附加查询包含与父查询使用的相同排序。否则，内部查询可能返回错误的行：

    ```py
    # incorrect, no ORDER BY
    stmt = select(User).options(subqueryload(User.addresses).limit(1))

    # incorrect if User.name is not unique
    stmt = select(User).options(subqueryload(User.addresses)).order_by(User.name).limit(1)

    # correct
    stmt = (
        select(User)
        .options(subqueryload(User.addresses))
        .order_by(User.name, User.id)
        .limit(1)
    )
    ```

    另请参见

    为什么推荐使用 ORDER BY 与 LIMIT（特别是与 subqueryload() 一起）？ - 详细示例

+   当在许多层次深的即时加载中使用“子查询”加载时，还会产生额外的性能/复杂性问题，因为子查询将被重复嵌套。

+   “子查询”加载与 Yield Per 提供的“批量”加载不兼容，无论是集合还是标量关系。

由于上述原因，“选择”策略应优先于“子查询”。

另请参见

选择 IN 加载

## 使用什么类型的加载？

使用哪种类型的加载通常归结为优化 SQL 执行次数、生成的 SQL 复杂度和获取的数据量之间的权衡。

**一对多/多对多集合** - 通常最好使用`selectinload()`加载策略。它发出一个额外的 SELECT，尽可能少地使用表，不影响原始语句，并且对于任何类型的起始查询都是最灵活的。它唯一的主要限制是在使用不支持“tuple IN”的后端上使用具有复合主键的表，目前包括 SQL Server 和非常旧的 SQLite 版本；所有其他包含的后端都支持它。

**多对一** - `joinedload()`策略是最通用的策略。在特殊情况下，如果存在非常少量的潜在相关值，则`immediateload()`策略也可能有用，因为如果相关对象已经存在，则此策略将从本地`Session`获取对象而不发出任何 SQL。

## 多态急加载

支持在每个急加载基础上指定多态选项。请参见 Eager Loading of Polymorphic Subtypes 部分，了解`PropComparator.of_type()`方法与`with_polymorphic()`函数的结合示例。

## 通配符加载策略

每个`joinedload()`、`subqueryload()`、`lazyload()`、`selectinload()`、`noload()`和`raiseload()`都可以用于设置特定查询的`relationship()`加载的默认样式，影响所有未在语句中另行指定的`relationship()` -映射属性。通过将字符串`'*'`作为这些选项中的任何一个的参数传递，可以使用此功能：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload

stmt = select(MyClass).options(lazyload("*"))
```

在上面，`lazyload('*')`选项将取代所有正在使用的`relationship()`构造的`lazy`设置，但不包括那些使用`lazy='write_only'`或`lazy='dynamic'`的情况。

如果某些关系指定了 `lazy='joined'` 或 `lazy='selectin'`，例如，使用 `lazyload('*')` 将单方面导致所有这些关系使用 `'select'` 加载，例如，当访问每个属性时发出一个 SELECT 语句。

该选项不会取代查询中指定的加载选项，例如`joinedload()`，`selectinload()`等。下面的查询仍将对`widget`关系使用连接加载：

```py
from sqlalchemy import select
from sqlalchemy.orm import lazyload
from sqlalchemy.orm import joinedload

stmt = select(MyClass).options(lazyload("*"), joinedload(MyClass.widget))
```

不管`joinedload()`指令出现在`lazyload()`选项之前还是之后，如果传递了包含 `"*"` 的多个选项，则最后一个选项将生效。

### 每个实体的通配符加载策略

通配符加载策略的变体是能够按实体基础设置策略的能力。例如，如果查询`User`和`Address`，我们可以指示`Address`上的所有关系使用延迟加载，同时通过首先应用`Load`对象，然后将 `*` 指定为链接选项：

```py
from sqlalchemy import select
from sqlalchemy.orm import Load

stmt = select(User, Address).options(Load(Address).lazyload("*"))
```

上述，`Address`上的所有关系都将设置为延迟加载。### 每个实体的通配符加载策略

通配符加载策略的变体是能够按实体基础设置策略的能力。例如，如果查询`User`和`Address`，我们可以指示`Address`上的所有关系使用延迟加载，同时通过首先应用`Load`对象，然后将 `*` 指定为链接选项：

```py
from sqlalchemy import select
from sqlalchemy.orm import Load

stmt = select(User, Address).options(Load(Address).lazyload("*"))
```

上述，`Address`上的所有关系都将设置为延迟加载。

## 将显式连接/语句路由到急加载集合

`joinedload()`的行为是自动创建连接，使用匿名别名作为目标，其结果路由到加载对象上的集合和标量引用。通常情况下，查询已经包含了表示特定集合或标量引用的必要连接，而由 joinedload 功能添加的连接是多余的 - 但您仍希望填充集合/引用。

对于此，SQLAlchemy 提供了 `contains_eager()` 选项。该选项的使用方式与 `joinedload()` 选项相同，只是假定 `Select` 对象将明确包括适当的连接，通常使用 `Select.join()` 等方法。下面，我们指定了 `User` 和 `Address` 之间的连接，并额外将其作为 `User.addresses` 的急切加载的基础：

```py
from sqlalchemy.orm import contains_eager

stmt = select(User).join(User.addresses).options(contains_eager(User.addresses))
```

如果语句的“急切”部分是“别名”，则应使用 `PropComparator.of_type()` 指定路径，这允许传递特定的 `aliased()` 构造：

```py
# use an alias of the Address entity
adalias = aliased(Address)

# construct a statement which expects the "addresses" results

stmt = (
    select(User)
    .outerjoin(User.addresses.of_type(adalias))
    .options(contains_eager(User.addresses.of_type(adalias)))
)

# get results normally
r = session.scalars(stmt).unique().all()
SELECT
  users.user_id  AS  users_user_id,
  users.user_name  AS  users_user_name,
  adalias.address_id  AS  adalias_address_id,
  adalias.user_id  AS  adalias_user_id,
  adalias.email_address  AS  adalias_email_address,
  (...other  columns...)
FROM  users
LEFT  OUTER  JOIN  email_addresses  AS  email_addresses_1
ON  users.user_id  =  email_addresses_1.user_id 
```

作为参数给出的路径必须是从起始实体开始的完整路径，例如，如果我们正在加载 `Users->orders->Order->items->Item`，则选项将如下使用：

```py
stmt = select(User).options(contains_eager(User.orders).contains_eager(Order.items))
```

### 使用 `contains_eager()` 来加载自定义过滤的集合结果。

当我们使用 `contains_eager()` 时，*我们*是自己构造将用于填充集合的 SQL。由此自然而然地，我们可以选择 **修改** 集合意图存储的值，通过编写我们的 SQL 来加载集合或标量属性的元素子集。

提示

SQLAlchemy 现在有了一个 **更简单的方法** 来实现这一点，即允许将 WHERE 条件直接添加到加载器选项，如 `joinedload()` 和 `selectinload()` 使用 `PropComparator.and_()`。参见 Adding Criteria to loader options 章节中的示例。

如果相关的集合要使用比简单的 WHERE 子句更复杂的 SQL 条件或修饰符进行查询，这里描述的技术仍然适用。

举例来说，我们可以加载一个`User`对象，并仅急切地加载其中特定的地址到其`.addresses`集合中，方法是通过过滤连接的数据，并使用 `contains_eager()` 路由它，同时还使用 Populate Existing 确保任何已加载的集合都被覆盖：

```py
stmt = (
    select(User)
    .join(User.addresses)
    .filter(Address.email_address.like("%@aol.com"))
    .options(contains_eager(User.addresses))
    .execution_options(populate_existing=True)
)
```

上面的查询将仅加载包含至少一个包含子字符串`'aol.com'`的`email`字段的`Address`对象的`User`对象；`User.addresses`集合将**仅**包含这些`Address`条目，*而不包括*实际与该集合关联的任何其他`Address`条目。

提示

在所有情况下，除非有明确的指示要这样做，否则 SQLAlchemy ORM **不会覆盖已加载的属性和集合**。由于正在使用标识映射，通常情况下，ORM 查询返回的对象实际上已经存在并加载到内存中。因此，当使用`contains_eager()`以另一种方式填充集合时，通常最好像上面所示那样使用填充现有，以便已加载的集合使用新数据进行刷新。`populate_existing` 选项将重置已存在的**所有**属性，包括挂起的更改，因此在使用之前请确保刷新所有数据。使用具有其默认行为的`Session`的 autoflush 足以。

注意

使用`contains_eager()`加载的定制集合不是“粘性”的；也就是说，下次加载该集合时，它将以其通常的默认内容加载。如果对象过期，则集合可能会重新加载，这在使用默认会话设置时每当使用 `Session.commit()`、`Session.rollback()` 方法时，或者使用 `Session.expire_all()` 或 `Session.expire()` 方法时会发生。

另请参阅

向加载器选项添加条件 - 现代 API 允许在任何关系加载器选项中直接添加 WHERE 条件

### 使用 contains_eager() 加载定制过滤的集合结果

当我们使用`contains_eager()`时，*我们*正在构建用于填充集合的 SQL。由此自然而然地，我们可以选择**修改**集合的预期存储值，通过编写我们的 SQL 以加载集合或标量属性的子集元素。

提示

SQLAlchemy 现在有一种**更简单的方法**来做到这一点，即允许将 WHERE 条件直接添加到加载器选项，例如`joinedload()`和`selectinload()`，使用`PropComparator.and_()`。有关示例，请参见向加载器选项添加条件部分。

如果要使用比简单的 WHERE 子句更复杂的 SQL 条件或修改器查询相关集合，则这里描述的技术仍然适用。

例如，我们可以加载一个`User`对象，并且仅通过过滤联接数据并使用`contains_eager()`将其路由到`.addresses`集合，从而急切地加载特定地址，还使用 Populate Existing 确保任何已加载的集合都被覆盖：

```py
stmt = (
    select(User)
    .join(User.addresses)
    .filter(Address.email_address.like("%@aol.com"))
    .options(contains_eager(User.addresses))
    .execution_options(populate_existing=True)
)
```

上面的查询将仅加载包含其`email`字段中包含子字符串`'aol.com'`的`Address`对象的`User`对象；`User.addresses`集合将**仅**包含这些`Address`条目，而*不是*与集合实际相关联的任何其他`Address`条目。

提示

在所有情况下，SQLAlchemy ORM **不会覆盖已加载的属性和集合**，除非被告知要这样做。由于正在使用身份映射，通常情况下，ORM 查询返回的对象实际上已经存在并且在内存中已加载。因此，当使用`contains_eager()`以替代方式填充集合时，通常最好使用上面示例中所示的 Populate Existing，以便已加载的集合可以用新数据刷新。`populate_existing`选项将重置已经存在的**所有**属性，包括待处理的更改，因此请确保在使用之前刷新所有数据。使用具有其默认行为的`Session`的 autoflush 足以。

注意

我们使用`contains_eager()`加载的定制集合不是“粘性”的；也就是说，下次加载此集合时，它将以其通常的默认内容加载。 如果对象过期，则可能重新加载集合，这会在使用默认会话设置时每当使用`Session.commit()`、`Session.rollback()` 方法，或者使用`Session.expire_all()` 或`Session.expire()` 方法时发生。

另请参阅

向加载器选项添加条件 - 现代 API 允许在任何关系加载器选项中直接添加 WHERE 条件

## 关系加载器 API

| 对象名称 | 描述 |
| --- | --- |
| contains_eager(*keys, **kw) | 表示应从查询中手动指定的列急切加载给定属性。 |
| defaultload(*keys) | 表示应使用预定义的加载器样式加载属性。 |
| immediateload(*keys, [recursion_depth]) | 表示应使用带有每个属性 SELECT 语句的立即加载来加载给定属性。 |
| joinedload(*keys, **kw) | 表示应使用连接的急切加载来加载给定属性。 |
| lazyload(*keys) | 表示应使用“懒惰”加载来加载给定属性。 |
| Load | 代表修改 ORM 启用的`Select`或传统`Query`状态以影响加载各种映射属性的加载器选项。 |
| noload(*keys) | 表示给定关系属性应保持未加载状态。 |
| raiseload(*keys, **kw) | 表示给定属性在访问时应引发错误。 |
| selectinload(*keys, [recursion_depth]) | 表示应使用 SELECT IN 急切加载来加载给定属性。 |
| subqueryload(*keys) | 表示应使用子查询急切加载来加载给定属性。 |

```py
function sqlalchemy.orm.contains_eager(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) → _AbstractLoad
```

表示应从查询中手动指定的列急���加载给定属性。

此函数是`Load`接口的一部分，支持方法链接和独立操作。

此选项与加载所需行的显式连接一起使用，即：

```py
sess.query(Order).join(Order.user).options(
    contains_eager(Order.user)
)
```

上述查询将从 `Order` 实体连接到其相关的 `User` 实体，并且返回的 `Order` 对象将预先填充 `Order.user` 属性。

还可以用于自定义急切加载集合中的条目；查询通常会使用 填充现有 执行选项，假设父对象的主要集合可能已经加载：

```py
sess.query(User).join(User.addresses).filter(
    Address.email_address.like("%@aol.com")
).options(contains_eager(User.addresses)).populate_existing()
```

有关完整的使用详细信息，请参阅 将显式连接/语句路由到急切加载的集合中 部分。

另请参见

关系加载技术

将显式连接/语句路由到急切加载的集合中

```py
function sqlalchemy.orm.defaultload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

指示属性应使用其预定义的加载器样式加载。

此加载选项的行为是不更改属性的当前加载样式，这意味着将使用先前配置的样式，或者如果没有选择先前的样式，则将使用默认加载。

此方法用于将其他加载器选项链接到属性链中的进一步位置，而不更改链中的链接的加载器样式。例如，要为元素的元素设置连接的急切加载：

```py
session.query(MyClass).options(
    defaultload(MyClass.someattribute).joinedload(
        MyOtherClass.someotherattribute
    )
)
```

`defaultload()` 也可用于在相关类上设置列级选项，即 `defer()` 和 `undefer()`：

```py
session.scalars(
    select(MyClass).options(
        defaultload(MyClass.someattribute)
        .defer("some_column")
        .undefer("some_other_column")
    )
)
```

另请参见

使用 Load.options() 指定子选项

`Load.options()`

```py
function sqlalchemy.orm.immediateload(*keys: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → _AbstractLoad
```

指示应使用立即加载和每个属性的 SELECT 语句加载给定属性。

加载是使用“懒加载器”策略实现的，不会触发任何额外的急切加载器。

`immediateload()` 选项通常被 `selectinload()` 选项替代，后者通过为所有加载的对象发出 SELECT 语句来更有效地执行相同的任务。

此函数是 `Load` 接口的一部分，并支持方法链接和独立操作。

参数：

**recursion_depth** – 递归深度

可选整数；当与自引用关系一起设置为正整数时，表示“selectin”加载将自动继续这么多级别，直到找不到任何项目为止。

注意

`immediateload.recursion_depth`选项目前仅支持自引用关系。目前还没有选项可以自动遍历涉及多个关系的递归结构。

警告

此参数是新的实验性参数，应该视为“alpha”状态

2.0 版本中新增：添加`immediateload.recursion_depth`

另请参见

关系加载技术

Select IN 加载

```py
function sqlalchemy.orm.joinedload(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) → _AbstractLoad
```

表示应该使用连接的快速加载来加载给定的属性。

此函数是`Load`接口的一部分，支持方法链和独立操作。

示例：

```py
# joined-load the "orders" collection on "User"
select(User).options(joinedload(User.orders))

# joined-load Order.items and then Item.keywords
select(Order).options(
    joinedload(Order.items).joinedload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# joined-load the keywords collection
select(Order).options(
    lazyload(Order.items).joinedload(Item.keywords)
)
```

参数：

**innerjoin** –

如果`True`，表示连接的急切加载应该使用内部连接而不是左外连接的默认值：

```py
select(Order).options(joinedload(Order.user, innerjoin=True))
```

为了链接多个急切加载在一起，其中一些可能是 OUTER，另一些是 INNER，右嵌套连接用于链接它们：

```py
select(A).options(
    joinedload(A.bs, innerjoin=False).joinedload(
        B.cs, innerjoin=True
    )
)
```

上述查询通过“outer” join 连接 A.bs 和通过“inner” join 连接 B.cs，将连接呈现为“a LEFT OUTER JOIN (b JOIN c)”。

`innerjoin`标志也可以用术语`"unnested"`来表示。这表示应该使用 INNER JOIN，*除非*连接到左侧的 LEFT OUTER JOIN，这种情况下它将呈现为 LEFT OUTER JOIN。例如，假设`A.bs`是一个 outerjoin：

```py
select(A).options(
    joinedload(A.bs).joinedload(B.cs, innerjoin="unnested")
)
```

上述连接将呈现为“a LEFT OUTER JOIN b LEFT OUTER JOIN c”，而不是“a LEFT OUTER JOIN (b JOIN c)”。

注意

“unnested”标志**不会**影响从多对多关联表（例如配置为`relationship.secondary`的表）到目标表的 JOIN 渲染；为了结果的正确性，这些连接始终是 INNER 的，因此如果连接到 OUTER join，它们将是右嵌套的。

注意

`joinedload()`生成的连接是**匿名别名**。连接进行的条件无法修改，ORM 启用的`Select`或传统的`Query`也不能以任何方式引用这些连接，包括排序。有关详细信息，请参见急切加载之道。

要生成一个明确可用的特定 SQL JOIN，请使用`Select.join()`和`Query.join()`。要将显式的 JOIN 与集合的急加载结合起来，请使用`contains_eager()`；参见将显式 JOIN/语句路由到急加载的集合。

另请参阅

关系加载技术

连接急加载

```py
function sqlalchemy.orm.lazyload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

表示应该使用“懒加载”来加载给定的属性。

此函数是`Load`接口的一部分，支持方法链和独立操作。

另请参阅

关系加载技术

懒加载

```py
class sqlalchemy.orm.Load
```

表示加载器选项，用于修改 ORM 启用的`Select`或传统`Query`的状态，以影响加载各种映射属性的方式。

在大多数情况下，当使用查询选项像`joinedload()`、`defer()`或类似的选项时，`Load`对象通常会在幕后隐式使用。除了一些非常特殊的情况外，通常不会直接实例化它。

另请参阅

每个实体的通配符加载策略 - 演示了直接使用`Load`可能有用的示例

**成员**

contains_eager(), defaultload(), defer(), get_children(), immediateload(), inherit_cache, joinedload(), lazyload(), load_only(), noload(), options(), process_compile_state(), process_compile_state_replaced_entities(), propagate_to_loaders, raiseload(), selectin_polymorphic(), selectinload(), subqueryload(), undefer(), undefer_group(), with_expression()

**类签名**

类`sqlalchemy.orm.Load`（`sqlalchemy.orm.strategy_options._AbstractLoad`）

```py
method contains_eager(attr: _AttrType, alias: _FromClauseArgument | None = None, _is_chain: bool = False) → Self
```

*从* `sqlalchemy.orm.strategy_options._AbstractLoad.contains_eager` *方法继承* `sqlalchemy.orm.strategy_options._AbstractLoad`

生成一个新的带有`contains_eager()`选项的`Load`对象。

查看`contains_eager()`的使用示例。

```py
method defaultload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*从* `sqlalchemy.orm.strategy_options._AbstractLoad.defaultload` *方法继承* `sqlalchemy.orm.strategy_options._AbstractLoad`

生成一个新的带有`defaultload()`选项的`Load`对象。

查看`defaultload()`的使用示例。

```py
method defer(key: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → Self
```

*从* `sqlalchemy.orm.strategy_options._AbstractLoad.defer` *方法继承* `sqlalchemy.orm.strategy_options._AbstractLoad`

生成一个新的带有`defer()`选项的`Load`对象。

查看`defer()`的使用示例。

```py
method get_children(*, omit_attrs: Tuple[str, ...] = (), **kw: Any) → Iterable[HasTraverseInternals]
```

*从* `HasTraverseInternals.get_children()` *方法继承* `HasTraverseInternals`

返回此`HasTraverseInternals`的即时子项`HasTraverseInternals`。

用于访问遍历。

**kw 可能包含更改返回集合的标志，例如返回子项的子集以减少较大的遍历，或者从不同上下文返回子项（例如模式级集合而不是从子句级返回的）。

```py
method immediateload(attr: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → Self
```

*从* `sqlalchemy.orm.strategy_options._AbstractLoad.immediateload` *方法继承* `sqlalchemy.orm.strategy_options._AbstractLoad`

生成一个新的带有`immediateload()`选项的`Load`对象。

查看`immediateload()`的使用示例。

```py
attribute inherit_cache: bool | None = None
```

*从* `HasCacheKey` *的* `HasCacheKey.inherit_cache` *属性继承*

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑是否适合参与缓存；这在功能上等效于将值设置为`False`，但还会发出警告。

如果对象对应的 SQL 不基于本类的属性而是本类的父类属性，则可以将此标志设置为`True`。

另请参阅

为自定义构造启用缓存支持 - 有关为第三方或用户定义的 SQL 构造设置`HasCacheKey.inherit_cache`属性的一般指南。

```py
method joinedload(attr: Literal['*'] | QueryableAttribute[Any], innerjoin: bool | None = None) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.joinedload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`joinedload()`选项生成一个新的`Load`对象。

请参阅`joinedload()`以查看用法示例。

```py
method lazyload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.lazyload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`lazyload()`选项生成一个新的`Load`对象。

请参阅`lazyload()`以查看用法示例。

```py
method load_only(*attrs: Literal['*'] | QueryableAttribute[Any], raiseload: bool = False) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.load_only` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`load_only()`选项生成一个新的`Load`对象。

请参阅`load_only()`以查看用法示例。

```py
method noload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.noload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`noload()`选项生成一个新的`Load`对象。

请参阅`noload()`以查看用法示例。

```py
method options(*opts: _AbstractLoad) → Self
```

将一系列选项作为此`Load`对象的子选项应用。

例如：

```py
query = session.query(Author)
query = query.options(
            joinedload(Author.book).options(
                load_only(Book.summary, Book.excerpt),
                joinedload(Book.citations).options(
                    joinedload(Citation.author)
                )
            )
        )
```

参数：

***opts** – 应应用于此`Load`对象指定路径的一系列加载器选项对象（最终为`Load`对象）。

新版本 1.3.6 中新增。

另请参阅

`defaultload()`

使用 Load.options() 指定子选项

```py
method process_compile_state(compile_state: ORMCompileState) → None
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.process_compile_state` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

对给定的`ORMCompileState`应用修改。

此方法是特定`CompileStateOption`的实现的一部分，仅在编译 ORM 查询时内部调用。

```py
method process_compile_state_replaced_entities(compile_state: ORMCompileState, mapper_entities: Sequence[_MapperEntity]) → None
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.process_compile_state_replaced_entities` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

对给定的`ORMCompileState`应用修改，给出仅由`with_only_columns()`或`with_entities()`替换的实体。

此方法是特定`CompileStateOption`的实现的一部分，仅在编译 ORM 查询时内部调用。

1.4.19 版本中的新功能。

```py
attribute propagate_to_loaders: bool
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.propagate_to_loaders` *属性的* `sqlalchemy.orm.strategy_options._AbstractLoad`

如果为 True，则表示此选项应该传递到关系懒加载器的“次要”SELECT 语句，以及属性加载/刷新操作。

```py
method raiseload(attr: Literal['*'] | QueryableAttribute[Any], sql_only: bool = False) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.raiseload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`raiseload()`选项生成一个新的`Load`对象。

查看`raiseload()`以查看用法示例。

```py
method selectin_polymorphic(classes: Iterable[Type[Any]]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.selectin_polymorphic` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`selectin_polymorphic()`选项生成一个新的`Load`对象。

查看`selectin_polymorphic()`以查看用法示例。

```py
method selectinload(attr: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.selectinload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`selectinload()`选项生成一个新的`Load`对象。

查看`selectinload()`以查看用法示例。

```py
method subqueryload(attr: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.subqueryload` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`subqueryload()`选项生成一个新的`Load`对象。

查看`subqueryload()`以查看用法示例。

```py
method undefer(key: Literal['*'] | QueryableAttribute[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.undefer` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`undefer()`选项生成一个新的`Load`对象。

请参见`undefer()`获取用法示例。

```py
method undefer_group(name: str) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.undefer_group` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`undefer_group()`选项生成新的`Load`对象。

请参见`undefer_group()`获取用法示例。

```py
method with_expression(key: _AttrType, expression: _ColumnExpressionArgument[Any]) → Self
```

*继承自* `sqlalchemy.orm.strategy_options._AbstractLoad.with_expression` *方法的* `sqlalchemy.orm.strategy_options._AbstractLoad`

使用`with_expression()`选项生成新的`Load`对象。

请参见`with_expression()`获取用法示例。

```py
function sqlalchemy.orm.noload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

表示给定的关系属性应保持未加载状态。

当访问关系属性时，关系属性将返回`None`，而不产生任何加载效果。

此功能是`Load`接口的一部分，支持方法链和独立操作。

`noload()`仅适用于`relationship()`属性。

注意

使用`relationship.lazy`参数将此加载策略设置为关系的默认策略可能会导致刷新时出现问题，例如，如果删除操作需要加载相关对象，而返回的是`None`。

参见

关系加载技术

```py
function sqlalchemy.orm.raiseload(*keys: Literal['*'] | QueryableAttribute[Any], **kw: Any) → _AbstractLoad
```

表示如果访问给定的属性，应该引发错误。

使用`raiseload()`配置的关系属性在访问时将引发`InvalidRequestError`。这种方式通常很有用，当应用程序试图确保在特定上下文中访问的所有关系属性都已通过急加载加载时。这种策略将导致立即引发异常，而不必查看 SQL 日志以确保不会发生延迟加载。

`raiseload()` 仅适用于 `relationship()` 属性。为了将在基于列的属性上应用 SQL 异常处理行为，应在 `defer()` 加载器选项的 `defer.raiseload` 参数上使用。

参数：

**sql_only** – 如果为 True，则仅在延迟加载会发出 SQL 时引发异常，但如果仅检查标识映射或确定相关值由于缺少键应为 None，则不会引发异常。当为 False 时，该策略将引发所有类型的关系加载异常。

此函数是 `Load` 接口的一部分，支持方法链式和独立操作。

另请参阅

关系加载技术

使用 raiseload 防止不需要的延迟加载

使用 raiseload 防止延迟加载列

```py
function sqlalchemy.orm.selectinload(*keys: Literal['*'] | QueryableAttribute[Any], recursion_depth: int | None = None) → _AbstractLoad
```

指示应使用 SELECT IN 即时加载来加载给定的属性。

此函数是 `Load` 接口的一部分，支持方法链式和独立操作。

示例：

```py
# selectin-load the "orders" collection on "User"
select(User).options(selectinload(User.orders))

# selectin-load Order.items and then Item.keywords
select(Order).options(
    selectinload(Order.items).selectinload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# selectin-load the keywords collection
select(Order).options(
    lazyload(Order.items).selectinload(Item.keywords)
)
```

参数：

**递归深度** –

可选整数；当与自引用关系一起设置为正整数时，表示“选择加载”将自动深入到指定的层级直到找不到任何项目。

注意

`selectinload.recursion_depth` 选项目前仅支持自引用关系。目前还没有自动遍历多个关系的递归结构的选项。

此外，`selectinload.recursion_depth` 参数是新的实验性参数，并且应被视为 2.0 系列的“alpha”状态。

2.0 版本中新增：添加 `selectinload.recursion_depth`

另请参阅

关系加载技术

选择 IN 加载

```py
function sqlalchemy.orm.subqueryload(*keys: Literal['*'] | QueryableAttribute[Any]) → _AbstractLoad
```

指示应使用子查询即时加载来加载给定的属性。

此函数是 `Load` 接口的一部分，支持方法链式和独立操作。

示例：

```py
# subquery-load the "orders" collection on "User"
select(User).options(subqueryload(User.orders))

# subquery-load Order.items and then Item.keywords
select(Order).options(
    subqueryload(Order.items).subqueryload(Item.keywords)
)

# lazily load Order.items, but when Items are loaded,
# subquery-load the keywords collection
select(Order).options(
    lazyload(Order.items).subqueryload(Item.keywords)
)
```

另请参阅

关系加载技术

子查询即时加载
