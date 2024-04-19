# 配置关系连接

> 原文：[`docs.sqlalchemy.org/en/20/orm/join_conditions.html`](https://docs.sqlalchemy.org/en/20/orm/join_conditions.html)

`relationship()`通常会通过检查两个表之间的外键关系来创建两个表之间的连接，以确定应该比较哪些列。有各种情况需要定制此行为。

## 处理多个连接路径

处理的最常见情况之一是两个表之间存在多个外键路径时。

考虑一个包含两个外键到`Address`类的`Customer`类：

```py
from sqlalchemy import Integer, ForeignKey, String, Column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Customer(Base):
    __tablename__ = "customer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
    shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

    billing_address = relationship("Address")
    shipping_address = relationship("Address")

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    street = mapped_column(String)
    city = mapped_column(String)
    state = mapped_column(String)
    zip = mapped_column(String)
```

当我们尝试使用上述映射时，将产生错误：

```py
sqlalchemy.exc.AmbiguousForeignKeysError: Could not determine join
condition between parent/child tables on relationship
Customer.billing_address - there are multiple foreign key
paths linking the tables.  Specify the 'foreign_keys' argument,
providing a list of those columns which should be
counted as containing a foreign key reference to the parent table.
```

上述消息相当长。`relationship()`可以返回许多潜在消息，这些消息已经被精心设计用于检测各种常见的配置问题；大多数都会建议需要哪些额外配置来解决模糊或其他缺失的信息。

在这种情况下，消息希望我们通过为每个指定的`relationship()`指定应该考虑哪个外键列来修饰每一个，并且适当的形式如下：

```py
class Customer(Base):
    __tablename__ = "customer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
    shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

    billing_address = relationship("Address", foreign_keys=[billing_address_id])
    shipping_address = relationship("Address", foreign_keys=[shipping_address_id])
```

在上面，我们指定了`foreign_keys`参数，它是一个`Column`或`Column`对象的列表，指示要考虑的“外键”列，或者换句话说，包含引用父表的值的列。从`Customer`对象加载`Customer.billing_address`关系将使用`billing_address_id`中的值来标识要加载的`Address`行；类似地，`shipping_address_id`用于`shipping_address`关系。这两列的关联在持久性期间也起到了作用；刚刚插入的`Address`对象的新生成的主键将在刷新期间复制到关联的`Customer`对象的适当外键列中。

在使用 Declarative 指定`foreign_keys`时，我们还可以使用字符串名称进行指定，但是如果使用列表，**列表应该是字符串的一部分**是很重要的：

```py
billing_address = relationship("Address", foreign_keys="[Customer.billing_address_id]")
```

在这个具体的例子中，在任何情况下列表都是不必要的，因为我们只需要一个`Column`：

```py
billing_address = relationship("Address", foreign_keys="Customer.billing_address_id")
```

警告

当作为 Python 可执行字符串传递时，`relationship.foreign_keys` 参数将使用 Python 的 `eval()` 函数进行解释。**请勿将不受信任的输入传递给此字符串**。详细信息请参见关系参数的评估。

`relationship()` 在构建连接时的默认行为是将一侧的主键列的值等同于另一侧的外键引用列的值。我们可以使用 `relationship.primaryjoin` 参数来更改此条件，以及在使用“次要”表时，还可以使用 `relationship.secondaryjoin` 参数。

在下面的示例中，我们使用 `User` 类以及存储街道地址的 `Address` 类来创建一个关系 `boston_addresses`，它将仅加载指定城市为“波士顿”的 `Address` 对象：

```py
from sqlalchemy import Integer, ForeignKey, String, Column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)
    boston_addresses = relationship(
        "Address",
        primaryjoin="and_(User.id==Address.user_id, Address.city=='Boston')",
    )

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(Integer, ForeignKey("user.id"))

    street = mapped_column(String)
    city = mapped_column(String)
    state = mapped_column(String)
    zip = mapped_column(String)
```

在此字符串 SQL 表达式中，我们使用了 `and_()` 连接构造来为连接条件建立两个不同的谓词 - 将 `User.id` 和 `Address.user_id` 列相互连接，同时将 `Address` 中的行限制为只有 `city='Boston'`。在使用声明式时，诸如 `and_()` 这样的基本 SQL 函数会自动在字符串 `relationship()` 参数的评估命名空间中可用。

警告

当作为 Python 可执行字符串传递时，`relationship.primaryjoin` 参数将使用 Python 的 `eval()` 函数进行解释。**请勿将不受信任的输入传递给此字符串**。详细信息请参见关系参数的评估。

我们在`relationship.primaryjoin`中使用的自定义标准通常只在 SQLAlchemy 渲染 SQL 以加载或表示此关系时才重要。也就是说，它用于在执行每个属性的延迟加载时发出的 SQL 语句中，或者在查询时构造联接时，例如通过`Select.join()`或通过渴望的“joined”或“subquery”加载样式。当操作内存中的对象时，我们可以将任何我们想要的`Address`对象放入`boston_addresses`集合中，而不管`.city`属性的值是什么。这些对象将一直保留在集合中，直到属性过期并重新从应用准则的数据库中加载为止。当发生刷新时，`boston_addresses`内的对象将被无条件地刷新，将主键`user.id`列的值分配给每一行的持有外键的`address.user_id`列。在这里，`city`标准没有影响，因为刷新过程只关心将主键值同步到引用外键值中。## 创建自定义外键条件

主要连接条件的另一个元素是如何确定那些被认为是“外部”的列的。通常，一些`Column`对象的子集将指定`ForeignKey`，或者否则将是与连接条件相关的`ForeignKeyConstraint`的一部分。`relationship()`查找此外键状态，因为它决定了它应该如何加载和持久化此关系的数据。然而，`relationship.primaryjoin`参数可以用来创建不涉及任何“架构”级外键的连接条件。我们可以结合`relationship.primaryjoin`以及`relationship.foreign_keys`和`relationship.remote_side`显式地建立这样一个连接。

下面，一个名为`HostEntry`的类与自身连接，将字符串`content`列与`ip_address`列相等，后者是一种名为`INET`的 PostgreSQL 类型。我们需要使用`cast()`来将连接的一侧转换为另一侧的类型：

```py
from sqlalchemy import cast, String, Column, Integer
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import INET

from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class HostEntry(Base):
    __tablename__ = "host_entry"

    id = mapped_column(Integer, primary_key=True)
    ip_address = mapped_column(INET)
    content = mapped_column(String(50))

    # relationship() using explicit foreign_keys, remote_side
    parent_host = relationship(
        "HostEntry",
        primaryjoin=ip_address == cast(content, INET),
        foreign_keys=content,
        remote_side=ip_address,
    )
```

上述关系将产生如下连接：

```py
SELECT  host_entry.id,  host_entry.ip_address,  host_entry.content
FROM  host_entry  JOIN  host_entry  AS  host_entry_1
ON  host_entry_1.ip_address  =  CAST(host_entry.content  AS  INET)
```

上述的另一种替代语法是在`relationship.primaryjoin`表达式内联使用`foreign()`和`remote()`注释。此语法表示了`relationship()`通常自动应用于连接条件的注释，给定了`relationship.foreign_keys`和`relationship.remote_side`参数。当存在显式连接条件时，这些函数可能更加简洁，并且还标记了“外键”或“远程”列的确切位置，无论该列是否多次声明或在复杂的 SQL 表达式中声明：

```py
from sqlalchemy.orm import foreign, remote

class HostEntry(Base):
    __tablename__ = "host_entry"

    id = mapped_column(Integer, primary_key=True)
    ip_address = mapped_column(INET)
    content = mapped_column(String(50))

    # relationship() using explicit foreign() and remote() annotations
    # in lieu of separate arguments
    parent_host = relationship(
        "HostEntry",
        primaryjoin=remote(ip_address) == cast(foreign(content), INET),
    )
```  ## 在连接条件中使用自定义运算符

关于关系的另一个用例是使用自定义运算符，例如在与`INET`和`CIDR`等类型结合时，使用 PostgreSQL 的“包含于”`<<`运算符。对于自定义布尔运算符，我们使用`Operators.bool_op()`函数：

```py
inet_column.bool_op("<<")(cidr_column)
```

类似上述的比较可以直接在构建`relationship()`时，与`relationship.primaryjoin`一起使用：

```py
class IPA(Base):
    __tablename__ = "ip_address"

    id = mapped_column(Integer, primary_key=True)
    v4address = mapped_column(INET)

    network = relationship(
        "Network",
        primaryjoin="IPA.v4address.bool_op('<<')(foreign(Network.v4representation))",
        viewonly=True,
    )

class Network(Base):
    __tablename__ = "network"

    id = mapped_column(Integer, primary_key=True)
    v4representation = mapped_column(CIDR)
```

上面的查询如下：

```py
select(IPA).join(IPA.network)
```

将呈现为：

```py
SELECT  ip_address.id  AS  ip_address_id,  ip_address.v4address  AS  ip_address_v4address
FROM  ip_address  JOIN  network  ON  ip_address.v4address  <<  network.v4representation
```  ## 基于 SQL 函数的自定义运算符

`Operators.op.is_comparison`的用例的一种变体是当我们不使用运算符，而是使用 SQL 函数时。这种用例的典型示例是 PostgreSQL PostGIS 函数，但任何数据库中解析为二进制条件的 SQL 函数都可以应用。为了适应这种用例，`FunctionElement.as_comparison()`方法可以修改任何 SQL 函数，例如从`func`命名空间调用的函数，以指示 ORM 函数生成两个表达式的比较。下面的示例用[Geoalchemy2](https://geoalchemy-2.readthedocs.io/)库说明了这一点：

```py
from geoalchemy2 import Geometry
from sqlalchemy import Column, Integer, func
from sqlalchemy.orm import relationship, foreign

class Polygon(Base):
    __tablename__ = "polygon"
    id = mapped_column(Integer, primary_key=True)
    geom = mapped_column(Geometry("POLYGON", srid=4326))
    points = relationship(
        "Point",
        primaryjoin="func.ST_Contains(foreign(Polygon.geom), Point.geom).as_comparison(1, 2)",
        viewonly=True,
    )

class Point(Base):
    __tablename__ = "point"
    id = mapped_column(Integer, primary_key=True)
    geom = mapped_column(Geometry("POINT", srid=4326))
```

在上面，`FunctionElement.as_comparison()`表明`func.ST_Contains()` SQL 函数正在比较`Polygon.geom`和`Point.geom`表达式。`foreign()`注释另外指出了在此特定关系中哪个列承担“外键”角色。

1.3 版本中的新增功能：添加了`FunctionElement.as_comparison()`。## 重叠的外键

当使用复合外键时，可能会出现罕见的情况，使得单个列可能是通过外键约束引用的多个列的主题。

考虑一个（诚然复杂的）映射，如`Magazine`对象，由`Writer`对象和`Article`对象使用包含`magazine_id`的复合主键方案引用；然后为了使`Article`也引用`Writer`，`Article.magazine_id`涉及到两个单独的关系；`Article.magazine`和`Article.writer`：

```py
class Magazine(Base):
    __tablename__ = "magazine"

    id = mapped_column(Integer, primary_key=True)

class Article(Base):
    __tablename__ = "article"

    article_id = mapped_column(Integer)
    magazine_id = mapped_column(ForeignKey("magazine.id"))
    writer_id = mapped_column()

    magazine = relationship("Magazine")
    writer = relationship("Writer")

    __table_args__ = (
        PrimaryKeyConstraint("article_id", "magazine_id"),
        ForeignKeyConstraint(
            ["writer_id", "magazine_id"], ["writer.id", "writer.magazine_id"]
        ),
    )

class Writer(Base):
    __tablename__ = "writer"

    id = mapped_column(Integer, primary_key=True)
    magazine_id = mapped_column(ForeignKey("magazine.id"), primary_key=True)
    magazine = relationship("Magazine")
```

配置上述映射后，我们将看到发出此警告：

```py
SAWarning: relationship 'Article.writer' will copy column
writer.magazine_id to column article.magazine_id,
which conflicts with relationship(s): 'Article.magazine'
(copies magazine.id to article.magazine_id). Consider applying
viewonly=True to read-only relationships, or provide a primaryjoin
condition marking writable columns with the foreign() annotation.
```

这指的是`Article.magazine_id`是两个不同外键约束的主题的事实；它直接引用`Magazine.id`作为源列，但也在`Writer`的复合键的上下文中引用`Writer.magazine_id`作为源列。如果我们将`Article`与特定的`Magazine`关联起来，然后将`Article`与与*不同*`Magazine`关联的`Writer`关联起来，ORM 将非确定性地覆盖`Article.magazine_id`，在不通知的情况下更改我们引用的杂志；如果我们将`Writer`从`Article`中解除关联，它还可能尝试将 NULL 放入此列。警告让我们知道这是这种情况。

要解决这个问题，我们需要将`Article`的行为分解，包括以下三个特性：

1.  首先，`Article`根据仅在`Article.magazine`关系中持久化的数据写入`Article.magazine_id`，即从`Magazine.id`复制的值。

1.  `Article`可以代表在`Article.writer`关系中持久化的数据写入`Article.writer_id`，但仅限于`Writer.id`列；`Writer.magazine_id`列不应写入`Article.magazine_id`，因为它最终源自`Magazine.id`。

1.  当加载`Article.writer`时，`Article`会考虑`Article.magazine_id`，尽管它在此关系中*不会*向其写入。

要获得仅#1 和#2，我们可以仅指定`Article.writer_id`作为`Article.writer`的“外键”：

```py
class Article(Base):
    # ...

    writer = relationship("Writer", foreign_keys="Article.writer_id")
```

然而，这会导致`Article.writer`在与`Writer`查询时不考虑`Article.magazine_id`：

```py
SELECT  article.article_id  AS  article_article_id,
  article.magazine_id  AS  article_magazine_id,
  article.writer_id  AS  article_writer_id
FROM  article
JOIN  writer  ON  writer.id  =  article.writer_id
```

因此，为了获得所有#1、#2 和#3，我们通过完全组合`relationship.primaryjoin`和`relationship.foreign_keys`参数，或者更简洁地通过用`foreign()`注释来表达连接条件以及要写入的列：

```py
class Article(Base):
    # ...

    writer = relationship(
        "Writer",
        primaryjoin="and_(Writer.id == foreign(Article.writer_id), "
        "Writer.magazine_id == Article.magazine_id)",
    )
```

## 非关系对比/实现路径

警告

此部分详细介绍了一个实验性功能。

使用自定义表达式意味着我们可以生成不遵循通常的主键/外键模型的非正统连接条件。其中一个例子是实现路径模式，其中我们比较字符串以产生重叠路径标记，以便生成树结构。

通过精心使用`foreign()`和`remote()`，我们可以构建一个有效地产生基本的物化路径系统的关系。基本上，当`foreign()`和`remote()`在*相同*的比较表达式的一侧时，关系被视为“一对多”；当它们在*不同*的一侧时，关系被视为“多对一”。对于我们将在此处使用的比较，我们将处理集合，因此保持配置为“一对多”：

```py
class Element(Base):
    __tablename__ = "element"

    path = mapped_column(String, primary_key=True)

    descendants = relationship(
        "Element",
        primaryjoin=remote(foreign(path)).like(path.concat("/%")),
        viewonly=True,
        order_by=path,
    )
```

上述情况下，如果给定一个具有路径属性为`"/foo/bar2"`的`Element`对象，我们寻求加载`Element.descendants`以如下形式：

```py
SELECT  element.path  AS  element_path
FROM  element
WHERE  element.path  LIKE  ('/foo/bar2'  ||  '/%')  ORDER  BY  element.path
```

## 自引用多对多关系

另见

本节记录了“邻接列表”模式的两表变体，该模式在邻接列表关系有所描述。务必查看子节自引用查询策略和配置自引用急切加载，这两者同样适用于此处讨论的映射模式。

多对多关系可以通过`relationship.primaryjoin`和`relationship.secondaryjoin`中的一个或两个进行自定义 - 后者对于使用`relationship.secondary`参数指定多对多引用的关系非常重要。一个常见的情况涉及使用`relationship.primaryjoin`和`relationship.secondaryjoin`来建立从一个类到自身的多对多关系，如下所示：

```py
from typing import List

from sqlalchemy import Integer, ForeignKey, Column, Table
from sqlalchemy.orm import DeclarativeBase, Mapped
from sqlalchemy.orm import mapped_column, relationship

class Base(DeclarativeBase):
    pass

node_to_node = Table(
    "node_to_node",
    Base.metadata,
    Column("left_node_id", Integer, ForeignKey("node.id"), primary_key=True),
    Column("right_node_id", Integer, ForeignKey("node.id"), primary_key=True),
)

class Node(Base):
    __tablename__ = "node"
    id: Mapped[int] = mapped_column(primary_key=True)
    label: Mapped[str]
    right_nodes: Mapped[List["Node"]] = relationship(
        "Node",
        secondary=node_to_node,
        primaryjoin=id == node_to_node.c.left_node_id,
        secondaryjoin=id == node_to_node.c.right_node_id,
        back_populates="left_nodes",
    )
    left_nodes: Mapped[List["Node"]] = relationship(
        "Node",
        secondary=node_to_node,
        primaryjoin=id == node_to_node.c.right_node_id,
        secondaryjoin=id == node_to_node.c.left_node_id,
        back_populates="right_nodes",
    )
```

在上述情况下，SQLAlchemy 无法自动知道哪些列应该连接到`right_nodes`和`left_nodes`关系。`relationship.primaryjoin`和`relationship.secondaryjoin`参数确定我们希望如何连接到关联表。在上面的声明形式中，由于我们正在声明这些条件，因此`id`变量直接可用作我们希望与之连接的`Column`对象。

或者，我们可以使用字符串来定义`relationship.primaryjoin`和`relationship.secondaryjoin`参数，这在我们的配置中可能没有`Node.id`列对象可用，或者`node_to_node`表可能还没有可用时很合适。当在声明字符串中引用普通的`Table`对象时，我们使用表的字符串名称，就像它在`MetaData`中一样：

```py
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    label = mapped_column(String)
    right_nodes = relationship(
        "Node",
        secondary="node_to_node",
        primaryjoin="Node.id==node_to_node.c.left_node_id",
        secondaryjoin="Node.id==node_to_node.c.right_node_id",
        backref="left_nodes",
    )
```

警告

当作为 Python 可评估字符串传递时，`relationship.primaryjoin`和`relationship.secondaryjoin`参数使用 Python 的`eval()`函数进行解释。**不要将不受信任的输入传递给这些字符串**。有关声明式评估`relationship()`参数的详细信息，请参阅关系参数的评估。

在此处的经典映射情况类似，其中`node_to_node`可以连接到`node.c.id`：

```py
from sqlalchemy import Integer, ForeignKey, String, Column, Table, MetaData
from sqlalchemy.orm import relationship, registry

metadata_obj = MetaData()
mapper_registry = registry()

node_to_node = Table(
    "node_to_node",
    metadata_obj,
    Column("left_node_id", Integer, ForeignKey("node.id"), primary_key=True),
    Column("right_node_id", Integer, ForeignKey("node.id"), primary_key=True),
)

node = Table(
    "node",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("label", String),
)

class Node:
    pass

mapper_registry.map_imperatively(
    Node,
    node,
    properties={
        "right_nodes": relationship(
            Node,
            secondary=node_to_node,
            primaryjoin=node.c.id == node_to_node.c.left_node_id,
            secondaryjoin=node.c.id == node_to_node.c.right_node_id,
            backref="left_nodes",
        )
    },
)
```

请注意，在两个示例中，`relationship.backref`关键字指定了一个`left_nodes`的 backref - 当`relationship()`在相反方向创建第二个关系时，它足够智能以反转`relationship.primaryjoin`和`relationship.secondaryjoin`参数。

另请参阅

+   邻接列表关系 - 单表版本

+   自引用查询策略 - 关于使用自引用映射进行查询的提示

+   配置自引用急切加载 - 使用自引用映射进行急切加载的提示  ## 复合“次要”连接

注意

本节介绍了 SQLAlchemy 支持的一些边缘案例，但建议尽可能以更简单的方式解决这类问题，例如使用合理的关系布局和/或 Python 属性内部。

有时，当需要在两个表之间建立`relationship()`时，需要涉及更多的表才能将它们连接起来。这是一种`relationship()`的领域，人们试图推动可能性的边界，而这类奇特用例的最终解决方案通常需要在 SQLAlchemy 邮件列表上讨论出来。

在较新的 SQLAlchemy 版本中，`relationship.secondary`参数可以在某些情况下使用，以提供由多个表组成的复合目标。下面是这种连接条件的示例（至少需要版本 0.9.2 才能正常运行）：

```py
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

    d = relationship(
        "D",
        secondary="join(B, D, B.d_id == D.id).join(C, C.d_id == D.id)",
        primaryjoin="and_(A.b_id == B.id, A.id == C.a_id)",
        secondaryjoin="D.id == B.d_id",
        uselist=False,
        viewonly=True,
    )

class B(Base):
    __tablename__ = "b"

    id = mapped_column(Integer, primary_key=True)
    d_id = mapped_column(ForeignKey("d.id"))

class C(Base):
    __tablename__ = "c"

    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))
    d_id = mapped_column(ForeignKey("d.id"))

class D(Base):
    __tablename__ = "d"

    id = mapped_column(Integer, primary_key=True)
```

在上面的示例中，我们直接引用了命名为`a`、`b`、`c`、`d`的表，提供了`relationship.secondary`、`relationship.primaryjoin`和`relationship.secondaryjoin`这三个声明式样式的参数。从`A`到`D`的查询如下所示：

```py
sess.scalars(select(A).join(A.d)).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (
  b  AS  b_1  JOIN  d  AS  d_1  ON  b_1.d_id  =  d_1.id
  JOIN  c  AS  c_1  ON  c_1.d_id  =  d_1.id)
  ON  a.b_id  =  b_1.id  AND  a.id  =  c_1.a_id  JOIN  d  ON  d.id  =  b_1.d_id 
```

在上面的示例中，我们利用能够将多个表放入“secondary”容器的优势，以便我们可以跨多个表进行连接，同时保持对`relationship()`的“简化”，在这种情况下，“左”和“右”两侧都只有“一个”表；复杂性保持在中间。

警告

像上面的关系通常标记为`viewonly=True`，使用`relationship.viewonly`，应视为只读。虽然有时可以使类似上面的关系可写，但这通常很复杂且容易出错。

另请参阅

使用 viewonly 关系参数的注意事项  ## 别名类的关系

在前一节中，我们介绍了一种技术，其中我们使用`relationship.secondary`来在连接条件中放置额外的表。有一种复杂的连接情况，即使使用这种技术也不够；当我们试图从`A`连接到`B`，并在其中使用任意数量的`C`、`D`等，但`A`和`B`之间也直接有连接条件时。在这种情况下，仅仅使用一个复杂的`relationship.primaryjoin`条件可能难以表达从`A`到`B`的连接，因为中间表可能需要特殊处理，并且也不能用`relationship.secondary`对象来表达，因为`A->secondary->B`模式不支持`A`和`B`之间的任何引用。当出现这种**极其复杂**的情况时，我们可以采用创建第二个映射作为关系目标的方法。这就是我们使用`AliasedClass`来制作一个包含我们所需的所有额外表的类的映射。为了将此映射作为我们类的“替代”映射生成，我们使用`aliased()`函数生成新的构造，然后针对该对象使用`relationship()`，就像它是一个普通的映射类一样。

下面说明了一个从`A`到`B`的`relationship()`，其中主要连接条件增加了两个额外的实体`C`和`D`，这两个实体必须同时与`A`和`B`中的行对齐：

```py
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

class B(Base):
    __tablename__ = "b"

    id = mapped_column(Integer, primary_key=True)

class C(Base):
    __tablename__ = "c"

    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))

    some_c_value = mapped_column(String)

class D(Base):
    __tablename__ = "d"

    id = mapped_column(Integer, primary_key=True)
    c_id = mapped_column(ForeignKey("c.id"))
    b_id = mapped_column(ForeignKey("b.id"))

    some_d_value = mapped_column(String)

# 1\. set up the join() as a variable, so we can refer
# to it in the mapping multiple times.
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

# 2\. Create an AliasedClass to B
B_viacd = aliased(B, j, flat=True)

A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

使用上述映射，简单的连接如下所示：

```py
sess.scalars(select(A).join(A.b)).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  ON  a.b_id  =  b.id 
```

### 将别名类映射与类型和避免早期映射器配置集成

对映射类使用`aliased()`构造会强制执行`configure_mappers()`步骤，该步骤将解析所有当前类及其关系。如果当前映射需要但尚未声明的不相关映射类，或者如果关系本身的配置需要访问尚未声明的类，则可能会出现问题。此外，当关系提前声明时，SQLAlchemy 的声明模式与 Python 类型最有效地配合使用。

为了组织关系的构建以解决这些问题，可以使用像`MapperEvents.before_mapper_configured()`这样的配置级事件钩子，该钩子仅在所有映射准备好进行配置时才会调用配置代码：

```py
from sqlalchemy import event

class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # do the above configuration in a configuration hook

    j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, j, flat=True)
    A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

上面，函数`_configure_ab_relationship()`仅在请求完全配置的`A`版本时调用，此时类`B`、`D`和`C`将可用。

对于与内联类型配合使用的方法，可以使用类似的技术有效地生成用于别名类的“单例”创建模式，其中它作为全局变量进行了延迟初始化，然后可以在关系内联中使用：

```py
from typing import Any

B_viacd: Any = None
b_viacd_join: Any = None

class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)
    b_id: Mapped[int] = mapped_column(ForeignKey("b.id"))

    # 1\. the relationship can be declared using lambdas, allowing it to resolve
    #    to targets that are late-configured
    b: Mapped[B] = relationship(
        lambda: B_viacd, primaryjoin=lambda: A.b_id == b_viacd_join.c.b_id
    )

# 2\. configure the targets of the relationship using a before_mapper_configured
#    hook.
@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # 3\. set up the join() and AliasedClass as globals from within
    #    the configuration hook.

    global B_viacd, b_viacd_join

    b_viacd_join = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, b_viacd_join, flat=True)
```

### 在查询中使用 AliasedClass 目标

在前面的示例中，`A.b`关系指的是`B_viacd`实体作为目标，而**不是**直接的`B`类。要添加涉及`A.b`关系的额外条件，通常需要直接引用`B_viacd`，而不是使用`B`，特别是在`A.b`的目标实体要转换为别名或子查询的情况下。下面用子查询而不是连接说明了相同的关系：

```py
subq = select(B).join(D, D.b_id == B.id).join(C, C.id == D.c_id).subquery()

B_viacd_subquery = aliased(B, subq)

A.b = relationship(B_viacd_subquery, primaryjoin=A.b_id == subq.c.id)
```

使用上述`A.b`关系的查询将呈现为一个子查询：

```py
sess.scalars(select(A).join(A.b)).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (SELECT  b.id  AS  id,  b.some_b_column  AS  some_b_column
FROM  b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  AS  anon_1  ON  a.b_id  =  anon_1.id 
```

如果我们想要基于`A.b`连接添加额外的条件，则必须根据`B_viacd_subquery`而不是直接根据`B`来做：

```py
sess.scalars(
    select(A)
    .join(A.b)
    .where(B_viacd_subquery.some_b_column == "some b")
    .order_by(B_viacd_subquery.id)
).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (SELECT  b.id  AS  id,  b.some_b_column  AS  some_b_column
FROM  b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  AS  anon_1  ON  a.b_id  =  anon_1.id
WHERE  anon_1.some_b_column  =  ?  ORDER  BY  anon_1.id 
```  ## 使用窗口函数限制行关系

另一个与`AliasedClass`对象关系的有趣用例是在关系需要连接到任意形式的专门 SELECT 的情况下。一种情况是当需要使用窗口函数时，例如限制应返回多少行以供关系使用。下面的示例说明了一个非主映射器关系，它将为每个集合加载前十个项目：

```py
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))

partition = select(
    B, func.row_number().over(order_by=B.id, partition_by=B.a_id).label("index")
).alias()

partitioned_b = aliased(B, partition)

A.partitioned_bs = relationship(
    partitioned_b, primaryjoin=and_(partitioned_b.a_id == A.id, partition.c.index < 10)
)
```

我们可以将上述`partitioned_bs`关系与大多数加载器策略一起使用，例如`selectinload()`：

```py
for a1 in session.scalars(select(A).options(selectinload(A.partitioned_bs))):
    print(a1.partitioned_bs)  # <-- will be no more than ten objects
```

上面，"selectinload" 查询如下所示：

```py
SELECT
  a_1.id  AS  a_1_id,  anon_1.id  AS  anon_1_id,  anon_1.a_id  AS  anon_1_a_id,
  anon_1.data  AS  anon_1_data,  anon_1.index  AS  anon_1_index
FROM  a  AS  a_1
JOIN  (
  SELECT  b.id  AS  id,  b.a_id  AS  a_id,  b.data  AS  data,
  row_number()  OVER  (PARTITION  BY  b.a_id  ORDER  BY  b.id)  AS  index
  FROM  b)  AS  anon_1
ON  anon_1.a_id  =  a_1.id  AND  anon_1.index  <  %(index_1)s
WHERE  a_1.id  IN  (  ...  primary  key  collection  ...)
ORDER  BY  a_1.id
```

上述情况下，对于“a”中的每个匹配主键，我们将按“b.id”排序获取前十个“bs”。通过在“a_id”上进行分区，我们确保每个“行号”都是相对于父“a_id”的局部的。

这样的映射通常也会包括从“A”到“B”的“普通”关系，用于持久性操作以及当需要每个“A”的完整“B”对象集合时。## 构建查询可用的属性

非常雄心勃勃的自定义连接条件可能无法直接持久化，并且在某些情况下甚至可能无法正确加载。要删除等式的持久性部分，请在`relationship()`上使用标志`relationship.viewonly`，将其建立为只读属性（写入到集合的数据将在刷新时被忽略）。然而，在极端情况下，考虑与`Query`一起使用普通的 Python 属性，如下所示：

```py
class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)

    @property
    def addresses(self):
        return object_session(self).query(Address).with_parent(self).filter(...).all()
```

在其他情况下，描述符可以构建以利用现有的 Python 数据。有关使用描述符和混合体的更一般讨论，请参见使用描述符和混合体部分。

另见

使用描述符和混合体  ## 关于使用 viewonly 关系参数的注意事项

当应用于`relationship()`构造时，`relationship.viewonly`参数指示这个`relationship()`不会参与任何 ORM 工作单元操作，并且该属性不希望在其表示的集合的 Python 变异中参与。这意味着虽然只读关系可能引用一个可变的 Python 集合，如列表或集合，但对该列表或集合进行更改，如在映射实例上存在的那样，对 ORM 刷新过程**没有影响**。

要探索这种情况，请考虑这种映射：

```py
from __future__ import annotations

import datetime

from sqlalchemy import and_
from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str | None]

    all_tasks: Mapped[list[Task]] = relationship()

    current_week_tasks: Mapped[list[Task]] = relationship(
        primaryjoin=lambda: and_(
            User.id == Task.user_account_id,
            # this expression works on PostgreSQL but may not be supported
            # by other database engines
            Task.task_date >= func.now() - datetime.timedelta(days=7),
        ),
        viewonly=True,
    )

class Task(Base):
    __tablename__ = "task"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_account_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    description: Mapped[str | None]
    task_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    user: Mapped[User] = relationship(back_populates="current_week_tasks")
```

以下各节将注意这种配置的不同方面。

### 包括反向引用的 Python 中的变异与 viewonly=True 不适用

上述映射将`User.current_week_tasks`只读关系作为`Task.user`属性的反向引用目标。这目前并未被 SQLAlchemy 的 ORM 配置过程标记，但这是一个配置错误。改变`Task`上的`.user`属性不会影响`.current_week_tasks`属性：

```py
>>> u1 = User()
>>> t1 = Task(task_date=datetime.datetime.now())
>>> t1.user = u1
>>> u1.current_week_tasks
[]
```

这里还有另一个参数叫做`relationship.sync_backrefs`，可以在这里打开，以允许在这种情况下对`.current_week_tasks`进行变异，然而这并不被认为是最佳实践，对于一个只读关系，不应该依赖于 Python 中的变异。

在这种映射中，可以在`User.all_tasks`和`Task.user`之间配置反向引用，因为这两者都不是只读的，将正常同步。

除了禁用只读关系的反向引用变异问题外，Python 中对`User.all_tasks`集合的普通更改也不会反映在`User.current_week_tasks`集合中，直到更改已刷新到数据库。

总的来说，对于一个需要立即响应 Python 中的变异的自定义集合的用例，只读关系通常不合适。更好的方法是使用 SQLAlchemy 的混合属性功能，或者对于仅实例情况，使用 Python 的`@property`，其中可以实现一个根据当前 Python 实例生成的用户定义集合。要将我们的示例更改为这种方式工作，我们修复`Task.user`上的`relationship.back_populates`参数，引用`User.all_tasks`，然后演示一个简单的`@property`，将以立即`User.all_tasks`集合的形式提供结果：

```py
class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str | None]

    all_tasks: Mapped[list[Task]] = relationship(back_populates="user")

    @property
    def current_week_tasks(self) -> list[Task]:
        past_seven_days = datetime.datetime.now() - datetime.timedelta(days=7)
        return [t for t in self.all_tasks if t.task_date >= past_seven_days]

class Task(Base):
    __tablename__ = "task"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_account_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    description: Mapped[str | None]
    task_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    user: Mapped[User] = relationship(back_populates="all_tasks")
```

使用在 Python 中每次动态计算的集合，我们保证始终有正确的答案，而无需使用数据库：

```py
>>> u1 = User()
>>> t1 = Task(task_date=datetime.datetime.now())
>>> t1.user = u1
>>> u1.current_week_tasks
[<__main__.Task object at 0x7f3d699523c0>]
```

### viewonly=True 的集合/属性直到过期才会重新查询

继续使用原始的 viewonly 属性，如果我们确实对 persistent 对象上的 `User.all_tasks` 集合进行更改，则在 **两个** 事情发生之后，viewonly 集合只能显示此更改的净结果。第一个是将更改刷新到 `User.all_tasks` 中，以便新数据在数据库中可用，至少在本地事务的范围内是如此。第二个是 `User.current_week_tasks` 属性被 expired 并通过对数据库进行新的 SQL 查询重新加载。

为了支持这一需求，最简单的流程是仅在主要是只读操作中使用 **viewonly 关系**。例如，如果我们从数据库中检索到一个新的 `User`，那么集合将是当前的：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b906b0>]
```

当我们对 `u1.all_tasks` 进行修改时，如果想要在 `u1.current_week_tasks` 视图关系中看到这些更改，这些更改需要被刷新，并且 `u1.current_week_tasks` 属性需要过期，以便在下一次访问时进行 惰性加载。最简单的方法是使用 `Session.commit()`，保持 `Session.expire_on_commit` 参数设置为其默认值 `True`：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.commit()
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b90ec0>, <__main__.Task object at 0x7f8711b90a10>]
```

上面，对 `Session.commit()` 的调用将更改刷新到了数据库中的 `u1.all_tasks`，然后使所有对象过期，因此当我们访问 `u1.current_week_tasks` 时，会发生 :term:`惰性加载`，从数据库中新鲜获取此属性的内容。

要拦截操作而不实际提交事务，需要先显式 expired 该属性。一种简单的方法是直接调用它。在下面的示例中，`Session.flush()` 将挂起的更改发送到数据库，然后使用 `Session.expire()` 来过期 `u1.current_week_tasks` 集合，以便在下一次访问时重新获取：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.flush()
...     sess.expire(u1, ["current_week_tasks"])
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
```

事实上，我们可以跳过对`Session.flush()`的调用，假设一个保持`Session.autoflush`为其默认值`True`的`Session`，因为过期的`current_week_tasks`属性在过期后访问时将触发自动刷新：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.expire(u1, ["current_week_tasks"])
...     print(u1.current_week_tasks)  # triggers autoflush before querying
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
```

继续使用上述方法进行更详细的处理，我们可以在相关的`User.all_tasks`集合发生变化时通过 event hooks 进行程序化过期。这是一种**高级技术**，应该首先检查更简单的架构，比如`@property`或坚持只读用例。在我们简单的示例中，这将被配置为：

```py
from sqlalchemy import event, inspect

@event.listens_for(User.all_tasks, "append")
@event.listens_for(User.all_tasks, "remove")
@event.listens_for(User.all_tasks, "bulk_replace")
def _expire_User_current_week_tasks(target, value, initiator):
    inspect(target).session.expire(target, ["current_week_tasks"])
```

有了上述钩子，突变操作被拦截并导致`User.current_week_tasks`集合自动过期：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f66d093ccb0>, <__main__.Task object at 0x7f66d093cce0>]
```

上述使用的`AttributeEvents`事件钩子也会被 backref 突变触发，因此，使用上述钩子会拦截对`Task.user`的更改：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     t1 = Task(task_date=datetime.datetime.now())
...     t1.user = u1
...     sess.add(t1)
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f3b0c070d10>, <__main__.Task object at 0x7f3b0c057d10>]
```  ## 处理多个连接路径

处理的最常见情况之一是两个表之间存在多个外键路径。

考虑一个包含对`Address`类的两个外键的`Customer`类：

```py
from sqlalchemy import Integer, ForeignKey, String, Column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class Customer(Base):
    __tablename__ = "customer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
    shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

    billing_address = relationship("Address")
    shipping_address = relationship("Address")

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    street = mapped_column(String)
    city = mapped_column(String)
    state = mapped_column(String)
    zip = mapped_column(String)
```

上述映射，在我们尝试使用它时，会产生错误：

```py
sqlalchemy.exc.AmbiguousForeignKeysError: Could not determine join
condition between parent/child tables on relationship
Customer.billing_address - there are multiple foreign key
paths linking the tables.  Specify the 'foreign_keys' argument,
providing a list of those columns which should be
counted as containing a foreign key reference to the parent table.
```

上述消息相当长。`relationship()`可能返回许多潜在消息，这些消息经过精心设计，以检测各种常见的配置问题；大多数消息都会建议需要解决模糊性或其他缺失信息的附加配置。

在这种情况下，消息希望我们为每个`relationship()`进行限定，指示每个外键列应该被考虑，并且适当的形式如下：

```py
class Customer(Base):
    __tablename__ = "customer"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)

    billing_address_id = mapped_column(Integer, ForeignKey("address.id"))
    shipping_address_id = mapped_column(Integer, ForeignKey("address.id"))

    billing_address = relationship("Address", foreign_keys=[billing_address_id])
    shipping_address = relationship("Address", foreign_keys=[shipping_address_id])
```

在上面的例子中，我们指定了`foreign_keys`参数，它是一个`Column`或`Column`对象列表，指示要考虑的“外键”列，或者换句话说，包含指向父表的值的列。从`Customer`对象加载`Customer.billing_address`关系将使用`billing_address_id`中存在的值来标识要加载的`Address`行；类似地，`shipping_address_id`用于`shipping_address`关系。这两列的关联在持久化过程中也起着作用；刚插入的`Address`对象的新生成的主键将在刷新期间被复制到关联的`Customer`对象的适当外键列中。

在使用 Declarative 指定`foreign_keys`时，我们还可以使用字符串名称进行指定，但是重要的是，如果使用列表，**列表是字符串的一部分**：

```py
billing_address = relationship("Address", foreign_keys="[Customer.billing_address_id]")
```

在这个具体的例子中，在任何情况下列表都是不必要的，因为我们只需要一个`Column`：

```py
billing_address = relationship("Address", foreign_keys="Customer.billing_address_id")
```

警告

当作为 Python 可评估字符串传递时，`relationship.foreign_keys` 参数将使用 Python 的 `eval()` 函数进行解释。**请勿将不受信任的输入传递给此字符串**。详情请参阅关系参数的评估以了解有关`relationship()` 参数的声明性评估的详细信息。

## 指定备用连接条件

在构建连接时，`relationship()`的默认行为是将一侧的主键列的值等同于另一侧的外键引用列的值。我们可以使用`relationship.primaryjoin`参数来更改此标准为任何我们喜欢的内容，以及在使用“次要”表时，在使用`relationship.secondaryjoin`参数。

在下面的例子中，我们使用`User`类以及一个存储街道地址的`Address`类，我们创建了一个关系`boston_addresses`，它只会加载那些指定城市为“Boston”的`Address`对象：

```py
from sqlalchemy import Integer, ForeignKey, String, Column
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)
    name = mapped_column(String)
    boston_addresses = relationship(
        "Address",
        primaryjoin="and_(User.id==Address.user_id, Address.city=='Boston')",
    )

class Address(Base):
    __tablename__ = "address"
    id = mapped_column(Integer, primary_key=True)
    user_id = mapped_column(Integer, ForeignKey("user.id"))

    street = mapped_column(String)
    city = mapped_column(String)
    state = mapped_column(String)
    zip = mapped_column(String)
```

在这个字符串的 SQL 表达式中，我们使用了 `and_()` 连接构造来建立两个不同的谓词，用于连接 `User.id` 和 `Address.user_id` 列，以及将 `Address` 中的行限制为只有 `city='Boston'`。当使用声明式时，类似 `and_()` 这样的基本 SQL 函数会自动在字符串 `relationship()` 参数的计算命名空间中可用。

警告

当作为 Python 可评估字符串传递时，`relationship.primaryjoin` 参数是使用 Python 的 `eval()` 函数解释的。**不要将不受信任的输入传递给此字符串**。有关声明式评估 `relationship()` 参数的详细信息，请参阅 关系参数的评估。

我们在 `relationship.primaryjoin` 中使用的自定义条件通常只在 SQLAlchemy 渲染 SQL 以加载或表示此关系时才重要。也就是说，在执行每个属性的惰性加载的 SQL 语句中使用它，或者在查询时构造连接，例如通过 `Select.join()` 或通过急切的“连接”或“子查询”加载样式。当操作内存中的对象时，我们可以将任何我们想要的 `Address` 对象放入 `boston_addresses` 集合中，而不管 `.city` 属性的值是什么。这些对象将保留在集合中，直到属性过期并重新从应用条件的数据库中加载为止。当执行刷新时，`boston_addresses` 中的对象将被无条件地刷新，将主键 `user.id` 列的值分配到每行的持有外键 `address.user_id` 列。这里的 `city` 条件没有效果，因为刷新过程只关心将主键值同步到引用外键值。

## 创建自定义外键条件

主要连接条件的另一个元素是如何确定那些被认为是“外部”的列的。通常，一些 `Column` 对象的子集将指定 `ForeignKey`，或者是 `ForeignKeyConstraint` 的一部分，这与连接条件相关。`relationship()` 查看这个外键状态，以确定它应该如何为这个关系加载和持久化数据。然而，`relationship.primaryjoin` 参数可以用来创建一个不涉及任何“模式”级外键的连接条件。我们可以显式地结合 `relationship.primaryjoin` 以及 `relationship.foreign_keys` 和 `relationship.remote_side` 来建立这样一个连接。

下面，一个 `HostEntry` 类与自身连接，将字符串 `content` 列等同于 `ip_address` 列，这是一个名为 `INET` 的 PostgreSQL 类型。我们需要使用 `cast()` 来将连接的一侧转换为另一侧的类型：

```py
from sqlalchemy import cast, String, Column, Integer
from sqlalchemy.orm import relationship
from sqlalchemy.dialects.postgresql import INET

from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

class HostEntry(Base):
    __tablename__ = "host_entry"

    id = mapped_column(Integer, primary_key=True)
    ip_address = mapped_column(INET)
    content = mapped_column(String(50))

    # relationship() using explicit foreign_keys, remote_side
    parent_host = relationship(
        "HostEntry",
        primaryjoin=ip_address == cast(content, INET),
        foreign_keys=content,
        remote_side=ip_address,
    )
```

上述关系将产生类似以下的连接：

```py
SELECT  host_entry.id,  host_entry.ip_address,  host_entry.content
FROM  host_entry  JOIN  host_entry  AS  host_entry_1
ON  host_entry_1.ip_address  =  CAST(host_entry.content  AS  INET)
```

以上的另一种语法是在 `relationship.primaryjoin` 表达式内联使用 `foreign()` 和 `remote()` annotations。这种语法表示了 `relationship()` 通常自己应用于连接条件的注释，考虑到 `relationship.foreign_keys` 和 `relationship.remote_side` 参数。当存在明确的连接条件时，这些函数可能更简洁，并且还可以标记出“外部”或“远程”的确切列，而不管该列是否在多次声明或在复杂的 SQL 表达式中：

```py
from sqlalchemy.orm import foreign, remote

class HostEntry(Base):
    __tablename__ = "host_entry"

    id = mapped_column(Integer, primary_key=True)
    ip_address = mapped_column(INET)
    content = mapped_column(String(50))

    # relationship() using explicit foreign() and remote() annotations
    # in lieu of separate arguments
    parent_host = relationship(
        "HostEntry",
        primaryjoin=remote(ip_address) == cast(foreign(content), INET),
    )
```

## 在连接条件中使用自定义运算符

另一个关系的用例是使用自定义运算符，比如 PostgreSQL 的“包含在内” `<<` 运算符，当与诸如 `INET` 和 `CIDR` 这样的类型连接时。对于自定义布尔运算符，我们使用 `Operators.bool_op()` 函数：

```py
inet_column.bool_op("<<")(cidr_column)
```

像上面的比较可以直接用于构造 `relationship()` 中的 `relationship.primaryjoin`：

```py
class IPA(Base):
    __tablename__ = "ip_address"

    id = mapped_column(Integer, primary_key=True)
    v4address = mapped_column(INET)

    network = relationship(
        "Network",
        primaryjoin="IPA.v4address.bool_op('<<')(foreign(Network.v4representation))",
        viewonly=True,
    )

class Network(Base):
    __tablename__ = "network"

    id = mapped_column(Integer, primary_key=True)
    v4representation = mapped_column(CIDR)
```

上述，像这样的查询：

```py
select(IPA).join(IPA.network)
```

将显示为：

```py
SELECT  ip_address.id  AS  ip_address_id,  ip_address.v4address  AS  ip_address_v4address
FROM  ip_address  JOIN  network  ON  ip_address.v4address  <<  network.v4representation
```

## 基于 SQL 函数的自定义运算符

与 `Operators.op.is_comparison` 用例的变体是当我们不是使用运算符，而是使用 SQL 函数。这种用例的典型例子是 PostgreSQL PostGIS 函数，但任何解析为二进制条件的任何数据库上的 SQL 函数都可能适用。为适应这种用例，`FunctionElement.as_comparison()` 方法可以修改任何 SQL 函数，例如从 `func` 命名空间调用的函数，以指示 ORM 该函数生成了两个表达式的比较。下面的例子使用了 [Geoalchemy2](https://geoalchemy-2.readthedocs.io/) 库说明了这一点：

```py
from geoalchemy2 import Geometry
from sqlalchemy import Column, Integer, func
from sqlalchemy.orm import relationship, foreign

class Polygon(Base):
    __tablename__ = "polygon"
    id = mapped_column(Integer, primary_key=True)
    geom = mapped_column(Geometry("POLYGON", srid=4326))
    points = relationship(
        "Point",
        primaryjoin="func.ST_Contains(foreign(Polygon.geom), Point.geom).as_comparison(1, 2)",
        viewonly=True,
    )

class Point(Base):
    __tablename__ = "point"
    id = mapped_column(Integer, primary_key=True)
    geom = mapped_column(Geometry("POINT", srid=4326))
```

上述，`FunctionElement.as_comparison()` 表示 `func.ST_Contains()` SQL 函数正在比较 `Polygon.geom` 和 `Point.geom` 表达式。`foreign()` 注释另外指出了在这种特定关系中扮演“外键”角色的列。

新版本 1.3 中新增了 `FunctionElement.as_comparison()`。

## 重叠的外键

很少见的情况可能会出现，即使用复合外键，以便单个列可能是通过外键约束引用的多个列的主题。

考虑一个（诚然复杂的）映射，例如`Magazine`对象，使用包括`magazine_id`的复合主键方案，分别由`Writer`对象和`Article`对象引用；然后，为了使`Article`也引用`Writer`，`Article.magazine_id`涉及到两个不同的关系；`Article.magazine`和`Article.writer`：

```py
class Magazine(Base):
    __tablename__ = "magazine"

    id = mapped_column(Integer, primary_key=True)

class Article(Base):
    __tablename__ = "article"

    article_id = mapped_column(Integer)
    magazine_id = mapped_column(ForeignKey("magazine.id"))
    writer_id = mapped_column()

    magazine = relationship("Magazine")
    writer = relationship("Writer")

    __table_args__ = (
        PrimaryKeyConstraint("article_id", "magazine_id"),
        ForeignKeyConstraint(
            ["writer_id", "magazine_id"], ["writer.id", "writer.magazine_id"]
        ),
    )

class Writer(Base):
    __tablename__ = "writer"

    id = mapped_column(Integer, primary_key=True)
    magazine_id = mapped_column(ForeignKey("magazine.id"), primary_key=True)
    magazine = relationship("Magazine")
```

当上述映射被配置时，我们将看到此警告被发出：

```py
SAWarning: relationship 'Article.writer' will copy column
writer.magazine_id to column article.magazine_id,
which conflicts with relationship(s): 'Article.magazine'
(copies magazine.id to article.magazine_id). Consider applying
viewonly=True to read-only relationships, or provide a primaryjoin
condition marking writable columns with the foreign() annotation.
```

这指的是`Article.magazine_id`是两个不同外键约束的主体；它直接引用`Magazine.id`作为源列，但在与`Writer`的复合键上下文中，也引用`Writer.magazine_id`作为源列。如果我们将`Article`与特定的`Magazine`关联起来，但然后将`Article`与另一个与*不同*`Magazine`关联的`Writer`关联起来，ORM 会非确定性地覆盖`Article.magazine_id`，悄悄地改变我们所引用的杂志；如果我们将`Writer`从`Article`中取消关联，它还可能尝试将 NULL 放入此列。警告让我们知道这种情况。

要解决这个问题，我们需要打破`Article`的行为，包括以下三个功能：

1.  首先，`Article`根据仅在`Article.magazine`关系中持久化的数据来写入`Article.magazine_id`，即从`Magazine.id`复制的值。

1.  `Article`可以代表在`Article.writer`关系中持久化的数据写入`Article.writer_id`，但只能写入`Writer.id`列；`Writer.magazine_id`列不应写入`Article.magazine_id`，因为它最终来自`Magazine.id`。

1.  当加载`Article.writer`时，`Article`考虑了`Article.magazine_id`，即使在此关系中并不代表它。

要获取只有#1 和#2，我们可以将`Article.writer_id`指定为`Article.writer`的“外键”：

```py
class Article(Base):
    # ...

    writer = relationship("Writer", foreign_keys="Article.writer_id")
```

然而，这会导致`Article.writer`在与`Writer`进行查询时不考虑`Article.magazine_id`：

```py
SELECT  article.article_id  AS  article_article_id,
  article.magazine_id  AS  article_magazine_id,
  article.writer_id  AS  article_writer_id
FROM  article
JOIN  writer  ON  writer.id  =  article.writer_id
```

因此，要获取#1、#2 和#3 的所有内容，我们需要通过完全组合`relationship.primaryjoin`来表达连接条件，以及要写入的列，同时使用`relationship.foreign_keys`参数，或者更简洁地使用`foreign()`进行注释：

```py
class Article(Base):
    # ...

    writer = relationship(
        "Writer",
        primaryjoin="and_(Writer.id == foreign(Article.writer_id), "
        "Writer.magazine_id == Article.magazine_id)",
    )
```

## 非关系比较 / 材料化路径

警告

本节详细介绍了一个实验性功能。

使用自定义表达式意味着我们可以生成不遵循通常的主键/外键模型的非正统连接条件。其中一个例子是材料化路径模式，我们在比较字符串以产生重叠路径标记时，以便生成树结构。

通过谨慎使用`foreign()`和`remote()`，我们可以构建一个有效地生成基本材料化路径系统的关系。基本上，当`foreign()`和`remote()`在*相同*的比较表达式一侧时，关系被认为是“一对多”；当它们在*不同*的一侧时，关系被认为是“多对一”。对于我们将在此处使用的比较，我们将处理集合，所以我们保持事物配置为“一对多”：

```py
class Element(Base):
    __tablename__ = "element"

    path = mapped_column(String, primary_key=True)

    descendants = relationship(
        "Element",
        primaryjoin=remote(foreign(path)).like(path.concat("/%")),
        viewonly=True,
        order_by=path,
    )
```

上文中，如果给定具有`"/foo/bar2"`路径属性的`Element`对象，则我们寻找对`Element.descendants`的加载应如下所示：

```py
SELECT  element.path  AS  element_path
FROM  element
WHERE  element.path  LIKE  ('/foo/bar2'  ||  '/%')  ORDER  BY  element.path
```

## 自我引用多对多关系

另请参阅

本节记录了“邻接列表”模式的两个表变体，该模式在邻接列表关系中有所记录。务必查看自我引用查询模式的子部分自我引用查询策略和配置自我引用急加载，它们同样适用于此处讨论的映射模式。

多对多关系可以通过`relationship.primaryjoin`和/或`relationship.secondaryjoin`中的一个或两个进行自定义 - 后者对于使用`relationship.secondary`参数指定多对多引用的关系非常重要。涉及使用`relationship.primaryjoin`和`relationship.secondaryjoin`的常见情况是在从一个类到自身建立多对多关系时，如下所示：

```py
from typing import List

from sqlalchemy import Integer, ForeignKey, Column, Table
from sqlalchemy.orm import DeclarativeBase, Mapped
from sqlalchemy.orm import mapped_column, relationship

class Base(DeclarativeBase):
    pass

node_to_node = Table(
    "node_to_node",
    Base.metadata,
    Column("left_node_id", Integer, ForeignKey("node.id"), primary_key=True),
    Column("right_node_id", Integer, ForeignKey("node.id"), primary_key=True),
)

class Node(Base):
    __tablename__ = "node"
    id: Mapped[int] = mapped_column(primary_key=True)
    label: Mapped[str]
    right_nodes: Mapped[List["Node"]] = relationship(
        "Node",
        secondary=node_to_node,
        primaryjoin=id == node_to_node.c.left_node_id,
        secondaryjoin=id == node_to_node.c.right_node_id,
        back_populates="left_nodes",
    )
    left_nodes: Mapped[List["Node"]] = relationship(
        "Node",
        secondary=node_to_node,
        primaryjoin=id == node_to_node.c.right_node_id,
        secondaryjoin=id == node_to_node.c.left_node_id,
        back_populates="right_nodes",
    )
```

在上述情况下，SQLAlchemy 无法自动知道哪些列应该连接到`right_nodes`和`left_nodes`关系的哪些列上。`relationship.primaryjoin`和`relationship.secondaryjoin`参数建立了我们希望如何加入关联表的方式。在上面的声明形式中，由于我们在对应于`Node`类的 Python 块中声明了这些条件，因此`id`变量直接作为我们希望与之连接的`Column`对象是可用的。

或者，我们可以使用字符串定义`relationship.primaryjoin`和`relationship.secondaryjoin`参数，在我们的配置中可能还没有`Node.id`列对象可用，或者`node_to_node`表可能还不可用的情况下非常适用。当在声明字符串中引用普通的`Table`对象时，我们使用表的字符串名称，就像在`MetaData`中一样：

```py
class Node(Base):
    __tablename__ = "node"
    id = mapped_column(Integer, primary_key=True)
    label = mapped_column(String)
    right_nodes = relationship(
        "Node",
        secondary="node_to_node",
        primaryjoin="Node.id==node_to_node.c.left_node_id",
        secondaryjoin="Node.id==node_to_node.c.right_node_id",
        backref="left_nodes",
    )
```

警告

当作为 Python 可评估字符串传递时，`relationship.primaryjoin`和`relationship.secondaryjoin`参数使用 Python 的`eval()`函数解释。**不要将不受信任的输入传递给这些字符串**。有关声明性`relationship()`参数的评估详细信息，请参阅关系参数的评估。

在这里的经典映射情况中，`node_to_node`可以连接到`node.c.id`：

```py
from sqlalchemy import Integer, ForeignKey, String, Column, Table, MetaData
from sqlalchemy.orm import relationship, registry

metadata_obj = MetaData()
mapper_registry = registry()

node_to_node = Table(
    "node_to_node",
    metadata_obj,
    Column("left_node_id", Integer, ForeignKey("node.id"), primary_key=True),
    Column("right_node_id", Integer, ForeignKey("node.id"), primary_key=True),
)

node = Table(
    "node",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("label", String),
)

class Node:
    pass

mapper_registry.map_imperatively(
    Node,
    node,
    properties={
        "right_nodes": relationship(
            Node,
            secondary=node_to_node,
            primaryjoin=node.c.id == node_to_node.c.left_node_id,
            secondaryjoin=node.c.id == node_to_node.c.right_node_id,
            backref="left_nodes",
        )
    },
)
```

请注意，在这两个示例中，`relationship.backref` 关键字都指定了一个 `left_nodes` 回引 - 当`relationship()`在反向创建第二个关系时，它足够聪明地反转了`relationship.primaryjoin` 和 `relationship.secondaryjoin` 参数。

另请参阅

+   邻接列表关系 - 单表版本

+   自引用查询策略 - 使用自引用映射查询的技巧

+   配置自引用预加载 - 使用自引用映射预加载的技巧

## 复合“次要”连接

注意

本节涵盖了一些在某种程度上受 SQLAlchemy 支持的边缘案例，但建议尽可能在可能的情况下通过使用合理的关系布局和/或 Python 内的属性来解决类似问题。

有时，当一个人试图在两个表之间建立一个`relationship()`时，需要涉及更多的表，而不仅仅是两个或三个表。这是一个`relationship()`的领域，在这个领域，人们试图推动可能性的边界，并且通常对许多这些奇特用例的最终解决方案需要在 SQLAlchemy 邮件列表上讨论。

在较新版本的 SQLAlchemy 中，`relationship.secondary` 参数可用于某些情况，以提供由多个表组成的复合目标。以下是这种连接条件的示例（至少需要版本 0.9.2 才能正常运行）：

```py
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

    d = relationship(
        "D",
        secondary="join(B, D, B.d_id == D.id).join(C, C.d_id == D.id)",
        primaryjoin="and_(A.b_id == B.id, A.id == C.a_id)",
        secondaryjoin="D.id == B.d_id",
        uselist=False,
        viewonly=True,
    )

class B(Base):
    __tablename__ = "b"

    id = mapped_column(Integer, primary_key=True)
    d_id = mapped_column(ForeignKey("d.id"))

class C(Base):
    __tablename__ = "c"

    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))
    d_id = mapped_column(ForeignKey("d.id"))

class D(Base):
    __tablename__ = "d"

    id = mapped_column(Integer, primary_key=True)
```

在上面的示例中，我们提供了`relationship.secondary`、`relationship.primaryjoin` 和`relationship.secondaryjoin` 这三个参数，在声明样式中直接引用了命名表 `a`、`b`、`c`、`d`。从 `A` 到 `D` 的查询如下：

```py
sess.scalars(select(A).join(A.d)).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (
  b  AS  b_1  JOIN  d  AS  d_1  ON  b_1.d_id  =  d_1.id
  JOIN  c  AS  c_1  ON  c_1.d_id  =  d_1.id)
  ON  a.b_id  =  b_1.id  AND  a.id  =  c_1.a_id  JOIN  d  ON  d.id  =  b_1.d_id 
```

在上面的示例中，我们利用能够将多个表填入“次要”容器的优势，这样我们就可以跨多个表进行连接，同时保持对`relationship()`的“简单”使用，因为“左”和“右”两侧只有“一个”表；复杂性被保留在中间。

警告

类似上述的关系通常标记为`viewonly=True`，使用`relationship.viewonly`，应当被视为只读。虽然有时可以使类似上述的关系可写，但这通常是复杂且容易出错的。

另请参阅

关于使用只读关系参数的注意事项

## 别名类的关系

在前一节中，我们说明了一种技术，在这种技术中，我们使用了`relationship.secondary`来将额外的表放置在连接条件中。有一种复杂的连接情况，即使使用这种技术也不足够；当我们试图从`A`连接到`B`时，中间可能会使用任意数量的`C`、`D`等，但是在`A`和`B`之间也有直接的连接条件。在这种情况下，仅使用复杂的`relationship.primaryjoin`条件可能难以表达，因为中间表可能需要特殊处理，而且也不能使用`relationship.secondary`对象来表达，因为`A->secondary->B`模式不支持`A`和`B`之间的任何引用。当出现这种**极其高级**的情况时，我们可以采用创建第二个映射作为关系的目标。这就是我们使用`AliasedClass`来创建一个包含我们所需的所有额外表的类的映射。为了将这个映射作为我们类的“替代”映射生成，我们使用`aliased()`函数来生成新的结构，然后针对该对象使用`relationship()`，就像它是一个普通的映射类一样。

下面演示了从 `A` 到 `B` 的简单连接的 `relationship()`，但是主连接条件增加了另外两个实体 `C` 和 `D`，这两个实体必须同时与 `A` 和 `B` 中的行对应起来：

```py
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

class B(Base):
    __tablename__ = "b"

    id = mapped_column(Integer, primary_key=True)

class C(Base):
    __tablename__ = "c"

    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))

    some_c_value = mapped_column(String)

class D(Base):
    __tablename__ = "d"

    id = mapped_column(Integer, primary_key=True)
    c_id = mapped_column(ForeignKey("c.id"))
    b_id = mapped_column(ForeignKey("b.id"))

    some_d_value = mapped_column(String)

# 1\. set up the join() as a variable, so we can refer
# to it in the mapping multiple times.
j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)

# 2\. Create an AliasedClass to B
B_viacd = aliased(B, j, flat=True)

A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

使用上述映射，简单连接如下所示：

```py
sess.scalars(select(A).join(A.b)).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  ON  a.b_id  =  b.id 
```

### 将别名类映射与类型化结合，并避免早期映射器配置

`aliased()` 构造对映射类的创建强制执行 `configure_mappers()` 步骤，该步骤将解析所有当前类及其关系。如果当前映射所需的不相关映射类尚未声明，或者如果关系本身的配置需要访问尚未声明的类，则可能会出现问题。此外，当关系在前面声明时，SQLAlchemy 的声明模式与 Python 类型化的协作效果最佳。

为了组织关系的构建以解决这些问题，可以使用配置级别的事件钩子，如 `MapperEvents.before_mapper_configured()`，该钩子将仅在所有映射准备好配置时调用配置代码：

```py
from sqlalchemy import event

class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # do the above configuration in a configuration hook

    j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, j, flat=True)
    A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

在上面的示例中，当请求完全配置的 `A` 版本时，函数 `_configure_ab_relationship()` 将被调用，此时类 `B`、`D` 和 `C` 将可用。

对于与内联类型化集成的方法，可以使用类似的技术来有效地生成别名类的“单例”创建模式，其中它作为全局变量进行延迟初始化，然后可以在关系内联中使用它：

```py
from typing import Any

B_viacd: Any = None
b_viacd_join: Any = None

class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)
    b_id: Mapped[int] = mapped_column(ForeignKey("b.id"))

    # 1\. the relationship can be declared using lambdas, allowing it to resolve
    #    to targets that are late-configured
    b: Mapped[B] = relationship(
        lambda: B_viacd, primaryjoin=lambda: A.b_id == b_viacd_join.c.b_id
    )

# 2\. configure the targets of the relationship using a before_mapper_configured
#    hook.
@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # 3\. set up the join() and AliasedClass as globals from within
    #    the configuration hook.

    global B_viacd, b_viacd_join

    b_viacd_join = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, b_viacd_join, flat=True)
```

### 在查询中使用别名类目标

在前面的示例中，`A.b` 关系将 `B_viacd` 实体作为目标，而 **不是** 直接的 `B` 类。要添加涉及 `A.b` 关系的附加条件，通常需要直接引用 `B_viacd` 而不是使用 `B`，特别是在将 `A.b` 的目标实体转换为别名或子查询的情况下。下面演示了使用子查询而不是连接的相同关系：

```py
subq = select(B).join(D, D.b_id == B.id).join(C, C.id == D.c_id).subquery()

B_viacd_subquery = aliased(B, subq)

A.b = relationship(B_viacd_subquery, primaryjoin=A.b_id == subq.c.id)
```

使用上述 `A.b` 关系的查询将呈现一个子查询：

```py
sess.scalars(select(A).join(A.b)).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (SELECT  b.id  AS  id,  b.some_b_column  AS  some_b_column
FROM  b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  AS  anon_1  ON  a.b_id  =  anon_1.id 
```

如果我们想要根据 `A.b` 连接添加额外的条件，必须以 `B_viacd_subquery` 而不是直接以 `B` 的形式添加：

```py
sess.scalars(
    select(A)
    .join(A.b)
    .where(B_viacd_subquery.some_b_column == "some b")
    .order_by(B_viacd_subquery.id)
).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (SELECT  b.id  AS  id,  b.some_b_column  AS  some_b_column
FROM  b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  AS  anon_1  ON  a.b_id  =  anon_1.id
WHERE  anon_1.some_b_column  =  ?  ORDER  BY  anon_1.id 
```

### 将别名类映射与类型化结合，并避免早期映射器配置

对映射类创建 `aliased()` 构造会强制进行 `configure_mappers()` 步骤，这将解析所有当前的类及其关系。如果当前映射需要未声明的不相关的映射类，或者如果关系的配置本身需要访问尚未声明的类，则可能会出现问题。另外，当关系提前声明时，SQLAlchemy 的声明式模式与 Python 类型的工作方式最有效。

要组织关系构建以处理这些问题，可以使用一个配置级别的事件钩子，比如 `MapperEvents.before_mapper_configured()`，它只有在所有映射都准备好配置时才会调用配置代码：

```py
from sqlalchemy import event

class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)
    b_id = mapped_column(ForeignKey("b.id"))

@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # do the above configuration in a configuration hook

    j = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, j, flat=True)
    A.b = relationship(B_viacd, primaryjoin=A.b_id == j.c.b_id)
```

上述函数 `_configure_ab_relationship()` 只有在请求一个完全配置好的 `A` 版本时才会被调用，在这时 `B`、`D` 和 `C` 类会被加载。

对于与内联类型集成的方法，可以使用类似的技术有效地为别名类生成“单例”创建模式，在其中作为全局变量进行延迟初始化，然后可以在关系中使用：

```py
from typing import Any

B_viacd: Any = None
b_viacd_join: Any = None

class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)
    b_id: Mapped[int] = mapped_column(ForeignKey("b.id"))

    # 1\. the relationship can be declared using lambdas, allowing it to resolve
    #    to targets that are late-configured
    b: Mapped[B] = relationship(
        lambda: B_viacd, primaryjoin=lambda: A.b_id == b_viacd_join.c.b_id
    )

# 2\. configure the targets of the relationship using a before_mapper_configured
#    hook.
@event.listens_for(A, "before_mapper_configured")
def _configure_ab_relationship(mapper, cls):
    # 3\. set up the join() and AliasedClass as globals from within
    #    the configuration hook.

    global B_viacd, b_viacd_join

    b_viacd_join = join(B, D, D.b_id == B.id).join(C, C.id == D.c_id)
    B_viacd = aliased(B, b_viacd_join, flat=True)
```

### 在查询中使用 AliasedClass 目标

在前面的示例中，`A.b` 关系将 `B_viacd` 实体作为目标，而**不是**直接使用 `B` 类。要添加涉及 `A.b` 关系的额外条件，通常需要直接引用 `B_viacd` 而不是使用 `B`，特别是在目标实体 `A.b` 需要转换为别名或子查询的情况下。以下是使用子查询而不是连接的相同关系的示例：

```py
subq = select(B).join(D, D.b_id == B.id).join(C, C.id == D.c_id).subquery()

B_viacd_subquery = aliased(B, subq)

A.b = relationship(B_viacd_subquery, primaryjoin=A.b_id == subq.c.id)
```

使用上述 `A.b` 关系的查询将呈现一个子查询：

```py
sess.scalars(select(A).join(A.b)).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (SELECT  b.id  AS  id,  b.some_b_column  AS  some_b_column
FROM  b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  AS  anon_1  ON  a.b_id  =  anon_1.id 
```

如果我们想要根据 `A.b` 连接添加额外的条件，必须以 `B_viacd_subquery` 而不是直接使用 `B`：

```py
sess.scalars(
    select(A)
    .join(A.b)
    .where(B_viacd_subquery.some_b_column == "some b")
    .order_by(B_viacd_subquery.id)
).all()

SELECT  a.id  AS  a_id,  a.b_id  AS  a_b_id
FROM  a  JOIN  (SELECT  b.id  AS  id,  b.some_b_column  AS  some_b_column
FROM  b  JOIN  d  ON  d.b_id  =  b.id  JOIN  c  ON  c.id  =  d.c_id)  AS  anon_1  ON  a.b_id  =  anon_1.id
WHERE  anon_1.some_b_column  =  ?  ORDER  BY  anon_1.id 
```

## 带窗口函数的行限制关系

另一个关系到 `AliasedClass` 对象的有趣用例是关系需要连接到任何形式的专门 SELECT 时。一种情况是当需要使用窗口函数时，例如限制返回的行数。下面的示例说明了一个非主映射器关系，该关系将为每个集合加载前十个项目：

```py
class A(Base):
    __tablename__ = "a"

    id = mapped_column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = mapped_column(Integer, primary_key=True)
    a_id = mapped_column(ForeignKey("a.id"))

partition = select(
    B, func.row_number().over(order_by=B.id, partition_by=B.a_id).label("index")
).alias()

partitioned_b = aliased(B, partition)

A.partitioned_bs = relationship(
    partitioned_b, primaryjoin=and_(partitioned_b.a_id == A.id, partition.c.index < 10)
)
```

我们可以使用上述 `partitioned_bs` 关系与大多数加载器策略，例如 `selectinload()`:

```py
for a1 in session.scalars(select(A).options(selectinload(A.partitioned_bs))):
    print(a1.partitioned_bs)  # <-- will be no more than ten objects
```

在上面的例子中，“selectinload”查询如下所示：

```py
SELECT
  a_1.id  AS  a_1_id,  anon_1.id  AS  anon_1_id,  anon_1.a_id  AS  anon_1_a_id,
  anon_1.data  AS  anon_1_data,  anon_1.index  AS  anon_1_index
FROM  a  AS  a_1
JOIN  (
  SELECT  b.id  AS  id,  b.a_id  AS  a_id,  b.data  AS  data,
  row_number()  OVER  (PARTITION  BY  b.a_id  ORDER  BY  b.id)  AS  index
  FROM  b)  AS  anon_1
ON  anon_1.a_id  =  a_1.id  AND  anon_1.index  <  %(index_1)s
WHERE  a_1.id  IN  (  ...  primary  key  collection  ...)
ORDER  BY  a_1.id
```

在上面的例子中，对于“a”中的每个匹配的主键，我们将按照“b.id”的顺序获取前十个“bs”。通过在“a_id”上分区，我们确保每个“行号”都局限于父“a_id”。

这样的映射通常还会包括从“A”到“B”的“普通”关系，用于持久性操作以及当需要“A”每个对象的完整集合时。

## 构建查询可用的属性

非常雄心勃勃的自定义连接条件可能无法直接持久化，有些情况下甚至可能无法正确加载。要消除持久性方程式的部分，使用标志`relationship.viewonly`在`relationship()`上，将其建立为只读属性（写入到集合的数据将在 flush()时被忽略）。但是，在极端情况下，请考虑与`Query`一起使用常规的 Python 属性，如下所示：

```py
class User(Base):
    __tablename__ = "user"
    id = mapped_column(Integer, primary_key=True)

    @property
    def addresses(self):
        return object_session(self).query(Address).with_parent(self).filter(...).all()
```

在其他情况下，可以构建描述符来利用现有的 Python 数据。有关更一般的 Python 属性的特殊讨论，请参阅使用描述符和混合部分。

另请参阅

使用描述符和混合

## 使用 viewonly 关系参数的注意事项

当应用于`relationship()`构造时，`relationship.viewonly`参数指示此`relationship()`不会参与任何 ORM 工作单元操作，此外，该属性也不会参与其表示的集合的 Python 变异。这意味着虽然 viewonly 关系可能引用可变的 Python 集合，如列表或集合，但对在映射实例上存在的该列表或集合进行更改对 ORM flush 过程没有影响。

要探索这种情景，请考虑以下映射：

```py
from __future__ import annotations

import datetime

from sqlalchemy import and_
from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str | None]

    all_tasks: Mapped[list[Task]] = relationship()

    current_week_tasks: Mapped[list[Task]] = relationship(
        primaryjoin=lambda: and_(
            User.id == Task.user_account_id,
            # this expression works on PostgreSQL but may not be supported
            # by other database engines
            Task.task_date >= func.now() - datetime.timedelta(days=7),
        ),
        viewonly=True,
    )

class Task(Base):
    __tablename__ = "task"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_account_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    description: Mapped[str | None]
    task_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    user: Mapped[User] = relationship(back_populates="current_week_tasks")
```

以下各节将说明此配置的不同方面。

### 在 Python 中，包括 backrefs 在内的变异操作不适用于 viewonly=True

上述映射针对`User.current_week_tasks`视图关系，作为`Task.user`属性的 backref 目标。目前，SQLAlchemy 的 ORM 配置过程尚未标记此项，但这是配置错误。更改`Task`上的`.user`属性不会影响`.current_week_tasks`属性：

```py
>>> u1 = User()
>>> t1 = Task(task_date=datetime.datetime.now())
>>> t1.user = u1
>>> u1.current_week_tasks
[]
```

还有另一个称为`relationship.sync_backrefs`的参数，可以在这里打开，以允许在这种情况下对`.current_week_tasks`进行突变，但是这并不被认为是`viewonly`关系的最佳实践，其不应该依赖于 Python 中的突变。

在此映射中，可以在`User.all_tasks`和`Task.user`之间配置反向引用，因为它们都不是`viewonly`并且将正常同步。

除了禁用`viewonly`关系的反向引用突变之外，Python 中对`User.all_tasks`集合的普通更改也不会反映在`User.current_week_tasks`集合中，直到更改已刷新到数据库中。

总的来说，对于一个自定义集合应该立即响应 Python 中突变的用例，`viewonly`关系通常不合适。更好的方法是使用 SQLAlchemy 的 Hybrid Attributes 功能，或者仅对于实例化的情况使用 Python 的`@property`，在这种情况下，可以实现一个用户定义的集合，该集合是以当前 Python 实例为基础生成的。要将我们的示例更改为这种工作方式，我们修复`Task.user`上的`relationship.back_populates`参数，以引用`User.all_tasks`，然后说明一个简单的`@property`，该`@property`将以`User.all_tasks`集合的立即结果的形式提供结果：

```py
class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str | None]

    all_tasks: Mapped[list[Task]] = relationship(back_populates="user")

    @property
    def current_week_tasks(self) -> list[Task]:
        past_seven_days = datetime.datetime.now() - datetime.timedelta(days=7)
        return [t for t in self.all_tasks if t.task_date >= past_seven_days]

class Task(Base):
    __tablename__ = "task"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_account_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    description: Mapped[str | None]
    task_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    user: Mapped[User] = relationship(back_populates="all_tasks")
```

每次在 Python 中计算的集合，我们都能保证始终得到正确的答案，而无需使用数据库：

```py
>>> u1 = User()
>>> t1 = Task(task_date=datetime.datetime.now())
>>> t1.user = u1
>>> u1.current_week_tasks
[<__main__.Task object at 0x7f3d699523c0>]
```

### `viewonly=True`的集合/属性在过期之前不会被重新查询

对于原始的`viewonly`属性，如果我们确实对`User.all_tasks`集合进行了更改，那么在**两**个事件发生后，`viewonly`集合才能显示这些更改的最终结果。第一个是将`User.all_tasks`的更改 flushed，以便新数据在数据库中可用，至少在本地事务范围内可用。第二个是`User.current_week_tasks`属性被 expired，并通过新的 SQL 查询重新加载到数据库。

为了支持这一要求，使用的最简单的流程是**只在主要是只读操作中使用`viewonly`关系**。比如下面，如果我们从数据库中检索到一个新的`User`，集合将是当前的：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b906b0>]
```

当我们对`u1.all_tasks`进行修改时，如果我们希望在`u1.current_week_tasks`的 viewonly 关系中看到这些更改反映出来，这些更改需要被刷新，并且`u1.current_week_tasks`属性需要过期，以便在下次访问时懒加载。这样做的最简单方法是使用`Session.commit()`，保持`Session.expire_on_commit`参数设置为其默认值`True`：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.commit()
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b90ec0>, <__main__.Task object at 0x7f8711b90a10>]
```

上述，对`Session.commit()`的调用将更改刷新到数据库的`u1.all_tasks`，然后过期所有对象，因此当我们访问`u1.current_week_tasks`时，将从数据库中新鲜地获取此属性的内容，触发了一个：term:`懒加载`。

要拦截操作而不实际提交事务，必须首先显式地过期属性。这样做的简单方法就是直接调用它。在下面的例子中，`Session.flush()`发送挂起的更改到数据库，然后使用`Session.expire()`来过期`u1.current_week_tasks`集合，以便在下次访问时重新获取：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.flush()
...     sess.expire(u1, ["current_week_tasks"])
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
```

实际上，我们可以跳过对`Session.flush()`的调用，假设`Session`保持其默认值为`True`的`Session.autoflush`，因为过期的`current_week_tasks`属性将在过期后在访问时触发自动刷新：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.expire(u1, ["current_week_tasks"])
...     print(u1.current_week_tasks)  # triggers autoflush before querying
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
```

进一步发展上述方法，我们可以在相关的`User.all_tasks`集合发生变化时，通过事件钩子来以编程方式应用过期。这是一种**高级技术**，应该首先检查像`@property`这样的更简单的架构或者坚持只读用例。在我们的简单示例中，配置如下：

```py
from sqlalchemy import event, inspect

@event.listens_for(User.all_tasks, "append")
@event.listens_for(User.all_tasks, "remove")
@event.listens_for(User.all_tasks, "bulk_replace")
def _expire_User_current_week_tasks(target, value, initiator):
    inspect(target).session.expire(target, ["current_week_tasks"])
```

使用上述钩子，突变操作被拦截，并导致`User.current_week_tasks`集合自动过期：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f66d093ccb0>, <__main__.Task object at 0x7f66d093cce0>]
```

上面使用的`AttributeEvents`事件钩子也会被后向引用突变触发，因此通过上面的钩子也会拦截对`Task.user`的更改：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     t1 = Task(task_date=datetime.datetime.now())
...     t1.user = u1
...     sess.add(t1)
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f3b0c070d10>, <__main__.Task object at 0x7f3b0c057d10>]
```

### 使用 viewonly=True 的 In-Python 突变不合适

上述映射将`User.current_week_tasks`视图关系作为`Task.user`属性的 backref 目标。这目前并未被 SQLAlchemy 的 ORM 配置过程标记，但这是一个配置错误。更改`Task`上的`.user`属性不会影响`.current_week_tasks`属性：

```py
>>> u1 = User()
>>> t1 = Task(task_date=datetime.datetime.now())
>>> t1.user = u1
>>> u1.current_week_tasks
[]
```

这里还有另一个参数叫做`relationship.sync_backrefs`，可以在这里打开，允许在这种情况下对`.current_week_tasks`进行变异，然而这并不被认为是最佳实践，对于只读关系，不应依赖于 Python 中的变异。

在这种映射中，可以在`User.all_tasks`和`Task.user`之间配置反向引用，因为这两者都不是只读的，将正常同步。

除了对于只读关系禁用反向引用变异的问题外，Python 中对`User.all_tasks`集合的普通更改也不会反映在`User.current_week_tasks`集合中，直到更改已刷新到数据库。

总的来说，对于一个自定义集合应立即响应 Python 中的变异的用例，只读关系通常不合适。更好的方法是使用 SQLAlchemy 的 Hybrid Attributes 功能，或者对于仅实例情况，使用 Python 的`@property`，其中可以实现以当前 Python 实例为基础生成的用户定义集合。要将我们的示例更改为这种方式工作，我们修复`Task.user`上的`relationship.back_populates`参数，引用`User.all_tasks`，然后演示一个简单的`@property`，将以即时`User.all_tasks`集合的结果为基础提供结果。

```py
class User(Base):
    __tablename__ = "user_account"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str | None]

    all_tasks: Mapped[list[Task]] = relationship(back_populates="user")

    @property
    def current_week_tasks(self) -> list[Task]:
        past_seven_days = datetime.datetime.now() - datetime.timedelta(days=7)
        return [t for t in self.all_tasks if t.task_date >= past_seven_days]

class Task(Base):
    __tablename__ = "task"

    id: Mapped[int] = mapped_column(primary_key=True)
    user_account_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    description: Mapped[str | None]
    task_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())

    user: Mapped[User] = relationship(back_populates="all_tasks")
```

使用每次都在 Python 中计算的即时集合，我们保证始终具有正确答案，而无需使用数据库：

```py
>>> u1 = User()
>>> t1 = Task(task_date=datetime.datetime.now())
>>> t1.user = u1
>>> u1.current_week_tasks
[<__main__.Task object at 0x7f3d699523c0>]
```

### viewonly=True 集合/属性在过期之前不会重新查询

继续使用原始的只读属性，如果实际上对持久对象上的`User.all_tasks`集合进行更改，那么只有在发生**两个**事情之后，只读集合才能显示这种更改的净结果。第一是刷新对`User.all_tasks`的更改，以便新数据在数据库中可用，至少在本地事务范围内。第二是`User.current_week_tasks`属性被过期并通过对数据库的新 SQL 查询重新加载。

为了支持这个要求，使用最简单的流程是仅在主要是只读操作中使用**仅视图关系**。比如，如果我们从数据库中获取一个新的`User`，那么集合将是当前的：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b906b0>]
```

当我们对`u1.all_tasks`进行修改时，如果我们希望在`u1.current_week_tasks`视图关系中看到这些更改的反映，这些更改需要被刷新，并且`u1.current_week_tasks`属性需要过期，这样它将在下一次访问时进行延迟加载。最简单的方法是使用`Session.commit()`，保持`Session.expire_on_commit`参数设置为其默认值`True`：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.commit()
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f8711b90ec0>, <__main__.Task object at 0x7f8711b90a10>]
```

上面，对`Session.commit()`的调用将更改刷新到数据库中，然后使所有对象过期，这样当我们访问`u1.current_week_tasks`时，一个:term:`延迟加载`会发生，从数据库中重新获取该属性的内容。

要拦截操作而不实际提交事务，需要首先显式地将属性过期。一个简单的方法是直接调用它。在下面的示例中，`Session.flush()`将挂起的更改发送到数据库，然后使用`Session.expire()`使`u1.current_week_tasks`集合过期，以便在下一次访问时重新获取：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.flush()
...     sess.expire(u1, ["current_week_tasks"])
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
```

我们实际上可以跳过对`Session.flush()`的调用，假设一个保持`Session.autoflush`值为默认值`True`的`Session`，因为过期的`current_week_tasks`属性在过期后被访问时会触发自动刷新：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     sess.expire(u1, ["current_week_tasks"])
...     print(u1.current_week_tasks)  # triggers autoflush before querying
[<__main__.Task object at 0x7fd95a4c8c50>, <__main__.Task object at 0x7fd95a4c8c80>]
```

继续上述方法到更复杂的内容，当相关的`User.all_tasks`集合发生变化时，我们可以在程序上应用过期，使用 event hooks。这是一种**高级技术**，应该首先检查简单的体系结构，比如`@property`或者坚持只读用例。在我们的简单示例中，这将配置为：

```py
from sqlalchemy import event, inspect

@event.listens_for(User.all_tasks, "append")
@event.listens_for(User.all_tasks, "remove")
@event.listens_for(User.all_tasks, "bulk_replace")
def _expire_User_current_week_tasks(target, value, initiator):
    inspect(target).session.expire(target, ["current_week_tasks"])
```

有了上述钩子，变更操作将被拦截，并导致`User.current_week_tasks`集合自动过期：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     u1.all_tasks.append(Task(task_date=datetime.datetime.now()))
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f66d093ccb0>, <__main__.Task object at 0x7f66d093cce0>]
```

上面使用的`AttributeEvents`事件钩子也会被反向引用的变化触发，因此通过上述钩子，对`Task.user`的更改也会被拦截：

```py
>>> with Session(e) as sess:
...     u1 = sess.scalar(select(User).where(User.id == 1))
...     t1 = Task(task_date=datetime.datetime.now())
...     t1.user = u1
...     sess.add(t1)
...     print(u1.current_week_tasks)
[<__main__.Task object at 0x7f3b0c070d10>, <__main__.Task object at 0x7f3b0c057d10>]
```
