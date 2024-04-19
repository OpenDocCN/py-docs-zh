# SQLAlchemy 0.8 中的新功能是什么？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_08.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_08.html)

关于本文档

本文档描述了 SQLAlchemy 版本 0.7（截至 2012 年 10 月正在进行维护发布）和 SQLAlchemy 版本 0.8（预计于 2013 年初发布）之间的更改。

文档日期：2012 年 10 月 25 日 更新日期：2013 年 3 月 9 日

## 介绍

本指南介绍了 SQLAlchemy 版本 0.8 中的新功能，并记录了影响用户将应用程序从 SQLAlchemy 0.7 系列迁移到 0.8 的更改。

SQLAlchemy 发布即将接近 1.0，自 0.5 以来的每个新版本都减少了主要的使用更改。大多数已经适应现代 0.7 模式的应用程序应该可以无需更改地迁移到 0.8。使用 0.6 甚至 0.5 模式的应用程序也应该可以直接迁移到 0.8，尽管较大的应用程序可能希望在每个中间版本中进行测试。

## 平台支持

### 现在面向 Python 2.5 及更高版本

SQLAlchemy 0.8 将面向 Python 2.5 及更高版本；对 Python 2.4 的兼容性将被删除。

内部将能够使用 Python 三元表达式（即，`x if y else z`），这将改善与使用`y and x or z`相比的情况，后者自然会导致一些错误，以及上下文管理器（即，`with:`）和在某些情况下可能会有助于代码可读性的`try:/except:/else:`块。

SQLAlchemy 最终将放弃对 2.5 的支持 - 当达到 2.6 作为基线时，SQLAlchemy 将转而使用 2.6/3.3 的就地兼容性，删除`2to3`工具的使用，并保持一个同时与 Python 2 和 3 兼容的源代码库。

## 新的 ORM 功能

### 重写的`relationship()`机制

0.8 版本在`relationship()`如何确定两个实体之间如何连接方面具有更加改进和强大的系统。新系统包括以下功能：

+   在构建针对具有多个外键路径指向目标的类的`relationship()`时，**不再需要**`primaryjoin`参数。只需要`foreign_keys`参数来指定应包括的列：

    ```py
    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        child_id_one = Column(Integer, ForeignKey("child.id"))
        child_id_two = Column(Integer, ForeignKey("child.id"))

        child_one = relationship("Child", foreign_keys=child_id_one)
        child_two = relationship("Child", foreign_keys=child_id_two)

    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
    ```

+   支持自引用、复合外键的关系，其中**一列指向自身**。典型案例如下：

    ```py
    class Folder(Base):
        __tablename__ = "folder"
        __table_args__ = (
            ForeignKeyConstraint(
                ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
            ),
        )

        account_id = Column(Integer, primary_key=True)
        folder_id = Column(Integer, primary_key=True)
        parent_id = Column(Integer)
        name = Column(String)

        parent_folder = relationship(
            "Folder", backref="child_folders", remote_side=[account_id, folder_id]
        )
    ```

    上面，`Folder`指的是从`account_id`到其自身的父`Folder`的连接，并且`parent_id`到`folder_id`的连接。当 SQLAlchemy 构造自动连接时，不能再假设“远程”侧的所有列都被别名化，而“本地”侧的所有列都没有被别名化 - `account_id`列**在两侧都存在**。因此，内部关系机制完全重写，以支持一个完全不同的系统，其中生成了两个副本的`account_id`，每个副本包含不同的*注释*以确定它们在语句中的作用。注意基本急加载中的连接条件：

    ```py
    SELECT
      folder.account_id  AS  folder_account_id,
      folder.folder_id  AS  folder_folder_id,
      folder.parent_id  AS  folder_parent_id,
      folder.name  AS  folder_name,
      folder_1.account_id  AS  folder_1_account_id,
      folder_1.folder_id  AS  folder_1_folder_id,
      folder_1.parent_id  AS  folder_1_parent_id,
      folder_1.name  AS  folder_1_name
    FROM  folder
      LEFT  OUTER  JOIN  folder  AS  folder_1
      ON
      folder_1.account_id  =  folder.account_id
      AND  folder.folder_id  =  folder_1.parent_id

    WHERE  folder.folder_id  =  ?  AND  folder.account_id  =  ?
    ```

+   以前的复杂自定义连接条件，比如涉及函数和/或类型转换（CASTing）的条件，现在在大多数情况下都将按预期运行：

    ```py
    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() using explicit foreign_keys, remote_side
        parent_host = relationship(
            "HostEntry",
            primaryjoin=ip_address == cast(content, INET),
            foreign_keys=content,
            remote_side=ip_address,
        )
    ```

    新的`relationship()`机制利用了 SQLAlchemy 的一个概念，称为注释。这些注释也可以通过`foreign()`和`remote()`函数明确提供给应用代码，无论是为了提高高级配置的可读性，还是直接注入一个精确的配置，绕过通常的连接检查启发式方法：

    ```py
    from sqlalchemy.orm import foreign, remote

    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() using explicit foreign() and remote() annotations
        # in lieu of separate arguments
        parent_host = relationship(
            "HostEntry",
            primaryjoin=remote(ip_address) == cast(foreign(content), INET),
        )
    ```

另请参阅

配置关系连接方式 - 一个重新修订的关于`relationship()`的部分，详细说明了定制相关属性和集合访问的最新技术。

[#1401](https://www.sqlalchemy.org/trac/ticket/1401) [#610](https://www.sqlalchemy.org/trac/ticket/610)  ### 新的类/对象检查系统

许多 SQLAlchemy 用户正在编写需要检查映射类属性的系统，包括能够获取主键列、对象关系、普通属性等等，通常是为了构建数据编组系统，比如 JSON/XML 转换方案以及各种表单库。

最初，`Table`和`Column`模型是最初的检查点，具有完整的文档系统。虽然 SQLAlchemy ORM 模型也是完全可自省的，但这从来都不是一个完全稳定和受支持的功能，用户往往不清楚如何获取这些信息。

0.8 现在为此提供了一致、稳定且完全文档化的 API，包括一个检查系统，该系统适用于映射类、实例、属性以及其他核心和 ORM 构造。该系统的入口点是核心级的 `inspect()` 函数。在大多数情况下，被检查的对象已经是 SQLAlchemy 系统的一部分，例如 `Mapper`、`InstanceState`、`Inspector`。在某些情况下，已经添加了新对象，其工作是在某些上下文中提供检查 API，例如 `AliasedInsp` 和 `AttributeState`。

以下是一些关键功能的介绍：

```py
>>> class User(Base):
...     __tablename__ = "user"
...     id = Column(Integer, primary_key=True)
...     name = Column(String)
...     name_syn = synonym(name)
...     addresses = relationship("Address")

>>> # universal entry point is inspect()
>>> b = inspect(User)

>>> # b in this case is the Mapper
>>> b
<Mapper at 0x101521950; User>

>>> # Column namespace
>>> b.columns.id
Column('id', Integer(), table=<user>, primary_key=True, nullable=False)

>>> # mapper's perspective of the primary key
>>> b.primary_key
(Column('id', Integer(), table=<user>, primary_key=True, nullable=False),)

>>> # MapperProperties available from .attrs
>>> b.attrs.keys()
['name_syn', 'addresses', 'id', 'name']

>>> # .column_attrs, .relationships, etc. filter this collection
>>> b.column_attrs.keys()
['id', 'name']

>>> list(b.relationships)
[<sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>]

>>> # they are also namespaces
>>> b.column_attrs.id
<sqlalchemy.orm.properties.ColumnProperty object at 0x101525090>

>>> b.relationships.addresses
<sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>

>>> # point inspect() at a mapped, class level attribute,
>>> # returns the attribute itself
>>> b = inspect(User.addresses)
>>> b
<sqlalchemy.orm.attributes.InstrumentedAttribute object at 0x101521fd0>

>>> # From here we can get the mapper:
>>> b.mapper
<Mapper at 0x101525810; Address>

>>> # the parent inspector, in this case a mapper
>>> b.parent
<Mapper at 0x101521950; User>

>>> # an expression
>>> print(b.expression)
"user".id  =  address.user_id
>>> # inspect works on instances
>>> u1 = User(id=3, name="x")
>>> b = inspect(u1)

>>> # it returns the InstanceState
>>> b
<sqlalchemy.orm.state.InstanceState object at 0x10152bed0>

>>> # similar attrs accessor refers to the
>>> b.attrs.keys()
['id', 'name_syn', 'addresses', 'name']

>>> # attribute interface - from attrs, you get a state object
>>> b.attrs.id
<sqlalchemy.orm.state.AttributeState object at 0x10152bf90>

>>> # this object can give you, current value...
>>> b.attrs.id.value
3

>>> # ... current history
>>> b.attrs.id.history
History(added=[3], unchanged=(), deleted=())

>>> # InstanceState can also provide session state information
>>> # lets assume the object is persistent
>>> s = Session()
>>> s.add(u1)
>>> s.commit()

>>> # now we can get primary key identity, always
>>> # works in query.get()
>>> b.identity
(3,)

>>> # the mapper level key
>>> b.identity_key
(<class '__main__.User'>, (3,))

>>> # state within the session
>>> b.persistent, b.transient, b.deleted, b.detached
(True, False, False, False)

>>> # owning session
>>> b.session
<sqlalchemy.orm.session.Session object at 0x101701150>
```

另请参见

运行时检查 API

[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

### 新的 with_polymorphic() 功能，可以在任何地方使用

`Query.with_polymorphic()` 方法允许用户指定在针对联接表实体进行查询时应该存在哪些表。不幸的是，该方法很笨拙，只适用于列表中的第一个实体，而且在使用和内部方面都有一些尴尬的行为。现在已经添加了一个新的增强功能到 `aliased()` 构造中，称为 `with_polymorphic()`，它允许任何实体被“别名”为其自身的“多态”版本，可以自由地在任何地方使用：

```py
from sqlalchemy.orm import with_polymorphic

palias = with_polymorphic(Person, [Engineer, Manager])
session.query(Company).join(palias, Company.employees).filter(
    or_(Engineer.language == "java", Manager.hair == "pointy")
)
```

另请参见

使用 with_polymorphic() - 用于多态加载控制的最新更新文档。

[#2333](https://www.sqlalchemy.org/trac/ticket/2333)

### of_type() 与 alias()、with_polymorphic()、any()、has()、joinedload()、subqueryload()、contains_eager() 配合使用

`PropComparator.of_type()` 方法用于在构建 SQL 表达式时指定要使用的特定子类型，该子类型作为其目标具有 多态 映射的 `relationship()` 的目标。现在可以使用该方法来针对 *任意数量* 的目标子类型，通过与新的 `with_polymorphic()` 函数结合使用：

```py
# use eager loading in conjunction with with_polymorphic targets
Job_P = with_polymorphic(Job, [SubJob, ExtraJob], aliased=True)
q = (
    s.query(DataContainer)
    .join(DataContainer.jobs.of_type(Job_P))
    .options(contains_eager(DataContainer.jobs.of_type(Job_P)))
)
```

该方法现在在大多数接受常规关系属性的地方同样有效，包括与`joinedload()`、`subqueryload()`、`contains_eager()`等加载器函数以及与`PropComparator.any()`和`PropComparator.has()`等比较方法一起：

```py
# use eager loading in conjunction with with_polymorphic targets
Job_P = with_polymorphic(Job, [SubJob, ExtraJob], aliased=True)
q = (
    s.query(DataContainer)
    .join(DataContainer.jobs.of_type(Job_P))
    .options(contains_eager(DataContainer.jobs.of_type(Job_P)))
)

# pass subclasses to eager loads (implicitly applies with_polymorphic)
q = s.query(ParentThing).options(
    joinedload_all(ParentThing.container, DataContainer.jobs.of_type(SubJob))
)

# control self-referential aliasing with any()/has()
Job_A = aliased(Job)
q = (
    s.query(Job)
    .join(DataContainer.jobs)
    .filter(
        DataContainer.jobs.of_type(Job_A).any(
            and_(Job_A.id < Job.id, Job_A.type == "fred")
        )
    )
)
```

另请参见

加入特定子类型或 with_polymorphic()实体

[#2438](https://www.sqlalchemy.org/trac/ticket/2438) [#1106](https://www.sqlalchemy.org/trac/ticket/1106)

### 事件可应用于未映射的超类

现在可以将映射器和实例事件与未映射的超类关联，这些事件将随着子类映射而传播。应使用`propagate=True`标志。此功能允许将事件与声明性基类关联：

```py
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

@event.listens_for("load", Base, propagate=True)
def on_load(target, context):
    print("New instance loaded:", target)

# on_load() will be applied to SomeClass
class SomeClass(Base):
    __tablename__ = "sometable"

    # ...
```

[#2585](https://www.sqlalchemy.org/trac/ticket/2585)

### Declarative 区分模块/包

Declarative 的一个关键特性是能够使用它们的字符串名称引用其他映射类。类名注册表现在对给定类的拥有模块和包敏感。这些类可以在表达式中通过点名引用：

```py
class Snack(Base):
    # ...

    peanuts = relationship(
        "nuts.Peanut", primaryjoin="nuts.Peanut.snack_id == Snack.id"
    )
```

该解析允许使用任何完整或部分消歧义的包名称。如果到特定类的路径仍然模糊，将引发错误。

[#2338](https://www.sqlalchemy.org/trac/ticket/2338)

### Declarative 中的新 DeferredReflection 功能

“延迟反射”示例已移至 Declarative 中的受支持功能。此功能允许仅使用占位符`Table`元数据构建声明性映射类，直到调用`prepare()`步骤，给定一个`Engine`，以完全反射所有表并建立实际映射。该系统支持列的覆盖、单一和联接继承，以及每个引擎的不同基础。现在可以在一个步骤中针对现有表创建完整的声明性配置：

```py
class ReflectedOne(DeferredReflection, Base):
    __abstract__ = True

class ReflectedTwo(DeferredReflection, Base):
    __abstract__ = True

class MyClass(ReflectedOne):
    __tablename__ = "mytable"

class MyOtherClass(ReflectedOne):
    __tablename__ = "myothertable"

class YetAnotherClass(ReflectedTwo):
    __tablename__ = "yetanothertable"

ReflectedOne.prepare(engine_one)
ReflectedTwo.prepare(engine_two)
```

另请参见

`DeferredReflection`

[#2485](https://www.sqlalchemy.org/trac/ticket/2485)

### ORM 类现在被核心构造所接受

虽然与`Query.filter()`一起使用的 SQL 表达式，如`User.id == 5`，一直与核心构造兼容，例如`select()`，但当传递给`select()`、`Select.select_from()`或`Select.correlate()`时，映射类本身将不被识别。一个新的 SQL 注册系统允许一个映射类作为核心中的 FROM 子句被接受：

```py
from sqlalchemy import select

stmt = select([User]).where(User.id == 5)
```

上面，映射的`User`类将扩展为`Table`，`User`被映射到其中的表。

[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

### Query.update()支持 UPDATE..FROM

新的 UPDATE..FROM 机制适用于 query.update()。下面，我们对`SomeEntity`执行 UPDATE 操作，添加一个 FROM 子句（或等效的，取决于后端）对`SomeOtherEntity`：

```py
query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
    SomeOtherEntity.foo == "bar"
).update({"data": "x"})
```

特别地，支持对连接继承实体的更新，前提是 UPDATE 的目标是本地表上的表，或者如果父表和子表混合，它们在查询中被显式连接。下面，假设`Engineer`是`Person`的一个连接子类：

```py
query(Engineer).filter(Person.id == Engineer.id).filter(
    Person.name == "dilbert"
).update({"engineer_data": "java"})
```

将产生：

```py
UPDATE  engineer  SET  engineer_data='java'  FROM  person
WHERE  person.id=engineer.id  AND  person.name='dilbert'
```

[#2365](https://www.sqlalchemy.org/trac/ticket/2365)

### rollback()仅会回滚从 begin_nested()开始的“脏”对象

一项行为变更应该提高那些通过`Session.begin_nested()`使用 SAVEPOINT 的用户的效率 - 在`rollback()`时，只有自上次刷新以来被标记为脏的对象将被过期，其余的`Session`保持不变。这是因为对 SAVEPOINT 的 ROLLBACK 不会终止包含事务的隔离，因此除了当前事务中未刷新的更改外，不需要过期。

[#2452](https://www.sqlalchemy.org/trac/ticket/2452)

### 缓存示例现在使用 dogpile.cache

缓存示例现在使用[dogpile.cache](https://dogpilecache.readthedocs.io/)。Dogpile.cache 是 Beaker 缓存部分的重写，具有更简单和更快的操作，以及支持分布式锁定。

请注意，Dogpile 示例以及之前的 Beaker 示例中使用的 SQLAlchemy API 略有变化，特别是需要如 Beaker 示例中所示的这种变化：

```py
--- examples/beaker_caching/caching_query.py
+++ examples/beaker_caching/caching_query.py
@@ -222,7 +222,8 @@

         """
         if query._current_path:
-            mapper, key = query._current_path[-2:]
+            mapper, prop = query._current_path[-2:]
+            key = prop.key

             for cls in mapper.class_.__mro__:
                 if (cls, key) in self._relationship_options:
```

另请参见

Dogpile 缓存

[#2589](https://www.sqlalchemy.org/trac/ticket/2589)

## 新的核心功能

### 完全可扩展，核心中支持类型级别的操作符

到目前为止，Core 从未有过任何系统来为 Column 和其他表达式构造添加对新 SQL 操作符的支持，除了`ColumnOperators.op()` 方法，这个方法“刚好”能使事情正常工作。此外，Core 中也从未存在过任何允许覆盖现有操作符行为的系统。直到现在，操作符能够灵活重新定义的唯一方式是在 ORM 层，使用给定 `comparator_factory` 参数的 `column_property()`。因此，像 GeoAlchemy 这样的第三方库被迫以 ORM 为中心，并依赖于一系列的黑客技巧来应用新的操作以及正确地传播它们。

Core 中的新操作符系统增加了一直缺失的一个关键点，即将新的和被覆盖的操作符与 *类型* 关联起来。毕竟，真正驱动操作存在的不是列、CAST 操作符或 SQL 函数，而是表达式的 *类型*。实现细节很少——只需向核心 `ColumnElement` 类型添加几个额外的方法，以便它向其 `TypeEngine` 对象咨询可选的一组操作符。新的或修订过的操作可以与任何类型关联，可以通过对现有类型进行子类化、使用 `TypeDecorator`，或者通过将新的 `Comparator` 对象附加到现有类型类来“全面覆盖”地关联。

例如，要为 `Numeric` 类型添加对数支持：

```py
from sqlalchemy.types import Numeric
from sqlalchemy.sql import func

class CustomNumeric(Numeric):
    class comparator_factory(Numeric.Comparator):
        def log(self, other):
            return func.log(self.expr, other)
```

新类型可以像任何其他类型一样使用：

```py
data = Table(
    "data",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x", CustomNumeric(10, 5)),
    Column("y", CustomNumeric(10, 5)),
)

stmt = select([data.c.x.log(data.c.y)]).where(data.c.x.log(2) < value)
print(conn.execute(stmt).fetchall())
```

从这里产生的新功能包括对 PostgreSQL 的 HSTORE 类型的支持，以及与 PostgreSQL 的 ARRAY 类型相关的新操作。它还为现有类型铺平了道路，使其能够获取更多特定于这些类型的运算符，例如更多的字符串、整数和日期运算符。

另请参阅

重新定义和创建新的操作符

`HSTORE`

[#2547](https://www.sqlalchemy.org/trac/ticket/2547)

### 对插入的多值支持

`Insert.values()` 方法现在支持字典列表，将呈现多 VALUES 语句，如 `VALUES (<row1>), (<row2>), ...`。这仅适用于支持此语法的后端，包括 PostgreSQL、SQLite 和 MySQL。这与通常的 `executemany()` 样式的 INSERT 不同：

```py
users.insert().values(
    [
        {"name": "some name"},
        {"name": "some other name"},
        {"name": "yet another name"},
    ]
)
```

另请参阅

`Insert.values()`

[#2623](https://www.sqlalchemy.org/trac/ticket/2623)

### 类型表达式

现在可以将 SQL 表达式与类型关联起来。从历史上看，`TypeEngine` 一直允许 Python 端函数接收绑定参数和结果行值，通过 Python 端转换函数来回传递到/从数据库。新功能允许类似的功能，但在数据库端实现：

```py
from sqlalchemy.types import String
from sqlalchemy import func, Table, Column, MetaData

class LowerString(String):
    def bind_expression(self, bindvalue):
        return func.lower(bindvalue)

    def column_expression(self, col):
        return func.lower(col)

metadata = MetaData()
test_table = Table("test_table", metadata, Column("data", LowerString))
```

在上面的例子中，`LowerString` 类型定义了一个 SQL 表达式，每当 `test_table.c.data` 列在 SELECT 语句的列子句中呈现时，该表达式将被发出：

```py
>>> print(select([test_table]).where(test_table.c.data == "HI"))
SELECT  lower(test_table.data)  AS  data
FROM  test_table
WHERE  test_table.data  =  lower(:data_1) 
```

这一功能也被新版的 GeoAlchemy 大量使用，可以根据类型规则在 SQL 中内联嵌入 PostGIS 表达式。

另请参阅

应用 SQL 级别的绑定/结果处理

[#1534](https://www.sqlalchemy.org/trac/ticket/1534)

### 核心检查系统

New Class/Object Inspection System 中引入的 `inspect()` 函数也适用于核心。应用于一个 `Engine` 会产生一个 `Inspector` 对象：

```py
from sqlalchemy import inspect
from sqlalchemy import create_engine

engine = create_engine("postgresql://scott:tiger@localhost/test")
insp = inspect(engine)
print(insp.get_table_names())
```

它也可以应用于任何 `ClauseElement`，它返回 `ClauseElement` 本身，比如 `Table`，`Column`，`Select` 等。这使得它可以在核心和 ORM 构造之间流畅工作。

### 新方法 `Select.correlate_except()`

`select()` 现在有一个方法 `Select.correlate_except()`，指定“除了指定的所有 FROM 子句之外的相关性”。它可用于映射场景，其中相关子查询应该正常关联，除了针对特定目标可选择的情况：

```py
class SnortEvent(Base):
    __tablename__ = "event"

    id = Column(Integer, primary_key=True)
    signature = Column(Integer, ForeignKey("signature.id"))

    signatures = relationship("Signature", lazy=False)

class Signature(Base):
    __tablename__ = "signature"

    id = Column(Integer, primary_key=True)

    sig_count = column_property(
        select([func.count("*")])
        .where(SnortEvent.signature == id)
        .correlate_except(SnortEvent)
    )
```

另请参阅

`Select.correlate_except()`

### PostgreSQL HSTORE 类型

PostgreSQL 的`HSTORE`类型的支持现在可用作为`HSTORE`。此类型充分利用了新的运算符系统，为 HSTORE 类型提供了一整套运算符，包括索引访问、连接和包含方法，如`comparator_factory.has_key()`、`comparator_factory.has_any()`和`comparator_factory.matrix()`：

```py
from sqlalchemy.dialects.postgresql import HSTORE

data = Table(
    "data_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("hstore_data", HSTORE),
)

engine.execute(select([data.c.hstore_data["some_key"]])).scalar()

engine.execute(select([data.c.hstore_data.matrix()])).scalar()
```

另请参阅

`HSTORE`

`hstore`

[#2606](https://www.sqlalchemy.org/trac/ticket/2606)

### 增强的 PostgreSQL ARRAY 类型

`ARRAY` 类型将接受一个可选的“维度”参数，将其固定到一个固定数量的维度，并在检索结果时大大提高效率：

```py
# old way, still works since PG supports N-dimensions per row:
Column("my_array", postgresql.ARRAY(Integer))

# new way, will render ARRAY with correct number of [] in DDL,
# will process binds and results more efficiently as we don't need
# to guess how many levels deep to go
Column("my_array", postgresql.ARRAY(Integer, dimensions=2))
```

该类型还引入了新的运算符，使用新的类型特定运算符框架。新操作包括索引访问：

```py
result = conn.execute(select([mytable.c.arraycol[2]]))
```

切片访问在 SELECT 中：

```py
result = conn.execute(select([mytable.c.arraycol[2:4]]))
```

切片更新在 UPDATE 中：

```py
conn.execute(mytable.update().values({mytable.c.arraycol[2:3]: [7, 8]}))
```

独立的数组文字：

```py
>>> from sqlalchemy.dialects import postgresql
>>> conn.scalar(select([postgresql.array([1, 2]) + postgresql.array([3, 4, 5])]))
[1, 2, 3, 4, 5]
```

数组连接，在下面，右侧的`[4, 5, 6]` 被强制转换为数组文字：

```py
select([mytable.c.arraycol + [4, 5, 6]])
```

另请参阅

`ARRAY`

`array`

[#2441](https://www.sqlalchemy.org/trac/ticket/2441)

### 新的、可配置的 DATE、TIME 类型用于 SQLite

SQLite 没有内置的 DATE、TIME 或 DATETIME 类型，而是提供了一些支持将日期和时间值存储为字符串或整数的方法。SQLite 的日期和时间类型在 0.8 中得到了增强，可以更加灵活地配置特定格式，包括“微秒”部分是可选的，以及几乎所有其他内容。

```py
Column("sometimestamp", sqlite.DATETIME(truncate_microseconds=True))
Column(
    "sometimestamp",
    sqlite.DATETIME(
        storage_format=(
            "%(year)04d%(month)02d%(day)02d"
            "%(hour)02d%(minute)02d%(second)02d%(microsecond)06d"
        ),
        regexp="(\d{4})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})(\d{6})",
    ),
)
Column(
    "somedate",
    sqlite.DATE(
        storage_format="%(month)02d/%(day)02d/%(year)04d",
        regexp="(?P<month>\d+)/(?P<day>\d+)/(?P<year>\d+)",
    ),
)
```

非常感谢 Nate Dub 在 Pycon 2012 上的努力。

另请参阅

`DATETIME`

`DATE`

`TIME`

[#2363](https://www.sqlalchemy.org/trac/ticket/2363)

### “COLLATE”在所有方言中都受支持；特别是 MySQL、PostgreSQL、SQLite

“collate”关键字，长期被 MySQL 方言接受，现在已经在所有`String`类型上建立，并且将在任何后端渲染，包括在使用`MetaData.create_all()`和`cast()`等功能时：

```py
>>> stmt = select([cast(sometable.c.somechar, String(20, collation="utf8"))])
>>> print(stmt)
SELECT  CAST(sometable.somechar  AS  VARCHAR(20)  COLLATE  "utf8")  AS  anon_1
FROM  sometable 
```

另请参阅

`String`

[#2276](https://www.sqlalchemy.org/trac/ticket/2276)

### 现在支持“前缀”用于`update()`、`delete()`

面向 MySQL，一个“前缀”可以在任何这些结构中渲染。例如：

```py
stmt = table.delete().prefix_with("LOW_PRIORITY", dialect="mysql")

stmt = table.update().prefix_with("LOW_PRIORITY", dialect="mysql")
```

该方法是新增的，除了已经存在于`insert()`、`select()`和`Query`上的方法之外。

另请参阅

`Update.prefix_with()`

`Delete.prefix_with()`

`Insert.prefix_with()`

`Select.prefix_with()`

`Query.prefix_with()`

[#2431](https://www.sqlalchemy.org/trac/ticket/2431)

## 行为变更

### 将“待定”对象视为“孤立”已经更加积极

这是对 0.8 系列的一个晚期补充，但希望新行为在更广泛的情况下更一致和直观。ORM 自至少版本 0.4 以来就包含了这样的行为，即一个“挂起”的对象，意味着它与`Session`相关联，但尚未插入数据库，当它变成“孤儿”时，即已与引用它的父对象解除关联，并且在配置的`relationship()`上指定了`delete-orphan`级联时，将自动从`Session`中清除。这种行为旨在大致模拟持久对象（即已插入）的行为，ORM 将根据分离事件的拦截发出 DELETE 来删除成为孤儿的对象。

行为变更适用于被多种父对象引用并且每个父对象都指定了`delete-orphan`的对象；典型示例是在多对多模式中桥接两种其他对象的关联对象。以前，行为是这样的，即挂起对象仅在与*所有*父对象解除关联时才会被清除。随着行为的变更，只要挂起对象与先前相关联的*任何*父对象解除关联，它就会被清除。这种行为旨在更接近持久对象的行为，即只要它们与任何父对象解除关联，它们就会被删除。

较旧行为的基本原因可以追溯至至少版本 0.4，基本上是一种防御性决定，试图在对象仍在构建 INSERT 时减轻混淆。但事实是，无论如何，一旦对象附加到任何新父对象，它就会重新与`Session`关联。

仍然可以刷新一个对象，即使它没有与所有必需的父对象关联，如果该对象一开始就没有与这些父对象关联，或者如果它被清除，但随后通过后续的附加事件重新与`Session`关联，但仍未完全关联。在这种情况下，预计数据库会发出完整性错误，因为可能存在未填充的 NOT NULL 外键列。ORM 决定让这些 INSERT 尝试发生，基于这样的判断：一个只与其必需的父对象部分关联但已经积极地与其中一些关联的对象，更多的情况下是用户错误，而不是应该被默默跳过的有意遗漏 - 在这里默默跳过 INSERT 会使这种用户错误非常难以调试。

对于可能依赖于旧行为的应用程序，可以通过将标志`legacy_is_orphan`作为映射器选项指定来重新启用旧行为。

新行为允许以下测试用例正常工作：

```py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship, backref
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True)
    name = Column(String(64))

class UserKeyword(Base):
    __tablename__ = "user_keyword"
    user_id = Column(Integer, ForeignKey("user.id"), primary_key=True)
    keyword_id = Column(Integer, ForeignKey("keyword.id"), primary_key=True)

    user = relationship(
        User, backref=backref("user_keywords", cascade="all, delete-orphan")
    )

    keyword = relationship(
        "Keyword", backref=backref("user_keywords", cascade="all, delete-orphan")
    )

    # uncomment this to enable the old behavior
    # __mapper_args__ = {"legacy_is_orphan": True}

class Keyword(Base):
    __tablename__ = "keyword"
    id = Column(Integer, primary_key=True)
    keyword = Column("keyword", String(64))

from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# note we're using PostgreSQL to ensure that referential integrity
# is enforced, for demonstration purposes.
e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)

Base.metadata.drop_all(e)
Base.metadata.create_all(e)

session = Session(e)

u1 = User(name="u1")
k1 = Keyword(keyword="k1")

session.add_all([u1, k1])

uk1 = UserKeyword(keyword=k1, user=u1)

# previously, if session.flush() were called here,
# this operation would succeed, but if session.flush()
# were not called here, the operation fails with an
# integrity error.
# session.flush()
del u1.user_keywords[0]

session.commit()
```

[#2655](https://www.sqlalchemy.org/trac/ticket/2655)

### `after_attach` 事件在项目与会话关联之后触发，而不是之前；`before_attach` 添加

使用`after_attach`的事件处理程序现在可以假定给定实例与给定会话关联：

```py
@event.listens_for(Session, "after_attach")
def after_attach(session, instance):
    assert instance in session
```

有些用例要求以这种方式工作。然而，其他用例要求项目尚未成为会话的一部分，比如当一个查询，旨在加载实例所需的某些状态，首先发出自动刷新，否则会过早刷新目标对象。这些用例应该使用新的“`before_attach`”事件：

```py
@event.listens_for(Session, "before_attach")
def before_attach(session, instance):
    instance.some_necessary_attribute = (
        session.query(Widget).filter_by(instance.widget_name).first()
    )
```

[#2464](https://www.sqlalchemy.org/trac/ticket/2464)

### 查询现在像`select()`一样自动关联

以前需要调用`Query.correlate()`才能使列或 WHERE 子查询与父级关联：

```py
subq = (
    session.query(Entity.value)
    .filter(Entity.id == Parent.entity_id)
    .correlate(Parent)
    .as_scalar()
)
session.query(Parent).filter(subq == "some value")
```

这与普通的`select()`构造相反，后者默认情况下会假定自动关联。在 0.8 中，上述语句将自动关联：

```py
subq = session.query(Entity.value).filter(Entity.id == Parent.entity_id).as_scalar()
session.query(Parent).filter(subq == "some value")
```

就像在`select()`中一样，可以通过调用`query.correlate(None)`来禁用关联，或者通过传递一个实体来手动设置关联，`query.correlate(someentity)`。

[#2179](https://www.sqlalchemy.org/trac/ticket/2179)

### 关联现在始终是上下文特定的

为了允许更广泛的相关性场景，`Select.correlate()` 和 `Query.correlate()` 的行为略有改变，以便 SELECT 语句仅在实际上下文中使用时才从 FROM 子句中省略“相关”的目标。此外，不再可能让作为外部 SELECT 语句中的 FROM 的 SELECT 语句“相关”（即省略）FROM 子句。

这个改变只会在渲染 SQL 方面变得更好，因为不再可能渲染出不合法的 SQL，其中所选内容相对于所选的 FROM 对象不足：

```py
from sqlalchemy.sql import table, column, select

t1 = table("t1", column("x"))
t2 = table("t2", column("y"))
s = select([t1, t2]).correlate(t1)

print(s)
```

在这个改变之前，上述内容将返回：

```py
SELECT  t1.x,  t2.y  FROM  t2
```

这是无效的 SQL，因为“t1”在任何 FROM 子句中都没有被引用。

现在，在没有外部 SELECT 的情况下，它将返回：

```py
SELECT  t1.x,  t2.y  FROM  t1,  t2
```

在 SELECT 中，相关性会如预期地生效：

```py
s2 = select([t1, t2]).where(t1.c.x == t2.c.y).where(t1.c.x == s)
print(s2)
```

```py
SELECT  t1.x,  t2.y  FROM  t1,  t2
WHERE  t1.x  =  t2.y  AND  t1.x  =
  (SELECT  t1.x,  t2.y  FROM  t2)
```

这个改变不会影响任何现有应用程序，因为对于正确构建的表达式，相关性行为保持不变。只有依赖于在非相关上下文中使用相关 SELECT 的无效字符串输出的应用程序（很可能是在测试场景中），才会看到任何变化。

[#2668](https://www.sqlalchemy.org/trac/ticket/2668)  ### create_all() 和 drop_all() 现在将空列表视为如此

方法 `MetaData.create_all()` 和 `MetaData.drop_all()` 现在将接受一个空的 `Table` 对象列表，并且不会发出任何 CREATE 或 DROP 语句。以前，空列表被解释为与传递 `None` 相同，对所有项目都会无条件发出 CREATE/DROP。

这是一个错误修复，但一些应用程序可能一直依赖于先前的行为。

[#2664](https://www.sqlalchemy.org/trac/ticket/2664)

### 修复了 `InstrumentationEvents` 的事件目标定位

`InstrumentationEvents`系列事件目标已经记录，事件将仅根据传递的实际类别触发。直到 0.7 版本，这并不是这种情况，应用于`InstrumentationEvents`的任何事件监听器都将为所有映射的类调用。在 0.8 中，添加了额外的逻辑，使事件仅对发送的那些类调用。这里的`propagate`标志默认设置为`True`，因为类仪器事件通常用于拦截尚未创建的类。

[#2590](https://www.sqlalchemy.org/trac/ticket/2590)

### 不再将“=”自动转换为 IN，当与 MS-SQL 中的子查询进行比较时

我们在 MSSQL 方言中发现了一个非常古老的行为，当用户尝试执行类似以下操作时，它会试图拯救用户：

```py
scalar_subq = select([someothertable.c.id]).where(someothertable.c.data == "foo")
select([sometable]).where(sometable.c.id == scalar_subq)
```

SQL Server 不允许将相等比较与标量 SELECT 进行比较，即，“x = (SELECT something)”。 MSSQL 方言会将其转换为 IN。然而，当进行类似“(SELECT something) = x”的比较时，也会发生同样的情况，总体上，这种猜测的水平超出了 SQLAlchemy 通常的范围，因此这种行为被移除。

[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

### 修复了`Session.is_modified()`的行为

`Session.is_modified()`方法接受一个参数`passive`，基本上不应该是必要的，所有情况下该参数的值应为`True` - 当保持默认值`False`时，它会导致命中数据库，并经常触发自动刷新，这将改变结果。在 0.8 中，`passive`参数将不起作用，并且未加载的属性永远不会被检查历史记录，因为根据定义，未加载的属性上不会有待处理的状态更改。

另请参阅

`Session.is_modified()`

[#2320](https://www.sqlalchemy.org/trac/ticket/2320)

### `Column.key`在`Select.c`属性中受到`Select.apply_labels()`的尊重

表达式系统的用户知道`Select.apply_labels()`会在每个列名前面添加表名，影响从`Select.c`中可用的名称：

```py
s = select([table1]).apply_labels()
s.c.table1_col1
s.c.table1_col2
```

在 0.8 版本之前，如果`Column`的`Column.key`不同，这个键会被忽略，与未使用`Select.apply_labels()`时不一致：

```py
# before 0.8
table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
s = select([table1])
s.c.column_one  # would be accessible like this
s.c.col1  # would raise AttributeError

s = select([table1]).apply_labels()
s.c.table1_column_one  # would raise AttributeError
s.c.table1_col1  # would be accessible like this
```

在 0.8 版本中，`Column.key`在两种情况下都受到尊重：

```py
# with 0.8
table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
s = select([table1])
s.c.column_one  # works
s.c.col1  # AttributeError

s = select([table1]).apply_labels()
s.c.table1_column_one  # works
s.c.table1_col1  # AttributeError
```

关于“name”和“key”的所有其他行为都是相同的，包括渲染的 SQL 仍然使用形式`<tablename>_<colname>` - 这里的重点是防止`Column.key`内容被渲染到`SELECT`语句中，以便在`Column.key`中使用特殊/非 ASCII 字符时不会出现问题。

[#2397](https://www.sqlalchemy.org/trac/ticket/2397)

### `single_parent`警告现在变成了错误

一个`relationship()`，它是多对一或多对多关系，并指定“cascade='all, delete-orphan'”，这是一个尴尬但仍然支持的用例（带有限制），如果关系没有指定`single_parent=True`选项，现在将引发错误。以前只会发出警告，但在任何情况下几乎立即会在属性系统中跟随失败。

[#2405](https://www.sqlalchemy.org/trac/ticket/2405)

### 添加`inspector`参数到`column_reflect`事件

0.7 版本添加了一个名为`column_reflect`的新事件，提供了每个列反射时可以增强的机会。我们在这个事件上稍微出了点错，因为事件没有提供获取当前用于反射的`Inspector`和`Connection`的方法，以防需要来自数据库的额外信息。由于这是一个尚未广泛使用的新事件，我们将直接在其中添加`inspector`参数：

```py
@event.listens_for(Table, "column_reflect")
def listen_for_col(inspector, table, column_info): ...
```

[#2418](https://www.sqlalchemy.org/trac/ticket/2418)

### 禁用 MySQL 的自动检测排序规则和大小写敏感性

MySQL 方言进行两次调用，其中一次非常昂贵，从数据库加载所有可能的排序规则以及大小写信息，第一次`Engine`连接时。这两个集合都不会用于任何 SQLAlchemy 函数，因此这些调用将不再自动发出。可能依赖于这些集合存在于`engine.dialect`上的应用程序将需要直接调用`_detect_collations()`和`_detect_casing()`。

[#2404](https://www.sqlalchemy.org/trac/ticket/2404)

### “未使用的列名”警告变成异常

在`insert()`或`update()`构造中引用不存在的列将引发错误而不是警告：

```py
t1 = table("t1", column("x"))
t1.insert().values(x=5, z=5)  # raises "Unconsumed column names: z"
```

[#2415](https://www.sqlalchemy.org/trac/ticket/2415)

### Inspector.get_primary_keys()已被弃用，请使用 Inspector.get_pk_constraint

`Inspector`上的这两种方法是多余的，其中`get_primary_keys()`将返回与`get_pk_constraint()`相同的信息，减去约束的名称：

```py
>>> insp.get_primary_keys()
["a", "b"]

>>> insp.get_pk_constraint()
{"name":"pk_constraint", "constrained_columns":["a", "b"]}
```

[#2422](https://www.sqlalchemy.org/trac/ticket/2422)

### 在大多数情况下，不区分大小写的结果行名称将被禁用

一个非常古老的行为，在`RowProxy`中的列名始终是不区分大小写比较的：

```py
>>> row = result.fetchone()
>>> row["foo"] == row["FOO"] == row["Foo"]
True
```

这是为了一些早期需要这样做的方言的好处，比如 Oracle 和 Firebird，但在现代用法中，我们有更准确的方法来处理这两个平台的不区分大小写行为。

未来，这种行为将仅可选地通过将标志``case_sensitive=False``传递给``create_engine()``来使用，但否则从行中请求的列名必须匹配大小写。

[#2423](https://www.sqlalchemy.org/trac/ticket/2423)

### `InstrumentationManager`和替代类仪器现在是一个扩展

`sqlalchemy.orm.interfaces.InstrumentationManager`类已移动到`sqlalchemy.ext.instrumentation.InstrumentationManager`。 “替代仪器”系统是为了极少数需要使用现有或不寻常的类仪器系统的安装而构建的，并且通常很少使用。这个系统的复杂性已经导出到一个`ext.`模块中。它保持未使用，直到被导入一次，通常是当第三方库导入`InstrumentationManager`时，此时它通过用`ExtendedInstrumentationRegistry`替换默认的`InstrumentationFactory`注入回`sqlalchemy.orm`。

## 已移除

### SQLSoup

SQLSoup 是一个方便的包，它在 SQLAlchemy ORM 的基础上提供了一个替代接口。SQLSoup 现在已经移动到自己的项目中，并且有单独的文档/发布；请参见[`bitbucket.org/zzzeek/sqlsoup`](https://bitbucket.org/zzzeek/sqlsoup)。

SQLSoup 是一个非常简单的工具，也可以受益于对其使用方式感兴趣的贡献者。

[#2262](https://www.sqlalchemy.org/trac/ticket/2262)

### MutableType

SQLAlchemy ORM 中的旧“可变”系统已被移除。这指的是应用于诸如`PickleType`的类型和有条件地应用于`TypeDecorator`的`MutableType`接口，并且自早期的 SQLAlchemy 版本以来一直提供了一种让 ORM 检测所谓的“可变”数据结构（如 JSON 结构和 pickled 对象）变化的方式。然而，实现从未合理，并迫使在单位操作期间发生昂贵的对象扫描的 ORM 使用方式。在 0.7 中，引入了[sqlalchemy.ext.mutable](https://docs.sqlalchemy.org/en/latest/orm/extensions/mutable.html)扩展，以便用户定义的数据类型可以在发生更改时适当地向单位操作发送事件。

如今，`MutableType` 的使用预计会很少，因为多年来一直有关于其效率低下的警告。

[#2442](https://www.sqlalchemy.org/trac/ticket/2442)

### sqlalchemy.exceptions（多年来一直是 sqlalchemy.exc）

我们曾留下了一个别名 `sqlalchemy.exceptions`，以使一些尚未升级以使用 `sqlalchemy.exc` 的非常老的库稍微容易一些。然而，一些用户仍然感到困惑，因此在 0.8 版本中我们将其完全删除，以消除任何困惑。

[#2433](https://www.sqlalchemy.org/trac/ticket/2433)

## 介绍

本指南介绍了 SQLAlchemy 0.8 版本的新功能，还记录了影响用户将其应用程序从 SQLAlchemy 0.7 系列迁移到 0.8 版本的更改。

SQLAlchemy 的发布版本即将接近 1.0，自 0.5 版本以来，每个新版本都减少了主要的使用变化。大多数已经适应现代 0.7 模式的应用程序应该可以无需更改地迁移到 0.8 版本。使用 0.6 甚至 0.5 模式的应用程序也应该可以直接迁移到 0.8 版本，尽管较大的应用程序可能需要测试每个中间版本。

## 平台支持

### 现在的目标是 Python 2.5 及以上版本

SQLAlchemy 0.8 将以 Python 2.5 为目标版本；不再兼容 Python 2.4。

内部将能够使用 Python 三元表达式（即，`x if y else z`），这将改善与使用 `y and x or z` 相比的情况，后者自然地导致了一些错误，以及上下文管理器（即，`with:`）和在某些情况下 `try:/except:/else:` 块，这将有助于提高代码的可读性。

SQLAlchemy 最终也会放弃对 2.5 版本的支持 - 当基线达到 2.6 时，SQLAlchemy 将转向使用 2.6/3.3 的就地兼容性，去除 `2to3` 工具的使用，并保持一个同时适用于 Python 2 和 3 的源代码库。

### 现在的目标是 Python 2.5 及以上版本

SQLAlchemy 0.8 将以 Python 2.5 为目标版本；不再兼容 Python 2.4。

内部将能够使用 Python 三元表达式（即，`x if y else z`），这将改善与使用 `y and x or z` 相比的情况，后者自然地导致了一些错误，以及上下文管理器（即，`with:`）和在某些情况下 `try:/except:/else:` 块，这将有助于提高代码的可读性。

SQLAlchemy 最终也会放弃对 2.5 版本的支持 - 当基线达到 2.6 时，SQLAlchemy 将转向使用 2.6/3.3 的就地兼容性，去除 `2to3` 工具的使用，并保持一个同时适用于 Python 2 和 3 的源代码库。

## 新的 ORM 特性

### 重写的 `relationship()` 机制

0.8 版本中关于 `relationship()` 如何确定如何在两个实体之间连接的能力得到了大大改进和增强。新系统包括以下功能：

+   当构建针对具有多个外键路径指向目标的类的 `relationship()` 时，**不再需要** `primaryjoin` 参数。只需要使用 `foreign_keys` 参数来指定应包含的列即可：

    ```py
    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        child_id_one = Column(Integer, ForeignKey("child.id"))
        child_id_two = Column(Integer, ForeignKey("child.id"))

        child_one = relationship("Child", foreign_keys=child_id_one)
        child_two = relationship("Child", foreign_keys=child_id_two)

    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
    ```

+   对于自引用、复合外键的关系，在其中**一个列指向自身**的情况下，现在已经得到支持。典型案例如下：

    ```py
    class Folder(Base):
        __tablename__ = "folder"
        __table_args__ = (
            ForeignKeyConstraint(
                ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
            ),
        )

        account_id = Column(Integer, primary_key=True)
        folder_id = Column(Integer, primary_key=True)
        parent_id = Column(Integer)
        name = Column(String)

        parent_folder = relationship(
            "Folder", backref="child_folders", remote_side=[account_id, folder_id]
        )
    ```

    上面的示例中，`Folder` 引用了其父 `Folder`，从 `account_id` 到自身的连接，并从 `parent_id` 到 `folder_id`。当 SQLAlchemy 构造自动连接时，不再假设“远程”一侧的所有列都被别名化，并且“本地”一侧的所有列都没有被别名化 - `account_id` 列在**两侧**都存在。因此，内部关系机制被完全重写，以支持一种完全不同的系统，其中生成了两个 `account_id` 的副本，每个副本包含不同的*注释*以确定它们在语句中的角色。注意基本急加载中的连接条件：

    ```py
    SELECT
      folder.account_id  AS  folder_account_id,
      folder.folder_id  AS  folder_folder_id,
      folder.parent_id  AS  folder_parent_id,
      folder.name  AS  folder_name,
      folder_1.account_id  AS  folder_1_account_id,
      folder_1.folder_id  AS  folder_1_folder_id,
      folder_1.parent_id  AS  folder_1_parent_id,
      folder_1.name  AS  folder_1_name
    FROM  folder
      LEFT  OUTER  JOIN  folder  AS  folder_1
      ON
      folder_1.account_id  =  folder.account_id
      AND  folder.folder_id  =  folder_1.parent_id

    WHERE  folder.folder_id  =  ?  AND  folder.account_id  =  ?
    ```

+   以前难以处理的自定义连接条件，比如涉及函数和/或类型转换的条件，现在在大多数情况下将按预期运行：

    ```py
    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() using explicit foreign_keys, remote_side
        parent_host = relationship(
            "HostEntry",
            primaryjoin=ip_address == cast(content, INET),
            foreign_keys=content,
            remote_side=ip_address,
        )
    ```

    新的 `relationship()` 机制利用了 SQLAlchemy 中称为 annotations 的概念。这些注释也可以通过 `foreign()` 和 `remote()` 函数显式地提供给应用程序代码，作为改善高级配置的可读性的手段，或者直接注入精确配置，绕过通常的连接检查启发式方法：

    ```py
    from sqlalchemy.orm import foreign, remote

    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() using explicit foreign() and remote() annotations
        # in lieu of separate arguments
        parent_host = relationship(
            "HostEntry",
            primaryjoin=remote(ip_address) == cast(foreign(content), INET),
        )
    ```

另请参阅

配置关系连接方式 - 对于 `relationship()` 的最新技术进行了全面修订，详细说明了自定义相关属性和集合访问的最新技术。

[#1401](https://www.sqlalchemy.org/trac/ticket/1401) [#610](https://www.sqlalchemy.org/trac/ticket/610)  ### 新的类/对象检查系统

许多 SQLAlchemy 用户正在编写需要检查映射类的属性的系统，包括能够访问主键列、对象关系、普通属性等，通常是为了构建数据编组系统，如 JSON/XML 转换方案和各种表单库。

最初，`Table` 和 `Column` 模型是最初的检查点，拥有一个完全文档化的系统。虽然 SQLAlchemy ORM 模型也是完全可内省的，但这从未是一个完全稳定和受支持的特性，用户往往不清楚如何获取这些信息。

0.8 现在为此提供了一致、稳定且完全文档化的 API，包括适用于映射类、实例、属性和其他核心和 ORM 结构的检查系统。此系统的入口是核心级别的 `inspect()` 函数。在大多数情况下，被检查的对象已经是 SQLAlchemy 系统的一部分，比如 `Mapper`、`InstanceState`、`Inspector` 等。在某些情况下，已经添加了新对象，用于在某些情境中提供检查 API，比如 `AliasedInsp` 和 `AttributeState`。

以下是一些关键功能的介绍：

```py
>>> class User(Base):
...     __tablename__ = "user"
...     id = Column(Integer, primary_key=True)
...     name = Column(String)
...     name_syn = synonym(name)
...     addresses = relationship("Address")

>>> # universal entry point is inspect()
>>> b = inspect(User)

>>> # b in this case is the Mapper
>>> b
<Mapper at 0x101521950; User>

>>> # Column namespace
>>> b.columns.id
Column('id', Integer(), table=<user>, primary_key=True, nullable=False)

>>> # mapper's perspective of the primary key
>>> b.primary_key
(Column('id', Integer(), table=<user>, primary_key=True, nullable=False),)

>>> # MapperProperties available from .attrs
>>> b.attrs.keys()
['name_syn', 'addresses', 'id', 'name']

>>> # .column_attrs, .relationships, etc. filter this collection
>>> b.column_attrs.keys()
['id', 'name']

>>> list(b.relationships)
[<sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>]

>>> # they are also namespaces
>>> b.column_attrs.id
<sqlalchemy.orm.properties.ColumnProperty object at 0x101525090>

>>> b.relationships.addresses
<sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>

>>> # point inspect() at a mapped, class level attribute,
>>> # returns the attribute itself
>>> b = inspect(User.addresses)
>>> b
<sqlalchemy.orm.attributes.InstrumentedAttribute object at 0x101521fd0>

>>> # From here we can get the mapper:
>>> b.mapper
<Mapper at 0x101525810; Address>

>>> # the parent inspector, in this case a mapper
>>> b.parent
<Mapper at 0x101521950; User>

>>> # an expression
>>> print(b.expression)
"user".id  =  address.user_id
>>> # inspect works on instances
>>> u1 = User(id=3, name="x")
>>> b = inspect(u1)

>>> # it returns the InstanceState
>>> b
<sqlalchemy.orm.state.InstanceState object at 0x10152bed0>

>>> # similar attrs accessor refers to the
>>> b.attrs.keys()
['id', 'name_syn', 'addresses', 'name']

>>> # attribute interface - from attrs, you get a state object
>>> b.attrs.id
<sqlalchemy.orm.state.AttributeState object at 0x10152bf90>

>>> # this object can give you, current value...
>>> b.attrs.id.value
3

>>> # ... current history
>>> b.attrs.id.history
History(added=[3], unchanged=(), deleted=())

>>> # InstanceState can also provide session state information
>>> # lets assume the object is persistent
>>> s = Session()
>>> s.add(u1)
>>> s.commit()

>>> # now we can get primary key identity, always
>>> # works in query.get()
>>> b.identity
(3,)

>>> # the mapper level key
>>> b.identity_key
(<class '__main__.User'>, (3,))

>>> # state within the session
>>> b.persistent, b.transient, b.deleted, b.detached
(True, False, False, False)

>>> # owning session
>>> b.session
<sqlalchemy.orm.session.Session object at 0x101701150>
```

另请参阅

运行时检查 API

[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

### 新的 with_polymorphic() 功能，可在任何地方使用

`Query.with_polymorphic()` 方法允许用户指定在针对连接表实体进行查询时应该存在哪些表。不幸的是，该方法很笨拙，仅适用于列表中的第一个实体，并且在使用和内部方面都有令人困扰的行为。已添加了一个名为 `with_polymorphic()` 的新增强功能，可以将任何实体“别名化”为其自身的“多态”版本，可在任何地方自由使用：

```py
from sqlalchemy.orm import with_polymorphic

palias = with_polymorphic(Person, [Engineer, Manager])
session.query(Company).join(palias, Company.employees).filter(
    or_(Engineer.language == "java", Manager.hair == "pointy")
)
```

另请参阅

使用 with_polymorphic() - 用于多态加载控制的新更新文档。

[#2333](https://www.sqlalchemy.org/trac/ticket/2333)

### of_type() 与 alias()、with_polymorphic()、any()、has()、joinedload()、subqueryload()、contains_eager() 一起使用

`PropComparator.of_type()`方法用于在构建 SQL 表达式时指定要使用的特定子类型，该子类型作为`relationship()`的目标具有多态映射。现在可以通过与新的`with_polymorphic()`函数结合使用该方法来定位*任意数量*的目标子类型：

```py
# use eager loading in conjunction with with_polymorphic targets
Job_P = with_polymorphic(Job, [SubJob, ExtraJob], aliased=True)
q = (
    s.query(DataContainer)
    .join(DataContainer.jobs.of_type(Job_P))
    .options(contains_eager(DataContainer.jobs.of_type(Job_P)))
)
```

该方法现在在大多数接受常规关系属性的地方同样有效，包括与加载器函数一起使用，如`joinedload()`、`subqueryload()`、`contains_eager()`以及比较方法，如`PropComparator.any()`和`PropComparator.has()`：

```py
# use eager loading in conjunction with with_polymorphic targets
Job_P = with_polymorphic(Job, [SubJob, ExtraJob], aliased=True)
q = (
    s.query(DataContainer)
    .join(DataContainer.jobs.of_type(Job_P))
    .options(contains_eager(DataContainer.jobs.of_type(Job_P)))
)

# pass subclasses to eager loads (implicitly applies with_polymorphic)
q = s.query(ParentThing).options(
    joinedload_all(ParentThing.container, DataContainer.jobs.of_type(SubJob))
)

# control self-referential aliasing with any()/has()
Job_A = aliased(Job)
q = (
    s.query(Job)
    .join(DataContainer.jobs)
    .filter(
        DataContainer.jobs.of_type(Job_A).any(
            and_(Job_A.id < Job.id, Job_A.type == "fred")
        )
    )
)
```

另请参阅

连接到特定子类型或 with_polymorphic()实体

[#2438](https://www.sqlalchemy.org/trac/ticket/2438) [#1106](https://www.sqlalchemy.org/trac/ticket/1106)

### 事件可以应用于未映射的超类

Mapper 和实例事件现在可以与未映射的超类关联，这些事件将随着子类被映射而传播。应该使用`propagate=True`标志。此功能允许将事件与声明基类关联：

```py
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

@event.listens_for("load", Base, propagate=True)
def on_load(target, context):
    print("New instance loaded:", target)

# on_load() will be applied to SomeClass
class SomeClass(Base):
    __tablename__ = "sometable"

    # ...
```

[#2585](https://www.sqlalchemy.org/trac/ticket/2585)

### Declarative 区分模块/包

Declarative 的一个关键特性是能够通过它们的字符串名称引用其他映射类。类名注册表现在对给定类的所属模块和包敏感。可以通过点名在表达式中引用这些类：

```py
class Snack(Base):
    # ...

    peanuts = relationship(
        "nuts.Peanut", primaryjoin="nuts.Peanut.snack_id == Snack.id"
    )
```

解析允许使用任何完整或部分消歧义的包名称。如果到特定类的路径仍然模糊不清，则会引发错误。

[#2338](https://www.sqlalchemy.org/trac/ticket/2338)

### Declarative 中的新 DeferredReflection 功能

“延迟反射”示例已移至 Declarative 中的一个支持功能。该功能允许仅使用占位符`Table`元数据构建声明性映射类，直到调用`prepare()`步骤，给定一个`Engine`以完全反映所有表并建立实际映射。该系统支持列的覆盖，单一和联合继承，以及每个引擎的不同基础。现在可以在一个步骤中在引擎创建时针对现有表创建完整的声明性配置：

```py
class ReflectedOne(DeferredReflection, Base):
    __abstract__ = True

class ReflectedTwo(DeferredReflection, Base):
    __abstract__ = True

class MyClass(ReflectedOne):
    __tablename__ = "mytable"

class MyOtherClass(ReflectedOne):
    __tablename__ = "myothertable"

class YetAnotherClass(ReflectedTwo):
    __tablename__ = "yetanothertable"

ReflectedOne.prepare(engine_one)
ReflectedTwo.prepare(engine_two)
```

另请参见

`DeferredReflection`

[#2485](https://www.sqlalchemy.org/trac/ticket/2485)

### ORM 类现在被核心构造所接受

虽然与`Query.filter()`一起使用的 SQL 表达式，例如`User.id == 5`，一直与核心构造（如`select()`）兼容，但传递给`select()`、`Select.select_from()`或`Select.correlate()`时，映射类本身将不被识别。一个新的 SQL 注册系统允许一个映射类作为核心中的 FROM 子句被接受：

```py
from sqlalchemy import select

stmt = select([User]).where(User.id == 5)
```

上面，映射的`User`类将扩展为`User`映射到的`Table`。

[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

### Query.update()支持 UPDATE..FROM

新的 UPDATE..FROM 机制在 query.update()中起作用。下面，我们对`SomeEntity`发出一个 UPDATE，添加一个 FROM 子句（或等效的，取决于后端）对`SomeOtherEntity`：

```py
query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
    SomeOtherEntity.foo == "bar"
).update({"data": "x"})
```

特别是，支持对联合继承实体的更新，前提是 UPDATE 的目标是本地表上的，或者如果父表和子表混合，则它们在查询中明确连接。下面，以`Engineer`作为`Person`的联合子类：

```py
query(Engineer).filter(Person.id == Engineer.id).filter(
    Person.name == "dilbert"
).update({"engineer_data": "java"})
```

会产生：

```py
UPDATE  engineer  SET  engineer_data='java'  FROM  person
WHERE  person.id=engineer.id  AND  person.name='dilbert'
```

[#2365](https://www.sqlalchemy.org/trac/ticket/2365)

### rollback()仅会回滚从 begin_nested()开始的“脏”对象

通过`Session.begin_nested()`使用 SAVEPOINT 的用户，应该改变行为以提高效率 - 在`rollback()`时，只有自上次刷新以来被标记为脏的对象将会过期，其余的`Session`保持不变。这是因为对 SAVEPOINT 的 ROLLBACK 不会终止包含事务的隔离，因此除了当前事务中未刷新的更改外，不需要过期。

[#2452](https://www.sqlalchemy.org/trac/ticket/2452)

### 缓存示例现在使用 dogpile.cache

缓存示例现在使用[dogpile.cache](https://dogpilecache.readthedocs.io/)。Dogpile.cache 是 Beaker 缓存部分的重写，具有更简单和更快的操作，以及对分布式锁定的支持。

请注意，Dogpile 示例以及之前的 Beaker 示例中使用的 SQLAlchemy API 略有变化，特别是在 Beaker 示例中所示的这种变化是必要的：

```py
--- examples/beaker_caching/caching_query.py
+++ examples/beaker_caching/caching_query.py
@@ -222,7 +222,8 @@

         """
         if query._current_path:
-            mapper, key = query._current_path[-2:]
+            mapper, prop = query._current_path[-2:]
+            key = prop.key

             for cls in mapper.class_.__mro__:
                 if (cls, key) in self._relationship_options:
```

另请参阅

Dogpile 缓存

[#2589](https://www.sqlalchemy.org/trac/ticket/2589)

### 重写的`relationship()`机制

0.8 版本在`relationship()`确定如何在两个实体之间连接方面具有更加改进和强大的系统。新系统包括以下功能：

+   当针对具有多个到目标的外键路径的类构建`relationship()`时，**不再需要**`primaryjoin`参数。只需要使用`foreign_keys`参数来指定应包含的列：

    ```py
    class Parent(Base):
        __tablename__ = "parent"
        id = Column(Integer, primary_key=True)
        child_id_one = Column(Integer, ForeignKey("child.id"))
        child_id_two = Column(Integer, ForeignKey("child.id"))

        child_one = relationship("Child", foreign_keys=child_id_one)
        child_two = relationship("Child", foreign_keys=child_id_two)

    class Child(Base):
        __tablename__ = "child"
        id = Column(Integer, primary_key=True)
    ```

+   支持自引用、复合外键的关系，其中**一列指向自身**。典型案例如下：

    ```py
    class Folder(Base):
        __tablename__ = "folder"
        __table_args__ = (
            ForeignKeyConstraint(
                ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
            ),
        )

        account_id = Column(Integer, primary_key=True)
        folder_id = Column(Integer, primary_key=True)
        parent_id = Column(Integer)
        name = Column(String)

        parent_folder = relationship(
            "Folder", backref="child_folders", remote_side=[account_id, folder_id]
        )
    ```

    在上面的示例中，`Folder`指向其父`Folder`，从`account_id`连接到自身，并且从`parent_id`连接到`folder_id`。当 SQLAlchemy 构建自动连接时，不能再假定“远程”一侧的所有列都被别名化，而“本地”一侧的所有列都没有被别名化 - `account_id`列**在两侧**都存在。因此，内部关系机制被完全重写以支持一个完全不同的系统，其中生成了两个`account_id`的副本，每个包含不同的*注释*以确定它们在语句中的角色。请注意基本贪婪加载中的连接条件：

    ```py
    SELECT
      folder.account_id  AS  folder_account_id,
      folder.folder_id  AS  folder_folder_id,
      folder.parent_id  AS  folder_parent_id,
      folder.name  AS  folder_name,
      folder_1.account_id  AS  folder_1_account_id,
      folder_1.folder_id  AS  folder_1_folder_id,
      folder_1.parent_id  AS  folder_1_parent_id,
      folder_1.name  AS  folder_1_name
    FROM  folder
      LEFT  OUTER  JOIN  folder  AS  folder_1
      ON
      folder_1.account_id  =  folder.account_id
      AND  folder.folder_id  =  folder_1.parent_id

    WHERE  folder.folder_id  =  ?  AND  folder.account_id  =  ?
    ```

+   以前难以处理的自定义连接条件，例如涉及函数和/或类型转换的情况，现在在大多数情况下将按预期运行：

    ```py
    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() using explicit foreign_keys, remote_side
        parent_host = relationship(
            "HostEntry",
            primaryjoin=ip_address == cast(content, INET),
            foreign_keys=content,
            remote_side=ip_address,
        )
    ```

    新的`relationship()`机制利用了 SQLAlchemy 中称为 annotations 的概念。这些注释也可以通过`foreign()`和`remote()`函数显式地提供给应用程序代码，作为改进高级配置的手段或直接注入精确配置的方式，绕过通常的联接检查启发式算法：

    ```py
    from sqlalchemy.orm import foreign, remote

    class HostEntry(Base):
        __tablename__ = "host_entry"

        id = Column(Integer, primary_key=True)
        ip_address = Column(INET)
        content = Column(String(50))

        # relationship() using explicit foreign() and remote() annotations
        # in lieu of separate arguments
        parent_host = relationship(
            "HostEntry",
            primaryjoin=remote(ip_address) == cast(foreign(content), INET),
        )
    ```

另请参阅

配置关系连接方式 - 一个新修订的关于`relationship()`的部分，详细介绍了定制相关属性和集合访问的最新技术。

[#1401](https://www.sqlalchemy.org/trac/ticket/1401) [#610](https://www.sqlalchemy.org/trac/ticket/610)

### 新的类/对象检查系统

许多 SQLAlchemy 用户正在编写需要检查映射类属性的系统，包括能够访问主键列、对象关系、普通属性等，通常用于构建数据编组系统，如 JSON/XML 转换方案和各种表单库。

最初，`Table`和`Column`模型是最初的检查点，具有良好记录的系统。虽然 SQLAlchemy ORM 模型也是完全可自省的，但这从未是一个完全稳定和受支持的功能，用户往往不清楚如何获取这些信息。

现在，0.8 版本为此提供了一致、稳定且完全文档化的 API，包括一个检查系统，可用于映射类、实例、属性和其他核心和 ORM 构造。 这个系统的入口是核心级别的`inspect()`函数。 在大多数情况下，被检查的对象已经是 SQLAlchemy 系统的一部分，比如`Mapper`、`InstanceState`、`Inspector`等。 在某些情况下，已添加了新对象，用于在某些上下文中提供检查 API，比如`AliasedInsp`和`AttributeState`。

以下是一些关键功能的演示：

```py
>>> class User(Base):
...     __tablename__ = "user"
...     id = Column(Integer, primary_key=True)
...     name = Column(String)
...     name_syn = synonym(name)
...     addresses = relationship("Address")

>>> # universal entry point is inspect()
>>> b = inspect(User)

>>> # b in this case is the Mapper
>>> b
<Mapper at 0x101521950; User>

>>> # Column namespace
>>> b.columns.id
Column('id', Integer(), table=<user>, primary_key=True, nullable=False)

>>> # mapper's perspective of the primary key
>>> b.primary_key
(Column('id', Integer(), table=<user>, primary_key=True, nullable=False),)

>>> # MapperProperties available from .attrs
>>> b.attrs.keys()
['name_syn', 'addresses', 'id', 'name']

>>> # .column_attrs, .relationships, etc. filter this collection
>>> b.column_attrs.keys()
['id', 'name']

>>> list(b.relationships)
[<sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>]

>>> # they are also namespaces
>>> b.column_attrs.id
<sqlalchemy.orm.properties.ColumnProperty object at 0x101525090>

>>> b.relationships.addresses
<sqlalchemy.orm.properties.RelationshipProperty object at 0x1015212d0>

>>> # point inspect() at a mapped, class level attribute,
>>> # returns the attribute itself
>>> b = inspect(User.addresses)
>>> b
<sqlalchemy.orm.attributes.InstrumentedAttribute object at 0x101521fd0>

>>> # From here we can get the mapper:
>>> b.mapper
<Mapper at 0x101525810; Address>

>>> # the parent inspector, in this case a mapper
>>> b.parent
<Mapper at 0x101521950; User>

>>> # an expression
>>> print(b.expression)
"user".id  =  address.user_id
>>> # inspect works on instances
>>> u1 = User(id=3, name="x")
>>> b = inspect(u1)

>>> # it returns the InstanceState
>>> b
<sqlalchemy.orm.state.InstanceState object at 0x10152bed0>

>>> # similar attrs accessor refers to the
>>> b.attrs.keys()
['id', 'name_syn', 'addresses', 'name']

>>> # attribute interface - from attrs, you get a state object
>>> b.attrs.id
<sqlalchemy.orm.state.AttributeState object at 0x10152bf90>

>>> # this object can give you, current value...
>>> b.attrs.id.value
3

>>> # ... current history
>>> b.attrs.id.history
History(added=[3], unchanged=(), deleted=())

>>> # InstanceState can also provide session state information
>>> # lets assume the object is persistent
>>> s = Session()
>>> s.add(u1)
>>> s.commit()

>>> # now we can get primary key identity, always
>>> # works in query.get()
>>> b.identity
(3,)

>>> # the mapper level key
>>> b.identity_key
(<class '__main__.User'>, (3,))

>>> # state within the session
>>> b.persistent, b.transient, b.deleted, b.detached
(True, False, False, False)

>>> # owning session
>>> b.session
<sqlalchemy.orm.session.Session object at 0x101701150>
```

另请参阅

运行时检查 API

[#2208](https://www.sqlalchemy.org/trac/ticket/2208)

### 新的 with_polymorphic()功能，可在任何地方使用

`Query.with_polymorphic()`方法允许用户指定在针对联接表实体进行查询时应该存在哪些表。 不幸的是，该方法很笨拙，只适用于列表中的第一个实体，否则在使用和内部方面都有一些尴尬的行为。 已添加了一个名为`with_polymorphic()`的新增强功能，它允许任何实体“别名”为其自身的“多态”版本，可自由在任何地方使用：

```py
from sqlalchemy.orm import with_polymorphic

palias = with_polymorphic(Person, [Engineer, Manager])
session.query(Company).join(palias, Company.employees).filter(
    or_(Engineer.language == "java", Manager.hair == "pointy")
)
```

另请参阅

使用 with_polymorphic() - 用于多态加载控制的新更新文档。

[#2333](https://www.sqlalchemy.org/trac/ticket/2333)

### of_type() 与 alias()、with_polymorphic()、any()、has()、joinedload()、subqueryload()、contains_eager()一起使用

`PropComparator.of_type()`方法用于在构建 SQL 表达式时指定要使用的特定子类型，该表达式沿着具有多态映射作为目标的`relationship()`。 现在，可以通过与新的`with_polymorphic()`函数结合使用，来指定*任意数量*的目标子类型：

```py
# use eager loading in conjunction with with_polymorphic targets
Job_P = with_polymorphic(Job, [SubJob, ExtraJob], aliased=True)
q = (
    s.query(DataContainer)
    .join(DataContainer.jobs.of_type(Job_P))
    .options(contains_eager(DataContainer.jobs.of_type(Job_P)))
)
```

此方法现在在大多数常规关系属性接受的地方同样有效，包括与加载器函数一起使用，如`joinedload()`、`subqueryload()`、`contains_eager()`，以及比较方法如`PropComparator.any()` 和 `PropComparator.has()`：

```py
# use eager loading in conjunction with with_polymorphic targets
Job_P = with_polymorphic(Job, [SubJob, ExtraJob], aliased=True)
q = (
    s.query(DataContainer)
    .join(DataContainer.jobs.of_type(Job_P))
    .options(contains_eager(DataContainer.jobs.of_type(Job_P)))
)

# pass subclasses to eager loads (implicitly applies with_polymorphic)
q = s.query(ParentThing).options(
    joinedload_all(ParentThing.container, DataContainer.jobs.of_type(SubJob))
)

# control self-referential aliasing with any()/has()
Job_A = aliased(Job)
q = (
    s.query(Job)
    .join(DataContainer.jobs)
    .filter(
        DataContainer.jobs.of_type(Job_A).any(
            and_(Job_A.id < Job.id, Job_A.type == "fred")
        )
    )
)
```

另请参阅

连接到特定子类型或 with_polymorphic() 实体

[#2438](https://www.sqlalchemy.org/trac/ticket/2438) [#1106](https://www.sqlalchemy.org/trac/ticket/1106)

### 事件可以应用于未映射的超类

现在可以将 Mapper 和实例事件与未映射的超类关联，这些事件将传播到子类中，当这些子类被映射时。应该使用`propagate=True`标志。此功能允许将事件与声明式基类关联起来：

```py
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

@event.listens_for("load", Base, propagate=True)
def on_load(target, context):
    print("New instance loaded:", target)

# on_load() will be applied to SomeClass
class SomeClass(Base):
    __tablename__ = "sometable"

    # ...
```

[#2585](https://www.sqlalchemy.org/trac/ticket/2585)

### 声明式区分模块/包

声明式的一个关键特性是能够使用其字符串名称引用其他映射类。现在，类名的注册表对给定类的拥有模块和包是敏感的。可以通过表达式中的点名引用这些类：

```py
class Snack(Base):
    # ...

    peanuts = relationship(
        "nuts.Peanut", primaryjoin="nuts.Peanut.snack_id == Snack.id"
    )
```

解析允许使用任何全名或部分消除歧义的包名称。如果对特定类的路径仍然不明确，将会引发错误。

[#2338](https://www.sqlalchemy.org/trac/ticket/2338)

### 声明式中的新延迟反射功能

“延迟反射”示例已移至声明式中的支持功能。此功能允许仅使用占位符`Table`元数据构建声明式映射类，直到调用`prepare()`步骤，并提供一个`Engine`以完全反射所有表并建立实际映射为止。该系统支持列的重写、单一和连接继承，以及每个引擎的不同基类。现在可以一次性在引擎创建时针对现有表创建完整的声明式配置：

```py
class ReflectedOne(DeferredReflection, Base):
    __abstract__ = True

class ReflectedTwo(DeferredReflection, Base):
    __abstract__ = True

class MyClass(ReflectedOne):
    __tablename__ = "mytable"

class MyOtherClass(ReflectedOne):
    __tablename__ = "myothertable"

class YetAnotherClass(ReflectedTwo):
    __tablename__ = "yetanothertable"

ReflectedOne.prepare(engine_one)
ReflectedTwo.prepare(engine_two)
```

另请参阅

`DeferredReflection`

[#2485](https://www.sqlalchemy.org/trac/ticket/2485)

### ORM 类现在被核心构造所接受

虽然与`Query.filter()`一起使用的 SQL 表达式，例如`User.id == 5`，一直与核心构造兼容，例如`select()`，但当传递给`select()`，`Select.select_from()`或`Select.correlate()`时，映射类本身将不被识别。一个新的 SQL 注册系统允许映射类作为核心中的 FROM 子句被接受：

```py
from sqlalchemy import select

stmt = select([User]).where(User.id == 5)
```

上面，映射的`User`类将扩展为`Table`，`User`映射到其中的表。

[#2245](https://www.sqlalchemy.org/trac/ticket/2245)

### Query.update()支持 UPDATE..FROM

新的 UPDATE..FROM 机制适用于 query.update()。下面，我们对`SomeEntity`发出一个带有 FROM 子句（或等效的，取决于后端）的 UPDATE：

```py
query(SomeEntity).filter(SomeEntity.id == SomeOtherEntity.id).filter(
    SomeOtherEntity.foo == "bar"
).update({"data": "x"})
```

特别是，支持对连接继承实体的更新，前提是 UPDATE 的目标是过滤表上的本地表，或者如果父表和子表混合，它们在查询中明确连接。下面，给定`Engineer`作为`Person`的连接子类： 

```py
query(Engineer).filter(Person.id == Engineer.id).filter(
    Person.name == "dilbert"
).update({"engineer_data": "java"})
```

会产生：

```py
UPDATE  engineer  SET  engineer_data='java'  FROM  person
WHERE  person.id=engineer.id  AND  person.name='dilbert'
```

[#2365](https://www.sqlalchemy.org/trac/ticket/2365)

### rollback()将仅回滚从 begin_nested()开始的“脏”对象

一项行为变更应该提高那些通过`Session.begin_nested()`使用 SAVEPOINT 的用户的效率 - 在`rollback()`时，只有自上次刷新以来被标记为脏的对象将被过期，其余的`Session`保持不变。这是因为对 SAVEPOINT 的 ROLLBACK 不会终止包含事务的隔离，因此除了当前事务中未刷新的更改外，不需要过期。

[#2452](https://www.sqlalchemy.org/trac/ticket/2452)

### 缓存示例现在使用 dogpile.cache

缓存示例现在使用[dogpile.cache](https://dogpilecache.readthedocs.io/)。Dogpile.cache 是 Beaker 缓存部分的重写，具有更简单和更快的操作，以及对分布式锁定的支持。

注意，Dogpile 示例以及之前的 Beaker 示例中使用的 SQLAlchemy API 略有变化，特别是正如 Beaker 示例中所示，这种变化是必要的：

```py
--- examples/beaker_caching/caching_query.py
+++ examples/beaker_caching/caching_query.py
@@ -222,7 +222,8 @@

         """
         if query._current_path:
-            mapper, key = query._current_path[-2:]
+            mapper, prop = query._current_path[-2:]
+            key = prop.key

             for cls in mapper.class_.__mro__:
                 if (cls, key) in self._relationship_options:
```

另请参见

Dogpile Caching

[#2589](https://www.sqlalchemy.org/trac/ticket/2589)

## 新的核心功能

### 完全可扩展，类型级别的核心操作符支持

到目前为止，核心从未有过任何系统来为列和其他表达式构造添加对新 SQL 运算符的支持，除了`ColumnOperators.op()`方法，这个方法“刚好”能让事情正常运行。此外，核心从未有过任何系统允许覆盖现有运算符的行为。直到现在，唯一灵活重新定义运算符的方式是在 ORM 层中，使用`column_property()`并提供一个`comparator_factory`参数。因此，像 GeoAlchemy 这样的第三方库被迫以 ORM 为中心，并依赖各种技巧来应用新操作以及使其正确传播。

核心中的新运算符系统添加了一直缺失的关键点，即将新的和覆盖的运算符与*类型*关联起来。毕竟，真正驱动操作类型的不是列、CAST 运算符或 SQL 函数，而是表达式的*类型*。实现细节很少 - 只需向核心`ColumnElement`类型添加几个额外方法，以便它向其`TypeEngine`对象查询可选的一组运算符。新的或修订的操作可以与任何类型关联，可以通过对现有类型进行子类化，使用`TypeDecorator`，或者通过将新的`Comparator`对象附加到现有类型类来“全面推广”。

例如，要为`Numeric`类型添加对数支持：

```py
from sqlalchemy.types import Numeric
from sqlalchemy.sql import func

class CustomNumeric(Numeric):
    class comparator_factory(Numeric.Comparator):
        def log(self, other):
            return func.log(self.expr, other)
```

这种新类型可以像任何其他类型一样使用：

```py
data = Table(
    "data",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x", CustomNumeric(10, 5)),
    Column("y", CustomNumeric(10, 5)),
)

stmt = select([data.c.x.log(data.c.y)]).where(data.c.x.log(2) < value)
print(conn.execute(stmt).fetchall())
```

由此带来的新功能包括立即支持 PostgreSQL 的 HSTORE 类型，以及与 PostgreSQL 的 ARRAY 类型相关的新操作。它还为现有类型开辟了更多特定于这些类型的运算符的道路，例如更多的字符串、整数和日期运算符。

另请参阅

重新定义和创建新运算符

`HSTORE`

[#2547](https://www.sqlalchemy.org/trac/ticket/2547)

### 插入的多值支持

`Insert.values()`方法现在支持字典列表，将生成多 VALUES 语句，如`VALUES (<row1>), (<row2>), ...`。这仅适用于支持此语法的后端，包括 PostgreSQL、SQLite 和 MySQL。这与通常的`executemany()`风格的 INSERT 不同：

```py
users.insert().values(
    [
        {"name": "some name"},
        {"name": "some other name"},
        {"name": "yet another name"},
    ]
)
```

另请参阅

`Insert.values()`

[#2623](https://www.sqlalchemy.org/trac/ticket/2623)

### 类型表达式

现在可以将 SQL 表达式与类型关联起来。从历史上看，`TypeEngine`一直允许 Python 端函数接收绑定参数和结果行值，通过 Python 端转换函数在到达/返回数据库时进行转换。新功能允许类似的功能，但在数据库端进行：

```py
from sqlalchemy.types import String
from sqlalchemy import func, Table, Column, MetaData

class LowerString(String):
    def bind_expression(self, bindvalue):
        return func.lower(bindvalue)

    def column_expression(self, col):
        return func.lower(col)

metadata = MetaData()
test_table = Table("test_table", metadata, Column("data", LowerString))
```

上面，`LowerString`类型定义了一个 SQL 表达式，每当`test_table.c.data`列在 SELECT 语句的列子句中呈现时，该表达式将被发出：

```py
>>> print(select([test_table]).where(test_table.c.data == "HI"))
SELECT  lower(test_table.data)  AS  data
FROM  test_table
WHERE  test_table.data  =  lower(:data_1) 
```

这个功能也被新版 GeoAlchemy 大量使用，以根据类型规则在 SQL 中内联嵌入 PostGIS 表达式。

另请参阅

应用 SQL 级别的绑定/结果处理

[#1534](https://www.sqlalchemy.org/trac/ticket/1534)

### 核心检查系统

`inspect()`函数引入了新的类/对象检查系统，也适用于核心。应用于`Engine`会产生一个`Inspector`对象：

```py
from sqlalchemy import inspect
from sqlalchemy import create_engine

engine = create_engine("postgresql://scott:tiger@localhost/test")
insp = inspect(engine)
print(insp.get_table_names())
```

它也可以应用于任何`ClauseElement`，它返回`ClauseElement`本身，比如`Table`、`Column`、`Select`等。这使得它可以在核心和 ORM 构造之间流畅地工作。

### 新方法`Select.correlate_except()`

`select()` 现在有一个方法`Select.correlate_except()`，指定“除了指定的所有 FROM 子句之外的所有 FROM 子句”。它可用于映射场景，其中相关子查询应该正常关联，除了针对特定目标可选择的情况：

```py
class SnortEvent(Base):
    __tablename__ = "event"

    id = Column(Integer, primary_key=True)
    signature = Column(Integer, ForeignKey("signature.id"))

    signatures = relationship("Signature", lazy=False)

class Signature(Base):
    __tablename__ = "signature"

    id = Column(Integer, primary_key=True)

    sig_count = column_property(
        select([func.count("*")])
        .where(SnortEvent.signature == id)
        .correlate_except(SnortEvent)
    )
```

另请参阅

`Select.correlate_except()`

### PostgreSQL HSTORE 类型

PostgreSQL 的`HSTORE`类型现在可以作为`HSTORE`使用。该类型充分利用了新的操作符系统，为 HSTORE 类型提供了一整套操作符，包括索引访问、连接和包含方法，如`comparator_factory.has_key()`、`comparator_factory.has_any()`和`comparator_factory.matrix()`：

```py
from sqlalchemy.dialects.postgresql import HSTORE

data = Table(
    "data_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("hstore_data", HSTORE),
)

engine.execute(select([data.c.hstore_data["some_key"]])).scalar()

engine.execute(select([data.c.hstore_data.matrix()])).scalar()
```

另请参阅

`HSTORE`

`hstore`

[#2606](https://www.sqlalchemy.org/trac/ticket/2606)

### 增强的 PostgreSQL ARRAY 类型

`ARRAY` 类型将接受一个可选的“维度”参数，将其固定到一个固定数量的维度，大大提高检索结果的效率：

```py
# old way, still works since PG supports N-dimensions per row:
Column("my_array", postgresql.ARRAY(Integer))

# new way, will render ARRAY with correct number of [] in DDL,
# will process binds and results more efficiently as we don't need
# to guess how many levels deep to go
Column("my_array", postgresql.ARRAY(Integer, dimensions=2))
```

该类型还引入了新的操作符，使用新的类型特定的操作符框架。新操作包括索引访问：

```py
result = conn.execute(select([mytable.c.arraycol[2]]))
```

在 SELECT 中的切片访问：

```py
result = conn.execute(select([mytable.c.arraycol[2:4]]))
```

在 UPDATE 中的切片更新：

```py
conn.execute(mytable.update().values({mytable.c.arraycol[2:3]: [7, 8]}))
```

独立的数组文字：

```py
>>> from sqlalchemy.dialects import postgresql
>>> conn.scalar(select([postgresql.array([1, 2]) + postgresql.array([3, 4, 5])]))
[1, 2, 3, 4, 5]
```

数组连接，在下面，右侧的`[4, 5, 6]`被强制转换为数组文字：

```py
select([mytable.c.arraycol + [4, 5, 6]])
```

另请参阅

`ARRAY`

`array`

[#2441](https://www.sqlalchemy.org/trac/ticket/2441)

### 新的可配置的 SQLite 日期、时间类型

SQLite 没有内置的 DATE、TIME 或 DATETIME 类型，而是提供了一些支持将日期和时间值存储为字符串或整数的方法。0.8 版本中增强了 SQLite 的日期和时间类型，使其更加可配置，包括“微秒”部分是可选的，以及几乎所有其他内容。

```py
Column("sometimestamp", sqlite.DATETIME(truncate_microseconds=True))
Column(
    "sometimestamp",
    sqlite.DATETIME(
        storage_format=(
            "%(year)04d%(month)02d%(day)02d"
            "%(hour)02d%(minute)02d%(second)02d%(microsecond)06d"
        ),
        regexp="(\d{4})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})(\d{6})",
    ),
)
Column(
    "somedate",
    sqlite.DATE(
        storage_format="%(month)02d/%(day)02d/%(year)04d",
        regexp="(?P<month>\d+)/(?P<day>\d+)/(?P<year>\d+)",
    ),
)
```

非常感谢 Nate Dub 在 Pycon 2012 上的努力。

另请参阅

`DATETIME`

`DATE`

`TIME`

[#2363](https://www.sqlalchemy.org/trac/ticket/2363)

### “COLLATE”在所有方言中都受支持；特别是 MySQL、PostgreSQL、SQLite

“collate”关键字，长期以来被 MySQL 方言接受，现在已经在所有 `String` 类型上建立，并且将在任何后端呈现，包括在使用 `MetaData.create_all()` 和 `cast()` 等特性时：

```py
>>> stmt = select([cast(sometable.c.somechar, String(20, collation="utf8"))])
>>> print(stmt)
SELECT  CAST(sometable.somechar  AS  VARCHAR(20)  COLLATE  "utf8")  AS  anon_1
FROM  sometable 
```

另见

`String`

[#2276](https://www.sqlalchemy.org/trac/ticket/2276)

### “Prefixes”现在支持于 `update()`, `delete()`

面向 MySQL，一个“前缀”可以在这些结构中的任何一个中呈现。例如：

```py
stmt = table.delete().prefix_with("LOW_PRIORITY", dialect="mysql")

stmt = table.update().prefix_with("LOW_PRIORITY", dialect="mysql")
```

该方法是新增的，除了已经存在于 `insert()`, `select()` 和 `Query` 上的方法。

另见

`Update.prefix_with()`

`Delete.prefix_with()`

`Insert.prefix_with()`

`Select.prefix_with()`

`Query.prefix_with()`

[#2431](https://www.sqlalchemy.org/trac/ticket/2431)

### 完全可扩展的核心级别操作符支持

迄今为止，核心从未有过为 Column 和其他表达式构造添加新 SQL 运算符的系统，除了 `ColumnOperators.op()` 方法，它“刚好足够”使事情正常工作。此外，核心中也从未建立过任何系统，允许覆盖现有运算符的行为。直到现在，灵活重新定义运算符的唯一方法是在 ORM 层中，使用 `column_property()` 给定一个 `comparator_factory` 参数。因此，像 GeoAlchemy 这样的第三方库被迫是 ORM 中心的，并且依赖于一系列的黑客来应用新的操作以及使其正确传播。

核心中的新运算符系统添加了一直缺失的一个钩子，即将新的和重写的运算符与*类型*关联起来。毕竟，真正驱动存在哪些操作的不是列、CAST 运算符或 SQL 函数，而是表达式的*类型*。实现细节很少——只需向核心 `ColumnElement` 类型添加几个额外的方法，以便它向其 `TypeEngine` 对象查询一组可选的运算符。新的或修改后的操作可以与任何类型关联，可以通过对现有类型的子类化、使用 `TypeDecorator` 或通过将新的 `Comparator` 对象附加到现有类型类来进行“全面的跨越边界”的关联。

例如，要向 `Numeric` 类型添加对数支持：

```py
from sqlalchemy.types import Numeric
from sqlalchemy.sql import func

class CustomNumeric(Numeric):
    class comparator_factory(Numeric.Comparator):
        def log(self, other):
            return func.log(self.expr, other)
```

新类型可像其他类型一样使用：

```py
data = Table(
    "data",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x", CustomNumeric(10, 5)),
    Column("y", CustomNumeric(10, 5)),
)

stmt = select([data.c.x.log(data.c.y)]).where(data.c.x.log(2) < value)
print(conn.execute(stmt).fetchall())
```

此举带来的新功能包括对 PostgreSQL 的 HSTORE 类型的支持，以及与 PostgreSQL 的 ARRAY 类型相关的新操作。它还为现有类型提供了更多专门针对这些类型的操作符的可能性，如更多字符串、整数和日期操作符。

另请参阅

重新定义和创建新运算符

`HSTORE`

[#2547](https://www.sqlalchemy.org/trac/ticket/2547)

### 插入的多值支持

`Insert.values()` 方法现在支持字典列表，这将呈现出多值语句，如 `VALUES (<row1>), (<row2>), ...`。这仅与支持此语法的后端相关，包括 PostgreSQL、SQLite 和 MySQL。这与通常的 `executemany()` 样式的 INSERT 不同：

```py
users.insert().values(
    [
        {"name": "some name"},
        {"name": "some other name"},
        {"name": "yet another name"},
    ]
)
```

另请参阅

`Insert.values()`

[#2623](https://www.sqlalchemy.org/trac/ticket/2623)

### 类型表达式

SQL 表达式现在可以与类型关联。在历史上，`TypeEngine` 一直允许 Python 端函数接收绑定参数和结果行值，并在传递到/从数据库的途中通过 Python 端转换函数进行转换。新功能允许类似的功能，但在数据库端执行：

```py
from sqlalchemy.types import String
from sqlalchemy import func, Table, Column, MetaData

class LowerString(String):
    def bind_expression(self, bindvalue):
        return func.lower(bindvalue)

    def column_expression(self, col):
        return func.lower(col)

metadata = MetaData()
test_table = Table("test_table", metadata, Column("data", LowerString))
```

上述中，`LowerString`类型定义了一个 SQL 表达式，每当`test_table.c.data`列在 SELECT 语句的列子句中被呈现时，该表达式就会被发出：

```py
>>> print(select([test_table]).where(test_table.c.data == "HI"))
SELECT  lower(test_table.data)  AS  data
FROM  test_table
WHERE  test_table.data  =  lower(:data_1) 
```

这个特性也被新版的 GeoAlchemy 大量使用，以根据类型规则在 SQL 中内联嵌入 PostGIS 表达式。

另请参阅

应用 SQL 级绑定/结果处理

[#1534](https://www.sqlalchemy.org/trac/ticket/1534)

### 核心检查系统

引入的`inspect()`函数新的类/对象检查系统也适用于核心。应用到一个`Engine`上会产生一个`Inspector`对象：

```py
from sqlalchemy import inspect
from sqlalchemy import create_engine

engine = create_engine("postgresql://scott:tiger@localhost/test")
insp = inspect(engine)
print(insp.get_table_names())
```

它也可以应用于任何返回自身的`ClauseElement`，例如`Table`、`Column`、`Select`等。这使它可以在核心和 ORM 构造之间流畅工作。

### 新方法`Select.correlate_except()`

`select()`现在有一个方法`Select.correlate_except()`，它指定“在除了指定的 FROM 子句之外的所有 FROM 子句上关联”。它可用于映射方案，其中相关子查询应该正常关联，除了针对特定目标可选择的情况：

```py
class SnortEvent(Base):
    __tablename__ = "event"

    id = Column(Integer, primary_key=True)
    signature = Column(Integer, ForeignKey("signature.id"))

    signatures = relationship("Signature", lazy=False)

class Signature(Base):
    __tablename__ = "signature"

    id = Column(Integer, primary_key=True)

    sig_count = column_property(
        select([func.count("*")])
        .where(SnortEvent.signature == id)
        .correlate_except(SnortEvent)
    )
```

另请参阅

`Select.correlate_except()`

### PostgreSQL HSTORE 类型

对 PostgreSQL 的`HSTORE`类型的支持现在可用作`HSTORE`。这种类型充分利用了新的运算符系统，为 HSTORE 类型提供了一整套运算符，包括索引访问、连接和包含方法，如`comparator_factory.has_key()`、`comparator_factory.has_any()`和`comparator_factory.matrix()`：

```py
from sqlalchemy.dialects.postgresql import HSTORE

data = Table(
    "data_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("hstore_data", HSTORE),
)

engine.execute(select([data.c.hstore_data["some_key"]])).scalar()

engine.execute(select([data.c.hstore_data.matrix()])).scalar()
```

另请参阅

`HSTORE`

`hstore`

[#2606](https://www.sqlalchemy.org/trac/ticket/2606)

### 增强的 PostgreSQL ARRAY 类型

`ARRAY` 类型将接受一个可选的“维度”参数，将其固定到一个固定数量的维度，大大提高检索结果的效率：

```py
# old way, still works since PG supports N-dimensions per row:
Column("my_array", postgresql.ARRAY(Integer))

# new way, will render ARRAY with correct number of [] in DDL,
# will process binds and results more efficiently as we don't need
# to guess how many levels deep to go
Column("my_array", postgresql.ARRAY(Integer, dimensions=2))
```

该类型还引入了新的操作符，使用新的类型特定的操作符框架。新操作包括索引访问：

```py
result = conn.execute(select([mytable.c.arraycol[2]]))
```

在 SELECT 中的切片访问：

```py
result = conn.execute(select([mytable.c.arraycol[2:4]]))
```

在 UPDATE 中的切片更新：

```py
conn.execute(mytable.update().values({mytable.c.arraycol[2:3]: [7, 8]}))
```

独立的数组文字：

```py
>>> from sqlalchemy.dialects import postgresql
>>> conn.scalar(select([postgresql.array([1, 2]) + postgresql.array([3, 4, 5])]))
[1, 2, 3, 4, 5]
```

数组连接，下面的右侧`[4, 5, 6]`被强制转换为数组文字：

```py
select([mytable.c.arraycol + [4, 5, 6]])
```

另请参见

`ARRAY`

`array`

[#2441](https://www.sqlalchemy.org/trac/ticket/2441)

### 新的、可配置的 SQLite 日期、时间类型

SQLite 没有内置的 DATE，TIME 或 DATETIME 类型，而是提供了一些支持，用于将日期和时间值存储为字符串或整数。SQLite 中的日期和时间类型在 0.8 中得到了增强，可以更具体地配置特定格式，包括“微秒”部分是可选的，以及几乎所有其他内容。

```py
Column("sometimestamp", sqlite.DATETIME(truncate_microseconds=True))
Column(
    "sometimestamp",
    sqlite.DATETIME(
        storage_format=(
            "%(year)04d%(month)02d%(day)02d"
            "%(hour)02d%(minute)02d%(second)02d%(microsecond)06d"
        ),
        regexp="(\d{4})(\d{2})(\d{2})(\d{2})(\d{2})(\d{2})(\d{6})",
    ),
)
Column(
    "somedate",
    sqlite.DATE(
        storage_format="%(month)02d/%(day)02d/%(year)04d",
        regexp="(?P<month>\d+)/(?P<day>\d+)/(?P<year>\d+)",
    ),
)
```

非常感谢 Nate Dub 在 Pycon 2012 上的努力。

另请参见

`DATETIME`

`DATE`

`TIME`

[#2363](https://www.sqlalchemy.org/trac/ticket/2363)

### “COLLATE”在所有方言中都受支持；特别是 MySQL，PostgreSQL，SQLite

“collate”关键字，长期以来被 MySQL 方言接受，现在已在所有`String` 类型上建立，并将在任何后端上呈现，包括在使用`MetaData.create_all()`和`cast()`等功能时：

```py
>>> stmt = select([cast(sometable.c.somechar, String(20, collation="utf8"))])
>>> print(stmt)
SELECT  CAST(sometable.somechar  AS  VARCHAR(20)  COLLATE  "utf8")  AS  anon_1
FROM  sometable 
```

另请参见

`String`

[#2276](https://www.sqlalchemy.org/trac/ticket/2276)

### 现在支持“前缀”用于`update()`, `delete()`

面向 MySQL，可以在任何这些结构中呈现“前缀”。例如：

```py
stmt = table.delete().prefix_with("LOW_PRIORITY", dialect="mysql")

stmt = table.update().prefix_with("LOW_PRIORITY", dialect="mysql")
```

该方法是新增的，除了已存在于`insert()`、`select()`和`Query`上的方法之外。

另请参阅

`Update.prefix_with()`

`Delete.prefix_with()`

`Insert.prefix_with()`

`Select.prefix_with()`

`Query.prefix_with()`

[#2431](https://www.sqlalchemy.org/trac/ticket/2431)

## 行为变化

### 将“待定”对象视为“孤儿”的考虑变得更加积极

这是 0.8 系列的一个后期添加，但希望新行为在更广泛的情况下更一致和直观。ORM 自至少 0.4 版本以来一直包括这样的行为，即一个“待定”对象，意味着它与`Session`关联但尚未插入数据库，当它成为“孤儿”时，即已与使用`delete-orphan`级联的父对象解除关联时，将自动从`Session`中清除。此行为旨在大致模拟持久对象的行为，其中 ORM 将根据分离事件的拦截发出 DELETE 以删除这些成为孤儿的对象。

行为变化适用于被多种父对象引用并且每个父对象都指定`delete-orphan`的对象；典型示例是在多对多模式中连接两种其他对象的关联对象。以前，行为是这样的，即当待定对象与*所有*父对象解除关联时才会被清除。随着行为变化，一旦待定对象与*任何*先前关联的父对象解除关联，该待定对象就会被清除。此行为旨在更接近持久对象的行为，即一旦与任何父对象解除关联，它们就会被删除。

较旧行为的基本原因可以追溯到至少版本 0.4，基本上是一种防御性决定，试图在对象仍在为 INSERT 构造时减轻混淆。但现实情况是，无论如何，只要对象附加到任何新父级，它就会立即重新与`Session`关联。

仍然可以刷新一个对象，该对象尚未与其所有必需的父级关联，如果该对象一开始就未与这些父级关联，或者如果它被清除，但随后通过后续附加事件重新与`Session`关联，但仍未完全关联。在这种情况下，预计数据库会发出完整性错误，因为很可能存在未填充的 NOT NULL 外键列。ORM 做出决定让这些 INSERT 尝试发生，基于这样的判断：一个只与其必需的父级部分关联但已经与其中一些父级积极关联的对象，更多的情况下是用户错误，而不是应该被悄悄跳过的有意遗漏 - 在这里悄悄跳过 INSERT 会使这种性质的用户错误非常难以调试。

对于可能依赖于旧行为的应用程序，可以通过将标志`legacy_is_orphan`指定为映射器选项来重新启用旧行为。

新行为允许以下测试用例正常工作：

```py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship, backref
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True)
    name = Column(String(64))

class UserKeyword(Base):
    __tablename__ = "user_keyword"
    user_id = Column(Integer, ForeignKey("user.id"), primary_key=True)
    keyword_id = Column(Integer, ForeignKey("keyword.id"), primary_key=True)

    user = relationship(
        User, backref=backref("user_keywords", cascade="all, delete-orphan")
    )

    keyword = relationship(
        "Keyword", backref=backref("user_keywords", cascade="all, delete-orphan")
    )

    # uncomment this to enable the old behavior
    # __mapper_args__ = {"legacy_is_orphan": True}

class Keyword(Base):
    __tablename__ = "keyword"
    id = Column(Integer, primary_key=True)
    keyword = Column("keyword", String(64))

from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# note we're using PostgreSQL to ensure that referential integrity
# is enforced, for demonstration purposes.
e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)

Base.metadata.drop_all(e)
Base.metadata.create_all(e)

session = Session(e)

u1 = User(name="u1")
k1 = Keyword(keyword="k1")

session.add_all([u1, k1])

uk1 = UserKeyword(keyword=k1, user=u1)

# previously, if session.flush() were called here,
# this operation would succeed, but if session.flush()
# were not called here, the operation fails with an
# integrity error.
# session.flush()
del u1.user_keywords[0]

session.commit()
```

[#2655](https://www.sqlalchemy.org/trac/ticket/2655)

### 在项目与会话关联之后而不是之前触发 after_attach 事件；before_attach 添加

使用 after_attach 的事件处理程序现在可以假定给定实例与给定会话关联：

```py
@event.listens_for(Session, "after_attach")
def after_attach(session, instance):
    assert instance in session
```

有些用例要求它按照这种方式工作。然而，其他用例要求该项*尚未*成为会话的一部分，例如，当一个查询旨在加载实例所需的某些状态时，首先会触发自动刷新并且否则会过早刷新目标对象。这些用例应该使用新的“before_attach”事件：

```py
@event.listens_for(Session, "before_attach")
def before_attach(session, instance):
    instance.some_necessary_attribute = (
        session.query(Widget).filter_by(instance.widget_name).first()
    )
```

[#2464](https://www.sqlalchemy.org/trac/ticket/2464)

### 查询现在像 select()一样自动相关

以前需要调用`Query.correlate()`才能使列或 WHERE 子查询与父级相关联：

```py
subq = (
    session.query(Entity.value)
    .filter(Entity.id == Parent.entity_id)
    .correlate(Parent)
    .as_scalar()
)
session.query(Parent).filter(subq == "some value")
```

这与普通的`select()`构造相反，后者默认情况下会自动相关。在 0.8 中，上述语句将自动相关：

```py
subq = session.query(Entity.value).filter(Entity.id == Parent.entity_id).as_scalar()
session.query(Parent).filter(subq == "some value")
```

就像在`select()`中一样，通过调用`query.correlate(None)`来禁用相关性，或者通过传递一个实体来手动设置相关性，`query.correlate(someentity)`。

[#2179](https://www.sqlalchemy.org/trac/ticket/2179)

### 相关性现在始终是上下文特定的

为了允许更广泛的相关性场景，`Select.correlate()` 和 `Query.correlate()` 的行为略有改变，即如果 SELECT 语句实际上在该上下文中使用，那么该语句将仅在 FROM 子句中省略“相关”的目标。此外，不再可能将作为外部 SELECT 语句中的 FROM 的 SELECT 语句“相关”（即省略）FROM 子句。

这个改变只会让 SQL 渲染变得更好，因为不再可能渲染出不合法的 SQL，即在选择的内容相对于 FROM 对象不足的情况下：

```py
from sqlalchemy.sql import table, column, select

t1 = table("t1", column("x"))
t2 = table("t2", column("y"))
s = select([t1, t2]).correlate(t1)

print(s)
```

在此更改之前，上述将返回：

```py
SELECT  t1.x,  t2.y  FROM  t2
```

当“t1”在任何 FROM 子句中都没有被引用时，这是无效的 SQL。

现在，在没有外部 SELECT 的情况下，它返回：

```py
SELECT  t1.x,  t2.y  FROM  t1,  t2
```

在 SELECT 中，相关性会如预期地生效：

```py
s2 = select([t1, t2]).where(t1.c.x == t2.c.y).where(t1.c.x == s)
print(s2)
```

```py
SELECT  t1.x,  t2.y  FROM  t1,  t2
WHERE  t1.x  =  t2.y  AND  t1.x  =
  (SELECT  t1.x,  t2.y  FROM  t2)
```

这个改变不会影响任何现有应用程序，因为对于正确构建的表达式，相关性行为保持不变。只有依赖于在非相关上下文中使用相关的 SELECT 的无效字符串输出的应用程序（很可能是在测试场景中），才会看到任何变化。

[#2668](https://www.sqlalchemy.org/trac/ticket/2668)  ### create_all() 和 drop_all() 现在将空列表视为如此

方法 `MetaData.create_all()` 和 `MetaData.drop_all()` 现在将接受一个空列表的 `Table` 对象，并且不会发出任何 CREATE 或 DROP 语句。以前，将空列表解释为对集合传递 `None`，并且将无条件地为所有项目发出 CREATE/DROP。

这是一个错误修复，但一些应用程序可能一直依赖于先前的行为。

[#2664](https://www.sqlalchemy.org/trac/ticket/2664)

### 修复了 `InstrumentationEvents` 的事件定位。

`InstrumentationEvents`系列事件目标已经记录，事件只会根据实际传递的类来触发。在 0.7 版本中，情况并非如此，应用于`InstrumentationEvents`的任何事件监听器都会对所有映射的类调用。在 0.8 版本中，添加了额外的逻辑，使事件只会为那些发送的类调用。这里的`propagate`标志默认设置为`True`，因为类仪器事件通常用于拦截尚未创建的类。

[#2590](https://www.sqlalchemy.org/trac/ticket/2590)

### 不再将“=”自动转换为 IN，用于与 MS-SQL 中的子查询进行比较

我们在 MSSQL 方言中发现了一个非常古老的行为，当用户执行类似以下操作时，它会试图拯救用户：

```py
scalar_subq = select([someothertable.c.id]).where(someothertable.c.data == "foo")
select([sometable]).where(sometable.c.id == scalar_subq)
```

SQL Server 不允许将相等比较与标量 SELECT 进行比较，即“x = (SELECT something)”。MSSQL 方言会将其转换为 IN。然而，当进行类似“(SELECT something) = x”的比较时，也会发生相同的情况，总体而言，这种猜测的水平超出了 SQLAlchemy 通常的范围，因此删除了这种行为。

[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

### 修复了`Session.is_modified()`的行为

`Session.is_modified()`方法接受一个参数`passive`，基本上不应该是必要的，所有情况下参数应该是值`True` - 当保持默认值`False`时，会影响数据库，并经常触发自动刷新，这将改变结果。在 0.8 版本中，`passive`参数将不起作用，并且未加载的属性永远不会检查历史记录，因为根据定义，未加载的属性上不会有待处理的状态更改。

另请参见

`Session.is_modified()`

[#2320](https://www.sqlalchemy.org/trac/ticket/2320)

### `Column.key`在`Select.c`属性中受到`Select.apply_labels()`的尊重

表达式系统的用户知道`Select.apply_labels()`会在每个列名前添加表名，影响从`Select.c`中可用的名称：

```py
s = select([table1]).apply_labels()
s.c.table1_col1
s.c.table1_col2
```

在 0.8 之前，如果`Column`具有不同的`Column.key`，则此键将被忽略，与未使用`Select.apply_labels()`时的不一致性：

```py
# before 0.8
table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
s = select([table1])
s.c.column_one  # would be accessible like this
s.c.col1  # would raise AttributeError

s = select([table1]).apply_labels()
s.c.table1_column_one  # would raise AttributeError
s.c.table1_col1  # would be accessible like this
```

在 0.8 中，`Column.key`在两种情况下都受到尊重：

```py
# with 0.8
table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
s = select([table1])
s.c.column_one  # works
s.c.col1  # AttributeError

s = select([table1]).apply_labels()
s.c.table1_column_one  # works
s.c.table1_col1  # AttributeError
```

关于“name”和“key”的所有其他行为都是相同的，包括渲染的 SQL 仍然使用形式`<tablename>_<colname>` - 这里的重点是防止`Column.key`内容被渲染到`SELECT`语句中，以便在`Column.key`中使用特殊/非 ASCII 字符时不会出现问题。

[#2397](https://www.sqlalchemy.org/trac/ticket/2397)

### `single_parent`警告现在是错误

一个`relationship()`，它是多对一或多对多关系，并指定了“cascade=’all, delete-orphan’”，这是一个尴尬但仍然支持的用例（带有限制），如果关系没有指定`single_parent=True`选项，现在将引发错误。以前只会发出警告，但在任何情况下几乎立即会在属性系统中跟随失败。

[#2405](https://www.sqlalchemy.org/trac/ticket/2405)

### 将`inspector`参数添加到`column_reflect`事件

0.7 添加了一个名为`column_reflect`的新事件，提供了每个列反射时可以增强的事件。我们在这个事件上稍微出了点错，因为事件没有提供获取当前用于反射的`Inspector`和`Connection`的方法，以防需要来自数据库的其他信息。由于这是一个尚未广泛使用的新事件，我们将直接在其中添加`inspector`参数：

```py
@event.listens_for(Table, "column_reflect")
def listen_for_col(inspector, table, column_info): ...
```

[#2418](https://www.sqlalchemy.org/trac/ticket/2418)

### 禁用 MySQL 的自动检测排序规则和大小写

MySQL 方言在`Engine`连接时第一次进行两次调用，其中一次非常昂贵，加载数据库中的所有可能排序规则以及大小写信息。这两个集合都不会用于任何 SQLAlchemy 函数，因此这些调用将被更改为不再自动发出。可能依赖于这些集合存在于`engine.dialect`上的应用程序将需要直接调用`_detect_collations()`和`_detect_casing()`。

[#2404](https://www.sqlalchemy.org/trac/ticket/2404)

### “未使用的列名”警告变为异常

在`insert()`或`update()`构造中引用不存在的列将引发错误而不是警告：

```py
t1 = table("t1", column("x"))
t1.insert().values(x=5, z=5)  # raises "Unconsumed column names: z"
```

[#2415](https://www.sqlalchemy.org/trac/ticket/2415)

### Inspector.get_primary_keys()已弃用，请使用 Inspector.get_pk_constraint

这两种`Inspector`上的方法是多余的，其中`get_primary_keys()`将返回与`get_pk_constraint()`相同的信息，只是不包括约束的名称：

```py
>>> insp.get_primary_keys()
["a", "b"]

>>> insp.get_pk_constraint()
{"name":"pk_constraint", "constrained_columns":["a", "b"]}
```

[#2422](https://www.sqlalchemy.org/trac/ticket/2422)

### 大多数情况下将禁用不区分大小写的结果行名称

一个非常古老的行为，`RowProxy`中的列名始终是不区分大小写比较的：

```py
>>> row = result.fetchone()
>>> row["foo"] == row["FOO"] == row["Foo"]
True
```

这是为了一些在早期需要这样做的方言，如 Oracle 和 Firebird，但在现代用法中，我们有更准确的方法来处理这两个平台的不区分大小写行为。

未来，此行为将仅可选地可用，通过将标志``case_sensitive=False``传递给``create_engine()``，但否则从行中请求的列名必须匹配大小写。

[#2423](https://www.sqlalchemy.org/trac/ticket/2423)

### `InstrumentationManager`和替代类仪器现在是一个扩展

`sqlalchemy.orm.interfaces.InstrumentationManager`类已移至`sqlalchemy.ext.instrumentation.InstrumentationManager`。 “替代仪器”系统是为了极少数需要使用现有或不寻常的类仪器系统的安装而构建的，并且通常很少使用。该系统的复杂性已导出到一个`ext.`模块中。直到被导入后才会使用，通常是当第三方库导入`InstrumentationManager`时，此时它将通过用`ExtendedInstrumentationRegistry`替换默认的`InstrumentationFactory`将其注入回`sqlalchemy.orm`中。

### 将“待定”对象视为“孤儿”的考虑变得更加激进

这是 0.8 系列的一个较晚添加，但希望新行为在更广泛的情况下通常更一致和直观。ORM 自至少 0.4 版本以来已经包含了这样的行为，即一个“待定”对象，意味着它与`Session`相关联，但尚未插入到数据库中，当它成为“孤儿”时，即已经与引用它的父对象解除关联，并且在配置的`relationship()`上使用`delete-orphan`级联时，将自动从`Session`中删除。此行为旨在大致反映持久对象（即已插入）的行为，ORM 将根据分离事件的拦截为这些成为孤儿的对象发出 DELETE。

行为变更适用于被多种类型父对象引用的对象，每种类型父对象都指定`delete-orphan`；典型示例是在多对多模式中桥接两种其他对象的关联对象。以前的行为是，挂起的对象仅在与*所有*父对象解除关联时才会被清除。通过行为变更，只要挂起的对象与先前相关联的*任何*父对象解除关联，它就会被清除。这种行为旨在更接近持久对象的行为，即只要它们与任何父对象解除关联，它们就会被删除。

较旧行为的基本理由可以追溯到至少版本 0.4，基本上是一种防御性决定，试图在对象仍在为 INSERT 构造时减轻混淆。但事实是，无论如何，只要对象附加到任何新父对象，它就会立即重新与`Session`相关联。

仍然可以刷新一个与所有必需父对象都不相关联的对象，如果该对象一开始就没有与这些父对象相关联，或者如果它被清除，但后来通过后续的附加事件重新与`Session`相关联但仍未完全相关联。在这种情况下，预计数据库会发出完整性错误，因为可能存在未填充的 NOT NULL 外键列。ORM 决定让这些 INSERT 尝试发生，基于这样的判断：一个只与其必需父对象部分相关联但已经积极与其中一些相关联的对象，往往更多是用户错误，而不是应该被悄悄跳过的有意遗漏 - 在这里悄悄跳过 INSERT 会使这种用户错误非常难以调试。

对于可能依赖于旧行为的应用程序，可以通过将标志`legacy_is_orphan`作为映射器选项指定，重新启用任何`Mapper`的旧行为。

新行为允许以下测试用例工作：

```py
from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship, backref
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True)
    name = Column(String(64))

class UserKeyword(Base):
    __tablename__ = "user_keyword"
    user_id = Column(Integer, ForeignKey("user.id"), primary_key=True)
    keyword_id = Column(Integer, ForeignKey("keyword.id"), primary_key=True)

    user = relationship(
        User, backref=backref("user_keywords", cascade="all, delete-orphan")
    )

    keyword = relationship(
        "Keyword", backref=backref("user_keywords", cascade="all, delete-orphan")
    )

    # uncomment this to enable the old behavior
    # __mapper_args__ = {"legacy_is_orphan": True}

class Keyword(Base):
    __tablename__ = "keyword"
    id = Column(Integer, primary_key=True)
    keyword = Column("keyword", String(64))

from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# note we're using PostgreSQL to ensure that referential integrity
# is enforced, for demonstration purposes.
e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)

Base.metadata.drop_all(e)
Base.metadata.create_all(e)

session = Session(e)

u1 = User(name="u1")
k1 = Keyword(keyword="k1")

session.add_all([u1, k1])

uk1 = UserKeyword(keyword=k1, user=u1)

# previously, if session.flush() were called here,
# this operation would succeed, but if session.flush()
# were not called here, the operation fails with an
# integrity error.
# session.flush()
del u1.user_keywords[0]

session.commit()
```

[#2655](https://www.sqlalchemy.org/trac/ticket/2655)

### after_attach 事件在项目与会话关联后而不是之前触发；添加了 before_attach

使用 after_attach 的事件处理程序现在可以假定给定的实例与给定的会话相关联：

```py
@event.listens_for(Session, "after_attach")
def after_attach(session, instance):
    assert instance in session
```

有些用例要求它按照这种方式工作。然而，其他用例要求该项*尚未*成为会话的一部分，比如当一个查询，旨在加载实例所需的某些状态时，首先发出自动刷新，否则会过早刷新目标对象。这些用例应该使用新的“before_attach”事件：

```py
@event.listens_for(Session, "before_attach")
def before_attach(session, instance):
    instance.some_necessary_attribute = (
        session.query(Widget).filter_by(instance.widget_name).first()
    )
```

[#2464](https://www.sqlalchemy.org/trac/ticket/2464)

### 查询现在会像`select()`一样自动关联

以前需要调用`Query.correlate()`才能使列或 WHERE 子查询与父级关联：

```py
subq = (
    session.query(Entity.value)
    .filter(Entity.id == Parent.entity_id)
    .correlate(Parent)
    .as_scalar()
)
session.query(Parent).filter(subq == "some value")
```

这是一个普通 `select()` 构造的相反行为，默认情况下会假定自动关联。在 0.8 中，上述语句将自动关联：

```py
subq = session.query(Entity.value).filter(Entity.id == Parent.entity_id).as_scalar()
session.query(Parent).filter(subq == "some value")
```

就像在 `select()` 中一样，可以通过调用 `query.correlate(None)` 来禁用关联，或者通过传递一个实体来手动设置，`query.correlate(someentity)`。

[#2179](https://www.sqlalchemy.org/trac/ticket/2179)

### 关联现在始终是上下文特定的

为了允许更广泛的关联情景，`Select.correlate()` 和`Query.correlate()` 的行为略有变化，这样 SELECT 语句将仅在实际使用它在该上下文中时，从 FROM 子句中省略“相关”的目标。此外，不再可能在一个封闭的 SELECT 语句中作为 FROM 放置一个 SELECT 语句来“关联”（即省略）一个 FROM 子句。

这个变化只会使 SQL 渲染变得更好，因为不再可能渲染出非法的 SQL，其中选择的 FROM 对象不足：

```py
from sqlalchemy.sql import table, column, select

t1 = table("t1", column("x"))
t2 = table("t2", column("y"))
s = select([t1, t2]).correlate(t1)

print(s)
```

在这个变化之前，上述内容会返回：

```py
SELECT  t1.x,  t2.y  FROM  t2
```

这是无效的 SQL，因为“t1”在任何 FROM 子句中都没有被引用。

现在，在没有封闭 SELECT 的情况下，它返回：

```py
SELECT  t1.x,  t2.y  FROM  t1,  t2
```

在 SELECT 中，关联会按预期生效：

```py
s2 = select([t1, t2]).where(t1.c.x == t2.c.y).where(t1.c.x == s)
print(s2)
```

```py
SELECT  t1.x,  t2.y  FROM  t1,  t2
WHERE  t1.x  =  t2.y  AND  t1.x  =
  (SELECT  t1.x,  t2.y  FROM  t2)
```

这个变化预计不会影响任何现有的应用程序，因为相关性行为对于正确构建的表达式保持不变。只有一个依赖于在非相关上下文中使用相关 SELECT 的无效字符串输出的应用程序，最有可能是在测试场景中，才会看到任何变化。

[#2668](https://www.sqlalchemy.org/trac/ticket/2668)

### create_all() 和 drop_all() 现在将尊重一个空列表

方法`MetaData.create_all()`和`MetaData.drop_all()`现在将接受一个空的`Table`对象列表，并且不会发出任何 CREATE 或 DROP 语句。以前，空列表被解释为传递`None`给一个集合，对于所有项目都会无条件发出 CREATE/DROP。

这是一个 bug 修复，但某些应用可能一直依赖于以前的行为。

[#2664](https://www.sqlalchemy.org/trac/ticket/2664)

### 修复了 `InstrumentationEvents` 的事件定位

`InstrumentationEvents` 系列事件目标已经记录，事件将根据实际传递的类来触发。直到 0.7 版本，这并不是这样，任何应用于 `InstrumentationEvents` 的事件监听器都会对所有映射的类调用。在 0.8 中，添加了额外的逻辑，使事件只会为那些传递的类调用。这里的 `propagate` 标志默认设置为 `True`，因为类仪器事件通常用于拦截尚未创建的类。

[#2590](https://www.sqlalchemy.org/trac/ticket/2590)

### 不再将“=”在 MS-SQL 中与子查询比较时自动转换为 IN

我们在 MSSQL 方言中发现了一个非常古老的行为，当用户尝试做类似这样的事情时，它会试图拯救用户：

```py
scalar_subq = select([someothertable.c.id]).where(someothertable.c.data == "foo")
select([sometable]).where(sometable.c.id == scalar_subq)
```

SQL Server 不允许将等号与标量 SELECT 进行比较，即，“x = (SELECT something)”。MSSQL 方言会将其转换为 IN。然而，当进行类似“(SELECT something) = x”的比较时，也会发生同样的情况，总体而言，这种猜测的水平超出了 SQLAlchemy 通常的范围，因此已移除该行为。

[#2277](https://www.sqlalchemy.org/trac/ticket/2277)

### 修复了 `Session.is_modified()` 的行为

`Session.is_modified()` 方法接受一个参数 `passive`，基本上不应该是必要的，所有情况下参数应该是值 `True` - 当保持默认值 `False` 时，它会影响到数据库，并经常触发自动刷新，这将改变结果。在 0.8 中，`passive` 参数将不会产生任何影响，并且未加载的属性永远不会被检查历史，因为根据定义，未加载的属性不会有待处理的状态更改。

另请参阅

`Session.is_modified()`

[#2320](https://www.sqlalchemy.org/trac/ticket/2320)

### `Column.key` 在 `select()` 的 `Select.c` 属性中通过 `Select.apply_labels()` 得到尊重

表达式系统的用户知道`Select.apply_labels()`会在每个列名前面添加表名，影响从`Select.c`中可用的名称：

```py
s = select([table1]).apply_labels()
s.c.table1_col1
s.c.table1_col2
```

在 0.8 版本之前，如果`Column`的`Column.key`不同，这个键会被忽略，与未使用`Select.apply_labels()`时不一致：

```py
# before 0.8
table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
s = select([table1])
s.c.column_one  # would be accessible like this
s.c.col1  # would raise AttributeError

s = select([table1]).apply_labels()
s.c.table1_column_one  # would raise AttributeError
s.c.table1_col1  # would be accessible like this
```

在 0.8 版本中，`Column.key`在两种情况下都受到尊重：

```py
# with 0.8
table1 = Table("t1", metadata, Column("col1", Integer, key="column_one"))
s = select([table1])
s.c.column_one  # works
s.c.col1  # AttributeError

s = select([table1]).apply_labels()
s.c.table1_column_one  # works
s.c.table1_col1  # AttributeError
```

关于“name”和“key”的所有其他行为都是相同的，包括渲染的 SQL 仍然会使用`<tablename>_<colname>`的形式 - 这里的重点是防止`Column.key`的内容被渲染到`SELECT`语句中，以避免在`Column.key`中使用特殊/非 ASCII 字符时出现问题。

[#2397](https://www.sqlalchemy.org/trac/ticket/2397)

### `single_parent`警告现在变成了错误

一个`relationship()`，它是多对一或多对多关系，并指定“cascade='all, delete-orphan'”，这是一个尴尬但仍然支持的用例（受限制），如果关系没有指定`single_parent=True`选项，现在将引发错误。以前它只会发出警告，但在任何情况下几乎立即会在属性系统中跟随失败。

[#2405](https://www.sqlalchemy.org/trac/ticket/2405)

### 将`inspector`参数添加到`column_reflect`事件中

0.7 版本添加了一个名为`column_reflect`的新事件，提供了对每个反射的列进行增强的方式。我们在这个事件中稍微出了点错，因为事件没有提供访问当前用于反射的`Inspector`和`Connection`的方法，以防需要来自数据库的附加信息。由于这是一个尚未广泛使用的新事件，我们将直接在其中添加`inspector`参数：

```py
@event.listens_for(Table, "column_reflect")
def listen_for_col(inspector, table, column_info): ...
```

[#2418](https://www.sqlalchemy.org/trac/ticket/2418)

### 禁用 MySQL 的自动检测排序规则和大小写敏感性

MySQL 方言进行两次调用，其中一次非常昂贵，从数据库加载所有可能的排序规则以及大小写敏感性的信息，第一次引擎连接时。这两个集合都不会被任何 SQLAlchemy 函数使用，因此这些调用将被更改为不再自动发出。可能依赖于这些集合存在于`engine.dialect`上的应用程序将需要直接调用`_detect_collations()`和`_detect_casing()`。

[#2404](https://www.sqlalchemy.org/trac/ticket/2404)

### “未消耗的列名” 警告变为异常

在 `insert()` 或 `update()` 构造中引用不存在的列将引发错误而不是警告：

```py
t1 = table("t1", column("x"))
t1.insert().values(x=5, z=5)  # raises "Unconsumed column names: z"
```

[#2415](https://www.sqlalchemy.org/trac/ticket/2415)

### Inspector.get_primary_keys() 已弃用，请使用 Inspector.get_pk_constraint

`Inspector` 上的这两种方法是多余的，`get_primary_keys()` 将返回与 `get_pk_constraint()` 相同的信息，但不包括约束的名称：

```py
>>> insp.get_primary_keys()
["a", "b"]

>>> insp.get_pk_constraint()
{"name":"pk_constraint", "constrained_columns":["a", "b"]}
```

[#2422](https://www.sqlalchemy.org/trac/ticket/2422)

### 大多数情况下将禁用不区分大小写的结果行名称

一个非常古老的行为，`RowProxy` 中的列名总是不区分大小写地进行比较：

```py
>>> row = result.fetchone()
>>> row["foo"] == row["FOO"] == row["Foo"]
True
```

这是为了一些在早期需要这样做的方言的利益，比如 Oracle 和 Firebird，但在现代用法中，我们有更准确的方法来处理这两个平台的不区分大小写行为。

未来，此行为将仅可选地通过向 `create_engine()` 传递标志 ``case_sensitive=False`` 来使用，但否则从行中请求的列名必须匹配大小写。

[#2423](https://www.sqlalchemy.org/trac/ticket/2423)

### `InstrumentationManager` 和替代类仪器现在是一个扩展

`sqlalchemy.orm.interfaces.InstrumentationManager` 类已移动到 `sqlalchemy.ext.instrumentation.InstrumentationManager`。 “替代仪器”系统是为了极少数需要使用现有或不寻常的类仪器系统的安装而构建的，并且通常很少使用。 这个系统的复杂性已导出到一个 `ext.` 模块中。 它在导入一次后保持未使用，通常是在第三方库导入 `InstrumentationManager` 时，此时通过用 `ExtendedInstrumentationRegistry` 替换默认的 `InstrumentationFactory` 将其注入回 `sqlalchemy.orm`。

## 已移除

### SQLSoup

SQLSoup 是一个方便的包，它在 SQLAlchemy ORM 之上提供了一个替代接口。 SQLSoup 现在已移至其自己的项目，并单独进行了文档化/发布；请参见 [`bitbucket.org/zzzeek/sqlsoup`](https://bitbucket.org/zzzeek/sqlsoup)。

SQLSoup 是一个非常简单的工具，也可以受益于对其使用方式感兴趣的贡献者。

[#2262](https://www.sqlalchemy.org/trac/ticket/2262)

### 可变类型

SQLAlchemy ORM 中的旧的“可变”系统已经移除。这指的是应用于诸如`PickleType`的类型和有条件地应用于`TypeDecorator`的`MutableType`接口，自很早的 SQLAlchemy 版本以来一直提供了一种让 ORM 检测所谓的“可变”数据结构（如 JSON 结构和 pickled 对象）变化的方式。然而，该实现从未合理，强制单元操作工作在非常低效的模式下运行，导致在 flush 期间对所有对象进行昂贵的扫描。在 0.7 版本中，引入了[sqlalchemy.ext.mutable](https://docs.sqlalchemy.org/en/latest/orm/extensions/mutable.html)扩展，以便用户定义的数据类型可以在发生更改时适当地向单元操作发送事件。

今天，`MutableType`的使用预计会很少，因为多年来已经发出了有关其低效性的警告。

[#2442](https://www.sqlalchemy.org/trac/ticket/2442)

### sqlalchemy.exceptions（多年来一直是 sqlalchemy.exc）

我们保留了别名`sqlalchemy.exceptions`，以尝试使一些非常旧的库稍微容易些，这些库尚未升级以使用`sqlalchemy.exc`。然而，一些用户仍然被困惑，因此在 0.8 版本中，我们将其完全删除，以消除任何困惑。

[#2433](https://www.sqlalchemy.org/trac/ticket/2433)

### SQLSoup

SQLSoup 是一个方便的包，它在 SQLAlchemy ORM 之上提供了一个替代接口。SQLSoup 现在已移动到自己的项目中，并且进行了单独的文档编写/发布；请参阅[`bitbucket.org/zzzeek/sqlsoup`](https://bitbucket.org/zzzeek/sqlsoup)。

SQLSoup 是一个非常简单的工具，也可以从对其使用风格感兴趣的贡献者中受益。

[#2262](https://www.sqlalchemy.org/trac/ticket/2262)

### MutableType

SQLAlchemy ORM 中的旧的“可变”系统已经移除。这指的是应用于诸如`PickleType`的类型和有条件地应用于`TypeDecorator`的`MutableType`接口，自很早的 SQLAlchemy 版本以来一直提供了一种让 ORM 检测所谓的“可变”数据结构（如 JSON 结构和 pickled 对象）变化的方式。然而，该实现从未合理，强制单元操作工作在非常低效的模式下运行，导致在 flush 期间对所有对象进行昂贵的扫描。在 0.7 版本中，引入了[sqlalchemy.ext.mutable](https://docs.sqlalchemy.org/en/latest/orm/extensions/mutable.html)扩展，以便用户定义的数据类型可以在发生更改时适当地向单元操作发送事件。

今天，`MutableType`的使用预计会很少，因为多年来已经发出了有关其低效性的警告。

[#2442](https://www.sqlalchemy.org/trac/ticket/2442)

### sqlalchemy.exceptions（多年来一直是 sqlalchemy.exc）

我们曾经在别名`sqlalchemy.exceptions`中尝试让一些非常老旧的库更容易使用`sqlalchemy.exc`。然而，一些用户仍然感到困惑，因此在 0.8 版本中，我们将完全删除它，以消除任何困惑。

[#2433](https://www.sqlalchemy.org/trac/ticket/2433)
