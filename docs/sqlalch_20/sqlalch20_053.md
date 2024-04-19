# 事务和连接管理

> 原文：[`docs.sqlalchemy.org/en/20/orm/session_transaction.html`](https://docs.sqlalchemy.org/en/20/orm/session_transaction.html)

## 事务管理

在 1.4 版本中更改：会话事务管理已经修订为更清晰、更易于使用。特别是，它现在具有“自动开始”操作，这意味着可以控制事务开始的时间点，而不使用传统的“自动提交”模式。

`Session`一次跟踪单个“虚拟”事务的状态，使用一个称为`SessionTransaction`的对象。然后，此对象利用绑定到`Session`对象的基础`Engine`或引擎，根据需要使用`Connection`对象开始真实的连接级事务。

此“虚拟”事务在需要时会自动创建，或者可以使用`Session.begin()`方法启动。尽可能地支持 Python 上下文管理器的使用，既在创建`Session`对象的层面，也在维护`SessionTransaction`的范围方面。

假设我们从`Session`开始：

```py
from sqlalchemy.orm import Session

session = Session(engine)
```

我们现在可以使用上下文管理器在一个确定的事务中运行操作：

```py
with session.begin():
    session.add(some_object())
    session.add(some_other_object())
# commits transaction at the end, or rolls back if there
# was an exception raised
```

在上下文结束时，假设没有引发任何异常，则任何待处理的对象都将被刷新到数据库，并且数据库事务将被提交。如果在上述块内引发了异常，则事务将被回滚。在这两种情况下，上述`Session`在退出块后都已准备好用于后续的事务。

`Session.begin()`方法是可选的，`Session`也可以使用按需自动开始事务的“随时提交”方法；这些只需要提交或回滚：

```py
session = Session(engine)

session.add(some_object())
session.add(some_other_object())

session.commit()  # commits

# will automatically begin again
result = session.execute(text("< some select statement >"))
session.add_all([more_objects, ...])
session.commit()  # commits

session.add(still_another_object)
session.flush()  # flush still_another_object
session.rollback()  # rolls back still_another_object
```

`Session` 本身具有一个 `Session.close()` 方法。如果 `Session` 是在尚未提交或回滚的事务内开始的，该方法将取消（即回滚）该事务，并清除 `Session` 对象状态中包含的所有对象。如果使用 `Session` 的方式不能保证调用 `Session.commit()` 或 `Session.rollback()`（例如，不在上下文管理器或类似结构中），则可以使用 `close` 方法来确保释放所有资源：

```py
# expunges all objects, releases all transactions unconditionally
# (with rollback), releases all database connections back to their
# engines
session.close()
```

最后，会话的构建/关闭过程本身也可以通过上下文管理器运行。这是确保 `Session` 对象使用范围在一个固定块内的最佳方式。首先通过 `Session` 构造函数进行说明：

```py
with Session(engine) as session:
    session.add(some_object())
    session.add(some_other_object())

    session.commit()  # commits

    session.add(still_another_object)
    session.flush()  # flush still_another_object

    session.commit()  # commits

    result = session.execute(text("<some SELECT statement>"))

# remaining transactional state from the .execute() call is
# discarded
```

同样，`sessionmaker` 可以以相同的方式使用：

```py
Session = sessionmaker(engine)

with Session() as session:
    with session.begin():
        session.add(some_object)
    # commits

# closes the Session
```

`sessionmaker` 本身包含一个 `sessionmaker.begin()` 方法，允许同时进行两个操作：

```py
with Session.begin() as session:
    session.add(some_object)
```

### 使用 SAVEPOINT

如果底层引擎支持，可以使用 `Session.begin_nested()` 方法来划定 SAVEPOINT 事务：

```py
Session = sessionmaker()

with Session.begin() as session:
    session.add(u1)
    session.add(u2)

    nested = session.begin_nested()  # establish a savepoint
    session.add(u3)
    nested.rollback()  # rolls back u3, keeps u1 and u2

# commits u1 and u2
```

每次调用 `Session.begin_nested()`，都会在当前数据库事务的范围内向数据库发出一个新的“BEGIN SAVEPOINT”命令（如果尚未开始，则开始一个），并返回一个类型为 `SessionTransaction` 的对象，代表这个 SAVEPOINT 的句柄。当调用该对象的 `.commit()` 方法时，会向数据库发出“RELEASE SAVEPOINT”，如果调用 `.rollback()` 方法，则会发出“ROLLBACK TO SAVEPOINT”。封闭的数据库事务仍在进行中。

`Session.begin_nested()` 通常用作上下文管理器，可以捕获特定的每个实例错误，并在该事务状态的部分发出回滚，而不会回滚整个事务，如下例所示：

```py
for record in records:
    try:
        with session.begin_nested():
            session.merge(record)
    except:
        print("Skipped record %s" % record)
session.commit()
```

当由 `Session.begin_nested()` 产生的上下文管理器完成时，“提交” savepoint，其中包括刷新所有挂起状态的常规行为。当出现错误时，保存点会被回滚，并且更改的对象的 `Session` 本地状态会过期。

这种模式非常适用于诸如使用 PostgreSQL 并捕获 `IntegrityError` 来检测重复行的情况；通常情况下，当出现此类错误时，PostgreSQL 会中止整个事务，但是使用 SAVEPOINT 时，外部事务会得以保留。在下面的示例中，将一系列数据持久化到数据库中，并且偶尔会跳过“重复的主键”记录，而无需回滚整个操作：

```py
from sqlalchemy import exc

with session.begin():
    for record in records:
        try:
            with session.begin_nested():
                obj = SomeRecord(id=record["identifier"], name=record["name"])
                session.add(obj)
        except exc.IntegrityError:
            print(f"Skipped record {record} - row already exists")
```

当调用 `Session.begin_nested()` 时，`Session` 首先会将当前所有挂起的状态刷新到数据库；这种情况会无条件发生，不管 `Session.autoflush` 参数的值是什么，该参数通常用于禁用自动刷新。这种行为的原因是，当此嵌套事务发生回滚时，`Session` 可以使在 SAVEPOINT 范围内创建的任何内存状态过期，同时确保在刷新这些过期对象时，SAVEPOINT 开始之前的对象图状态将可用于重新从数据库加载。

在 SQLAlchemy 的现代版本中，当由 `Session.begin_nested()` 启动的 SAVEPOINT 被回滚时，自 SAVEPOINT 创建以来已修改的内存对象状态会过期，但自 SAVEPOINT 开始后未修改的其他对象状态将保持不变。这样，后续操作可以继续使用其它未受影响的数据，而无需从数据库刷新。

另请参见

`Connection.begin_nested()` - 核心 SAVEPOINT API  ### 会话级别与引擎级别的事务控制

在核心中的`Connection`和 ORM 中的`_session.Session`具有等效的事务语义，无论是在`sessionmaker`与`Engine`的级别，还是在`Session`与`Connection`的级别。以下部分根据以下方案详细说明这些情况：

```py
ORM                                           Core
-----------------------------------------     -----------------------------------
sessionmaker                                  Engine
Session                                       Connection
sessionmaker.begin()                          Engine.begin()
some_session.commit()                         some_connection.commit()
with some_sessionmaker() as session:          with some_engine.connect() as conn:
with some_sessionmaker.begin() as session:    with some_engine.begin() as conn:
with some_session.begin_nested() as sp:       with some_connection.begin_nested() as sp:
```

#### 边提交边进行

`Session`和`Connection`都具有`Connection.commit()`和`Connection.rollback()`方法。使用 SQLAlchemy 2.0 风格的操作，这些方法在所有情况下都会影响**最外层**的事务。对于`Session`，假定`Session.autobegin`保持默认值`True`。

`Engine`:

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.connect() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    conn.commit()
```

`Session`:

```py
Session = sessionmaker(engine)

with Session() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    session.commit()
```

#### 一次开始

`sessionmaker`和`Engine`都具有`Engine.begin()`方法，该方法将获取一个新对象来执行 SQL 语句（分别是`Session`和`Connection`），然后返回一个上下文管理器，用于维护该对象的开始/提交/回滚上下文。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
# commits and closes automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
# commits and closes automatically
```

#### 嵌套事务

使用 SAVEPOINT 通过 `Session.begin_nested()` 或 `Connection.begin_nested()` 方法时，必须使用返回的事务对象来提交或回滚 SAVEPOINT。 调用 `Session.commit()` 或 `Connection.commit()` 方法将始终提交**最外层**事务；这是 SQLAlchemy 2.0 特定的行为，与 1.x 系列相反。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    savepoint = conn.begin_nested()
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    savepoint.commit()  # or rollback

# commits automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    savepoint = session.begin_nested()
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    savepoint.commit()  # or rollback
# commits automatically
```  ### 显式开始

`Session` 具有“自动开始”行为，这意味着一旦操作开始进行，它就会确保存在一个 `SessionTransaction` 来跟踪正在进行的操作。 当调用 `Session.commit()` 时，此事务将完成。

通常希望，特别是在框架集成中，控制“开始”操作发生的时间点。 为此，`Session` 使用“自动开始”策略，使得可以直接调用 `Session.begin()` 方法，以便为尚未启动事务的 `Session` 调用：

```py
Session = sessionmaker(bind=engine)
session = Session()
session.begin()
try:
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
    session.commit()
except:
    session.rollback()
    raise
```

上述模式更常用地使用上下文管理器调用：

```py
Session = sessionmaker(bind=engine)
session = Session()
with session.begin():
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
```

`Session.begin()` 方法和会话的“自动开始”过程使用相同的步骤序列开始事务。 这包括在发生时调用 `SessionEvents.after_transaction_create()` 事件；此挂钩被框架用于将其自己的事务处理过程与 ORM `Session` 集成。

对于支持两阶段操作的后端（目前支持 MySQL 和 PostgreSQL），会话可以被指示使用两阶段提交语义。这将协调跨数据库的事务提交，以便在所有数据库中要么提交事务，要么回滚事务。您还可以`Session.prepare()` 会话以与 SQLAlchemy 不管理的事务进行交互。要使用两阶段事务，请在会话上设置标志 `twophase=True`：

```py
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker(twophase=True)

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()

# .... work with accounts and users

# commit.  session will issue a flush to all DBs, and a prepare step to all DBs,
# before committing both transactions
session.commit()
```  ### 设置事务隔离级别 / DBAPI 自动提交

大多数 DBAPI 支持可配置的事务隔离级别的概念。传统上有四个级别：“READ UNCOMMITTED”，“READ COMMITTED”，“REPEATABLE READ” 和 “SERIALIZABLE”。这些通常应用于 DBAPI 连接在开始新事务之前，注意大多数 DBAPI 在首次发出 SQL 语句时会隐式开始此事务。

支持隔离级别的 DBAPI 通常也支持真正的 “自动提交” 概念，这意味着 DBAPI 连接本身将被置于非事务自动提交模式。这通常意味着数据库自动发出 “BEGIN” 的典型 DBAPI 行为不再发生，但它也可能包括其他指令。在使用此模式时，**DBAPI 在任何情况下都不使用事务**。SQLAlchemy 方法如 `.begin()`、`.commit()` 和 `.rollback()` 将静默传递。

SQLAlchemy 的方言支持在每个 `Engine` 或每个 `Connection` 基础上设置可设置的隔离模式，使用 `create_engine()` 层级以及 `Connection.execution_options()` 层级的标志。

当使用 ORM `Session` 时，它充当引擎和连接的 *facade*，但不直接暴露事务隔离。因此，为了影响事务隔离级别，我们需要根据情况对 `Engine` 或 `Connection` 采取行动。

另请参阅

设置事务隔离级别包括 DBAPI 自动提交 - 请确保查看 SQLAlchemy `Connection` 对象的隔离级别工作方式。

#### 为 Sessionmaker / Engine 设置隔离

要为特定的隔离级别全局设置`Session`或`sessionmaker`，第一种技术是可以针对所有情况构建一个具有特定隔离级别的`Engine`，然后将其用作`Session`和/或`sessionmaker`的连接源：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    isolation_level="REPEATABLE READ",
)

Session = sessionmaker(eng)
```

另一个选项，如果同时存在具有不同隔离级别的两个引擎，是使用`Engine.execution_options()`方法，该方法将生成一个原始`Engine`的浅拷贝，该浅拷贝与父引擎共享相同的连接池。当操作将被分成“事务”和“自动提交”操作时，这通常是更可取的：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

transactional_session = sessionmaker(eng)
autocommit_session = sessionmaker(autocommit_engine)
```

在上述示例中，“`eng`”和`"autocommit_engine"`共享相同的方言和连接池。然而，当从`autocommit_engine`获取连接时，将设置“AUTOCOMMIT”模式。这两个`sessionmaker`对象“`transactional_session`”和“`autocommit_session`”在与数据库连接工作时继承这些特性。

“`autocommit_session`” **仍然具有事务语义**，包括`Session.commit()`和`Session.rollback()`仍然认为自己在“提交”和“回滚”对象，然而事务将会默默地不存在。因此，**通常情况下，尽管不是严格要求，使用 AUTOCOMMIT 隔离的会话应该以只读方式使用**，即：

```py
with autocommit_session() as session:
    some_objects = session.execute(text("<statement>"))
    some_other_objects = session.execute(text("<statement>"))

# closes connection
```

#### 设置单个会话的隔离级别

当我们创建一个新的`Session`，可以直接使用构造函数，也可以在调用由`sessionmaker`生成的可调用对象时，直接传递`bind`参数，覆盖预先存在的绑定。例如，我们可以从默认的`sessionmaker`创建我们的`Session`并传递一个设置为自动提交的引擎：

```py
plain_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = plain_engine.execution_options(isolation_level="AUTOCOMMIT")

# will normally use plain_engine
Session = sessionmaker(plain_engine)

# make a specific Session that will use the "autocommit" engine
with Session(bind=autocommit_engine) as session:
    # work with session
    ...
```

对于`Session`或`sessionmaker`配置了多个`binds`的情况，我们可以重新完整指定`binds`参数，或者如果我们只想替换特定的 binds，则可以使用`Session.bind_mapper()`或`Session.bind_table()`方法：

```py
with Session() as session:
    session.bind_mapper(User, autocommit_engine)
```

#### 为个别事务设置隔离级别

关于隔离级别的一个关键警告是，在已经开始事务的`Connection`上不能安全地修改设置。数据库不能在进行中的事务中更改隔离级别，而一些 DBAPIs 和 SQLAlchemy 方言在这方面的行为不一致。

因此，最好使用一个提前绑定到具有所需隔离级别的引擎的`Session`。然而，通过在事务开始时使用`Session.connection()`方法可以影响每个连接的隔离级别：

```py
from sqlalchemy.orm import Session

# assume session just constructed
sess = Session(bind=engine)

# call connection() with options before any other operations proceed.
# this will procure a new connection from the bound engine and begin a real
# database transaction.
sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

# ... work with session in SERIALIZABLE isolation level...

# commit transaction.  the connection is released
# and reverted to its previous isolation level.
sess.commit()

# subsequent to commit() above, a new transaction may be begun if desired,
# which will proceed with the previous default isolation level unless
# it is set again.
```

在上面的例子中，我们首先使用构造函数或`sessionmaker`生成一个`Session`。然后，我们通过调用`Session.connection()`显式设置数据库级事务的开始，该方法提供了在数据库级事务开始之前将传递给连接的执行选项。事务使用此选定的隔离级别进行。当事务完成时，隔离级别会重置为其默认值，然后将连接返回到连接池。

`Session.begin()`方法也可以用于开始`Session`级事务；在调用该方法后，可以使用`Session.connection()`来设置每个连接的事务隔离级别：

```py
sess = Session(bind=engine)

with sess.begin():
    # call connection() with options before any other operations proceed.
    # this will procure a new connection from the bound engine and begin a
    # real database transaction.
    sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    # ... work with session in SERIALIZABLE isolation level...

# outside the block, the transaction has been committed.  the connection is
# released and reverted to its previous isolation level.
```

### 使用事件跟踪事务状态

请参阅事务事件部分，了解有关会话事务状态更改的可用事件挂钩的概述。 ## 加入会话到外部事务（例如用于测试套件）

如果正在使用处于事务状态的 `Connection`（即已建立 `Transaction`），则可以通过将 `Session` 绑定到该 `Connection` 来使 `Session` 参与该事务。通常的理由是允许 ORM 代码自由地与 `Session` 一起工作，包括调用 `Session.commit()`，之后整个数据库交互都被回滚。

在 2.0 版本中更改：2.0 版本再次对“加入到外部事务”配方进行了改进；不再需要事件处理程序来“重置”嵌套事务。

该配方的工作方式是在事务内部建立一个 `Connection`，可选地建立一个 SAVEPOINT，然后将其传递给 `Session` 作为“bind”；`Session.join_transaction_mode` 参数传递了设置为 `"create_savepoint"`，表示应该创建新的 SAVEPOINT 来实现 `Session` 的 BEGIN/COMMIT/ROLLBACK，这将使外部事务处于传递时的相同状态。

当测试拆解时，外部事务会被回滚，以便将测试中的任何数据更改还原：

```py
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine
from unittest import TestCase

# global application scope.  create Session class, engine
Session = sessionmaker()

engine = create_engine("postgresql+psycopg2://...")

class SomeTest(TestCase):
    def setUp(self):
        # connect to the database
        self.connection = engine.connect()

        # begin a non-ORM transaction
        self.trans = self.connection.begin()

        # bind an individual Session to the connection, selecting
        # "create_savepoint" join_transaction_mode
        self.session = Session(
            bind=self.connection, join_transaction_mode="create_savepoint"
        )

    def test_something(self):
        # use the session in tests.

        self.session.add(Foo())
        self.session.commit()

    def test_something_with_rollbacks(self):
        self.session.add(Bar())
        self.session.flush()
        self.session.rollback()

        self.session.add(Foo())
        self.session.commit()

    def tearDown(self):
        self.session.close()

        # rollback - everything that happened with the
        # Session above (including calls to commit())
        # is rolled back.
        self.trans.rollback()

        # return connection to the Engine
        self.connection.close()
```

上述配方是 SQLAlchemy 自己的 CI 的一部分，以确保它仍然按预期工作。 ## 管理事务

在 1.4 版本中更改：会话事务管理已经进行了修改，使其更清晰、更易于使用。特别是，现在它具有“自动开始”操作，这意味着可以控制事务开始的时间点，而无需使用传统的“自动提交”模式。

`Session` 跟踪一次性的“虚拟”事务的状态，使用一个叫做 `SessionTransaction` 的对象。然后，该对象利用底层的 `Engine` 或引擎来启动使用 `Connection` 对象所需的真实连接级事务。

当需要时，这个“虚拟”事务会自动创建，或者可以使用 `Session.begin()` 方法手动开始。尽可能大程度地支持 Python 上下文管理器的使用，不仅在创建 `Session` 对象的级别上，还在维护 `SessionTransaction` 的范围上。

下面，假设我们从一个 `Session` 开始：

```py
from sqlalchemy.orm import Session

session = Session(engine)
```

现在我们可以使用上下文管理器在标记的事务中运行操作：

```py
with session.begin():
    session.add(some_object())
    session.add(some_other_object())
# commits transaction at the end, or rolls back if there
# was an exception raised
```

在上述上下文结束时，假设没有引发任何异常，任何待处理的对象都将被刷新到数据库，并且数据库事务将被提交。如果在上述块中引发了异常，则事务将被回滚。在这两种情况下，上述 `Session` 在退出块后都可以在后续事务中使用。

`Session.begin()` 方法是可选的，`Session` 也可以使用逐步提交的方法，在需要时自动开始事务；只需提交或回滚：

```py
session = Session(engine)

session.add(some_object())
session.add(some_other_object())

session.commit()  # commits

# will automatically begin again
result = session.execute(text("< some select statement >"))
session.add_all([more_objects, ...])
session.commit()  # commits

session.add(still_another_object)
session.flush()  # flush still_another_object
session.rollback()  # rolls back still_another_object
```

`Session`本身具有`Session.close()`方法。如果`Session`是在尚未提交或回滚的事务内开始的，则此方法将取消（即回滚）该事务，并且还将清除`Session`对象状态中包含的所有对象。如果`Session`的使用方式不保证调用`Session.commit()`或`Session.rollback()`（例如不在上下文管理器或类似位置），则可以使用`close`方法确保释放所有资源：

```py
# expunges all objects, releases all transactions unconditionally
# (with rollback), releases all database connections back to their
# engines
session.close()
```

最后，会话构建/关闭过程本身也可以通过上下文管理器运行。这是确保`Session`对象使用范围在固定块内的最佳方法。首先通过`Session`构造函数进行说明：

```py
with Session(engine) as session:
    session.add(some_object())
    session.add(some_other_object())

    session.commit()  # commits

    session.add(still_another_object)
    session.flush()  # flush still_another_object

    session.commit()  # commits

    result = session.execute(text("<some SELECT statement>"))

# remaining transactional state from the .execute() call is
# discarded
```

同样，`sessionmaker`也可以以相同的方式使用：

```py
Session = sessionmaker(engine)

with Session() as session:
    with session.begin():
        session.add(some_object)
    # commits

# closes the Session
```

`sessionmaker`本身包括一个`sessionmaker.begin()`方法，允许同时执行这两个操作：

```py
with Session.begin() as session:
    session.add(some_object)
```

### 使用 SAVEPOINT

如果底层引擎支持 SAVEPOINT 事务，则可以使用`Session.begin_nested()`方法来区分 SAVEPOINT 事务：

```py
Session = sessionmaker()

with Session.begin() as session:
    session.add(u1)
    session.add(u2)

    nested = session.begin_nested()  # establish a savepoint
    session.add(u3)
    nested.rollback()  # rolls back u3, keeps u1 and u2

# commits u1 and u2
```

每次调用`Session.begin_nested()`时，都会在当前数据库事务的范围内（如果尚未开始，则开始一个新的事务）向数据库发送新的“BEGIN SAVEPOINT”命令，并返回一个类型为`SessionTransaction`的对象，该对象表示对此 SAVEPOINT 的句柄。当调用该对象的`.commit()`方法时，将向数据库发出“RELEASE SAVEPOINT”命令；如果调用`.rollback()`方法，则发出“ROLLBACK TO SAVEPOINT”命令。封闭的数据库事务保持进行中。

`Session.begin_nested()`通常用作上下文管理器，其中可以捕获特定的每个实例错误，与事务状态的部分回滚一起使用，而不是回滚整个事务，如下例所示：

```py
for record in records:
    try:
        with session.begin_nested():
            session.merge(record)
    except:
        print("Skipped record %s" % record)
session.commit()
```

当由`Session.begin_nested()`生成的上下文管理器完成时，它“提交”了保存点，其中包括刷新所有待定状态的通常行为。当发生错误时，保存点被回滚，并且被更改的对象的`Session`本地状态会被过期。

这种模式非常适合诸如使用 PostgreSQL 并捕获`IntegrityError`以检测重复行的情况；当出现此类错误时，PostgreSQL 通常会中止整个事务，但是在使用 SAVEPOINT 时，外部事务会被维持。在下面的示例中，一组数据被持久化到数据库中，偶尔会跳过“重复的主键”记录，而不会回滚整个操作：

```py
from sqlalchemy import exc

with session.begin():
    for record in records:
        try:
            with session.begin_nested():
                obj = SomeRecord(id=record["identifier"], name=record["name"])
                session.add(obj)
        except exc.IntegrityError:
            print(f"Skipped record {record} - row already exists")
```

当调用`Session.begin_nested()`时，`Session`首先将当前所有待定状态刷新到数据库；这是无条件发生的，不管`Session.autoflush`参数的值如何，该参数通常用于禁用自动刷新。这种行为的理由是，当在这个嵌套事务上发生回滚时，`Session`可以使在 SAVEPOINT 范围内创建的任何内存状态过期，同时确保当这些过期对象被刷新时，SAVEPOINT 开始之前的对象图状态可重新从数据库加载。

在现代版本的 SQLAlchemy 中，当由`Session.begin_nested()`发起的 SAVEPOINT 被回滚时，自从创建 SAVEPOINT 以来被修改的内存对象状态会被过期，然而自 SAVEPOINT 开始以来未被改变的其他对象状态会被保留。这样，后续操作可以继续使用未受影响的数据，而无需从数据库中刷新。

另请参阅

`Connection.begin_nested()` - 核心 SAVEPOINT API  ### 会话级别 vs. 引擎级别的事务控制

在核心中的`连接`和 ORM 中的`_session.Session`具有等效的事务语义，都在`sessionmaker`与`引擎`的级别，以及`会话`与`连接`的级别。以下各节根据以下方案详细说明了这些情况：

```py
ORM                                           Core
-----------------------------------------     -----------------------------------
sessionmaker                                  Engine
Session                                       Connection
sessionmaker.begin()                          Engine.begin()
some_session.commit()                         some_connection.commit()
with some_sessionmaker() as session:          with some_engine.connect() as conn:
with some_sessionmaker.begin() as session:    with some_engine.begin() as conn:
with some_session.begin_nested() as sp:       with some_connection.begin_nested() as sp:
```

#### 边做边提交

`会话`和`连接`都具有`Connection.commit()`和`Connection.rollback()`方法。使用 SQLAlchemy 2.0 风格的操作，这些方法在所有情况下都会影响**最外层**的事务。对于`会话`，假定`Session.autobegin`保持默认值`True`。

`引擎`：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.connect() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    conn.commit()
```

`会话`：

```py
Session = sessionmaker(engine)

with Session() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    session.commit()
```

#### 单次开始

`sessionmaker`和`引擎`都具有`Engine.begin()`方法，该方法将获取一个用于执行 SQL 语句的新对象（分别是`会话`和`连接`），然后返回一个上下文管理器，该管理器将为该对象维护一个开始/提交/回滚的上下文。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
# commits and closes automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
# commits and closes automatically
```

#### 嵌套事务

当使用 SAVEPOINT 通过 `Session.begin_nested()` 或 `Connection.begin_nested()` 方法时，返回的事务对象必须用于提交或回滚 SAVEPOINT。调用 `Session.commit()` 或 `Connection.commit()` 方法将始终提交**最外层**事务；这是 SQLAlchemy 2.0 特定于行为的，与 1.x 系列相反。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    savepoint = conn.begin_nested()
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    savepoint.commit()  # or rollback

# commits automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    savepoint = session.begin_nested()
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    savepoint.commit()  # or rollback
# commits automatically
```  ### 显式开始

`Session` 具有“自动开始”行为，这意味着一旦操作开始进行，它就会确保存在一个用于跟踪正在进行的操作的 `SessionTransaction`。当调用 `Session.commit()` 时，该事务将被完成。

通常希望特别是在框架集成中，控制“开始”操作发生的时机。为此，`Session` 使用“自动开始”策略，使得可以直接调用 `Session.begin()` 方法来为尚未启动事务的 `Session` 开始事务：

```py
Session = sessionmaker(bind=engine)
session = Session()
session.begin()
try:
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
    session.commit()
except:
    session.rollback()
    raise
```

上述模式更惯用地使用上下文管理器调用：

```py
Session = sessionmaker(bind=engine)
session = Session()
with session.begin():
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
```

`Session.begin()` 方法和会话的“自动开始”过程使用相同的步骤序列开始事务。这包括当 `SessionEvents.after_transaction_create()` 事件发生时调用；此钩子被框架用于将其自己的事务处理过程与 ORM `Session` 集成。  ### 启用两阶段提交

对于支持两阶段操作的后端（当前为 MySQL 和 PostgreSQL），可以指示会话使用两阶段提交语义。这将协调跨数据库的事务提交，以便在所有数据库中提交或回滚事务。您还可以`Session.prepare()` 会话以与 SQLAlchemy 未管理的事务进行交互。要使用两阶段事务，请在会话上设置标志 `twophase=True`：

```py
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker(twophase=True)

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()

# .... work with accounts and users

# commit.  session will issue a flush to all DBs, and a prepare step to all DBs,
# before committing both transactions
session.commit()
```  ### 设置事务隔离级别 / DBAPI AUTOCOMMIT

大多数 DBAPI 支持可配置的事务隔离级别概念。传统上有四个级别：“READ UNCOMMITTED”、“READ COMMITTED”、“REPEATABLE READ”和“SERIALIZABLE”。这些通常在 DBAPI 连接开始新事务之前应用，需要注意的是，大多数 DBAPI 在首次发出 SQL 语句时会隐式开始此事务。

支持隔离级别的 DBAPI 通常还支持真实的“自动提交”概念，这意味着 DBAPI 连接本身将被放置在非事务自动提交模式中。这通常意味着自动向数据库发出“BEGIN”的典型 DBAPI 行为不再发生，但也可能包括其他指令。在使用此模式时，**DBAPI 在任何情况下都不使用事务**。SQLAlchemy 方法如`.begin()`、`.commit()` 和 `.rollback()` 会静默通过。

SQLAlchemy 的方言支持在每个`Engine` 或每个`Connection` 的基础上设置隔离模式，使用`create_engine()` 层次上以及 `Connection.execution_options()` 层次上的标志。

当使用 ORM `Session` 时，它充当引擎和连接的*外观*，但不直接暴露事务隔离。因此，为了影响事务隔离级别，我们需要在适当时对`Engine` 或 `Connection` 进行操作。

另请参阅

设置事务隔离级别，包括 DBAPI 自动提交 - 一定要查看 SQLAlchemy `Connection` 对象级别的隔离级别是如何工作的。

#### 为 Sessionmaker / Engine 设置隔离级别

要为 `Session` 或 `sessionmaker` 设置特定的隔离级别，全局首选技巧是可以始终根据特定的隔离级别构建一个 `Engine`，然后将其用作 `Session` 和/或 `sessionmaker` 的连接源：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    isolation_level="REPEATABLE READ",
)

Session = sessionmaker(eng)
```

另一个选项，如果同时存在具有不同隔离级别的两个引擎，是使用 `Engine.execution_options()` 方法，该方法将产生原始 `Engine` 的浅拷贝，该拷贝与父引擎共享相同的连接池。当操作将被分为“事务性”和“自动提交”操作时，通常最好使用此方法：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

transactional_session = sessionmaker(eng)
autocommit_session = sessionmaker(autocommit_engine)
```

在上述情况中，“`eng`” 和 `"autocommit_engine"` 共享相同的方言和连接池。但是，当从 `autocommit_engine` 获取连接时，将设置“AUTOCOMMIT”模式。当这两个 `sessionmaker` 对象“`transactional_session`” 和 “`autocommit_session"` 与数据库连接一起工作时，它们会继承这些特性。

“`autocommit_session`” 仍然具有事务语义，包括 `Session.commit()` 和 `Session.rollback()` 仍然将其自己视为“提交”和“回滚”对象，但是事务将会默默消失。因此，**通常情况下，虽然不是严格要求，但 AUTOCOMMIT 隔离级别的会话应该以只读方式使用**，也就是：

```py
with autocommit_session() as session:
    some_objects = session.execute(text("<statement>"))
    some_other_objects = session.execute(text("<statement>"))

# closes connection
```

#### 设置单独会话的隔离级别

当我们创建一个新的 `Session`，无论是直接使用构造函数还是调用由 `sessionmaker` 生成的可调用函数时，我们都可以直接传递 `bind` 参数，覆盖预先存在的绑定。例如，我们可以从默认的 `sessionmaker` 创建我们的 `Session`，并传递一个设置为自动提交的引擎：

```py
plain_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = plain_engine.execution_options(isolation_level="AUTOCOMMIT")

# will normally use plain_engine
Session = sessionmaker(plain_engine)

# make a specific Session that will use the "autocommit" engine
with Session(bind=autocommit_engine) as session:
    # work with session
    ...
```

对于配置了多个“绑定”（`Session`或`sessionmaker`）的情况，我们可以重新完全指定`binds`参数，或者如果我们只想替换特定的绑定，则可以使用`Session.bind_mapper()`或`Session.bind_table()`方法：

```py
with Session() as session:
    session.bind_mapper(User, autocommit_engine)
```

#### 为单个事务设置隔离级别

关于隔离级别的一个关键注意事项是，不能在已经开始事务的`Connection`上安全地修改设置。数据库不能更改正在进行的事务的隔离级别，并且一些 DBAPIs 和 SQLAlchemy 方言在这个领域的行为不一致。

因此，最好使用一个与所需隔离级别的引擎直接绑定的`Session`。然而，可以通过在事务开始时使用`Session.connection()`方法来影响每个连接的隔离级别：

```py
from sqlalchemy.orm import Session

# assume session just constructed
sess = Session(bind=engine)

# call connection() with options before any other operations proceed.
# this will procure a new connection from the bound engine and begin a real
# database transaction.
sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

# ... work with session in SERIALIZABLE isolation level...

# commit transaction.  the connection is released
# and reverted to its previous isolation level.
sess.commit()

# subsequent to commit() above, a new transaction may be begun if desired,
# which will proceed with the previous default isolation level unless
# it is set again.
```

在上面的例子中，我们首先使用构造函数或`sessionmaker`生成一个`Session`。然后，我们通过调用`Session.connection()`显式设置数据库级别事务的开始，该方法提供了将传递给连接的执行选项，在开始数据库级别事务之前进行设置。事务使用所选的隔离级别进行。事务完成后，将在将连接返回到连接池之前将连接上的隔离级别重置为其默认值。

`Session.begin()`方法也可以用于开始`Session`级别的事务；在调用该方法后，可以使用`Session.connection()`设置每个连接的事务隔离级别：

```py
sess = Session(bind=engine)

with sess.begin():
    # call connection() with options before any other operations proceed.
    # this will procure a new connection from the bound engine and begin a
    # real database transaction.
    sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    # ... work with session in SERIALIZABLE isolation level...

# outside the block, the transaction has been committed.  the connection is
# released and reverted to its previous isolation level.
```

### 使用事件跟踪事务状态

请参阅事务事件部分，了解会话事务状态更改的可用事件挂钩的概述。

### 使用保存点

如果底层引擎支持 SAVEPOINT 事务，则可以使用`Session.begin_nested()`方法进行分割：

```py
Session = sessionmaker()

with Session.begin() as session:
    session.add(u1)
    session.add(u2)

    nested = session.begin_nested()  # establish a savepoint
    session.add(u3)
    nested.rollback()  # rolls back u3, keeps u1 and u2

# commits u1 and u2
```

每次调用`Session.begin_nested()`时，都会在当前数据库事务的范围内（如果尚未开始，则开始一个）向数据库发出新的“BEGIN SAVEPOINT”命令，并返回一个`SessionTransaction`类型的对象，该对象表示对此保存点的句柄。当调用此对象的`.commit()`方法时，将向数据库发出“RELEASE SAVEPOINT”，如果调用`.rollback()`方法，则会发出“ROLLBACK TO SAVEPOINT”。外层数据库事务仍然在进行中。

`Session.begin_nested()`通常用作上下文管理器，其中可以捕获特定的每个实例错误，在此事务状态的一部分上发出回滚，而无需回滚整个事务，就像下面的示例中一样：

```py
for record in records:
    try:
        with session.begin_nested():
            session.merge(record)
    except:
        print("Skipped record %s" % record)
session.commit()
```

当由`Session.begin_nested()`生成的上下文管理器完成时，它会“提交”保存点，其中包括刷新所有待处理状态的通常行为。当出现错误时，保存点会被回滚，并且对已更改的对象的`Session`的状态将被过期。

此模式非常适合于使用 PostgreSQL 并捕获`IntegrityError`以检测重复行的情况；当引发此类错误时，PostgreSQL 通常会中止整个事务，但是当使用 SAVEPOINT 时，外部事务会得以保留。在下面的示例中，将一系列数据持久化到数据库中，偶尔会跳过“重复主键”记录，而不会回滚整个操作：

```py
from sqlalchemy import exc

with session.begin():
    for record in records:
        try:
            with session.begin_nested():
                obj = SomeRecord(id=record["identifier"], name=record["name"])
                session.add(obj)
        except exc.IntegrityError:
            print(f"Skipped record {record} - row already exists")
```

当调用`Session.begin_nested()`时，`Session`首先会将当前所有待定状态刷新到数据库；无论`Session.autoflush`参数的值是什么，这都是无条件的，通常可以用来禁用自动刷新。这种行为的原因是当此嵌套事务上发生回滚时，`Session`可以使在保存点范围内创建的任何内存状态过期，同时确保在刷新这些过期对象时，保存点开始前的对象图状态将可用于重新从数据库加载。

在现代版本的 SQLAlchemy 中，当由`Session.begin_nested()`初始化的保存点被回滚时，自从保存点创建以来被修改的内存对象状态将会被过期，但是其他自保存点开始时未改变的对象状态将会被保留。这样做是为了让后续操作可以继续使用那些未受影响的数据，而无需从数据库中刷新。

另请参阅

`Connection.begin_nested()` - 核心保存点 API

### 会话级与引擎级事务控制

Core 中的`Connection`和 ORM 中的`_session.Session`都具有等效的事务语义，无论是在`sessionmaker`与`Engine`之间，还是在`Session`与`Connection`之间。以下各节详细说明了这些情景，基于以下方案：

```py
ORM                                           Core
-----------------------------------------     -----------------------------------
sessionmaker                                  Engine
Session                                       Connection
sessionmaker.begin()                          Engine.begin()
some_session.commit()                         some_connection.commit()
with some_sessionmaker() as session:          with some_engine.connect() as conn:
with some_sessionmaker.begin() as session:    with some_engine.begin() as conn:
with some_session.begin_nested() as sp:       with some_connection.begin_nested() as sp:
```

#### 边做边提交

`Session` 和 `Connection` 均提供了 `Connection.commit()` 和 `Connection.rollback()` 方法。使用 SQLAlchemy 2.0 风格的操作时，这些方法在所有情况下都会影响**最外层**的事务。对于 `Session`，假定 `Session.autobegin` 保持默认值 `True`。

`Engine`:

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.connect() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    conn.commit()
```

`Session`:

```py
Session = sessionmaker(engine)

with Session() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    session.commit()
```

#### 只初始化一次

`sessionmaker` 和 `Engine` 均提供了 `Engine.begin()` 方法，该方法将获取一个新对象来执行 SQL 语句（分别是 `Session` 和 `Connection`），然后返回一个上下文管理器，用于维护该对象的开始/提交/回滚上下文。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
# commits and closes automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
# commits and closes automatically
```

#### 嵌套事务

当使用 SAVEPOINT 通过 `Session.begin_nested()` 或 `Connection.begin_nested()` 方法时，必须使用返回的事务对象来提交或回滚 SAVEPOINT。调用 `Session.commit()` 或 `Connection.commit()` 方法将始终提交**最外层**的事务；这是 SQLAlchemy 2.0 特有的行为，与 1.x 系列相反。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    savepoint = conn.begin_nested()
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    savepoint.commit()  # or rollback

# commits automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    savepoint = session.begin_nested()
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    savepoint.commit()  # or rollback
# commits automatically
```

#### 边提交边执行

`Session`和`Connection`均提供`Connection.commit()`和`Connection.rollback()`方法。使用 SQLAlchemy 2.0 风格的操作，这些方法在所有情况下都会影响到**最外层**的事务。对于`Session`，假定`Session.autobegin`保持其默认值为`True`。

`Engine`：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.connect() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    conn.commit()
```

`Session`：

```py
Session = sessionmaker(engine)

with Session() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    session.commit()
```

#### 开始一次

`sessionmaker`和`Engine`均提供`Engine.begin()`方法，该方法将获取一个新对象以执行 SQL 语句（分别是`Session`和`Connection`），然后返回一个上下文管理器，用于维护该对象的开始/提交/回滚上下文。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
# commits and closes automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
# commits and closes automatically
```

#### 嵌套事务

使用通过`Session.begin_nested()`或`Connection.begin_nested()`方法创建的 SAVEPOINT 时，必须使用返回的事务对象提交或回滚 SAVEPOINT。调用`Session.commit()`或`Connection.commit()`方法将始终提交**最外层**的事务；这是 SQLAlchemy 2.0 特定行为，与 1.x 系列相反。

引擎：

```py
engine = create_engine("postgresql+psycopg2://user:pass@host/dbname")

with engine.begin() as conn:
    savepoint = conn.begin_nested()
    conn.execute(
        some_table.insert(),
        [
            {"data": "some data one"},
            {"data": "some data two"},
            {"data": "some data three"},
        ],
    )
    savepoint.commit()  # or rollback

# commits automatically
```

会话：

```py
Session = sessionmaker(engine)

with Session.begin() as session:
    savepoint = session.begin_nested()
    session.add_all(
        [
            SomeClass(data="some data one"),
            SomeClass(data="some data two"),
            SomeClass(data="some data three"),
        ]
    )
    savepoint.commit()  # or rollback
# commits automatically
```

### 显式开始

`Session` 具有“自动开始”行为，这意味着一旦开始执行操作，它就会确保存在一个 `SessionTransaction` 来跟踪正在进行的操作。当调用 `Session.commit()` 时，此事务将被完成。

通常情况下，特别是在框架集成中，控制“开始”操作发生的时间点是可取的。为此，`Session`使用“自动开始”策略，以便可以直接调用 `Session.begin()` 方法来启动一个尚未开始事务的 `Session`：

```py
Session = sessionmaker(bind=engine)
session = Session()
session.begin()
try:
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
    session.commit()
except:
    session.rollback()
    raise
```

上述模式通常使用上下文管理器更具惯用性：

```py
Session = sessionmaker(bind=engine)
session = Session()
with session.begin():
    item1 = session.get(Item, 1)
    item2 = session.get(Item, 2)
    item1.foo = "bar"
    item2.bar = "foo"
```

`Session.begin()` 方法和会话的“自动开始”过程使用相同的步骤序列来开始事务。这包括当它发生时调用 `SessionEvents.after_transaction_create()` 事件；此钩子由框架使用，以便将其自己的事务处理集成到 ORM `Session` 的事务处理中。

### 启用两阶段提交

对于支持两阶段操作的后端（目前是 MySQL 和 PostgreSQL），可以指示会话使用两阶段提交语义。这将协调跨数据库的事务提交，以便在所有数据库中要么提交事务，要么回滚事务。您还可以为与 SQLAlchemy 未管理的事务交互的会话 `Session.prepare()` 。要使用两阶段事务，请在会话上设置标志 `twophase=True`：

```py
engine1 = create_engine("postgresql+psycopg2://db1")
engine2 = create_engine("postgresql+psycopg2://db2")

Session = sessionmaker(twophase=True)

# bind User operations to engine 1, Account operations to engine 2
Session.configure(binds={User: engine1, Account: engine2})

session = Session()

# .... work with accounts and users

# commit.  session will issue a flush to all DBs, and a prepare step to all DBs,
# before committing both transactions
session.commit()
```

### 设置事务隔离级别 / DBAPI 自动提交

大多数 DBAPI 支持可配置的事务隔离级别的概念。传统上，这些级别是“READ UNCOMMITTED”、“READ COMMITTED”、“REPEATABLE READ”和“SERIALIZABLE”。这些通常在 DBAPI 连接开始新事务之前应用，注意大多数 DBAPI 在首次发出 SQL 语句时会隐式地开始此事务。

支持隔离级别的 DBAPI 通常也支持真正的“自动提交”概念，这意味着 DBAPI 连接本身将被放置到非事务自动提交模式中。这通常意味着数据库自动不再发出“BEGIN”，但也可能包括其他指令。使用此模式时，**DBAPI 在任何情况下都不使用事务**。SQLAlchemy 方法如`.begin()`、`.commit()`和`.rollback()`会悄无声息地传递。

SQLAlchemy 的方言支持在每个`Engine`或每个`Connection` 上设置可设置的隔离模式，使用的标志既可以在`create_engine()`级别，也可以在`Connection.execution_options()`级别。

当使用 ORM `Session` 时，它充当引擎和连接的*门面*，但不直接暴露事务隔离。因此，为了影响事务隔离级别，我们需要在适当的时候对`Engine`或`Connection`进行操作。

也请参阅

设置事务隔离级别，包括 DBAPI 自动提交 - 一定要查看 SQLAlchemy `Connection` 对象级别上隔离级别的工作方式。

#### 设置会话/引擎范围的隔离级别

要全局设置一个具有特定隔离级别的`Session`或`sessionmaker`，第一种技术是可以在所有情况下构造一个具有特定隔离级别的`Engine`，然后将其用作`Session`和/或`sessionmaker`的连接来源：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    isolation_level="REPEATABLE READ",
)

Session = sessionmaker(eng)
```

另一个选项，如果同时存在两个具有不同隔离级别的引擎，可以使用`Engine.execution_options()`方法，该方法将生成原始`Engine`的浅拷贝，该浅拷贝与父引擎共享相同的连接池。当操作将被分为“事务”和“自动提交”操作时，通常更可取：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

transactional_session = sessionmaker(eng)
autocommit_session = sessionmaker(autocommit_engine)
```

在上面的例子中，“`eng`”和“`autocommit_engine`”共享相同的方言和连接池。然而，当从`autocommit_engine`获取连接时，将设置“AUTOCOMMIT”模式。然后，这两个`sessionmaker`对象“`transactional_session`”和“`autocommit_session`”在与数据库连接一起工作时继承这些特性。

“`autocommit_session`” **保持事务语义**，包括`Session.commit()`和`Session.rollback()`仍然认为自己是“提交”和“回滚”对象，但事务将会静默不存在。因此，**通常情况下，尽管不是严格要求，但一个具有 AUTOCOMMIT 隔离级别的 Session 应该以只读方式使用**，即：

```py
with autocommit_session() as session:
    some_objects = session.execute(text("<statement>"))
    some_other_objects = session.execute(text("<statement>"))

# closes connection
```

#### 为单个 Session 设置隔离级别

当我们创建一个新的`Session`时，可以直接传递`bind`参数，覆盖预先存在的绑定，无论是直接使用构造函数还是调用由`sessionmaker`产生的可调用对象。例如，我们可以从默认的`sessionmaker`创建我们的`Session`并传递一个设置为自动提交的引擎：

```py
plain_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = plain_engine.execution_options(isolation_level="AUTOCOMMIT")

# will normally use plain_engine
Session = sessionmaker(plain_engine)

# make a specific Session that will use the "autocommit" engine
with Session(bind=autocommit_engine) as session:
    # work with session
    ...
```

对于配置有多个`binds`的`Session`或`sessionmaker`的情况，我们可以重新指定完整的`binds`参数，或者如果我们只想替换特定的绑定，我们可以使用`Session.bind_mapper()`或`Session.bind_table()`方法：

```py
with Session() as session:
    session.bind_mapper(User, autocommit_engine)
```

#### 为单个事务设置隔离级别

关于隔离级别的一个关键注意事项是，不能在已经启动事务的 `Connection` 上安全地修改设置。数据库无法更改正在进行的事务的隔离级别，并且一些 DBAPI 和 SQLAlchemy 方言在这个领域的行为不一致。

因此最好使用一个最初绑定到具有所需隔离级别的引擎的 `Session`。但是，通过在事务开始时使用 `Session.connection()` 方法，可以影响每个连接的隔离级别：

```py
from sqlalchemy.orm import Session

# assume session just constructed
sess = Session(bind=engine)

# call connection() with options before any other operations proceed.
# this will procure a new connection from the bound engine and begin a real
# database transaction.
sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

# ... work with session in SERIALIZABLE isolation level...

# commit transaction.  the connection is released
# and reverted to its previous isolation level.
sess.commit()

# subsequent to commit() above, a new transaction may be begun if desired,
# which will proceed with the previous default isolation level unless
# it is set again.
```

在上面的示例中，我们首先使用构造函数或 `sessionmaker` 生成一个 `Session`。然后，通过调用 `Session.connection()` 明确设置数据库级事务的开始，该方法提供了将传递给连接的执行选项，在开始数据库级事务之前。事务使用此选择的隔离级别进行。当事务完成时，连接上的隔离级别将重置为默认值，然后将连接返回到连接池。

`Session.begin()` 方法也可用于开始 `Session` 级事务；在此调用之后调用 `Session.connection()` 可以用于设置每个连接的事务隔离级别：

```py
sess = Session(bind=engine)

with sess.begin():
    # call connection() with options before any other operations proceed.
    # this will procure a new connection from the bound engine and begin a
    # real database transaction.
    sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    # ... work with session in SERIALIZABLE isolation level...

# outside the block, the transaction has been committed.  the connection is
# released and reverted to its previous isolation level.
```

#### 为 Sessionmaker / Engine 设置隔离级别

要为 `Session` 或 `sessionmaker` 全局设置特定的隔离级别，第一种技术是可以在所有情况下构建一个针对特定隔离级别的 `Engine`，然后将其用作 `Session` 和/或 `sessionmaker` 的连接来源：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    isolation_level="REPEATABLE READ",
)

Session = sessionmaker(eng)
```

另一个选项，如果同时有两个具有不同隔离级别的引擎，则可以使用`Engine.execution_options()`方法，它将生成原始`Engine`的浅拷贝，与父引擎共享相同的连接池。当操作将分成“事务”和“自动提交”操作时，这通常是首选：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")

transactional_session = sessionmaker(eng)
autocommit_session = sessionmaker(autocommit_engine)
```

在上述示例中，“`eng`”和“`autocommit_engine`”都共享相同的方言和连接池。然而，当从`autocommit_engine`获取连接时，将设置“AUTOCOMMIT”模式。然后，当这两个`sessionmaker`对象“`transactional_session`”和“`autocommit_session`”与数据库连接一起工作时，它们继承了这些特征。

“`autocommit_session`”**仍然具有事务语义**，包括当它们从`autocommit_engine`获取时，`Session.commit()`和`Session.rollback()`仍然认为自己是“提交”和“回滚”对象，但事务将默默地不存在。因此，**通常，虽然不是严格要求，但具有 AUTOCOMMIT 隔离的会话应该以只读方式使用**，即：

```py
with autocommit_session() as session:
    some_objects = session.execute(text("<statement>"))
    some_other_objects = session.execute(text("<statement>"))

# closes connection
```

#### 为单个会话设置隔离

当我们创建一个新的`Session`时，可以直接传递`bind`参数，覆盖预先存在的绑定。例如，我们可以从默认的`sessionmaker`创建我们的`Session`，并传递设置为自动提交的引擎：

```py
plain_engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = plain_engine.execution_options(isolation_level="AUTOCOMMIT")

# will normally use plain_engine
Session = sessionmaker(plain_engine)

# make a specific Session that will use the "autocommit" engine
with Session(bind=autocommit_engine) as session:
    # work with session
    ...
```

对于配置了多个`binds`的`Session`或`sessionmaker`的情况，我们可以重新完全指定`binds`参数，或者如果我们只想替换特定的绑定，我们可以使用`Session.bind_mapper()`或`Session.bind_table()`方法：

```py
with Session() as session:
    session.bind_mapper(User, autocommit_engine)
```

#### 为单个事务设置隔离

关于隔离级别的一个关键警告是，在已经开始事务的 `Connection` 上无法安全地修改设置。数据库无法更改正在进行的事务的隔离级别，并且一些 DBAPI 和 SQLAlchemy 方言在这个领域的行为不一致。

因此，最好使用一个明确绑定到具有所需隔离级别的引擎的 `Session`。但是，可以通过在事务开始时使用 `Session.connection()` 方法来影响每个连接的隔离级别：

```py
from sqlalchemy.orm import Session

# assume session just constructed
sess = Session(bind=engine)

# call connection() with options before any other operations proceed.
# this will procure a new connection from the bound engine and begin a real
# database transaction.
sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

# ... work with session in SERIALIZABLE isolation level...

# commit transaction.  the connection is released
# and reverted to its previous isolation level.
sess.commit()

# subsequent to commit() above, a new transaction may be begun if desired,
# which will proceed with the previous default isolation level unless
# it is set again.
```

在上面的例子中，我们首先使用构造函数或 `sessionmaker` 来生成一个 `Session`。然后，我们通过调用 `Session.connection()` 显式设置数据库级事务的开始，该方法提供将在开始数据库级事务之前传递给连接的执行选项。事务会以此选定的隔离级别继续进行。当事务完成时，连接上的隔离级别将被重置为其默认值，然后连接将返回到连接池。

`Session.begin()` 方法也可用于开始 `Session` 级别的事务；在此调用之后调用 `Session.connection()` 可用于设置每个连接的事务隔离级别：

```py
sess = Session(bind=engine)

with sess.begin():
    # call connection() with options before any other operations proceed.
    # this will procure a new connection from the bound engine and begin a
    # real database transaction.
    sess.connection(execution_options={"isolation_level": "SERIALIZABLE"})

    # ... work with session in SERIALIZABLE isolation level...

# outside the block, the transaction has been committed.  the connection is
# released and reverted to its previous isolation level.
```

### 使用事件跟踪事务状态

请参阅 事务事件 部分，了解会话事务状态更改的可用事件挂钩的概述。

## 将会话加入到外部事务（例如用于测试套件）

如果正在使用的`Connection`已经处于事务状态（即已建立`Transaction`），则可以通过将`Session`绑定到该`Connection`来使`Session`参与该事务。通常的理由是一个测试套件允许 ORM 代码自由地与`Session`一起工作，包括能够调用`Session.commit()`，之后整个数据库交互都被回滚。

从版本 2.0 开始更改：在 2.0 中，“加入外部事务”配方再次得到了改进；不再需要事件处理程序来“重置”嵌套事务。

该配方通过在事务内部建立`Connection`（可选地建立 SAVEPOINT），然后将其传递给`Session`作为“bind”来实现；传递了`Session.join_transaction_mode`参数，设置为`"create_savepoint"`，这表示应创建新的 SAVEPOINT 以实现`Session`的 BEGIN/COMMIT/ROLLBACK，这将使外部事务保持与传递时相同的状态。

当测试拆卸时，外部事务将被回滚，以便撤消测试期间的任何数据更改：

```py
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine
from unittest import TestCase

# global application scope.  create Session class, engine
Session = sessionmaker()

engine = create_engine("postgresql+psycopg2://...")

class SomeTest(TestCase):
    def setUp(self):
        # connect to the database
        self.connection = engine.connect()

        # begin a non-ORM transaction
        self.trans = self.connection.begin()

        # bind an individual Session to the connection, selecting
        # "create_savepoint" join_transaction_mode
        self.session = Session(
            bind=self.connection, join_transaction_mode="create_savepoint"
        )

    def test_something(self):
        # use the session in tests.

        self.session.add(Foo())
        self.session.commit()

    def test_something_with_rollbacks(self):
        self.session.add(Bar())
        self.session.flush()
        self.session.rollback()

        self.session.add(Foo())
        self.session.commit()

    def tearDown(self):
        self.session.close()

        # rollback - everything that happened with the
        # Session above (including calls to commit())
        # is rolled back.
        self.trans.rollback()

        # return connection to the Engine
        self.connection.close()
```

上述配方是 SQLAlchemy 自身 CI 的一部分，以确保其按预期工作。
