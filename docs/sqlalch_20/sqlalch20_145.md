# SQLAlchemy 1.0 中的新功能？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_10.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_10.html)

关于本文档

本文档描述了 SQLAlchemy 版本 0.9 与 2014 年 5 月维护发布的版本之间的更改，以及于 2015 年 4 月发布的版本 1.0 之间的更改。

文档最后更新日期：2015 年 6 月 9 日

## 介绍

本指南介绍了 SQLAlchemy 版本 1.0 中的新功能，并记录了影响用户将其应用程序从 SQLAlchemy 0.9 系列迁移到 1.0 的更改。

请仔细查看行为变化部分，可能会有不兼容的行为变化。

## 新功能和改进 - ORM

### 新会话批量插入/更新 API

创建了一系列新的 `Session` 方法，直接提供钩子进入工作单元的发出 INSERT 和 UPDATE 语句的功能。当正确使用时，这个面向专家的系统可以允许使用 ORM 映射生成批量插入和更新语句批量执行，使语句以与直接使用 Core 相媲美的速度进行。

另请参阅

批量操作 - 介绍和完整文档

[#3100](https://www.sqlalchemy.org/trac/ticket/3100)

### 新性能示例套件

受到批量操作功能以及 FAQ 中的如何对 SQLAlchemy 驱动的应用程序进行性能分析？部分进行的基准测试的启发，添加了一个新的示例部分，其中包含几个旨在说明各种核心和 ORM 技术的相对性能特征的脚本。这些脚本按用例组织，并打包在一个单一的控制台界面下，以便可以运行任何组合的演示，输出时间、Python 分析结果和/或 RunSnake 分析显示。

另请参阅

性能

### “烘焙”查询

“烘焙”查询功能是一种不同寻常的新方法，允许使用缓存直接构建和调用 `Query` 对象，通过连续调用大大减少了 Python 函数调用开销（超过 75%）。通过将一个 `Query` 对象指定为一系列仅调用一次的 lambda，查询作为一个预编译单元开始变得可行：

```py
from sqlalchemy.ext import baked
from sqlalchemy import bindparam

bakery = baked.bakery()

def search_for_user(session, username, email=None):
    baked_query = bakery(lambda session: session.query(User))
    baked_query += lambda q: q.filter(User.name == bindparam("username"))

    baked_query += lambda q: q.order_by(User.id)

    if email:
        baked_query += lambda q: q.filter(User.email == bindparam("email"))

    result = baked_query(session).params(username=username, email=email).all()

    return result
```

另请参阅

烘焙查询

[#3054](https://www.sqlalchemy.org/trac/ticket/3054)

### 改进声明性混合，`@declared_attr` 和相关功能

声明式系统与`declared_attr`结合进行了大幅改进，以支持新的功能。

用`declared_attr`修饰的函数现在仅在生成基于混合的列副本之后才被调用。这意味着该函数可以调用混合建立的列，并将接收到正确的`Column`对象的引用：

```py
class HasFooBar(object):
    foobar = Column(Integer)

    @declared_attr
    def foobar_prop(cls):
        return column_property("foobar: " + cls.foobar)

class SomeClass(HasFooBar, Base):
    __tablename__ = "some_table"
    id = Column(Integer, primary_key=True)
```

在上述示例中，`SomeClass.foobar_prop`将针对`SomeClass`调用，并且`SomeClass.foobar`将是要映射到`SomeClass`的最终`Column`对象，而不是直接出现在`HasFooBar`上的非副本对象，即使列尚未映射。

`declared_attr`函数现在会根据每个类缓存返回的值，这样对同一属性的重复调用将返回相同的值。我们可以修改示例来说明这一点：

```py
class HasFooBar(object):
    @declared_attr
    def foobar(cls):
        return Column(Integer)

    @declared_attr
    def foobar_prop(cls):
        return column_property("foobar: " + cls.foobar)

class SomeClass(HasFooBar, Base):
    __tablename__ = "some_table"
    id = Column(Integer, primary_key=True)
```

以前，`SomeClass`将使用`foobar`列的特定副本进行映射，但通过第二次调用`foobar`来调用`foobar_prop`将会产生不同的列。在声明性设置时间内，`SomeClass.foobar`的值现在被记忆，因此即使在属性由映射器映射之前，每次调用`declared_attr`时，中间列值都将保持一致。

上述两种行为应该极大地帮助声明式定义许多从其他属性派生的映射器属性类型，其中`declared_attr`函数是从其他本地`declared_attr`函数调用的，这些函数在类实际映射之前出现。

对于一个相当特殊的边缘情况，其中希望构建一个声明性混合类，为每个子类建立不同的列，添加了一个新的修饰符 `declared_attr.cascading`。使用此修饰符，装饰的函数将为映射继承层次结构中的每个类单独调用。虽然对于特殊属性如 `__table_args__` 和 `__mapper_args__`，这已经是行为，但对于列和其他属性，默认情况下假定该属性仅附加到基类，并且仅从子类继承。使用 `declared_attr.cascading`，可以应用个别行为：

```py
class HasIdMixin(object):
    @declared_attr.cascading
    def id(cls):
        if has_inherited_table(cls):
            return Column(ForeignKey("myclass.id"), primary_key=True)
        else:
            return Column(Integer, primary_key=True)

class MyClass(HasIdMixin, Base):
    __tablename__ = "myclass"
    # ...

class MySubClass(MyClass):
  """ """

    # ...
```

另请参见

使用 _orm.declared_attr() 生成特定表继承列

最后，`AbstractConcreteBase` 类已经重新设计，以便在抽象基类上内联设置关系或其他映射器属性：

```py
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import (
    declarative_base,
    declared_attr,
    AbstractConcreteBase,
)

Base = declarative_base()

class Something(Base):
    __tablename__ = "something"
    id = Column(Integer, primary_key=True)

class Abstract(AbstractConcreteBase, Base):
    id = Column(Integer, primary_key=True)

    @declared_attr
    def something_id(cls):
        return Column(ForeignKey(Something.id))

    @declared_attr
    def something(cls):
        return relationship(Something)

class Concrete(Abstract):
    __tablename__ = "cca"
    __mapper_args__ = {"polymorphic_identity": "cca", "concrete": True}
```

上述映射将设置一个带有 `id` 和 `something_id` 列的表 `cca`，并且 `Concrete` 还将具有一个名为 `something` 的关系。新功能是 `Abstract` 也将具有一个独立配置的关系 `something`，该关系构建在基类的多态联合上。

[#3150](https://www.sqlalchemy.org/trac/ticket/3150) [#2670](https://www.sqlalchemy.org/trac/ticket/2670) [#3149](https://www.sqlalchemy.org/trac/ticket/3149) [#2952](https://www.sqlalchemy.org/trac/ticket/2952) [#3050](https://www.sqlalchemy.org/trac/ticket/3050)

### ORM 完整对象获取速度提高 25%

`loading.py` 模块的机制以及标识映射已经经历了几次内联、重构和修剪，因此现在原始行的加载速度大约快了 25%。假设有一个包含 100 万行的表，下面的脚本演示了改进最多的加载类型：

```py
import time
from sqlalchemy import Integer, Column, create_engine, Table
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Foo(Base):
    __table__ = Table(
        "foo",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("a", Integer(), nullable=False),
        Column("b", Integer(), nullable=False),
        Column("c", Integer(), nullable=False),
    )

engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo=True)

sess = Session(engine)

now = time.time()

# avoid using all() so that we don't have the overhead of building
# a large list of full objects in memory
for obj in sess.query(Foo).yield_per(100).limit(1000000):
    pass

print("Total time: %d" % (time.time() - now))
```

本地 MacBookPro 结果从 0.9 秒降至 1.0 秒的时间为 19 秒，降至 14 秒。在批量处理大量行时，`Query.yield_per()` 的调用总是一个好主意，因为它可以防止 Python 解释器一次性为所有对象及其仪器分配大量内存。没有 `Query.yield_per()`，在 MacBookPro 上的上述脚本在 0.9 上需要 31 秒，在 1.0 上需要 26 秒，额外的时间用于设置非常大的内存缓冲区。

### 新的 KeyedTuple 实现速度显著提高

我们研究了 `KeyedTuple` 实现，希望改进这样的查询：

```py
rows = sess.query(Foo.a, Foo.b, Foo.c).all()
```

使用 `KeyedTuple` 类而不是 Python 的 `collections.namedtuple()`，因为后者具有一个非常复杂的类型创建例程，比 `KeyedTuple` 慢得多。然而，当获取数十万行时，`collections.namedtuple()` 很快超过 `KeyedTuple`，随着实例调用次数的增加，`KeyedTuple` 的速度会急剧变慢。怎么办？一种新类型，介于两者之间的方法。对于“size”（返回的行数）和“num”（不同查询的数量）对所有三种类型进行测试，新的“轻量级键值元组”要么优于两者，要么略逊于更快的对象，具体取决于情况。在“甜蜜点”上，我们既创建了大量新类型，又获取了大量行，轻量级对象完全超过了 namedtuple 和 KeyedTuple：

```py
-----------------
size=10 num=10000                 # few rows, lots of queries
namedtuple: 3.60302400589         # namedtuple falls over
keyedtuple: 0.255059957504        # KeyedTuple very fast
lw keyed tuple: 0.582715034485    # lw keyed trails right on KeyedTuple
-----------------
size=100 num=1000                 # <--- sweet spot
namedtuple: 0.365247011185
keyedtuple: 0.24896979332
lw keyed tuple: 0.0889317989349   # lw keyed blows both away!
-----------------
size=10000 num=100
namedtuple: 0.572599887848
keyedtuple: 2.54251694679
lw keyed tuple: 0.613876104355
-----------------
size=1000000 num=10               # few queries, lots of rows
namedtuple: 5.79669594765         # namedtuple very fast
keyedtuple: 28.856498003          # KeyedTuple falls over
lw keyed tuple: 6.74346804619     # lw keyed trails right on namedtuple
```

[#3176](https://www.sqlalchemy.org/trac/ticket/3176)### 结构化内存使用方面的显著改进

通过对许多内部对象更显著地使用 `__slots__`，改进了结构化内存使用。这种优化特别针对具有大量表和列的大型应用程序的基本内存大小，并减少了各种高容量对象的内存大小，包括事件监听内部、比较器对象以及 ORM 属性和加载器策略系统的部分。

一个使用 heapy 测量 Nova 启动大小的工作台展示了 SQLAlchemy 对象、相关字典以及弱引用在“nova.db.sqlalchemy.models”基本导入中占用的空间减少了约 3.7 兆字节，或者说减少了 46%：

```py
# reported by heapy, summation of SQLAlchemy objects +
# associated dicts + weakref-related objects with core of Nova imported:

    Before: total count 26477 total bytes 7975712
    After: total count 18181 total bytes 4236456

# reported for the Python module space overall with the
# core of Nova imported:

    Before: Partition of a set of 355558 objects. Total size = 61661760 bytes.
    After: Partition of a set of 346034 objects. Total size = 57808016 bytes.
```### UPDATE 语句现在在刷新中与 executemany() 批处理

现在可以将 UPDATE 语句批处理到 ORM 刷新中，以更高效的 executemany() 调用执行，类似于 INSERT 语句可以批处理；这将根据以下标准在刷新中调用：

+   两个或更多连续的 UPDATE 语句涉及相同的要修改的列集。

+   语句在 SET 子句中没有嵌入的 SQL 表达式。

+   映射不使用 `mapper.version_id_col`，或者后端方言支持 executemany() 操作的“合理”行数；大多数 DBAPI 现在正确支持这一点。### Session.get_bind() 处理更广泛的继承场景

每当查询或工作单元刷新过程寻找与特定类对应的数据库引擎时，都会调用 `Session.get_bind()` 方法。该方法已经改进，以处理各种继承导向的场景，包括：

+   绑定到一个 Mixin 或抽象类：

    ```py
    class MyClass(SomeMixin, Base):
        __tablename__ = "my_table"
        # ...

    session = Session(binds={SomeMixin: some_engine})
    ```

+   基于表格，分别绑定到继承的具体子类：

    ```py
    class BaseClass(Base):
        __tablename__ = "base"

        # ...

    class ConcreteSubClass(BaseClass):
        __tablename__ = "concrete"

        # ...

        __mapper_args__ = {"concrete": True}

    session = Session(binds={base_table: some_engine, concrete_table: some_other_engine})
    ```

[#3035](https://www.sqlalchemy.org/trac/ticket/3035)  ### 在所有相关的查询情况下，Session.get_bind()将接收到 Mapper

修复了一系列问题，其中`Session.get_bind()`未接收到`Query`的主要`Mapper`，尽管此映射器是 readily available 的（主映射器是与`Query`对象关联的单个映射器，或者替代是与查询关联的第一个映射器）。

当传递给`Session.get_bind()`的`Mapper`对象通常由使用`Session.binds`参数的会话使用，以将映射器与一系列引擎关联（虽然在这种用例中，通常情况下“工作”，因为绑定通常会通过映射的表对象找到），或者更具体地实现一个用户定义的`Session.get_bind()`方法，该方法基于映射器提供一些选择引擎的模式，例如水平分片或所谓的“路由”会话，将查询路由到不同的后端。

这些场景包括：

+   `Query.count()`:

    ```py
    session.query(User).count()
    ```

+   `Query.update()` 和 `Query.delete()`，用于 UPDATE/DELETE 语句以及“fetch”策略所使用的 SELECT：

    ```py
    session.query(User).filter(User.id == 15).update(
        {"name": "foob"}, synchronize_session="fetch"
    )

    session.query(User).filter(User.id == 15).delete(synchronize_session="fetch")
    ```

+   对个别列的查询：

    ```py
    session.query(User.id, User.name).all()
    ```

+   对间接映射（例如`column_property`）的 SQL 函数和其他表达式：

    ```py
    class User(Base):
        ...

        score = column_property(func.coalesce(self.tables.users.c.name, None))

    session.query(func.max(User.score)).scalar()
    ```

[#3227](https://www.sqlalchemy.org/trac/ticket/3227) [#3242](https://www.sqlalchemy.org/trac/ticket/3242) [#1326](https://www.sqlalchemy.org/trac/ticket/1326)  ### .info 字典改进

`InspectionAttr.info` 集合现在可用于从`Mapper.all_orm_descriptors`集合中检索到的每种对象。这包括`hybrid_property`和`association_proxy()`。然而，由于这些对象是类绑定的描述符，必须**分开**从它们附加到的类中访问以获取属性。以下是使用`Mapper.all_orm_descriptors`命名空间进行说明：

```py
class SomeObject(Base):
    # ...

    @hybrid_property
    def some_prop(self):
        return self.value + 5

inspect(SomeObject).all_orm_descriptors.some_prop.info["foo"] = "bar"
```

它还可作为所有`SchemaItem`对象（例如`ForeignKey`，`UniqueConstraint`等）的构造函数参数，以及剩余的 ORM 构造，如`synonym()`。

[#2971](https://www.sqlalchemy.org/trac/ticket/2971)

[#2963](https://www.sqlalchemy.org/trac/ticket/2963)  ### ColumnProperty 构造与别名，order_by 配合效果更好

关于`column_property()`的各种问题已得到解决，特别是关于`aliased()`构造以及在 0.9 版本中引入的“按标签排序”逻辑（参见标签构造现在可以单独作为其名称在 ORDER BY 中呈现）。

给定如下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

A.b = column_property(select([func.max(B.id)]).where(B.a_id == A.id).correlate(A))
```

简单的场景中，包含两次“A.b”将无法正确渲染：

```py
print(sess.query(A, a1).order_by(a1.b))
```

这会导致错误的列排序：

```py
SELECT  a.id  AS  a_id,  (SELECT  max(b.id)  AS  max_1  FROM  b
WHERE  b.a_id  =  a.id)  AS  anon_1,  a_1.id  AS  a_1_id,
(SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a_1.id)  AS  anon_2
FROM  a,  a  AS  a_1  ORDER  BY  anon_1
```

新的输出：

```py
SELECT  a.id  AS  a_id,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1,  a_1.id  AS  a_1_id,
(SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a_1.id)  AS  anon_2
FROM  a,  a  AS  a_1  ORDER  BY  anon_2
```

还有许多情况下，“order by”逻辑会无法按标签排序，例如如果映射为“多态”：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {"polymorphic_on": type, "with_polymorphic": "*"}
```

order_by 将无法使用标签，因为由于多态加载，标签将被匿名化：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1
FROM  a  ORDER  BY  (SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a.id)
```

现在，由于按标签排序跟踪了匿名化标签，这现在有效：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1
FROM  a  ORDER  BY  anon_1
```

这些修复中还包括了一系列能够破坏`aliased()`构造状态的 heisenbugs，从而导致标签逻辑再次失败；这些问题也已得到解决。

[#3148](https://www.sqlalchemy.org/trac/ticket/3148) [#3188](https://www.sqlalchemy.org/trac/ticket/3188)

## 新功能和改进 - 核心

### 选择/查询 LIMIT / OFFSET 可以指定为任意 SQL 表达式

`Select.limit()` 和 `Select.offset()` 方法现在接受任何 SQL 表达式作为参数，而不仅仅是整数值。ORM `Query` 对象也会将任何表达式传递给底层的 `Select` 对象。通常情况下，这用于允许传递绑定参数，稍后可以用值替换：

```py
sel = select([table]).limit(bindparam("mylimit")).offset(bindparam("myoffset"))
```

不支持非整数 LIMIT 或 OFFSET 表达式的方言可能会继续不支持此行为；第三方方言可能还需要修改以利用新行为。当前使用 `._limit` 或 `._offset` 属性的方言将继续对指定为简单整数值的限制/偏移的情况进行处理。但是，当指定 SQL 表达式时，这两个属性在访问时将引发 `CompileError`。希望支持新功能的第三方方言现在应调用 `._limit_clause` 和 `._offset_clause` 属性以接收完整的 SQL 表达式，而不是整数值。### `ForeignKeyConstraint` 上的 `use_alter` 标志（通常）不再需要

`MetaData.create_all()` 和 `MetaData.drop_all()` 方法现在将使用一个系统，自动为涉及表之间相互依赖循环的外键约束生成 ALTER 语句，无需指定 `ForeignKeyConstraint.use_alter`。此外，外键约束现在不再需要具有名称才能通过 ALTER 创建；只有 DROP 操作需要名称。在 DROP 的情况下，该功能将确保只有具有显式名称的约束实际上包含在 ALTER 语句中。在 DROP 中存在无法解决的循环的情况下，如果无法继续执行 DROP，系统现在会发出简洁明了的错误消息。

`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 标志仍然存在，并且继续具有相同的效果，用于在 CREATE/DROP 场景中需要 ALTER 的约束条件的建立。

从版本 1.0.1 开始，在 SQLite 的情况下，特殊逻辑接管，在 DROP 过程中，给定表存在无法解决的循环；在这种情况下会发出警告，并且表将以**无**顺序删除，这在 SQLite 上通常是可以接受的，除非启用了约束。要解决警告并在 SQLite 数据库上至少进行部分排序，特别是在启用了约束的情况下，重新应用“use_alter”标志到那些应明确从排序中省略的 `ForeignKey` 和 `ForeignKeyConstraint` 对象。

另请参见

通过 ALTER 创建/删除外键约束 - 新行为的完整描述。

[#3282](https://www.sqlalchemy.org/trac/ticket/3282)  ### ResultProxy “auto close” 现在是“soft” close

在许多版本中，`ResultProxy` 对象一直在获取所有结果行后自动关闭。这是为了允许在不需要显式调用 `ResultProxy.close()` 的情况下使用对象；因为所有的 DBAPI 资源都已被释放，对象可以安全丢弃。然而，对象保持了严格的“closed”行为，这意味着任何后续对 `ResultProxy.fetchone()`、`ResultProxy.fetchmany()` 或 `ResultProxy.fetchall()` 的调用现在会引发 `ResourceClosedError`：

```py
>>> result = connection.execute(stmt)
>>> result.fetchone()
(1, 'x')
>>> result.fetchone()
None  # indicates no more rows
>>> result.fetchone()
exception: ResourceClosedError
```

这种行为与 pep-249 所述的不一致，pep-249 表明，即使结果已经耗尽，也可以重复调用获取方法。它还干扰了某些结果代理的行为，例如 cx_oracle 方言用于某些数据类型的 `BufferedColumnResultProxy`。

为了解决这个问题，`ResultProxy` 的“closed”状态被分为两个状态；一个“soft close”执行了“close”大部分功能，释放了 DBAPI 游标，并且在“close with result”对象的情况下还会释放连接，另一个“closed”状态包括了“soft close”的所有内容以及将获取方法设为“closed”。`ResultProxy.close()` 方法现在不会隐式调用，只会调用非公开的 `ResultProxy._soft_close()` 方法：

```py
>>> result = connection.execute(stmt)
>>> result.fetchone()
(1, 'x')
>>> result.fetchone()
None  # indicates no more rows
>>> result.fetchone()
None  # still None
>>> result.fetchall()
[]
>>> result.close()
>>> result.fetchone()
exception: ResourceClosedError  # *now* it raises
```

[#3330](https://www.sqlalchemy.org/trac/ticket/3330) [#3329](https://www.sqlalchemy.org/trac/ticket/3329)

### CHECK 约束现在支持命名约定中的`%(column_0_name)s`标记

`%(column_0_name)s`将从`CheckConstraint`表达式中找到的第一列派生：

```py
metadata = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table("foo", metadata, Column("value", Integer))

CheckConstraint(foo.c.value > 5)
```

将呈现：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value  CHECK  (value  >  5)
)
```

命名约束与由`SchemaType`生成的约束的组合，例如`Boolean`或`Enum`，现在也将使用所有 CHECK 约束约定。

另请参阅

命名 CHECK 约束

为布尔值、枚举和其他模式类型配置命名

[#3299](https://www.sqlalchemy.org/trac/ticket/3299)

### 当引用的列未附加到表时，约束条件可以在其引用的列附加到表时自动附加

自至少版本 0.8 以来，`Constraint`已经具有根据传递的与表关联的列“自动附加”到`Table`的能力：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

t = Table("t", m, Column("a", Integer), Column("b", Integer))

uq = UniqueConstraint(t.c.a, t.c.b)  # will auto-attach to Table

assert uq in t.constraints
```

为了帮助处理在声明性中经常出现的一些情况，即使`Column`对象尚未与`Table`关联，这种自动附加逻辑现在也可以运行；建立了额外的事件，以便当这些`Column`对象关联时，`Constraint`也被添加：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

uq = UniqueConstraint(a, b)

t = Table("t", m, a, b)

assert uq in t.constraints  # constraint auto-attached
```

以上功能是在版本 1.0.0b3 中作为一个晚期添加的。截至版本 1.0.4 的修复[#3411](https://www.sqlalchemy.org/trac/ticket/3411)确保如果`Constraint`引用了`Column`对象和字符串列名的混合，则不会发生此逻辑；因为我们尚未跟踪将名称添加到`Table`的操作：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

uq = UniqueConstraint(a, "b")

t = Table("t", m, a, b)

# constraint *not* auto-attached, as we do not have tracking
# to locate when a name 'b' becomes available on the table
assert uq not in t.constraints
```

在上面的示例中，将列“a”附加到表“t”的附件事件将在附加列“b”之前触发（因为在“b”之前在`Table`构造函数中声明了“a”），如果尝试附加约束，则约束将无法找到“b”。为了保持一致性，如果约束引用任何字符串名称，则会跳过在列附加时自动附加的逻辑。

当`Constraint`构造时，如果`Table`已经包含所有目标`Column`对象，则原始的自动附加逻辑仍然存在：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

t = Table("t", m, a, b)

uq = UniqueConstraint(a, "b")

# constraint auto-attached normally as in older versions
assert uq in t.constraints
```

[#3341](https://www.sqlalchemy.org/trac/ticket/3341) [#3411](https://www.sqlalchemy.org/trac/ticket/3411)  ### INSERT FROM SELECT 现在包括 Python 和 SQL 表达式默认值

如果未指定，默认情况下，`Insert.from_select()`现在包括 Python 和 SQL 表达式默认值；现在解除了不包括非服务器列默认值在 INSERT FROM SELECT 中的限制，并将这些表达式呈现为常量插入 SELECT 语句中：

```py
from sqlalchemy import Table, Column, MetaData, Integer, select, func

m = MetaData()

t = Table(
    "t", m, Column("x", Integer), Column("y", Integer, default=func.somefunction())
)

stmt = select([t.c.x])
print(t.insert().from_select(["x"], stmt))
```

将呈现为：

```py
INSERT  INTO  t  (x,  y)  SELECT  t.x,  somefunction()  AS  somefunction_1
FROM  t
```

可以使用`Insert.from_select.include_defaults`来禁用该功能。### 现在列服务器默认值呈现为字面值

当由`Column.server_default`设置为 SQL 表达式的`DefaultClause`存在时，将打开“literal binds”编译器标志。这允许在 SQL 中嵌入的字面值正确呈现，例如：

```py
from sqlalchemy import Table, Column, MetaData, Text
from sqlalchemy.schema import CreateTable
from sqlalchemy.dialects.postgresql import ARRAY, array
from sqlalchemy.dialects import postgresql

metadata = MetaData()

tbl = Table(
    "derp",
    metadata,
    Column("arr", ARRAY(Text), server_default=array(["foo", "bar", "baz"])),
)

print(CreateTable(tbl).compile(dialect=postgresql.dialect()))
```

现在呈现为：

```py
CREATE  TABLE  derp  (
  arr  TEXT[]  DEFAULT  ARRAY['foo',  'bar',  'baz']
)
```

以前，字面值`"foo", "bar", "baz"`会呈现为绑定参数，在 DDL 中无用。

[#3087](https://www.sqlalchemy.org/trac/ticket/3087)  ### UniqueConstraint 现在是表反射过程的一部分

使用`autoload=True`填充的`Table`对象现在将包括`UniqueConstraint`构造以及`Index`构造。对于 PostgreSQL 和 MySQL，这种逻辑有一些注意事项：

#### PostgreSQL

当创建唯一约束时，PostgreSQL 的行为是隐式地创建一个与该约束对应的唯一索引。`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法将继续**分别**返回这些条目，其中 `Inspector.get_indexes()` 现在在索引条目中特征上带有一个标记 `duplicates_constraint`，指示检测到的相应约束。然而，当使用 `Table(..., autoload=True)` 执行完整的表反射时，`Index` 构造被检测为与 `UniqueConstraint` 关联，并且**不会**出现在 `Table.indexes` 集合中；只有 `UniqueConstraint` 会出现在 `Table.constraints` 集合中。这种去重逻辑通过在查询 `pg_index` 时连接到 `pg_constraint` 表来实现，以查看这两个结构是否关联。

#### MySQL

MySQL 没有单独的 UNIQUE INDEX 和 UNIQUE 约束概念。虽然在创建表和索引时支持两种语法，但在存储时并没有任何不同。`Inspector.get_indexes()`和`Inspector.get_unique_constraints()`方法将继续**同时**返回 MySQL 中 UNIQUE 索引的条目，其中`Inspector.get_unique_constraints()`在约束条目中具有一个新的标记`duplicates_index`，指示这是与该索引对应的重复条目。然而，在使用`Table(..., autoload=True)`执行完整表反射时，`UniqueConstraint`构造在任何情况下都**不**是完全反映的`Table`构造的一部分；这个构造始终由在`Table.indexes`集合中具有`unique=True`设置的`Index`表示。

另请参阅

PostgreSQL Index Reflection

MySQL / MariaDB Unique Constraints and Reflection

[#3184](https://www.sqlalchemy.org/trac/ticket/3184)

### 安全地发出参数化警告的新系统

长期以来，存在一个限制，即警告消息不能引用数据元素，这样一个特定函数可能会发出无限数量的唯一警告。这种情况最常见的地方是在`Unicode type received non-unicode bind param value`警告中。将数据值放入此消息中意味着该模块的 Python `__warningregistry__`，或在某些情况下是 Python 全局的`warnings.onceregistry`，将无限增长，因为在大多数警告场景中，这两个集合中的一个会填充每个不同的警告消息。

通过使用特殊的`string`类型，有意改变字符串的哈希方式，我们可以控制大量参数化消息仅在一小组可能的哈希值上进行哈希，这样一个警告，比如`Unicode type received non-unicode bind param value`，可以被定制为仅发出特定次数；此后，Python 警告注册表将开始记录它们为重复项。

为了说明，以下测试脚本将仅显示对于 1000 个参数集中的十个参数集发出的十个警告：

```py
from sqlalchemy import create_engine, Unicode, select, cast
import random
import warnings

e = create_engine("sqlite://")

# Use the "once" filter (which is also the default for Python
# warnings).  Exactly ten of these warnings will
# be emitted; beyond that, the Python warnings registry will accumulate
# new values as dupes of one of the ten existing.
warnings.filterwarnings("once")

for i in range(1000):
    e.execute(
        select([cast(("foo_%d" % random.randint(0, 1000000)).encode("ascii"), Unicode)])
    )
```

这里的警告格式是：

```py
/path/lib/sqlalchemy/sql/sqltypes.py:186: SAWarning: Unicode type received
  non-unicode bind param value 'foo_4852'. (this warning may be
  suppressed after 10 occurrences)
```

[#3178](https://www.sqlalchemy.org/trac/ticket/3178)

## 关键行为变化 - ORM

### query.update()现在将字符串名称解析为映射的属性名称

`Query.update()`的文档说明给定的`values`字典是“以属性名称为键的字典”，这意味着这些是映射的属性名称。不幸的是，该函数更多地是设计为接收属性和 SQL 表达式，而不是字符串；当传递字符串时，这些字符串将直接传递到核心更新语句，而不解析这些名称在映射类上如何表示，这意味着名称必须与表列的名称完全匹配，而不是映射到类的属性的名称。

现在字符串名称被认真解析为属性名称：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column("user_name", String(50))
```

上面，列`user_name`被映射为`name`。以前，传递字符串的`Query.update()`调用必须如下调用：

```py
session.query(User).update({"user_name": "moonbeam"})
```

现在给定的字符串将根据实体解析：

```py
session.query(User).update({"name": "moonbeam"})
```

通常最好直接使用属性，以避免任何歧义：

```py
session.query(User).update({User.name: "moonbeam"})
```

此更改还表明，同义词和混合属性也可以通过字符串名称进行引用：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column("user_name", String(50))

    @hybrid_property
    def fullname(self):
        return self.name

session.query(User).update({"fullname": "moonbeam"})
```

[#3228](https://www.sqlalchemy.org/trac/ticket/3228)  ### 当将对象与 None 值比较到关系时发出警告

这个更改是从 1.0.1 版本开始的。一些用户正在执行基本上是这种形式的查询：

```py
session.query(Address).filter(Address.user == User(id=None))
```

目前 SQLAlchemy 不支持这种模式。对于所有版本，它会生成类似的 SQL：

```py
SELECT  address.id  AS  address_id,  address.user_id  AS  address_user_id,
address.email_address  AS  address_email_address
FROM  address  WHERE  ?  =  address.user_id
(None,)
```

请注意上面，有一个比较`WHERE ? = address.user_id`，其中绑定值`?`接收`None`，或在 SQL 中为`NULL`。**这在 SQL 中将始终返回 False**。这里的比较理论上会生成以下 SQL：

```py
SELECT  address.id  AS  address_id,  address.user_id  AS  address_user_id,
address.email_address  AS  address_email_address
FROM  address  WHERE  address.user_id  IS  NULL
```

但是，**目前还没有**。依赖于“NULL = NULL”在所有情况下产生 False 的应用程序存在风险，因为有一天，SQLAlchemy 可能会修复此问题以生成“IS NULL”，然后查询将产生不同的结果。因此，在这种操作中，您将看到一个警告：

```py
SAWarning: Got None for value of column user.id; this is unsupported
for a relationship comparison and will not currently produce an
IS comparison (but may in a future release)
```

请注意，这种模式在 1.0.0 版本中的大多数情况下都被打破，包括所有的测试版；会生成类似`SYMBOL('NEVER_SET')`的值。这个问题已经修复，但由于识别了这种模式，现在有了警告，以便我们可以更安全地修复这种破损行为（现在在[#3373](https://www.sqlalchemy.org/trac/ticket/3373)中捕获）在未来的版本中。

[#3371](https://www.sqlalchemy.org/trac/ticket/3371)  ### “否定包含或等于”关系比较将使用属性的当前值，而不是数据库的值

这个改变是在 1.0.1 中新增的；虽然我们本来希望这个改变在 1.0.0 中，但这只是由于[#3371](https://www.sqlalchemy.org/trac/ticket/3371)才显现出来的。

给定一个映射：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    a = relationship("A")
```

给定`A`，主键为 7，但我们将其更改为 10 而没有刷新：

```py
s = Session(autoflush=False)
a1 = A(id=7)
s.add(a1)
s.commit()

a1.id = 10
```

对于以这个对象为目标的多对一关系的查询，将在绑定参数中使用值 10：

```py
s.query(B).filter(B.a == a1)
```

生成：

```py
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  ?  =  b.a_id
(10,)
```

然而，在这个改变之前，这个条件的否定**不会**使用 10，而是会使用 7，除非对象首先被刷新：

```py
s.query(B).filter(B.a != a1)
```

生成（在 0.9 版本和 1.0.1 之前的所有版本中）：

```py
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  b.a_id  !=  ?  OR  b.a_id  IS  NULL
(7,)
```

对于一个瞬态对象，它会产生一个错误的查询：

```py
SELECT  b.id,  b.a_id
FROM  b
WHERE  b.a_id  !=  :a_id_1  OR  b.a_id  IS  NULL
-- {u'a_id_1': symbol('NEVER_SET')}
```

这种不一致性已经修复，在所有查询中，当前属性值，例如本例中的`10`，现在将被使用。

[#3374](https://www.sqlalchemy.org/trac/ticket/3374)  ### 关于没有预先存在的值的属性事件和其他操作的更改

在这个改变中，当访问一个对象时，默认的返回值`None`现在会在每次访问时动态返回，而不是在首次访问时通过特殊的“设置”操作隐式地设置属性的状态。这个改变的可见结果是，在获取时不会隐式修改`obj.__dict__`，并且对于`get_history()`和相关函数也有一些微小的行为变化。

给定一个没有状态的对象：

```py
>>> obj = Foo()
```

SQLAlchemy 一直以来的行为是，如果我们访问一个从未设置过的标量或多对一属性，它会返回`None`：

```py
>>> obj.someattr
None
```

这个`None`值实际上现在是`obj`状态的一部分，就像我们明确设置了属性一样，例如`obj.someattr = None`。然而，在这里“获取时设置”的行为会因为历史和事件而有所不同。它不会触发任何属性事件，并且另外如果我们查看历史，我们会看到这样：

```py
>>> inspect(obj).attrs.someattr.history
History(added=(), unchanged=[None], deleted=())   # 0.9 and below
```

也就是说，就好像属性始终是`None`，并且从未更改过一样。这与我们首先设置属性的情况明显不同：

```py
>>> obj = Foo()
>>> obj.someattr = None
>>> inspect(obj).attrs.someattr.history
History(added=[None], unchanged=(), deleted=())  # all versions
```

以上意味着我们的“设置”操作的行为可以被访问到的值通过“获取”而访问的事实破坏。在 1.0 中，这种不一致性已经解决了，不再实际设置任何东西当使用默认的“getter”时。

```py
>>> obj = Foo()
>>> obj.someattr
None
>>> inspect(obj).attrs.someattr.history
History(added=(), unchanged=(), deleted=())  # 1.0
>>> obj.someattr = None
>>> inspect(obj).attrs.someattr.history
History(added=[None], unchanged=(), deleted=())
```

上述行为之所以没有产生太大影响，是因为在关系数据库中的插入语句在大多数情况下将缺失值视为 NULL。无论 SQLAlchemy 是否收到了针对特定属性设置为 None 的历史事件，通常都不会有影响；因为发送 None/NULL 或不发送的区别不会产生影响。但是，正如 [#3060](https://www.sqlalchemy.org/trac/ticket/3060)（在关系绑定属性与 FK 绑定属性上的属性更改的优先级可能会发生变化中描述的那样）所示，有一些罕见的边缘情况，我们实际上确实希望明确设置为 `None`。此外，在此处允许属性事件意味着现在可以为 ORM 映射属性创建“默认值”函数。

作为这一变化的一部分，现在已禁用了在其他情况下生成隐式`None`的功能；这包括在接收到对一对多的属性设置操作时；以前，如果“旧”值未设置，则“旧”值将为`None`；现在将发送值 `NEVER_SET`，这是一个现在可以发送到属性监听器的值。当调用诸如 `Mapper.primary_key_from_instance()` 这样的映射器实用程序函数时，如果主键属性根本没有设置，而以前的值为 `None`，那么现在将是 `NEVER_SET` 符号，并且不会更改对象的状态。

[#3061](https://www.sqlalchemy.org/trac/ticket/3061)  ### 关系绑定属性与 FK 绑定属性上的属性更改的优先级可能会发生变化

作为 [#3060](https://www.sqlalchemy.org/trac/ticket/3060) 的副作用，将关系绑定属性设置为 `None` 现在是一个跟踪的历史事件，它指的是将 `None` 持久化到该属性的意图。由于一直都是设置关系绑定属性将优先于直接赋值给外键属性，因此在分配 None 时可以看到行为的变化。给定一个映射：

```py
class A(Base):
    __tablename__ = "table_a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "table_b"

    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("table_a.id"))
    a = relationship(A)
```

在 1.0 版本中，无论我们分配的值是对 `A` 对象的引用还是 `None`，关系绑定属性都优先于 FK 绑定属性，这在所有情况下都是成立的。在 0.9 版本中，行为不一致，并且只有在分配了一个值时才会生效；`None` 不会被考虑：

```py
a1 = A(id=1)
a2 = A(id=2)
session.add_all([a1, a2])
session.flush()

b1 = B()
b1.a = a1  # we expect a_id to be '1'; takes precedence in 0.9 and 1.0

b2 = B()
b2.a = None  # we expect a_id to be None; takes precedence only in 1.0

b1.a_id = 2
b2.a_id = 2

session.add_all([b1, b2])
session.commit()

assert b1.a is a1  # passes in both 0.9 and 1.0
assert b2.a is None  # passes in 1.0, in 0.9 it's a2
```

[#3060](https://www.sqlalchemy.org/trac/ticket/3060)  ### session.expunge() 将完全分离已删除的对象

`Session.expunge()` 的行为存在一个 bug，导致关于已删除对象的行为存在不一致性。`object_session()` 函数以及 `InstanceState.session` 属性仍然会报告对象属于 `Session`，即使已经执行了 expunge 操作：

```py
u1 = sess.query(User).first()
sess.delete(u1)

sess.flush()

assert u1 not in sess
assert inspect(u1).session is sess  # this is normal before commit

sess.expunge(u1)

assert u1 not in sess
assert inspect(u1).session is None  # would fail
```

注意，当事务正在进行中且尚未调用 `Session.expunge()` 时，`u1 not in sess` 为 True 是正常的，而 `inspect(u1).session` 仍然引用会话；完全分离通常在事务提交后完成。此问题也会影响依赖于 `Session.expunge()` 的函数，如 `make_transient()`。

[#3139](https://www.sqlalchemy.org/trac/ticket/3139)  ### 与 yield_per 明确不兼容的连接/子查询预加载

为了使 `Query.yield_per()` 方法更容易使用，如果在使用 yield_per 时要生效任何子查询预加载程序，或者使用集合的连接预加载程序，则会引发异常，因为这些当前与 yield-per 不兼容（理论上子查询加载可以兼容）。当引发此错误时，可以使用 `lazyload()` 选项发送一个星号：

```py
q = sess.query(Object).options(lazyload("*")).yield_per(100)
```

或者使用 `Query.enable_eagerloads()`：

```py
q = sess.query(Object).enable_eagerloads(False).yield_per(100)
```

`lazyload()` 选项的优点是仍然可以使用附加的一对多连接加载器选项：

```py
q = (
    sess.query(Object)
    .options(lazyload("*"), joinedload("some_manytoone"))
    .yield_per(100)
)
```  ### 处理重复连接目标的更改和修复

此处的更改包括了一些 bug，当连接两次到一个实体时，或者连接到多个单表实体对同一张表时会出现意外和不一致的行为，而不使用基于关系的 ON 子句时，以及当多次连接到相同目标关系时。

以以下映射开始：

```py
from sqlalchemy import Integer, Column, String, ForeignKey
from sqlalchemy.orm import Session, relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

查询两次连接到 `A.bs` 的情况：

```py
print(s.query(A).join(A.bs).join(A.bs))
```

将呈现为：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  a.id  =  b.a_id
```

查询对冗余的 `A.bs` 进行了去重，因为它试图支持以下情况：

```py
s.query(A).join(A.bs).filter(B.foo == "bar").reset_joinpoint().join(A.bs, B.cs).filter(
    C.bar == "bat"
)
```

也就是说，`A.bs` 是“路径”的一部分。作为 [#3367](https://www.sqlalchemy.org/trac/ticket/3367) 的一部分，两次到达相同的终点而不是作为更大路径的一部分将会发出警告：

```py
SAWarning: Pathed join target A.bs has already been joined to; skipping
```

更大的变化涉及加入到一个实体而不使用关系绑定路径。如果我们两次加入到 `B`：

```py
print(s.query(A).join(B, B.a_id == A.id).join(B, B.a_id == A.id))
```

在 0.9 版本中，会呈现如下：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  b  AS  b_1  ON  b_1.a_id  =  a.id
```

这是有问题的，因为别名是隐式的，在不同的 ON 子句的情况下可能导致不可预测的结果。

在 1.0 版本中，不会自动应用别名，我们会得到：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  b  ON  b.a_id  =  a.id
```

这将从数据库中引发错误。虽然如果我们从冗余关系和冗余非关系目标中都加入时，“重复加入目标”表现相同可能更好，但目前我们只在以前会发生隐式别名的更严重情况下更改行为，并且在关系情况下只发出警告。最终，在所有情况下，两次加入相同的内容而没有任何别名以消除歧义应该引发错误。

这个变化也对单表继承目标产生影响。使用以下映射：

```py
from sqlalchemy import Integer, Column, String, ForeignKey
from sqlalchemy.orm import Session, relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {"polymorphic_on": type, "polymorphic_identity": "a"}

class ASub1(A):
    __mapper_args__ = {"polymorphic_identity": "asub1"}

class ASub2(A):
    __mapper_args__ = {"polymorphic_identity": "asub2"}

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)

    a_id = Column(Integer, ForeignKey("a.id"))

    a = relationship("A", primaryjoin="B.a_id == A.id", backref="b")

s = Session()

print(s.query(ASub1).join(B, ASub1.b).join(ASub2, B.a))

print(s.query(ASub1).join(B, ASub1.b).join(ASub2, ASub2.id == B.a_id))
```

底部的两个查询是等效的，应该都呈现相同的 SQL：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  a  ON  b.a_id  =  a.id  AND  a.type  IN  (:type_1)
WHERE  a.type  IN  (:type_2)
```

上面的 SQL 是无效的，因为在 FROM 列表中两次呈现了“a”。然而，隐式别名 bug 只会在第二个查询中发生，并呈现如下：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  a  AS  a_1
ON  a_1.id  =  b.a_id  AND  a_1.type  IN  (:type_1)
WHERE  a_1.type  IN  (:type_2)
```

在上面的例子中，第二次加入到“a”是有别名的。虽然这看起来很方便，但这不是单继承查询的一般工作方式，而且是误导性和不一致的。

应用程序依赖于这个 bug 的应用程序现在会由数据库引发错误。解决方法是使用预期的形式。在查询中引用单继承实体的多个子类时，必须手动使用别名来消除表的歧义，因为所有子类通常指向同一张表：

```py
asub2_alias = aliased(ASub2)

print(s.query(ASub1).join(B, ASub1.b).join(asub2_alias, B.a.of_type(asub2_alias)))
```

[#3233](https://www.sqlalchemy.org/trac/ticket/3233) [#3367](https://www.sqlalchemy.org/trac/ticket/3367)

### 延迟列不再隐式取消延迟

标记为延迟的映射属性，即使其列以某种方式出现在结果集中，现在也将保持“延迟”。这是一个性能增强，因为 ORM 加载不再花时间搜索每个延迟列，当结果集被获取时。然而，对于一直依赖于此的应用程序，现在应该使用显式的 `undefer()` 或类似选项，以防止在访问属性时发出 SELECT。

### 废弃的 ORM 事件钩子已移除

自 0.5 版本以来已被弃用的以下 ORM 事件钩子已被移除：`translate_row`、`populate_instance`、`append_result`、`create_instance`。这些钩子的用例起源于非常早期的 0.1 / 0.2 系列的 SQLAlchemy，并且早已不再需要。特别是，这些钩子在很大程度上无法使用，因为这些事件中的行为契约与周围内部紧密相关，例如实例如何需要被创建和初始化以及列如何在 ORM 生成的行中定位。移除这些钩子极大地简化了 ORM 对象加载的机制。  ### 当使用自定义行加载器时，新 Bundle 功能的 API 更改

0.9 版本的新`Bundle`对象在 API 上有一个小改变，当在自定义类上覆盖`create_row_processor()`方法时。之前，示例代码如下：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
  """Override create_row_processor to return values as dictionaries"""

        def proc(row, result):
            return dict(zip(labels, (proc(row, result) for proc in procs)))

        return proc
```

未使用的`result`成员现已移除：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
  """Override create_row_processor to return values as dictionaries"""

        def proc(row):
            return dict(zip(labels, (proc(row) for proc in procs)))

        return proc
```

另请参见

使用捆绑组合选择属性  ### 现在默认为使用内连接加载时的右嵌套连接

`joinedload.innerjoin`以及`relationship.innerjoin`的行为现在是使用“nested”内连接，也就是右嵌套，作为内连接急加载链接到外连接急加载时的默认行为。为了获得当存在外连接时将所有连接急加载链接为外连接的旧行为，请使用`innerjoin="unnested"`。 

如从 0.9 版本中的右嵌套内连接可用于连接的急加载中介绍的，`innerjoin="nested"`的行为是，内连接急加载链接到外连接急加载时将使用右嵌套连接。使用`innerjoin=True`时现在隐含了`"nested"`：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin=True)
)
```

使用新的默认设置，这将使 FROM 子句呈现为：

```py
FROM users LEFT OUTER JOIN (orders JOIN items ON <onclause>) ON <onclause>
```

也就是说，使用右嵌套连接进行内连接，以便返回`users`的完整结果。使用内连接比使用外连接更有效，并且允许`joinedload.innerjoin`优化参数在所有情况下生效。

要获得旧的行为，请使用`innerjoin="unnested"`：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin="unnested")
)
```

这将避免右嵌套连接，并使用所有外连接将连接链接在一起，尽管有内连接指令：

```py
FROM users LEFT OUTER JOIN orders ON <onclause> LEFT OUTER JOIN items ON <onclause>
```

如 0.9 版本说明中所述，唯一在右嵌套连接方面存在困难的数据库后端是 SQLite；截至 0.9 版本，SQLAlchemy 将右嵌套连接转换为 SQLite 上的子查询作为连接目标。

另请参见

右嵌套内连接在联接急切加载中可用 - 介绍了 0.9.4 中引入的功能。

[#3008](https://www.sqlalchemy.org/trac/ticket/3008)  ### 子查询不再应用于 uselist=False 的联接急切加载

给定如下的联接急切加载：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b = relationship("B", uselist=False)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

s = Session()
print(s.query(A).options(joinedload(A.b)).limit(5))
```

SQLAlchemy 认为关系`A.b`是“一对多，加载为单个值”，本质上是“一对一”关系。然而，联接的急切加载一直将上述情况视为主查询需要在子查询中的情况，就像在主查询应用了 LIMIT 时通常需要的 B 对象集合一样：

```py
SELECT  anon_1.a_id  AS  anon_1_a_id,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  (SELECT  a.id  AS  a_id
FROM  a  LIMIT  :param_1)  AS  anon_1
LEFT  OUTER  JOIN  b  AS  b_1  ON  anon_1.a_id  =  b_1.a_id
```

然而，由于内部查询与外部查询的关系在`uselist=False`的情况下最多只共享一行（就像一对多一样），因此在这种情况下现在会删除带有 LIMIT +联接急切加载的“子查询”：

```py
SELECT  a.id  AS  a_id,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  a  LEFT  OUTER  JOIN  b  AS  b_1  ON  a.id  =  b_1.a_id
LIMIT  :param_1
```

如果 LEFT OUTER JOIN 返回多行，ORM 一直会在这里发出警告并忽略`uselist=False`的额外结果，因此在这种错误情况下结果不应更改。

[#3249](https://www.sqlalchemy.org/trac/ticket/3249)

### query.update() / query.delete()如果与 join()，select_from()，from_self()一起使用会引发异常

在 SQLAlchemy 0.9.10 中（截至 2015 年 6 月 9 日尚未发布），当调用`Query.update()`或`Query.delete()`方法时，如果查询还调用了`Query.join()`，`Query.outerjoin()`，`Query.select_from()`或`Query.from_self()`，则会发出警告。这些是不受支持的用例，在 0.9 系列中静默失败，直到 0.9.10 发出警告。在 1.0 中，这些情况会引发异常。

[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

### 使用`synchronize_session='evaluate'`的 query.update()在多表更新时会引发异常

`Query.update()`的“评估器”在多表更新时不起作用，当存在多个表时需要将其设置为`synchronize_session=False`或`synchronize_session='fetch'`。新行为是现在会显式引发异常，并提醒更改同步设置。这是从 0.9.7 开始发出的警告升级而来。

[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

### 复活的事件已被移除

“复活”ORM 事件已完全移除。自 0.8 版本移除工作单元中的旧“可变”系统以来，此事件已不再起作用。

### 使用 from_self()，count()时更改为单表继承条件

给定单表继承映射，例如：

```py
class Widget(Base):
    __table__ = "widget_table"

class FooWidget(Widget):
    pass
```

对子类使用`Query.from_self()`或`Query.count()`会产生一个子查询，然后将子类型的“WHERE”条件添加到外部：

```py
sess.query(FooWidget).from_self().all()
```

渲染：

```py
SELECT
  anon_1.widgets_id  AS  anon_1_widgets_id,
  anon_1.widgets_type  AS  anon_1_widgets_type
FROM  (SELECT  widgets.id  AS  widgets_id,  widgets.type  AS  widgets_type,
FROM  widgets)  AS  anon_1
WHERE  anon_1.widgets_type  IN  (?)
```

问题在于，如果内部查询没有指定所有列，那么我们无法在外部添加 WHERE 子句（实际上会尝试，并生成错误的查询）。这个决定显然可以追溯到 0.6.5 版本，注释中写着“可能需要对此进行更多调整”。好吧，这些调整已经到位！因此，上述查询现在将渲染为：

```py
SELECT
  anon_1.widgets_id  AS  anon_1_widgets_id,
  anon_1.widgets_type  AS  anon_1_widgets_type
FROM  (SELECT  widgets.id  AS  widgets_id,  widgets.type  AS  widgets_type,
FROM  widgets
WHERE  widgets.type  IN  (?))  AS  anon_1
```

这样，即使不包括“type”的查询也能正常工作！：

```py
sess.query(FooWidget.id).count()
```

渲染：

```py
SELECT  count(*)  AS  count_1
FROM  (SELECT  widgets.id  AS  widgets_id
FROM  widgets
WHERE  widgets.type  IN  (?))  AS  anon_1
```

[#3177](https://www.sqlalchemy.org/trac/ticket/3177)  ### 单表继承条件无条件添加到所有 ON 子句

当连接到单表继承子类目标时，ORM 始终在连接关系时添加“单表条件”。给定映射如下：

```py
class Widget(Base):
    __tablename__ = "widget"
    id = Column(Integer, primary_key=True)
    type = Column(String)
    related_id = Column(ForeignKey("related.id"))
    related = relationship("Related", backref="widget")
    __mapper_args__ = {"polymorphic_on": type}

class FooWidget(Widget):
    __mapper_args__ = {"polymorphic_identity": "foo"}

class Related(Base):
    __tablename__ = "related"
    id = Column(Integer, primary_key=True)
```

长期以来，JOIN 关系的行为是为类型渲染“单一继承”子句：

```py
s.query(Related).join(FooWidget, Related.widget).all()
```

SQL 输出：

```py
SELECT  related.id  AS  related_id
FROM  related  JOIN  widget  ON  related.id  =  widget.related_id  AND  widget.type  IN  (:type_1)
```

上面，因为我们连接到子类`FooWidget`，`Query.join()`知道要将`AND widget.type IN ('foo')`条件添加到 ON 子句中。

此处的更改是现在`AND widget.type IN()`条件现在附加到*任何*ON 子句，而不仅仅是从关系生成的子句，包括明确声明的子句：

```py
# ON clause will now render as
# related.id = widget.related_id AND widget.type IN (:type_1)
s.query(Related).join(FooWidget, FooWidget.related_id == Related.id).all()
```

以及当没有任何 ON 子句时的“隐式”连接：

```py
# ON clause will now render as
# related.id = widget.related_id AND widget.type IN (:type_1)
s.query(Related).join(FooWidget).all()
```

以前，这些的 ON 子句不会包括单一继承条件。已经为了解决此问题而添加此条件的应用程序将希望删除其显式使用，尽管在此期间如果条件恰好被重复渲染，它应该继续正常工作。

另请参见

处理重复连接目标的更改和修复

[#3222](https://www.sqlalchemy.org/trac/ticket/3222)

## 关键行为变化 - 核心

### 在将完整 SQL 片段强制转换为 text()时发出警告

自 SQLAlchemy 创建以来，一直强调不妨碍使用纯文本。Core 和 ORM 表达式系统旨在允许用户在任何可以使用纯文本 SQL 表达式的地方使用纯文本，不仅仅是您可以将完整的 SQL 字符串发送到`Connection.execute()`，而且您可以将带有 SQL 表达式的字符串发送到许多函数中，例如`Select.where()`，`Query.filter()`和`Select.order_by()`。

请注意，这里所说的“SQL 表达式”是指**完整的 SQL 字符串片段**，例如：

```py
# the argument sent to where() is a full SQL expression
stmt = select([sometable]).where("somecolumn = 'value'")
```

我们**不是在讨论字符串参数**，即传递成为参数化的字符串值的正常行为：

```py
# This is a normal Core expression with a string argument -
# we aren't talking about this!!
stmt = select([sometable]).where(sometable.c.somecolumn == "value")
```

Core 教程长期以来一直以使用这种技术的示例为特色，使用`select()`构造，其中几乎所有组件都被指定为纯字符串。然而，尽管存在这种长期行为和示例，用户显然对这种行为感到惊讶，当在社区中询问时，我无法找到任何用户实际上*不*感到惊讶，即您可以将完整字符串发送到像`Query.filter()`这样的方法中。

因此，这里的更改是鼓励用户在部分或完全由文本片段组成的 SQL 中对文本字符串进行限定。在如下组合 select 时：

```py
stmt = select(["a", "b"]).where("a = b").select_from("sometable")
```

语句正常构建，与以前的所有强制转换相同。然而，将会看到以下警告被发出：

```py
SAWarning: Textual column expression 'a' should be explicitly declared
with text('a'), or use column('a') for more specificity
(this warning may be suppressed after 10 occurrences)

SAWarning: Textual column expression 'b' should be explicitly declared
with text('b'), or use column('b') for more specificity
(this warning may be suppressed after 10 occurrences)

SAWarning: Textual SQL expression 'a = b' should be explicitly declared
as text('a = b') (this warning may be suppressed after 10 occurrences)

SAWarning: Textual SQL FROM expression 'sometable' should be explicitly
declared as text('sometable'), or use table('sometable') for more
specificity (this warning may be suppressed after 10 occurrences)
```

这些警告试图通过显示参数以及字符串接收位置来准确指出问题所在。这些警告利用 Session.get_bind()处理更广泛的继承场景，以便可以安全地发出参数化警告而不会耗尽内存，如果希望将警告作为异常处理，应使用[Python 警告过滤器](https://docs.python.org/2/library/warnings.html)：

```py
import warnings

warnings.simplefilter("error")  # all warnings raise an exception
```

鉴于上述警告，我们的语句运行正常，但为了摆脱警告，我们将重写我们的语句如下：

```py
from sqlalchemy import select, text

stmt = (
    select([text("a"), text("b")]).where(text("a = b")).select_from(text("sometable"))
)
```

正如警告所建议的那样，如果我们使用`column()`和`table()`，我们可以使我们的语句对文本更具体：

```py
from sqlalchemy import select, text, column, table

stmt = (
    select([column("a"), column("b")])
    .where(text("a = b"))
    .select_from(table("sometable"))
)
```

还请注意，现在可以从“sqlalchemy”导入 `table()` 和 `column()` 而不需要“sql”部分。

此处的行为适用于 `select()` 以及 `Query` 上的关键方法，包括 `Query.filter()`、`Query.from_statement()` 和 `Query.having()`。

#### ORDER BY 和 GROUP BY 是特殊情况。

有一种情况下，使用字符串具有特殊含义，并且作为此更改的一部分，我们增强了其功能。当我们有一个引用某列名或命名标签的 `select()` 或 `Query` 时，我们可能想要对已知列或标签进行 GROUP BY 和/或 ORDER BY：

```py
stmt = (
    select([user.c.name, func.count(user.c.id).label("id_count")])
    .group_by("name")
    .order_by("id_count")
)
```

在上述语句中，我们期望看到“ORDER BY id_count”，而不是函数的重新陈述。在编译期间，字符串参数会被主动与列子句中的条目匹配，因此上述语句将产生我们期望的结果，不会有警告（尽管请注意 `"name"` 表达式已解析为 `users.name`！）：

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  GROUP  BY  users.name  ORDER  BY  id_count
```

但是，如果我们引用无法定位的名称，则会再次收到警告，如下所示：

```py
stmt = select([user.c.name, func.count(user.c.id).label("id_count")]).order_by(
    "some_label"
)
```

输出确实按我们说的做了，但再次警告我们：

```py
SAWarning: Can't resolve label reference 'some_label'; converting to
text() (this warning may be suppressed after 10 occurrences)
```

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  ORDER  BY  some_label
```

上述行为适用于我们可能想要引用所谓的“标签引用”的所有地方；ORDER BY 和 GROUP BY，以及 OVER 子句和 DISTINCT ON 子句内引用列的地方（例如 PostgreSQL 语法）。

我们仍然可以使用 `text()` 指定任意表达式用于 ORDER BY 或其他地方：

```py
stmt = select([users]).order_by(text("some special expression"))
```

整个更改的结果是，SQLAlchemy 现在希望我们告诉它当发送一个字符串时，该字符串明确是一个 `text()` 构造，或者是列、表等，如果我们将其用作 order by、group by 或其他表达式中的标签名称，则 SQLAlchemy 期望该字符串解析为已知内容，否则它应该再次使用 `text()` 或类似内容进行限定。

[#2992](https://www.sqlalchemy.org/trac/ticket/2992)  ### 使用多值插入时，为每一行单独调用 Python 端默认值

当使用`Insert.values()`的多值版本时，对于 Python 端列默认值的支持基本上没有实现，并且只会在特定情况下“意外”工作，当使用的方言使用非位置（例如，命名）风格的绑定参数，并且不需要为每一行调用 Python 端可调用时。

该功能已进行了改进，使其更类似于“executemany”样式的调用：

```py
import itertools

counter = itertools.count(1)
t = Table(
    "my_table",
    metadata,
    Column("id", Integer, default=lambda: next(counter)),
    Column("data", String),
)

conn.execute(
    t.insert().values(
        [
            {"data": "d1"},
            {"data": "d2"},
            {"data": "d3"},
        ]
    )
)
```

上述示例将为每一行单独调用`next(counter)`，这是预期的行为：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?),  (?,  ?),  (?,  ?)
(1,  'd1',  2,  'd2',  3,  'd3')
```

以前，位置方言会失败，因为不会为额外的位置生成绑定：

```py
Incorrect number of bindings supplied. The current statement uses 6,
and there are 4 supplied.
[SQL: u'INSERT INTO my_table (id, data) VALUES (?, ?), (?, ?), (?, ?)']
[parameters: (1, 'd1', 'd2', 'd3')]
```

并且使用“命名”方言时，“id”的相同值将在每一行中重新使用（因此，此更改与依赖于此的系统不兼容）：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (:id,  :data_0),  (:id,  :data_1),  (:id,  :data_2)
-- {u'data_2': 'd3', u'data_1': 'd2', u'data_0': 'd1', 'id': 1}
```

系统还将拒绝将“服务器端”默认值作为内联渲染的 SQL 调用，因为无法保证服务器端默认值与此兼容。如果 VALUES 子句为特定列渲染，则需要 Python 端值；如果省略的值仅引用服务器端默认值，则会引发异常：

```py
t = Table(
    "my_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String, server_default="some default"),
)

conn.execute(
    t.insert().values(
        [
            {"data": "d1"},
            {"data": "d2"},
            {},
        ]
    )
)
```

将会引发：

```py
sqlalchemy.exc.CompileError: INSERT value for column my_table.data is
explicitly rendered as a boundparameter in the VALUES clause; a
Python-side value or SQL expression is required
```

以前，“d1”值将被复制到第三行的值中（但仅适用于命名格式！）：

```py
INSERT  INTO  my_table  (data)  VALUES  (:data_0),  (:data_1),  (:data_0)
-- {u'data_1': 'd2', u'data_0': 'd1'}
```

[#3288](https://www.sqlalchemy.org/trac/ticket/3288)  ### 无法在事件的运行器内添加或删除事件侦听器

在事件内部从中删除事件侦听器将在迭代过程中修改列表的元素，这将导致仍附加的事件侦听器无声地失败。为了防止这种情况，同时保持性能，这些列表已被替换为`collections.deque()`，在迭代过程中不允许添加或删除，并且会引发`RuntimeError`。

[#3163](https://www.sqlalchemy.org/trac/ticket/3163)  ### INSERT…FROM SELECT 构造现在意味着 `inline=True`

现在使用`Insert.from_select()`会隐含在`insert()`上设置`inline=True`。这有助于修复一个错误，即 INSERT…FROM SELECT 结构会在支持的后端上不经意地编译为“implicit returning”，这会导致在插入零行的情况下（因为 implicit returning 需要一行）出现故障，以及在插入多行的情况下出现任意返回数据（例如，许多行中的第一行）。类似的更改也应用于具有多个参数集的 INSERT..VALUES；这种语句也不会再发出 implicit RETURNING。由于这两个结构都涉及可变数量的行，因此`ResultProxy.inserted_primary_key`访问器不适用。以前，有一个文档说明，即在一些数据库不支持返回的情况下，可能更喜欢`inline=True`与 INSERT..FROM SELECT，因此无法执行“implicit”返回，但无论如何，INSERT…FROM SELECT 都不需要 implicit returning。如果需要插入的数据，应该使用常规的显式`Insert.returning()`来返回可变数量的结果行。

[#3169](https://www.sqlalchemy.org/trac/ticket/3169)  ### `autoload_with` 现在隐含了 `autoload=True`

通过仅传递`Table.autoload_with`，可以设置`Table`以进行反射。

```py
my_table = Table("my_table", metadata, autoload_with=some_engine)
```

[#3027](https://www.sqlalchemy.org/trac/ticket/3027)  ### DBAPI 异常包装和`handle_error()`事件改进

在`Connection`对象失效后，尝试重新连接并遇到错误时，SQLAlchemy 对 DBAPI 异常的包装没有发生；这个问题已经解决。

另外，最近添加的`ConnectionEvents.handle_error()`事件现在在初始连接、重新连接时以及当通过`create_engine()`给定自定义连接函数时被调用。

`ExceptionContext` 对象有一个新的数据成员`ExceptionContext.engine`，它将始终指向正在使用的`Engine`，在那些`Connection`对象不可用的情况下（例如在初始连接时）。

[#3266](https://www.sqlalchemy.org/trac/ticket/3266)  ### ForeignKeyConstraint.columns 现在是一个 ColumnCollection

`ForeignKeyConstraint.columns`以前是一个普通列表，其中包含字符串或`Column`对象，具体取决于如何构建`ForeignKeyConstraint`以及它是否与表相关联。该集合现在是一个`ColumnCollection`，并且仅在`ForeignKeyConstraint`与`Table`相关联后才初始化。添加了一个新的访问器`ForeignKeyConstraint.column_keys`，无论对象如何构建或其当前状态如何，都会无条件地返回本地列集的字符串键。  ### MetaData.sorted_tables 访问器是“确定性的”

由`MetaData.sorted_tables`访问器返回的表的排序是“确定性的”；无论 Python 哈希如何，排序在所有情况下都应该是相同的。这是通过首先按名称对表进行排序，然后将它们传递给拓扑算法来实现的，该算法在迭代时保持该顺序。

请注意，此更改**尚未**应用于发出`MetaData.create_all()`或`MetaData.drop_all()`时应用的排序。

[#3084](https://www.sqlalchemy.org/trac/ticket/3084)  ### null()、false()和 true()常量不再是单例

这三个常量在 0.9 中被更改为返回“单例”值；不幸的是，这将导致类似以下的查询无法按预期渲染：

```py
select([null(), null()])
```

只渲染 `SELECT NULL AS anon_1`，因为两个 `null()` 构造将输出相同的 `NULL` 对象，而 SQLAlchemy 的核心模型是基于对象标识来确定词法重要性的。0.9 中的更改除了希望节省对象开销外并无重要性；通常，未命名的构造需要保持词法唯一，以便被唯一标记。

[#3170](https://www.sqlalchemy.org/trac/ticket/3170)  ### SQLite/Oracle 有不同的临时表/视图名称报告方法

在 SQLite/Oracle 的情况下，`Inspector.get_table_names()` 和 `Inspector.get_view_names()` 方法也会返回临时表和视图的名称，这是任何其他方言都不提供的（至少在 MySQL 的情况下甚至不可能）。这一逻辑已经移出到两个新方法 `Inspector.get_temp_table_names()` 和 `Inspector.get_temp_view_names()`。

请注意，对于大多数（如果不是全部）方言，通过 `Table('name', autoload=True)` 或通过 `Inspector.get_columns()` 等方法反射特定命名的临时表或临时视图仍然有效。特别是对于 SQLite，还修复了从临时表中反射 UNIQUE 约束的 bug，这是 [#3203](https://www.sqlalchemy.org/trac/ticket/3203)。

[#3204](https://www.sqlalchemy.org/trac/ticket/3204)

## 方言改进和变更 - PostgreSQL

### ENUM 类型创建/删除规则的全面改进

对于 PostgreSQL 的 `ENUM` 规则在创建和删除类型时变得更加严格。

创建的 `ENUM` 如果没有明确与 `MetaData` 对象关联，将会在 `Table.create()` 和 `Table.drop()` 时创建 *和* 删除：

```py
table = Table(
    "sometable", metadata, Column("some_enum", ENUM("a", "b", "c", name="myenum"))
)

table.create(engine)  # will emit CREATE TYPE and CREATE TABLE
table.drop(engine)  # will emit DROP TABLE and DROP TYPE - new for 1.0
```

这意味着如果第二个表也有一个名为 'myenum' 的枚举，上述 DROP 操作现在将失败。为了适应共享枚举类型的常见用例，元数据关联的枚举行为已经得到增强。

创建的 `ENUM` 如果明确与 `MetaData` 对象关联，将不会在 `Table.create()` 和 `Table.drop()` 时创建 *或* 删除，除了在调用带有 `checkfirst=True` 标志的 `Table.create()` 时：

```py
my_enum = ENUM("a", "b", "c", name="myenum", metadata=metadata)

table = Table("sometable", metadata, Column("some_enum", my_enum))

# will fail: ENUM 'my_enum' does not exist
table.create(engine)

# will check for enum and emit CREATE TYPE
table.create(engine, checkfirst=True)

table.drop(engine)  # will emit DROP TABLE, *not* DROP TYPE

metadata.drop_all(engine)  # will emit DROP TYPE

metadata.create_all(engine)  # will emit CREATE TYPE
```

[#3319](https://www.sqlalchemy.org/trac/ticket/3319)

### 新的 PostgreSQL 表选项

在通过 `Table` 构造渲染 DDL 时，增加了对 PG 表选项 TABLESPACE、ON COMMIT、WITH(OUT) OIDS 和 INHERITS 的支持。

另请参见

PostgreSQL 表选项

[#2051](https://www.sqlalchemy.org/trac/ticket/2051)

### 具有 PostgreSQL 方言的新 get_enums() 方法

`inspect()` 方法在 PostgreSQL 的情况下返回一个 `PGInspector` 对象，其中包括一个新的 `PGInspector.get_enums()` 方法，返回所有可用 `ENUM` 类型的信息：

```py
from sqlalchemy import inspect, create_engine

engine = create_engine("postgresql+psycopg2://host/dbname")
insp = inspect(engine)
print(insp.get_enums())
```

另请参见

`PGInspector.get_enums()`  ### PostgreSQL 方言反映了物化视图、外部表

更改如下：

+   具有 `autoload=True` 的 `Table` 构造现在将匹配数据库中存在的物化视图或外部表的名称。

+   `Inspector.get_view_names()` 将返回普通和物化视图名称。

+   `Inspector.get_table_names()` 对于 PostgreSQL 不会发生变化，它继续只返回普通表的名称。

+   添加了一个新方法 `PGInspector.get_foreign_table_names()` ，它将返回在 PostgreSQL 模式表中明确标记为“外部”的表的名称。

反射的变化涉及在查询 `pg_class.relkind` 时添加 `'m'` 和 `'f'` 到我们使用的修饰符列表，但这个变化是在 1.0.0 中新增的，以避免对那些在生产中运行 0.9 版本的用户造成任何不兼容的惊喜。

[#2891](https://www.sqlalchemy.org/trac/ticket/2891)  ### PostgreSQL `has_table()` 现在适用于临时表

这是一个简单的修复，使得临时表的“有表”现在可以正常工作，因此像下面的代码可以继续执行：

```py
from sqlalchemy import *

metadata = MetaData()
user_tmp = Table(
    "user_tmp",
    metadata,
    Column("id", INT, primary_key=True),
    Column("name", VARCHAR(50)),
    prefixes=["TEMPORARY"],
)

e = create_engine("postgresql://scott:tiger@localhost/test", echo="debug")
with e.begin() as conn:
    user_tmp.create(conn, checkfirst=True)

    # checkfirst will succeed
    user_tmp.create(conn, checkfirst=True)
```

如果这种行为导致一个非失败的应用程序表现出不同的行为，那是因为 PostgreSQL 允许一个非临时表悄悄地覆盖一个临时表。因此，像下面的代码现在将完全不同，不再创建真实表来跟随临时表：

```py
from sqlalchemy import *

metadata = MetaData()
user_tmp = Table(
    "user_tmp",
    metadata,
    Column("id", INT, primary_key=True),
    Column("name", VARCHAR(50)),
    prefixes=["TEMPORARY"],
)

e = create_engine("postgresql://scott:tiger@localhost/test", echo="debug")
with e.begin() as conn:
    user_tmp.create(conn, checkfirst=True)

    m2 = MetaData()
    user = Table(
        "user_tmp",
        m2,
        Column("id", INT, primary_key=True),
        Column("name", VARCHAR(50)),
    )

    # in 0.9, *will create* the new table, overwriting the old one.
    # in 1.0, *will not create* the new table
    user.create(conn, checkfirst=True)
```

[#3264](https://www.sqlalchemy.org/trac/ticket/3264)  ### PostgreSQL FILTER 关键字

PostgreSQL 现在支持聚合函数的 SQL 标准 FILTER 关键字，从 9.4 版开始。SQLAlchemy 允许使用 `FunctionElement.filter()` 来实现这一点：

```py
func.count(1).filter(True)
```

另请参阅

`FunctionElement.filter()`

`FunctionFilter`

### PG8000 方言支持客户端端编码

现在，`create_engine.encoding` 参数已被 pg8000 方言所遵守，使用连接处理程序发出 `SET CLIENT_ENCODING` 匹配所选编码。

### PG8000 原生 JSONB 支持

添加了对大于 1.10.1 版本的 PG8000 的支持，其中原生支持 JSONB。

### 对 PyPy 上的 psycopg2cffi 方言的支持

添加了对 pypy psycopg2cffi 方言的支持。

另请参阅

`sqlalchemy.dialects.postgresql.psycopg2cffi`

## 方言改进和更改 - MySQL

### MySQL TIMESTAMP 类型现在在所有情况下呈现 NULL / NOT NULL

MySQL 方言一直通过为具有`nullable=True`设置的 TIMESTAMP 列发出 NULL 来解决 MySQL 与 TIMESTAMP 列相关的隐式 NOT NULL 默认值的问题。然而，MySQL 5.6.6 及以上版本具有一个新标志[explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp)，它修复了 MySQL 的非标准行为，使其表现得像任何其他类型；为了适应这一点，SQLAlchemy 现在无条件地为所有 TIMESTAMP 列发出 NULL/NOT NULL。

另请参见

TIMESTAMP 列和 NULL

[#3155](https://www.sqlalchemy.org/trac/ticket/3155)  ### MySQL SET 类型进行了全面改进，以支持空集、unicode、空值处理

历史上，`SET`类型并未包含处理空集和空值的系统；由于不同的驱动程序对空字符串和空字符串集表示的处理方式不同，SET 类型仅尝试在这些行为之间进行权衡，选择将空集视为`set([''])`，这仍然是 MySQL-Connector-Python DBAPI 的当前行为。这里的部分理由是，否则在 MySQL SET 中实际上无法存储空字符串，因为驱动程序返回没有办法区分`set([''])`和`set()`的字符串。用户需要确定`set([''])`是否实际上表示“空集”。

新行为将空字符串的用例移动到一个特殊情况中，这是一个在 MySQL 文档中甚至没有记录的不寻常情况，并且`SET`的默认行为现在是：

+   将由 MySQL-python 返回的空字符串`''`视为空集`set()`；

+   将 MySQL-Connector-Python 返回的单空值集`set([''])`转换为空集`set()`；

+   为了处理实际希望在其可能值列表中包含空值`''`的集合类型的情况，实现了一个新功能（在这种用例中是必需的），其中集合值被持久化和加载为位整数值；添加了标志`SET.retrieve_as_bitwise`以启用此功能。

使用`SET.retrieve_as_bitwise`标志允许集合在没有值歧义的情况下持久化和检索。理论上，只要给定的值列表与数据库中声明的顺序完全匹配，就可以在所有情况下打开此标志；它只会使 SQL 回显输出有点不寻常。

否则，`SET`的默认行为保持不变，使用字符串来往复值。基于字符串的行为现在完全支持 unicode，包括使用 use_unicode=0 的 MySQL-python。

[#3283](https://www.sqlalchemy.org/trac/ticket/3283)

### MySQL 内部的“无此表”异常不会传递给事件处理程序

MySQL 方言现在将禁用`ConnectionEvents.handle_error()`事件，以防止这些语句触发内部用于检测表是否存在的事件处理程序。这是通过使用一个执行选项`skip_user_error_events`来实现的，该选项在该执行范围内禁用处理错误事件。通过这种方式，重写异常的用户代码不需要担心 MySQL 方言或其他偶尔需要捕获 SQLAlchemy 特定异常的方言。

### 更改了 MySQL-Connector 的`raise_on_warnings`默认值

将“raise_on_warnings”的默认值更改为 False，以用于 MySQL-Connector。由于某种原因，此值设置为 True。不幸的是，“buffered”标志必须保持为 True，因为 MySQL 连接器不允许关闭游标，除非所有结果都完全获取。

[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### MySQL 布尔符号“true”、“false”再次有效

0.9 版本对 IS/IS NOT 运算符以及[#2682](https://www.sqlalchemy.org/trac/ticket/2682)中的布尔类型进行了彻底改造，禁止 MySQL 方言在“IS”/“IS NOT”上下文中使用“true”和“false”符号。显然，即使 MySQL 没有“布尔”类型，但当使用特殊的“true”和“false”符号时，它支持 IS/IS NOT，尽管这些符号在其他情况下与“1”和“0”是同义的（并且 IS/IS NOT 不能与数字一起使用）。

因此，这里的变化是 MySQL 方言仍然保持“非本地布尔”，但`true()`和`false()`符号再次产生关键字“true”和“false”，因此像`column.is_(true())`这样的表达式在 MySQL 上再次有效。

[#3186](https://www.sqlalchemy.org/trac/ticket/3186)  ### match()运算符现在返回与 MySQL 浮点返回值兼容的 MatchType

`ColumnOperators.match()`表达式的返回类型现在是一个称为`MatchType`的新类型。这是`Boolean`的子类，可以被方言拦截以在 SQL 执行时产生不同的结果类型。

像下面这样的代码现在将正常运行并在 MySQL 上返回浮点数：

```py
>>> connection.execute(
...     select(
...         [
...             matchtable.c.title.match("Agile Ruby Programming").label("ruby"),
...             matchtable.c.title.match("Dive Python").label("python"),
...             matchtable.c.title,
...         ]
...     ).order_by(matchtable.c.id)
... )
[
 (2.0, 0.0, 'Agile Web Development with Ruby On Rails'),
 (0.0, 2.0, 'Dive Into Python'),
 (2.0, 0.0, "Programming Matz's Ruby"),
 (0.0, 0.0, 'The Definitive Guide to Django'),
 (0.0, 1.0, 'Python in a Nutshell')
]
```

[#3263](https://www.sqlalchemy.org/trac/ticket/3263)  ### Drizzle 方言现在是一个外部方言

[Drizzle](https://www.drizzle.org/)的方言现在是一个外部方言，可在[`bitbucket.org/zzzeek/sqlalchemy-drizzle`](https://bitbucket.org/zzzeek/sqlalchemy-drizzle)上找到。这个方言是在 SQLAlchemy 能够很好地适应第三方方言之前添加到 SQLAlchemy 的；未来，所有不属于“普遍使用”类别的数据库都是第三方方言。方言的实现没有改变，仍然基于 SQLAlchemy 中的 MySQL + MySQLdb 方言。该方言尚未发布，处于“attic”状态；但是它通过了大部分测试，通常工作正常，如果有人想要继续完善它。

## 方言改进和变化 - SQLite

### SQLite 命名和未命名的唯一和外键约束将进行检查和反映

SQLite 现在完全反映了有名称和无名称的唯一和外键约束。以前，外键名称被忽略，未命名的唯一约束被跳过。特别是这将有助于 Alembic 的新 SQLite 迁移功能。

为了实现这一点，对于外键和唯一约束，将 PRAGMA foreign_keys、index_list 和 index_info 的结果与对 CREATE TABLE 语句的正则表达式解析相结合，以形成对约束名称的完整描述，以及区分作为唯一约束创建的唯一约束与未命名 INDEX 的不同。

[#3244](https://www.sqlalchemy.org/trac/ticket/3244)

[#3261](https://www.sqlalchemy.org/trac/ticket/3261)

## 方言改进和变化 - SQL Server

### 使用基于主机名的 SQL Server 连接需要 PyODBC 驱动程序名称

使用无 DSN 连接的 PyODBC 连接到 SQL Server，例如使用显式主机名，现在需要一个驱动程序名称 - SQLAlchemy 将不再尝试猜测默认值：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=SQL+Server+Native+Client+10.0"
)
```

SQLAlchemy 在 Windows 上以前硬编码的默认值“SQL Server”已经过时，SQLAlchemy 不能根据操作系统/驱动程序检测来猜测最佳驱动程序。在使用 ODBC 时，始终首选使用 DSN 以避免这个问题。

[#3182](https://www.sqlalchemy.org/trac/ticket/3182)

### SQL Server 2012 大文本/二进制类型呈现为 VARCHAR、NVARCHAR、VARBINARY

对于 SQL Server 2012 及更高版本，`TextClause`、`UnicodeText` 和 `LargeBinary` 类型的呈现已经更改，可以完全控制行为，根据 Microsoft 的弃用指南。有关详细信息，请参阅大文本/二进制类型弃用。

## 方言改进和更改 - Oracle

### 改进的 Oracle CTE 支持

CTE 在 Oracle 中已经修复，还有一个新功能 `CTE.with_suffixes()` 可以帮助处理 Oracle 的特殊指令：

```py
included_parts = (
    select([part.c.sub_part, part.c.part, part.c.quantity])
    .where(part.c.part == "p1")
    .cte(name="included_parts", recursive=True)
    .suffix_with(
        "search depth first by part set ord1",
        "cycle part set y_cycle to 1 default 0",
        dialect="oracle",
    )
)
```

[#3220](https://www.sqlalchemy.org/trac/ticket/3220)

### DDL 的新 Oracle 关键字

COMPRESS、ON COMMIT、BITMAP 等关键字：

Oracle 表选项

Oracle 特定索引选项

## 介绍

本指南介绍了 SQLAlchemy 版本 1.0 中的新功能，并记录了影响用户将其应用程序从 SQLAlchemy 0.9 系列迁移到 1.0 的更改。

请仔细查看行为变化部分，可能会有不兼容的行为变化。

## 新功能和改进 - ORM

### 新会话批量插入/更新 API

创建了一系列新的`Session`方法，直接提供钩子到工作单元的功能，用于生成批量插入和更新语句分组，使语句可以以与直接使用 Core 相媲美的速度进行批量处理。

另请参阅

批量操作 - 介绍和完整文档

[#3100](https://www.sqlalchemy.org/trac/ticket/3100)

### 新性能示例套件

受到为批量操作功能以及如何对 SQLAlchemy 驱动的应用程序进行性能分析？FAQ 部分进行的基准测试的启发，添加了一个新的示例部分，其中包含几个旨在说明各种核心和 ORM 技术的相对性能特征的脚本。这些脚本按用例组织，并打包在一个单一的控制台界面下，以便可以运行任何组合的演示，输出时间、Python 分析结果和/或 RunSnake 分析显示。

另请参阅

性能

### “烘焙”查询

“烘焙”查询功能是一种不同寻常的新方法，允许直接构造和调用`Query`对象，并使用缓存，在连续调用时大大减少了 Python 函数调用的开销（超过 75%）。通过将`Query`对象指定为一系列仅调用一次的 lambda 表达式，可以开始将查询作为预编译单元来实现：

```py
from sqlalchemy.ext import baked
from sqlalchemy import bindparam

bakery = baked.bakery()

def search_for_user(session, username, email=None):
    baked_query = bakery(lambda session: session.query(User))
    baked_query += lambda q: q.filter(User.name == bindparam("username"))

    baked_query += lambda q: q.order_by(User.id)

    if email:
        baked_query += lambda q: q.filter(User.email == bindparam("email"))

    result = baked_query(session).params(username=username, email=email).all()

    return result
```

另请参见

烘焙查询

[#3054](https://www.sqlalchemy.org/trac/ticket/3054)

### 改进声明混合，`@declared_attr` 和相关功能

使用`declared_attr`的声明系统已经进行了全面改进，以支持新的功能。

现在使用`declared_attr`装饰的函数仅在生成任何基于混合的列副本后才调用。这意味着函数可以调用混合建立的列，并将接收到正确的`Column` 对象的引用：

```py
class HasFooBar(object):
    foobar = Column(Integer)

    @declared_attr
    def foobar_prop(cls):
        return column_property("foobar: " + cls.foobar)

class SomeClass(HasFooBar, Base):
    __tablename__ = "some_table"
    id = Column(Integer, primary_key=True)
```

在上面的例子中，`SomeClass.foobar_prop` 将针对 `SomeClass` 调用，并且 `SomeClass.foobar` 将是要映射到 `SomeClass` 的最终 `Column` 对象，而不是直接存在于 `HasFooBar` 上的非复制对象，即使列还没有映射。

现在，`declared_attr`函数在每个类基础上**备忘**返回的值，因此对相同属性的重复调用将返回相同的值。我们可以修改示例来说明这一点：

```py
class HasFooBar(object):
    @declared_attr
    def foobar(cls):
        return Column(Integer)

    @declared_attr
    def foobar_prop(cls):
        return column_property("foobar: " + cls.foobar)

class SomeClass(HasFooBar, Base):
    __tablename__ = "some_table"
    id = Column(Integer, primary_key=True)
```

以前，`SomeClass` 将使用一个特定的 `foobar` 列进行映射，但通过第二次调用 `foobar` 来调用 `foobar_prop` 将产生一个不同的列。现在，在声明设置时间内对 `SomeClass.foobar` 的值进行了备忘，因此即使在映射器将属性映射之前，`declared_attr` 被调用多少次，临时列值也将保持一致。

上述两个行为应大大有助于声明定义许多类型的映射器属性，这些属性源自其他属性，其中`declared_attr`函数在实际映射类之前从其他`declared_attr`函数本地调用。

对于一个相当特殊的边缘情况，其中希望构建一个声明性混合类，为每个子类建立不同的列，添加了一个新的修饰符 `declared_attr.cascading`。使用这个修饰符，装饰的函数将为映射继承层次结构中的每个类单独调用。虽然对于特殊属性如 `__table_args__` 和 `__mapper_args__`，这已经是默认行为，但对于列和其他属性，默认行为假定该属性仅附加到基类，并从子类继承。使用 `declared_attr.cascading`，可以应用个别行为：

```py
class HasIdMixin(object):
    @declared_attr.cascading
    def id(cls):
        if has_inherited_table(cls):
            return Column(ForeignKey("myclass.id"), primary_key=True)
        else:
            return Column(Integer, primary_key=True)

class MyClass(HasIdMixin, Base):
    __tablename__ = "myclass"
    # ...

class MySubClass(MyClass):
  """ """

    # ...
```

另请参见

使用 _orm.declared_attr() 生成特定表继承列的链接

最后，`AbstractConcreteBase` 类已经重新设计，以便在抽象基类上内联设置关系或其他映射属性：

```py
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import (
    declarative_base,
    declared_attr,
    AbstractConcreteBase,
)

Base = declarative_base()

class Something(Base):
    __tablename__ = "something"
    id = Column(Integer, primary_key=True)

class Abstract(AbstractConcreteBase, Base):
    id = Column(Integer, primary_key=True)

    @declared_attr
    def something_id(cls):
        return Column(ForeignKey(Something.id))

    @declared_attr
    def something(cls):
        return relationship(Something)

class Concrete(Abstract):
    __tablename__ = "cca"
    __mapper_args__ = {"polymorphic_identity": "cca", "concrete": True}
```

上述映射将建立一个名为 `cca` 的表，其中包含 `id` 和 `something_id` 列，而 `Concrete` 还将具有一个名为 `something` 的关系。新功能是 `Abstract` 也将具有一个独立配置的关系 `something`，该关系针对基类的多态联合构建。

[#3150](https://www.sqlalchemy.org/trac/ticket/3150) [#2670](https://www.sqlalchemy.org/trac/ticket/2670) [#3149](https://www.sqlalchemy.org/trac/ticket/3149) [#2952](https://www.sqlalchemy.org/trac/ticket/2952) [#3050](https://www.sqlalchemy.org/trac/ticket/3050)

### ORM 完整对象提取速度提高了 25%

`loading.py` 模块的机制以及标识映射已经经历了几次内联、重构和修剪，因此现在原始行的加载速度比以前快了大约 25%。假设有一个包含 100 万行的表，下面的脚本说明了哪种加载方式得到了最大的改进：

```py
import time
from sqlalchemy import Integer, Column, create_engine, Table
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Foo(Base):
    __table__ = Table(
        "foo",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("a", Integer(), nullable=False),
        Column("b", Integer(), nullable=False),
        Column("c", Integer(), nullable=False),
    )

engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo=True)

sess = Session(engine)

now = time.time()

# avoid using all() so that we don't have the overhead of building
# a large list of full objects in memory
for obj in sess.query(Foo).yield_per(100).limit(1000000):
    pass

print("Total time: %d" % (time.time() - now))
```

本地 MacBookPro 结果从 0.9 的 19 秒降至 1.0 的 14 秒。当批处理大量行时，`Query.yield_per()` 的调用总是一个好主意，因为它可以防止 Python 解释器一次性为所有对象及其仪器分配大量内存。没有 `Query.yield_per()`，在 MacBookPro 上的上述脚本在 0.9 上为 31 秒，在 1.0 上为 26 秒，额外的时间花费在设置非常大的内存缓冲区上。

### 新的 KeyedTuple 实现速度大幅提升

我们研究了 `KeyedTuple` 的实现，希望改进这样的查询：

```py
rows = sess.query(Foo.a, Foo.b, Foo.c).all()
```

使用 `KeyedTuple` 类而不是 Python 的 `collections.namedtuple()`，因为后者具有一个非常复杂的类型创建程序，比 `KeyedTuple` 的速度慢得多。然而，当获取数十万行时，`collections.namedtuple()` 很快就会超过 `KeyedTuple`，随着实例调用次数的增加，`KeyedTuple` 的速度会急剧变慢。怎么办？一种新类型，介于两者之间的方法。对于“大小”（返回的行数）和“num”（不同查询的数量）对所有三种类型进行测试，新的“轻量级键值元组”要么优于两者，要么略逊于更快的对象，取决于情况。在“甜蜜点”，即我们既创建了大量新类型又获取了大量行时，轻量级对象完全超过了 namedtuple 和 KeyedTuple：

```py
-----------------
size=10 num=10000                 # few rows, lots of queries
namedtuple: 3.60302400589         # namedtuple falls over
keyedtuple: 0.255059957504        # KeyedTuple very fast
lw keyed tuple: 0.582715034485    # lw keyed trails right on KeyedTuple
-----------------
size=100 num=1000                 # <--- sweet spot
namedtuple: 0.365247011185
keyedtuple: 0.24896979332
lw keyed tuple: 0.0889317989349   # lw keyed blows both away!
-----------------
size=10000 num=100
namedtuple: 0.572599887848
keyedtuple: 2.54251694679
lw keyed tuple: 0.613876104355
-----------------
size=1000000 num=10               # few queries, lots of rows
namedtuple: 5.79669594765         # namedtuple very fast
keyedtuple: 28.856498003          # KeyedTuple falls over
lw keyed tuple: 6.74346804619     # lw keyed trails right on namedtuple
```

[#3176](https://www.sqlalchemy.org/trac/ticket/3176)  ### 结构内存使用显著改进

通过更多内部对象更显著地使用 `__slots__`，改进了结构内存使用。这种优化特别针对具有大量表和列的大型应用程序的基本内存大小，减少了各种高容量对象的内存大小，包括事件监听内部、比较器对象以及 ORM 属性和加载器策略系统的部分。

一张长椅利用堆积测量 Nova 的启动大小，显示出 SQLAlchemy 的对象、相关字典以及弱引用所占空间减少了约 3.7 兆字节，或者说减少了 46%，在基本导入“nova.db.sqlalchemy.models”时：

```py
# reported by heapy, summation of SQLAlchemy objects +
# associated dicts + weakref-related objects with core of Nova imported:

    Before: total count 26477 total bytes 7975712
    After: total count 18181 total bytes 4236456

# reported for the Python module space overall with the
# core of Nova imported:

    Before: Partition of a set of 355558 objects. Total size = 61661760 bytes.
    After: Partition of a set of 346034 objects. Total size = 57808016 bytes.
```  ### UPDATE 语句现在使用 executemany() 进行批处理

现在可以在 ORM 刷新中批处理 UPDATE 语句，以更高效的 executemany() 调用，类似于 INSERT 语句的批处理；这将根据以下标准在刷新中调用：

+   连续两个或更多个 UPDATE 语句涉及相同的要修改的列集。

+   语句在 SET 子句中没有嵌入的 SQL 表达式。

+   映射不使用 `mapper.version_id_col`，或者后端方言支持 executemany() 操作的“合理”行数；大多数 DBAPI 现在都正确支持这一点。  ### Session.get_bind() 处理更广泛的继承场景

每当查询或工作单元刷新过程寻找与特定类对应的数据库引擎时，都会调用 `Session.get_bind()` 方法。该方法已经改进，以处理各种继承导向的场景，包括：

+   绑定到 Mixin 或抽象类：

    ```py
    class MyClass(SomeMixin, Base):
        __tablename__ = "my_table"
        # ...

    session = Session(binds={SomeMixin: some_engine})
    ```

+   根据表单独绑定到继承的具体子类：

    ```py
    class BaseClass(Base):
        __tablename__ = "base"

        # ...

    class ConcreteSubClass(BaseClass):
        __tablename__ = "concrete"

        # ...

        __mapper_args__ = {"concrete": True}

    session = Session(binds={base_table: some_engine, concrete_table: some_other_engine})
    ```

[#3035](https://www.sqlalchemy.org/trac/ticket/3035)  ### 在所有相关的查询情况下，`Session.get_bind()` 将接收到 Mapper

修复了一系列问题，其中 `Session.get_bind()` 不会接收到 `Query` 的主要 `Mapper`，尽管该映射器是 readily available 的（主要映射器是与 `Query` 对象关联的单个映射器，或者是第一个映射器）。

当传递给 `Session.get_bind()` 的 `Mapper` 对象通常由使用 `Session.binds` 参数将映射与一系列引擎关联的会话使用（尽管在这种情况下，大多数情况下事情通常“工作”，因为绑定通常通过映射表对象找到），或更具体地实现一个用户定义的 `Session.get_bind()` 方法，根据映射器选择引擎的某种模式，比如水平分片或所谓的“路由”会话，将查询路由到不同的后端。

这些场景包括：

+   `Query.count()`:

    ```py
    session.query(User).count()
    ```

+   `Query.update()` 和 `Query.delete()`，用于 UPDATE/DELETE 语句以及“fetch”策略所使用的 SELECT：

    ```py
    session.query(User).filter(User.id == 15).update(
        {"name": "foob"}, synchronize_session="fetch"
    )

    session.query(User).filter(User.id == 15).delete(synchronize_session="fetch")
    ```

+   针对单独列的查询：

    ```py
    session.query(User.id, User.name).all()
    ```

+   针对间接映射的 SQL 函数和其他表达式，比如 `column_property`:

    ```py
    class User(Base):
        ...

        score = column_property(func.coalesce(self.tables.users.c.name, None))

    session.query(func.max(User.score)).scalar()
    ```

[#3227](https://www.sqlalchemy.org/trac/ticket/3227) [#3242](https://www.sqlalchemy.org/trac/ticket/3242) [#1326](https://www.sqlalchemy.org/trac/ticket/1326)  ### .info 字典改进

`InspectionAttr.info` 集合现在可以在从`Mapper.all_orm_descriptors` 集合中检索到的每种对象上使用。这包括`hybrid_property` 和 `association_proxy()`。然而，由于这些对象是类绑定描述符，必须**单独**从它们所附加的类中访问以获取属性。以下是使用`Mapper.all_orm_descriptors` 命名空间进行说明：

```py
class SomeObject(Base):
    # ...

    @hybrid_property
    def some_prop(self):
        return self.value + 5

inspect(SomeObject).all_orm_descriptors.some_prop.info["foo"] = "bar"
```

它也作为所有`SchemaItem` 对象（例如`ForeignKey`、`UniqueConstraint`等）的构造函数参数，以及剩余的 ORM 构造，如`synonym()`。

[#2971](https://www.sqlalchemy.org/trac/ticket/2971)

[#2963](https://www.sqlalchemy.org/trac/ticket/2963)  ### ColumnProperty 结构在使用别名、order_by 时效果更好

已修复了关于`column_property()`的各种问题，特别是关于`aliased()`构造以及 0.9 版本中引入的“order by label”逻辑（参见 Label constructs can now render as their name alone in an ORDER BY）。

给定如下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

A.b = column_property(select([func.max(B.id)]).where(B.a_id == A.id).correlate(A))
```

一个简单的包含两次“A.b”的场景将无法正确呈现：

```py
print(sess.query(A, a1).order_by(a1.b))
```

这将按照错误的列排序：

```py
SELECT  a.id  AS  a_id,  (SELECT  max(b.id)  AS  max_1  FROM  b
WHERE  b.a_id  =  a.id)  AS  anon_1,  a_1.id  AS  a_1_id,
(SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a_1.id)  AS  anon_2
FROM  a,  a  AS  a_1  ORDER  BY  anon_1
```

新输出：

```py
SELECT  a.id  AS  a_id,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1,  a_1.id  AS  a_1_id,
(SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a_1.id)  AS  anon_2
FROM  a,  a  AS  a_1  ORDER  BY  anon_2
```

还有许多情况下，“order by”逻辑将无法按照标签排序，例如如果映射是“多态”的情况：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {"polymorphic_on": type, "with_polymorphic": "*"}
```

order_by 将无法使用标签，因为由于多态加载而被匿名化：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1
FROM  a  ORDER  BY  (SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a.id)
```

现在 order by 标签跟踪匿名化标签，这现在可以工作：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1
FROM  a  ORDER  BY  anon_1
```

这些修复中包括了各种可能破坏`aliased()`构造状态的 heisenbugs；这些问题也已经修复。

[#3148](https://www.sqlalchemy.org/trac/ticket/3148) [#3188](https://www.sqlalchemy.org/trac/ticket/3188)

### 新的会话批量插入/更新 API

创建了一系列新的`Session`方法，直接提供钩子进入工作单元的功能，用于发出 INSERT 和 UPDATE 语句。当正确使用时，这个面向专家的系统可以允许 ORM 映射用于生成批量插入和更新语句，分批执行到 executemany 组，使语句以与直接使用 Core 相媲美的速度进行。

另请参阅

批量操作 - 介绍和完整文档

[#3100](https://www.sqlalchemy.org/trac/ticket/3100)

### 新的性能示例套件

受到对批量操作功能以及 FAQ 中的如何对 SQLAlchemy 应用程序进行性能分析？部分进行的基准测试的启发，添加了一个新的示例部分，其中包含几个旨在说明各种 Core 和 ORM 技术的相对性能概况的脚本。这些脚本按用例组织，并打包在一个单一的控制台界面下，以便可以运行任意组合的演示，输出时间、Python 性能分析结果和/或 RunSnake 性能显示。

另请参阅

性能

### “烘焙”查询

“烘焙”查询功能是一种不同寻常的新方法，允许使用缓存直接构建和调用`Query`对象，随着连续调用，Python 函数调用开销大大降低（超过 75%）。通过将`Query`对象指定为一系列仅调用一次的 lambda 表达式，作为预编译单元的查询开始变得可行：

```py
from sqlalchemy.ext import baked
from sqlalchemy import bindparam

bakery = baked.bakery()

def search_for_user(session, username, email=None):
    baked_query = bakery(lambda session: session.query(User))
    baked_query += lambda q: q.filter(User.name == bindparam("username"))

    baked_query += lambda q: q.order_by(User.id)

    if email:
        baked_query += lambda q: q.filter(User.email == bindparam("email"))

    result = baked_query(session).params(username=username, email=email).all()

    return result
```

另请参阅

烘焙查询

[#3054](https://www.sqlalchemy.org/trac/ticket/3054)

### 改进声明性混合类，`@declared_attr`和相关功能

与`declared_attr`结合使用的声明性系统已经进行了全面改进，以支持新的功能。

现在，使用`declared_attr`装饰的函数仅在生成基于混合类的列副本之后才被调用。这意味着该函数可以调用基于混合类建立的列，并将接收到正确的`Column`对象的引用：

```py
class HasFooBar(object):
    foobar = Column(Integer)

    @declared_attr
    def foobar_prop(cls):
        return column_property("foobar: " + cls.foobar)

class SomeClass(HasFooBar, Base):
    __tablename__ = "some_table"
    id = Column(Integer, primary_key=True)
```

在上面的例子中，`SomeClass.foobar_prop` 将针对 `SomeClass` 调用，而 `SomeClass.foobar` 将是最终要映射到 `SomeClass` 的 `Column` 对象，而不是直接存在于 `HasFooBar` 上的非复制对象，即使列尚未映射。

`declared_attr` 函数现在会基于每个类记忆返回的值，以便对相同属性的重复调用会返回相同的值。我们可以修改示例来说明这一点：

```py
class HasFooBar(object):
    @declared_attr
    def foobar(cls):
        return Column(Integer)

    @declared_attr
    def foobar_prop(cls):
        return column_property("foobar: " + cls.foobar)

class SomeClass(HasFooBar, Base):
    __tablename__ = "some_table"
    id = Column(Integer, primary_key=True)
```

以前，`SomeClass` 将以一个特定的 `foobar` 列的副本进行映射，但通过第二次调用 `foobar` 来调用 `foobar_prop` 将产生一个不同的列。在声明式设置期间，`SomeClass.foobar` 的值现在被记忆，因此即使在属性由映射器映射之前，临时列值也将保持一致，无论 `declared_attr` 被调用多少次。

上述两种行为应该极大地帮助声明性定义许多类型的映射器属性，这些属性派生自其他属性，在类实际映射之前从其他 `declared_attr` 函数中调用。

对于一个非常罕见的边缘情况，其中希望建立一个在每个子类中建立不同列的声明性混合类，添加了一个新的修饰符 `declared_attr.cascading`。使用这个修饰符，装饰的函数将分别为映射的继承层次结构中的每个类调用。虽然这已经是诸如 `__table_args__` 和 `__mapper_args__` 等特殊属性的行为，但是对于列和其他属性，默认情况下假定该属性仅附加到基类，并且仅从子类继承。使用 `declared_attr.cascading`，可以应用单独的行为：

```py
class HasIdMixin(object):
    @declared_attr.cascading
    def id(cls):
        if has_inherited_table(cls):
            return Column(ForeignKey("myclass.id"), primary_key=True)
        else:
            return Column(Integer, primary_key=True)

class MyClass(HasIdMixin, Base):
    __tablename__ = "myclass"
    # ...

class MySubClass(MyClass):
  """ """

    # ...
```

另见

使用 _orm.declared_attr() 生成特定于表的继承列

最后，`AbstractConcreteBase` 类已经重新设计，以便在抽象基类上内联设置关系或其他映射器属性：

```py
from sqlalchemy import Column, Integer, ForeignKey
from sqlalchemy.orm import relationship
from sqlalchemy.ext.declarative import (
    declarative_base,
    declared_attr,
    AbstractConcreteBase,
)

Base = declarative_base()

class Something(Base):
    __tablename__ = "something"
    id = Column(Integer, primary_key=True)

class Abstract(AbstractConcreteBase, Base):
    id = Column(Integer, primary_key=True)

    @declared_attr
    def something_id(cls):
        return Column(ForeignKey(Something.id))

    @declared_attr
    def something(cls):
        return relationship(Something)

class Concrete(Abstract):
    __tablename__ = "cca"
    __mapper_args__ = {"polymorphic_identity": "cca", "concrete": True}
```

上述映射将设置一个名为 `cca` 的表，其中包含 `id` 和 `something_id` 列，而 `Concrete` 还将具有一个名为 `something` 的关系。新功能是 `Abstract` 也将有一个独立配置的关系 `something`，该关系构建在基类的多态联合上。

[#3150](https://www.sqlalchemy.org/trac/ticket/3150) [#2670](https://www.sqlalchemy.org/trac/ticket/2670) [#3149](https://www.sqlalchemy.org/trac/ticket/3149) [#2952](https://www.sqlalchemy.org/trac/ticket/2952) [#3050](https://www.sqlalchemy.org/trac/ticket/3050)

### ORM 完整对象的提取速度提高了 25%

`loading.py` 模块的机制以及标识映射经历了几次内联、重构和修剪，因此现在原始行的加载速度比基于 ORM 的对象快约 25%。假设有一个包含 1 百万行的表，下面的脚本演示了最大程度改进的加载类型：

```py
import time
from sqlalchemy import Integer, Column, create_engine, Table
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Foo(Base):
    __table__ = Table(
        "foo",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("a", Integer(), nullable=False),
        Column("b", Integer(), nullable=False),
        Column("c", Integer(), nullable=False),
    )

engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo=True)

sess = Session(engine)

now = time.time()

# avoid using all() so that we don't have the overhead of building
# a large list of full objects in memory
for obj in sess.query(Foo).yield_per(100).limit(1000000):
    pass

print("Total time: %d" % (time.time() - now))
```

本地 MacBookPro 的结果从 0.9 秒降至 1.0 秒的时间为 19 秒降至 14 秒。在批量处理大量行时，`Query.yield_per()` 的调用总是一个好主意，因为它可以防止 Python 解释器一次性为所有对象及其仪器分配大量内存。没有 `Query.yield_per()`，在 MacBookPro 上，0.9 版本上的上述脚本需要 31 秒，1.0 版本上需要 26 秒，额外的时间用于设置非常大的内存缓冲区。

### 新的 KeyedTuple 实现速度显著提高

我们研究了 `KeyedTuple` 实现，希望改进这样的查询：

```py
rows = sess.query(Foo.a, Foo.b, Foo.c).all()
```

使用 `KeyedTuple` 类而不是 Python 的 `collections.namedtuple()`，因为后者具有一个非常复杂的类型创建程序，其性能比 `KeyedTuple` 慢得多。然而，当提取数十万行时，`collections.namedtuple()` 很快就会超过 `KeyedTuple`，随着实例调用次数的增加，`KeyedTuple` 的性能会急剧下降。怎么办？一个新类型，介于两者之间的方法。对于“大小”（返回的行数）和“num”（不同查询的数量）对所有三种类型进行测试，新的“轻量级键值元组”要么优于两者，要么略逊于更快的对象，具体取决于情况。在“甜蜜点”，我们既创建了大量新类型，又提取了大量行时，轻量级对象完全超越了 namedtuple 和 KeyedTuple：

```py
-----------------
size=10 num=10000                 # few rows, lots of queries
namedtuple: 3.60302400589         # namedtuple falls over
keyedtuple: 0.255059957504        # KeyedTuple very fast
lw keyed tuple: 0.582715034485    # lw keyed trails right on KeyedTuple
-----------------
size=100 num=1000                 # <--- sweet spot
namedtuple: 0.365247011185
keyedtuple: 0.24896979332
lw keyed tuple: 0.0889317989349   # lw keyed blows both away!
-----------------
size=10000 num=100
namedtuple: 0.572599887848
keyedtuple: 2.54251694679
lw keyed tuple: 0.613876104355
-----------------
size=1000000 num=10               # few queries, lots of rows
namedtuple: 5.79669594765         # namedtuple very fast
keyedtuple: 28.856498003          # KeyedTuple falls over
lw keyed tuple: 6.74346804619     # lw keyed trails right on namedtuple
```

[#3176](https://www.sqlalchemy.org/trac/ticket/3176)

### 结构内存使用方面的显著改进

通过更多内部对象的`__slots__`的更显著使用改进了结构性内存使用。这种优化特别针对具有大量表和列的大型应用程序的基本内存大小，并减少了各种高容量对象的内存大小，包括事件监听内部、比较器对象以及 ORM 属性和加载器策略系统的部分。

一个利用 heapy 测量 Nova 启动大小的工作台展示了 SQLAlchemy 对象、相关字典以及弱引用在“nova.db.sqlalchemy.models”基本导入中占用的空间约减少了 3.7 兆字节，或 46%：

```py
# reported by heapy, summation of SQLAlchemy objects +
# associated dicts + weakref-related objects with core of Nova imported:

    Before: total count 26477 total bytes 7975712
    After: total count 18181 total bytes 4236456

# reported for the Python module space overall with the
# core of Nova imported:

    Before: Partition of a set of 355558 objects. Total size = 61661760 bytes.
    After: Partition of a set of 346034 objects. Total size = 57808016 bytes.
```

### UPDATE 语句现在在 flush 中与 executemany()批处理

现在可以将 UPDATE 语句批量处理到 ORM flush 中，以更高效的 executemany()调用，类似于 INSERT 语句可以批量处理；这将根据以下标准在 flush 中调用：

+   连续两个或更多的 UPDATE 语句涉及相同的要修改的列集。

+   该语句在 SET 子句中没有嵌入 SQL 表达式。

+   映射不使用`mapper.version_id_col`，或者后端方言支持对 executemany()操作的“合理”行数计数；现在大多数 DBAPI 都正确支持这一点。

### Session.get_bind()处理更广泛的继承场景

每当查询或工作单元 flush 过程试图定位与特定类对应的数据库引擎时，都会调用`Session.get_bind()`方法。该方法已经改进，以处理各种基于继承的场景，包括：

+   绑定到 Mixin 或抽象类：

    ```py
    class MyClass(SomeMixin, Base):
        __tablename__ = "my_table"
        # ...

    session = Session(binds={SomeMixin: some_engine})
    ```

+   根据表单独绑定到继承的具体子类：

    ```py
    class BaseClass(Base):
        __tablename__ = "base"

        # ...

    class ConcreteSubClass(BaseClass):
        __tablename__ = "concrete"

        # ...

        __mapper_args__ = {"concrete": True}

    session = Session(binds={base_table: some_engine, concrete_table: some_other_engine})
    ```

[#3035](https://www.sqlalchemy.org/trac/ticket/3035)

### Session.get_bind()将在所有相关的 Query 情况下接收到 Mapper

修复了一系列问题，其中`Session.get_bind()`不会接收到`Query`的主要`Mapper`，即使该映射器是 readily available 的（主要映射器是与`Query`对象关联的单个映射器，或者是第一个映射器）。

当 `Mapper` 对象传递给 `Session.get_bind()` 时，通常由使用 `Session.binds` 参数关联映射器与一系列引擎的会话使用，或更具体地实现一个用户定义的 `Session.get_bind()` 方法，根据映射器提供一种基于模式选择引擎的方式，例如水平分片或所谓的“路由”会话，将查询路由到不同的后端。

这些场景包括：

+   `Query.count()`:

    ```py
    session.query(User).count()
    ```

+   `Query.update()` 和 `Query.delete()`，都用于 UPDATE/DELETE 语句以及“fetch”策略中使用的 SELECT：

    ```py
    session.query(User).filter(User.id == 15).update(
        {"name": "foob"}, synchronize_session="fetch"
    )

    session.query(User).filter(User.id == 15).delete(synchronize_session="fetch")
    ```

+   针对单个列的查询：

    ```py
    session.query(User.id, User.name).all()
    ```

+   针对间接映射的 SQL 函数和其他表达式，例如 `column_property`：

    ```py
    class User(Base):
        ...

        score = column_property(func.coalesce(self.tables.users.c.name, None))

    session.query(func.max(User.score)).scalar()
    ```

[#3227](https://www.sqlalchemy.org/trac/ticket/3227) [#3242](https://www.sqlalchemy.org/trac/ticket/3242) [#1326](https://www.sqlalchemy.org/trac/ticket/1326)

### .info 字典改进

`InspectionAttr.info` 集合现在可以在从 `Mapper.all_orm_descriptors` 集合中检索的任何类型的对象上使用。这包括 `hybrid_property` 和 `association_proxy()`。然而，由于这些对象是类绑定描述符，必须**单独**从附加到的类中访问它们，以便访问属性。下面使用 `Mapper.all_orm_descriptors` 命名空间进行说明：

```py
class SomeObject(Base):
    # ...

    @hybrid_property
    def some_prop(self):
        return self.value + 5

inspect(SomeObject).all_orm_descriptors.some_prop.info["foo"] = "bar"
```

它也作为所有`SchemaItem`对象（例如`ForeignKey`、`UniqueConstraint`等）的构造函数参数可用，以及剩余的 ORM 构造，如`synonym()`。

[#2971](https://www.sqlalchemy.org/trac/ticket/2971)

[#2963](https://www.sqlalchemy.org/trac/ticket/2963)

### ColumnProperty 构造与别名、order_by 更好地配合

解决了关于`column_property()`的各种问题，特别是关于 0.9 版本引入的关于“order by label”逻辑的问题（参见 Label constructs can now render as their name alone in an ORDER BY）。

给定以下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

A.b = column_property(select([func.max(B.id)]).where(B.a_id == A.id).correlate(A))
```

一个简单的场景，包含两次“A.b”将无法正确呈现：

```py
print(sess.query(A, a1).order_by(a1.b))
```

这将按错误的列排序：

```py
SELECT  a.id  AS  a_id,  (SELECT  max(b.id)  AS  max_1  FROM  b
WHERE  b.a_id  =  a.id)  AS  anon_1,  a_1.id  AS  a_1_id,
(SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a_1.id)  AS  anon_2
FROM  a,  a  AS  a_1  ORDER  BY  anon_1
```

新输出：

```py
SELECT  a.id  AS  a_id,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1,  a_1.id  AS  a_1_id,
(SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a_1.id)  AS  anon_2
FROM  a,  a  AS  a_1  ORDER  BY  anon_2
```

在许多情况下，“order by”逻辑无法按标签排序，例如如果映射是“多态”的：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {"polymorphic_on": type, "with_polymorphic": "*"}
```

由于多态加载，order_by 将无法使用标签，因为它将被匿名化：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1
FROM  a  ORDER  BY  (SELECT  max(b.id)  AS  max_2
FROM  b  WHERE  b.a_id  =  a.id)
```

现在，由于 order by 标签跟踪了匿名化标签，这现在可以工作：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type,  (SELECT  max(b.id)  AS  max_1
FROM  b  WHERE  b.a_id  =  a.id)  AS  anon_1
FROM  a  ORDER  BY  anon_1
```

这些修复中包括了一系列可能会破坏`aliased()`构造状态的 heisenbugs，使标签逻辑再次失败；这些问题也已经修复。

[#3148](https://www.sqlalchemy.org/trac/ticket/3148) [#3188](https://www.sqlalchemy.org/trac/ticket/3188)

## 新功能和改进 - 核心

### Select/Query LIMIT / OFFSET 可以指定为任意 SQL 表达式

`Select.limit()`和`Select.offset()`方法现在接受任何 SQL 表达式作为参数，而不仅仅是整数值。ORM `Query`对象也将任何表达式传递给底层的`Select`对象。通常用于允许传递绑定参数，稍后可以用值替换：

```py
sel = select([table]).limit(bindparam("mylimit")).offset(bindparam("myoffset"))
```

不支持非整数 LIMIT 或 OFFSET 表达式的方言可能继续不支持此行为；第三方方言可能还需要修改以利用新行为。当前使用 `._limit` 或 `._offset` 属性的方言将继续为那些限制/偏移指定为简单整数值的情况下运行。然而，当指定 SQL 表达式时，这两个属性将在访问时引发 `CompileError`。希望支持新功能的第三方方言现在应调用 `._limit_clause` 和 `._offset_clause` 属性以接收完整的 SQL 表达式，而不是整数值。### `ForeignKeyConstraint` 上的 `use_alter` 标志（通常）不再需要。

`MetaData.create_all()` 和 `MetaData.drop_all()` 方法现在将使用一个系统，自动为涉及表之间相互依赖循环的外键约束渲染 ALTER 语句，无需指定 `ForeignKeyConstraint.use_alter`。此外，外键约束现在不再需要名称即可通过 ALTER 创建；只有 DROP 操作需要名称。在 DROP 的情况下，该功能将确保只有具有显式名称的约束实际上包含在 ALTER 语句中。在 DROP 中存在无法解决的循环的情况下，如果无法继续执行 DROP，系统现在会发出简洁明了的错误消息。

`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 标志仍然存在，并且继续具有相同的效果，在 CREATE/DROP 场景中建立需要 ALTER 的约束。

从版本 1.0.1 开始，针对 SQLite 的特殊逻辑接管了 ALTER 的情况，在 DROP 期间，如果给定的表存在不可解析的循环，将发出警告，并且这些表将以 **无** 排序的方式删除，这在 SQLite 上通常是可以接受的，除非启用了约束。为了解决警告并在 SQLite 数据库上至少进行部分排序，特别是在启用了约束的数据库上，请重新将“use_alter”标志应用于那些应该明确排除的 `ForeignKey` 和 `ForeignKeyConstraint` 对象。

另请参阅

通过 ALTER 创建/删除外键约束 - 新行为的完整描述。

[#3282](https://www.sqlalchemy.org/trac/ticket/3282)  ### ResultProxy “auto close” 现在是 “soft” close

在许多版本中，`ResultProxy` 对象总是在获取所有结果行后自动关闭。这是为了允许在不需要显式调用 `ResultProxy.close()` 的情况下使用该对象；由于所有的 DBAPI 资源都已被释放，因此可以安全地丢弃该对象。但是，该对象保持了严格的“关闭”行为，这意味着对 `ResultProxy.fetchone()`、`ResultProxy.fetchmany()` 或 `ResultProxy.fetchall()` 的任何后续调用现在都将引发 `ResourceClosedError`：

```py
>>> result = connection.execute(stmt)
>>> result.fetchone()
(1, 'x')
>>> result.fetchone()
None  # indicates no more rows
>>> result.fetchone()
exception: ResourceClosedError
```

这种行为与 pep-249 规定的不一致，即使结果耗尽后仍然可以重复调用 fetch 方法。这也会干扰某些实现结果代理的行为，例如某些数据类型的 cx_oracle 方言所使用的 `BufferedColumnResultProxy`。

为了解决这个问题，`ResultProxy`的“closed”状态被分解为两个状态；一个“软关闭”执行了“关闭”的大部分功能，释放了 DBAPI 游标，并且在“带有结果的关闭”对象的情况下还会释放连接，并且一个“关闭”状态包括“软关闭”所包含的一切以及将提取方法设为“关闭”。`ResultProxy.close()` 方法现在不会被隐式调用，只会调用 `ResultProxy._soft_close()` 方法，该方法是非公开的：

```py
>>> result = connection.execute(stmt)
>>> result.fetchone()
(1, 'x')
>>> result.fetchone()
None  # indicates no more rows
>>> result.fetchone()
None  # still None
>>> result.fetchall()
[]
>>> result.close()
>>> result.fetchone()
exception: ResourceClosedError  # *now* it raises
```

[#3330](https://www.sqlalchemy.org/trac/ticket/3330) [#3329](https://www.sqlalchemy.org/trac/ticket/3329)

### CHECK 约束现在支持命名约定中的 `%(column_0_name)s` 占位符。

`%(column_0_name)s` 将派生自 `CheckConstraint` 表达式中找到的第一列：

```py
metadata = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table("foo", metadata, Column("value", Integer))

CheckConstraint(foo.c.value > 5)
```

将呈现：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value  CHECK  (value  >  5)
)
```

与由`SchemaType`生成的约束的命名约定的组合，例如`Boolean`或`Enum`现在也将使用所有 CHECK 约束约定。

另请参阅

命名 CHECK 约束

配置布尔值、枚举和其他模式类型的命名

[#3299](https://www.sqlalchemy.org/trac/ticket/3299)

### 当约束引用未附加的列时，可以在其引用的列附加到表时自动附加约束

自版本 0.8 起，`Constraint`至少具有根据传递的与表附加的列“自动附加”到`Table`的能力：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

t = Table("t", m, Column("a", Integer), Column("b", Integer))

uq = UniqueConstraint(t.c.a, t.c.b)  # will auto-attach to Table

assert uq in t.constraints
```

为了帮助一些在声明时经常出现的情况，即使`Column`对象尚未与`Table`关联，此自动附加逻辑现在也可以起作用；建立了额外的事件，以便当这些`Column`对象关联时，也会添加`Constraint`:

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

uq = UniqueConstraint(a, b)

t = Table("t", m, a, b)

assert uq in t.constraints  # constraint auto-attached
```

以上功能是版本 1.0.0b3 之后的晚期添加的。从版本 1.0.4 开始修复了[#3411](https://www.sqlalchemy.org/trac/ticket/3411)，以确保如果`Constraint`引用混合了`Column`对象和字符串列名，则不会发生此逻辑；因为我们尚未跟踪将名称添加到`Table`的情况：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

uq = UniqueConstraint(a, "b")

t = Table("t", m, a, b)

# constraint *not* auto-attached, as we do not have tracking
# to locate when a name 'b' becomes available on the table
assert uq not in t.constraints
```

在上面，对于列“a”到表“t”的附加事件将在列“b”被附加之前触发（因为“a”在`Table`构造函数中在“b”之前声明），如果尝试进行附加，则约束将无法定位“b”。为了保持一致，如果约束涉及任何字符串名称，则会跳过自动在列附加时附加的逻辑。

当 `Constraint` 构造时，如果 `Table` 已经包含所有目标 `Column` 对象，则原始的自动附加逻辑仍然存在：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

t = Table("t", m, a, b)

uq = UniqueConstraint(a, "b")

# constraint auto-attached normally as in older versions
assert uq in t.constraints
```

[#3341](https://www.sqlalchemy.org/trac/ticket/3341) [#3411](https://www.sqlalchemy.org/trac/ticket/3411)  ### INSERT FROM SELECT 现在包括 Python 和 SQL 表达式默认值

如果未另行指定，`Insert.from_select()` 现在包括 Python 和 SQL 表达式默认值；解除了非服务器列默认值不包括在 INSERT FROM SELECT 中的限制，这些表达式被渲染为常量插入到 SELECT 语句中：

```py
from sqlalchemy import Table, Column, MetaData, Integer, select, func

m = MetaData()

t = Table(
    "t", m, Column("x", Integer), Column("y", Integer, default=func.somefunction())
)

stmt = select([t.c.x])
print(t.insert().from_select(["x"], stmt))
```

将呈现：

```py
INSERT  INTO  t  (x,  y)  SELECT  t.x,  somefunction()  AS  somefunction_1
FROM  t
```

可以使用 `Insert.from_select.include_defaults` 来禁用此功能。  ### Column 服务器默认值现在呈现为字面值

当由 `Column.server_default` 设置的 `DefaultClause` 存在作为要编译的 SQL 表达式时，“literal binds” 编译器标志会被打开。这允许嵌入在 SQL 中的字面值正确渲染，例如：

```py
from sqlalchemy import Table, Column, MetaData, Text
from sqlalchemy.schema import CreateTable
from sqlalchemy.dialects.postgresql import ARRAY, array
from sqlalchemy.dialects import postgresql

metadata = MetaData()

tbl = Table(
    "derp",
    metadata,
    Column("arr", ARRAY(Text), server_default=array(["foo", "bar", "baz"])),
)

print(CreateTable(tbl).compile(dialect=postgresql.dialect()))
```

现在呈现为：

```py
CREATE  TABLE  derp  (
  arr  TEXT[]  DEFAULT  ARRAY['foo',  'bar',  'baz']
)
```

以前，字面值 `"foo", "bar", "baz"` 会被渲染为绑定参数，在 DDL 中毫无用处。

[#3087](https://www.sqlalchemy.org/trac/ticket/3087)  ### UniqueConstraint 现在是表反射过程的一部分

使用 `autoload=True` 填充的 `Table` 对象现在也会包括 `UniqueConstraint` 构造以及 `Index` 构造。这个逻辑在 PostgreSQL 和 MySQL 中有一些注意事项：

#### PostgreSQL

PostgreSQL 在创建唯一约束时会隐式创建对应的唯一索引。`Inspector.get_indexes()`和`Inspector.get_unique_constraints()`方法将继续**分别**返回这些条目，其中`Inspector.get_indexes()`现在在索引条目中包含一个`duplicates_constraint`标记，指示检测到的相应约束。然而，在使用`Table(..., autoload=True)`进行完整表反射时，检测到`Index`构造与`UniqueConstraint`相关联，并且**不**会出现在`Table.indexes`集合中；只有`UniqueConstraint`会出现在`Table.constraints`集合中。这种去重逻辑通过在查询`pg_index`时连接到`pg_constraint`表来查看这两个构造是否相关联。

#### MySQL

MySQL 没有唯一索引和唯一约束的单独概念。虽然它在创建表和索引时都支持两种语法，但在存储时没有任何区别。`Inspector.get_indexes()`和`Inspector.get_unique_constraints()`方法将**同时**返回 MySQL 中唯一索引的条目，其中`Inspector.get_unique_constraints()`在约束条目中包含一个新的标记 `duplicates_index`，表示这是与该索引对应的重复条目。但是，在使用`Table(..., autoload=True)`执行完整表反射时，`UniqueConstraint`构造**不**会在任何情况下成为完全反射的`Table`构造的一部分；该构造始终由`Index`表示，并且在`Table.indexes`集合中存在`unique=True`设置。

另请参阅

PostgreSQL 索引反射

MySQL / MariaDB 唯一约束和反射

[#3184](https://www.sqlalchemy.org/trac/ticket/3184)

### 新的系统以安全方式发出参数化警告

长期以来，存在着一个限制，即警告消息不能引用数据元素，因此特定函数可能会发出无限数量的唯一警告。这种情况最常见的地方是在`Unicode 类型接收到非 Unicode 绑定参数值`警告中。将数据值放入此消息中意味着该模块的 Python `__warningregistry__`，或在某些情况下是 Python 全局的 `warnings.onceregistry`，会无限增长，因为在大多数警告情况下，这两个集合中的一个会填充每个不同的警告消息。

此处的更改是通过使用一个特殊的`string`类型，故意更改字符串的哈希方式，我们可以控制大量参数化消息仅在一小组可能的哈希值上进行哈希，使得像`Unicode 类型接收到非 Unicode 绑定参数值`这样的警告可以被定制为仅发出特定次数；超出此次数，Python 警告注册表将开始记录它们作为重复项。

举例来说，以下测试脚本将仅对一千个参数集中的十个发出警告：

```py
from sqlalchemy import create_engine, Unicode, select, cast
import random
import warnings

e = create_engine("sqlite://")

# Use the "once" filter (which is also the default for Python
# warnings).  Exactly ten of these warnings will
# be emitted; beyond that, the Python warnings registry will accumulate
# new values as dupes of one of the ten existing.
warnings.filterwarnings("once")

for i in range(1000):
    e.execute(
        select([cast(("foo_%d" % random.randint(0, 1000000)).encode("ascii"), Unicode)])
    )
```

这里的警告格式为：

```py
/path/lib/sqlalchemy/sql/sqltypes.py:186: SAWarning: Unicode type received
  non-unicode bind param value 'foo_4852'. (this warning may be
  suppressed after 10 occurrences)
```

[#3178](https://www.sqlalchemy.org/trac/ticket/3178)

### Select/Query LIMIT / OFFSET 可以指定为任意 SQL 表达式

`Select.limit()` 和 `Select.offset()` 方法现在接受任何 SQL 表达式作为参数，而不仅仅是整数值。ORM `Query` 对象也会将任何表达式传递给底层的 `Select` 对象。通常用于允许传递绑定参数，稍后可以用值替换：

```py
sel = select([table]).limit(bindparam("mylimit")).offset(bindparam("myoffset"))
```

不支持非整数 LIMIT 或 OFFSET 表达式的方言可能继续不支持此行为；第三方方言可能还需要修改以利用新行为。当前使用 `._limit` 或 `._offset` 属性的方言将继续对指定为简单整数值的限制/偏移量的情况进行处理。但是，当指定 SQL 表达式时，这两个属性将在访问时引发 `CompileError`。希望支持新功能的第三方方言现在应调用 `._limit_clause` 和 `._offset_clause` 属性以接收完整的 SQL 表达式，而不是整数值。

### `ForeignKeyConstraint` 上的 `use_alter` 标志（通常）不再需要

`MetaData.create_all()` 和 `MetaData.drop_all()` 方法现在将使用一个系统，自动为涉及表之间相互依赖循环的外键约束渲染 ALTER 语句，无需指定 `ForeignKeyConstraint.use_alter`。此外，外键约束现在不再需要名称即可通过 ALTER 创建；仅在 DROP 操作时需要名称。在 DROP 的情况下，该功能将确保只有具有显式名称的约束实际上包含在 ALTER 语句中。在 DROP 中存在无法解决的循环的情况下，如果无法继续进行 DROP，系统现在会发出简洁明了的错误消息。

`ForeignKeyConstraint.use_alter` 和 `ForeignKey.use_alter` 标志保持不变，并且继续具有相同的效果，即在 CREATE/DROP 情景中需要 ALTER 来建立这些约束条件。

自版本 1.0.1 起，在 SQLite 的情况下，特殊逻辑会接管，SQLite 不支持 ALTER，在 DROP 过程中，如果给定的表存在无法解析的循环，则会发出警告，并且这些表将无序删除，这在 SQLite 上通常没问题，除非启用了约束条件。要解决警告并在 SQLite 数据库上至少实现部分排序，特别是在启用约束条件的数据库中，重新将 “use_alter” 标志应用于那些应该在排序中显式省略的 `ForeignKey` 和 `ForeignKeyConstraint` 对象。

另见

通过 ALTER 创建/删除外键约束 - 新行为的完整描述。

[#3282](https://www.sqlalchemy.org/trac/ticket/3282)

### ResultProxy “auto close” 现在是 “soft” close

在很多版本中，`ResultProxy` 对象一直会在获取所有结果行后自动关闭。这是为了允许在不需要显式调用 `ResultProxy.close()` 的情况下使用该对象；由于所有的 DBAPI 资源都已被释放，因此可以安全地丢弃该对象。然而，该对象仍保持严格的“closed”行为，这意味着任何后续对 `ResultProxy.fetchone()`、`ResultProxy.fetchmany()` 或 `ResultProxy.fetchall()` 的调用都会引发 `ResourceClosedError`：

```py
>>> result = connection.execute(stmt)
>>> result.fetchone()
(1, 'x')
>>> result.fetchone()
None  # indicates no more rows
>>> result.fetchone()
exception: ResourceClosedError
```

此行为与 pep-249 的规定不一致，pep-249 规定即使结果已经耗尽，仍然可以重复调用 fetch 方法。这也会影响到某些结果代理的实现行为，比如 cx_oracle 方言中某些数据类型使用的 `BufferedColumnResultProxy`。

为了解决这个问题，`ResultProxy`的“closed”状态已被分为两个状态；“软关闭”执行了“关闭”的大部分操作，即释放了 DBAPI 游标，并且在 “close with result” 对象的情况下还会释放连接，并且“closed”状态包括了“soft close”中的所有内容，同时还将 fetch 方法设为“closed”。现在永远不会隐式调用 `ResultProxy.close()` 方法，只会调用非公开的 `ResultProxy._soft_close()` 方法：

```py
>>> result = connection.execute(stmt)
>>> result.fetchone()
(1, 'x')
>>> result.fetchone()
None  # indicates no more rows
>>> result.fetchone()
None  # still None
>>> result.fetchall()
[]
>>> result.close()
>>> result.fetchone()
exception: ResourceClosedError  # *now* it raises
```

[#3330](https://www.sqlalchemy.org/trac/ticket/3330) [#3329](https://www.sqlalchemy.org/trac/ticket/3329)

### CHECK Constraints 现在支持命名约定中的`%(column_0_name)s`标记

`%(column_0_name)s`将从`CheckConstraint`的表达式中找到的第一列派生：

```py
metadata = MetaData(naming_convention={"ck": "ck_%(table_name)s_%(column_0_name)s"})

foo = Table("foo", metadata, Column("value", Integer))

CheckConstraint(foo.c.value > 5)
```

将呈现为：

```py
CREATE  TABLE  foo  (
  value  INTEGER,
  CONSTRAINT  ck_foo_value  CHECK  (value  >  5)
)
```

命名约定与由`SchemaType`（如`Boolean`或`Enum`）产生的约束的组合现在也将使用所有 CHECK 约束约定。

另见

命名 CHECK 约束

为布尔值、枚举和其他模式类型配置命名

[#3299](https://www.sqlalchemy.org/trac/ticket/3299)

### 当其引用的列附加时，引用未附加的列的约束可以自动附加到表上

至少从版本 0.8 开始，`Constraint`已经能够根据传递的表附加列自动“附加”到`Table`上：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

t = Table("t", m, Column("a", Integer), Column("b", Integer))

uq = UniqueConstraint(t.c.a, t.c.b)  # will auto-attach to Table

assert uq in t.constraints
```

为了帮助处理一些在声明时经常出现的情况，即使`Column`对象尚未与`Table`关联，此相同的自动附加逻辑现在也可以起作用；额外的事件被建立，以便当这些`Column`对象关联时，也添加了`Constraint`：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

uq = UniqueConstraint(a, b)

t = Table("t", m, a, b)

assert uq in t.constraints  # constraint auto-attached
```

上述功能是截至版本 1.0.0b3 的最后添加的。从版本 1.0.4 开始对[#3411](https://www.sqlalchemy.org/trac/ticket/3411)的修复确保如果`Constraint`引用了`Column`对象和字符串列名称的混合；因为我们尚未跟踪将名称添加到`Table`中：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

uq = UniqueConstraint(a, "b")

t = Table("t", m, a, b)

# constraint *not* auto-attached, as we do not have tracking
# to locate when a name 'b' becomes available on the table
assert uq not in t.constraints
```

在上面的示例中，将列“a”附加到表“t”的附加事件将在附加列“b”之前触发（因为“a”在构造`Table`时在“b”之前声明），如果约束尝试附加时无法找到“b”，约束将失败。为了保持一致性，如果约束引用任何字符串名称，则跳过在列附加时自动附加的逻辑。

当`Constraint`构造时，如果`Table`已经包含所有目标`Column`对象，则原始的自动附加逻辑当然仍然存在：

```py
from sqlalchemy import Table, Column, MetaData, Integer, UniqueConstraint

m = MetaData()

a = Column("a", Integer)
b = Column("b", Integer)

t = Table("t", m, a, b)

uq = UniqueConstraint(a, "b")

# constraint auto-attached normally as in older versions
assert uq in t.constraints
```

[#3341](https://www.sqlalchemy.org/trac/ticket/3341) [#3411](https://www.sqlalchemy.org/trac/ticket/3411)

### INSERT FROM SELECT 现在包括 Python 和 SQL 表达式默认值

如果未另行指定，则`Insert.from_select()`现在将包括 Python 和 SQL 表达式默认值；现在解除了非服务器列默认值不包括在 INSERT FROM SELECT 中的限制，并将这些表达式作为常量呈现到 SELECT 语句中：

```py
from sqlalchemy import Table, Column, MetaData, Integer, select, func

m = MetaData()

t = Table(
    "t", m, Column("x", Integer), Column("y", Integer, default=func.somefunction())
)

stmt = select([t.c.x])
print(t.insert().from_select(["x"], stmt))
```

将呈现：

```py
INSERT  INTO  t  (x,  y)  SELECT  t.x,  somefunction()  AS  somefunction_1
FROM  t
```

可以使用`Insert.from_select.include_defaults`来禁用此功能。

### Column 服务器默认值现在呈现为字面值

当由`Column.server_default`设置的`DefaultClause`作为要编译的 SQL 表达式存在时，“字面绑定”编译器标志将被打开。这允许嵌入在 SQL 中的字面值正确呈现，例如：

```py
from sqlalchemy import Table, Column, MetaData, Text
from sqlalchemy.schema import CreateTable
from sqlalchemy.dialects.postgresql import ARRAY, array
from sqlalchemy.dialects import postgresql

metadata = MetaData()

tbl = Table(
    "derp",
    metadata,
    Column("arr", ARRAY(Text), server_default=array(["foo", "bar", "baz"])),
)

print(CreateTable(tbl).compile(dialect=postgresql.dialect()))
```

现在呈现：

```py
CREATE  TABLE  derp  (
  arr  TEXT[]  DEFAULT  ARRAY['foo',  'bar',  'baz']
)
```

以前，字面值`"foo", "bar", "baz"`会呈现为绑定参数，在 DDL 中毫无用处。

[#3087](https://www.sqlalchemy.org/trac/ticket/3087)

### UniqueConstraint 现在是 Table 反射过程的一部分

使用`autoload=True`填充的`Table`对象现在将包括`UniqueConstraint`构造以及`Index`构造。对于 PostgreSQL 和 MySQL，这种逻辑有一些注意事项：

#### PostgreSQL

PostgreSQL 的行为是，当创建一个唯一约束时，它会隐式地创建一个对应该约束的唯一索引。`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法将继续**分别**返回这些条目，其中 `Inspector.get_indexes()` 现在在索引条目中包含一个 `duplicates_constraint` 标记，指示检测到的相应约束。然而，在使用 `Table(..., autoload=True)` 进行完整表反射时，检测到 `Index` 构造与 `UniqueConstraint` 相关联，并且**不**出现在 `Table.indexes` 集合中；只有 `UniqueConstraint` 将出现在 `Table.constraints` 集合中。这种去重逻辑通过在查询 `pg_index` 时连接到 `pg_constraint` 表来查看这两个构造是否关联。

#### MySQL

MySQL 没有单独的概念来区分唯一索引和唯一约束。虽然在创建表和索引时都支持两种语法，但在存储时并没有任何区别。`Inspector.get_indexes()`和`Inspector.get_unique_constraints()`方法将继续**同时**返回 MySQL 中唯一索引的条目，其中`Inspector.get_unique_constraints()`在约束条目中使用新标记`duplicates_index`指示这是对应于该索引的重复条目。然而，在使用`Table(..., autoload=True)`执行完整表反射时，`UniqueConstraint`构造在任何情况下都**不**是完全反映的`Table`构造的一部分；这个构造始终由在`Table.indexes`集合中存在`unique=True`设置的`Index`表示。

另请参阅

PostgreSQL 索引反射

MySQL / MariaDB 唯一约束和反射

[#3184](https://www.sqlalchemy.org/trac/ticket/3184)

#### PostgreSQL

当创建唯一约束时，PostgreSQL 的行为是隐式创建与该约束对应的唯一索引。`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法将继续**分别**返回这些条目，其中 `Inspector.get_indexes()` 现在在索引条目中特征化了一个 `duplicates_constraint` 标记，表示当检测到相应约束时。然而，在使用 `Table(..., autoload=True)` 进行完整表反射时，`Index` 结构被检测为与 `UniqueConstraint` 相关联，并且**不**会出现在 `Table.indexes` 集合中；只有 `UniqueConstraint` 会出现在 `Table.constraints` 集合中。这个去重逻辑通过在查询 `pg_index` 时连接到 `pg_constraint` 表来查看这两个结构是否相关联。

#### MySQL

MySQL 没有单独的概念来区分**唯一索引**和**唯一约束**。虽然在创建表和索引时支持两种语法，但在存储时并没有任何区别。`Inspector.get_indexes()`和`Inspector.get_unique_constraints()`方法将继续**同时**返回 MySQL 中唯一索引的条目，其中`Inspector.get_unique_constraints()`在约束条目中具有一个新的标记`duplicates_index`，表示这是对应该索引的重复条目。然而，在使用`Table(..., autoload=True)`执行完整表反射时，`UniqueConstraint`构造在任何情况下都**不**是完全反映的`Table`构造的一部分；这个构造始终由在`Table.indexes`集合中存在`unique=True`设置的`Index`表示。

另请参阅

PostgreSQL 索引反射

MySQL / MariaDB 唯一约束和反射

[#3184](https://www.sqlalchemy.org/trac/ticket/3184)

### 安全发出参数化警告的新系统

长期以来，存在一个限制，即警告消息不能引用数据元素，这样一个特定函数可能会发出无限数量的唯一警告。这种情况最常见的地方是在`Unicode type received non-unicode bind param value`警告中。将数据值放入此消息中意味着该模块的 Python `__warningregistry__`，或在某些情况下是 Python 全局的`warnings.onceregistry`，将无限增长，因为在大多数警告场景中，这两个集合中的一个会填充每个不同的警告消息。

这里的变化是通过使用一种特殊的`string`类型，故意改变字符串的哈希方式，我们可以控制大量参数化消息仅在一小组可能的哈希值上进行哈希，这样一个警告，比如`Unicode type received non-unicode bind param value`，可以被定制为仅发出特定次数；在那之后，Python 警告注册表将开始记录它们作为重复项。

为了说明，以下测试脚本将仅显示对于 1000 个参数集中的十个参数集发出的十个警告：

```py
from sqlalchemy import create_engine, Unicode, select, cast
import random
import warnings

e = create_engine("sqlite://")

# Use the "once" filter (which is also the default for Python
# warnings).  Exactly ten of these warnings will
# be emitted; beyond that, the Python warnings registry will accumulate
# new values as dupes of one of the ten existing.
warnings.filterwarnings("once")

for i in range(1000):
    e.execute(
        select([cast(("foo_%d" % random.randint(0, 1000000)).encode("ascii"), Unicode)])
    )
```

这里的警告格式是：

```py
/path/lib/sqlalchemy/sql/sqltypes.py:186: SAWarning: Unicode type received
  non-unicode bind param value 'foo_4852'. (this warning may be
  suppressed after 10 occurrences)
```

[#3178](https://www.sqlalchemy.org/trac/ticket/3178)

## 关键行为变化 - ORM

### query.update()现在将字符串名称解析为映射属性名称

`Query.update()`的文档说明给定的`values`字典是“以属性名称为键的字典”，这意味着这些是映射的属性名称。不幸的是，该函数更多地是设计为接收属性和 SQL 表达式，而不是字符串；当传递字符串时，这些字符串将直接传递到核心更新语句，而不解析这些名称在映射类上的表示方式，这意味着名称必须与表列的名称完全匹配，而不是该名称被映射到类的属性上的方式。

现在字符串名称会认真解析为属性名称：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column("user_name", String(50))
```

上面，列`user_name`被映射为`name`。以前，传递字符串的`Query.update()`调用必须如下调用：

```py
session.query(User).update({"user_name": "moonbeam"})
```

现在给定的字符串将根据实体解析：

```py
session.query(User).update({"name": "moonbeam"})
```

通常最好直接使用属性，以避免任何歧义：

```py
session.query(User).update({User.name: "moonbeam"})
```

此更改还表明，同义词和混合属性也可以通过字符串名称引用：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column("user_name", String(50))

    @hybrid_property
    def fullname(self):
        return self.name

session.query(User).update({"fullname": "moonbeam"})
```

[#3228](https://www.sqlalchemy.org/trac/ticket/3228)  ### 当将对象与 None 值比较时发出警告

这个更改是从 1.0.1 版本开始的。一些用户正在执行基本上是这种形式的查询：

```py
session.query(Address).filter(Address.user == User(id=None))
```

SQLAlchemy 当前不支持这种模式。对于所有版本，它生成类似于以下的 SQL：

```py
SELECT  address.id  AS  address_id,  address.user_id  AS  address_user_id,
address.email_address  AS  address_email_address
FROM  address  WHERE  ?  =  address.user_id
(None,)
```

请注意上面，有一个比较`WHERE ? = address.user_id`，其中绑定值`?`接收`None`，或者在 SQL 中是`NULL`。**这在 SQL 中总是返回 False**。这里的比较理论上会生成以下 SQL：

```py
SELECT  address.id  AS  address_id,  address.user_id  AS  address_user_id,
address.email_address  AS  address_email_address
FROM  address  WHERE  address.user_id  IS  NULL
```

但是现在**并不是这样**。依赖于“NULL = NULL”在所有情况下产生 False 的应用程序存在风险，因为有一天，SQLAlchemy 可能会修复这个问题以生成“IS NULL”，然后查询将产生不同的结果。因此，在这种操作中，你会看到一个警告：

```py
SAWarning: Got None for value of column user.id; this is unsupported
for a relationship comparison and will not currently produce an
IS comparison (but may in a future release)
```

请注意，这种模式在 1.0.0 版本中的大多数情况下都被破坏，包括所有的 beta 版本；像`SYMBOL('NEVER_SET')`这样的值将被生成。这个问题已经修复，但由于识别到这种模式，现在有了警告，以便我们可以更安全地修复这个破损的行为（现在在[#3373](https://www.sqlalchemy.org/trac/ticket/3373)中捕获）在未来的版本中。

[#3371](https://www.sqlalchemy.org/trac/ticket/3371)  ### “否定包含或等于”关系比较将使用属性的当前值，而不是数据库值

此更改是 1.0.1 的新功能；虽然我们希望这个功能在 1.0.0 中就存在，但这只是作为 [#3371](https://www.sqlalchemy.org/trac/ticket/3371) 的结果才变得明显。

给定一个映射：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    a = relationship("A")
```

给定 `A`，其主键为 7，但我们在不刷新的情况下将其更改为 10：

```py
s = Session(autoflush=False)
a1 = A(id=7)
s.add(a1)
s.commit()

a1.id = 10
```

针对此对象作为目标的多对一关系的查询将使用绑定参数中的值 10：

```py
s.query(B).filter(B.a == a1)
```

生成：

```py
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  ?  =  b.a_id
(10,)
```

然而，在此更改之前，这个条件的否定**不会**使用 10，而是使用 7，除非先刷新对象：

```py
s.query(B).filter(B.a != a1)
```

生成（在 0.9 版本和所有 1.0.1 版本之前的版本中）：

```py
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  b.a_id  !=  ?  OR  b.a_id  IS  NULL
(7,)
```

对于一个临时对象，它会产生一个错误的查询：

```py
SELECT  b.id,  b.a_id
FROM  b
WHERE  b.a_id  !=  :a_id_1  OR  b.a_id  IS  NULL
-- {u'a_id_1': symbol('NEVER_SET')}
```

此不一致性已经修复，在所有查询中，当前属性值，例如此示例中的 `10`，现在将被使用。

[#3374](https://www.sqlalchemy.org/trac/ticket/3374)  ### 关于没有预先存在值的属性的属性事件和其他操作的更改

在这个更改中，当访问对象时，`None`的默认返回值现在会在每次访问时动态返回，而不是在首次访问时通过特殊的“设置”操作隐式地设置属性的状态。这个更改的可见结果是，`obj.__dict__`在获取时不会被隐式修改，并且对于 `get_history()` 和相关函数也有一些轻微的行为变化。

给定一个没有状态的对象：

```py
>>> obj = Foo()
```

SQLAlchemy 的行为一直是这样的，如果我们访问一个从未设置过的标量或多对一属性，它会返回 `None`：

```py
>>> obj.someattr
None
```

这个`None`值实际上现在是`obj`状态的一部分，并且与我们明确设置属性的情况类似，例如 `obj.someattr = None`。然而，这里的“获取时设置”在历史和事件方面会有不同的行为。它不会触发任何属性事件，此外，如果我们查看历史记录，我们会看到这样的情况：

```py
>>> inspect(obj).attrs.someattr.history
History(added=(), unchanged=[None], deleted=())   # 0.9 and below
```

也就是说，就好像属性始终是`None`，且从未更改过一样。这与我们首先设置属性的情况明显不同：

```py
>>> obj = Foo()
>>> obj.someattr = None
>>> inspect(obj).attrs.someattr.history
History(added=[None], unchanged=(), deleted=())  # all versions
```

上述意味着我们的“设置”操作的行为可能会被通过“获取”访问值的事实所破坏。在 1.0 中，这个不一致性已经得到了解决，不再在首次访问时实际设置任何东西。

```py
>>> obj = Foo()
>>> obj.someattr
None
>>> inspect(obj).attrs.someattr.history
History(added=(), unchanged=(), deleted=())  # 1.0
>>> obj.someattr = None
>>> inspect(obj).attrs.someattr.history
History(added=[None], unchanged=(), deleted=())
```

以上行为之所以没有产生太大影响，是因为在关系数据库中的 INSERT 语句在大多数情况下将缺失的值视为 NULL。对于 SQLAlchemy 是否接收到了将特定属性设置为 None 的历史事件，通常并不重要；因为发送 None/NULL 或者不发送之间的区别通常不会产生影响。然而，正如[#3060](https://www.sqlalchemy.org/trac/ticket/3060)（在属性变更的优先级：与关系绑定的属性相比，外键绑定的属性可能会出现变化中描述的那样）所示，有一些罕见的边缘情况，我们确实希望明确将`None`设置为属性。此外，允许在此处发生属性事件意味着现在可以为 ORM 映射的属性创建“默认值”函数。

作为这一变更的一部分，现在已禁用了在其他情况下生成隐式`None`的功能；这包括在收到对 many-to-one 进行属性设置操作时；以前，如果“旧”值没有被设置，那么“旧”值将是`None`；现在它将发送值`NEVER_SET`，这是一个现在可以发送到属性监听器的值。在调用诸如`Mapper.primary_key_from_instance()`之类的映射器实用程序函数时，也可能接收到此符号；如果主键属性根本没有设置，而以前值是`None`，那么现在将是`NEVER_SET`符号，并且对象的状态不会发生任何更改。

[#3061](https://www.sqlalchemy.org/trac/ticket/3061)  ### 属性变更的优先级：与关系绑定的属性相比，外键绑定的属性可能会出现变化

作为[#3060](https://www.sqlalchemy.org/trac/ticket/3060)的副作用，将关系绑定的属性设置为`None`现在是一个被追踪的历史事件，它指的是将`None`持久化到该属性的意图。由于一直以来，设置关系绑定的属性将优先于直接分配给外键属性，因此在分配 None 时可以看到行为的变化。给定一个映射：

```py
class A(Base):
    __tablename__ = "table_a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "table_b"

    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("table_a.id"))
    a = relationship(A)
```

在 1.0 中，无论我们分配的值是指向`A`对象的引用还是`None`，关系绑定的属性都优先于 FK 绑定的属性。在 0.9 中，行为是不一致的，只有在分配了值时才生效；None 不被考虑：

```py
a1 = A(id=1)
a2 = A(id=2)
session.add_all([a1, a2])
session.flush()

b1 = B()
b1.a = a1  # we expect a_id to be '1'; takes precedence in 0.9 and 1.0

b2 = B()
b2.a = None  # we expect a_id to be None; takes precedence only in 1.0

b1.a_id = 2
b2.a_id = 2

session.add_all([b1, b2])
session.commit()

assert b1.a is a1  # passes in both 0.9 and 1.0
assert b2.a is None  # passes in 1.0, in 0.9 it's a2
```

[#3060](https://www.sqlalchemy.org/trac/ticket/3060)  ### session.expunge() 将完全分离已被删除的对象

`Session.expunge()` 的行为存在一个错误，导致已删除对象的行为不一致。`object_session()` 函数以及 `InstanceState.session` 属性会在 expunge 之后仍然报告对象属于 `Session`：

```py
u1 = sess.query(User).first()
sess.delete(u1)

sess.flush()

assert u1 not in sess
assert inspect(u1).session is sess  # this is normal before commit

sess.expunge(u1)

assert u1 not in sess
assert inspect(u1).session is None  # would fail
```

请注意，`u1 not in sess` 为 True 而 `inspect(u1).session` 仍然指向会话，当事务正在进行删除操作之后，但尚未调用 `Session.expunge()` 时，完全分离通常会在事务提交后完成。这个问题也会影响到依赖于 `Session.expunge()` 的函数，比如 `make_transient()`。

[#3139](https://www.sqlalchemy.org/trac/ticket/3139)  ### 使用 yield_per 明确禁止连接/子查询即时加载

为了使 `Query.yield_per()` 方法更容易使用，在使用 yield_per 时如果任何子查询即时加载器或将使用集合的连接即时加载器生效，则会引发异常，因为这些当前与 yield_per 不兼容（理论上子查询加载可以，然而）。当引发此错误时，可以使用带有星号的 `lazyload()` 选项：

```py
q = sess.query(Object).options(lazyload("*")).yield_per(100)
```

或者使用 `Query.enable_eagerloads()`：

```py
q = sess.query(Object).enable_eagerloads(False).yield_per(100)
```

`lazyload()` 选项的优点是仍然可以使用额外的多对一连接加载器选项：

```py
q = (
    sess.query(Object)
    .options(lazyload("*"), joinedload("some_manytoone"))
    .yield_per(100)
)
```  ### 对重复连接目标的处理中的更改和修复

对于在两次连接到同一实体或多次连接到同一张表的单表实体而不使用基于关系的 ON 子句时，某些情况下可能会出现意外和不一致行为的错误进行了更改，以及当多次连接到同一目标关系时。

从一个映射开始：

```py
from sqlalchemy import Integer, Column, String, ForeignKey
from sqlalchemy.orm import Session, relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

一个连接到 `A.bs` 两次的查询：

```py
print(s.query(A).join(A.bs).join(A.bs))
```

将会渲染：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  a.id  =  b.a_id
```

查询对冗余的 `A.bs` 进行了去重，因为它试图支持以下情况：

```py
s.query(A).join(A.bs).filter(B.foo == "bar").reset_joinpoint().join(A.bs, B.cs).filter(
    C.bar == "bat"
)
```

也就是说，`A.bs` 是“路径”的一部分。作为 [#3367](https://www.sqlalchemy.org/trac/ticket/3367) 的一部分，如果到达相同的终点两次而不是作为更大路径的一部分，现在会发出警告：

```py
SAWarning: Pathed join target A.bs has already been joined to; skipping
```

更大的变化涉及加入到一个实体而不使用关系绑定路径。如果我们两次加入到 `B`：

```py
print(s.query(A).join(B, B.a_id == A.id).join(B, B.a_id == A.id))
```

在 0.9 版本中，这将呈现如下：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  b  AS  b_1  ON  b_1.a_id  =  a.id
```

这是有问题的，因为别名是隐式的，在不同的 ON 子句的情况下可能导致不可预测的结果。

在 1.0 版本中，不会应用自动别名，我们会得到：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  b  ON  b.a_id  =  a.id
```

这将从数据库中引发错误。虽然如果“重复加入目标”在我们从冗余关系 vs. 冗余非关系目标中都加入时表现相同可能会很好，但目前我们只在以前会发生隐式别名的更严重情况下更改行为，并且在关系情况下只发出警告。最终，在所有情况下，加入到相同的东西两次而没有任何别名以消除歧义应该引发错误。

此更改还影响单表继承目标。使用以下映射：

```py
from sqlalchemy import Integer, Column, String, ForeignKey
from sqlalchemy.orm import Session, relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {"polymorphic_on": type, "polymorphic_identity": "a"}

class ASub1(A):
    __mapper_args__ = {"polymorphic_identity": "asub1"}

class ASub2(A):
    __mapper_args__ = {"polymorphic_identity": "asub2"}

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)

    a_id = Column(Integer, ForeignKey("a.id"))

    a = relationship("A", primaryjoin="B.a_id == A.id", backref="b")

s = Session()

print(s.query(ASub1).join(B, ASub1.b).join(ASub2, B.a))

print(s.query(ASub1).join(B, ASub1.b).join(ASub2, ASub2.id == B.a_id))
```

底部的两个查询是等效的，应该都呈现相同的 SQL：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  a  ON  b.a_id  =  a.id  AND  a.type  IN  (:type_1)
WHERE  a.type  IN  (:type_2)
```

上述 SQL 是无效的，因为在 FROM 列表中两次呈现了“a”。然而，隐式别名 bug 只会在第二个查询中发生，并呈现如下：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  a  AS  a_1
ON  a_1.id  =  b.a_id  AND  a_1.type  IN  (:type_1)
WHERE  a_1.type  IN  (:type_2)
```

在上面，对“a”的第二次加入被别名。虽然这看起来很方便，但这不是单继承查询的一般工作方式，而且是误导性和不一致的。

其净效果是依赖于此 bug 的应用程序现在将由数据库引发错误。解决方案是使用预期的形式。在查询中引用单继承实体的多个子类时，必须手动使用别名来消除表的歧义，因为所有子类通常指向同一张表：

```py
asub2_alias = aliased(ASub2)

print(s.query(ASub1).join(B, ASub1.b).join(asub2_alias, B.a.of_type(asub2_alias)))
```

[#3233](https://www.sqlalchemy.org/trac/ticket/3233) [#3367](https://www.sqlalchemy.org/trac/ticket/3367)

### 延迟列不再隐式取消延迟

标记为延迟的映射属性如果没有明确取消延迟，现在即使它们的列以某种方式存在于结果集中，也将保持“延迟”。这是一个性能增强，因为 ORM 加载在获得结果集时不再花时间搜索每个延迟列。然而，对于一直依赖于此的应用程序，现在应该使用显式的 `undefer()` 或类似选项，以防止在访问属性时发出 SELECT。

### 废弃的 ORM 事件钩子已移除

自 0.5 以来已被弃用的以下 ORM 事件钩子已被删除：`translate_row`、`populate_instance`、`append_result`、`create_instance`。这些钩子的用例起源于早期的 0.1 / 0.2 系列 SQLAlchemy，并且早已不再需要。特别是，这些钩子在很大程度上无法使用，因为这些事件中的行为契约与周围内部的强烈联系，例如需要如何创建和初始化实例以及如何在 ORM 生成的行中定位列。删除这些钩子极大地简化了 ORM 对象加载的机制。  ### 当使用自定义行加载器时，新 Bundle 功能的 API 变更

0.9 版本的新 `Bundle` 对象在 API 上有一个小变化，当在自定义类上覆盖 `create_row_processor()` 方法时。以前的示例代码如下：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
  """Override create_row_processor to return values as dictionaries"""

        def proc(row, result):
            return dict(zip(labels, (proc(row, result) for proc in procs)))

        return proc
```

未使用的 `result` 成员现已被删除：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
  """Override create_row_processor to return values as dictionaries"""

        def proc(row):
            return dict(zip(labels, (proc(row) for proc in procs)))

        return proc
```

另见

使用 Bundles 对选定属性进行分组  ### 内连接右嵌套现在是 `joinedload` 的默认设置，`innerjoin=True`

当内连接贪婪加载链接到外连接贪婪加载时，`joinedload.innerjoin` 的行为以及 `relationship.innerjoin` 现在默认使用“嵌套”内连接，即右嵌套。为了获得当外连接存在时将所有贪婪加载链接为外连接的旧行为，请使用 `innerjoin="unnested"`。

正如从 0.9 版本开始介绍的 内连接右嵌套可用于连接贪婪加载，`innerjoin="nested"` 的行为是将内连接贪婪加载链到外连接贪婪加载时使用右嵌套连接。当使用 `innerjoin=True` 时，现在隐含了 `"nested"`：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin=True)
)
```

使用新的默认设置，FROM 子句将以以下形式呈现：

```py
FROM users LEFT OUTER JOIN (orders JOIN items ON <onclause>) ON <onclause>
```

也就是说，使用右嵌套连接进行 INNER 连接，以便能够返回 `users` 的完整结果。使用 INNER 连接比使用 OUTER 连接更高效，并且允许在所有情况下使用 `joinedload.innerjoin` 优化参数。

要获得旧的行为，请使用 `innerjoin="unnested"`：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin="unnested")
)
```

这将避免右嵌套连接，并使用所有 OUTER 连接将连接链在一起，尽管有内连接指令：

```py
FROM users LEFT OUTER JOIN orders ON <onclause> LEFT OUTER JOIN items ON <onclause>
```

如 0.9 备注中所述，唯一在右嵌套连接方面有困难的数据库后端是 SQLite；从 0.9 开始，SQLAlchemy 会将右嵌套连接转换为子查询作为 SQLite 上的连接目标。

另见

连接式加载中可用的右嵌套内连接 - 介绍了 0.9.4 版本中引入的功能。

[#3008](https://www.sqlalchemy.org/trac/ticket/3008)  ### 不再将子查询应用于 uselist=False 的连接式加载

给定如下的连接式加载：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b = relationship("B", uselist=False)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

s = Session()
print(s.query(A).options(joinedload(A.b)).limit(5))
```

SQLAlchemy 认为关系`A.b`是“一对多，加载为单个值”，本质上是“一对一”关系。然而，连接式加载一直将上述情况视为主查询需要在子查询中的情况，就像对主查询应用 LIMIT 时通常需要的那样：

```py
SELECT  anon_1.a_id  AS  anon_1_a_id,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  (SELECT  a.id  AS  a_id
FROM  a  LIMIT  :param_1)  AS  anon_1
LEFT  OUTER  JOIN  b  AS  b_1  ON  anon_1.a_id  =  b_1.a_id
```

然而，由于内部查询与外部查询的关系在`uselist=False`的情况下最多只共享一行（与一对多的方式相同），因此在使用 LIMIT +连接式加载时，“子查询”在这种情况下现在被取消：

```py
SELECT  a.id  AS  a_id,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  a  LEFT  OUTER  JOIN  b  AS  b_1  ON  a.id  =  b_1.a_id
LIMIT  :param_1
```

在左外连接返回多于一行的情况下，ORM 一直在此处发出警告，并对于`uselist=False`忽略额外的结果，因此在该错误情况下结果不应更改。

[#3249](https://www.sqlalchemy.org/trac/ticket/3249)

### 使用 join()，select_from()，from_self()时，query.update() / query.delete()会引发异常。

在 SQLAlchemy 0.9.10 版本（截至 2015 年 6 月 9 日尚未发布）中，当调用`Query.update()`或`Query.delete()`方法时，如果查询还调用了`Query.join()`，`Query.outerjoin()`，`Query.select_from()`或`Query.from_self()`，则会发出警告。这些是不受支持的用例，在 0.9 系列中静默失败，直到 0.9.10 版本发出警告。在 1.0 版本中，这些情况会引发异常。

[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

### 使用`synchronize_session='evaluate'`时，query.update()在多表更新时会引发异常。

`Query.update()`的“评估器”在多表更新时不起作用，当存在多个表时，需要将其设置为`synchronize_session=False`或`synchronize_session='fetch'`。新的行为是现在会显式引发异常，并提醒更改同步设置。这是从 0.9.7 版本开始升级的，之前只是发出警告。

[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

### 复活事件已被移除

“复活”ORM 事件已完全删除。自版本 0.8 删除了工作单元中的旧“可变”系统以来，此事件已不再起作用。

### 使用 from_self()，count() 时对单表继承条件的更改

给定单表继承映射，例如：

```py
class Widget(Base):
    __table__ = "widget_table"

class FooWidget(Widget):
    pass
```

使用 `Query.from_self()` 或 `Query.count()` 对子类进行操作将生成一个子查询，然后将子类型的“WHERE”条件添加到外部：

```py
sess.query(FooWidget).from_self().all()
```

渲染：

```py
SELECT
  anon_1.widgets_id  AS  anon_1_widgets_id,
  anon_1.widgets_type  AS  anon_1_widgets_type
FROM  (SELECT  widgets.id  AS  widgets_id,  widgets.type  AS  widgets_type,
FROM  widgets)  AS  anon_1
WHERE  anon_1.widgets_type  IN  (?)
```

这个问题在于，如果内部查询没有指定所有列，那么我们就无法在外部添加 WHERE 子句（实际上会尝试，并生成一个糟糕的查询）。这个决定显然可以追溯到 0.6.5 版本，带有注释“可能需要对此进行更多调整”。好吧，这些调整已经到位！因此，上述查询现在将渲染为：

```py
SELECT
  anon_1.widgets_id  AS  anon_1_widgets_id,
  anon_1.widgets_type  AS  anon_1_widgets_type
FROM  (SELECT  widgets.id  AS  widgets_id,  widgets.type  AS  widgets_type,
FROM  widgets
WHERE  widgets.type  IN  (?))  AS  anon_1
```

这样，不包括“type”的查询仍将有效！：

```py
sess.query(FooWidget.id).count()
```

渲染：

```py
SELECT  count(*)  AS  count_1
FROM  (SELECT  widgets.id  AS  widgets_id
FROM  widgets
WHERE  widgets.type  IN  (?))  AS  anon_1
```

[#3177](https://www.sqlalchemy.org/trac/ticket/3177)  ### 单表继承条件无条件地添加到所有 ON 子句中

当连接到单表继承子类目标时，ORM 在连接关系时始终添加“单表条件”。给定一个映射如下：

```py
class Widget(Base):
    __tablename__ = "widget"
    id = Column(Integer, primary_key=True)
    type = Column(String)
    related_id = Column(ForeignKey("related.id"))
    related = relationship("Related", backref="widget")
    __mapper_args__ = {"polymorphic_on": type}

class FooWidget(Widget):
    __mapper_args__ = {"polymorphic_identity": "foo"}

class Related(Base):
    __tablename__ = "related"
    id = Column(Integer, primary_key=True)
```

长时间以来，JOIN 关系将为类型渲染“单继承”子句：

```py
s.query(Related).join(FooWidget, Related.widget).all()
```

SQL 输出： 

```py
SELECT  related.id  AS  related_id
FROM  related  JOIN  widget  ON  related.id  =  widget.related_id  AND  widget.type  IN  (:type_1)
```

上面，因为我们连接到了子类 `FooWidget`，`Query.join()` 知道要将 `AND widget.type IN ('foo')` 条件添加到 ON 子句中。

这里的变化是 `AND widget.type IN()` 条件现在附加到*任何* ON 子句，不仅仅是从关系生成的那些，包括明确声明的一个：

```py
# ON clause will now render as
# related.id = widget.related_id AND widget.type IN (:type_1)
s.query(Related).join(FooWidget, FooWidget.related_id == Related.id).all()
```

以及当没有任何 ON 子句时的“隐式”连接：

```py
# ON clause will now render as
# related.id = widget.related_id AND widget.type IN (:type_1)
s.query(Related).join(FooWidget).all()
```

以前，这些的 ON 子句不会包括单继承条件。已经在应用程序中添加此条件以解决此问题的应用程序将希望删除其显式使用，尽管如果在此期间该条件恰好被渲染两次，则应该继续正常工作。

另请参阅

处理重复连接目标的更改和修复

[#3222](https://www.sqlalchemy.org/trac/ticket/3222)  ### query.update() 现在将字符串名称解析为映射属性名称

`Query.update()`的文档说明给定的`values`字典是“以属性名称为键的字典”，暗示这些是映射的属性名称。不幸的是，该函数更多地设计为接收属性和 SQL 表达式，而不是字符串；当传递字符串时，这些字符串将直接传递到核心更新语句，而不解析这些名称在映射类上如何表示，这意味着名称必须与表列的名称完全匹配，而不是映射到类的属性的名称。

字符���名称现在认真解析为属性名称：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column("user_name", String(50))
```

上面，列`user_name`被映射为`name`。以前，传递字符串的`Query.update()`调用必须按照以下方式调用：

```py
session.query(User).update({"user_name": "moonbeam"})
```

现在给定的字符串已经解析为实体：

```py
session.query(User).update({"name": "moonbeam"})
```

通常最好直接使用属性，以避免任何歧义：

```py
session.query(User).update({User.name: "moonbeam"})
```

此更改还表明，同义词和混合属性也可以通过字符串名称进行引用：

```py
class User(Base):
    __tablename__ = "user"

    id = Column(Integer, primary_key=True)
    name = Column("user_name", String(50))

    @hybrid_property
    def fullname(self):
        return self.name

session.query(User).update({"fullname": "moonbeam"})
```

[#3228](https://www.sqlalchemy.org/trac/ticket/3228)

### 当将具有 None 值的对象与关系进行比较时发出警告

这个更改是从 1.0.1 版本开始的。一些用户正在执行基本上是这种形式的查询：

```py
session.query(Address).filter(Address.user == User(id=None))
```

SQLAlchemy 目前不支持这种模式。对于所有版本，它生成类似于以下的 SQL：

```py
SELECT  address.id  AS  address_id,  address.user_id  AS  address_user_id,
address.email_address  AS  address_email_address
FROM  address  WHERE  ?  =  address.user_id
(None,)
```

请注意上面，有一个比较`WHERE ? = address.user_id`，其中绑定值`?`接收到`None`，或者在 SQL 中是`NULL`。**这在 SQL 中总是返回 False**。理论上，这里的比较会生成如下的 SQL：

```py
SELECT  address.id  AS  address_id,  address.user_id  AS  address_user_id,
address.email_address  AS  address_email_address
FROM  address  WHERE  address.user_id  IS  NULL
```

但是现在，**它不会**。依赖于“NULL = NULL”在所有情况下产生 False 的应用程序可能会面临这样的风险，即有一天，SQLAlchemy 可能会修复此问题以生成“IS NULL”，然后查询将产生不同的结果。因此，在进行这种操作时，您将看到一个警告：

```py
SAWarning: Got None for value of column user.id; this is unsupported
for a relationship comparison and will not currently produce an
IS comparison (but may in a future release)
```

请注意，这种模式在 1.0.0 版本中的大多数情况下都被破坏，包括所有的 beta 版本；像`SYMBOL('NEVER_SET')`这样的值将被生成。这个问题已经修复，但由于识别到这种模式，现在有了警告，以便我们可以更安全地修复这个破损的行为（现在在[#3373](https://www.sqlalchemy.org/trac/ticket/3373)中捕获）在未来的版本中。

[#3371](https://www.sqlalchemy.org/trac/ticket/3371)

### “否定包含或等于”关系比较将使用属性的当前值，而不是数据库值

这个更改是从 1.0.1 版本开始的；虽然我们更希望这在 1.0.0 中实现，但只有通过[#3371](https://www.sqlalchemy.org/trac/ticket/3371)才变得明显。

给定一个映射：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    a = relationship("A")
```

给定`A`，主键为 7，但我们将其更改为 10 而没有刷新：

```py
s = Session(autoflush=False)
a1 = A(id=7)
s.add(a1)
s.commit()

a1.id = 10
```

对于以这个对象为目标的一对多关系的查询将使用绑定参数中的值 10：

```py
s.query(B).filter(B.a == a1)
```

产生的结果：

```py
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  ?  =  b.a_id
(10,)
```

然而，在这个改变之前，对这个标准的否定将**不会**使用 10，而是使用 7，除非对象首先被刷新：

```py
s.query(B).filter(B.a != a1)
```

产生的结果（在 0.9 版本和所有 1.0.1 版本之前的版本中）：

```py
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  b.a_id  !=  ?  OR  b.a_id  IS  NULL
(7,)
```

对于一个瞬态对象，它将产生一个错误的查询：

```py
SELECT  b.id,  b.a_id
FROM  b
WHERE  b.a_id  !=  :a_id_1  OR  b.a_id  IS  NULL
-- {u'a_id_1': symbol('NEVER_SET')}
```

这种不一致性已经得到修复，在所有查询中，当前属性值，比如这个例子中的`10`，现在将被使用。

[#3374](https://www.sqlalchemy.org/trac/ticket/3374)

### 关于没有预先存在值的属性事件和其他操作的更改

在这个改变中，当访问一个对象时，默认的返回值`None`现在会在每次访问时动态返回，而不是在第一次访问时隐式地使用特殊的“set”操作设置属性的状态。这个改变的可见结果是，`obj.__dict__`在获取时不会隐式修改，并且对于`get_history()`和相关函数也有一些轻微的行为变化。

鉴于一个没有状态的对象：

```py
>>> obj = Foo()
```

一直以来，SQLAlchemy 的行为是，如果我们访问一个从未设置过的标量或一对多属性，它会返回`None`：

```py
>>> obj.someattr
None
```

这个`None`值实际上现在是`obj`的状态的一部分，与我们明确设置属性的情况类似，比如`obj.someattr = None`。然而，在这里“获取时设置”会在历史和事件方面有所不同。它不会触发任何属性事件，而且如果我们查看历史记录，我们会看到这样：

```py
>>> inspect(obj).attrs.someattr.history
History(added=(), unchanged=[None], deleted=())   # 0.9 and below
```

也就是说，就好像属性始终是`None`，并且从未更改过。这与我们首先设置属性的情况明显不同：

```py
>>> obj = Foo()
>>> obj.someattr = None
>>> inspect(obj).attrs.someattr.history
History(added=[None], unchanged=(), deleted=())  # all versions
```

上述意味着我们的“set”操作的行为可能会受到早期通过“get”访问值的影响而被破坏。在 1.0 版本中，这种不一致性已经得到解决，不再在使用默认“getter”时实际设置任何内容。

```py
>>> obj = Foo()
>>> obj.someattr
None
>>> inspect(obj).attrs.someattr.history
History(added=(), unchanged=(), deleted=())  # 1.0
>>> obj.someattr = None
>>> inspect(obj).attrs.someattr.history
History(added=[None], unchanged=(), deleted=())
```

上述行为之所以没有太大影响，是因为在关系数据库中的 INSERT 语句在大多数情况下将缺失值视为 NULL。对于特定属性设置为 None 的情况，SQLAlchemy 是否收到历史事件通常不重要；因为发送 None/NULL 或不发送的区别通常不会产生影响。然而，正如[#3060](https://www.sqlalchemy.org/trac/ticket/3060)（在关于关系绑定属性与 FK 绑定属性的属性更改优先级可能会发生变化中描述）所示，有一些很少见的边缘情况，我们实际上确实希望明确设置`None`。此外，在这里允许属性事件意味着现在可以为 ORM 映射属性创建“默认值”函数。

作为这一变化的一部分，现在已禁用了在其他情况下生成隐式`None`的操作；这包括当接收到对多对一属性的属性设置操作时；以前，如果“旧”值未设置，那么“旧”值将为`None`；现在将发送值`NEVER_SET`，这是一个现在可能发送给属性监听器的值。当调用映射器实用函数时，也可能收到此符号，例如`Mapper.primary_key_from_instance()`；如果主键属性根本没有设置，而以前的值为`None`，那么现在将是`NEVER_SET`符号，并且对象状态不会发生任何变化。

[#3061](https://www.sqlalchemy.org/trac/ticket/3061)

### 属性变化对关系绑定属性和外键绑定属性的优先级可能会发生变化

作为[#3060](https://www.sqlalchemy.org/trac/ticket/3060)的一个副作用，将关系绑定属性设置为`None`现在是一个被跟踪的历史事件，指的是将`None`持久化到该属性的意图。由于一直以来设置关系绑定属性将优先于直接赋值给外键属性，因此在分配`None`时可以看到行为上的变化。给定一个映射：

```py
class A(Base):
    __tablename__ = "table_a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "table_b"

    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("table_a.id"))
    a = relationship(A)
```

在 1.0 版本中，无论我们分配的值是对`A`对象的引用还是`None`，关系绑定属性在所有情况下都优先于外键绑定属性。在 0.9 版本中，行为是不一致的，只有在分配值时才会生效；`None`不被考虑：

```py
a1 = A(id=1)
a2 = A(id=2)
session.add_all([a1, a2])
session.flush()

b1 = B()
b1.a = a1  # we expect a_id to be '1'; takes precedence in 0.9 and 1.0

b2 = B()
b2.a = None  # we expect a_id to be None; takes precedence only in 1.0

b1.a_id = 2
b2.a_id = 2

session.add_all([b1, b2])
session.commit()

assert b1.a is a1  # passes in both 0.9 and 1.0
assert b2.a is None  # passes in 1.0, in 0.9 it's a2
```

[#3060](https://www.sqlalchemy.org/trac/ticket/3060)

### session.expunge()将完全分离已删除的对象

`Session.expunge()`的行为存在一个错误，导致关于已删除对象的行为不一致。`object_session()`函数以及`InstanceState.session`属性仍会报告对象属于`Session`，即使在执行 expunge 之后：

```py
u1 = sess.query(User).first()
sess.delete(u1)

sess.flush()

assert u1 not in sess
assert inspect(u1).session is sess  # this is normal before commit

sess.expunge(u1)

assert u1 not in sess
assert inspect(u1).session is None  # would fail
```

请注意，`u1 not in sess`为 True 而`inspect(u1).session`仍然指向会话是正常的，而事务正在进行中，删除操作之后尚未调用`Session.expunge()`；完全分离通常在事务提交后完成。这个问题也会影响依赖于`Session.expunge()`的函数，比如`make_transient()`。

[#3139](https://www.sqlalchemy.org/trac/ticket/3139)

### 使用 yield_per 明确禁止连接/子查询的急加载

为了使`Query.yield_per()`方法更容易使用，如果在使用 yield_per 时要生效任何子查询急加载器或使用集合的连接急加载器，将引发异常，因为这些当前与 yield-per 不兼容（子查询加载理论上可能是兼容的）。当出现此错误时，可以使用带有星号的`lazyload()`选项：

```py
q = sess.query(Object).options(lazyload("*")).yield_per(100)
```

或使用`Query.enable_eagerloads()`:

```py
q = sess.query(Object).enable_eagerloads(False).yield_per(100)
```

`lazyload()`选项的优势在于仍然可以使用额外的多对一连接加载器选项：

```py
q = (
    sess.query(Object)
    .options(lazyload("*"), joinedload("some_manytoone"))
    .yield_per(100)
)
```

### 处理重复连接目标的更改和修复

这里的更改涵盖了在某些情况下连接到实体两次或对同一表的多个单表实体进行多次连接时会发生意外和不一致行为的错误，而不使用基于关系的 ON 子句时，以及在多次连接到相同目标关系时。

从一个映射开始：

```py
from sqlalchemy import Integer, Column, String, ForeignKey
from sqlalchemy.orm import Session, relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

一个查询两次连接到`A.bs`的问题：

```py
print(s.query(A).join(A.bs).join(A.bs))
```

将呈现为：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  a.id  =  b.a_id
```

查询去重了冗余的`A.bs`，因为它试图支持以下情况：

```py
s.query(A).join(A.bs).filter(B.foo == "bar").reset_joinpoint().join(A.bs, B.cs).filter(
    C.bar == "bat"
)
```

也就是说，`A.bs`是“路径”的一部分。作为[#3367](https://www.sqlalchemy.org/trac/ticket/3367)的一部分，到达相同的终点两次而不是作为更大路径的一部分现在会发出警告：

```py
SAWarning: Pathed join target A.bs has already been joined to; skipping
```

更大的变化涉及在不使用基于关系的路径连接到实体时。如果我们两次连接到`B`：

```py
print(s.query(A).join(B, B.a_id == A.id).join(B, B.a_id == A.id))
```

在 0.9 版本中，这将呈现如下：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  b  AS  b_1  ON  b_1.a_id  =  a.id
```

这是有问题的，因为别名是隐式的，在不同的 ON 子句的情况下可能导致结果不可预测。

在 1.0 中，不会自动应用别名，我们得到：

```py
SELECT  a.id  AS  a_id
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  b  ON  b.a_id  =  a.id
```

这将从数据库中引发错误。虽然如果“重复连接目标”在我们从冗余关系 vs. 冗余非关系目标中都加入时表现相同的话会很好，但是目前我们只在以前会发生隐式别名时更改行为，在关系情况下只发出警告。最终，在所有情况下，未加别名以消除歧义地两次连接到相同内容都应该引发错误。

此更改还对单表继承目标产生影响。使用以下映射：

```py
from sqlalchemy import Integer, Column, String, ForeignKey
from sqlalchemy.orm import Session, relationship
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {"polymorphic_on": type, "polymorphic_identity": "a"}

class ASub1(A):
    __mapper_args__ = {"polymorphic_identity": "asub1"}

class ASub2(A):
    __mapper_args__ = {"polymorphic_identity": "asub2"}

class B(Base):
    __tablename__ = "b"

    id = Column(Integer, primary_key=True)

    a_id = Column(Integer, ForeignKey("a.id"))

    a = relationship("A", primaryjoin="B.a_id == A.id", backref="b")

s = Session()

print(s.query(ASub1).join(B, ASub1.b).join(ASub2, B.a))

print(s.query(ASub1).join(B, ASub1.b).join(ASub2, ASub2.id == B.a_id))
```

底部的两个查询是等价的，应该都会渲染相同的 SQL：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  a  ON  b.a_id  =  a.id  AND  a.type  IN  (:type_1)
WHERE  a.type  IN  (:type_2)
```

上面的 SQL 是无效的，因为它在 FROM 列表中两次呈现 “a”。然而，隐式别名错误只会在第二个查询中发生，而不是呈现以下结果：

```py
SELECT  a.id  AS  a_id,  a.type  AS  a_type
FROM  a  JOIN  b  ON  b.a_id  =  a.id  JOIN  a  AS  a_1
ON  a_1.id  =  b.a_id  AND  a_1.type  IN  (:type_1)
WHERE  a_1.type  IN  (:type_2)
```

在上面的情况下，第二次加入到 “a” 的连接已经被别名化。虽然这看起来很方便，但这并不是单继承查询通常的工作方式，这是误导性和不一致的。

净影响是依赖于此错误的应用程序现在将由数据库引发错误。解决方案是使用期望的形式。在查询中引用单继承实体的多个子类时，必须手动使用别名来消除表的歧义，因为所有子类通常都指向相同的表：

```py
asub2_alias = aliased(ASub2)

print(s.query(ASub1).join(B, ASub1.b).join(asub2_alias, B.a.of_type(asub2_alias)))
```

[#3233](https://www.sqlalchemy.org/trac/ticket/3233) [#3367](https://www.sqlalchemy.org/trac/ticket/3367)

### 不再隐式取消延迟列

如果未明确取消设置延迟的延迟列，现在标记为延迟的映射属性将始终保持“延迟”，即使它们的列以某种方式出现在结果集中。这是一种性能增强，因为 ORM 加载在获取结果集时不再花时间搜索每个延迟列。然而，对于依赖于此的应用程序，现在应该使用显式 `undefer()` 或类似选项，以防止在访问属性时发出 SELECT。

### 废弃的 ORM 事件钩子已移除

以下 ORM 事件钩子，其中一些从 0.5 版本开始已被弃用，已被移除：`translate_row`、`populate_instance`、`append_result`、`create_instance`。这些钩子的用例始于早期的 0.1 / 0.2 版本的 SQLAlchemy，并且早已不再需要。特别是，这些钩子在很大程度上无法使用，因为这些事件内部的行为约定与周围内部的密切联系，比如实例需要如何创建和初始化以及如何在 ORM 生成的行中定位列。移除这些钩子大大简化了 ORM 对象加载的机制。

### 在使用自定义行加载器时对新 Bundle 功能的 API 更改

新的 `Bundle` 对象在自定义类上覆盖 `create_row_processor()` 方法时 API 有所变化。以前，示例代码如下所示：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
  """Override create_row_processor to return values as dictionaries"""

        def proc(row, result):
            return dict(zip(labels, (proc(row, result) for proc in procs)))

        return proc
```

未使用的 `result` 成员现已删除：

```py
from sqlalchemy.orm import Bundle

class DictBundle(Bundle):
    def create_row_processor(self, query, procs, labels):
  """Override create_row_processor to return values as dictionaries"""

        def proc(row):
            return dict(zip(labels, (proc(row) for proc in procs)))

        return proc
```

另请参阅

使用 Bundles 分组选定属性

### 右嵌套内连接现在是 `joinedload` 的默认值，`innerjoin=True`

当 INNER JOIN 连接式预加载链接到 OUTER JOIN 连接式预加载时，默认行为为使用“嵌套” INNER JOIN，也就是右嵌套。如果要获得当存在 OUTER JOIN 时链接所有 INNER JOIN 连接式预加载的旧行为，请使用 `innerjoin="unnested"`。

正如在 右嵌套内连接可用于连接式预加载 中介绍的，`innerjoin="nested"` 的行为是，当 INNER JOIN 连接式预加载链接到 OUTER JOIN 连接式预加载时，将使用右嵌套连接。当使用 `innerjoin=True` 时，现在会隐含 `"nested"`：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin=True)
)
```

使用新的默认值，FROM 子句将呈现为以下形式：

```py
FROM users LEFT OUTER JOIN (orders JOIN items ON <onclause>) ON <onclause>
```

也就是说，对于 INNER JOIN，使用右嵌套连接，以便返回`users`的完整结果集。使用 INNER JOIN 比使用 OUTER JOIN 更有效，并且允许在所有情况下生效 `joinedload.innerjoin` 优化参数。

要获得旧行为，请使用 `innerjoin="unnested"`：

```py
query(User).options(
    joinedload("orders", innerjoin=False).joinedload("items", innerjoin="unnested")
)
```

这将避免右嵌套连接，并使用所有 OUTER JOIN 将连接链接在一起，尽管存在 innerjoin 指令：

```py
FROM users LEFT OUTER JOIN orders ON <onclause> LEFT OUTER JOIN items ON <onclause>
```

如在 0.9 版本的说明中指出的，唯一有困难的数据库后端是 SQLite；从 0.9 版本开始，SQLAlchemy 会将右嵌套连接转换为 SQLite 上的子查询作为连接目标。

另请参阅

右嵌套内连接可用于连接式预加载 - 在 0.9.4 版本中引入的功能的描述。

[#3008](https://www.sqlalchemy.org/trac/ticket/3008)

### 不再将子查询应用于 uselist=False 的连接式预加载

给定如下连接式预加载：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b = relationship("B", uselist=False)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

s = Session()
print(s.query(A).options(joinedload(A.b)).limit(5))
```

SQLAlchemy 认为关系 `A.b` 是“加载为单个值的一对多”，实质上是“一对一”关系。然而，连接式预加载始终将以上情况视为需要主查询位于子查询内的情况，正如在应用 LIMIT 于主查询时通常需要的情况下一样：

```py
SELECT  anon_1.a_id  AS  anon_1_a_id,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  (SELECT  a.id  AS  a_id
FROM  a  LIMIT  :param_1)  AS  anon_1
LEFT  OUTER  JOIN  b  AS  b_1  ON  anon_1.a_id  =  b_1.a_id
```

然而，由于内部查询与外部查询的关系，在 `uselist=False` 的情况下最多只有一行是共享的（与多对一相同），因此在此情况下现已放弃使用带有 LIMIT + joined eager loading 的“子查询”：

```py
SELECT  a.id  AS  a_id,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  a  LEFT  OUTER  JOIN  b  AS  b_1  ON  a.id  =  b_1.a_id
LIMIT  :param_1
```

如果 LEFT OUTER JOIN 返回多于一行的情况下，ORM 一直会在此处发出警告，并且对于 `uselist=False`，会忽略额外的结果，因此在这种错误情况下，结果不应更改。

[#3249](https://www.sqlalchemy.org/trac/ticket/3249)

### 当与 join()、select_from()、from_self() 一起使用时，query.update() / query.delete() 会引发异常。

当调用 `Query.update()` 或 `Query.delete()` 方法的查询还调用了 `Query.join()`、`Query.outerjoin()`、`Query.select_from()` 或 `Query.from_self()` 时，SQLAlchemy 0.9.10 发出警告（截至 2015 年 6 月 9 日尚未发布）。这些是不受支持的用例，直到 0.9.10 发出警告，但在 1.0 中，这些情况会引发异常。

[#3349](https://www.sqlalchemy.org/trac/ticket/3349)

### 当使用 `synchronize_session='evaluate'` 进行多表更新时，query.update() 会引发异常。

当进行多表更新时，`Query.update()` 的“评估器”不适用，并且在存在多个表时需要将其设置为 `synchronize_session=False` 或 `synchronize_session='fetch'`。新的行为是现在会显式引发异常，并提供消息以更改同步设置。这是从 0.9.7 开始发出的警告升级而来。

[#3117](https://www.sqlalchemy.org/trac/ticket/3117)

### 已删除 Resurrect 事件

完全删除了“复活”ORM 事件。自从版本 0.8 移除了工作单元中的旧“可变”系统以来，该事件已不再起作用。

### 使用 from_self()、count() 时的单表继承条件更改

给定单表继承映射，例如：

```py
class Widget(Base):
    __table__ = "widget_table"

class FooWidget(Widget):
    pass
```

对子类使用 `Query.from_self()` 或 `Query.count()` 会生成子查询，但然后将子类型的“WHERE”条件添加到外部：

```py
sess.query(FooWidget).from_self().all()
```

渲染：

```py
SELECT
  anon_1.widgets_id  AS  anon_1_widgets_id,
  anon_1.widgets_type  AS  anon_1_widgets_type
FROM  (SELECT  widgets.id  AS  widgets_id,  widgets.type  AS  widgets_type,
FROM  widgets)  AS  anon_1
WHERE  anon_1.widgets_type  IN  (?)
```

这样做的问题是，如果内部查询没有指定所有列，那么我们无法在外部添加 WHERE 子句（实际上尝试了，并生成了一个错误的查询）。这个决定显然可以追溯到 0.6.5，注明“可能需要对此进行更多调整”。好吧，这些调整已经到来了！因此，现在上述查询将呈现为：

```py
SELECT
  anon_1.widgets_id  AS  anon_1_widgets_id,
  anon_1.widgets_type  AS  anon_1_widgets_type
FROM  (SELECT  widgets.id  AS  widgets_id,  widgets.type  AS  widgets_type,
FROM  widgets
WHERE  widgets.type  IN  (?))  AS  anon_1
```

因此，即使查询不包括“type”，也会工作！：

```py
sess.query(FooWidget.id).count()
```

渲染：

```py
SELECT  count(*)  AS  count_1
FROM  (SELECT  widgets.id  AS  widgets_id
FROM  widgets
WHERE  widgets.type  IN  (?))  AS  anon_1
```

[#3177](https://www.sqlalchemy.org/trac/ticket/3177)

### 单表继承条件无条件添加到所有 ON 子句

当加入到单表继承子类目标时，ORM 总是在关系上加入“单表条件”。假设有一个映射如下：

```py
class Widget(Base):
    __tablename__ = "widget"
    id = Column(Integer, primary_key=True)
    type = Column(String)
    related_id = Column(ForeignKey("related.id"))
    related = relationship("Related", backref="widget")
    __mapper_args__ = {"polymorphic_on": type}

class FooWidget(Widget):
    __mapper_args__ = {"polymorphic_identity": "foo"}

class Related(Base):
    __tablename__ = "related"
    id = Column(Integer, primary_key=True)
```

很长一段时间以来，JOIN 关系都会为类型生成“单继承”子句：

```py
s.query(Related).join(FooWidget, Related.widget).all()
```

SQL 输出：

```py
SELECT  related.id  AS  related_id
FROM  related  JOIN  widget  ON  related.id  =  widget.related_id  AND  widget.type  IN  (:type_1)
```

在上面的例子中，由于我们连接到了子类 `FooWidget`，`Query.join()` 知道要将 `AND widget.type IN ('foo')` 的条件添加到 ON 子句中。

此处的更改是现在 `AND widget.type IN()` 的条件现在附加到 *任何* ON 子句上，不仅仅是从关系生成的那些，包括明确声明的一个：

```py
# ON clause will now render as
# related.id = widget.related_id AND widget.type IN (:type_1)
s.query(Related).join(FooWidget, FooWidget.related_id == Related.id).all()
```

以及当没有任何类型的 ON 子句时的“隐式”连接：

```py
# ON clause will now render as
# related.id = widget.related_id AND widget.type IN (:type_1)
s.query(Related).join(FooWidget).all()
```

以前，这些 ON 子句不会包含单继承的条件。已经添加了此条件以解决此问题的应用程序将希望删除其显式使用，尽管在此期间如果条件恰好被重复呈现，则应该继续正常工作。

另请参阅

处理重复的联接目标中的更改和修复

[#3222](https://www.sqlalchemy.org/trac/ticket/3222)

## 关键行为更改 - 核心

### 将完整的 SQL 片段强制转换为 text() 时发出警告

自 SQLAlchemy 成立以来，一直强调不妨碍纯文本的使用。核心和 ORM 表达系统旨在允许用户在许多地方使用纯文本 SQL 表达式，不仅仅是在你可以将完整的 SQL 字符串发送到 `Connection.execute()`，而且还可以将带有 SQL 表达式的字符串发送到许多函数中，例如 `Select.where()`，`Query.filter()` 和 `Select.order_by()`。

请注意，“SQL 表达式”指的是完整的 SQL 字符串片段，例如：

```py
# the argument sent to where() is a full SQL expression
stmt = select([sometable]).where("somecolumn = 'value'")
```

我们并不是在谈论字符串参数，也就是传递字符串值并成为参数化的正常行为：

```py
# This is a normal Core expression with a string argument -
# we aren't talking about this!!
stmt = select([sometable]).where(sometable.c.somecolumn == "value")
```

核心教程长期以来一直展示了使用这种技术的示例，使用了一个`select()`构造，其中几乎所有组件都被指定为直接字符串。然而，尽管存在这种长期行为和示例，用户显然对这种行为感到惊讶，当在社区中询问时，我无法找到任何一个用户实际上*不*感到惊讶，即您可以将完整字符串发送到像`Query.filter()`这样的方法中。

因此，这里的更改是鼓励用户在部分或完全由文本片段组成的 SQL 中对文本字符串进行限定。在以下组成选择时：

```py
stmt = select(["a", "b"]).where("a = b").select_from("sometable")
```

语句通常构建，具有与以前相同的强制转换。然而，将看到以下警告被发出：

```py
SAWarning: Textual column expression 'a' should be explicitly declared
with text('a'), or use column('a') for more specificity
(this warning may be suppressed after 10 occurrences)

SAWarning: Textual column expression 'b' should be explicitly declared
with text('b'), or use column('b') for more specificity
(this warning may be suppressed after 10 occurrences)

SAWarning: Textual SQL expression 'a = b' should be explicitly declared
as text('a = b') (this warning may be suppressed after 10 occurrences)

SAWarning: Textual SQL FROM expression 'sometable' should be explicitly
declared as text('sometable'), or use table('sometable') for more
specificity (this warning may be suppressed after 10 occurrences)
```

这些警告试图准确显示问题所在，显示参数以及字符串接收的位置。警告使用 Session.get_bind()处理更广泛的继承场景，以便可以安全地发出参数化警告而不会耗尽内存，如常，如果希望将警告作为异常处理，应使用[Python 警告过滤器](https://docs.python.org/2/library/warnings.html)：

```py
import warnings

warnings.simplefilter("error")  # all warnings raise an exception
```

给出上述警告后，我们的语句运行正常，但为了消除警告，我们将重写我们的语句如下：

```py
from sqlalchemy import select, text

stmt = (
    select([text("a"), text("b")]).where(text("a = b")).select_from(text("sometable"))
)
```

正如警告所建议的，如果我们使用`column()`和`table()`，我们可以使我们的语句更具体：

```py
from sqlalchemy import select, text, column, table

stmt = (
    select([column("a"), column("b")])
    .where(text("a = b"))
    .select_from(table("sometable"))
)
```

还要注意，`table()`和`column()`现在可以从“sqlalchemy”中导入，而不需要“sql”部分。

此行为也适用于`select()`以及`Query`上的关键方法，包括`Query.filter()`、`Query.from_statement()`和`Query.having()`。

#### ORDER BY 和 GROUP BY 是特殊情况

有一种情况下，使用字符串具有特殊含义，并且作为这一变化的一部分，我们已增强了其功能。当我们有一个`select()`或`Query`引用某个列名或命名标签时，我们可能想要对已知列或标签进行 GROUP BY 和/或 ORDER BY：

```py
stmt = (
    select([user.c.name, func.count(user.c.id).label("id_count")])
    .group_by("name")
    .order_by("id_count")
)
```

在上面的语句中，我们期望看到“ORDER BY id_count”，而不是函数的重新声明。在编译过程中，给定的字符串参数会与列子句中的条目进行主动匹配，因此上述语句将按我们的期望产生，没有警告（尽管请注意，`"name"`表达式已解析为`users.name`！）：

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  GROUP  BY  users.name  ORDER  BY  id_count
```

然而，如果我们引用一个无法找到的名称，那么我们会再次收到警告，如下所示：

```py
stmt = select([user.c.name, func.count(user.c.id).label("id_count")]).order_by(
    "some_label"
)
```

输出确实按我们说的做了，但再次警告我们：

```py
SAWarning: Can't resolve label reference 'some_label'; converting to
text() (this warning may be suppressed after 10 occurrences)
```

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  ORDER  BY  some_label
```

上述行为适用于所有那些我们可能想要引用所谓“标签引用”的地方；ORDER BY 和 GROUP BY，但也在 OVER 子句以及引用列的 DISTINCT ON 子句中（例如 PostgreSQL 语法）：

我们仍然可以使用`text()`指定任意表达式用于 ORDER BY 或其他操作：

```py
stmt = select([users]).order_by(text("some special expression"))
```

整个变化的结果是，SQLAlchemy 现在希望我们告诉它，当发送一个字符串时，这个字符串明确是一个`text()`构造，或者是一个列、表等，如果我们在 order by、group by 或其他表达式中使用它作为标签名称，SQLAlchemy 期望该字符串解析为已知的内容，否则应再次用`text()`或类似的方式进行限定。

[#2992](https://www.sqlalchemy.org/trac/ticket/2992)  ### 使用多值插入时，每行都会单独调用 Python 端的默认值

当使用`Insert.values()`的多值版本时，对于 Python 端列默认值的支持基本上没有实现，并且只会在特定情况下“偶然”工作，当使用的方言使用非位置（例如命名）风格的绑定参数时，并且不需要为每一行调用 Python 端可调用函���时。

该功能已进行了全面改进，使其更类似于“executemany”样式的调用：

```py
import itertools

counter = itertools.count(1)
t = Table(
    "my_table",
    metadata,
    Column("id", Integer, default=lambda: next(counter)),
    Column("data", String),
)

conn.execute(
    t.insert().values(
        [
            {"data": "d1"},
            {"data": "d2"},
            {"data": "d3"},
        ]
    )
)
```

上面的示例将为每一行单独调用`next(counter)`，这是可以预期的：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?),  (?,  ?),  (?,  ?)
(1,  'd1',  2,  'd2',  3,  'd3')
```

以前，位置方言会失败，因为不会为额外的位置生成绑定：

```py
Incorrect number of bindings supplied. The current statement uses 6,
and there are 4 supplied.
[SQL: u'INSERT INTO my_table (id, data) VALUES (?, ?), (?, ?), (?, ?)']
[parameters: (1, 'd1', 'd2', 'd3')]
```

并且对于“命名”方言，相同的“id”值将在每一行中重新使用（因此这种更改与依赖于此的系统不兼容）：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (:id,  :data_0),  (:id,  :data_1),  (:id,  :data_2)
-- {u'data_2': 'd3', u'data_1': 'd2', u'data_0': 'd1', 'id': 1}
```

系统还将拒绝将“服务器端”默认值作为内联渲染的 SQL 调用，因为无法保证服务器端默认值与此兼容。如果 VALUES 子句为特定列呈现，则需要 Python 端的值；如果省略的值仅指向服务器端默认值，则会引发异常：

```py
t = Table(
    "my_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String, server_default="some default"),
)

conn.execute(
    t.insert().values(
        [
            {"data": "d1"},
            {"data": "d2"},
            {},
        ]
    )
)
```

将引发：

```py
sqlalchemy.exc.CompileError: INSERT value for column my_table.data is
explicitly rendered as a boundparameter in the VALUES clause; a
Python-side value or SQL expression is required
```

以前，“d1”值将被复制到第三行的值中（但再次，仅适用于命名格式！）：

```py
INSERT  INTO  my_table  (data)  VALUES  (:data_0),  (:data_1),  (:data_0)
-- {u'data_1': 'd2', u'data_0': 'd1'}
```

[#3288](https://www.sqlalchemy.org/trac/ticket/3288)  ### 事件监听器不能在该事件的运行程序内添加或删除

从同一事件中删除事件监听器将在迭代期间修改列表的元素，这将导致仍然附加的事件监听器无法静默触发。为了防止这种情况，同时保持性能，列表已被替换为`collections.deque()`，在迭代期间不允许任何添加或删除，并且会引发`RuntimeError`。

[#3163](https://www.sqlalchemy.org/trac/ticket/3163)  ### INSERT…FROM SELECT 结构现在意味着`inline=True`

使用`Insert.from_select()`现在意味着在`insert()`上隐含`inline=True`。这有助于修复一个 bug，即在支持的后端上，INSERT…FROM SELECT 结构会被错误地编译为“隐式返回”，这会导致在插入零行的情况下出现故障（因为隐式返回期望一行），以及在插入多行的情况下出现任意返回数据（例如，只有许多行中的第一行）。类似的更改也适用于具有多个参数集的 INSERT..VALUES；对于此语句，隐式 RETURNING 也不再发出。由于这两个结构处理可变数量的行，因此`ResultProxy.inserted_primary_key`访问器不适用。以前，有一个文档注释，即对于 INSERT..FROM SELECT，可能更喜欢使用`inline=True`，因为一些数据库不支持返回，因此无法进行“隐式”返回，但无论如何，INSERT…FROM SELECT 都不需要隐式返回。如果需要返回插入的数据的可变数量的结果行，则应使用常规的显式`Insert.returning()`。

[#3169](https://www.sqlalchemy.org/trac/ticket/3169)  ### `autoload_with`现在意味着`autoload=True`

通过仅传递 `Table.autoload_with` ，可以设置一个 `Table` 进行反射：

```py
my_table = Table("my_table", metadata, autoload_with=some_engine)
```

[#3027](https://www.sqlalchemy.org/trac/ticket/3027)  ### DBAPI 异常封装和 handle_error() 事件改进

SQLAlchemy 对 DBAPI 异常的封装在以下情况下未发生：当一个 `Connection` 对象被失效，然后尝试重新连接并遇到错误时；这个问题已经解决了。

另外，最近添加的 `ConnectionEvents.handle_error()` 事件现在会在初始连接时、重新连接时以及通过 `create_engine()` 使用自定义连接函数时（通过 `create_engine.creator` ）调用。

`ExceptionContext` 对象现在有一个新的数据成员 `ExceptionContext.engine` ，在那些 `Connection` 对象不可用的情况下（例如在初始连接时）将始终引用正在使用的 `Engine` 。

[#3266](https://www.sqlalchemy.org/trac/ticket/3266)  ### ForeignKeyConstraint.columns 现在是 ColumnCollection

`ForeignKeyConstraint.columns`以前是一个普通列表，其中包含字符串或根据`ForeignKeyConstraint`如何构建以及是否与表关联而包含`Column`对象。该集合现在是一个`ColumnCollection`，并且仅在`ForeignKeyConstraint`与`Table`关联后才初始化。添加了一个新的访问器`ForeignKeyConstraint.column_keys`，无条件地返回本地列集的字符串键，而不管对象是如何构建的或其当前状态如何。### MetaData.sorted_tables 访问器是“确定性的”

由`MetaData.sorted_tables`访问器导致的表的排序是“确定性的”；无论 Python 哈希如何，排序在所有情况下都应该相同。这是通过首先按名称对表进行排序，然后将它们传递给拓扑算法来实现的，该算法在迭代时保持该顺序。

注意，这种变化**尚未**应用于在发出`MetaData.create_all()`或`MetaData.drop_all()`时应用的顺序。

[#3084](https://www.sqlalchemy.org/trac/ticket/3084)### null()、false()和 true()常量不再是单例

这三个常量在 0.9 中被更改为返回“单例”值；不幸的是，这会导致以下查询无法按预期渲染：

```py
select([null(), null()])
```

仅渲染`SELECT NULL AS anon_1`，因为两个`null()`构造将输出相同的`NULL`对象，而 SQLAlchemy 的核心模型基于对象标识来确定词法重要性。0.9 中的更改除了希望节省对象开销外并无重要性；通常，未命名的构造需要保持词法上的唯一性，以便被唯一标记。

[#3170](https://www.sqlalchemy.org/trac/ticket/3170)### SQLite/Oracle 有用于临时表/视图名称报告的不同方法

`Inspector.get_table_names()` 和 `Inspector.get_view_names()` 方法在 SQLite/Oracle 的情况下也会返回临时表和视图的名称，这是任何其他方言都不提供的（至少在 MySQL 的情况下甚至不可能）。这一逻辑已经移出到两个新方法 `Inspector.get_temp_table_names()` 和 `Inspector.get_temp_view_names()`。

请注意，对于大多数方言，反射特定命名的临时表或临时视图，无论是通过 `Table('name', autoload=True)` 还是通过 `Inspector.get_columns()` 等方法，继续正常运行。特别是对于 SQLite，还修复了从临时表中反射 UNIQUE 约束的 bug，即 [#3203](https://www.sqlalchemy.org/trac/ticket/3203)。

[#3204](https://www.sqlalchemy.org/trac/ticket/3204)  ### 当将完整的 SQL 片段强制转换为文本时发出警告

自 SQLAlchemy 创建以来，始终强调不妨碍使用纯文本的方式。核心和 ORM 表达系统旨在允许用户在任何时候使用纯文本 SQL 表达式，不仅仅是可以将完整的 SQL 字符串发送给 `Connection.execute()`，还可以将带有 SQL 表达式的字符串发送到许多函数中，例如 `Select.where()`，`Query.filter()` 和 `Select.order_by()`。

请注意，“SQL 表达式”指的是**完整的 SQL 字符串片段**，例如：

```py
# the argument sent to where() is a full SQL expression
stmt = select([sometable]).where("somecolumn = 'value'")
```

我们并**不是在讨论字符串参数**，也就是传递字符串值并变成参数化的正常行为：

```py
# This is a normal Core expression with a string argument -
# we aren't talking about this!!
stmt = select([sometable]).where(sometable.c.somecolumn == "value")
```

核心教程长期以来一直展示了使用这种技术的示例，使用了`select()`构造，其中几乎所有组件都被指定为直接字符串。然而，尽管有这种长期存在的行为和示例，用户显然对此行为感到惊讶，当在社区中询问时，我无法找到任何一个用户实际上*不*感到惊讶的，即您可以将完整的字符串发送到`Query.filter()`等方法中。

因此，这里的变化是鼓励用户在从文本片段部分或完全组成的 SQL 中组合文本字符串时进行限定。当如下组合选择时：

```py
stmt = select(["a", "b"]).where("a = b").select_from("sometable")
```

语句按照通常的方式构建，具有与以前相同的强制转换。然而，你会看到以下警告被发出：

```py
SAWarning: Textual column expression 'a' should be explicitly declared
with text('a'), or use column('a') for more specificity
(this warning may be suppressed after 10 occurrences)

SAWarning: Textual column expression 'b' should be explicitly declared
with text('b'), or use column('b') for more specificity
(this warning may be suppressed after 10 occurrences)

SAWarning: Textual SQL expression 'a = b' should be explicitly declared
as text('a = b') (this warning may be suppressed after 10 occurrences)

SAWarning: Textual SQL FROM expression 'sometable' should be explicitly
declared as text('sometable'), or use table('sometable') for more
specificity (this warning may be suppressed after 10 occurrences)
```

这些警告试图通过显示参数以及字符串接收位置的方式准确指出问题所在。这些警告利用了会话.get_bind()处理更广泛的继承场景，以便可以安全地发出参数化警告，而不会耗尽内存。如往常一样，如果希望将警告作为异常处理，应该使用[Python 警告过滤器](https://docs.python.org/2/library/warnings.html)：

```py
import warnings

warnings.simplefilter("error")  # all warnings raise an exception
```

给出上述警告，我们的语句运行良好，但要消除警告，我们需要重写我们的语句如下：

```py
from sqlalchemy import select, text

stmt = (
    select([text("a"), text("b")]).where(text("a = b")).select_from(text("sometable"))
)
```

正如警告所建议的那样，如果我们使用`column()`和`table()`，我们可以对语句的文本给予更多的具体性：

```py
from sqlalchemy import select, text, column, table

stmt = (
    select([column("a"), column("b")])
    .where(text("a = b"))
    .select_from(table("sometable"))
)
```

还请注意，现在可以从“sqlalchemy”中导入`table()`和`column()`而不需要“sql”部分。

这里的行为适用于`select()`以及`Query`的关键方法，包括`Query.filter()`、`Query.from_statement()`和`Query.having()`。

#### ORDER BY 和 GROUP BY 是特例

有一种情况下，使用字符串具有特殊含义，并且作为这种更改的一部分，我们已经增强了其功能。当我们有一个`select()`或`Query`引用某个列名或命名标签时，我们可能希望按已知列或标签进行分组和/或排序：

```py
stmt = (
    select([user.c.name, func.count(user.c.id).label("id_count")])
    .group_by("name")
    .order_by("id_count")
)
```

在上述语句中，我们期望看到“ORDER BY id_count”，而不是函数的重新声明。在编译过程中，给定的字符串参数会被主动匹配到列子句中的条目，因此上述语句会按我们的期望产生结果，没有警告（尽管请注意`"name"`表达式已解析为`users.name`！）：

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  GROUP  BY  users.name  ORDER  BY  id_count
```

然而，如果我们引用一个无法找到的名称，那么我们会再次收到警告，如下所示：

```py
stmt = select([user.c.name, func.count(user.c.id).label("id_count")]).order_by(
    "some_label"
)
```

输出确实按我们说的做了，但再次警告我们：

```py
SAWarning: Can't resolve label reference 'some_label'; converting to
text() (this warning may be suppressed after 10 occurrences)
```

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  ORDER  BY  some_label
```

上述行为适用于所有那些我们可能想要引用所谓的“标签引用”的地方；ORDER BY 和 GROUP BY，还有在 OVER 子句中以及引用列的 DISTINCT ON 子句中（例如 PostgreSQL 语法）。

我们仍然可以使用`text()`指定任意表达式用于 ORDER BY 或其他操作：

```py
stmt = select([users]).order_by(text("some special expression"))
```

整个更改的要点是，现在 SQLAlchemy 希望我们告诉它，当发送一个字符串时，这个字符串明确是一个`text()`构造，或者是一个列、表等，如果我们将其用作 order by、group by 或其他表达式中的标签名称，SQLAlchemy 期望该字符串解析为已知内容，否则应再次使用`text()`或类似内容进行限定。

[#2992](https://www.sqlalchemy.org/trac/ticket/2992)

#### ORDER BY 和 GROUP BY 是特殊情况

有一种情况下，使用字符串具有特殊含义，并且作为这种更改的一部分，我们已经增强了其功能。当我们有一个`select()`或`Query`引用某个列名或命名标签时，我们可能希望按已知列或标签进行分组和/或排序：

```py
stmt = (
    select([user.c.name, func.count(user.c.id).label("id_count")])
    .group_by("name")
    .order_by("id_count")
)
```

在上述语句中，我们期望看到“ORDER BY id_count”，而不是函数的重新声明。在编译过程中，给定的字符串参数会被主动匹配到列子句中的条目，因此上述语句会按我们的期望产生结果，���有警告（尽管请注意`"name"`表达式已解析为`users.name`！）：

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  GROUP  BY  users.name  ORDER  BY  id_count
```

然而，如果我们引用一个无法找到的名称，那么我们会再次收到警告，如下所示：

```py
stmt = select([user.c.name, func.count(user.c.id).label("id_count")]).order_by(
    "some_label"
)
```

输出确实按我们说的做了，但再次警告我们：

```py
SAWarning: Can't resolve label reference 'some_label'; converting to
text() (this warning may be suppressed after 10 occurrences)
```

```py
SELECT  users.name,  count(users.id)  AS  id_count
FROM  users  ORDER  BY  some_label
```

上述行为适用于所有那些我们可能想要引用所谓的“标签引用”的地方；ORDER BY 和 GROUP BY，还有在 OVER 子句以及引用列的 DISTINCT ON 子句中（例如 PostgreSQL 语法）。

我们仍然可以使用`text()`指定任意表达式用于 ORDER BY 或其他操作：

```py
stmt = select([users]).order_by(text("some special expression"))
```

整个变化的结果是，SQLAlchemy 现在希望我们告诉它当发送一个字符串时，这个字符串明确是一个`text()` 构造，或者一个列、表等，如果我们将其用作 ORDER BY、GROUP BY 或其他表达式中的标签名称，SQLAlchemy 期望该字符串解析为已知的内容，否则应再次使用`text()` 或类似的进行限定。

[#2992](https://www.sqlalchemy.org/trac/ticket/2992)

### 当使用多值插入时，为每一行分别调用 Python 端默认值

当使用多值版本的`Insert.values()`时，对于 Python 端列默认值的支持基本上没有实现，并且只会在特定情况下“偶然”起作用，当使用的方言采用非位置（例如命名）风格的绑定参数时，并且不需要为每一行调用 Python 端可调用函数时。

该功能已经进行了改进，使其更类似于“executemany”风格的调用：

```py
import itertools

counter = itertools.count(1)
t = Table(
    "my_table",
    metadata,
    Column("id", Integer, default=lambda: next(counter)),
    Column("data", String),
)

conn.execute(
    t.insert().values(
        [
            {"data": "d1"},
            {"data": "d2"},
            {"data": "d3"},
        ]
    )
)
```

上面的示例将为每一行分别调用`next(counter)`，正如预期的那样：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?),  (?,  ?),  (?,  ?)
(1,  'd1',  2,  'd2',  3,  'd3')
```

以前，位置方言会失败，因为不会为额外的位置生成绑定：

```py
Incorrect number of bindings supplied. The current statement uses 6,
and there are 4 supplied.
[SQL: u'INSERT INTO my_table (id, data) VALUES (?, ?), (?, ?), (?, ?)']
[parameters: (1, 'd1', 'd2', 'd3')]
```

并且使用“命名”方言时，“id” 的相同值将在每一行中重新使用（因此这个改变与依赖于此的系统不兼容）：

```py
INSERT  INTO  my_table  (id,  data)  VALUES  (:id,  :data_0),  (:id,  :data_1),  (:id,  :data_2)
-- {u'data_2': 'd3', u'data_1': 'd2', u'data_0': 'd1', 'id': 1}
```

该系统还会拒绝将“服务器端”默认值作为内联渲染的 SQL 调用，因为无法保证服务器端默认值与此兼容。如果 VALUES 子句为特定列渲染，则需要一个 Python 端值；如果省略的值只是引用服务器端默认值，则会引发异常：

```py
t = Table(
    "my_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String, server_default="some default"),
)

conn.execute(
    t.insert().values(
        [
            {"data": "d1"},
            {"data": "d2"},
            {},
        ]
    )
)
```

将引发：

```py
sqlalchemy.exc.CompileError: INSERT value for column my_table.data is
explicitly rendered as a boundparameter in the VALUES clause; a
Python-side value or SQL expression is required
```

以前，“d1” 的值会被复制到第三行的值中（但仅适用于命名格式！）：

```py
INSERT  INTO  my_table  (data)  VALUES  (:data_0),  (:data_1),  (:data_0)
-- {u'data_1': 'd2', u'data_0': 'd1'}
```

[#3288](https://www.sqlalchemy.org/trac/ticket/3288)

### 事件监听器不能在该事件的运行程序内添加或移除

在同一事件中从内部移除事件侦听器会在迭代过程中修改列表的元素，这将导致仍然附加的事件侦听器无法静默触发。为了防止这种情况，同时保持性能，列表已被替换为 `collections.deque()`，在迭代过程中不允许添加或删除，并且会引发 `RuntimeError`。

[#3163](https://www.sqlalchemy.org/trac/ticket/3163)

### INSERT…FROM SELECT 结构现在意味着 `inline=True`

在使用 `Insert.from_select()` 时，现在会隐含 `inline=True` 在 `insert()` 上。这有助于修复一个 bug，即 INSERT…FROM SELECT 结构会在支持的后端意外地被编译为“implicit returning”，这会导致在插入零行的情况下出现故障（因为 implicit returning 需要一行），以及在插入多行的情况下出现任意返回数据（例如，多行中的第一行）。类似的更改也适用于具有多个参数集的 INSERT..VALUES；此语句也不再发出 implicit RETURNING。由于这两个结构处理可变数量的行，因此 `ResultProxy.inserted_primary_key` 访问器不适用。以前，有一个文档说明，即在某些数据库不支持返回并且因此无法执行“implicit”返回的情况下，可能更喜欢使用 `inline=True` 与 INSERT..FROM SELECT，但无论如何，INSERT…FROM SELECT 都不需要 implicit returning。如果需要返回插入数据的可变数量的结果行，则应使用常规的显式 `Insert.returning()`。

[#3169](https://www.sqlalchemy.org/trac/ticket/3169)

### `autoload_with` 现在意味着 `autoload=True`

通过仅传递 `Table.autoload_with`，可以为 `Table` 设置反射：

```py
my_table = Table("my_table", metadata, autoload_with=some_engine)
```

[#3027](https://www.sqlalchemy.org/trac/ticket/3027)

### DBAPI 异常包装和 handle_error() 事件改进

在 `Connection` 对象失效并尝试重新连接并遇到错误的情况下，SQLAlchemy 对 DBAPI 异常的包装未发生，这个问题已经解决。

此外，最近添加的`ConnectionEvents.handle_error()`事件现在在初始连接时、重新连接时以及通过`create_engine()`给定自定义连接函数时通过`create_engine.creator`调用。

`ExceptionContext` 对象有一个新的数据成员`ExceptionContext.engine`，它将始终引用正在使用的`Engine`，在`Connection`对象不可用的情况下（例如在初始连接时）。

[#3266](https://www.sqlalchemy.org/trac/ticket/3266)

### ForeignKeyConstraint.columns 现在是一个 ColumnCollection

`ForeignKeyConstraint.columns` 以前是一个普通列表，其中包含字符串或`Column`对象，取决于`ForeignKeyConstraint`的构造方式以及是否与表相关联。该集合现在是一个`ColumnCollection`，只有在`ForeignKeyConstraint`与`Table`相关联后才会初始化。新增了一个访问器`ForeignKeyConstraint.column_keys`，无条件地返回本地列集的字符串键，而不管对象是如何构造的或其当前状态如何。

### MetaData.sorted_tables 访问器是“确定性的”

由`MetaData.sorted_tables`访问器导致的表的排序是“确定性的”；无论 Python 哈希如何，排序在所有情况下应该是相同的。这是通过首先按名称对表进行排序，然后将它们传递给拓扑算法来实现的，该算法在迭代时保持该顺序。

注意，此更改**尚未**应用于在发出`MetaData.create_all()`或`MetaData.drop_all()`时应用的排序。

[#3084](https://www.sqlalchemy.org/trac/ticket/3084)

### null()、false()和 true()常量不再是单例

这三个常量在 0.9 中被更改为返回“单例”值；不幸的是，这将导致像以下查询一样的内容无法按预期渲染：

```py
select([null(), null()])
```

仅渲染`SELECT NULL AS anon_1`，因为两个`null()`构造将产生相同的`NULL`对象，并且 SQLAlchemy 的 Core 模型是基于对象标识来确定词法重要性的。0.9 中的变化除了希望节省对象开销外没有任何重要性；一般来说，一个未命名的构造需要保持词法上的唯一性，以便得到唯一的标记。

[#3170](https://www.sqlalchemy.org/trac/ticket/3170)

### SQLite/Oracle 有用于临时表/视图名称报告的不同方法

在 SQLite/Oracle 的情况下，`Inspector.get_table_names()`和`Inspector.get_view_names()`方法还将返回临时表和视图的名称，这是其他方言不提供的（至少在 MySQL 的情况下甚至不可能）。这种逻辑已移至两个新方法`Inspector.get_temp_table_names()`和`Inspector.get_temp_view_names()`。

注意，特定命名的临时表或临时视图的反射，无论是通过`Table('name', autoload=True)`还是通过`Inspector.get_columns()`等方法，对大多数（如果不是全部）方言仍然有效。对于 SQLite 特别地，还修复了有关从临时表中反射 UNIQUE 约束的错误，这是[#3203](https://www.sqlalchemy.org/trac/ticket/3203)。

[#3204](https://www.sqlalchemy.org/trac/ticket/3204)

## 方言改进和变更 - PostgreSQL

### 枚举类型创建/删除规则的彻底改造

与创建和删除类型有关的 PostgreSQL `ENUM`规则已经更加严格。

通过未明确与`MetaData`对象关联创建的`ENUM`将根据`Table.create()`和`Table.drop()`创建和删除：

```py
table = Table(
    "sometable", metadata, Column("some_enum", ENUM("a", "b", "c", name="myenum"))
)

table.create(engine)  # will emit CREATE TYPE and CREATE TABLE
table.drop(engine)  # will emit DROP TABLE and DROP TYPE - new for 1.0
```

这意味着如果第二个表也有一个名为‘myenum’的枚举，上述 DROP 操作现在将失败。为了适应共享枚举类型的用例，元数据关联的枚举行为已经得到增强。

通过与`MetaData`对象明确关联创建的`ENUM`将不会根据`Table.create()`和`Table.drop()`创建或删除，除了使用`checkfirst=True`标志调用的`Table.create()`：

```py
my_enum = ENUM("a", "b", "c", name="myenum", metadata=metadata)

table = Table("sometable", metadata, Column("some_enum", my_enum))

# will fail: ENUM 'my_enum' does not exist
table.create(engine)

# will check for enum and emit CREATE TYPE
table.create(engine, checkfirst=True)

table.drop(engine)  # will emit DROP TABLE, *not* DROP TYPE

metadata.drop_all(engine)  # will emit DROP TYPE

metadata.create_all(engine)  # will emit CREATE TYPE
```

[#3319](https://www.sqlalchemy.org/trac/ticket/3319)

### 新的 PostgreSQL 表选项

在通过`Table`构造渲染 DDL 时，添加了对 PG 表选项 TABLESPACE、ON COMMIT、WITH(OUT) OIDS 和 INHERITS 的支持。

另请参见

PostgreSQL 表选项

[#2051](https://www.sqlalchemy.org/trac/ticket/2051)

### 具有 PostgreSQL 方言的新 get_enums()方法

在 PostgreSQL 的情况下，`inspect()`方法返回一个`PGInspector`对象，其中包括一个新的`PGInspector.get_enums()`方法，返回所有可用的`ENUM`类型的信息：

```py
from sqlalchemy import inspect, create_engine

engine = create_engine("postgresql+psycopg2://host/dbname")
insp = inspect(engine)
print(insp.get_enums())
```

另请参见

`PGInspector.get_enums()`  ### PostgreSQL 方言反映了物化视图、外部表

更改如下：

+   使用 `autoload=True` 的 `Table` 构造现在将匹配数据库中存在的作为物化视图或外部表的名称。

+   `Inspector.get_view_names()` 将返回普通和物化视图的名称。

+   `Inspector.get_table_names()` 对于 PostgreSQL 不会发生变化，它继续只返回普通表的名称。

+   添加了一个新方法 `PGInspector.get_foreign_table_names()`，它将返回在 PostgreSQL 模式表中明确标记为“外部”的表的名称。

反射的变化涉及在查询 `pg_class.relkind` 时添加 `'m'` 和 `'f'` 到我们使用的修饰符列表，但这个变化是在 1.0.0 中新增的，以避免对正在生产中运行 0.9 的用户造成任何不兼容的惊喜。

[#2891](https://www.sqlalchemy.org/trac/ticket/2891)  ### PostgreSQL `has_table()` 现在适用于临时表

这是一个简单的修复，使得临时表的“has table”现在可以正常工作，因此像下面这样的代码可以继续执行：

```py
from sqlalchemy import *

metadata = MetaData()
user_tmp = Table(
    "user_tmp",
    metadata,
    Column("id", INT, primary_key=True),
    Column("name", VARCHAR(50)),
    prefixes=["TEMPORARY"],
)

e = create_engine("postgresql://scott:tiger@localhost/test", echo="debug")
with e.begin() as conn:
    user_tmp.create(conn, checkfirst=True)

    # checkfirst will succeed
    user_tmp.create(conn, checkfirst=True)
```

这种行为可能导致一个不会失败的应用程序表现不同的极不可能的情况，是因为 PostgreSQL 允许非临时表悄悄地覆盖临时表。因此，像下面这样的代码现在将完全不同，不再创建临时表后面的真实表：

```py
from sqlalchemy import *

metadata = MetaData()
user_tmp = Table(
    "user_tmp",
    metadata,
    Column("id", INT, primary_key=True),
    Column("name", VARCHAR(50)),
    prefixes=["TEMPORARY"],
)

e = create_engine("postgresql://scott:tiger@localhost/test", echo="debug")
with e.begin() as conn:
    user_tmp.create(conn, checkfirst=True)

    m2 = MetaData()
    user = Table(
        "user_tmp",
        m2,
        Column("id", INT, primary_key=True),
        Column("name", VARCHAR(50)),
    )

    # in 0.9, *will create* the new table, overwriting the old one.
    # in 1.0, *will not create* the new table
    user.create(conn, checkfirst=True)
```

[#3264](https://www.sqlalchemy.org/trac/ticket/3264)  ### PostgreSQL FILTER 关键字

SQL 标准的 FILTER 关键字现在由 PostgreSQL 支持，从 9.4 版开始。SQLAlchemy 允许使用 `FunctionElement.filter()` 来实现：

```py
func.count(1).filter(True)
```

另请参阅

`FunctionElement.filter()`

`FunctionFilter`

### PG8000 方言支持客户端编码

`create_engine.encoding` 参数现在由 pg8000 方言遵守，使用连接处理程序发出 `SET CLIENT_ENCODING` 匹配所选编码。

### PG8000 原生 JSONB 支持

添加了对大于 1.10.1 版本的 PG8000 的支持，其中原生支持 JSONB。

### 对 PyPy 上的 psycopg2cffi 方言的支持

添加了对 pypy psycopg2cffi 方言的支持。

另请参阅

`sqlalchemy.dialects.postgresql.psycopg2cffi`

### ENUM 类型创建/删除规则的全面改革

有关创建和删除 TYPE 的 PostgreSQL `ENUM`的规则已经更加严格。

一个`ENUM`如果没有明确与`MetaData`对象关联，将根据`Table.create()`和`Table.drop()`创��和删除：

```py
table = Table(
    "sometable", metadata, Column("some_enum", ENUM("a", "b", "c", name="myenum"))
)

table.create(engine)  # will emit CREATE TYPE and CREATE TABLE
table.drop(engine)  # will emit DROP TABLE and DROP TYPE - new for 1.0
```

这意味着如果第二个表也有一个名为‘myenum’的枚举，上述 DROP 操作现在将失败。为了适应共享枚举类型的使用情况，元数据关联的枚举行为已经增强。

一个`ENUM`如果明确与`MetaData`对象关联，将不会根据`Table.create()`和`Table.drop()`创建或删除，除非使用`checkfirst=True`标志调用`Table.create()`：

```py
my_enum = ENUM("a", "b", "c", name="myenum", metadata=metadata)

table = Table("sometable", metadata, Column("some_enum", my_enum))

# will fail: ENUM 'my_enum' does not exist
table.create(engine)

# will check for enum and emit CREATE TYPE
table.create(engine, checkfirst=True)

table.drop(engine)  # will emit DROP TABLE, *not* DROP TYPE

metadata.drop_all(engine)  # will emit DROP TYPE

metadata.create_all(engine)  # will emit CREATE TYPE
```

[#3319](https://www.sqlalchemy.org/trac/ticket/3319)

### 新的 PostgreSQL 表选项

添加了对 PG 表选项 TABLESPACE、ON COMMIT、WITH(OUT) OIDS 和 INHERITS 的支持，通过`Table`构造渲染 DDL 时。

另请参阅

PostgreSQL 表选项

[#2051](https://www.sqlalchemy.org/trac/ticket/2051)

### 具有 PostgreSQL 方言的新 get_enums()方法

`inspect()`方法在 PostgreSQL 情况下返回一个`PGInspector`对象，其中包括一个新的`PGInspector.get_enums()`方法，返回所有可用`ENUM`类型的信息：

```py
from sqlalchemy import inspect, create_engine

engine = create_engine("postgresql+psycopg2://host/dbname")
insp = inspect(engine)
print(insp.get_enums())
```

另请参阅

`PGInspector.get_enums()`

### PostgreSQL 方言反映了物化视图、外部表

更改如下：

+   使用 `autoload=True` 的 `Table` 构造现在将匹配数据库中存在的物化视图或外部表的名称。

+   `Inspector.get_view_names()` 将返回普通视图和物化视图的名称。

+   `Inspector.get_table_names()` 对于 PostgreSQL 不会发生变化，它仍然只返回普通表的名称。

+   添加了一个新方法 `PGInspector.get_foreign_table_names()`，它将返回在 PostgreSQL 模式表中明确标记为“外部”的表的名称。

反射的变化涉及在查询`pg_class.relkind`时将`'m'`和`'f'`添加到我们使用的限定符列表中，但这个变化是在 1.0.0 版本中新增的，以避免对于在生产环境中运行 0.9 版本的用户造成任何不兼容的惊喜。

[#2891](https://www.sqlalchemy.org/trac/ticket/2891)

### PostgreSQL `has_table()` 现在适用于临时表

这是一个简单的修复，使得临时表的“有表”现在可以工作，因此像下面这样的代码可以继续进行：

```py
from sqlalchemy import *

metadata = MetaData()
user_tmp = Table(
    "user_tmp",
    metadata,
    Column("id", INT, primary_key=True),
    Column("name", VARCHAR(50)),
    prefixes=["TEMPORARY"],
)

e = create_engine("postgresql://scott:tiger@localhost/test", echo="debug")
with e.begin() as conn:
    user_tmp.create(conn, checkfirst=True)

    # checkfirst will succeed
    user_tmp.create(conn, checkfirst=True)
```

这种行为可能导致一个不会失败的应用程序表现不同的极端情况，是因为 PostgreSQL 允许非临时表悄悄地覆盖临时表。因此，像下面这样的代码现在将完全不同，不再创建真实表来跟随临时表：

```py
from sqlalchemy import *

metadata = MetaData()
user_tmp = Table(
    "user_tmp",
    metadata,
    Column("id", INT, primary_key=True),
    Column("name", VARCHAR(50)),
    prefixes=["TEMPORARY"],
)

e = create_engine("postgresql://scott:tiger@localhost/test", echo="debug")
with e.begin() as conn:
    user_tmp.create(conn, checkfirst=True)

    m2 = MetaData()
    user = Table(
        "user_tmp",
        m2,
        Column("id", INT, primary_key=True),
        Column("name", VARCHAR(50)),
    )

    # in 0.9, *will create* the new table, overwriting the old one.
    # in 1.0, *will not create* the new table
    user.create(conn, checkfirst=True)
```

[#3264](https://www.sqlalchemy.org/trac/ticket/3264)

### PostgreSQL FILTER 关键字

作为 9.4 版本的 PostgreSQL 支持 SQL 标准的 FILTER 关键字。SQLAlchemy 允许使用 `FunctionElement.filter()`：

```py
func.count(1).filter(True)
```

另请参阅

`FunctionElement.filter()`

`FunctionFilter`

### PG8000 方言支持客户端编码

`create_engine.encoding` 参数现在受到 pg8000 方言的尊重，使用连接处理程序发出与所选编码匹配的 `SET CLIENT_ENCODING`。

### PG8000 原生 JSONB 支持

添加了对大于 1.10.1 版本的 PG8000 的支持，其中原生支持 JSONB。

### 支持在 PyPy 上的 psycopg2cffi 方言

添加了对 pypy psycopg2cffi 方言的支持。

另请参阅

`sqlalchemy.dialects.postgresql.psycopg2cffi`

## 方言改进和更改 - MySQL

### MySQL TIMESTAMP 类型现在在所有情况下都渲染为 NULL / NOT NULL

MySQL 方言始终通过为 `nullable=True` 设置的列发出 NULL 来解决与 TIMESTAMP 列关联的隐式 NOT NULL 默认值的问题。然而，MySQL 5.6.6 及以上版本具有一个新标志[explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp)，修复了 MySQL 的非标准行为，使其像任何其他类型一样运行；为了适应这一点，SQLAlchemy 现在无条件地为所有 TIMESTAMP 列发出 NULL/NOT NULL。

另请参阅

TIMESTAMP 列和 NULL

[#3155](https://www.sqlalchemy.org/trac/ticket/3155)  ### MySQL SET 类型进行了全面改进，以支持空集、unicode、空值处理

`SET` 类型历史上没有包括处理空集和空值的系统；由于不同的驱动程序对空字符串和空字符串集表示的处理方式不同，因此 SET 类型尝试只在这些行为之间进行权衡，选择将空集视为 `set([''])`，这仍然是 MySQL-Connector-Python DBAPI 的当前行为。其中部分理由是否则无法实际在 MySQL SET 中存储空字符串，因为驱动程序返回没有办法区分 `set([''])` 和 `set()` 的字符串。留给用户确定 `set([''])` 实际上是否意味着“空集”或其他情况。

新行为将使用空字符串的用例（这是一个不寻常的情况，甚至在 MySQL 的文档中都没有记录），移入特殊情况中，而`SET`的默认行为现在是：

+   将由 MySQL-python 返回的空字符串 `''` 视为空集 `set()`；

+   将由 MySQL-Connector-Python 返回的单空值集 `set([''])` 转换为空集 `set()`；

+   为了处理实际希望包含空值 `''` 在其可能值列表中的集合类型的情况，实施了一个新功能（在这种情况下是必需的），即将集合值持久化和加载为位整数值；添加了标志 `SET.retrieve_as_bitwise` 以启用此功能。

使用 `SET.retrieve_as_bitwise` 标志允许集合以无歧义的值进行持久化和检索。理论上，只要给定的值列表与数据库中声明的顺序完全匹配，就可以在所有情况下打开此标志；它只是使 SQL 回显输出有点不寻常。

否则，`SET` 的默认行为保持不变，使用字符串循环传递值。基于字符串的行为现在完全支持 Unicode，包括 MySQL-python 使用 `use_unicode=0`。

[#3283](https://www.sqlalchemy.org/trac/ticket/3283)

### MySQL 内部的“无此表”异常不会传递给事件处理程序

MySQL 方言现在将禁用 `ConnectionEvents.handle_error()` 事件，以防止这些语句触发用于内部检测表是否存在的事件。这是通过使用一个执行选项 `skip_user_error_events` 实现的，该选项在该执行范围内禁用了处理错误事件。这样，重写异常的用户代码不需要担心 MySQL 方言或其他偶尔需要捕获 SQLAlchemy 特定异常的方言。

### 更改了 MySQL-Connector 的 `raise_on_warnings` 的默认值

将 MySQL-Connector 的 “raise_on_warnings” 的默认值更改为 False。出于某种原因，它被设置为 True。不幸的是，“buffered” 标志必须保持为 True，因为 MySQLconnector 不允许关闭游标，除非所有结果都被完全获取。

[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### MySQL 布尔符号“true”、“false”再次有效

0.9 版本对 IS/IS NOT 操作符以及 [#2682](https://www.sqlalchemy.org/trac/ticket/2682) 中的布尔类型进行了彻底改造，禁止 MySQL 方言在“IS”/“IS NOT”的上下文中使用“true”和“false”符号。显然，即使 MySQL 没有“布尔”类型，但当使用特殊的“true”和“false”符号时，它支持 IS/IS NOT，尽管这些符号在其他情况下与“1”和“0”是同义词（并且 IS/IS NOT 与数字不兼容）。

因此，这里的更改是 MySQL 方言仍然是“非本地布尔”，但 `true()` 和 `false()` 符号再次生成关键字“true”和“false”，因此像 `column.is_(true())` 这样的表达式再次在 MySQL 上起作用。

[#3186](https://www.sqlalchemy.org/trac/ticket/3186)  ### `match()` 操作符现在返回与 MySQL 浮点返回值兼容的不可知 MatchType

`ColumnOperators.match()`表达式的返回类型现在是一个称为`MatchType`的新类型。这是`Boolean`的子类，可以被方言拦截，以在 SQL 执行时产生不同的结果类型。

类似以下代码现在将在 MySQL 上正确运行并返回浮点数：

```py
>>> connection.execute(
...     select(
...         [
...             matchtable.c.title.match("Agile Ruby Programming").label("ruby"),
...             matchtable.c.title.match("Dive Python").label("python"),
...             matchtable.c.title,
...         ]
...     ).order_by(matchtable.c.id)
... )
[
 (2.0, 0.0, 'Agile Web Development with Ruby On Rails'),
 (0.0, 2.0, 'Dive Into Python'),
 (2.0, 0.0, "Programming Matz's Ruby"),
 (0.0, 0.0, 'The Definitive Guide to Django'),
 (0.0, 1.0, 'Python in a Nutshell')
]
```

[#3263](https://www.sqlalchemy.org/trac/ticket/3263)  ### Drizzle Dialect is now an External Dialect

[Drizzle](https://www.drizzle.org/)的方言现在是一个外部方言，可在[`bitbucket.org/zzzeek/sqlalchemy-drizzle`](https://bitbucket.org/zzzeek/sqlalchemy-drizzle)上找到。这个方言是在 SQLAlchemy 能够很好地适应第三方方言之前添加到 SQLAlchemy 的；未来，所有不属于“普遍使用”类别的数据库都是第三方方言。该方言的实现没有改变，仍然基于 SQLAlchemy 中的 MySQL + MySQLdb 方言。该方言目前尚未发布，处于“attic”状态；但是它通过了大部分测试，一般工作正常，如果有人想要继续完善它的话。  ### MySQL TIMESTAMP Type now renders NULL / NOT NULL in all cases

MySQL 方言一直通过在`nullable=True`的情况下为 TIMESTAMP 列发出 NULL 来解决 MySQL 隐式 NOT NULL 默认值的问题。然而，MySQL 5.6.6 及以上版本引入了一个新标志[explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp)，修复了 MySQL 的非标准行为，使其表现得像任何其他类型；为了适应这一点，SQLAlchemy 现在无条件地为所有 TIMESTAMP 列发出 NULL/NOT NULL。

另请参见

TIMESTAMP Columns and NULL

[#3155](https://www.sqlalchemy.org/trac/ticket/3155)

### MySQL SET Type Overhauled to support empty sets, unicode, blank value handling

`SET`类型历史上不包括处理空集和空值的系统；由于不同的驱动程序对空字符串和空字符串集表示的处理方式不同，因此 SET 类型仅尝试在这些行为之间做出取舍，选择将空集视为`set([''])`，这仍然是 MySQL-Connector-Python DBAPI 的当前行为。这样做的部分原因是，否则无法实际在 MySQL SET 中存储空字符串，因为驱动程序返回的字符串没有办法区分`set([''])`和`set()`之间的区别。用户需要确定`set([''])`是否实际上表示“空集”。

新行为将空字符串的使用情况移至一个特殊情况，这是一个不常见的情况，甚至在 MySQL 的文档中也没有记录，而`SET`的默认行为现在是：

+   将由 MySQL-python 返回的空字符串`''`视为空集`set()`；

+   将 MySQL-Connector-Python 返回的单个空值集`set([''])`转换为空集`set()`。

+   为了处理实际希望在其可能值列表中包含空值`''`的集合类型的情况，实现了一个新特性（在此用例中需要），即集合值被持久化并作为位整数值加载；添加了标志`SET.retrieve_as_bitwise`以启用此功能。

使用标志`SET.retrieve_as_bitwise`允许集合被持久化和检索而没有值的歧义。从理论上讲，只要类型的给定值列表与数据库中声明的顺序完全匹配，就可以在所有情况下打开此标志；它只会使 SQL 回显输出略显不同寻常。

否则，`SET`的默认行为保持不变，使用字符串往返值。基于字符串的行为现在完全支持 Unicode，包括 MySQL-python，并且使用`use_unicode=0`。

[#3283](https://www.sqlalchemy.org/trac/ticket/3283)

### MySQL 内部的“无此表”异常不会传递给事件处理程序。

MySQL 方言现在将禁用 `ConnectionEvents.handle_error()` 事件对内部用于检测表是否存在的语句触发。这是通过使用一个执行选项 `skip_user_error_events` 来实现的，该选项在该执行范围内禁用了处理错误事件。这样，重写异常的用户代码不需要担心 MySQL 方言或其他偶尔需要捕获 SQLAlchemy 特定异常的方言。

### 更改了 MySQL-Connector 的 `raise_on_warnings` 默认值

将 MySQL-Connector 的“raise_on_warnings”默认值更改为 False。由于某种原因，这被设置为 True。不幸的是，“buffered” 标志必须保持为 True，因为 MySQLconnector 不允许关闭游标，除非所有结果都被完全获取。

[#2515](https://www.sqlalchemy.org/trac/ticket/2515)

### MySQL 布尔符号“true”、“false”再次生效

0.9 版本对 IS/IS NOT 操作符以及布尔类型的改进在 [#2682](https://www.sqlalchemy.org/trac/ticket/2682) 中禁止了 MySQL 方言在“IS”/“IS NOT”上使用“true”和“false”符号。显然，即使 MySQL 没有“布尔”类型，但在使用特殊的“true”和“false”符号时，它支持 IS/IS NOT，尽管这些符号在其他情况下与“1”和“0”是同义的（并且 IS/IS NOT 不能与数字一起使用）。

这里的变化是 MySQL 方言仍然保持“非本地布尔”，但 `true()` 和 `false()` 符号再次产生关键字“true”和“false”，因此像 `column.is_(true())` 这样的表达式在 MySQL 上再次生效。

[#3186](https://www.sqlalchemy.org/trac/ticket/3186)

### `match()` 操作符现在返回一个与 MySQL 浮点返回值兼容的不可知的 MatchType

`ColumnOperators.match()` 表达式的返回类型现在是一个称为 `MatchType` 的新类型。这是 `Boolean` 的子类，可以被方言拦截以在 SQL 执行时产生不同的结果类型。

类似以下代码现在将正确运行并在 MySQL 上返回浮点数：

```py
>>> connection.execute(
...     select(
...         [
...             matchtable.c.title.match("Agile Ruby Programming").label("ruby"),
...             matchtable.c.title.match("Dive Python").label("python"),
...             matchtable.c.title,
...         ]
...     ).order_by(matchtable.c.id)
... )
[
 (2.0, 0.0, 'Agile Web Development with Ruby On Rails'),
 (0.0, 2.0, 'Dive Into Python'),
 (2.0, 0.0, "Programming Matz's Ruby"),
 (0.0, 0.0, 'The Definitive Guide to Django'),
 (0.0, 1.0, 'Python in a Nutshell')
]
```

[#3263](https://www.sqlalchemy.org/trac/ticket/3263)

### Drizzle 方言现在是外部方言

[Drizzle](https://www.drizzle.org/) 的方言现在是一个外部方言，可在 [`bitbucket.org/zzzeek/sqlalchemy-drizzle`](https://bitbucket.org/zzzeek/sqlalchemy-drizzle) 上获得。这个方言是在 SQLAlchemy 能够很好地适应第三方方言之前添加到 SQLAlchemy 的；未来，所有不属于“普遍使用”类别的数据库都是第三方方言。方言的实现没有改变，仍然基于 SQLAlchemy 中的 MySQL + MySQLdb 方言。该方言尚未发布，处于“attic”状态；但是它通过了大部分测试，通常工作正常，如果有人想要继续完善它。

## 方言改进和更改 - SQLite

### SQLite 命名和未命名的 UNIQUE 和 FOREIGN KEY 约束将进行检查和反映

在 SQLite 上，现在完全反映了具有和不具有名称的 UNIQUE 和 FOREIGN KEY 约束。以前，外键名称被忽略，未命名的唯一约束被跳过。特别是这将有助于 Alembic 的新 SQLite 迁移功能。

为了实现这一点，对于外键和唯一约束，PRAGMA foreign_keys、index_list 和 index_info 的结果与对 CREATE TABLE 语句的正则表达式解析相结合，以形成对约束名称的完整了解，以及区分作为 UNIQUE 创建的 UNIQUE 约束与未命名的 INDEX。

[#3244](https://www.sqlalchemy.org/trac/ticket/3244)

[#3261](https://www.sqlalchemy.org/trac/ticket/3261)

### SQLite 命名和未命名的 UNIQUE 和 FOREIGN KEY 约束将进行检查和反映

在 SQLite 上，现在完全反映了具有和不具有名称的 UNIQUE 和 FOREIGN KEY 约束。以前，外键名称被忽略，未命名的唯一约束被跳过。特别是这将有助于 Alembic 的新 SQLite 迁移功能。

为了实现这一点，对于外键和唯一约束，PRAGMA foreign_keys、index_list 和 index_info 的结果与对 CREATE TABLE 语句的正则表达式解析相结合，以形成对约束名称的完整了解，以及区分作为 UNIQUE 创建的 UNIQUE 约束与未命名的 INDEX。

[#3244](https://www.sqlalchemy.org/trac/ticket/3244)

[#3261](https://www.sqlalchemy.org/trac/ticket/3261)

## 方言改进和更改 - SQL Server

### 需要在基于主机名的 SQL Server 连接中提供 PyODBC 驱动程序名称

使用无 DSN 连接的 PyODBC 连接到 SQL Server，例如使用显式主机名，现在需要提供驱动程序名称 - SQLAlchemy 将不再尝试猜测默认值：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=SQL+Server+Native+Client+10.0"
)
```

SQLAlchemy 在 Windows 上以前硬编码的默认值“SQL Server”已经过时，SQLAlchemy 不能根据操作系统/驱动程序检测来猜测最佳驱动程序。在使用 ODBC 时，始终首选使用 DSN 来避免这个问题。

[#3182](https://www.sqlalchemy.org/trac/ticket/3182)

### SQL Server 2012 大型文本/二进制类型呈现为 VARCHAR、NVARCHAR、VARBINARY

SQL Server 2012 及更高版本的 `TextClause`、`UnicodeText` 和 `LargeBinary` 类型的呈现已更改，完全控制行为的选项，基于微软的弃用指南。详情请参阅 大型文本/二进制类型弃用。

### 在基于主机名的 SQL Server 连接中需要 PyODBC 驱动程序名称

使用无 DSN 连接的方式连接到 SQL Server，例如使用显式主机名，现在需要驱动程序名称 - SQLAlchemy 不再尝试猜测默认值：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=SQL+Server+Native+Client+10.0"
)
```

SQLAlchemy 在 Windows 上先前固定的默认值“SQL Server”已经过时，不能根据操作系统/驱动程序检测猜测最佳驱动程序。使用 DSN 总是首选，以避免此问题。

[#3182](https://www.sqlalchemy.org/trac/ticket/3182)

### SQL Server 2012 大型文本/二进制类型呈现为 VARCHAR、NVARCHAR、VARBINARY

SQL Server 2012 及更高版本的 `TextClause`、`UnicodeText` 和 `LargeBinary` 类型的呈现已更改，完全控制行为的选项，基于微软的弃用指南。详情请参阅 大型文本/二进制类型弃用。

## Oracle 方言的改进和变化 - Oracle

### Oracle 中 CTE 的改进支持

Oracle 的 CTE 支持已经修复，并且还有一个新功能 `CTE.with_suffixes()` 可以帮助处理 Oracle 的特殊指令：

```py
included_parts = (
    select([part.c.sub_part, part.c.part, part.c.quantity])
    .where(part.c.part == "p1")
    .cte(name="included_parts", recursive=True)
    .suffix_with(
        "search depth first by part set ord1",
        "cycle part set y_cycle to 1 default 0",
        dialect="oracle",
    )
)
```

[#3220](https://www.sqlalchemy.org/trac/ticket/3220)

### 新的 Oracle DDL 关键字

关键词如 COMPRESS、ON COMMIT、BITMAP：

Oracle 表选项

Oracle 特定索引选项

### Oracle 中 CTE 的改进支持

Oracle 的 CTE 支持已经修复，并且还有一个新功能 `CTE.with_suffixes()` 可以帮助处理 Oracle 的特殊指令：

```py
included_parts = (
    select([part.c.sub_part, part.c.part, part.c.quantity])
    .where(part.c.part == "p1")
    .cte(name="included_parts", recursive=True)
    .suffix_with(
        "search depth first by part set ord1",
        "cycle part set y_cycle to 1 default 0",
        dialect="oracle",
    )
)
```

[#3220](https://www.sqlalchemy.org/trac/ticket/3220)

### 新的 Oracle DDL 关键字

关键词如 COMPRESS、ON COMMIT、BITMAP：

Oracle 表选项

Oracle 特定索引选项
