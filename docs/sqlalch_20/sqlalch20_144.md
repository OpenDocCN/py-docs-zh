# What’s New in SQLAlchemy 1.1?

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_11.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_11.html)

About this Document

本文档描述了 SQLAlchemy 版本 1.0 和版本 1.1 之间的变化。

## Introduction

本指南介绍了 SQLAlchemy 1.1 版本的新功能，并记录了影响用户将其应用程序从 SQLAlchemy 1.0 系列迁移到 1.1 系列的变化。

请仔细查看关于行为变化的部分，可能存在与旧版本不兼容的行为变化。

## 平台/安装程序变更

### Setuptools is now required for install

SQLAlchemy 的`setup.py`文件多年来一直支持安装 Setuptools 或不安装 Setuptools 两种方式；支持“回退”模式，即使用普通的 Distutils。由于现在几乎不再听说不安装 Setuptools 的 Python 环境了，并且为了更全面地支持 Setuptools 的特性集，特别是支持 py.test 与其集成以及诸如“extras”之类的功能，`setup.py`现在完全依赖于 Setuptools。

See also

安装指南

[#3489](https://www.sqlalchemy.org/trac/ticket/3489)

### 仅通过环境变量启用/禁用 C 扩展构建

默认情况下，在安装时构建 C 扩展，只要可能。要禁用 C 扩展构建，从 SQLAlchemy 0.8.6 / 0.9.4 开始，可使用`DISABLE_SQLALCHEMY_CEXT`环境变量。之前使用`--without-cextensions`参数的方法已被移除，因为它依赖于已弃用的 setuptools 功能。

See also

构建 Cython 扩展

[#3500](https://www.sqlalchemy.org/trac/ticket/3500)

## 新功能和改进 - ORM

### 新的会话生命周期事件

`Session`长期以来一直支持事件，允许在某种程度上跟踪对象状态的变化，包括`SessionEvents.before_attach()`、`SessionEvents.after_attach()`和`SessionEvents.before_flush()`。Session 文档还在快速对象状态介绍中记录了主要的对象状态。然而，过去从未有过专门跟踪对象经过这些转换的系统。此外，“已删除”对象的状态一直比较模糊，因为这些对象的行为介于“持久”和“分离”状态之间。

为了清理这个领域并使会话状态转换的范围完全透明化，已添加了一系列新事件，旨在涵盖对象可能在状态之间转换的每种可能方式，而且还给“已删除”状态在会话对象状态领域内赋予了自己的正式状态名称。

#### 新状态转换事件

对象所有状态之间的转换，如 persistent、pending 等，现在都可以通过会话级事件的方式进行拦截，以涵盖特定的转换。当对象进入 `Session`、从 `Session` 移出，甚至当使用 `Session.rollback()` 回滚事务时发生的所有转换都明确出现在 `SessionEvents` 接口中。

总共有**十个新事件**。这些事件的摘要在新编写的文档部分对象生命周期事件中。

#### 添加了新的对象状态“deleted”，已删除的对象不再“persistent”

对象在 `Session` 中的 persistent 状态一直以来都被记录为具有有效的数据库标识；然而，对于在 flush 中被删除的对象，它们一直处于一个灰色地带，在这个地带中，它们并没有真正“分离”出 `Session`，因为它们仍然可以在回滚时恢复，但它们并不真正“persistent”，因为它们的数据库标识已被删除，而且它们不在标识映射中。

为了解决这一新事件中的灰色地带，引入了一个新的对象状态删除。此状态存在于“持久”状态和“分离”状态之间。通过 `Session.delete()` 标记为删除的对象保持在“持久”状态，直到刷新进行为止；在那时，它将从身份映射中删除，转移到“已删除”状态，并调用 `SessionEvents.persistent_to_deleted()` 钩子。如果 `Session` 对象的事务被回滚，则对象将被恢复为持久状态；将调用 `SessionEvents.deleted_to_persistent()` 转换。否则，如果 `Session` 对象的事务被提交，则调用 `SessionEvents.deleted_to_detached()` 转换。

另外，`InstanceState.persistent` 访问器**不再返回 True**，表示对象处于新的“已删除”状态；相反，增强了 `InstanceState.deleted` 访问器以可靠地报告此新状态。当对象分离时，`InstanceState.deleted` 返回 False，并且 `InstanceState.detached` 访问器为 True。要确定对象是否在当前事务或上一个事务中被删除，请使用 `InstanceState.was_deleted` 访问器。

#### 强身份映射已被弃用。

新系列过渡事件的灵感之一是为了实现对象在进出标识映射时的无泄漏跟踪，以便维护“强引用”，反映对象在此映射中进出的情况。有了这种新能力，就不再需要 `Session.weak_identity_map` 参数和相应的 `StrongIdentityMap` 对象。这个选项在 SQLAlchemy 中已经存在多年，因为“强引用”行为曾经是唯一可用的行为，许多应用程序都被写成假定这种行为。长期以来，一直建议对象的强引用跟踪不是 `Session` 的固有工作，而是一个应用级别的构造，根据应用程序的需要构建；新的事件模型甚至允许复制强标识映射的确切行为。查看 会话引用行为 以获取一个新的示例，说明如何替换强标识映射。

[#2677](https://www.sqlalchemy.org/trac/ticket/2677)  ### 新的 init_scalar() 事件拦截 ORM 级别的默认值

当首次访问未设置的属性时，ORM 会为非持久化对象生成一个值为 `None`：

```py
>>> obj = MyObj()
>>> obj.some_value
None
```

有一个用例是为了在对象持久化之前，使得 Python 中的值与 Core 生成的默认值对应。为了适应这种用例，添加了一个新的事件 `AttributeEvents.init_scalar()`。在 属性仪器化 中的新示例 `active_column_defaults.py` 说明了一个示例用法，因此效果可以是：

```py
>>> obj = MyObj()
>>> obj.some_value
"my default"
```

[#1311](https://www.sqlalchemy.org/trac/ticket/1311)  ### 关于“不可哈希”类型的更改，影响 ORM 行的去重

`Query` 对象具有“去重”返回行的良好行为，其中包含至少一个 ORM 映射实体（例如，一个完全映射的对象，而不是单独的列值）。这样做的主要目的是为了使实体的处理与标识映射顺利配合，包括适应通常在连接式急加载中表示的重复实体，以及在使用连接来过滤其他列时。

这种去重依赖于行内元素的可哈希性。随着 PostgreSQL 的特殊类型如`ARRAY`、`HSTORE`和`JSON`的引入，行内类型被标记为不可哈希并在这里遇到问题的经验比以前更普遍。

实际上，SQLAlchemy 自版本 0.8 以来在被标记为“unhashable”的数据类型上包含了一个标志，然而这个标志在内置类型上并没有一致使用。正如 ARRAY 和 JSON 类型现在正确指定“unhashable”中所描述的，这个标志现在对所有 PostgreSQL 的“结构”类型一致设置。

“unhashable”标志也设置在`NullType`类型上，因为`NullType`用于引用任何未知类型的表达式。

由于`NullType`应用于大多数`func`的用法，因为`func`实际上在大多数情况下并不知道给定的函数名称，**使用 func()通常会禁用行去重，除非应用了显式类型**。以下示例说明了将`func.substr()`应用于字符串表达式，以及将`func.date()`应用于日期时间表达式；这两个示例将由于连接的急加载而返回重复行，除非应用了显式类型：

```py
result = (
    session.query(func.substr(A.some_thing, 0, 4), A).options(joinedload(A.bs)).all()
)

users = (
    session.query(
        func.date(User.date_created, "start of month").label("month"),
        User,
    )
    .options(joinedload(User.orders))
    .all()
)
```

为了保留去重，上述示例应该指定为：

```py
result = (
    session.query(func.substr(A.some_thing, 0, 4, type_=String), A)
    .options(joinedload(A.bs))
    .all()
)

users = (
    session.query(
        func.date(User.date_created, "start of month", type_=DateTime).label("month"),
        User,
    )
    .options(joinedload(User.orders))
    .all()
)
```

另外，所谓的“unhashable”类型的处理与以前的版本略有不同；在内部，我们使用`id()`函数从这些结构中获取“哈希值”，就像我们对待任何普通映射对象一样。这取代了以前将计数器应用于对象的方法。

[#3499](https://www.sqlalchemy.org/trac/ticket/3499)  ### 为传递映射类、实例作为 SQL 文字添加了特定检查

现在，类型系统对于在本应被处理为字面值的上下文中传递 SQLAlchemy“可检查”对象具有特定检查。任何 SQLAlchemy 内置对象，如果作为 SQL 值传递是合法的（而不是已经是`ClauseElement`实例），都包括一个`__clause_element__()`方法，该方法为该对象提供有效的 SQL 表达式。对于不提供此方法的 SQLAlchemy 对象，例如映射类、映射器和映射实例，会发出更具信息性的错误消息，而不是允许 DBAPI 接收对象并稍后失败。下面举例说明了一个情况，其中基于字符串的属性`User.name`与`User()`的完整实例进行比较，而不是与字符串值进行比较：

```py
>>> some_user = User()
>>> q = s.query(User).filter(User.name == some_user)
sqlalchemy.exc.ArgumentError: Object <__main__.User object at 0x103167e90> is not legal as a SQL literal value
```

当比较`User.name == some_user`时，异常现在会立即发生。以前，像上面这样的比较会产生一个 SQL 表达式，只有在解析为 DBAPI 执行调用时才会失败；映射的`User`对象最终会变成一个被 DBAPI 拒绝的绑定参数。

请注意，在上面的示例中，表达式失败是因为`User.name`是一个基于字符串的（例如基于列的）属性。这种变化*不会*影响通常情况下将多对一关系属性与对象进行比较的情况，这种情况会被单独处理：

```py
>>> # Address.user refers to the User mapper, so
>>> # this is of course still OK!
>>> q = s.query(Address).filter(Address.user == some_user)
```

[#3321](https://www.sqlalchemy.org/trac/ticket/3321)  ### 新的可索引 ORM 扩展

可索引 扩展是混合属性功能的扩展，允许构建引用“可索引”数据类型的特定元素的属性，例如数组或 JSON 字段：

```py
class Person(Base):
    __tablename__ = "person"

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    name = index_property("data", "name")
```

上面，`name`属性将从 JSON 列`data`中读取/写入字段`"name"`，在初始化为空字典后：

```py
>>> person = Person(name="foobar")
>>> person.name
foobar
```

当修改属性时，该扩展还会触发一个更改事件，因此无需使用`MutableDict`来跟踪此更改。

另请参阅

可索引  ### 新选项允许显式持久化 NULL 值而不是默认值

与 PostgreSQL 中新增的 JSON-NULL 支持相关，作为 JSON “null” 在 ORM 操作中被插入时的预期行为，当不存在时被省略 的一部分，基础 `TypeEngine` 类现在支持一个方法 `TypeEngine.evaluates_none()`，允许将属性上的 `None` 值设置为 NULL，而不是在 INSERT 语句中省略列，这会导致使用列级默认值的效果。这允许在映射器级别配置现有的对象级技术，将 `null()` 分配给属性。

另请参阅

强制在具有默认值的列上使用 NULL

[#3250](https://www.sqlalchemy.org/trac/ticket/3250)  ### 进一步修复单表继承查询

继续从 1.0 的 使用 from_self(), count() 时对单表继承条件的更改，`Query` 在查询针对子查询表达式时，如 exists 时，不应再不适当地添加“单一继承”条件：

```py
class Widget(Base):
    __tablename__ = "widget"
    id = Column(Integer, primary_key=True)
    type = Column(String)
    data = Column(String)
    __mapper_args__ = {"polymorphic_on": type}

class FooWidget(Widget):
    __mapper_args__ = {"polymorphic_identity": "foo"}

q = session.query(FooWidget).filter(FooWidget.data == "bar").exists()

session.query(q).all()
```

产生：

```py
SELECT  EXISTS  (SELECT  1
FROM  widget
WHERE  widget.data  =  :data_1  AND  widget.type  IN  (:type_1))  AS  anon_1
```

里面的 IN 子句是适当的，以限制为 FooWidget 对象，但以前 IN 子句也会在子查询之外生成第二次。

[#3582](https://www.sqlalchemy.org/trac/ticket/3582)  ### 当数据库取消 SAVEPOINT 时改进了会话状态

MySQL 的一个常见情况是在事务中发生死锁时取消 SAVEPOINT。`Session` 已经被修改，以更为优雅地处理这种失败模式，使得外部的非 SAVEPOINT 事务仍然可用：

```py
s = Session()
s.begin_nested()

s.add(SomeObject())

try:
    # assume the flush fails, flush goes to rollback to the
    # savepoint and that also fails
    s.flush()
except Exception as err:
    print("Something broke, and our SAVEPOINT vanished too")

# this is the SAVEPOINT transaction, marked as
# DEACTIVE so the rollback() call succeeds
s.rollback()

# this is the outermost transaction, remains ACTIVE
# so rollback() or commit() can succeed
s.rollback()
```

这个问题是 [#2696](https://www.sqlalchemy.org/trac/ticket/2696) 的延续，我们发出警告，以便在 Python 2 上运行时可以看到原始错误，即使 SAVEPOINT 异常优先。在 Python 3 上，异常被链接在一起，因此两个失败都会被单独报告。

[#3680](https://www.sqlalchemy.org/trac/ticket/3680)  ### 修复了错误的“新实例 X 与持久实例 Y 冲突”刷新错误

`Session.rollback()`方法负责删除在数据库中插入的对象，例如在那个现在回滚的事务中从挂起状态移动到持久状态。进行此状态更改的对象在一个弱引用集合中被跟踪，如果一个对象从该集合中被垃圾回收，`Session`不再关心它（否则对于在事务中插入许多新对象的操作不会扩展）。然而，如果应用程序在事务中重新加载相同的被垃圾回收的行，在回滚发生之前会出现问题；如果对这个对象的强引用保留到下一个事务中，那么这个对象未被插入并应该被删除的事实将丢失，并且 flush 将不正确地引发错误：

```py
from sqlalchemy import Column, create_engine
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

s = Session(e)

# persist an object
s.add(A(id=1))
s.flush()

# rollback buffer loses reference to A

# load it again, rollback buffer knows nothing
# about it
a1 = s.query(A).first()

# roll back the transaction; all state is expired but the
# "a1" reference remains
s.rollback()

# previous "a1" conflicts with the new one because we aren't
# checking that it never got committed
s.add(A(id=1))
s.commit()
```

上述程序将引发：

```py
FlushError: New instance <User at 0x7f0287eca4d0> with identity key
(<class 'test.orm.test_transaction.User'>, ('u1',)) conflicts
with persistent instance <User at 0x7f02889c70d0>
```

bug 在于当上述异常被触发时，工作单元正在操作原始对象，假设它是一个活动行，而实际上对象已过期，并在测试中发现它已经消失。修复现在测试这种情况，因此在 SQL 日志中我们看到：

```py
BEGIN  (implicit)

INSERT  INTO  a  (id)  VALUES  (?)
(1,)

SELECT  a.id  AS  a_id  FROM  a  LIMIT  ?  OFFSET  ?
(1,  0)

ROLLBACK

BEGIN  (implicit)

SELECT  a.id  AS  a_id  FROM  a  WHERE  a.id  =  ?
(1,)

INSERT  INTO  a  (id)  VALUES  (?)
(1,)

COMMIT
```

上面，工作单元现在为我们即将报告为冲突的行执行 SELECT，看到它不存在，并正常进行。只有在我们本来会在任何情况下错误地引发异常时，才会发生这个 SELECT 的开销。

[#3677](https://www.sqlalchemy.org/trac/ticket/3677)  ### 联接继承映射的 passive_deletes 功能

现在，联接表继承映射可能允许 DELETE 继续进行，作为`Session.delete()`的结果，该方法仅为基表发出 DELETE，而不是子类表，从而允许配置的 ON DELETE CASCADE 发生在配置的外键上。这是使用`mapper.passive_deletes`选项配置的：

```py
from sqlalchemy import Column, Integer, String, ForeignKey, create_engine
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column("id", Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "a",
        "passive_deletes": True,
    }

class B(A):
    __tablename__ = "b"
    b_table_id = Column("b_table_id", Integer, primary_key=True)
    bid = Column("bid", Integer, ForeignKey("a.id", ondelete="CASCADE"))
    data = Column("data", String)

    __mapper_args__ = {"polymorphic_identity": "b"}
```

使用上述映射，`mapper.passive_deletes`选项在基本映射器上配置；它对于所有具有设置选项的祖先映射器的非基本映射器生效。对于类型为`B`的对象的 DELETE 不再需要检索`b_table_id`的主键值（如果未加载），也不需要为表本身发出 DELETE 语句：

```py
session.delete(some_b)
session.commit()
```

将生成的 SQL 如下：

```py
DELETE  FROM  a  WHERE  a.id  =  %(id)s
-- {'id': 1}
COMMIT
```

一如既往，目标数据库必须具有启用 ON DELETE CASCADE 的外键支持。

[#2349](https://www.sqlalchemy.org/trac/ticket/2349)  ### 同名反向引用应用于具体继承子类时不会引发错误

以下映射一直是可能的而没有问题：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b = relationship("B", foreign_keys="B.a_id", backref="a")

class A1(A):
    __tablename__ = "a1"
    id = Column(Integer, primary_key=True)
    b = relationship("B", foreign_keys="B.a1_id", backref="a1")
    __mapper_args__ = {"concrete": True}

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)

    a_id = Column(ForeignKey("a.id"))
    a1_id = Column(ForeignKey("a1.id"))
```

上面，即使类`A`和类`A1`有一个名为`b`的关系，也不会出现冲突警告或错误，因为类`A1`被标记为“具体”。

然而，如果关系配置反过来，就会出现错误：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class A1(A):
    __tablename__ = "a1"
    id = Column(Integer, primary_key=True)
    __mapper_args__ = {"concrete": True}

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)

    a_id = Column(ForeignKey("a.id"))
    a1_id = Column(ForeignKey("a1.id"))

    a = relationship("A", backref="b")
    a1 = relationship("A1", backref="b")
```

修复增强了反向引用功能，以便不发出错误，并在映射器逻辑内部进行额外检查，以跳过替换属性的警告。

[#3630](https://www.sqlalchemy.org/trac/ticket/3630)  ### 继承映射器上的同名关系不再发出警告

在继承场景中创建两个映射器时，在两者上都放置同名关系会发出警告“映射器`<name>`上的关系‘<name>’取代了继承映射器‘<name>`上的相同关系；这可能在刷新期间引起依赖问题”。示例如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

class ASub(A):
    __tablename__ = "a_sub"
    id = Column(Integer, ForeignKey("a.id"), primary_key=True)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

此警告可以追溯到 2007 年的 0.4 系列，基于一个自那时完全重写的工作单元代码版本。目前，没有关于在基类和子类上放置同名关系的已知问题，因此警告被取消。但是，请注意，由于警告，这种用例在现实世界中可能不常见。虽然为这种用例添加了基本的测试支持，但可能会发现这种模式的一些新问题。

1.1.0b3 版本中新增。

[#3749](https://www.sqlalchemy.org/trac/ticket/3749)  ### 混合属性和方法现在也传播文档字符串以及`.info`

现在，混合方法或属性将反映原始文档字符串中存在的`__doc__`值：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    name = Column(String)

    @hybrid_property
    def some_name(self):
  """The name field"""
        return self.name
```

现在，`A.some_name.__doc__`的上述值被尊重：

```py
>>> A.some_name.__doc__
The name field
```

然而，为了实现这一点，混合属性的机制必然变得更加复杂。以前，混合的类级访问器将是一个简单的传递，也就是说，这个测试将成功：

```py
>>> assert A.name is A.some_name
```

随着变化，`A.some_name`返回的表达式被包装在自己的`QueryableAttribute`包装器内：

```py
>>> A.some_name
<sqlalchemy.orm.attributes.hybrid_propertyProxy object at 0x7fde03888230>
```

进行了大量测试，以确保此包装器正常工作，包括对[自定义值对象](https://techspot.zzzeek.org/2011/10/21/hybrids-and-value-agnostic-types/)配方等复杂方案的测试，但我们将继续查看用户是否出现其他退化情况。

作为这一变化的一部分，`hybrid_property.info`集合现在也从混合描述符本身传播，而不是从底层表达式传播。也就是说，访问`A.some_name.info`现在返回与`inspect(A).all_orm_descriptors['some_name'].info`相同的字典：

```py
>>> A.some_name.info["foo"] = "bar"
>>> from sqlalchemy import inspect
>>> inspect(A).all_orm_descriptors["some_name"].info
{'foo': 'bar'}
```

请注意，这个`.info`字典**独立**于混合描述符可能直接代理的映射属性的字典；这是从 1.0 开始的行为变化。包装器仍将代理镜像属性的其他有用属性，如`QueryableAttribute.property`和`QueryableAttribute.class_`。

[#3653](https://www.sqlalchemy.org/trac/ticket/3653)  ### Session.merge 解决挂起冲突与持久性相同

`Session.merge()`方法现在将跟踪图中给定对象的标识，以维护主键的唯一性，然后再发出 INSERT。当遇到相同标识的重复对象时，非主键属性会被**覆盖**，因为对象被遇到时是基本上是非确定性的。这种行为与持久对象的行为相匹配，也就是通过主键已经位于数据库中的对象，因此这种行为更具内部一致性。

给定：

```py
u1 = User(id=7, name="x")
u1.orders = [
    Order(description="o1", address=Address(id=1, email_address="a")),
    Order(description="o2", address=Address(id=1, email_address="b")),
    Order(description="o3", address=Address(id=1, email_address="c")),
]

sess = Session()
sess.merge(u1)
```

在上面的例子中，我们将一个`User`对象与三个新的`Order`对象合并，每个对象都指向一个不同的`Address`对象，但是它们都具有相同的主键。`Session.merge()`的当前行为是在标识映射中查找这个`Address`对象，并将其用作目标。如果对象存在，意味着数据库已经有一个带有主键“1”的`Address`行，我们可以看到`Address`的`email_address`字段将在这种情况下被覆盖三次，分别为 a，b 和最后是 c。

然而，如果主键“1”对应的`Address`行不存在，`Session.merge()`将创建三个单独的`Address`实例，然后在 INSERT 时会出现主键冲突。新行为是，这些`Address`对象的拟议主键被跟踪在一个单独的字典中，以便我们将三个拟议的`Address`对象的状态合并到一个要插入的`Address`对象上。

如果原始情况下发出某种警告，表明在单个合并树中存在冲突数据可能更好，但是对于持久情况，多年来非确定性合并值一直是行为方式；现在对于挂起情况也是如此。警告存在冲突值的功能仍然对于两种情况都是可行的，但是会增加相当大的性能开销，因为在合并过程中每个列值都必须进行比较。

[#3601](https://www.sqlalchemy.org/trac/ticket/3601)  ### 修复涉及用户发起的外键操作的多对一对象移动

已修复了涉及用另一个对象替换多对一引用机制的 bug。在属性操作期间，先前引用的对象位置现在使用数据库提交的外键值，而不是当前外键值。修复的主要效果是，当进行多对一更改时，即使在手动将外键属性移动到新值之前，也会更准确地触发向集合的 backref 事件。假设类 `Parent` 和 `SomeClass` 的映射，其中 `SomeClass.parent` 指向 `Parent`，`Parent.items` 指向 `SomeClass` 对象的集合：

```py
some_object = SomeClass()
session.add(some_object)
some_object.parent_id = some_parent.id
some_object.parent = some_parent
```

在上面，我们创建了一个待处理对象 `some_object`，将其外键指向 `Parent`，*然后*我们实际设置了关系。在修复错误之前，backref 不会被触发：

```py
# before the fix
assert some_object not in some_parent.items
```

现在的修复是，当我们试图定位 `some_object.parent` 的先前值时，我们忽略了手动设置的父 id，并寻找数据库提交的值。在这种情况下，它是 None，因为对象是待处理的，所以事件系统将 `some_object.parent` 记录为净变化：

```py
# after the fix, backref fired off for some_object.parent = some_parent
assert some_object in some_parent.items
```

尽管不鼓励操作由关系管理的外键属性，但对于这种用例有有限的支持。为了允许加载继续进行，经常会使用 `Session.enable_relationship_loading()` 和 `RelationshipProperty.load_on_pending` 功能，这些功能会导致基于内存中未持久化的外键值发出延迟加载的关系。无论是否使用这些功能，这种行为改进现在都会显而易见。

[#3708](https://www.sqlalchemy.org/trac/ticket/3708)  ### 改进了具有多态实体的 Query.correlate 方法

在最近的 SQLAlchemy 版本中，许多形式的“多态”查询生成的 SQL 比以前更“扁平化”，不再无条件地将多个表的 JOIN 捆绑到子查询中。为了适应这一点，`Query.correlate()` 方法现在从这样的多态可选择中提取各个表，并确保所有表都是子查询的“correlate” 的一部分。假设映射文档中的 `Person/Manager/Engineer->Company` 设置，使用 with_polymorphic：

```py
sess.query(Person.name).filter(
    sess.query(Company.name)
    .filter(Company.company_id == Person.company_id)
    .correlate(Person)
    .as_scalar()
    == "Elbonia, Inc."
)
```

上述查询现在产生：

```py
SELECT  people.name  AS  people_name
FROM  people
LEFT  OUTER  JOIN  engineers  ON  people.person_id  =  engineers.person_id
LEFT  OUTER  JOIN  managers  ON  people.person_id  =  managers.person_id
WHERE  (SELECT  companies.name
FROM  companies
WHERE  companies.company_id  =  people.company_id)  =  ?
```

在修复之前，对 `correlate(Person)` 的调用会错误地尝试将 `Person`、`Engineer` 和 `Manager` 的连接作为单个单元进行关联，因此 `Person` 不会被关联：

```py
-- old, incorrect query
SELECT  people.name  AS  people_name
FROM  people
LEFT  OUTER  JOIN  engineers  ON  people.person_id  =  engineers.person_id
LEFT  OUTER  JOIN  managers  ON  people.person_id  =  managers.person_id
WHERE  (SELECT  companies.name
FROM  companies,  people
WHERE  companies.company_id  =  people.company_id)  =  ?
```

对多态映射使用相关子查询仍然存在一些未完善的地方。例如，如果`Person`被多态链接到所谓的“具体多态联合”查询，上述子查询可能无法正确引用这个子查询。在所有情况下，完全引用“多态”实体的一种方法是首先从中创建一个`aliased()`对象：

```py
# works with all SQLAlchemy versions and all types of polymorphic
# aliasing.

paliased = aliased(Person)
sess.query(paliased.name).filter(
    sess.query(Company.name)
    .filter(Company.company_id == paliased.company_id)
    .correlate(paliased)
    .as_scalar()
    == "Elbonia, Inc."
)
```

`aliased()` 构造保证了“多态可选择性”被包裹在子查询中。通过在相关子查询中明确引用它，多态形式被正确使用。

[#3662](https://www.sqlalchemy.org/trac/ticket/3662)  ### 查询的字符串化将向会话查询正确的方言

对`Query`对象调用`str()`将向`Session`查询正确的“绑定”，以便渲染将传递给数据库的 SQL。特别是，这允许引用特定于方言的 SQL 构造的`Query`可呈现，假设`Query`与适当的`Session`相关联。以前，只有当映射关联到目标`Engine`的`MetaData`才会生效。

如果底层的`MetaData`或`Session`都没有与任何绑定的`Engine`相关联，则将使用“默认”方言回退生成 SQL 字符串。

另请参见

“友好”地将核心 SQL 构造字符串化而不使用方言

[#3081](https://www.sqlalchemy.org/trac/ticket/3081)  ### 在一行中多次出现相同实体的连接急加载

已经修复了一个情况，即使实体已经从不包括属性的不同“路径”上的行加载，也将通过连接的急加载加载属性。这是一个难以复现的深层用例，但一般思路如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b_id = Column(ForeignKey("b.id"))
    c_id = Column(ForeignKey("c.id"))

    b = relationship("B")
    c = relationship("C")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    c_id = Column(ForeignKey("c.id"))

    c = relationship("C")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)
    d_id = Column(ForeignKey("d.id"))
    d = relationship("D")

class D(Base):
    __tablename__ = "d"
    id = Column(Integer, primary_key=True)

c_alias_1 = aliased(C)
c_alias_2 = aliased(C)

q = s.query(A)
q = q.join(A.b).join(c_alias_1, B.c).join(c_alias_1.d)
q = q.options(
    contains_eager(A.b).contains_eager(B.c, alias=c_alias_1).contains_eager(C.d)
)
q = q.join(c_alias_2, A.c)
q = q.options(contains_eager(A.c, alias=c_alias_2))
```

上述查询生成的 SQL 如下：

```py
SELECT
  d.id  AS  d_id,
  c_1.id  AS  c_1_id,  c_1.d_id  AS  c_1_d_id,
  b.id  AS  b_id,  b.c_id  AS  b_c_id,
  c_2.id  AS  c_2_id,  c_2.d_id  AS  c_2_d_id,
  a.id  AS  a_id,  a.b_id  AS  a_b_id,  a.c_id  AS  a_c_id
FROM
  a
  JOIN  b  ON  b.id  =  a.b_id
  JOIN  c  AS  c_1  ON  c_1.id  =  b.c_id
  JOIN  d  ON  d.id  =  c_1.d_id
  JOIN  c  AS  c_2  ON  c_2.id  =  a.c_id
```

我们可以看到`c`表被两次选择；一次在`A.b.c -> c_alias_1`的上下文中，另一次在`A.c -> c_alias_2`的上下文中。此外，我们可以看到对于单个行，`C`标识很可能对于`c_alias_1`和`c_alias_2`是**相同**的，这意味着一行中的两组列导致只向标识映射中添加一个新对象。

上述查询选项仅要求在`c_alias_1`的上下文中加载属性`C.d`，而不是`c_alias_2`。因此，我们在标识映射中得到的最终`C`对象是否具有加载的`C.d`属性取决于映射如何遍历，虽然不是完全随机，但基本上是不确定的。修复方法是，即使对于它们都引用相同标识的单个行，`c_alias_1`的加载程序在`c_alias_2`的加载程序之后处理，`C.d`元素仍将被加载。以前，加载程序不寻求修改已通过不同路径加载的实体的加载。首先到达实体的加载程序一直是不确定的，因此在某些情况下，此修复可能可检测为行为变化，而在其他情况下则不会。

修复包括对“多个路径指向一个实体”的两个变体的测试，并且修复应该希望覆盖此类其他情况。

[#3431](https://www.sqlalchemy.org/trac/ticket/3431)

### 新的 MutableList 和 MutableSet 辅助程序已添加到变异跟踪扩展中

新的辅助类`MutableList`和`MutableSet`已添加到 Mutation Tracking 扩展中，以补充现有的`MutableDict`辅助程序。

[#3297](https://www.sqlalchemy.org/trac/ticket/3297)

### 新的“raise” / “raise_on_sql”加载策略

为了帮助防止在加载一系列对象后发生不必要的延迟加载，可以将新的“lazy='raise'”和“lazy='raise_on_sql'”策略及相应的加载程序选项`raiseload()`应用于关系属性，当尝试读取非急切加载的属性时，将导致引发`InvalidRequestError`。这两个变体测试任何类型的延迟加载，包括那些只会返回 None 或从标识映射中检索的延迟加载：

```py
>>> from sqlalchemy.orm import raiseload
>>> a1 = s.query(A).options(raiseload(A.some_b)).first()
>>> a1.some_b
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'A.some_b' is not available due to lazy='raise'
```

或者仅在需要发出 SQL 时进行延迟加载：

```py
>>> from sqlalchemy.orm import raiseload
>>> a1 = s.query(A).options(raiseload(A.some_b, sql_only=True)).first()
>>> a1.some_b
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'A.bs' is not available due to lazy='raise_on_sql'
```

[#3512](https://www.sqlalchemy.org/trac/ticket/3512)  ### Mapper.order_by 已弃用

这个来自 SQLAlchemy 最早版本的旧参数是 ORM 的原始设计的一部分，其中包括`Mapper`对象作为公共查询结构。这个角色早已被`Query`对象取代，我们使用`Query.order_by()`来指示结果的排序方式，这种方式对于任何组合的 SELECT 语句、实体和 SQL 表达式都能保持一致。有许多情况下`Mapper.order_by`不能按预期工作（或者预期的结果不清楚），比如当查询组合成联合时；这些情况不受支持。

[#3394](https://www.sqlalchemy.org/trac/ticket/3394)

## 新功能和改进 - 核心

### Engines 现在作废连接，运行 BaseException 的错误处理程序

新功能在版本 1.1 中新增：这个变化是在 1.1 系列最终版本之前的一个晚期添加，不包含在 1.1 beta 版本中。

Python `BaseException` 类位于`Exception`之下，但是是系统级异常的可识别基类，例如`KeyboardInterrupt`，`SystemExit`，以及特别是`GreenletExit`异常，这些异常被 eventlet 和 gevent 使用。这个异常类现在被`Connection`的异常处理例程拦截，并包括由`ConnectionEvents.handle_error()`事件处理。在系统级异常不是`Exception`的子类的情况下，默认情况下`Connection`现在被**作废**，因为假定操作被中断，连接可能处于不可用状态。这个变化主要针对 MySQL 驱动程序，但是这个变化适用于所有的 DBAPIs。

请注意，在作废时，`Connection`使用的即时 DBAPI 连接被释放，如果在异常抛出后仍在使用`Connection`，则在下次使用时将使用新的 DBAPI 连接进行后续操作；然而，正在进行的任何事务的状态将丢失，并且必须在重新使用之前调用适当的`.rollback()`方法（如果适用）。

为了识别这种变化，当这些异常发生在连接执行工作过程中时，很容易展示一个 pymysql 或 mysqlclient / MySQL-Python 连接进入损坏状态；连接将被返回到连接池，随后的使用将失败，甚至在返回到池之前会导致调用`.rollback()`的上下文管理器中出现次要故障。这里的行为预期将减少 MySQL 错误“commands out of sync”的发生率，以及在 MySQL 驱动程序未能正确报告`cursor.description`时可能发生的`ResourceClosedError`，在绿色线程条件下运行时，绿色线程被终止，或者处理`KeyboardInterrupt`异常而不完全退出程序时。

这种行为与通常的自动失效功能不同，它不假设后端数据库本身已关闭或重新启动；它不像通常的 DBAPI 断开连接异常那样重新生成整个连接池。

这个变化应该对所有用户都是一个净改进，除了**任何当前拦截``KeyboardInterrupt``或``GreenletExit``并希望在同一事务中继续工作的应用程序**。对于不受`KeyboardInterrupt`影响的其他 DBAPIs，如 psycopg2，这样的操作在理论上是可能的。对于这些 DBAPIs，以下解决方法将禁用特定异常的连接被回收：

```py
engine = create_engine("postgresql+psycopg2://")

@event.listens_for(engine, "handle_error")
def cancel_disconnect(ctx):
    if isinstance(ctx.original_exception, KeyboardInterrupt):
        ctx.is_disconnect = False
```

[#3803](https://www.sqlalchemy.org/trac/ticket/3803)  ### CTE 支持 INSERT、UPDATE、DELETE

最广泛请求的功能之一是支持与 INSERT、UPDATE、DELETE 一起工作的通用表达式（CTE），现在已经实现。INSERT/UPDATE/DELETE 可以从位于 SQL 顶部的 WITH 子句中提取，也可以作为更大语句上下文中的 CTE 自身使用。

作为这个变化的一部分，包含 CTE 的 SELECT 插入现在将在整个语句的顶部呈现 CTE，而不像在 1.0 版本中嵌套在 SELECT 语句中。

下面是一个将 UPDATE、INSERT 和 SELECT 全部放在一个语句中的示例：

```py
>>> from sqlalchemy import table, column, select, literal, exists
>>> orders = table(
...     "orders",
...     column("region"),
...     column("amount"),
...     column("product"),
...     column("quantity"),
... )
>>>
>>> upsert = (
...     orders.update()
...     .where(orders.c.region == "Region1")
...     .values(amount=1.0, product="Product1", quantity=1)
...     .returning(*(orders.c._all_columns))
...     .cte("upsert")
... )
>>>
>>> insert = orders.insert().from_select(
...     orders.c.keys(),
...     select([literal("Region1"), literal(1.0), literal("Product1"), literal(1)]).where(
...         ~exists(upsert.select())
...     ),
... )
>>>
>>> print(insert)  # Note: formatting added for clarity
WITH  upsert  AS
(UPDATE  orders  SET  amount=:amount,  product=:product,  quantity=:quantity
  WHERE  orders.region  =  :region_1
  RETURNING  orders.region,  orders.amount,  orders.product,  orders.quantity
)
INSERT  INTO  orders  (region,  amount,  product,  quantity)
SELECT
  :param_1  AS  anon_1,  :param_2  AS  anon_2,
  :param_3  AS  anon_3,  :param_4  AS  anon_4
WHERE  NOT  (
  EXISTS  (
  SELECT  upsert.region,  upsert.amount,
  upsert.product,  upsert.quantity
  FROM  upsert)) 
```

[#2551](https://www.sqlalchemy.org/trac/ticket/2551)  ### 支持窗口函数中的 RANGE 和 ROWS 规范

新的 `over.range_` 和 `over.rows` 参数允许为窗口函数使用 RANGE 和 ROWS 表达式：

```py
>>> from sqlalchemy import func

>>> print(func.row_number().over(order_by="x", range_=(-5, 10)))
row_number()  OVER  (ORDER  BY  x  RANGE  BETWEEN  :param_1  PRECEDING  AND  :param_2  FOLLOWING)
>>> print(func.row_number().over(order_by="x", rows=(None, 0)))
row_number()  OVER  (ORDER  BY  x  ROWS  BETWEEN  UNBOUNDED  PRECEDING  AND  CURRENT  ROW)
>>> print(func.row_number().over(order_by="x", range_=(-2, None)))
row_number()  OVER  (ORDER  BY  x  RANGE  BETWEEN  :param_1  PRECEDING  AND  UNBOUNDED  FOLLOWING) 
```

`over.range_` 和 `over.rows` 被指定为 2 元组，表示特定范围的负值和正值，“CURRENT ROW” 为 0，UNBOUNDED 为 None。

另请参阅

使用窗口函数

[#3049](https://www.sqlalchemy.org/trac/ticket/3049)  ### 支持 SQL `LATERAL` 关键字

`LATERAL` 关键字目前仅被 PostgreSQL 9.3 及更高版本支持，然而由于它是 SQL 标准的一部分，对于该关键字的支持已经添加到 Core 中。`Select.lateral()` 的实现采用了特殊逻辑，不仅仅是渲染 `LATERAL` 关键字，还允许对从相同 `FROM` 子句派生的表进行相关性，例如，侧向相关性：

```py
>>> from sqlalchemy import table, column, select, true
>>> people = table("people", column("people_id"), column("age"), column("name"))
>>> books = table("books", column("book_id"), column("owner_id"))
>>> subq = (
...     select([books.c.book_id])
...     .where(books.c.owner_id == people.c.people_id)
...     .lateral("book_subq")
... )
>>> print(select([people]).select_from(people.join(subq, true())))
SELECT  people.people_id,  people.age,  people.name
FROM  people  JOIN  LATERAL  (SELECT  books.book_id  AS  book_id
FROM  books  WHERE  books.owner_id  =  people.people_id)
AS  book_subq  ON  true 
```

另请参阅

侧向相关性

`Lateral`

`Select.lateral()`

[#2857](https://www.sqlalchemy.org/trac/ticket/2857)  ### 支持 TABLESAMPLE

SQL 标准的 TABLESAMPLE 可以使用 `FromClause.tablesample()` 方法呈现，该方法返回一个类似于别名的 `TableSample` 构造。

```py
from sqlalchemy import func

selectable = people.tablesample(func.bernoulli(1), name="alias", seed=func.random())
stmt = select([selectable.c.people_id])
```

假设 `people` 有一个列 `people_id`，上述语句将渲染为：

```py
SELECT  alias.people_id  FROM
people  AS  alias  TABLESAMPLE  bernoulli(:bernoulli_1)
REPEATABLE  (random())
```

[#3718](https://www.sqlalchemy.org/trac/ticket/3718)  ### 对于复合主键列，`.autoincrement` 指令不再隐式启用

SQLAlchemy 一直具有为单列整数主键启用后端数据库的“自增”功能的便利特性；通过“自增”，我们指的是数据库列将包括数据库提供的任何 DDL 指令，以指示自增整数标识符，例如 PostgreSQL 上的 SERIAL 关键字或 MySQL 上的 AUTO_INCREMENT，并且方言还将通过使用适合该后端的技术从执行 `Table.insert()` 构造中接收这些生成的值。

发生变化的是，这个特性不再自动为*复合*主键打开；以前，表定义如下：

```py
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True),
)
```

只有因为它是主键列列表中的第一个列，才会对`'x'`列应用`autoincrement`语义。为了禁用这个，必须关闭所有列上的`autoincrement`：

```py
# old way
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True, autoincrement=False),
    Column("y", Integer, primary_key=True, autoincrement=False),
)
```

有了新的行为，组合主键将不具有自动增量语义，除非列明确标记为`autoincrement=True`：

```py
# column 'y' will be SERIAL/AUTO_INCREMENT/ auto-generating
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
)
```

为了预见一些潜在的不兼容情况，`Table.insert()`构造将对没有设置自动增量的复合主键列上缺失的主键值执行更彻底的检查；给定这样一个表：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True),
)
```

对于这个表没有提供值的 INSERT 将产生此警告：

```py
SAWarning: Column 'b.x' is marked as a member of the primary
key for table 'b', but has no Python-side or server-side default
generator indicated, nor does it indicate 'autoincrement=True',
and no explicit value is passed.  Primary key columns may not
store NULL. Note that as of SQLAlchemy 1.1, 'autoincrement=True'
must be indicated explicitly for composite (e.g. multicolumn)
primary keys if AUTO_INCREMENT/SERIAL/IDENTITY behavior is
expected for one of the columns in the primary key. CREATE TABLE
statements are impacted by this change as well on most backends.
```

对于从服务器端默认值或触发器等不太常见的东西接收主键值的列，可以使用`FetchedValue`指示存在值生成器：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True, server_default=FetchedValue()),
    Column("y", Integer, primary_key=True, server_default=FetchedValue()),
)
```

对于极少情况下实际上意图在其列中存储 NULL 的复合主键（仅在 SQLite 和 MySQL 上支持），请使用`nullable=True`指定列：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, nullable=True),
)
```

在相关更改中，`autoincrement`标志可以设置为 True，对于具有客户端或服务器端默认值的列。这通常不会对 INSERT 期间列的行为产生太大影响。

另请参见

不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

[#3216](https://www.sqlalchemy.org/trac/ticket/3216)  ### 支持 IS DISTINCT FROM 和 IS NOT DISTINCT FROM

新操作符`ColumnOperators.is_distinct_from()`和`ColumnOperators.isnot_distinct_from()`允许进行 IS DISTINCT FROM 和 IS NOT DISTINCT FROM sql 操作：

```py
>>> print(column("x").is_distinct_from(None))
x  IS  DISTINCT  FROM  NULL 
```

处理 NULL、True 和 False：

```py
>>> print(column("x").isnot_distinct_from(False))
x  IS  NOT  DISTINCT  FROM  false 
```

对于 SQLite，它没有这个运算符，“IS” / “IS NOT”会被渲染，在 SQLite 上可以处理 NULL，不像其他后端：

```py
>>> from sqlalchemy.dialects import sqlite
>>> print(column("x").is_distinct_from(None).compile(dialect=sqlite.dialect()))
x  IS  NOT  NULL 
```  ### 核心和 ORM 对 FULL OUTER JOIN 的支持

新标志`FromClause.outerjoin.full`，在核心和 ORM 级别可用，指示编译器在通常渲染`LEFT OUTER JOIN`的地方渲染`FULL OUTER JOIN`：

```py
stmt = select([t1]).select_from(t1.outerjoin(t2, full=True))
```

该标志也适用于 ORM 级别：

```py
q = session.query(MyClass).outerjoin(MyOtherClass, full=True)
```

[#1957](https://www.sqlalchemy.org/trac/ticket/1957)  ### ResultSet 列匹配增强；文本 SQL 的位置列设置

在 1.0 系列中对`ResultProxy`系统进行了一系列改进，作为[#918](https://www.sqlalchemy.org/trac/ticket/918)的一部分，重新组织内部以按位置匹配游标绑定的结果列与表/ORM 元数据，而不是通过匹配名称，用于包含有关要返回的结果行的完整信息的编译 SQL 构造。这样可以大大节省 Python 的开销，并且在将 ORM 和 Core SQL 表达式链接到结果行时具有更高的准确性。在 1.1 版本中，这种重新组织在内部进一步进行了，并且还通过最近添加的`TextClause.columns()`方法对纯文本 SQL 构造进行了提供。

#### TextAsFrom.columns() 现在按位置工作

`TextClause.columns()` 方法在 0.9 版本中添加，接受基于列的参数按位置传递；在 1.1 版本中，当所有列按位置传递时，这些列与最终结果集的关联也按位置执行。这里的关键优势在于，现在可以将文本 SQL 与 ORM 级别的结果集链接起来，而无需处理模糊或重复的列名，也无需将标签方案与 ORM 级别的标签方案匹配。现在只需要在文本 SQL 中的列顺序与传递给`TextClause.columns()`的列参数中保持一致即可：

```py
from sqlalchemy import text

stmt = text(
    "SELECT users.id, addresses.id, users.id, "
    "users.name, addresses.email_address AS email "
    "FROM users JOIN addresses ON users.id=addresses.user_id "
    "WHERE users.id = 1"
).columns(User.id, Address.id, Address.user_id, User.name, Address.email_address)

query = session.query(User).from_statement(stmt).options(contains_eager(User.addresses))
result = query.all()
```

在上面的文本 SQL 中，“id”列出现了三次，这通常会产生歧义。使用新功能，我们可以直接应用来自`User`和`Address`类的映射列，甚至可以将`Address.user_id`列链接到文本 SQL 中的`users.id`列，以供娱乐，`Query`对象将按需接收正确可定位的行，包括用于急加载。

这个改变与将列按照与文本语句中不同顺序传递给方法的代码**不兼容**。希望由于这个方法一直以来都是按照文本 SQL 语句中列的顺序传递的，即使内部没有检查这一点，因此影响会很小。这个方法只是在 0.9 版本中添加的，可能还没有广泛使用。关于如何处理使用该方法的应用程序的行为变化的详细说明，请参见 TextClause.columns() will match columns positionally, not by name, when passed positionally。

另请参见

使用文本列表达式进行选择

> 当传递位置参数时，TextClause.columns() 将按位置匹配列，而不是按名称匹配 - 向后兼容性说明

#### 对于 Core/ORM SQL 构造，基于位置的匹配比基于名称的匹配更可靠

这一变化的另一个方面是，对于已编译的 SQL 构造，匹配列的规则也已经修改为更完全地依赖于“位置”匹配。考虑到如下语句：

```py
ua = users.alias("ua")
stmt = select([users.c.user_id, ua.c.user_id])
```

上述语句将被编译为：

```py
SELECT  users.user_id,  ua.user_id  FROM  users,  users  AS  ua
```

在 1.0 版中，执行上述语句时，将会根据位置匹配与其原始编译构造相匹配，但是由于语句中包含了重复的 `'user_id'` 标签，因此“模糊列”规则仍然会介入并阻止从行中提取列。从 1.1 版开始，“模糊列”规则不会影响从列构造到 SQL 列的精确匹配，这是 ORM 用于获取列的方法：

```py
result = conn.execute(stmt)
row = result.first()

# these both match positionally, so no error
user_id = row[users.c.user_id]
ua_id = row[ua.c.user_id]

# this still raises, however
user_id = row["user_id"]
```

#### 很少出现“模糊列”错误消息

作为这一变化的一部分，错误消息 `Ambiguous column name '<name>' in result set! try 'use_labels' option on select statement.` 的措辞已经被减弱；因为当使用 ORM 或 Core 编译的 SQL 构造时，这个消息现在应该非常少见，它仅仅陈述了 `Ambiguous column name '<name>' in result set column descriptions`，并且只有在从实际上是模糊的字符串名称检索结果列时才会出现，例如在上面的示例中 `row['user_id']`。它现在还引用了来自呈现的 SQL 语句本身的实际模糊名称，而不是指示用于获取的构造本地的键或名称。

[#3501](https://www.sqlalchemy.org/trac/ticket/3501)  ### 支持 Python 的原生 `enum` 类型和兼容形式

`Enum` 类型现在可以使用任何符合 PEP-435 的枚举类型构造。在使用此模式时，输入值和返回值是实际的枚举对象，而不是字符串/整数等值：

```py
import enum
from sqlalchemy import Table, MetaData, Column, Enum, create_engine

class MyEnum(enum.Enum):
    one = 1
    two = 2
    three = 3

t = Table("data", MetaData(), Column("value", Enum(MyEnum)))

e = create_engine("sqlite://")
t.create(e)

e.execute(t.insert(), {"value": MyEnum.two})
assert e.scalar(t.select()) is MyEnum.two
```

#### `Enum.enums` 集合现在是列表，而不是元组

作为对 `Enum` 的更改的一部分，元素的 `Enum.enums` 集合现在是列表，而不是元组。这是因为列表适用于长度可变的同类项序列，其中元素的位置不具有语义上的重要性。

[#3292](https://www.sqlalchemy.org/trac/ticket/3292)  ### 核心结果行支持负整数索引

`RowProxy` 对象现在支持单个负整数索引，就像普通的 Python 序列一样，在纯 Python 和 C 扩展版本中都可以使用。之前，负值只能在切片中起作用：

【PRE60】### 现在，`Enum` 类型在 Python 中对值进行验证

为了适应 Python 本地枚举对象，以及诸如在 ARRAY 中使用非本地 ENUM 类型且 CHECK 约束不可行等边缘情况，当使用`Enum.validate_strings`标志时，`Enum`数据类型现在在 Python 中对输入值进行验证（1.1.0b2）：

```py
>>> from sqlalchemy import Table, MetaData, Column, Enum, create_engine
>>> t = Table(
...     "data",
...     MetaData(),
...     Column("value", Enum("one", "two", "three", validate_strings=True)),
... )
>>> e = create_engine("sqlite://")
>>> t.create(e)
>>> e.execute(t.insert(), {"value": "four"})
Traceback (most recent call last):
  ...
sqlalchemy.exc.StatementError: (exceptions.LookupError)
"four" is not among the defined enum values
[SQL: u'INSERT INTO data (value) VALUES (?)']
[parameters: [{'value': 'four'}]]
```

默认情况下关闭此验证，因为已经确定了用户不希望进行此类验证的用例（例如字符串比较）。对于非字符串类型，它在所有情况下都必须进行。当从数据库返回值时，结果处理方面也无条件地进行检查。

此验证是在使用非本地枚举类型时创建 CHECK 约束的现有行为之外的。现在可以使用新的`Enum.create_constraint`标志来禁用此 CHECK 约束的创建。

[#3095](https://www.sqlalchemy.org/trac/ticket/3095)  ### 非本地布尔整数值在所有情况下被强制为零/一/None

`Boolean`数据类型将 Python 布尔值强制转换为整数值，用于没有本地布尔类型的后端，例如 SQLite 和 MySQL。在这些后端上，通常设置一个 CHECK 约束，以确保数据库中的值实际上是这两个值之一。但是，MySQL 忽略 CHECK 约束，该约束是可选的，并且现有数据库可能没有此约束。已修复`Boolean`数据类型，使得已经是整数值的 Python 端值被强制转换为零或一，而不仅仅是传递原样；此外，结果的 int-to-boolean 处理器的 C 扩展版本现在使用与 Python 布尔值解释相同的值，而不是断言确切的一或零值。这现在与纯 Python 的 int-to-boolean 处理器一致，并且对数据库中已有的数据更宽容。None/NULL 的值与以前一样保留为 None/NULL。

注意

这个改变产生了一个意外的副作用，即非整数值（如字符串）的解释也发生了变化，例如字符串值`"0"`将被解释为“true”，但仅在没有本地布尔数据类型的后端上 - 在像 PostgreSQL 这样的“本地布尔”后端上，字符串值`"0"`将直接传递给驱动程序，并被解释为“false”。这是一个之前实现中没有发生的不一致性。值得注意的是，将字符串或任何其他值传递给`Boolean`数据类型之外的`None`、`True`、`False`、`1`、`0`是**不受支持**的，版本 1.2 将对此场景引发错误（或可能���是发出警告，待定）。另请参阅[#4102](https://www.sqlalchemy.org/trac/ticket/4102)。

[#3730](https://www.sqlalchemy.org/trac/ticket/3730)  ### 在日志和异常显示中现在截断了大参数和行值

SQL 语句中作为绑定参数的大值，以及结果行中存在的大值，现在在日志记录、异常报告以及`repr()`中的显示时将被截断：

```py
>>> from sqlalchemy import create_engine
>>> import random
>>> e = create_engine("sqlite://", echo="debug")
>>> some_value = "".join(chr(random.randint(52, 85)) for i in range(5000))
>>> row = e.execute("select ?", [some_value]).first()
... # (lines are wrapped for clarity) ...
2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine select ?
2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine
('E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;NM6GU
LUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7>4=4:P
GJ7HQ6 ... (4702 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=RJP
HDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
K:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine Col ('?',)
2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine
Row (u'E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;
NM6GULUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7
>4=4:PGJ7HQ ... (4703 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=
RJPHDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
K:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
>>> print(row)
(u'E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;NM6
GULUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7>4
=4:PGJ7HQ ... (4703 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;
=RJPHDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9H
MK:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
```

[#2837](https://www.sqlalchemy.org/trac/ticket/2837)  ### 核心中添加了 JSON 支持

由于 MySQL 现在除了 PostgreSQL JSON 数据类型外还有 JSON 数据类型，核心现在获得了一个`sqlalchemy.types.JSON`数据类型，这是这两种数据类型的基础。使用这种类型允许以一种跨越 PostgreSQL 和 MySQL 的方式访问“getitem”操作符和“getpath”操作符。

新数据类型还对 NULL 值的处理以及表达式处理进行了一系列改进。

另请参阅

MySQL JSON 支持

`JSON`

`JSON`

`JSON`

[#3619](https://www.sqlalchemy.org/trac/ticket/3619)

#### JSON “null” 在 ORM 操作中被插入，当不存在时被省略

`JSON`类型及其派生类型`JSON`和`JSON`有一个标志`JSON.none_as_null`，当设置为 True 时，表示 Python 值`None`应该转换为 SQL NULL 而不是 JSON NULL 值。该标志默认为 False，这意味着 Python 值`None`应该导致 JSON NULL 值。

在以下情况下，此逻辑将失败，并已得到纠正：

1\. 当列还包含默认值或 server_default 值时，对于期望持久化 JSON “null”的映射属性上的正值 `None` 仍将触发列级默认值，替换 `None` 值：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), default="some default")

# would insert "some default" instead of "'null'",
# now will insert "'null'"
obj = MyObject(json_value=None)
session.add(obj)
session.commit()
```

2\. 当列*没有*包含默认值或 server_default 值时，对于配置了 none_as_null=False 的 JSON 列的缺失值仍然会呈现为 JSON NULL，而不是回退到不插入任何值，与所有其他数据类型的行为不一致：

```py
class MyObject(Base):
    # ...

    some_other_value = Column(String(50))
    json_value = Column(JSON(none_as_null=False))

# would result in NULL for some_other_value,
# but json "'null'" for json_value.  Now results in NULL for both
# (the json_value is omitted from the INSERT)
obj = MyObject()
session.add(obj)
session.commit()
```

这是一个行为变更，对于依赖此功能将缺失值默认为 JSON null 的应用程序来说是不兼容的。这实际上确立了**缺失值与存在的 None 值有所区别**。详细信息请参见如果未提供值且未设置默认值，则 JSON 列将不插入 JSON NULL。

3\. 当使用 `Session.bulk_insert_mappings()` 方法时，`None` 在所有情况下都会被忽略：

```py
# would insert SQL NULL and/or trigger defaults,
# now inserts "'null'"
session.bulk_insert_mappings(MyObject, [{"json_value": None}])
```

`JSON` 类型现在实现了 `TypeEngine.should_evaluate_none` 标志，指示此处不应忽略 `None`；它会根据 `JSON.none_as_null` 的值自动配置。感谢 [#3061](https://www.sqlalchemy.org/trac/ticket/3061)，我们可以区分用户主动设置的值 `None` 与根本未设置的情况。

该功能同样适用于新的基础 `JSON` 类型及其派生类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)  #### 新增 JSON.NULL 常量

为了确保应用程序始终可以在值级别上完全控制一个 `JSON`、`JSON`、`JSON` 或 `JSONB` 列是否接收 SQL NULL 或 JSON `"null"` 值，已添加了常量 `JSON.NULL`，它与 `null()` 结合使用可以完全确定 SQL NULL 和 JSON `"null"` 之间的差别，无论 `JSON.none_as_null` 设置为什么：

```py
from sqlalchemy import null
from sqlalchemy.dialects.postgresql import JSON

obj1 = MyObject(json_value=null())  # will *always* insert SQL NULL
obj2 = MyObject(json_value=JSON.NULL)  # will *always* insert JSON string "null"

session.add_all([obj1, obj2])
session.commit()
```

此功能同样适用于新的基础 `JSON` 类型及其后代类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)  ### Core 添加了数组支持；新的 ANY 和 ALL 运算符

随着对 PostgreSQL `ARRAY` 类型的增强描述在 Correct SQL Types are Established from Indexed Access of ARRAY, JSON, HSTORE 中，`ARRAY` 类型的基类本身已经移动到 Core 中，成为一个新的类 `ARRAY`。

数组是 SQL 标准的一部分，还有一些面向数组的函数，如 `array_agg()` 和 `unnest()`。为了支持这些构造，不仅仅是 PostgreSQL，未来可能还包括其他支持数组的后端，如 DB2，大部分 SQL 表达式的数组逻辑现在都在 Core 中。然而，`ARRAY` 类型仍**只在 PostgreSQL 上工作**，但它可以直接使用，支持特殊的数组用例，如索引访问，以及支持 ANY 和 ALL：

```py
mytable = Table("mytable", metadata, Column("data", ARRAY(Integer, dimensions=2)))

expr = mytable.c.data[5][6]

expr = mytable.c.data[5].any(12)
```

为了支持 ANY 和 ALL，`ARRAY`类型保留了与 PostgreSQL 类型相同的`Comparator.any()`和`Comparator.all()`方法，但也将这些操作导出为新的独立运算符函数`any_()`和`all_()`。这两个函数以更传统的 SQL 方式工作，允许右侧表达式形式，如下：

```py
from sqlalchemy import any_, all_

select([mytable]).where(12 == any_(mytable.c.data[5]))
```

对于 PostgreSQL 特定的运算符“contains”、“contained_by”和“overlaps”，应继续直接使用`ARRAY`类型，该类型还提供了`ARRAY`类型的所有功能。

`any_()`和`all_()`运算符在核心级别是开放式的，但是后端数据库对它们的解释是有限的。在 PostgreSQL 后端，这两个运算符**只接受数组值**。而在 MySQL 后端，它们**只接受子查询值**。在 MySQL 中，可以使用如下表达式：

```py
from sqlalchemy import any_, all_

subq = select([mytable.c.value])
select([mytable]).where(12 > any_(subq))
```

[#3516](https://www.sqlalchemy.org/trac/ticket/3516)  ### 新功能特性，“WITHIN GROUP”，array_agg 和 set 聚合函数

使用新的`ARRAY`类型，我们还可以实现一个预定义函数，用于返回数组的`array_agg()` SQL 函数，现在可以使用`array_agg`：

```py
from sqlalchemy import func

stmt = select([func.array_agg(table.c.value)])
```

通过`aggregate_order_by`，为聚合 ORDER BY 添加了一个 PostgreSQL 元素：

```py
from sqlalchemy.dialects.postgresql import aggregate_order_by

expr = func.array_agg(aggregate_order_by(table.c.a, table.c.b.desc()))
stmt = select([expr])
```

生成：

```py
SELECT  array_agg(table1.a  ORDER  BY  table1.b  DESC)  AS  array_agg_1  FROM  table1
```

PG 方言本身还提供了一个`array_agg()`包装器，以确保`ARRAY`类型：

```py
from sqlalchemy.dialects.postgresql import array_agg

stmt = select([array_agg(table.c.value).contains("foo")])
```

另外，像`percentile_cont()`、`percentile_disc()`、`rank()`、`dense_rank()`等需要通过`WITHIN GROUP (ORDER BY <expr>)`进行排序的函数现在可以通过`FunctionElement.within_group()`修饰符来使用：

```py
from sqlalchemy import func

stmt = select(
    [
        department.c.id,
        func.percentile_cont(0.5).within_group(department.c.salary.desc()),
    ]
)
```

上述语句将生成类似于的 SQL：

```py
SELECT  department.id,  percentile_cont(0.5)
WITHIN  GROUP  (ORDER  BY  department.salary  DESC)
```

现在为这些函数提供了正确返回类型的占位符，包括`percentile_cont`、`percentile_disc`、`rank`、`dense_rank`、`mode`、`percent_rank`和`cume_dist`。

[#3132](https://www.sqlalchemy.org/trac/ticket/3132) [#1370](https://www.sqlalchemy.org/trac/ticket/1370)  ### TypeDecorator 现在自动与 Enum、Boolean、“schema” 类型配合工作

`SchemaType` 类型包括诸如`Enum`和`Boolean`等类型，除了对应数据库类型外，还会生成一个 CHECK 约束或在 PostgreSQL ENUM 的情况下生成一个新的 CREATE TYPE 语句，现在会自动与`TypeDecorator`配方一起工作。以前，对于`ENUM`的`TypeDecorator`必须像这样：

```py
# old way
class MyEnum(TypeDecorator, SchemaType):
    impl = postgresql.ENUM("one", "two", "three", name="myenum")

    def _set_table(self, table):
        self.impl._set_table(table)
```

`TypeDecorator`现在会传播这些额外的事件，因此可以像处理其他类型一样处理：

```py
# new way
class MyEnum(TypeDecorator):
    impl = postgresql.ENUM("one", "two", "three", name="myenum")
```

[#2919](https://www.sqlalchemy.org/trac/ticket/2919)  ### 多租户模式下的表对象翻译

为了支持一个应用程序使用相同一组`Table`对象在许多模式中的用例，比如每个用户一个模式，添加了一个新的执行选项`Connection.execution_options.schema_translate_map`。 使用这个映射，一组`Table`对象可以在每个连接基础上指向任何一组模式，而不是它们被分配到的`Table.schema`。 翻译适用于 DDL 和 SQL 生成，以及 ORM。

例如，如果`User`类被分配了模式“per_user”：

```py
class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True)

    __table_args__ = {"schema": "per_user"}
```

在每个请求上，`Session` 可以设置为每次引用不同的模式：

```py
session = Session()
session.connection(
    execution_options={"schema_translate_map": {"per_user": "account_one"}}
)

# will query from the ``account_one.user`` table
session.query(User).get(5)
```

另请参见

模式名称的翻译

[#2685](https://www.sqlalchemy.org/trac/ticket/2685)  ### Core SQL 构造的“友好”字符串化，没有方言

对核心 SQL 构造调用`str()`现在会在更多情况下生成一个字符串，支持各种通常不在默认 SQL 中出现的 SQL 构造，如 RETURNING、数组索引和非标准数据类型：

```py
>>> from sqlalchemy import table, column
t>>> t = table('x', column('a'), column('b'))
>>> print(t.insert().returning(t.c.a, t.c.b))
INSERT  INTO  x  (a,  b)  VALUES  (:a,  :b)  RETURNING  x.a,  x.b 
```

`str()`函数现在调用一个完全独立的方言/编译器，专门用于纯字符串打印，没有设置特定方言，因此当出现更多“只是显示给��一个字符串！”的情况时，可以将这些情况添加到这个方言/编译器中，而不会影响真实方言上的行为。

另请参见

查询的字符串化将查询会话以获取正确的方言

[#3631](https://www.sqlalchemy.org/trac/ticket/3631)  ### type_coerce 函数现在是一个持久的 SQL 元素

`type_coerce()` 函数以前会返回一个`BindParameter` 或 `Label` 类型的对象，取决于输入。 这样做的一个影响是，在使用表达式转换的情况下，比如将元素从`Column` 转换为`BindParameter`，这对于 ORM 级别的延迟加载至关重要，类型强制转换信息将不会被使用，因为它已经丢失了。

为了改进这种行为，该函数现在返回一个围绕给定表达式的持久`TypeCoerce`容器，该表达式本身保持不变；这个构造显式地由 SQL 编译器评估。这允许内部表达式的强制转换保持不变，无论语句如何修改，包括如果包含的元素被替换为不同的元素，这在 ORM 的延迟加载功能中很常见。

用于说明效果的测试用例利用了异构主连接条件与自定义类型和延迟加载。给定一个应用 CAST 作为“绑定表达式”的自定义类型：

```py
class StringAsInt(TypeDecorator):
    impl = String

    def column_expression(self, col):
        return cast(col, Integer)

    def bind_expression(self, value):
        return cast(value, String)
```

然后，一个映射，我们将一个表上的字符串“id”列与另一个表上的整数“id”列进行等价：

```py
class Person(Base):
    __tablename__ = "person"
    id = Column(StringAsInt, primary_key=True)

    pets = relationship(
        "Pets",
        primaryjoin=(
            "foreign(Pets.person_id)==cast(type_coerce(Person.id, Integer), Integer)"
        ),
    )

class Pets(Base):
    __tablename__ = "pets"
    id = Column("id", Integer, primary_key=True)
    person_id = Column("person_id", Integer)
```

在`relationship.primaryjoin`表达式中，我们使用`type_coerce()`来处理通过延迟加载传递的绑定参数作为整数，因为我们已经知道这些将来自我们的`StringAsInt`类型，该类型在 Python 中将值保持为整数。然后我们使用`cast()`，以便作为 SQL 表达式，VARCHAR“id”列将被 CAST 为整数，用于常规非转换连接，如`Query.join()`或`joinedload()`。也就是说，`.pets`的 joinedload 看起来像：

```py
SELECT  person.id  AS  person_id,  pets_1.id  AS  pets_1_id,
  pets_1.person_id  AS  pets_1_person_id
FROM  person
LEFT  OUTER  JOIN  pets  AS  pets_1
ON  pets_1.person_id  =  CAST(person.id  AS  INTEGER)
```

在连接的 ON 子句中没有 CAST，像 PostgreSQL 这样的强类型数据库将拒绝隐式比较整数并失败。

`.pets`的延迟加载情况依赖于在加载时用一个绑定参数替换`Person.id`列，该参数接收一个 Python 加载的值。这种替换特别是我们`type_coerce()`函数意图会丢失的地方。在更改之前，这种延迟加载如下所示：

```py
SELECT  pets.id  AS  pets_id,  pets.person_id  AS  pets_person_id
FROM  pets
WHERE  pets.person_id  =  CAST(CAST(%(param_1)s  AS  VARCHAR)  AS  INTEGER)
-- {'param_1': 5}
```

在上面的例子中，我们看到我们在 Python 中的值`5`首先被转换为 VARCHAR，然后再转换回 SQL 中的 INTEGER；这是一个双重转换，虽然有效，但并不是我们要求的。

随着更改，`type_coerce()`函数在列被替换为绑定参数后仍保持一个包装器，现在查询看起来像：

```py
SELECT  pets.id  AS  pets_id,  pets.person_id  AS  pets_person_id
FROM  pets
WHERE  pets.person_id  =  CAST(%(param_1)s  AS  INTEGER)
-- {'param_1': 5}
```

我们的外部 CAST，即我们的主要连接，仍然生效，但是在 `StringAsInt` 自定义类型的一部分中不必要的 CAST 被 `type_coerce()` 函数按预期移除了。

[#3531](https://www.sqlalchemy.org/trac/ticket/3531)

## 键行为变化 - ORM

### JSON 列如果没有提供值且没有默认值，则不会插入 JSON NULL

如 JSON “null” is inserted as expected with ORM operations, omitted when not present 中所述，`JSON` 如果完全缺少值，则不会渲染 JSON “null” 值。为了防止 SQL NULL，应设置默认值。给定以下映射：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), nullable=False)
```

以下刷新操作将失败并引发完整性错误：

```py
obj = MyObject()  # note no json_value
session.add(obj)
session.commit()  # will fail with integrity error
```

如果列的默认值应为 JSON NULL，则在列上设置它：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), nullable=False, default=JSON.NULL)
```

或者，确保对象上存在该值：

```py
obj = MyObject(json_value=None)
session.add(obj)
session.commit()  # will insert JSON NULL
```

请注意，为默认值设置 `None` 与完全省略它是相同的；`JSON.none_as_null` 标志不影响传递给 `Column.default` 或 `Column.server_default` 的 `None` 的值：

```py
# default=None is the same as omitting it entirely, does not apply JSON NULL
json_value = Column(JSON(none_as_null=False), nullable=False, default=None)
```

另请参阅

JSON “null” is inserted as expected with ORM operations, omitted when not present  ### DISTINCT + ORDER BY 不再冗余添加列

如下查询现在仅增广缺少于 SELECT 列表中的列，而不会出现重复：

```py
q = (
    session.query(User.id, User.name.label("name"))
    .distinct()
    .order_by(User.id, User.name, User.fullname)
)
```

产生：

```py
SELECT  DISTINCT  user.id  AS  a_id,  user.name  AS  name,
  user.fullname  AS  a_fullname
FROM  a  ORDER  BY  user.id,  user.name,  user.fullname
```

以前，它会产生：

```py
SELECT  DISTINCT  user.id  AS  a_id,  user.name  AS  name,  user.name  AS  a_name,
  user.fullname  AS  a_fullname
FROM  a  ORDER  BY  user.id,  user.name,  user.fullname
```

在上述情况下，不必要地添加了 `user.name` 列。结果不会受影响，因为额外的列无论如何都不包含在结果中，但是这些列是不必要的。

另外，当通过向 `Query.distinct()` 传递表达式来使用 PostgreSQL DISTINCT ON 格式时，上述“添加列”逻辑将被完全禁用。

当查询被捆绑成子查询以进行连接式快速加载时，“增广列列表”规则必须更加积极，以便仍然可以满足 ORDER BY，因此这种情况保持不变。

[#3641](https://www.sqlalchemy.org/trac/ticket/3641)  ### 同名的 @validates 装饰器现在会引发异常

`validates()` 装饰器仅打算对于特定属性名称的类创建一次。现在创建多于一个会引发错误，而以前则会静默选择最后定义的验证器：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    data = Column(String)

    @validates("data")
    def _validate_data_one(self):
        assert "x" in data

    @validates("data")
    def _validate_data_two(self):
        assert "y" in data

configure_mappers()
```

将引发：

```py
sqlalchemy.exc.InvalidRequestError: A validation function for mapped attribute 'data'
on mapper Mapper|A|a already exists.
```

[#3776](https://www.sqlalchemy.org/trac/ticket/3776)

## 关键行为变化 - 核心

### 当通过位置传递时，TextClause.columns()将按位置匹配列，而不是按名称匹配

`TextClause.columns()`方法的新行为，该方法本身是最近在 0.9 系列中添加的，是，当列通过位置传递而没有任何额外的关键字参数时，它们与最终结果集列位置相关联，而不再根据名称。希望这种变化的影响很小，因为该方法始终以文档形式说明传递的列与文本 SQL 语句的顺序相同，这似乎是直观的，即使内部部件不检查这一点也是如此。

使用此方法通过位置传递`Column`对象的应用程序必须确保这些`Column`对象的位置与文本 SQL 中这些列声明的位置相匹配。

例如，像下面这样的代码：

```py
stmt = text("SELECT id, name, description FROM table")

# no longer matches by name
stmt = stmt.columns(my_table.c.name, my_table.c.description, my_table.c.id)
```

将不再按预期工作；给定列的顺序现在很重要：

```py
# correct version
stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)
```

可能更有可能的是，一个像这样工作的声明：

```py
stmt = text("SELECT * FROM table")
stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)
```

现在稍微有点冒险，因为“*”规范通常会按照它们在表中出现的顺序传递列。如果表的结构因模式更改而更改，则此排序可能不再相同。因此，在使用`TextClause.columns()`时，建议在文本 SQL 中明确列出所需的列，尽管在文本 SQL 中不再需要担心列名本身。

另请参见

ResultSet 列匹配增强；文本 SQL 的位置列设置  ### 字符串 server_default 现在是文字引用

作为普通 Python 字符串传递给`Column.server_default`的服务器默认值现在通过字面引用系统传递：

```py
>>> from sqlalchemy.schema import MetaData, Table, Column, CreateTable
>>> from sqlalchemy.types import String
>>> t = Table("t", MetaData(), Column("x", String(), server_default="hi ' there"))
>>> print(CreateTable(t))
CREATE  TABLE  t  (
  x  VARCHAR  DEFAULT  'hi '' there'
) 
```

以前，引用将直接呈现。对于具有此类用例并且正在解决此问题的应用程序，此更改可能是向后不兼容的。

[#3809](https://www.sqlalchemy.org/trac/ticket/3809)  ### 一个带有 LIMIT/OFFSET/ORDER BY 的 UNION 或类似的 SELECT 现在将嵌入的 SELECT 括起来

一个问题，像其他问题一样，长期以来受 SQLite 功能不足的驱动，现在已经增强，可以在所有支持的后端上工作。我们指的是一个查询，它是 SELECT 语句的 UNION，这些语句本身包含行限制或排序功能，其中包括 LIMIT、OFFSET 和/或 ORDER BY：

```py
(SELECT  x  FROM  table1  ORDER  BY  y  LIMIT  1)  UNION
(SELECT  x  FROM  table2  ORDER  BY  y  LIMIT  2)
```

上述查询需要在每个子选择中使用括号以正确分组子结果。在 SQLAlchemy Core 中生成上述语句的方式如下：

```py
stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1)
stmt2 = select([table1.c.x]).order_by(table2.c.y).limit(2)

stmt = union(stmt1, stmt2)
```

以前，上述结构不会为内部的 SELECT 语句生成括号，导致查询在所有后端上都失败。

上述格式在 SQLite 上**仍将失败**；此外，包含 ORDER BY 但不包含 LIMIT/SELECT 的格式在 Oracle 上**仍将失败**。这不是一个不兼容的更改，因为没有括号的查询也会失败；通过修复，查询至少在所有其他数据库上能够正常工作。

在所有情况下，为了生成一个在 SQLite 上也能正常工作并且在所有情况下在 Oracle 上也能正常工作的有限 SELECT 语句的 UNION，子查询必须是一个 ALIAS 的 SELECT：

```py
stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1).alias().select()
stmt2 = select([table2.c.x]).order_by(table2.c.y).limit(2).alias().select()

stmt = union(stmt1, stmt2)
```

这个解决方法适用于所有 SQLAlchemy 版本。在 ORM 中，它看起来像：

```py
stmt1 = session.query(Model1).order_by(Model1.y).limit(1).subquery().select()
stmt2 = session.query(Model2).order_by(Model2.y).limit(1).subquery().select()

stmt = session.query(Model1).from_statement(stmt1.union(stmt2))
```

这里的行为与 SQLAlchemy 0.9 中引入的“连接重写”行为有许多相似之处，详见许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在（SELECT * FROM ..）AS ANON_1 中；然而，在这种情况下，我们选择不添加新的重写行为来适应 SQLite 的情况。现有的重写行为已经非常复杂，而带有括号的 SELECT 语句的 UNION 情况比该功能的“右嵌套连接”用例要少得多。

[#2528](https://www.sqlalchemy.org/trac/ticket/2528)

## 方言改进和更改 - PostgreSQL

### 支持 INSERT..ON CONFLICT（DO UPDATE | DO NOTHING）

从 PostgreSQL 9.5 版本开始添加到`INSERT`的`ON CONFLICT`子句现在可以使用`sqlalchemy.dialects.postgresql.dml.insert()`的 PostgreSQL 特定版本的`Insert`对象来支持。这个`Insert`子类添加了两个新方法`Insert.on_conflict_do_update()`和`Insert.on_conflict_do_nothing()`，它们实现了 PostgreSQL 9.5 在这个领域支持的完整语法：

```py
from sqlalchemy.dialects.postgresql import insert

insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

do_update_stmt = insert_stmt.on_conflict_do_update(
    index_elements=[my_table.c.id], set_=dict(data="some data to update")
)

conn.execute(do_update_stmt)
```

上述将呈现：

```py
INSERT  INTO  my_table  (id,  data)
VALUES  (:id,  :data)
ON  CONFLICT  id  DO  UPDATE  SET  data=:data_2
```

另请参阅

INSERT…ON CONFLICT（Upsert）

[#3529](https://www.sqlalchemy.org/trac/ticket/3529)  ### ARRAY 和 JSON 类型现在正确指定为“不可哈希”

如关于“不可哈希”类型的更改，影响 ORM 行的去重所述，当查询的选定实体混合了完整的 ORM 实体和列表达式时，ORM 依赖于能够为列值生成哈希函数。现在，对于 PG 的所有“数据结构”类型，包括`ARRAY`和`JSON`，`hashable=False`标志已正确设置。`JSONB`和`HSTORE`类型已经包含了这个标志。对于`ARRAY`，这取决于`ARRAY.as_tuple`标志，但是现在不再需要设置此标志以使数组值出现在组合的 ORM 行中。

另请参阅

关于“不可哈希”类型的更改，影响 ORM 行的去重

从 ARRAY、JSON、HSTORE 的索引访问中建立正确的 SQL 类型

[#3499](https://www.sqlalchemy.org/trac/ticket/3499)  ### 从 ARRAY、JSON、HSTORE 的索引访问中建立正确的 SQL 类型

对于`ARRAY`、`JSON`和`HSTORE`三者，通过索引访问返回的表达式的 SQL 类型，例如`col[someindex]`，在所有情况下都应正确。

这包括：

+   对于通过索引访问`ARRAY`的 SQL 类型将考虑配置的维度数量。具有三个维度的`ARRAY`将返回一个维度少一的`ARRAY`类型的 SQL 表达式。给定类型为`ARRAY(Integer, dimensions=3)`的列，我们现在可以执行以下表达式：

    ```py
    int_expr = col[5][6][7]  # returns an Integer expression object
    ```

    以前，对于`col[5]`的索引访问会返回一个类型为`Integer`的表达式，在这种情况下，除非我们使用`cast()`或者`type_coerce()`，否则我们无法对剩余的维度进行索引访问。

+   `JSON`和`JSONB`类型现在反映了 PostgreSQL 本身对于索引访问的处理方式。这意味着对于`JSON`或`JSONB`类型的所有索引访问都会返回一个表达式，该表达式本身*始终*是`JSON`或`JSONB`本身，除非使用了`Comparator.astext`修饰符。这意味着无论 JSON 结构的索引访问最终是引用字符串、列表、数字还是其他 JSON 结构，PostgreSQL 始终将其视为 JSON 本身，除非明确以不同方式进行转换。就像`ARRAY`类型一样，现在可以直接生成具有多层索引访问的 JSON 表达式：

    ```py
    json_expr = json_col["key1"]["attr1"][5]
    ```

+   通过对`HSTORE`进行索引访问返回的“文本”类型，以及通过与`Comparator.astext`修饰符结合使用对`JSON`和`JSONB`进行索引访问返回的“文本”类型现在是可配置的；在这两种情况下，默认为`TextClause`，但可以使用`JSON.astext_type`或`HSTORE.text_type`参数将其设置为用户定义的类型。

另请参见

JSON cast()操作现在需要显式调用.astext

[#3499](https://www.sqlalchemy.org/trac/ticket/3499) [#3487](https://www.sqlalchemy.org/trac/ticket/3487)  ### JSON cast()操作现在需要显式调用`.astext`

作为从 ARRAY、JSON、HSTORE 的索引访问中正确建立 SQL 类型的更改的一部分，`ColumnElement.cast()`操作符在`JSON`和`JSONB`上不再隐式调用`Comparator.astext`修饰符；PostgreSQL 的 JSON/JSONB 类型支持彼此之间的 CAST 操作而无需“astext”方面。

这意味着在大多数情况下，执行此操作的应用程序：

```py
expr = json_col["somekey"].cast(Integer)
```

现在需要更改为：

```py
expr = json_col["somekey"].astext.cast(Integer)
```  ### 带有 ENUM 的 ARRAY 现在将发出 ENUM 的 CREATE TYPE

类似以下的表定义现在将按预期发出 CREATE TYPE：

```py
enum = Enum(
    "manager",
    "place_admin",
    "carwash_admin",
    "parking_admin",
    "service_admin",
    "tire_admin",
    "mechanic",
    "carwasher",
    "tire_mechanic",
    name="work_place_roles",
)

class WorkPlacement(Base):
    __tablename__ = "work_placement"
    id = Column(Integer, primary_key=True)
    roles = Column(ARRAY(enum))

e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
Base.metadata.create_all(e)
```

发出：

```py
CREATE  TYPE  work_place_roles  AS  ENUM  (
  'manager',  'place_admin',  'carwash_admin',  'parking_admin',
  'service_admin',  'tire_admin',  'mechanic',  'carwasher',
  'tire_mechanic')

CREATE  TABLE  work_placement  (
  id  SERIAL  NOT  NULL,
  roles  work_place_roles[],
  PRIMARY  KEY  (id)
)
```

[#2729](https://www.sqlalchemy.org/trac/ticket/2729)

### 检查约束现在反映

PostgreSQL 方言现在支持反射 CHECK 约束，方法包括 `Inspector.get_check_constraints()` 和 `Table` 反射中的 `Table.constraints` 集合。

### 可以单独检查“普通”和“物化”视图

新参数 `PGInspector.get_view_names.include` 允许指定应返回哪些视图子类型：

```py
from sqlalchemy import inspect

insp = inspect(engine)

plain_views = insp.get_view_names(include="plain")
all_views = insp.get_view_names(include=("plain", "materialized"))
```

[#3588](https://www.sqlalchemy.org/trac/ticket/3588)

### 在 Index 中添加了 tablespace 选项

`Index` 对象现在接受参数 `postgresql_tablespace`，以指定 TABLESPACE，与 `Table` 对象接受的方式相同。

另请参阅

索引存储参数

[#3720](https://www.sqlalchemy.org/trac/ticket/3720)

### 对 PyGreSQL 的支持

[PyGreSQL](https://pypi.org/project/PyGreSQL) DBAPI 现在得到支持。

### “postgres” 模块已被移除

长期弃用的 `sqlalchemy.dialects.postgres` 模块已被移除；多年来一直发出警告，项目应该调用 `sqlalchemy.dialects.postgresql`。形式为 `postgres://` 的 Engine URLs 仍将继续运行。

### 对 FOR UPDATE SKIP LOCKED / FOR NO KEY UPDATE / FOR KEY SHARE 的支持

新参数 `GenerativeSelect.with_for_update.skip_locked` 和 `GenerativeSelect.with_for_update.key_share` 在 Core 和 ORM 中都对 PostgreSQL 后端的 “SELECT…FOR UPDATE” 或 “SELECT…FOR SHARE” 查询应用修改：

+   选择 FOR NO KEY UPDATE:

    ```py
    stmt = select([table]).with_for_update(key_share=True)
    ```

+   选择 FOR UPDATE SKIP LOCKED:

    ```py
    stmt = select([table]).with_for_update(skip_locked=True)
    ```

+   选择 FOR KEY SHARE:

    ```py
    stmt = select([table]).with_for_update(read=True, key_share=True)
    ```

## 方言改进和变更 - MySQL

### MySQL JSON 支持

MySQL 方言新增了一个类型 `JSON`，支持 MySQL 5.7 新增的 JSON 类型。该类型既提供 JSON 的持久性，又在内部使用 `JSON_EXTRACT` 函数进行基本的索引访问。通过使用 MySQL 和 PostgreSQL 共同支持的 `JSON` 数据类型，可以实现跨 MySQL 和 PostgreSQL 的可索引 JSON 列。

另请参阅

Core 添加了 JSON 支持

[#3547](https://www.sqlalchemy.org/trac/ticket/3547)  ### 添加了对 AUTOCOMMIT“隔离级别”的支持

MySQL 方言现在接受值“AUTOCOMMIT”用于`create_engine.isolation_level`和`Connection.execution_options.isolation_level`参数：

```py
connection = engine.connect()
connection = connection.execution_options(isolation_level="AUTOCOMMIT")
```

隔离级别利用了大多数 MySQL DBAPI 提供的各种“自动提交”属性。

[#3332](https://www.sqlalchemy.org/trac/ticket/3332)  ### 不再为带有 AUTO_INCREMENT 的复合主键生成隐式 KEY

MySQL 方言的行为是，如果 InnoDB 表上的复合主键中有 AUTO_INCREMENT 的列不是第一列，则如下所示：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True, autoincrement=False),
    Column("y", Integer, primary_key=True, autoincrement=True),
    mysql_engine="InnoDB",
)
```

将生成 DDL，例如以下内容：

```py
CREATE  TABLE  some_table  (
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL  AUTO_INCREMENT,
  PRIMARY  KEY  (x,  y),
  KEY  idx_autoinc_y  (y)
)ENGINE=InnoDB
```

请注意上面带有自动生成名称的“KEY”；这是很多年前为了解决 AUTO_INCREMENT 在 InnoDB 上否则会失败而添加到方言中的一个变更。

这种解决方法已被移除，并替换为更好的系统，即在主键中**首先**声明 AUTO_INCREMENT 列：

```py
CREATE  TABLE  some_table  (
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL  AUTO_INCREMENT,
  PRIMARY  KEY  (y,  x)
)ENGINE=InnoDB
```

要显式控制主键列的顺序，请显式使用`PrimaryKeyConstraint`构造（1.1.0b2）（以及根据 MySQL 需要为自动递增列添加 KEY），例如：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
    PrimaryKeyConstraint("x", "y"),
    UniqueConstraint("y"),
    mysql_engine="InnoDB",
)
```

除了变更不再为复合主键列隐式启用.autoincrement 指令外，现在更容易指定具有或不具有自动递增的复合主键；`Column.autoincrement`现在默认为值`"auto"`，不再需要`autoincrement=False`指令：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
    mysql_engine="InnoDB",
)
```

## 方言改进和变化 - SQLite

### SQLite 版本 3.7.16 取消了右嵌套连接的解决方法

在 0.9 版本中，由 Many JOIN and LEFT OUTER JOIN expressions will no longer be wrapped in (SELECT * FROM ..) AS ANON_1 引入的功能经历了大量努力，以支持在 SQLite 上重写连接以始终使用子查询以实现“right-nested-join”效果，因为 SQLite 多年来一直不支持此语法。具有讽刺意味的是，在该迁移说明中指出的 SQLite 版本 3.7.15.2 实际上是*最后*一个实际存在此限制的 SQLite 版本！下一个发布版本是 3.7.16，支持右嵌套连接被悄悄地添加。在 1.1 中，对特定 SQLite 版本和源提交进行了识别，其中进行了此更改（SQLite 的更改日志将其称为“增强查询优化器以利用传递连接约束”，而没有链接到任何问题编号、更改编号或进一步解释），并且当 DBAPI 报告版本 3.7.16 或更高版本生效时，此更改中存在的解决方法现在已经被取消。

[#3634](https://www.sqlalchemy.org/trac/ticket/3634)  ### SQLite 版本 3.10.0 解决了带点列名的问题

SQLite 方言长期以来一直存在一个问题的解决方法，即数据库驱动程序在某些 SQL 结果集中未报告正确的列名，特别是在使用 UNION 时。解决方法详见 Dotted Column Names，并要求 SQLAlchemy 假定任何带有点的列名实际上是通过这种错误行为传递的`tablename.columnname`组合，可以通过`sqlite_raw_colnames`执行选项关闭此选项。

从 SQLite 版本 3.10.0 开始，UNION 和其他查询中的 bug 已经修复；就像 Right-nested join workaround lifted for SQLite version 3.7.16 中描述的更改一样，SQLite 的更改日志只将其神秘地标识为“为 sqlite3_module.xBestIndex 方法添加了 colUsed 字段”，但是 SQLAlchemy 对这些带点列名的翻译在此版本中不再需要，因此当检测到版本 3.10.0 或更高版本时会关闭。

总的来说，截至 1.0 系列，SQLAlchemy 的`ResultProxy`在为 Core 和 ORM SQL 构造交付结果时，对结果集中的列名依赖要少得多，因此这个问题的重要性在任何情况下已经降低了。

[#3633](https://www.sqlalchemy.org/trac/ticket/3633)  ### 改进对远程模式的支持

SQLite 方言现在实现了`Inspector.get_schema_names()`方法，并且对从远程模式创建和反射的表和索引提供了改进的支持，在 SQLite 中，远程模式是通过`ATTACH`语句分配名称的数据库；以前，``CREATE INDEX`` DDL 对于绑定模式的表无法正常工作，并且`Inspector.get_foreign_keys()`方法现在将在结果中指示给定的模式。不支持跨模式外键。### 主键约束名称的反射

SQLite 后端现在利用 SQLite 的“sqlite_master”视图来从原始 DDL 中提取表的主键约束名称，就像最近的 SQLAlchemy 版本中为外键约束所实现的方式一样。

[#3629](https://www.sqlalchemy.org/trac/ticket/3629)

### 现在反映检查约束

SQLite 方言现在支持在方法`Inspector.get_check_constraints()`以及在`Table`反射中的`Table.constraints`集合中反映检查约束。

### ON DELETE 和 ON UPDATE 外键短语现在反映

`Inspector`现在将包括 SQLite 方言上的外键约束的 ON DELETE 和 ON UPDATE 短语，并且作为`Table`的一部分反映的`ForeignKeyConstraint`对象也将指示这些短语。

## 方言改进和变化 - SQL Server

### 为 SQL Server 添加了事务隔离级别支持

所有 SQL Server 方言都支持通过`create_engine.isolation_level`和`Connection.execution_options.isolation_level`参数设置事务隔离级别。支持四个标准级别以及`SNAPSHOT`：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@ms_2008", isolation_level="REPEATABLE READ"
)
```

另请参见

事务隔离级别

[#3534](https://www.sqlalchemy.org/trac/ticket/3534)  ### 字符串/可变长度类型在反射中不再明确表示“max”

当反射类型如`String`、`TextClause`等包含长度时，在 SQL Server 下，一个“无长度”的类型会将“length”参数复制为值`"max"`：

```py
>>> from sqlalchemy import create_engine, inspect
>>> engine = create_engine("mssql+pyodbc://scott:tiger@ms_2008", echo=True)
>>> engine.execute("create table s (x varchar(max), y varbinary(max))")
>>> insp = inspect(engine)
>>> for col in insp.get_columns("s"):
...     print(col["type"].__class__, col["type"].length)
<class 'sqlalchemy.sql.sqltypes.VARCHAR'> max
<class 'sqlalchemy.dialects.mssql.base.VARBINARY'> max
```

基本类型中的“length”参数预期只能是整数值或 None；None 表示无界长度，SQL Server 方言将其解释为“max”。因此，修复这些长度为 None，以便类型对象在非 SQL Server 上下文中工作：

```py
>>> for col in insp.get_columns("s"):
...     print(col["type"].__class__, col["type"].length)
<class 'sqlalchemy.sql.sqltypes.VARCHAR'> None
<class 'sqlalchemy.dialects.mssql.base.VARBINARY'> None
```

可能一直依赖于将“length”值直接与字符串“max”进行比较的应用程序应该考虑将`None`的值视为相同。

[#3504](https://www.sqlalchemy.org/trac/ticket/3504)

### 支持在主键上“非聚集”以允许在其他地方进行聚集

`UniqueConstraint`、`PrimaryKeyConstraint`、`Index`上现在默认为`None`的`mssql_clustered`标志，可以设置为 False，这将特别为主键渲染 NONCLUSTERED 关键字，允许使用不同的索引作为“clustered”。

另请参见

聚集索引支持

### legacy_schema_aliasing 标志现在设置为 False

SQLAlchemy 1.0.5 引入了`legacy_schema_aliasing`标志到 MSSQL 方言，允许关闭所谓的“传统模式”别名。这种别名试图将模式限定的表转换为别名；给定一个表如下：

```py
account_table = Table(
    "account",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("info", String(100)),
    schema="customer_schema",
)
```

传统行为模式将尝试将模式限定的表名转换为别名：

```py
>>> eng = create_engine("mssql+pymssql://mydsn", legacy_schema_aliasing=True)
>>> print(account_table.select().compile(eng))
SELECT  account_1.id,  account_1.info
FROM  customer_schema.account  AS  account_1 
```

然而，这种别名已被证明是不必要的，在许多情况下会产生不正确的 SQL。

在 SQLAlchemy 1.1 中，`legacy_schema_aliasing`标志现在默认为 False，禁用这种行为模式，允许 MSSQL 方言正常处理模式限定的表。对于可能依赖于此行为的应用程序，请将标志设置回 True。

[#3434](https://www.sqlalchemy.org/trac/ticket/3434)

## 方言改进和变化 - Oracle

### 支持 SKIP LOCKED

新参数`GenerativeSelect.with_for_update.skip_locked`在 Core 和 ORM 中都会为“SELECT…FOR UPDATE”或“SELECT.. FOR SHARE”查询生成“SKIP LOCKED”后缀。

## 介绍

本指南介绍了 SQLAlchemy 1.1 版本的新功能，并记录了影响从 SQLAlchemy 1.0 系列迁移应用程序的用户的更改。

请仔细查看关于行为变更的部分，可能会有影响向后不兼容的行为变更。

## 平台/安装程序更改

### 现在安装需要 Setuptools

多年来，SQLAlchemy 的`setup.py`文件一直支持安装 Setuptools 和不安装 Setuptools 两种操作方式；支持一种使用纯 Distutils 的“回退”模式。由于现在几乎听不到没有安装 Setuptools 的 Python 环境了，并且为了更充分地支持 Setuptools 的功能集，特别是为了支持 py.test 与其集成以及诸如“extras”之类的功能，`setup.py`现在完全依赖于 Setuptools。

另请参阅

安装指南

[#3489](https://www.sqlalchemy.org/trac/ticket/3489)

### 仅通过环境变量启用/禁用 C 扩展构建

默认情况下，在安装过程中会构建 C 扩展，只要可能。要禁用 C 扩展构建，从 SQLAlchemy 0.8.6 / 0.9.4 版本开始，可以使用`DISABLE_SQLALCHEMY_CEXT`环境变量。以前使用`--without-cextensions`参数的方法已被移除，因为它依赖于 setuptools 的已弃用功能。

另请参阅

构建 Cython 扩展

[#3500](https://www.sqlalchemy.org/trac/ticket/3500)

### 现在安装需要 Setuptools

多年来，SQLAlchemy 的`setup.py`文件一直支持安装 Setuptools 和不安装 Setuptools 两种操作方式；支持一种使用纯 Distutils 的“回退”模式。由于现在几乎听不到没有安装 Setuptools 的 Python 环境了，并且为了更充分地支持 Setuptools 的功能集，特别是为了支持 py.test 与其集成以及诸如“extras”之类的功能，`setup.py`现在完全依赖于 Setuptools。

另请参阅

安装指南

[#3489](https://www.sqlalchemy.org/trac/ticket/3489)

### 仅通过环境变量启用/禁用 C 扩展构建

默认情况下，在安装过程中会构建 C 扩展，只要可能。要禁用 C 扩展构建，从 SQLAlchemy 0.8.6 / 0.9.4 版本开始，可以使用`DISABLE_SQLALCHEMY_CEXT`环境变量。以前使用`--without-cextensions`参数的方法已被移除，因为它依赖于 setuptools 的已弃用功能。

另请参阅

构建 Cython 扩展

[#3500](https://www.sqlalchemy.org/trac/ticket/3500)

## 新功能和改进 - ORM

### 新的会话生命周期事件

`Session`长期以来一直支持事件，允许在对象状态变化方面进行一定程度的跟踪，包括`SessionEvents.before_attach()`、`SessionEvents.after_attach()`和`SessionEvents.before_flush()`。会话文档还在快速介绍对象状态中记录了主要对象状态。然而，从未有过一种系统来跟踪对象特别是当它们通过这些转换时。此外，“已删除”对象的状态历来是模糊的，因为对象在“持久”状态和“分离”状态之间的行为之间存在某种程度的不确定性。

为了清理这个领域并使会话状态转换的领域完全透明，已经添加了一系列新事件，旨在涵盖对象可能在状态之间转换的每种可能方式，并且“已删除”状态还在会话对象状态领域内被赋予了自己的官方状态名称。

#### 新状态转换事件

现在可以拦截对象的所有状态之间的转换，例如持久、挂起等，以便覆盖特定转换的会话级事件。对象进入`Session`、离开`Session`以及甚至在使用`Session.rollback()`回滚事务时发生的所有转换都明确地出现在`SessionEvents`的接口中。

总共有**十个新事件**。这些事件的摘要在新编写的文档部分对象生命周期事件中。

#### 新对象状态“已删除”已添加，已删除对象不再是“持久”状态。

在`Session`中对象的持久状态一直被记录为具有有效的数据库标识的对象；然而，在刷新中被删除的对象的情况下，它们一直处于一个灰色地带，它们并不真正“分离”于`Session`，因为它们仍然可以在回滚中恢复，但也不真正“持久”，因为它们的数据库标识已被删除，并且它们不在标识映射中。

为了解决这种灰色地带的新事件，引入了一个新的对象状态 deleted。这种状态存在于“持久”和“分离”状态之间。通过`Session.delete()`标记为删除的对象将保持在“持久”状态，直到进行刷新；在那时，它将从标识映射中移除，转移到“已删除”状态，并调用`SessionEvents.persistent_to_deleted()`钩子。如果`Session`对象的事务被回滚，对象将恢复为持久状态；将调用`SessionEvents.deleted_to_persistent()`转换。否则，如果`Session`对象的事务被提交，将调用`SessionEvents.deleted_to_detached()`转换。

此外，`InstanceState.persistent`访问器**不再返回 True**，表示对象处于新的“已删除”状态；相反，`InstanceState.deleted`访问器已经增强，可可靠地报告这种新状态。当对象被分离时，`InstanceState.deleted`返回 False，而`InstanceState.detached`访问器返回 True。要确定对象是在当前事务中还是在以前的事务中被删除，请使用`InstanceState.was_deleted`访问器。

#### 强身份映射已被弃用

新系列过渡事件的灵感之一是实现对物体的无泄漏跟踪，使其在身份映射中进出时可以保持“强引用”，从而反映物体在此映射中的移动。有了这种新功能，不再需要`Session.weak_identity_map`参数和相应的`StrongIdentityMap`对象。这个选项在 SQLAlchemy 中已经存在多年，因为“强引用”行为曾经是唯一可用的行为，许多应用程序都假定了这种行为。长期以来，强引用跟踪对象不应该是`Session`的固有工作，而应该是应用程序级别的构造，根据应用程序的需要构建；新的事件模型甚至允许复制强身份映射的确切行为。查看 Session Referencing Behavior 以获取一个新的示例，说明如何替换强身份映射。

[#2677](https://www.sqlalchemy.org/trac/ticket/2677)  ### 新的 init_scalar()事件在 ORM 级别拦截默认值

当首次访问未设置的属性时，ORM 会为非持久对象产生一个值为`None`的值：

```py
>>> obj = MyObj()
>>> obj.some_value
None
```

即使在对象持久化之前，这种在 Python 中的值与 Core 生成的默认值对应的用例也是存在的。为了适应这种用例，新增了一个名为 `AttributeEvents.init_scalar()` 的事件。在 Attribute Instrumentation 中的新示例 `active_column_defaults.py` 展示了一个样例用法，所以效果可以是：

```py
>>> obj = MyObj()
>>> obj.some_value
"my default"
```

[#1311](https://www.sqlalchemy.org/trac/ticket/1311)  ### 关于“不可哈希”类型的更改，影响 ORM 行的去重

`Query` 对象具有“去重”返回的行的良好行为，其中包含至少一个 ORM 映射实体（例如，一个完全映射的对象，而不是单个列值）。这主要是为了确保实体的处理与标识映射一起顺利进行，包括在连接的急加载中通常表示的重复实体，以及当用于过滤附加列时使用连接时。

这种去重依赖于行中元素的可哈希性。随着 PostgreSQL 的特殊类型（如 `ARRAY`、`HSTORE` 和 `JSON`）的引入，行内类型不可哈希并在这里遇到问题的经历比以前更加普遍。

实际上，自 SQLAlchemy 版本 0.8 起，已经在被标记为“不可哈希”的数据类型上包含了一个标志，然而这个标志在内置类型上并不一致。如 ARRAY 和 JSON 类型现在正确指定“不可哈希” 中描述的那样，现在这个标志已经一致地设置在了所有 PostgreSQL 的“结构”类型上。

`NullType` 类型上也设置了“不可哈希”标志，因为 `NullType` 用于指代任何未知类型的表达式。

由于大多数使用 `func` 的地方都应用了 `NullType`，因为在大多数情况下，`func` 实际上并不了解给定的函数名称，**使用 func() 通常会禁用行去重，除非显式类型化**。以下示例说明了将 `func.substr()` 应用于字符串表达式和将 `func.date()` 应用于日期时间表达式；两个示例都将由于连接的急加载而返回重复行，除非应用了显式类型化：

```py
result = (
    session.query(func.substr(A.some_thing, 0, 4), A).options(joinedload(A.bs)).all()
)

users = (
    session.query(
        func.date(User.date_created, "start of month").label("month"),
        User,
    )
    .options(joinedload(User.orders))
    .all()
)
```

为了保留去重，上述示例应指定为：

```py
result = (
    session.query(func.substr(A.some_thing, 0, 4, type_=String), A)
    .options(joinedload(A.bs))
    .all()
)

users = (
    session.query(
        func.date(User.date_created, "start of month", type_=DateTime).label("month"),
        User,
    )
    .options(joinedload(User.orders))
    .all()
)
```

此外，对所谓的“不可哈希”类型的处理与之前的版本略有不同；在内部，我们使用`id()`函数从这些结构中获取“哈希值”，就像我们对待任何普通的映射对象一样。这取代了之前将计数器应用于对象的方法。

[#3499](https://www.sqlalchemy.org/trac/ticket/3499)  ### 添加了针对传递映射类、实例作为 SQL 字面值的特定检查

现在，类型系统对在上下文中传递 SQLAlchemy“可检查”对象进行了特定检查，否则它们将被处理为字面值。任何 SQLAlchemy 内置对象，只要作为 SQL 值传递是合法的（不是已经是`ClauseElement`实例），都包括一个方法`__clause_element__()`，为该对象提供有效的 SQL 表达式。对于不提供此方法的 SQLAlchemy 对象，如映射类、映射器和映射实例，会发出更具信息性的错误消息，而不是允许 DBAPI 接收对象并稍后失败。下面举例说明，其中基于字符串的属性`User.name`与`User()`的完整实例进行比较，而不是与字符串值进行比较：

```py
>>> some_user = User()
>>> q = s.query(User).filter(User.name == some_user)
sqlalchemy.exc.ArgumentError: Object <__main__.User object at 0x103167e90> is not legal as a SQL literal value
```

当进行`User.name == some_user`比较时，异常现在是立即的。以前，类似上面的比较会产生一个 SQL 表达式，只有在解析为 DBAPI 执行调用时才会失败；映射的`User`对象最终会成为一个被 DBAPI 拒绝的绑定参数。

请注意，在上面的示例中，表达式失败是因为`User.name`是基于字符串的（例如基于列的）属性。此更改*不会*影响将多对一关系属性与对象进行比较的常规情况，这是另外处理的：

```py
>>> # Address.user refers to the User mapper, so
>>> # this is of course still OK!
>>> q = s.query(Address).filter(Address.user == some_user)
```

[#3321](https://www.sqlalchemy.org/trac/ticket/3321)  ### 新的可索引 ORM 扩展

可索引扩展是混合属性功能的一个扩展，允许构建引用“可索引”数据类型（如数组或 JSON 字段）特定元素的属性：

```py
class Person(Base):
    __tablename__ = "person"

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    name = index_property("data", "name")
```

上面，`name`属性将读取/写入 JSON 列`data`中的字段`"name"`，在初始化为空字典后：

```py
>>> person = Person(name="foobar")
>>> person.name
foobar
```

该扩展还在修改属性时触发更改事件，因此无需使用`MutableDict`来跟踪此更改。

另请参见

可索引  ### 新选项允许显式持久化 NULL 覆盖默认值

与 PostgreSQL 中添加的新 JSON-NULL 支持相关，作为 JSON “null”在 ORM 操作中如预期般插入，当不存在时被省略的一部分，基础`TypeEngine`类现在支持一个方法`TypeEngine.evaluates_none()`，允许将属性上的`None`值的正值集合持久化为 NULL，而不是从 INSERT 语句中省略列，这会导致使用列级默认值。这允许对现有对象级技术分配`null()`到属性的映射器级配置。

另请参阅

强制在具有默认值的列上使用 NULL

[#3250](https://www.sqlalchemy.org/trac/ticket/3250)  ### 进一步修复了单表继承查询问题

继续从 1.0 的在使用 from_self()，count()时更改单表继承条件，`Query`在查询针对子查询表达式时，如 exists 时，不应再不当地添加“单一继承”条件：

```py
class Widget(Base):
    __tablename__ = "widget"
    id = Column(Integer, primary_key=True)
    type = Column(String)
    data = Column(String)
    __mapper_args__ = {"polymorphic_on": type}

class FooWidget(Widget):
    __mapper_args__ = {"polymorphic_identity": "foo"}

q = session.query(FooWidget).filter(FooWidget.data == "bar").exists()

session.query(q).all()
```

生成：

```py
SELECT  EXISTS  (SELECT  1
FROM  widget
WHERE  widget.data  =  :data_1  AND  widget.type  IN  (:type_1))  AS  anon_1
```

内部的 IN 子句是合适的，以限制为 FooWidget 对象，但以前 IN 子句也会在子查询的外部生成第二次。

[#3582](https://www.sqlalchemy.org/trac/ticket/3582)  ### 当数据库取消 SAVEPOINT 时改进的 Session 状态

MySQL 的一个常见情况是，在事务中发生死锁时，SAVEPOINT 被取消。`Session`已经修改以更优雅地处理这种失败模式，使得外部的非 SAVEPOINT 事务仍然可用：

```py
s = Session()
s.begin_nested()

s.add(SomeObject())

try:
    # assume the flush fails, flush goes to rollback to the
    # savepoint and that also fails
    s.flush()
except Exception as err:
    print("Something broke, and our SAVEPOINT vanished too")

# this is the SAVEPOINT transaction, marked as
# DEACTIVE so the rollback() call succeeds
s.rollback()

# this is the outermost transaction, remains ACTIVE
# so rollback() or commit() can succeed
s.rollback()
```

这个问题是[#2696](https://www.sqlalchemy.org/trac/ticket/2696)的延续，在 Python 2 上运行时我们发出警告，即使 SAVEPOINT 异常优先。在 Python 3 上，异常被链接，因此两个失败都会被单独报告。

[#3680](https://www.sqlalchemy.org/trac/ticket/3680)  ### 修复了错误的“新实例 X 与持久实例 Y 冲突”刷新错误

`Session.rollback()` 方法负责移除在数据库中被 INSERT 的对象，例如在那个现在被回滚的事务中从挂起状态移动到持久状态的对象。进行此状态更改的对象在一个弱引用集合中被跟踪，如果一个对象从该集合中被垃圾回收，`Session` 将不再关心它（否则对于在事务中插入许多新对象的操作不会扩展）。然而，如果应用程序在回滚发生之前重新加载了同一被垃圾回收的行，那么会出现问题；如果对这个对象的强引用仍然存在于下一个事务中，那么这个对象未被插入且应该被移除的事实将丢失，并且 flush 将错误地引发错误：

```py
from sqlalchemy import Column, create_engine
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

s = Session(e)

# persist an object
s.add(A(id=1))
s.flush()

# rollback buffer loses reference to A

# load it again, rollback buffer knows nothing
# about it
a1 = s.query(A).first()

# roll back the transaction; all state is expired but the
# "a1" reference remains
s.rollback()

# previous "a1" conflicts with the new one because we aren't
# checking that it never got committed
s.add(A(id=1))
s.commit()
```

上述程序将引发：

```py
FlushError: New instance <User at 0x7f0287eca4d0> with identity key
(<class 'test.orm.test_transaction.User'>, ('u1',)) conflicts
with persistent instance <User at 0x7f02889c70d0>
```

问题在于当引发上述异常时，工作单元正在处理原始对象，假设它是一个活动行，而实际上该对象已过期，并在测试中显示它已经消失。修复现在测试这个条件，因此在 SQL 日志中我们看到：

```py
BEGIN  (implicit)

INSERT  INTO  a  (id)  VALUES  (?)
(1,)

SELECT  a.id  AS  a_id  FROM  a  LIMIT  ?  OFFSET  ?
(1,  0)

ROLLBACK

BEGIN  (implicit)

SELECT  a.id  AS  a_id  FROM  a  WHERE  a.id  =  ?
(1,)

INSERT  INTO  a  (id)  VALUES  (?)
(1,)

COMMIT
```

在上述情况下，工作单元现在会为我们即将报告为冲突的行执行一个 SELECT，看到它不存在，然后正常进行。这个 SELECT 的开销只在我们本来会在任何情况下错误地引发异常时才会发生。

[#3677](https://www.sqlalchemy.org/trac/ticket/3677)  ### 被动删除功能用于连接继承映射

现在，一个连接表继承映射现在可以允许 DELETE 操作继续进行，作为 `Session.delete()` 的结果，它只为基本表发出 DELETE，而不是子类表，允许配置的 ON DELETE CASCADE 为配置的外键发生。这是使用 `mapper.passive_deletes` 选项配置的：

```py
from sqlalchemy import Column, Integer, String, ForeignKey, create_engine
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column("id", Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "a",
        "passive_deletes": True,
    }

class B(A):
    __tablename__ = "b"
    b_table_id = Column("b_table_id", Integer, primary_key=True)
    bid = Column("bid", Integer, ForeignKey("a.id", ondelete="CASCADE"))
    data = Column("data", String)

    __mapper_args__ = {"polymorphic_identity": "b"}
```

使用上述映射，`mapper.passive_deletes` 选项在基本映射器上进行配置；它对所有具有该选项设置的映射器的非基本映射器生效。对于类型为 `B` 的对象的 DELETE 不再需要检索 `b_table_id` 的主键值（如果未加载），也不需要为表本身发出 DELETE 语句：

```py
session.delete(some_b)
session.commit()
```

将生成的 SQL 如下：

```py
DELETE  FROM  a  WHERE  a.id  =  %(id)s
-- {'id': 1}
COMMIT
```

一如既往，目标数据库必须支持启用 ON DELETE CASCADE 的外键支持。

[#2349](https://www.sqlalchemy.org/trac/ticket/2349)  ### 同名反向引用应用于具体继承子类时不会引发错误

以下映射一直是可以无问题地进行的：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b = relationship("B", foreign_keys="B.a_id", backref="a")

class A1(A):
    __tablename__ = "a1"
    id = Column(Integer, primary_key=True)
    b = relationship("B", foreign_keys="B.a1_id", backref="a1")
    __mapper_args__ = {"concrete": True}

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)

    a_id = Column(ForeignKey("a.id"))
    a1_id = Column(ForeignKey("a1.id"))
```

上述情况下，即使类 `A` 和类 `A1` 有一个名为 `b` 的关系，也不会出现冲突警告或错误，因为类 `A1` 被标记为“具体”。

然而，如果关系配置反过来，将会出现错误：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class A1(A):
    __tablename__ = "a1"
    id = Column(Integer, primary_key=True)
    __mapper_args__ = {"concrete": True}

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)

    a_id = Column(ForeignKey("a.id"))
    a1_id = Column(ForeignKey("a1.id"))

    a = relationship("A", backref="b")
    a1 = relationship("A1", backref="b")
```

修复增强了反向引用功能，以便不会发出错误，同时在映射器逻辑中增加了额外的检查，以避免替换属性时发出警告。

[#3630](https://www.sqlalchemy.org/trac/ticket/3630)  ### 不再对继承映射器上的同名关系发出警告

在继承场景中创建两个映射器时，在两者上都放置同名关系会发出警告“关系'<name>'在映射器<name>上取代了继承映射器'<name>'上的相同关系；这可能会在刷新期间引起依赖问题”。示例如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

class ASub(A):
    __tablename__ = "a_sub"
    id = Column(Integer, ForeignKey("a.id"), primary_key=True)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

这个警告可以追溯到 2007 年的 0.4 系列，基于一个自那时起已完全重写的工作单元代码版本。目前，没有关于在基类和派生类上放置同名关系的已知问题，因此警告已被取消。但是，请注意，由于警告，这种用例在现实世界中可能并不常见。虽然为这种用例添加了基本的测试支持，但可能会发现这种模式的一些新问题。

1.1.0b3 版本中新增。

[#3749](https://www.sqlalchemy.org/trac/ticket/3749)  ### 混合属性和方法现在也传播文档字符串以及.info

现在，混合方法或属性将反映原始文档字符串中存在的`__doc__`值：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    name = Column(String)

    @hybrid_property
    def some_name(self):
  """The name field"""
        return self.name
```

现在，`A.some_name.__doc__`的上述值将被尊重：

```py
>>> A.some_name.__doc__
The name field
```

然而，为了实现这一点，混合属性的机制必然变得更加复杂。以前，混合的类级访问器将是一个简单的传递，也就是说，这个测试将成功：

```py
>>> assert A.name is A.some_name
```

随着这一变化，`A.some_name`返回的表达式现在被包装在自己的`QueryableAttribute`包装器中：

```py
>>> A.some_name
<sqlalchemy.orm.attributes.hybrid_propertyProxy object at 0x7fde03888230>
```

为了确保这个包装器能够正确工作，进行了大量测试，包括对[自定义值对象](https://techspot.zzzeek.org/2011/10/21/hybrids-and-value-agnostic-types/)配方的复杂方案，但我们将继续关注用户是否出现其他退化情况。

作为这一变化的一部分，`hybrid_property.info`集合现在也从混合描述符本身传播，而不是从底层表达式传播。也就是说，访问`A.some_name.info`现在返回与`inspect(A).all_orm_descriptors['some_name'].info`相同的字典：

```py
>>> A.some_name.info["foo"] = "bar"
>>> from sqlalchemy import inspect
>>> inspect(A).all_orm_descriptors["some_name"].info
{'foo': 'bar'}
```

请注意，这个`.info`字典**独立**于可能直接代理的混合描述符的映射属性的字典；这是从 1.0 版本开始的行为变化。包装器仍将代理镜像属性的其他有用属性，如`QueryableAttribute.property`和`QueryableAttribute.class_`。

[#3653](https://www.sqlalchemy.org/trac/ticket/3653)  ### Session.merge 解决挂起冲突与持久性相同

`Session.merge()` 方法现在将跟踪给定图中对象的标识，以在发出 INSERT 之前维护主键唯一性。当遇到相同标识的重复对象时，非主键属性会在遇到对象时被**覆盖**，这本质上是非确定性的。这种行为与持久对象的处理方式相匹配，即通过主键已经位于数据库中的对象，因此这种行为更具内部一致性。

给定：

```py
u1 = User(id=7, name="x")
u1.orders = [
    Order(description="o1", address=Address(id=1, email_address="a")),
    Order(description="o2", address=Address(id=1, email_address="b")),
    Order(description="o3", address=Address(id=1, email_address="c")),
]

sess = Session()
sess.merge(u1)
```

在上面的例子中，我们将一个`User`对象与三个新的`Order`对象合并，每个对象都引用一个不同的`Address`对象，但每个对象都具有相同的主键。`Session.merge()` 的当前行为是在标识映射中查找这个`Address`对象，并将其用作目标。如果对象存在，意味着数据库已经有了主键为“1”的`Address`行，我们可以看到`Address`的`email_address`字段将被覆盖三次，在这种情况下分别为 a、b 和最后是 c。

然而，如果主键为“1”的`Address`行不存在，`Session.merge()` 将创建三个单独的`Address`实例，然后在插入时会出现主键冲突。新的行为是，这些`Address`对象的拟议主键被跟踪在一个单独的字典中，以便我们将三个拟议的`Address`对象的状态合并到一个要插入的`Address`对象上。

如果原始情况发出某种警告，表明单个合并树中存在冲突数据可能更好，然而多年来，对于持久情况，值的非确定性合并一直是行为；现在对于挂起情况也是如此。警告存在冲突值的功能仍然对于两种情况都是可行的，但会增加相当大的性能开销，因为在合并过程中每个列值都必须进行比较。

[#3601](https://www.sqlalchemy.org/trac/ticket/3601)  ### 修复涉及用户发起的外键操作的多对一对象移动问题

已修复涉及用另一个对象替换对对象的多对一引用的机制的错误。在属性操作期间，先前引用的对象的位置现在使用数据库提交的外键值，而不是当前的外键值。修复的主要效果是，当进行多对一更改时，向集合发出的反向引用事件将更准确地触发，即使在之前手动将外键属性移动到新值。假设类`Parent`和`SomeClass`的映射，其中`SomeClass.parent`指向`Parent`，`Parent.items`指向`SomeClass`对象的集合：

```py
some_object = SomeClass()
session.add(some_object)
some_object.parent_id = some_parent.id
some_object.parent = some_parent
```

在上面的例子中，我们创建了一个待处理的对象`some_object`，将其外键指向`Parent`以引用它，*然后*我们实际设置了关系。在修复错误之前，反向引用不会触发：

```py
# before the fix
assert some_object not in some_parent.items
```

现在的修复是，当我们试图定位`some_object.parent`的先前值时，我们忽略了先前手动设置的父 id，并寻找数据库提交的值。在这种情况下，它是 None，因为对象是待处理的，所以事件系统将`some_object.parent`记录为净变化：

```py
# after the fix, backref fired off for some_object.parent = some_parent
assert some_object in some_parent.items
```

尽管不鼓励操纵由关系管理的外键属性，但对于这种用例有有限的支持。为了允许加载继续进行，经常会使用`Session.enable_relationship_loading()`和`RelationshipProperty.load_on_pending`功能，这些功能会导致基于内存中未持久化的外键值发出惰性加载的关系。无论是否使用这些功能，这种行为改进现在将变得明显。

[#3708](https://www.sqlalchemy.org/trac/ticket/3708)  ### 改进 Query.correlate 方法与多态实体

在最近的 SQLAlchemy 版本中，许多形式的“多态”查询生成的 SQL 比以前更“扁平化”，其中多个表的 JOIN 不再无条件地捆绑到子查询中。为了适应这一变化，`Query.correlate()`方法现在从这样的多态可选择中提取各个表，并确保它们都是子查询的“相关”部分。假设映射文档中的`Person/Manager/Engineer->Company`设置，使用`with_polymorphic`：

```py
sess.query(Person.name).filter(
    sess.query(Company.name)
    .filter(Company.company_id == Person.company_id)
    .correlate(Person)
    .as_scalar()
    == "Elbonia, Inc."
)
```

上述查询现在会产生：

```py
SELECT  people.name  AS  people_name
FROM  people
LEFT  OUTER  JOIN  engineers  ON  people.person_id  =  engineers.person_id
LEFT  OUTER  JOIN  managers  ON  people.person_id  =  managers.person_id
WHERE  (SELECT  companies.name
FROM  companies
WHERE  companies.company_id  =  people.company_id)  =  ?
```

在修复之前，调用`correlate(Person)`会无意中尝试将`Person`、`Engineer`和`Manager`的连接作为一个单元进行关联，因此`Person`不会被关联：

```py
-- old, incorrect query
SELECT  people.name  AS  people_name
FROM  people
LEFT  OUTER  JOIN  engineers  ON  people.person_id  =  engineers.person_id
LEFT  OUTER  JOIN  managers  ON  people.person_id  =  managers.person_id
WHERE  (SELECT  companies.name
FROM  companies,  people
WHERE  companies.company_id  =  people.company_id)  =  ?
```

对多态映射使用相关子查询仍然存在一些未完善的地方。例如，如果`Person`多态链接到所谓的“具体多态联合”查询，上述子查询可能无法正确引用此子查询。在所有情况下，完全引用“多态”实体的一种方法是首先从中创建一个`aliased()`对象：

```py
# works with all SQLAlchemy versions and all types of polymorphic
# aliasing.

paliased = aliased(Person)
sess.query(paliased.name).filter(
    sess.query(Company.name)
    .filter(Company.company_id == paliased.company_id)
    .correlate(paliased)
    .as_scalar()
    == "Elbonia, Inc."
)
```

`aliased()`构造保证了“多态可选择”被包装在一个子查询中。通过在相关子查询中明确引用它，多态形式将被正确使用。

[#3662](https://www.sqlalchemy.org/trac/ticket/3662)  ### 查询的字符串化将查询会话以获取正确的方言

对`Query`对象调用`str()`将会查询`Session`以获取正确的“绑定”，以便渲染将传递给数据库的 SQL。特别是，这允许引用特定于方言的 SQL 构造的`Query`可呈现，假设`Query`与适当的`Session`相关联。以前，只有当映射关联到的`MetaData`本身绑定到目标`Engine`时，此行为才会生效。

如果底层的`MetaData`或`Session`都未与任何绑定的`Engine`相关联，则将使用“默认”方言回退来生成 SQL 字符串。

另请参见

“友好”的核心 SQL 构造的字符串化，没有方言

[#3081](https://www.sqlalchemy.org/trac/ticket/3081)  ### 在一行中多次出现相同实体的连接贪婪加载

已修复了一个情况，即通过连接贪婪加载加载属性，即使实体已经从不包括属性的不同“路径”上的行加载。这是一个难以复现的深层用例，但一般思路如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b_id = Column(ForeignKey("b.id"))
    c_id = Column(ForeignKey("c.id"))

    b = relationship("B")
    c = relationship("C")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    c_id = Column(ForeignKey("c.id"))

    c = relationship("C")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)
    d_id = Column(ForeignKey("d.id"))
    d = relationship("D")

class D(Base):
    __tablename__ = "d"
    id = Column(Integer, primary_key=True)

c_alias_1 = aliased(C)
c_alias_2 = aliased(C)

q = s.query(A)
q = q.join(A.b).join(c_alias_1, B.c).join(c_alias_1.d)
q = q.options(
    contains_eager(A.b).contains_eager(B.c, alias=c_alias_1).contains_eager(C.d)
)
q = q.join(c_alias_2, A.c)
q = q.options(contains_eager(A.c, alias=c_alias_2))
```

上述查询生成的 SQL 如下：

```py
SELECT
  d.id  AS  d_id,
  c_1.id  AS  c_1_id,  c_1.d_id  AS  c_1_d_id,
  b.id  AS  b_id,  b.c_id  AS  b_c_id,
  c_2.id  AS  c_2_id,  c_2.d_id  AS  c_2_d_id,
  a.id  AS  a_id,  a.b_id  AS  a_b_id,  a.c_id  AS  a_c_id
FROM
  a
  JOIN  b  ON  b.id  =  a.b_id
  JOIN  c  AS  c_1  ON  c_1.id  =  b.c_id
  JOIN  d  ON  d.id  =  c_1.d_id
  JOIN  c  AS  c_2  ON  c_2.id  =  a.c_id
```

我们可以看到`c`表被选择两次；一次在`A.b.c -> c_alias_1`的上下文中，另一次在`A.c -> c_alias_2`的上下文中。此外，我们可以看到对于单行来说，`C`标识很可能对于`c_alias_1`和`c_alias_2`是**相同**的，这意味着一行中的两组列导致只有一个新对象被添加到标识映射中。

上述查询选项仅要求在`c_alias_1`的上下文中加载属性`C.d`，而不是`c_alias_2`。因此，我们在标识映射中得到的最终`C`对象是否加载了`C.d`属性取决于映射如何遍历，尽管不完全是随机的，但基本上是不确定的。修复的方法是，即使对于它们都引用相同标识的单行，`c_alias_1`的加载器在`c_alias_2`的加载器之后处理，`C.d`元素仍将被加载。以前，加载器不寻求修改已通过不同路径加载的实体的加载。首先到达实体的加载器一直是不确定的，因此在某些情况下，这种修复可能会被检测为行为变化，而在其他情况下则不会。

修复包括两种“多条路径到一个实体”的情况的测试，并且修复应该希望覆盖所有其他类似情况。

[#3431](https://www.sqlalchemy.org/trac/ticket/3431)

### 新增了对变异跟踪扩展的 MutableList 和 MutableSet 助手

新的助手类`MutableList`和`MutableSet`已添加到变异跟踪扩展中，以补充现有的`MutableDict`助手。

[#3297](https://www.sqlalchemy.org/trac/ticket/3297)

### 新的“raise” / “raise_on_sql”加载策略

为了帮助防止一系列对象加载后发生不必要的延迟加载，可以将新的“lazy='raise'”和“lazy='raise_on_sql'”策略以及相应的加载器选项`raiseload()`应用于关系属性，当访问非急切加载属性进行读取时，将引发`InvalidRequestError`。这两个变体测试任何类型的延迟加载，包括那些只会返回 None 或从标识映射中检索的延迟加载：

```py
>>> from sqlalchemy.orm import raiseload
>>> a1 = s.query(A).options(raiseload(A.some_b)).first()
>>> a1.some_b
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'A.some_b' is not available due to lazy='raise'
```

或仅在 SQL 会被发出的情况下进行延迟加载：

```py
>>> from sqlalchemy.orm import raiseload
>>> a1 = s.query(A).options(raiseload(A.some_b, sql_only=True)).first()
>>> a1.some_b
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'A.bs' is not available due to lazy='raise_on_sql'
```

[#3512](https://www.sqlalchemy.org/trac/ticket/3512)  ### Mapper.order_by 已弃用

这个来自 SQLAlchemy 最早版本的旧参数是 ORM 的原始设计的一部分，其中包括`Mapper`对象作为一个公共查询结构。这个角色早已被`Query`对象取代，我们使用`Query.order_by()`来指示结果的排序方式，这种方式对于任何组合的 SELECT 语句、实体和 SQL 表达式都能一致工作。有许多情况下`Mapper.order_by`不按预期工作（或者预期的结果不清楚），比如当查询组合成联合时；这些情况不受支持。

[#3394](https://www.sqlalchemy.org/trac/ticket/3394)  ### 新的会话生命周期事件

`Session`长期以来一直支持事件，允许在某种程度上跟踪对象状态的变化，包括`SessionEvents.before_attach()`、`SessionEvents.after_attach()`和`SessionEvents.before_flush()`。会话文档还记录了主要对象状态在对象状态快速入门。然而，从来没有一种系统可以跟踪对象特别是当它们通过这些转换时。此外，“已删除”对象的状态历来是模糊的，因为对象在“持久”状态和“分离”状态之间的行为。

为了清理这个领域并使会话状态转换的领域完全透明，已经添加了一系列新的事件，旨在涵盖对象可能在状态之间转换的每种可能方式，此外，“已删除”状态已在会话对象状态领域内被赋予了自己的官方状态名称。

#### 新的状态转换事件

现在可以拦截对象的所有状态之间的转换，比如 persistent、pending 等，以便覆盖特定转换的会话级事件。对象进入`Session`、离开`Session`，甚至在使用`Session.rollback()`回滚事务时发生的所有转换都明确地出现在`SessionEvents`的接口中。

总共有**十个新事件**。这些事件的摘要在新编写的文档部分对象生命周期事件中。

#### 添加了新的对象状态“deleted”，被删除的对象不再是“persistent”。

对象在`Session`中的 persistent 状态一直被记录为具有有效的数据库标识的对象；然而，在刷新时被删除的对象的情况下，它们一直处于一个灰色地带，因为它们并没有真正“分离”出`Session`，因为它们仍然可以在回滚时恢复，但又不真正“持久”，因为它们的数据库标识已被删除，而且它们不在标识映射中。

为了解决这个新事件所带来的灰色地带，引入了一个新的对象状态 deleted。此状态存在于“持久”状态和“分离”状态之间。通过 `Session.delete()` 标记为删除的对象将保持在“持久”状态，直到进行刷新为止；在那时，它将从标识映射中移除，转移到“已删除”状态，并调用 `SessionEvents.persistent_to_deleted()` 钩子。如果 `Session` 对象的事务被回滚，则对象将被恢复为持久状态；将调用 `SessionEvents.deleted_to_persistent()` 过渡。否则，如果 `Session` 对象的事务被提交，则调用 `SessionEvents.deleted_to_detached()` 过渡。

另外，`InstanceState.persistent` 访问器**不再返回 True**以表示处于新“已删除”状态的对象；相反，`InstanceState.deleted` 访问器已经增强，可可靠地报告此新状态。当对象被分离时，`InstanceState.deleted` 返回 False，而 `InstanceState.detached` 访问器则返回 True。要确定对象是在当前事务中还是在以前的事务中被删除，请使用 `InstanceState.was_deleted` 访问器。

#### 强身份映射已不推荐使用。

新系列转换事件的灵感之一是实现对象在进出标识映射时的无泄漏跟踪，以便维护一个“强引用”，反映对象在此映射中的进出情况。有了这种新的功能，就不再需要`Session.weak_identity_map`参数和相应的`StrongIdentityMap`对象。多年来，此选项一直保留在 SQLAlchemy 中，因为“强引用”行为曾经是唯一可用的行为，并且许多应用程序都是根据这种行为编写的。长期以来，已建议不要将对象的强引用跟踪作为`Session`的内在工作，并且应该作为应用程序需要时由应用程序构建的构造体；新的事件模型甚至允许复制强标识映射的确切行为。请参阅会话引用行为以了解如何替换强标识映射的新方法。

[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

#### 新的状态转换事件

所有对象状态之间的转换，如 persistent、pending 等，现在都可以通过会话级事件进行拦截，以涵盖特定转换。对象转换为`Session`时，移出`Session`时，甚至在使用`Session.rollback()`回滚事务时发生的所有转换，在`SessionEvents`的接口中都明确存在。

总共有**十个新事件**。这些事件的摘要在新编写的文档部分对象生命周期事件中。

#### 添加了新的对象状态“已删除”，已删除的对象不再“持久”。

对象在`Session`中的持久状态一直被记录为具有有效的数据库标识符；然而，在被删除的对象的情况下，在刷新时它们一直处于一个灰色地带，它们并不真正“分离”于`Session`，因为它们仍然可以在回滚中恢复，但并不真正“持久”，因为它们的数据库标识已被删除，并且不在标识映射中。

为了解决这个灰色地带，引入了一个新的对象状态删除。这种状态存在于“持久”和“分离”状态之间。通过`Session.delete()`标记为删除的对象将保持在“持久”状态，直到刷新进行；在那时，它将从标识映射中移除，转移到“删除”状态，并调用`SessionEvents.persistent_to_deleted()`钩子。如果`Session`对象的事务被回滚，对象将恢复为持久状态；调用`SessionEvents.deleted_to_persistent()`转换。否则，如果`Session`对象的事务被提交，将调用`SessionEvents.deleted_to_detached()`转换。

此外，`InstanceState.persistent`访问器**不再返回 True**，用于处于新“已删除”状态的对象；相反，`InstanceState.deleted`访问器已经增强，可可靠地报告这种新状态。当对象被分离时，`InstanceState.deleted`返回 False，而`InstanceState.detached`访问器返回 True。要确定对象是在当前事务中删除还是在以前的事务中删除，使用`InstanceState.was_deleted`访问器。

#### 强标识映射已弃用

新系列过渡事件的灵感之一是为了实现对象在进出标识映射时的无泄漏跟踪，以便维护“强引用”，反映对象在此映射中进出的情况。有了这种新能力，就不再需要`Session.weak_identity_map`参数和相应的`StrongIdentityMap`对象。这个选项在 SQLAlchemy 中已经存在多年，因为“强引用”行为曾经是唯一可用的行为，许多应用程序都假定了这种行为。长期以来，强引用跟踪对象不是`Session`的固有工作，而是一个应用程序级别的构造，根据应用程序的需要构建；新的事件模型甚至允许复制强标识映射的确切行为。查看 Session Referencing Behavior 以获取一个新的示例，说明如何替换强标识映射。

[#2677](https://www.sqlalchemy.org/trac/ticket/2677)

### 新的`init_scalar()`事件在 ORM 级别拦截默认值

当首次访问未设置的属性时，ORM 会为非持久对象生成一个值为`None`：

```py
>>> obj = MyObj()
>>> obj.some_value
None
```

在对象持久化之前，有一个用例是使此 Python 值对应于 Core 生成的默认值。为了适应这种用例，添加了一个新的事件`AttributeEvents.init_scalar()`。在属性仪器化中的新示例`active_column_defaults.py`说明了一个示例用法，因此效果可以是：

```py
>>> obj = MyObj()
>>> obj.some_value
"my default"
```

[#1311](https://www.sqlalchemy.org/trac/ticket/1311)

### 关于“不可哈希”类型的更改，影响 ORM 行的去重

`Query`对象具有“去重”返回行的良好行为，其中包含至少一个 ORM 映射实体（例如，完全映射对象，而不是单独的列值）。这主要是为了使实体的处理与标识映射平滑配合，包括适应通常在连接的急加载中表示的重复实体，以及在使用连接以过滤其他列的目的时。

此去重依赖于行内元素的可哈希性。随着 PostgreSQL 引入特殊类型如`ARRAY`、`HSTORE`和`JSON`，行内类型不可哈希且在此遇到问题的情况比以往更普遍。

实际上，自 SQLAlchemy 版本 0.8 以来，已经在被标记为“不可哈希”的数据类型上包含了一个标志，然而这个标志在内置类型上并没有一致使用。正如 ARRAY 和 JSON 类型现在正确指定“不可哈希”所述，这个标志现在已经为所有 PostgreSQL 的“结构”类型一致设置。

“不可哈希”标志也设置在`NullType`类型上，因为`NullType`用于引用任何未知类型的表达式。

由于`NullType`应用于大多数`func`的用法，因为`func`实际上并不知道在大多数情况下给定的函数名称，**使用 func()通常会禁用行去重，除非应用了显式类型**。以下示例说明了将`func.substr()`应用于字符串表达式，以及将`func.date()`应用于日期时间表达式；这两个示例将由于连接的急加载而返回重复行，除非应用了显式类型：

```py
result = (
    session.query(func.substr(A.some_thing, 0, 4), A).options(joinedload(A.bs)).all()
)

users = (
    session.query(
        func.date(User.date_created, "start of month").label("month"),
        User,
    )
    .options(joinedload(User.orders))
    .all()
)
```

为了保留去重，上面的示例应指定为：

```py
result = (
    session.query(func.substr(A.some_thing, 0, 4, type_=String), A)
    .options(joinedload(A.bs))
    .all()
)

users = (
    session.query(
        func.date(User.date_created, "start of month", type_=DateTime).label("month"),
        User,
    )
    .options(joinedload(User.orders))
    .all()
)
```

另外，对于所谓的“不可哈希”类型的处理略有不同，与之前的发布版本有些不同；在内部，我们使用 `id()` 函数从这些结构中获取“哈希值”，就像我们对任何普通映射对象一样。这取代了以前的方法，该方法对对象应用了一个计数器。

[#3499](https://www.sqlalchemy.org/trac/ticket/3499)

### 添加了用于传递映射类、实例作为 SQL 文字的特定检查

现在，类型系统对于在否则会被处理为文字值的上下文中传递 SQLAlchemy “可检查”对象具有特定检查。任何可以作为 SQL 值传递的 SQLAlchemy 内置对象（它不是已经是 `ClauseElement` 实例的对象）都包含一个方法 `__clause_element__()`，该方法为该对象提供一个有效的 SQL 表达式。对于不提供此功能的 SQLAlchemy 对象，例如映射类、映射器和映射实例，将发出更详细的错误消息，而不是允许 DBAPI 接收对象并稍后失败。下面举例说明了一个示例，其中将字符串属性 `User.name` 与 `User()` 的完整实例进行比较，而不是与字符串值进行比较：

```py
>>> some_user = User()
>>> q = s.query(User).filter(User.name == some_user)
sqlalchemy.exc.ArgumentError: Object <__main__.User object at 0x103167e90> is not legal as a SQL literal value
```

现在，当比较`User.name == some_user`时，异常立即发生。以前，类似上述的比较会产生一个 SQL 表达式，只有在解析为 DBAPI 执行调用时才会失败；映射的 `User` 对象最终会变成一个被 DBAPI 拒绝的绑定参数。

请注意，在上面的示例中，表达式失败是因为 `User.name` 是基于字符串的（例如列导向）属性。此更改*不会*影响通常情况下将多对一关系属性与对象进行比较的情况，这是单独处理的：

```py
>>> # Address.user refers to the User mapper, so
>>> # this is of course still OK!
>>> q = s.query(Address).filter(Address.user == some_user)
```

[#3321](https://www.sqlalchemy.org/trac/ticket/3321)

### 新的可索引 ORM 扩展

可索引 扩展是对混合属性功能的扩展，它允许构建引用特定元素的属性，这些元素属于“可索引”数据类型，例如数组或 JSON 字段：

```py
class Person(Base):
    __tablename__ = "person"

    id = Column(Integer, primary_key=True)
    data = Column(JSON)

    name = index_property("data", "name")
```

在上面的示例中，`name` 属性将从 JSON 列 `data` 读取/写入字段 `"name"`，在将其初始化为空字典之后：

```py
>>> person = Person(name="foobar")
>>> person.name
foobar
```

该扩展还在修改属性时触发更改事件，因此无需使用 `MutableDict` 来跟踪此更改。

另请参阅

可索引

### 新选项允许明确持久化 NULL 覆盖默认值

与 PostgreSQL 中新增的 JSON-NULL 支持相关，作为 JSON “null” is inserted as expected with ORM operations, omitted when not present 的一部分，基础 `TypeEngine` 类现在支持一个方法 `TypeEngine.evaluates_none()`，允许将属性上的 `None` 值设置为 NULL，而不是在 INSERT 语句中省略该列，这会导致使用列级默认值。这允许在映射器级别配置现有的对象级别将 `null()` 分配给属性的技术。

另请参见

强制在具有默认值的列上使用 NULL

[#3250](https://www.sqlalchemy.org/trac/ticket/3250)

### 进一步修复单表继承查询

继续从 1.0 的 Change to single-table-inheritance criteria when using from_self(), count() 中，`Query` 在查询针对子查询表达式（如 exists）时不应再不适当地添加“单一继承”条件：

```py
class Widget(Base):
    __tablename__ = "widget"
    id = Column(Integer, primary_key=True)
    type = Column(String)
    data = Column(String)
    __mapper_args__ = {"polymorphic_on": type}

class FooWidget(Widget):
    __mapper_args__ = {"polymorphic_identity": "foo"}

q = session.query(FooWidget).filter(FooWidget.data == "bar").exists()

session.query(q).all()
```

产生：

```py
SELECT  EXISTS  (SELECT  1
FROM  widget
WHERE  widget.data  =  :data_1  AND  widget.type  IN  (:type_1))  AS  anon_1
```

内部的 IN 子句是适当的，以限制为 FooWidget 对象，但以前 IN 子句也会在子查询的外部生成第二次。

[#3582](https://www.sqlalchemy.org/trac/ticket/3582)

### 当数据库取消 SAVEPOINT 时改进的 Session 状态

MySQL 的一个常见情况是在事务中发生死锁时取消 SAVEPOINT。`Session` 已经修改以更优雅地处理这种失败模式，使得外部的非 SAVEPOINT 事务仍然可用：

```py
s = Session()
s.begin_nested()

s.add(SomeObject())

try:
    # assume the flush fails, flush goes to rollback to the
    # savepoint and that also fails
    s.flush()
except Exception as err:
    print("Something broke, and our SAVEPOINT vanished too")

# this is the SAVEPOINT transaction, marked as
# DEACTIVE so the rollback() call succeeds
s.rollback()

# this is the outermost transaction, remains ACTIVE
# so rollback() or commit() can succeed
s.rollback()
```

这个问题是 [#2696](https://www.sqlalchemy.org/trac/ticket/2696) 的延续，在 Python 2 上运行时我们发出警告，以便可以看到原始错误，即使 SAVEPOINT 异常优先。在 Python 3 上，异常被链接在一起，因此两个失败都会被单独报告。

[#3680](https://www.sqlalchemy.org/trac/ticket/3680)

### 修复了错误的“新实例 X 与持久实例 Y 冲突”刷新错误

`Session.rollback()` 方法负责移除在数据库中被插入的对象，例如从挂起状态移动到持久状态的对象，在被回滚的事务中。使得这种状态改变的对象被跟踪在一个弱引用集合中，如果一个对象从该集合中被垃圾回收，`Session` 就不再关心它（否则对于在事务中插入许多新对象的操作来说，这种方式不会扩展）。然而，如果在回滚发生之前，应用程序重新加载了同一个被垃圾回收的行；如果对这个对象仍然存在强引用到下一个事务中，那么这个对象没有被插入并且应该被移除的事实将会丢失，刷新将会错误地引发一个错误：

```py
from sqlalchemy import Column, create_engine
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

s = Session(e)

# persist an object
s.add(A(id=1))
s.flush()

# rollback buffer loses reference to A

# load it again, rollback buffer knows nothing
# about it
a1 = s.query(A).first()

# roll back the transaction; all state is expired but the
# "a1" reference remains
s.rollback()

# previous "a1" conflicts with the new one because we aren't
# checking that it never got committed
s.add(A(id=1))
s.commit()
```

上述程序将引发：

```py
FlushError: New instance <User at 0x7f0287eca4d0> with identity key
(<class 'test.orm.test_transaction.User'>, ('u1',)) conflicts
with persistent instance <User at 0x7f02889c70d0>
```

这个 bug 是当上述异常被引发时，工作单元正在处理原始对象，假设它是一个活动行，而实际上该对象已过期，并在测试中显示它已经消失。修复现在测试这个条件，所以在 SQL 日志中我们看到：

```py
BEGIN  (implicit)

INSERT  INTO  a  (id)  VALUES  (?)
(1,)

SELECT  a.id  AS  a_id  FROM  a  LIMIT  ?  OFFSET  ?
(1,  0)

ROLLBACK

BEGIN  (implicit)

SELECT  a.id  AS  a_id  FROM  a  WHERE  a.id  =  ?
(1,)

INSERT  INTO  a  (id)  VALUES  (?)
(1,)

COMMIT
```

上面，工作单元现在对我们即将报告���冲突的行进行 SELECT，看到它不存在，然后正常进行。这个 SELECT 的开销只在我们本来会错误地引发异常的情况下才会发生。 

[#3677](https://www.sqlalchemy.org/trac/ticket/3677)

### 联接继承映射的被动删除功能

一个联接表继承映射现在可能允许一个 DELETE 操作继续进行，作为 `Session.delete()` 的结果，它只对基表发出 DELETE，而不是子类表，允许配置的 ON DELETE CASCADE 为配置的外键发生。这是使用 `mapper.passive_deletes` 选项进行配置的：

```py
from sqlalchemy import Column, Integer, String, ForeignKey, create_engine
from sqlalchemy.orm import Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column("id", Integer, primary_key=True)
    type = Column(String)

    __mapper_args__ = {
        "polymorphic_on": type,
        "polymorphic_identity": "a",
        "passive_deletes": True,
    }

class B(A):
    __tablename__ = "b"
    b_table_id = Column("b_table_id", Integer, primary_key=True)
    bid = Column("bid", Integer, ForeignKey("a.id", ondelete="CASCADE"))
    data = Column("data", String)

    __mapper_args__ = {"polymorphic_identity": "b"}
```

在上述映射中，`mapper.passive_deletes` 选项被配置在基本映射器上；它对于所有具有该选项设置的映射器的非基本映射器生效。对于类型为 `B` 的对象的 DELETE 不再需要检索 `b_table_id` 的主键值（如果未加载），也不需要为表本身发出 DELETE 语句：

```py
session.delete(some_b)
session.commit()
```

将会发出 SQL 语句：

```py
DELETE  FROM  a  WHERE  a.id  =  %(id)s
-- {'id': 1}
COMMIT
```

一如既往，目标数据库必须支持启用 ON DELETE CASCADE 的外键支持。

[#2349](https://www.sqlalchemy.org/trac/ticket/2349)

### 同名反向引用应用于具体继承子类时不会引发错误

以下映射一直以来都是可能的而没有问题：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b = relationship("B", foreign_keys="B.a_id", backref="a")

class A1(A):
    __tablename__ = "a1"
    id = Column(Integer, primary_key=True)
    b = relationship("B", foreign_keys="B.a1_id", backref="a1")
    __mapper_args__ = {"concrete": True}

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)

    a_id = Column(ForeignKey("a.id"))
    a1_id = Column(ForeignKey("a1.id"))
```

上面，即使类 `A` 和类 `A1` 有一个名为 `b` 的关系，也不会发生冲突警告或错误，因为类 `A1` 被标记为“具体”。

然而，如果关系被配置为另一种方式，将会发生错误：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

class A1(A):
    __tablename__ = "a1"
    id = Column(Integer, primary_key=True)
    __mapper_args__ = {"concrete": True}

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)

    a_id = Column(ForeignKey("a.id"))
    a1_id = Column(ForeignKey("a1.id"))

    a = relationship("A", backref="b")
    a1 = relationship("A1", backref="b")
```

此修复增强了 backref 特性，以便不发出错误，以及在映射器逻辑中进一步检查是否应该绕过替换属性的警告。

[#3630](https://www.sqlalchemy.org/trac/ticket/3630)

### 在继承映射器上具有相同名称的关系不再发出警告

在继承情景中创建两个映射器时，在两者上放置具有相同名称的关系将发出警告：“关系'<name>'在映射器<name>上取代了继承的映射器'<name>'上的相同关系；这可能会在刷新时引起依赖问题”。示例如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B")

class ASub(A):
    __tablename__ = "a_sub"
    id = Column(Integer, ForeignKey("a.id"), primary_key=True)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
```

这个警告可以追溯到 2007 年的 0.4 系列，基于一个自那时完全重写的工作单元代码版本。目前，将同名关系放置在基类和派生类上没有已知问题，因此警告已解除。然而，请注意，由于警告，这种用例在现实世界中可能并不普遍。尽管为此用例添加了基本的测试支持，但可能会发现这种模式的一些新问题。

版本 1.1.0b3 中的新功能。

[#3749](https://www.sqlalchemy.org/trac/ticket/3749)

### 现在混合属性和方法也会传播文档字符串以及.info

现在混合方法或属性将反映原始文档字符串中存在的`__doc__`值：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    name = Column(String)

    @hybrid_property
    def some_name(self):
  """The name field"""
        return self.name
```

上述值`A.some_name.__doc__`现在被尊重：

```py
>>> A.some_name.__doc__
The name field
```

然而，为了实现这一点，混合属性的机制必然变得更加复杂。以前，混合的类级访问器是一个简单的透传，也就是说，这个测试会成功：

```py
>>> assert A.name is A.some_name
```

随着变化，由`A.some_name`返回的表达式现在被包装在其自己的`QueryableAttribute`包装器中：

```py
>>> A.some_name
<sqlalchemy.orm.attributes.hybrid_propertyProxy object at 0x7fde03888230>
```

已经进行了大量测试，以确保此包装器能够正常工作，包括对[自定义值对象](https://techspot.zzzeek.org/2011/10/21/hybrids-and-value-agnostic-types/)配方的复杂方案，但我们将继续关注用户是否出现其他退化。

作为这个改变的一部分，`hybrid_property.info`集合现在也从混合描述符本身传播，而不是从底层表达式传播。也就是说，现在访问`A.some_name.info`会返回与`inspect(A).all_orm_descriptors['some_name'].info`相同的字典：

```py
>>> A.some_name.info["foo"] = "bar"
>>> from sqlalchemy import inspect
>>> inspect(A).all_orm_descriptors["some_name"].info
{'foo': 'bar'}
```

请注意，此`.info`字典**与**由混合描述符可能直接代理的映射属性的字典**不同**；这是从 1.0 开始的行为变更。包装器仍将代理来自镜像属性的其他有用属性，例如`QueryableAttribute.property`和`QueryableAttribute.class_`。

[#3653](https://www.sqlalchemy.org/trac/ticket/3653)

### Session.merge 解决未解决的冲突与持久性相同

现在，`Session.merge()`方法将跟踪给定图中对象的标识，以维护主键的唯一性，然后再发出 INSERT。当遇到相同标识的重复对象时，非主键属性会被**覆盖**，因为对象被遇到时，这基本上是非确定性的。这种行为与持久对象的处理方式相匹配，即通过主键已经位于数据库中的对象，因此这种行为更加内部一致。

给定：

```py
u1 = User(id=7, name="x")
u1.orders = [
    Order(description="o1", address=Address(id=1, email_address="a")),
    Order(description="o2", address=Address(id=1, email_address="b")),
    Order(description="o3", address=Address(id=1, email_address="c")),
]

sess = Session()
sess.merge(u1)
```

在上面的例子中，我们将一个`User`对象与三个新的`Order`对象合并，每个对象都引用一个不同的`Address`对象，但每个对象都具有相同的主键。`Session.merge()`的当前行为是在标识映射中查找这个`Address`对象，并将其用作目标。如果对象存在，意味着数据库已经有一个主键为“1”的`Address`行，我们可以看到`Address`的`email_address`字段将在这种情况下被三次覆盖，分别为值 a、b 和最后的 c。

然而，如果主键“1”对应的`Address`行不存在，`Session.merge()`将会创建三个独立的`Address`实例，然后在插入时会出现主键冲突。新的行为是，这些`Address`对象的拟议主键被跟踪在一个单独的字典中，这样我们就可以将三个拟议的`Address`对象的状态合并到一个要插入的`Address`对象上。

如果原始情况下发出某种警告，指出在单个合并树中存在冲突数据可能更好，然而，多年来，对于持久情况，非确定性值的合并一直是行为，现在也适用于挂起情况。警告存在冲突值的功能仍然适用于两种情况，但会增加相当大的性能开销，因为在合并过程中必须比较每个列值。

[#3601](https://www.sqlalchemy.org/trac/ticket/3601)

### 修复涉及用户发起的外键操作的多对一对象移动

修复了涉及将对对象的多对一引用替换为另一个对象的机制的错误。在属性操作期间，先前引用的对象的位置现在使用数据库提交的外键值，而不是当前的外键值。修复的主要效果是，当进行多对一更改时，即使在之前手动将外键属性移动到新值之前，也将更准确地触发对集合的 backref 事件。假设类`Parent`和`SomeClass`的映射，其中`SomeClass.parent`指向`Parent`，而`Parent.items`指向`SomeClass`对象的集合：

```py
some_object = SomeClass()
session.add(some_object)
some_object.parent_id = some_parent.id
some_object.parent = some_parent
```

上面，我们创建了一个待处理的对象`some_object`，并将其外键指向`Parent`以引用它，*然后*我们实际设置了关系。在修复错误之前，backref 不会被触发：

```py
# before the fix
assert some_object not in some_parent.items
```

现在修复的问题是，当我们试图定位`some_object.parent`的先前值时，我们会忽略手动设置的父 id，并寻找数据库提交的值。在这种情况下，它是 None，因为对象是待处理的，所以事件系统将`some_object.parent`记录为净变化：

```py
# after the fix, backref fired off for some_object.parent = some_parent
assert some_object in some_parent.items
```

虽然不鼓励操纵由关系管理的外键属性，但对于这种用例有一定的支持。为了允许加载继续进行，经常会使用`Session.enable_relationship_loading()`和`RelationshipProperty.load_on_pending`功能，这会导致基于内存中尚未持久化的外键值的惰性加载关系。无论是否使用了这些功能，这种行为改进现在都会显而易见。

[#3708](https://www.sqlalchemy.org/trac/ticket/3708)

### 改进查询中的 Query.correlate 方法与多态实体

在最近的 SQLAlchemy 版本中，许多形式的“多态”查询生成的 SQL 比以前更“扁平化”，其中多个表的 JOIN 不再无条件地捆绑到子查询中。为了适应这一点，`Query.correlate()`方法现在会从这样一个多态可选择的地方提取各个表，并确保它们都是子查询的“相关部分”。假设从映射文档中的`Person/Manager/Engineer->Company`设置开始，使用`with_polymorphic`：

```py
sess.query(Person.name).filter(
    sess.query(Company.name)
    .filter(Company.company_id == Person.company_id)
    .correlate(Person)
    .as_scalar()
    == "Elbonia, Inc."
)
```

上述查询现在会产生：

```py
SELECT  people.name  AS  people_name
FROM  people
LEFT  OUTER  JOIN  engineers  ON  people.person_id  =  engineers.person_id
LEFT  OUTER  JOIN  managers  ON  people.person_id  =  managers.person_id
WHERE  (SELECT  companies.name
FROM  companies
WHERE  companies.company_id  =  people.company_id)  =  ?
```

在修复之前，调用`correlate(Person)`会错误地尝试将`Person`，`Engineer`和`Manager`的连接作为一个单元进行关联，因此`Person`不会被关联：

```py
-- old, incorrect query
SELECT  people.name  AS  people_name
FROM  people
LEFT  OUTER  JOIN  engineers  ON  people.person_id  =  engineers.person_id
LEFT  OUTER  JOIN  managers  ON  people.person_id  =  managers.person_id
WHERE  (SELECT  companies.name
FROM  companies,  people
WHERE  companies.company_id  =  people.company_id)  =  ?
```

对多态映射使用相关子查询仍然存在一些未完善的地方。例如，如果`Person`多态链接到所谓的“具体多态联合”查询，上述子查询可能无法正确引用此子查询。在所有情况下，完全引用“多态”实体的一种方法是首先从中创建一个`aliased()`对象：

```py
# works with all SQLAlchemy versions and all types of polymorphic
# aliasing.

paliased = aliased(Person)
sess.query(paliased.name).filter(
    sess.query(Company.name)
    .filter(Company.company_id == paliased.company_id)
    .correlate(paliased)
    .as_scalar()
    == "Elbonia, Inc."
)
```

`aliased()` 构造保证了“多态可选择性”被包裹在子查询中。通过在相关子查询中明确引用它，多态形式被正确使用。

[#3662](https://www.sqlalchemy.org/trac/ticket/3662)

### 查询的字符串化将向会话咨询正确的方言

对`Query`对象调用`str()`将向`Session`咨询要使用的正确“绑定”，以便呈现将传递给数据库的 SQL。特别是，这允许引用特定于方言的 SQL 结构的`Query`可呈现，假设`Query`与适当的`Session`相关联。以前，只有当映射关联到的`MetaData`本身绑定到目标`Engine`时，此行为才会生效。

如果底层的`MetaData`或`Session`都没有与任何绑定的`Engine`相关联，则会使用“默认”方言回退以生成 SQL 字符串。

另请参见

没有方言的核心 SQL 结构的“友好”字符串化

[#3081](https://www.sqlalchemy.org/trac/ticket/3081)

### 在一行中多次出现相同实体的连接式预加载

已对通过连接式预加载加载属性的情况进行了修复，即使实体已经从不包括属性的不同“路径”上的行加载。这是一个难以复现的深层用例，但一般思路如下：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    b_id = Column(ForeignKey("b.id"))
    c_id = Column(ForeignKey("c.id"))

    b = relationship("B")
    c = relationship("C")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    c_id = Column(ForeignKey("c.id"))

    c = relationship("C")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)
    d_id = Column(ForeignKey("d.id"))
    d = relationship("D")

class D(Base):
    __tablename__ = "d"
    id = Column(Integer, primary_key=True)

c_alias_1 = aliased(C)
c_alias_2 = aliased(C)

q = s.query(A)
q = q.join(A.b).join(c_alias_1, B.c).join(c_alias_1.d)
q = q.options(
    contains_eager(A.b).contains_eager(B.c, alias=c_alias_1).contains_eager(C.d)
)
q = q.join(c_alias_2, A.c)
q = q.options(contains_eager(A.c, alias=c_alias_2))
```

上述查询生成的 SQL 如下：

```py
SELECT
  d.id  AS  d_id,
  c_1.id  AS  c_1_id,  c_1.d_id  AS  c_1_d_id,
  b.id  AS  b_id,  b.c_id  AS  b_c_id,
  c_2.id  AS  c_2_id,  c_2.d_id  AS  c_2_d_id,
  a.id  AS  a_id,  a.b_id  AS  a_b_id,  a.c_id  AS  a_c_id
FROM
  a
  JOIN  b  ON  b.id  =  a.b_id
  JOIN  c  AS  c_1  ON  c_1.id  =  b.c_id
  JOIN  d  ON  d.id  =  c_1.d_id
  JOIN  c  AS  c_2  ON  c_2.id  =  a.c_id
```

我们可以看到 `c` 表被选中了两次；一次是在 `A.b.c -> c_alias_1` 的上下文中，另一次是在 `A.c -> c_alias_2` 的上下文中。此外，我们可以看到对于单个行来说，`C` 的标识很可能对于 `c_alias_1` 和 `c_alias_2` 是**相同**的，这意味着一行中的两组列只会导致将一个新对象添加到标识映射中。

上面的查询选项只要求在 `c_alias_1` 的上下文中加载属性 `C.d`，而不是在 `c_alias_2` 中加载。因此，我们在标识映射中得到的最终 `C` 对象是否加载了 `C.d` 属性取决于映射是如何遍历的，虽然不是完全随机的，但基本上是不确定的。修复方法是，即使对于已通过不同路径加载的实体，加载器也会对 `C.d` 元素进行加载。先到达实体的加载器一直是不确定的，所以这个修复在某些情况下可能会被检测到是一种行为上的改变，而在其他情况下则不会。

修复包括两种“多个路径指向一个实体”的情况的测试，并且修复希望能够涵盖此类其他场景的问题。

[#3431](https://www.sqlalchemy.org/trac/ticket/3431)

### 新增了 MutableList 和 MutableSet 辅助类到变化跟踪扩展

变化跟踪扩展中新增了新的辅助类`MutableList`和`MutableSet`，以补充现有的`MutableDict`助手。

[#3297](https://www.sqlalchemy.org/trac/ticket/3297)

### 新的“raise”/“raise_on_sql”加载策略

为了帮助防止在加载一系列对象后发生不需要的惰性加载，可以将新的“lazy=’raise’”和“lazy=’raise_on_sql’”策略及相应的加载器选项`raiseload()`应用于关系属性，这将导致在读取非急切加载的属性时引发`InvalidRequestError`。两种变体测试任何类型的惰性加载，包括那些只返回 None 或从标识映射中检索的加载：

```py
>>> from sqlalchemy.orm import raiseload
>>> a1 = s.query(A).options(raiseload(A.some_b)).first()
>>> a1.some_b
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'A.some_b' is not available due to lazy='raise'
```

或者只有在会发出 SQL 时才进行惰性加载：

```py
>>> from sqlalchemy.orm import raiseload
>>> a1 = s.query(A).options(raiseload(A.some_b, sql_only=True)).first()
>>> a1.some_b
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: 'A.bs' is not available due to lazy='raise_on_sql'
```

[#3512](https://www.sqlalchemy.org/trac/ticket/3512)

### Mapper.order_by 已弃用

这个参数是 SQLAlchemy 最初版本的一部分，它是 ORM 的原始设计的一部分，其中包含`Mapper`对象作为公共面向的查询结构。这个角色早已被`Query`对象取代，我们在这里使用`Query.order_by()`来指示结果的排序方式，这种方式对于任何组合的 SELECT 语句、实体和 SQL 表达式都是一致的。有许多情况下，`Mapper.order_by`不像预期那样工作（或者预期的结果不清楚），比如当查询组合成联合时；这些情况是不受支持的。

[#3394](https://www.sqlalchemy.org/trac/ticket/3394)

## 新特性和改进 - 核心

### Engines 现在使连接无效，并为 BaseException 运行错误处理程序

版本 1.1 中的新内容：此更改是在 1.1 系列的 1.1 final 版本之前的最后添加，不包含在 1.1 beta 版本中。

Python 的`BaseException`类位于`Exception`之下，但是是诸如`KeyboardInterrupt`、`SystemExit`等系统级异常的可识别基类，特别是`GreenletExit`异常，该异常由 eventlet 和 gevent 使用。这个异常类现在被`Connection`的异常处理例程拦截，并且包括由`ConnectionEvents.handle_error()`事件处理。在不是`Exception`子类的系统级异常的情况下，默认情况下，`Connection` **被失效**，因为假定操作被中断并且连接可能处于不可用状态。这个变化主要针对 MySQL 驱动程序，但是这个变化适用于所有的 DBAPIs。

注意，在失效时，`Connection`所使用的即时 DBAPI 连接被处理，并且如果在异常抛出后仍然在使用`Connection`，则在下次使用时将使用新的 DBAPI 连接进行后续操作；但是，正在进行中的任何事务的状态都会丢失，并且在重新使用之前，必须调用适当的`.rollback()`方法（如果适用）。

为了识别这种变化，很容易证明当这些异常发生在连接正在执行其工作时，一个 pymysql 或 mysqlclient/MySQL-Python 连接会进入一种已损坏的状态；然后，连接将被返回到连接池，在那里后续使用会失败，甚至在返回到池之前，在异常捕获时调用`.rollback()`的上下文管理器中会导致次要失败。这里的行为预计将减少 MySQL 错误“commands out of sync”的发生率，以及在 MySQL 驱动程序未能正确报告`cursor.description`时发生的`ResourceClosedError`，当在杀死 greenlet 的条件下运行时，在处理`KeyboardInterrupt`异常时不完全退出程序。

该行为与通常的自动失效功能不同，它不假设后端数据库本身已关闭或重新启动；对于通常的 DBAPI 断开异常情况，它不会重新循环整个连接池。

此更改应该对所有用户都是净改进，**除了当前拦截`KeyboardInterrupt`或`GreenletExit`并希望在同一事务中继续工作的任何应用程序**。这样的操作在其他不受`KeyboardInterrupt`影响的 DBAPI（如 psycopg2）中理论上是可能的。对于这些 DBAPI，以下解决方法将禁用特定异常时的连接重新循环：

```py
engine = create_engine("postgresql+psycopg2://")

@event.listens_for(engine, "handle_error")
def cancel_disconnect(ctx):
    if isinstance(ctx.original_exception, KeyboardInterrupt):
        ctx.is_disconnect = False
```

[#3803](https://www.sqlalchemy.org/trac/ticket/3803)  ### CTE 支持 INSERT、UPDATE、DELETE

最广泛请求的功能之一是支持与 INSERT、UPDATE、DELETE 一起工作的通用表达式（CTE），现在已实现。INSERT/UPDATE/DELETE 可以从 SQL 顶部陈述的 WITH 子句中获取，也可以作为更大语句上下文中的 CTE 本身使用。

作为这一更改的一部分，包含 CTE 的 INSERT FROM SELECT 现在将在整个语句的顶部呈现 CTE，而不是像 1.0 版本中的 SELECT 语句中嵌套 CTE 那样。

以下是一个示例，它在一条语句中呈现了 UPDATE、INSERT 和 SELECT：

```py
>>> from sqlalchemy import table, column, select, literal, exists
>>> orders = table(
...     "orders",
...     column("region"),
...     column("amount"),
...     column("product"),
...     column("quantity"),
... )
>>>
>>> upsert = (
...     orders.update()
...     .where(orders.c.region == "Region1")
...     .values(amount=1.0, product="Product1", quantity=1)
...     .returning(*(orders.c._all_columns))
...     .cte("upsert")
... )
>>>
>>> insert = orders.insert().from_select(
...     orders.c.keys(),
...     select([literal("Region1"), literal(1.0), literal("Product1"), literal(1)]).where(
...         ~exists(upsert.select())
...     ),
... )
>>>
>>> print(insert)  # Note: formatting added for clarity
WITH  upsert  AS
(UPDATE  orders  SET  amount=:amount,  product=:product,  quantity=:quantity
  WHERE  orders.region  =  :region_1
  RETURNING  orders.region,  orders.amount,  orders.product,  orders.quantity
)
INSERT  INTO  orders  (region,  amount,  product,  quantity)
SELECT
  :param_1  AS  anon_1,  :param_2  AS  anon_2,
  :param_3  AS  anon_3,  :param_4  AS  anon_4
WHERE  NOT  (
  EXISTS  (
  SELECT  upsert.region,  upsert.amount,
  upsert.product,  upsert.quantity
  FROM  upsert)) 
```

[#2551](https://www.sqlalchemy.org/trac/ticket/2551)  ### 对窗口函数内的 RANGE 和 ROWS 规范的支持

新的`over.range_`和`over.rows`参数允许 RANGE 和 ROWS 表达式用于窗口函数：

```py
>>> from sqlalchemy import func

>>> print(func.row_number().over(order_by="x", range_=(-5, 10)))
row_number()  OVER  (ORDER  BY  x  RANGE  BETWEEN  :param_1  PRECEDING  AND  :param_2  FOLLOWING)
>>> print(func.row_number().over(order_by="x", rows=(None, 0)))
row_number()  OVER  (ORDER  BY  x  ROWS  BETWEEN  UNBOUNDED  PRECEDING  AND  CURRENT  ROW)
>>> print(func.row_number().over(order_by="x", range_=(-2, None)))
row_number()  OVER  (ORDER  BY  x  RANGE  BETWEEN  :param_1  PRECEDING  AND  UNBOUNDED  FOLLOWING) 
```

`over.range_` 和 `over.rows` 被指定为 2 元组，表示特定范围的负值和正值，0 表示“当前行”，None 表示无限制。

另请参阅

使用窗口函数

[#3049](https://www.sqlalchemy.org/trac/ticket/3049)  ### 支持 SQL 的 LATERAL 关键字

LATERAL 关键字目前仅被 PostgreSQL 9.3 及更高版本支持，但由于它是 SQL 标准的一部分，因此在 Core 中增加了对此关键字的支持。 `Select.lateral()` 的实现除了仅呈现 LATERAL 关键字之外，还采用了特殊逻辑，以允许从与可选择器相同的 FROM 子句派生的表进行相关联，例如横向相关性：

```py
>>> from sqlalchemy import table, column, select, true
>>> people = table("people", column("people_id"), column("age"), column("name"))
>>> books = table("books", column("book_id"), column("owner_id"))
>>> subq = (
...     select([books.c.book_id])
...     .where(books.c.owner_id == people.c.people_id)
...     .lateral("book_subq")
... )
>>> print(select([people]).select_from(people.join(subq, true())))
SELECT  people.people_id,  people.age,  people.name
FROM  people  JOIN  LATERAL  (SELECT  books.book_id  AS  book_id
FROM  books  WHERE  books.owner_id  =  people.people_id)
AS  book_subq  ON  true 
```

另请参阅

LATERAL correlation

`Lateral`

`Select.lateral()`

[#2857](https://www.sqlalchemy.org/trac/ticket/2857)  ### 对 TABLESAMPLE 的支持

SQL 标准的 TABLESAMPLE 可以使用 `FromClause.tablesample()` 方法呈现，该方法返回一个类似于别名的 `TableSample` 构造：

```py
from sqlalchemy import func

selectable = people.tablesample(func.bernoulli(1), name="alias", seed=func.random())
stmt = select([selectable.c.people_id])
```

假设 `people` 有一个列 `people_id`，上述语句将渲染为：

```py
SELECT  alias.people_id  FROM
people  AS  alias  TABLESAMPLE  bernoulli(:bernoulli_1)
REPEATABLE  (random())
```

[#3718](https://www.sqlalchemy.org/trac/ticket/3718)  ### 对于复合主键列，不再隐式启用 `.autoincrement` 指令

SQLAlchemy 一直以来都具有便利功能，可以为单列整数主键启用后端数据库的“自动增量”功能；所谓“自动增量”是指数据库列将包括数据库提供的任何 DDL 指令，以指示自增长整数标识符，例如 PostgreSQL 上的 SERIAL 关键字或 MySQL 上的 AUTO_INCREMENT，并且此外，方言将使用适合于该后端的技术从执行 `Table.insert()` 构造中接收这些生成的值。

发生变化的是，此功能不再自动为 *复合* 主键打开；以前，表定义如下：

```py
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True),
)
```

只会将`autoincrement`语义应用于`'x'`列，仅因为它是主键列列表中的第一个。为了禁用这个，必须关闭所有列上的`autoincrement`：

```py
# old way
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True, autoincrement=False),
    Column("y", Integer, primary_key=True, autoincrement=False),
)
```

使用新行为，除非列明确标记为`autoincrement=True`，否则复合主键不会具有自动增量语义：

```py
# column 'y' will be SERIAL/AUTO_INCREMENT/ auto-generating
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
)
```

为了预见一些潜在的不兼容情况，`Table.insert()`构造将对没有设置自动增量的复合主键列上缺失的主键值执行更彻底的检查；给定这样一个表：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True),
)
```

当对此表进行无值插入时，会产生以下警告：

```py
SAWarning: Column 'b.x' is marked as a member of the primary
key for table 'b', but has no Python-side or server-side default
generator indicated, nor does it indicate 'autoincrement=True',
and no explicit value is passed.  Primary key columns may not
store NULL. Note that as of SQLAlchemy 1.1, 'autoincrement=True'
must be indicated explicitly for composite (e.g. multicolumn)
primary keys if AUTO_INCREMENT/SERIAL/IDENTITY behavior is
expected for one of the columns in the primary key. CREATE TABLE
statements are impacted by this change as well on most backends.
```

对于从服务器端默认值或者更少见的情况如触发器接收主键值的列，可以使用`FetchedValue`来指示值生成器的存在：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True, server_default=FetchedValue()),
    Column("y", Integer, primary_key=True, server_default=FetchedValue()),
)
```

对于极少数情况下，复合主键实际上打算在其中一个或多个列中存储 NULL 的情况（仅在 SQLite 和 MySQL 上支持），请使用`nullable=True`指定列：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, nullable=True),
)
```

在相关更改中，`autoincrement`标志可以设置为 True，用于具有客户端或服务器端默认值的列。这通常不会对插入期间列的行为产生太大影响。

另请参见

不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

[#3216](https://www.sqlalchemy.org/trac/ticket/3216)  ### 支持 IS DISTINCT FROM 和 IS NOT DISTINCT FROM

新操作符`ColumnOperators.is_distinct_from()`和`ColumnOperators.isnot_distinct_from()`允许 IS DISTINCT FROM 和 IS NOT DISTINCT FROM sql 操作：

```py
>>> print(column("x").is_distinct_from(None))
x  IS  DISTINCT  FROM  NULL 
```

处理 NULL、True 和 False：

```py
>>> print(column("x").isnot_distinct_from(False))
x  IS  NOT  DISTINCT  FROM  false 
```

对于 SQLite，它没有这个运算符，“IS” / “IS NOT”会被渲染，这在 SQLite 上可以用于 NULL，不像其他后端：

```py
>>> from sqlalchemy.dialects import sqlite
>>> print(column("x").is_distinct_from(None).compile(dialect=sqlite.dialect()))
x  IS  NOT  NULL 
```  ### Core 和 ORM 支持 FULL OUTER JOIN

新标志`FromClause.outerjoin.full`，在 Core 和 ORM 级别可用，指示编译器在通常渲染`LEFT OUTER JOIN`的地方渲染`FULL OUTER JOIN`：

```py
stmt = select([t1]).select_from(t1.outerjoin(t2, full=True))
```

该标志也适用于 ORM 级别：

```py
q = session.query(MyClass).outerjoin(MyOtherClass, full=True)
```

[#1957](https://www.sqlalchemy.org/trac/ticket/1957)  ### ResultSet 列匹配增强；文本 SQL 的位置列设置

在 1.0 系列中对 `ResultProxy` 系统进行了一系列改进，作为 [#918](https://www.sqlalchemy.org/trac/ticket/918) 的一部分，它重新组织了内部，使游标绑定的结果列与表/ORM 元数据按位置匹配，而不是按名称匹配，用于包含有关要返回的结果行的完整信息的编译 SQL 构造。这允许大大节省 Python 开销，并且在将 ORM 和 Core SQL 表达式链接到结果行时具有更高的准确性。在 1.1 版本中，此重新组织已进一步在内部进行，并且还通过最近添加的 `TextClause.columns()` 方法可用于纯文本 SQL 构造。

#### TextAsFrom.columns() 现在按位置工作

`TextClause.columns()` 方法在 0.9 版本中新增，接受基于列的参数位置；在 1.1 版本中，当所有列都按位置传递时，这些列与最终结果集的关联也将按位置执行。这里的关键优势在于，现在可以将文本 SQL 链接到 ORM 级别的结果集，而无需处理模糊或重复的列名称，也无需匹配标签方案到 ORM 级别的标签方案。现在所需的只是在文本 SQL 中以及传递给 `TextClause.columns()` 的列参数内部的相同列顺序：

```py
from sqlalchemy import text

stmt = text(
    "SELECT users.id, addresses.id, users.id, "
    "users.name, addresses.email_address AS email "
    "FROM users JOIN addresses ON users.id=addresses.user_id "
    "WHERE users.id = 1"
).columns(User.id, Address.id, Address.user_id, User.name, Address.email_address)

query = session.query(User).from_statement(stmt).options(contains_eager(User.addresses))
result = query.all()
```

在上面的文本 SQL 中，“id” 列重复出现了三次，这通常是不明确的。使用新功能，我们可以直接应用来自 `User` 和 `Address` 类的映射列，甚至将 `Address.user_id` 列链接到文本 SQL 中的 `users.id` 列以供娱乐，而 `Query` 对象将收到正确的可按需定位的行，包括用于急加载。

这种更改与使用不同顺序将列传递给方法的代码**不兼容**。希望由于这种方法一直以来都是按照文本 SQL 语句中列的相同顺序传递列的方式来记录的，因此其影响将会很小，即使内部未进行此检查也是如此。无论如何，该方法仅从 0.9 版本开始添加，可能尚未被广泛使用。有关如何处理使用此方法的应用程序的行为更改的详细说明，请参见当传递列位置性地传递时，TextClause.columns() 将不按名称匹配列。

另请参阅

使用文本列表达式进行选择

> 当传递位置参数时，`TextClause.columns()`将按位置而不是按名称匹配列 - 向后兼容说明

#### 位置匹配优先于基于名称的匹配用于 Core/ORM SQL 构造

此更改的另一个方面是对于编译后的 SQL 构造，匹配列的规则也已经修改，更充分地依赖于“位置”匹配。给定以下语句：

```py
ua = users.alias("ua")
stmt = select([users.c.user_id, ua.c.user_id])
```

上述语句将编译为：

```py
SELECT  users.user_id,  ua.user_id  FROM  users,  users  AS  ua
```

在 1.0 中，当执行上述语句时，将使用位置匹配来匹配其原始编译结构，但是因为该语句包含重复的`'user_id'`标签，所以“模糊列”规则仍然会涉及并阻止从行中获取列。从 1.1 开始，“模糊列”规则不会影响从列构造到 SQL 列的精确匹配，这是 ORM 用于获取列的方式：

```py
result = conn.execute(stmt)
row = result.first()

# these both match positionally, so no error
user_id = row[users.c.user_id]
ua_id = row[ua.c.user_id]

# this still raises, however
user_id = row["user_id"]
```

#### 很少会收到“模糊列”错误消息

作为此更改的一部分，错误消息`结果集中的模糊列名'<name>'！尝试在 select 语句上使用 'use_labels' 选项。`的措辞已经有所减少；因为现在当使用 ORM 或 Core 编译后的 SQL 构造时，这个消息应该极其罕见，所以它只是简单地陈述了`结果集列描述中的模糊列名'<name>'`，仅当使用实际模糊的名称从渲染的 SQL 语句中检索结果列时，例如上面的`row['user_id']`。它现在还引用了来自渲染的 SQL 语句本身的实际模糊名称，而不是指示用于获取的构造本地的键或名称。

[#3501](https://www.sqlalchemy.org/trac/ticket/3501)  ### 支持 Python 的原生`enum`类型及兼容形式

`Enum`类型现在可以使用任何符合 PEP-435 的枚举类型构造。在使用此模式时，输入值和返回值是实际的枚举对象，而不是字符串/整数等值：

```py
import enum
from sqlalchemy import Table, MetaData, Column, Enum, create_engine

class MyEnum(enum.Enum):
    one = 1
    two = 2
    three = 3

t = Table("data", MetaData(), Column("value", Enum(MyEnum)))

e = create_engine("sqlite://")
t.create(e)

e.execute(t.insert(), {"value": MyEnum.two})
assert e.scalar(t.select()) is MyEnum.two
```

#### `Enum.enums`集合现在是一个列表而不是一个元组

作为对 `Enum` 的更改的一部分，`Enum.enums`元素的集合现在是一个列表而不是一个元组。这是因为列表适用于长度可变的同类项序列，其中元素的位置没有语义上的重要性。

[#3292](https://www.sqlalchemy.org/trac/ticket/3292)  ### 核心结果行支持负整数索引

`RowProxy`对象现在像常规的 Python 序列一样支持单个负整数索引，无论是在纯 Python 版本还是在 C 扩展版本中。以前，负值仅在切片中起作用：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://")
>>> row = e.execute("select 1, 2, 3").first()
>>> row[-1], row[-2], row[1], row[-2:2]
3 2 2 (2,)
```  ### `Enum`类型现在在 Python 中对值进行验证

为了适应 Python 本机枚举对象，以及诸如在 ARRAY 中使用非本地 ENUM 类型且 CHECK 约束不可行等边缘情况，当使用`Enum.validate_strings`标志时，`Enum`数据类型现在在 Python 端验证输入值（1.1.0b2）：

```py
>>> from sqlalchemy import Table, MetaData, Column, Enum, create_engine
>>> t = Table(
...     "data",
...     MetaData(),
...     Column("value", Enum("one", "two", "three", validate_strings=True)),
... )
>>> e = create_engine("sqlite://")
>>> t.create(e)
>>> e.execute(t.insert(), {"value": "four"})
Traceback (most recent call last):
  ...
sqlalchemy.exc.StatementError: (exceptions.LookupError)
"four" is not among the defined enum values
[SQL: u'INSERT INTO data (value) VALUES (?)']
[parameters: [{'value': 'four'}]]
```

默认情况下，此验证是关闭的，因为已经确定了用户不希望进行此类验证的用例（如字符串比较）。对于非字符串类型，它在所有情况下必须进行。当从数据库返回值时，结果处理方面的检查也是无条件发生的。

这种验证是在使用非本地枚举类型时创建 CHECK 约束的现有行为之外的。现在可以使用新的`Enum.create_constraint`标志禁用此 CHECK 约束的创建。

[#3095](https://www.sqlalchemy.org/trac/ticket/3095)  ### 非本地布尔整数值在所有情况下被强制转换为零/一/None

`Boolean`数据类型将 Python 布尔值强制转换为整数值，用于那些没有本地布尔类型的后端，如 SQLite 和 MySQL。在这些后端，通常设置一个 CHECK 约束，以确保数据库中的值实际上是这两个值之一。然而，MySQL 忽略 CHECK 约束，该约束是可选的，现有数据库可能没有这个约束。`Boolean`数据类型已修复，使得已经是整数值的 Python 端值被强制转换为零或一，而不仅仅是原样传递；此外，结果的 C 扩展版本的整数到布尔处理器现在使用与 Python 布尔值解释相同的值，而不是断言一个确切的一或零值。这现在与纯 Python 整数到布尔处理器一致，并且对数据库中已有的数据更宽容。None/NULL 值仍然保留为 None/NULL。

注意

此更改意外地导致非整数值（例如字符串）的解释行为也发生了更改，使得字符串值 `"0"` 被解释为“true”，但仅适用于没有本机布尔数据类型的后端 - 在“本机布尔”后端（如 PostgreSQL）上，字符串值 `"0"` 直接传递给驱动程序，并解释为“false”。这是以前实现中没有发生的不一致性。应注意，将字符串或任何其他值传递给 `Boolean` 数据类型外的值 `None`、`True`、`False`、`1`、`0` 是 **不受支持** 的，版本 1.2 将为此场景引发错误（或可能只是发出警告，待定）。另请参阅 [#4102](https://www.sqlalchemy.org/trac/ticket/4102)。

[#3730](https://www.sqlalchemy.org/trac/ticket/3730)  ### 在日志和异常显示中，现在会截断大参数和行值

在 SQL 语句的绑定参数中存在大值，以及在结果行中存在大值，现在在日志记录、异常报告以及行本身的 `repr()` 中都将被截断显示：

```py
>>> from sqlalchemy import create_engine
>>> import random
>>> e = create_engine("sqlite://", echo="debug")
>>> some_value = "".join(chr(random.randint(52, 85)) for i in range(5000))
>>> row = e.execute("select ?", [some_value]).first()
... # (lines are wrapped for clarity) ...
2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine select ?
2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine
('E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;NM6GU
LUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7>4=4:P
GJ7HQ6 ... (4702 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=RJP
HDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
K:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine Col ('?',)
2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine
Row (u'E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;
NM6GULUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7
>4=4:PGJ7HQ ... (4703 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=
RJPHDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
K:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
>>> print(row)
(u'E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;NM6
GULUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7>4
=4:PGJ7HQ ... (4703 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;
=RJPHDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9H
MK:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
```

[#2837](https://www.sqlalchemy.org/trac/ticket/2837)  ### Core 添加了 JSON 支持

由于 MySQL 现在除了 PostgreSQL JSON 数据类型外还有 JSON 数据类型，核心现在获得了一个 `sqlalchemy.types.JSON` 数据类型，它是这两者的基础。使用此类型允许以 PostgreSQL 和 MySQL 通用的方式访问“getitem”操作符和“getpath”操作符。

新数据类型还对 NULL 值的处理以及表达式处理进行了一系列改进。

另请参阅

MySQL JSON 支持

`JSON`

`JSON`

`JSON`

[#3619](https://www.sqlalchemy.org/trac/ticket/3619)

#### 使用 ORM 操作时，“null” 会如预期地插入，当不存在时则被省略

`JSON` 类型及其子类型 `JSON` 和 `JSON` 具有一个标志 `JSON.none_as_null`，当设置为 True 时表示 Python 值 `None` 应该转换为 SQL NULL 而不是 JSON NULL 值。此标志默认为 False，这意味着 Python 值 `None` 应该导致 JSON NULL 值。

在以下情况下，此逻辑将失败，现在已进行更正：

1\. 当列还包含默认值或`server_default`值时，对预期持久化 JSON “null” 的映射属性上的`None`的正值仍将触发列级默认值，替换`None`值：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), default="some default")

# would insert "some default" instead of "'null'",
# now will insert "'null'"
obj = MyObject(json_value=None)
session.add(obj)
session.commit()
```

2\. 当列*没有*包含默认值或`server_default`值时，在配置了`none_as_null=False`的 JSON 列上的缺失值仍将呈现 JSON NULL，而不是回退到不插入任何值，这与所有其他数据类型的行为不一致：

```py
class MyObject(Base):
    # ...

    some_other_value = Column(String(50))
    json_value = Column(JSON(none_as_null=False))

# would result in NULL for some_other_value,
# but json "'null'" for json_value.  Now results in NULL for both
# (the json_value is omitted from the INSERT)
obj = MyObject()
session.add(obj)
session.commit()
```

这是一种行为变更，对于依赖此行为将缺失值默认为 JSON null 的应用程序而言，这是不兼容的。这基本上表明**缺失值与存在的`None`值是有区别的**。有关更多详细信息，请参见 JSON Columns will not insert JSON NULL if no value is supplied and no default is established。

3\. 当使用`Session.bulk_insert_mappings()`方法时，无论何种情况下都会忽略`None`：

```py
# would insert SQL NULL and/or trigger defaults,
# now inserts "'null'"
session.bulk_insert_mappings(MyObject, [{"json_value": None}])
```

`JSON`类型现在实现了`TypeEngine.should_evaluate_none`标志，指示此处不应忽略`None`；它会根据`JSON.none_as_null`的值自动配置。感谢 [#3061](https://www.sqlalchemy.org/trac/ticket/3061)，我们可以区分用户主动设置`None`值和根本没有设置的情况。

该特性同样适用于新的基本`JSON`类型及其后代类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)  #### 新增 JSON.NULL 常量

为了确保应用程序始终可以完全控制一个 `JSON`、`JSON`、`JSON` 或 `JSONB` 列是否接收 SQL NULL 或 JSON `"null"` 值，添加了常量 `JSON.NULL`，它与 `null()` 结合使用，可以完全确定 SQL NULL 和 JSON `"null"` 之间的区别，不管 `JSON.none_as_null` 设置为何值：

```py
from sqlalchemy import null
from sqlalchemy.dialects.postgresql import JSON

obj1 = MyObject(json_value=null())  # will *always* insert SQL NULL
obj2 = MyObject(json_value=JSON.NULL)  # will *always* insert JSON string "null"

session.add_all([obj1, obj2])
session.commit()
```

这个功能同样适用于新的基础 `JSON` 类型及其派生类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)  ### Core 中添加了数组支持；新的 ANY 和 ALL 运算符

除了对 PostgreSQL `ARRAY` 类型所做的增强描述在 通过数组、JSON、HSTORE 的索引访问建立正确的 SQL 类型 中，`ARRAY` 的基类本身已经移动到 Core 中，成为一个新的类 `ARRAY`。

数组是 SQL 标准的一部分，还有一些面向数组的函数，比如 `array_agg()` 和 `unnest()`。为了支持这些构造不仅仅是针对 PostgreSQL，还有可能是未来其他支持数组的后端，比如 DB2，现在大部分 SQL 表达式的数组逻辑都在 Core 中。`ARRAY` 类型仍然**只在 PostgreSQL 上工作**，但可以直接使用，支持特殊的数组用例，比如索引访问，以及支持 ANY 和 ALL：

```py
mytable = Table("mytable", metadata, Column("data", ARRAY(Integer, dimensions=2)))

expr = mytable.c.data[5][6]

expr = mytable.c.data[5].any(12)
```

为了支持 ANY 和 ALL，`ARRAY` 类型保留了与 PostgreSQL 类型相同的 `Comparator.any()` 和 `Comparator.all()` 方法，但也将这些操作导出为新的独立运算符函数 `any_()` 和 `all_()`。这两个函数以更传统的 SQL 方式工作，允许右侧表达式形式，如：

```py
from sqlalchemy import any_, all_

select([mytable]).where(12 == any_(mytable.c.data[5]))
```

对于 PostgreSQL 特定的运算符“contains”，“contained_by” 和 “overlaps”，应继续直接使用 `ARRAY` 类型，该类型还提供了 `ARRAY` 类型的所有功能。

`any_()` 和 `all_()` 运算符在核心层面是开放的，但是后端数据库对它们的解释是有限的。在 PostgreSQL 后端，这两个运算符**只接受数组值**。而在 MySQL 后端，它们**只接受子查询值**。在 MySQL 中，可以使用如下表达式：

```py
from sqlalchemy import any_, all_

subq = select([mytable.c.value])
select([mytable]).where(12 > any_(subq))
```

[#3516](https://www.sqlalchemy.org/trac/ticket/3516)  ### 新功能特性，“WITHIN GROUP”，array_agg 和 set 聚合函数

使用新的 `ARRAY` 类型，我们还可以实现一个预定义的函数，用于返回一个数组的 `array_agg()` SQL 函数，现在可以使用 `array_agg`：

```py
from sqlalchemy import func

stmt = select([func.array_agg(table.c.value)])
```

通过 `aggregate_order_by` 还添加了一个用于聚合 ORDER BY 的 PostgreSQL 元素��

```py
from sqlalchemy.dialects.postgresql import aggregate_order_by

expr = func.array_agg(aggregate_order_by(table.c.a, table.c.b.desc()))
stmt = select([expr])
```

产生：

```py
SELECT  array_agg(table1.a  ORDER  BY  table1.b  DESC)  AS  array_agg_1  FROM  table1
```

PG 方言本身还提供了一个 `array_agg()` 包装器，以确保 `ARRAY` 类型：

```py
from sqlalchemy.dialects.postgresql import array_agg

stmt = select([array_agg(table.c.value).contains("foo")])
```

此外，像 `percentile_cont()`、`percentile_disc()`、`rank()`、`dense_rank()` 等需要通过 `WITHIN GROUP (ORDER BY <expr>)` 进行排序的函数现在可以通过 `FunctionElement.within_group()` 修改器使用：

```py
from sqlalchemy import func

stmt = select(
    [
        department.c.id,
        func.percentile_cont(0.5).within_group(department.c.salary.desc()),
    ]
)
```

上述语句会产生类似于以下 SQL：

```py
SELECT  department.id,  percentile_cont(0.5)
WITHIN  GROUP  (ORDER  BY  department.salary  DESC)
```

现在为这些函数提供了正确返回类型的占位符，并包括 `percentile_cont`、`percentile_disc`、`rank`、`dense_rank`、`mode`、`percent_rank` 和 `cume_dist`。

[#3132](https://www.sqlalchemy.org/trac/ticket/3132) [#1370](https://www.sqlalchemy.org/trac/ticket/1370)  ### TypeDecorator 现在自动与 Enum、Boolean、“schema” 类型一起工作

`SchemaType` 类型包括诸如 `Enum` 和 `Boolean` 等类型，除了对应数据库类型外，还会自动生成 CHECK 约束或者在 PostgreSQL ENUM 的情况下生成新的 CREATE TYPE 语句，现在可以自动与 `TypeDecorator` 配方一起使用了。以前，`ENUM` 的 `TypeDecorator` 需要像这样：

```py
# old way
class MyEnum(TypeDecorator, SchemaType):
    impl = postgresql.ENUM("one", "two", "three", name="myenum")

    def _set_table(self, table):
        self.impl._set_table(table)
```

`TypeDecorator` 现在传播这些额外的事件，所以它可以像任何其他类型一样进行操作：

```py
# new way
class MyEnum(TypeDecorator):
    impl = postgresql.ENUM("one", "two", "three", name="myenum")
```

[#2919](https://www.sqlalchemy.org/trac/ticket/2919)  ### 多租户模式下的 Table 对象的架构翻译

为了支持一个应用程序使用许多模式中相同的一组 `Table` 对象的用例，例如每个用户一个模式，添加了一个新的执行选项 `Connection.execution_options.schema_translate_map`。 使用这个映射，一组 `Table` 对象可以在每个连接基础上被制作，以引用任何一组模式，而不是它们被分配到的 `Table.schema`。 该翻译适用于 DDL 和 SQL 生成，以及与 ORM 一起使用。

例如，如果 `User` 类被分配了模式“per_user”：

```py
class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True)

    __table_args__ = {"schema": "per_user"}
```

在每个请求上，`Session` 可以设置为每次引用不同的模式：

```py
session = Session()
session.connection(
    execution_options={"schema_translate_map": {"per_user": "account_one"}}
)

# will query from the ``account_one.user`` table
session.query(User).get(5)
```

另请参阅

模式名称的翻译

[#2685](https://www.sqlalchemy.org/trac/ticket/2685)  ### “友好”的 Core SQL 构造的字符串化，没有方言

对 Core SQL 构造调用 `str()` 现在会在更多情况下产生一个字符串，支持各种通常不在默认 SQL 中出现的 SQL 构造，如 RETURNING、数组索引和非标准数据类型：

```py
>>> from sqlalchemy import table, column
t>>> t = table('x', column('a'), column('b'))
>>> print(t.insert().returning(t.c.a, t.c.b))
INSERT  INTO  x  (a,  b)  VALUES  (:a,  :b)  RETURNING  x.a,  x.b 
```

`str()` 函数现在调用一个完全独立的方言/编译器，仅用于普通字符串打印而没有特定的方言设置，因此当出现更多“只是显示给我一个字符串！”的情况时，这些可以添加到这个方言/编译器中，而不会影响真实方言上的行为。

另请参阅

查询的字符串化将咨询 Session 获取正确的方言

[#3631](https://www.sqlalchemy.org/trac/ticket/3631)  ### `type_coerce` 函数现在是一个持久的 SQL 元素

`type_coerce()` 函数以前会返回一个对象，要么是类型为 `BindParameter`，要么是类型为 `Label`，取决于输入。 这将产生的影响是，在使用表达式转换的情况下，例如将元素从 `Column` 转换为 `BindParameter`，这对 ORM 级别的延迟加载至关重要，类型强制转换信息将不会被使用，因为它已经丢失了。

为了改进这种行为，该函数现在返回一个围绕给定表达式的持久`TypeCoerce`容器，该表达式本身保持不变；这个结构会被 SQL 编译器显式评估。这样可以确保内部表达式的强制转换在语句如何修改时都能保持不变，包括如果包含的元素被替换为另一个元素，这在 ORM 的延迟加载功能中很常见。

展示效果的测试案例利用了异构的 primaryjoin 条件，结合自定义类型和延迟加载。给定一个应用 CAST 作为“绑定表达式”的自定义类型：

```py
class StringAsInt(TypeDecorator):
    impl = String

    def column_expression(self, col):
        return cast(col, Integer)

    def bind_expression(self, value):
        return cast(value, String)
```

然后，一个映射，我们将一个表上的字符串 “id” 列与另一个表上的整数 “id” 列进行等价比较：

```py
class Person(Base):
    __tablename__ = "person"
    id = Column(StringAsInt, primary_key=True)

    pets = relationship(
        "Pets",
        primaryjoin=(
            "foreign(Pets.person_id)==cast(type_coerce(Person.id, Integer), Integer)"
        ),
    )

class Pets(Base):
    __tablename__ = "pets"
    id = Column("id", Integer, primary_key=True)
    person_id = Column("person_id", Integer)
```

在上面的 `relationship.primaryjoin` 表达式中，我们使用 `type_coerce()` 处理通过延迟加载传递的绑定参数作为整数，因为我们已经知道这些参数将来自我们的 `StringAsInt` 类型，该类型在 Python 中保持为整数值。然后我们使用 `cast()`，以便作为 SQL 表达式，VARCHAR “id” 列将被 CAST 为整数，用于常规非转换连接，如 `Query.join()` 或 `joinedload()`。也就是说，`.pets` 的 joinedload 如下所示：

```py
SELECT  person.id  AS  person_id,  pets_1.id  AS  pets_1_id,
  pets_1.person_id  AS  pets_1_person_id
FROM  person
LEFT  OUTER  JOIN  pets  AS  pets_1
ON  pets_1.person_id  =  CAST(person.id  AS  INTEGER)
```

在连接的 ON 子句中没有 CAST，像 PostgreSQL 这样的强类型数据库将拒绝隐式比较整数并失败。

`.pets` 的延迟加载情况依赖于在加载时用绑定参数替换 `Person.id` 列，该参数接收一个 Python 加载的值。这种替换是我们的 `type_coerce()` 函数意图会丢失的具体地方。在更改之前，这种延迟加载如下所示：

```py
SELECT  pets.id  AS  pets_id,  pets.person_id  AS  pets_person_id
FROM  pets
WHERE  pets.person_id  =  CAST(CAST(%(param_1)s  AS  VARCHAR)  AS  INTEGER)
-- {'param_1': 5}
```

在上面的例子中，我们看到我们在 Python 中的值 `5` 首先被 CAST 为 VARCHAR，然后在 SQL 中再次被 CAST 为 INTEGER；这是一个双重 CAST，虽然有效，但并不是我们要求的。

随着这一变化，`type_coerce()`函数在列被替换为绑定参数后仍保持一个包装器，查询现在看起来像这样：

```py
SELECT  pets.id  AS  pets_id,  pets.person_id  AS  pets_person_id
FROM  pets
WHERE  pets.person_id  =  CAST(%(param_1)s  AS  INTEGER)
-- {'param_1': 5}
```

我们主要连接中的外部 CAST 仍然生效，但是`StringAsInt`自定义类型的不必要的 CAST 部分已经被`type_coerce()`函数按预期移除。

[#3531](https://www.sqlalchemy.org/trac/ticket/3531)  ### 引擎现在使连接无效，运行 BaseException 的错误处理程序

新版本 1.1 中新增：这个变化是在 1.1 最终版本之前的 1.1 系列中添加的，不包含在 1.1 beta 版本中。

Python 的 `BaseException` 类位于 `Exception` 之下，但是是系统级异常（如`KeyboardInterrupt`、`SystemExit`，尤其是由 eventlet 和 gevent 使用的`GreenletExit`异常）的可识别基类。现在，`Connection` 的异常处理例程拦截了这个异常类，并包括由 `ConnectionEvents.handle_error()` 事件处理。在不是`Exception`子类的系统级异常的情况下，默认情况下现在会**使连接无效**，因为假定操作被中断，连接可能处于不可用状态。这个变化主要针对 MySQL 驱动程序，但是这个变化适用于所有 DBAPI。

在无效化时，`Connection` 使用的即时 DBAPI 连接会被释放，如果在异常抛出后仍在使用 `Connection`，则在下次使用时会使用新的 DBAPI 连接进行后续操作；然而，任何正在进行中的事务状态都会丢失，必须在重新使用之前调用适当的`.rollback()`方法（如果适用）。

为了识别这种变化，很容易证明当这些异常发生在连接正在工作时，pymysql 或 mysqlclient / MySQL-Python 连接会进入损坏状态；然后连接将被返回到连接池，后续使用将失败，甚至在返回到池之前会导致在调用`.rollback()`的上下文管理器中引发次要故障。这里的行为预期将减少 MySQL 错误“commands out of sync”的发生，以及在 MySQL 驱动程序未能正确报告`cursor.description`时可能发生的`ResourceClosedError`，当在 greenlet 条件下运行时，其中 greenlets 被终止，或者处理`KeyboardInterrupt`异常而不完全退出程序时。

这种行为与通常的自动失效功能不同，它不假设后端数据库本身已关闭或重新启动；它不像通常的 DBAPI 断开异常那样重新生成整个连接池。

这个改变应该对所有用户都是一个净改进，除了**任何当前拦截``KeyboardInterrupt``或``GreenletExit``并希望在同一事务中继续工作的应用程序**。对于那些理论上不受`KeyboardInterrupt`影响的其他 DBAPIs，比如 psycopg2，这样的操作是可能的。对于这些 DBAPIs，以下解决方法将禁用特定异常的连接被回收：

```py
engine = create_engine("postgresql+psycopg2://")

@event.listens_for(engine, "handle_error")
def cancel_disconnect(ctx):
    if isinstance(ctx.original_exception, KeyboardInterrupt):
        ctx.is_disconnect = False
```

[#3803](https://www.sqlalchemy.org/trac/ticket/3803)

### 支持 INSERT、UPDATE、DELETE 的 CTE

最广泛请求的功能之一是支持与 INSERT、UPDATE、DELETE 一起工作的通用表达式（CTE），现在已经实现。INSERT/UPDATE/DELETE 可以从位于 SQL 顶部的 WITH 子句中提取，也可以作为更大语句上下文中的 CTE 使用。

作为这个改变的一部分，包含 CTE 的 INSERT FROM SELECT 现在将在整个语句的顶部呈现 CTE，而不是像 1.0 中那样嵌套在 SELECT 语句中。

下面是一个在一个语句中呈现 UPDATE、INSERT 和 SELECT 的示例：

```py
>>> from sqlalchemy import table, column, select, literal, exists
>>> orders = table(
...     "orders",
...     column("region"),
...     column("amount"),
...     column("product"),
...     column("quantity"),
... )
>>>
>>> upsert = (
...     orders.update()
...     .where(orders.c.region == "Region1")
...     .values(amount=1.0, product="Product1", quantity=1)
...     .returning(*(orders.c._all_columns))
...     .cte("upsert")
... )
>>>
>>> insert = orders.insert().from_select(
...     orders.c.keys(),
...     select([literal("Region1"), literal(1.0), literal("Product1"), literal(1)]).where(
...         ~exists(upsert.select())
...     ),
... )
>>>
>>> print(insert)  # Note: formatting added for clarity
WITH  upsert  AS
(UPDATE  orders  SET  amount=:amount,  product=:product,  quantity=:quantity
  WHERE  orders.region  =  :region_1
  RETURNING  orders.region,  orders.amount,  orders.product,  orders.quantity
)
INSERT  INTO  orders  (region,  amount,  product,  quantity)
SELECT
  :param_1  AS  anon_1,  :param_2  AS  anon_2,
  :param_3  AS  anon_3,  :param_4  AS  anon_4
WHERE  NOT  (
  EXISTS  (
  SELECT  upsert.region,  upsert.amount,
  upsert.product,  upsert.quantity
  FROM  upsert)) 
```

[#2551](https://www.sqlalchemy.org/trac/ticket/2551)

### 支持窗口函数中的 RANGE 和 ROWS 规范

新的`over.range_`和`over.rows`参数允许窗口函数使用 RANGE 和 ROWS 表达式：

```py
>>> from sqlalchemy import func

>>> print(func.row_number().over(order_by="x", range_=(-5, 10)))
row_number()  OVER  (ORDER  BY  x  RANGE  BETWEEN  :param_1  PRECEDING  AND  :param_2  FOLLOWING)
>>> print(func.row_number().over(order_by="x", rows=(None, 0)))
row_number()  OVER  (ORDER  BY  x  ROWS  BETWEEN  UNBOUNDED  PRECEDING  AND  CURRENT  ROW)
>>> print(func.row_number().over(order_by="x", range_=(-2, None)))
row_number()  OVER  (ORDER  BY  x  RANGE  BETWEEN  :param_1  PRECEDING  AND  UNBOUNDED  FOLLOWING) 
```

`over.range_`和`over.rows`被指定为 2 元组，表示特定范围的负值和正值，“CURRENT ROW”为 0，UNBOUNDED 为 None。

另请参见

使用窗口函数

[#3049](https://www.sqlalchemy.org/trac/ticket/3049)

### 支持 SQL 的 LATERAL 关键字

LATERAL 关键字目前只被已知支持 PostgreSQL 9.3 及更高版本，然而，由于它是 SQL 标准的一部分，因此 Core 添加了对该关键字的支持。`Select.lateral()`的实现采用了特殊逻辑，不仅仅是呈现 LATERAL 关键字，还允许来自与可选择器相同 FROM 子句派生的表的相关性，例如，侧向相关性：

```py
>>> from sqlalchemy import table, column, select, true
>>> people = table("people", column("people_id"), column("age"), column("name"))
>>> books = table("books", column("book_id"), column("owner_id"))
>>> subq = (
...     select([books.c.book_id])
...     .where(books.c.owner_id == people.c.people_id)
...     .lateral("book_subq")
... )
>>> print(select([people]).select_from(people.join(subq, true())))
SELECT  people.people_id,  people.age,  people.name
FROM  people  JOIN  LATERAL  (SELECT  books.book_id  AS  book_id
FROM  books  WHERE  books.owner_id  =  people.people_id)
AS  book_subq  ON  true 
```

另请参见

LATERAL correlation

`Lateral`

`Select.lateral()`

[#2857](https://www.sqlalchemy.org/trac/ticket/2857)

### 支持 TABLESAMPLE

SQL 标准的 TABLESAMPLE 可以使用`FromClause.tablesample()`方法呈现，该方法返回一个类似于别名的`TableSample`构造：

```py
from sqlalchemy import func

selectable = people.tablesample(func.bernoulli(1), name="alias", seed=func.random())
stmt = select([selectable.c.people_id])
```

假设`people`有一个列`people_id`，上述语句会被渲染为：

```py
SELECT  alias.people_id  FROM
people  AS  alias  TABLESAMPLE  bernoulli(:bernoulli_1)
REPEATABLE  (random())
```

[#3718](https://www.sqlalchemy.org/trac/ticket/3718)

### 对于复合主键列，`.autoincrement`指令不再隐式启用

SQLAlchemy 一直以来都有一个方便的特性，即为单列整数主键启用后端数据库的“自增”功能；所谓的“自增”，是指数据库列将包含任何 DDL 指令，以指示自增长整数标识符，例如在 PostgreSQL 上的 SERIAL 关键字或在 MySQL 上的 AUTO_INCREMENT，并且此外，方言将通过执行`Table.insert()`构造使用适合于该后端的技术来接收这些生成的值。

改变的是，此功能不再自动应用于*复合*主键；以前，表定义如下：

```py
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True),
)
```

仅因为它在主键列列表中排在第一位，因此会应用于`'x'`列的“自增”语义。为了禁用此功能，必须关闭所有列上的`autoincrement`：

```py
# old way
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True, autoincrement=False),
    Column("y", Integer, primary_key=True, autoincrement=False),
)
```

使用新的行为，除非列被明确地标记为`autoincrement=True`，否则复合主键将不具有自增语义：

```py
# column 'y' will be SERIAL/AUTO_INCREMENT/ auto-generating
Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
)
```

为了预测一些潜在的不向后兼容的情况，`Table.insert()`构造将对没有设置自增的复合主键列上的缺失主键值执行更彻底的检查；给定一个表，如下所示：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True),
)
```

对于这个表没有提供任何值的 INSERT 将会产生警告：

```py
SAWarning: Column 'b.x' is marked as a member of the primary
key for table 'b', but has no Python-side or server-side default
generator indicated, nor does it indicate 'autoincrement=True',
and no explicit value is passed.  Primary key columns may not
store NULL. Note that as of SQLAlchemy 1.1, 'autoincrement=True'
must be indicated explicitly for composite (e.g. multicolumn)
primary keys if AUTO_INCREMENT/SERIAL/IDENTITY behavior is
expected for one of the columns in the primary key. CREATE TABLE
statements are impacted by this change as well on most backends.
```

对于从服务器端默认值或者触发器等不太常见的情况下接收主键值的列，可以使用`FetchedValue`来指示存在值生成器：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True, server_default=FetchedValue()),
    Column("y", Integer, primary_key=True, server_default=FetchedValue()),
)
```

对于极少情况下实际上意图在其列中存储 NULL 的复合主键（仅在 SQLite 和 MySQL 上支持），请使用 `nullable=True` 指定列：

```py
Table(
    "b",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, nullable=True),
)
```

在相关更改中，可以在具有客户端或服务器端默认值的列上将 `autoincrement` 标志设置为 True。这通常不会对插入时列的行为产生太大影响。

另请参阅

不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

[#3216](https://www.sqlalchemy.org/trac/ticket/3216)

### 支持 IS DISTINCT FROM 和 IS NOT DISTINCT FROM

新运算符 `ColumnOperators.is_distinct_from()` 和 `ColumnOperators.isnot_distinct_from()` 允许 IS DISTINCT FROM 和 IS NOT DISTINCT FROM sql 操作：

```py
>>> print(column("x").is_distinct_from(None))
x  IS  DISTINCT  FROM  NULL 
```

处理 NULL、True 和 False：

```py
>>> print(column("x").isnot_distinct_from(False))
x  IS  NOT  DISTINCT  FROM  false 
```

对于不具有此运算符的 SQLite，将渲染“IS” / “IS NOT”，在 SQLite 上与其他后端不同，对 NULL 有效：

```py
>>> from sqlalchemy.dialects import sqlite
>>> print(column("x").is_distinct_from(None).compile(dialect=sqlite.dialect()))
x  IS  NOT  NULL 
```

### 完全外连接的核心和 ORM 支持

新标志 `FromClause.outerjoin.full`，在核心和 ORM 级别可用，指示编译器在通常渲染 `LEFT OUTER JOIN` 的地方渲染 `FULL OUTER JOIN`：

```py
stmt = select([t1]).select_from(t1.outerjoin(t2, full=True))
```

该标志也适用于 ORM 级别：

```py
q = session.query(MyClass).outerjoin(MyOtherClass, full=True)
```

[#1957](https://www.sqlalchemy.org/trac/ticket/1957)

### ResultSet 列匹配增强；文本 SQL 的位置列设置

在 1.0 系列中对 `ResultProxy` 系统进行了一系列改进，作为 [#918](https://www.sqlalchemy.org/trac/ticket/918) 的一部分，重新组织了内部结构，以便通过位置而不是通过匹配名称将游标绑定的结果列与表/ORM 元数据进行匹配，用于包含有关要返回的结果行的完整信息的编译 SQL 构造。这样可以大大节省 Python 的开销，同时在将 ORM 和 Core SQL 表达式与结果行链接时提供更高的准确性。在 1.1 中，这种重新组织在内部进一步进行了，并且还通过最近添加的 `TextClause.columns()` 方法可用于纯文本 SQL 构造。

#### TextAsFrom.columns() 现在按位置工作

`TextClause.columns()`方法是在 0.9 版中添加的，接受基于列的参数位置；在 1.1 版中，当所有列被位置传递时，这些列与最终结果集的关联也将按位置执行。这里的关键优势在于，现在可以将文本 SQL 链接到 ORM 级别的结果集，而无需处理模糊或重复的列名，也无需将标签方案与 ORM 级别的标签方案进行匹配。现在所需要的只是在文本 SQL 中的列的顺序与传递给`TextClause.columns()`的列参数相同：

```py
from sqlalchemy import text

stmt = text(
    "SELECT users.id, addresses.id, users.id, "
    "users.name, addresses.email_address AS email "
    "FROM users JOIN addresses ON users.id=addresses.user_id "
    "WHERE users.id = 1"
).columns(User.id, Address.id, Address.user_id, User.name, Address.email_address)

query = session.query(User).from_statement(stmt).options(contains_eager(User.addresses))
result = query.all()
```

在上述文本 SQL 中，“id”列出现了三次，这通常会产生歧义。使用新功能，我们可以直接应用来自`User`和`Address`类的映射列，甚至可以将文本 SQL 中的`Address.user_id`列链接到`users.id`列，以供娱乐之用，而`Query`对象将接收到正确的可定位行，包括用于及早加载。

此更改与通过不同顺序将列传递给该方法的代码不兼容。希望由于这种方法一直以来都是以与文本 SQL 语句相同的顺序传递列而被记录的，这种影响将会很小，尽管内部并未检查此顺序。无论如何，该方法仅在 0.9 版中才被添加，并且可能尚未广泛使用。有关如何处理使用该方法的应用程序的行为更改的详细说明，请参阅 TextClause.columns() will match columns positionally, not by name, when passed positionally。

另请参阅

使用文本列表达式进行选择

> TextClause.columns() will match columns positionally, not by name, when passed positionally - 向后兼容性说明

#### 对于 Core/ORM SQL 构造，基于位置的匹配比基于名称的匹配更受信任

此更改的另一个方面是，匹配列的规则也已经修改，以更充分地依赖于编译后 SQL 结构中的“位置”匹配。考虑到以下语句：

```py
ua = users.alias("ua")
stmt = select([users.c.user_id, ua.c.user_id])
```

上述语句将编译为：

```py
SELECT  users.user_id,  ua.user_id  FROM  users,  users  AS  ua
```

在 1.0 版本中，上述语句在执行时将通过位置匹配与其原始编译结构相匹配，但由于语句中包含重复的 `'user_id'` 标签，因此“模糊列”规则仍然会介入并阻止从行中获取列。从 1.1 版开始，“模糊列”规则不会影响从列构造到 SQL 列的精确匹配，这是 ORM 用于获取列的方法：

```py
result = conn.execute(stmt)
row = result.first()

# these both match positionally, so no error
user_id = row[users.c.user_id]
ua_id = row[ua.c.user_id]

# this still raises, however
user_id = row["user_id"]
```

#### 几乎不太可能出现“模糊列”错误消息

作为这一更改的一部分，错误消息 `Ambiguous column name '<name>' in result set! try 'use_labels' option on select statement.` 的措辞已经有所减少；由于使用 ORM 或 Core 编译的 SQL 结构时，此消息现在应该极为罕见，因此它仅在检索使用实际上具有歧义的字符串名称的结果列时才会说明 `Ambiguous column name '<name>' in result set column descriptions`。它还现在引用了来自渲染的 SQL 语句本身的实际模糊名称，而不是指示用于获取的构造体的键或名称。

[#3501](https://www.sqlalchemy.org/trac/ticket/3501)

#### `TextAsFrom.columns()` 现在按位置工作

`TextClause.columns()` 方法在 0.9 版中添加了，它接受基于列的参数按位置排列；在 1.1 版中，当所有列按位置传递时，这些列与最终结果集的相关性也按位置执行。这里的关键优势是，现在可以将文本 SQL 链接到 ORM 级别的结果集，而无需处理模糊或重复的列名，也无需将标签方案与 ORM 级别的标签方案进行匹配。现在所需的是文本 SQL 中的列顺序与传递给 `TextClause.columns()` 的列参数的相同顺序：

```py
from sqlalchemy import text

stmt = text(
    "SELECT users.id, addresses.id, users.id, "
    "users.name, addresses.email_address AS email "
    "FROM users JOIN addresses ON users.id=addresses.user_id "
    "WHERE users.id = 1"
).columns(User.id, Address.id, Address.user_id, User.name, Address.email_address)

query = session.query(User).from_statement(stmt).options(contains_eager(User.addresses))
result = query.all()
```

在上述文本 SQL 中，“id” 列出现三次，这通常会是模糊的。使用新功能，我们可以直接应用来自 `User` 和 `Address` 类的映射列，甚至可以将 `Address.user_id` 列链接到文本 SQL 中的 `users.id` 列以供娱乐，并且 `Query` 对象将接收到正确可用的行，包括用于急加载。

这一变化与将列以与文本语句中的顺序不同的顺序传递给方法的代码不兼容。希望由于这个方法一直以来都是以与文本 SQL 语句相同的顺序传递列而被记录的，因此这种影响将会很小，即使内部没有检查这一点。无论如何，该方法仅在 0.9 版本中添加，并且可能尚未广泛使用。有关如何处理使用它的应用程序的这种行为变化的详细说明，请参阅 TextClause.columns() will match columns positionally, not by name, when passed positionally。

另请参阅

使用文本列表达式进行选择

> 当按位置传递时，TextClause.columns() 将按位置而不是按名称匹配列 - 向后兼容性说明

#### 对于核心/ORM SQL 构造，位置匹配比基于名称的匹配更可靠

这一变化的另一个方面是，匹配列的规则也已经修改，更充分地依赖“位置”匹配来编译 SQL 构造。给定如下语句：

```py
ua = users.alias("ua")
stmt = select([users.c.user_id, ua.c.user_id])
```

上述语句将编译为：

```py
SELECT  users.user_id,  ua.user_id  FROM  users,  users  AS  ua
```

在 1.0 版本中，上述语句在执行时将使用位置匹配与其原始编译构造相匹配，但是因为该语句包含了重复的 `'user_id'` 标签，所以“模糊列”规则仍然会介入并阻止从行中获取列。从 1.1 版本开始，“模糊列”规则不会影响从列构造到 SQL 列的精确匹配，这是 ORM 用于获取列的方法：

```py
result = conn.execute(stmt)
row = result.first()

# these both match positionally, so no error
user_id = row[users.c.user_id]
ua_id = row[ua.c.user_id]

# this still raises, however
user_id = row["user_id"]
```

#### 很少出现“模糊列”错误消息

作为此更改的一部分，错误消息 `Ambiguous column name '<name>' in result set! try 'use_labels' option on select statement.` 的措辞已经减弱；因为使用 ORM 或核心编译 SQL 构造时，此消息现在应该极为罕见，它只是简单地说明了 `Ambiguous column name '<name>' in result set column descriptions`，仅当通过实际模糊的字符串名称检索结果列时，例如在上面的示例中使用 `row['user_id']`。它现在还引用了来自渲染的 SQL 语句本身的实际模糊名称，而不是指示用于获取的构造的键或名称。

[#3501](https://www.sqlalchemy.org/trac/ticket/3501)

### 支持 Python 的本机 `enum` 类型和兼容形式

`Enum` 类型现在可以使用任何符合 PEP-435 的枚举类型来构造。在使用此模式时，输入值和返回值是实际的枚举对象，而不是字符串/整数等值：

```py
import enum
from sqlalchemy import Table, MetaData, Column, Enum, create_engine

class MyEnum(enum.Enum):
    one = 1
    two = 2
    three = 3

t = Table("data", MetaData(), Column("value", Enum(MyEnum)))

e = create_engine("sqlite://")
t.create(e)

e.execute(t.insert(), {"value": MyEnum.two})
assert e.scalar(t.select()) is MyEnum.two
```

#### `Enum.enums` 集合现在是列表而不是元组

作为对`Enum`的更改的一部分，`Enum.enums`元素集合现在是一个列表而不是元组。这是因为列表适用于元素的长度可变的同质项序列，其中元素的位置在语义上不重要。

[#3292](https://www.sqlalchemy.org/trac/ticket/3292)

#### `Enum.enums`集合现在是一个列表而不是元组

作为对`Enum`的更改的一部分，`Enum.enums`元素集合现在是一个列表而不是元组。这是因为列表适用于元素的长度可变的同质项序列，其中元素的位置在语义上不重要。

[#3292](https://www.sqlalchemy.org/trac/ticket/3292)

### Core 结果行容纳负整数索引

`RowProxy`对象现在像常规 Python 序列一样容纳单个负整数索引，无论是在纯 Python 版本还是 C 扩展版本中。以前，负值只能在切片中起作用：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://")
>>> row = e.execute("select 1, 2, 3").first()
>>> row[-1], row[-2], row[1], row[-2:2]
3 2 2 (2,)
```

### `Enum`类型现在对值进行 Python 内部验证

为了适应 Python 本机枚举对象，以及诸如在数组中使用非本地 ENUM 类型且 CHECK 约束不可行等边缘情况，当使用`Enum.validate_strings`标志时，`Enum`数据类型现在对输入值进行 Python 内部验证（1.1.0b2）:

```py
>>> from sqlalchemy import Table, MetaData, Column, Enum, create_engine
>>> t = Table(
...     "data",
...     MetaData(),
...     Column("value", Enum("one", "two", "three", validate_strings=True)),
... )
>>> e = create_engine("sqlite://")
>>> t.create(e)
>>> e.execute(t.insert(), {"value": "four"})
Traceback (most recent call last):
  ...
sqlalchemy.exc.StatementError: (exceptions.LookupError)
"four" is not among the defined enum values
[SQL: u'INSERT INTO data (value) VALUES (?)']
[parameters: [{'value': 'four'}]]
```

此验证默认关闭，因为已经确定存在用户不希望进行此类验证的用例（例如字符串比较）。对于非字符串类型，它在所有情况下都必须进行。当从数据库返回值时，检查也会无条件地发生在结果处理方面。

此验证是在使用非本地枚举类型时创建 CHECK 约束的现有行为之外的。现在可以使用新的`Enum.create_constraint`标志来禁用此 CHECK 约束的创建。

[#3095](https://www.sqlalchemy.org/trac/ticket/3095)

### 所有情况下将非本地布尔整数值强制转换为零/一/None

`Boolean` 数据类型将 Python 布尔值强制转换为整数值，以用于没有本地布尔类型的后端，例如 SQLite 和 MySQL。在这些后端上，通常设置一个 CHECK 约束，以确保数据库中的值实际上是这两个值之一。但是，MySQL 忽略 CHECK 约束，该约束是可选的，并且现有数据库可能没有此约束。已修复`Boolean` 数据类型，使得已经是整数值的 Python 端值被强制转换为零或一，而不仅仅是传递原样；此外，结果的 C 扩展版本的整数到布尔处理器现在使用与 Python 布尔值解释相同的值，而不是断言确切的一或零值。这现在与纯 Python 的整数到布尔处理器一致，并且更容忍数据库中已有的数据。值为 None/NULL 仍然保留为 None/NULL。

注意

此更改导致了一个意外的副作用，即非整数值（如字符串）的解释也发生了变化，使得字符串值`"0"`被解释为“true”，但仅在没有本地布尔数据类型的后端上 - 在像 PostgreSQL 这样的“本地布尔”后端上，字符串值`"0"`直接传递给驱动程序，并被解释为“false”。这是以前的实现中没有发生的不一致性。应注意，将字符串或任何其他值传递给`Boolean` 数据类型之外的`None`、`True`、`False`、`1`、`0`是**不受支持**的，版本 1.2 将对此场景引发错误（或可能只是发出警告，待定）。另请参阅[#4102](https://www.sqlalchemy.org/trac/ticket/4102)。

[#3730](https://www.sqlalchemy.org/trac/ticket/3730)

### 在日志和异常显示中现在截断大的��数和行值

作为 SQL 语句的绑定参数以及结果行中存在的大值现在在日志记录、异常报告以及行本身的`repr()`中显示时将被截断：

```py
>>> from sqlalchemy import create_engine
>>> import random
>>> e = create_engine("sqlite://", echo="debug")
>>> some_value = "".join(chr(random.randint(52, 85)) for i in range(5000))
>>> row = e.execute("select ?", [some_value]).first()
... # (lines are wrapped for clarity) ...
2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine select ?
2016-02-17 13:23:03,027 INFO sqlalchemy.engine.base.Engine
('E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;NM6GU
LUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7>4=4:P
GJ7HQ6 ... (4702 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=RJP
HDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
K:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine Col ('?',)
2016-02-17 13:23:03,027 DEBUG sqlalchemy.engine.base.Engine
Row (u'E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;
NM6GULUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7
>4=4:PGJ7HQ ... (4703 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;=
RJPHDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9HM
K:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
>>> print(row)
(u'E6@?>9HPOJB<<BHR:@=TS:5ILU=;JLM<4?B9<S48PTNG9>:=TSTLA;9K;9FPM4M8M@;NM6
GULUAEBT9QGHNHTHR5EP75@OER4?SKC;D:TFUMD:M>;C6U:JLM6R67GEK<A6@S@C@J7>4
=4:PGJ7HQ ... (4703 characters truncated) ... J6IK546AJMB4N6S9L;;9AKI;
=RJPHDSSOTNBUEEC9@Q:RCL:I@5?FO<9K>KJAGAO@E6@A7JI8O:J7B69T6<8;F:S;4BEIJS9H
MK:;5OLPM@JR;R:J6<SOTTT=>Q>7T@I::OTDC:CC<=NGP6C>BC8N',)
```

[#2837](https://www.sqlalchemy.org/trac/ticket/2837)

### 核心添加了 JSON 支持

由于 MySQL 现在除了 PostgreSQL JSON 数据类型外还有 JSON 数据类型，核心现在获得了一个`sqlalchemy.types.JSON` 数据类型，这是这两种数据类型的基础。使用这种类型可以在 PostgreSQL 和 MySQL 之间以一种不可知的方式访问“getitem”运算符和“getpath”运算符。

新数据类型还对 NULL 值的处理以及表达式处理进行了一系列改进。

另请参阅

MySQL JSON 支持

`JSON`

`JSON`

`JSON`

[#3619](https://www.sqlalchemy.org/trac/ticket/3619)

#### 与 ORM 操作一起插入 JSON“null”时，如预期那样插入，当不存在时则被省略

`JSON`类型及其后代类型`JSON`和`JSON`有一个标志`JSON.none_as_null`，当设置为 True 时，表示 Python 值`None`应该转换为 SQL NULL 而不是 JSON NULL 值。此标志默认为 False，这意味着 Python 值`None`应该导致 JSON NULL 值。

在以下情况下，此逻辑将失败，并已得到纠正：

1\. 当列还包含默认值或服务器默认值时，对于期望持久化 JSON“null”的映射属性上的`None`正值仍会触发列级默认值，替换`None`值：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), default="some default")

# would insert "some default" instead of "'null'",
# now will insert "'null'"
obj = MyObject(json_value=None)
session.add(obj)
session.commit()
```

2\. 当列*没有*包含默认值或服务器默认值时，配置为 none_as_null=False 的 JSON 列上的缺失值仍会呈现 JSON NULL，而不是回退到不插入任何值，与所有其他数据类型的行为不一致：

```py
class MyObject(Base):
    # ...

    some_other_value = Column(String(50))
    json_value = Column(JSON(none_as_null=False))

# would result in NULL for some_other_value,
# but json "'null'" for json_value.  Now results in NULL for both
# (the json_value is omitted from the INSERT)
obj = MyObject()
session.add(obj)
session.commit()
```

这是一个行为变更，对于依赖此默认缺失值为 JSON null 的应用程序来说是不兼容的。这基本上确立了**缺失值与存在的 None 值有所区别**。有关更多详细信息，请参见 JSON 列如果未提供任何值且未建立默认值，则不会插入 JSON NULL。

3\. 当使用`Session.bulk_insert_mappings()`方法时，在所有情况下都会忽略`None`：

```py
# would insert SQL NULL and/or trigger defaults,
# now inserts "'null'"
session.bulk_insert_mappings(MyObject, [{"json_value": None}])
```

`JSON`类型现在实现了`TypeEngine.should_evaluate_none`标志，指示此处不应忽略`None`；它会根据`JSON.none_as_null`的值自动配置。感谢[#3061](https://www.sqlalchemy.org/trac/ticket/3061)，我们可以区分用户主动设置的`None`值与根本未设置的情况。

该功能同样适用于新的基础 `JSON` 类型及其派生类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)  #### 新增了 JSON.NULL 常量

为确保应用程序始终可以完全控制 `JSON`、`JSON`、`JSON` 或 `JSONB` 列在值级别是否应接收 SQL NULL 或 JSON `"null"` 值，已添加了常量 `JSON.NULL`，它与 `null()` 结合使用，可以完全确定 SQL NULL 和 JSON `"null"` 之间的区别，而不受 `JSON.none_as_null` 的设置影响：

```py
from sqlalchemy import null
from sqlalchemy.dialects.postgresql import JSON

obj1 = MyObject(json_value=null())  # will *always* insert SQL NULL
obj2 = MyObject(json_value=JSON.NULL)  # will *always* insert JSON string "null"

session.add_all([obj1, obj2])
session.commit()
```

该功能同样适用于新的基础 `JSON` 类型及其派生类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)  #### 在 ORM 操作中插入 JSON “null” 时会被预期地插入，当未出现时会被省略

`JSON` 类型及其派生类型 `JSON` 和 `JSON` 具有一个标志 `JSON.none_as_null`，当设置为 True 时，表示 Python 值 `None` 应转换为 SQL NULL 而不是 JSON NULL 值。该标志默认为 False，这意味着 Python 值 `None` 应导致 JSON NULL 值。

在以下情况下，此逻辑将失败，并已纠正：

1\. 当列还包含默认值或 server_default 值时，在期望持久化 JSON “null”的映射属性上的正值 `None` 仍会触发列级默认值，替换 `None` 值：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), default="some default")

# would insert "some default" instead of "'null'",
# now will insert "'null'"
obj = MyObject(json_value=None)
session.add(obj)
session.commit()
```

2\. 当列*不*包含默认值或 server_default 值时，针对配置了 none_as_null=False 的 JSON 列上的缺失值仍会呈现 JSON NULL 而不是回退到不插入任何值，与所有其他数据类型的行为不一致：

```py
class MyObject(Base):
    # ...

    some_other_value = Column(String(50))
    json_value = Column(JSON(none_as_null=False))

# would result in NULL for some_other_value,
# but json "'null'" for json_value.  Now results in NULL for both
# (the json_value is omitted from the INSERT)
obj = MyObject()
session.add(obj)
session.commit()
```

这是一个行为变更，对于依赖默认将缺失值设为 JSON null 的应用程序来说，这是不兼容的。这实际上确立了**缺失值与存在的 None 值有所区别**。详细信息请参见 如果未提供值且未设置默认值，则 JSON 列将不插入 JSON NULL。

3\. 当使用 `Session.bulk_insert_mappings()` 方法时，`None` 在所有情况下都会被忽略：

```py
# would insert SQL NULL and/or trigger defaults,
# now inserts "'null'"
session.bulk_insert_mappings(MyObject, [{"json_value": None}])
```

`JSON` 类型现在实现了 `TypeEngine.should_evaluate_none` 标志，指示此处不应忽略 `None`；它会根据 `JSON.none_as_null` 的值自动配置。感谢 [#3061](https://www.sqlalchemy.org/trac/ticket/3061)，我们可以区分用户主动设置的值 `None` 与根本未设置的值。

该功能同样适用于新的基础 `JSON` 类型及其派生类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

#### 新增 JSON.NULL 常量

为了确保应用程序始终可以在值级别上完全控制 `JSON`、`JSON`、`JSON` 或 `JSONB` 列是否应接收 SQL NULL 或 JSON `"null"` 值，已添加常量 `JSON.NULL`，它与 `null()` 结合使用，可以完全确定 SQL NULL 和 JSON `"null"` 之间的区别，而不受 `JSON.none_as_null` 的设置影响：

```py
from sqlalchemy import null
from sqlalchemy.dialects.postgresql import JSON

obj1 = MyObject(json_value=null())  # will *always* insert SQL NULL
obj2 = MyObject(json_value=JSON.NULL)  # will *always* insert JSON string "null"

session.add_all([obj1, obj2])
session.commit()
```

该功能同样适用于新的基础 `JSON` 类型及其派生类型。

[#3514](https://www.sqlalchemy.org/trac/ticket/3514)

### Core 添加了数组支持；新增 ANY 和 ALL 运算符

除了在 Correct SQL Types are Established from Indexed Access of ARRAY, JSON, HSTORE 中描述的 PostgreSQL `ARRAY` 类型的增强功能外，`ARRAY` 的基类本身已经移动到核心中的一个新类 `ARRAY` 中。

数组是 SQL 标准的一部分，还有一些面向数组的函数，如 `array_agg()` 和 `unnest()`。为了支持这些构造，不仅仅是针对 PostgreSQL，还有可能是将来其他支持数组的后端，如 DB2，现在大部分 SQL 表达式的数组逻辑都在核心中。`ARRAY` 类型仍然**只在 PostgreSQL 上工作**，但可以直接使用，支持特殊的数组用例，如索引访问，以及对 ANY 和 ALL 的支持：

```py
mytable = Table("mytable", metadata, Column("data", ARRAY(Integer, dimensions=2)))

expr = mytable.c.data[5][6]

expr = mytable.c.data[5].any(12)
```

为了支持 ANY 和 ALL，`ARRAY` 类型保留了与 PostgreSQL 类型相同的 `Comparator.any()` 和 `Comparator.all()` 方法，但也将这些操作导出到新的独立运算符函数 `any_()` 和 `all_()` 中。这两个函数以更传统的 SQL 方式工作，允许右侧表达式形式，如：

```py
from sqlalchemy import any_, all_

select([mytable]).where(12 == any_(mytable.c.data[5]))
```

对于 PostgreSQL 特定的运算符“contains”、“contained_by” 和 “overlaps”，应继续直接使用 `ARRAY` 类型，该类型还提供了 `ARRAY` 类型的所有功能。

`any_()` 和 `all_()` 运算符在核心层面是开放的，但是后端数据库对它们的解释是有限的。在 PostgreSQL 后端，这两个运算符**只接受数组值**。而在 MySQL 后端，它们**只接受子查询值**。在 MySQL 中，可以使用如下表达式：

```py
from sqlalchemy import any_, all_

subq = select([mytable.c.value])
select([mytable]).where(12 > any_(subq))
```

[#3516](https://www.sqlalchemy.org/trac/ticket/3516)

### 新功能特性���“WITHIN GROUP”，array_agg 和 set 聚合函数

使用新的 `ARRAY` 类型，我们还可以实现一个预先类型化的函数，用于返回一个数组的 `array_agg()` SQL 函数，现在可以使用 `array_agg` 进行调用：

```py
from sqlalchemy import func

stmt = select([func.array_agg(table.c.value)])
```

还添加了一个 PostgreSQL 元素，用于聚合 ORDER BY，通过`aggregate_order_by`：

```py
from sqlalchemy.dialects.postgresql import aggregate_order_by

expr = func.array_agg(aggregate_order_by(table.c.a, table.c.b.desc()))
stmt = select([expr])
```

生成：

```py
SELECT  array_agg(table1.a  ORDER  BY  table1.b  DESC)  AS  array_agg_1  FROM  table1
```

PG 方言本身还提供了一个 `array_agg()` 包装器，以确保 `ARRAY` 类型：

```py
from sqlalchemy.dialects.postgresql import array_agg

stmt = select([array_agg(table.c.value).contains("foo")])
```

另外，像`percentile_cont()`、`percentile_disc()`、`rank()`、`dense_rank()`等函数，需要通过`WITHIN GROUP (ORDER BY <expr>)`进行排序，现在可以通过`FunctionElement.within_group()`修饰符进行使用：

```py
from sqlalchemy import func

stmt = select(
    [
        department.c.id,
        func.percentile_cont(0.5).within_group(department.c.salary.desc()),
    ]
)
```

上述语句将生成类似于以下的 SQL：

```py
SELECT  department.id,  percentile_cont(0.5)
WITHIN  GROUP  (ORDER  BY  department.salary  DESC)
```

现在为这些函数提供了正确返回类型的占位符，包括 `percentile_cont`、`percentile_disc`、`rank`、`dense_rank`、`mode`、`percent_rank` 和 `cume_dist`。

[#3132](https://www.sqlalchemy.org/trac/ticket/3132) [#1370](https://www.sqlalchemy.org/trac/ticket/1370)

### TypeDecorator 现在自动与 Enum、Boolean、“schema” 类型一起工作

`SchemaType` 类型包括诸如`Enum` 和`Boolean` 这样的类型，除了对应于数据库类型外，还会生成一个 CHECK 约束或在 PostgreSQL ENUM 的情况下生成一个新的 CREATE TYPE 语句，现在将自动与`TypeDecorator` 配方一起工作。以前，`ENUM` 的`TypeDecorator` 必须像这样：

```py
# old way
class MyEnum(TypeDecorator, SchemaType):
    impl = postgresql.ENUM("one", "two", "three", name="myenum")

    def _set_table(self, table):
        self.impl._set_table(table)
```

`TypeDecorator` 现在传播这些额外的事件，因此可以像任何其他类型一样完成：

```py
# new way
class MyEnum(TypeDecorator):
    impl = postgresql.ENUM("one", "two", "three", name="myenum")
```

[#2919](https://www.sqlalchemy.org/trac/ticket/2919)

### 用于表对象的多租户模式翻译

为了支持一个应用程序使用许多模式中相同的`Table`对象的用例，例如每个用户一个模式，添加了一个新的执行选项`Connection.execution_options.schema_translate_map`。使用此映射，一组`Table`对象可以在每个连接基础上被制作，以引用任何一组模式，而不是它们被分配到的`Table.schema`。翻译适用于 DDL 和 SQL 生成，以及 ORM。

例如，如果 `User` 类被分配到模式“per_user”：

```py
class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True)

    __table_args__ = {"schema": "per_user"}
```

在每个请求上，`Session` 可以设置为每次引用不同的模式：

```py
session = Session()
session.connection(
    execution_options={"schema_translate_map": {"per_user": "account_one"}}
)

# will query from the ``account_one.user`` table
session.query(User).get(5)
```

另请参阅

模式名称的翻译

[#2685](https://www.sqlalchemy.org/trac/ticket/2685)

### “友好”地将 Core SQL 构造字符串化而不使用方言

对 Core SQL 构造调用 `str()` 现在会在更多情况下生成字符串，支持各种通常不在默认 SQL 中出现的 SQL 构造，如 RETURNING、数组索引和非标准数据类型：

```py
>>> from sqlalchemy import table, column
t>>> t = table('x', column('a'), column('b'))
>>> print(t.insert().returning(t.c.a, t.c.b))
INSERT  INTO  x  (a,  b)  VALUES  (:a,  :b)  RETURNING  x.a,  x.b 
```

`str()` 函数现在调用一个完全独立的方言/编译器，仅用于普通字符串打印而没有设置特定方言，因此随着更多“只是显示给我一个字符串！”的情况出现，这些可以添加到此方言/编译器中，而不会影响真实方言上的行为。

另请参阅

查询的字符串化将询问会话以获取正确的方言

[#3631](https://www.sqlalchemy.org/trac/ticket/3631)

### type_coerce 函数现在是一个持久的 SQL 元素

`type_coerce()` 函数之前会返回一个类型为 `BindParameter` 或 `Label` 的对象，取决于输入。这样做的效果是，如果使用了表达式转换，例如将元素从 `Column` 转换为 `BindParameter` 的过程对 ORM 级别的延迟加载至关重要，那么类型强制信息将不会被使用，因为它已经丢失了。

为了改进这种行为，该函数现在返回一个持久的 `TypeCoerce` 容器，该容器围绕给定表达式自身保持不受影响；此构造由 SQL 编译器显式评估。这允许内部表达式的强制转换得以保持，无论语句如何修改，包括如果所包含的元素被替换为不同的元素，这在 ORM 的延迟加载功能中是常见的。

展示效果的测试案例利用了异构的 primaryjoin 条件与自定义类型和延迟加载。给定一个将 CAST 应用为“绑定表达式”的自定义类型：

```py
class StringAsInt(TypeDecorator):
    impl = String

    def column_expression(self, col):
        return cast(col, Integer)

    def bind_expression(self, value):
        return cast(value, String)
```

接着，一个映射，我们将一个表上的字符串 “id” 列与另一个表上的整数 “id” 列进行等同：

```py
class Person(Base):
    __tablename__ = "person"
    id = Column(StringAsInt, primary_key=True)

    pets = relationship(
        "Pets",
        primaryjoin=(
            "foreign(Pets.person_id)==cast(type_coerce(Person.id, Integer), Integer)"
        ),
    )

class Pets(Base):
    __tablename__ = "pets"
    id = Column("id", Integer, primary_key=True)
    person_id = Column("person_id", Integer)
```

在 `relationship.primaryjoin` 表达式中，我们使用 `type_coerce()` 处理通过延迟加载传递的绑定参数，因为我们已经知道这些参数将来自于我们的 `StringAsInt` 类型，该类型在 Python 中将值维护为整数。然后，我们使用 `cast()` ，以便作为 SQL 表达式，VARCHAR “id” 列将在常规的非转换连接中被 CAST 为整数，如 `Query.join()` 或 `joinedload()`。也就是说，`.pets` 的 joinedload 看起来像这样：

```py
SELECT  person.id  AS  person_id,  pets_1.id  AS  pets_1_id,
  pets_1.person_id  AS  pets_1_person_id
FROM  person
LEFT  OUTER  JOIN  pets  AS  pets_1
ON  pets_1.person_id  =  CAST(person.id  AS  INTEGER)
```

在连接的 ON 子句中没有 CAST，像 PostgreSQL 这样的强类型数据库将拒绝隐式比较整数并失败。

`.pets`的 lazyload 情况依赖于在加载时用绑定参数替换`Person.id`列，该参数接收 Python 加载的值。这种替换特别是我们的`type_coerce()`函数意图会丢失的地方。在更改之前，这种延迟加载如下：

```py
SELECT  pets.id  AS  pets_id,  pets.person_id  AS  pets_person_id
FROM  pets
WHERE  pets.person_id  =  CAST(CAST(%(param_1)s  AS  VARCHAR)  AS  INTEGER)
-- {'param_1': 5}
```

在上面的例子中，我们看到我们在 Python 中的值`5`首先被 CAST 为 VARCHAR，然后在 SQL 中再次被 CAST 为 INTEGER；这是一个双重的 CAST，虽然有效，但并不是我们要求的。

随着变化，`type_coerce()`函数在列被替换为绑定参数后仍保持包装，查询现在如下所示：

```py
SELECT  pets.id  AS  pets_id,  pets.person_id  AS  pets_person_id
FROM  pets
WHERE  pets.person_id  =  CAST(%(param_1)s  AS  INTEGER)
-- {'param_1': 5}
```

我们的外部 CAST 仍然起作用，但是`StringAsInt`自定义类型中不必要的 CAST 已被`type_coerce()`函数按照意图移除。

[#3531](https://www.sqlalchemy.org/trac/ticket/3531)

## 关键行为变化 - ORM

### 如果没有提供值且没有建立默认值，JSON 列将不会插入 JSON NULL

如 JSON “null” is inserted as expected with ORM operations, omitted when not present 中详细说明，如果值完全缺失，`JSON`将不会呈现 JSON“null”值。为了防止 SQL NULL，应该设置一个默认值。给定以下映射：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), nullable=False)
```

以下刷新操作将由于完整性错误而失败：

```py
obj = MyObject()  # note no json_value
session.add(obj)
session.commit()  # will fail with integrity error
```

如果列的默认值应为 JSON NULL，请在列上设置此值：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), nullable=False, default=JSON.NULL)
```

或者，确保对象上存在该值：

```py
obj = MyObject(json_value=None)
session.add(obj)
session.commit()  # will insert JSON NULL
```

请注意，为默认值设置`None`与完全省略它相同；`JSON.none_as_null`标志不影响传递给`Column.default`或`Column.server_default`的`None`值：

```py
# default=None is the same as omitting it entirely, does not apply JSON NULL
json_value = Column(JSON(none_as_null=False), nullable=False, default=None)
```

另请参见

JSON “null” is inserted as expected with ORM operations, omitted when not present  ### 列不再通过 DISTINCT + ORDER BY 冗余添加

现在，类似以下的查询将仅增加那些在 SELECT 列表中缺失的列，而不会重复：

```py
q = (
    session.query(User.id, User.name.label("name"))
    .distinct()
    .order_by(User.id, User.name, User.fullname)
)
```

产生：

```py
SELECT  DISTINCT  user.id  AS  a_id,  user.name  AS  name,
  user.fullname  AS  a_fullname
FROM  a  ORDER  BY  user.id,  user.name,  user.fullname
```

以前，它会产生：

```py
SELECT  DISTINCT  user.id  AS  a_id,  user.name  AS  name,  user.name  AS  a_name,
  user.fullname  AS  a_fullname
FROM  a  ORDER  BY  user.id,  user.name,  user.fullname
```

在上面的例子中，`user.name` 列被不必要地添加。结果不会受影响，因为额外的列在任何情况下都不包含在结果中，但这些列是不必要的。

此外，当通过向 `Query.distinct()` 传递表达式来使用 PostgreSQL DISTINCT ON 格式时，上述“添加列”逻辑将被完全禁用。

当查询被捆绑到子查询中以实现连接的急加载时，"增强列列表"规则必须更加积极，以便仍然可以满足 ORDER BY，因此此情况保持不变。

[#3641](https://www.sqlalchemy.org/trac/ticket/3641)  ### 相同名称的 @validates 装饰器现在将引发异常

`validates()` 装饰器只打算为特定属性名称的每个类创建一次。创建多个现在会引发错误，而以前会悄悄地选择最后定义的验证器：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    data = Column(String)

    @validates("data")
    def _validate_data_one(self):
        assert "x" in data

    @validates("data")
    def _validate_data_two(self):
        assert "y" in data

configure_mappers()
```

将引发：

```py
sqlalchemy.exc.InvalidRequestError: A validation function for mapped attribute 'data'
on mapper Mapper|A|a already exists.
```

[#3776](https://www.sqlalchemy.org/trac/ticket/3776)  ### 如果未提供值且未建立默认值，则 JSON 列将不插入 JSON NULL

如 JSON “null” 在 ORM 操作中如预期地插入，当不存在时被省略 中详细说明的，`JSON` 如果完全缺少值，则不会呈现 JSON “null” 值。为了防止 SQL NULL，应设置默认值。给定以下映射：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), nullable=False)
```

以下刷新操作将因完整性错误而失败：

```py
obj = MyObject()  # note no json_value
session.add(obj)
session.commit()  # will fail with integrity error
```

如果列的默认值应为 JSON NULL，请在 Column 上设置此值：

```py
class MyObject(Base):
    # ...

    json_value = Column(JSON(none_as_null=False), nullable=False, default=JSON.NULL)
```

或者，确保对象上存在该值：

```py
obj = MyObject(json_value=None)
session.add(obj)
session.commit()  # will insert JSON NULL
```

请注意，为默认设置 `None` 与完全省略它相同；`JSON.none_as_null` 标志不影响传递给 `Column.default` 或 `Column.server_default` 的 `None` 的值：

```py
# default=None is the same as omitting it entirely, does not apply JSON NULL
json_value = Column(JSON(none_as_null=False), nullable=False, default=None)
```

另请参阅

JSON “null” 在 ORM 操作中如预期地插入，当不存在时被省略

### 使用 DISTINCT + ORDER BY 不再冗余添加列

以下查询现在只会增补那些在 SELECT 列表中缺失的列，而不会重复：

```py
q = (
    session.query(User.id, User.name.label("name"))
    .distinct()
    .order_by(User.id, User.name, User.fullname)
)
```

产生：

```py
SELECT  DISTINCT  user.id  AS  a_id,  user.name  AS  name,
  user.fullname  AS  a_fullname
FROM  a  ORDER  BY  user.id,  user.name,  user.fullname
```

以前，它会产生：

```py
SELECT  DISTINCT  user.id  AS  a_id,  user.name  AS  name,  user.name  AS  a_name,
  user.fullname  AS  a_fullname
FROM  a  ORDER  BY  user.id,  user.name,  user.fullname
```

在上面的例子中，`user.name` 列被不必要地添加。结果不会受影响，因为额外的列在任何情况下都不包含在结果中，但这些列是不必要的。

此外，当通过将表达式传递给`Query.distinct()`来使用 PostgreSQL DISTINCT ON 格式时，上述“添加列”逻辑将完全禁用。

当查询被捆绑到子查询中以进行连接式贪婪加载时，“增补列列表”规则必须更加积极，以便仍然可以满足 ORDER BY，因此这种情况保持不变。

[#3641](https://www.sqlalchemy.org/trac/ticket/3641)

### 相同名称的@validates 装饰器现在将引发异常

`validates()`装饰器只打算为特定属性名称的类创建一次。现在创建多个会引发错误，而以前它会悄悄地选择最后定义的验证器：

```py
class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)

    data = Column(String)

    @validates("data")
    def _validate_data_one(self):
        assert "x" in data

    @validates("data")
    def _validate_data_two(self):
        assert "y" in data

configure_mappers()
```

将引发：

```py
sqlalchemy.exc.InvalidRequestError: A validation function for mapped attribute 'data'
on mapper Mapper|A|a already exists.
```

[#3776](https://www.sqlalchemy.org/trac/ticket/3776)

## 核心的关键行为变化

### TextClause.columns()将按位置匹配列，而不是按名称匹配

`TextClause.columns()`方法的新行为，它本身是在 0.9 系列中最近添加的，是当列按位置传递而没有任何额外的关键字参数时，它们与最终结果集的列按位置链接，而不再按名称。希望这种变化的影响会很小，因为该方法一直以来都有文档说明传递的列与文本 SQL 语句的顺序相同，这似乎是直观的，即使内部没有检查这一点。

通过将`Column`对象按位置传递给该方法的应用程序必须确保这些`Column`对象的位置与这些列在文本 SQL 中声明的位置相匹配。

例如，像下面这样的代码：

```py
stmt = text("SELECT id, name, description FROM table")

# no longer matches by name
stmt = stmt.columns(my_table.c.name, my_table.c.description, my_table.c.id)
```

现在不再按预期工作；给定列的顺序现在很重要：

```py
# correct version
stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)
```

可能更有可能的是，像这样工作的语句：

```py
stmt = text("SELECT * FROM table")
stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)
```

现在略有风险，因为“*”规范通常会按照表本身中的顺序提供列。如果表的结构因模式更改而更改，则此顺序可能不再相同。因此，在使用`TextClause.columns()`时，建议在文本 SQL 中明确列出所需的列，尽管在文本 SQL 中不再需要担心列名本身。

另请参阅

ResultSet column matching enhancements; positional column setup for textual SQL  ### 字符串 server_default 现在以文本引号引用

作为纯 Python 字符串传递给 `Column.server_default` 的服务器默认值现在通过文本引用系统传递：

```py
>>> from sqlalchemy.schema import MetaData, Table, Column, CreateTable
>>> from sqlalchemy.types import String
>>> t = Table("t", MetaData(), Column("x", String(), server_default="hi ' there"))
>>> print(CreateTable(t))
CREATE  TABLE  t  (
  x  VARCHAR  DEFAULT  'hi '' there'
) 
```

以前的引用会直接渲染。对于存在此类用例并且正在解决此问题的应用程序，此更改可能不兼容。

[#3809](https://www.sqlalchemy.org/trac/ticket/3809)  ### 具有 LIMIT/OFFSET/ORDER BY 的 SELECT 的 UNION 或类似结构现在会对嵌入的 SELECT 进行括号化

一个问题，就像其他问题一样，长期以来受 SQLite 缺乏功能的驱动，现在已经增强以在所有支持的后端上工作。我们指的是一个查询，它是 SELECT 语句的 UNION，这些语句本身包含了包含 LIMIT、OFFSET 和/或 ORDER BY 的行限制或排序功能：

```py
(SELECT  x  FROM  table1  ORDER  BY  y  LIMIT  1)  UNION
(SELECT  x  FROM  table2  ORDER  BY  y  LIMIT  2)
```

上述查询需要在每个子查询中加上括号，以便正确地对子结果进行分组。在 SQLAlchemy 核心中生成上述语句的形式如下：

```py
stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1)
stmt2 = select([table1.c.x]).order_by(table2.c.y).limit(2)

stmt = union(stmt1, stmt2)
```

以前，上述结构不会为内部 SELECT 语句产生括号化，从而产生一个在所有后端上都失败的查询。

上述格式在 SQLite 上**仍然会失败**；此外，包含 ORDER BY 但不包含 LIMIT/SELECT 的格式在 Oracle 上**仍然会失败**。这不是一个向后不兼容的更改，因为查询如果没有括号也会失败；有了修复，查询至少在所有其他数据库上可以工作。

在所有情况下，为了产生一个在 SQLite 上和在所有情况下在 Oracle 上都有效的有限 SELECT 语句的 UNION，子查询必须是一个 ALIAS 的 SELECT：

```py
stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1).alias().select()
stmt2 = select([table2.c.x]).order_by(table2.c.y).limit(2).alias().select()

stmt = union(stmt1, stmt2)
```

这种解决方法适用于所有 SQLAlchemy 版本。在 ORM 中，它的形式如下：

```py
stmt1 = session.query(Model1).order_by(Model1.y).limit(1).subquery().select()
stmt2 = session.query(Model2).order_by(Model2.y).limit(1).subquery().select()

stmt = session.query(Model1).from_statement(stmt1.union(stmt2))
```

这里的行为与 SQLAlchemy 0.9 中引入的“join 重写”行为有许多相似之处许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在 (SELECT * FROM ..) AS ANON_1 中；然而，在这种情况下，我们选择不添加新的重写行为以适应 SQLite 的情况。现有的重写行为已经非常复杂了，而具有带括号的 SELECT 语句的 UNION 的情况比该功能的“右嵌套连接”用例要少得多。

[#2528](https://www.sqlalchemy.org/trac/ticket/2528)  ### `TextClause.columns()` 将按位置匹配列，而不是按名称匹配

`TextClause.columns()` 方法的新行为，它本身是最近添加的 0.9 系列的一部分，是当列被以位置传递且没有任何额外的关键字参数时，它们链接到最终结果集的列的位置，而不再是按名称。希望由于该方法始终被记录为说明列按照文本 SQL 语句的相同顺序传递，这个更改的影响会很小，尽管内部并没有检查这一点。

使用这种方法的应用程序通过按位置传递 `Column` 对象来确保这些 `Column` 对象的位置与文本 SQL 中这些列的位置相匹配。

例如，像下面的代码：

```py
stmt = text("SELECT id, name, description FROM table")

# no longer matches by name
stmt = stmt.columns(my_table.c.name, my_table.c.description, my_table.c.id)
```

将不再按预期工作；给出的列的顺序现在很重要：

```py
# correct version
stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)
```

更有可能的是，像这样工作的语句：

```py
stmt = text("SELECT * FROM table")
stmt = stmt.columns(my_table.c.id, my_table.c.name, my_table.c.description)
```

现在稍微有风险，因为“*”规范通常会按照它们在表本身中出现的顺序传送列。如果表的结构因模式更改而更改，则此顺序可能不再相同。因此，在使用 `TextClause.columns()` 时，建议在文本 SQL 中明确列出所需的列，尽管在文本 SQL 中不再需要担心列名本身。

另见

ResultSet 列匹配增强；文本 SQL 的位置列设置

### 字符串 server_default 现在是字面引用

传递给 `Column.server_default` 的服务器默认值，作为一个带有引号的普通 Python 字符串，现在通过字面引用系统传递：

```py
>>> from sqlalchemy.schema import MetaData, Table, Column, CreateTable
>>> from sqlalchemy.types import String
>>> t = Table("t", MetaData(), Column("x", String(), server_default="hi ' there"))
>>> print(CreateTable(t))
CREATE  TABLE  t  (
  x  VARCHAR  DEFAULT  'hi '' there'
) 
```

以前引号会直接呈现。此更改对于具有这种用例并围绕此问题进行工作的应用程序可能不兼容。

[#3809](https://www.sqlalchemy.org/trac/ticket/3809)

### UNION 或类似 SELECT 的带有 LIMIT/OFFSET/ORDER BY 的现在括号化嵌入式 SELECTs

像其他问题一样，长期受 SQLite 能力不足的驱动，现在已经增强以在所有支持的后端上工作。我们指的是一个查询，它是 SELECT 语句的 UNION，这些语句本身包含行限制或排序功能，包括 LIMIT、OFFSET 和/或 ORDER BY：

```py
(SELECT  x  FROM  table1  ORDER  BY  y  LIMIT  1)  UNION
(SELECT  x  FROM  table2  ORDER  BY  y  LIMIT  2)
```

上述查询要求每个子选择中都要有括号，以便正确分组子结果。在 SQLAlchemy Core 中生成上述语句如下：

```py
stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1)
stmt2 = select([table1.c.x]).order_by(table2.c.y).limit(2)

stmt = union(stmt1, stmt2)
```

以前，上述结构不会为内部 SELECT 语句产生括号，导致在所有后端上都失败的查询。

上述格式在 SQLite 上**仍将失败**；此外，包含 ORDER BY 但没有 LIMIT/SELECT 的格式在 Oracle 上**仍将失败**。这不是一个不兼容的更改，因为即使没有括号，查询也会失败；通过修复，查询至少在所有其他数据库上都能正常工作。

在所有情况下，为了生成一个在 SQLite 上和在所有情况下在 Oracle 上都能正常工作的有限 SELECT 语句的 UNION，子查询必须是一个 ALIAS 的 SELECT：

```py
stmt1 = select([table1.c.x]).order_by(table1.c.y).limit(1).alias().select()
stmt2 = select([table2.c.x]).order_by(table2.c.y).limit(2).alias().select()

stmt = union(stmt1, stmt2)
```

这个解决方法适用于所有 SQLAlchemy 版本。在 ORM 中，它看起来像：

```py
stmt1 = session.query(Model1).order_by(Model1.y).limit(1).subquery().select()
stmt2 = session.query(Model2).order_by(Model2.y).limit(1).subquery().select()

stmt = session.query(Model1).from_statement(stmt1.union(stmt2))
```

这里的行为与 SQLAlchemy 0.9 中引入的“连接重写”行为有许多相似之处，许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在 (SELECT * FROM ..) AS ANON_1 中；然而，在这种情况下，我们选择不添加新的重写行为来适应 SQLite 的情况。现有的重写行为已经非常复杂，而带有括号的 SELECT 语句的 UNION 情况比该功能的“右嵌套连接”用例要少得多。

[#2528](https://www.sqlalchemy.org/trac/ticket/2528)

## 方言改进和更改 - PostgreSQL

### 支持 INSERT..ON CONFLICT (DO UPDATE | DO NOTHING)

自 PostgreSQL 9.5 版本起添加的 `INSERT` 的 `ON CONFLICT` 子句现在可以使用 PostgreSQL 特定版本的 `Insert` 对象来支持，通过 `sqlalchemy.dialects.postgresql.dml.insert()`。这个 `Insert` 子类添加了两个新方法 `Insert.on_conflict_do_update()` 和 `Insert.on_conflict_do_nothing()`，实现了 PostgreSQL 9.5 在这个领域支持的完整语法：

```py
from sqlalchemy.dialects.postgresql import insert

insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

do_update_stmt = insert_stmt.on_conflict_do_update(
    index_elements=[my_table.c.id], set_=dict(data="some data to update")
)

conn.execute(do_update_stmt)
```

上述内容将呈现为：

```py
INSERT  INTO  my_table  (id,  data)
VALUES  (:id,  :data)
ON  CONFLICT  id  DO  UPDATE  SET  data=:data_2
```

另请参阅

INSERT…ON CONFLICT (Upsert)

[#3529](https://www.sqlalchemy.org/trac/ticket/3529)  ### ARRAY 和 JSON 类型现在正确指定为“不可哈希”

如 关于“不可哈希”类型的更改，影响 ORM 行的去重 中描述的，ORM 在查询的选择实体混合了完整的 ORM 实体和列表达式时，依赖于能够为列值生成哈希函数。`hashable=False` 标志现在已正确设置在 PG 的“数据结构”类型上，包括 `ARRAY` 和 `JSON`。`JSONB` 和 `HSTORE` 类型已经包含了此标志。对于 `ARRAY`，这取决于 `ARRAY.as_tuple` 标志，但是现在不再需要设置此标志以使数组值出现在组合的 ORM 行中。

另请参阅

关于“不可哈希”类型的更改，影响 ORM 行的去重

从 ARRAY、JSON、HSTORE 的索引访问正确地建立 SQL 类型

[#3499](https://www.sqlalchemy.org/trac/ticket/3499)  ### 从 ARRAY、JSON、HSTORE 的索引访问正确地建立 SQL 类型

对于所有三种类型 `ARRAY`、`JSON` 和 `HSTORE`，通过索引访问返回的表达式的 SQL 类型，例如 `col[someindex]`，在所有情况下都应正确。

包括：

+   对于索引访问的 `ARRAY`，分配的 SQL 类型将考虑配置的维度数量。一个具有三个维度的 `ARRAY` 将返回一个类型为 `ARRAY` 的 SQL 表达式，维度减少一个。给定类型为 `ARRAY(Integer, dimensions=3)` 的列，现在我们可以执行以下表达式：

    ```py
    int_expr = col[5][6][7]  # returns an Integer expression object
    ```

    以前，对于 `col[5]` 的索引访问将返回一个 `Integer` 类型的表达式，我们无法再对剩余维度进行索引访问，除非使用 `cast()` 或 `type_coerce()`。

+   `JSON` 和 `JSONB` 类型现在与 PostgreSQL 本身对于索引访问的操作一致。这意味着对于 `JSON` 或 `JSONB` 类型的所有索引访问都会返回一个表达式，该表达式本身*始终*是 `JSON` 或 `JSONB` 本身，除非使用了 `Comparator.astext` 修饰符。这意味着，无论 JSON 结构的索引访问最终指向字符串、列表、数字还是其他 JSON 结构，PostgreSQL 总是将其视为 JSON 本身，除非明确进行了不同的类型转换。与 `ARRAY` 类型类似，现在可以轻松地生成具有多层索引访问的 JSON 表达式：

    ```py
    json_expr = json_col["key1"]["attr1"][5]
    ```

+   通过`HSTORE` 的索引访问返回的“文本”类型，以及与`Comparator.astext` 修饰符一起返回的`JSON` 和 `JSONB` 的“文本”类型现在是可配置的；在这两种情况下，默认为 `TextClause`，但可以使用 `JSON.astext_type` 或 `HSTORE.text_type` 参数将其设置为用户定义的类型。

另请参阅

JSON cast() 操作现在需要显式调用 .astext

[#3499](https://www.sqlalchemy.org/trac/ticket/3499) [#3487](https://www.sqlalchemy.org/trac/ticket/3487)  ### JSON cast() 操作现在需要显式调用 `.astext`

作为从数组、JSON、HSTORE 的索引访问中正确建立 SQL 类型的更改的一部分，`ColumnElement.cast()` 操作符在 `JSON` 和 `JSONB` 上不再隐式调用 `Comparator.astext` 修饰符；PostgreSQL 的 JSON/JSONB 类型支持彼此之间的 CAST 操作，无需“astext”方面。

这意味着在大多数情况下，一个应用程序正在执行这个操作：

```py
expr = json_col["somekey"].cast(Integer)
```

现在需要更改为：

```py
expr = json_col["somekey"].astext.cast(Integer)
```  ### 带有 ENUM 的数组现在将发出 ENUM 的 CREATE TYPE

表定义如下将会按预期发出 CREATE TYPE：

```py
enum = Enum(
    "manager",
    "place_admin",
    "carwash_admin",
    "parking_admin",
    "service_admin",
    "tire_admin",
    "mechanic",
    "carwasher",
    "tire_mechanic",
    name="work_place_roles",
)

class WorkPlacement(Base):
    __tablename__ = "work_placement"
    id = Column(Integer, primary_key=True)
    roles = Column(ARRAY(enum))

e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
Base.metadata.create_all(e)
```

发出：

```py
CREATE  TYPE  work_place_roles  AS  ENUM  (
  'manager',  'place_admin',  'carwash_admin',  'parking_admin',
  'service_admin',  'tire_admin',  'mechanic',  'carwasher',
  'tire_mechanic')

CREATE  TABLE  work_placement  (
  id  SERIAL  NOT  NULL,
  roles  work_place_roles[],
  PRIMARY  KEY  (id)
)
```

[#2729](https://www.sqlalchemy.org/trac/ticket/2729)

### 检查约束现在反映

PostgreSQL 方言现在支持检索 CHECK 约束，方法包括 `Inspector.get_check_constraints()` 和 `Table` 反射中的 `Table.constraints` 集合。

### “Plain” 和 “Materialized” 视图可以分别检查

新参数 `PGInspector.get_view_names.include` 允许指定应返回哪些视图子类型：

```py
from sqlalchemy import inspect

insp = inspect(engine)

plain_views = insp.get_view_names(include="plain")
all_views = insp.get_view_names(include=("plain", "materialized"))
```

[#3588](https://www.sqlalchemy.org/trac/ticket/3588)

### Index 添加 tablespace 选项

`Index` 对象现在接受参数 `postgresql_tablespace`，以指定 TABLESPACE，与 `Table` 对象所接受的方式相同。

另请参见

Index Storage Parameters

[#3720](https://www.sqlalchemy.org/trac/ticket/3720)

### PyGreSQL 的支持

[PyGreSQL](https://pypi.org/project/PyGreSQL) DBAPI 现在得到支持。

### “postgres” 模块已被移除

`sqlalchemy.dialects.postgres` 模块，长期弃用，已被移除；多年来一直发出警告，项目应调用 `sqlalchemy.dialects.postgresql`。形式为 `postgres://` 的 Engine URLs 仍将继续运行。

### 支持 FOR UPDATE SKIP LOCKED / FOR NO KEY UPDATE / FOR KEY SHARE

新参数 `GenerativeSelect.with_for_update.skip_locked` 和 `GenerativeSelect.with_for_update.key_share` ��� Core 和 ORM 中都应用于 PostgreSQL 后端的 “SELECT…FOR UPDATE” 或 “SELECT…FOR SHARE” 查询的修改：

+   SELECT FOR NO KEY UPDATE:

    ```py
    stmt = select([table]).with_for_update(key_share=True)
    ```

+   SELECT FOR UPDATE SKIP LOCKED:

    ```py
    stmt = select([table]).with_for_update(skip_locked=True)
    ```

+   SELECT FOR KEY SHARE:

    ```py
    stmt = select([table]).with_for_update(read=True, key_share=True)
    ```

### 支持 INSERT..ON CONFLICT (DO UPDATE | DO NOTHING)

`INSERT` 的 `ON CONFLICT` 子句，自 PostgreSQL 版本 9.5 起添加，现在可使用 PostgreSQL 特定版本的 `Insert` 对象支持，通过 `sqlalchemy.dialects.postgresql.dml.insert()`。这个 `Insert` 子类添加了两个新方法 `Insert.on_conflict_do_update()` 和 `Insert.on_conflict_do_nothing()`，实现了 PostgreSQL 9.5 在这个领域支持的完整语法：

```py
from sqlalchemy.dialects.postgresql import insert

insert_stmt = insert(my_table).values(id="some_id", data="some data to insert")

do_update_stmt = insert_stmt.on_conflict_do_update(
    index_elements=[my_table.c.id], set_=dict(data="some data to update")
)

conn.execute(do_update_stmt)
```

上述内容将渲染为：

```py
INSERT  INTO  my_table  (id,  data)
VALUES  (:id,  :data)
ON  CONFLICT  id  DO  UPDATE  SET  data=:data_2
```

另请参见

INSERT…ON CONFLICT (Upsert)

[#3529](https://www.sqlalchemy.org/trac/ticket/3529)

### ARRAY 和 JSON 类型现在正确指定“不可哈希”

如关于“不可哈希”类型的变更，影响了 ORM 行的去重所述，ORM 在查询的选定实体中混合全 ORM 实体与列表达式时，依赖于能够为列值产生哈希函数。现在，`hashable=False` 标志已正确设置在所有 PG 的“数据结构”类型上，包括`ARRAY`和`JSON`。`JSONB`和`HSTORE` 类型已经包括了这个标志。对于`ARRAY`，这取决于`ARRAY.as_tuple` 标志，然而，现在应该不再需要设置这个标志，以便在组合的 ORM 行中具有数组值。  

另请参见

关于“不可哈希”类型的变更，影响了 ORM 行的去重

从 ARRAY、JSON、HSTORE 的索引访问正确建立 SQL 类型

[#3499](https://www.sqlalchemy.org/trac/ticket/3499)

### 从 ARRAY、JSON、HSTORE 的索引访问正确建立 SQL 类型

对于`ARRAY`、`JSON`和`HSTORE`三者，通过索引访问返回的表达式所分配的 SQL 类型，例如 `col[someindex]`，在所有情况下都应该是正确的。

这包括：

+   对于`ARRAY`的索引访问所分配的 SQL 类型将考虑到配置的维度数量。一个具有三个维度的`ARRAY`将返回一个维度少一的`ARRAY`的 SQL 表达式类型。给定一个类型为 `ARRAY(Integer, dimensions=3)` 的列，我们现在可以执行这个表达式：

    ```py
    int_expr = col[5][6][7]  # returns an Integer expression object
    ```

    以前，对于 `col[5]` 的索引访问将返回一个 `Integer` 类型的表达式，我们无法再对剩余维度进行索引访问，除非使用 `cast()` 或 `type_coerce()`。

+   `JSON` 和 `JSONB` 类型现在与 PostgreSQL 本身对于索引访问的操作一致。这意味着对于 `JSON` 或 `JSONB` 类型的所有索引访问都会返回一个表达式，该表达式本身*始终*是 `JSON` 或 `JSONB` 本身，除非使用了 `Comparator.astext` 修饰符。这意味着，无论 JSON 结构的索引访问最终指向字符串、列表、数字还是其他 JSON 结构，PostgreSQL 总是将其视为 JSON 本身，除非明确进行了不同的类型转换。与 `ARRAY` 类型类似，现在可以轻松地生成具有多层索引访问的 JSON 表达式：

    ```py
    json_expr = json_col["key1"]["attr1"][5]
    ```

+   通过对 `HSTORE` 的索引访问返回的“文本”类型，以及通过对 `JSON` 和 `JSONB` 的索引访问返回的“文本”类型，再结合 `Comparator.astext` 修饰符，现在是可配置的；在这两种情况下，默认为 `TextClause`，但可以使用 `JSON.astext_type` 或 `HSTORE.text_type` 参数将其设置为用户定义的类型。

另请参阅

JSON cast() 操作现在需要显式调用 .astext

[#3499](https://www.sqlalchemy.org/trac/ticket/3499) [#3487](https://www.sqlalchemy.org/trac/ticket/3487)

### JSON cast() 操作现在需要显式调用 `.astext`

作为 从数组、JSON、HSTORE 的索引访问正确建立 SQL 类型的更改 的一部分，`ColumnElement.cast()` 操作在 `JSON` 和 `JSONB` 上不再隐式调用 `Comparator.astext` 修饰符；PostgreSQL 的 JSON/JSONB 类型支持彼此之间的 CAST 操作，无需 "astext" 方面。

这意味着在大多数情况下，一个应用程序如果在做这个操作：

```py
expr = json_col["somekey"].cast(Integer)
```

现在需要更改为：

```py
expr = json_col["somekey"].astext.cast(Integer)
```

### 带有 ENUM 的数组现在会发出 CREATE TYPE 用于 ENUM

类似以下的表定义现在会按预期发出 CREATE TYPE：

```py
enum = Enum(
    "manager",
    "place_admin",
    "carwash_admin",
    "parking_admin",
    "service_admin",
    "tire_admin",
    "mechanic",
    "carwasher",
    "tire_mechanic",
    name="work_place_roles",
)

class WorkPlacement(Base):
    __tablename__ = "work_placement"
    id = Column(Integer, primary_key=True)
    roles = Column(ARRAY(enum))

e = create_engine("postgresql://scott:tiger@localhost/test", echo=True)
Base.metadata.create_all(e)
```

发出：

```py
CREATE  TYPE  work_place_roles  AS  ENUM  (
  'manager',  'place_admin',  'carwash_admin',  'parking_admin',
  'service_admin',  'tire_admin',  'mechanic',  'carwasher',
  'tire_mechanic')

CREATE  TABLE  work_placement  (
  id  SERIAL  NOT  NULL,
  roles  work_place_roles[],
  PRIMARY  KEY  (id)
)
```

[#2729](https://www.sqlalchemy.org/trac/ticket/2729)

### 检查约束现在反映

PostgreSQL 方言现在支持反射 CHECK 约束，方法包括 `Inspector.get_check_constraints()` 以及 `Table` 反射中的 `Table.constraints` 集合。

### 可以单独检查“普通”和“物化”视图

新参数 `PGInspector.get_view_names.include` 允许指定应返回哪些视图子类型：

```py
from sqlalchemy import inspect

insp = inspect(engine)

plain_views = insp.get_view_names(include="plain")
all_views = insp.get_view_names(include=("plain", "materialized"))
```

[#3588](https://www.sqlalchemy.org/trac/ticket/3588)

### 向 Index 添加 tablespace 选项

`Index` 对象现在接受参数 `postgresql_tablespace`，以指定 TABLESPACE，与 `Table` 对象接受的方式相同。

参见

索引存储参数

[#3720](https://www.sqlalchemy.org/trac/ticket/3720)

### 对 PyGreSQL 的支持

[PyGreSQL](https://pypi.org/project/PyGreSQL) DBAPI 现在受支持。

### “postgres” 模块已移除

长期弃用的 `sqlalchemy.dialects.postgres` 模块已被移除；多年来一直发出警告，项目应该调用 `sqlalchemy.dialects.postgresql`。形式为 `postgres://` 的 Engine URLs 仍将继续运行。

### 支持 FOR UPDATE SKIP LOCKED / FOR NO KEY UPDATE / FOR KEY SHARE

Core 和 ORM 中的新参数 `GenerativeSelect.with_for_update.skip_locked` 和 `GenerativeSelect.with_for_update.key_share` 在 PostgreSQL 后端上对 “SELECT…FOR UPDATE” 或 “SELECT…FOR SHARE” 查询应用修改：

+   SELECT FOR NO KEY UPDATE:

    ```py
    stmt = select([table]).with_for_update(key_share=True)
    ```

+   SELECT FOR UPDATE SKIP LOCKED:

    ```py
    stmt = select([table]).with_for_update(skip_locked=True)
    ```

+   SELECT FOR KEY SHARE:

    ```py
    stmt = select([table]).with_for_update(read=True, key_share=True)
    ```

## 方言改进和更改 - MySQL

### MySQL JSON 支持

新类型 `JSON` 已添加到 MySQL 方言，支持 MySQL 5.7 新增的 JSON 类型。该类型提供 JSON 的持久性以及内部使用 `JSON_EXTRACT` 函数进行基本索引访问。通过使用 MySQL 和 PostgreSQL 共同的 `JSON` 数据类型，可以实现跨 MySQL 和 PostgreSQL 的可索引 JSON 列。

参见

核心中添加的 JSON 支持

[#3547](https://www.sqlalchemy.org/trac/ticket/3547)  ### 增加了对 AUTOCOMMIT “隔离级别”的支持

MySQL 方言现在接受值“AUTOCOMMIT”作为 `create_engine.isolation_level` 和 `Connection.execution_options.isolation_level` 参数：

```py
connection = engine.connect()
connection = connection.execution_options(isolation_level="AUTOCOMMIT")
```

隔离级别利用大多数 MySQL DBAPI 提供的各种“autocommit”属性。

[#3332](https://www.sqlalchemy.org/trac/ticket/3332)  ### 不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

MySQL 方言的行为是，如果 InnoDB 表上的复合主键中的一个列具有 AUTO_INCREMENT 但不是第一列，例如：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True, autoincrement=False),
    Column("y", Integer, primary_key=True, autoincrement=True),
    mysql_engine="InnoDB",
)
```

会生成如下的 DDL：

```py
CREATE  TABLE  some_table  (
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL  AUTO_INCREMENT,
  PRIMARY  KEY  (x,  y),
  KEY  idx_autoinc_y  (y)
)ENGINE=InnoDB
```

请注意上述带有自动生成名称的“KEY”；这是多年前针对 AUTO_INCREMENT 在 InnoDB 上否则会失败的问题而进入方言的变更。

这种解决方法已被移除，并替换为更好的系统，即在主键中仅将 AUTO_INCREMENT 列 *放在第一位*：

```py
CREATE  TABLE  some_table  (
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL  AUTO_INCREMENT,
  PRIMARY  KEY  (y,  x)
)ENGINE=InnoDB
```

要明确控制主键列的排序，请显式使用 `PrimaryKeyConstraint` 构造（1.1.0b2）（以及根据 MySQL 要求的自动增量列的 KEY），例如：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
    PrimaryKeyConstraint("x", "y"),
    UniqueConstraint("y"),
    mysql_engine="InnoDB",
)
```

除了变更 不再为复合主键列隐式启用 .autoincrement 指令 外，现在更容易指定具有或不具有自动增量的复合主键；`Column.autoincrement` 现在默认为值 `"auto"`，不再需要 `autoincrement=False` 指令：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
    mysql_engine="InnoDB",
)
```  ### MySQL JSON 支持

MySQL 方言现在支持新增于 MySQL 5.7 的 JSON 类型，为此添加了一种新类型 `JSON`。该类型提供了 JSON 的持久性以及内部使用 `JSON_EXTRACT` 函数进行基本索引访问。通过使用在 MySQL 和 PostgreSQL 中通用的 `JSON` 数据类型，可以实现跨 MySQL 和 PostgreSQL 的可索引 JSON 列。

另请参见

核心中添加的 JSON 支持

[#3547](https://www.sqlalchemy.org/trac/ticket/3547)

### 增加了对 AUTOCOMMIT “隔离级别”的支持

MySQL 方言现在接受值“AUTOCOMMIT”作为`create_engine.isolation_level`和`Connection.execution_options.isolation_level`参数：

```py
connection = engine.connect()
connection = connection.execution_options(isolation_level="AUTOCOMMIT")
```

隔离级别利用大多数 MySQL DBAPI 提供的各种“autocommit”属性。

[#3332](https://www.sqlalchemy.org/trac/ticket/3332)

### 不再为具有 AUTO_INCREMENT 的复合主键生成隐式 KEY

MySQL 方言的行为是，如果 InnoDB 表上的复合主键中的一个列具有 AUTO_INCREMENT 且不是第一列，例如：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True, autoincrement=False),
    Column("y", Integer, primary_key=True, autoincrement=True),
    mysql_engine="InnoDB",
)
```

将生成以下 DDL：

```py
CREATE  TABLE  some_table  (
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL  AUTO_INCREMENT,
  PRIMARY  KEY  (x,  y),
  KEY  idx_autoinc_y  (y)
)ENGINE=InnoDB
```

请注意上述带有自动生成名称的“KEY”；这是多年前为了解决 AUTO_INCREMENT 在 InnoDB 上否则会失败的问题而引入的方言变化。

这个解决方法已被移除，并用更好的系统替代，即在主键中首先声明 AUTO_INCREMENT 列：

```py
CREATE  TABLE  some_table  (
  x  INTEGER  NOT  NULL,
  y  INTEGER  NOT  NULL  AUTO_INCREMENT,
  PRIMARY  KEY  (y,  x)
)ENGINE=InnoDB
```

为了明确控制主键列的顺序，请显式使用`PrimaryKeyConstraint`构造（1.1.0b2）（以及 MySQL 所需的自动增量列的 KEY），例如：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
    PrimaryKeyConstraint("x", "y"),
    UniqueConstraint("y"),
    mysql_engine="InnoDB",
)
```

伴随着变化.autoincrement 指令不再隐式启用复合主键列，现在更容易指定具有或不具有自动增量的复合主键；`Column.autoincrement`现在默认为值`"auto"`，不再需要`autoincrement=False`指令：

```py
t = Table(
    "some_table",
    metadata,
    Column("x", Integer, primary_key=True),
    Column("y", Integer, primary_key=True, autoincrement=True),
    mysql_engine="InnoDB",
)
```

## 方言改进和变化 - SQLite

### SQLite 版本 3.7.16 解决的右嵌套连接问题

在 0.9 版本中，由 许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包装在 (SELECT * FROM ..) AS ANON_1 中 引入的功能经历了大量努力，以支持在 SQLite 上重写连接以始终使用子查询以实现“右嵌套连接”效果，因为多年来 SQLite 并不支持这种语法。具有讽刺意味的是，在该迁移说明中指出的 SQLite 版本 3.7.15.2，实际上是 SQLite 最后一个具有这种限制的版本！下一个版本是 3.7.16，支持右嵌套连接被悄悄地添加了进去。在 1.1 版本中，已经完成了识别特定 SQLite 版本和源提交的工作，这个变化的解决方案现在在 SQLite 报告版本 3.7.16 或更高版本时被取消。

[#3634](https://www.sqlalchemy.org/trac/ticket/3634)  ### 取消 SQLite 版本 3.10.0 中的带点列名变通方法

SQLite 方言长期以来一直存在一个问题的变通方法，即数据库驱动程序在某些 SQL 结果集中没有报告正确的列名，特别是在使用 UNION 时。这个变通方法在 带点的列名 中有详细说明，并要求 SQLAlchemy 假定任何带有点的列名实际上是通过这种错误行为传递的 `tablename.columnname` 组合，可以通过 `sqlite_raw_colnames` 执行选项关闭这个选项。

截至 SQLite 版本 3.10.0，UNION 和其他查询中的错误已经修复；就像 SQLite 版本 3.7.16 中取消右嵌套连接的变通方法 中描述的变化一样，SQLite 的变更日志只将其神秘地标识为“为 sqlite3_module.xBestIndex 方法添加了 colUsed 字段”，但是 SQLAlchemy 对这些带点的列名的转换在这个版本中不再需要，因此当检测到版本 3.10.0 或更高版本时会关闭这个功能。

总的来说，截至 1.0 系列，SQLAlchemy 的 `ResultProxy` 在为 Core 和 ORM SQL 结构提供结果时，对列名的依赖要少得多，因此这个问题的重要性在任何情况下都已经降低了。

[#3633](https://www.sqlalchemy.org/trac/ticket/3633)  ### 改进对远程模式的支持

SQLite 方言现在实现了 `Inspector.get_schema_names()`，并且对于从远程模式创建和反映的表和索引提供了改进的支持，在 SQLite 中，远程模式是通过 `ATTACH` 语句分配名称的数据库；先前，``CREATE INDEX`` DDL 对于模式绑定表不起作用，并且 `Inspector.get_foreign_keys()` 方法现在将在结果中指示给定的模式。不支持跨模式外键。### 主键约束名称的反射

SQLite 后端现在利用 SQLite 的“sqlite_master”视图来提取表的原始 DDL 中主键约束的名称，这与 SQLAlchemy 的最新版本中用于提取外键约束的方式相同。

[#3629](https://www.sqlalchemy.org/trac/ticket/3629)

### 检查约束现在反映

SQLite 方言现在支持检查约束的反射，包括在方法 `Inspector.get_check_constraints()` 内以及在 `Table` 反映中的 `Table.constraints` 集合内。

### ON DELETE 和 ON UPDATE 外键短语现在反映

`Inspector` 现在将包括来自 SQLite 方言的外键约束的 ON DELETE 和 ON UPDATE 短语，并且作为 `Table` 的一部分反映的 `ForeignKeyConstraint` 对象也将指示这些短语。

### 右嵌套连接解决方案针对 SQLite 版本 3.7.16 解除

在版本 0.9 中，由 许多 JOIN 和 LEFT OUTER JOIN 表达式将不再被包裹在 (SELECT * FROM ..) AS ANON_1 中 引入的功能经历了大量努力，以支持在 SQLite 上重写连接以始终使用子查询以实现“右嵌套连接”效果，因为 SQLite 多年来一直不支持这种语法。具有讽刺意味的是，在迁移说明中指出的 SQLite 版本 3.7.15.2 实际上是 SQLite 最后一个具有这种限制的版本！下一个发布版本是 3.7.16，支持右嵌套连接被悄悄地添加了进去。在 1.1 中，已经完成了识别特定 SQLite 版本和源提交的工作，其中进行了这一更改（SQLite 的更改日志用神秘的短语“增强查询优化器以利用传递连接约束”来指称，没有链接到任何问题编号、更改编号或进一步解释），并且当 DBAPI 报告版本 3.7.16 或更高版本生效时，此更改中存在的变通方法现在已经被取消了。

[#3634](https://www.sqlalchemy.org/trac/ticket/3634)

### 取消 SQLite 版本 3.10.0 中的带点列名变通方法

SQLite 方言长期以来一直有一个解决方案，用于解决数据库驱动程序在某些 SQL 结果集中未报告正确列名的问题，特别是在使用 UNION 时。这个解决方案在 带点列名 中有详细说明，并要求 SQLAlchemy 假定任何带有点的列名实际上是通过这种错误行为传递的 `tablename.columnname` 组合，可以通过 `sqlite_raw_colnames` 执行选项关闭这个选项。

截至 SQLite 版本 3.10.0，UNION 和其他查询中的错误已经修复；就像 SQLite 版本 3.7.16 中取消右嵌套连接变通方法 中描述的更改一样，SQLite 的更改日志只将其神秘地标识为“为 sqlite3_module.xBestIndex 方法添加了 colUsed 字段”，然而 SQLAlchemy 对这些带点的列名的转换在此版本中不再需要，因此当检测到版本 3.10.0 或更高版本时，将其关闭。

总的来说，在 1.0 系列中的 SQLAlchemy `ResultProxy` 在为 Core 和 ORM SQL 结构提供结果时，对列名的依赖要少得多，因此这个问题的重要性在任何情况下已经降低了。

[#3633](https://www.sqlalchemy.org/trac/ticket/3633)

### 改进对远程模式的支持

SQLite 方言现在实现了`Inspector.get_schema_names()`，并且对于从远程模式创建和反映的表和索引提供了改进的支持，在 SQLite 中，远程模式是通过`ATTACH`语句分配名称的数据库；以前，``CREATE INDEX`` DDL 对于绑定模式表的情况无法正确工作，而`Inspector.get_foreign_keys()`方法现在将在结果中指示给定的模式。不支持跨模式外键。

### 反映主键约束的名称

SQLite 后端现在利用 SQLite 的“sqlite_master”视图，以从原始 DDL 中提取表的主键约束的名称，就像最近 SQLAlchemy 版本中为外键约束所实现的方式一样。

[#3629](https://www.sqlalchemy.org/trac/ticket/3629)

### 检查约束现在反映

SQLite 方言现在支持在方法`Inspector.get_check_constraints()`以及在`Table`反映中的`Table.constraints`集合中反映 CHECK 约束。

### ON DELETE 和 ON UPDATE 外键短语现在反映

`Inspector`现在将包括 SQLite 方言上外键约束的 ON DELETE 和 ON UPDATE 短语，并且作为`Table`的一部分反映的`ForeignKeyConstraint`对象也将指示这些短语。

## 方言改进和更改 - SQL Server

### 为 SQL Server 添加了事务隔离级别支持

所有 SQL Server 方言都支持通过`create_engine.isolation_level`和`Connection.execution_options.isolation_level`参数设置事务隔离级别。四个标准级别以及`SNAPSHOT`都受支持：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@ms_2008", isolation_level="REPEATABLE READ"
)
```

另请参见

事务隔离级别

[#3534](https://www.sqlalchemy.org/trac/ticket/3534)  ### 反射时字符串/可变长度类型不再明确表示“max”

当反射诸如`String`、`TextClause`等包含长度的类型时，在 SQL Server 下，“无长度”类型将复制“length”参数作为值`"max"`：

```py
>>> from sqlalchemy import create_engine, inspect
>>> engine = create_engine("mssql+pyodbc://scott:tiger@ms_2008", echo=True)
>>> engine.execute("create table s (x varchar(max), y varbinary(max))")
>>> insp = inspect(engine)
>>> for col in insp.get_columns("s"):
...     print(col["type"].__class__, col["type"].length)
<class 'sqlalchemy.sql.sqltypes.VARCHAR'> max
<class 'sqlalchemy.dialects.mssql.base.VARBINARY'> max
```

基本类型中的“length”参数预期只能是整数值或`None`；`None`表示无限长度，SQL Server 方言将其解释为“max”。修复方法是将这些长度输出为`None`，以便类型对象在非 SQL Server 上下文中正常工作：

```py
>>> for col in insp.get_columns("s"):
...     print(col["type"].__class__, col["type"].length)
<class 'sqlalchemy.sql.sqltypes.VARCHAR'> None
<class 'sqlalchemy.dialects.mssql.base.VARBINARY'> None
```

可能一直依赖于将“length”值直接与字符串“max”进行比较的应用程序应该考虑将值设为`None`表示相同的含义。

[#3504](https://www.sqlalchemy.org/trac/ticket/3504)

### 支持在主键上使用“非聚集”以允许在其他地方进行聚集

`UniqueConstraint`、`PrimaryKeyConstraint`、`Index` 上可用的 `mssql_clustered` 标志现在默认为 `None`，可以设置为 False，这将特别为主键渲染 `NONCLUSTERED` 关键字，允许使用不同的索引作为“聚集”。

另请参见

聚集索引支持

### `legacy_schema_aliasing` 标志现在设置为 False

SQLAlchemy 1.0.5 引入了 `legacy_schema_aliasing` 标志到 MSSQL 方言，允许关闭所谓的“传统模式”别名。这种别名尝试将模式限定的表转换为别名；给定一个表如下：

```py
account_table = Table(
    "account",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("info", String(100)),
    schema="customer_schema",
)
```

传统行为模式将尝试将模式限定的表名转换为别名：

```py
>>> eng = create_engine("mssql+pymssql://mydsn", legacy_schema_aliasing=True)
>>> print(account_table.select().compile(eng))
SELECT  account_1.id,  account_1.info
FROM  customer_schema.account  AS  account_1 
```

然而，已经证明这种别名是不必要的，在许多情况下会产生不正确的 SQL。

在 SQLAlchemy 1.1 中，`legacy_schema_aliasing` 标志现在默认为 False，禁用此行为模式，并允许 MSSQL 方言以模式限定的表正常运行。对于可能依赖此行为的应用程序，请将标志设置回 True。

[#3434](https://www.sqlalchemy.org/trac/ticket/3434)  ### 为 SQL Server 添加了事务隔离级别支持

所有 SQL Server 方言都支持通过`create_engine.isolation_level`和`Connection.execution_options.isolation_level`参数设置事务隔离级别。四个标准级别都受支持，以及`SNAPSHOT`：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@ms_2008", isolation_level="REPEATABLE READ"
)
```

另请参阅

事务隔离级别

[#3534](https://www.sqlalchemy.org/trac/ticket/3534)

### 字符串/可变长度类型不再在反射时显式表示“max”

当反射像`String`、`TextClause`等包含长度的类型时，SQL Server 下的“无长度”类型会将“长度”参数复制为值`"max"`：

```py
>>> from sqlalchemy import create_engine, inspect
>>> engine = create_engine("mssql+pyodbc://scott:tiger@ms_2008", echo=True)
>>> engine.execute("create table s (x varchar(max), y varbinary(max))")
>>> insp = inspect(engine)
>>> for col in insp.get_columns("s"):
...     print(col["type"].__class__, col["type"].length)
<class 'sqlalchemy.sql.sqltypes.VARCHAR'> max
<class 'sqlalchemy.dialects.mssql.base.VARBINARY'> max
```

基本类型中的“长度”参数预期只是一个整数值或仅为 None；None 表示无界长度，而 SQL Server 方言将其解释为“max”。因此修复这些长度为 None，以便类型对象在非 SQL Server 上下文中工作：

```py
>>> for col in insp.get_columns("s"):
...     print(col["type"].__class__, col["type"].length)
<class 'sqlalchemy.sql.sqltypes.VARCHAR'> None
<class 'sqlalchemy.dialects.mssql.base.VARBINARY'> None
```

可能已经依赖将“长度”值直接与字符串“max”进行比较的应用程序应考虑将值`None`视为相同的含义。

[#3504](https://www.sqlalchemy.org/trac/ticket/3504)

### 支持在其他地方集群化以允许主键上的“非聚集”

现在`UniqueConstraint`、`PrimaryKeyConstraint`、`Index`上可用的`mssql_clustered`标志默认为`None`，并且可以设置为 False，这将在特定情况下渲染 NONCLUSTERED 关键字用于主键，允许使用不同的索引作为“集群化”。

另请参阅

集群索引支持

### `legacy_schema_aliasing`标志现在设置为 False

SQLAlchemy 1.0.5 引入了`legacy_schema_aliasing`标志到 MSSQL 方言中，允许所谓的“遗留模式”别名化被关闭。这种别名化试图将架构限定的表转换为别名；给定这样一个表：

```py
account_table = Table(
    "account",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("info", String(100)),
    schema="customer_schema",
)
```

行为的遗留模式将尝试将架构限定的表名转换为别名：

```py
>>> eng = create_engine("mssql+pymssql://mydsn", legacy_schema_aliasing=True)
>>> print(account_table.select().compile(eng))
SELECT  account_1.id,  account_1.info
FROM  customer_schema.account  AS  account_1 
```

然而，这种别名化已被证明是不必要的，并且在许多情况下会产生不正确的 SQL。

在 SQLAlchemy 1.1 中，`legacy_schema_aliasing`标志现在默认为 False，禁用此行为模式，允许 MSSQL 方言与模式限定的表正常运行。对于可能依赖此行为的应用程序，请将标志设置回 True。

[#3434](https://www.sqlalchemy.org/trac/ticket/3434)

## 方言改进和变更 - Oracle

### 支持 SKIP LOCKED

在 Core 和 ORM 中的新参数`GenerativeSelect.with_for_update.skip_locked`将为“SELECT…FOR UPDATE”或“SELECT.. FOR SHARE”查询生成“SKIP LOCKED”后缀。

### 支持 SKIP LOCKED

在 Core 和 ORM 中的新参数`GenerativeSelect.with_for_update.skip_locked`将为“SELECT…FOR UPDATE”或“SELECT.. FOR SHARE”查询生成“SKIP LOCKED”后缀。
