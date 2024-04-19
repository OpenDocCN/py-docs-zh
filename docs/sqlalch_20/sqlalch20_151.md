# SQLAlchemy 0.4 有什么新东西？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_04.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_04.html)

关于本文档

本文档描述了 SQLAlchemy 版本 0.3（2007 年 10 月 14 日最后发布）与 SQLAlchemy 版本 0.4（2008 年 10 月 12 日最后发布）之间的变化。

文档日期：2008 年 3 月 21 日

## 先说最重要的事情

如果您使用了任何 ORM 特性，请确保从 `sqlalchemy.orm` 中导入：

```py
from sqlalchemy import *
from sqlalchemy.orm import *
```

其次，您曾经在任何地方使用 `engine=`，`connectable=`，`bind_to=`，`something.engine`，`metadata.connect()`，现在使用 `bind`：

```py
myengine = create_engine("sqlite://")

meta = MetaData(myengine)

meta2 = MetaData()
meta2.bind = myengine

session = create_session(bind=myengine)

statement = select([table], bind=myengine)
```

明白了吗？好！你现在（95%）兼容了 0.4 版本。如果你在使用 0.3.10，你可以立即做出这些更改；它们也会在那里起作用。

## 模块导入

在 0.3 版本中，“`from sqlalchemy import *`”会将所有 sqlalchemy 的子模块导入到您的命名空间中。版本 0.4 不再将子模块导入到命名空间中。这可能意味着您需要在您的代码中添加额外的导入。

在 0.3 版本中，这段代码可以工作：

```py
from sqlalchemy import *

class UTCDateTime(types.TypeDecorator):
    pass
```

在 0.4 版本中，必须这样做：

```py
from sqlalchemy import *
from sqlalchemy import types

class UTCDateTime(types.TypeDecorator):
    pass
```

## 对象关系映射

### 查询

#### 新的查询 API

查询已标准化为生成接口（旧接口仍然存在，只是已弃用）。虽然大多数生成接口在 0.3 版本中都可用，但 0.4 版本的查询具有与生成外部相匹配的内部实质，而且有更多的技巧。所有结果的缩小都通过`filter()`和`filter_by()`进行，限制/偏移要么通过数组切片，要么通过`limit()`/`offset()`进行，连接是通过`join()`和`outerjoin()`（或者更手动地，通过`select_from()`以及手动形成的条件）。

为了避免弃用警告，您必须对您的 03 代码进行一些更改

User.query.get_by( **kwargs )

```py
User.query.filter_by(**kwargs).first()
```

User.query.select_by( **kwargs )

```py
User.query.filter_by(**kwargs).all()
```

User.query.select()

```py
User.query.filter(xxx).all()
```

#### 新的基于属性的表达式构造

在 ORM 中最明显的区别是，您现在可以直接使用基于类的属性构造您的查询条件。在使用映射类时，不再需要“`.c.`”前缀：

```py
session.query(User).filter(and_(User.name == "fred", User.id > 17))
```

虽然简单的基于列的比较不是什么大问题，但类属性有一些新的“更高级”的构造可用，包括以前只在 `filter_by()` 中可用的内容：

```py
# comparison of scalar relations to an instance
filter(Address.user == user)

# return all users who contain a particular address
filter(User.addresses.contains(address))

# return all users who *dont* contain the address
filter(~User.address.contains(address))

# return all users who contain a particular address with
# the email_address like '%foo%'
filter(User.addresses.any(Address.email_address.like("%foo%")))

# same, email address equals 'foo@bar.com'.  can fall back to keyword
# args for simple comparisons
filter(User.addresses.any(email_address="foo@bar.com"))

# return all Addresses whose user attribute has the username 'ed'
filter(Address.user.has(name="ed"))

# return all Addresses whose user attribute has the username 'ed'
# and an id > 5 (mixing clauses with kwargs)
filter(Address.user.has(User.id > 5, name="ed"))
```

映射类上的`.c`属性中仍然可用 `Column` 集合。请注意，基于属性的表达式仅在映射类的映射属性中可用。`.c`仍然用于访问常规表中的列和从 SQL 表达式产生的可选择对象中的列。

#### 自动连接别名

我们已经有了 join() 和 outerjoin() 一段时间了：

```py
session.query(Order).join("items")
```

现在你可以给它们起别名了：

```py
session.query(Order).join("items", aliased=True).filter(Item.name="item 1").join(
    "items", aliased=True
).filter(Item.name == "item 3")
```

以上代码将创建两个别名从 orders->items 的连接。每个连接后的`filter()`调用将其表条件调整为别名的条件。要获取`Item`对象，请使用`add_entity()`，并使用`id`目标每个连接：

```py
session.query(Order).join("items", id="j1", aliased=True).filter(
    Item.name == "item 1"
).join("items", aliased=True, id="j2").filter(Item.name == "item 3").add_entity(
    Item, id="j1"
).add_entity(
    Item, id="j2"
)
```

返回的元组形式为：`（Order，Item，Item）`。

#### 自引用查询

因此，query.join() 现在可以创建别名。这给了我们什么？自引用查询！可以在没有任何 `Alias` 对象的情况下进行连接：

```py
# standard self-referential TreeNode mapper with backref
mapper(
    TreeNode,
    tree_nodes,
    properties={
        "children": relation(
            TreeNode, backref=backref("parent", remote_side=tree_nodes.id)
        )
    },
)

# query for node with child containing "bar" two levels deep
session.query(TreeNode).join(["children", "children"], aliased=True).filter_by(
    name="bar"
)
```

要为沿途每个表添加条件在别名连接中，您可以使用 `from_joinpoint` 继续针对相同别名行进行连接：

```py
# search for the treenode along the path "n1/n12/n122"

# first find a Node with name="n122"
q = sess.query(Node).filter_by(name="n122")

# then join to parent with "n12"
q = q.join("parent", aliased=True).filter_by(name="n12")

# join again to the next parent with 'n1'.  use 'from_joinpoint'
# so we join from the previous point, instead of joining off the
# root table
q = q.join("parent", aliased=True, from_joinpoint=True).filter_by(name="n1")

node = q.first()
```

#### `query.populate_existing()`

`query.load()` 的贪婪版本（或 `session.refresh()`）。从查询加载的每个实例，包括所有贪婪加载的项目，如果已经存在于会话中，则立即刷新：

```py
session.query(Blah).populate_existing().all()
```

### 关系

#### 嵌入在更新/插入中的 SQL 子句

用于内联执行 SQL 子句，直接嵌入在 `flush()` 期间的 UPDATE 或 INSERT 中：

```py
myobject.foo = mytable.c.value + 1

user.pwhash = func.md5(password)

order.hash = text("select hash from hashing_table")
```

在操作之后，列属性设置为延迟加载器，以便在下次访问时发出加载新值的 SQL。

#### 自引用和循环贪婪加载

由于我们的别名技术已经改进，`relation()` 可以沿着相同的表进行任意次数的连接；您告诉它您想要深入多深。让我们更清楚地展示自引用的 `TreeNode`：

```py
nodes = Table(
    "nodes",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("parent_id", Integer, ForeignKey("nodes.id")),
    Column("name", String(30)),
)

class TreeNode(object):
    pass

mapper(
    TreeNode,
    nodes,
    properties={"children": relation(TreeNode, lazy=False, join_depth=3)},
)
```

那么当我们说：

```py
create_session().query(TreeNode).all()
```

? 沿着别名进行连接，从父级开始深入三级：

```py
SELECT
nodes_3.id  AS  nodes_3_id,  nodes_3.parent_id  AS  nodes_3_parent_id,  nodes_3.name  AS  nodes_3_name,
nodes_2.id  AS  nodes_2_id,  nodes_2.parent_id  AS  nodes_2_parent_id,  nodes_2.name  AS  nodes_2_name,
nodes_1.id  AS  nodes_1_id,  nodes_1.parent_id  AS  nodes_1_parent_id,  nodes_1.name  AS  nodes_1_name,
nodes.id  AS  nodes_id,  nodes.parent_id  AS  nodes_parent_id,  nodes.name  AS  nodes_name
FROM  nodes  LEFT  OUTER  JOIN  nodes  AS  nodes_1  ON  nodes.id  =  nodes_1.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_2  ON  nodes_1.id  =  nodes_2.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_3  ON  nodes_2.id  =  nodes_3.parent_id
ORDER  BY  nodes.oid,  nodes_1.oid,  nodes_2.oid,  nodes_3.oid
```

注意这些漂亮干净的别名。连接不在乎是针对同一个立即表还是一些其他对象，然后再循环回开始。当指定了 `join_depth` 时，任何类型的贪婪加载链都可以在自身上循环。当不存在时，贪婪加载在遇到循环时会自动停止。

#### 复合类型

这是来自 Hibernate 阵营的一个例子。复合类型允许您定义一个由多个列（或一个列，如果您愿意的话）组成的自定义数据类型。让我们定义一个新类型，`Point`。存储一个 x/y 坐标：

```py
class Point(object):
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __composite_values__(self):
        return self.x, self.y

    def __eq__(self, other):
        return other.x == self.x and other.y == self.y

    def __ne__(self, other):
        return not self.__eq__(other)
```

`Point` 对象的定义方式是特定于自定义类型的；构造函数接受一个参数列表，并且 `__composite_values__()` 方法生成这些参数的序列。顺序将与我们的映射器匹配，我们马上就会看到。

让我们创建一个存储每行两个点的顶点表：

```py
vertices = Table(
    "vertices",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x1", Integer),
    Column("y1", Integer),
    Column("x2", Integer),
    Column("y2", Integer),
)
```

然后，映射它！我们将创建一个存储两个 `Point` 对象的 `Vertex` 对象：

```py
class Vertex(object):
    def __init__(self, start, end):
        self.start = start
        self.end = end

mapper(
    Vertex,
    vertices,
    properties={
        "start": composite(Point, vertices.c.x1, vertices.c.y1),
        "end": composite(Point, vertices.c.x2, vertices.c.y2),
    },
)
```

一旦设置了您的复合类型，它就可以像任何其他类型一样使用：

```py
v = Vertex(Point(3, 4), Point(26, 15))
session.save(v)
session.flush()

# works in queries too
q = session.query(Vertex).filter(Vertex.start == Point(3, 4))
```

如果您想定义映射属性在表达式中生成 SQL 子句的方式，请创建自己的 `sqlalchemy.orm.PropComparator` 子类，定义任何常见操作符（如 `__eq__()`，`__le__()` 等），并将其发送到 `composite()`。复合类型也可以作为主键，并且可以在 `query.get()` 中使用：

```py
# a Document class which uses a composite Version
# object as primary key
document = query.get(Version(1, "a"))
```

#### `dynamic_loader()` 关系

一个返回所有读操作的实时 `Query` 对象的 `relation()`。写操作仅限于 `append()` 和 `remove()`，对集合的更改在会话刷新之前不可见。此功能在“自动刷新”会话中特别方便，该会话会在每次查询之前刷新。

```py
mapper(
    Foo,
    foo_table,
    properties={
        "bars": dynamic_loader(
            Bar,
            backref="foo",
            # <other relation() opts>
        )
    },
)

session = create_session(autoflush=True)
foo = session.query(Foo).first()

foo.bars.append(Bar(name="lala"))

for bar in foo.bars.filter(Bar.name == "lala"):
    print(bar)

session.commit()
```

#### 新选项：`undefer_group()`，`eagerload_all()`

一些方便的查询选项。`undefer_group()`将整个“延迟”列组标记为未延迟：

```py
mapper(
    Class,
    table,
    properties={
        "foo": deferred(table.c.foo, group="group1"),
        "bar": deferred(table.c.bar, group="group1"),
        "bat": deferred(table.c.bat, group="group1"),
    },
)

session.query(Class).options(undefer_group("group1")).filter(...).all()
```

`eagerload_all()`设置一系列属性为一次性急切加载：

```py
mapper(Foo, foo_table, properties={"bar": relation(Bar)})
mapper(Bar, bar_table, properties={"bat": relation(Bat)})
mapper(Bat, bat_table)

# eager load bar and bat
session.query(Foo).options(eagerload_all("bar.bat")).filter(...).all()
```

#### 新的集合 API

集合不再由{{{InstrumentedList}}}代理代理，对成员、方法和属性的访问是直接的。装饰器现在拦截进入和离开集合的对象，并且现在可以轻松编写一个自定义集合类来管理自己的成员。灵活的装饰器还取代了 0.3 中自定义集合的命名方法接口，允许任何类轻松适应用作集合容器。

基于字典的集合现在更容易使用，完全类似于`dict`。不再需要更改`__iter__`以用于`dict`，新的内置`dict`类型涵盖了许多需求：

```py
# use a dictionary relation keyed by a column
relation(Item, collection_class=column_mapped_collection(items.c.keyword))
# or named attribute
relation(Item, collection_class=attribute_mapped_collection("keyword"))
# or any function you like
relation(Item, collection_class=mapped_collection(lambda entity: entity.a + entity.b))
```

现有的 0.3 类似`dict`和自由形式对象派生的集合类需要更新以适应新的 API。在大多数情况下，这只是在类定义中添加一些装饰器的问题。

#### 从外部表/子查询映射的关系

这个功能在 0.3 中悄悄出现，但在 0.4 中得到改进，这要归功于更好地能够将针对表的子查询转换为该表的别名的子查询；这对于急切加载、查询中的别名连接等非常重要。当您只需要添加一些额外列或子查询时，它减少了对选择语句创建映射器的需求：

```py
mapper(
    User,
    users,
    properties={
        "fullname": column_property(
            (users.c.firstname + users.c.lastname).label("fullname")
        ),
        "numposts": column_property(
            select([func.count(1)], users.c.id == posts.c.user_id)
            .correlate(users)
            .label("posts")
        ),
    },
)
```

典型的查询如下：

```py
SELECT  (SELECT  count(1)  FROM  posts  WHERE  users.id  =  posts.user_id)  AS  count,
users.firstname  ||  users.lastname  AS  fullname,
users.id  AS  users_id,  users.firstname  AS  users_firstname,  users.lastname  AS  users_lastname
FROM  users  ORDER  BY  users.oid
```

### 水平扩展（分片）API

[browser:/sqlalchemy/trunk/examples/sharding/attribute_shard .py]

### 会话

#### 新的会话创建范式；SessionContext，assignmapper 已弃用

是的，整个事情都被两个配置函数替换了。同时使用两者将产生自 0.1 以来最接近的感觉（即，输入最少）。

在定义`engine`（或任何地方）的地方配置自己的`Session`类：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("myengine://")
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

# use the new Session() freely
sess = Session()
sess.save(someobject)
sess.flush()
```

如果您需要后配置您的会话，比如使用引擎，稍后使用`configure()`添加：

```py
Session.configure(bind=create_engine(...))
```

所有`SessionContext`的行为以及`assignmapper`的`query`和`__init__`方法都移动到新的`scoped_session()`函数中，该函数与`sessionmaker`和`create_session()`兼容：

```py
from sqlalchemy.orm import scoped_session, sessionmaker

Session = scoped_session(sessionmaker(autoflush=True, transactional=True))
Session.configure(bind=engine)

u = User(name="wendy")

sess = Session()
sess.save(u)
sess.commit()

# Session constructor is thread-locally scoped.  Everyone gets the same
# Session in the thread when scope="thread".
sess2 = Session()
assert sess is sess2
```

使用线程本地`Session`时，返回的类已实现了所有`Session`的接口作为类方法，并且使用`mapper`类方法可以使用“assignmapper”的功能。就像旧的`objectstore`时代一样...。

```py
# "assignmapper"-like functionality available via ScopedSession.mapper
Session.mapper(User, users_table)

u = User(name="wendy")

Session.commit()
```

#### 会话再次默认使用弱引用

默认情况下，在 Session 上将 weak_identity_map 标志设置为 `True`。外部解除引用并超出范围的实例将自动从会话中移除。但是，具有“脏”更改的项目将保持强引用，直到这些更改被刷新，此时对象将恢复为弱引用（这适用于像可选属性这样的‘可变’类型）。将 weak_identity_map 设置为 `False` 将为那些像缓存一样使用会话的人恢复旧的强引用行为。

#### 自动事务会话

正如您可能已经注意到的，我们在 `Session` 上调用了 `commit()`。标志 `transactional=True` 意味着 `Session` 总是处于事务中，`commit()` 永久保存。

#### 自动刷新会话

此外，`autoflush=True` 意味着 `Session` 在每次 `query` 之前都会执行 `flush()`，以及在调用 `flush()` 或 `commit()` 时。所以现在这将起作用：

```py
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

u = User(name="wendy")

sess = Session()
sess.save(u)

# wendy is flushed, comes right back from a query
wendy = sess.query(User).filter_by(name="wendy").one()
```

#### 事务方法转移到了会话中

`commit()` 和 `rollback()`，以及 `begin()` 现在直接在 `Session` 上。不再需要为任何事情使用 `SessionTransaction`（它仍然在后台运行）。

```py
Session = sessionmaker(autoflush=True, transactional=False)

sess = Session()
sess.begin()

# use the session

sess.commit()  # commit transaction
```

与封闭引擎级（即非 ORM）事务共享 `Session` 很容易：

```py
Session = sessionmaker(autoflush=True, transactional=False)

conn = engine.connect()
trans = conn.begin()
sess = Session(bind=conn)

# ... session is transactional

# commit the outermost transaction
trans.commit()
```

#### 带有 SAVEPOINT 的嵌套会话事务

在引擎和 ORM 级别可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

#### 两阶段提交会话

在引擎和 ORM 级别可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

### 继承

#### 无连接或联合的多态继承

继承的新文档：[`www.sqlalchemy.org/docs/04`](https://www.sqlalchemy.org/docs/04) /mappers.html#advdatamapping_mapper_inheritance_joined

#### 使用 `get()` 时更好的多态行为

加入表继承层次结构中的所有类都使用基类获得 `_instance_key`，即 `(BaseClass, (1, ), None)`。这样，当您针对基类调用 `get()` 时，它可以在当前标识映射中定位子类实例，而无需查询数据库。

### 类型

#### `sqlalchemy.types.TypeDecorator` 的自定义子类

有一个[新 API](https://www.sqlalchemy.org/docs/04/types.html#types_custom)用于子类化 TypeDecorator。在某些情况下，使用 0.3 API 会导致编译错误。

## SQL 表达式

### 全新的、确定性的标签/别名生成

所有“匿名”标签和别名现在都使用简单的 <name>_<number> 格式。SQL 更容易阅读，并且与计划优化器缓存兼容。只需查看一些教程中的示例：[`www.sqlalchemy.org/docs/04/ormtutorial.html`](https://www.sqlalchemy.org/docs/04/ormtutorial.html) [`www.sqlalchemy.org/docs/04/sqlexpression.html`](https://www.sqlalchemy.org/docs/04/sqlexpression.html)

### 生成式 select() 构造

这绝对是使用`select()`的方法。请参阅 htt p://www.sqlalchemy.org/docs/04/sqlexpression.html#sql_transf orm 。

### 新的运算符系统

SQL 运算符和几乎每个 SQL 关键字现在都被抽象为编译器层。它们现在表现智能，并且具有类型/后端感知性，参见：[`www.sqlalchemy.org/docs/04/sqlexpression.html#sql_operators`](https://www.sqlalchemy.org/docs/04/sqlexpression.html#sql_operators)

### 所有`type`关键字参数重命名为`type_`

就像它说的那样：

```py
b = bindparam("foo", type_=String)
```

### `in_`函数更改为接受序列或可选择的

`in_`函数现在接受一个值序列或可选择的可选参数。仍然可以使用以前将值作为位置参数传递的 API，但现在已被弃用。这意味着

```py
my_table.select(my_table.c.id.in_(1, 2, 3))
my_table.select(my_table.c.id.in_(*listOfIds))
```

应更改为

```py
my_table.select(my_table.c.id.in_([1, 2, 3]))
my_table.select(my_table.c.id.in_(listOfIds))
```

## 模式和反射

### `MetaData`、`BoundMetaData`、`DynamicMetaData`…

在 0.3.x 系列中，`BoundMetaData`和`DynamicMetaData`已被弃用，取而代之的是`MetaData`和`ThreadLocalMetaData`。旧名称已在 0.4 版本中移除。更新很简单：

```py
+-------------------------------------+-------------------------+
|If You Had                           | Now Use                 |
+=====================================+=========================+
| ``MetaData``                        | ``MetaData``            |
+-------------------------------------+-------------------------+
| ``BoundMetaData``                   | ``MetaData``            |
+-------------------------------------+-------------------------+
| ``DynamicMetaData`` (with one       | ``MetaData``            |
| engine or threadlocal=False)        |                         |
+-------------------------------------+-------------------------+
| ``DynamicMetaData``                 | ``ThreadLocalMetaData`` |
| (with different engines per thread) |                         |
+-------------------------------------+-------------------------+
```

很少使用的`name`参数已从`MetaData`类型中删除。`ThreadLocalMetaData`构造函数现在不再接受参数。这两种类型现在都可以绑定到一个`Engine`或一个单独的`Connection`。

### 一步多表反射

现在您可以加载表定义，并在一个步骤中自动创建整个数据库或模式的`Table`对象：

```py
>>> metadata = MetaData(myengine, reflect=True)
>>> metadata.tables.keys()
['table_a', 'table_b', 'table_c', '...']
```

`MetaData`还增加了一个`.reflect()`方法，可以更精细地控制加载过程，包括指定要加载的可用表的子集。

## SQL 执行

### `engine`、`connectable`和`bind_to`现在都是`bind`

### `Transactions`、`NestedTransactions`和`TwoPhaseTransactions`

### 连接池事件

连接池现在在创建新的 DB-API 连接、检出和检入池时触发事件。您可以使用这些事件在新连接上执行会话范围的 SQL 设置语句，例如。

### 修复了 Oracle Engine

在 0.3.11 版本中，Oracle Engine 在处理主键时存在错误。这些错误可能导致在使用 Oracle Engine 时，其他引擎（如 sqlite）正常工作的程序失败。在 0.4 版本中，Oracle Engine 已经重新设计，修复了这些主键问题。

### Oracle 的输出参数

```py
result = engine.execute(
    text(
        "begin foo(:x, :y, :z); end;",
        bindparams=[
            bindparam("x", Numeric),
            outparam("y", Numeric),
            outparam("z", Numeric),
        ],
    ),
    x=5,
)
assert result.out_parameters == {"y": 10, "z": 75}
```

### 连接绑定的`MetaData`、`Sessions`

`MetaData`和`Session`可以显式绑定到连接：

```py
conn = engine.connect()
sess = create_session(bind=conn)
```

### 更快、更可靠的`ResultProxy`对象

## 首要事项

如果您正在使用任何 ORM 功能，请确保从`sqlalchemy.orm`导入：

```py
from sqlalchemy import *
from sqlalchemy.orm import *
```

其次，无论您以前使用`engine=`、`connectable=`、`bind_to=`、`something.engine`、`metadata.connect()`，现在都使用`bind`：

```py
myengine = create_engine("sqlite://")

meta = MetaData(myengine)

meta2 = MetaData()
meta2.bind = myengine

session = create_session(bind=myengine)

statement = select([table], bind=myengine)
```

明白了吗？很好！您现在（95%）兼容 0.4 版本。如果您正在使用 0.3.10 版本，您可以立即进行这些更改；它们也适用于那里。

## 模块导入

在 0.3 版本中，“`from sqlalchemy import *`” 将所有 sqlalchemy 的子模块导入到您的命名空间中。0.4 版本不再将子模块导入到命名空间中。这可能意味着您需要在代码中添加额外的导入。

在 0.3 版本中，这段代码有效：

```py
from sqlalchemy import *

class UTCDateTime(types.TypeDecorator):
    pass
```

在 0.4 版本中，必须执行：

```py
from sqlalchemy import *
from sqlalchemy import types

class UTCDateTime(types.TypeDecorator):
    pass
```

## 对象关系映射

### 查询

#### 新的查询 API

查询标准化为生成式接口（旧接口仍然存在，只是已弃用）。虽然大部分生成式接口在 0.3 版本中可用，但 0.4 版本的 Query 具有与生成式外部匹配的内部实现，并且有更多技巧。所有结果缩小都通过 `filter()` 和 `filter_by()` 进行，限制/偏移要么通过数组切片要么通过 `limit()`/`offset()` 进行，连接通过 `join()` 和 `outerjoin()` 进行（或者更手动地，通过 `select_from()` 以及手动形成的条件）。

为了避免弃用警告，您必须对您的 03 代码进行一些更改

User.query.get_by( **kwargs )

```py
User.query.filter_by(**kwargs).first()
```

User.query.select_by( **kwargs )

```py
User.query.filter_by(**kwargs).all()
```

User.query.select()

```py
User.query.filter(xxx).all()
```

#### 新的基于属性的表达式构造

在 ORM 中最明显的区别是，现在您可以直接使用基于类的属性构建查询条件。在处理映射类时不再需要“`.c.`”前缀：

```py
session.query(User).filter(and_(User.name == "fred", User.id > 17))
```

虽然简单的基于列的比较不是什么大问题，但类属性具有一些新的“更高级”构造可用，包括以前仅在 `filter_by()` 中可用的内容：

```py
# comparison of scalar relations to an instance
filter(Address.user == user)

# return all users who contain a particular address
filter(User.addresses.contains(address))

# return all users who *dont* contain the address
filter(~User.address.contains(address))

# return all users who contain a particular address with
# the email_address like '%foo%'
filter(User.addresses.any(Address.email_address.like("%foo%")))

# same, email address equals 'foo@bar.com'.  can fall back to keyword
# args for simple comparisons
filter(User.addresses.any(email_address="foo@bar.com"))

# return all Addresses whose user attribute has the username 'ed'
filter(Address.user.has(name="ed"))

# return all Addresses whose user attribute has the username 'ed'
# and an id > 5 (mixing clauses with kwargs)
filter(Address.user.has(User.id > 5, name="ed"))
```

`Column` 集合仍然可以在 `.c` 属性中的映射类中使用。请注意，基于属性的表达式仅适用于映射类的映射属性。`.c` 仍然用于访问常规表中的列以及从 SQL 表达式生成的可选择对象。

#### 自动连接别名

我们已经有了 join() 和 outerjoin() 一段时间了：

```py
session.query(Order).join("items")
```

现在您可以为它们创建别名：

```py
session.query(Order).join("items", aliased=True).filter(Item.name="item 1").join(
    "items", aliased=True
).filter(Item.name == "item 3")
```

以上将使用别名从 orders->items 创建两个连接。每个连接后续的 `filter()` 调用将调整其表条件以符合别名。要访问 `Item` 对象，请使用 `add_entity()` 并将每个连接的 `id` 作为目标：

```py
session.query(Order).join("items", id="j1", aliased=True).filter(
    Item.name == "item 1"
).join("items", aliased=True, id="j2").filter(Item.name == "item 3").add_entity(
    Item, id="j1"
).add_entity(
    Item, id="j2"
)
```

返回元组形式：`(Order, Item, Item)`。

#### 自引用查询

因此 query.join() 现在可以创建别名。这给我们带来了什么？自引用查询！可以在没有任何 `Alias` 对象的情况下进行连接：

```py
# standard self-referential TreeNode mapper with backref
mapper(
    TreeNode,
    tree_nodes,
    properties={
        "children": relation(
            TreeNode, backref=backref("parent", remote_side=tree_nodes.id)
        )
    },
)

# query for node with child containing "bar" two levels deep
session.query(TreeNode).join(["children", "children"], aliased=True).filter_by(
    name="bar"
)
```

要为每个表添加条件，可以使用 `from_joinpoint` 在别名连接中保持针对相同别名行的连接：

```py
# search for the treenode along the path "n1/n12/n122"

# first find a Node with name="n122"
q = sess.query(Node).filter_by(name="n122")

# then join to parent with "n12"
q = q.join("parent", aliased=True).filter_by(name="n12")

# join again to the next parent with 'n1'.  use 'from_joinpoint'
# so we join from the previous point, instead of joining off the
# root table
q = q.join("parent", aliased=True, from_joinpoint=True).filter_by(name="n1")

node = q.first()
```

#### `query.populate_existing()`

`query.load()` 的贪婪版本（或 `session.refresh()`）。从查询加载的每个实例，包括所有贪婪加载的项目，如果已经存在于会话中，则立即刷新：

```py
session.query(Blah).populate_existing().all()
```

### 关系

#### 嵌入在更新/插入中的 SQL 子句

对于内联执行 SQL 子句，嵌入在 `flush()` 中的 UPDATE 或 INSERT 中：

```py
myobject.foo = mytable.c.value + 1

user.pwhash = func.md5(password)

order.hash = text("select hash from hashing_table")
```

在操作后，列属性设置为延迟加载器，因此当您下次访问时，它会发出 SQL 来加载新值。

#### 自引用和循环贪婪加载

由于我们的别名技术已经改进，`relation()`可以沿着相同的表*任意次数*连接；你告诉它你想要多深。让我们更清楚地展示自引用的`TreeNode`：

```py
nodes = Table(
    "nodes",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("parent_id", Integer, ForeignKey("nodes.id")),
    Column("name", String(30)),
)

class TreeNode(object):
    pass

mapper(
    TreeNode,
    nodes,
    properties={"children": relation(TreeNode, lazy=False, join_depth=3)},
)
```

那么当我们说：

```py
create_session().query(TreeNode).all()
```

? 通过别名连接，从父级开始深入三级：

```py
SELECT
nodes_3.id  AS  nodes_3_id,  nodes_3.parent_id  AS  nodes_3_parent_id,  nodes_3.name  AS  nodes_3_name,
nodes_2.id  AS  nodes_2_id,  nodes_2.parent_id  AS  nodes_2_parent_id,  nodes_2.name  AS  nodes_2_name,
nodes_1.id  AS  nodes_1_id,  nodes_1.parent_id  AS  nodes_1_parent_id,  nodes_1.name  AS  nodes_1_name,
nodes.id  AS  nodes_id,  nodes.parent_id  AS  nodes_parent_id,  nodes.name  AS  nodes_name
FROM  nodes  LEFT  OUTER  JOIN  nodes  AS  nodes_1  ON  nodes.id  =  nodes_1.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_2  ON  nodes_1.id  =  nodes_2.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_3  ON  nodes_2.id  =  nodes_3.parent_id
ORDER  BY  nodes.oid,  nodes_1.oid,  nodes_2.oid,  nodes_3.oid
```

注意漂亮干净的别名名称。连接不在乎是否针对同一个直接表或一些其他对象，然后循环回开始。当指定`join_depth`时，任何类型的链式急切加载可以循环回自身。当不存在时，急切加载在遇到循环时会自动停止。

#### 复合类型

这是来自 Hibernate 阵营的一个。复合类型让你定义一个由多个列（或一个列，如果你愿意）组成的自定义数据类型。让我们定义一个新类型，`Point`。存储一个 x/y 坐标：

```py
class Point(object):
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __composite_values__(self):
        return self.x, self.y

    def __eq__(self, other):
        return other.x == self.x and other.y == self.y

    def __ne__(self, other):
        return not self.__eq__(other)
```

`Point`对象的定义方式是特定于自定义类型的；构造函数接受一个参数列表，`__composite_values__()`方法生成这些参数的序列。顺序将与我们的映射器匹配，我们马上就会看到。

让我们创建一个存储每行两个点的顶点表：

```py
vertices = Table(
    "vertices",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x1", Integer),
    Column("y1", Integer),
    Column("x2", Integer),
    Column("y2", Integer),
)
```

然后，映射它！我们将创建一个存储两个`Point`对象的`Vertex`对象：

```py
class Vertex(object):
    def __init__(self, start, end):
        self.start = start
        self.end = end

mapper(
    Vertex,
    vertices,
    properties={
        "start": composite(Point, vertices.c.x1, vertices.c.y1),
        "end": composite(Point, vertices.c.x2, vertices.c.y2),
    },
)
```

一旦设置了复合类型，它就可以像任何其他类型一样使用：

```py
v = Vertex(Point(3, 4), Point(26, 15))
session.save(v)
session.flush()

# works in queries too
q = session.query(Vertex).filter(Vertex.start == Point(3, 4))
```

如果你想定义映射属性在表达式中生成 SQL 子句的方式，创建自己的`sqlalchemy.orm.PropComparator`子类，定义任何常见操作符（如`__eq__()`，`__le__()`等），并将其发送到`composite()`。复合类型也可以作为主键，并且可用于`query.get()`：

```py
# a Document class which uses a composite Version
# object as primary key
document = query.get(Version(1, "a"))
```

#### `dynamic_loader()`关系

一个返回所有读操作的实时`Query`对象的`relation()`。写操作仅限于`append()`和`remove()`，对集合的更改在会话刷新之前不可见。这个特性在“自动刷新”会话中特别方便，它会在每次查询之前刷新。

```py
mapper(
    Foo,
    foo_table,
    properties={
        "bars": dynamic_loader(
            Bar,
            backref="foo",
            # <other relation() opts>
        )
    },
)

session = create_session(autoflush=True)
foo = session.query(Foo).first()

foo.bars.append(Bar(name="lala"))

for bar in foo.bars.filter(Bar.name == "lala"):
    print(bar)

session.commit()
```

#### 新选项：`undefer_group()`，`eagerload_all()`

一些方便的查询选项。`undefer_group()`将整个“延迟”列组标记为未延迟：

```py
mapper(
    Class,
    table,
    properties={
        "foo": deferred(table.c.foo, group="group1"),
        "bar": deferred(table.c.bar, group="group1"),
        "bat": deferred(table.c.bat, group="group1"),
    },
)

session.query(Class).options(undefer_group("group1")).filter(...).all()
```

`eagerload_all()`设置一系列属性在一次遍历中急切加载：

```py
mapper(Foo, foo_table, properties={"bar": relation(Bar)})
mapper(Bar, bar_table, properties={"bat": relation(Bat)})
mapper(Bat, bat_table)

# eager load bar and bat
session.query(Foo).options(eagerload_all("bar.bat")).filter(...).all()
```

#### 新的集合 API

集合不再由{{{InstrumentedList}}}代理进行代理，对成员、方法和属性的访问是直接的。装饰器现在拦截进入和离开集合的对象，现在可以轻松编写一个自定义集合类来管理自己的成员。灵活的装饰器还取代了 0.3 版本中自定义集合的命名方法接口，允许任何类轻松适应用作集合容器。

基于字典的集合现在更容易使用，完全类似于`dict`。不再需要更改`dict`的`__iter__`，新的内置`dict`类型涵盖了许多需求：

```py
# use a dictionary relation keyed by a column
relation(Item, collection_class=column_mapped_collection(items.c.keyword))
# or named attribute
relation(Item, collection_class=attribute_mapped_collection("keyword"))
# or any function you like
relation(Item, collection_class=mapped_collection(lambda entity: entity.a + entity.b))
```

现有的 0.3 版本的类似 `dict` 和自由形式对象派生的集合类需要更新到新的 API。在大多数情况下，这只是在类定义中添加几个装饰器的问题。

#### 从外部表/子查询映射关系

这个功能在 0.3 版本中悄然出现，但在 0.4 版本中得到改进，这要归功于更好地将针对表的子查询转换为针对该表的别名的能力；这对于急加载、查询中的别名连接等非常重要。这减少了在只需要添加一些额外列或子查询时对选择语句创建映射器的需求：

```py
mapper(
    User,
    users,
    properties={
        "fullname": column_property(
            (users.c.firstname + users.c.lastname).label("fullname")
        ),
        "numposts": column_property(
            select([func.count(1)], users.c.id == posts.c.user_id)
            .correlate(users)
            .label("posts")
        ),
    },
)
```

典型的查询如下：

```py
SELECT  (SELECT  count(1)  FROM  posts  WHERE  users.id  =  posts.user_id)  AS  count,
users.firstname  ||  users.lastname  AS  fullname,
users.id  AS  users_id,  users.firstname  AS  users_firstname,  users.lastname  AS  users_lastname
FROM  users  ORDER  BY  users.oid
```

### 水平扩展（分片）API

[browser:/sqlalchemy/trunk/examples/sharding/attribute_shard .py]

### 会话

#### 新的会话创建范式；SessionContext，assignmapper 已弃用

是的，整个事情都被两个配置函数替换了。同时使用两者将产生自 0.1 版本以来最接近 0.1 版本的感觉（即，输入最少）。

在定义引擎（或任何地方）的地方配置您自己的 `Session` 类��

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("myengine://")
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

# use the new Session() freely
sess = Session()
sess.save(someobject)
sess.flush()
```

如果您需要在之后使用 `configure()` 来后置配置您的会话，比如添加引擎：

```py
Session.configure(bind=create_engine(...))
```

`SessionContext` 的所有行为以及 `assignmapper` 的 `query` 和 `__init__` 方法都移动到了新的 `scoped_session()` 函数中，该函数与 `sessionmaker` 和 `create_session()` 兼容：

```py
from sqlalchemy.orm import scoped_session, sessionmaker

Session = scoped_session(sessionmaker(autoflush=True, transactional=True))
Session.configure(bind=engine)

u = User(name="wendy")

sess = Session()
sess.save(u)
sess.commit()

# Session constructor is thread-locally scoped.  Everyone gets the same
# Session in the thread when scope="thread".
sess2 = Session()
assert sess is sess2
```

当使用线程本地的 `Session` 时，返回的类已经实现了所有 `Session` 的接口作为类方法，并且可以使用 `mapper` 类方法来使用 “assignmapper” 的功能。就像旧的 `objectstore` 时代一样……。

```py
# "assignmapper"-like functionality available via ScopedSession.mapper
Session.mapper(User, users_table)

u = User(name="wendy")

Session.commit()
```

#### 会话再次默认为弱引用

weak_identity_map 标志现在默认设置为 `True` 在 Session 上。外部解除引用并且超出范围的实例会自动从会话中移除。但是，具有“脏”更改的项目将保持强引用，直到这些更改被刷新，此时对象将恢复为弱引用（这适用于像可选属性这样的“可变”类型）。将 weak_identity_map 设置为 `False` 将为那些像缓存一样使用会话的人恢复旧的强引用行为。

#### 自动事务会话

正如您在上面所注意到的，我们在 `Session` 上调用 `commit()`。标志 `transactional=True` 意味着 `Session` 总是处于事务中，`commit()` 永久持久化。

#### 自动刷新会话

此外，`autoflush=True` 意味着 `Session` 将在每次 `query` 之前刷新，以及在调用 `flush()` 或 `commit()` 时。所以现在这将起作用：

```py
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

u = User(name="wendy")

sess = Session()
sess.save(u)

# wendy is flushed, comes right back from a query
wendy = sess.query(User).filter_by(name="wendy").one()
```

#### 事务方法移动到会话上

`commit()` 和 `rollback()`，以及 `begin()` 现在直接在 `Session` 上。不再需要为任何事情使用 `SessionTransaction`（它仍然在后台运行）。

```py
Session = sessionmaker(autoflush=True, transactional=False)

sess = Session()
sess.begin()

# use the session

sess.commit()  # commit transaction
```

与封闭的引擎级（即非 ORM）事务共享 `Session` 很容易：

```py
Session = sessionmaker(autoflush=True, transactional=False)

conn = engine.connect()
trans = conn.begin()
sess = Session(bind=conn)

# ... session is transactional

# commit the outermost transaction
trans.commit()
```

#### 使用 SAVEPOINT 的嵌套会话事务

在引擎和 ORM 级别可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

#### 两阶段提交会话

在引擎和 ORM 级别可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

### 继承

#### 无联接或联合的多态继承

继承的新文档：[`www.sqlalchemy.org/docs/04`](https://www.sqlalchemy.org/docs/04) /mappers.html#advdatamapping_mapper_inheritance_joined

#### 使用`get()`实现更好的多态行为

在联接表继承层次结构中，所有类都使用基类获得`_instance_key`，即`(BaseClass, (1, ), None)`。这样，当您针对基类调用`get()`时，它可以在当前标识映射中定位子类实例，而无需查询数据库。

### 类型

#### 自定义`sqlalchemy.types.TypeDecorator`的子类

有一个[新的 API](https://www.sqlalchemy.org/docs/04/types.html#types_custom)用于子类化 TypeDecorator。在某些情况下，使用 0.3 API 会导致编译错误。

### 查询

#### 新的查询 API

查询标准化为生成式接口（旧接口仍然存在，只是已弃用）。虽然大部分生成式接口在 0.3 中可用，但 0.4 查询具有与生成式外部匹配的内部实现，并且有更多技巧。所有结果缩小都通过`filter()`和`filter_by()`进行，限制/偏移要么通过数组切片要么通过`limit()`/`offset()`进行，连接通过`join()`和`outerjoin()`进行（或更手动地，通过`select_from()`以及手动形成的条件）。

为避免弃用警告，您必须对您的 03 代码进行一些更改

User.query.get_by( **kwargs )

```py
User.query.filter_by(**kwargs).first()
```

User.query.select_by( **kwargs )

```py
User.query.filter_by(**kwargs).all()
```

User.query.select()

```py
User.query.filter(xxx).all()
```

#### 新的基于属性的表达式构造

在 ORM 中最明显的区别是，现在你可以直接使用基于类的属性构建查询条件。在使用映射类时不再需要“.c.”前缀：

```py
session.query(User).filter(and_(User.name == "fred", User.id > 17))
```

虽然简单的基于列的比较不是什么大问题，但类属性有一些新的“更高级”的构造可用，包括以前仅在`filter_by()`中可用的内容：

```py
# comparison of scalar relations to an instance
filter(Address.user == user)

# return all users who contain a particular address
filter(User.addresses.contains(address))

# return all users who *dont* contain the address
filter(~User.address.contains(address))

# return all users who contain a particular address with
# the email_address like '%foo%'
filter(User.addresses.any(Address.email_address.like("%foo%")))

# same, email address equals 'foo@bar.com'.  can fall back to keyword
# args for simple comparisons
filter(User.addresses.any(email_address="foo@bar.com"))

# return all Addresses whose user attribute has the username 'ed'
filter(Address.user.has(name="ed"))

# return all Addresses whose user attribute has the username 'ed'
# and an id > 5 (mixing clauses with kwargs)
filter(Address.user.has(User.id > 5, name="ed"))
```

`Column`集合仍然可以在映射类的`.c`属性中使用。请注意，基于属性的表达式仅适用于映射类的映射属性。在正常表和从 SQL 表达式生成的可选择对象中，仍然使用`.c`来访问列。

#### 自动连接别名

我们现在已经有了 join()和 outerjoin()：

```py
session.query(Order).join("items")
```

现在你可以给它们起别名：

```py
session.query(Order).join("items", aliased=True).filter(Item.name="item 1").join(
    "items", aliased=True
).filter(Item.name == "item 3")
```

上述将从 orders->items 创建两个连接，使用别名。每个连接后续的`filter()`调用将调整其表条件为别名的条件。要获取`Item`对象，请使用`add_entity()`并将每个连接的`id`作为目标：

```py
session.query(Order).join("items", id="j1", aliased=True).filter(
    Item.name == "item 1"
).join("items", aliased=True, id="j2").filter(Item.name == "item 3").add_entity(
    Item, id="j1"
).add_entity(
    Item, id="j2"
)
```

返回形式为的元组：`（Order，Item，Item）`。

#### 自引用查询

因此，query.join()现在可以创建别名。那给了我们什么？自引用查询！连接可以在没有任何`Alias`对象的情况下完成：

```py
# standard self-referential TreeNode mapper with backref
mapper(
    TreeNode,
    tree_nodes,
    properties={
        "children": relation(
            TreeNode, backref=backref("parent", remote_side=tree_nodes.id)
        )
    },
)

# query for node with child containing "bar" two levels deep
session.query(TreeNode).join(["children", "children"], aliased=True).filter_by(
    name="bar"
)
```

要为沿途的每个表添加条件以进行别名连接，您可以使用`from_joinpoint`来继续针对相同行别名进行连接：

```py
# search for the treenode along the path "n1/n12/n122"

# first find a Node with name="n122"
q = sess.query(Node).filter_by(name="n122")

# then join to parent with "n12"
q = q.join("parent", aliased=True).filter_by(name="n12")

# join again to the next parent with 'n1'.  use 'from_joinpoint'
# so we join from the previous point, instead of joining off the
# root table
q = q.join("parent", aliased=True, from_joinpoint=True).filter_by(name="n1")

node = q.first()
```

#### `query.populate_existing()`

`query.load()`（或`session.refresh()`）的急切版本。如果查询中加载的每个实例，包括所有急切加载的项，已经存在于会话中，则立即刷新它们：

```py
session.query(Blah).populate_existing().all()
```

#### 新的查询 API

查询标准化为生成接口（旧接口仍在，只是已弃用）。虽然大多数生成接口在 0.3 中可用，但 0.4 版本的查询具有匹配生成外部的内部要领，并且有更多技巧。所有结果缩小都通过`filter()`和`filter_by()`进行，限制/偏移要么通过数组切片，要么通过`limit()`/`offset()`进行，连接是通过`join()`和`outerjoin()`进行的（或者更手动地，通过`select_from()`以及手动形成的条件）。

为了避免弃用警告，您必须对您的 03 代码进行一些更改

User.query.get_by(**kwargs)

```py
User.query.filter_by(**kwargs).first()
```

User.query.select_by(**kwargs)

```py
User.query.filter_by(**kwargs).all()
```

User.query.select()

```py
User.query.filter(xxx).all()
```

#### 新的基于属性的表达式构造

在 ORM 中最明显的区别是，现在您可以直接使用基于类的属性构造查询条件。当使用映射类时，不再需要“ .c.”前缀：

```py
session.query(User).filter(and_(User.name == "fred", User.id > 17))
```

虽然简单的基于列的比较不是什么大不了的事，但类属性具有一些新的“更高级别”的构造可用，包括以前仅在`filter_by()`中可用的内容：

```py
# comparison of scalar relations to an instance
filter(Address.user == user)

# return all users who contain a particular address
filter(User.addresses.contains(address))

# return all users who *dont* contain the address
filter(~User.address.contains(address))

# return all users who contain a particular address with
# the email_address like '%foo%'
filter(User.addresses.any(Address.email_address.like("%foo%")))

# same, email address equals 'foo@bar.com'.  can fall back to keyword
# args for simple comparisons
filter(User.addresses.any(email_address="foo@bar.com"))

# return all Addresses whose user attribute has the username 'ed'
filter(Address.user.has(name="ed"))

# return all Addresses whose user attribute has the username 'ed'
# and an id > 5 (mixing clauses with kwargs)
filter(Address.user.has(User.id > 5, name="ed"))
```

`.c`属性上的`Column`集合仍然可用于映射类中。请注意，基于属性的表达式仅适用于映射类的映射属性。`.c`仍然用于访问常规表中的列以及从 SQL 表达式生成的可选择对象。

#### 自动连接别名

我们已经有了一段时间的 join()和 outerjoin()：

```py
session.query(Order).join("items")
```

现在你可以给它们取别名：

```py
session.query(Order).join("items", aliased=True).filter(Item.name="item 1").join(
    "items", aliased=True
).filter(Item.name == "item 3")
```

以上将使用别名从订单->项目创建两个连接。每个之后的`filter()`调用将其表条件调整为别名的条件。要访问`Item`对象，请使用`add_entity()`并将每个连接目标化为`id`：

```py
session.query(Order).join("items", id="j1", aliased=True).filter(
    Item.name == "item 1"
).join("items", aliased=True, id="j2").filter(Item.name == "item 3").add_entity(
    Item, id="j1"
).add_entity(
    Item, id="j2"
)
```

返回形式为的元组：`（Order，Item，Item）`。

#### 自引用查询

因此，query.join()现在可以创建别名。那给了我们什么？自引用查询！连接可以在没有任何`Alias`对象的情况下完成：

```py
# standard self-referential TreeNode mapper with backref
mapper(
    TreeNode,
    tree_nodes,
    properties={
        "children": relation(
            TreeNode, backref=backref("parent", remote_side=tree_nodes.id)
        )
    },
)

# query for node with child containing "bar" two levels deep
session.query(TreeNode).join(["children", "children"], aliased=True).filter_by(
    name="bar"
)
```

要为沿途的每个表添加条件以进行别名连接，您可以使用`from_joinpoint`来继续针对相同行别名进行连接：

```py
# search for the treenode along the path "n1/n12/n122"

# first find a Node with name="n122"
q = sess.query(Node).filter_by(name="n122")

# then join to parent with "n12"
q = q.join("parent", aliased=True).filter_by(name="n12")

# join again to the next parent with 'n1'.  use 'from_joinpoint'
# so we join from the previous point, instead of joining off the
# root table
q = q.join("parent", aliased=True, from_joinpoint=True).filter_by(name="n1")

node = q.first()
```

#### `query.populate_existing()`

`query.load()`（或`session.refresh()`）的急切版本。如果查询中加载的每个实例，包括所有急切加载的项，已经存在于会话中，则立即刷新它们：

```py
session.query(Blah).populate_existing().all()
```

### 关系

#### 嵌入在更新/插入中的 SQL 子句

对于内联执行 SQL 子句，嵌入在`flush()`期间的 UPDATE 或 INSERT 中：

```py
myobject.foo = mytable.c.value + 1

user.pwhash = func.md5(password)

order.hash = text("select hash from hashing_table")
```

操作后使用延迟加载器设置列属性，以便在下次访问时发出加载新值的 SQL。

#### 自引用和循环急加载

由于我们的别名技术已经提高，`relation()`可以在同一张表上*任意次*进行连接；您告诉它您想要深入多少层。让我们更清楚地展示自引用的`TreeNode`：

```py
nodes = Table(
    "nodes",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("parent_id", Integer, ForeignKey("nodes.id")),
    Column("name", String(30)),
)

class TreeNode(object):
    pass

mapper(
    TreeNode,
    nodes,
    properties={"children": relation(TreeNode, lazy=False, join_depth=3)},
)
```

当我们说：

```py
create_session().query(TreeNode).all()
```

? 通过别名进行连接，从父级深入三层：

```py
SELECT
nodes_3.id  AS  nodes_3_id,  nodes_3.parent_id  AS  nodes_3_parent_id,  nodes_3.name  AS  nodes_3_name,
nodes_2.id  AS  nodes_2_id,  nodes_2.parent_id  AS  nodes_2_parent_id,  nodes_2.name  AS  nodes_2_name,
nodes_1.id  AS  nodes_1_id,  nodes_1.parent_id  AS  nodes_1_parent_id,  nodes_1.name  AS  nodes_1_name,
nodes.id  AS  nodes_id,  nodes.parent_id  AS  nodes_parent_id,  nodes.name  AS  nodes_name
FROM  nodes  LEFT  OUTER  JOIN  nodes  AS  nodes_1  ON  nodes.id  =  nodes_1.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_2  ON  nodes_1.id  =  nodes_2.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_3  ON  nodes_2.id  =  nodes_3.parent_id
ORDER  BY  nodes.oid,  nodes_1.oid,  nodes_2.oid,  nodes_3.oid
```

注意漂亮干净的别名名称。连接不在乎是否针对同一立即表或一些其他对象，然后循环回开始。当指定`join_depth`时，任何类型的急加载链都可以循环回自身。当不存在时，急加载在遇到循环时会自动停止。

#### 复合类型

这是来自 Hibernate 阵营的一个例子。复合类型允许您定义一个由多个列（或一个列，如果您愿意）组成的自定义数据类型。让我们定义一个新类型，`Point`。存储一个 x/y 坐标：

```py
class Point(object):
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __composite_values__(self):
        return self.x, self.y

    def __eq__(self, other):
        return other.x == self.x and other.y == self.y

    def __ne__(self, other):
        return not self.__eq__(other)
```

`Point`对象的定义方式是特定于自定义类型的；构造函数接受参数列表，并且`__composite_values__()`方法生成这些参数的序列。顺序将与我们的映射器匹配，我们马上就会看到。

让我们创建一个存储每行两个点的顶点表：

```py
vertices = Table(
    "vertices",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x1", Integer),
    Column("y1", Integer),
    Column("x2", Integer),
    Column("y2", Integer),
)
```

然后，映射它！我们将创建一个存储两个`Point`对象的`Vertex`对象：

```py
class Vertex(object):
    def __init__(self, start, end):
        self.start = start
        self.end = end

mapper(
    Vertex,
    vertices,
    properties={
        "start": composite(Point, vertices.c.x1, vertices.c.y1),
        "end": composite(Point, vertices.c.x2, vertices.c.y2),
    },
)
```

一旦设置了您的复合类型，它就可以像任何其他类型一样使用：

```py
v = Vertex(Point(3, 4), Point(26, 15))
session.save(v)
session.flush()

# works in queries too
q = session.query(Vertex).filter(Vertex.start == Point(3, 4))
```

如果您想定义映射属性在表达式中生成 SQL 子句的方式，请创建自己的`sqlalchemy.orm.PropComparator`子类，定义任何常见操作符（如`__eq__()`，`__le__()`等），并将其发送到`composite()`。复合类型也可以作为主键，并且可在`query.get()`中使用：

```py
# a Document class which uses a composite Version
# object as primary key
document = query.get(Version(1, "a"))
```

#### `dynamic_loader()`关系

一个返回所有读操作的实时`Query`对象的`relation()`。写操作仅限于`append()`和`remove()`，对集合的更改在会话刷新之前不可见。此功能在“自动刷新”会话中特别方便，该会话会在每次查询之前刷新。

```py
mapper(
    Foo,
    foo_table,
    properties={
        "bars": dynamic_loader(
            Bar,
            backref="foo",
            # <other relation() opts>
        )
    },
)

session = create_session(autoflush=True)
foo = session.query(Foo).first()

foo.bars.append(Bar(name="lala"))

for bar in foo.bars.filter(Bar.name == "lala"):
    print(bar)

session.commit()
```

#### 新选项：`undefer_group()`，`eagerload_all()`

一些方便的查询选项。`undefer_group()`将整个“延迟”列组标记为未延迟：

```py
mapper(
    Class,
    table,
    properties={
        "foo": deferred(table.c.foo, group="group1"),
        "bar": deferred(table.c.bar, group="group1"),
        "bat": deferred(table.c.bat, group="group1"),
    },
)

session.query(Class).options(undefer_group("group1")).filter(...).all()
```

和`eagerload_all()`设置一系列属性在一次遍历中急加载：

```py
mapper(Foo, foo_table, properties={"bar": relation(Bar)})
mapper(Bar, bar_table, properties={"bat": relation(Bat)})
mapper(Bat, bat_table)

# eager load bar and bat
session.query(Foo).options(eagerload_all("bar.bat")).filter(...).all()
```

#### 新的集合 API

集合不再由 {{{InstrumentedList}}} 代理进行代理，并且对成员、方法和属性的访问是直接的。装饰器现在拦截进入和离开集合的对象，并且现在可以轻松地编写一个自定义集合类来管理自己的成员。灵活的装饰器还取代了 0.3 版本中自定义集合的命名方法接口，允许任何类轻松地适应用作集合容器。

基于字典的集合现在更易于使用，并且完全类似于`dict`。对`dict`来说，不再需要更改`__iter__`，新的内置`dict`类型涵盖了许多需求：

```py
# use a dictionary relation keyed by a column
relation(Item, collection_class=column_mapped_collection(items.c.keyword))
# or named attribute
relation(Item, collection_class=attribute_mapped_collection("keyword"))
# or any function you like
relation(Item, collection_class=mapped_collection(lambda entity: entity.a + entity.b))
```

现有的 0.3 版本类似`dict`和自由对象衍生的集合类将需要更新到新的 API。在大多数情况下，这只是在类定义中添加几个装饰器的问题。

#### 来自外部表/子查询的映射关系

该功能在 0.3 版本中悄悄出现，但由于更好地能够将针对表的子查询转换为针对该表的别名的子查询而得到改进，在 0.4 版本中得到改进；这对于贪婪加载、查询中的别名连接等非常重要。它减少了在只需要添加一些额外列或子查询时创建针对选择语句的映射器的需求：

```py
mapper(
    User,
    users,
    properties={
        "fullname": column_property(
            (users.c.firstname + users.c.lastname).label("fullname")
        ),
        "numposts": column_property(
            select([func.count(1)], users.c.id == posts.c.user_id)
            .correlate(users)
            .label("posts")
        ),
    },
)
```

典型的查询看起来像：

```py
SELECT  (SELECT  count(1)  FROM  posts  WHERE  users.id  =  posts.user_id)  AS  count,
users.firstname  ||  users.lastname  AS  fullname,
users.id  AS  users_id,  users.firstname  AS  users_firstname,  users.lastname  AS  users_lastname
FROM  users  ORDER  BY  users.oid
```

#### 更新/插入中嵌入的 SQL 子句

对于嵌入在`flush()`期间的 UPDATE 或 INSERT 中的 SQL 子句的内联执行：

```py
myobject.foo = mytable.c.value + 1

user.pwhash = func.md5(password)

order.hash = text("select hash from hashing_table")
```

在操作后设置了延迟加载器的列属性，以便在下次访问时发出加载新值的 SQL。

#### 自引用和循环贪婪加载

由于我们的别名技术已经提高，`relation()`可以沿着同一张表*任意次数*进行连接；您告诉它您想要多深。让我们更清楚地展示自引用的`TreeNode`：

```py
nodes = Table(
    "nodes",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("parent_id", Integer, ForeignKey("nodes.id")),
    Column("name", String(30)),
)

class TreeNode(object):
    pass

mapper(
    TreeNode,
    nodes,
    properties={"children": relation(TreeNode, lazy=False, join_depth=3)},
)
```

那么，当我们说：

```py
create_session().query(TreeNode).all()
```

? 在父级的别名上三级深度的连接：

```py
SELECT
nodes_3.id  AS  nodes_3_id,  nodes_3.parent_id  AS  nodes_3_parent_id,  nodes_3.name  AS  nodes_3_name,
nodes_2.id  AS  nodes_2_id,  nodes_2.parent_id  AS  nodes_2_parent_id,  nodes_2.name  AS  nodes_2_name,
nodes_1.id  AS  nodes_1_id,  nodes_1.parent_id  AS  nodes_1_parent_id,  nodes_1.name  AS  nodes_1_name,
nodes.id  AS  nodes_id,  nodes.parent_id  AS  nodes_parent_id,  nodes.name  AS  nodes_name
FROM  nodes  LEFT  OUTER  JOIN  nodes  AS  nodes_1  ON  nodes.id  =  nodes_1.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_2  ON  nodes_1.id  =  nodes_2.parent_id
LEFT  OUTER  JOIN  nodes  AS  nodes_3  ON  nodes_2.id  =  nodes_3.parent_id
ORDER  BY  nodes.oid,  nodes_1.oid,  nodes_2.oid,  nodes_3.oid
```

注意这些漂亮干净的别名。连接不在乎它是针对同一即时表还是针对某个其他对象，然后又回到开头。当指定了`join_depth`时，任何类型的贪婪加载都可以在自身上循环回来。当不存在时，贪婪加载在碰到循环时会自动停止。

#### 复合类型

这是 Hibernate 阵营中的一个特点。复合类型允许您定义一个由多个列（或一个列，如果您想要的话）组成的自定义数据类型。让我们定义一个新类型，`Point`。存储一个 x/y 坐标：

```py
class Point(object):
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __composite_values__(self):
        return self.x, self.y

    def __eq__(self, other):
        return other.x == self.x and other.y == self.y

    def __ne__(self, other):
        return not self.__eq__(other)
```

`Point` 对象的定义方式是特定于自定义类型的；构造函数接受一个参数列表，并且`__composite_values__()`方法生成这些参数的序列。稍后我们将看到，顺序将与我们的映射器相匹配。

让我们创建一个存储每行两个点的顶点表：

```py
vertices = Table(
    "vertices",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x1", Integer),
    Column("y1", Integer),
    Column("x2", Integer),
    Column("y2", Integer),
)
```

然后，映射它！我们将创建一个存储两个`Point`对象的`Vertex`对象：

```py
class Vertex(object):
    def __init__(self, start, end):
        self.start = start
        self.end = end

mapper(
    Vertex,
    vertices,
    properties={
        "start": composite(Point, vertices.c.x1, vertices.c.y1),
        "end": composite(Point, vertices.c.x2, vertices.c.y2),
    },
)
```

一旦设置了复合类型，它就可以像任何其他类型一样使用：

```py
v = Vertex(Point(3, 4), Point(26, 15))
session.save(v)
session.flush()

# works in queries too
q = session.query(Vertex).filter(Vertex.start == Point(3, 4))
```

如果您想要定义映射属性在表达式中生成 SQL 子句的方式，请创建自己的 `sqlalchemy.orm.PropComparator` 子类，定义任何常见操作符（如 `__eq__()`，`__le__()` 等），并将其发送到 `composite()`。组合类型也可以作为主键，并且可在 `query.get()` 中使用：

```py
# a Document class which uses a composite Version
# object as primary key
document = query.get(Version(1, "a"))
```

#### `dynamic_loader()` 关系

`relation()` 返回一个用于所有读取操作的实时 `Query` 对象。写操作仅限于 `append()` 和 `remove()`，对集合的更改在会话刷新之前不可见。这个特性在“自动刷新”会话中特别方便，在每次查询之前都会刷新。

```py
mapper(
    Foo,
    foo_table,
    properties={
        "bars": dynamic_loader(
            Bar,
            backref="foo",
            # <other relation() opts>
        )
    },
)

session = create_session(autoflush=True)
foo = session.query(Foo).first()

foo.bars.append(Bar(name="lala"))

for bar in foo.bars.filter(Bar.name == "lala"):
    print(bar)

session.commit()
```

#### 新选项：`undefer_group()`，`eagerload_all()`

一些方便的查询选项。`undefer_group()` 将整个“延迟”列组标记为未延迟：

```py
mapper(
    Class,
    table,
    properties={
        "foo": deferred(table.c.foo, group="group1"),
        "bar": deferred(table.c.bar, group="group1"),
        "bat": deferred(table.c.bat, group="group1"),
    },
)

session.query(Class).options(undefer_group("group1")).filter(...).all()
```

和 `eagerload_all()` 一次性设置一系列属性为急加载：

```py
mapper(Foo, foo_table, properties={"bar": relation(Bar)})
mapper(Bar, bar_table, properties={"bat": relation(Bat)})
mapper(Bat, bat_table)

# eager load bar and bat
session.query(Foo).options(eagerload_all("bar.bat")).filter(...).all()
```

#### 新的集合 API

集合不再由 `InstrumentedList` 代理代理，并且对成员、方法和属性的访问是直接的。装饰器现在拦截进入和离开集合的对象，并且现在可以轻松编写一个自定义的集合类来管理其自己的成员资格。灵活的装饰器也替代了 0.3 版本中自定义集合的命名方法接口，允许任何类容易地被调整为用作集合容器。

基于字典的集合现在更易于使用，并完全类似于 `dict`。不再需要更改 `__iter__` 来使用 `dict`，并且新的内置 `dict` 类型满足了许多需求：

```py
# use a dictionary relation keyed by a column
relation(Item, collection_class=column_mapped_collection(items.c.keyword))
# or named attribute
relation(Item, collection_class=attribute_mapped_collection("keyword"))
# or any function you like
relation(Item, collection_class=mapped_collection(lambda entity: entity.a + entity.b))
```

现有的 0.3 版本类似于 `dict` 和自由形式对象派生的集合类需要更新到新的 API。在大多数情况下，这只是简单地向类定义中添加几个装饰器。

#### 来自外部表/子查询的映射关系

这个特性在 0.3 中悄然出现，但在 0.4 中得到了改进，这要归功于更好地将针对表的子查询转换为针对该表的别名的子查询的能力；这对于急加载、查询中的别名连接等非常重要。它减少了在只需要添加一些额外列或子查询时创建针对选择语句的映射器的必要性：

```py
mapper(
    User,
    users,
    properties={
        "fullname": column_property(
            (users.c.firstname + users.c.lastname).label("fullname")
        ),
        "numposts": column_property(
            select([func.count(1)], users.c.id == posts.c.user_id)
            .correlate(users)
            .label("posts")
        ),
    },
)
```

典型的查询看起来像这样：

```py
SELECT  (SELECT  count(1)  FROM  posts  WHERE  users.id  =  posts.user_id)  AS  count,
users.firstname  ||  users.lastname  AS  fullname,
users.id  AS  users_id,  users.firstname  AS  users_firstname,  users.lastname  AS  users_lastname
FROM  users  ORDER  BY  users.oid
```

### 水平扩展（分片）API

[browser:/sqlalchemy/trunk/examples/sharding/attribute_shard .py]

### 会话

#### 新的会话创建范例；SessionContext，assignmapper 弃用

是的，整个流程正在用两个配置函数替换。同时使用两者将产生自 0.1 版以来最接近 0.1 版感觉的体验（即，输入最少）。

在定义您的 `engine`（或任何位置）的地方配置您自己的 `Session` 类：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("myengine://")
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

# use the new Session() freely
sess = Session()
sess.save(someobject)
sess.flush()
```

如果需要在会话后配置您的会话，比如说使用引擎，请稍后使用 `configure()` 添加：

```py
Session.configure(bind=create_engine(...))
```

所有`SessionContext`的行为以及`assignmapper`的`query`和`__init__`方法都移至新的`scoped_session()`函数中，该函数与`sessionmaker`和`create_session()`兼容：

```py
from sqlalchemy.orm import scoped_session, sessionmaker

Session = scoped_session(sessionmaker(autoflush=True, transactional=True))
Session.configure(bind=engine)

u = User(name="wendy")

sess = Session()
sess.save(u)
sess.commit()

# Session constructor is thread-locally scoped.  Everyone gets the same
# Session in the thread when scope="thread".
sess2 = Session()
assert sess is sess2
```

当使用线程本地`Session`时，返回的类已实现了所有`Session`的接口作为类方法，并且可以使用`mapper`类方法来使用“assignmapper”的功能。就像旧的`objectstore`时代一样……。

```py
# "assignmapper"-like functionality available via ScopedSession.mapper
Session.mapper(User, users_table)

u = User(name="wendy")

Session.commit()
```

#### 会话再次默认使用弱引用

`weak_identity_map`标志现在默认设置为`True`在`Session`上。外部解除引用并超出范围的实例将自动从会话中移除。但是，具有“脏”更改的项目将保持强引用，直到这些更改被刷新，此时对象将恢复为弱引用（这适用于“可变”类型，如可选属性）。将`weak_identity_map`设置为`False`将为那些像缓存一样使用会话的人恢复旧的强引用行为。

#### 自动事务会话

正如您可能已经注意到的，我们在`Session`上调用`commit()`。标志`transactional=True`意味着`Session`始终处于事务中，`commit()`会永久保存。

#### 自动刷新会话

此外，`autoflush=True`意味着`Session`将在每次`query`之前`flush()`，以及在调用`flush()`或`commit()`时。因此，现在这将起作用：

```py
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

u = User(name="wendy")

sess = Session()
sess.save(u)

# wendy is flushed, comes right back from a query
wendy = sess.query(User).filter_by(name="wendy").one()
```

#### 事务方法移至会话

`commit()`和`rollback()`，以及`begin()`现在直接在`Session`上。不再需要为任何事情使用`SessionTransaction`（它仍然在后台运行）。

```py
Session = sessionmaker(autoflush=True, transactional=False)

sess = Session()
sess.begin()

# use the session

sess.commit()  # commit transaction
```

与包含引擎级（即非 ORM）事务共享`Session`很容易：

```py
Session = sessionmaker(autoflush=True, transactional=False)

conn = engine.connect()
trans = conn.begin()
sess = Session(bind=conn)

# ... session is transactional

# commit the outermost transaction
trans.commit()
```

#### 使用 SAVEPOINT 的嵌套会话事务

在引擎和 ORM 级别可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

#### 两阶段提交会话

在引擎和 ORM 级别可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

#### 新的会话创建范式；SessionContext，assignmapper 已弃用

是的，整个设置正在被两个配置函数替换。同时使用两者将产生自 0.1 版本以来最接近 0.1 版本的感觉（即，键入的数量最少）。

在定义您的`engine`（或任何地方）时配置自己的`Session`类：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("myengine://")
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

# use the new Session() freely
sess = Session()
sess.save(someobject)
sess.flush()
```

如果您需要对会话进行后期配置，比如添加引擎，请稍后使用`configure()`进行添加：

```py
Session.configure(bind=create_engine(...))
```

所有`SessionContext`的行为以及`assignmapper`的`query`和`__init__`方法都移至新的`scoped_session()`函数中，该函数与`sessionmaker`和`create_session()`兼容：

```py
from sqlalchemy.orm import scoped_session, sessionmaker

Session = scoped_session(sessionmaker(autoflush=True, transactional=True))
Session.configure(bind=engine)

u = User(name="wendy")

sess = Session()
sess.save(u)
sess.commit()

# Session constructor is thread-locally scoped.  Everyone gets the same
# Session in the thread when scope="thread".
sess2 = Session()
assert sess is sess2
```

使用线程本地 `Session` 时，返回的类实现了所有 `Session` 的接口作为类方法，并且可以使用 `mapper` 类方法来使用 “assignmapper” 的功能。就像旧的 `objectstore` 时代一样……。

```py
# "assignmapper"-like functionality available via ScopedSession.mapper
Session.mapper(User, users_table)

u = User(name="wendy")

Session.commit()
```

#### 会话再次默认为弱引用

weak_identity_map 标志现在默认设置为 `True` 在 Session 上。外部解除引用并超出范围的实例会自动从会话中移除。但是，具有“脏”更改的项目将保持强引用，直到这些更改被刷新，此时对象将恢复为弱引用（这适用于‘可变’类型，如可选属性）。将 weak_identity_map 设置为 `False` 可以为那些像缓存一样使用会话的人恢复旧的强引用行为。

#### 自动事务会话

正如您可能已经注意到的，我们在 `Session` 上调用 `commit()`。标志 `transactional=True` 意味着 `Session` 总是处于事务中，`commit()` 永久保存。

#### 自动刷新会话

此外，`autoflush=True` 意味着 `Session` 在每次 `query` 之前都会执行 `flush()`，以及在调用 `flush()` 或 `commit()` 时也会执行。所以现在这将起作用：

```py
Session = sessionmaker(bind=engine, autoflush=True, transactional=True)

u = User(name="wendy")

sess = Session()
sess.save(u)

# wendy is flushed, comes right back from a query
wendy = sess.query(User).filter_by(name="wendy").one()
```

#### 事务方法移至会话

`commit()` 和 `rollback()`，以及 `begin()` 现在直接在 `Session` 上。不再需要为任何事情使用 `SessionTransaction`（它仍然在后台运行）。

```py
Session = sessionmaker(autoflush=True, transactional=False)

sess = Session()
sess.begin()

# use the session

sess.commit()  # commit transaction
```

与封闭的引擎级（即非 ORM）事务共享 `Session` 很容易：

```py
Session = sessionmaker(autoflush=True, transactional=False)

conn = engine.connect()
trans = conn.begin()
sess = Session(bind=conn)

# ... session is transactional

# commit the outermost transaction
trans.commit()
```

#### 使用 SAVEPOINT 的嵌套会话事务

在引擎和 ORM 层面都可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

#### 两阶段提交会话

在引擎和 ORM 层面都可用。迄今为止的 ORM 文档：

[`www.sqlalchemy.org/docs/04/session.html#unitofwork_managing`](https://www.sqlalchemy.org/docs/04/session.html#unitofwork_managing)

### 继承

#### 无连接或联合的多态继承

继承的新文档：[`www.sqlalchemy.org/docs/04`](https://www.sqlalchemy.org/docs/04) /mappers.html#advdatamapping_mapper_inheritance_joined

#### 使用 `get()` 时更好的多态行为

在加入表继承层次结构中，所有类都使用基类获得 `_instance_key`，即 `(BaseClass, (1, ), None)`。这样，当您对基类进行 `get()` 查询时，它可以在当前标识映射中定位子类实例，而无需查询数据库。

#### 无连接或联合的多态继承

继承的新文档：[`www.sqlalchemy.org/docs/04`](https://www.sqlalchemy.org/docs/04) /mappers.html#advdatamapping_mapper_inheritance_joined

#### 使用 `get()` 时更好的多态行为

在连接表继承层次结构中，所有类都使用基类获取`_instance_key`，即`(BaseClass, (1, ), None)`。这样当你对基类调用`get()`时，它可以在当前标识映射中定位子类实例，而无需查询数据库。

### 类型

#### `sqlalchemy.types.TypeDecorator`的自定义子类

有一个[新 API](https://www.sqlalchemy.org/docs/04/types.html#types_custom)用于子类化 TypeDecorator。在某些情况下使用 0.3 API 会导致编译错误。

#### `sqlalchemy.types.TypeDecorator`的自定义子类

有一个[新 API](https://www.sqlalchemy.org/docs/04/types.html#types_custom)用于子类化 TypeDecorator。在某些情况下使用 0.3 API 会导致编译错误。

## SQL 表达式

### 所有新的、确定性的标签/别名生成

所有“匿名”标签和别名现在都使用简单的`<name>_<number>`格式。SQL 更容易阅读，并且与计划优化器缓存兼容。只需查看一些教程中的示例：[`www.sqlalchemy.org/docs/04/ormtutorial.html`](https://www.sqlalchemy.org/docs/04/ormtutorial.html) [`www.sqlalchemy.org/docs/04/sqlexpression.html`](https://www.sqlalchemy.org/docs/04/sqlexpression.html)

### 生成式`select()`构造

这绝对是使用`select()`的正确方法。查看 htt p://www.sqlalchemy.org/docs/04/sqlexpression.html#sql_transf orm。

### 新操作系统

SQL 运算符和几乎每个 SQL 关键字现在都被抽象为编译器层。它们现在具有智能行为，并且具有类型/后端感知能力，请参阅：[`www.sqlalchemy.org/docs/04/sqlexpression.html#sql_operators`](https://www.sqlalchemy.org/docs/04/sqlexpression.html#sql_operators)

### 所有`type`关键字参数重命名为`type_`

就像它所说的：

```py
b = bindparam("foo", type_=String)
```

### `in_`函数更改为接受序列或可选择项

`in_`函数现在以其唯一参数接受值序列或可选择的。以前的 API 仍然支持传递值作为位置参数，但现在已过时。这意味着

```py
my_table.select(my_table.c.id.in_(1, 2, 3))
my_table.select(my_table.c.id.in_(*listOfIds))
```

应更改为

```py
my_table.select(my_table.c.id.in_([1, 2, 3]))
my_table.select(my_table.c.id.in_(listOfIds))
```

### 所有新的、确定性的标签/别名生成

所有“匿名”标签和别名现在都使用简单的`<name>_<number>`格式。SQL 更容易阅读，并且与计划优化器缓存兼容。只需查看一些教程中的示例：[`www.sqlalchemy.org/docs/04/ormtutorial.html`](https://www.sqlalchemy.org/docs/04/ormtutorial.html) [`www.sqlalchemy.org/docs/04/sqlexpression.html`](https://www.sqlalchemy.org/docs/04/sqlexpression.html)

### 生成式`select()`构造

这绝对是使用`select()`的正确方法。查看 htt p://www.sqlalchemy.org/docs/04/sqlexpression.html#sql_transf orm。

### 新操作系统

SQL 操作符以及几乎每个 SQL 关键字都现在抽象成了编译器层。它们现在具有智能行为，并且具有类型/后端感知性，请参阅：[`www.sqlalchemy.org/docs/04/sqlexpression.html#sql_operators`](https://www.sqlalchemy.org/docs/04/sqlexpression.html#sql_operators)

### 所有 `type` 关键字参数重命名为 `type_`

就像它说的那样：

```py
b = bindparam("foo", type_=String)
```

### `in_` 函数更改为接受序列或可选择项

`in_` 函数现在接受一个值序列或可选择项作为其唯一参数。之前的传递值作为位置参数的 API 仍然有效，但现在已被弃用。这意味着

```py
my_table.select(my_table.c.id.in_(1, 2, 3))
my_table.select(my_table.c.id.in_(*listOfIds))
```

应更改为

```py
my_table.select(my_table.c.id.in_([1, 2, 3]))
my_table.select(my_table.c.id.in_(listOfIds))
```

## 模式和反射

### `MetaData`、`BoundMetaData`、`DynamicMetaData`…

在 0.3.x 系列中，`BoundMetaData` 和 `DynamicMetaData` 已被弃用，而不是 `MetaData` 和 `ThreadLocalMetaData`。旧名称已在 0.4 版本中移除。更新很简单：

```py
+-------------------------------------+-------------------------+
|If You Had                           | Now Use                 |
+=====================================+=========================+
| ``MetaData``                        | ``MetaData``            |
+-------------------------------------+-------------------------+
| ``BoundMetaData``                   | ``MetaData``            |
+-------------------------------------+-------------------------+
| ``DynamicMetaData`` (with one       | ``MetaData``            |
| engine or threadlocal=False)        |                         |
+-------------------------------------+-------------------------+
| ``DynamicMetaData``                 | ``ThreadLocalMetaData`` |
| (with different engines per thread) |                         |
+-------------------------------------+-------------------------+
```

`MetaData` 类型中不常用的 `name` 参数已被移除。`ThreadLocalMetaData` 构造函数现在不接受任何参数。这两种类型现在可以绑定到一个 `Engine` 或单个 `Connection`。

### 一步多表反射

现在您可以在一次通行中从整个数据库或模式加载表定义并自动创建 `Table` 对象：

```py
>>> metadata = MetaData(myengine, reflect=True)
>>> metadata.tables.keys()
['table_a', 'table_b', 'table_c', '...']
```

`MetaData` 还增加了一个 `.reflect()` 方法，可以更精细地控制加载过程，包括指定要加载的可用表的子集。

### `MetaData`、`BoundMetaData`、`DynamicMetaData`…

在 0.3.x 系列中，`BoundMetaData` 和 `DynamicMetaData` 已被弃用，而不是 `MetaData` 和 `ThreadLocalMetaData`。旧名称已在 0.4 版本中移除。更新很简单：

```py
+-------------------------------------+-------------------------+
|If You Had                           | Now Use                 |
+=====================================+=========================+
| ``MetaData``                        | ``MetaData``            |
+-------------------------------------+-------------------------+
| ``BoundMetaData``                   | ``MetaData``            |
+-------------------------------------+-------------------------+
| ``DynamicMetaData`` (with one       | ``MetaData``            |
| engine or threadlocal=False)        |                         |
+-------------------------------------+-------------------------+
| ``DynamicMetaData``                 | ``ThreadLocalMetaData`` |
| (with different engines per thread) |                         |
+-------------------------------------+-------------------------+
```

`MetaData` 类型中不常用的 `name` 参数已经被移除。`ThreadLocalMetaData` 构造函数现在不接受任何参数。这两种类型现在可以绑定到一个 `Engine` 或单个 `Connection`。

### 一步多表反射

现在您可以在一次通行中从整个数据库或模式加载表定义并自动创建 `Table` 对象：

```py
>>> metadata = MetaData(myengine, reflect=True)
>>> metadata.tables.keys()
['table_a', 'table_b', 'table_c', '...']
```

`MetaData` 还增加了一个 `.reflect()` 方法，可以更精细地控制加载过程，包括指定要加载的可用表的子集。

## SQL 执行

### `engine`、`connectable` 和 `bind_to` 现在都是 `bind`

### `Transactions`、`NestedTransactions` 和 `TwoPhaseTransactions`

### 连接池事件

当新的 DB-API 连接被创建、检出和重新放回到池中时，连接池现在会触发事件。您可以使用这些事件在新连接上执行会话范围的 SQL 设置语句，例如。

### Oracle Engine 修复

在 0.3.11 版本中，Oracle Engine 处理主键的方式存在 bug。这些 bug 可能导致在使用 Oracle Engine 时，那些在其他引擎（如 sqlite）上运行良好的程序失败。在 0.4 版本中，Oracle Engine 已经重做，修复了这些主键问题。

### Oracle 的输出参数

```py
result = engine.execute(
    text(
        "begin foo(:x, :y, :z); end;",
        bindparams=[
            bindparam("x", Numeric),
            outparam("y", Numeric),
            outparam("z", Numeric),
        ],
    ),
    x=5,
)
assert result.out_parameters == {"y": 10, "z": 75}
```

### 连接绑定的 `MetaData`、`Sessions`

`MetaData` 和 `Session` 可以明确绑定到一个连接：

```py
conn = engine.connect()
sess = create_session(bind=conn)
```

### 更快、更可靠的 `ResultProxy` 对象

### `engine`，`connectable` 和 `bind_to` 现在都改为 `bind`

### `Transactions`，`NestedTransactions` 和 `TwoPhaseTransactions`

### 连接池事件

现在连接池在创建新的 DB-API 连接、检出和检入连接池时会触发事件。您可以利用这些事件在新连接上执行会话范围的 SQL 设置语句，例如。

### Oracle 引擎已修复

在 0.3.11 版本中，Oracle 引擎在处理主键时存在 bug。这些 bug 可能导致在使用 Oracle 引擎时，那些在其他引擎（如 sqlite）上正常运行的程序失败。在 0.4 版本中，Oracle 引擎已经重新设计，修复了这些主键问题。

### 用于 Oracle 的输出参数

```py
result = engine.execute(
    text(
        "begin foo(:x, :y, :z); end;",
        bindparams=[
            bindparam("x", Numeric),
            outparam("y", Numeric),
            outparam("z", Numeric),
        ],
    ),
    x=5,
)
assert result.out_parameters == {"y": 10, "z": 75}
```

### 连接绑定的 `MetaData`，`Sessions`

`MetaData` 和 `Session` 可以显式绑定到连接：

```py
conn = engine.connect()
sess = create_session(bind=conn)
```

### 更快、更可靠的 `ResultProxy` 对象
