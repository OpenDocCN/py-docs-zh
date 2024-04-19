# 会话基础

> 原文：[`docs.sqlalchemy.org/en/20/orm/session_basics.html`](https://docs.sqlalchemy.org/en/20/orm/session_basics.html)

## 会话的作用是什么？

从最一般的意义上讲，`Session` 建立了与数据库的所有交流，并代表了其生命周期内加载或关联的所有对象的“存储区”。它提供了一个接口，用于进行 SELECT 和其他查询，这些查询将返回并修改 ORM 映射的对象。ORM 对象本身保存在 `Session` 内部，位于一个称为 identity map 的结构中 - 这是一种维护每个对象唯一副本的数据结构，其中“唯一”意味着“只有一个具有特定主键的对象”。

在其最常见的使用模式中，`Session` 以大多数无状态形式开始。一旦发出查询或使用其他对象进行持久化，它将从与 `Session` 关联的 `Engine` 请求连接资源，然后在该连接上建立事务。该事务保持生效直到 `Session` 被指示提交或回滚事务。当事务结束时，与 `Engine` 关联的连接资源将被 释放 到引擎管理的连接池中。然后，使用新的连接检出开始新的事务。

由 `Session` 维护的 ORM 对象被 instrumented 以便每当 Python 程序中的属性或集合被修改时，都会生成一个变更事件，该事件会被 `Session` 记录下来。每当数据库即将被查询或事务即将被提交时，`Session` 首先 **flushes** 所有存储在内存中的待定更改到数据库中。这被称为 unit of work 模式。

当使用 `Session` 时，将其维护的 ORM 映射对象视为**代理对象**，它们对应于事务在 `Session` 中保持的本地数据库行。为了保持对象的状态与实际数据库中的状态相匹配，存在各种事件会导致对象重新访问数据库以保持同步。可以“分离”对象与 `Session`，并继续使用它们，尽管这种做法有其注意事项。通常情况下，当您想要再次使用它们时，您会重新将已分离的对象与另一个 `Session` 关联起来，以便它们可以恢复其表示数据库状态的正常任务。

## 使用会话的基础知识

这里介绍了最基本的 `Session` 使用模式。

### 打开和关闭会话

`Session` 可以单独构建，也可以使用 `sessionmaker` 类构建。通常，它会在开始时传递一个单独的 `Engine` 作为连接源。典型的用法可能如下所示：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# create session and add objects
with Session(engine) as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
```

在上述代码中，`Session` 是使用与特定数据库 URL 关联的 `Engine` 实例化的。然后，在 Python 上下文管理器（即 `with:` 语句）中使用它，以便在块结束时自动关闭；这相当于调用 `Session.close()` 方法。

对 `Session.commit()` 的调用是可选的，仅在我们与 `Session` 执行的工作包含要持久化到数据库的新数据时才需要。如果我们只发出 SELECT 调用并且不需要写入任何更改，则对 `Session.commit()` 的调用是不必要的。

注意

注意，在调用 `Session.commit()` 之后，无论是显式地还是在使用上下文管理器时，与 `Session` 关联的所有对象都会被过期，这意味着它们的内容将被清除以在下一个事务中重新加载。 如果这些对象被分离，除非使用 `Session.expire_on_commit` 参数来禁用此行为，否则它们将无法使用，直到与新的 `Session` 重新关联。 更多细节请参见提交部分。  ### 划定一个开始/提交/回滚块的框架

对于那些将要向数据库提交数据的情况，我们还可以将 `Session.commit()` 调用和整个事务的“框架”置于上下文管理器中。 在这里，“框架”意味着如果所有操作成功，则会调用 `Session.commit()` 方法，但如果引发任何异常，则会立即调用 `Session.rollback()` 方法，以便在将异常向外传播之前立即回滚事务。 在 Python 中，这主要是通过使用类似 `try: / except: / else:` 的代码块来表达的：

```py
# verbose version of what a context manager will do
with Session(engine) as session:
    session.begin()
    try:
        session.add(some_object)
        session.add(some_other_object)
    except:
        session.rollback()
        raise
    else:
        session.commit()
```

通过使用 `Session.begin()` 方法返回的 `SessionTransaction` 对象可以更简洁地实现上述的长形操作序列，该对象为相同操作序列提供了上下文管理器接口：

```py
# create session and add objects
with Session(engine) as session:
    with session.begin():
        session.add(some_object)
        session.add(some_other_object)
    # inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
```

更简洁地说，这两个上下文可以合并：

```py
# create session and add objects
with Session(engine) as session, session.begin():
    session.add(some_object)
    session.add(some_other_object)
# inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
```

### 使用一个 sessionmaker

`sessionmaker` 的目的是为具有固定配置的 `Session` 对象提供一个工厂。 由于典型的应用程序会在模块范围内具有一个 `Engine` 对象，因此 `sessionmaker` 可以为与此引擎相对应的 `Session` 对象提供一个工厂：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources, typically in module scope
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() without needing to pass the
# engine each time
with Session() as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
# closes the session
```

`sessionmaker` 类似于 `Engine`，是一个模块级别的工厂，用于函数级别的会话 / 连接。因此，它也有自己的 `sessionmaker.begin()` 方法，类似于 `Engine.begin()`，它返回一个 `Session` 对象，并维护一个开始 / 提交 / 回滚块：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() and include begin()/commit()/rollback()
# at once
with Session.begin() as session:
    session.add(some_object)
    session.add(some_other_object)
# commits the transaction, closes the session
```

在上述 `with:` 块结束时，`Session` 的事务将提交，并且 `Session` 将关闭。

在编写应用程序时，`sessionmaker` 工厂应与 `create_engine()` 创建的 `Engine` 对象的范围相同，通常是模块级或全局范围。由于这些对象都是工厂，因此它们可以被任意数量的函数和线程同时使用。

另请参阅

`sessionmaker`

`Session`

### 查询

查询的主要方法是利用 `select()` 构造来创建一个 `Select` 对象，然后使用诸如 `Session.execute()` 和 `Session.scalars()` 等方法执行该对象以返回结果。然后以 `Result` 对象的形式返回结果，包括 `ScalarResult` 等子变体。

SQLAlchemy ORM 查询的完整指南可在 ORM 查询指南 找到。以下是一些简要示例：

```py
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    # query for ``User`` objects
    statement = select(User).filter_by(name="ed")

    # list of ``User`` objects
    user_obj = session.scalars(statement).all()

    # query for individual columns
    statement = select(User.name, User.fullname)

    # list of Row objects
    rows = session.execute(statement).all()
```

从 2.0 版本开始更改：现在采用“2.0”样式的查询作为标准。请参阅 2.0 迁移 - ORM 用法 获取从 1.x 系列迁移的注意事项。

另请参阅

ORM 查询指南  ### 添加新项目或现有项目

`Session.add()` 用于将实例放入会话中。对于暂时的（即全新的）实例，这将在下一次刷新时对这些实例执行插入操作。对于持久的（即由此会话加载的）实例，它们已经存在，不需要添加。已分离的（即已从会话中移除的）实例可以使用此方法重新关联到会话中：

```py
user1 = User(name="user1")
user2 = User(name="user2")
session.add(user1)
session.add(user2)

session.commit()  # write changes to the database
```

要一次将一系列项目添加到会话中，请使用 `Session.add_all()`:

```py
session.add_all([item1, item2, item3])
```

`Session.add()` 操作 **级联** 到 `save-update` 级联。有关更多详细信息，请参阅 级联 部分。### 删除

`Session.delete()` 方法将实例放入会话的待删除对象列表中：

```py
# mark two objects to be deleted
session.delete(obj1)
session.delete(obj2)

# commit (or flush)
session.commit()
```

`Session.delete()` 标记对象以进行删除，这将导致针对每个受影响的主键发出 DELETE 语句。在待刷新的删除之前，被“删除”标记的对象存在于 `Session.deleted` 集合中。DELETE 后，它们从 `Session` 中删除，该会话在事务提交后变为永久。

`Session.delete()` 操作有各种与其他对象和集合的关系相关的重要行为。有关此操作的详细信息，请参阅 级联 部分，但总的规则是：

+   与通过 `relationship()` 指令与被删除对象相关的映射对象对应的行**默认情况下不会被删除**。如果这些对象有一个外键约束返回到被删除的行，这些列将被设置为 NULL。如果这些列是非空的，这将导致约束违规。

+   要将相关对象的行的“SET NULL”更改为删除，请在 `relationship()` 上使用 delete 级联。

+   当表中的行通过`relationship.secondary` 参数链接为“多对多”表时，当它们所指向的对象被删除时，这些行在所有情况下都会被删除。

+   当相关对象包含指向要删除对象的外键约束，并且它们所属的相关集合目前未加载到内存中时，工作单元将发出 SELECT 语句以获取所有相关行，以便它们的主键值可用于发出这些相关行的 UPDATE 或 DELETE 语句。这样，即使在 Core `ForeignKeyConstraint` 对象上配置了此功能，ORM 也会在没有进一步指示的情况下执行 ON DELETE CASCADE 的功能。

+   `relationship.passive_deletes` 参数可用于调整此行为并更自然地依赖于“ON DELETE CASCADE”；当设置为 True 时，此 SELECT 操作将不再发生，但是仍然存在的行仍将受到显式的 SET NULL 或 DELETE 影响。将 `relationship.passive_deletes` 设置为字符串`"all"`将禁用所有相关对象的更新/删除。

+   当标记为删除的对象发生 DELETE 时，对象不会自动从引用它的集合或对象引用中删除。当`Session`过期时，这些集合可能会被重新加载，以便对象不再存在。然而，最好的做法是，不要对这些对象使用`Session.delete()`，而是应该从其集合中删除对象，然后使用 delete-orphan 来确保它在集合删除的次要影响下被删除。有关此的示例，请参阅删除说明 - 从集合和标量关系中删除对象 部分。

另请参阅

delete - 描述了“删除级联”，当主对象被删除时，标记相关对象以删除。

delete-orphan - 描述了“删除孤立对象级联”，当它们与其主对象解除关联时，将标记相关对象以删除。

删除注释 - 从集合和标量关系中删除引用的对象 - 关于`Session.delete()`的重要背景，涉及到在内存中刷新关系。### 刷新

当`Session`使用其默认配置时，刷新步骤几乎总是透明完成的。具体来说，在因`Query`或 2.0 风格的`Session.execute()`调用而发出任何单个 SQL 语句之前，以及在`Session.commit()`调用中在事务提交之前，都会发生刷新。当使用`Session.begin_nested()`时，也会在发出 SAVEPOINT 之前发生刷新。

可以通过调用`Session.flush()`方法在任何时候强制进行`Session`刷新：

```py
session.flush()
```

在某些方法的范围内自动发生的刷新称为**自动刷新**。自动刷新被定义为一种可配置的自动刷新调用，该调用发生在包括以下方法的开头：

+   当针对 ORM 启用的 SQL 构造（例如引用 ORM 实体和/或 ORM 映射属性的`select()`对象）使用`Session.execute()`和其他执行 SQL 的方法时

+   当调用`Query`以将 SQL 发送到数据库时

+   在查询数据库之前的`Session.merge()`方法的过程中

+   当对象被刷新

+   当针对未加载对象属性进行 ORM 延迟加载操作时。

还有一些刷新发生的**无条件**点； 这些点位于关键的事务边界内，包括：

+   在`Session.commit()`方法的过程中

+   当调用`Session.begin_nested()`时

+   当使用`Session.prepare()` 2PC 方法时。

通过构造一个传递 `Session.autoflush` 参数为 `False` 的 `Session` 或 `sessionmaker`，可以禁用 **autoflush** 行为，如前述条目列表所示：

```py
Session = sessionmaker(autoflush=False)
```

此外，可以在使用 `Session` 期间通过使用 `Session.no_autoflush` 上下文管理器临时禁用自动刷新（autoflush）：

```py
with mysession.no_autoflush:
    mysession.add(some_object)
    mysession.flush()
```

**再次强调：** 当调用事务方法，如 `Session.commit()` 和 `Session.begin_nested()` 时，无论任何“autoflush”设置，当 `Session` 仍有待处理的更改时，刷新过程 **总是发生**。

由于 `Session` 仅在 DBAPI 事务上下文中调用数据库的 SQL，所有“flush”操作本身仅发生在数据库事务内部（取决于数据库事务的 隔离级别），前提是 DBAPI 不处于 驱动程序级别的自动提交 模式。这意味着假设数据库连接在其事务设置中提供了 原子性，如果 flush 内的任何单个 DML 语句失败，则整个操作将被回滚。

当 flush 中发生故障时，为了继续使用相同的 `Session`，需要在 flush 失败后显式调用 `Session.rollback()`，即使底层事务已经被回滚（即使数据库驱动程序在技术上处于驱动程序级别的自动提交模式）。这样做是为了始终保持所谓的“子事务”的整体嵌套模式。FAQ 部分 “This Session’s transaction has been rolled back due to a previous exception during flush.” (或类似内容) 中包含对此行为的更详细描述。

另请参阅

“此会话的事务已因刷新期间的先前异常而回滚。”（或类似内容） - 进一步解释为何在刷新失败时必须调用 `Session.rollback()`。 ### 通过主键获取

由于 `Session` 利用了一个身份映射，通过主键引用当前内存中的对象，因此 `Session.get()` 方法被提供为一种通过主键定位对象的方法，首先在当前身份映射中查找，然后查询数据库以获取不存在的值。例如，要定位主键标识为 `(5, )` 的 `User` 实体：

```py
my_user = session.get(User, 5)
```

`Session.get()` 也包括用于复合主键值的调用形式，可以作为元组或字典传递，以及允许特定加载器和执行选项的其他参数。有关完整参数列表，请参阅 `Session.get()`。

另请参阅

`Session.get()`  ### 过期 / 刷新

使用 `Session` 时经常遇到的一个重要考虑因素是处理从数据库加载的对象上存在的状态，以使其与当前事务的状态保持同步。SQLAlchemy ORM 基于身份映射的概念，这意味着当对象从 SQL 查询中“加载”时，将维护一个对应于特定数据库标识的唯一 Python 对象实例。这意味着如果我们发出两个单独的查询，每个查询都针对同一行，并获得一个映射对象，则两个查询将返回相同的 Python 对象：

```py
>>> u1 = session.scalars(select(User).where(User.id == 5)).one()
>>> u2 = session.scalars(select(User).where(User.id == 5)).one()
>>> u1 is u2
True
```

接下来，当 ORM 从查询中获取行时，它将**跳过已经加载的对象的属性填充**。这里的设计假设是假设一个完全隔离的事务，然后在事务不完全隔离的程度上，应用程序可以根据需要从数据库事务中刷新对象。 此 FAQ 条目 在更详细地讨论了这个概念。

当 ORM 映射对象加载到内存中时，有三种常见方法可以使用当前事务中的新数据刷新其内容：

+   **expire() 方法** - `Session.expire()` 方法将擦除对象的选定或全部属性的内容，以便在下次访问时从数据库加载它们，例如使用 延迟加载 模式：

    ```py
    session.expire(u1)
    u1.some_attribute  # <-- lazy loads from the transaction
    ```

+   **refresh() 方法** - 相关联的是 `Session.refresh()` 方法，它做的事情与 `Session.expire()` 方法相同，但还会立即发出一个或多个 SQL 查询，以实际刷新对象的内容：

    ```py
    session.refresh(u1)  # <-- emits a SQL query
    u1.some_attribute  # <-- is refreshed from the transaction
    ```

+   **populate_existing() 方法或执行选项** - 这现在是一个在 Populate Existing 中记录的执行选项；在传统形式中，它位于 `Query` 对象上，作为 `Query.populate_existing()` 方法。无论以哪种形式，此操作都表示从查询返回的对象应无条件地从数据库中重新填充：

    ```py
    u2 = session.scalars(
        select(User).where(User.id == 5).execution_options(populate_existing=True)
    ).one()
    ```

关于刷新 / 过期概念的进一步讨论，请参阅 Refreshing / Expiring。

参见

刷新 / 过期

我正在使用我的 Session 重新加载数据，但它没有看到我在其他地方提交的更改

### 使用任意 WHERE 子句的 UPDATE 和 DELETE

SQLAlchemy 2.0 包括增强的功能，用于发出几种类型的启用 ORM 的 INSERT、UPDATE 和 DELETE 语句。有关文档，请参阅 ORM-Enabled INSERT, UPDATE, and DELETE statements。

参见

启用 ORM 的 INSERT、UPDATE 和 DELETE 语句

使用自定义 WHERE 条件的 ORM UPDATE 和 DELETE

### 自动开始

`Session` 对象具有一种称为 **autobegin** 的行为。这表示，一旦使用 `Session` 执行了任何工作，无论是涉及修改 `Session` 内部状态的工作还是涉及需要数据库连接的操作，`Session` 将在内部将自身视为处于“事务”状态。

当首次构建 `Session` 时，不存在事务状态。 当调用诸如 `Session.add()` 或 `Session.execute()` 这样的方法时，或者类似地执行 `Query` 以返回结果（最终使用 `Session.execute()`），或者如果在 持久化 对象上修改属性时，将自动开始事务状态。

可以通过访问 `Session.in_transaction()` 方法来检查事务状态，该方法返回 `True` 或 `False`，指示“自动开始”步骤是否已执行。 虽然通常不需要，但 `Session.get_transaction()` 方法将返回表示此事务状态的实际 `SessionTransaction` 对象。

`Session` 的事务状态也可以通过显式调用 `Session.begin()` 方法来启动。 当调用此方法时，`Session` 无条件地置于“事务性”状态。 `Session.begin()` 可以像在 框架化一个 begin / commit / rollback 块 中描述的那样用作上下文管理器。

#### 禁用自动开始以防止隐式事务

可以使用 `Session.autobegin` 参数将“自动开始”行为设置为 `False` 以禁用该行为。通过使用此参数，`Session` 将要求显式调用 `Session.begin()` 方法。在构造后以及调用 `Session.rollback()`、`Session.commit()` 或 `Session.close()` 方法后，`Session` 不会隐式开始任何新事务，并且如果在首次调用 `Session.begin()` 之前尝试使用 `Session`，则会引发错误：

```py
with Session(engine, autobegin=False) as session:
    session.begin()  # <-- required, else InvalidRequestError raised on next call

    session.add(User(name="u1"))
    session.commit()

    session.begin()  # <-- required, else InvalidRequestError raised on next call

    u1 = session.scalar(select(User).filter_by(name="u1"))
```

新版本 2.0 中新增了 `Session.autobegin`，允许禁用“自动开始”行为。 ### 提交

`Session.commit()` 用于提交当前事务。在核心上，这表示它会对所有当前具有进行中事务的数据库连接发出 `COMMIT`；从 DBAPI 的角度来看，这意味着会在每个 DBAPI 连接上调用 `connection.commit()` DBAPI 方法。

当对 `Session` 没有进行事务操作时，表示自上次调用 `Session.commit()` 以来未对此 `Session` 执行任何操作时，该方法将开始并提交一个仅限内部的“逻辑”事务，通常不会影响数据库，除非检测到挂起的刷新更改，但仍会调用事件处理程序和对象过期规则。

`Session.commit()` 操作在发出相关数据库连接的 COMMIT 前无条件发出 `Session.flush()`。如果未检测到挂起的更改，则不会向数据库发出 SQL。此行为不可配置，并且不受 `Session.autoflush` 参数的影响。

在此之后，假设`Session`绑定到一个`Engine`，那么`Session.commit()`将会提交实际的数据库事务，如果有的话。提交之后，与该事务相关联的`Connection`对象将被关闭，导致其底层的 DBAPI 连接被释放回与`Session`绑定的`Engine`相关联的连接池中。

对于绑定到多个引擎的`Session`（例如在分区策略中描述的），对于每个正在进行“逻辑”提交的`Engine` / `Connection`，相同的提交步骤将继续进行。这些数据库事务在未启用两阶段特性的情况下是不协调的。

还有其他的连接交互模式可用，通过直接将`Session`绑定到一个`Connection`；在这种情况下，假定存在一个外部管理的事务，并且在这种情况下不会自动发出真正的 COMMIT；请参阅将会话加入外部事务（例如用于测试套件）部分了解此模式的背景。

最后，在关闭事务时，`Session`中的所有对象都将被过期。这样，当实例下次被访问时，无论是通过属性访问还是通过它们出现在 SELECT 的结果中，它们都会接收到最新的状态。此行为可以通过`Session.expire_on_commit`标志来控制，当此行为不希望时，可以将其设置为`False`。

另请参见

自动开始  ### 回滚

`Session.rollback()` 方法用于回滚当前事务（如果有的话）。当没有事务存在时，该方法会悄然通过。

使用默认配置的会话，在通过自动开始或显式调用`Session.begin()`方法开始事务后，会话的回滚后状态如下：

> +   数据库事务被回滚。对于绑定到单个`Engine`的`Session`，这意味着对当前正在使用的最多一个`Connection`进行 ROLLBACK。对于绑定到多个`Engine`对象的`Session`对象，将对所有检出的`Connection`对象发出 ROLLBACK。
> +   
> +   数据库连接被释放。这遵循与提交中注意到的相同的与连接相关的行为，其中从`Engine`对象获取的`Connection`对象被关闭，导致 DBAPI 连接被释放到`Engine`中的连接池。如果开始新的事务，新连接将从`Engine`中检出。
> +   
> +   对于直接绑定到`Connection`的`Session`（如加入外部事务的会话（例如用于测试套件）中描述的），此`Connection`上的回滚行为将遵循由`Session.join_transaction_mode`参数指定的行为，这可能涉及回滚保存点或发出真正的 ROLLBACK。
> +   
> +   在事务生命周期内将初始处于 pending 状态的对象从被添加到`Session`中的情况下，将被清除，对应于它们的 INSERT 语句被回滚。它们属性的状态保持不变。
> +   
> +   在事务生命周期内标记为 deleted 的对象将被提升回 persistent 状态，对应于它们的 DELETE 语句被回滚。请注意，如果这些对象在事务中首先是 pending，那么该操作将优先执行。
> +   
> +   所有未被清除的对象都完全过期 - 这与`Session.expire_on_commit`设置无关。

在理解了这种状态之后，`Session`可以在发生回滚后安全地继续使用。

从版本 1.4 开始更改：`Session`对象现在具有延迟“begin”行为，如 autobegin 中所述。如果没有开始事务，则`Session.commit()`和`Session.rollback()`等方法将无效。在 1.4 之前不会观察到此行为，因为在非自动提交模式下，事务总是隐式存在。

当`Session.flush()`失败时，通常是由于主键、外键或“非空”约束违反等原因，会自动发出 ROLLBACK（目前不可能在部分失败后继续刷新）。然而，此时`Session`进入一种称为“不活动”的状态，调用应用程序必须始终显式调用`Session.rollback()`方法，以便`Session`可以恢复到可用状态（也可以简单地关闭和丢弃）。有关进一步讨论，请参阅“由于刷新期间的先前异常，此会话的事务已被回滚。”（或类似）的常见问题解答。

参见

自动开始  ### 关闭

`Session.close()` 方法会调用 `Session.expunge_all()`，从会话中移除所有 ORM 映射的对象，并释放任何与其绑定的 `Engine` 对象的事务/连接资源。当连接返回到连接池时，事务状态也会回滚。

默认情况下，当 `Session` 关闭时，它实际上处于创建时的原始状态，**可以再次使用**。从这个意义上说，`Session.close()` 方法更像是一个“重置”到干净状态，而不是像一个“关闭数据库”的方法。在这种操作模式下，方法 `Session.reset()` 是 `Session.close()` 的别名，并且行为相同。

通过将参数 `Session.close_resets_only` 设置为 `False`，可以更改 `Session.close()` 的默认行为，表示在调用方法 `Session.close()` 后不能重新使用 `Session`。在这种操作模式下，当 `Session.close_resets_only` 设置为 `True` 时，方法 `Session.reset()` 将允许会话的多次“重置”，行为类似于当 `Session.close_resets_only` 设置为 `True` 时的 `Session.close()`。

版本 2.0.22 中新增。

建议通过在结尾处调用 `Session.close()` 来限制 `Session` 的范围，特别是如果没有使用 `Session.commit()` 或 `Session.rollback()` 方法。`Session` 可以作为上下文管理器使用，以确保调用 `Session.close()`：

```py
with Session(engine) as session:
    result = session.execute(select(User))

# closes session automatically
```

在版本 1.4 中做了更改：`Session` 对象具有延迟“开始”行为，如 autobegin 中所述。在调用 `Session.close()` 方法后不再立即开始新的事务。

到此为止，许多用户已经对会话有了问题。本节介绍了使用 `Session` 时所面临的最基本问题的迷你 FAQ（请注意，我们还有一个 真正的 FAQ）。

### 何时创建 `sessionmaker`？

只需一次，在应用程序的全局范围的某个地方。它应被视为应用程序配置的一部分。如果您的应用程序在一个包中有三个 .py 文件，您可以将 `sessionmaker` 行放在 `__init__.py` 文件中；从那时起，您的其他模块会说“from mypackage import Session”。这样，其他所有人只需使用 `Session()`，并且该会话的配置由该中心点控制。

如果您的应用程序启动，进行导入操作，但不知道将要连接到哪个数据库，您可以稍后在“类”级别将 `Session` 绑定到引擎，使用 `sessionmaker.configure()`。

在本节中的示例中，我们经常会在实际调用 `Session` 的行的正上方显示 `sessionmaker` 被创建。但那只是为了举例说明！实际上，`sessionmaker` 会在模块级别的某个地方。对 `Session` 进行实例化的调用将放置在应用程序中开始数据库会话的地方。

### 我什么时候构建一个 `Session`，什么时候提交它，什么时候关闭它？

一个 `Session` 通常在可能预期到数据库访问的逻辑操作开始时构建。

每当 `Session` 用于与数据库通信时，它会在开始通信时立即启动数据库事务。此事务将持续进行，直到 `Session` 被回滚、提交或关闭。如果再次使用 `Session`，则会开始新的事务，接着之前的事务结束；由此可知，`Session` 可以跨多个事务具有生命周期，但一次只能进行一个。我们将这两个概念称为 **事务范围** 和 **会话范围**。

通常很容易确定何时开始和结束 `Session` 的范围，尽管可能存在多种应用程序架构，可能会引入具有挑战性的情况。

一些示例场景包括：

+   Web 应用程序。在这种情况下，最好利用所使用的 Web 框架提供的 SQLAlchemy 集成。或者，基本模式是在 Web 请求开始时创建一个`Session`，在执行 POST、PUT 或 DELETE 的 Web 请求结束时调用 `Session.commit()` 方法，然后在 Web 请求结束时关闭会话。通常还应该设置 `Session.expire_on_commit` 为 False，以便来自视图层的对象在事务已经提交后不需要发出新的 SQL 查询来刷新对象。

+   一个后台守护进程会生成子进程，可能会想要为每个子进程创建一个`Session`，在该子进程处理的“作业”生命周期内使用该`Session`，然后在作业完成时将其销毁。

+   对于命令行脚本，应用程序会创建一个在程序开始工作时建立的单一全局 `Session`，并在程序完成任务时立即提交它。

+   对于基于 GUI 接口的应用程序，`Session` 的范围可能最好在用户生成的事件范围内，比如按钮点击。或者，范围可能与明确的用户交互相对应，比如用户“打开”一系列记录，然后“保存”它们。

一般来说，应用程序应该在处理特定数据的函数**外部**管理会话的生命周期。这是一种基本的关注点分离，使得特定于数据的操作不受其访问和操作数据的上下文的影响。

例如，**不要这样做**：

```py
### this is the **wrong way to do it** ###

class ThingOne:
    def go(self):
        session = Session()
        try:
            session.execute(update(FooBar).values(x=5))
            session.commit()
        except:
            session.rollback()
            raise

class ThingTwo:
    def go(self):
        session = Session()
        try:
            session.execute(update(Widget).values(q=18))
            session.commit()
        except:
            session.rollback()
            raise

def run_my_program():
    ThingOne().go()
    ThingTwo().go()
```

将会话（通常也是事务）的生命周期**分离和外部化**。下面的示例说明了这样做的方式，并且另外使用了 Python 上下文管理器（即 `with:` 关键字）来自动管理 `Session` 及其事务的范围：

```py
### this is a **better** (but not the only) way to do it ###

class ThingOne:
    def go(self, session):
        session.execute(update(FooBar).values(x=5))

class ThingTwo:
    def go(self, session):
        session.execute(update(Widget).values(q=18))

def run_my_program():
    with Session() as session:
        with session.begin():
            ThingOne().go(session)
            ThingTwo().go(session)
```

自 1.4 版更改：`Session` 可以作为上下文管理器使用，无需使用外部辅助函数。

### 会话是缓存吗？

嗯……不是。它有点像缓存，因为它实现了 identity map 模式，并将对象存储为主键键控制的对象。但是，它不执行任何类型的查询缓存。这意味着，如果你说`session.scalars(select(Foo).filter_by(name='bar'))`，即使`Foo(name='bar')`就在那里，在 identity map 中，会话也不知道。它必须向数据库发出 SQL，获取行，然后当它看到行中的主键时，*然后*它才能查看本地 identity map，并查看对象是否已存在。只有当你说`query.get({some primary key})`时，`Session`才不需要发出查询。

另外，默认情况下，会话(Session)使用弱引用来存储对象实例。这也违背了使用会话作为缓存的初衷。

`Session`不是设计为所有人都可以向其查询作为“注册表”的全局对象。这更像是**第二级缓存**的工作。SQLAlchemy 提供了使用[dogpile.cache](https://dogpilecache.readthedocs.io/)实现第二级缓存的模式，通过 Dogpile Caching 示例。

### 如何获取某个对象的`Session`？

使用`Session.object_session()`类方法，该方法可用于`Session`：

```py
session = Session.object_session(someobject)
```

较新的运行时检查 API 系统也可以使用：

```py
from sqlalchemy import inspect

session = inspect(someobject).session
```

### 会话(Session)是线程安全的吗？AsyncSession 在并发任务中安全吗？

`Session`是一个**可变的、有状态的**对象，表示一个**单一的数据库事务**。因此，`Session`的实例**不能在并发线程或 asyncio 任务之间共享，除非进行仔细的同步**。`Session`旨在以**非并发**方式使用，即，特定的`Session`实例应在同一时间只在一个线程或任务中使用。

当使用 SQLAlchemy 的 asyncio 扩展中的`AsyncSession`对象时，该对象只是`Session`的一个薄代理，同样的规则适用；它是一个**未同步的、可变的、有状态的对象**，因此**不安全**在多个 asyncio 任务中同时使用单个`AsyncSession`实例。

`Session`或`AsyncSession`的一个实例代表一个单一的逻辑数据库事务，每次只引用一个特定`Engine`或`AsyncEngine`的单个`Connection`（请注意，这些对象都支持同时绑定到多个引擎，但在这种情况下，在事务范围内仍然只有一个连接）。

事务中的数据库连接也是一个有状态的对象，应该以非并发、顺序的方式进行操作。命令按照序列在连接上发出，并由数据库服务器按照发出的确切顺序处理。当`Session`发出命令并接收结果时，`Session`本身正在经历与此连接上的命令和数据状态相一致的内部状态更改；这些状态包括事务是否已启动、提交或回滚，正在使用的 SAVEPOINT（如果有），以及将数据库行的状态与本地 ORM 映射的对象同步的细粒度同步。

在为并发设计数据库应用程序时，适当的模型是每个并发任务/线程都使用自己的数据库事务。这就是为什么在讨论数据库并发问题时，使用的标准术语是**多个并发事务**。在传统的关系型数据库管理系统中，没有类似于同时接收和处理多个命令的单个数据库事务的模拟。

因此，SQLAlchemy 的`Session`和`AsyncSession`的并发模型是**每个线程一个 Session，每个任务一个 AsyncSession**。一个使用多个线程或多个 asyncio 任务的应用（例如使用像`asyncio.gather()`这样的 API）将希望确保每个线程有其自己的`Session`，每个 asyncio 任务有其自己的`AsyncSession`。

确保此使用的最佳方法是在线程或任务内部的顶级 Python 函数中本地使用标准上下文管理器模式，这将确保`Session`或`AsyncSession`的生命周期在局部范围内维护。

对于那些受益于具有“全局”`Session`的应用，其中无法将`Session`对象传递给需要它的特定函数和方法的情况，`scoped_session`方法可以提供一个“线程本地”的`Session`对象；请参阅上下文/线程本地会话一节了解背景。在 asyncio 环境中，`async_scoped_session`对象是`scoped_session`的 asyncio 模拟，但是更难配置，因为它需要一个自定义的“上下文”函数。

## Session 做什么？

在最一般的意义上，`Session`建立了与数据库的所有交流，并代表了在其生命周期内加载或关联的所有对象的“持有区”。它提供了 SELECT 和其他查询的接口，这些查询将返回和修改 ORM 映射的对象。ORM 对象本身被维护在`Session`内部，存储在一个称为 identity map 的结构中 - 这是一种维护每个对象唯一副本的数据结构，其中“唯一”意味着“只有一个具有特定主键的对象”。

`Session` 在最常见的使用模式下，以大部分无状态的形式开始。一旦发出查询或使用其他对象进行持久化，它会从与 `Session` 关联的 `Engine` 请求连接资源，然后在该连接上建立事务。这个事务会一直保持到 `Session` 被指示提交或回滚事务。事务结束时，与 `Engine` 关联的连接资源会被释放到引擎管理的连接池中。然后，一个新的事务会开始，使用一个新的连接。

由 `Session` 维护的 ORM 对象被装饰，这样每当 Python 程序中的属性或集合被修改时，就会生成一个变更事件，由 `Session` 记录下来。每当即将查询数据库或即将提交事务时，`Session` 首先会将内存中的所有待定更改**刷新**到数据库中。这被称为工作单元模式。

当使用 `Session` 时，将其维护的 ORM 映射对象视为**代理对象**以访问数据库行，这些对象局限于由 `Session` 持有的事务。为了保持对象的状态与实际数据库中的内容一致，有多种事件会导致对象重新访问数据库以保持同步。可以将对象“分离”（detach）出 `Session`，并继续使用它们，尽管这种做法有其注意事项。通常情况下，你应该重新将分离的对象与另一个 `Session` 关联，以便在需要时恢复其正常的数据库状态表示任务。

## 使用会话的基础知识

这里介绍了最基本的 `Session` 使用模式。

### 开启和关闭会话

`Session`可以独立构造，也可以使用`sessionmaker`类构造。通常，它被作为连通性的源传递给单个`Engine`。典型的用法可能如下所示：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# create session and add objects
with Session(engine) as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
```

在上面的示例中，`Session`与特定数据库 URL 关联的`Engine`一起实例化。然后，它在 Python 上下文管理器（即`with:`语句）中使用，因此它在块结束时会自动关闭；这相当于调用`Session.close()`方法。

对`Session.commit()`的调用是可选的，只有当我们与`Session`一起完成的工作包括要持久化到数据库的新数据时才需要。如果我们只发出 SELECT 调用并且不需要写入任何更改，则对`Session.commit()`的调用将是不必要的。

注意

注意，在调用`Session.commit()`之后，无论是明确调用还是使用上下文管理器，与`Session`关联的所有对象都将被过期，这意味着它们的内容将被擦除以在下一个事务中重新加载。如果这些对象是分离的，则在重新关联到新的`Session`之前，它们将无法正常工作，除非使用`Session.expire_on_commit`参数来禁用此行为。更多细节请参阅 Committing 部分。

我们还可以将 `Session.commit()` 调用和事务的整体“框架”封装在一个上下文管理器中，对于那些需要将数据提交到数据库的情况。所谓的“框架”是指如果所有操作都成功，则会调用 `Session.commit()` 方法，但如果引发任何异常，则会立即调用 `Session.rollback()` 方法，以便立即回滚事务，然后向外传播异常。在 Python 中，这主要是使用 `try: / except: / else:` 块来表达的，例如：

```py
# verbose version of what a context manager will do
with Session(engine) as session:
    session.begin()
    try:
        session.add(some_object)
        session.add(some_other_object)
    except:
        session.rollback()
        raise
    else:
        session.commit()
```

通过使用 `Session.begin()` 方法返回的 `SessionTransaction` 对象，可以更简洁地实现上面示例中的长序列操作，该对象为相同序列操作提供了一个上下文管理器接口：

```py
# create session and add objects
with Session(engine) as session:
    with session.begin():
        session.add(some_object)
        session.add(some_other_object)
    # inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
```

更简洁地说，这两个上下文可以合并：

```py
# create session and add objects
with Session(engine) as session, session.begin():
    session.add(some_object)
    session.add(some_other_object)
# inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
```

### 使用 sessionmaker

`sessionmaker` 的目的是为具有固定配置的 `Session` 对象提供一个工厂。由于典型的情况是应用程序将在模块范围内拥有一个 `Engine` 对象，`sessionmaker` 可以为针对此引擎的 `Session` 对象提供一个工厂：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources, typically in module scope
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() without needing to pass the
# engine each time
with Session() as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
# closes the session
```

`sessionmaker` 类似于 `Engine`，作为一个模块级别的工厂，用于创建函数级别的会话 / 连接。因此，它也有自己的 `sessionmaker.begin()` 方法，类似于 `Engine.begin()`，它返回一个 `Session` 对象，并且也维护一个 begin/commit/rollback 块：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() and include begin()/commit()/rollback()
# at once
with Session.begin() as session:
    session.add(some_object)
    session.add(some_other_object)
# commits the transaction, closes the session
```

在上面的情况下，当上述 `with:` 块结束时，`Session` 的事务将被提交，并且 `Session` 将被关闭。

当您编写应用程序时，`sessionmaker`工厂应该与由`create_engine()`创建的`Engine`对象作用域相同，通常是在模块级或全局级。由于这些对象都是工厂，因此它们可以被任意数量的函数和线程同时使用。

另请参阅

`sessionmaker`

`Session`

### 查询

查询的主要手段是利用`select()`构造来创建一个`Select`对象，然后使用诸如`Session.execute()`和`Session.scalars()`等方法执行该对象以返回结果。结果以`Result`对象的形式返回，包括诸如`ScalarResult`等子变体。

SQLAlchemy ORM 查询的完整指南可在 ORM 查询指南中找到。以下是一些简要示例：

```py
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    # query for ``User`` objects
    statement = select(User).filter_by(name="ed")

    # list of ``User`` objects
    user_obj = session.scalars(statement).all()

    # query for individual columns
    statement = select(User.name, User.fullname)

    # list of Row objects
    rows = session.execute(statement).all()
```

从 2.0 版本开始更改：“2.0”风格查询现在是标准的。请参阅 2.0 迁移 - ORM 使用了解从 1.x 系列迁移的注意事项。

另请参阅

ORM 查询指南  ### 添加新项目或现有项目

使用`Session.add()`将实例放入会话中。对于瞬态(即全新的)实例，这将导致在下一次刷新时对这些实例进行插入操作。对于持久(即由此会话加载的)实例，它们已经存在，不需要添加。可以使用此方法将分离(即已从会话中删除的)实例重新关联到会话中：

```py
user1 = User(name="user1")
user2 = User(name="user2")
session.add(user1)
session.add(user2)

session.commit()  # write changes to the database
```

要一次向会话中添加一系列项目，请使用`Session.add_all()`：

```py
session.add_all([item1, item2, item3])
```

`Session.add()` 操作**沿着** `save-update` 级联进行。有关详细信息，请参阅 Cascades 部分。### 删除

`Session.delete()` 方法将一个实例放入会话的待删除对象列表中：

```py
# mark two objects to be deleted
session.delete(obj1)
session.delete(obj2)

# commit (or flush)
session.commit()
```

`Session.delete()`标记一个对象为删除状态，这将导致为每个受影响的主键发出一个 DELETE 语句。在挂起的删除被刷新之前，由“delete”标记的对象存在于 `Session.deleted` 集合中。删除之后，它们将从 `Session` 中删除，在事务提交后，这变得永久。

有关`Session.delete()` 操作的各种重要行为，特别是关于如何处理与其他对象和集合的关系的行为。有关此工作原理的更多信息，请参阅 Cascades 部分，但总的来说规则是：

+   通过`relationship()`指令关联到已删除对象的映射对象行默认**不会被删除**。如果这些对象具有指回被删除行的外键约束，这些列将设置为 NULL。如果列是非空的，这将导致约束违规。

+   要将相关对象行的“SET NULL”更改为删除，请在 `relationship()` 上使用 delete 级联。

+   当链接为“many-to-many”表的表中的行通过`relationship.secondary`参数链接时，当它们所引用的对象被删除时，**它们**在所有情况下都将被删除。

+   当相关对象包含指回正在删除的对象的外键约束，并且它们所属的相关集合当前未加载到内存中时，工作单元将发出一个 SELECT 来获取所有相关行，以便它们的主键值可以用于发出 UPDATE 或 DELETE 语句以处理这些相关行。通过这种方式，ORM 即使在 Core `ForeignKeyConstraint` 对象上配置了 ON DELETE CASCADE，也会执行这个功能，而不需要进一步的指示。

+   `relationship.passive_deletes`参数可用于调整此行为，并更自然地依赖于“ON DELETE CASCADE”；当设置为 True 时，此 SELECT 操作将不再发生，但仍然会对本地存在的行进行显式的 SET NULL 或 DELETE。将`relationship.passive_deletes`设置为字符串`"all"`将禁用**所有**相关对象的更新/删除。

+   当标记为删除的对象发生删除时，该对象不会自动从引用它的集合或对象引用中移除。当`Session`过期时，这些集合可能会再次加载，以便对象不再存在。然而，最好的做法是，不要对这些对象使用`Session.delete()`，而是应该从其集合中移除对象，然后使用 delete-orphan 以便作为该集合移除的次要效果而被删除。请参阅 Notes on Delete - Deleting Objects Referenced from Collections and Scalar Relationships 部分以获取示例。

另请参见

delete - 描述了“删除级联”，当主对象被删除时，会标记相关对象以进行删除。

delete-orphan - 描述了“删除孤儿级联”，当相关对象与其主对象解除关联时，会标记这些相关对象以进行删除。

Notes on Delete - Deleting Objects Referenced from Collections and Scalar Relationships - 关于`Session.delete()`的重要背景，涉及在内存中刷新关系。### 刷新

当 `Session` 使用其默认配置时，刷新步骤几乎总是透明进行的。具体来说，在由 `Query` 或 2.0 风格 的 `Session.execute()` 调用导致发出任何单个 SQL 语句之前，以及在 `Session.commit()` 调用中在提交事务之前，都会发生刷新。当使用 `Session.begin_nested()` 时，还会在发出 SAVEPOINT 之前发生刷新。

可以随时通过调用 `Session.flush()` 方法来强制执行 `Session` 的刷新：

```py
session.flush()
```

在某些方法的范围内自动发生的刷新称为**自动刷新**。自动刷新定义为在包括以下方法的开头发生的可配置的自动刷新调用：

+   `Session.execute()` 和其他执行 SQL 的方法，在针对启用了 ORM 的 SQL 构造时使用，比如指向 ORM 实体和/或 ORM 映射属性的 `select()` 对象

+   当调用 `Query` 发送 SQL 到数据库时

+   在查询数据库之前 `Session.merge()` 方法中

+   当对象被刷新

+   当针对未加载的对象属性执行 ORM 延迟加载 操作时。

还有一些无条件发生刷新的点；这些点位于关键事务边界内，包括：

+   在 `Session.commit()` 方法的过程中

+   调用 `Session.begin_nested()` 时

+   当使用 `Session.prepare()` 2PC 方法时。

对于上述项目列表所应用的**自动刷新**行为，可以通过构建一个传递了`Session.autoflush`参数为`False`的`Session`或`sessionmaker`来禁用它：

```py
Session = sessionmaker(autoflush=False)
```

此外，可以在使用`Session`时使用`Session.no_autoflush`上下文管理器临时禁用自动刷新：

```py
with mysession.no_autoflush:
    mysession.add(some_object)
    mysession.flush()
```

**重申一下：** 当调用事务方法（如`Session.commit()`和`Session.begin_nested()`）时，刷新过程**总是发生**，无论任何“自动刷新”设置如何，当`Session`仍有待处理的更改时。

由于`Session`只在 DBAPI 事务的上下文中调用 SQL 到数据库，所有“flush”操作本身只发生在数据库事务内（受数据库事务的隔离级别的影响），前提是 DBAPI 不处于驱动级别自动提交模式。这意味着假设数据库连接在其事务设置中提供了原子性，如果刷新内部的任何个别 DML 语句失败，整个操作将被回滚。

当刷新过程中发生故障时，为了继续使用相同的`Session`，在刷新失败后需要显式调用`Session.rollback()`，即使底层事务已经回滚了（即使数据库驱动程序在技术上处于驱动程序级别的自动提交模式）。这样做是为了始终保持所谓“子事务”的整体嵌套模式。 FAQ 部分“由于刷新期间的先前异常，此会话的事务已回滚。”（或类似）中包含了对此行为的更详细描述。

另请参阅

“由于刷新期间发生的先前异常，此会话的事务已回滚。”（或类似） - 关于在刷新失败时必须调用`Session.rollback()`的更多背景信息。 ### 按主键获取

由于`Session`利用了一个标识映射，该映射通过主键引用当前内存中的对象，因此`Session.get()`方法被提供用于通过主键定位对象，首先在当前标识映射内查找，然后在数据库中查询不存在的值。例如，要定位主键标识为`(5, )`的`User`实体：

```py
my_user = session.get(User, 5)
```

`Session.get()` 还包括对复合主键值的调用形式，可以作为元组或字典传递，以及允许特定加载程序和执行选项的其他参数。请参阅`Session.get()`获取完整的参数列表。

另请参阅

`Session.get()`  ### 过期/刷新

在使用`Session`时经常会出现的一个重要考虑因素是处理从数据库加载的对象上存在的状态，以保持它们与事务的当前状态同步。SQLAlchemy ORM 基于标识映射的概念，因此当从 SQL 查询中“加载”对象时，将维护一个对应于特定数据库标识的唯一 Python 对象实例。这意味着如果我们发出两个单独的查询，每个查询都针对同一行，并返回一个映射对象，则这两个查询将返回相同的 Python 对象：

```py
>>> u1 = session.scalars(select(User).where(User.id == 5)).one()
>>> u2 = session.scalars(select(User).where(User.id == 5)).one()
>>> u1 is u2
True
```

由此可见，当 ORM 从查询中返回行时，将**跳过已加载的对象的属性填充**。这里的设计假设是假定一个完全隔离的事务，然后在事务不完全隔离的程度上，应用程序可以根据需要从数据库事务中刷新对象。在我正在重新加载我的 Session 中的数据，但它没有看到我在其他地方提交的更改的 FAQ 条目中更详细地讨论了这个概念。

当 ORM 映射对象加载到内存中时，有三种常规方法可以使用当前事务中的新数据刷新其内容：

+   **expire() 方法** - `Session.expire()` 方法将擦除对象的选定或全部属性的内容，以便在下次访问它们时从数据库加载，例如使用惰性加载模式：

    ```py
    session.expire(u1)
    u1.some_attribute  # <-- lazy loads from the transaction
    ```

+   **refresh() 方法** - 与之密切相关的是`Session.refresh()` 方法，它执行`Session.expire()` 方法执行的所有操作，但还立即发出一个或多个 SQL 查询来实际刷新对象的内容：

    ```py
    session.refresh(u1)  # <-- emits a SQL query
    u1.some_attribute  # <-- is refreshed from the transaction
    ```

+   **populate_existing() 方法或执行选项** - 现在这是一个在填充现有中记录的执行选项；在传统形式中，它在`Query`对象上作为`Query.populate_existing()`方法找到。无论采取哪种形式，此操作表示应从数据库中的内容无条件地重新填充从查询返回的对象：

    ```py
    u2 = session.scalars(
        select(User).where(User.id == 5).execution_options(populate_existing=True)
    ).one()
    ```

关于刷新/过期概念的进一步讨论可在刷新/过期找到。

另请参阅

刷新/过期

我正在使用我的会话重新加载数据，但它没有看到我在其他地方提交的更改

### 使用任意 WHERE 子句的 UPDATE 和 DELETE

SQLAlchemy 2.0 包括增强功能，可发出几种类型的 ORM 启用的 INSERT、UPDATE 和 DELETE 语句。有关文档，请参阅 ORM-启用的 INSERT、UPDATE 和 DELETE 语句。

另请参阅

ORM-启用的 INSERT、UPDATE 和 DELETE 语句

带有自定义 WHERE 条件的 ORM UPDATE 和 DELETE

### 自动开始

`Session` 对象具有称为**autobegin**的行为。这表示`Session`一旦执行了与`Session`相关的任何工作，无论涉及对`Session`的内部状态进行修改还是需要数据库连接的操作，它都会在内部将自身视为处于“事务”状态。

当首次构造`Session`时，不存在事务状态。当调用方法如`Session.add()`或`Session.execute()`时，或类似地执行用于返回结果的`Query`（最终使用`Session.execute()`），或者在持久化对象上修改属性时，事务状态将自动开始。

检查事务状态可以通过访问`Session.in_transaction()`方法来实现，该方法返回`True`或`False`，指示“自动开始”步骤是否已经执行。虽然通常不需要，但`Session.get_transaction()`方法将返回表示此事务状态的实际`SessionTransaction`对象。

也可以通过调用`Session.begin()`方法显式启动`Session`的事务状态。调用此方法时，`Session`无条件地被置于“事务性”状态。`Session.begin()`可以像描述的那样用作上下文管理器，详见构建开始 / 提交 / 回滚块的框架。

#### 禁用 Autobegin 以防止隐式事务

可以通过将`Session.autobegin`参数设置为`False`来禁用“自动开始”行为。通过使用此参数，`Session`将要求显式调用`Session.begin()`方法。在构造之后以及在调用任何`Session.rollback()`、`Session.commit()`或`Session.close()`方法之后，如果尝试在没有首先调用`Session.begin()`的情况下使用`Session`，则不会隐式启动任何新事务，并且将引发错误：

```py
with Session(engine, autobegin=False) as session:
    session.begin()  # <-- required, else InvalidRequestError raised on next call

    session.add(User(name="u1"))
    session.commit()

    session.begin()  # <-- required, else InvalidRequestError raised on next call

    u1 = session.scalar(select(User).filter_by(name="u1"))
```

新版本 2.0 中新增：添加`Session.autobegin`，允许禁用“自动开始”行为  ### 提交

`Session.commit()` 用于提交当前事务。本质上，这表示在所有当前具有正在进行的事务的数据库连接上发出`COMMIT`；从 DBAPI 的角度来看，这意味着在每个 DBAPI 连接上调用`connection.commit()` DBAPI 方法。

当`Session`没有处于事务中时，表示自从上次调用`Session.commit()`以来，在此`Session`上未调用任何操作，该方法将启动并提交一个仅“逻辑”的内部事务，通常不会影响数据库，除非检测到未决刷新更改，但仍将调用事件处理程序和对象过期规则。

在发出相关数据库连接上的 COMMIT 之前，`Session.commit()` 操作无条件地发出`Session.flush()`。如果未检测到待处理的更改，则不会向数据库发出任何 SQL。此行为不可配置，并且不受`Session.autoflush`参数的影响。

在此之后，假设`Session`绑定到一个`Engine`，`Session.commit()`将提交当前的数据库事务，如果已经启动。提交后，与该事务关联的`Connection`对象将关闭，导致其底层的 DBAPI 连接被释放回与`Session`绑定的`Engine`相关联的连接池。

对于绑定到多个引擎的`Session`（例如在分区策略中描述的方式），对正在提交的“逻辑”事务中涉及的每个`Engine` / `Connection`都将执行相同的 COMMIT 步骤。除非启用了两阶段功能，否则这些数据库事务之间不协调。

其他连接-交互模式也是可用的，通过将`Session`直接绑定到`Connection`；在这种情况下，假定存在外部管理的事务，并且在这种情况下不会自动发出真正的 COMMIT；有关此模式的背景，请参阅将 Session 加入外部事务（例如测试套件）部分。

最后，在事务关闭时，`Session`中的所有对象都会被过期。这样，当下次访问实例时，无论是通过属性访问还是通过它们出现在 SELECT 的结果中，它们都会接收到最新状态。这种行为可以由`Session.expire_on_commit`标志来控制，当此行为不可取时可以将其设置为`False`。

另请参阅

自动开始  ### 回滚

`Session.rollback()`回滚当前事务（如果有）。当没有事务时，该方法会静默地通过。

默认配置的会话后回滚状态，即通过 autobegin 或显式调用`Session.begin()`方法开始事务后的状态如下：

> +   数据库事务被回滚。对于绑定到单个`Engine`的`Session`，这意味着对当前正在使用的最多一个`Connection`进行回滚。对于绑定到多个`Engine`对象的`Session`对象，将对所有被检出的`Connection`对象进行回滚。
> +   
> +   数据库连接被释放。这遵循了提交中注意到的与连接相关的相同行为，即从`Engine`对象获取的`Connection`对象被关闭，导致 DBAPI 连接被释放到 `Engine` 中的连接池中。如果有新的事务开始，会从`Engine`中检出新的连接。
> +   
> +   对于直接绑定到`Connection`的`Session`，如将会话加入外部事务（例如测试套件）中描述的，此`Connection`上的回滚行为将遵循由`Session.join_transaction_mode`参数指定的行为，这可能涉及回滚保存点或发出真正的 ROLLBACK。
> +   
> +   在事务的生命周期内，当对象被添加到`Session`时最初处于挂起状态的对象将被清除，对应其 INSERT 语句被回滚的情况。它们属性的状态保持不变。
> +   
> +   在事务的生命周期内被标记为已删除的对象将被提升回持久状态，对应其 DELETE 语句被回滚的情况。请注意，如果这些对象在事务中首先处于挂起状态，则该操作优先级较高。
> +   
> +   所有未清除的对象都将完全过期 - 这与`Session.expire_on_commit`设置无关。

在了解了这种状态后，`Session`在回滚发生后可以安全地继续使用。

从版本 1.4 开始更改：`Session`对象现在具有延迟“begin”行为，如自动开始中所述。如果未开始任何事务，则`Session.commit()`和`Session.rollback()`等方法将不起作用。在 1.4 之前，不会观察到这种行为，因为在非自动提交模式下，事务总是隐式存在的。

当`Session.flush()`失败时，通常是由于主键、外键或“非空”约束违规等原因，会自动发出 ROLLBACK（当前不可能在部分失败后继续 flush）。但是，在此时，`Session`进入一种称为“不活跃”的状态，调用应用程序必须始终显式调用`Session.rollback()`方法，以便`Session`可以回到可用状态（也可以简单地关闭并丢弃）。有关进一步讨论，请参阅 FAQ 条目“此会话的事务由于刷新时的先前异常而被回滚。”（或类似）。

另请参阅

自动开始  ### 结束

`Session.close()` 方法会发出一个`Session.expunge_all()`，它将从会话中删除所有 ORM 映射的对象，并且释放与其绑定的`Engine`对象的所有事务/连接资源。当连接返回到连接池时，事务状态也会被回滚。

默认情况下，当`Session`关闭时，它基本上处于最初构建时的原始状态，并且**可以再次使用**。从这个意义上说，`Session.close()` 方法更像是“重置”回到清洁状态，而不太像是“关闭数据库”的方法。在这种操作模式下，方法`Session.reset()`是`Session.close()`的别名，并且行为相同。

`Session.close()` 的默认行为可以通过将参数`Session.close_resets_only`设置为`False`来更改，这表明在调用方法`Session.close()`后，`Session`不能被重用。在这种操作模式下，当`Session.close_resets_only`设置为`True`时，方法`Session.reset()`将允许会话的多次“重置”，表现得像`Session.close()`一样。

2.0.22 版本中的新增功能。

建议在结束时通过调用 `Session.close()` 来限制 `Session` 的范围，特别是如果未使用 `Session.commit()` 或 `Session.rollback()` 方法。 `Session` 可以用作上下文管理器，以确保调用 `Session.close()`：

```py
with Session(engine) as session:
    result = session.execute(select(User))

# closes session automatically
```

自 1.4 版更改：`Session` 对象具有延迟“begin”行为，如 autobegin 中所述。在调用 `Session.close()` 方法后，不再立即开始新事务。### 开启和关闭会话

`Session` 可以自行构建，也可以使用 `sessionmaker` 类。通常，它最初会作为连接性的源传递单个 `Engine`。典型的用法可能如下：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# create session and add objects
with Session(engine) as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
```

上面，`Session` 是通过与特定数据库 URL 关联的 `Engine` 实例化的。然后在 Python 上下文管理器中使用（即 `with:` 语句），以便在块结束时自动关闭；这相当于调用 `Session.close()` 方法。

调用 `Session.commit()` 是可选的，只有在我们与 `Session` 一起完成的工作包括要持久化到数据库的新数据时才需要。如果我们只是发出 SELECT 调用并且不需要写入任何更改，则调用 `Session.commit()` 将是不必要的。

注

注意，在调用`Session.commit()`之后，无论是显式调用还是使用上下文管理器，与`Session`相关联的所有对象都将被过期，这意味着它们的内容将被擦除以在下一个事务中重新加载。如果这些对象被分离，它们将无法正常工作，直到与新的`Session`重新关联，除非使用`Session.expire_on_commit`参数来禁用此行为。更多详细信息请参见提交部分。

### 制定开始/提交/回滚块的框架

我们还可以将`Session.commit()`调用和事务的整体“框架”封装在上下文管理器中，以用于那些将数据提交到数据库的情况。所谓“框架”是指如果所有操作成功，则会调用`Session.commit()`方法，但如果引发任何异常，则会立即调用`Session.rollback()`方法，以便立即回滚事务，然后将异常传播出去。在 Python 中，这主要是通过`try:/except:/else:`块来表达的，例如：

```py
# verbose version of what a context manager will do
with Session(engine) as session:
    session.begin()
    try:
        session.add(some_object)
        session.add(some_other_object)
    except:
        session.rollback()
        raise
    else:
        session.commit()
```

上面示例的长形操作序列可以通过使用`Session.begin()`方法返回的`SessionTransaction`对象来更简洁地实现，该对象为相同操作序列提供了上下文管理器接口：

```py
# create session and add objects
with Session(engine) as session:
    with session.begin():
        session.add(some_object)
        session.add(some_other_object)
    # inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
```

更简洁地，这两个上下文可以结合使用：

```py
# create session and add objects
with Session(engine) as session, session.begin():
    session.add(some_object)
    session.add(some_other_object)
# inner context calls session.commit(), if there were no exceptions
# outer context calls session.close()
```

### 使用 sessionmaker

`sessionmaker`的目的是提供一个固定配置的`Session`对象工厂。由于典型的应用程序在模块范围内会有一个`Engine`对象，`sessionmaker`可以为与此引擎对应的`Session`对象提供一个工厂：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources, typically in module scope
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() without needing to pass the
# engine each time
with Session() as session:
    session.add(some_object)
    session.add(some_other_object)
    session.commit()
# closes the session
```

`sessionmaker` 与 `Engine` 类似，是用于函数级会话/连接的模块级工厂。因此，它还有自己的 `sessionmaker.begin()` 方法，类似于 `Engine.begin()`，它返回一个 `Session` 对象，并且还维护一个开始/提交/回滚块：

```py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# an Engine, which the Session will use for connection
# resources
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/")

# a sessionmaker(), also in the same scope as the engine
Session = sessionmaker(engine)

# we can now construct a Session() and include begin()/commit()/rollback()
# at once
with Session.begin() as session:
    session.add(some_object)
    session.add(some_other_object)
# commits the transaction, closes the session
```

在上述情况下，当上述 `with:` 块结束时，`Session` 将提交其事务，并且 `Session` 将关闭。

在编写应用程序时，`sessionmaker` 工厂应该与 `create_engine()` 创建的 `Engine` 对象保持相同的作用域，通常在模块级或全局级别。由于这些对象都是工厂，因此它们可以被任意数量的函数和线程同时使用。

另请参阅

`sessionmaker`

`Session`

### 查询

查询的主要方式是利用 `select()` 构造创建一个 `Select` 对象，然后使用诸如 `Session.execute()` 和 `Session.scalars()` 等方法执行以返回结果。结果然后以 `Result` 对象的形式返回，包括诸如 `ScalarResult` 等子变体。

SQLAlchemy ORM 查询指南提供了完整的 SQLAlchemy ORM 查询指南。以下是一些简要示例：

```py
from sqlalchemy import select
from sqlalchemy.orm import Session

with Session(engine) as session:
    # query for ``User`` objects
    statement = select(User).filter_by(name="ed")

    # list of ``User`` objects
    user_obj = session.scalars(statement).all()

    # query for individual columns
    statement = select(User.name, User.fullname)

    # list of Row objects
    rows = session.execute(statement).all()
```

从版本 2.0 开始更改：“2.0”样式查询现在是标准的。请参阅 2.0 迁移 - ORM 使用 查看从 1.x 系列迁移的注意事项。

另请参阅

ORM 查询指南

### 添加新的或现有的项目

`Session.add()` 用于将实例放入会话中。对于临时（即全新）的实例，这将在下一次刷新时导致对这些实例进行 INSERT。对于持久（即由此会话加载的）实例，它们已经存在，不需要添加。分离（即已从会话中移除的）实例可以使用此方法重新关联到会话中：

```py
user1 = User(name="user1")
user2 = User(name="user2")
session.add(user1)
session.add(user2)

session.commit()  # write changes to the database
```

要一次性向会话中添加项目列表，请使用`Session.add_all()`：

```py
session.add_all([item1, item2, item3])
```

`Session.add()` 操作**级联**执行`save-update`级联。更多详情请参阅 Cascades 部分。

### 删除

`Session.delete()` 方法将一个实例放入会话的待删除对象列表中：

```py
# mark two objects to be deleted
session.delete(obj1)
session.delete(obj2)

# commit (or flush)
session.commit()
```

`Session.delete()` 标记一个对象为删除状态，这将导致为每个受影响的主键发出 DELETE 语句。在挂起的删除被刷新之前，“delete”标记的对象存在于`Session.deleted`集合中。删除后，它们将从`Session`中移除，在事务提交后，这将变得永久。

与`Session.delete()`操作相关的各种重要行为，尤其是与其他对象和集合的关系如何处理的行为。有关此操作的更多信息，请参见 Cascades 部分，但通常规则如下：

+   通过`relationship()`指令与已删除对象相关联的映射对象对应的行**默认不会被删除**。如果这些对象对被删除的行有一个外键约束，则这些列将设置为 NULL。如果这些列是非空的，这将导致约束违反。

+   要将“SET NULL”更改为相关对象行的 DELETE，请使用`relationship()` 上的删除级联。

+   作为“多对多”表链接的表中的行，通过`relationship.secondary`参数，**在**所有情况下都会被删除，当它们引用的对象被删除时。

+   当相关对象包含返回到正在删除的对象的外键约束，并且它们所属的相关集合当前未加载到内存中时，工作单元将发出 SELECT 来获取所有相关行，以便它们的主键值可以用于发出 UPDATE 或 DELETE 语句在这些相关行上。通过这种方式，ORM 将在没有进一步指示的情况下执行 ON DELETE CASCADE 的功能，即使这在 Core `ForeignKeyConstraint`对象上进行了配置。

+   `relationship.passive_deletes`参数可用于调整此行为，并更自然地依赖于“ON DELETE CASCADE”；当设置为 True 时，此 SELECT 操作将不再发生，但是仍将对本地存在的行进行显式的 SET NULL 或 DELETE。将`relationship.passive_deletes`设置为字符串`"all"`将禁用**所有**相关对象的更新/删除。

+   当标记为删除的对象发生删除时，并不会自动从引用它的集合或对象引用中删除该对象。当`Session`过期时，这些集合可能会再次加载，以便对象不再存在。但是，最好不要对这些对象使用`Session.delete()`，而是应该从其集合中移除对象，然后使用 delete-orphan，以使其作为该集合移除的副作用而被删除。请参见部分关于删除 - 从集合和标量关系中删除引用的对象以获取示例。

参见

delete - 描述了“级联删除”，当主对象被删除时，会标记相关对象为删除。

delete-orphan - 描述了“孤立删除级联”，当它们从其主对象中取消关联时，会标记相关对象为删除。

删除说明 - 删除从集合和标量关系引用的对象 - 有关`Session.delete()`的重要背景，涉及在内存中刷新关系。

### 刷新

当`Session`以其默认配置使用时，刷新步骤几乎总是在透明地完成。具体来说，在由`Query`或 2.0 风格的`Session.execute()`调用引发任何单个 SQL 语句的情况下，以及在提交事务之前的`Session.commit()`调用之前，都会发生刷新。它还在使用`Session.begin_nested()`时发出 SAVEPOINT 前发生。

可以通过调用`Session.flush()`方法随时强制执行`Session`刷新：

```py
session.flush()
```

在某些方法的范围内自动发生的刷新称为**自动刷新**。自动刷新被定义为在以下方法开始时发生的可配置的自动刷新调用：

+   当针对启用 ORM 的 SQL 构造执行方法（如针对 ORM 实体和/或 ORM 映射属性的`select()`对象）使用`Session.execute()`和其他执行 SQL 的方法时

+   当调用`Query`来将 SQL 发送到数据库时

+   在查询数据库之前的`Session.merge()`方法内

+   当对象被刷新

+   当针对未加载对象属性进行 ORM 惰性加载 操作时。

也有一些刷新会**无条件**发生；这些点位于关键事务边界内，包括：

+   在`Session.commit()`方法的过程中

+   当调用`Session.begin_nested()`时

+   当使用`Session.prepare()` 2PC 方法时。

作为前面项目的一部分应用的**自动刷新**行为可以通过构造一个传递`Session`或`sessionmaker`的`Session.autoflush`参数为`False`来禁用：

```py
Session = sessionmaker(autoflush=False)
```

另外，自动刷新可以在使用`Session`的流程中暂时禁用，使用`Session.no_autoflush`上下文管理器：

```py
with mysession.no_autoflush:
    mysession.add(some_object)
    mysession.flush()
```

**重申一下：** 当调用事务方法（例如`Session.commit()`和`Session.begin_nested()`）时，刷新过程**总是发生**，而不管任何“自动刷新”设置如何，当`Session`还有未处理的待处理更改时。

由于`Session`仅在 DBAPI 事务的上下文中调用数据库的 SQL，所有“刷新”操作本身仅发生在数据库事务内（取决于数据库事务的隔离级别），前提是 DBAPI 不处于驱动程序级别的自动提交模式。这意味着假设数据库连接在其事务设置中提供了原子性，如果刷新中的任何单个 DML 语句失败，整个操作都将被回滚。

在刷新过程中发生故障时，为了继续使用相同的`Session`，即使底层事务已经回滚（即使数据库驱动程序在技术上处于驱动程序级别的自动提交模式），也需要在刷新失败后显式调用`Session.rollback()`，这样可以保持所谓的“子事务”的整体嵌套模式的一致性。“由于刷新期间发生的先前异常，此会话的事务已被回滚。”（或类似）FAQ 部分包含了对此行为的更详细描述。

另请参阅

“由于在刷新期间发生的先前异常，此会话的事务已回滚。”（或类似） - 关于在刷新失败时为什么必须调用 `Session.rollback()` 的更多背景信息。

### 按主键获取

由于 `Session` 使用的是一个 标识映射，它通过主键引用当前内存中的对象，因此 `Session.get()` 方法被提供作为一种通过主键定位对象的方法，首先查找当前标识映射，然后查询数据库以获取不存在的值。例如，要定位具有主键标识 `(5, )` 的 `User` 实体：

```py
my_user = session.get(User, 5)
```

`Session.get()` 还包括用于复合主键值的调用形式，可以作为元组或字典传递，以及允许特定加载器和执行选项的附加参数。参见 `Session.get()` 以获取完整的参数列表。

另请参阅

`Session.get()`

### 过期 / 刷新

在使用 `Session` 时经常会遇到的一个重要考虑因素是处理从数据库加载的对象上存在的状态，以保持它们与事务的当前状态同步。SQLAlchemy ORM 是基于一个 标识映射 的概念，即当从 SQL 查询中“加载”对象时，将维护一个与特定数据库标识相对应的唯一 Python 对象实例。这意味着如果我们发出两个单独的查询，每个查询都针对同一行，并获得一个映射对象，则这两个查询将返回相同的 Python 对象：

```py
>>> u1 = session.scalars(select(User).where(User.id == 5)).one()
>>> u2 = session.scalars(select(User).where(User.id == 5)).one()
>>> u1 is u2
True
```

由此可见，当 ORM 从查询中获取行时，它将**跳过已加载对象的属性的填充**。这里的设计假设是假设一个完全隔离的事务，然后根据事务的隔离程度，应用程序可以根据需要采取步骤从数据库事务中刷新对象。我正在使用我的会话重新加载数据，但它看不到我在其他地方提交的更改的 FAQ 条目更详细地讨论了这个概念。

当将 ORM 映射对象加载到内存中时，有三种常见方法可以使用当前事务的新数据刷新其内容：

+   **expire() 方法** - `Session.expire()` 方法将擦除对象的选定或所有属性的内容，以便在下次访问时从数据库加载它们，例如使用延迟加载模式：

    ```py
    session.expire(u1)
    u1.some_attribute  # <-- lazy loads from the transaction
    ```

+   **refresh() 方法** - 与之密切相关的是`Session.refresh()`方法，它执行`Session.expire()`方法的所有操作，但还会立即发出一个或多个 SQL 查询以实际刷新对象的内容：

    ```py
    session.refresh(u1)  # <-- emits a SQL query
    u1.some_attribute  # <-- is refreshed from the transaction
    ```

+   **populate_existing() 方法或执行选项** - 这现在是一个在 Populate Existing 文档中记录的执行选项；在传统形式中，它在`Query`对象上作为`Query.populate_existing()`方法找到。无论以哪种形式，此操作都表示从查询返回的对象应无条件地从数据库中重新填充：

    ```py
    u2 = session.scalars(
        select(User).where(User.id == 5).execution_options(populate_existing=True)
    ).one()
    ```

有关刷新 / 过期概念的进一步讨论，请参阅刷新 / 过期。

另请参阅

刷新 / 过期

我正在使用我的 Session 重新加载数据，但它没有看到我在其他地方提交的更改

### 使用任意 WHERE 子句的 UPDATE 和 DELETE

SQLAlchemy 2.0 包括增强功能，用于发出几种 ORM 启用的 INSERT、UPDATE 和 DELETE 语句。请参阅 ORM-Enabled INSERT, UPDATE, and DELETE statements 文档。

另请参阅

ORM-Enabled INSERT, UPDATE, and DELETE statements

使用自定义 WHERE 条件的 ORM UPDATE 和 DELETE

### 自动开始

`Session`对象具有称为**autobegin**的行为。这表示当使用`Session`执行任何工作时，无论涉及修改`Session`的内部状态以进行对象状态更改，还是涉及需要数据库连接的操作，`Session`将在内部认为自己处于“事务”状态。

当 `Session` 第一次被构造时，没有事务状态存在。当调用诸如 `Session.add()` 或 `Session.execute()` 这样的方法时，事务状态会自动开始，或者类似地，如果执行 `Query` 来返回结果（最终使用 `Session.execute()`），或者如果在 持久化 对象上修改属性。

可以通过访问 `Session.in_transaction()` 方法来检查事务状态，该方法返回 `True` 或 `False`，指示“自动开始”步骤是否已执行。虽然通常不需要，但 `Session.get_transaction()` 方法将返回表示此事务状态的实际 `SessionTransaction` 对象。

`Session` 的事务状态也可以通过显式调用 `Session.begin()` 方法来启动。当调用此方法时，`Session` 无条件地处于“事务”状态。`Session.begin()` 可以像 框架化一个 begin / commit / rollback 块 中描述的那样用作上下文管理器。

#### 禁用 Autobegin 以防止隐式事务

“自动开始”行为可以通过将`Session.autobegin`参数设置为`False`来禁用。通过使用此参数，`Session`将要求显式调用`Session.begin()`方法。在构造时，以及在调用任何`Session.rollback()`、`Session.commit()`或`Session.close()`方法之后，`Session`不会隐式开始任何新事务，并且如果尝试在未首先调用`Session.begin()`的情况下使用`Session`，将会引发错误：

```py
with Session(engine, autobegin=False) as session:
    session.begin()  # <-- required, else InvalidRequestError raised on next call

    session.add(User(name="u1"))
    session.commit()

    session.begin()  # <-- required, else InvalidRequestError raised on next call

    u1 = session.scalar(select(User).filter_by(name="u1"))
```

新版本 2.0 中：添加了 `Session.autobegin`，允许禁用“自动开始”行为 #### 禁用自动开始以防止隐式事务

“自动开始”行为可以通过将`Session.autobegin`参数设置为`False`来禁用。通过使用此参数，`Session`将要求显式调用`Session.begin()`方法。在构造时，以及在调用任何`Session.rollback()`、`Session.commit()`或`Session.close()`方法之后，`Session`不会隐式开始任何新事务，并且如果尝试在未首先调用`Session.begin()`的情况下使用`Session`，将会引发错误：

```py
with Session(engine, autobegin=False) as session:
    session.begin()  # <-- required, else InvalidRequestError raised on next call

    session.add(User(name="u1"))
    session.commit()

    session.begin()  # <-- required, else InvalidRequestError raised on next call

    u1 = session.scalar(select(User).filter_by(name="u1"))
```

新版本 2.0 中：添加了 `Session.autobegin`，允许禁用“自动开始”行为

### 提交

`Session.commit()` 用于提交当前事务。在其核心，这表示对所有当前具有正在进行事务的数据库连接发出`COMMIT`；从 DBAPI 的角度来看，这意味着在每个 DBAPI 连接上调用 `connection.commit()` DBAPI 方法。

当`Session`没有处于事务中时，表示自上次调用`Session.commit()`以来，对此`Session`没有调用操作，该方法将开始并提交一个仅“逻辑”事务，通常不会影响数据库，除非检测到待定的刷新更改，但仍然会调用事件处理程序和对象过期规则。

`Session.commit()` 操作在发出相关数据库连接的 COMMIT 之前无条件地发出 `Session.flush()`。如果未检测到待定更改，则不会向数据库发出 SQL。此行为不可配置，并且不受 `Session.autoflush` 参数的影响。

在此之后，假设`Session`绑定到一个`Engine`，则`Session.commit()`将 COMMIT 实际的数据库事务，如果已启动。提交后，与该事务关联的`Connection`对象将关闭，导致其底层的 DBAPI 连接被释放回与`Session`绑定的`Engine`相关联的连接池。

对于绑定到多个引擎（例如在分区策略中描述的那样）的`Session`，相同的 COMMIT 步骤将为每个在“逻辑”事务中使用的`Engine` / `Connection` 进行。除非启用了两阶段功能，否则这些数据库事务之间是不协调的。

还有其他的连接交互模式，可以直接将`Session`绑定到`Connection`上；在这种情况下，假定存在外部管理的事务，并且在这种情况下不会自动发出真正的 COMMIT；有关此模式的背景信息，请参见加入外部事务的会话（例如测试套件）一节。

最后，`Session`内的所有对象在事务关闭时都会被过期。这样，在下次访问实例时，无论是通过属性访问还是通过它们存在于 SELECT 结果中，它们都会接收到最新的状态。此行为可以通过`Session.expire_on_commit`标志来控制，当不希望这种行为时，可以将其设置为`False`。

另请参阅

自动开始

### 回滚

`Session.rollback()`回滚当前事务（如果有）。当没有事务时，该方法会静默地通过。

默认配置的会话（session）后，会话的事务回滚状态，其后续是通过自动开始或显式调用`Session.begin()`方法开始事务后的情况如下：

> +   数据库事务将被回滚。对于绑定到单个`Engine`的`Session`，这意味着针对当前正在使用的最多一个`Connection`发出 ROLLBACK。对于绑定到多个`Engine`对象的`Session`对象，针对所有被检出的`Connection`对象发出 ROLLBACK。
> +   
> +   数据库连接将被释放。这遵循了提交中指出的与连接相关的相同行为，其中从`Engine`对象获取的`Connection`对象将被关闭，导致 DBAPI 连接被释放到`Engine`中的连接池中。如果有新的事务开始，则会从`Engine`中检出新连接。
> +   
> +   对于直接绑定到`Connection`的`Session`，如将会话加入外部事务（比如测试套件），此`Connection`上的回滚行为将遵循`Session.join_transaction_mode`参数指定的行为，这可能涉及回滚保存点或发出真正的 ROLLBACK。
> +   
> +   在事务生命周期内将挂起状态的对象从添加到`Session`中时的状态是被移除的，对应于其 INSERT 语句的回滚。它们的属性状态保持不变。
> +   
> +   已删除对象在事务生命周期内被重新提升到持久化状态，对应其 DELETE 语句被回滚。请注意，如果这些对象首先在事务内为挂起状态，那么该操作将优先进行。
> +   
> +   所有未清除的对象都将完全过期 - 这与`Session.expire_on_commit`设置无关。

了解了该状态后，`Session`可以在回滚发生后安全地继续使用。

从 1.4 版本开始更改：`Session`对象现在具有延迟“开始”行为，如 autobegin 中所述。如果未开始任何事务，则`Session.commit()`和`Session.rollback()`等方法不会产生任何效果。在 1.4 版本之前不会观察到此行为，因为在非自动提交模式下，事务始终会隐式存在。

当`Session.flush()`失败时，通常是由于主键、外键或“非空”约束违反等原因，将自动发出 ROLLBACK（目前不可能在部分失败后继续 flush）。但是，此时`Session`处于一种称为“不活跃”的状态，并且调用应用程序必须始终显式调用`Session.rollback()`方法，以使`Session`可以恢复到可用状态（也可以简单地关闭和丢弃）。有关详细讨论，请参阅“由于刷新期间发生先前异常，此会话的事务已被回滚。”（或类似）的常见问题解答条目。

另请参阅

自动开始

### 结束

`Session.close()` 方法会调用 `Session.expunge_all()` 方法，该方法会将会话中的所有 ORM 映射对象移除，并且释放与其绑定的 `Engine` 对象的所有事务/连接资源。当连接返回到连接池时，事务状态也会回滚。

默认情况下，当 `Session` 被关闭时，它实际上处于最初构造时的原始状态，并且**可以再次使用**。从这个意义上讲，`Session.close()` 方法更像是一个“重置”回到干净状态，而不太像一个“数据库关闭”方法。在这种操作模式下，方法 `Session.reset()` 是 `Session.close()` 的别名，并且以相同的方式运行。

`Session.close()` 的默认行为可以通过设置参数 `Session.close_resets_only` 为 `False` 来更改，表示在调用方法 `Session.close()` 后，`Session` 不能被重复使用。在这种操作模式下，当 `Session.close_resets_only` 设置为 `True` 时，方法 `Session.reset()` 将允许多次“重置”会话，表现得像 `Session.close()`。

新功能在版本 2.0.22 中添加。

强烈建议通过调用 `Session.close()` 在结束时限制 `Session` 的范围，特别是如果没有使用 `Session.commit()` 或 `Session.rollback()` 方法。`Session` 可以作为上下文管理器使用，以确保调用 `Session.close()`：

```py
with Session(engine) as session:
    result = session.execute(select(User))

# closes session automatically
```

自 1.4 版更改：`Session` 对象具有延迟“开始”行为，如 autobegin 中所述，在调用 `Session.close()` 方法后不会立即开始新的事务。

## Session 常见问题

到了这个时候，许多用户已经对会话有了疑问。本节介绍了使用 `Session` 时常见问题的迷你常见问题解答（请注意我们也有一个真实常见问题解答）。

### 我什么时候使用 `sessionmaker`？

只需一次，在你应用程序的全局范围内的某个地方。它应该被视为应用程序配置的一部分。如果你的应用程序在一个包中有三个 .py 文件，你可以将 `sessionmaker` 行放在你的 `__init__.py` 文件中；从那时起，你的其他模块会说“from mypackage import Session”。这样，其他人只需使用 `Session()`，而该会话的配置由该中心点控制。

如果你的应用程序启动，进行导入，但不知道将连接到什么数据库，你可以稍后在“类”级别将 `Session` 绑定到引擎，使用 `sessionmaker.configure()`。

在本节的示例中，我们经常会在实际调用`Session`的代码行的上方展示`sessionmaker`的创建。但那只是为了举例而已！在实际情况中，`sessionmaker`可能会在模块级别的某处。然后，实例化`Session`的调用将放置在应用程序开始数据库交谈的地方。

### 我什么时候构建一个`Session`，什么时候提交它，什么时候关闭它？

通常在预期可能需要数据库访问的逻辑操作的开始处构造`Session`。

每当使用`Session`与数据库通信时，`Session`都会在开始通信时启动数据库事务。此事务将持续进行，直到`Session`被回滚、提交或关闭。如果再次使用它，则`Session`将开始一个新的事务，继续上一个事务结束的位置；因此，`Session`能够在多个事务中具有生命周期，尽管一次只能有一个事务。我们将这两个概念称为**事务范围**和**会话范围**。

通常情况下，确定`Session`的范围的最佳时机并不是很困难，尽管可能存在各种各样的应用架构，这可能会引入具有挑战性的情况。

一些示例场景包括：

+   Web 应用程序。在这种情况下，最好利用正在使用的 Web 框架提供的 SQLAlchemy 集成。或者，基本模式是在 Web 请求开始时创建一个`Session`，在执行 POST、PUT 或 DELETE 的 Web 请求结束时调用`Session.commit()` 方法，然后在 Web 请求结束时关闭会话。通常也是一个好主意将`Session.expire_on_commit`设置为 False，这样视图层中来自`Session`的对象的后续访问就不需要发出新的 SQL 查询来刷新对象，如果事务已经提交。

+   一个产生子进程的后台守护程序将希望为每个子进程创建一个本地的`Session`，在该子进程处理的“作业”的生命周期内使用该`Session`，然后在作业完成时将其关闭。

+   对于命令行脚本，应用程序将创建一个单一的全局`Session`，当程序开始工作时建立，完成任务时立即提交。

+   对于 GUI 接口驱动的应用程序，`Session` 的范围最好在用户生成的事件范围内，比如按钮点击。或者，范围可能对应于显式用户交互，比如用户“打开”一系列记录，然后“保存”它们。

作为一般规则，应用程序应该在**外部**管理会话的生命周期，而不是在处理特定数据的函数中。这是一种基本的关注点分离，使得特定于数据的操作与它们访问和操作数据的上下文无关。

例如**不要这样做**：

```py
### this is the **wrong way to do it** ###

class ThingOne:
    def go(self):
        session = Session()
        try:
            session.execute(update(FooBar).values(x=5))
            session.commit()
        except:
            session.rollback()
            raise

class ThingTwo:
    def go(self):
        session = Session()
        try:
            session.execute(update(Widget).values(q=18))
            session.commit()
        except:
            session.rollback()
            raise

def run_my_program():
    ThingOne().go()
    ThingTwo().go()
```

将会话的生命周期（通常是事务）**分开并置于外部**。下面的示例说明了这可能是什么样子，并且另外利用了 Python 上下文管理器（即`with:`关键字）来自动管理`Session`及其事务的范围：

```py
### this is a **better** (but not the only) way to do it ###

class ThingOne:
    def go(self, session):
        session.execute(update(FooBar).values(x=5))

class ThingTwo:
    def go(self, session):
        session.execute(update(Widget).values(q=18))

def run_my_program():
    with Session() as session:
        with session.begin():
            ThingOne().go(session)
            ThingTwo().go(session)
```

从版本 1.4 开始：`Session` 可以作为上下文管理器使用，而无需使用外部帮助函数。

### 会话是缓存吗？

是的……不。它在某种程度上被用作缓存，因为它实现了标识映射模式，并将对象按其主键键入存储。但它不执行任何查询缓存。这意味着，即使 `Foo(name='bar')` 就在那里，位于标识映射中，如果你说 `session.scalars(select(Foo).filter_by(name='bar'))`，会话也不知道那个。它必须向数据库发出 SQL，获取行，然后当它看到行中的主键时，*然后*它才能查看本地标识映射，并看到对象已经在那里。只有当你说 `query.get({some primary key})` 时，`Session` 才不需要发出查询。

此外，默认情况下，会话使用弱引用存储对象实例。这也使得将会话用作缓存失去了意义。

`Session` 并不是设计成每个人都可以作为“注册表”查阅的全局对象。这更像是**第二级缓存**的工作。SQLAlchemy 提供了使用 [dogpile.cache](https://dogpilecache.readthedocs.io/) 实现第二级缓存的模式，通过 Dogpile Caching 示例。

### 如何获取某个对象的 `Session`？

使用 `Session.object_session()` 类方法，该方法可用于 `Session`：

```py
session = Session.object_session(someobject)
```

较新的 Runtime Inspection API 系统也可以使用：

```py
from sqlalchemy import inspect

session = inspect(someobject).session
```

### 会话是否线程安全？AsyncSession 在并发任务中是否安全共享？

`Session` 是一个**可变的，有状态的**对象，表示一个**单个的数据库事务**。因此，`Session` 的实例**不能在并发线程或 asyncio 任务之间共享，除非进行了仔细的同步**。`Session` 应该以**非并发**的方式使用，也就是说，一个特定的 `Session` 实例一次只应该在一个线程或任务中使用。

当使用 SQLAlchemy 的 asyncio 扩展中的`AsyncSession`对象时，此对象仅是`Session`的一个薄代理，并且相同的规则适用；它是一个**非同步、可变、有状态的对象**，因此**不**安全在多个 asyncio 任务中同时使用单个`AsyncSession`实例。

一个`Session`或`AsyncSession`的实例表示一个单一的逻辑数据库事务，每次仅引用一个特定的`Engine`或`AsyncEngine`的单一`Connection`（注意，这些对象都支持同时绑定到多个引擎，但在这种情况下，在事务范围内仍然每个引擎只有一个连接在起作用）。

事务中的数据库连接也是一个有状态对象，旨在以非并发、顺序方式进行操作。命令按顺序在连接上发出，数据库服务器以发出的确切顺序处理它们。当`Session`发出针对此连接的命令并接收结果时，`Session`本身正在通过与此连接上存在的命令和数据状态相一致的内部状态变化进行过渡；这些状态包括是否开始、提交或回滚事务，如果有的话，正在起作用的 SAVEPOINT，以及将数据库行的状态与本地 ORM 映射对象的细粒度同步。

在设计并发数据库应用程序时，适当的模型是每个并发任务/线程都使用自己的数据库事务。这就是为什么在讨论数据库并发问题时，使用的标准术语是**多个并发事务**。在传统的关系数据库管理系统中，没有单个数据库事务的类似物，它可以同时接收和处理多个命令。

因此，SQLAlchemy 的 `Session` 和 `AsyncSession` 的并发模型是 **每个线程一个 Session，每个任务一个 AsyncSession**。一个使用多个线程或多个 asyncio 任务的应用程序，例如在使用 `asyncio.gather()` 这样的 API 时，会希望确保每个线程都有自己的`Session`，每个 asyncio 任务都有自己的 `AsyncSession`。

确保此用法的最佳方法是在位于线程或任务内的顶级 Python 函数中本地使用 标准上下文管理器模式，这将确保`Session`或`AsyncSession`的生命周期在本地范围内维护。

对于那些受益于拥有“全局”`Session`的应用程序，在不方便将`Session`对象传递给需要它的特定函数和方法的情况下，`scoped_session`方法可以提供一个“线程本地”的`Session`对象；详见 上下文/线程本地会话 部分。在 asyncio 上下文中，`async_scoped_session`对象是 `scoped_session` 的 asyncio 类比，然而更难配置，因为它需要一个自定义的“上下文”函数。

### 在什么时候我会使用`sessionmaker`？

只需要一次，在应用程序的全局范围内的某处。它应被视为应用程序配置的一部分。如果你的应用程序在一个包中有三个 .py 文件，你可以将`sessionmaker`行放在你的 `__init__.py` 文件中；从那时起，你的其他模块会说“from mypackage import Session”。这样，其他人只需使用`Session()`，而该会话的配置由该中心点控制。

如果你的应用程序启动，导入了模块，但不知道要连接到哪个数据库，你可以在之后将`Session`在“类”级别绑定到引擎上，使用`sessionmaker.configure()`。

在本节的示例中，我们经常会在实际调用`Session`的代码行的上方创建`sessionmaker`。但那只是为了举例说明！实际上，`sessionmaker`应该在模块级别的某个地方。然后，在应用程序中开始数据库会话的地方会放置对`Session`的实例化调用。

### 我什么时候构建`Session`，什么时候提交它，什么时候关闭它？

`Session`通常是在可能需要访问数据库的逻辑操作开始时构建的。

每当使用`Session`与数据库通信时，会立即开始一个数据库事务。该事务会一直持续到`Session`被回滚、提交或关闭。如果再次使用了`Session`，则会开始一个新的事务，这意味着`Session`可以跨越多个事务的生命周期，尽管一次只能有一个事务。我们将这两个概念称为**事务范围**和**会话范围**。

通常很容易确定开始和结束`Session`范围的最佳时机，尽管可能出现各种各样的应用程序架构，会引入具有挑战性的情况。

一些示例场景包括：

+   Web 应用程序。在这种情况下，最好利用所使用的 Web 框架提供的 SQLAlchemy 集成。或者，基本模式是在 Web 请求开始时创建一个`Session`，在进行 POST、PUT 或 DELETE 的 Web 请求结束时调用 `Session.commit()` 方法，然后在 Web 请求结束时关闭该会话。通常还建议将 `Session.expire_on_commit` 设置为 False，以便在视图层中访问来自 `Session` 的对象后，如果事务已经提交，就不需要发出新的 SQL 查询来刷新对象。

+   一个后台守护进程，会派生出子进程，希望为每个子进程创建一个`Session`，在子进程处理“任务”期间使用该`Session`，然后在任务完成时将其销毁。

+   对于命令行脚本，应用程序将创建一个单一的全局`Session`，当程序开始工作时建立，并在程序完成任务时立即提交。

+   对于 GUI 接口驱动的应用程序，`Session` 的范围最好在用户生成的事件范围内，例如按钮按下。或者，范围可能对应于显式用户交互，例如用户“打开”一系列记录，然后“保存”它们。

通常情况下，应用程序应该将会话的生命周期 *外部* 管理到处理特定数据的函数之外。这是一种将数据特定操作与它们访问和操作数据的上下文隔离开来的基本分离原则。

例如，**不要这样做**：

```py
### this is the **wrong way to do it** ###

class ThingOne:
    def go(self):
        session = Session()
        try:
            session.execute(update(FooBar).values(x=5))
            session.commit()
        except:
            session.rollback()
            raise

class ThingTwo:
    def go(self):
        session = Session()
        try:
            session.execute(update(Widget).values(q=18))
            session.commit()
        except:
            session.rollback()
            raise

def run_my_program():
    ThingOne().go()
    ThingTwo().go()
```

保持会话（通常是事务）的生命周期**分离和外部**。下面的示例说明了这可能的外观，并且另外利用了 Python 上下文管理器（即 `with:` 关键字）来自动管理`Session`及其事务的范围：

```py
### this is a **better** (but not the only) way to do it ###

class ThingOne:
    def go(self, session):
        session.execute(update(FooBar).values(x=5))

class ThingTwo:
    def go(self, session):
        session.execute(update(Widget).values(q=18))

def run_my_program():
    with Session() as session:
        with session.begin():
            ThingOne().go(session)
            ThingTwo().go(session)
```

从 1.4 版本开始更改：`Session` 可以作为上下文管理器使用，无需使用外部辅助函数。

### 会话是一个缓存吗？

不是的。在某种程度上它被用作缓存，因为它实现了身份映射模式，并将对象键入其主键。但是，它不会做任何类型的查询缓存。这意味着，如果你说 `session.scalars(select(Foo).filter_by(name='bar'))`，即使 `Foo(name='bar')` 正好在那里，在身份映射中，会话也不知道。它必须向数据库发出 SQL，获取行，然后当它看到行中的主键时，*然后*它可以查看本地身份映射并查看对象是否已经存在。只有当你说 `query.get({some primary key})` 时，`Session` 才不必发出查询。

另外，`Session` 默认使用弱引用存储对象实例。这也破坏了将 `Session` 用作缓存的目的。

`Session` 并不是设计成一个所有人都可以查询的全局对象作为“注册表”对象的。这更像是**二级缓存**的工作。SQLAlchemy 提供了使用 [dogpile.cache](https://dogpilecache.readthedocs.io/) 实现二级缓存的模式，通过 Dogpile Caching 示例。

### 如何获取某个对象的 `Session`？

使用 `Session.object_session()` 类方法，该方法在 `Session` 上可用：

```py
session = Session.object_session(someobject)
```

较新的 运行时检查 API 系统也可以使用：

```py
from sqlalchemy import inspect

session = inspect(someobject).session
```

### `Session` 线程安全吗？`AsyncSession` 在并发任务中安全吗？

`Session` 是一个**可变的、有状态的**对象，表示一个**单一的数据库事务**。因此，`Session` 的实例**不能在并发线程或 asyncio 任务之间共享，除非进行仔细的同步**。`Session` 应该以**非并发**方式使用，也就是说，特定的 `Session` 实例一次只能在一个线程或任务中使用。

当使用 SQLAlchemy 的 asyncio 扩展中的`AsyncSession`对象时，此对象只是`Session`的一个薄代理，相同的规则适用；它是一个**不同步的、可变的、有状态的对象**，因此**不**安全在多个 asyncio 任务中同时使用单个`AsyncSession`实例。

一个`Session`或`AsyncSession`实例代表一个单一的逻辑数据库事务，一次只引用一个特定`Engine`或`AsyncEngine`的单一`Connection`，该对象绑定到其中（请注意，这些对象都支持同时绑定到多个引擎，但在这种情况下，在事务范围内仍然每个引擎只有一个连接在使用）。

事务中的数据库连接也是一个有状态的对象，旨在以非并发、顺序方式进行操作。命令按顺序在连接上发出，数据库服务器按照发出的顺序精确处理它们。当`Session`在此连接上发出命令并接收结果时，`Session`本身正在通过与此连接上存在的命令和数据状态一致的内部状态更改过渡；这些状态包括事务是否已开始、已提交或已回滚，是否存在任何 SAVEPOINT，以及个别数据库行的状态与本地 ORM 映射对象的状态之间的细粒度同步。

在为并发设计数据库应用程序时，适当的模型是每个并发任务/线程使用自己的数据库事务。这就是为什么在讨论数据库并发问题时，使用的标准术语是**多个并发事务**。在传统的 RDMS 中，没有类似于同时接收和处理多个命令的单个数据库事务的类比。

因此，SQLAlchemy 的 `Session` 和 `AsyncSession` 的并发模型是 **每线程一个 Session，每任务一个 AsyncSession**。使用多个线程或在 asyncio 中使用多个任务（例如使用 `asyncio.gather()` 这样的 API）的应用程序将希望确保每个线程都有其自己的 `Session`，每个 asyncio 任务都有其自己的 `AsyncSession`。

确保此用法的最佳方式是在线程或任务内部的顶级 Python 函数中本地使用标准上下文管理器模式，这将确保 `Session` 或 `AsyncSession` 的生命周期在本地范围内保持不变。

对于从具有“全局” `Session` 中受益的应用程序，在不将 `Session` 对象传递给需要它的特定函数和方法的情况下，`scoped_session` 方法可以提供“线程本地” `Session` 对象；请参阅上下文/线程本地会话 部分了解背景。在 asyncio 上下文中，`async_scoped_session` 对象是 `scoped_session` 的 asyncio 版本，但更难配置，因为它需要自定义的“上下文”函数。
