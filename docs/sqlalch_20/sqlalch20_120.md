# 会话 / 查询

> 原文：[`docs.sqlalchemy.org/en/20/faq/sessions.html`](https://docs.sqlalchemy.org/en/20/faq/sessions.html)

+   我正在使用 Session 重新加载数据，但它没有看到我在其他地方提交的更改

+   “此会话的事务已由于刷新期间的先前异常而回滚。”（或类似消息）

    +   但为什么 flush() 坚持发出 ROLLBACK？

    +   但为什么一个自动调用 ROLLBACK 不够？为什么我必须再次 ROLLBACK？

+   如何创建一个始终向每个查询添加特定过滤器的查询？

+   我的查询返回的对象数与 query.count() 告诉我的不一致 - 为什么？

+   我已经创建了一个对 Outer Join 的映射，虽然查询返回行，但没有返回对象。为什么？

+   我使用 `joinedload()` 或 `lazy=False` 创建 JOIN/OUTER JOIN，但当我尝试添加 WHERE、ORDER BY、LIMIT 等条件时，SQLAlchemy 没有构造正确的查询（这取决于 (OUTER) JOIN）

+   查询没有 `__len__()`，为什么？

+   如何在 ORM 查询中使用文本 SQL？

+   调用 `Session.delete(myobject)` 后，我的对象未从父集合中移除！

+   加载对象时为什么不调用我的 `__init__()`？

+   如何在 SA 的 ORM 中使用 ON DELETE CASCADE？

+   我将实例的“foo_id”属性设置为“7”，但“foo”属性仍然为 `None` - 应该加载 id 为 #7 的 Foo 吗？

+   如何遍历与给定对象相关的所有对象？

+   有没有一种方法可以自动只获取唯一关键字（或其他类型的对象），而不需要查询关键字并获取包含该关键字的行的引用？

+   为什么 post_update 除了第一个 UPDATE 之外还会发出 UPDATE？

## 我重新加载了我的会话中的数据，但它没有看到我在其他地方提交的更改

这种行为的主要问题在于，会话表现得好像事务处于*可串行化*隔离状态一样，即使实际上并非如此（通常也不是）。从实际角度来看，这意味着会话在事务范围内已经读取的数据不会发生任何更改。

如果术语“隔离级别”不熟悉，那么您首先需要阅读此链接：

[隔离级别](https://en.wikipedia.org/wiki/Isolation_%28database_systems%29)

简而言之，可串行化隔离级通常意味着一旦在事务中选择了一系列行，每次重新发出该 SELECT 时都会获得*相同的数据*。如果您处于较低的隔离级别“可重复读”，您将看到新添加的行（不再看到已删除的行），但对于您已经加载的行，您不会看到任何更改。只有当您处于较低的隔离级别，例如“读取提交”，才有可能看到数据行更改其值。

有关在使用 SQLAlchemy ORM 时控制隔离级别的信息，请参阅设置事务隔离级别 / DBAPI AUTOCOMMIT。

为了极大地简化事情，`Session`本身是基于完全隔离的事务运行的，并且不会覆盖已经读取的任何映射属性，除非您告诉它这样做。尝试在进行中的事务中重新读取已加载的数据的用例是一个*不常见*的用例，在许多情况下没有任何效果，因此这被认为是例外而不是规范；为了在这种例外情况下工作，提供了几种方法允许在进行中的事务上下文中重新加载特定数据。

要理解我们在谈论`Session`时所说的“事务”是什么意思，您的`Session`只能在事务内部工作。有关概述，请参阅管理事务。

一旦我们弄清楚了我们的隔离级别是什么，并且我们认为我们的隔离级别设置得足够低，以便如果我们重新选择一行，我们应该能够在我们的 `Session` 中看到新数据，那么我们该如何看到它？

从最常见到最不常见的三种方式：

1.  我们只需结束当前事务，并在下一次访问时启动新事务，通过调用 `Session.commit()`（请注意，如果 `Session` 处于较少使用的“自动提交”模式，则还会调用 `Session.begin()`）。绝大多数应用和用例不会出现无法“看到”其他事务中的数据的问题，因为它们遵循了这一模式，这是**短事务**最佳实践的核心。有关此问题的一些想法，请参阅 何时构建 Session，何时提交它，何时关闭它？。

1.  我们告诉我们的 `Session` 重新读取它已经读取过的行，要么在下次使用 `Session.expire_all()` 或 `Session.expire()` 查询它们时，要么立即在对象上使用 `refresh`。有关此事的详细信息，请参阅 刷新 / 过期。

1.  我们可以在设置了“填充现有”选项的情况下运行整个查询，这样它们读取行时就会覆盖已加载的对象。这是一个在 填充现有 中描述的执行选项。

但请记住，**如果我们的隔离级别是可重复读或更高，则 ORM 无法看到行中的更改，除非我们开始新的事务**。## “此会话的事务由于在 flush 期间发生的先前异常而回滚。”（或类似）

当 `Session.flush()` 引发异常、回滚事务，但后续对 `Session` 的命令未显式调用 `Session.rollback()` 或 `Session.close()` 时会出现此错误。

通常对应于在 `Session.flush()` 或 `Session.commit()` 上捕获异常并且不正确处理异常的应用程序。例如：

```py
from sqlalchemy import create_engine, Column, Integer
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base(create_engine("sqlite://"))

class Foo(Base):
    __tablename__ = "foo"
    id = Column(Integer, primary_key=True)

Base.metadata.create_all()

session = sessionmaker()()

# constraint violation
session.add_all([Foo(id=1), Foo(id=1)])

try:
    session.commit()
except:
    # ignore error
    pass

# continue using session without rolling back
session.commit()
```

使用 `Session` 应该符合类似于此结构：

```py
try:
    # <use session>
    session.commit()
except:
    session.rollback()
    raise
finally:
    session.close()  # optional, depends on use case
```

除了 flushes 外，很多事情都可能导致 try/except 内部的失败。应用程序应确保对基于 ORM 的过程应用某种“框架”系统，以便连接和事务资源具有明确的边界，并且如果发生任何失败条件，则可以显式地回滚事务。

这并不意味着整个应用程序都应该有 try/except 块，这将不是一种可扩展的架构。相反，一种典型的方法是，当首次调用基于 ORM 的方法和函数时，从最顶层调用函数的过程将处于一个块中，该块在一系列操作成功完成时提交事务，并且在任何原因失败时，包括失败的 flushes 时回滚事务。也有使用函数装饰器或上下文管理器来实现类似结果的方法。采取的方法取决于正在编写的应用程序的类型。

关于如何组织使用 `Session` 的详细讨论，请参阅何时构造会话，何时提交它，何时关闭它？。

### 但为什么 flush() 一定要发出 ROLLBACK 呢？

如果 `Session.flush()` 能够部分完成而不回滚，那将是很好的，但是由于其当前能力有限，因此这超出了它的当前能力范围，因为其内部簿记必须被修改，以便可以随时停止，并且与已刷新到数据库的内容完全一致。虽然从理论上讲这是可能的，但是增强功能的有用性因为许多数据库操作在任何情况下都需要回滚而大大降低。特别是，Postgres 有一些操作，一旦失败，事务就不允许继续进行：

```py
test=> create table foo(id integer primary key);
NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "foo_pkey" for table "foo"
CREATE TABLE
test=> begin;
BEGIN
test=> insert into foo values(1);
INSERT 0 1
test=> commit;
COMMIT
test=> begin;
BEGIN
test=> insert into foo values(1);
ERROR:  duplicate key value violates unique constraint "foo_pkey"
test=> insert into foo values(2);
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

SQLAlchemy 提供的解决这两个问题的方法是支持 SAVEPOINT，通过 `Session.begin_nested()`。使用 `Session.begin_nested()`，您可以在事务内部对可能失败的操作进行框架化，然后在保持封闭事务的同时“回滚”到失败之前的点。

### 但为什么一个自动调用 ROLLBACK 不够？为什么我必须再次 ROLLBACK？

由`flush()`引起的回滚并不是完整事务块的结束；虽然它结束了正在进行的数据库事务，但从`Session`的角度来看，仍然存在一个现在处于非活动状态的事务。

给定这样一个块：

```py
sess = Session()  # begins a logical transaction
try:
    sess.flush()

    sess.commit()
except:
    sess.rollback()
```

在上面，当首次创建一个`Session`时，假设没有使用“自动提交模式”，在`Session`内建立了一个逻辑事务。这个事务是“逻辑”的，因为直到调用 SQL 语句时才会实际使用任何数据库资源，此时会启动连接级和 DBAPI 级事务。然而，无论数据库级事务是否是其状态的一部分，逻辑事务将保持不变，直到使用`Session.commit()`、`Session.rollback()`或`Session.close()`结束它。

当上面的`flush()`失败时，代码仍然位于由 try/commit/except/rollback 块框定的事务中。如果`flush()`完全回滚逻辑事务，那么当我们到达`except:`块时，`Session`将处于干净状态，准备在全新事务上发出新的 SQL，并且调用`Session.rollback()`将不按顺序进行。特别是，此时`Session`已经开始了一个新事务，而`Session.rollback()`将错误地对其进行操作。在这个正常情况下应该进行回滚的地方，而不是允许 SQL 操作在新事务上继续进行，`Session`会拒绝继续直到显式回滚实际发生。

换句话说，期望调用代码**始终**调用 `Session.commit()`、`Session.rollback()` 或 `Session.close()` 来对应当前事务块。`flush()` 保持 `Session` 在这个事务块内，以便上述代码的行为是可预测和一致的。

## 如何使一个查询始终对每个查询添加某个过滤器？

请查看 [FilteredQuery](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/FilteredQuery) 的配方。

## 我的查询返回的对象数与 query.count() 告诉我的不一样 - 为什么？

当 `Query` 对象被要求返回一个 ORM 映射对象列表时，将**根据主键对对象进行去重**。也就是说，如果我们例如使用了在 使用 ORM 声明形式定义表元数据 中描述的 `User` 映射，并且我们有一个如下所示的 SQL 查询：

```py
q = session.query(User).outerjoin(User.addresses).filter(User.name == "jack")
```

在教程中使用的示例数据中，`addresses` 表中有两行数据，其中 `name` 为 `'jack'`、主键值为 5 的 `users` 行。如果我们要求上述查询的 `Query.count()`，我们将得到答案 **2**：

```py
>>> q.count()
2
```

但是，如果我们运行 `Query.all()` 或遍历查询，我们将得到**一个元素**：

```py
>>> q.all()
[User(id=5, name='jack', ...)]
```

这是因为当 `Query` 对象返回完整实体时，它们会被**去重**。如果我们改为请求单个列返回，则不会发生这种情况：

```py
>>> session.query(User.id, User.name).outerjoin(User.addresses).filter(
...     User.name == "jack"
... ).all()
[(5, 'jack'), (5, 'jack')]
```

`Query` 将去重的主要原因有两个：

+   **允许联接预加载正常工作** - 联接预加载通过使用与相关表的连接来查询行，然后将这些连接的行路由到主对象的集合中来工作。为了做到这一点，它必须获取主对象主键在每个子条目中重复的行。这种模式可以继续到更深层的子集合，以便为单个主对象（如`User(id=5)`）处理多行。去重允许我们按照查询的方式接收对象，例如所有名为'jack'的`User()`对象，对于我们来说是一个对象，其中`User.addresses`集合已经被急加载，就像在`relationship()`上通过`lazy='joined'`或通过`joinedload()`选项指示的那样。为了保持一致性，无论是否建立了联接加载，去重仍然适用，因为急加载背后的关键理念是这些选项永远不会影响结果。

+   **消除关于身份映射的混淆** - 这显然是较不重要的原因。由于`Session`使用了身份映射，即使我们的 SQL 结果集中有两行主键为 5 的记录，`Session` 中只有一个`User(id=5)`对象，必须在其身份上保持唯一，即其主键/类组合。如果一个查询`User()`对象，多次在列表中获取相同对象实际上并没有太多意义。有序集合可能更好地表示`Query` 在返回完整对象时所寻求的内容。

`Query`去重问题仍然存在问题，主要是因为`Query.count()`方法不一致，并且当前状态是最近的版本中的连接急加载首先被“子查询急加载”策略取代，最近是“select IN 急加载”策略，这两种策略通常更适合于集合急加载。随着这一演变的继续，SQLAlchemy 可能会更改 `Query`的行为，这也可能涉及新的 API，以更直接地控制此行为，并且也可能更改连接的急加载的行为，以创建更一致的使用模式。

## 我已经针对外连接创建了映射，但是虽然查询返回行，但没有返回对象。为什么？

由外连接返回的行可能包含主键的部分 NULL，因为主键是两个表的组合。`Query`对象忽略不具有可接受主键的传入行。根据`Mapper`上的`allow_partial_pks`标志的设置，如果该值至少具有一个非 NULL 值，则接受主键，或者如果该值没有 NULL 值，则接受该值。请参见 `Mapper`上的`allow_partial_pks`。

## 我正在使用`joinedload()`或`lazy=False`来创建 JOIN/OUTER JOIN，当我尝试添加 WHERE、ORDER BY、LIMIT 等条件时，SQLAlchemy 没有构造正确的查询。（这依赖于（OUTER）JOIN）

由连接的急加载生成的连接仅用于完全加载相关集合，并设计为不影响查询的主要结果。由于它们是匿名别名，因此不能直接引用。

关于这种行为的详细信息，请参见急加载的禅意。

## 查询没有`__len__()`，为什么？

应用于对象的 Python `__len__()` 魔术方法允许使用`len()`内置函数来确定集合的长度。直觉上，SQL 查询对象将`__len__()`链接到`Query.count()`方法，该方法发出 SELECT COUNT。不可能的原因是评估查询为列表将导致两次 SQL 调用而不是一次：

```py
class Iterates:
    def __len__(self):
        print("LEN!")
        return 5

    def __iter__(self):
        print("ITER!")
        return iter([1, 2, 3, 4, 5])

list(Iterates())
```

输出：

```py
ITER!
LEN!
```

## 如何在 ORM 查询中使用文本 SQL？

参见：

+   从文本语句获取 ORM 结果 - 使用 `Query` 进行即席文本块。

+   在会话中使用 SQL 表达式 - 直接使用 `Session` 进行文本 SQL 操作。

## 我调用 `Session.delete(myobject)`，但它没有从父集合中删除！

查看关于删除的注释 - 从集合和标量关系中删除对象以了解此行为的描述。

## 当加载对象时，为什么我的 `__init__()` 没有被调用？

查看跨加载保持非映射状态以了解此行为的描述。

## 我如何在 SA 的 ORM 中使用 ON DELETE CASCADE？

SQLAlchemy 总是对当前加载在 `Session` 中的依赖行发出 UPDATE 或 DELETE 语句。对于未加载的行，默认情况下会发出 SELECT 语句来加载这些行并更新/删除它们；换句话说，它假定没有配置 ON DELETE CASCADE。要配置 SQLAlchemy 与 ON DELETE CASCADE 协作，请参见使用 ORM 关系的外键 ON DELETE cascade。

## 我将实例的“foo_id”属性设置为“7”，但“foo”属性仍然为`None` - 它不应该加载具有 id #7 的 Foo 吗？

ORM 的构建不支持根据外键属性变化驱动的关系的立即填充 - 相反，它被设计成反向工作 - 外键属性由 ORM 在幕后处理，最终用户自然设置对象关系。因此，设置 `o.foo` 的推荐方式是做到这一点 - 设置它！：

```py
foo = session.get(Foo, 7)
o.foo = foo
Session.commit()
```

当然，对外键属性进行操作是完全合法的。但是，将外键属性设置为新值当前不会触发 `relationship()` 中涉及的“expire”事件。这意味着对于以下序列：

```py
o = session.scalars(select(SomeClass).limit(1)).first()

# assume the existing o.foo_id value is None;
# accessing o.foo will reconcile this as ``None``, but will effectively
# "load" the value of None
assert o.foo is None

# now set foo_id to something.  o.foo will not be immediately affected
o.foo_id = 7
```

当首次访问时，`o.foo` 会加载其有效的数据库值为 `None`。将 `o.foo_id = 7` 设置为挂起更改的值为“7”，但尚未刷新 - 因此 `o.foo` 仍然为 `None`：

```py
# attribute is already "loaded" as None, has not been
# reconciled with o.foo_id = 7 yet
assert o.foo is None
```

对于 `o.foo` 基于外键变化进行加载，通常在提交后自然实现，因为提交既刷新了新的外键值，又使所有状态过期：

```py
session.commit()  # expires all attributes

foo_7 = session.get(Foo, 7)

# o.foo will lazyload again, this time getting the new object
assert o.foo is foo_7
```

更精简的操作是单独过期属性 - 这可以为任何持久的对象执行，使用`Session.expire()`：

```py
o = session.scalars(select(SomeClass).limit(1)).first()
o.foo_id = 7
Session.expire(o, ["foo"])  # object must be persistent for this

foo_7 = session.get(Foo, 7)

assert o.foo is foo_7  # o.foo lazyloads on access
```

请注意，如果对象不是持久的但存在于`Session`中，则被称为待定。这意味着对象的行尚未插入到数据库中。对于这样的对象，设置`foo_id`在行被插入之前没有意义；否则还没有行：

```py
new_obj = SomeClass()
new_obj.foo_id = 7

Session.add(new_obj)

# returns None but this is not a "lazyload", as the object is not
# persistent in the DB yet, and the None value is not part of the
# object's state
assert new_obj.foo is None

Session.flush()  # emits INSERT

assert new_obj.foo is foo_7  # now it loads
```

该配方[ExpireRelationshipOnFKChange](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/ExpireRelationshipOnFKChange)展示了一个使用 SQLAlchemy 事件的示例，以协调设置具有多对一关系的外键属性。

## 如何遍历所有与给定对象相关联的对象？

具有其他对象相关联的对象将对应于设置在映射器之间的`relationship()`构造。此代码片段将迭代所有对象，并校正循环：

```py
from sqlalchemy import inspect

def walk(obj):
    deque = [obj]

    seen = set()

    while deque:
        obj = deque.pop(0)
        if obj in seen:
            continue
        else:
            seen.add(obj)
            yield obj
        insp = inspect(obj)
        for relationship in insp.mapper.relationships:
            related = getattr(obj, relationship.key)
            if relationship.uselist:
                deque.extend(related)
            elif related is not None:
                deque.append(related)
```

该函数可以演示如下：

```py
Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    c_id = Column(ForeignKey("c.id"))
    c = relationship("C", backref="bs")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)

a1 = A(bs=[B(), B(c=C())])

for obj in walk(a1):
    print(obj)
```

输出：

```py
<__main__.A object at 0x10303b190>
<__main__.B object at 0x103025210>
<__main__.B object at 0x10303b0d0>
<__main__.C object at 0x103025490>
```

## 是否有一种方法可以自动地仅获取唯一的关键词（或其他类型的对象），而不是对关键词进行查询并获取包含该关键词的行的引用？

当人们阅读文档中的多对多示例时，他们会遇到一个事实，即如果您两次创建相同的`Keyword`，它会被放入数据库两次。这有点不方便。

这个[UniqueObject](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/UniqueObject)配方是为了解决这个问题而创建的。

## 为什么 post_update 除了第一个 UPDATE 外还发出 UPDATE？

post_update 功能，在指向自身的行/相互依赖的行文档中记录，涉及在对特定关系绑定的外键进行更改时发出 UPDATE 语句，除了通常会对目标行发出的 INSERT/UPDATE/DELETE 之外。虽然这个 UPDATE 语句的主要目的是与 INSERT 或 DELETE 配对，以便它可以在 INSERT 或 DELETE 操作后设置或取消设置一个外键引用，以断开与相互依赖的外键的循环，但它目前也被捆绑为在目标行本身被更新时发出的第二个 UPDATE。在这种情况下，post_update 发出的 UPDATE *通常* 是不必要的，并且通常会显得浪费。

然而，对尝试删除这种“UPDATE / UPDATE”行为的一些研究表明，不仅需要在 post_update 实现中进行重大更改，而且还需要在与 post_update 不相关的区域进行更改，以使其工作，因为在某些情况下需要对非 post_update 部分的操作顺序进行反转，这反过来又会影响其他情况，例如正确处理引用主键值的 UPDATE（参见[#1063](https://www.sqlalchemy.org/trac/ticket/1063) 以获取概念验证）。

答案是，“post_update”用于打破两个相互依赖的外键之间的循环，并且使得此循环打破仅限于目标表的 INSERT/DELETE 意味着其他地方的 UPDATE 语句的顺序需要变得自由化，导致其他边缘情况的破坏。## 我正在使用我的会话重新加载数据，但它没有看到我在其他地方提交的更改

关于这种行为的主要问题是，会话的行为就像事务处于*可串行化*隔离状态一样，即使事务并不是（通常情况下并不是）。从实际角度来看，这意味着会话不会更改已经在事务范围内读取的任何数据。

如果术语“隔离级别”不熟悉，那么您首先需要阅读此链接：

[隔离级别](https://en.wikipedia.org/wiki/Isolation_%28database_systems%29)

简而言之，可串行化隔离级别通常意味着一旦您在事务中选择一系列行，您每次重新发出该 SELECT 时都会得到*相同的数据*。如果您处于较低的隔离级别，例如“可重复读”，您将看到新添加的行（不再看到删除的行），但对于您已经*加载*的行，您不会看到任何更改。只有当您处于较低的隔离级别时，例如“读取已提交的”，才有可能看到数据行更改其值。

关于在使用 SQLAlchemy ORM 时控制隔离级别的信息，请参阅设置事务隔离级别 / DBAPI AUTOCOMMIT。

要极大地简化事情，`Session` 本身是在完全隔离的事务中运行的，并且不会覆盖任何已经读取的映射属性，除非你告诉它这样做。在进行中的事务中尝试重新读取已经加载的数据的用例是一个*不常见*的用例，在许多情况下没有效果，因此这被认为是例外而不是规范；为了在这个例外中工作，提供了几种方法，允许在进行中的事务的上下文中重新加载特定的数据。

当我们谈论`Session`时，理解我们所说的“事务”是什么意思，你的`Session`只能在事务内工作。关于此的概述请参阅管理事务。

一旦我们确定了我们的隔离级别，并且我们认为我们的隔离级别设置得足够低，以至于如果我们重新选择一行，我们应该在我们的`Session`中看到新数据，那我们如何看到它呢？

三种方式，从最常见到最不常见：

1.  我们只需结束当前事务，并在下一次访问时通过调用`Session.commit()`（请注意，如果`Session`处于较少使用的“自动提交”模式，则还将调用`Session.begin()`）。绝大多数应用程序和用例不会出现无法在其他事务中“看到”数据的问题，因为它们遵循这种模式，这是**短事务**最佳实践的核心。有关此问题的一些想法，请参阅我何时构造一个会话，何时提交它，何时关闭它？。

1.  我们告诉我们的`Session`重新读取已经读取的行，要么在下次查询它们时使用`Session.expire_all()`或`Session.expire()`，要么立即在对象上使用`refresh`。有关此操作的详细信息，请参阅刷新/过期。

1.  我们可以在设置了“填充现有”选项的情况下运行整个查询，以确保在读取行时覆盖已加载的对象。这是一种在填充现有中描述的执行选项。

但请记住，**如果我们的隔离级别是可重复读或更高级别，ORM 无法看到行中的更改，除非我们启动一个新的事务**。

## “此会话的事务由于刷新期间的先前异常已被回滚。”（或类似内容）

当`Session.flush()`引发异常，回滚事务，但在未显式调用`Session.rollback()`或`Session.close()`的情况下调用`Session`上的进一步命令时，就会发生这种错误。

这通常对应于一个应用程序在`Session.flush()`或`Session.commit()`上捕获异常，但未正确处理异常。 例如：

```py
from sqlalchemy import create_engine, Column, Integer
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base(create_engine("sqlite://"))

class Foo(Base):
    __tablename__ = "foo"
    id = Column(Integer, primary_key=True)

Base.metadata.create_all()

session = sessionmaker()()

# constraint violation
session.add_all([Foo(id=1), Foo(id=1)])

try:
    session.commit()
except:
    # ignore error
    pass

# continue using session without rolling back
session.commit()
```

使用`Session`应该符合类似于这样的结构：

```py
try:
    # <use session>
    session.commit()
except:
    session.rollback()
    raise
finally:
    session.close()  # optional, depends on use case
```

许多事情除了刷新之外，都可能导致 try/except 中的失败。 应用程序应确保对 ORM 导向的进程应用某种“框架”系统，以便连接和事务资源具有明确定界，并且如果发生任何失败条件，则可以显式回滚事务。

这并不意味着整个应用程序中应该到处都是 try/except 块，这不是可扩展的架构。 相反，一个典型的方法是，当首次调用 ORM 导向的方法和函数时，从最顶层调用函数的进程将在成功完成一系列操作时提交事务，并且如果操作因任何原因失败，包括失败的刷新，则回滚事务。 还有使用函数装饰器或上下文管理器来实现类似结果的方法。 采取的方法取决于正在编写的应用程序的类型。

有关如何组织使用`Session`的详细讨论，请参见何时构建会话，何时提交会话，何时关闭会话？。

### 但为什么 flush()坚持发出 ROLLBACK？

如果 `Session.flush()` 能部分完成然后不回滚，那将会很好，但是由于它当前的能力限制，这是不可能的，因为它的内部记录必须被修改，以便随时停止，并且与已刷新到数据库的内容完全一致。虽然这在理论上是可能的，但增强功能的有用性大大降低了，因为许多数据库操作在任何情况下都需要回滚。特别是 Postgres 有一些操作，一旦失败，事务就不允许继续：

```py
test=> create table foo(id integer primary key);
NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "foo_pkey" for table "foo"
CREATE TABLE
test=> begin;
BEGIN
test=> insert into foo values(1);
INSERT 0 1
test=> commit;
COMMIT
test=> begin;
BEGIN
test=> insert into foo values(1);
ERROR:  duplicate key value violates unique constraint "foo_pkey"
test=> insert into foo values(2);
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

SQLAlchemy 提供的解决这两个问题的方法是通过支持 SAVEPOINT，通过 `Session.begin_nested()`。使用 `Session.begin_nested()`，您可以在事务中设置一个可能会失败的操作，然后在保持封闭事务的同时“回滚”到其失败之前的点。

### 但为什么一次自动调用 ROLLBACK 不够？为什么我还必须再次 ROLLBACK？

由 flush() 引起的回滚并不是完整事务块的结束；尽管它结束了正在进行的数据库事务，但从 `Session` 的角度来看，仍然存在一个处于非活动状态的事务。

鉴于这样的代码块：

```py
sess = Session()  # begins a logical transaction
try:
    sess.flush()

    sess.commit()
except:
    sess.rollback()
```

在上面的例子中，当一个 `Session` 第一次被创建时，假设没有使用“自动提交模式”，则在 `Session` 内建立了一个逻辑事务。这个事务是“逻辑”的，因为它实际上并不使用任何数据库资源，直到调用 SQL 语句时，此时会启动一个连接级别和 DBAPI 级别的事务。然而，无论数据库级别的事务是否是其状态的一部分，逻辑事务都会保持不变，直到使用 `Session.commit()`、`Session.rollback()` 或 `Session.close()` 结束它。

当上面的`flush()`失败时，代码仍位于由 try/commit/except/rollback 块框定的事务内。 如果`flush()`完全回滚逻辑事务，那么当我们到达`except:`块时，`Session`将处于干净状态，准备在全新的事务上发出新的 SQL，并且对`Session.rollback()`的调用将处于不正确的顺序。 特别是，到这一点为止，`Session`已经开始了一个新的事务，而`Session.rollback()`将错误地对其进行操作。 与其允许 SQL 操作在此处继续新事务，而正常用法规定要进行回滚的地方，则`Session`拒绝继续，直到显式回滚实际发生。

换句话说，预期调用代码将**始终**调用`Session.commit()`、`Session.rollback()`或`Session.close()`与当前事务块对应。 `flush()`保持`Session`在此事务块中，以便上述代码的行为可预测且一致。

### 但为什么`flush()`坚持要发出一个 ROLLBACK 呢？

如果`Session.flush()`可以部分完成然后不回滚，那将是很好的，但是由于其当前能力范围之外，因为其内部记账必须被修改，以便它可以随时停止，并且与已经刷新到数据库的内容完全一致。 尽管理论上可能，但增强功能的实用性大大降低了，因为许多数据库操作无论如何都要求回滚。 特别是，Postgres 有一些操作，一旦失败，就不允许事务继续：

```py
test=> create table foo(id integer primary key);
NOTICE:  CREATE TABLE / PRIMARY KEY will create implicit index "foo_pkey" for table "foo"
CREATE TABLE
test=> begin;
BEGIN
test=> insert into foo values(1);
INSERT 0 1
test=> commit;
COMMIT
test=> begin;
BEGIN
test=> insert into foo values(1);
ERROR:  duplicate key value violates unique constraint "foo_pkey"
test=> insert into foo values(2);
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

SQLAlchemy 提供解决这两个问题的方法是通过 `Session.begin_nested()` 支持 SAVEPOINT。使用 `Session.begin_nested()`，您可以在事务中执行一个可能会失败的操作，然后在维持封闭事务的同时“回滚”到失败之前的状态。

### 但为什么一次自动调用 ROLLBACK 不够？为什么我必须再次 ROLLBACK？

由 flush() 引起的回滚不是完整事务块的结束；虽然它结束了正在进行的数据库事务，在`Session`的视角下仍然存在一个现在处于不活动状态的事务。

给定一个如下的块：

```py
sess = Session()  # begins a logical transaction
try:
    sess.flush()

    sess.commit()
except:
    sess.rollback()
```

在上述情况中，当首次创建一个`Session`时，假设没有使用“自动提交模式”，则在`Session`内建立了一个逻辑事务。该事务是“逻辑”的，因为它实际上不使用任何数据库资源，直到调用 SQL 语句，此时开始连接级和 DBAPI 级的事务。但是，无论数据库级事务是否是其状态的一部分，逻辑事务将保持不变，直到使用`Session.commit()`、`Session.rollback()`或`Session.close()`结束为止。

当上面的`flush()`失败时，代码仍然处于由 try/commit/except/rollback 块框定的事务中。如果`flush()`完全回滚了逻辑事务，这意味着当我们到达`except:`块时，`Session`将处于干净的状态，准备在一个全新的事务中发出新的 SQL，并且对`Session.rollback()`的调用将会处于顺序错误的状态。特别是，`Session`此时已经开始了一个新的事务，而`Session.rollback()`将在错误地执行。与其在这个地方允许 SQL 操作在新的事务中进行，而正常使用指示将要进行回滚的地方，则`Session`拒绝继续，直到显式回滚实际发生为止。

换句话说，期望调用代码**始终**调用`Session.commit()`、`Session.rollback()`或`Session.close()`与当前事务块相对应。`flush()`保持`Session`在这个事务块内，以便上述代码的行为是可预测且一致的。

## 如何创建一个始终向每个查询添加特定过滤器的查询？

参见[FilteredQuery](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/FilteredQuery)中的配方。

## 我的查询返回的对象数量与 query.count() 告诉我的数量不一样 - 为什么？

当`Query`对象被要求返回一个 ORM 映射对象列表时，将根据主键对对象进行**去重**。也就是说，如果我们例如使用了在使用 ORM 声明形式定义表元数据中描述的`User`映射，并且我们有一个如下的 SQL 查询：

```py
q = session.query(User).outerjoin(User.addresses).filter(User.name == "jack")
```

在上面的例子中，教程中使用的样例数据在`addresses`表中有两行，对应于名为`'jack'`的`users`行，主键值为 5。如果我们对上述查询使用`Query.count()`，我们将得到答案**2**：

```py
>>> q.count()
2
```

然而，如果我们运行`Query.all()`或者迭代查询，我们会得到**一个元素**：

```py
>>> q.all()
[User(id=5, name='jack', ...)]
```

这是因为当`Query`对象返回完整实体时，它们会被**去重**。如果我们请求单个列返回，则不会发生这种情况：

```py
>>> session.query(User.id, User.name).outerjoin(User.addresses).filter(
...     User.name == "jack"
... ).all()
[(5, 'jack'), (5, 'jack')]
```

`Query`会进行去重的两个主要原因有：

+   **允许连接式贪婪加载正常工作** - 连接式贪婪加载通过使用与相关表的连接查询行，然后将这些连接查询行路由到导航对象的集合中来工作。为了做到这一点，它必须获取重复了主导对象主键的行，以便每个子条目。这种模式可以继续到更进一步的子集合，以便为单个主导对象，如`User(id=5)`，处理多行。去重允许我们按照查询时的方式接收对象，例如，所有`User()`对象其名称为`'jack'`，对我们来说是一个对象，并且`User.addresses`集合被贪婪加载，就像在`relationship()`上使用`lazy='joined'`或通过`joinedload()`选项指示的那样。为了保持一致性，去重仍然适用于是否已建立连接加载，因为贪婪加载的核心理念是这些选项从不影响结果。

+   **消除关于身份映射的混淆** - 这显然是较不重要的原因。由于`Session`使用了一个身份映射，即使我们的 SQL 结果集有两行主键为 5 的记录，`Session`内也只有一个`User(id=5)`对象，必须以其身份唯一性进行维护，即其主键/类组合。如果查询`User()`对象，获取相同对象多次在列表中实际上没有太多意义。有序集合可能更能代表`Query`在返回完整对象时所寻求的内容。

`Query` 去重的问题仍然存在问题，主要原因是 `Query.count()` 方法不一致，当前状态是，在最近的发布中，联合急加载首先被“子查询急加载”策略所取代，更近期的是“选择 IN 急加载”策略，这两者通常更适用于集合急加载。随着这种演变的继续，SQLAlchemy 可能会改变 `Query` 的行为，这也可能涉及到新的 API，以更直接地控制这种行为，并且还可能改变联合急加载的行为，以创建更一致的使用模式。

## 我已经创建了一个针对 Outer Join 的映射，虽然查询返回了行，但没有返回对象。为什么？

外部连接返回的行可能会对主键的某部分包含 NULL，因为主键是两个表的组合。`Query` 对象忽略那些没有可接受主键的传入行。根据 `Mapper` 上 `allow_partial_pks` 标志的设置，如果值至少有一个非 NULL 值，则接受主键，或者如果值没有 NULL 值，则接受主键。请参阅 `Mapper` 上的 `allow_partial_pks`。

## 当我尝试添加 WHERE、ORDER BY、LIMIT 等条件（这依赖于（外部）JOIN）时，我使用 `joinedload()` 或 `lazy=False` 创建了一个 JOIN/OUTER JOIN，但 SQLAlchemy 在构造查询时出现了问题。

由联合急加载生成的连接仅用于完全加载相关集合，并且设计为不会影响查询的主要结果。由于它们是匿名别名，因此不能直接引用。

关于这种行为的详细信息，请参见 Joined Eager Loading 的禅意。

## Query 没有 `__len__()`，为什么？

Python 中的 `__len__()` 魔法方法应用于对象，允许使用 `len()` 内置函数来确定集合的长度。很直观地，一个 SQL 查询对象会将 `__len__()` 关联到 `Query.count()` 方法，该方法会发出一个 SELECT COUNT。然而，不可能做到这一点的原因是因为将查询作为列表进行评估会导致两个 SQL 调用而不是一个：

```py
class Iterates:
    def __len__(self):
        print("LEN!")
        return 5

    def __iter__(self):
        print("ITER!")
        return iter([1, 2, 3, 4, 5])

list(Iterates())
```

输出：

```py
ITER!
LEN!
```

## 如何在 ORM 查询中使用 Textual SQL？

请参阅：

+   从文本语句获取 ORM 结果 - 使用 `Query` 进行自定义文本块。

+   使用 SQL 表达式与会话 - 直接使用文本 SQL 与 `Session`。

## 我调用`Session.delete(myobject)`但它没有从父集合中删除！

有关此行为的描述，请参阅 关于删除的说明 - 从集合和标量关系引用的对象删除。

## 当我加载对象时，为什么我的`__init__()`没有被调用？

有关此行为的描述，请参阅 跨加载保持非映射状态。

## 我如何在 SA 的 ORM 中使用 ON DELETE CASCADE？

SQLAlchemy 总是针对当前加载在 `Session` 中的依赖行发出 UPDATE 或 DELETE 语句。对于未加载的行，默认情况下会发出 SELECT 语句来加载这些行，并对其进行更新/删除；换句话说，它假定未配置 ON DELETE CASCADE。要配置 SQLAlchemy 以配合 ON DELETE CASCADE，请参阅 使用 ORM 关系的外键 ON DELETE cascade。

## 我将我的实例的“foo_id”属性设置为“7”，但“foo”属性仍然为`None` - 它不应该加载 ID 为#7 的 Foo 吗？

ORM 并非以支持从外键属性更改驱动的关系的即时填充方式构建的 - 相反，它设计为以相反的方式工作 - 外键属性由 ORM 在幕后处理，最终用户自然设置对象关系。因此，设置`o.foo`的推荐方法就是这样 - 设置它！：

```py
foo = session.get(Foo, 7)
o.foo = foo
Session.commit()
```

当然，操作外键属性是完全合法的。但是，目前设置外键属性为新值不会触发其中涉及的 `relationship()` 的“过期”事件。这意味着对于以下序列：

```py
o = session.scalars(select(SomeClass).limit(1)).first()

# assume the existing o.foo_id value is None;
# accessing o.foo will reconcile this as ``None``, but will effectively
# "load" the value of None
assert o.foo is None

# now set foo_id to something.  o.foo will not be immediately affected
o.foo_id = 7
```

当首次访问时，`o.foo`加载为其有效的数据库值`None`。设置`o.foo_id = 7`将使值“7”作为挂起更改，但尚未刷新 - 因此`o.foo`仍然为`None`：

```py
# attribute is already "loaded" as None, has not been
# reconciled with o.foo_id = 7 yet
assert o.foo is None
```

对于`o.foo`的加载，基于外键变异通常在提交后自然实现，这既刷新了新的外键值，也使所有状态失效：

```py
session.commit()  # expires all attributes

foo_7 = session.get(Foo, 7)

# o.foo will lazyload again, this time getting the new object
assert o.foo is foo_7
```

一个更简单的操作是单独使属性过期 - 这可以针对任何 persistent 对象使用`Session.expire()`:

```py
o = session.scalars(select(SomeClass).limit(1)).first()
o.foo_id = 7
Session.expire(o, ["foo"])  # object must be persistent for this

foo_7 = session.get(Foo, 7)

assert o.foo is foo_7  # o.foo lazyloads on access
```

请注意，如果对象不是持久的但存在于`Session`中，则称为 pending。这意味着对象的行尚未 INSERT 到数据库中。对于这样的对象，设置`foo_id`在行被插入之前没有意义；否则还没有行：

```py
new_obj = SomeClass()
new_obj.foo_id = 7

Session.add(new_obj)

# returns None but this is not a "lazyload", as the object is not
# persistent in the DB yet, and the None value is not part of the
# object's state
assert new_obj.foo is None

Session.flush()  # emits INSERT

assert new_obj.foo is foo_7  # now it loads
```

这个方案[ExpireRelationshipOnFKChange](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/ExpireRelationshipOnFKChange)提供了一个使用 SQLAlchemy 事件的示例，以便协调与多对一关系中的外键属性的设置。

## 如何遍历与给定对象相关的所有对象？

与之相关的其他对象的对象将与映射器之间设置的`relationship()`构造相对应。这段代码片段将迭代所有对象，纠正循环：

```py
from sqlalchemy import inspect

def walk(obj):
    deque = [obj]

    seen = set()

    while deque:
        obj = deque.pop(0)
        if obj in seen:
            continue
        else:
            seen.add(obj)
            yield obj
        insp = inspect(obj)
        for relationship in insp.mapper.relationships:
            related = getattr(obj, relationship.key)
            if relationship.uselist:
                deque.extend(related)
            elif related is not None:
                deque.append(related)
```

函数可以如下所示演示：

```py
Base = declarative_base()

class A(Base):
    __tablename__ = "a"
    id = Column(Integer, primary_key=True)
    bs = relationship("B", backref="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    c_id = Column(ForeignKey("c.id"))
    c = relationship("C", backref="bs")

class C(Base):
    __tablename__ = "c"
    id = Column(Integer, primary_key=True)

a1 = A(bs=[B(), B(c=C())])

for obj in walk(a1):
    print(obj)
```

输出：

```py
<__main__.A object at 0x10303b190>
<__main__.B object at 0x103025210>
<__main__.B object at 0x10303b0d0>
<__main__.C object at 0x103025490>
```

## 有没有一种方法可以自动地只有唯一的关键词（或其他类型的对象），而不必查询关键词并获取包含该关键词的行的引用呢？

当人们阅读文档中的多对多示例时，他们会发现如果您创建相同的`Keyword`两次，它会在数据库中出现两次。这有点不方便。

这个[UniqueObject](https://www.sqlalchemy.org/trac/wiki/UsageRecipes/UniqueObject)方案是为了解决这个问题而创建的。

## 为什么 post_update 除了第一个 UPDATE 之外还会发出 UPDATE？

该特性，详细说明请参见指向自身的行/相互依赖行，会在特定关系绑定的外键发生更改时发出 UPDATE 语句，除了会针对目标行通常发出的 INSERT/UPDATE/DELETE 之外。虽然此 UPDATE 语句的主要目的是与该行的 INSERT 或 DELETE 配对，以便它可以在后设置或前取消外键引用，以打破与相互依赖的外键的循环，但目前它也被捆绑为第二个 UPDATE，当目标行本身被 UPDATE 时发出。在这种情况下，post_update 发出的 UPDATE *通常* 是不必要的，并且通常会显得浪费。

然而，一些研究试图消除这种“UPDATE / UPDATE”行为的努力表明，不仅需要在 post_update 实现中进行重大更改，还需要在与 post_update 无关的领域进行一些变更，以使其生效，因为在某些情况下，非 post_update 方面的操作顺序需要被颠倒，这反过来可能会影响其他情况，比如正确处理引用主键值的 UPDATE（参见[#1063](https://www.sqlalchemy.org/trac/ticket/1063)以获取概念验证）。

答案是，“post_update”用于打破两个相互依赖的外键之间的循环，并且使得这种循环打破仅限于目标表的 INSERT/DELETE 意味着其他地方 UPDATE 语句的排序需要被放宽，导致其他边缘情况的破坏。
