# 使用引擎和连接

> 原文：[`docs.sqlalchemy.org/en/20/core/connections.html`](https://docs.sqlalchemy.org/en/20/core/connections.html)

本节详细介绍了 `Engine`、`Connection` 和相关对象的直接用法。值得注意的是，在使用 SQLAlchemy ORM 时，通常不直接访问这些对象；相反，`Session` 对象用作与数据库的接口。但是，对于以直接使用文本 SQL 语句和/或 SQL 表达式构造为中心，而不涉及 ORM 的高级管理服务的应用程序，`Engine` 和 `Connection` 是王者（和女王？）- 继续阅读。

## 基本用法

从引擎配置中回想起，`Engine` 是通过 `create_engine()` 调用创建的：

```py
engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test")
```

`create_engine()` 的典型用法是针对每个特定的数据库 URL，在单个应用程序进程的生命周期中全局持有一次。一个 `Engine` 管理了许多个体的 DBAPI 连接代表该进程，并且旨在以并发方式调用。`Engine` 与 DBAPI 的 `connect()` 函数**不**是同义词，后者只代表一个连接资源 - 当应用程序的模块级别仅创建一次时，`Engine` 在效率上最高，而不是每个对象或每个函数调用一次。

`Engine` 最基本的功能是提供对 `Connection` 的访问，然后可以调用 SQL 语句。向数据库发送文本语句的示例：

```py
from sqlalchemy import text

with engine.connect() as connection:
    result = connection.execute(text("select username from users"))
    for row in result:
        print("username:", row.username)
```

上面，`Engine.connect()` 方法返回一个`Connection` 对象，通过在 Python 上下文管理器中使用它（例如 `with:` 语句），`Connection.close()` 方法会在块结束时自动调用。`Connection` 是一个**代理**对象，用于实际的 DBAPI 连接。DBAPI 连接是在创建`Connection` 时从连接池中检索的。

返回的对象称为`CursorResult`，它引用一个 DBAPI 游标并提供类似于 DBAPI 游标的获取行的方法。当所有结果行（如果有）耗尽时，`CursorResult` 将关闭 DBAPI 游标。一个不返回行的`CursorResult`，例如没有返回行的 UPDATE 语句，会在构造时立即释放游标资源。

当`Connection` 在`with:`块结束时关闭时，引用的 DBAPI 连接将被释放到连接池中。从数据库本身的角度来看，连接池实际上不会“关闭”连接，假设池有空间来存储此连接以供下次使用。当连接返回到池中以供重新使用时，池机制会对 DBAPI 连接发出`rollback()`调用，以便删除任何事务状态或锁定（这被称为 Reset On Return），并且连接已准备好供下次使用。

我们上面的示例演示了执行文本 SQL 字符串，应该使用`text()` 构造来指示我们想要使用文本 SQL。`Connection.execute()` 方法当然可以容纳更多内容；请参阅 Working with Data 中的 SQLAlchemy Unified Tutorial 进行教程。

## 使用事务

注意

本节描述了在直接使用 `Engine` 和 `Connection` 对象时如何使用事务。当使用 SQLAlchemy ORM 时，事务控制的公共 API 是通过 `Session` 对象实现的，该对象在内部使用 `Transaction` 对象。有关更多信息，请参阅管理事务。

### 边做边提交

`Connection` 对象始终在事务块的上下文中发出 SQL 语句。第一次调用 `Connection.execute()` 方法执行 SQL 语句时，将自动开始此事务，使用的行为称为**自动开始**。事务保持在 `Connection` 对象的范围内，直到调用 `Connection.commit()` 或 `Connection.rollback()` 方法。在事务结束后，`Connection` 等待再次调用 `Connection.execute()` 方法，此时它会再次自动开始。

这种调用风格被称为**边做边提交**，在下面的示例中进行了说明：

```py
with engine.connect() as connection:
    connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

    connection.commit()  # commit the transaction
```

在“边做边提交”风格中，我们可以在使用 `Connection.execute()` 发出的其他语句序列中自由调用 `Connection.commit()` 和 `Connection.rollback()` 方法；每次事务结束并发出新语句时，都会隐式开始新事务：

```py
with engine.connect() as connection:
    connection.execute(text("<some statement>"))
    connection.commit()  # commits "some statement"

    # new transaction starts
    connection.execute(text("<some other statement>"))
    connection.rollback()  # rolls back "some other statement"

    # new transaction starts
    connection.execute(text("<a third statement>"))
    connection.commit()  # commits "a third statement"
```

2.0 版本新增：“边做边提交”风格是 SQLAlchemy 2.0 的新功能。在使用“未来”风格引擎时，它也可在 SQLAlchemy 1.4 的“过渡”模式中使用。

### 一次开始

`Connection`对象提供了一种更明确的事务管理风格，称为**一次性开始**。与“按照进度提交”相比，“一次性开始”允许显式声明事务的起始点，并允许事务本身可以被框定为上下文管理器块，以便事务的结束变得隐式。要使用“一次性开始”，使用`Connection.begin()`方法，该方法返回一个表示 DBAPI 事务的`Transaction`对象。此对象还通过其自身的`Transaction.commit()`和`Transaction.rollback()`方法支持显式管理，但作为首选实践，还支持上下文管理器接口，其中当块正常结束时，它将自行提交并在引发异常时发出回滚，然后将异常传播到外部。以下说明了“一次性开始”块的形式：

```py
with engine.connect() as connection:
    with connection.begin():
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

    # transaction is committed
```

### 连接并从引擎一次性开始

上述“一次性开始”块的方便缩写形式是在原始`Engine`对象的级别使用`Engine.begin()`方法，而不是执行两个分开的步骤`Engine.connect()`和`Connection.begin()`；`Engine.begin()`方法返回一个特殊的上下文管理器，该管理器内部同时维护`Connection`的上下文管理器以及通常由`Connection.begin()`方法返回的`Transaction`的上下文管理器：

```py
with engine.begin() as connection:
    connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

# transaction is committed, and Connection is released to the connection
# pool
```

提示

在`Engine.begin()`块中，我们可以调用`Connection.commit()`或`Connection.rollback()`方法，它们将提前结束由该块标记的事务。但是，如果我们这样做，直到该块结束之前，`Connection`上将不会发出进一步的 SQL 操作：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo=True)
>>> with e.begin() as conn:
...     conn.commit()
...     conn.begin()
2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine COMMIT
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: Can't operate on closed transaction inside
context manager.  Please complete the context manager before emitting
further commands.
```

### 混合风格

在一个`Engine.connect()`块中可以自由混合“随时提交”和“一次性开始”的风格，只要调用`Connection.begin()`不会与“自动开始”行为冲突。为了实现这一点，`Connection.begin()`应该在发出任何 SQL 语句之前或直接在前一次调用`Connection.commit()`或`Connection.rollback()`之后调用：

```py
with engine.connect() as connection:
    with connection.begin():
        # run statements in a "begin once" block
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})

    # transaction is committed

    # run a new statement outside of a block. The connection
    # autobegins
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

    # commit explicitly
    connection.commit()

    # can use a "begin once" block here
    with connection.begin():
        # run more statements
        connection.execute(...)
```

当开发使用“一次性开始”（begin once）的代码时，如果事务已经“自动开始”，库将引发`InvalidRequestError`。

## 设置事务隔离级别，包括 DBAPI 自动提交

大多数 DBAPI 都支持可配置的事务隔离级别的概念。传统上，这些级别有“READ UNCOMMITTED”、“READ COMMITTED”、“REPEATABLE READ”和“SERIALIZABLE”四个级别。这些通常在 DBAPI 连接开始新事务之前应用，注意大多数 DBAPI 在首次发出 SQL 语句时会隐式开始这个事务。

支持隔离级别的 DBAPI 通常也支持真正的“自动提交”概念，这意味着 DBAPI 连接本身将被放置在非事务性的自动提交模式中。这通常意味着数据库自动不再发出“BEGIN”，但也可能包括其他指令。SQLAlchemy 将“自动提交”的概念视为任何其他隔离级别；因为它是一个不仅丢失“读取提交”而且丢失原子性的隔离级别。

提示

需要注意的是，正如将在下面的部分进一步讨论的那样，在理解 DBAPI 级别的自动提交隔离级别中，“自动提交”隔离级别不像任何其他隔离级别一样，**不会**影响`Connection`对象的“事务”行为，该对象继续调用 DBAPI 的`.commit()`和`.rollback()`方法（它们在自动提交下没有任何效果），并且`.begin()`方法假定 DBAPI 将隐式启动一个事务（这意味着 SQLAlchemy 的“begin”**不会更改自动提交模式**）。

SQLAlchemy 方言应尽可能支持这些隔离级别以及自动提交。

### 设置连接的隔离级别或 DBAPI 自动提交

对于从 `Engine.connect()` 获取的单个 `Connection` 对象，可以使用 `Connection.execution_options()` 方法设置该 `Connection` 对象的隔离级别。该参数被称为 `Connection.execution_options.isolation_level`，其值是字符串，通常是以下名称的子集：

```py
# possible values for Connection.execution_options(isolation_level="<value>")

"AUTOCOMMIT"
"READ COMMITTED"
"READ UNCOMMITTED"
"REPEATABLE READ"
"SERIALIZABLE"
```

并非每个 DBAPI 都支持每个值；如果在某个后端使用不支持的值，则会引发错误。

例如，要在特定连接上强制执行 REPEATABLE READ，然后开始事务：

```py
with engine.connect().execution_options(
    isolation_level="REPEATABLE READ"
) as connection:
    with connection.begin():
        connection.execute(text("<statement>"))
```

提示

`Connection.execution_options()` 方法的返回值是调用该方法的相同 `Connection` 对象，这意味着它直接修改了 `Connection` 对象的状态。这是 SQLAlchemy 2.0 的新行为。这种行为不适用于 `Engine.execution_options()` 方法；该方法仍然返回一个 `Engine` 的副本，并且如下所述，可以用来构造具有不同执行选项的多个 `Engine` 对象，但仍共享相同的方言和连接池。

注

`Connection.execution_options.isolation_level` 参数在语句级别选项上不适用，例如 `Executable.execution_options()`，如果在此级别设置将被拒绝。这是因为该选项必须在每个事务的 DBAPI 连接上设置。

### 设置引擎的隔离级别或 DBAPI 自动提交

`Connection.execution_options.isolation_level` 选项也可以在引擎范围内设置，通常更可取。这可以通过将 `create_engine.isolation_level` 参数传递给 `create_engine()` 来实现：

```py
from sqlalchemy import create_engine

eng = create_engine(
    "postgresql://scott:tiger@localhost/test", isolation_level="REPEATABLE READ"
)
```

在上述设置下，每个新的 DBAPI 连接在创建时将被设置为使用`"REPEATABLE READ"`隔离级别设置来进行所有后续操作。

### 为单个引擎维护多个隔离级别

隔离级别也可以针对每个引擎进行设置，使用`create_engine.execution_options`参数或`Engine.execution_options()`方法，后者将创建一个共享方言和连接池但具有自己的每个连接隔离级别设置的`Engine`的副本：

```py
from sqlalchemy import create_engine

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    execution_options={"isolation_level": "REPEATABLE READ"},
)
```

在上述设置下，每个新启动的事务都将 DBAPI 连接设置为使用`"REPEATABLE READ"`隔离级别；但连接池中的连接将被重置为连接首次出现时存在的原始隔离级别。在`create_engine()`的级别上，最终效果与使用`create_engine.isolation_level`参数没有任何区别。

然而，经常选择在不同隔离级别中运行操作的应用程序可能希望为一个主`Engine`创建多个“子引擎”，每个引擎都配置为不同的隔离级别。其中一种用例是具有“事务性”和“只读”操作的应用程序，可以将一个单独的`Engine`从主引擎中分离出来，并使用`"AUTOCOMMIT"`：

```py
from sqlalchemy import create_engine

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")
```

上述，`Engine.execution_options()`方法创建了原始`Engine`的浅复制。`eng`和`autocommit_engine`共享相同的方言和连接池。然而，当从`autocommit_engine`获取连接时，将设置“AUTOCOMMIT”模式。

无论隔离级别设置为何种级别，当连接返回到连接池时，隔离级别都会无条件地恢复。

另请参阅

SQLite 事务隔离

PostgreSQL 事务隔离

MySQL 事务隔离

SQL Server 事务隔离

Oracle 事务隔离级别

设置事务隔离级别 / DBAPI 自动提交 - 用于 ORM

使用 DBAPI 自动提交允许进行只读版本的透明重新连接 - 一个使用 DBAPI 自动提交来透明重新连接到数据库以进行只读操作的示例 ### 理解 DBAPI 级别的自动提交隔离级别

在上一节中，我们介绍了 `Connection.execution_options.isolation_level` 参数的概念以及它如何用于设置数据库隔离级别，包括 SQLAlchemy 处理为另一个事务隔离级别的 DBAPI 级别的“自动提交”。在本节中，我们将尝试澄清这种方法的影响。

如果我们想要检出一个 `Connection` 对象并使用它的“自动提交”模式，我们将按以下步骤进行：

```py
with engine.connect() as connection:
    connection.execution_options(isolation_level="AUTOCOMMIT")
    connection.execute(text("<statement>"))
    connection.execute(text("<statement>"))
```

上面演示了“DBAPI 自动提交”模式的正常用法。不需要使用 `Connection.begin()` 或 `Connection.commit()` 等方法，因为所有语句都会立即提交到数据库。当块结束时，`Connection` 对象将恢复“自动提交”隔离级别，并且 DBAPI 连接将释放到连接池，在连接池中通常会调用 DBAPI `connection.rollback()` 方法，但由于上述语句已经提交，此回滚对数据库状态没有影响。

注意，“自动提交”模式即使在调用 `Connection.begin()` 方法时仍然持续存在；DBAPI 不会向数据库发送任何 BEGIN 命令，也不会在调用 `Connection.commit()` 时提交。这种用法也不是错误情况，因为可以预期“自动提交”隔离级别可能应用于原本假定事务上下文的代码；毕竟，“隔离级别”就像事务本身的配置细节一样。

在下面的示例中，无论是否有 SQLAlchemy 级别的事务块，语句都会保持**自动提交**状态：

```py
with engine.connect() as connection:
    connection = connection.execution_options(isolation_level="AUTOCOMMIT")

    # this begin() does not affect the DBAPI connection, isolation stays at AUTOCOMMIT
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
        connection.execute(text("<statement>"))
```

当我们打开日志记录并运行像上面这样的块时，日志记录将尝试指示，尽管调用了 DBAPI 级别的 `.commit()`，但由于自动提交模式，它可能不会起作用：

```py
INFO sqlalchemy.engine.Engine BEGIN (implicit)
...
INFO sqlalchemy.engine.Engine COMMIT using DBAPI connection.commit(), DBAPI should ignore due to autocommit mode
```

与此同时，即使我们使用“DBAPI 自动提交”，SQLAlchemy 的事务语义，即`Connection.begin()`的 Python 内部行为以及“自动开始”的行为，**仍然存在，即使这些不影响 DBAPI 连接本身**。举例说明，下面的代码将引发错误，因为在自动开始已经发生后调用了`Connection.begin()`：

```py
with engine.connect() as connection:
    connection = connection.execution_options(isolation_level="AUTOCOMMIT")

    # "transaction" is autobegin (but has no effect due to autocommit)
    connection.execute(text("<statement>"))

    # this will raise; "transaction" is already begun
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
```

上面的示例还展示了同样的主题，即“自动提交”隔离级别是底层数据库事务的配置细节，独立于 SQLAlchemy Connection 对象的开始/提交行为。 “自动提交”模式不会以任何方式与`Connection.begin()`交互，`Connection`在执行其自身与事务相关的状态更改时不会查询此状态（除了在引擎日志中建议这些块实际上没有提交）。这种设计的原因是为了保持与`Connection`完全一致的使用模式，在此模式中，DBAPI 自动提交模式可以独立更改，而无需指示任何其他代码更改。

#### 在隔离级别之间进行切换

当连接被释放回连接池时，隔离级别设置，包括自动提交模式，会自动重置。因此，最好避免尝试在单个`Connection`对象上切换隔离级别，因为这会导致冗余。

为了说明如何在单个`Connection`检出的范围内以即时模式使用“自动提交”，必须重新应用`Connection.execution_options.isolation_level`参数，以恢复先前的隔离级别。上一节示例了在自动提交进行时调用`Connection.begin()`以启动事务的尝试；我们可以通过在调用`Connection.begin()`之前先恢复隔离级别来重写该示例：

```py
# if we wanted to flip autocommit on and off on a single connection/
# which... we usually don't.

with engine.connect() as connection:
    connection.execution_options(isolation_level="AUTOCOMMIT")

    # run statement(s) in autocommit mode
    connection.execute(text("<statement>"))

    # "commit" the autobegun "transaction"
    connection.commit()

    # switch to default isolation level
    connection.execution_options(isolation_level=connection.default_isolation_level)

    # use a begin block
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
```

以上，在手动恢复隔离级别时，我们使用了`Connection.default_isolation_level`来恢复默认隔离级别（假设这是我们想要的）。然而，更好的做法可能是与`Connection`的架构一起工作，该架构已经在签入时自动处理重置隔离级别。编写上述内容的**首选**方法是使用两个块

```py
# use an autocommit block
with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as connection:
    # run statement in autocommit mode
    connection.execute(text("<statement>"))

# use a regular block
with engine.begin() as connection:
    connection.execute(text("<statement>"))
```

总而言之：

1.  “DBAPI 级别的自动提交”隔离级别与`Connection`对象的“开始”和“提交”概念完全独立。

1.  每个隔离级别使用单独的`Connection`签出。避免在单个连接签出之间尝试来回切换“自动提交”；让引擎完成恢复默认隔离级别的工作  ## 使用服务器端游标（又名流式结果）

一些后端特性明确支持“服务器端游标”和“客户端游标”的概念。这里的客户端游标意味着数据库驱动在语句执行完成之前完全将结果集中的所有行都提取到内存中。例如，PostgreSQL 和 MySQL/MariaDB 的驱动通常默认使用客户端游标。相比之下，服务器端游标表示结果行在客户端消耗时仍保持在数据库服务器状态中。例如，Oracle 的驱动通常使用“服务器端”模型，而 SQLite 方言虽然没有使用真正的“客户端/服务器”架构，但仍然使用一种未缓冲的结果提取方法，在结果行被消耗之前会将其留在进程内存之外。

从这个基本架构可以得出结论，当提取非常大的结果集时，“服务器端游标”在内存效率上更高，同时可能会在客户端/服务器通信过程中引入更多复杂性，并且对于小结果集（通常少于 10000 行）效率较低。

对于那些有条件支持缓冲或未缓冲结果的方言，通常对使用“未缓冲”或服务器端游标模式会有一些注意事项。例如，当使用 psycopg2 方言时，如果使用服务器端游标与任何类型的 DML 或 DDL 语句，将会引发错误。当使用 MySQL 驱动程序与服务器端游标时，DBAPI 连接处于更脆弱的状态，并且无法从错误条件中优雅地恢复，也不会允许回滚进行，直到游标完全关闭。

由于这个原因，SQLAlchemy 的方言将始终默认使用游标的较少出错版本，这意味着对于 PostgreSQL 和 MySQL 方言，默认情况下使用缓冲的“客户端”游标，在从游标调用任何提取方法之前，会将完整结果集拉入内存。这种操作模式在**绝大多数**情况下都是适用的；未缓冲的游标通常不是很有用，除非是在应用程序以块的方式提取大量行的罕见情况下，这些行的处理可以在提取更多行之前完成。

对于提供客户端和服务器端游标选项的数据库驱动程序，`Connection.execution_options.stream_results` 和 `Connection.execution_options.yield_per` 执行选项提供了在每个`Connection` 或每个语句基础上访问“服务器端游标”的能力。在使用 ORM `Session` 时也存在类似的选项。

### 通过 yield_per 进行固定缓冲区流式处理

由于完全未缓冲的服务器端游标的单个行提取操作通常比一次提取批量行要昂贵，`Connection.execution_options.yield_per` 执行选项配置了一个`Connection` 或语句以利用服务器端游标（如果可用），同时配置了一个固定大小的行缓冲区，该缓冲区将在消耗时按批次从服务器检索行。此参数可以使用`Connection.execution_options()` 方法在`Connection` 上或使用`Executable.execution_options()` 方法在语句上设置为正整数值。

新版本 1.4.40 中的新功能：`Connection.execution_options.yield_per` 作为仅限核心的选项是 SQLAlchemy 1.4.40 中的新功能；对于先前的 1.4 版本，请直接结合使用 `Connection.execution_options.stream_results` 和 `Result.yield_per()`。

使用此选项等同于手动设置`Connection.execution_options.stream_results`选项，如下一节所述，并在给定整数值的`Result`对象上调用`Result.yield_per()`方法。在这两种情况下，这种组合的效果包括：

+   为给定后端选择了服务器端游标模式，如果可用且尚未是该后端的默认行为

+   当获取结果行时，它们将被分批缓冲，每个批次的大小直到最后一个批次将等于传递给`Connection.execution_options.yield_per`选项或`Result.yield_per()`方法的整数参数；然后最后一个批次的大小将根据少于此大小的剩余行数确定

+   如果使用，`Result.partitions()`方法使用的默认分区大小也将设置为此整数大小。

以下示例说明了这三种行为：

```py
with engine.connect() as conn:
    with conn.execution_options(yield_per=100).execute(
        text("select * from table")
    ) as result:
        for partition in result.partitions():
            # partition is an iterable that will be at most 100 items
            for row in partition:
                print(f"{row}")
```

上面的示例说明了`yield_per=100`与使用`Result.partitions()`方法一起批量处理与从服务器获取的大小匹配的行的组合。使用`Result.partitions()`是可选的，如果直接迭代`Result`，每获取 100 行将缓冲一个新的批次行。不应使用诸如`Result.all()`之类的方法，因为这将一次性完全获取所有剩余行，从而破坏使用`yield_per`的目的。

提示

如上所示，`Result`对象可以用作上下文管理器。在使用服务器端游标进行迭代时，这是确保`Result`对象关闭的最佳方式，即使在迭代过程中引发异常。

`Connection.execution_options.yield_per` 选项也可以被 ORM 使用，被 `Session` 用于获取 ORM 对象，它还限制一次生成的 ORM 对象的数量。请参阅 使用 `Connection.execution_options.yield_per` 从大型结果集中提取 - 在 ORM 查询指南 中了解如何在 ORM 中使用 `Connection.execution_options.yield_per` 的更多背景信息。

新版本 1.4.40 中新增了`Connection.execution_options.yield_per`，作为一个核心级别的执行选项，可以方便地设置流式结果、缓冲区大小和分区大小，一次性完成，这种方式可以转移到 ORM 的类似用例中。

### 使用 `stream_results` 动态增长缓冲区进行流式传输

若要启用无特定分区大小的服务器端游标，可以使用 `Connection.execution_options.stream_results` 选项，这与 `Connection.execution_options.yield_per` 相似，可以在 `Connection` 对象或语句对象上调用。

当使用 `Connection.execution_options.stream_results` 选项传递的 `Result` 对象被直接迭代时，行会内部使用默认的缓冲方案进行获取，首先缓冲一小部分行，然后在每次提取时增加越来越大的缓冲区，直到预配置的 1000 行的限制。此缓冲区的最大大小可以通过 `Connection.execution_options.max_row_buffer` 执行选项受到影响：

```py
with engine.connect() as conn:
    with conn.execution_options(stream_results=True, max_row_buffer=100).execute(
        text("select * from table")
    ) as result:
        for row in result:
            print(f"{row}")
```

虽然`Connection.execution_options.stream_results`选项可以与`Result.partitions()`方法结合使用，但应向`Result.partitions()`传递特定的分区大小，以避免获取整个结果。通常，在设置使用`Result.partitions()`方法时，使用`Connection.execution_options.yield_per`选项更为简单。

另请参阅

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

`Result.partitions()`

`Result.yield_per()`  ## 模式名称的翻译

为支持将常见的表集分布到多个模式中的多租户应用程序，可以使用`Connection.execution_options.schema_translate_map`执行选项，以重新配置一组`Table`对象，使其在不进行任何更改的情况下呈现不同的模式名称。

给定一个表：

```py
user_table = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)
```

此`Table`的“模式”，由`Table.schema`属性定义，为 `None`。 `Connection.execution_options.schema_translate_map`可以指定所有模式为 `None` 的`Table`对象实际上将模式呈现为 `user_schema_one`：

```py
connection = engine.connect().execution_options(
    schema_translate_map={None: "user_schema_one"}
)

result = connection.execute(user_table.select())
```

上述代码将在数据库上调用如下形式的 SQL：

```py
SELECT  user_schema_one.user.id,  user_schema_one.user.name  FROM
user_schema_one.user
```

也就是说，模式名称被替换为我们翻译后的名称。该映射可以指定任意数量的目标->目标模式：

```py
connection = engine.connect().execution_options(
    schema_translate_map={
        None: "user_schema_one",  # no schema name -> "user_schema_one"
        "special": "special_schema",  # schema="special" becomes "special_schema"
        "public": None,  # Table objects with schema="public" will render with no schema
    }
)
```

`Connection.execution_options.schema_translate_map`参数影响从 SQL 表达式语言生成的所有 DDL 和 SQL 构造，这些构造是从`Table`或`Sequence`对象派生的。它不会影响通过`text()`构造使用的文本字符串 SQL，也不会影响通过`Connection.execute()`传递的纯字符串。

该特性仅在以下情况下生效，即模式名称直接来源于`Table`或`Sequence`的名称；它不会影响直接传递字符串模式名称的方法。按照这种模式，它在`MetaData.create_all()`或`MetaData.drop_all()`等方法执行的“可以创建”/“可以删除”检查中生效，并且在给定`Table`对象的情况下使用表反射时生效。然而，它不会影响`Inspector`对象上存在的操作，因为模式名称是显式传递给这些方法的。

提示

要在 ORM`Session`中使用模式翻译功能，请在`Engine`级别设置此选项，然后将该引擎传递给`Session`。`Session`为每个事务使用一个新的`Connection`：

```py
schema_engine = engine.execution_options(schema_translate_map={...})

session = Session(schema_engine)

...
```

警告

在没有扩展的情况下使用 ORM`Session`时，模式翻译功能仅支持**每个 Session 的单个模式翻译映射**。如果在每个语句的基础上提供了不同的模式翻译映射，则它**不会起作用**，因为 ORM`Session`不考虑用于单个对象的当前模式翻译值。

要在多个 `schema_translate_map` 配置中使用单个 `Session`，可以使用 Horizontal Sharding 扩展。请参阅 Horizontal Sharding 中的示例。## SQL 编译缓存

新功能：1.4 版本中新增了 SQLAlchemy 具有透明查询缓存系统，大大降低了在 Core 和 ORM 中将 SQL 语句构造转换为 SQL 字符串所涉及的 Python 计算开销。请参阅 Transparent SQL Compilation Caching added to All DQL, DML Statements in Core, ORM 中的介绍。

SQLAlchemy 包含了一个全面的 SQL 编译器缓存系统，以及其 ORM 变体。该缓存系统在 `Engine` 内部是透明的，并提供了对于给定的 Core 或 ORM SQL 语句的 SQL 编译过程，以及为该语句组装结果获取机制的相关计算，只会对具有相同结构的语句对象执行一次，而对于具有相同结构的引擎的“已编译缓存”中的所有其他语句对象，在特定结构保持在引擎的“已编译缓存”中的持续时间内。所谓“具有相同结构的语句对象”，通常对应于在函数内构造的 SQL 语句，并且每次运行该函数时都会构建该语句：

```py
def run_my_statement(connection, parameter):
    stmt = select(table)
    stmt = stmt.where(table.c.col == parameter)
    stmt = stmt.order_by(table.c.id)
    return connection.execute(stmt)
```

上述语句将生成类似于 `SELECT id, col FROM table WHERE col = :col ORDER BY id` 的 SQL 语句，注意，虽然 `parameter` 的值是一个普通的 Python 对象，比如一个字符串或一个整数，但该语句的字符串 SQL 形式不包括该值，因为它使用了绑定参数。在 `connection.execute()` 调用的范围内，上述 `run_my_statement()` 函数的后续调用将使用缓存的编译结构，以提高性能。

注意

需要注意的是，SQL 编译缓存仅缓存**传递给数据库的 SQL 字符串**，而不是查询返回的数据。它绝对不是数据缓存，也不会影响返回特定 SQL 语句的结果，也不意味着与提取结果行相关联的内存使用。

虽然 SQLAlchemy 从早期的 1.x 系列就有了一个基本的语句缓存，并且还在 ORM 中提供了“Baked Query”扩展，但这两个系统都需要高度特殊的 API 使用才能使缓存生效。从 1.4 版本开始的新缓存完全是自动的，无需改变编程风格即可生效。

该缓存会自动使用，无需进行任何配置更改，也不需要任何特殊步骤来启用它。以下各节详细介绍了缓存的配置和高级使用模式。

### 配置

缓存本身是一个名为 `LRUCache` 的类似于字典的对象，它是 SQLAlchemy 的内部字典子类，跟踪特定键的使用情况，并在缓存大小达到某个阈值时特性地执行定期的“修剪”步骤，移除最近最少使用的项目。此缓存的大小默认为 500，并且可以使用 `create_engine.query_cache_size` 参数进行配置：

```py
engine = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test", query_cache_size=1200
)
```

缓存的大小可以增长到给定大小的 150%，然后再减少到目标大小。因此，1200 大小的缓存可以增长到 1800 大小，然后被修剪到 1200。

缓存的大小基于每个引擎渲染的每个唯一 SQL 语句条目。从 Core 和 ORM 生成的 SQL 语句被平等对待。DDL 语句通常不会被缓存。为了确定缓存的作用，引擎日志将包括有关缓存行为的详细信息，下一节将对其进行描述。

### 使用日志估算缓存性能

上述 1200 大小的缓存实际上相当大。对于小型应用程序，100 大小可能足够。为了估算缓存的最佳大小，假设目标主机上存在足够的内存，缓存的大小应基于可能为使用的目标引擎渲染的唯一 SQL 字符串数量。查看此的最便捷方法是使用 SQL 回显，可以通过使用 `create_engine.echo` 标志直接启用，也可以通过使用 Python 日志记录；有关日志配置的背景，请参阅配置日志部分。

作为示例，我们将检查以下程序产生的日志记录：

```py
from sqlalchemy import Column
from sqlalchemy import create_engine
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy import String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy.orm import Session

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(String)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

s = Session(e)

s.add_all([A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()])])
s.commit()

for a_rec in s.scalars(select(A)):
    print(a_rec.bs)
```

运行时，每个记录的 SQL 语句将在传递的参数左侧包括一个带括号的缓存统计徽章。我们可能看到的四种消息类型如下所示：

+   `[原始 SQL]` - 使用 `Connection.exec_driver_sql()` 方法的驱动程序或最终用户发出的原始 SQL - 不适用缓存

+   `[无键]` - 语句对象是一个不被缓存的 DDL 语句，或者语句对象包含不可缓存的元素，例如用户定义的结构或任意大的 VALUES 子句。

+   `[在 X 秒内生成]` - 该语句是一个**缓存未命中**，必须编译，然后存储在缓存中。生成编译结构花费了 X 秒。数字 X 将是很小的分数秒。

+   `[cached since Xs ago]` - 该语句是**缓存命中**，无需重新编译。该语句自 X 秒前起已存储在缓存中。数字 X 将与应用程序运行时间和语句缓存时间成比例，例如，对于一个 24 小时的时间段，X 将为 86400。

每个徽章的详细描述如下。

上述程序中我们看到的第一条语句是 SQLite 方言检查“a”和“b”表是否存在：

```py
INFO sqlalchemy.engine.Engine PRAGMA temp.table_info("a")
INFO sqlalchemy.engine.Engine [raw sql] ()
INFO sqlalchemy.engine.Engine PRAGMA main.table_info("b")
INFO sqlalchemy.engine.Engine [raw sql] ()
```

对于上述两个 SQLite PRAGMA 语句，徽章显示为`[raw sql]`，这表示驱动程序直接将 Python 字符串通过`Connection.exec_driver_sql()`发送到数据库。这些语句不适用于缓存，因为它们已经以字符串形式存在，而且由于 SQLAlchemy 不提前解析 SQL 字符串，对将返回的结果行的类型也一无所知。

我们看到的下一条语句是 CREATE TABLE 语句：

```py
INFO  sqlalchemy.engine.Engine
CREATE  TABLE  a  (
  id  INTEGER  NOT  NULL,
  data  VARCHAR,
  PRIMARY  KEY  (id)
)

INFO  sqlalchemy.engine.Engine  [no  key  0.00007s]  ()
INFO  sqlalchemy.engine.Engine
CREATE  TABLE  b  (
  id  INTEGER  NOT  NULL,
  a_id  INTEGER,
  data  VARCHAR,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(a_id)  REFERENCES  a  (id)
)

INFO  sqlalchemy.engine.Engine  [no  key  0.00006s]  ()
```

对于每个语句，徽章显示为`[no key 0.00006s]`。这表示这两个特定语句，由于 DDL 导向的`CreateTable`构造未生成缓存键，因此未发生缓存。DDL 构造通常不参与缓存，因为它们通常不会被重复执行第二次，而且 DDL 也是数据库配置步骤，性能并不那么关键。

`[no key]`徽章之所以重要，是因为它可以用于生成可缓存的 SQL 语句，除了某些当前不可缓存的特定子构造。这些包括未定义缓存参数的自定义用户定义 SQL 元素，以及生成任意长且不可重现的 SQL 字符串的某些构造，主要示例包括`Values`构造以及使用`Insert.values()`方法进行“多值插入”时。

到目前为止，我们的缓存仍然是空的。下一条语句将被缓存，一个段落看起来像：

```py
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [generated  in  0.00011s]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0003533s  ago]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0005326s  ago]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [generated  in  0.00010s]  (1,  None)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0003232s  ago]  (1,  None)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0004887s  ago]  (1,  None)
```

在上面，我们实际上看到了两个唯一的 SQL 字符串；`"INSERT INTO a (data) VALUES (?)"`和`"INSERT INTO b (a_id, data) VALUES (?, ?)"`。由于 SQLAlchemy 对所有文字值使用绑定参数，即使这些语句为不同对象重复多次，由于参数是分开的，实际的 SQL 字符串保持不变。

注意

上述两个语句是由 ORM 工作单元生成的，实际上会将这些语句缓存在每个映射器的本地缓存中。但是机制和术语是相同的。下面的章节禁用或使用备用字典缓存某些（或全部）语句将描述用户代码如何基于每个语句使用备用缓存容器。

我们看到的每个这两个语句的第一次出现的缓存徽章是`[generated in 0.00011s]`。这表示该语句**不在缓存中，在 0.00011 秒内编译成字符串，然后被缓存**。当我们看到`[generated]`徽章时，我们知道这意味着发生了**缓存未命中**。对于特定语句的第一次出现，这是可以预料的。然而，如果长时间运行的应用程序通常一遍又一遍地使用相同系列的 SQL 语句，但是观察到了大量新的`[generated]`徽章，那可能意味着`create_engine.query_cache_size`参数设置得太小。当从缓存中驱逐了被缓存的语句，因为 LRU 缓存修剪了较少使用的项目时，当下次使用它时，它将显示`[generated]`徽章。

对于每个这两个语句的后续出现，我们看到的缓存徽章看起来像是`[cached since 0.0003533s ago]`。这表示该语句**在缓存中被找到，并且最初放入缓存中 0.0003533 秒前**。需要注意的是，虽然`[generated]`和`[cached since]`徽章都指的是秒数，但它们表示的含义不同；在`[generated]`的情况下，该数字是编译该语句所需的大致时间，并且会是一个极小的时间量。在`[cached since]`的情况下，这是语句在缓存中存在的总时间。对于运行了六个小时的应用程序，该数字可能会显示`[cached since 21600 seconds ago]`，这是个好事。看到“cached since”的数字很高表明这些语句很长时间以来都没有发生缓存未命中。即使应用程序运行了很长时间，但是经常出现“cached since”的低数字的语句可能表明这些语句太频繁地发生了缓存未命中，并且可能需要增加`create_engine.query_cache_size`参数。

我们的示例程序然后执行了一些 SELECT 查询，在这些查询中，我们可以看到“generated”然后“cached”的相同模式，对于“a”表的 SELECT 以及“b”表的后续惰性加载也是如此：

```py
INFO sqlalchemy.engine.Engine SELECT a.id AS a_id, a.data AS a_data
FROM a
INFO sqlalchemy.engine.Engine [generated in 0.00009s] ()
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
INFO sqlalchemy.engine.Engine [generated in 0.00010s] (1,)
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
INFO sqlalchemy.engine.Engine [cached since 0.0005922s ago] (2,)
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
```

根据我们上面的程序，完整运行显示总共缓存了四个不同的 SQL 字符串。这表明缓存大小为**四**足以。这显然是一个极小的大小，500 的默认大小可以放心使用。

### 缓存使用了多少内存？

前面的章节详细介绍了一些技术，用于检查`create_engine.query_cache_size`是否需要增加。我们如何知道缓存大小不太大？我们可能希望将`create_engine.query_cache_size`设置为不超过某个数字的原因是，我们的应用程序可能会使用非常大量的不同语句，例如一个从搜索 UX 动态构建查询的应用程序，如果过去 24 小时运行了十万条不同的查询并且它们都被缓存，我们不希望我们的主机因为内存不足而运行失败。

很难测量 Python 数据结构占用了多少内存，但是通过使用`top`来测量内存增长，当连续添加了 250 个新语句时，建议一个中等核心语句大约占用 12K，而一个小型 ORM 语句大约占用 20K，包括为 ORM 获取结果的结构，这些结构对于 ORM 来说要大得多。

### 禁用或使用替代字典缓存一些（或全部）语句

内部使用的缓存称为`LRUCache`，但这基本上只是一个字典。可以使用`Connection.execution_options.compiled_cache`选项作为执行选项的一部分，通过在语句上设置执行选项，在`Engine`或`Connection`上设置执行选项，或者在使用 SQLAlchemy-2.0 样式调用 ORM `Session.execute()` 方法时设置执行选项，使用任何字典作为任何语句系列的缓存。例如，要运行一系列 SQL 语句并将它们缓存到特定字典中：

```py
my_cache = {}
with engine.connect().execution_options(compiled_cache=my_cache) as conn:
    conn.execute(table.select())
```

SQLAlchemy ORM 使用上述技术在单元工作“flush”过程中保留每个 mapper 的缓存，这些缓存与`Engine`上配置的默认缓存分开，以及用于某些关系加载器查询。

使用该参数将缓存禁用，方法是发送一个值为`None`：

```py
# disable caching for this connection
with engine.connect().execution_options(compiled_cache=None) as conn:
    conn.execute(table.select())
```  ### 第三方方言的缓存

缓存功能要求方言的编译器生成的 SQL 字符串可以安全地用于许多语句调用，给定一个特定的缓存键，该键与该 SQL 字符串相关联。这意味着语句中的任何字面值，例如 SELECT 的 LIMIT/OFFSET 值，不能在方言的编译方案中硬编码，因为编译后的字符串将无法重用。SQLAlchemy 支持使用`BindParameter.render_literal_execute()`方法渲染绑定参数，该方法可以应用于现有的 `Select._limit_clause` 和 `Select._offset_clause` 属性，由自定义编译器实现，后文将对此进行说明。

由于有许多第三方方言，其中许多可能会从 SQL 语句中生成字面值，而不使用更新的“字面执行”功能，因此自 SQLAlchemy 1.4.5 起，为方言添加了一个名为`Dialect.supports_statement_cache`的属性。此属性在运行时直接在特定方言类上检查其是否存在，即使它已经存在于超类上，因此，即使第三方方言是现有可缓存的 SQLAlchemy 方言的子类，例如`sqlalchemy.dialects.postgresql.PGDialect`，仍必须显式包含此属性以启用缓存。该属性应仅在方言经过必要的修改并测试过使用不同参数的编译 SQL 语句的可重用性后才能**启用**。

对于所有不支持此属性的第三方方言，其日志将指示`方言不支持缓存`。

当一个方言已经针对缓存进行了测试，特别是 SQL 编译器已更新为不直接在 SQL 字符串中渲染任何字面值 LIMIT / OFFSET 时，方言作者可以如下应用该属性：

```py
from sqlalchemy.engine.default import DefaultDialect

class MyDialect(DefaultDialect):
    supports_statement_cache = True
```

该标志也需要应用于所有方言子类：

```py
class MyDBAPIForMyDialect(MyDialect):
    supports_statement_cache = True
```

新版本 1.4.5 中新增了`Dialect.supports_statement_cache`属性。

典型的方言修改案例如下。

#### 示例：使用后编译参数渲染 LIMIT / OFFSET

举例来说，假设一个方言重写了`SQLCompiler.limit_clause()`方法，该方法为 SQL 语句生成“LIMIT / OFFSET”子句，像这样：

```py
# pre 1.4 style code
def limit_clause(self, select, **kw):
    text = ""
    if select._limit is not None:
        text += " \n LIMIT %d" % (select._limit,)
    if select._offset is not None:
        text += " \n OFFSET %d" % (select._offset,)
    return text
```

以上例程将`Select._limit`和`Select._offset`整数值呈现为嵌入在 SQL 语句中的字面整数。这是对于不支持在 SELECT 语句的 LIMIT/OFFSET 子句中使用绑定参数的数据库的常见要求。然而，在初始编译阶段内呈现整数值与缓存直接**不兼容**，因为`Select`对象的限制和偏移整数值不是缓存键的一部分，因此许多具有不同限制/偏移值的`Select`语句将无法正确呈现值。

以上代码的更正是将字面整数移入 SQLAlchemy 的后编译设施，这将使字面整数在初始编译阶段之外渲染，而是在执行时在语句发送到 DBAPI 之前。这在编译阶段使用`BindParameter.render_literal_execute()`方法访问，同时使用`Select._limit_clause`和`Select._offset_clause`属性，它们表示 LIMIT/OFFSET 作为完整的 SQL 表达式，如下所示：

```py
# 1.4 cache-compatible code
def limit_clause(self, select, **kw):
    text = ""

    limit_clause = select._limit_clause
    offset_clause = select._offset_clause

    if select._simple_int_clause(limit_clause):
        text += " \n LIMIT %s" % (
            self.process(limit_clause.render_literal_execute(), **kw)
        )
    elif limit_clause is not None:
        # assuming the DB doesn't support SQL expressions for LIMIT.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for LIMIT"
        )
    if select._simple_int_clause(offset_clause):
        text += " \n OFFSET %s" % (
            self.process(offset_clause.render_literal_execute(), **kw)
        )
    elif offset_clause is not None:
        # assuming the DB doesn't support SQL expressions for OFFSET.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for OFFSET"
        )

    return text
```

上述方法将生成一个编译后的 SELECT 语句，看起来像：

```py
SELECT  x  FROM  y
LIMIT  __[POSTCOMPILE_param_1]
OFFSET  __[POSTCOMPILE_param_2]
```

在上述情况下，`__[POSTCOMPILE_param_1]`和`__[POSTCOMPILE_param_2]`指示符将在语句执行时填充其相应的整数值，此时 SQL 字符串已从缓存中检索出来。

在适当进行类似上述更改后，`Dialect.supports_statement_cache`标志应设置为`True`。强烈建议第三方方言使用[dialect 第三方测试套件](https://github.com/sqlalchemy/sqlalchemy/blob/main/README.dialects.rst)，该套件将断言具有 LIMIT/OFFSET 的 SELECT 操作是否正确呈现和缓存。

另请参阅

为什么升级到 1.4 和/或 2.x 后我的应用程序变慢？ - 在常见问题解答部分  ### 使用 Lambda 在语句生成中获得显著的速度提升

深度炼金术

除非在非常性能密集的场景下，否则这种技术通常是非必要的，并且适用于有经验的 Python 程序员。虽然相当简单，但涉及到元编程概念，对于初学者的 Python 开发者来说并不合适。Lambda 方法可以在稍后以最小的努力应用于现有代码。

Python 函数通常表示为 lambda，可以用来生成基于 lambda 函数本身的 Python 代码位置以及 lambda 内部的闭包变量可缓存的 SQL 表达式。 其基本原理是允许缓存不仅是 SQL 表达式构造的 SQL 字符串编译形式，这是 SQLAlchemy 在未使用 lambda 系统时的正常行为，而且也是 SQL 表达式构造本身在 Python 中的组合，这也具有一定的 Python 开销。

lambda SQL 表达式功能可用作性能增强功能，并且也可以选择在 `with_loader_criteria()` ORM 选项中使用，以提供通用 SQL 片段。

#### 概要

Lambda 语句是使用 `lambda_stmt()` 函数构造的，该函数返回 `StatementLambdaElement` 的实例，它本身是一个可执行的语句构造。 使用 Python 的加法运算符 `+`，或者 `StatementLambdaElement.add_criteria()` 方法添加附加修饰符和条件到对象中，该方法允许更多的选项。

假设在 Python 中的一个封闭函数或方法内调用了 `lambda_stmt()` 构造函数，以便在应用程序中多次使用，以便在第一次执行之后的后续执行可以利用已缓存的编译 SQL。 当 lambda 在 Python 的封闭函数内构造时，它也将受到闭包变量的影响，这对整个方法非常重要：

```py
from sqlalchemy import lambda_stmt

def run_my_statement(connection, parameter):
    stmt = lambda_stmt(lambda: select(table))
    stmt += lambda s: s.where(table.c.col == parameter)
    stmt += lambda s: s.order_by(table.c.id)

    return connection.execute(stmt)

with engine.connect() as conn:
    result = run_my_statement(some_connection, "some parameter")
```

在上面，用于定义 SELECT 语句结构的三个 `lambda` 可调用对象只被调用一次，并且生成的 SQL 字符串被缓存在引擎的编译缓存中。 从那时起，`run_my_statement()` 函数可以被调用任意次数，而其中的 `lambda` 可调用对象不会被调用，只用作缓存键来检索已经编译的 SQL。

注意

当 lambda 系统未被使用时，已经存在 SQL 缓存，这一点很重要。 lambda 系统只是在每个 SQL 语句调用时通过缓存 SQL 构建本身和使用更简单的缓存键来添加额外的工作减少层。

#### Lambda 的快速指南

在 lambda SQL 系统中，重点是确保为 lambda 生成的缓存键与其将产生的 SQL 字符串永远不会不匹配。`LambdaElement` 及其相关对象将运行并分析给定的 lambda，以便计算在每次运行时应该如何对其进行缓存，尝试检测任何潜在问题。基本指南包括：

+   **任何类型的语句都受支持** - 虽然预期 `select()` 构造是 `lambda_stmt()` 的主要用例，但 DML 语句如 `insert()` 和 `update()` 同样可用：

    ```py
    def upd(id_, newname):
        stmt = lambda_stmt(lambda: users.update())
        stmt += lambda s: s.values(name=newname)
        stmt += lambda s: s.where(users.c.id == id_)
        return stmt

    with engine.begin() as conn:
        conn.execute(upd(7, "foo"))
    ```

+   **ORM 支持的用例直接得到支持** - `lambda_stmt()` 可以完全适应 ORM 功能，并直接与 `Session.execute()` 一起使用：

    ```py
    def select_user(session, name):
        stmt = lambda_stmt(lambda: select(User))
        stmt += lambda s: s.where(User.name == name)

        row = session.execute(stmt).first()
        return row
    ```

+   **绑定参数会自动适应** - 与 SQLAlchemy 之前的“烘焙查询”系统相比，lambda SQL 系统会自动适应成为 SQL 绑定参数的 Python 文字值。这意味着即使一个给定的 lambda 只运行一次，每次运行时生成的绑定参数的值也会从 lambda 的 **闭包** 中提取出来：

    ```py
    >>> def my_stmt(x, y):
    ...     stmt = lambda_stmt(lambda: select(func.max(x, y)))
    ...     return stmt
    >>> engine = create_engine("sqlite://", echo=True)
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    ...     print(conn.scalar(my_stmt(12, 8)))
    SELECT  max(?,  ?)  AS  max_1
    [generated  in  0.00057s]  (5,  10)
    10
    SELECT  max(?,  ?)  AS  max_1
    [cached  since  0.002059s  ago]  (12,  8)
    12
    ```

    在上面的例子中，`StatementLambdaElement` 从每次调用 `my_stmt()` 时生成的 lambda 的 **闭包** 中提取了 `x` 和 `y` 的值；这些值被替换为参数的值，并缓存到 SQL 结构中。

+   **最好的情况是 lambda 在所有情况下产生相同的 SQL 结构** - 避免在 lambda 内部使用条件语句或自定义可调用对象，因为这可能会根据输入产生不同的 SQL；如果一个函数可能会有条件地使用两个不同的 SQL 片段，则使用两个单独的 lambda：

    ```py
    # **Don't** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        stmt += lambda s: (
            s.where(table.c.x > parameter) if thing else s.where(table.c.y == parameter)
        )
        return stmt

    # **Do** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        if thing:
            stmt += lambda s: s.where(table.c.x > parameter)
        else:
            stmt += lambda s: s.where(table.c.y == parameter)
        return stmt
    ```

    如果 lambda 表达式不能生成一致的 SQL 结构，则可能会发生各种故障，并且有些故障目前无法轻松检测到。

+   **不要在 lambda 中使用函数来生成绑定值** - 绑定值跟踪方法要求 SQL 语句中要使用的实际值在 lambda 的闭包中本地存在。如果值是从其他函数生成的，则这是不可能的，并且如果尝试这样做，`LambdaElement` 通常会引发错误：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     stmt = lambda_stmt(lambda: select(func.max(get_x(), get_y())))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Can't invoke Python callable get_x()
    inside of lambda expression argument at
    <code object <lambda> at 0x7fed15f350e0, file "<stdin>", line 6>;
    lambda SQL constructs should not invoke functions from closure variables
    to produce literal values since the lambda SQL system normally extracts
    bound values without actually invoking the lambda or any functions within it.
    ```

    在上述示例中，如果需要，`get_x()` 和 `get_y()` 的使用应该发生在 lambda 的**外部**，并分配给一个本地闭包变量：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     x_param, y_param = get_x(), get_y()
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

+   **避免在 lambda 内部引用非 SQL 结构，因为默认情况下它们是不可缓存的** - 这个问题涉及到 `LambdaElement` 如何从语句内的其他闭包变量创建缓存键。为了提供最佳的缓存键准确性保证，lambda 闭包中的所有对象都被认为是重要的，并且默认情况下不会假设它们适用于缓存键。因此，以下示例还将引发一个相当详细的错误消息：

    ```py
    >>> class Foo:
    ...     def __init__(self, x, y):
    ...         self.x = x
    ...         self.y = y
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(lambda: select(func.max(foo.x, foo.y)))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(Foo(5, 10))))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Closure variable named 'foo' inside of
    lambda callable <code object <lambda> at 0x7fed15f35450, file
    "<stdin>", line 2> does not refer to a cacheable SQL element, and also
    does not appear to be serving as a SQL literal bound value based on the
    default SQL expression returned by the function.  This variable needs to
    remain outside the scope of a SQL-generating lambda so that a proper cache
    key may be generated from the lambda's state.  Evaluate this variable
    outside of the lambda, set track_on=[<elements>] to explicitly select
    closure elements to track, or set track_closure_variables=False to exclude
    closure variables from being part of the cache key.
    ```

    上述错误表明 `LambdaElement` 不会假设传入的 `Foo` 对象在所有情况下都会保持相同的行为。它也不会假设默认情况下可以使用 `Foo` 作为缓存键；如果将 `Foo` 对象用作缓存键，如果有许多不同的 `Foo` 对象，这将用重复信息填充缓存，并且还会对所有这些对象保持长期引用。

    解决上述情况的最佳方法是不在 lambda 内部引用 `foo`，而是在**外部**引用它：

    ```py
    >>> def my_stmt(foo):
    ...     x_param, y_param = foo.x, foo.y
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

    在某些情况下，如果 lambda 的 SQL 结构保证不会根据输入更改，则传递 `track_closure_variables=False`，这将禁用除了用于绑定参数的闭包变量之外的任何追踪：

    ```py
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(func.max(foo.x, foo.y)), track_closure_variables=False
    ...     )
    ...     return stmt
    ```

    还有一种选项可以将对象添加到元素中，显式形成缓存键的一部分，使用 `track_on` 参数；使用此参数允许特定值作为缓存键，并且还将阻止考虑其他闭包变量。这对于构造的 SQL 的一部分源自某种上下文对象的情况非常有用，该对象可能具有许多不同的值。在下面的示例中，SELECT 语句的第一部分将禁用对 `foo` 变量的跟踪，而第二部分将显式跟踪 `self` 作为缓存键的一部分：

    ```py
    >>> def my_stmt(self, foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(*self.column_expressions), track_closure_variables=False
    ...     )
    ...     stmt = stmt.add_criteria(lambda: self.where_criteria, track_on=[self])
    ...     return stmt
    ```

    使用 `track_on` 意味着给定对象将长期存储在 lambda 的内部缓存中，并且只要缓存不清除这些对象（默认情况下使用 1000 个条目的 LRU 方案）就会具有强引用。

#### 缓存键生成

为了理解与 lambda SQL 构造相关的一些选项和行为，了解缓存系统是有帮助的。

SQLAlchemy 的缓存系统通常通过生成表示构造中所有状态的结构来从给定的 SQL 表达式构造中生成缓存键：

```py
>>> from sqlalchemy import select, column
>>> stmt = select(column("q"))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)  # somewhat paraphrased
CacheKey(key=(
 '0',
 <class 'sqlalchemy.sql.selectable.Select'>,
 '_raw_columns',
 (
 (
 '1',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 ),
 # a few more elements are here, and many more for a more
 # complicated SELECT statement
),)
```

上述键存储在基本上是字典的缓存中，值是一个结构，其中包括 SQL 语句的字符串形式，本例中是短语“SELECT q”。我们可以观察到，即使对于一个极短的查询，缓存键也是相当冗长的，因为它必须表示关于正在呈现和潜在执行的一切可能变化的内容。

相比之下，lambda 构造系统创建了一种不同类型的缓存键：

```py
>>> from sqlalchemy import lambda_stmt
>>> stmt = lambda_stmt(lambda: select(column("q")))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)
CacheKey(key=(
 <code object <lambda> at 0x7fed1617c710, file "<stdin>", line 1>,
 <class 'sqlalchemy.sql.lambdas.StatementLambdaElement'>,
),)
```

上面，我们看到的缓存键比非 lambda 语句的要短得多，而且甚至不需要产生`select(column("q"))`构造本身；Python lambda 本身包含一个名为`__code__`的属性，该属性引用一个 Python 代码对象，在应用程序的运行时是不可变和永久的。

当 lambda 还包括闭包变量时，在这些变量引用 SQL 构造（例如列对象）的常规情况下，它们将成为缓存键的一部分；如果它们引用将绑定参数的文字值，则它们将放置在缓存键的一个单独元素中：

```py
>>> def my_stmt(parameter):
...     col = column("q")
...     stmt = lambda_stmt(lambda: select(col))
...     stmt += lambda s: s.where(col == parameter)
...     return stmt
```

上述`StatementLambdaElement`包括两个 lambda，两者都引用`col`闭包变量，因此缓存键将表示这两个片段以及`column()`对象：

```py
>>> stmt = my_stmt(5)
>>> key = stmt._generate_cache_key()
>>> print(key)
CacheKey(key=(
 <code object <lambda> at 0x7f07323c50e0, file "<stdin>", line 3>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 <code object <lambda> at 0x7f07323c5190, file "<stdin>", line 4>,
 <class 'sqlalchemy.sql.lambdas.LinkedLambdaElement'>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
),)
```

缓存键的第二部分已检索到将在调用语句时使用的绑定参数：

```py
>>> key.bindparams
[BindParameter('%(139668884281280 parameter)s', 5, type_=Integer())]
```

有关带有性能比较的“lambda”缓存的一系列示例，请参阅性能示例中的“short_selects”测试套件。  ## “对于 INSERT 语句的插入多个值”行为

2.0 版本中的新功能：参见除 MySQL 外所有后端现已实现的优化 ORM 批量插入以了解更改的背景，包括示例性能测试

提示

insertmanyvalues 功能是一个**透明可用**的性能特性，不需要用户干预即可按需进行。本节描述了该特性的架构以及如何测量其性能并调整其行为以优化 ORM 使用的批量 INSERT 语句的速度。

随着更多的数据库增加了对 INSERT..RETURNING 的支持，SQLAlchemy 在处理需要获取服务器生成值的 INSERT 语句的方式上发生了重大变化，其中最重要的是服务器生成的主键值，它们允许在后续操作中引用新行。特别是，在 ORM 中，这种情况长期以来一直是一个重大的性能问题，因为它依赖于能够检索服务器生成的主键值，以便正确填充 标识映射。

随着 SQLite 和 MariaDB 最近对 RETURNING 的支持，SQLAlchemy 不再需要依赖于大多数后端的单行限制 [cursor.lastrowid](https://peps.python.org/pep-0249/#lastrowid) 属性；除了 MySQL 外，现在可以在所有 SQLAlchemy 包含的 后端使用 RETURNING。剩下的性能限制是，[cursor.executemany()](https://peps.python.org/pep-0249/#executemany) DBAPI 方法不允许获取行，对于大多数后端来说，这个问题已经通过放弃使用 `executemany()`，改为重构单个 INSERT 语句来容纳大量行，并在单个语句中使用 `cursor.execute()` 调用来解决。这种方法源自于 `psycopg2` DBAPI 的 [psycopg2 快速执行辅助功能](https://www.psycopg.org/docs/extras.html#fast-execution-helpers)，SQLAlchemy 在最近的发布系列中逐渐增加了对其的支持。

### 当前支持

该功能对所有支持 RETURNING 的 SQLAlchemy 后端都已启用，但对于 Oracle，除了 cx_Oracle 和 OracleDB 驱动程序提供其自己的等效功能外，其他均支持。该功能通常在使用 `Insert.returning()` 方法的 `Insert` 结构与 executemany 执行配合使用时发生，即当将字典列表传递给 `Connection.execute()` 或 `Session.execute()` 方法的 `Connection.execute.parameters` 参数时（以及 asyncio 和 `Session.scalars()` 等简写方法下的等效方法）。在使用诸如 `Session.add()` 和 `Session.add_all()` 等方法添加行时，它也在 ORM 工作单元 过程中发生。

对于 SQLAlchemy 包含的方言，支持或等效支持目前如下：

+   SQLite - 支持 SQLite 版本 3.35 及以上

+   PostgreSQL - 所有支持的 Postgresql 版本（9 及以上）

+   SQL Server - 所有支持的 SQL Server 版本 [[1]](#id2)

+   MariaDB - 支持 MariaDB 版本 10.5 及以上

+   MySQL - 不支持，没有 RETURNING 功能

+   Oracle - 支持使用本地 cx_Oracle / OracleDB API 执行多行 OUT 参数的 RETURNING，支持所有支持的 Oracle 版本 9 及以上。这不是与“executemanyvalues”相同的实现，但具有相同的使用模式和等效的性能优势。

在版本 2.0.10 中更改：

### 禁用该功能

要禁用给定后端的“insertmanyvalues”功能，以及 `create_engine()` 中的 `create_engine.use_insertmanyvalues` 参数作为 `False` 传递给 `Engine`:

```py
engine = create_engine(
    "mariadb+mariadbconnector://scott:tiger@host/db", use_insertmanyvalues=False
)
```

该功能也可以通过将 `Table.implicit_returning` 参数传递为 `False` 来禁止为特定的 `Table` 对象隐式使用：

```py
t = Table(
    "t",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x", Integer),
    implicit_returning=False,
)
```

有可能想要针对特定表禁用 RETURNING 的原因是为了解决特定后端的限制。

### 批处理模式操作

该特性有两种操作模式，根据每个方言、每个`Table`自动透明选择。一种是**批处理模式**，通过重写 INSERT 语句的形式：

```py
INSERT  INTO  a  (data,  x,  y)  VALUES  (%(data)s,  %(x)s,  %(y)s)  RETURNING  a.id
```

转换成“批处理”形式，如下所示：

```py
INSERT  INTO  a  (data,  x,  y)  VALUES
  (%(data_0)s,  %(x_0)s,  %(y_0)s),
  (%(data_1)s,  %(x_1)s,  %(y_1)s),
  (%(data_2)s,  %(x_2)s,  %(y_2)s),
  ...
  (%(data_78)s,  %(x_78)s,  %(y_78)s)
RETURNING  a.id
```

当上述语句针对输入数据的子集（“批处理”）进行组织时，其大小由数据库后端确定，并且每个批次的参数数量与已知的语句大小/参数数量相对应。该特性然后针对每个输入数据批次执行一次 INSERT 语句，直到所有记录都被消耗完毕，将每个批次的 RETURNING 结果连接成一个单个大行集，可以从单个`Result`对象中获取。

这种“批处理”形式允许使用更少的数据库往返进行许多行的 INSERT，并且已经证明在大多数支持的后端上可以实现显著的性能提升。

### 将 RETURNING 行与参数集关联起来

新版本 2.0.10 中新增。

在前一节中示例的“批处理”模式查询不保证返回的记录顺序与输入数据的顺序相对应。当由 SQLAlchemy ORM 的 unit of work 过程使用时，以及对与输入数据相关的返回的服务器生成的值进行关联的应用程序时，`Insert.returning()`和`UpdateBase.return_defaults()`方法包括一个选项`Insert.returning.sort_by_parameter_order`，指示“insertmanyvalues”模式应该保证这种对应关系。这与数据库后端实际 INSERT 的记录顺序无关，在任何情况下都不能假设；只有当收到返回的记录时应该有序排列，以与原始输入数据传递的顺序相对应。

当 `Insert.returning.sort_by_parameter_order` 参数存在时，对于使用服务器生成的整数主键值（如 `IDENTITY`、PostgreSQL `SERIAL`、MariaDB `AUTO_INCREMENT` 或 SQLite 的 `ROWID` 方案）的表，可能会选择使用更复杂的 INSERT..RETURNING 形式，并根据返回的值对行进行后执行排序，或者如果不存在这样的形式，则 “insertmanyvalues” 功能可能会优雅地降级到“非批处理”模式，为每个参数集运行单独的 INSERT 语句。

例如，在 SQL Server 中，当自动增量的 `IDENTITY` 列用作主键时，使用以下 SQL 形式：

```py
INSERT  INTO  a  (data,  x,  y)
OUTPUT  inserted.id,  inserted.id  AS  id__1
SELECT  p0,  p1,  p2  FROM  (VALUES
  (?,  ?,  ?,  0),  (?,  ?,  ?,  1),  (?,  ?,  ?,  2),
  ...
  (?,  ?,  ?,  77)
)  AS  imp_sen(p0,  p1,  p2,  sen_counter)  ORDER  BY  sen_counter
```

对于 PostgreSQL 也是类似的形式，当主键列使用 SERIAL 或 IDENTITY 时。上述形式**不**保证插入行的顺序。但它确保 IDENTITY 或 SERIAL 值将按照每个参数集的顺序创建[[2]](#id5)。然后，“insertmanyvalues” 功能通过递增整数标识对上述 INSERT 语句返回的行进行排序。

对于 SQLite 数据库，不存在适当的 INSERT 形式可以将新的 ROWID 值的生成与传递参数集的顺序进行关联。因此，在请求有序 RETURNING 时，使用服务器生成的主键值时，SQLite 后端将降级为“非批处理”模式。对于 MariaDB，默认的 INSERT 形式对 insertmanyvalues 足够，因为此数据库后端在使用 InnoDB 时会将 AUTO_INCREMENT 的顺序与输入数据的顺序对齐[[3]](#id6)。

对于客户端生成的主键，例如当使用 Python 的 `uuid.uuid4()` 函数为 `Uuid` 列生成新值时，“insertmanyvalues” 功能会透明地将此列包含在 RETURNING 记录中，并将其值与给定输入记录的值进行关联，从而保持输入记录和结果行之间的对应关系。由此可见，当使用客户端生成的主键值时，所有后端都允许批处理，参数相关的 RETURNING 顺序。

“insertmanyvalues” 的 “batch” 模式确定了一列或多列用作输入参数和返回行之间对应点的列，这被称为插入标记，它是用于跟踪这些值的特定列。通常会自动选择“插入标记”，但也可以根据极端特殊情况进行用户配置；章节配置标记列对此进行了描述。

对于不提供适当的 INSERT 形式以确定地与输入值对齐的服务器生成值的后端，或者对于具有其他类型服务器生成的主键值的`Table`配置，“insertmanyvalues”模式将在保证请求 RETURNING 排序时使用**非批处理**模式。

另请参阅

> +   Microsoft SQL Server 的原理
> +   
>     “使用 SELECT 和 ORDER BY 来填充行的 INSERT 查询保证了如何计算标识值，但不保证插入行的顺序。” [`learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql?view=sql-server-ver16#limitations-and-restrictions`](https://learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql?view=sql-server-ver16#limitations-and-restrictions)
>     
> +   PostgreSQL 批处理 INSERT 讨论
> +   
>     2018 年的原始描述 [`www.postgresql.org/message-id/29386.1528813619@sss.pgh.pa.us`](https://www.postgresql.org/message-id/29386.1528813619@sss.pgh.pa.us)
>     
>     2023 年的后续 - [`www.postgresql.org/message-id/be108555-da2a-4abc-a46b-acbe8b55bd25%40app.fastmail.com`](https://www.postgresql.org/message-id/be108555-da2a-4abc-a46b-acbe8b55bd25%40app.fastmail.com)

+   MariaDB AUTO_INCREMENT 行为（使用与 MySQL 相同的 InnoDB 引擎）：

    [`dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html`](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)

    [`dba.stackexchange.com/a/72099`](https://dba.stackexchange.com/a/72099)  ### 非批处理模式操作

对于`Table`配置，如果没有客户端主键值，并且提供服务器生成的主键值（或没有主键），而数据库无法根据多个参数集以确定性或可排序的方式调用，则“insertmanyvalues”功能在满足`Insert.returning.sort_by_parameter_order`要求的情况下，可能会选择使用**非批处理模式**。

在这种模式下，保持原始的 INSERT SQL 形式，并且“insertmanyvalues”功能将为每个参数集单独运行给定的语句，将返回的行组织成完整的结果集。与以前的 SQLAlchemy 版本不同，它会在最小化 Python 开销的紧凑循环中执行。在某些情况下，例如在 SQLite 上，“非批处理”模式的性能与“批处理”模式完全相同。

### 语句执行模型

对于“批量”和“非批量”模式，该特性必然会使用 DBAPI `cursor.execute()` 方法调用**多个 INSERT 语句**，在**单个**对核心级 `Connection.execute()` 方法的调用范围内，每个语句包含多达固定数量的参数集。此限制可按下面的描述进行配置，位于 控制批量大小。对 `cursor.execute()` 的单独调用被单独记录，并且单独传递给事件侦听器，如 `ConnectionEvents.before_cursor_execute()`（请参阅下面的 日志和事件）。

#### 配置哨兵列

在典型情况下，为了提供具有确定性行顺序的 INSERT..RETURNING，"insertmanyvalues" 特性将自动从给定表的主键中确定一个哨兵列，如果无法识别，则优雅地降级到“逐行”模式。作为一个完全**可选**的特性，为了获得对于具有服务器生成的主键的表的完整“insertmanyvalues”批量性能，其默认生成器函数与“哨兵”用例不兼容的情况，其他非主键列可以被标记为“哨兵”列，假设它们满足某些要求。一个典型的例子是具有客户端默认值的非主键 `Uuid` 列，如 Python 的 `uuid.uuid4()` 函数。还有一种构造方法可以创建具有面向“insertmanyvalues”用例的客户端整数计数器的简单整数列。

可以通过在合格的列上添加 `Column.insert_sentinel` 来指示哨兵列。最基本的“合格”列是一个非空且唯一的列，具有客户端默认值，例如 UUID 列如下所示：

```py
import uuid

from sqlalchemy import Column
from sqlalchemy import FetchedValue
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid

my_table = Table(
    "some_table",
    metadata,
    # assume some arbitrary server-side function generates
    # primary key values, so cannot be tracked by a bulk insert
    Column("id", String(50), server_default=FetchedValue(), primary_key=True),
    Column("data", String(50)),
    Column(
        "uniqueid",
        Uuid(),
        default=uuid.uuid4,
        nullable=False,
        unique=True,
        insert_sentinel=True,
    ),
)
```

在使用 ORM Declarative 模型时，可以使用 `mapped_column` 构造相同的形式：

```py
import uuid

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))
    uniqueid: Mapped[uuid.UUID] = mapped_column(
        default=uuid.uuid4, unique=True, insert_sentinel=True
    )
```

尽管默认生成器生成的值**必须**是唯一的，但上述“哨兵”列上的实际 UNIQUE 约束，由 `unique=True` 参数指示，本身是可选的，如果不需要，可以省略。

还有一种特殊的“插入标志”形式，它是一个专用的可空整数列，该列使用仅在“insertmanyvalues”操作期间使用的特殊默认整数计数器；作为额外的行为，该列将在 SQL 语句和结果集中省略自身，并以基本透明的方式行为。但是，它确实需要在实际的数据库表中物理存在。可以使用函数 `insert_sentinel()` 构建这种 `Column` 样式：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid
from sqlalchemy import insert_sentinel

Table(
    "some_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String(50)),
    insert_sentinel("sentinel"),
)
```

在使用 ORM Declarative 时，有一个友好的版本 `insert_sentinel()` 称为 `orm_insert_sentinel()` 可供使用，它可以在 Base 类或 mixin 上使用；如果使用 `declared_attr()` 封装，该列将应用于所有包括在联合继承层次结构中的表绑定子类中：

```py
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import orm_insert_sentinel

class Base(DeclarativeBase):
    @declared_attr
    def _sentinel(cls) -> Mapped[int]:
        return orm_insert_sentinel()

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))

class MySubClass(MyClass):
    __tablename__ = "sub_table"

    id: Mapped[str] = mapped_column(ForeignKey("my_table.id"), primary_key=True)

class MySingleInhClass(MyClass):
    pass
```

在上面的示例中，“my_table” 和 “sub_table” 都将有一个名为 “_sentinel” 的额外整数列，该列可供 ORM 使用的 “insertmanyvalues” 功能来帮助优化批量插入。 ### 控制批次大小

“insertmanyvalues”的一个关键特性是 INSERT 语句的大小受到固定最大数量的“values”子句以及方言特定的固定的一次性可在一个 INSERT 语句中表示的绑定参数总数的限制。当给定的参数字典数量超过固定限制时，或者当要在单个 INSERT 语句中呈现的绑定参数总数超过固定限制时（这两个固定限制是分开的），将在单个 `Connection.execute()` 调用范围内调用多个 INSERT 语句，其中每个 INSERT 语句都容纳一部分参数字典，称为“批次”。每个“批次”中表示的参数字典数量就是“批次大小”。例如，批次大小为 500 意味着每个发出的 INSERT 语句最多将插入 500 行。

能够调整批处理大小可能是很重要的，因为较大的批处理大小对于插入（INSERT）操作可能更有效率，其中值集本身相对较小，并且较小的批处理大小可能更适合使用非常大的值集的插入操作，其中渲染的 SQL 大小以及传递给一个语句的总数据大小可能受到基于后端行为和内存约束的特定大小的限制的影响。因此，批处理大小可以根据每个 `Engine` 和每个语句的基础进行配置。另一方面，参数限制是根据正在使用的数据库的已知特性固定的。

大多数后端的批处理大小默认为 1000，具有额外的每个方言的“最大参数数”限制因素，可能会进一步减小每个语句的批处理大小。参数的最大数目因方言和服务器版本而异；最大值为 32700（选择了一个距离 PostgreSQL 的限制 32767 和 SQLite 的现代限制 32766 很大的距离，同时为语句中的附加参数以及 DBAPI 的怪异性留出了空间）。旧版本的 SQLite（在 3.32.0 之前）将此值设置为 999。MariaDB 没有确定的限制，但 32700 仍然作为 SQL 消息大小的限制因素。

“批处理大小”的值可以通过 `Engine` 的 `create_engine.insertmanyvalues_page_size` 参数进行全局设置。例如，要影响包含每个语句中的最多 100 个参数集的 INSERT 语句：

```py
e = create_engine("sqlite://", insertmanyvalues_page_size=100)
```

批处理大小也可以使用 `Connection.execution_options.insertmanyvalues_page_size` 执行选项在每个语句的基础上进行影响，例如每次执行：

```py
with e.begin() as conn:
    result = conn.execute(
        table.insert().returning(table.c.id),
        parameterlist,
        execution_options={"insertmanyvalues_page_size": 100},
    )
```

或者在语句本身上进行配置：

```py
stmt = (
    table.insert()
    .returning(table.c.id)
    .execution_options(insertmanyvalues_page_size=100)
)
with e.begin() as conn:
    result = conn.execute(stmt, parameterlist)
```  ### 记录和事件

“insertmanyvalues” 功能与 SQLAlchemy 的语句记录以及游标事件完全集成，例如 `ConnectionEvents.before_cursor_execute()`。当参数列表被拆分成单独的批次时，**每个 INSERT 语句都将被单独记录并传递给事件处理程序**。这与之前 SQLAlchemy 1.x 系列中仅使用 psycopg2 的功能的工作方式相比是一个重大变化，之前的工作方式中，多个 INSERT 语句的生成对于记录和事件是隐藏的。日志显示会截断长参数列表以便阅读，并且还会指示每个语句的特定批次。下面的示例说明了此日志的摘录：

```py
INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[generated in 0.00177s (insertmanyvalues) 1/10 (unordered)] ('d0', 0, 0, 'd1',  ...
INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[insertmanyvalues 2/10 (unordered)] ('d100', 100, 1000, 'd101', ...

...

INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[insertmanyvalues 10/10 (unordered)] ('d900', 900, 9000, 'd901', ...
```

当 非批处理模式 发生时，日志记录将会指示这一点，并显示 insertmanyvalues 消息：

```py
...

INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 67/78 (ordered; batch not supported)] ('d66', 66, 66)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 68/78 (ordered; batch not supported)] ('d67', 67, 67)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 69/78 (ordered; batch not supported)] ('d68', 68, 68)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 70/78 (ordered; batch not supported)] ('d69', 69, 69)

...
```

另请参阅

配置日志

### Upsert 支持

PostgreSQL、SQLite 和 MariaDB 方言提供了特定于后端的“upsert”构造 `insert()`、`insert()` 和 `insert()`，它们分别是 `Insert` 构造，具有诸如 `on_conflict_do_update()` 或 ``on_duplicate_key()` 的附加方法。当这些构造与 RETURNING 一起使用时，它们还支持“insertmanyvalues”行为，允许高效地进行带 RETURNING 的 upsert 操作。  ## 引擎处理

`Engine` 指的是一个连接池，这意味着在正常情况下，当 `Engine` 对象仍然驻留在内存中时，存在着打开的数据库连接。当一个 `Engine` 被垃圾回收时，它的连接池将不再被该 `Engine` 引用，并且假设没有连接仍然被检出，那么连接池及其连接也将被垃圾回收，这将关闭实际的数据库连接。但是除此之外，`Engine` 将保持打开的数据库连接，假设它使用的是通常的默认连接池实现 `QueuePool`。

`Engine` 通常应该是一个事先建立并在应用程序的整个生命周期中维护的永久性构造。它**不**应该按照每个连接的方式创建和处理；相反，它是一个注册表，既维护着一组连接池，又维护着关于正在使用的数据库和 DBAPI 的配置信息，以及某种程度上的针对每个数据库资源的内部缓存。

然而，有许多情况下希望所有由`Engine`引用的连接资源都被完全关闭。通常不建议依赖 Python 垃圾回收来处理这些情况；相反，可以使用`Engine.dispose()`方法显式地释放`Engine`。这会处理引擎的底层连接池，并用一个空的新连接池替换它。只要此时丢弃了`Engine`并且不再使用它，它引用的所有**检入**连接也将被完全关闭。

调用`Engine.dispose()`的有效用例包括：

+   当程序想要释放连接池中的所有剩余已检入连接，并且不再期望对该数据库进行任何未来操作时。

+   当程序使用多进程或`fork()`，并且将`Engine`对象复制到子进程时，应调用`Engine.dispose()`，以便引擎在该 fork 中创建全新的数据库连接。数据库连接通常不会跨进程边界传输。在这种情况下，使用`Engine.dispose.close`参数设置为 False。有关此用例的更多背景信息，请参见使用连接池进行多进程或 os.fork()部分。

+   在测试套件或多租户场景中，可能会创建和处理许多临时的短寿命`Engine`对象。

当引擎被处理或被垃圾回收时，**已签出**的连接不会被丢弃，因为这些连接仍然在应用程序的其他地方被强引用。但是，在调用`Engine.dispose()`之后，这些连接将不再与该`Engine`相关联；当它们关闭时，它们将返回到它们现在孤立的连接池中，该连接池最终会在所有引用它的连接都不再在任何地方被引用时被垃圾回收。由于这个过程不容易控制，强烈建议仅在所有签出的连接都已签入或以其他方式与它们的池解除关联后才调用`Engine.dispose()`。

对于受到 `Engine` 对象的连接池使用影响的应用程序，另一种选择是完全禁用连接池。这通常只会对使用新连接产生轻微的性能影响，并且意味着当连接被检入时，它会完全关闭并且不会在内存中保留。请参阅 切换连接池实现 以获取有关如何禁用连接池的指南。

另请参阅

连接池

在多进程或 os.fork() 中使用连接池  ## 使用驱动程序 SQL 和原始 DBAPI 连接

在介绍使用 `Connection.execute()` 时，使用了 `text()` 构造来说明如何调用文本 SQL 语句。在使用 SQLAlchemy 时，文本 SQL 实际上更多地是例外而不是规范，因为核心表达式语言和 ORM 都将 SQL 的文本表示抽象化了。但是，`text()` 构造本身也提供了对文本 SQL 的一些抽象，它规范了如何传递绑定参数，以及支持参数和结果集行的数据类型行为。

### 直接调用驱动程序的 SQL 字符串

对于希望直接传递给底层驱动程序（称为 DBAPI）的文本 SQL 而不经过 `text()` 构造的用例，可以使用 `Connection.exec_driver_sql()` 方法：

```py
with engine.connect() as conn:
    conn.exec_driver_sql("SET param='bar'")
```

从版本 1.4 开始：新增了 `Connection.exec_driver_sql()` 方法。

### 直接使用 DBAPI 游标

有些情况下，SQLAlchemy 并没有提供一种通用化的方法来访问一些 DBAPI 函数，例如调用存储过程以及处理多个结果集。在这些情况下，直接处理原始的 DBAPI 连接同样是一种方便的方式。

访问原始 DBAPI 连接的最常见方法是直接从已有的 `Connection` 对象中获取。这可以通过 `Connection.connection` 属性进行访问：

```py
connection = engine.connect()
dbapi_conn = connection.connection
```

这里的 DBAPI 连接实际上是一个“代理”，就连接池的原始连接而言，然而这是一个大多数情况下可以忽略的实现细节。由于这个 DBAPI 连接仍然包含在一个拥有的`Connection`对象的范围内，最好使用`Connection`对象来进行大多数功能，如事务控制以及调用`Connection.close()`方法；如果这些操作直接在 DBAPI 连接上执行，拥有的`Connection`将不会意识到这些状态的变化。

为了克服由拥有的`Connection`维护的 DBAPI 连接所施加的限制，还可以使用 `Engine.raw_connection()` 方法获取一个 DBAPI 连接，而不需要先获取一个 `Connection`：

```py
dbapi_conn = engine.raw_connection()
```

这个 DBAPI 连接再次是一个“代理”形式，就像以前的情况一样。这种代理的目的现在显而易见，当我们调用此连接的 `.close()` 方法时，DBAPI 连接通常实际上不会关闭，而是被释放回引擎的连接池：

```py
dbapi_conn.close()
```

虽然 SQLAlchemy 可能会在未来添加更多用于更多 DBAPI 使用案例的内置模式，但由于这些情况往往很少需要，并且它们也高度依赖于正在使用的 DBAPI 的类型，所以在任何情况下，直接使用 DBAPI 调用模式始终存在于那些需要的情况下。

另见

在使用 Engine 时，如何访问原始 DBAPI 连接？ - 包括有关如何访问 DBAPI 连接以及在使用 asyncio 驱动程序时“驱动程序”连接的其他详细信息。

一些用于 DBAPI 连接的方法如下。### 调用存储过程和用户定义的函数

SQLAlchemy 支持以几种方式调用存储过程和用户定义的函数。请注意，所有的 DBAPI 都有不同的做法，因此您必须咨询底层 DBAPI 的文档以获取与您特定用途相关的具体信息。以下示例是假设性的，并且可能不适用于您的底层 DBAPI。

对于具有特殊语法或参数问题的存储过程或函数，可以使用 DBAPI 级别的 [callproc](https://legacy.python.org/dev/peps/pep-0249/#callproc)，您的 DBAPI 可能可以使用。这种模式的一个例子是：

```py
connection = engine.raw_connection()
try:
    cursor_obj = connection.cursor()
    cursor_obj.callproc("my_procedure", ["x", "y", "z"])
    results = list(cursor_obj.fetchall())
    cursor_obj.close()
    connection.commit()
finally:
    connection.close()
```

注意

并非所有的 DBAPI 都使用 callproc，总体使用细节会有所不同。上面的例子只是说明如何使用特定的 DBAPI 函数。

您的 DBAPI 可能没有`callproc`要求，*或*可能要求使用另一种模式调用存储过程或用户定义的函数，例如正常的 SQLAlchemy 连接使用。这种用法模式的一个例子是，在撰写本文档时，使用 psycopg2 DBAPI 在 PostgreSQL 数据库中执行存储过程，应该使用正常的连接使用方式调用：

```py
connection.execute("CALL my_procedure();")
```

上面的例子是假设性的。底层数据库不保证在这些情况下支持“CALL”或“SELECT”，关键字可能会根据函数是存储过程还是用户定义的函数而变化。在这些情况下，您应该查阅底层的 DBAPI 和数据库文档，以确定使用的正确语法和模式。

### 多结果集

多结果集支持可以通过原始 DBAPI 游标使用[nextset](https://legacy.python.org/dev/peps/pep-0249/#nextset)方法获得：

```py
connection = engine.raw_connection()
try:
    cursor_obj = connection.cursor()
    cursor_obj.execute("select * from table1; select * from table2")
    results_one = cursor_obj.fetchall()
    cursor_obj.nextset()
    results_two = cursor_obj.fetchall()
    cursor_obj.close()
finally:
    connection.close()
```

## 注册新方言

`create_engine()`函数调用通过 setuptools 入口点定位给定的方言。这些入口点可以在 setup.py 脚本中为第三方方言建立。例如，要创建一个名为“foodialect://”的新方言，步骤如下：

1.  创建一个名为`foodialect`的包。

1.  该包应该有一个包含方言类的模块，该类通常是`sqlalchemy.engine.default.DefaultDialect`的子类。在这个例子中，假设它被称为`FooDialect`，并且通过`foodialect.dialect`访问其模块。

1.  入口点可以在`setup.cfg`中建立如下：

    ```py
    [options.entry_points]
    sqlalchemy.dialects  =
      foodialect  =  foodialect.dialect:FooDialect
    ```

如果方言为现有的 SQLAlchemy 支持的数据库提供了对特定 DBAPI 的支持，则可以给出名称，包括数据库限定符。例如，如果`FooDialect`实际上是一个 MySQL 方言，可以像这样建立入口点：

```py
[options.entry_points]
sqlalchemy.dialects
  mysql.foodialect  =  foodialect.dialect:FooDialect
```

上述入口点将被访问为`create_engine("mysql+foodialect://")`。

### 在进程内注册方言

SQLAlchemy 还允许在当前进程中注册一个方言，绕过需要单独安装的必要性。使用`register()`函数如下：

```py
from sqlalchemy.dialects import registry

registry.register("mysql.foodialect", "myapp.dialect", "MyMySQLDialect")
```

上述将响应`create_engine("mysql+foodialect://")`并从`myapp.dialect`模块加载`MyMySQLDialect`类。

## 连接/引擎 API

| 对象名称 | 描述 |
| --- | --- |
| 连接 | 为包装的 DB-API 连接提供高级功能。 |
| CreateEnginePlugin | 一组旨在根据 URL 中的入口点名称增强 `Engine` 对象构造的钩子。 |
| Engine | 将 `Pool` 和 `Dialect` 连接在一起，提供数据库连接和行为的来源。 |
| ExceptionContext | 封装有关正在进行的错误条件的信息。 |
| NestedTransaction | 表示“嵌套”或 SAVEPOINT 事务。 |
| RootTransaction | 表示 `Connection` 上的“根”事务。 |
| Transaction | 表示正在进行的数据库事务。 |
| TwoPhaseTransaction | 表示两阶段事务。 |

```py
class sqlalchemy.engine.Connection
```

为封装的 DB-API 连接提供高级功能。

通过调用 `Engine.connect()` 方法获取 `Engine` 对象的 `Connection` 对象，并提供执行 SQL 语句以及事务控制的服务。

`Connection` 对象**不是**线程安全的。虽然一个 Connection 可以通过正确同步的访问在线程之间共享，但底层的 DBAPI 连接可能不支持线程之间的共享访问。查看 DBAPI 文档以获取详细信息。

**成员**

__init__(), begin(), begin_nested(), begin_twophase(), close(), closed, commit(), connection, default_isolation_level, detach(), exec_driver_sql(), execute(), execution_options(), get_execution_options(), get_isolation_level(), get_nested_transaction(), get_transaction(), in_nested_transaction(), in_transaction(), info, invalidate(), invalidated, rollback(), scalar(), scalars(), schema_for_object()

连接对象表示从连接池中检出的单个 DBAPI 连接。在此状态下，连接池不会影响连接，包括其到期或超时状态。为了让连接池正确管理连接，连接应在未被使用时返回到连接池（即 `connection.close()`）。

**类签名**

类 `sqlalchemy.engine.Connection` (`sqlalchemy.engine.interfaces.ConnectionEventsTarget`, `sqlalchemy.inspection.Inspectable`)

```py
method __init__(engine: Engine, connection: PoolProxiedConnection | None = None, _has_events: bool | None = None, _allow_revalidate: bool = True, _allow_autobegin: bool = True)
```

构建一个新的连接。

```py
method begin() → RootTransaction
```

开始一个事务，以便在自动开始之前进行。

例如：

```py
with engine.connect() as conn:
    with conn.begin() as trans:
        conn.execute(table.insert(), {"username": "sandy"})
```

返回的对象是 `RootTransaction` 的实例。该对象表示事务的“范围”，当 `Transaction.rollback()` 或 `Transaction.commit()` 方法被调用时完成事务；该对象还可作为上述示例中所示的上下文管理器。

`Connection.begin()` 方法开始一个事务，通常在连接首次用于执行语句时始终会开始。可能使用此方法的原因是在特定时间调用 `ConnectionEvents.begin()` 事件，或者在连接检出的范围内组织代码以利用上下文管理的块，例如：

```py
with engine.connect() as conn:
    with conn.begin():
        conn.execute(...)
        conn.execute(...)

    with conn.begin():
        conn.execute(...)
        conn.execute(...)
```

上述代码在行为上与不使用 `Connection.begin()` 的以下代码基本没有区别；下面的样式称为“随时提交”样式：

```py
with engine.connect() as conn:
    conn.execute(...)
    conn.execute(...)
    conn.commit()

    conn.execute(...)
    conn.execute(...)
    conn.commit()
```

从数据库的角度来看，`Connection.begin()` 方法不会发出任何 SQL 或以任何方式更改底层 DBAPI 连接的状态；Python DBAPI 没有任何显式事务开始的概念。

另请参阅

处理事务和 DBAPI - 在 SQLAlchemy 统一教程中

`Connection.begin_nested()` - 使用保存点

`Connection.begin_twophase()` - 使用两阶段 / XID 事务

`Engine.begin()` - 可从 `Engine` 使用的上下文管理器

```py
method begin_nested() → NestedTransaction
```

开始一个嵌套事务（即保存点），并返回一个控制保存点范围的事务句柄。

例如：

```py
with engine.begin() as connection:
    with connection.begin_nested():
        connection.execute(table.insert(), {"username": "sandy"})
```

返回的对象是`NestedTransaction`的实例，其中包括事务方法`NestedTransaction.commit()`和`NestedTransaction.rollback()`；对于嵌套事务，这些方法对应于操作“RELEASE SAVEPOINT <name>”和“ROLLBACK TO SAVEPOINT <name>”。保存点的名称是局限于`NestedTransaction`对象的，并且会自动生成。与任何其他`Transaction`一样，`NestedTransaction`可以用作上面示例中所说明的上下文管理器，该上下文管理器将“释放”或“回滚”相应于块内操作是否成功或引发异常。

嵌套事务要求底层数据库支持 SAVEPOINT，否则行为未定义。SAVEPOINT 通常用于在事务内运行可能失败的操作，同时继续外部事务。例如：

```py
from sqlalchemy import exc

with engine.begin() as connection:
    trans = connection.begin_nested()
    try:
        connection.execute(table.insert(), {"username": "sandy"})
        trans.commit()
    except exc.IntegrityError:  # catch for duplicate username
        trans.rollback()  # rollback to savepoint

    # outer transaction continues
    connection.execute( ... )
```

如果在调用 `Connection.begin_nested()` 之前没有先调用 `Connection.begin()` 或 `Engine.begin()`，则 `Connection` 对象将“自动开始”外部事务。这个外部事务可以使用“随时提交”样式提交，例如：

```py
with engine.connect() as connection:  # begin() wasn't called

    with connection.begin_nested(): will auto-"begin()" first
        connection.execute( ... )
    # savepoint is released

    connection.execute( ... )

    # explicitly commit outer transaction
    connection.commit()

    # can continue working with connection here
```

2.0 版本更改：`Connection.begin_nested()` 现在将参与连接的“自动开始”行为，这是自 2.0 版本 / 1.4 版本“未来”风格连接的新功能。

另请参阅

`Connection.begin()`

使用 SAVEPOINT - 保存点的 ORM 支持

```py
method begin_twophase(xid: Any | None = None) → TwoPhaseTransaction
```

开始一个两阶段或 XA 事务并返回一个事务句柄。

返回的对象是`TwoPhaseTransaction`的实例，除了由`Transaction`提供的方法之外，还提供了一个`TwoPhaseTransaction.prepare()`方法。

参数：

**xid** – 两阶段事务 id。如果未提供，将生成一个随机 id。

另请参阅

`Connection.begin()`

`Connection.begin_twophase()`

```py
method close() → None
```

关闭此`Connection`。

这会导致底层数据库资源的释放，即内部引用的 DBAPI 连接。DBAPI 连接通常会恢复到产生此`Connection`的`Engine`所引用的连接持有`Pool`。无论是否存在任何与此`Connection`相关的`Transaction`对象，DBAPI 连接上的任何事务状态都将通过 DBAPI 连接的`rollback()`方法无条件释放。

如果存在任何事务，则此方法还会调用`Connection.rollback()`。

在调用`Connection.close()`之后，`Connection`将永久处于关闭状态，不允许进行任何进一步的操作。

```py
attribute closed
```

如果此连接已关闭，则返回 True。

```py
method commit() → None
```

提交当前正在进行的事务。

如果已经开始了当前事务，则此方法提交当前事务。如果没有启动事务，则此方法不起作用，假设连接处于非无效状态。

每当首次执行语句或调用`Connection.begin()`方法时，`Connection`会自动开始事务。

注意

`Connection.commit()` 方法仅对链接到 `Connection` 对象的主数据库事务起作用。它不会操作从 `Connection.begin_nested()` 方法调用的 SAVEPOINT；要控制 SAVEPOINT，请对 `Connection.begin_nested()` 方法本身返回的 `NestedTransaction` 调用 `NestedTransaction.commit()`。

```py
attribute connection
```

此连接管理的底层 DB-API 连接。

这是一个 SQLAlchemy 连接池代理连接，然后具有属性 `_ConnectionFairy.dbapi_connection`，该属性引用实际的驱动程序连接。

另请参阅

使用 Driver SQL 和原始 DBAPI 连接

```py
attribute default_isolation_level
```

与正在使用的 `Dialect` 关联的初始连接时间隔离级别。

此值独立于 `Connection.execution_options.isolation_level` 和 `Engine.execution_options.isolation_level` 执行选项，并且由 `Dialect` 在创建第一个连接时确定，通过针对数据库执行当前隔离级别的 SQL 查询，在发出任何其他命令之前。

调用此访问器不会触发任何新的 SQL 查询。

另请参阅

`Connection.get_isolation_level()` - 查看当前实际隔离级别

`create_engine.isolation_level` - 设置每个 `Engine` 的隔离级别

`Connection.execution_options.isolation_level` - 设置每个 `Connection` 的隔离级别

```py
method detach() → None
```

从其连接池中分离底层 DB-API 连接。

例如：

```py
with engine.connect() as conn:
    conn.detach()
    conn.execute(text("SET search_path TO schema1, schema2"))

    # work with connection

# connection is fully closed (since we used "with:", can
# also call .close())
```

此`Connection`实例将保持可用。当关闭（或从上下文管理器上下文中退出）时，DB-API 连接将被真正关闭，不会返回到其原始池中。

此方法可用于隔离连接上的应用程序的其余部分的修改状态（例如事务隔离级别或类似内容）。

```py
method exec_driver_sql(statement: str, parameters: _DBAPIAnyExecuteParams | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

直接在 DBAPI 游标上执行字符串 SQL 语句，无需任何 SQL 编译步骤。

这可以用于直接将任何字符串传递给正在使用的 DBAPI 的`cursor.execute()`方法。

参数：

+   `statement` – 要执行的语句字符串。绑定参数必须使用底层 DBAPI 的 paramstyle，例如“qmark”，“pyformat”，“format”等。

+   `parameters` – 表示要在执行中使用的绑定参数值。格式之一：具有命名参数的字典，具有位置参数的元组，或包含用于多次执行支持的字典或元组的列表。

返回值：

`CursorResult`。

例如，多个字典：

```py
conn.exec_driver_sql(
    "INSERT INTO table (id, value) VALUES (%(id)s, %(value)s)",
    [{"id":1, "value":"v1"}, {"id":2, "value":"v2"}]
)
```

单个字典：

```py
conn.exec_driver_sql(
    "INSERT INTO table (id, value) VALUES (%(id)s, %(value)s)",
    dict(id=1, value="v1")
)
```

单个元组：

```py
conn.exec_driver_sql(
    "INSERT INTO table (id, value) VALUES (?, ?)",
    (1, 'v1')
)
```

注意

`Connection.exec_driver_sql()`方法不参与`ConnectionEvents.before_execute()`和`ConnectionEvents.after_execute()`事件。要拦截对`Connection.exec_driver_sql()`的调用，请使用`ConnectionEvents.before_cursor_execute()`和`ConnectionEvents.after_cursor_execute()`。

另请参阅

[**PEP 249**](https://peps.python.org/pep-0249/)

```py
method execute(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

执行 SQL 语句构造并返回`CursorResult`。

参数：

+   `statement` –

    要执行的语句。这始终是`ClauseElement`和`Executable`层次结构中的对象，包括：

    +   `Select`

    +   `Insert`、`Update`、`Delete`

    +   `TextClause` 和 `TextualSelect`

    +   `DDL` 和从 `ExecutableDDLElement` 继承的对象

+   `parameters` – 将绑定到语句中的参数。这可以是一个参数名到值的字典，或一个可变序列（例如列表）的字典。当传递一个字典列表时，底层语句执行将使用 DBAPI 的 `cursor.executemany()` 方法。当传递一个单一字典时，将使用 DBAPI 的 `cursor.execute()` 方法。

+   `execution_options` – 可选的执行选项字典，它将与语句执行相关联。该字典可以提供一组接受 `Connection.execution_options()` 的选项的子集。

返回：

一个 `Result` 对象。

```py
method execution_options(**opt: Any) → Connection
```

设置在执行期间生效的连接非 SQL 选项。

此方法在原地修改了这个 `Connection`；返回值是调用该方法的相同 `Connection` 对象。请注意，这与其他对象（如 `Engine.execution_options()` 和 `Executable.execution_options()`）上的 `execution_options` 方法的行为相反。其原理是，许多此类执行选项在任何情况下都会修改基本 DBAPI 连接的状态，因此没有可行的方法将此类选项的效果局限于“子”连接。

2.0 版本中的变化：`Connection.execution_options()` 方法与具有此方法的其他对象不同，它会在原地修改连接而不创建副本。

如其他地方所述，`Connection.execution_options()` 方法接受任意的参数，包括用户定义的名称。所有给定的参数都可以以多种方式被使用，包括使用 `Connection.get_execution_options()` 方法。请参阅 `Executable.execution_options()` 和 `Engine.execution_options()` 中的示例。

SQLAlchemy 自身当前识别的关键字包括所有 `Executable.execution_options()` 下列出的关键字，以及特定于 `Connection` 的其他关键字。

参数：

+   `compiled_cache` –

    可用于：`Connection`，`Engine`。

    一个字典，当 `Connection` 将子句表达式编译为 `Compiled` 对象时，`Compiled` 对象将被缓存。这个字典将覆盖可能在 `Engine` 上配置的语句缓存。如果设置为 None，则禁用缓存，即使引擎配置了缓存大小。

    请注意，ORM 在某些操作中使用了自己的“已编译”缓存，包括 flush 操作。ORM 内部使用的缓存会覆盖此处指定的缓存字典。

+   `logging_token` –

    可用于：`Connection`，`Engine`，`Executable`。

    在连接记录的日志消息中添加由括号括起的指定字符串令牌，即启用了 `create_engine.echo` 标志或通过 `logging.getLogger("sqlalchemy.engine")` 记录器启用的日志记录。这允许可用于调试并发连接场景的每个连接或每个子引擎令牌。

    版本 1.4.0b2 中的新功能。

    另请参阅

    设置每个连接/子引擎令牌 - 使用示例

    `create_engine.logging_name` - 为 Python 日志记录器对象本身添加一个名称。

+   `isolation_level` –

    可用于：`Connection`, `Engine`.

    为此`Connection`对象的生命周期设置事务隔离级别。有效值包括`create_engine()`传递给`create_engine.isolation_level`参数接受的那些字符串值。这些级别是半数据库特定的；请参阅各个方言文档以获取有效级别。

    隔离级别选项通过在 DBAPI 连接上发出语句来应用隔离级别，并**必然会影响原始 Connection 对象的整体**。隔离级别将保持在给定设置，直到明确更改，或者当 DBAPI 连接本身被释放到连接池时，即调用`Connection.close()`方法时，此时事件处理程序将在 DBAPI 连接上发出附加语句，以恢复隔离级别更改。

    注意

    `isolation_level`执行选项只能在调用`Connection.begin()`方法之前建立，以及在发出任何否则会触发“自动开始”的 SQL 语句之前，或者在调用`Connection.commit()`或`Connection.rollback()`之后直接调用。数据库无法更改进行中的事务的隔离级别。

    注意

    如果通过`Connection.invalidate()`方法使`Connection`无效，或者发生断开连接错误，则`isolation_level`执行选项会被隐式重置。在无效后产生的新连接将**不会**自动重新应用所选的隔离级别。

    另请参阅

    设置事务隔离级别，包括 DBAPI 自动提交

    `Connection.get_isolation_level()` - 查看当前实际级别

+   `no_parameters` –

    可用于：`Connection`，`Executable`。

    当为 `True` 时，如果最终的参数列表或字典完全为空，则会像 `cursor.execute(statement)` 那样在游标上调用语句，完全不传递参数集合。一些 DBAPI，如 psycopg2 和 mysql-python，只有在存在参数时才将百分号视为重要；这个选项允许代码生成包含百分号（可能还有其他字符）的 SQL，不管它是由 DBAPI 执行还是被管道传输到稍后由命令行工具调用的脚本中。

+   `stream_results` –

    可用于：`Connection`，`Executable`。

    如果可能的话，向方言指示结果应该是“流式”的，而不是预先缓冲的。对于诸如 PostgreSQL、MySQL 和 MariaDB 等后端，这表示使用“服务器端游标”而不是客户端游标。其他后端，如 Oracle 的后端，可能已经默认使用服务器端游标。

    通常，使用 `Connection.execution_options.stream_results` 与设置要以批次获取的固定行数结合使用，以便在同时不一次加载所有结果行到内存中的情况下有效迭代数据库行；可以在执行返回一个新的 `Result` 对象后，通过 `Result.yield_per()` 方法在 `Result` 对象上进行配置。如果未使用 `Result.yield_per()`，则 `Connection.execution_options.stream_results` 操作模式将使用一个动态大小的缓冲区，它会一次缓冲一组行，根据固定的增长大小在每个批次上增长，直到通过使用 `Connection.execution_options.max_row_buffer` 参数进行配置的限制为止。

    当使用 ORM 从结果中获取 ORM 映射对象时，应始终使用 `Result.yield_per()` 与 `Connection.execution_options.stream_results` 一起使用，以便 ORM 不会一次将所有行都提取到新的 ORM 对象中。

    对于典型用法，应优先考虑 `Connection.execution_options.yield_per` 执行选项，该选项一次设置了 `Connection.execution_options.stream_results` 和 `Result.yield_per()`。此选项在 `Connection` 以及 ORM `Session` 的核心级别都受支持；后者在 使用 Yield Per 获取大结果集 中进行了描述。

    另请参阅

    使用服务器端游标（也称为流式结果） - 关于 `Connection.execution_options.stream_results` 的背景信息

    `Connection.execution_options.max_row_buffer`

    `Connection.execution_options.yield_per`

    使用 Yield Per 获取大结果集 - 在 ORM 查询指南 中描述了 `yield_per` 的 ORM 版本

+   `max_row_buffer` –

    可用于：`Connection`，`Executable`。当在支持服务器端游标的后端使用 `Connection.execution_options.stream_results` 执行选项时，设置要使用的最大缓冲区大小。如果未指定默认值，则默认值为 1000。

    另请参阅

    `Connection.execution_options.stream_results`

    使用服务器端游标（也称为流式结果）

+   `yield_per` –

    可用于：`Connection`、`Executable`。设置 `Connection.execution_options.stream_results` 执行选项的整数值，并立即自动调用 `Result.yield_per()`。允许等效的功能，与使用此参数时与 ORM 存在的功能相同。

    版本 1.4.40 中的新增内容。

    另请参见

    使用服务器端游标（又名流式结果） - 关于在核心中使用服务器端游标的背景和示例。

    使用 `yield_per` 逐步获取大型结果集 - 在 ORM 查询指南 中描述了 ORM 版本的 `yield_per`。

+   `insertmanyvalues_page_size` –

    可用于：`Connection`、`Engine`。将要格式化为 INSERT 语句的行数，当语句使用“insertmanyvalues”模式时，该模式是一种分页形式的批量插入，通常与 executemany 执行结合使用，用于许多后端，通常与 RETURNING 一起使用。默认为 1000。还可以使用 `create_engine.insertmanyvalues_page_size` 参数在每个引擎的基础上进行修改。

    版本 2.0 中的新增内容。

    另请参见

    插入语句的“插入多个值”行为

+   `schema_translate_map` –

    可用于：`Connection`、`Engine`、`Executable`。

    将模式名称映射到模式名称的字典，将应用于编译 SQL 或 DDL 表达式元素为字符串时遇到的每个 `Table` 的 `Table.schema` 元素；结果模式名称将根据原始名称在映射中的存在进行转换。

    另请参见

    模式名称的翻译

+   `preserve_rowcount` –

    布尔值；当为 True 时，`cursor.rowcount` 属性将无条件地被记忆在结果中，并通过 `CursorResult.rowcount` 属性提供。通常，此属性仅对 UPDATE 和 DELETE 语句保留。使用此选项，可以访问 DBAPI 的 rowcount 值，以用于 INSERT 和 SELECT 等其他类型的语句，只要 DBAPI 支持这些语句。有关此属性行为的说明，请参阅 `CursorResult.rowcount`。

    版本 2.0.28 中的新内容。

另请参阅

`Engine.execution_options()`

`Executable.execution_options()`

`Connection.get_execution_options()`

ORM 执行选项 - 所有 ORM 特定执行选项的文档

```py
method get_execution_options() → _ExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

版本 1.3 中的新内容。

另请参阅

`Connection.execution_options()`

```py
method get_isolation_level() → Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']
```

返回此连接范围内存在于数据库中的当前**实际**隔离级别。

此属性将执行与数据库的实时 SQL 操作，以获取当前隔离级别，因此返回的值是底层 DBAPI 连接上的实际级别，而不管此状态如何设置。这将是四个实际隔离模式之一 `READ UNCOMMITTED`、`READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE`。它**不**包括 `AUTOCOMMIT` 隔离级别设置。第三方方言也可能具有额外的隔离级别设置。

注

此方法**不会报告** AUTOCOMMIT 隔离级别，该级别是独立于**实际**隔离级别的 dbapi 设置。当使用 AUTOCOMMIT 时，数据库连接仍然具有正在使用的“传统”隔离模式，通常是四个值之一 `READ UNCOMMITTED`、`READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE`。

与返回初始连接时数据库上存在的隔离级别的 `Connection.default_isolation_level` 访问器进行比较。

另请参阅

`Connection.default_isolation_level` - 查看默认级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

`Connection.execution_options.isolation_level` - 设置每个`Connection`的隔离级别

```py
method get_nested_transaction() → NestedTransaction | None
```

返回当前正在进行的嵌套事务（如果有）。

版本 1.4 中的新功能。

```py
method get_transaction() → RootTransaction | None
```

返回当前正在进行的根事务（如果有）。

版本 1.4 中的新功能。

```py
method in_nested_transaction() → bool
```

如果事务正在进行中，则返回 True。

```py
method in_transaction() → bool
```

如果事务正在进行中，则返回 True。

```py
attribute info
```

与由此`Connection`引用的底层 DBAPI 连接相关联的信息字典，允许将用户定义的数据与连接关联起来。

这里的数据将随着 DBAPI 连接一起，包括在将其返回到连接池并在后续的`Connection`实例中再次使用时。

```py
method invalidate(exception: BaseException | None = None) → None
```

使与这个`Connection`关联的底层 DBAPI 连接无效。

将立即尝试关闭底层的 DBAPI 连接；但是，如果此操作失败，则会记录错误但不会引发错误。无论 close()是否成功，连接都将被丢弃。

在下一次使用时（“使用”通常意味着使用`Connection.execute()`方法或类似方法），这个`Connection`将尝试使用`Pool`的服务来获取一个新的 DBAPI 连接作为连接源（例如“重新连接”）。

如果在调用`Connection.invalidate()`方法时正在进行事务（例如已调用`Connection.begin()`方法），则在 DBAPI 级别上，与此事务关联的所有状态都会丢失，因为 DBAPI 连接已关闭。在调用`Transaction.rollback()`方法结束`Transaction`对象之前，`Connection`不会允许重新连接；在那之前，任何继续使用`Connection`的尝试都将引发`InvalidRequestError`。这是为了防止应用程序在事务由于失效而丢失的情况下意外继续进行事务操作。

`Connection.invalidate()`方法，就像自动失效一样，将在连接池级别调用`PoolEvents.invalidate()`事件。

参数：

**exception** – 可选的`Exception`实例，表示失效的原因，将传递给事件处理程序和日志记录函数。

另请参阅

更多关于失效的信息

```py
attribute invalidated
```

如果此连接已失效，则返回 True。

这并不表示连接是否在池级别被失效。

```py
method rollback() → None
```

回滚当前正在进行的事务。

如果已经启动了事务，则此方法会回滚当前事务。如果没有启动事务，则此方法不起作用。如果已启动事务且连接处于失效状态，则使用此方法清除事务。

每当首次执行语句或调用`Connection.begin()`方法时，将自动在`Connection`上启动事务。

注意

`Connection.rollback()`方法仅作用于与`Connection`对象关联的主数据库事务。它不会操作从`Connection.begin_nested()`方法调用的 SAVEPOINT；要控制 SAVEPOINT，请在`Connection.begin_nested()`方法本身返回的`NestedTransaction`上调用`NestedTransaction.rollback()`。

```py
method scalar(statement: Executable, parameters: _CoreSingleExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → Any
```

执行 SQL 语句构造并返回标量对象。

此方法是在调用`Connection.execute()`方法后调用`Result.scalar()`方法的简写。参数是相等的。

返回：

表示返回的第一行的第一列的标量 Python 值。

```py
method scalars(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → ScalarResult[Any]
```

执行并返回标量结果集，该结果集从每行的第一列中产生标量值。

此方法相当于调用`Connection.execute()`以接收`Result`对象，然后调用`Result.scalars()`方法生成`ScalarResult`实例。

返回：

一个`ScalarResult`

在版本 1.4.24 中新增。

```py
method schema_for_object(obj: HasSchemaAttr) → str | None
```

返回给定架构项的架构名称，考虑到当前架构翻译映射。

```py
class sqlalchemy.engine.CreateEnginePlugin
```

一组旨在根据 URL 中的入口点名称增强`Engine`对象构建的钩子集。

`CreateEnginePlugin`的目的是允许第三方系统应用引擎、池和方言级事件侦听器，而无需修改目标应用程序；相反，插件名称可以添加到数据库 URL。`CreateEnginePlugin`的目标应用程序包括：

+   连接和 SQL 性能工具，例如使用事件来跟踪检查次数和/或语句所花费的时间

+   连接插件和 SQL 性能工具，例如代理

一个简陋的将日志记录器附加到`Engine`对象的`CreateEnginePlugin`可能如下所示：

```py
import logging

from sqlalchemy.engine import CreateEnginePlugin
from sqlalchemy import event

class LogCursorEventsPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        # consume the parameter "log_cursor_logging_name" from the
        # URL query
        logging_name = url.query.get("log_cursor_logging_name", "log_cursor")

        self.log = logging.getLogger(logging_name)

    def update_url(self, url):
        "update the URL to one that no longer includes our parameters"
        return url.difference_update_query(["log_cursor_logging_name"])

    def engine_created(self, engine):
        "attach an event listener after the new Engine is constructed"
        event.listen(engine, "before_cursor_execute", self._log_event)

    def _log_event(
        self,
        conn,
        cursor,
        statement,
        parameters,
        context,
        executemany):

        self.log.info("Plugin logged cursor event: %s", statement)
```

插件的注册方式与方言类似，都是使用入口点：

```py
entry_points={
    'sqlalchemy.plugins': [
        'log_cursor_plugin = myapp.plugins:LogCursorEventsPlugin'
    ]
```

使用上述名称的插件将从数据库 URL 中调用，如下所示：

```py
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?"
    "plugin=log_cursor_plugin&log_cursor_logging_name=mylogger"
)
```

`plugin` URL 参数支持多个实例，因此 URL 可以指定多个插件；它们按照 URL 中指定的顺序加载：

```py
engine = create_engine(
  "mysql+pymysql://scott:tiger@localhost/test?"
  "plugin=plugin_one&plugin=plugin_twp&plugin=plugin_three")
```

插件名称也可以直接通过`create_engine()`使用 `create_engine.plugins` 参数传递：

```py
engine = create_engine(
  "mysql+pymysql://scott:tiger@localhost/test",
  plugins=["myplugin"])
```

从版本 1.2.3 开始新增功能：插件名称也可以作为列表指定给`create_engine()`。

插件可能会从`URL` 对象以及`kwargs` 字典中获取特定于插件的参数，该字典是传递给 `create_engine()` 调用的参数字典。"消耗"这些参数包括，它们在插件初始化时必须被移除，以便不将参数传递给 `Dialect` 构造函数，否则它们将引发 `ArgumentError`，因为方言不知道它们。

自 SQLAlchemy 版本 1.4 起，应继续直接从`kwargs` 字典中消耗参数，通过诸如 `dict.pop` 的方法删除值。应通过实现 `CreateEnginePlugin.update_url()` 方法来消耗来自 `URL` 对象的参数，返回一个去除了特定于插件的参数的新 `URL` 副本：

```py
class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        self.my_argument_one = url.query['my_argument_one']
        self.my_argument_two = url.query['my_argument_two']
        self.my_argument_three = kwargs.pop('my_argument_three', None)

    def update_url(self, url):
        return url.difference_update_query(
            ["my_argument_one", "my_argument_two"]
        )
```

像上面所示的参数将被从`create_engine()`调用中消耗掉，例如：

```py
from sqlalchemy import create_engine

engine = create_engine(
  "mysql+pymysql://scott:tiger@localhost/test?"
  "plugin=myplugin&my_argument_one=foo&my_argument_two=bar",
  my_argument_three='bat'
)
```

从版本 1.4 开始变更：`URL` 对象现在是不可变的；一个需要修改 `URL` 的 `CreateEnginePlugin` 应该实现新添加的 `CreateEnginePlugin.update_url()` 方法，在构造插件后调用该方法。

对于迁移，以以下方式构造插件，检查`CreateEnginePlugin.update_url()`方法的存在以检测运行的版本：

```py
class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        if hasattr(CreateEnginePlugin, "update_url"):
            # detect the 1.4 API
            self.my_argument_one = url.query['my_argument_one']
            self.my_argument_two = url.query['my_argument_two']
        else:
            # detect the 1.3 and earlier API - mutate the
            # URL directly
            self.my_argument_one = url.query.pop('my_argument_one')
            self.my_argument_two = url.query.pop('my_argument_two')

        self.my_argument_three = kwargs.pop('my_argument_three', None)

    def update_url(self, url):
        # this method is only called in the 1.4 version
        return url.difference_update_query(
            ["my_argument_one", "my_argument_two"]
        )
```

另请参阅

URL 对象现在是不可变的 - 对`URL`变更的概述，其中还包括有关`CreateEnginePlugin`的说明。

**成员**

__init__(), engine_created(), handle_dialect_kwargs(), handle_pool_kwargs(), update_url()

当引擎创建过程完成并产生`Engine`对象时，它会再次通过`CreateEnginePlugin.engine_created()`钩子传递给插件。在此钩子中，可以对引擎进行其他更改，通常涉及事件的设置（例如在 Core Events 中定义的事件）。

```py
method __init__(url: URL, kwargs: Dict[str, Any])
```

构造一个新的`CreateEnginePlugin`。

对于每次调用`create_engine()`，都会单独实例化插件对象。一个单独的`Engine`将传递给相应的此 URL 的`CreateEnginePlugin.engine_created()`方法。

参数：

+   `url` –

    `URL`对象。插件可以检查`URL`以获取参数。插件使用的参数应通过从`CreateEnginePlugin.update_url()`方法返回的更新后的`URL`来删除。

    1.4 版更改：`URL`对象现在是不可变的，因此需要更改`URL`对象的`CreateEnginePlugin`应实现`CreateEnginePlugin.update_url()`方法。

+   `kwargs` – 传递给 `create_engine()` 的关键字参数。

```py
method engine_created(engine: Engine) → None
```

在完全构造时接收 `Engine` 对象。

插件可能会对引擎进行额外的更改，例如注册引擎或连接池事件。

```py
method handle_dialect_kwargs(dialect_cls: Type[Dialect], dialect_args: Dict[str, Any]) → None
```

解析和修改方言关键字参数

```py
method handle_pool_kwargs(pool_cls: Type[Pool], pool_args: Dict[str, Any]) → None
```

解析和修改池关键字参数

```py
method update_url(url: URL) → URL
```

更新 `URL`。

应返回一个新的 `URL`。通常使用此方法来消耗从 `URL` 中的配置参数，必须删除这些参数，因为方言不会识别它们。`URL.difference_update_query()` 方法可用于删除这些参数。有关示例，请参见 `CreateEnginePlugin` 的文档字符串。

版本 1.4 中的新功能。

```py
class sqlalchemy.engine.Engine
```

将 `Pool` 和 `Dialect` 连接起来，以提供数据库连接和行为的源。

通过 `create_engine()` 函数公开实例化一个 `Engine` 对象。

另请参阅

引擎配置

与引擎和连接一起工作

**成员**

begin(), clear_compiled_cache(), connect(), dispose(), driver, engine, execution_options(), get_execution_options(), name, raw_connection(), update_execution_options()

**类签名**

类 `sqlalchemy.engine.Engine` (`sqlalchemy.engine.interfaces.ConnectionEventsTarget`, `sqlalchemy.log.Identified`, `sqlalchemy.inspection.Inspectable`)

```py
method begin() → Iterator[Connection]
```

返回一个上下文管理器，提供一个已建立的 `Connection` 和 `Transaction`。

例如：

```py
with engine.begin() as conn:
    conn.execute(
        text("insert into table (x, y, z) values (1, 2, 3)")
    )
    conn.execute(text("my_special_procedure(5)"))
```

操作成功后，`Transaction` 将被提交。如果发生错误，则 `Transaction` 将被回滚。

另请参见

`Engine.connect()` - 从 `Engine` 中获取 `Connection`。

`Connection.begin()` - 为特定的 `Connection` 开始一个 `Transaction`。

```py
method clear_compiled_cache() → None
```

清除与方言相关联的编译缓存。

这仅适用于通过 `create_engine.query_cache_size` 参数建立的内置缓存。它不会影响通过 `Connection.execution_options.compiled_cache` 参数传递的任何字典缓存。

版本 `1.4` 中的新功能。

```py
method connect() → Connection
```

返回一个新的 `Connection` 对象。

`Connection` 作为 Python 上下文管理器，因此该方法的典型用法如下：

```py
with engine.connect() as connection:
    connection.execute(text("insert into table values ('foo')"))
    connection.commit()
```

在上述代码块完成后，连接被“关闭”，其底层 DBAPI 资源被返回到连接池。这也会导致回滚任何明确启动的事务或通过 autobegin 启动的事务，并且如果已经开始并且仍在进行中，则会发出 `ConnectionEvents.rollback()` 事件。

另请参见

`Engine.begin()`

```py
method dispose(close: bool = True) → None
```

处理此 `Engine` 使用的连接池。

在旧连接池被处理后立即创建一个新的连接池。之前的连接池会被主动处理，通过关闭该池中当前所有已签入的连接，或者被动处理，即失去对其的引用，但不关闭任何连接。后一种策略更适用于 forked Python 进程中的初始化程序。

参数：

**close** –

如果保留在其默认值 `True`，则会完全关闭所有**当前检入**的数据库连接。仍然检出的连接将**不会**被关闭，但它们将不再与此 `Engine` 关联，因此当它们逐个关闭时，它们将与之关联的`Pool`最终将被垃圾收集，如果尚未在检入时关闭，则将完全关闭。

如果设置为 `False`，则先前的连接池将被取消引用，否则不会以任何方式触及。

版本 1.4.33 中的新增功能：添加了`Engine.dispose.close`参数，以允许在子进程中替换连接池而不干扰父进程使用的连接。

请参阅

引擎处理

在多进程或 os.fork() 中使用连接池

```py
attribute driver
```

此 `Engine` 使用的 `Dialect` 的驱动程序名称。

```py
attribute engine
```

返回此 `Engine`。

用于接受同一变量中的 `Connection` / `Engine` 对象的旧方案。

```py
method execution_options(**opt: Any) → OptionEngine
```

返回一个新的`Engine`，它将提供具有给定执行选项的`Connection`对象。

返回的`Engine`与原始`Engine`相关联，因为它共享相同的连接池和其他状态：

+   新`Engine`使用的`Pool`是同一实例。`Engine.dispose()`方法将替换父引擎的连接池实例以及此引擎的连接池实例。

+   事件监听器是“级联”的 - 意思是，新的`Engine`继承了父级的事件，并且新事件可以单独与新的`Engine`相关联。

+   日志配置和日志名称是从父`Engine`复制的。

`Engine.execution_options()`方法的目的是实现多个`Engine`对象引用相同连接池的方案，但是通过影响每个引擎的一些执行级别行为的选项进行区分。其中一个示例是将其分成单独的“读取器”和“写入器”`Engine`实例，其中一个`Engine`配置了较低的隔离级别设置，甚至使用“autocommit”禁用事务。此配置的示例位于 Maintaining Multiple Isolation Levels for a Single Engine。

另一个示例是使用一个自定义选项`shard_id`，该选项由事件消耗以在数据库连接上更改当前模式：

```py
from sqlalchemy import event
from sqlalchemy.engine import Engine

primary_engine = create_engine("mysql+mysqldb://")
shard1 = primary_engine.execution_options(shard_id="shard1")
shard2 = primary_engine.execution_options(shard_id="shard2")

shards = {"default": "base", "shard_1": "db1", "shard_2": "db2"}

@event.listens_for(Engine, "before_cursor_execute")
def _switch_shard(conn, cursor, stmt,
        params, context, executemany):
    shard_id = conn.get_execution_options().get('shard_id', "default")
    current_shard = conn.info.get("current_shard", None)

    if current_shard != shard_id:
        cursor.execute("use %s" % shards[shard_id])
        conn.info["current_shard"] = shard_id
```

上述示例展示了两个`Engine`对象，它们分别用作工厂，用于创建具有预先建立的“shard_id”执行选项的`Connection`对象。然后，`ConnectionEvents.before_cursor_execute()`事件处理程序解释此执行选项，以在语句执行之前发出 MySQL `use`语句以切换数据库，同时使用`Connection.info`字典跟踪我们已经建立的数据库。

参见

`Connection.execution_options()` - 更新`Connection`对象上的执行选项。

`Engine.update_execution_options()` - 更新给定`Engine`的执行选项。

`Engine.get_execution_options()`

```py
method get_execution_options() → _ExecuteOptions
```

获取执行期间将生效的非 SQL 选项。

参见

`Engine.execution_options()`

```py
attribute name
```

正在使用的`Engine`的`Dialect`的字符串名称。

```py
method raw_connection() → PoolProxiedConnection
```

从连接池返回“原始”DBAPI 连接。

返回的对象是底层驱动程序正在使用的 DBAPI 连接对象的代理版本。该对象将具有与真实的 DBAPI 连接相同的所有行为，只是它的`close()`方法将导致连接返回到池中，而不是真正关闭。

当`Connection`对象已经存在时，可以使用`Connection.connection`访问器获取 DBAPI 连接。

另请参阅

使用驱动程序 SQL 和原始 DBAPI 连接

```py
method update_execution_options(**opt: Any) → None
```

更新此`Engine`的默认`execution_options`字典。

在**opt 中给定的键/值将添加到将用于所有连接的默认执行选项中。此字典的初始内容可以通过`execution_options`参数发送到`create_engine()`。

另请参阅

`Connection.execution_options()`

`Engine.execution_options()`

```py
class sqlalchemy.engine.ExceptionContext
```

封装正在进行中的错误条件的信息。

**成员**

chained_exception, connection, cursor, dialect, engine, execution_context, invalidate_pool_on_disconnect, is_disconnect, is_pre_ping, original_exception, parameters, sqlalchemy_exception, statement

此对象仅用于传递给`DialectEvents.handle_error()`事件，支持可在不向后不兼容地扩展的接口。

```py
attribute chained_exception: BaseException | None
```

如果有的话，前一个处理程序在异常链中返回的异常。

如果存在，则此异常将最终由 SQLAlchemy 引发，除非后续处理程序替换它。

可能为 None。

```py
attribute connection: Connection | None
```

异常发生时使用的`Connection`。

此成员存在，除非首次连接失败。

另请参阅

`ExceptionContext.engine`

```py
attribute cursor: DBAPICursor | None
```

DBAPI 游标对象。

可能为 None。

```py
attribute dialect: Dialect
```

正在使用的`Dialect`。

此成员对所有事件钩子的调用都存在。

在 2.0 版中新增。

```py
attribute engine: Engine | None
```

发生异常时正在使用的`Engine`。

除了在连接池“预连接”过程中处理错误时，此成员在所有情况下都存在。

```py
attribute execution_context: ExecutionContext | None
```

正在进行的执行操作对应的`ExecutionContext`。

对于语句执行操作，此标志存在，但对于诸如事务开始/结束之类的操作则不存在。当在构造`ExecutionContext`之前引发异常时，此标志也不存在。

请注意，`ExceptionContext.statement`和`ExceptionContext.parameters`成员可能表示与`ExecutionContext`的不同值，可能是在`ConnectionEvents.before_cursor_execute()`事件或类似事件修改了要发送的语句/参数的情况下。

可能为 None。

```py
attribute invalidate_pool_on_disconnect: bool
```

表示在“断开连接”条件生效时是否应使池中的所有连接失效。

在`DialectEvents.handle_error()`事件的范围内将此标志设置为 False 将导致在断开连接时不会使池中的所有连接失效；只有实际上受到错误影响的当前连接将被使失效。

此标志的目的是用于自定义断开连接处理方案，在此方案中，池中其他连接的失效是基于其他条件进行的，甚至是基于每个连接的条件进行的。

```py
attribute is_disconnect: bool
```

表示发生的异常是否代表“断开连接”条件。

在`DialectEvents.handle_error()`处理程序的范围内，此标志将始终为 True 或 False。

SQLAlchemy 将推迟到此标志以确定是否随后应使连接失效。也就是说，通过分配给此标志，可以通过更改此标志来调用或阻止连接和池失效的“断开”事件。

注意

使用`create_engine.pool_pre_ping`参数启用的池“pre_ping”处理程序在决定“ping”返回 false 而不是接收到未处理错误之前不会查看此事件。对于这种用例，可以使用基于 engine_connect()的遗留配方。将来的 API 允许在所有功能中更全面地自定义“断开”检测机制。

```py
attribute is_pre_ping: bool
```

指示此错误是否发生在设置`create_engine.pool_pre_ping`为`True`时执行的“pre-ping”步骤中。在此模式下，`ExceptionContext.engine`属性将为`None`。正在使用的方言可通过`ExceptionContext.dialect`属性访问。

版本 2.0.5 中的新功能。

```py
attribute original_exception: BaseException
```

被捕获的异常对象。

此成员始终存在。

```py
attribute parameters: _DBAPIAnyExecuteParams | None
```

直接发送到 DBAPI 的参数集合。

可能是 None。

```py
attribute sqlalchemy_exception: StatementError | None
```

包装原始异常的`sqlalchemy.exc.StatementError`，如果未绕过事件处理，则将引发该异常。

可能为 None，因为不是所有的异常类型都被 SQLAlchemy 包装。对于子类化 dbapi 的 Error 类的 DBAPI 级别异常，此字段将始终存在。

```py
attribute statement: str | None
```

直接发送到 DBAPI 的字符串 SQL 语句。

可能是 None。

```py
class sqlalchemy.engine.NestedTransaction
```

表示“嵌套”或 SAVEPOINT 事务。

`NestedTransaction`对象是通过调用`Connection.begin_nested()`方法创建的`Connection`。

使用`NestedTransaction`时，“begin”/“commit”/“rollback”的语义如下：

+   “begin”操作对应于“BEGIN SAVEPOINT”命令，其中保存点被赋予一个显式名称，该名称是此对象状态的一部分。

+   `NestedTransaction.commit()`方法对应于“RELEASE SAVEPOINT”操作，使用与此`NestedTransaction`关联的保存点标识符。

+   `NestedTransaction.rollback()`方法对应于“ROLLBACK TO SAVEPOINT”操作，使用与此`NestedTransaction`关联的保存点标识符。

模仿外部事务的语义以便代码可以以一种不可知的方式处理“保存点”事务和“外部”事务。

另请参阅

使用 SAVEPOINT - SAVEPOINT API 的 ORM 版本。

**成员**

close(), commit(), rollback()

**类签名**

类`sqlalchemy.engine.NestedTransaction`（`sqlalchemy.engine.Transaction`）

```py
method close() → None
```

*继承自* `Transaction.close()` *方法的* `Transaction`

关闭此`Transaction`。

如果此事务是嵌套的开始/提交中的基本事务，则事务将回滚。否则，该方法将返回。

这用于取消事务而不影响封闭事务的范围。

```py
method commit() → None
```

*继承自* `Transaction.commit()` *方法的* `Transaction`

提交此`Transaction`。

这个实现可能会根据使用的事务类型而有所不同：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于一个 COMMIT。

+   对于`NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
method rollback() → None
```

*继承自* `Transaction.rollback()` *方法的* `Transaction`

回滚此`Transaction`。

此实现可能根据使用的事务类型而有所不同：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于一个 ROLLBACK。

+   对于`NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
class sqlalchemy.engine.RootTransaction
```

表示`Connection`上的“根”事务。

这对应于当前正在为`Connection`执行的“BEGIN/COMMIT/ROLLBACK”。通过调用`Connection.begin()`方法创建`RootTransaction`，并且在其活动范围内与`Connection`关联。当前使用的`RootTransaction`可通过`Connection.get_transaction`方法访问`Connection`。

在 2.0 style 中使用时，`Connection`还采用“autobegin”行为，每当处于非事务状态的连接用于在 DBAPI 连接上发出命令时，就会创建一个新的`RootTransaction`。在 2.0 style 使用中，`RootTransaction` 的范围可以使用`Connection.commit()`和`Connection.rollback()`方法进行控制。

**成员**

close(), commit(), rollback()

**类签名**

类`sqlalchemy.engine.RootTransaction`（`sqlalchemy.engine.Transaction`）

```py
method close() → None
```

继承自`Transaction.close()` *方法的* `Transaction`

关闭此 `Transaction`。

如果此事务是嵌套在 begin/commit 中的基本事务，则事务将回滚()。否则，该方法返回。

这用于取消事务，而不影响封闭事务的范围。

```py
method commit() → None
```

继承自`Transaction.commit()` *方法的* `Transaction`

提交此 `Transaction`。

实现可能根据正在使用的事务类型而变化：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于一个 COMMIT。

+   对于 `NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
method rollback() → None
```

继承自`Transaction.rollback()` *方法的* `Transaction`

回滚此 `Transaction`。

实现可能根据正在使用的事务类型而变化：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于一个 ROLLBACK。

+   对于 `NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
class sqlalchemy.engine.Transaction
```

表示正在进行的数据库事务。

调用 `Connection.begin()` 方法的 `Connection` 获得 `Transaction` 对象：

```py
from sqlalchemy import create_engine
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
connection = engine.connect()
trans = connection.begin()
connection.execute(text("insert into x (a, b) values (1, 2)"))
trans.commit()
```

该对象提供了`rollback()`和`commit()`方法以控制事务边界。它还实现了上下文管理器接口，以便 Python `with`语句可以与`Connection.begin()`方法一起使用：

```py
with connection.begin():
    connection.execute(text("insert into x (a, b) values (1, 2)"))
```

Transaction 对象**不**是线程安全的。

**成员**

close()，commit()，rollback()

另请参阅

`Connection.begin()`

`Connection.begin_twophase()`

`Connection.begin_nested()`

**类签名**

类`sqlalchemy.engine.Transaction`（`sqlalchemy.engine.util.TransactionalContext`）

```py
method close() → None
```

关闭此`Transaction`。

如果此事务是 begin/commit 嵌套中的基本事务，则事务将回滚()。否则，该方法返回。

这用于取消 Transaction 而不影响封闭事务的范围。

```py
method commit() → None
```

提交此`Transaction`。

其实现可能根据使用的事务类型而变化：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于 COMMIT。

+   对于`NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可能会使用特定于 DBAPI 的两阶段事务方法。

```py
method rollback() → None
```

回滚此`Transaction`。

其实现可能根据使用的事务类型而变化：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于 ROLLBACK。

+   对于`NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可能会使用特定于 DBAPI 的两阶段事务方法。

```py
class sqlalchemy.engine.TwoPhaseTransaction
```

表示两阶段事务。

可以使用 `Connection.begin_twophase()` 方法获取新的 `TwoPhaseTransaction` 对象。

接口与 `Transaction` 相同，只增加了 `prepare()` 方法。

**成员**

close(), commit(), prepare(), rollback()

**类签名**

类 `sqlalchemy.engine.TwoPhaseTransaction` (`sqlalchemy.engine.RootTransaction`)

```py
method close() → None
```

*继承自* `Transaction.close()` *方法的* `Transaction`

关闭此 `Transaction`。

如果此事务是开始/提交嵌套中的基本事务，则事务将回滚()。否则，该方法返回。

这用于取消事务，而不影响封闭事务的范围。

```py
method commit() → None
```

*继承自* `Transaction.commit()` *方法的* `Transaction`

提交此 `Transaction`。

其实现可能会根据所使用的事务类型而变化：

+   对于简单的数据库事务（例如 `RootTransaction`），它对应于 COMMIT。

+   对于 `NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
method prepare() → None
```

准备此 `TwoPhaseTransaction`。

在 PREPARE 之后，可以提交事务。

```py
method rollback() → None
```

*继承自* `Transaction.rollback()` *方法的* `Transaction`

回滚此 `Transaction`。

实现可能会根据使用的事务类型而有所不同：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于 ROLLBACK。

+   对于`NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可以使用特定于 DBAPI 的方法进行两阶段事务。

## Result Set API

| 对象名称 | 描述 |
| --- | --- |
| ChunkedIteratorResult | 从生成迭代器的可调用对象工作的`IteratorResult`。 |
| CursorResult | 表示来自 DBAPI 游标的状态的结果。 |
| FilterResult | 一个`Result`的包装器，返回除`Row`对象之外的对象，例如字典或标量对象。 |
| FrozenResult | 代表一个适合缓存的“冻结”状态的`Result`对象。 |
| IteratorResult | 从 Python 迭代器的`Row`对象或类似行的数据获取数据的`Result`。 |
| MappingResult | 一个`Result`的包装器，返回字典值而不是`Row`值。 |
| MergedResult | 从任意数量的`Result`对象合并的`Result`。 |
| Result | 代表一组数据库结果。 |
| Row | 代表一个单一的结果行。 |
| RowMapping | 一个将列名和对象映射到`Row`值的`Mapping`。 |
| ScalarResult | 一个`Result`的包装器，返回标量值而不是`Row`值。 |
| TupleResult | 一个`Result`，被类型化为返回纯 Python 元组而不是行。 |

```py
class sqlalchemy.engine.ChunkedIteratorResult
```

一个从生成迭代器的可调用对象中工作的 `IteratorResult`。

给定的 `chunks` 参数是一个函数，该函数给出每个块中要返回的行数，或者为 `None` 以返回所有行。该函数应返回一个未使用的列表迭代器，每个列表的大小为请求的大小。

可以在任何时候再次调用该函数，在这种情况下，它应从相同的结果集继续，但根据给定的块大小进行调整。

自 1.4 版开始。

**成员**

yield_per()

**类签名**

类 `sqlalchemy.engine.ChunkedIteratorResult` (`sqlalchemy.engine.IteratorResult`)

```py
method yield_per(num: int) → Self
```

配置行提取策略，以一次提取 `num` 行。

这会影响结果在迭代结果对象时的基础行为，或者在使用诸如 `Result.fetchone()` 这样一次返回一行的方法时进行使用。来自底层游标或其他数据源的数据将在内存中缓冲到这么多行，并且缓冲集合然后将一行一次或请求的行数作为输出。每次缓冲清除时，它都将刷新为这么多行或如果剩余的行数少于这么多行则为剩余的行数。

`Result.yield_per()` 方法通常与 `Connection.execution_options.stream_results` 执行选项一起使用，该选项将允许正在使用的数据库方言使用服务器端游标，如果 DBAPI 支持与其默认操作模式分离的特定“服务器端游标”模式。

提示

考虑使用 `Connection.execution_options.yield_per` 执行选项，它将同时设置 `Connection.execution_options.stream_results` 以确保使用服务器端游标，并自动调用 `Result.yield_per()` 方法一次性建立固定的行缓冲区大小。

`Connection.execution_options.yield_per` 执行选项可用于 ORM 操作，其与`Session`相关的用法在使用 Yield Per 获取大型结果集中进行了描述。与`Connection`配合使用的仅限于 Core 的版本是 SQLAlchemy 1.4.40 的新功能。

1.4 版中的新功能。

参数：

**num** – 每次重新填充缓冲区时要获取的行数。如果设置为小于 1 的值，则获取下一个缓冲区的所有行。

另请参阅

使用服务器端游标（又名流式结果） - 描述了`Result.yield_per()`的核心行为。

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.engine.CursorResult
```

表示来自 DBAPI 游标的状态的 Result。

自 1.4 版更改：`CursorResult` 类取代了以前的 `ResultProxy` 接口。这些类基于`Result`调用 API，为 SQLAlchemy Core 和 SQLAlchemy ORM 提供了更新的使用模型和调用外观。

通过`Row`类返回数据库行，该类在 DBAPI 返回的原始数据之上提供了其他 API 功能和行为。通过诸如`Result.scalars()`方法之类的过滤器，还可以返回其他类型的对象。

另请参阅

使用 SELECT 语句 - 用于访问`CursorResult`和`Row`对象的入门材料。

**成员**

all(), close(), columns(), fetchall(), fetchmany(), fetchone(), first(), freeze(), inserted_primary_key, inserted_primary_key_rows, is_insert, keys(), last_inserted_params(), last_updated_params(), lastrow_has_defaults(), lastrowid, mappings(), merge(), one(), one_or_none(), partitions(), postfetch_cols(), prefetch_cols(), returned_defaults, returned_defaults_rows, returns_rows, rowcount, scalar(), scalar_one(), scalar_one_or_none(), scalars(), splice_horizontally(), splice_vertically(), supports_sane_multi_rowcount(), supports_sane_rowcount(), t, tuples(), unique(), yield_per()

**类签名**

类 `sqlalchemy.engine.CursorResult` (`sqlalchemy.engine.Result`)

```py
method all() → Sequence[Row[_TP]]
```

*继承自* `Result.all()` *方法的* `Result`

返回序列中的所有行。

调用后关闭结果集。后续调用将返回一个空序列。

版本 1.4 中的新功能。

返回：

一系列 `Row` 对象。

另请参见

使用服务器端游标（又称流式结果） - 如何在 Python 中流式传输大型结果集而不完全加载它。

```py
method close() → Any
```

关闭此 `CursorResult`。

如果仍然存在底层 DBAPI 游标，则关闭该游标对应的语句执行。请注意，当 `CursorResult` 耗尽所有可用行时，DBAPI 游标会自动释放。通常，`CursorResult.close()` 是一个可选方法，除非在丢弃一个仍具有待提取的附加行的 `CursorResult` 时。

调用此方法后，再调用提取方法将不再有效，并且在后续使用时会引发 `ResourceClosedError`。

另请参见

使用引擎和连接

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

*继承自* `Result.columns()` *方法的* `Result`

确定每行应返回的列。

此方法可用于限制返回的列，也可用于重新排序列。给定的表达式列表通常是一系列整数或字符串键名。它们也可以是适当的 `ColumnElement` 对象，这些对象对应于给定的语句构造。

2.0 版本中的更改：由于 1.4 版本中的一个错误，`Result.columns()` 方法具有了错误的行为，仅使用一个索引调用该方法会导致 `Result` 对象生成标量值，而不是 `Row` 对象。在 2.0 版本中，已经纠正了这种行为，调用 `Result.columns()` 时使用单个索引将产生一个继续生成 `Row` 对象的 `Result` 对象，该对象仅包含单个列。

例如：

```py
statement = select(table.c.x, table.c.y, table.c.z)
result = connection.execute(statement)

for z, y in result.columns('z', 'y'):
    # ...
```

使用语句本身的列对象的示例：

```py
for z, y in result.columns(
        statement.selected_columns.c.z,
        statement.selected_columns.c.y
):
    # ...
```

1.4 版本中的新内容。

参数：

***col_expressions** – 表示要返回的列。元素可以是整数行索引、字符串列名称或与选择构造相对应的适当 `ColumnElement` 对象。

返回：

带有给定修改的此 `Result` 对象。

```py
method fetchall() → Sequence[Row[_TP]]
```

*继承自* `Result.fetchall()` *方法的* `Result`

`Result.all()` 方法的同义词。

```py
method fetchmany(size: int | None = None) → Sequence[Row[_TP]]
```

*继承自* `Result.fetchmany()` *方法的* `Result`

获取多行。

当所有行都用完时，返回空序列。

此方法用于向后兼容 SQLAlchemy 1.x.x。

要按组获取行，请使用 `Result.partitions()` 方法。

返回：

一系列 `Row` 对象。

另请参阅

`Result.partitions()`

```py
method fetchone() → Row[_TP] | None
```

*继承自* `Result.fetchone()` *方法的* `Result`

获取一行。

当所有行都用完时，返回 None。

此方法用于向后兼容 SQLAlchemy 1.x.x。

要仅获取结果的第一行，请使用 `Result.first()` 方法。要遍历所有行，请直接遍历 `Result` 对象。

返回：

如果未应用任何过滤器，则返回一个 `Row` 对象，否则返回 `None`。

```py
method first() → Row[_TP] | None
```

*继承自* `Result.first()` *方法的* `Result`

获取第一行或者如果不存在行则返回 `None`。

关闭结果集并丢弃剩余行。

注意

默认情况下，此方法返回一行，例如元组。要返回确切的一个单一标量值，即第一行的第一列，请使用 `Result.scalar()` 方法，或组合 `Result.scalars()` 和 `Result.first()`。

此外，与传统 ORM `Query.first()` 方法的行为相比，**不会应用限制**到用于生成此 `Result` 的 SQL 查询；对于在生成行之前在内存中缓冲结果的 DBAPI 驱动程序，所有行将被发送到 Python 进程，并且除了第一行之外的所有行都将被丢弃。

另请参阅

ORM 查询与核心选择统一

返回：

一个 `Row` 对象，如果没有剩余行则为 None。

另请参阅

`Result.scalar()`

`Result.one()`

```py
method freeze() → FrozenResult[_TP]
```

*继承自* `Result.freeze()` *方法的* `Result`

返回一个可调用对象，当调用时将产生此 `Result` 的副本。

返回的可调用对象是 `FrozenResult` 的实例。

用于结果集缓存。必须在结果未被消耗时调用该方法，并且调用该方法将完全消耗结果。当从缓存中检索到 `FrozenResult` 时，可以任意多次调用它，每次对其存储的行集产生一个新的 `Result` 对象。

另请参阅

重新执行语句 - 在 ORM 中示例用法以实现结果集缓存。

```py
attribute inserted_primary_key
```

返回刚刚插入行的主键。

返回值是一个按照源 `Table` 中配置的主键列顺序表示的 `Row` 对象的命名元组。

从版本 1.4.8 开始更改：- `CursorResult.inserted_primary_key` 值现在是通过 `Row` 类的命名元组，而不是普通元组。

此访问器仅适用于未明确指定`Insert.returning()`的单行`insert()`构造。对于多行插入，虽然大多数后端尚不支持，但可以使用`CursorResult.inserted_primary_key_rows`访问器。

请注意，指定了 server_default 子句或以其他方式不符合“自增”列（请参阅`Column`中的注释），并且是使用数据库端默认值生成的主键列，除非后端支持“returning”并且执行了启用“隐式 returning”的插入语句，否则将在此列表中显示为`None`。

如果执行的语句不是编译的表达式构造或不是 insert() 构造，则引发`InvalidRequestError`。

```py
attribute inserted_primary_key_rows
```

返回`CursorResult.inserted_primary_key`的值，作为包含在列表中的行；一些方言可能还支持多行形式。

注意

如下所示，在当前的 SQLAlchemy 版本中，仅当使用 psycopg2 方言时，此访问器才有用。未来的版本希望将此功能推广到更多的方言。

此访问器被添加以支持目前仅由 Psycopg2 快速执行助手功能实现的方言，目前**仅适用于 psycopg2 方言**，它允许一次插入多行同时仍保留能够返回服务器生成的主键值的行为。

+   `当使用 psycopg2 方言或其他可能在即将发布的版本中支持“快速 executemany”样式插入的方言时`：在调用 INSERT 语句时，将行列表作为第二个参数传递给`Connection.execute()`时，此访问器将提供一个行列表，其中每一行包含被插入的每一行的主键值。

+   `当使用所有其他尚不支持此功能的方言/后端时`：此访问器仅对`单行 INSERT 语句`有用，并返回与`CursorResult.inserted_primary_key`相同的信息，放在一个单元素列表中。当 INSERT 语句与要插入的行的列表一起执行时，列表将包含语句中插入的每一行，但对于任何服务器生成的值，它将包含`None`。

未来的 SQLAlchemy 版本将进一步将 psycopg2 的“快速执行助手”功能泛化以适应其他方言，从而使此访问器更具一般性。

在 1.4 版本中新增。

另请参阅

`CursorResult.inserted_primary_key`

```py
attribute is_insert
```

如果此`CursorResult`是执行表达式语言编译的`insert()`构造的结果，则返回 True。

当为 True 时，这意味着可以访问`inserted_primary_key`属性，假设语句未包含用户定义的“returning”构造。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys.keys` *方法的* `sqlalchemy.engine._WithKeys`

返回一个可迭代的视图，该视图产生每个`Row`表示的字符串键。

键可以表示核心语句返回的列的标签或 ORM 执行返回的 ORM 类的名称。

该视图还可以使用 Python 的`in`运算符进行键包含性测试，该运算符将测试视图中表示的字符串键，以及列对象等备用键。

在 1.4 版本中更改：返回一个键视图对象，而不是一个普通列表。

```py
method last_inserted_params()
```

从此执行返回插入的参数集合。

如果执行的语句不是编译的表达式构造或不是 insert()构造，则引发`InvalidRequestError`。

```py
method last_updated_params()
```

从此执行返回更新的参数集合。

如果执行的语句不是编译的表达式构造或不是 update()构造，则引发`InvalidRequestError`。

```py
method lastrow_has_defaults()
```

从底层的`ExecutionContext`返回`lastrow_has_defaults()`。

有关详细信息，请参阅`ExecutionContext`。

```py
attribute lastrowid
```

返回 DBAPI 游标上的 ‘lastrowid’ 访问器。

这是一个特定于 DBAPI 的方法，仅在支持的后端中才起作用，适用于适当的语句。其行为在各后端之间不一致。

在使用 insert() 表达式构造时，通常不需要使用此方法；`CursorResult.inserted_primary_key` 属性提供了一个新插入行的主键值元组，无论数据库后端如何。

```py
method mappings() → MappingResult
```

*继承自* `Result.mappings()` *方法的* `Result`

应用一个映射过滤器到返回的行，返回一个 `MappingResult` 实例。

当应用此过滤器时，获取行将返回 `RowMapping` 对象而不是 `Row` 对象。

新版本 1.4 中新增。

返回：

一个指向此 `Result` 对象的新 `MappingResult` 过滤对象。

```py
method merge(*others: Result[Any]) → MergedResult[Any]
```

将此`Result`与其他兼容的结果对象合并。

返回的对象是一个 `MergedResult` 的实例，它将由给定结果对象的迭代器组成。

新结果将使用此结果对象的元数据。后续的结果对象必须针对相同的结果/游标元数据集，否则行为是未定义的。

```py
method one() → Row[_TP]
```

*继承自* `Result.one()` *方法的* `Result`

仅返回一行数据或引发异常。

如果结果没有返回行，则引发 `NoResultFound`，如果返回多行，则引发 `MultipleResultsFound`。

注意

此方法默认返回一个 **行**，例如元组。要返回确切的一个单一标量值，即第一行的第一列，请使用 `Result.scalar_one()` 方法，或者结合使用 `Result.scalars()` 和 `Result.one()`。 

新版本 1.4 中新增。

返回：

第一个 `Row`。

引发：

`MultipleResultsFound`, `NoResultFound`

另请参阅

`Result.first()`

`Result.one_or_none()`

`Result.scalar_one()`

```py
method one_or_none() → Row[_TP] | None
```

*继承自* `Result.one_or_none()` *方法的* `Result`

返回至多一个结果或引发异常。

如果结果没有行，则返回 `None`。如果返回多行，则引发 `MultipleResultsFound`。

版本 1.4 中的新功能。

返回：

第一个 `Row` 或 `None`（如果没有可用行）。

引发：

`MultipleResultsFound`

另请参阅

`Result.first()`

`Result.one()`

```py
method partitions(size: int | None = None) → Iterator[Sequence[Row[_TP]]]
```

*继承自* `Result.partitions()` *方法的* `Result`

迭代给定大小的行子列表。

每个列表将具有给定的大小，最后一个要产生的列表除外，该列表可能具有少量行。不会产生空列表。

迭代器完全消耗时，结果对象会自动关闭。

请注意，除非使用了 `Connection.execution_options.stream_results` 执行选项指示驱动程序不应在可能的情况下预先缓冲结果，否则后端驱动程序通常会提前缓冲整个结果。不是所有驱动程序都支持此选项，对于不支持此选项的驱动程序，该选项会被静默忽略。

在使用 ORM 时，`Result.partitions()` 方法通常在内存方面更有效，当与 yield_per execution option 结合使用时，该方法指示 DBAPI 驱动程序在可用时使用服务器端游标，并指示 ORM 加载内部仅在每次从结果中产生一定数量的 ORM 对象之前构建一定数量的 ORM 对象。

版本 1.4 中的新功能。

参数：

**size** – 表示每个生成的列表中应该存在的最大行数。如果为 `None`，则使用由 `Result.yield_per()` 方法设置的值，如果已调用，或者使用等效的 `Connection.execution_options.yield_per` 执行选项。如果未设置 yield_per，则使用 `Result.fetchmany()` 的默认值，这可能是特定于后端的并且不太定义明确的。

返回：

列表的迭代器

另请参阅

使用服务器端游标（也称为流式结果）

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
method postfetch_cols()
```

从底层 `ExecutionContext` 返回 `postfetch_cols()`。

详细信息请参阅 `ExecutionContext`。

如果执行的语句不是编译后的表达式构造或不是 `insert()` 或 `update()` 构造，则会引发`InvalidRequestError`。

```py
method prefetch_cols()
```

从底层 `ExecutionContext` 返回 `prefetch_cols()`。

详细信息请参阅 `ExecutionContext`。

如果执行的语句不是编译后的表达式构造或不是 `insert()` 或 `update()` 构造，则会引发`InvalidRequestError`。

```py
attribute returned_defaults
```

返回使用 `ValuesBase.return_defaults()` 功能提取的默认列的值。

如果 `ValuesBase.return_defaults()` 未使用或后端不支持 RETURNING，则值为 `Row` 的实例，或者为 `None`。 

另请参阅

`ValuesBase.return_defaults()`

```py
attribute returned_defaults_rows
```

返回包含使用 `ValuesBase.return_defaults()` 功能提取的默认列的值的行列表。

返回值是 `Row` 对象的列表。

版本 `1.4` 中的新功能。

```py
attribute returns_rows
```

如果此 `CursorResult` 返回零个或多个行，则为 `True`。

即如果可以调用方法 `CursorResult.fetchone()`, `CursorResult.fetchmany()` `CursorResult.fetchall()`.

总的来说，`CursorResult.returns_rows` 的值应该始终与 DBAPI 游标是否具有 `.description` 属性同义，指示结果列的存在，需要注意的是，即使游标返回零行，如果发出了返回行的语句，游标仍然具有 `.description`。

这个属性对于所有针对 SELECT 语句的结果都应该为 True，以及对于使用 RETURNING 的 DML 语句 INSERT/UPDATE/DELETE 也应该为 True。对于没有使用 RETURNING 的 INSERT/UPDATE/DELETE 语句，该值通常为 False，但是有一些方言特定的例外情况，比如使用 MSSQL / pyodbc 方言时，会内联发出一个 SELECT 来检索插入的主键值。

```py
attribute rowcount
```

返回此结果的 'rowcount'。

‘rowcount’ 的主要目的是报告执行一次 UPDATE 或 DELETE 语句的 WHERE 条件匹配的行数（即对于单个参数集），然后可以将其与预期更新或删除的行数进行比较，作为断言数据完整性的手段。

此属性是从 DBAPI 的 `cursor.rowcount` 属性转移而来，之后游标关闭，以支持不在游标关闭后提供此值的 DBAPI。一些 DBAPI 可能为其他类型的语句（例如 INSERT 和 SELECT 语句）提供有意义的值。为了检索这些语句的 `cursor.rowcount`，将 `Connection.execution_options.preserve_rowcount` 执行选项设置为 True，这将导致在返回任何结果或游标关闭之前，无条件地缓存 `cursor.rowcount` 值，无论语句类型如何。

对于 DBAPI 不支持某种类型的语句和/或执行的情况，返回的值将为 `-1`，这是直接从 DBAPI 传递的，是 [**PEP 249**](https://peps.python.org/pep-0249/) 的一部分。所有 DBAPI 都应支持单参数集 UPDATE 和 DELETE 语句的 rowcount，但是。

注意

有关 `CursorResult.rowcount` 的注意事项：

+   此属性返回*匹配*的行数，这不一定与实际*修改*的行数相同。例如，如果 UPDATE 语句中的 SET 值与行中已存在的值相同，则对给定行可能没有净变化。这样的行将被匹配但不会被修改。在具有两种样式的后端（例如 MySQL）上，rowcount 被配置为在所有情况下返回匹配计数。

+   在默认情况下，`CursorResult.rowcount` 仅在与 UPDATE 或 DELETE 语句结合使用时才有用，并且仅适用于单组参数。对于其他类型的语句，除非使用`Connection.execution_options.preserve_rowcount`执行选项，否则 SQLAlchemy 不会尝试预先缓存该值。请注意，与[**PEP 249**](https://peps.python.org/pep-0249/)相反，许多 DBAPI 不支持不是 UPDATE 或 DELETE 的语句的 rowcount 值，特别是当返回的行没有完全预先缓冲时。不支持某种语句类型的 rowcount 的 DBAPI 应为这些语句返回值`-1`。

+   在执行具有多个参数集的单个语句（即 executemany）时，`CursorResult.rowcount`可能没有意义。大多数 DBAPI 不会跨多个参数集对“rowcount”值求和，并在访问时返回`-1`。

+   SQLAlchemy 的“INSERT 语句的多值插入”行为功能在设置`Connection.execution_options.preserve_rowcount`执行选项为 True 时支持正确填充`CursorResult.rowcount`。

+   使用 RETURNING 的语句可能不支持 rowcount，而是返回值`-1`。

另请参阅

从 UPDATE、DELETE 获取受影响的行数 - 在 SQLAlchemy 统一教程中

`Connection.execution_options.preserve_rowcount`

```py
method scalar() → Any
```

*继承自* `Result.scalar()` *方法的* `Result`

获取第一行的第一列，并关闭结果集。

如果没有要获取的行，则返回`None`。

不执行验证以测试是否还有其他行。

调用此方法后，对象已完全关闭，例如已调用 `CursorResult.close()` 方法。

返回：

一个 Python 标量值，或者如果没有剩余行则为 `None`。

```py
method scalar_one() → Any
```

*继承自* `Result.scalar_one()` *方法的* `Result`

返回确切的一个标量结果或引发异常。

这等同于调用 `Result.scalars()` 然后调用 `Result.one()`。

参见

`Result.one()`

`Result.scalars()`

```py
method scalar_one_or_none() → Any | None
```

*继承自* `Result.scalar_one_or_none()` *方法的* `Result`

返回确切的一个标量结果或 `None`。

这等同于调用 `Result.scalars()` 然后调用 `Result.one_or_none()`。

参见

`Result.one_or_none()`

`Result.scalars()`

```py
method scalars(index: _KeyIndexType = 0) → ScalarResult[Any]
```

*继承自* `Result.scalars()` *方法的* `Result`

返回一个 `ScalarResult` 过滤对象，该对象将返回单个元素而不是 `Row` 对象。

例如：

```py
>>> result = conn.execute(text("select int_id from table"))
>>> result.scalars().all()
[1, 2, 3]
```

当结果从 `ScalarResult` 过滤对象中提取时，将返回 `Result` 将返回的单列行作为列的值。

1.4 版本新增。

参数：

**index** – 表示要从每行提取的列的整数或行键，默认为 `0` 表示第一列。

返回：

一个新的 `ScalarResult` 过滤对象，指向此 `Result` 对象。

```py
method splice_horizontally(other)
```

返回一个新的`CursorResult`，“水平拼接”这个`CursorResult`的行与另一个`CursorResult`的行。

提示

此方法用于 SQLAlchemy ORM 的利益，并不适用于一般用途。

“水平拼接”意味着对于第一个和第二个结果集中的每一行，都会产生一个新行，该行将两行连接在一起，然后成为新行。传入的`CursorResult`必须具有相同数量的行。通常期望两个结果集也来自相同的排序顺序，因为结果行是基于它们在结果中的位置进行拼接的。

预期的用例是，针对不同表的多个 INSERT..RETURNING 语句（肯定需要排序）可以产生一个看起来像这两个表的 JOIN 的单个结果。

例如：

```py
r1 = connection.execute(
    users.insert().returning(
        users.c.user_name,
        users.c.user_id,
        sort_by_parameter_order=True
    ),
    user_values
)

r2 = connection.execute(
    addresses.insert().returning(
        addresses.c.address_id,
        addresses.c.address,
        addresses.c.user_id,
        sort_by_parameter_order=True
    ),
    address_values
)

rows = r1.splice_horizontally(r2).all()
assert (
    rows ==
    [
        ("john", 1, 1, "foo@bar.com", 1),
        ("jack", 2, 2, "bar@bat.com", 2),
    ]
)
```

2.0 版中的新功能。

另请参阅

`CursorResult.splice_vertically()`

```py
method splice_vertically(other)
```

返回一个新的`CursorResult`，“垂直拼接”，即“扩展”，这个`CursorResult`的行与另一个`CursorResult`的行。

提示

此方法用于 SQLAlchemy ORM 的利益，并不适用于一般用途。

“垂直拼接”意味着给定结果的行附加到此游标结果的行。传入的`CursorResult`必须具有与此`CursorResult`中列的相同列表和相同顺序的行。

2.0 版中的新功能。

另请参阅

`CursorResult.splice_horizontally()`

```py
method supports_sane_multi_rowcount()
```

从方言返回`supports_sane_multi_rowcount`。

有关背景，请参阅`CursorResult.rowcount`。

```py
method supports_sane_rowcount()
```

从方言返回`supports_sane_rowcount`。

有关背景，请参阅`CursorResult.rowcount`。

```py
attribute t
```

*继承自* `Result` *的* `Result.t` *属性*

对返回的行应用“类型化元组”类型过滤器。

`Result.t` 属性是调用 `Result.tuples()` 方法的同义词。

新增于版本 2.0。

```py
method tuples() → TupleResult[_TP]
```

*继承自* `Result.tuples()` *方法的* `Result`

对返回的行应用“类型化元组”类型过滤器。

此方法在运行时返回相同的 `Result` 对象，但标注为返回 `TupleResult` 对象，该对象将指示[**PEP 484**](https://peps.python.org/pep-0484/)类型工具返回普通类型的 `Tuple` 实例，而不是行。这允许元组解包和对 `Row` 对象的 `__getitem__` 访问进行类型标注，用于语句本身包含类型信息的情况。

新增于版本 2.0。

返回：

在编写时查看 `TupleResult` 类型。

另请参阅

`Result.t` - 缩写同义词

`Row._t` - `Row` 版本

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

*继承自* `Result.unique()` *方法的* `Result`

对此 `Result` 返回的对象应用唯一过滤。

当不带参数应用此过滤器时，返回的行或对象将被过滤，使得每行唯一。用于确定此唯一性的算法默认为整个元组的 Python 哈希标识。在某些情况下，可能会使用专门针对每个实体的哈希方案，例如当使用 ORM 时，将应用一种针对返回对象的主键标识的方案。

唯一过滤器是在**所有其他过滤器之后**应用的，这意味着如果通过方法如 `Result.columns()` 或 `Result.scalars()` 对返回的列进行了细化，唯一性将仅应用于**返回的列或列**。这将发生在对 `Result` 对象调用这些方法的顺序无关紧要的情况下。

唯一过滤器还会更改像`Result.fetchmany()`和`Result.partitions()`这样的方法的计算。在使用`Result.unique()`时，这些方法将继续提供请求的行数或对象数，在应用唯一性后。然而，这必然会影响底层游标或数据源的缓冲行为，以便可能需要多次底层调用`cursor.fetchmany()`，以便累积足够的对象以提供所请求大小的唯一集合。

参数：

**策略** – 一个可应用于正在迭代的行或对象的可调用函数，它应返回表示行的唯一值的对象。Python 的`set()`用于存储这些标识。如果未传递，则使用默认的唯一性策略，该策略可能已由此`Result`对象的来源组装。

```py
method yield_per(num: int) → Self
```

将行提取策略配置为一次提取`num`行。

当在结果对象上进行迭代或以其他方式利用诸如`Result.fetchone()`这样一次返回一行的方法时，此参数会影响结果的基础行为。来自底层游标或其他数据源的数据将在内存中缓冲多达这么多行，然后缓冲集合将一次提供一行或根据请求提供尽可能多的行。每次缓冲清除时，它将刷新为这么多行或者如果剩余的行数较少，则刷新为剩余的行数。

`Result.yield_per()`方法通常与`Connection.execution_options.stream_results`执行选项一起使用，该选项允许使用的数据库方言利用服务器端游标，如果 DBAPI 支持与其默认操作模式不同的特定“服务器端游标”模式。

提示

考虑使用`Connection.execution_options.yield_per`执行选项，该选项将同时设置`Connection.execution_options.stream_results`以确保使用服务器端游标，并自动调用`Result.yield_per()`方法一次性建立固定的行缓冲大小。

`Connection.execution_options.yield_per` 执行选项适用于 ORM 操作，在 使用 Yield Per 获取大型结果集 中描述了面向 `Session` 使用的方法。与 `Connection` 兼容的仅核心版本是 SQLAlchemy 1.4.40 中的新功能。

新版本为 1.4。

参数：

**num** – 在重新填充缓冲区时每次提取的行数。如果设置为小于 1 的值，则提取下一个缓冲区的所有行。

另请参阅

使用服务器端游标（也称为流式结果） - 描述了 `Result.yield_per()` 的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
class sqlalchemy.engine.FilterResult
```

一个对 `Result` 的包装器，返回除 `Row` 对象以外的对象，例如字典或标量对象。

`FilterResult` 是额外结果 API 的通用基础，包括 `MappingResult`、`ScalarResult` 和 `AsyncResult`。

**成员**

close(), closed, yield_per()

**类签名**

类 `sqlalchemy.engine.FilterResult` (`sqlalchemy.engine.ResultInternal`)

```py
method close() → None
```

关闭此 `FilterResult`。

新版本为 1.4.43。

```py
attribute closed
```

如果底层 `Result` 报告已关闭，则返回 `True`。

新版本为 1.4.43。

```py
method yield_per(num: int) → Self
```

配置每次填充缓冲区时获取 `num` 行的行提取策略。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的传递。请参阅该方法的文档以获取使用说明。

新版本为 1.4.40：- 添加 `FilterResult.yield_per()`，以便在所有结果集实现中都可用此方法

另请参阅

使用服务器端游标（即流式结果） - 描述`Result.yield_per()`的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中的 ORM 查询指南中

```py
class sqlalchemy.engine.FrozenResult
```

表示一个适用于缓存的“冻结”状态的`Result`对象。

`Result.freeze()`方法返回`Result`对象的`FrozenResult`对象。

每次将`FrozenResult`作为可调用对象调用时，都会从固定数据集生成一个新的可迭代的`Result`对象：

```py
result = connection.execute(query)

frozen = result.freeze()

unfrozen_result_one = frozen()

for row in unfrozen_result_one:
    print(row)

unfrozen_result_two = frozen()
rows = unfrozen_result_two.all()

# ... etc
```

新版本 1.4 中新增。

另请参见

重新执行语句 - 在 ORM 中的示例用法来实现结果集缓存。

`merge_frozen_result()` - 将冻结的结果合并回`Session`的 ORM 函数。

**类签名**

类`sqlalchemy.engine.FrozenResult`（`typing.Generic`）

```py
class sqlalchemy.engine.IteratorResult
```

从 Python 迭代器或类似行数据的`Row`对象获取数据的`Result`。

新版本 1.4 中新增。

**成员**

关闭

**类签名**

类`sqlalchemy.engine.IteratorResult`（`sqlalchemy.engine.Result`）

```py
attribute closed
```

如果此`IteratorResult`已关闭，则返回`True`。

新版本 1.4.43 中新增。

```py
class sqlalchemy.engine.MergedResult
```

从任意数量的`Result`对象合并的`Result`。

由`Result.merge()`方法返回。

新版本 1.4 中新增。

**类签名**

类`sqlalchemy.engine.MergedResult`（`sqlalchemy.engine.IteratorResult`）

```py
class sqlalchemy.engine.Result
```

表示一组数据库结果。

版本 1.4 中的新内容：`Result`对象为 SQLAlchemy 核心和 SQLAlchemy ORM 提供了一个完全更新的使用模型和调用外观。在核心中，它构成了取代先前`ResultProxy`接口的`CursorResult`对象的基础。在使用 ORM 时，通常使用一个称为`ChunkedIteratorResult`的更高级对象。

注意

在 SQLAlchemy 1.4 及以上版本中，此对象用于由`Session.execute()`返回的 ORM 结果，可以逐个返回 ORM 映射对象的实例或在元组行内。请注意，与旧的`Query`对象相比，`Result`对象不会自动对实例或行进行去重。要在 Python 中对实例或行进行去重，请使用`Result.unique()`修改器方法。

另请参阅

获取行 - 在 SQLAlchemy 统一教程中

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), freeze(), keys(), mappings(), merge(), one(), one_or_none(), partitions(), scalar(), scalar_one(), scalar_one_or_none(), scalars(), t, tuples(), unique(), yield_per()

**类签名**

类`sqlalchemy.engine.Result` (`sqlalchemy.engine._WithKeys`, `sqlalchemy.engine.ResultInternal`)

```py
method all() → Sequence[Row[_TP]]
```

返回序列中的所有行。

在调用后关闭结果集。后续调用将返回一个空序列。

版本 1.4 中的新内容。

返回：

一系列`Row`对象。

另请参阅

使用服务器端游标（又名流式结果） - 如何在 Python 中流式传输大型结果集而不完全加载它。

```py
method close() → None
```

关闭此 `Result`。

此方法的行为是特定于实现的，并且默认情况下未实现。该方法通常应结束结果对象正在使用的资源，并且还应导致任何后续迭代或行获取引发 `ResourceClosedError`。

版本 1.4.27 中的新功能：- `.close()` 以前通常不适用于所有 `Result` 类，而是仅适用于返回给核心语句执行的 `CursorResult`。由于几乎所有其他结果对象，即 ORM 使用的结果对象，在任何情况下都在代理 `CursorResult`，因此这允许从外部外观关闭底层游标结果，以处理 ORM 查询使用了 `yield_per` 执行选项的情况，其中它不会立即耗尽和自动关闭数据库游标。

```py
attribute closed
```

如果此 `Result` 报告已关闭，则返回 `True`。

版本 1.4.43 中的新功能。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定应在每行中返回的列。

此方法可用于限制返回的列，以及重新排序它们。给定的表达式列表通常是一系列整数或字符串键名。它们也可以是适当的 `ColumnElement` 对象，这些对象对应于给定的语句构造。

自版本 2.0 更改：由于 1.4 版本中的一个错误，`Result.columns()` 方法存在错误行为，即仅使用一个索引调用该方法将导致 `Result` 对象产生标量值，而不是 `Row` 对象。在 2.0 版本中，已修正此行为，使得使用单个索引调用 `Result.columns()` 将产生一个继续生成 `Row` 对象的 `Result` 对象，该对象仅包含单个列。

例如：

```py
statement = select(table.c.x, table.c.y, table.c.z)
result = connection.execute(statement)

for z, y in result.columns('z', 'y'):
    # ...
```

从语句本身使用列对象的示例：

```py
for z, y in result.columns(
        statement.selected_columns.c.z,
        statement.selected_columns.c.y
):
    # ...
```

版本 1.4 中的新功能。

参数：

***col_expressions** – 表示要返回的列。元素可以是整数行索引、字符串列名，或者与选择构造对应的适当`ColumnElement`对象。

返回：

使用给定的修改项返回此`Result`对象。

```py
method fetchall() → Sequence[Row[_TP]]
```

`Result.all()`方法的同义词。

```py
method fetchmany(size: int | None = None) → Sequence[Row[_TP]]
```

获取多行。

当所有行都已耗尽时，返回一个空序列。

此方法是为了与 SQLAlchemy 1.x.x 向后兼容而提供的。

要以组的形式获取行，请使用`Result.partitions()`方法。

返回：

一个`Row`对象的序列。

另请参阅

`Result.partitions()`

```py
method fetchone() → Row[_TP] | None
```

获取一行。

当所有行都已耗尽时，返回 None。

此方法是为了与 SQLAlchemy 1.x.x 向后兼容而提供的。

要仅获取结果的第一行，请使用`Result.first()`方法。要遍历所有行，请直接迭代`Result`对象。

返回：

如果未应用过滤器，则返回`Row`对象，如果没有行剩余则返回`None`。

```py
method first() → Row[_TP] | None
```

获取第一行或者如果没有行存在则返回`None`。

关闭结果集并丢弃剩余的行。

注意

默认情况下，此方法返回一个**行**，例如元组。要返回确切的单个标量值，即第一行的第一列，请使用`Result.scalar()`方法，或者结合`Result.scalars()`和`Result.first()`。

此外，与传统的 ORM `Query.first()`方法的行为相比，**SQL 查询未应用任何限制**，以产生此`Result`; 对于在向 Python 进程发送行之前在内存中缓冲结果的 DBAPI 驱动程序，所有行都将发送到 Python 进程，除了第一行外所有行都将被丢弃。

另请参阅

ORM 查询与核心选择合并

返回：

`Row`对象，如果没有行剩余则返回 None。

另请参阅

`Result.scalar()`

`Result.one()`

```py
method freeze() → FrozenResult[_TP]
```

返回一个可调用对象，当调用时将产生此`Result`的副本。

返回的可调用对象是一个`FrozenResult`的实例。

这用于结果集缓存。当结果尚未被消耗时，必须在结果上调用该方法，并且调用该方法将完全消耗结果。当从缓存中检索到`FrozenResult`时，可以调用任意次数，它将每次产生一个新的`Result`对象，针对其存储的行集。

另请参阅

重新执行语句 - 在 ORM 中实现结果集缓存的示例用法。

```py
method keys() → RMKeyView
```

*从* `sqlalchemy.engine._WithKeys` *的* `sqlalchemy.engine._WithKeys.keys` *方法继承*

返回一个可迭代的视图，该视图会产生每个`Row`表示的字符串键。

键可以表示核心语句返回的列的标签，或者 ORM 执行返回的 orm 类的名称。

这个视图也可以使用 Python `in` 运算符进行键包含测试，该运算符将测试视图中表示的字符串键，以及列对象等备用键。

版本 1.4 中的更改：返回一个键视图对象，而不是普通列表。

```py
method mappings() → MappingResult
```

对返回的行应用映射过滤器，返回一个`MappingResult`的实例。

应用此过滤器时，获取行将返回`RowMapping`对象，而不是`Row`对象。

版本 1.4 中的新内容。

返回：

一个新的指向这个`Result`对象的`MappingResult`过滤对象。

```py
method merge(*others: Result[Any]) → MergedResult[_TP]
```

将此`Result`与其他兼容的结果对象合并。

返回的对象是一个`MergedResult`的实例，它将由给定结果对象的迭代器组成。

新的结果将使用此结果对象的元数据。后续结果对象必须针对相同的结果/游标元数据进行，否则行为未定义。

```py
method one() → Row[_TP]
```

返回一行或抛出异常。

如果结果没有返回任何行，则引发`NoResultFound`，如果返回多行，则引发`MultipleResultsFound`。

注意

此方法默认返回一个**行**，例如元组。要返回确切的单个标量值，即第一行的第一列，请使用`Result.scalar_one()`方法，或者结合使用`Result.scalars()`和`Result.one()`。

版本 1.4 中的新功能。

返回：

第一个`Row`。

引发：

`MultipleResultsFound`, `NoResultFound`

另请参阅

`Result.first()`

`Result.one_or_none()`

`Result.scalar_one()`

```py
method one_or_none() → Row[_TP] | None
```

返回最多一个结果或引发异常。

如果结果没有行，则返回`None`。如果返回多行，则引发`MultipleResultsFound`。

版本 1.4 中的新功能。

返回：

第一个`Row`或`None`（如果没有可用行）。

引发：

`MultipleResultsFound`

另请参阅

`Result.first()`

`Result.one()`

```py
method partitions(size: int | None = None) → Iterator[Sequence[Row[_TP]]]
```

遍历给定大小的子行列表。

每个列表将具有给定的大小，不包括最后一个将被产生的列表，该列表可能有少量行。不会产生空列表。

当迭代器完全消耗时，结果对象将自动关闭。

请注意，除非使用了`Connection.execution_options.stream_results`执行选项指示驱动程序不应预先缓冲结果，否则后端驱动程序通常会预先缓冲整个结果。并非所有驱动程序都支持此选项，对于不支持的驱动程序，该选项将被静默忽略。

在使用 ORM 时，当与 yield_per 执行选项结合使用时，`Result.partitions()`方法通常在内存方面更有效，该选项指示 DBAPI 驱动程序使用服务器端游标（如果可用），以及指示 ORM 加载内部一次只构建某些数量的 ORM 对象结果然后将其输出。

新版本中新增。

参数：

**size** – 指示每个生成的列表中应存在的最大行数。如果为 None，则使用 `Result.yield_per()` 设置的值（如果调用了该方法），或者 `Connection.execution_options.yield_per` 执行选项，在这方面是等效的。如果未设置 yield_per，则使用 `Result.fetchmany()` 的默认值，该值可能是特定于后端并且未明确定义的。

返回：

列表的迭代器

另请参阅

使用服务器端游标（也称为流式结果）

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
method scalar() → Any
```

获取第一行的第一列，并关闭结果集。

如果没有要提取的行，则返回`None`。

不执行任何验证以测试是否存在其他行。

调用此方法后，对象将完全关闭，例如 `CursorResult.close()` 方法将被调用。

返回：

一个 Python 标量值，或者如果没有剩余行，则为`None`。

```py
method scalar_one() → Any
```

返回确切的一个标量结果或引发异常。

这等同于调用 `Result.scalars()` 然后调用 `Result.one()`。

另请参阅

`Result.one()`

`Result.scalars()`

```py
method scalar_one_or_none() → Any | None
```

返回确切的一个标量结果或`None`。

这相当于调用 `Result.scalars()` 然后调用 `Result.one_or_none()`。

另请参阅

`Result.one_or_none()`

`Result.scalars()`

```py
method scalars(index: _KeyIndexType = 0) → ScalarResult[Any]
```

返回一个 `ScalarResult` 过滤对象，该对象将返回单个元素而不是 `Row` 对象。

例如：

```py
>>> result = conn.execute(text("select int_id from table"))
>>> result.scalars().all()
[1, 2, 3]
```

当从 `ScalarResult` 过滤对象中获取结果时，将返回由 `Result` 返回的单列行作为列的值。

新版本中新增。

参数：

**index** – 表示要从每行提取的列的整数或行键，默认为`0`表示第一列。

返回：

一个新的指向此`Result`对象的`ScalarResult`过滤对象。

```py
attribute t
```

对返回的行应用“类型化元组”类型筛选。

`Result.t`属性是调用`Result.tuples()`方法的同义词。

从版本 2.0 开始新增。

```py
method tuples() → TupleResult[_TP]
```

对返回的行应用“类型化元组”类型筛选。

此方法在运行时返回相同的`Result`对象，但会注释为返回一个`TupleResult`对象，这将指示[**PEP 484**](https://peps.python.org/pep-0484/)类型工具返回普通的类型化`Tuple`实例而不是行。这允许元组拆包和对`Row`对象的`__getitem__`访问进行类型化，对于那些语句本身包含了类型信息的情况。

从版本 2.0 开始新增。

返回：

在编写时引用`TupleResult`类型。

另见

`Result.t` - 更短的同义词

`Row._t` - `Row` 版本

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

对此`Result`返回的对象应用唯一过滤。

当此过滤器应用时没有参数时，返回的行或对象将被过滤，以确保每行都是唯一的。确定此唯一性的算法默认情况下是整个元组的 Python 哈希标识。在某些情况下，可能会使用专门的每实体哈希方案，例如当使用 ORM 时，将应用一种针对返回对象的主键标识的方案。

唯一过滤器应用在**所有其他过滤器之后**，这意味着如果返回的列已经通过诸如`Result.columns()`或`Result.scalars()`等方法进行了细化，则唯一性仅应用于**返回的列或列**。这发生在这些方法被调用在`Result`对象上的任何顺序之后。

唯一过滤器还改变了像`Result.fetchmany()`和`Result.partitions()`这样的方法所使用的计算方法。当使用`Result.unique()`时，这些方法将在应用唯一性后继续返回请求的行数或对象数。然而，这必然会影响底层游标或数据源的缓冲行为，因此可能需要多次底层调用`cursor.fetchmany()`以累积足够数量的对象，以便提供请求大小的唯一集合。

参数：

**strategy** – 一个可调用的函数，将应用于正在迭代的行或对象，它应返回一个表示行的唯一值的对象。一个 Python `set()` 用于存储这些标识。如果未传递，则使用默认的唯一性策略，该策略可能已由此 `Result` 对象的源代码组装。

```py
method yield_per(num: int) → Self
```

配置行提取策略以一次提取`num`行。

这会影响在迭代结果对象或者其他使用`Result.fetchone()`等返回一行的方法时的底层行为。来自底层游标或其他数据源的数据将被缓冲到内存中的这么多行，并且缓冲集合将以一行或者请求的行数的方式逐行返回。每次缓冲清空时，它将被刷新到这么多行，或者如果剩余行数更少则刷新为剩余行数。

`Result.yield_per()`方法通常与`Connection.execution_options.stream_results`执行选项一起使用，该选项允许使用中的数据库方言使用服务器端游标，如果 DBAPI 支持与默认操作模式分离的特定“服务器端游标”模式。

提示

考虑使用`Connection.execution_options.yield_per`执行选项，它将同时设置`Connection.execution_options.stream_results`以确保使用服务器端游标，同时自动调用`Result.yield_per()`方法一次性建立固定的行缓冲区大小。

`Connection.execution_options.yield_per` 执行选项适用于 ORM 操作，具有在 使用 Yield Per 获取大型结果集 中描述的针对 `Session` 的用法。仅适用于与 `Connection` 配合使用的 Core 版本是 SQLAlchemy 1.4.40 的新功能。

自版本 1.4 新增。

参数：

**num** – 每次重新填充缓冲区时获取的行数。如果设置为小于 1 的值，则获取下一个缓冲区的所有行。

另请参阅

使用服务器端游标（即流式结果） - 描述了 `Result.yield_per()` 的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
class sqlalchemy.engine.ScalarResult
```

一个 `Result` 的包装器，返回标量值而不是 `Row` 值。

`ScalarResult` 对象是通过调用 `Result.scalars()` 方法获取的。

`ScalarResult` 的一个特殊限制是它没有 `fetchone()` 方法；因为 `fetchone()` 的语义是 `None` 值表示没有更多结果，这与 `ScalarResult` 不兼容，因为无法区分 `None` 作为行值与 `None` 作为指示符的情况。使用 `next(result)` 逐个接收值。

**成员**

all(), close(), closed, fetchall(), fetchmany(), first(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类 `sqlalchemy.engine.ScalarResult` （`sqlalchemy.engine.FilterResult`）

```py
method all() → Sequence[_R]
```

返回所有标量值的序列。

等同于`Result.all()`，但返回的是标量值，而不是`Row`对象。

```py
method close() → None
```

*继承自* `FilterResult.close()` *方法的* `FilterResult`

关闭此`FilterResult`。

在 1.4.43 版本中新增。

```py
attribute closed
```

*继承自* `FilterResult.closed` *属性的* `FilterResult`

如果底层的`Result`报告已关闭，则返回`True`。

在 1.4.43 版本中新增。

```py
method fetchall() → Sequence[_R]
```

`ScalarResult.all()`方法的同义词。

```py
method fetchmany(size: int | None = None) → Sequence[_R]
```

获取多个对象。

等同于`Result.fetchmany()`，但返回的是标量值，而不是`Row`对象。

```py
method first() → _R | None
```

获取第一个对象，如果没有对象存在则返回`None`。

等同于`Result.first()`，但返回的是标量值，而不是`Row`对象。

```py
method one() → _R
```

返回一个对象，或者引发异常。

等同于`Result.one()`，但返回的是标量值，而不是`Row`对象。

```py
method one_or_none() → _R | None
```

返回最多一个对象，或者引发异常。

等同于`Result.one_or_none()`，但返回的是标量值，而不是`Row`对象。

```py
method partitions(size: int | None = None) → Iterator[Sequence[_R]]
```

迭代给定大小的子列表元素。

等同于`Result.partitions()`，但返回的是标量值，而不是`Row`对象。

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

对由此`ScalarResult`返回的对象应用唯一过滤。

有关使用详细信息，请参阅`Result.unique()`。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

配置行获取策略，一次获取`num`行。

`FilterResult.yield_per()`方法是对`Result.yield_per()`方法的传递。请参阅该方法的文档以获取使用说明。

版本 1.4.40 中的新增内容：- 添加了`FilterResult.yield_per()`，以便在所有结果集实现中都可用。

另请参阅

使用服务器端游标（即流式结果） - 描述了`Result.yield_per()`的核心行为。

使用每次产生的大型结果集 - 在 ORM 查询指南中。

```py
class sqlalchemy.engine.MappingResult
```

一个`Result`的包装器，返回的是字典值而不是`Row`的值。

调用`Result.mappings()`方法获取`MappingResult`对象。

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), keys(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类`sqlalchemy.engine.MappingResult` (`sqlalchemy.engine._WithKeys`, `sqlalchemy.engine.FilterResult`)

```py
method all() → Sequence[RowMapping]
```

在序列中返回所有标量值。

与`Result.all()`相当，只是返回的是`RowMapping`值，而不是`Row`对象。

```py
method close() → None
```

*继承自* `FilterResult.close()` *方法的* `FilterResult`。

关闭此`FilterResult`。

版本 1.4.43 中的新增内容。

```py
attribute closed
```

*继承自* `FilterResult.closed` *属性的* `FilterResult`

如果底层的`Result`报告已关闭，则返回`True`。

1.4.43 版本中的新功能。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定应在每行中返回的列。

```py
method fetchall() → Sequence[RowMapping]
```

`MappingResult.all()`方法的同义词。

```py
method fetchmany(size: int | None = None) → Sequence[RowMapping]
```

检索多个对象。

等同于`Result.fetchmany()`，除了返回`RowMapping`值，而不是`Row`对象。

```py
method fetchone() → RowMapping | None
```

检索一个对象。

等同于`Result.fetchone()`，除了返回`RowMapping`值，而不是`Row`对象。

```py
method first() → RowMapping | None
```

检索第一个对象或如果没有对象则返回`None`。

等同于`Result.first()`，除了返回`RowMapping`值，而不是`Row`对象。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys.keys` *方法的* `sqlalchemy.engine._WithKeys`

返回一个可迭代视图，该视图产生每个`Row`将表示的字符串键。

键可以表示核心语句返回的列的标签或 orm 执行返回的 orm 类的名称。

视图还可以使用 Python `in` 运算符进行键包含性测试，该运算符将测试视图中表示的字符串键以及列对象等替代键。

在 1.4 版本中更改：返回一个键视图对象而不是一个简单的列表。

```py
method one() → RowMapping
```

返回一个对象或引发异常。

等同于`Result.one()`，除了返回`RowMapping`值，而不是`Row`对象。

```py
method one_or_none() → RowMapping | None
```

返回至多一个对象或引发异常。

等同于`Result.one_or_none()`，除了返回`RowMapping`值，而不是`Row`对象。

```py
method partitions(size: int | None = None) → Iterator[Sequence[RowMapping]]
```

迭代给定大小的元素子列表。

与 `Result.partitions()` 相当，但返回的是 `RowMapping` 值，而不是 `Row` 对象。

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

将唯一过滤应用于此 `MappingResult` 返回的对象。

请参阅 `Result.unique()` 获取使用详情。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

将行获取策略配置为一次获取 `num` 行。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的简单封装。查看该方法的文档以获取使用说明。

版本 1.4.40 中新增：- 添加了 `FilterResult.yield_per()` 以便在所有结果集实现上都可用该方法

另请参阅

使用服务器端游标（即流式结果） - 描述 `Result.yield_per()` 的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
class sqlalchemy.engine.Row
```

表示单个结果行。

`Row` 对象表示数据库结果的一行。在 SQLAlchemy 1.x 系列中，它通常与 `CursorResult` 对象关联，但自 SQLAlchemy 1.4 起也被 ORM 用于类似元组的结果。

`Row` 对象致力于尽可能像 Python 命名元组一样行事。要获取行上的映射（即字典）行为，例如检查键的包含，请参阅 `Row._mapping` 属性。

另请参阅

使用 SELECT 语句 - 包括从 SELECT 语句中选择行的示例。

从 1.4 版本开始更改：将 `RowProxy` 重命名为 `Row`。 `Row` 不再是“代理”对象，因为它包含其中的最终数据形式，现在大部分像命名元组一样操作。类似映射的功能移到了 `Row._mapping` 属性。有关此更改的背景，请参阅 RowProxy 不再是“代理”；现在称为 Row 并像增强的命名元组一样运行。

**成员**

_asdict(), _fields, _mapping, _t, _tuple(), count, index, t, tuple()

**类签名**

类 `sqlalchemy.engine.Row` (`sqlalchemy.engine._py_row.BaseRow`, `collections.abc.Sequence`, `typing.Generic`)

```py
method _asdict() → Dict[str, Any]
```

返回一个新字典，将字段名映射到其相应的值。

此方法类似于 Python 命名元组的 `._asdict()` 方法，并通过将 `dict()` 构造函数应用于 `Row._mapping` 属性来工作。

自 1.4 版本新增。

另请参阅

`Row._mapping`

```py
attribute _fields
```

返回此 `Row` 所代表的字符串键的元组。

键可以表示核心语句返回的列的标签或 orm 执行返回的 orm 类的名称。

此属性类似于 Python 命名元组 `._fields` 属性。

自 1.4 版本新增。

另请参阅

`Row._mapping`

```py
attribute _mapping
```

返回此 `Row` 的 `RowMapping`。

此对象为行中包含的数据提供了一致的 Python 映射（即字典）接口。 单独的 `Row` 行为类似命名元组。

另请参阅

`Row._fields`

自 1.4 版本新增。

```py
attribute _t
```

`Row._tuple()` 的同义词。

从 2.0.19 版本新增：`Row._t` 属性取代了先前的 `Row.t` 属性，现在以下划线开头以避免与列名发生冲突，与 `Row` 上的其他命名元组方法一样。

另请参阅

`Result.t`

```py
method _tuple() → _TP
```

返回此 `Row` 的‘元组’形式。

在运行时，此方法返回“self”；`Row` 对象已经是一个命名元组。然而，在类型级别上，如果此 `Row` 被类型化，那么“元组”返回类型将是一个 [**PEP 484**](https://peps.python.org/pep-0484/) 的 `Tuple` 数据类型，其中包含有关各个元素的类型信息，支持类型化的解包和属性访问。

自 2.0.19 版新增：`Row._tuple()` 方法取代了以前的 `Row.tuple()` 方法，现在该方法已经被下划线标记，以避免与列名发生名称冲突，方式与其他命名元组方法在 `Row` 上一样。

另请参见

`Row._t` - 简写属性表示法

`Result.tuples()`

```py
attribute count
```

```py
attribute index
```

```py
attribute t
```

`Row._tuple()` 的同义词。

自 2.0.19 版起不推荐使用：`Row.t` 属性已被废弃，建议使用 `Row._t`；所有 `Row` 方法和库级属性都应以下划线开头，以避免名称冲突。请使用 `Row._t`。

自 2.0 版本新增。

```py
method tuple() → _TP
```

返回此 `Row` 的‘元组’形式。

自 2.0.19 版起不推荐使用：`Row.tuple()` 方法已被废弃，建议使用 `Row._tuple()`；所有 `Row` 方法和库级属性都应以下划线开头，以避免名称冲突。请使用 `Row._tuple()`。

自 2.0 版本新增。

```py
class sqlalchemy.engine.RowMapping
```

将列名和对象映射到 `Row` 值的映射。

`RowMapping` 可以通过 `Row` 的 `Row._mapping` 属性获得，也可以通过 `Result.mappings()` 方法返回的 `MappingResult` 对象提供的可迭代接口获得。

`RowMapping` 提供了对行内容的 Python 映射（即字典）访问。这包括支持测试特定键（字符串列名或对象）的包含性，以及对键、值和项的迭代：

```py
for row in result:
    if 'a' in row._mapping:
        print("Column 'a': %s" % row._mapping['a'])

    print("Column b: %s" % row._mapping[table.c.b])
```

新版本 1.4 中：`RowMapping` 对象取代了以前由数据库结果行提供的类似映射的访问，现在它主要表现得像一个命名元组。

**成员**

items(), keys(), values()

**类签名**

类`sqlalchemy.engine.RowMapping` (`sqlalchemy.engine._py_row.BaseRow`, `collections.abc.Mapping`, `typing.Generic`)

```py
method items() → ROMappingItemsView
```

返回底层`Row`中元素的键/值元组视图。

```py
method keys() → RMKeyView
```

返回底层`Row`中表示的字符串列名的‘keys’视图。

```py
method values() → ROMappingKeysValuesView
```

返回底层`Row`中表示的值的视图。

```py
class sqlalchemy.engine.TupleResult
```

一个`Result`，其类型为返回普通 Python 元组而不是行。

由于`Row`在任何方面都像一个元组，因��这个类只是一个类型类，运行时仍然使用常规`Result`。

**类签名**

类`sqlalchemy.engine.TupleResult`（`sqlalchemy.engine.FilterResult`, `sqlalchemy.util.langhelpers.TypingOnly`)

## 基本用法

从 Engine Configuration 中回想，`Engine` 是通过`create_engine()`调用创建的：

```py
engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test")
```

`create_engine()` 的典型用法是针对特定数据库 URL 每次一次，全局保存在单个应用程序进程的生命周期中。一个`Engine` 代表进程上的许多个体 DBAPI 连接，并且旨在以并发方式调用。`Engine` **不**等同于 DBAPI `connect()` 函数，后者仅表示一个连接资源 - 当在应用程序的模块级别创建一次时，`Engine` 在效率上最高，而不是每个对象或每个函数调用。

`Engine` 最基本的功能是提供对 `Connection` 的访问，然后可以调用 SQL 语句。向数据库发出文本语句如下所示：

```py
from sqlalchemy import text

with engine.connect() as connection:
    result = connection.execute(text("select username from users"))
    for row in result:
        print("username:", row.username)
```

在上述中，`Engine.connect()` 方法返回一个 `Connection` 对象，通过在 Python 上下文管理器中使用它（例如 `with:` 语句），`Connection.close()` 方法会在块结束时自动调用。`Connection` 是一个实际的 DBAPI 连接的 **代理** 对象。DBAPI 连接是在创建 `Connection` 时从连接池中检索的。

返回的对象称为 `CursorResult`，它引用了一个 DBAPI 游标，并提供了类似于 DBAPI 游标的获取行的方法。当所有结果行（如果有）都耗尽时，`CursorResult` 将关闭 DBAPI 游标。一个不返回行的 `CursorResult`，例如没有返回任何行的 UPDATE 语句，立即在构造时释放游标资源。

当在 `with:` 块的末尾关闭 `Connection` 时，引用的 DBAPI 连接被释放到连接池中。从数据库本身的角度来看，假设连接池有空间存储该连接以供下次使用，连接池实际上不会“关闭”连接。当将连接返回给连接池以供重用时，池化机制会在 DBAPI 连接上发出 `rollback()` 调用，以便删除任何事务状态或锁定（这称为 Reset On Return），并且连接准备好供下次使用。

上面的示例说明了执行文本 SQL 字符串，应该使用 `text()` 构造来指示我们想要使用文本 SQL。`Connection.execute()` 方法当然可以容纳更多内容；请参阅 Working with Data 在 SQLAlchemy 统一教程 中进行教程。

## 使用事务

注意

本节描述了在直接使用`Engine`和`Connection`对象时如何使用事务。当使用 SQLAlchemy ORM 时，事务控制的公共 API 是通过`Session`对象，该对象在内部使用`Transaction`对象。有关更多信息，请参阅管理事务。

### 按需提交

`Connection`对象始终在事务块的上下文中发出 SQL 语句。第一次调用`Connection.execute()`方法来执行 SQL 语句时，此事务会自动开始，使用一种称为**autobegin**的行为。事务在`Connection`对象的范围内保持不变，直到调用`Connection.commit()`或`Connection.rollback()`方法。在事务结束后，`Connection`等待再次调用`Connection.execute()`方法，此时它会自动重新开始。

此调用风格称为**按需提交**，如下面的示例所示：

```py
with engine.connect() as connection:
    connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

    connection.commit()  # commit the transaction
```

在“按需提交”的风格中，我们可以在使用`Connection.execute()`发出的一系列其他语句中自由地调用`Connection.commit()`和`Connection.rollback()`方法；每次事务结束，并发出新语句时，都会隐式开始一个新的事务：

```py
with engine.connect() as connection:
    connection.execute(text("<some statement>"))
    connection.commit()  # commits "some statement"

    # new transaction starts
    connection.execute(text("<some other statement>"))
    connection.rollback()  # rolls back "some other statement"

    # new transaction starts
    connection.execute(text("<a third statement>"))
    connection.commit()  # commits "a third statement"
```

2.0 版本中的新功能：“按需提交”风格是 SQLAlchemy 2.0 的新功能。当使用“future”风格引擎时，它也可用于 SQLAlchemy 1.4 的“过渡”模式中。

### 只需开始一次

`Connection`对象提供了一种更明确的事务管理样式，称为**仅开始一次**。与“按需提交”相比，“仅开始一次”允许显式声明事务的起始点，并允许将事务本身构建为上下文管理器块，以便事务的结束变得隐式。要使用“仅开始一次”，使用`Connection.begin()`方法，它返回一个代表 DBAPI 事务的`Transaction`对象。该对象还支持通过其自己的`Transaction.commit()`和`Transaction.rollback()`方法进行显式管理，但作为首选做法，还支持上下文管理器接口，其中当块正常结束时，它将自行提交，并在引发异常时发出回滚，然后将异常传播到外部。下面说明了“仅开始一次”块的形式：

```py
with engine.connect() as connection:
    with connection.begin():
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

    # transaction is committed
```

### 从引擎连接和开始一次

上述“仅开始一次”块的便捷简写形式是在起始`Engine`对象的级别上使用`Engine.begin()`方法，而不是执行`Engine.connect()`和`Connection.begin()`这两个单独的步骤；`Engine.begin()`方法返回一个特殊的上下文管理器，内部同时维护`Connection`的上下文管理器以及通常由`Connection.begin()`方法返回的`Transaction`的上下文管理器：

```py
with engine.begin() as connection:
    connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

# transaction is committed, and Connection is released to the connection
# pool
```

提示

在`Engine.begin()`块中，我们可以调用`Connection.commit()`或`Connection.rollback()`方法，这将提前结束由该块正常标记的事务。但是，如果我们这样做，就不能在`Connection`上进一步发出 SQL 操作，直到块结束为止：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo=True)
>>> with e.begin() as conn:
...     conn.commit()
...     conn.begin()
2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine COMMIT
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: Can't operate on closed transaction inside
context manager.  Please complete the context manager before emitting
further commands.
```

### 混合样式

“一次性开始”和“边执行边提交”样式可以自由混合在单个 `Engine.connect()` 块中，只要对 `Connection.begin()` 的调用不与“自动开始”行为冲突。为了实现这一点，`Connection.begin()` 应该在发出任何 SQL 语句之前或在直接调用之后调用 `Connection.commit()` 或 `Connection.rollback()`：

```py
with engine.connect() as connection:
    with connection.begin():
        # run statements in a "begin once" block
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})

    # transaction is committed

    # run a new statement outside of a block. The connection
    # autobegins
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

    # commit explicitly
    connection.commit()

    # can use a "begin once" block here
    with connection.begin():
        # run more statements
        connection.execute(...)
```

当开发使用“一次性开始”的代码时，如果事务已经“自动开始”，库将引发 `InvalidRequestError`。

### 边执行边提交

`Connection` 对象总是在事务块的上下文中发出 SQL 语句。当第一次调用 `Connection.execute()` 方法执行 SQL 语句时，将自动开始此事务，使用的是称为**自动开始**的行为。事务将一直保持到 `Connection` 对象的范围内，直到调用 `Connection.commit()` 或 `Connection.rollback()` 方法。在事务结束后，`Connection` 等待再次调用 `Connection.execute()` 方法，此时将再次自动开始。

这种调用方式称为**边执行边提交**，下面的示例中有所说明：

```py
with engine.connect() as connection:
    connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

    connection.commit()  # commit the transaction
```

在“边执行边提交”样式中，我们可以随时调用 `Connection.commit()` 和 `Connection.rollback()` 方法，在使用 `Connection.execute()` 发出的其他语句序列中；每次事务结束，并发出新的语句时，都会隐式开始一个新的事务：

```py
with engine.connect() as connection:
    connection.execute(text("<some statement>"))
    connection.commit()  # commits "some statement"

    # new transaction starts
    connection.execute(text("<some other statement>"))
    connection.rollback()  # rolls back "some other statement"

    # new transaction starts
    connection.execute(text("<a third statement>"))
    connection.commit()  # commits "a third statement"
```

2.0 版本新增：“边执行边提交”样式是 SQLAlchemy 2.0 的新功能。当使用“未来”样式引擎时，它也可在 SQLAlchemy 1.4 的“过渡”模式中使用。

### 一次性开始

`Connection`对象提供了一种更明确的事务管理样式，称为**begin once**。与“随着操作进行提交”相比，“begin once”允许明确指定事务的起始点，并允许将事务本身构建为上下文管理器块，以便事务的结束是隐式的。要使用“begin once”，使用`Connection.begin()`方法，该方法返回一个表示 DBAPI 事务的`Transaction`对象。该对象还支持通过自己的`Transaction.commit()`和`Transaction.rollback()`方法的显式管理，但作为首选做法，还支持上下文管理器接口，在块正常结束时将自动提交，并在引发异常时发出回滚，然后将异常传播出去。下面示例说明了“begin once”块的形式：

```py
with engine.connect() as connection:
    with connection.begin():
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
        connection.execute(
            some_other_table.insert(), {"q": 8, "p": "this is some more data"}
        )

    # transaction is committed
```

### 从引擎连接和开始一次

对于上述“begin once”块的一个便捷的缩写形式是在源`Engine`对象的级别上使用`Engine.begin()`方法，而不是执行两个分开的步骤`Engine.connect()`和`Connection.begin()`；`Engine.begin()`方法返回一个特殊的上下文管理器，内部同时维护`Connection`的上下文管理器以及通常由`Connection.begin()`方法返回的`Transaction`的上下文管理器：

```py
with engine.begin() as connection:
    connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

# transaction is committed, and Connection is released to the connection
# pool
```

提示

在`Engine.begin()`块中，我们可以调用`Connection.commit()`或`Connection.rollback()`方法，这将提前结束由块正常标记的事务。但是，如果我们这样做，直到块结束之前，`Connection`上不会再发出任何 SQL 操作：

```py
>>> from sqlalchemy import create_engine
>>> e = create_engine("sqlite://", echo=True)
>>> with e.begin() as conn:
...     conn.commit()
...     conn.begin()
2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2021-11-08 09:49:07,517 INFO sqlalchemy.engine.Engine COMMIT
Traceback (most recent call last):
...
sqlalchemy.exc.InvalidRequestError: Can't operate on closed transaction inside
context manager.  Please complete the context manager before emitting
further commands.
```

### 混合样式

“随行提交”和“一次开始”样式可以在单个 `Engine.connect()` 块内自由混合使用，只要对 `Connection.begin()` 的调用不与“自动开始”行为冲突即可。为了实现这一点，`Connection.begin()` 应该在发出任何 SQL 语句之前或直接在先前对 `Connection.commit()` 或 `Connection.rollback()` 的调用之后被调用：

```py
with engine.connect() as connection:
    with connection.begin():
        # run statements in a "begin once" block
        connection.execute(some_table.insert(), {"x": 7, "y": "this is some data"})

    # transaction is committed

    # run a new statement outside of a block. The connection
    # autobegins
    connection.execute(
        some_other_table.insert(), {"q": 8, "p": "this is some more data"}
    )

    # commit explicitly
    connection.commit()

    # can use a "begin once" block here
    with connection.begin():
        # run more statements
        connection.execute(...)
```

在开发使用“一次开始”的代码时，如果事务已经“自动开始”，库将引发 `InvalidRequestError`。

## 设置事务隔离级别，包括 DBAPI 自动提交

大多数 DBAPI 支持可配置的事务隔离级别的概念。这些传统上是四个级别，“READ UNCOMMITTED”、“READ COMMITTED”、“REPEATABLE READ” 和 “SERIALIZABLE”。这些通常在 DBAPI 连接开始新事务之前应用，需要注意的是，当首次发出 SQL 语句时，大多数 DBAPI 会隐式开始此事务。

支持隔离级别的 DBAPI 通常也支持真正的“自动提交”概念，这意味着 DBAPI 连接本身将被放置在非事务性自动提交模式中。这通常意味着数据库自动发出“BEGIN”的典型 DBAPI 行为不再发生，但也可能包括其他指令。SQLAlchemy 将“自动提交”的概念视为任何其他隔离级别；因为它是一个不仅失去“读已提交”而且失去原子性的隔离级别。

提示

需要注意的是，如下面部分将在 理解 DBAPI 级别的自动提交隔离级别 中进一步讨论的那样，“自动提交”隔离级别像任何其他隔离级别一样**不会**影响 `Connection` 对象的“事务”行为，该对象继续调用 DBAPI 的 `.commit()` 和 `.rollback()` 方法（它们在自动提交模式下没有效果），并且 `.begin()` 方法假定 DBAPI 将隐式启动事务（这意味着 SQLAlchemy 的“begin”**不会更改自动提交模式**）。

SQLAlchemy 方言应尽可能支持这些隔离级别以及自动提交。

### 设置连接的隔离级别或 DBAPI 自动提交

对于从 `Engine.connect()` 获得的每个单独的 `Connection` 对象，可以使用 `Connection.execution_options()` 方法为该 `Connection` 对象设置隔离级别。该参数被称为 `Connection.execution_options.isolation_level`，其值通常为以下名称的子集：

```py
# possible values for Connection.execution_options(isolation_level="<value>")

"AUTOCOMMIT"
"READ COMMITTED"
"READ UNCOMMITTED"
"REPEATABLE READ"
"SERIALIZABLE"
```

并非每个 DBAPI 都支持每个数值；如果在某个后端使用了不支持的值，将会引发错误。

例如，要强制在特定连接上使用 REPEATABLE READ，然后开始一个事务：

```py
with engine.connect().execution_options(
    isolation_level="REPEATABLE READ"
) as connection:
    with connection.begin():
        connection.execute(text("<statement>"))
```

提示

`Connection.execution_options()` 方法的返回值是调用该方法的同一个 `Connection` 对象，这意味着它直接修改了该 `Connection` 对象的状态。这是 SQLAlchemy 2.0 新增的行为。这个行为不适用于 `Engine.execution_options()` 方法；该方法仍然返回一个 `Engine` 的副本，并且如下所述，可以用于构建具有不同执行选项的多个 `Engine` 对象，但它们仍然共享相同的方言和连接池。

注意

`Connection.execution_options.isolation_level` 参数不适用于语句级别选项，例如 `Executable.execution_options()` 的选项，并且如果在此级别设置，将被拒绝。这是因为该选项必须在每个事务基础上针对 DBAPI 连接进行设置。

### 设置引擎的隔离级别或 DBAPI 自动提交

`Connection.execution_options.isolation_level` 选项也可以设置为引擎范围内，通常更可取。这可以通过将 `create_engine.isolation_level` 参数传递给 `create_engine()` 来实现：

```py
from sqlalchemy import create_engine

eng = create_engine(
    "postgresql://scott:tiger@localhost/test", isolation_level="REPEATABLE READ"
)
```

使用以上设置，每个新的 DBAPI 连接在创建时将被设置为在所有后续操作中使用`"REPEATABLE READ"`隔离级别设置。

### 维护单个引擎的多个隔离级别

隔离级别也可以针对每个引擎进行设置，使用`create_engine.execution_options`参数或`Engine.execution_options()`方法，后者将创建一个`Engine`的副本，该副本共享方言和连接池，但具有自己的每个连接的隔离级别设置：

```py
from sqlalchemy import create_engine

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    execution_options={"isolation_level": "REPEATABLE READ"},
)
```

使用以上设置，DBAPI 连接将被设置为在每个新的事务开始时使用`"REPEATABLE READ"`隔离级别设置；但是，作为池化的连接将被重置为在连接首次发生时存在的原始隔离级别。在`create_engine()`的级别上，最终效果与使用`create_engine.isolation_level`参数没有任何不同。

然而，经常选择在不同隔离级别中运行操作的应用程序可能希望创建多个“子引擎”来使用一个主`Engine`，其中每个引擎将配置为不同的隔离级别。一个这样的用例是一个应用程序，它有分为“事务性”和“只读”操作的操作，一个单独的`Engine`，使用`"AUTOCOMMIT"`可能被分离出来，从主引擎中分离出来：

```py
from sqlalchemy import create_engine

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")
```

上面，`Engine.execution_options()`方法创建了原始`Engine`的浅层副本。`eng`和`autocommit_engine`共享相同的方言和连接池。然而，“AUTOCOMMIT”模式将在从`autocommit_engine`获取连接时设置。

无论设置的隔离级别是什么，在连接返回到连接池时都会无条件地恢复。

另见

SQLite 事务隔离

PostgreSQL 事务隔离

MySQL 事务隔离

SQL Server 事务隔离

Oracle 事务隔离

设置事务隔离级别 / DBAPI 自动提交 - 用于 ORM

使用 DBAPI 自动提交允许只读版本的透明重连接 - 一种利用 DBAPI 自动提交来对数据库进行透明重连接以进行只读操作的方法  ### 理解 DBAPI 级别的自动提交隔离级别

在上一部分中，我们介绍了 `Connection.execution_options.isolation_level` 参数的概念以及如何使用它来设置数据库隔离级别，包括 SQLAlchemy 将其视为另一个事务隔离级别的 DBAPI 级别的“自动提交”。在本节中，我们将试图澄清这种方法的含义。

如果我们想要检查一个 `Connection` 对象并使用它的“自动提交”模式，我们将按如下方式进行：

```py
with engine.connect() as connection:
    connection.execution_options(isolation_level="AUTOCOMMIT")
    connection.execute(text("<statement>"))
    connection.execute(text("<statement>"))
```

上面说明了“DBAPI 自动提交”模式的正常使用。无需使用 `Connection.begin()` 或 `Connection.commit()` 等方法，因为所有语句都会立即提交到数据库。当块结束时，`Connection` 对象将恢复“自动提交”隔离级别，并且 DBAPI 连接将释放到连接池，其中 DBAPI 的 `connection.rollback()` 方法通常会被调用，但是由于上面的语句已经提交了，这个回滚对数据库状态没有任何改变。

需要注意的是，“自动提交”模式甚至在调用 `Connection.begin()` 方法时也会持续存在；DBAPI 不会向数据库发送任何 BEGIN，也不会在调用 `Connection.commit()` 时发送 COMMIT。这种用法也不是错误的情景，因为可以预期“自动提交”隔离级别可能被应用于原本假设处于事务上下文的代码；毕竟，“隔离级别”本身就是事务的配置细节，就像任何其他隔离级别一样。

在下面的示例中，无论是什么样的 SQLAlchemy 级别的事务块，语句都保持 **自动提交**：

```py
with engine.connect() as connection:
    connection = connection.execution_options(isolation_level="AUTOCOMMIT")

    # this begin() does not affect the DBAPI connection, isolation stays at AUTOCOMMIT
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
        connection.execute(text("<statement>"))
```

当我们像上面的示例一样运行一个带有日志记录的代码块时，日志记录将尝试指示，尽管调用了 DBAPI 级别的 `.commit()`，但由于自动提交模式，它可能没有任何效果：

```py
INFO sqlalchemy.engine.Engine BEGIN (implicit)
...
INFO sqlalchemy.engine.Engine COMMIT using DBAPI connection.commit(), DBAPI should ignore due to autocommit mode
```

同时，尽管我们正在使用“DBAPI autocommit”，但 SQLAlchemy 的事务语义，即`Connection.begin()`的 Python 内部行为以及“autobegin”的行为，**仍然存在，尽管这些不会影响 DBAPI 连接本身**。为了说明，下面的代码将引发错误，因为在自动开始已经发生后调用了`Connection.begin()`：

```py
with engine.connect() as connection:
    connection = connection.execution_options(isolation_level="AUTOCOMMIT")

    # "transaction" is autobegin (but has no effect due to autocommit)
    connection.execute(text("<statement>"))

    # this will raise; "transaction" is already begun
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
```

上述示例还展示了“autocommit”隔离级别是底层数据库事务的配置细节，并且独立于 SQLAlchemy 连接对象的开始/提交行为。 “autocommit”模式不会以任何方式与`Connection.begin()`交互，并且在执行有关事务的自身状态更改时（除了在引擎日志中建议这些块实际上并没有提交之外），`Connection`不会查询此状态。这种设计的理念是保持与`Connection`完全一致的使用模式，其中 DBAPI 自动提交模式可以独立更改，而无需指示其他地方的任何代码更改。

#### 在不同隔离级别之间切换

隔离级别设置，包括自动提交模式，在连接释放回连接池时会自动重置。因此，最好避免尝试在单个`Connection`对象上切换隔离级别，因为这会导致冗余性过高。

为了演示如何在单个`Connection`检出的范围内以临时模式使用“autocommit”，必须重新应用`Connection.execution_options.isolation_level`参数以恢复先前的隔离级别。前一节说明了在进行自动提交时尝试调用`Connection.begin()`来启动事务的尝试；我们可以通过在调用`Connection.begin()`之前先恢复隔离级别来重写该示例：

```py
# if we wanted to flip autocommit on and off on a single connection/
# which... we usually don't.

with engine.connect() as connection:
    connection.execution_options(isolation_level="AUTOCOMMIT")

    # run statement(s) in autocommit mode
    connection.execute(text("<statement>"))

    # "commit" the autobegun "transaction"
    connection.commit()

    # switch to default isolation level
    connection.execution_options(isolation_level=connection.default_isolation_level)

    # use a begin block
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
```

上面，为了手动恢复隔离级别，我们利用了`Connection.default_isolation_level`来恢复默认的隔离级别（假设这是我们想要的）。然而，更好的做法可能是使用`Connection`的架构，该架构已经在检入时自动处理重置隔离级别。编写上述内容的**首选**方式是使用两个块。

```py
# use an autocommit block
with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as connection:
    # run statement in autocommit mode
    connection.execute(text("<statement>"))

# use a regular block
with engine.begin() as connection:
    connection.execute(text("<statement>"))
```

总结一下：

1.  “DBAPI 级别自动提交”隔离级别完全独立于`Connection`对象对“开始”和“提交”的概念。

1.  使用单独的`Connection`检出每个隔离级别。避免在单个连接检出之间试图来回切换“自动提交”；让引擎来恢复默认的隔离级别。

### 设置连接的隔离级别或 DBAPI 自动提交

对于从`Engine.connect()`获取的单个`Connection`对象，可以使用`Connection.execution_options()`方法来设置该`Connection`对象的隔离级别。参数称为`Connection.execution_options.isolation_level`，其值通常是以下名称的子集：

```py
# possible values for Connection.execution_options(isolation_level="<value>")

"AUTOCOMMIT"
"READ COMMITTED"
"READ UNCOMMITTED"
"REPEATABLE READ"
"SERIALIZABLE"
```

并非每个 DBAPI 都支持每个值；如果对于某个后端使用了不支持的值，则会引发错误。

例如，要在特定连接上强制使用**可重复读**，然后开始一个事务：

```py
with engine.connect().execution_options(
    isolation_level="REPEATABLE READ"
) as connection:
    with connection.begin():
        connection.execute(text("<statement>"))
```

提示

`Connection.execution_options()` 方法的返回值是调用该方法的相同 `Connection` 对象，这意味着它会直接修改 `Connection` 对象的状态。这是 SQLAlchemy 2.0 的新行为。这种行为不适用于 `Engine.execution_options()` 方法；该方法仍然返回一个 `Engine` 的副本，并且如下所述，可以用于构建具有不同执行选项的多个 `Engine` 对象，但它们仍然共享相同的方言和连接池。

注意

`Connection.execution_options.isolation_level` 参数不适用于语句级别选项，例如 `Executable.execution_options()`，如果在此级别设置将会被拒绝。这是因为该选项必须在每个事务的 DBAPI 连接上设置。

### 设置引擎的隔离级别或 DBAPI 自动提交

`Connection.execution_options.isolation_level` 选项也可以设置为整个引擎范围内，通常更可取。这可以通过将 `create_engine.isolation_level` 参数传递给 `create_engine()` 来实现：

```py
from sqlalchemy import create_engine

eng = create_engine(
    "postgresql://scott:tiger@localhost/test", isolation_level="REPEATABLE READ"
)
```

使用上述设置，每个新的 DBAPI 连接在创建时将被设置为对所有后续操作使用 `"REPEATABLE READ"` 隔离级别设置。

### 为单个引擎维护多个隔离级别

隔离级别也可以针对每个引擎进行设置，具有更大的灵活性，可以使用 `create_engine.execution_options` 参数传递给 `create_engine()` 或 `Engine.execution_options()` 方法，后者将创建一个与原始引擎共享方言和连接池但具有自己的每个连接隔离级别设置的 `Engine` 的副本：

```py
from sqlalchemy import create_engine

eng = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test",
    execution_options={"isolation_level": "REPEATABLE READ"},
)
```

通过上述设置，每开始一个新事务时，DBAPI 连接将设置为使用 `"REPEATABLE READ"` 隔离级别设置；但是连接在被池化时将被重置为首次发生连接时存在的原始隔离级别。在 `create_engine()` 的级别上，最终效果与使用 `create_engine.isolation_level` 参数没有任何不同。

然而，频繁选择在不同隔离级别内运行操作的应用程序可能希望创建多个 "子引擎" 以及一个主 `Engine`，每个引擎都配置为不同的隔离级别。一个这样的用例是一个具有分为 "事务性" 和 "只读" 操作的应用程序，可以从主引擎中分离出一个单独的 `Engine`，该引擎使用 `"AUTOCOMMIT"`：

```py
from sqlalchemy import create_engine

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")
```

上述，`Engine.execution_options()` 方法创建原 `Engine` 的一个浅拷贝。`eng` 和 `autocommit_engine` 共享相同的方言和连接池。然而，当从 `autocommit_engine` 获取连接时，将设置 "AUTOCOMMIT" 模式。

无论隔离级别设置是哪一个，在连接返回到连接池时都会无条件地恢复。

另请参见

SQLite 事务隔离

PostgreSQL 事务隔离

MySQL 事务隔离

SQL Server 事务隔离

Oracle 事务隔离

设置事务隔离级别 / DBAPI AUTOCOMMIT - 用于 ORM

使用 DBAPI 自动提交允许用于只读版本的透明重新连接 - 一个使用 DBAPI 自动提交来透明地重新连接到数据库进行只读操作的示例

### 理解 DBAPI 级别的自动提交隔离级别

在上一节中，我们介绍了`Connection.execution_options.isolation_level`参数的概念以及如何使用它来设置数据库隔离级别，包括由 SQLAlchemy 视为另一种事务隔离级别的 DBAPI 级别“自动提交”。在本节中，我们将尝试澄清这种方法的含义。

如果我们想要检查一个`Connection`对象并将其用于“自动提交”模式，我们将按如下步骤进行：

```py
with engine.connect() as connection:
    connection.execution_options(isolation_level="AUTOCOMMIT")
    connection.execute(text("<statement>"))
    connection.execute(text("<statement>"))
```

上面说明了“DBAPI 自动提交”模式的正常用法。无需使用诸如`Connection.begin()`或`Connection.commit()`等方法，因为所有语句都会立即提交到数据库。当块结束时，`Connection`对象将恢复“自动提交”隔离级别，并且 DBAPI 连接被释放到连接池，其中 DBAPI `connection.rollback()`方法通常会被调用，但由于上述语句已经提交，此回滚对数据库状态没有任何更改。

需要注意的是，“自动提交”模式在调用`Connection.begin()`方法时仍然持续存在；DBAPI 不会向数据库发出任何 BEGIN，也不会在调用`Connection.commit()`时发出 COMMIT。这种用法也不是错误场景，因为预期可能会将“自动提交”隔离级别应用于原本假定了事务上下文的代码；毕竟，“隔离级别”本身就像任何其他隔离级别一样，是事务本身的配置细节。

在下面的示例中，语句仍然**自动提交**，而不管 SQLAlchemy 级别的事务块如何：

```py
with engine.connect() as connection:
    connection = connection.execution_options(isolation_level="AUTOCOMMIT")

    # this begin() does not affect the DBAPI connection, isolation stays at AUTOCOMMIT
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
        connection.execute(text("<statement>"))
```

当我们像上面那样带有日志记录的运行一个块时，日志记录将尝试指出，虽然调用了 DBAPI 级别的`.commit()`，但由于自动提交模式，它可能不会产生任何效果：

```py
INFO sqlalchemy.engine.Engine BEGIN (implicit)
...
INFO sqlalchemy.engine.Engine COMMIT using DBAPI connection.commit(), DBAPI should ignore due to autocommit mode
```

与此同时，即使我们使用了“DBAPI 自动提交”，SQLAlchemy 的事务语义，即`Connection.begin()`的 Python 内部行为以及“autobegin”的行为**仍然存在，即使这些行为不影响 DBAPI 连接本身**。为了说明，下面的代码将引发错误，因为在自动开始已经发生后调用了`Connection.begin()`：

```py
with engine.connect() as connection:
    connection = connection.execution_options(isolation_level="AUTOCOMMIT")

    # "transaction" is autobegin (but has no effect due to autocommit)
    connection.execute(text("<statement>"))

    # this will raise; "transaction" is already begun
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
```

上面的例子还演示了相同的主题，即“自动提交”隔离级别是底层数据库事务的配置细节，与 SQLAlchemy 连接对象的 begin/commit 行为无关。 “自动提交”模式不会以任何方式与`Connection.begin()`交互，且`Connection`在执行自身与事务相关的状态更改时不会查询此状态（除了在引擎日志中建议这些块实际上并未提交之外）。这种设计的理由是保持与`Connection`完全一致的使用模式，其中 DBAPI 自动提交模式可以独立更改，而无需指示其他位置的代码更改。

#### 在不同隔离级别之间切换

隔离级别设置，包括自动提交模式，在连接释放回连接池时会自动重置。因此，最好避免尝试在单个`Connection`对象上切换隔离级别，因为这会导致冗余的繁琐。

为了说明如何在单个`Connection`检出的范围内以即时模式使用“自动提交”，必须重新应用`Connection.execution_options.isolation_level`参数以恢复先前的隔离级别。前一节演示了在自动提交进行时调用`Connection.begin()`以启动事务的尝试；我们可以通过在调用`Connection.begin()`之前先恢复隔离级别来重写该示例，以实际执行：

```py
# if we wanted to flip autocommit on and off on a single connection/
# which... we usually don't.

with engine.connect() as connection:
    connection.execution_options(isolation_level="AUTOCOMMIT")

    # run statement(s) in autocommit mode
    connection.execute(text("<statement>"))

    # "commit" the autobegun "transaction"
    connection.commit()

    # switch to default isolation level
    connection.execution_options(isolation_level=connection.default_isolation_level)

    # use a begin block
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
```

在上面的例子中，为了手动恢复隔离级别，我们使用`Connection.default_isolation_level`来恢复默认的隔离级别（假设这是我们想要的）。然而，最好的做法可能是利用`Connection`的架构，该架构已经在检查时自动处理了隔离级别的重置。上述写法的**首选**方式是使用两个块。

```py
# use an autocommit block
with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as connection:
    # run statement in autocommit mode
    connection.execute(text("<statement>"))

# use a regular block
with engine.begin() as connection:
    connection.execute(text("<statement>"))
```

总结一下：

1.  “DBAPI 级别的自动提交”隔离级别完全独立于`Connection`对象对“begin”和“commit”的概念。

1.  使用每个隔离级别的单独 `Connection` 检出。避免在单个连接检出上来回切换“自动提交”；让引擎来恢复默认的隔离级别。

#### 在不同隔离级别之间切换

隔离级别设置，包括自动提交模式，在连接释放回连接池时会自动重置。因此，最好避免尝试在单个 `Connection` 对象上切换隔离级别，因为这会导致过多的冗余。

为了说明如何在单个 `Connection` 检出的范围内以临时模式使用“自动提交”，必须重新应用 `Connection.execution_options.isolation_level` 参数以前的隔离级别。上一节演示了在自动提交进行时调用 `Connection.begin()` 来启动事务的尝试；我们可以通过在调用 `Connection.begin()` 之前首先恢复隔离级别来重写该示例：

```py
# if we wanted to flip autocommit on and off on a single connection/
# which... we usually don't.

with engine.connect() as connection:
    connection.execution_options(isolation_level="AUTOCOMMIT")

    # run statement(s) in autocommit mode
    connection.execute(text("<statement>"))

    # "commit" the autobegun "transaction"
    connection.commit()

    # switch to default isolation level
    connection.execution_options(isolation_level=connection.default_isolation_level)

    # use a begin block
    with connection.begin() as trans:
        connection.execute(text("<statement>"))
```

上面，在手动恢复隔离级别时，我们使用了 `Connection.default_isolation_level` 来恢复默认的隔离级别（假设这是我们想要的）。然而，更好的做法可能是利用 `Connection` 的架构，该架构已经在检入时自动处理重置隔离级别。上述写法的**首选**方式是使用两个代码块。

```py
# use an autocommit block
with engine.connect().execution_options(isolation_level="AUTOCOMMIT") as connection:
    # run statement in autocommit mode
    connection.execute(text("<statement>"))

# use a regular block
with engine.begin() as connection:
    connection.execute(text("<statement>"))
```

总结一下：

1.  “DBAPI 级别自动提交”隔离级别完全独立于 `Connection` 对象的“开始”和“提交”概念。

1.  使用每个隔离级别的单独 `Connection` 检出。避免在单个连接检出上来回切换“自动提交”；让引擎来恢复默认的隔离级别。

## 使用服务器端游标（又名流式结果）

一些后端提供对“服务器端游标”与“客户端游标”的概念的明确支持。这里的客户端游标意味着数据库驱动程序在从语句执行返回之前完全将所有行从结果集中获取到内存中。例如，像 PostgreSQL 和 MySQL/MariaDB 这样的驱动程序通常默认使用客户端游标。相比之下，服务器端游标表示结果行在客户端消耗时保留在数据库服务器的状态中。例如，Oracle 的驱动程序通常使用“服务器端”模型，而 SQLite 方言虽然不使用真正的“客户端/服务器”架构，但仍使用一种未缓冲的结果获取方法，将结果行保留在进程内存之外，直到它们被消耗。

从这个基本架构可以得出，“服务器端游标”在获取非常大的结果集时更加内存高效，同时可能会在客户端/服务器通信过程中引入更多复杂性，并且对于小的结果集（通常少于 10000 行）效率更低。

对于那些有条件支持缓冲或未缓冲结果的方言，通常使用“未缓冲”或服务器端游标模式时会有注意事项。例如，当使用 psycopg2 方言时，如果使用服务器端游标与任何类型的 DML 或 DDL 语句，则会引发错误。当使用带有服务器端游标的 MySQL 驱动程序时，DBAPI 连接处于更脆弱的状态，并且不会像从容处理错误条件那样优雅，也不会允许回滚操作继续进行，直到游标完全关闭。

因此，SQLAlchemy 的方言总是默认为游标的较少错误版本，这意味着对于 PostgreSQL 和 MySQL 方言，默认情况下使用缓冲的“客户端”游标，在调用游标的任何获取方法之前将结果集完全拉入内存。这种操作模式在**绝大多数**情况下都是适当的；未缓冲的游标通常除了在应用程序以分块方式获取非常大量的行的罕见情况下，此类行的处理可以在获取更多行之前完成之外，一般没有用处。

对于提供客户端和服务器端游标选项的数据库驱动程序，`Connection.execution_options.stream_results` 和 `Connection.execution_options.yield_per` 执行选项提供了在每个 `Connection` 或每个语句基础上访问“服务器端游标”的能力。在使用 ORM `Session` 时也存在类似的选项。

### 通过 yield_per 进行固定缓冲的流式处理

由于单独的行提取操作使用完全未缓冲的服务器端游标通常比一次提取多行的批次更昂贵，`Connection.execution_options.yield_per`执行选项配置了一个`Connection`或语句以使用可用的服务器端游标，同时配置了一个固定大小的行缓冲区，该缓冲区将在消耗时按批次从服务器检索行。使用`Connection.execution_options()`方法在`Connection`上或在语句上使用`Executable.execution_options()`方法可以将此参数设置为正整数值。

从 1.4.40 版本开始新增：`Connection.execution_options.yield_per`作为一个仅限核心的选项是自 SQLAlchemy 1.4.40 版本开始新增的；对于先前的 1.4 版本，请直接使用`Connection.execution_options.stream_results`与`Result.yield_per()`方法的组合。

使用此选项相当于手动设置`Connection.execution_options.stream_results`选项，该选项在下一节中描述，并在给定整数值的情况下调用`Result.yield_per()`方法。在这两种情况下，这种组合的效果包括：

+   如果可用且尚未是该后端的默认行为，则为给定后端选择了服务器端游标模式。

+   当结果行被获取时，它们将被分批缓冲，直到最后一批，每个批次的大小将等于传递给`Connection.execution_options.yield_per`选项或`Result.yield_per()`方法的整数参数；然后最后一批会根据剩余行数小于此大小来确定大小。

+   如果使用`Result.partitions()`方法，默认使用的分区大小将与此整数大小相等。

下面的示例说明了这三种行为：

```py
with engine.connect() as conn:
    with conn.execution_options(yield_per=100).execute(
        text("select * from table")
    ) as result:
        for partition in result.partitions():
            # partition is an iterable that will be at most 100 items
            for row in partition:
                print(f"{row}")
```

上述示例说明了将 `yield_per=100` 与使用 `Result.partitions()` 方法结合起来，在与从服务器获取的大小相匹配的批次中处理行。使用 `Result.partitions()` 是可选的，如果直接迭代 `Result`，将为每获取 100 行新建一个行批次缓冲区。不应使用诸如 `Result.all()` 的方法，因为这会一次性获取所有剩余的行，并且会使使用 `yield_per` 的目的丧失。

提示

如上所示，`Result` 对象可以用作上下文管理器。当使用服务器端游标进行迭代时，这是确保 `Result` 对象关闭的最佳方法，即使在迭代过程中发生异常也是如此。

`Connection.execution_options.yield_per` 选项同样适用于 ORM，由 `Session` 使用以获取 ORM 对象，在这里它还限制了一次生成的 ORM 对象的数量。有关使用 `Connection.execution_options.yield_per` 与 ORM 的进一步背景，请参阅 ORM 查询指南 中的 使用 Yield Per 获取大型结果集 部分。

新版本 1.4.40 中新增了 `Connection.execution_options.yield_per` 作为核心级执行选项，方便设置流式结果、缓冲区大小和分区大小，以一种可转移至 ORM 的类似用例方式进行设置。

### 使用 stream_results 实现动态增长缓冲区进行流式处理

要在没有特定分区大小的情况下启用服务器端游标，可以使用`Connection.execution_options.stream_results`选项，类似于`Connection.execution_options.yield_per`，可以在`Connection`对象或语句对象上调用。

当使用`Connection.execution_options.stream_results`选项直接迭代传递的`Result`对象时，内部使用默认的缓冲方案来获取行，首先缓冲一小组行，然后在每次获取时缓冲越来越大的缓冲区，直到预先配置的 1000 行的限制。可以使用`Connection.execution_options.max_row_buffer`执行选项来影响此缓冲区的最大大小：

```py
with engine.connect() as conn:
    with conn.execution_options(stream_results=True, max_row_buffer=100).execute(
        text("select * from table")
    ) as result:
        for row in result:
            print(f"{row}")
```

当`Connection.execution_options.stream_results`选项与`Result.partitions()`方法结合时，应该向`Result.partitions()`传递特定的分区大小，以避免获取整个结果集。设置使用`Result.partitions()`方法时，通常更直接的方法是使用`Connection.execution_options.yield_per`选项。

另请参见

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

`Result.partitions()`

`Result.yield_per()`

### 通过 yield_per 使用固定缓冲区进行流式传输

由于单独的行获取操作与完全无缓冲的服务器端游标通常比一次获取多行的批次更昂贵，`Connection.execution_options.yield_per`执行选项配置`Connection`或语句以利用服务器端可用的游标，同时配置一定大小的行缓冲区，以批次方式从服务器检索行，因为它们被使用。这个参数可以使用`Connection.execution_options()`方法设置为正整数值，也可以在语句上使用`Executable.execution_options()`方法设置。

新版本 1.4.40 中新增：`Connection.execution_options.yield_per`作为一个仅限于核心的选项是 SQLAlchemy 1.4.40 中的新功能；对于先前的 1.4 版本，请直接使用 `Connection.execution_options.stream_results` 与 `Result.yield_per()` 结合使用。

使用此选项等效于手动设置下一节中描述的`Connection.execution_options.stream_results`选项，然后在给定的整数值上调用`Result.yield_per()`方法。在这两种情况下，此组合的效果包括：

+   如果可用且尚未是该后端的默认行为，则为给定后端选择服务器端游标模式

+   当结果行被获取时，它们将被分批缓冲，每个批次的大小直到最后一个批次将等于传递给`Connection.execution_options.yield_per`选项或`Result.yield_per()`方法的整数参数；然后最后一个批次根据少于这个大小的剩余行大小进行调整

+   如果使用`Result.partitions()`方法，则默认分区大小也将设置为此整数大小。

这三种行为如下示例所示：

```py
with engine.connect() as conn:
    with conn.execution_options(yield_per=100).execute(
        text("select * from table")
    ) as result:
        for partition in result.partitions():
            # partition is an iterable that will be at most 100 items
            for row in partition:
                print(f"{row}")
```

上述示例说明了使用`yield_per=100`与使用`Result.partitions()`方法一起以批量处理与从服务器获取的大小匹配的行的组合。使用`Result.partitions()`是可选的，如果直接迭代`Result`，则每 100 行获取一次新的行批量。不应使用诸如`Result.all()`之类的方法，因为这将一次性完全获取所有剩余的行，从而使使用`yield_per`的目的失效。

提示

`Result`对象可以像上面示例的那样用作上下文管理器。当使用服务器端游标进行迭代时，这是确保`Result`对象关闭的最佳方法，即使在迭代过程中引发异常。

`Connection.execution_options.yield_per`选项也可移植到 ORM，由`Session`用于获取 ORM 对象，在这里它还限制了一次生成的 ORM 对象的数量。有关在 ORM 中使用`Connection.execution_options.yield_per`的更多背景信息，请参阅使用 Yield Per 获取大结果集 - ORM 查询指南中的相应部分。

新版本 1.4.40 中新增了`Connection.execution_options.yield_per`作为核心级别的执行选项，可以方便地设置流式结果、缓冲区大小和分区大小，一次性完成，以便与 ORM 的类似用例相匹配。

### 使用 stream_results 进行动态增长缓冲区的流式传输

要在没有特定分区大小的情况下启用服务器端游标，可以使用 `Connection.execution_options.stream_results` 选项，这与 `Connection.execution_options.yield_per` 相似，可以在 `Connection` 对象或语句对象上调用。

当使用 `Connection.execution_options.stream_results` 选项直接迭代生成的 `Result` 对象时，内部使用默认的缓冲方案来获取行，该方案首先缓冲少量行，然后在每次获取时缓冲越来越多的行，直到预先配置的 1000 行的限制。可以使用 `Connection.execution_options.max_row_buffer` 执行选项来影响此缓冲区的最大大小：

```py
with engine.connect() as conn:
    with conn.execution_options(stream_results=True, max_row_buffer=100).execute(
        text("select * from table")
    ) as result:
        for row in result:
            print(f"{row}")
```

当 `Connection.execution_options.stream_results` 选项与 `Result.partitions()` 方法结合使用时，应向 `Result.partitions()` 传递特定的分区大小，以避免获取整个结果集。通常，使用 `Connection.execution_options.yield_per` 选项设置使用 `Result.partitions()` 方法会更为直接。

另请参阅

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

`Result.partitions()`

`Result.yield_per()`

## 模式名称的翻译

为了支持将共享的表集分布到多个模式的多租户应用程序，可以使用 `Connection.execution_options.schema_translate_map` 执行选项将一组 `Table` 对象重新用不同的模式名称渲染，而不需要进行任何更改。

给定一张表：

```py
user_table = Table(
    "user",
    metadata_obj,
    Column("id", Integer, primary_key=True),
    Column("name", String(50)),
)
```

此 `Table` 的“模式”由 `Table.schema` 属性定义为 `None`。`Connection.execution_options.schema_translate_map` 可以指定所有模式为 `None` 的 `Table` 对象应将模式渲染为 `user_schema_one`：

```py
connection = engine.connect().execution_options(
    schema_translate_map={None: "user_schema_one"}
)

result = connection.execute(user_table.select())
```

上述代码将在数据库上调用以下形式的 SQL：

```py
SELECT  user_schema_one.user.id,  user_schema_one.user.name  FROM
user_schema_one.user
```

也就是说，模式名称被替换为我们翻译过的名称。映射可以指定任意数量的目标->目的地模式：

```py
connection = engine.connect().execution_options(
    schema_translate_map={
        None: "user_schema_one",  # no schema name -> "user_schema_one"
        "special": "special_schema",  # schema="special" becomes "special_schema"
        "public": None,  # Table objects with schema="public" will render with no schema
    }
)
```

`Connection.execution_options.schema_translate_map` 参数影响从 SQL 表达语言生成的所有 DDL 和 SQL 构造，这些构造是从 `Table` 或 `Sequence` 对象派生而来的。它 **不会** 影响通过 `text()` 构造使用的文本字符串 SQL，也不会影响传递给 `Connection.execute()` 的普通字符串。

此功能**仅在**模式的名称直接从`Table` 或 `Sequence` 派生的情况下生效；它不影响直接传递字符串模式名称的方法。根据此模式，在调用诸如`MetaData.create_all()` 或 `MetaData.drop_all()` 等方法执行的“可以创建”/“可以删除”检查中生效，并且在使用给定 `Table` 对象的表反射时生效。然而，它**不会**影响`Inspector` 对象上存在的操作，因为模式名称是显式传递给这些方法的。

提示

要在 ORM `Session` 中使用模式转换功能，请将此选项设置在`Engine` 的级别上，然后将该引擎传递给 `Session`。`Session` 为每个事务使用一个新的`Connection`：

```py
schema_engine = engine.execution_options(schema_translate_map={...})

session = Session(schema_engine)

...
```

警告

当在没有扩展的情况下使用 ORM `Session` 时，仅支持**每个 Session 一个单一的模式转换映射**。如果在每个语句的基础上给出了不同的模式转换映射，则**不会生效**，因为 ORM `Session` 不会考虑当前模式转换值对各个对象的影响。

要在多个 `schema_translate_map` 配置中使用单个 `Session`，可以使用水平分片扩展。请参阅水平分片中的示例。

## SQL 编译缓存

从版本 1.4 开始：SQLAlchemy 现在具有一个透明的查询缓存系统，大大降低了将 SQL 语句结构转换为 SQL 字符串时涉及的 Python 计算开销，包括 Core 和 ORM。请参阅透明 SQL 编译缓存添加到 Core、ORM 中的所有 DQL、DML 语句的介绍。

SQLAlchemy 包括一个全面的缓存系统，用于 SQL 编译器及其 ORM 变体。此缓存系统在`Engine`中是透明的，并且提供了对于给定的 Core 或 ORM SQL 语句的 SQL 编译过程以及为该语句组装结果获取机制的相关计算，只会对该语句对象及所有具有相同结构的其他语句执行一次，只要特定结构在引擎的“编译缓存”中保持。关于“具有相同结构的语句对象”，这通常对应于在函数内构造的 SQL 语句，每次运行该函数时都会构建：

```py
def run_my_statement(connection, parameter):
    stmt = select(table)
    stmt = stmt.where(table.c.col == parameter)
    stmt = stmt.order_by(table.c.id)
    return connection.execute(stmt)
```

上述语句将生成类似于`SELECT id, col FROM table WHERE col = :col ORDER BY id`的 SQL 语句，注意，虽然`parameter`的值是一个普通的 Python 对象，比如一个字符串或一个整数，但是语句的字符串 SQL 形式不包括此值，因为它使用了绑定参数。上述`run_my_statement()`函数的后续调用将在`connection.execute()`调用的范围内使用缓存的编译构造以提高性能。

注意

需要注意的是，SQL 编译缓存仅缓存传递给数据库的**SQL 字符串**，而不是查询返回的数据。它绝不是数据缓存，也不会影响特定 SQL 语句返回的结果，也不会暗示与结果行提取相关联的任何内存使用。

虽然 SQLAlchemy 自 1.x 系列早期就有了一个基本的语句缓存，并且此外还提供了 ORM 的“烘焙查询”扩展，但这两个系统都需要高度特殊的 API 使用，以便缓存起作用。自 1.4 版本以来的新缓存完全是自动的，不需要更改编程风格即可生效。

缓存是自动使用的，无需进行任何配置更改，也不需要任何特殊步骤来启用它。以下部分详细介绍了缓存的配置和高级使用模式。

### 配置

缓存本身是一个类似字典的对象，称为`LRUCache`，它是一个内部 SQLAlchemy 字典子类，跟踪特定键的使用情况，并具有定期的“修剪”步骤，当缓存的大小达到一定阈值时会删除最近未使用的项目。此缓存的大小默认为 500，可以使用`create_engine.query_cache_size`参数进行配置：

```py
engine = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test", query_cache_size=1200
)
```

缓存的大小可以增长到给定大小的 150%左右，然后将其修剪回目标大小。因此，大小为 1200 的缓存可以增长到 1800 个元素的大小，然后将其修剪回 1200。

缓存的大小基于每个唯一 SQL 语句渲染的单个条目，每个引擎。从 Core 和 ORM 生成的 SQL 语句被等同对待。DDL 语句通常不会被缓存。为了确定缓存正在做什么，引擎日志将包含有关缓存行为的详细信息，下一节描述了此信息。

### 使用日志估算缓存性能

上述缓存大小为 1200 实际上是相当大的。对于小型应用程序，大小为 100 可能足够。要估算缓存的最佳大小，假设目标主机上有足够的内存，缓存的大小应基于在使用中的目标引擎中可以呈现的唯一 SQL 字符串的数量。最快速的方法是使用 SQL 回显，最直接的方法是使用`create_engine.echo` 标志启用，或者使用 Python 日志记录；有关日志记录配置的背景，请参阅配置日志记录 部分。

作为示例，我们将检查以下程序产生的日志记录：

```py
from sqlalchemy import Column
from sqlalchemy import create_engine
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy import String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy.orm import Session

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(String)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

s = Session(e)

s.add_all([A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()])])
s.commit()

for a_rec in s.scalars(select(A)):
    print(a_rec.bs)
```

运行时，每个记录的 SQL 语句将在传递的参数左侧包含一个带方括号的缓存统计徽章。我们可能看到的四种消息类型总结如下：

+   `[原始 SQL]` - 驱动程序或最终用户使用`Connection.exec_driver_sql()` 发出原始 SQL - 不适用缓存

+   `[无键]` - 该语句对象是一个不被缓存的 DDL 语句，或者该语句对象包含不可缓存的元素，如用户定义的结构或任意大的 VALUES 子句。

+   `[在 X 秒内生成]` - 该语句是一个**缓存未命中**，必须被编译，然后存储在缓存中。生成编译结构花费了 X 秒。数字 X 将是小数秒数。

+   `[自 X 秒前缓存]` - 该语句是**缓存命中**，不需要重新编译。该语句已经存储在缓存中自 X 秒前。数字 X 与应用程序运行的时间以及语句被缓存的时间成比例，例如 24 小时的时间段将是 86400。

下面更详细地描述了每个徽章。

我们看到上面程序的第一个语句将是 SQLite 方言检查 “a” 和 “b” 表的存在：

```py
INFO sqlalchemy.engine.Engine PRAGMA temp.table_info("a")
INFO sqlalchemy.engine.Engine [raw sql] ()
INFO sqlalchemy.engine.Engine PRAGMA main.table_info("b")
INFO sqlalchemy.engine.Engine [raw sql] ()
```

对于上述两个 SQLite PRAGMA 语句，徽章显示为`[raw sql]`，这表示驱动程序正在使用`Connection.exec_driver_sql()`直接将 Python 字符串发送到数据库。对于这样的语句，缓存不适用，因为它们已经以字符串形式存在，而且由于 SQLAlchemy 不会提前解析 SQL 字符串，因此不知道将返回什么类型的结果行。

接下来我们看到的是 CREATE TABLE 语句：

```py
INFO  sqlalchemy.engine.Engine
CREATE  TABLE  a  (
  id  INTEGER  NOT  NULL,
  data  VARCHAR,
  PRIMARY  KEY  (id)
)

INFO  sqlalchemy.engine.Engine  [no  key  0.00007s]  ()
INFO  sqlalchemy.engine.Engine
CREATE  TABLE  b  (
  id  INTEGER  NOT  NULL,
  a_id  INTEGER,
  data  VARCHAR,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(a_id)  REFERENCES  a  (id)
)

INFO  sqlalchemy.engine.Engine  [no  key  0.00006s]  ()
```

对于每个语句，徽章显示为`[no key 0.00006s]`。这表示这两个特定语句，由于以 DDL 为导向的`CreateTable`构造未生成缓存键，因此缓存未发生。DDL 构造通常不参与缓存，因为它们通常不会被重复执行，而且 DDL 也是一个数据库配置步骤，性能并不那么关键。

`[no key]` 徽章还有一个重要原因，即它可能适用于可缓存的 SQL 语句，除了某些当前不可缓存的特定子构造。这些例子包括未定义缓存参数的自定义用户定义 SQL 元素，以及生成任意长且不可重现的 SQL 字符串的某些构造，主要示例包括`Values`构造以及使用`Insert.values()`方法进行“多值插入”时。

到目前为止，我们的缓存仍然是空的。然而，接下来的语句将被缓存，一个片段看起来像：

```py
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [generated  in  0.00011s]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0003533s  ago]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0005326s  ago]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [generated  in  0.00010s]  (1,  None)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0003232s  ago]  (1,  None)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0004887s  ago]  (1,  None)
```

在上面，我们看到了两个基本上是唯一的 SQL 字符串；`"INSERT INTO a (data) VALUES (?)"` 和 `"INSERT INTO b (a_id, data) VALUES (?, ?)"`。由于 SQLAlchemy 对所有文字值使用绑定参数，即使这些语句为不同对象重复多次，由于参数是分开的，实际的 SQL 字符串保持不变。

注意

上述两个语句是由 ORM 工作单元流程生成的，并且实际上将这些语句缓存在每个映射器本地的单独缓存中。然而，机制和术语是相同的。下面的部分禁用或使用替代字典缓存某些（或全部）语句将描述用户代码如何在每个语句基础上使用替代缓存容器。

我们在看到这两个语句的第一次出现时看到的缓存徽章是`[生成于 0.00011s]`。这表示该语句**不在缓存中，已在 0.00011s 内编译为字符串，然后被缓存**。当我们看到`[生成]`徽章时，我们知道这意味着发生了**缓存未命中**。这对于特定语句的第一次出现是可以预料的。然而，如果一个长时间运行的应用程序通常一遍又一遍地使用相同的一系列 SQL 语句，并且观察到大量新的`[生成]`徽章，这可能是`create_engine.query_cache_size`参数设置过小的迹象。当一个已被缓存的语句由于 LRU 缓存删除了不常用的项而被逐出缓存时，当它下次被使用时，它将显示`[生成]`徽章。

我们随后看到的每个这两个语句的后续出现的缓存徽章看起来像`[自 0.0003533s 前缓存]`。这表示该语句**在缓存中找到，并且最初放入缓存中 0.0003533 秒前**。重要的是要注意，虽然`[生成]`和`[自]`徽章都指的是秒数，但它们表示的是不同的含义；对于`[生成]`，数字是编译语句所需的大致时间，并且将是一个极小的时间量。对于`[自]`，这是语句在缓存中存在的总时间。对于运行了六个小时的应用程序，这个数字可能读作`[自 21600 秒前缓存]`，这是件好事。看到“自”徽章的高数字表明这些语句很长时间没有发生缓存未命中。即使应用程序运行了很长时间，语句经常具有较低的“自”数也可能表明这些语句太频繁地发生缓存未命中，而`create_engine.query_cache_size`可能需要增加。

我们的示例程序然后执行了一些 SELECT，我们可以看到“生成”然后“缓存”的相同模式，对于“a”表的 SELECT 以及“b”表的后续延迟加载：

```py
INFO sqlalchemy.engine.Engine SELECT a.id AS a_id, a.data AS a_data
FROM a
INFO sqlalchemy.engine.Engine [generated in 0.00009s] ()
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
INFO sqlalchemy.engine.Engine [generated in 0.00010s] (1,)
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
INFO sqlalchemy.engine.Engine [cached since 0.0005922s ago] (2,)
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
```

从我们上面的程序中，完整运行显示一共缓存了四个不同的 SQL 字符串。这表明缓存大小为**四**将是足够的。这显然是一个极小的大小，而默认大小为 500 是可以保持不变的。

### 缓存使用多少内存？

前一节详细介绍了一些技术，用于检查`create_engine.query_cache_size`是否需要更大。我们如何知道缓存不会太大？我们可能希望将`create_engine.query_cache_size`设置为不高于某个数字的原因是，我们可能有一个应用程序，可能会使用非常多不同的语句，比如一个从搜索 UX 动态构建查询的应用程序，如果过去 24 小时运行了十万个不同的查询并且它们都被缓存，我们不希望我们的主机内存耗尽。

测量 Python 数据结构占用多少内存是非常困难的，然而，通过使用`top`进程来测量内存增长，当连续添加 250 个新语句到缓存时，表明一个中等大小的核心语句大约占用 12K，而一个小型 ORM 语句大约占用 20K，包括 ORM 的结果获取结构，对于 ORM 来说，这将更大。

### 禁用或使用备用字典来缓存一些（或全部）语句

使用的内部缓存称为`LRUCache`，但这主要只是一个字典。可以通过使用`Connection.execution_options.compiled_cache`选项作为执行选项，为任何一系列语句使用任何字典作为缓存。执行选项可以在语句上设置，在`Engine`或`Connection`上设置，以及在使用 ORM `Session.execute()`方法进行 SQLAlchemy-2.0 风格调用时设置。例如，要运行一系列 SQL 语句并将它们缓存在特定字典中：

```py
my_cache = {}
with engine.connect().execution_options(compiled_cache=my_cache) as conn:
    conn.execute(table.select())
```

SQLAlchemy ORM 在工作单元“flush”过程中使用上述技术来保留每个映射器缓存，这些缓存与`Engine`上配置的默认缓存分开，以及一些关系加载器查询。

也可以通过发送`None`值来禁用缓存：

```py
# disable caching for this connection
with engine.connect().execution_options(compiled_cache=None) as conn:
    conn.execute(table.select())
```  ### 第三方方言的缓存

缓存功能要求方言的编译器生成安全可重用的 SQL 字符串，给定一个特定的与该 SQL 字符串关联的缓存键，这意味着语句中的任何文字值，例如 SELECT 的 LIMIT/OFFSET 值，不能在方言的编译方案中硬编码，因为编译后的字符串将无法重复使用。SQLAlchemy 支持使用`BindParameter.render_literal_execute()`方法呈现绑定参数，该方法可以应用于自定义编译器的现有`Select._limit_clause`和`Select._offset_clause`属性，这些属性稍后在本节中进行了说明。

由于有许多第三方方言，其中许多可能会从 SQL 语句中生成文字值而没有新的“文字执行”功能的好处，因此 SQLAlchemy 在版本 1.4.5 中为方言添加了一个名为`Dialect.supports_statement_cache`的属性。此属性在运行时直接在特定方言类上检查其是否存在，即使它已经存在于超类上，因此即使第三方方言是现有可缓存的 SQLAlchemy 方言的子类，比如`sqlalchemy.dialects.postgresql.PGDialect`，也必须明确包含此属性以启用缓存。该属性应该在方言经过必要的修改并经过测试以确保编译的 SQL 语句具有不同参数的可重用性后才能**启用**。

对于所有不支持此属性的第三方方言，该方言的日志将指示`方言不支持缓存`。

当方言经过缓存测试，特别是 SQL 编译器已更新以不直接在 SQL 字符串中呈现任何文字 LIMIT / OFFSET 时，方言作者可以按照以下方式应用该属性：

```py
from sqlalchemy.engine.default import DefaultDialect

class MyDialect(DefaultDialect):
    supports_statement_cache = True
```

标志需要应用到方言的所有子类中：

```py
class MyDBAPIForMyDialect(MyDialect):
    supports_statement_cache = True
```

版本 1.4.5 中的新功能：添加了`Dialect.supports_statement_cache`属性。

方言修改的典型情况如下。

#### 示例：使用后编译参数渲染 LIMIT / OFFSET

举个例子，假设一个方言重写了`SQLCompiler.limit_clause()`方法，该方法为 SQL 语句生成“LIMIT / OFFSET”子句，如下所示：

```py
# pre 1.4 style code
def limit_clause(self, select, **kw):
    text = ""
    if select._limit is not None:
        text += " \n LIMIT %d" % (select._limit,)
    if select._offset is not None:
        text += " \n OFFSET %d" % (select._offset,)
    return text
```

上述例程将`Select._limit`和`Select._offset`整数值呈现为嵌入在 SQL 语句中的字面整数。这对于不支持在 SELECT 语句的 LIMIT/OFFSET 子句中使用绑定参数的数据库是常见的要求。但是，在初始编译阶段内呈现整数值直接**不兼容**缓存，因为`Select`对象的 limit 和 offset 整数值不是缓存键的一部分，因此许多带有不同 limit/offset 值的`Select`语句将无法以正确的值呈现。

以上代码的修正是将字面整数移到 SQLAlchemy 的后编译设施中，这将使字面整数在初始编译阶段之外渲染，而是在执行时在将语句发送到 DBAPI 之前。这在编译阶段使用`BindParameter.render_literal_execute()`方法进行访问，与使用`Select._limit_clause`和`Select._offset_clause`属性结合使用，这些属性将 LIMIT/OFFSET 表示为完整的 SQL 表达式，如下所示：

```py
# 1.4 cache-compatible code
def limit_clause(self, select, **kw):
    text = ""

    limit_clause = select._limit_clause
    offset_clause = select._offset_clause

    if select._simple_int_clause(limit_clause):
        text += " \n LIMIT %s" % (
            self.process(limit_clause.render_literal_execute(), **kw)
        )
    elif limit_clause is not None:
        # assuming the DB doesn't support SQL expressions for LIMIT.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for LIMIT"
        )
    if select._simple_int_clause(offset_clause):
        text += " \n OFFSET %s" % (
            self.process(offset_clause.render_literal_execute(), **kw)
        )
    elif offset_clause is not None:
        # assuming the DB doesn't support SQL expressions for OFFSET.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for OFFSET"
        )

    return text
```

上述方法将生成一个编译后的 SELECT 语句，看起来像：

```py
SELECT  x  FROM  y
LIMIT  __[POSTCOMPILE_param_1]
OFFSET  __[POSTCOMPILE_param_2]
```

在上述情况下，`__[POSTCOMPILE_param_1]`和`__[POSTCOMPILE_param_2]`指示符将在语句执行时填充其相应的整数值，此时 SQL 字符串已从缓存中检索出。

在做出适当的类似上述的更改后，应将`Dialect.supports_statement_cache`标志设置为`True`。强烈建议第三方方言使用[dialect third party test suite](https://github.com/sqlalchemy/sqlalchemy/blob/main/README.dialects.rst)，该套件将断言 SELECT 带有 LIMIT/OFFSET 的操作是否正确呈现和缓存。

另请参阅

我升级到 1.4 和/或 2.x 后，为什么我的应用程序变慢了？ - 在常见问题部分 ### 使用 Lambda 在语句生成中添加显著的速度增益

深度炼金术

除了在非常性能密集的场景中通常是非必要的，并且专为有经验的 Python 程序员而设计，此技术也不适合初学者 Python 开发人员。虽然相当简单，但它涉及到不适合初学者 Python 开发人员的元编程概念。Lambda 方法可以稍后应用于现有代码，而只需付出最小的努力。

Python 函数，通常表示为 lambda，可以用于生成可基于 lambda 函数本身的 Python 代码位置以及 lambda 内的闭包变量进行缓存的 SQL 表达式。其原因是允许缓存 SQL 表达式构造的 SQL 字符串编译形式，这是 SQLAlchemy 在未使用 lambda 系统时的正常行为，以及 SQL 表达式构造本身的 Python 组合，这也具有一定程度的 Python 开销。

lambda SQL 表达式功能可作为性能增强功能使用，并且也可选择在 `with_loader_criteria()` ORM 选项中使用，以提供通用的 SQL 片段。

#### 梗概

Lambda 语句是使用 `lambda_stmt()` 函数构造的，该函数返回 `StatementLambdaElement` 的实例，它本身是一个可执行的语句构造。可以使用 Python 加法运算符 `+` 或者 `StatementLambdaElement.add_criteria()` 方法向对象添加其他修饰符和条件，该方法允许更多选项。

假定 `lambda_stmt()` 构造在期望在应用程序中被多次使用的封闭函数或方法内被调用，以便超出第一次调用后的后续执行可以利用被缓存的编译 SQL。当 lambda 在 Python 的封闭函数内部构造时，也受到具有闭包变量的影响，这对整个方法至关重要：

```py
from sqlalchemy import lambda_stmt

def run_my_statement(connection, parameter):
    stmt = lambda_stmt(lambda: select(table))
    stmt += lambda s: s.where(table.c.col == parameter)
    stmt += lambda s: s.order_by(table.c.id)

    return connection.execute(stmt)

with engine.connect() as conn:
    result = run_my_statement(some_connection, "some parameter")
```

在上面，用于定义 SELECT 语句结构的三个 `lambda` 可调用对象仅被调用一次，并且生成的 SQL 字符串被缓存在引擎的编译缓存中。从那时起，`run_my_statement()` 函数可以被调用任意次数，并且其中的 `lambda` 可调用对象不会被调用，而只用作缓存键来检索已编译的 SQL。

注意

当 lambda 系统未被使用时，已经存在 SQL 缓存，这一点很重要。lambda 系统只是在每个 SQL 语句调用时添加了额外的工作减少层，通过缓存 SQL 构建本身以及使用更简单的缓存键来实现。

#### Lambda 的快速指南

最重要的是，Lambda SQL 系统的重点是确保生成的 lambda 的缓存密钥与其将产生的 SQL 字符串之间永远不会不匹配。`LambdaElement` 和相关对象将运行并分析给定的 lambda，以计算每次运行时应如何缓存它，尝试检测任何潜在问题。基本指南包括：

+   **支持任何类型的语句** - 虽然 `select()` 构造预计是 `lambda_stmt()` 的主要用例，但 DML 语句，如 `insert()` 和 `update()` 也同样可用：

    ```py
    def upd(id_, newname):
        stmt = lambda_stmt(lambda: users.update())
        stmt += lambda s: s.values(name=newname)
        stmt += lambda s: s.where(users.c.id == id_)
        return stmt

    with engine.begin() as conn:
        conn.execute(upd(7, "foo"))
    ```

+   **ORM 用例也直接支持** - `lambda_stmt()` 可完全容纳 ORM 功能，并可直接与 `Session.execute()` 一起使用：

    ```py
    def select_user(session, name):
        stmt = lambda_stmt(lambda: select(User))
        stmt += lambda s: s.where(User.name == name)

        row = session.execute(stmt).first()
        return row
    ```

+   **绑定参数会自动适应** - 与 SQLAlchemy 以前的“烘焙查询”系统相比，Lambda SQL 系统会自动适应成为 SQL 绑定参数的 Python 字面值。这意味着即使给定的 Lambda 只运行一次，但成为绑定参数的值会在每次运行时从 Lambda 的 **闭包** 中提取出来：

    ```py
    >>> def my_stmt(x, y):
    ...     stmt = lambda_stmt(lambda: select(func.max(x, y)))
    ...     return stmt
    >>> engine = create_engine("sqlite://", echo=True)
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    ...     print(conn.scalar(my_stmt(12, 8)))
    SELECT  max(?,  ?)  AS  max_1
    [generated  in  0.00057s]  (5,  10)
    10
    SELECT  max(?,  ?)  AS  max_1
    [cached  since  0.002059s  ago]  (12,  8)
    12
    ```

    在上面的示例中，`StatementLambdaElement` 从每次调用 `my_stmt()` 时生成的 lambda 的 **闭包** 中提取了 `x` 和 `y` 的值；这些值被替换为参数的值并嵌入到缓存的 SQL 结构中。

+   **理想情况下，Lambda 应该在所有情况下产生相同的 SQL 结构** - 避免在 lambda 内部使用条件语句或自定义可调用对象，这可能会根据输入产生不同的 SQL；如果函数可能会有条件地使用两个不同的 SQL 片段，请使用两个单独的 lambda：

    ```py
    # **Don't** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        stmt += lambda s: (
            s.where(table.c.x > parameter) if thing else s.where(table.c.y == parameter)
        )
        return stmt

    # **Do** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        if thing:
            stmt += lambda s: s.where(table.c.x > parameter)
        else:
            stmt += lambda s: s.where(table.c.y == parameter)
        return stmt
    ```

    如果 lambda 不生成一致的 SQL 结构，则可能会发生各种失败，其中一些目前不容易检测到。

+   **不要在 lambda 内部使用函数生成绑定值** - 绑定值跟踪方法要求 SQL 语句中要使用的实际值在 lambda 的闭包中本地存在。如果值是从其他函数生成的，则不可能实现这一点，并且如果尝试执行此操作，`LambdaElement`通常会引发错误：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     stmt = lambda_stmt(lambda: select(func.max(get_x(), get_y())))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Can't invoke Python callable get_x()
    inside of lambda expression argument at
    <code object <lambda> at 0x7fed15f350e0, file "<stdin>", line 6>;
    lambda SQL constructs should not invoke functions from closure variables
    to produce literal values since the lambda SQL system normally extracts
    bound values without actually invoking the lambda or any functions within it.
    ```

    上面，如果需要，`get_x()`和`get_y()`的使用应该在 lambda 的**外部**发生，并分配给本地闭包变量：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     x_param, y_param = get_x(), get_y()
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

+   **避免在 lambda 内部引用非 SQL 构造，因为它们默认情况下不可缓存** - 这个问题涉及到`LambdaElement`如何从语句中的其他闭包变量创建缓存键。为了提供准确的缓存键的最佳保证，lambda 闭包中的所有对象都被认为是重要的，且默认情况下不会假设它们适合作为缓存键。因此，下面的示例也将引发一个相当详细的错误消息：

    ```py
    >>> class Foo:
    ...     def __init__(self, x, y):
    ...         self.x = x
    ...         self.y = y
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(lambda: select(func.max(foo.x, foo.y)))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(Foo(5, 10))))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Closure variable named 'foo' inside of
    lambda callable <code object <lambda> at 0x7fed15f35450, file
    "<stdin>", line 2> does not refer to a cacheable SQL element, and also
    does not appear to be serving as a SQL literal bound value based on the
    default SQL expression returned by the function.  This variable needs to
    remain outside the scope of a SQL-generating lambda so that a proper cache
    key may be generated from the lambda's state.  Evaluate this variable
    outside of the lambda, set track_on=[<elements>] to explicitly select
    closure elements to track, or set track_closure_variables=False to exclude
    closure variables from being part of the cache key.
    ```

    上述错误表明`LambdaElement`不会假设传入的`Foo`对象在所有情况下都会保持相同的行为。它也不会默认假设它可以将`Foo`作为缓存键的一部分使用；如果将`Foo`对象用作缓存键的一部分，如果有许多不同的`Foo`对象，这将使缓存填满重复信息，并且还将长时间保留对所有这些对象的引用。

    解决上述情况的最佳方法是不要在 lambda 内部引用`foo`，而是在**外部**引用它：

    ```py
    >>> def my_stmt(foo):
    ...     x_param, y_param = foo.x, foo.y
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

    在某些情况下，如果可以保证 lambda 的 SQL 结构不会根据输入改变，可以传递`track_closure_variables=False`来禁用对除绑定参数外的任何闭包变量的跟踪：

    ```py
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(func.max(foo.x, foo.y)), track_closure_variables=False
    ...     )
    ...     return stmt
    ```

    还有一种选择，即通过`track_on`参数将对象添加到元素中，以明确形成缓存键的一部分；使用此参数允许特定值作为缓存键，并且还将阻止考虑其他闭包变量。这对于构造的 SQL 的一部分源自某种上下文对象并且可能具有许多不同值的情况非常有用。在下面的示例中，SELECT 语句的第一个段将禁用对`foo`变量的跟踪，而第二个段将明确跟踪`self`作为缓存键的一部分：

    ```py
    >>> def my_stmt(self, foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(*self.column_expressions), track_closure_variables=False
    ...     )
    ...     stmt = stmt.add_criteria(lambda: self.where_criteria, track_on=[self])
    ...     return stmt
    ```

    使用`track_on`意味着给定的对象将长期存储在 lambda 的内部缓存中，并且只要缓存不清除这些对象（默认使用 1000 个条目的 LRU 方案）就会具有强引用。

#### 缓存键生成

要理解与 lambda SQL 构造相关的一些选项和行为，了解缓存系统是有帮助的。

SQLAlchemy 的缓存系统通常通过生成一个表示构造内所有状态的结构来从给定的 SQL 表达式构造中生成缓存键：

```py
>>> from sqlalchemy import select, column
>>> stmt = select(column("q"))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)  # somewhat paraphrased
CacheKey(key=(
 '0',
 <class 'sqlalchemy.sql.selectable.Select'>,
 '_raw_columns',
 (
 (
 '1',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 ),
 # a few more elements are here, and many more for a more
 # complicated SELECT statement
),)
```

上面的键存储在本质上是一个字典的缓存中，值是一个结构，其中包括 SQL 语句的字符串形式，本例中是短语 “SELECT q”。我们可以观察到，即使对于一个极短的查询，缓存键也非常冗长，因为它必须表示有关正在呈现和可能执行的所有内容。

相比之下，lambda 构造系统会创建一种不同类型的缓存键：

```py
>>> from sqlalchemy import lambda_stmt
>>> stmt = lambda_stmt(lambda: select(column("q")))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)
CacheKey(key=(
 <code object <lambda> at 0x7fed1617c710, file "<stdin>", line 1>,
 <class 'sqlalchemy.sql.lambdas.StatementLambdaElement'>,
),)
```

上面，我们看到的缓存键比非 lambda 语句的要短得多，而且甚至生产 `select(column("q"))` 构造本身也不是必要的；Python lambda 本身包含一个称为 `__code__` 的属性，它引用了一个在应用程序运行时是不可变和永久的 Python 代码对象。

当 lambda 还包含闭包变量时，在正常情况下，这些变量引用诸如列对象等 SQL 构造，它们将成为缓存键的一部分，或者如果它们引用将绑定参数的文字值，则它们将放置在缓存键的单独元素中：

```py
>>> def my_stmt(parameter):
...     col = column("q")
...     stmt = lambda_stmt(lambda: select(col))
...     stmt += lambda s: s.where(col == parameter)
...     return stmt
```

上述 `StatementLambdaElement` 包含两个 lambda，两者都引用 `col` 闭包变量，因此缓存键将表示这两个段以及 `column()` 对象：

```py
>>> stmt = my_stmt(5)
>>> key = stmt._generate_cache_key()
>>> print(key)
CacheKey(key=(
 <code object <lambda> at 0x7f07323c50e0, file "<stdin>", line 3>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 <code object <lambda> at 0x7f07323c5190, file "<stdin>", line 4>,
 <class 'sqlalchemy.sql.lambdas.LinkedLambdaElement'>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
),)
```

缓存键的第二部分已检索到在调用语句时将使用的绑定参数：

```py
>>> key.bindparams
[BindParameter('%(139668884281280 parameter)s', 5, type_=Integer())]
```

有关带有性能比较的 “lambda” 缓存的一系列示例，请参阅 性能 性能示例中的 “short_selects” 测试套件。

### 配置

缓存本身是一个名为 `LRUCache` 的类似字典的对象，它是一个内部 SQLAlchemy 字典子类，用于跟踪特定键的使用情况，并具有周期性的 “修剪” 步骤，当缓存的大小达到一定阈值时，将删除最近未使用的项目。该缓存的大小默认为 500，并可以使用 `create_engine.query_cache_size` 参数进行配置：

```py
engine = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test", query_cache_size=1200
)
```

缓存的大小可以增长为给定大小的 150%，然后将其修剪回目标大小。因此，大小为 1200 的缓存可以增长到 1800 个元素的大小，此时它将被修剪为 1200。

缓存的大小基于每个引擎呈现的唯一 SQL 语句的单个条目。来自 Core 和 ORM 的生成的 SQL 语句被等同对待。DDL 语句通常不会被缓存。为了确定缓存的行为，引擎日志将包括有关缓存行为的详细信息，将在下一节描述。

### 使用日志估算缓存性能

上述缓存大小为 1200 实际上相当大。对于小型应用程序，大小为 100 可能足够。要估算缓存的最佳大小，假设目标主机上有足够的内存，缓存的大小应基于可能在使用的目标引擎中呈现的唯一 SQL 字符串的数量。看到这一点最快捷的方法是使用 SQL 回显，最直接的方法是使用`create_engine.echo`标志启用，或使用 Python 记录;有关日志配置的背景，请参阅配置日志部分。

作为示例，我们将检查以下程序生成的日志：

```py
from sqlalchemy import Column
from sqlalchemy import create_engine
from sqlalchemy import ForeignKey
from sqlalchemy import Integer
from sqlalchemy import select
from sqlalchemy import String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy.orm import Session

Base = declarative_base()

class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)
    data = Column(String)
    bs = relationship("B")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))
    data = Column(String)

e = create_engine("sqlite://", echo=True)
Base.metadata.create_all(e)

s = Session(e)

s.add_all([A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()]), A(bs=[B(), B(), B()])])
s.commit()

for a_rec in s.scalars(select(A)):
    print(a_rec.bs)
```

运行时，每个记录的 SQL 语句都将在传递的参数左侧包含带方括号的缓存统计徽章。我们可能看到的四种消息总结如下：

+   `[raw sql]` - 驱动程序或最终用户使用`Connection.exec_driver_sql()`发出原始 SQL - 不适用缓存

+   `[no key]` - 该语句对象是一个未缓存的 DDL 语句，或者该语句对象包含无法缓存的元素，例如用户定义的构造或任意大的 VALUES 子句。

+   `[generated in Xs]` - 该语句是一个**缓存未命中**，必须编译，然后存储在缓存中。生成编译结构消耗了 X 秒。数字 X 将为小数秒。

+   `[cached since Xs ago]` - 该语句是一个**缓存命中**，无需重新编译。该语句自 X 秒前起已存储在缓存中。数字 X 将与应用程序运行的时间以及语句被缓存的时间成比例，因此，例如，对于 24 小时周期将为 86400。

下面更详细地描述了每个徽章。

对于上述程序，我们首先看到的语句将是 SQLite 方言检查“a”和“b”表是否存在：

```py
INFO sqlalchemy.engine.Engine PRAGMA temp.table_info("a")
INFO sqlalchemy.engine.Engine [raw sql] ()
INFO sqlalchemy.engine.Engine PRAGMA main.table_info("b")
INFO sqlalchemy.engine.Engine [raw sql] ()
```

对于上面的两个 SQLite PRAGMA 语句，标记显示为 `[原始 SQL]`，这表示驱动程序使用 `Connection.exec_driver_sql()` 将 Python 字符串直接发送到数据库。对这样的语句不适用缓存，因为它们已经以字符串形式存在，而且 SQLAlchemy 在事先不解析 SQL 字符串。

我们看到的下一条语句是 CREATE TABLE 语句：

```py
INFO  sqlalchemy.engine.Engine
CREATE  TABLE  a  (
  id  INTEGER  NOT  NULL,
  data  VARCHAR,
  PRIMARY  KEY  (id)
)

INFO  sqlalchemy.engine.Engine  [no  key  0.00007s]  ()
INFO  sqlalchemy.engine.Engine
CREATE  TABLE  b  (
  id  INTEGER  NOT  NULL,
  a_id  INTEGER,
  data  VARCHAR,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(a_id)  REFERENCES  a  (id)
)

INFO  sqlalchemy.engine.Engine  [no  key  0.00006s]  ()
```

对于这些语句的每个，标记显示为 `[无键 0.00006s]`。这表示这两个特定的语句，由于 DDL 导向的 `CreateTable` 构造未生成缓存键，因此未发生缓存。DDL 构造通常不参与缓存，因为它们通常不会被重复执行，而且 DDL 还是一个数据库配置步骤，性能并不那么关键。

`[无键]` 标记对另一个原因很重要，因为它可以用于生成可缓存的 SQL 语句，除了某些当前不可缓存的特定子构造。这些示例包括不定义缓存参数的自定义用户定义的 SQL 元素，以及一些生成任意长且不可重现的 SQL 字符串的构造，主要示例包括 `Values` 构造以及在使用 `Insert.values()` 方法进行“多值插入”时。

到目前为止，我们的缓存仍然是空的。然而，接下来的语句将被缓存，一个片段看起来像是：

```py
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [generated  in  0.00011s]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0003533s  ago]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  a  (data)  VALUES  (?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0005326s  ago]  (None,)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [generated  in  0.00010s]  (1,  None)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0003232s  ago]  (1,  None)
INFO  sqlalchemy.engine.Engine  INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)
INFO  sqlalchemy.engine.Engine  [cached  since  0.0004887s  ago]  (1,  None)
```

上面，我们基本上看到了两个唯一的 SQL 字符串；`"INSERT INTO a (data) VALUES (?)"` 和 `"INSERT INTO b (a_id, data) VALUES (?, ?)"`。由于 SQLAlchemy 对所有文本值使用绑定参数，即使这些语句为不同的对象重复多次，由于参数是分开的，实际的 SQL 字符串仍然相同。

注意

上述两个语句是由 ORM 工作单元流程生成的，实际上将这些语句缓存在每个映射器本地的单独缓存中。然而，机制和术语是相同的。下面的部分 禁用或使用备用字典来缓存一些（或全部）语句 将描述用户代码如何也可以在每个语句的基础上使用备用缓存容器。

我们在这两个语句的首次出现时看到的缓存徽章是`[生成于 0.00011s]`。这表明该语句**不在缓存中，被编译为一个字符串需时 0.00011 秒，然后被缓存**。当我们看到`[生成]`徽章时，我们知道这意味着**缓存未命中**。对于特定语句的首次出现，这是可以预料的。然而，如果在长时间运行的应用程序中频繁观察到大量新的`[生成]`徽章，而该应用程序通常会一遍又一遍地使用相同的一系列 SQL 语句，这可能是`create_engine.query_cache_size`参数设置过小的迹象。当一个被缓存的语句因为 LRU 缓存淘汰了较少使用的项而被驱逐出缓存时，它在下次使用时将显示`[生成]`徽章。

然后，我们看到每个这两个语句的后续出现所显示的缓存徽章类似于`[缓存自 0.0003533 秒前]`。这表明该语句**在缓存中找到，并且最初是在 0.0003533 秒前放入缓存**。需要注意的是，虽然`[生成]`和`[缓存自]`徽章都涉及到一定数量的秒数，但它们表示的是不同的含义；在`[生成]`的情况下，这个数字是编译语句所需的大致时间，将是一个极小的时间量。而在`[缓存自]`的情况下，这是语句在缓存中存在的总时间。对于运行了六小时的应用程序，这个数字可能会显示`[缓存自 21600 秒前]`，这是一件好事。观察到“缓存自”数值较高是这些语句长时间没有遇到缓存未命中的迹象。即使应用程序运行了很长时间，频繁具有较低的“缓存自”数值的语句，可能表明这些语句太频繁地遇到缓存未命中，而`create_engine.query_cache_size`可能需要增加。

然后，我们的示例程序执行了一些 SELECT 查询，我们可以看到“生成”然后“缓存”的相同模式，对于“a”表的 SELECT 以及“b”表的后续惰性加载也是如此。

```py
INFO sqlalchemy.engine.Engine SELECT a.id AS a_id, a.data AS a_data
FROM a
INFO sqlalchemy.engine.Engine [generated in 0.00009s] ()
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
INFO sqlalchemy.engine.Engine [generated in 0.00010s] (1,)
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
INFO sqlalchemy.engine.Engine [cached since 0.0005922s ago] (2,)
INFO sqlalchemy.engine.Engine SELECT b.id AS b_id, b.a_id AS b_a_id, b.data AS b_data
FROM b
WHERE ? = b.a_id
```

从我们上面的程序中，完整运行显示了总共四个不同的 SQL 字符串被缓存。这表明缓存大小为**四**将足够。这显然是一个极小的大小，默认大小为 500 可以保持不变。

### 缓存使用了多少内存？

前一节详细介绍了一些技术，以检查是否需要增大 `create_engine.query_cache_size`。我们如何知道缓存大小是否不太大？我们可能希望设置 `create_engine.query_cache_size` 不要大于某个数值，因为我们的应用可能会使用非常多不同的语句，例如从搜索 UX 动态构建查询的应用程序，如果在过去 24 小时内运行了十万个不同的查询并且它们都被缓存，我们不希望主机耗尽内存。

测量 Python 数据结构占用的内存量非常困难，然而，通过使用 `top` 进程测量内存增长的过程，当连续添加 250 个新语句到缓存中时，暗示一个中等大小的核心语句大约占用 12K，而一个小型 ORM 语句大约占用 20K，其中包括 ORM 的结果获取结构，后者会更大。

### 禁用或使用替代字典缓存一些（或全部）语句

使用的内部缓存称为 `LRUCache`，但这基本上只是一个字典。可以通过将 `Connection.execution_options.compiled_cache` 选项作为执行选项来使用任何字典作为任何一系列语句的缓存。执行选项可以在语句、`Engine` 或 `Connection` 上设置，以及在使用 SQLAlchemy-2.0 风格调用 ORM `Session.execute()` 方法时设置。例如，要运行一系列 SQL 语句并将它们缓存到特定字典中：

```py
my_cache = {}
with engine.connect().execution_options(compiled_cache=my_cache) as conn:
    conn.execute(table.select())
```

SQLAlchemy ORM 使用上述技术在工作单元“刷新”过程中保持每个映射器缓存，这些缓存与 `Engine` 上配置的默认缓存分开，以及一些关系加载器查询。

通过将该参数设置为 `None` 可以禁用缓存：

```py
# disable caching for this connection
with engine.connect().execution_options(compiled_cache=None) as conn:
    conn.execute(table.select())
```

### 第三方方言的缓存

缓存功能要求方言的编译器生成安全的 SQL 字符串，以便在给定特定缓存键的情况下重用许多语句调用，该缓存键与该 SQL 字符串的键对齐。这意味着语句中的任何字面值，例如 SELECT 的 LIMIT/OFFSET 值，不能在方言的编译方案中硬编码，因为编译后的字符串不可重复使用。SQLAlchemy 支持使用 `BindParameter.render_literal_execute()` 方法呈现绑定参数，该方法可以由自定义编译器应用于现有的 `Select._limit_clause` 和 `Select._offset_clause` 属性，在本节后面有所说明。

由于存在许多第三方方言，其中许多可能从 SQL 语句中生成字面值而不使用较新的“字面量执行”功能，因此从版本 1.4.5 起，SQLAlchemy 已向方言添加了一个名为 `Dialect.supports_statement_cache` 的属性。此属性在运行时直接在特定方言的类上检查其存在，即使它已经存在于超类上，因此即使第三方方言是现有可缓存的 SQLAlchemy 方言的子类，例如 `sqlalchemy.dialects.postgresql.PGDialect`，仍然必须明确包含此属性以启用缓存。该属性应仅在方言已根据需要进行更改并已测试对具有不同参数的编译 SQL 语句的可重用性后才能启用。

对于所有不支持此属性的第三方方言，该方言的日志将指示 `dialect does not support caching`。

当方言已经针对缓存进行了测试，并且特别是 SQL 编译器已经更新为不直接在 SQL 字符串中呈现任何字面 LIMIT / OFFSET 时，方言作者可以如下应用该属性：

```py
from sqlalchemy.engine.default import DefaultDialect

class MyDialect(DefaultDialect):
    supports_statement_cache = True
```

该标志还需要应用于方言的所有子类：

```py
class MyDBAPIForMyDialect(MyDialect):
    supports_statement_cache = True
```

新版本 1.4.5 中新增了 `Dialect.supports_statement_cache` 属性。

方言修改的典型案例如下。

#### 示例：使用后编译参数呈现 LIMIT / OFFSET

例如，假设方言覆盖了 `SQLCompiler.limit_clause()` 方法，该方法为 SQL 语句生成“LIMIT / OFFSET”子句，如下所示：

```py
# pre 1.4 style code
def limit_clause(self, select, **kw):
    text = ""
    if select._limit is not None:
        text += " \n LIMIT %d" % (select._limit,)
    if select._offset is not None:
        text += " \n OFFSET %d" % (select._offset,)
    return text
```

上述例程将`Select._limit`和`Select._offset`整数值呈现为嵌入在 SQL 语句中的文字整数。这是对于不支持在 SELECT 语句的 LIMIT/OFFSET 子句中使用绑定参数的数据库的常见要求。然而，在初始编译阶段呈现整数值直接**不兼容**缓存，因为`Select`对象的限制和偏移整数值不是缓存键的一部分，因此许多具有不同限制/偏移值的`Select`语句将无法正确呈现值。

以上代码的更正是将文字整数移至 SQLAlchemy 的后编译功能中，该功能将在初始编译阶段之外的执行时呈现文字整数，而是在将语句发送到 DBAPI 之前的执行时。这在编译阶段使用`BindParameter.render_literal_execute()`方法访问，同时使用`Select._limit_clause`和`Select._offset_clause`属性，这些属性表示 LIMIT/OFFSET 作为完整的 SQL 表达式，如下所示：

```py
# 1.4 cache-compatible code
def limit_clause(self, select, **kw):
    text = ""

    limit_clause = select._limit_clause
    offset_clause = select._offset_clause

    if select._simple_int_clause(limit_clause):
        text += " \n LIMIT %s" % (
            self.process(limit_clause.render_literal_execute(), **kw)
        )
    elif limit_clause is not None:
        # assuming the DB doesn't support SQL expressions for LIMIT.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for LIMIT"
        )
    if select._simple_int_clause(offset_clause):
        text += " \n OFFSET %s" % (
            self.process(offset_clause.render_literal_execute(), **kw)
        )
    elif offset_clause is not None:
        # assuming the DB doesn't support SQL expressions for OFFSET.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for OFFSET"
        )

    return text
```

上述方法将生成一个编译后的 SELECT 语句，看起来像：

```py
SELECT  x  FROM  y
LIMIT  __[POSTCOMPILE_param_1]
OFFSET  __[POSTCOMPILE_param_2]
```

在上述情况下，`__[POSTCOMPILE_param_1]`和`__[POSTCOMPILE_param_2]`指示符将在语句执行时填充其相应的整数值，此时 SQL 字符串已从缓存中检索出。

在适当进行类似上述更改之后，应将`Dialect.supports_statement_cache`标志设置为`True`。强烈建议第三方方言使用[dialect 第三方测试套件](https://github.com/sqlalchemy/sqlalchemy/blob/main/README.dialects.rst)，该套件将断言具有正确呈现和缓存的带有 LIMIT/OFFSET 的 SELECT 等操作。

另请参阅

升级到 1.4 和/或 2.x 后，为什么我的应用程序变慢？ - 在常见问题部分

#### 例子：使用后编译参数呈现 LIMIT / OFFSET

举例来说，假设一个方言重写了`SQLCompiler.limit_clause()`方法，该方法为 SQL 语句生成“LIMIT / OFFSET”子句，如下所示：

```py
# pre 1.4 style code
def limit_clause(self, select, **kw):
    text = ""
    if select._limit is not None:
        text += " \n LIMIT %d" % (select._limit,)
    if select._offset is not None:
        text += " \n OFFSET %d" % (select._offset,)
    return text
```

上述例程将把 `Select._limit` 和 `Select._offset` 整数值呈现为嵌入在 SQL 语句中的字面整数。这是对于不支持在 SELECT 语句的 LIMIT/OFFSET 子句中使用绑定参数的数据库的常见要求。然而，将整数值呈现在初始编译阶段直接**与缓存不兼容**，因为 `Select` 对象的限制和偏移整数值不是缓存键的一部分，因此许多具有不同限制/偏移值的 `Select` 语句不会呈现正确的值。

以上代码的修正是将字面整数移动到 SQLAlchemy 的 post-compile 设施中，这将在初始编译阶段之外呈现字面整数，而是在执行时间之前将语句发送到 DBAPI。这在编译阶段使用 `BindParameter.render_literal_execute()` 方法访问，结合使用 `Select._limit_clause` 和 `Select._offset_clause` 属性，这些属性表示 LIMIT/OFFSET 作为完整 SQL 表达式，如下所示：

```py
# 1.4 cache-compatible code
def limit_clause(self, select, **kw):
    text = ""

    limit_clause = select._limit_clause
    offset_clause = select._offset_clause

    if select._simple_int_clause(limit_clause):
        text += " \n LIMIT %s" % (
            self.process(limit_clause.render_literal_execute(), **kw)
        )
    elif limit_clause is not None:
        # assuming the DB doesn't support SQL expressions for LIMIT.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for LIMIT"
        )
    if select._simple_int_clause(offset_clause):
        text += " \n OFFSET %s" % (
            self.process(offset_clause.render_literal_execute(), **kw)
        )
    elif offset_clause is not None:
        # assuming the DB doesn't support SQL expressions for OFFSET.
        # Otherwise render here normally
        raise exc.CompileError(
            "dialect 'mydialect' can only render simple integers for OFFSET"
        )

    return text
```

上述方法将生成一个编译后的 SELECT 语句，如下所示：

```py
SELECT  x  FROM  y
LIMIT  __[POSTCOMPILE_param_1]
OFFSET  __[POSTCOMPILE_param_2]
```

在上述情况下，`__[POSTCOMPILE_param_1]` 和 `__[POSTCOMPILE_param_2]` 指示符将在语句执行时用其对应的整数值填充，此时 SQL 字符串已从缓存中检索到。

在适当进行了类似上述更改之后，应将 `Dialect.supports_statement_cache` 标志设置为 `True`。强烈建议第三方方言使用 [方言第三方测试套件](https://github.com/sqlalchemy/sqlalchemy/blob/main/README.dialects.rst)，该测试套件将断言像带有 LIMIT/OFFSET 的 SELECT 语句的操作是否正确呈现和缓存。

另请参阅

为什么升级到 1.4 和/或 2.x 后我的应用变慢？ - 在 常见问题解答 部分

### 使用 Lambda 来显著提高语句生成的速度

深度合成

此技术通常在非常性能密集的情况下非必要，并且面向经验丰富的 Python 程序员。虽然相当简单直接，但涉及到不适合初学者 Python 开发者的元编程概念。Lambda 方法可以稍后应用于现有代码，而付出的努力很小。

Python 函数通常以 lambda 表达式的形式表达，可以用于生成可基于 lambda 函数本身的 Python 代码位置和 lambda 内的闭包变量进行缓存的 SQL 表达式。其原理是允许缓存不仅是 SQL 表达式构造的 SQL 字符串编译形式，这是 SQLAlchemy 在未使用 lambda 系统时的正常行为，还有 SQL 表达式构造本身的 Python 组合，这也具有一定程度的 Python 开销。

Lambda SQL 表达式特性可作为性能增强功能使用，也可选择性地用于 `with_loader_criteria()` ORM 选项中，以提供通用的 SQL 片段。

#### 概要

Lambda 语句使用 `lambda_stmt()` 函数构建，该函数返回一个 `StatementLambdaElement` 实例，它本身是可执行的语句构造。可以使用 Python 加法运算符 `+` 或者 `StatementLambdaElement.add_criteria()` 方法向对象添加额外的修改器和条件，从而提供更多选项。

假设 `lambda_stmt()` 构造被调用在一个期望在应用程序中多次使用的封闭函数或方法中，以便在第一次执行之后可以利用已缓存的编译 SQL。当 lambda 在 Python 的封闭函数内构建时，它也可能具有闭包变量，这对整个方法至关重要：

```py
from sqlalchemy import lambda_stmt

def run_my_statement(connection, parameter):
    stmt = lambda_stmt(lambda: select(table))
    stmt += lambda s: s.where(table.c.col == parameter)
    stmt += lambda s: s.order_by(table.c.id)

    return connection.execute(stmt)

with engine.connect() as conn:
    result = run_my_statement(some_connection, "some parameter")
```

在上述例子中，用于定义 SELECT 语句结构的三个 `lambda` 可调用对象仅被调用一次，并且生成的 SQL 字符串被缓存到引擎的编译缓存中。从那时起，可以多次调用 `run_my_statement()` 函数，而其中的 `lambda` 可调用对象不会被再次调用，仅用作缓存键以检索已经编译的 SQL。

注意

需要注意的是，当未使用 lambda 系统时，已经存在 SQL 缓存。Lambda 系统只是在每个 SQL 语句调用时增加了额外的工作减少层，通过缓存构建 SQL 构造本身并且使用更简单的缓存键。

#### Lambda 的快速指南

最重要的是，在 lambda SQL 系统中，重点是确保为 lambda 生成的缓存键和它将产生的 SQL 字符串之间永远不会出现不匹配。`LambdaElement` 和相关对象将运行和分析给定的 lambda，以计算在每次运行时应如何缓存它，试图检测任何潜在问题。基本准则包括：

+   **支持任何类型的语句** - 虽然预期 `select()` 构造是 `lambda_stmt()` 的主要用例，但诸如 `insert()` 和 `update()` 等 DML 语句同样可用：

    ```py
    def upd(id_, newname):
        stmt = lambda_stmt(lambda: users.update())
        stmt += lambda s: s.values(name=newname)
        stmt += lambda s: s.where(users.c.id == id_)
        return stmt

    with engine.begin() as conn:
        conn.execute(upd(7, "foo"))
    ```

+   **ORM 使用案例也得到直接支持** - `lambda_stmt()` 可完全适应 ORM 功能，并可直接与 `Session.execute()` 一起使用：

    ```py
    def select_user(session, name):
        stmt = lambda_stmt(lambda: select(User))
        stmt += lambda s: s.where(User.name == name)

        row = session.execute(stmt).first()
        return row
    ```

+   **绑定参数会自动适应** - 与 SQLAlchemy 以前的“烘焙查询”系统相比，lambda SQL 系统会自动适应成为 SQL 绑定参数的 Python 文本值。这意味着即使给定的 lambda 只运行一次，但成为绑定参数的值是从 lambda 的 **闭包** 中提取的，每次运行都会提取：

    ```py
    >>> def my_stmt(x, y):
    ...     stmt = lambda_stmt(lambda: select(func.max(x, y)))
    ...     return stmt
    >>> engine = create_engine("sqlite://", echo=True)
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    ...     print(conn.scalar(my_stmt(12, 8)))
    SELECT  max(?,  ?)  AS  max_1
    [generated  in  0.00057s]  (5,  10)
    10
    SELECT  max(?,  ?)  AS  max_1
    [cached  since  0.002059s  ago]  (12,  8)
    12
    ```

    在上面的例子中，`StatementLambdaElement` 从每次调用 `my_stmt()` 时生成的 lambda 的 **闭包** 中提取了 `x` 和 `y` 的值；这些值被替换为参数的值，并缓存到 SQL 构造中。

+   **理想情况下，lambda 应该在所有情况下产生相同的 SQL 结构** - 避免在 lambda 内部使用条件语句或自定义可调用对象，这可能会根据输入产生不同的 SQL；如果一个函数可能会有条件地使用两个不同的 SQL 片段，那么请使用两个单独的 lambda：

    ```py
    # **Don't** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        stmt += lambda s: (
            s.where(table.c.x > parameter) if thing else s.where(table.c.y == parameter)
        )
        return stmt

    # **Do** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        if thing:
            stmt += lambda s: s.where(table.c.x > parameter)
        else:
            stmt += lambda s: s.where(table.c.y == parameter)
        return stmt
    ```

    如果 lambda 未产生一致的 SQL 构造，可能会发生各种失败，并且有些失败目前并不容易检测到。

+   **不要在 lambda 内部使用函数来产生绑定值** - 绑定值跟踪方法要求 SQL 语句中要使用的实际值在 lambda 的闭包中是本地存在的。如果值是从其他函数生成的，则这是不可能的，并且`LambdaElement`通常应该在尝试这样做时引发错误：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     stmt = lambda_stmt(lambda: select(func.max(get_x(), get_y())))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Can't invoke Python callable get_x()
    inside of lambda expression argument at
    <code object <lambda> at 0x7fed15f350e0, file "<stdin>", line 6>;
    lambda SQL constructs should not invoke functions from closure variables
    to produce literal values since the lambda SQL system normally extracts
    bound values without actually invoking the lambda or any functions within it.
    ```

    上述情况下，如果需要使用`get_x()`和`get_y()`，应该在 lambda 外部**定义**并分配给一个本地闭包变量：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     x_param, y_param = get_x(), get_y()
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

+   **避免在 lambda 内部引用非 SQL 构造，因为它们默认情况下不能被缓存** - 这个问题涉及到`LambdaElement`如何从语句中的其他闭包变量创建缓存键。为了提供准确的缓存键保证，lambda 闭包中的所有对象都被认为是重要的，而且默认情况下不会被假定适合作为缓存键。因此，以下示例也会引发一个相当详细的错误消息：

    ```py
    >>> class Foo:
    ...     def __init__(self, x, y):
    ...         self.x = x
    ...         self.y = y
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(lambda: select(func.max(foo.x, foo.y)))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(Foo(5, 10))))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Closure variable named 'foo' inside of
    lambda callable <code object <lambda> at 0x7fed15f35450, file
    "<stdin>", line 2> does not refer to a cacheable SQL element, and also
    does not appear to be serving as a SQL literal bound value based on the
    default SQL expression returned by the function.  This variable needs to
    remain outside the scope of a SQL-generating lambda so that a proper cache
    key may be generated from the lambda's state.  Evaluate this variable
    outside of the lambda, set track_on=[<elements>] to explicitly select
    closure elements to track, or set track_closure_variables=False to exclude
    closure variables from being part of the cache key.
    ```

    上述错误表明`LambdaElement`不会假定传递的`Foo`对象在所有情况下都会继续以相同的方式工作。它也不会假定默认情况下可以将`Foo`用作缓存键的一部分；如果它要将`Foo`对象用作缓存键，如果有许多不同的`Foo`对象，这将填满缓存重复信息，并且还将长期持有对所有这些对象的引用。

    解决上述情况的最佳方法是不要在 lambda 内部引用`foo`，而是**在外部**引用：

    ```py
    >>> def my_stmt(foo):
    ...     x_param, y_param = foo.x, foo.y
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

    在某些情况下，如果 lambda 的 SQL 结构保证不会根据输入而改变，则可以传递`track_closure_variables=False`，这将禁用除绑定参数之外的任何闭包变量的跟踪：

    ```py
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(func.max(foo.x, foo.y)), track_closure_variables=False
    ...     )
    ...     return stmt
    ```

    还有一个选项是将对象添加到元素中，明确形成缓存键的一部分，使用`track_on`参数；使用该参数允许特定值作为缓存键，并且还将防止考虑其他闭包变量。这对于 SQL 的一部分是来自某种上下文对象的情况很有用，该对象可能具有许多不同的值。在下面的示例中，SELECT 语句的第一个段将禁用对`foo`变量的跟踪，而第二个段将明确跟踪`self`作为缓存键的一部分：

    ```py
    >>> def my_stmt(self, foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(*self.column_expressions), track_closure_variables=False
    ...     )
    ...     stmt = stmt.add_criteria(lambda: self.where_criteria, track_on=[self])
    ...     return stmt
    ```

    使用`track_on`意味着给定的对象将长期存储在 lambda 的内部缓存中，并且只要缓存不清除这些对象（默认情况下使用 1000 个条目的 LRU 方案）就会有强引用。

#### 缓存键生成

为了理解 lambda SQL 构造中发生的一些选项和行为，了解缓存系统是有帮助的。

SQLAlchemy 的缓存系统通常通过生成一个表示构造内所有状态的结构来从给定的 SQL 表达式构造生成一个缓存键：

```py
>>> from sqlalchemy import select, column
>>> stmt = select(column("q"))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)  # somewhat paraphrased
CacheKey(key=(
 '0',
 <class 'sqlalchemy.sql.selectable.Select'>,
 '_raw_columns',
 (
 (
 '1',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 ),
 # a few more elements are here, and many more for a more
 # complicated SELECT statement
),)
```

上述键存储在本质上是一个字典的缓存中，值是一个构造，其中包括 SQL 语句的字符串形式，本例中是短语 “SELECT q”。我们可以观察到，即使对于一个非常简短的查询，缓存键也相当冗长，因为它必须表示关于正在渲染和潜在执行的一切变化。

与此相反，lambda 构造系统创建了一种不同类型的缓存键：

```py
>>> from sqlalchemy import lambda_stmt
>>> stmt = lambda_stmt(lambda: select(column("q")))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)
CacheKey(key=(
 <code object <lambda> at 0x7fed1617c710, file "<stdin>", line 1>,
 <class 'sqlalchemy.sql.lambdas.StatementLambdaElement'>,
),)
```

在上面，我们看到的缓存键远远比非 lambda 语句的键要短得多，并且甚至生产 `select(column("q"))` 构造本身也是不必要的；Python lambda 本身包含一个称为 `__code__` 的属性，该属性引用应用程序运行时中不可变且永久的 Python 代码对象。

当 lambda 还包含闭包变量时，在这些变量引用 SQL 构造（如列对象）的常规情况下，它们将成为缓存键的一部分；或者如果它们引用将成为绑定参数的文字值，则它们将放置在缓存键的一个单独元素中：

```py
>>> def my_stmt(parameter):
...     col = column("q")
...     stmt = lambda_stmt(lambda: select(col))
...     stmt += lambda s: s.where(col == parameter)
...     return stmt
```

上述 `StatementLambdaElement` 包括两个 lambda，两者都引用 `col` 闭包变量，因此缓存键将表示这两个段以及 `column()` 对象：

```py
>>> stmt = my_stmt(5)
>>> key = stmt._generate_cache_key()
>>> print(key)
CacheKey(key=(
 <code object <lambda> at 0x7f07323c50e0, file "<stdin>", line 3>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 <code object <lambda> at 0x7f07323c5190, file "<stdin>", line 4>,
 <class 'sqlalchemy.sql.lambdas.LinkedLambdaElement'>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
),)
```

缓存键的第二部分已检索到将在调用语句时使用的绑定参数：

```py
>>> key.bindparams
[BindParameter('%(139668884281280 parameter)s', 5, type_=Integer())]
```

关于“lambda”缓存的一系列示例及性能比较，请参见性能示例中的“short_selects”测试套件。

#### 概要

Lambda 语句使用 `lambda_stmt()` 函数构建，该函数返回一个 `StatementLambdaElement` 实例，它本身是一个可执行语句构造。使用 Python 加法运算符 `+` 或者 `StatementLambdaElement.add_criteria()` 方法可以向对象添加其他修饰符和条件。

假定 `lambda_stmt()` 构造被调用在一个期望在应用程序中被多次使用的封闭函数或方法内部，以便后续的执行除了第一次之外都可以利用已编译的 SQL 的缓存。当 lambda 在 Python 的封闭函数内部构造时，它也受到闭包变量的影响，这对整个方法都是重要的：

```py
from sqlalchemy import lambda_stmt

def run_my_statement(connection, parameter):
    stmt = lambda_stmt(lambda: select(table))
    stmt += lambda s: s.where(table.c.col == parameter)
    stmt += lambda s: s.order_by(table.c.id)

    return connection.execute(stmt)

with engine.connect() as conn:
    result = run_my_statement(some_connection, "some parameter")
```

在上面的例子中，用于定义 SELECT 语句结构的三个 `lambda` 可调用对象仅被调用一次，并且生成的 SQL 字符串被缓存到引擎的编译缓存中。从那时起，`run_my_statement()` 函数可以被调用任意次数，而其中的 `lambda` 可调用对象将不会被调用，只会被用作缓存键来检索已经编译的 SQL。

注意

注意，当未使用 lambda 系统时，已经存在 SQL 缓存。lambda 系统只是在每个 SQL 语句调用时添加了一个额外的工作减少层，通过缓存 SQL 构造的构建以及使用一个更简单的缓存键。

#### Lambdas 的快速指南

最重要的是，在 lambda SQL 系统中，重点是确保生成的 lambda 的缓存键与它将产生的 SQL 字符串之间永远不会不匹配。`LambdaElement` 和相关对象将运行和分析给定的 lambda，以便计算应该在每次运行时如何缓存它，试图检测任何潜在问题。基本指南包括：

+   **支持任何类型的语句** - 虽然预期 `select()` 构造是 `lambda_stmt()` 的主要用例，但诸如 `insert()` 和 `update()` 的 DML 语句同样可用：

    ```py
    def upd(id_, newname):
        stmt = lambda_stmt(lambda: users.update())
        stmt += lambda s: s.values(name=newname)
        stmt += lambda s: s.where(users.c.id == id_)
        return stmt

    with engine.begin() as conn:
        conn.execute(upd(7, "foo"))
    ```

+   **ORM 用例直接支持** - `lambda_stmt()` 完全可以容纳 ORM 功能，并直接与 `Session.execute()` 一起使用：

    ```py
    def select_user(session, name):
        stmt = lambda_stmt(lambda: select(User))
        stmt += lambda s: s.where(User.name == name)

        row = session.execute(stmt).first()
        return row
    ```

+   **绑定参数会自动适应** - 与 SQLAlchemy 以前的“烘焙查询”系统相比，lambda SQL 系统会自动适应成为 SQL 绑定参数的 Python 文字值。这意味着即使给定的 lambda 只运行一次，成为绑定参数的值也会在每次运行时从 lambda 的**闭包**中提取出来：

    ```py
    >>> def my_stmt(x, y):
    ...     stmt = lambda_stmt(lambda: select(func.max(x, y)))
    ...     return stmt
    >>> engine = create_engine("sqlite://", echo=True)
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    ...     print(conn.scalar(my_stmt(12, 8)))
    SELECT  max(?,  ?)  AS  max_1
    [generated  in  0.00057s]  (5,  10)
    10
    SELECT  max(?,  ?)  AS  max_1
    [cached  since  0.002059s  ago]  (12,  8)
    12
    ```

    上面，`StatementLambdaElement` 从每次调用 `my_stmt()` 时生成的 lambda 的**闭包**中提取了 `x` 和 `y` 的值；这些值被替换为参数的值，并缓存在 SQL 构造中。

+   **lambda 最好在所有情况下生成相同的 SQL 结构** - 避免在 lambda 内部使用条件语句或自定义可调用对象，这些可能会基于输入生成不同的 SQL；如果函数可能会有条件地使用两个不同的 SQL 片段，请使用两个单独的 lambda：

    ```py
    # **Don't** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        stmt += lambda s: (
            s.where(table.c.x > parameter) if thing else s.where(table.c.y == parameter)
        )
        return stmt

    # **Do** do this:

    def my_stmt(parameter, thing=False):
        stmt = lambda_stmt(lambda: select(table))
        if thing:
            stmt += lambda s: s.where(table.c.x > parameter)
        else:
            stmt += lambda s: s.where(table.c.y == parameter)
        return stmt
    ```

    如果 lambda 表达式不能生成一致的 SQL 结构，可能会发生各种故障，其中一些目前并不容易检测到。

+   **不要在 lambda 内部使用函数生成绑定值** - 绑定值跟踪方法要求 SQL 语句中要使用的实际值在 lambda 的闭包中局部存在。如果值是从其他函数生成的，则这是不可能的，并且如果尝试这样做，`LambdaElement` 通常应该引发错误：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     stmt = lambda_stmt(lambda: select(func.max(get_x(), get_y())))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(5, 10)))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Can't invoke Python callable get_x()
    inside of lambda expression argument at
    <code object <lambda> at 0x7fed15f350e0, file "<stdin>", line 6>;
    lambda SQL constructs should not invoke functions from closure variables
    to produce literal values since the lambda SQL system normally extracts
    bound values without actually invoking the lambda or any functions within it.
    ```

    上面，如果必要，应该在 lambda 外部使用 `get_x()` 和 `get_y()`，并将其分配给本地闭包变量：

    ```py
    >>> def my_stmt(x, y):
    ...     def get_x():
    ...         return x
    ...
    ...     def get_y():
    ...         return y
    ...
    ...     x_param, y_param = get_x(), get_y()
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

+   **避免在 lambda 内部引用非 SQL 结构，因为它们默认情况下无法缓存** - 此问题涉及 `LambdaElement` 如何从语句中的其他闭包变量创建缓存键。为了提供对准确缓存键的最佳保证， lambda 中闭包中的所有对象都被认为是重要的，且默认情况下不会假设适用于缓存键。因此，以下示例也会引发相当详细的错误消息：

    ```py
    >>> class Foo:
    ...     def __init__(self, x, y):
    ...         self.x = x
    ...         self.y = y
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(lambda: select(func.max(foo.x, foo.y)))
    ...     return stmt
    >>> with engine.connect() as conn:
    ...     print(conn.scalar(my_stmt(Foo(5, 10))))
    Traceback (most recent call last):
     # ...
    sqlalchemy.exc.InvalidRequestError: Closure variable named 'foo' inside of
    lambda callable <code object <lambda> at 0x7fed15f35450, file
    "<stdin>", line 2> does not refer to a cacheable SQL element, and also
    does not appear to be serving as a SQL literal bound value based on the
    default SQL expression returned by the function.  This variable needs to
    remain outside the scope of a SQL-generating lambda so that a proper cache
    key may be generated from the lambda's state.  Evaluate this variable
    outside of the lambda, set track_on=[<elements>] to explicitly select
    closure elements to track, or set track_closure_variables=False to exclude
    closure variables from being part of the cache key.
    ```

    上述错误表明 `LambdaElement` 不会假设传入的 `Foo` 对象在所有情况下都会保持相同的行为。它也不会假设默认情况下可以将 `Foo` 用作缓存键的一部分；如果将 `Foo` 对象用作缓存键的一部分，如果有许多不同的 `Foo` 对象，这将填满缓存并且还会对所有这些对象保持长时间的引用。

    解决上述情况的最佳方法是不在 Lambda 内部引用 `foo`，而是在**外部**引用它：

    ```py
    >>> def my_stmt(foo):
    ...     x_param, y_param = foo.x, foo.y
    ...     stmt = lambda_stmt(lambda: select(func.max(x_param, y_param)))
    ...     return stmt
    ```

    在某些情况下，如果 Lambda 的 SQL 结构保证不会根据输入改变，可以传递 `track_closure_variables=False`，这将禁用对除了用于绑定参数的变量之外的任何闭包变量的跟踪：

    ```py
    >>> def my_stmt(foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(func.max(foo.x, foo.y)), track_closure_variables=False
    ...     )
    ...     return stmt
    ```

    还有一个选项可以将对象添加到元素中，以明确形成缓存键的一部分，使用 `track_on` 参数；使用此参数允许特定值作为缓存键，并且还会阻止其他闭包变量被考虑。这对于 SQL 的一部分构造源自某种上下文对象且可能具有许多不同值的情况非常有用。在下面的示例中，SELECT 语句的第一个片段将禁用对 `foo` 变量的跟踪，而第二个片段将明确跟踪 `self` 作为缓存键的一部分：

    ```py
    >>> def my_stmt(self, foo):
    ...     stmt = lambda_stmt(
    ...         lambda: select(*self.column_expressions), track_closure_variables=False
    ...     )
    ...     stmt = stmt.add_criteria(lambda: self.where_criteria, track_on=[self])
    ...     return stmt
    ```

    使用 `track_on` 意味着给定对象将长期存储在 Lambda 的内部缓存中，并且只要缓存不清除这些对象（默认使用 1000 条记录的 LRU 方案），它们就会具有强引用。

#### 缓存键生成

为了理解 Lambda SQL 构造中发生的一些选项和行为，了解缓存系统是有帮助的。

SQLAlchemy 的缓存系统通常通过生成一个表示构造内所有状态的结构来从给定的 SQL 表达式构造生成缓存键：

```py
>>> from sqlalchemy import select, column
>>> stmt = select(column("q"))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)  # somewhat paraphrased
CacheKey(key=(
 '0',
 <class 'sqlalchemy.sql.selectable.Select'>,
 '_raw_columns',
 (
 (
 '1',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 ),
 # a few more elements are here, and many more for a more
 # complicated SELECT statement
),)
```

上述键存储在基本上是字典的缓存中，而值是一个构造，其中包括 SQL 语句的字符串形式，本例中是短语 “SELECT q”。我们可以观察到，即使是一个极短的查询，缓存键也非常冗长，因为它必须表示关于正在渲染和潜在执行的所有可能变化的内容。

相比之下，Lambda 构造系统创建了一种不同类型的缓存键：

```py
>>> from sqlalchemy import lambda_stmt
>>> stmt = lambda_stmt(lambda: select(column("q")))
>>> cache_key = stmt._generate_cache_key()
>>> print(cache_key)
CacheKey(key=(
 <code object <lambda> at 0x7fed1617c710, file "<stdin>", line 1>,
 <class 'sqlalchemy.sql.lambdas.StatementLambdaElement'>,
),)
```

以上是一个缓存键，远比非 Lambda 语句的要短得多，而且此处生产 `select(column("q"))` 构造本身甚至都不是必要的；Python Lambda 本身包含一个名为 `__code__` 的属性，指向一个在应用程序运行时不可变且永久存在的 Python 代码对象。

当 Lambda 也包含闭包变量时，在这些变量引用 SQL 构造（如列对象）的常规情况下，它们成为缓存键的一部分，或者如果它们引用将作为绑定参数的文字值，则将它们放在缓存键的一个单独元素中：

```py
>>> def my_stmt(parameter):
...     col = column("q")
...     stmt = lambda_stmt(lambda: select(col))
...     stmt += lambda s: s.where(col == parameter)
...     return stmt
```

上述 `StatementLambdaElement` 包含两个 lambda，两者都引用了 `col` 闭包变量，因此缓存键将表示这两个段以及 `column()` 对象。

```py
>>> stmt = my_stmt(5)
>>> key = stmt._generate_cache_key()
>>> print(key)
CacheKey(key=(
 <code object <lambda> at 0x7f07323c50e0, file "<stdin>", line 3>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 <code object <lambda> at 0x7f07323c5190, file "<stdin>", line 4>,
 <class 'sqlalchemy.sql.lambdas.LinkedLambdaElement'>,
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
 (
 '0',
 <class 'sqlalchemy.sql.elements.ColumnClause'>,
 'name',
 'q',
 'type',
 (
 <class 'sqlalchemy.sql.sqltypes.NullType'>,
 ),
 ),
),)
```

缓存键的第二部分已检索出在调用语句时将使用的绑定参数：

```py
>>> key.bindparams
[BindParameter('%(139668884281280 parameter)s', 5, type_=Integer())]
```

有关使用性能比较的“lambda”缓存的一系列示例，请参见性能 示例中的 “short_selects” 测试套件。

## 插入语句的“插入多个值”行为

新功能 2.0 版本中：有关更改的背景，请参见 优化的 ORM 批量插入现在已针对除 MySQL 外的所有后端实现，其中包括示例性能测试

提示

insertmanyvalues 功能是一种**透明可用**的性能特性，无需最终用户进行干预即可按需发生。本节描述了该特性的架构以及如何衡量其性能并调整其行为，以优化批量 INSERT 语句的速度，特别是 ORM 中使用的情况。

随着越来越多的数据库支持 INSERT..RETURNING，SQLAlchemy 在处理需要获取服务器生成值的 INSERT 语句的方式上发生了重大变化，最重要的是服务器生成的主键值，它允许在后续操作中引用新行。特别是，这种情况长期以来一直是 ORM 中的一个重大性能问题，ORM 依赖于能够检索服务器生成的主键值，以便正确填充 identity map。

随着最近对 SQLite 和 MariaDB 添加了对 RETURNING 的支持，SQLAlchemy 不再需要依赖于大多数后端仅支持单行 [cursor.lastrowid](https://peps.python.org/pep-0249/#lastrowid) 属性提供的 DBAPI；现在可以对所有 SQLAlchemy 包含的 后端使用 RETURNING，除了 MySQL 外。剩下的性能限制是，[cursor.executemany()](https://peps.python.org/pep-0249/#executemany) DBAPI 方法不允许获取行，对于大多数后端来说，通过放弃使用 `executemany()`，而是重构单个 INSERT 语句以适应在单个语句中容纳大量行，并使用 `cursor.execute()` 调用该语句来解决。这种方法源自于 `psycopg2` DBAPI 的 [psycopg2 快速执行辅助功能](https://www.psycopg.org/docs/extras.html#fast-execution-helpers) 特性，SQLAlchemy 在最近的发布系列中逐渐增加了对其的更多支持。

### 当前支持

该功能对支持 RETURNING 的 SQLAlchemy 中的所有后端启用，但 Oracle 是个例外，因为 cx_Oracle 和 OracleDB 驱动程序都提供了自己的等效功能。该功能通常在使用 `Insert.returning()` 方法与 executemany 执行结合使用时发生，这发生在将字典列表传递给 `Connection.execute.parameters` 参数的 `Connection.execute()` 或 `Session.execute()` 方法（以及 asyncio 下的等效方法和类似 `Session.scalars()` 的速记方法）。在使用方法如 `Session.add()` 和 `Session.add_all()` 向表中添加行时，它也在 ORM 工作单元 过程中发生。

对于 SQLAlchemy 的包含方言，支持或等效支持目前如下：

+   SQLite - 对 SQLite 版本 3.35 及以上提供支持

+   PostgreSQL - 所有支持的 Postgresql 版本（9 及以上）

+   SQL Server - 所有支持的 SQL Server 版本 [[1]](#id2)

+   MariaDB - 对 MariaDB 版本 10.5 及以上提供支持

+   MySQL - 没有支持，没有 RETURNING 功能

+   Oracle - 支持 RETURNING，使用本机 cx_Oracle / OracleDB API 和 executemany，适用于所有支持的 Oracle 版本 9 及以上，使用多行 OUT 参数。这与“executemanyvalues”不是同一实现，但具有相同的使用模式和等效性能优势。

从版本 2.0.10 开始更改：

### 禁用该功能

要为特定后端禁用“insertmanyvalues”功能，可以将 `create_engine.use_insertmanyvalues` 参数传递为 `False` 给 `create_engine()`：

```py
engine = create_engine(
    "mariadb+mariadbconnector://scott:tiger@host/db", use_insertmanyvalues=False
)
```

还可以通过将 `Table.implicit_returning` 参数传递为 `False` 来禁用对特定 `Table` 对象的隐式使用。

```py
t = Table(
    "t",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x", Integer),
    implicit_returning=False,
)
```

禁用 RETURNING 对于特定表格的原因是为了解决特定后端的限制。

### 批量模式操作

这个特性有两种操作模式，根据方言和`Table`选择透明模式。其中一种是**批量模式**，通过重写形如以下的 INSERT 语句来减少数据库往返次数：

```py
INSERT  INTO  a  (data,  x,  y)  VALUES  (%(data)s,  %(x)s,  %(y)s)  RETURNING  a.id
```

转换为“批量”形式，如下：

```py
INSERT  INTO  a  (data,  x,  y)  VALUES
  (%(data_0)s,  %(x_0)s,  %(y_0)s),
  (%(data_1)s,  %(x_1)s,  %(y_1)s),
  (%(data_2)s,  %(x_2)s,  %(y_2)s),
  ...
  (%(data_78)s,  %(x_78)s,  %(y_78)s)
RETURNING  a.id
```

在上面的语句中，语句是针对输入数据的一个子集（一个“批量”）组织的，其大小由数据库后端以及每个批次中参数的数量确定，以对应于已知的语句大小/参数数量限制。然后，该特性对每个输入数据批次执行一次 INSERT 语句，直到所有记录都被使用，将每个批次的 RETURNING 结果连接成一个单一的大型结果集，可从单个`Result` 对象获取。

这种“批量”形式允许使用更少的数据库往返次数插入多行，并且已经证明在大多数支持的后端中可以实现显着的性能提升。

### 将 RETURNING 行与参数集相关联

2.0.10 版本中的新功能。

在上一节中示例的“批量”模式查询不保证返回的记录顺序与输入数据的顺序相对应。当由 SQLAlchemy ORM 工作单元 进程使用时，以及与将返回的服务器生成的值与输入数据进行关联的应用程序一起使用时，`Insert.returning()` 和 `UpdateBase.return_defaults()` 方法包括一个选项 `Insert.returning.sort_by_parameter_order`，表示“insertmanyvalues”模式应保证此对应关系。这与实际由数据库后端实际插入记录的顺序无关，这在任何情况下都不被假定；只有在接收返回的记录时，返回的记录应该被组织起来，以对应于原始输入数据传递的顺序。

当 `Insert.returning.sort_by_parameter_order` 参数存在时，对于使用服务器生成的整数主键值的表，如 `IDENTITY`、PostgreSQL `SERIAL`、MariaDB `AUTO_INCREMENT` 或 SQLite 的 `ROWID` 方案，"batch" 模式可能会选择使用更复杂的 INSERT..RETURNING 形式，并结合基于返回值的行的后执行排序，或者如果不存在这样的形式，"insertmanyvalues" 功能可能会优雅地降级到 "non-batched" 模式，为每个参数集运行单独的 INSERT 语句。

例如，在 SQL Server 上，当自动增量的 `IDENTITY` 列用作主键时，将使用以下 SQL 形式：

```py
INSERT  INTO  a  (data,  x,  y)
OUTPUT  inserted.id,  inserted.id  AS  id__1
SELECT  p0,  p1,  p2  FROM  (VALUES
  (?,  ?,  ?,  0),  (?,  ?,  ?,  1),  (?,  ?,  ?,  2),
  ...
  (?,  ?,  ?,  77)
)  AS  imp_sen(p0,  p1,  p2,  sen_counter)  ORDER  BY  sen_counter
```

当主键列使用 SERIAL 或 IDENTITY 时，PostgreSQL 也使用类似的形式。上述形式**不**保证插入行的顺序。但是，它确保 IDENTITY 或 SERIAL 值将与每个参数集按顺序创建[[2]](#id5)。然后，"insertmanyvalues" 功能通过递增整数标识对上述 INSERT 语句的返回行进行排序。

对于 SQLite 数据库，没有适当的 INSERT 形式可以将新的 ROWID 值的生成与传递的参数集的顺序相关联。因此，当使用服务器生成的主键值时，当请求有序 RETURNING 时，SQLite 后端将降级为 "non-batched" 模式。对于 MariaDB，默认的 INSERT 形式由 insertmanyvalues 使用，因为此数据库后端在使用 InnoDB 时会将 AUTO_INCREMENT 的顺序与输入数据的顺序对齐[[3]](#id6)。

对于客户端生成的主键，例如使用 Python 的 `uuid.uuid4()` 函数为 `Uuid` 列生成新值时，"insertmanyvalues" 功能会透明地将此列包含在 RETURNING 记录中，并将其值与给定输入记录的值相关联，从而保持输入记录和结果行之间的对应关系。由此可见，当使用客户端生成的主键值时，所有后端都允许批量、参数相关的 RETURNING 顺序。

“insertmanyvalues” “batch” 模式确定用作输入参数和 RETURNING 行之间对应点的列或列的主题被称为 insert sentinel，这是一种特定的列或列，用于跟踪此类值。通常会自动选择“insert sentinel”，但也可以为极端特殊情况进行用户配置；章节 配置 Sentinel 列 描述了这一点。

对于没有提供适当的 INSERT 形式以确定性地提供与输入值对齐的服务器生成值的后端，或对于具有其他类型的服务器生成主键值的 `Table` 配置，当请求保证 RETURNING 排序时，“insertmanyvalues” 模式将在需要时使用 **非批量** 模式。

另请参阅

> +   Microsoft SQL Server 的原理
> +   
>     “使用 SELECT 结合 ORDER BY 来填充行的 INSERT 查询保证了如何计算标识值，但不保证插入行的顺序。” [`learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql?view=sql-server-ver16#limitations-and-restrictions`](https://learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql?view=sql-server-ver16#limitations-and-restrictions)
>     
> +   PostgreSQL 批量插入讨论
> +   
>     2018 年的原始描述 [`www.postgresql.org/message-id/29386.1528813619@sss.pgh.pa.us`](https://www.postgresql.org/message-id/29386.1528813619@sss.pgh.pa.us)
>     
>     2023 年的跟进 - [`www.postgresql.org/message-id/be108555-da2a-4abc-a46b-acbe8b55bd25%40app.fastmail.com`](https://www.postgresql.org/message-id/be108555-da2a-4abc-a46b-acbe8b55bd25%40app.fastmail.com)

+   MariaDB AUTO_INCREMENT 行为（使用与 MySQL 相同的 InnoDB 引擎）：

    [`dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html`](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)

    [`dba.stackexchange.com/a/72099`](https://dba.stackexchange.com/a/72099)  ### 非批量模式操作

对于没有客户端主键值并提供服务器生成主键值（或没有主键）的 `Table` 配置，并且数据库无法根据多个参数集以确定性或可排序的方式调用的情况，“insertmanyvalues” 特性在为 `Insert` 语句满足 `Insert.returning.sort_by_parameter_order` 要求时可能选择使用 **非批量** 模式。

在这种模式下，保留了原始的 SQL INSERT 形式，并且 “insertmanyvalues” 特性将为每个参数集单独运行语句，将返回的行组织成完整的结果集。与之前的 SQLAlchemy 版本不同，它通过最小化 Python 开销来紧密循环执行。在某些情况下，例如在 SQLite 上，“非批量” 模式的性能与 “批量” 模式完全相同。

### 语句执行模型

无论是“批量”模式还是“非批量”模式，该功能都会必然使用 DBAPI `cursor.execute()` 方法调用 **多个 INSERT 语句**，在 **单个** 调用核心级别 `Connection.execute()` 方法的范围内，每个语句包含多达一组固定参数的限制。如下所述，此限制可以配置在 控制批处理大小 处。对 `cursor.execute()` 的单独调用将分别记录，并且也将分别传递给事件监听器，例如 `ConnectionEvents.before_cursor_execute()`（请参阅下面的 日志和事件）。

#### 配置哨兵列

在典型情况下，“insertmanyvalues”功能为了提供带有确定性行顺序的 INSERT..RETURNING 将自动从给定表的主键中确定哨兵列，如果无法识别，则会优雅地降级为“逐行”模式。作为一个完全 **可选** 的功能，为了获取对于具有服务器生成的主键的表的完整“insertmanyvalues”批量性能，其默认生成器函数与“哨兵”用例不兼容，其他非主键列可以被标记为“哨兵”列，假设它们符合某些要求。一个典型的例子是具有客户端默认值的非主键 `Uuid` 列，例如 Python `uuid.uuid4()` 函数。还有一种构造用于创建简单的整数列，其具有面向“insertmanyvalues”用例的客户端整数计数器。

可以通过在合格列上添加 `Column.insert_sentinel` 来指示哨兵列。最基本的“合格”列是一个非空唯一列，具有客户端默认值，例如以下 UUID 列：

```py
import uuid

from sqlalchemy import Column
from sqlalchemy import FetchedValue
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid

my_table = Table(
    "some_table",
    metadata,
    # assume some arbitrary server-side function generates
    # primary key values, so cannot be tracked by a bulk insert
    Column("id", String(50), server_default=FetchedValue(), primary_key=True),
    Column("data", String(50)),
    Column(
        "uniqueid",
        Uuid(),
        default=uuid.uuid4,
        nullable=False,
        unique=True,
        insert_sentinel=True,
    ),
)
```

在使用 ORM 声明性模型时，可以使用 `mapped_column` 构造相同的表单：

```py
import uuid

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))
    uniqueid: Mapped[uuid.UUID] = mapped_column(
        default=uuid.uuid4, unique=True, insert_sentinel=True
    )
```

虽然默认生成器生成的值 **必须** 是唯一的，但上述“哨兵”列上的实际 UNIQUE 约束，由 `unique=True` 参数指示，本身是可选的，如果不需要可以省略。

还有一种特殊形式的“插入哨兵”，它是一个专用的可空整数列，利用一个特殊的默认整数计数器，仅在“insertmanyvalues”操作期间使用；作为额外的行为，该列将在 SQL 语句和结果集中省略自身，并以基本透明的方式行为。但是，它确实需要在实际数据库表中物理存在。可以使用`insert_sentinel()`函数构建这种`Column`的风格：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid
from sqlalchemy import insert_sentinel

Table(
    "some_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String(50)),
    insert_sentinel("sentinel"),
)
```

在使用 ORM Declarative 时，可以使用 Declarative 友好版本的`insert_sentinel()`，称为`orm_insert_sentinel()`，它可以用于 Base 类或 mixin；如果使用`declared_attr()`打包，该列将应用于所有绑定表的子类，包括连接继承层次结构：

```py
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import orm_insert_sentinel

class Base(DeclarativeBase):
    @declared_attr
    def _sentinel(cls) -> Mapped[int]:
        return orm_insert_sentinel()

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))

class MySubClass(MyClass):
    __tablename__ = "sub_table"

    id: Mapped[str] = mapped_column(ForeignKey("my_table.id"), primary_key=True)

class MySingleInhClass(MyClass):
    pass
```

在上面的示例中，“my_table”和“sub_table”都将有一个额外的整数列名为“_sentinel”，可以被“insertmanyvalues”功能使用，以帮助优化 ORM 使用的批量插入。### 控制批量大小

“insertmanyvalues”的一个关键特征是 INSERT 语句的大小受限于固定数量的“values”子句以及每次在一个 INSERT 语句中可以表示的特定方言固定总数的绑定参数。当给定的参数字典数量超过固定限制，或者当要在单个 INSERT 语句中呈现的绑定参数总数超过固定限制时（这两个固定限制是分开的），将在单个`Connection.execute()`调用范围内调用多个 INSERT 语句，每个 INSERT 语句都适应一部分参数字典，称为“批量”。每个“批量”中表示的参数字典数量称为“批量大小”。例如，批量大小为 500 意味着每个发出的 INSERT 语句最多插入 500 行。

能够调整批处理大小可能是很重要的，因为较大的批处理大小对于值集本身相对较小的插入可能更有效率，而较小的批处理大小可能更适合于使用非常大的值集的插入，其中渲染的 SQL 大小以及传递给一个语句的总数据大小可能受益于根据后端行为和内存约束限制到某个特定大小。因此，批处理大小可以在每个`Engine`以及每个语句的基础上进行配置。另一方面，参数限制是基于正在使用的数据库的已知特性固定的。

对于大多数后端，默认的批处理大小为 1000，还有一个每个方言的“最大参数数”限制因素，可能会在每个语句的基础上进一步减少批处理大小。最大参数数因方言和服务器版本而异；最大大小为 32700（选择了一个距离 PostgreSQL 限制的 32767 和 SQLite 现代限制的 32766 的健康距离，同时为语句中的额外参数以及 DBAPI 的怪癖留出空间）。较旧版本的 SQLite（3.32.0 之前）将此值设置为 999。MariaDB 没有确定的限制��但是 32700 仍然是 SQL 消息大小的限制因素。

“批处理大小”的值可以通过`create_engine.insertmanyvalues_page_size`参数在`Engine`范围内受到影响。例如，为了影响 INSERT 语句在每个语句中包含多达 100 个参数集：

```py
e = create_engine("sqlite://", insertmanyvalues_page_size=100)
```

批处理大小也可以通过`Connection.execution_options.insertmanyvalues_page_size`执行选项在每个语句的基础上受到影响，例如每次执行：

```py
with e.begin() as conn:
    result = conn.execute(
        table.insert().returning(table.c.id),
        parameterlist,
        execution_options={"insertmanyvalues_page_size": 100},
    )
```

或在语句本身上进行配置：

```py
stmt = (
    table.insert()
    .returning(table.c.id)
    .execution_options(insertmanyvalues_page_size=100)
)
with e.begin() as conn:
    result = conn.execute(stmt, parameterlist)
```  ### 日志记录和事件

“insertmanyvalues”功能与 SQLAlchemy 的语句日志记录以及游标事件完全集成，例如`ConnectionEvents.before_cursor_execute()`。当参数列表被分成单独的批次时，**每个 INSERT 语句都会被记录并单独传递给事件处理程序**。这与之前 SQLAlchemy 1.x 系列中仅适用于 psycopg2 的功能的工作方式相比是一个重大变化，其中多个 INSERT 语句的生成被隐藏在日志记录和事件之外。日志显示将截断长参数列表以便阅读，并且还将指示每个语句的特定批次。下面的示例说明了此日志的摘录：

```py
INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[generated in 0.00177s (insertmanyvalues) 1/10 (unordered)] ('d0', 0, 0, 'd1',  ...
INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[insertmanyvalues 2/10 (unordered)] ('d100', 100, 1000, 'd101', ...

...

INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[insertmanyvalues 10/10 (unordered)] ('d900', 900, 9000, 'd901', ...
```

当 非批处理模式 发生时，日志将指示此情况以及 insertmanyvalues 消息：

```py
...

INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 67/78 (ordered; batch not supported)] ('d66', 66, 66)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 68/78 (ordered; batch not supported)] ('d67', 67, 67)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 69/78 (ordered; batch not supported)] ('d68', 68, 68)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 70/78 (ordered; batch not supported)] ('d69', 69, 69)

...
```

另请参阅

配置日志记录

### Upsert 支持

PostgreSQL、SQLite 和 MariaDB 方言提供了特定于后端的“upsert” 构造 `insert()`、`insert()` 和 `insert()`，它们都是 `Insert` 构造，具有额外的方法，如 `on_conflict_do_update()` 或 ``on_duplicate_key()``。当它们与 RETURNING 一起使用时，这些构造还支持“insertmanyvalues”行为，允许有效地进行带 RETURNING 的 upsert 操作。

### 当前支持

该功能适用于 SQLAlchemy 中支持 RETURNING 的所有后端，但不包括 Oracle，因为 cx_Oracle 和 OracleDB 驱动程序都提供了自己的等效功能。当使用 `Insert.returning()` 方法与 executemany 执行结合使用时，该功能通常发生在将字典列表传递给 `Connection.execute.parameters` 参数的 `Connection.execute()` 或 `Session.execute()` 方法（以及 asyncio 和类似方法如 `Session.scalars()`）时。它还在使用诸如 `Session.add()` 和 `Session.add_all()` 等方法添加行时，在 ORM 工作单元 过程中发生。

对于 SQLAlchemy 包含的方言，支持或等效支持目前如下：

+   SQLite - 支持 SQLite 版本 3.35 及以上

+   PostgreSQL - 所有支持的 PostgreSQL 版本（9 及以上）

+   SQL Server - 所有支持的 SQL Server 版本 [[1]](#id2)

+   MariaDB - 支持 MariaDB 版本 10.5 及以上

+   MySQL - 不支持，没有 RETURNING 功能

+   Oracle - 使用本机 cx_Oracle / OracleDB API 支持 `executemany` 与 `RETURNING`，适用于所有支持的 Oracle 版本 9 及以上，使用多行 OUT 参数。这与 “executemanyvalues” 不是同一实现，但具有相同的使用模式和等效的性能优势。

自版本 2.0.10 更改：

### 禁用该功能

要为给定后端禁用 “insertmanyvalues” 功能，可以将 `create_engine.use_insertmanyvalues` 参数设置为 `False` 以 `create_engine()` 的方式传递：

```py
engine = create_engine(
    "mariadb+mariadbconnector://scott:tiger@host/db", use_insertmanyvalues=False
)
```

该功能也可以通过将 `Table.implicit_returning` 参数设置为 `False`，显式禁用特定的 `Table` 对象的使用隐式 RETURNING：

```py
t = Table(
    "t",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("x", Integer),
    implicit_returning=False,
)
```

有人可能想要为特定表禁用 RETURNING 的原因是为了解决特定后端的限制。

### 批处理模式操作

该功能有两种操作模式，可在每个方言、每个 `Table` 基础上进行透明选择。一种是**批处理模式**，它通过重写形如以下 INSERT 语句来减少数据库往返次数：

```py
INSERT  INTO  a  (data,  x,  y)  VALUES  (%(data)s,  %(x)s,  %(y)s)  RETURNING  a.id
```

转换为“批处理”形式，例如：

```py
INSERT  INTO  a  (data,  x,  y)  VALUES
  (%(data_0)s,  %(x_0)s,  %(y_0)s),
  (%(data_1)s,  %(x_1)s,  %(y_1)s),
  (%(data_2)s,  %(x_2)s,  %(y_2)s),
  ...
  (%(data_78)s,  %(x_78)s,  %(y_78)s)
RETURNING  a.id
```

在上述情况下，语句是针对输入数据的子集（一个“批次”）组织的，其大小由数据库后端以及每个批次中的参数数量确定，以对应于已知的语句大小 / 参数数量的限制。然后，该功能为每个输入数据批次执行一次 INSERT 语句，直到所有记录都被消耗完毕，并将每个批次的 RETURNING 结果连接到一个单独的大行集中，该行集可以从单个 `Result` 对象中访问。

这种 “批处理” 形式允许使用更少的数据库往返进行许多行的 INSERT，并已被证明可以允许在大多数支持它的后端上实现显着的性能改进。

### 将 RETURNING 行与参数集关联起来

版本 2.0.10 中的新功能。

在前一节中说明的“批量”模式查询并不保证返回的记录顺序与输入数据的顺序相对应。当被 SQLAlchemy ORM 工作单元 过程使用时，以及用于将返回的服务器生成的值与输入数据相关联的应用程序时，`Insert.returning()` 和 `UpdateBase.return_defaults()` 方法包括一个选项 `Insert.returning.sort_by_parameter_order`，指示“insertmanyvalues”模式应保证这种对应关系。这与数据库后端实际执行的记录插入顺序无关，**在任何情况下都不**假设；只是返回的记录应该在接收时有序排列，以对应原始输入数据传递的顺序。

当存在 `Insert.returning.sort_by_parameter_order` 参数时，对于使用服务器生成的整数主键值（如 `IDENTITY`、PostgreSQL `SERIAL`、MariaDB `AUTO_INCREMENT` 或 SQLite 的 `ROWID` 方案）的表，"批量"模式可能选择使用更复杂的 INSERT..RETURNING 形式，结合基于返回值的行后执行排序，或者如果这样的形式不可用，则“insertmanyvalues”功能可能会优雅地降级为运行每个参数集合的单独 INSERT 语句的“非批量”模式。

例如，在 SQL Server 中，当使用自增的 `IDENTITY` 列作为主键时，使用以下 SQL 表单：

```py
INSERT  INTO  a  (data,  x,  y)
OUTPUT  inserted.id,  inserted.id  AS  id__1
SELECT  p0,  p1,  p2  FROM  (VALUES
  (?,  ?,  ?,  0),  (?,  ?,  ?,  1),  (?,  ?,  ?,  2),
  ...
  (?,  ?,  ?,  77)
)  AS  imp_sen(p0,  p1,  p2,  sen_counter)  ORDER  BY  sen_counter
```

在 PostgreSQL 中也使用类似的形式，当主键列使用 SERIAL 或 IDENTITY 时。上述形式**并不**保证插入行的顺序。但是，它确保了 IDENTITY 或 SERIAL 值将按照每个参数集合的顺序创建[[2]](#id5)。然后，“insertmanyvalues”功能通过递增整数标识对上述 INSERT 语句返回的行进行排序。

对于 SQLite 数据库，没有适当的 INSERT 形式可以将新的 ROWID 值的生成与传递的参数集合的顺序相关联。因此，当使用服务器生成的主键值时，SQLite 后端将在请求有序返回时降级为“非批量”模式。对于 MariaDB，insertmanyvalues 使用的默认 INSERT 形式足够，因为在使用 InnoDB 时，这个数据库后端会将 AUTO_INCREMENT 的顺序与输入数据的顺序对齐[[3]](#id6)。

对于客户端生成的主键，例如在使用 Python `uuid.uuid4()` 函数为 `Uuid` 列生成新值时，"insertmanyvalues" 特性会将该列透明地包含在 RETURNING 记录中，并将其值与给定的输入记录相对应，从而保持输入记录和结果行之间的对应关系。由此可见，当使用客户端生成的主键值时，所有后端都允许在批处理时进行参数相关的 RETURNING 排序。

“insertmanyvalues” “batch” 模式确定用作输入参数和 RETURNING 行之间对应点的列或列的主题被称为 insert sentinel，这是用于跟踪此类值的特定列或列。通常会自动选择“insert sentinel”，但也可以对极端特殊情况进行用户配置；章节 配置 Sentinel 列 对此进行了描述。

对于不提供适当的 INSERT 表单以可以确定地与输入值对齐生成服务器值的后端，或对于 `Table` 配置具有其他类型的服务器生成的主键值的情况，“insertmanyvalues” 模式将在请求保证 RETURNING 排序时使用 **非批处理** 模式。

另请参阅

> +   Microsoft SQL Server 的基本原理
> +   
>     “使用 SELECT 和 ORDER BY 填充行的 INSERT 查询保证了如何计算标识值，但不能保证插入行的顺序。” [`learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql?view=sql-server-ver16#limitations-and-restrictions`](https://learn.microsoft.com/en-us/sql/t-sql/statements/insert-transact-sql?view=sql-server-ver16#limitations-and-restrictions)
>     
> +   PostgreSQL 批处理 INSERT 讨论
> +   
>     2018 年的原始描述 [`www.postgresql.org/message-id/29386.1528813619@sss.pgh.pa.us`](https://www.postgresql.org/message-id/29386.1528813619@sss.pgh.pa.us)
>     
>     2023 年的后续讨论 - [`www.postgresql.org/message-id/be108555-da2a-4abc-a46b-acbe8b55bd25%40app.fastmail.com`](https://www.postgresql.org/message-id/be108555-da2a-4abc-a46b-acbe8b55bd25%40app.fastmail.com)

+   MariaDB AUTO_INCREMENT 行为（使用与 MySQL 相同的 InnoDB 引擎）：

    [`dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html`](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)

    [`dba.stackexchange.com/a/72099`](https://dba.stackexchange.com/a/72099)

### 非批处理模式操作

对于没有客户端主键值的`Table`配置，并提供由服务器生成的主键值（或没有主键）的数据库无法以确定性或可排序的方式调用多个参数集相对于的情况，当“insertmanyvalues”功能被要求满足`Insert.returning.sort_by_parameter_order`对于`Insert`语句的要求时，可能会选择使用**非批处理模式**。

在这种模式下，保持了原始的 SQL 形式的 INSERT，并且“insertmanyvalues”功能将代替为每个参数集单独运行给定的语句，将返回的行组织成完整的结果集。与以前的 SQLAlchemy 版本不同，它在一个紧凑的循环中执行，最大限度地减少了 Python 的开销。在某些情况下，例如在 SQLite 上，“非批处理”模式的性能与“批处理”模式完全一样。

### 语句执行模型

对于“批处理”和“非批处理”两种模式，该功能将必须使用 DBAPI `cursor.execute()`方法调用**多个 INSERT 语句**，在**单个**对 Core 级别的`Connection.execute()`方法的调用范围内，每个语句包含多达固定数量的参数集。如下所述，此限制可配置为控制批处理大小。对`cursor.execute()`的单独调用将被单独记录，并且也单独传递给事件侦听器，例如`ConnectionEvents.before_cursor_execute()`（请参阅下面的日志记录和事件）。

#### 配置哨兵列

在典型情况下，为了从给定表的主键提供具有确定性行顺序的 INSERT..RETURNING 功能，将自动确定一个哨兵列，并在无法识别时优雅地降级到“逐行”模式。作为完全**可选**的功能，为了对具有服务器生成的主键的表提供完整的“insertmanyvalues”批量性能，其默认生成函数与“sentinel”用例不兼容，其他非主键列可以标记为“sentinel”列，假设它们满足一定要求。一个典型的例子是一个具有客户端默认值的非主键 `Uuid` 列，例如 Python 的 `uuid.uuid4()` 函数。还有一种构造方法可以创建带有客户端整数计数器的简单整数列，以满足“insertmanyvalues”用例。

可以通过将 `Column.insert_sentinel` 添加到符合条件的列来指示哨兵列。最基本的“符合条件”列是一个非空、唯一的列，具有客户端默认值，例如以下 UUID 列：

```py
import uuid

from sqlalchemy import Column
from sqlalchemy import FetchedValue
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid

my_table = Table(
    "some_table",
    metadata,
    # assume some arbitrary server-side function generates
    # primary key values, so cannot be tracked by a bulk insert
    Column("id", String(50), server_default=FetchedValue(), primary_key=True),
    Column("data", String(50)),
    Column(
        "uniqueid",
        Uuid(),
        default=uuid.uuid4,
        nullable=False,
        unique=True,
        insert_sentinel=True,
    ),
)
```

当使用 ORM Declarative 模型时，可以使用 `mapped_column` 结构来使用相同的形式：

```py
import uuid

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))
    uniqueid: Mapped[uuid.UUID] = mapped_column(
        default=uuid.uuid4, unique=True, insert_sentinel=True
    )
```

虽然默认生成器生成的值**必须**是唯一的，但上述“哨兵”列上的实际 UNIQUE 约束（由 `unique=True` 参数指示）本身是可选的，如果不需要可以省略。

还有一种特殊形式的“插入哨兵”，它是一个专用的可空整数列，它利用了一个特殊的默认整数计数器，仅在“insertmanyvalues”操作期间使用；作为附加行为，该列将在 SQL 语句和结果集中省略自身，并以基本透明的方式行事。然而，它确实需要在实际数据库表中存在。这种类型的 `Column` 可以使用函数 `insert_sentinel()` 构建：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid
from sqlalchemy import insert_sentinel

Table(
    "some_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String(50)),
    insert_sentinel("sentinel"),
)
```

当使用 ORM Declarative 时，提供了一个友好的版本 `insert_sentinel()`，称为 `orm_insert_sentinel()`，它具有在 Base 类或 mixin 上使用的能力；如果使用 `declared_attr()` 封装，该列将应用于所有表绑定的子类，包括联接继承层次结构内：

```py
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import orm_insert_sentinel

class Base(DeclarativeBase):
    @declared_attr
    def _sentinel(cls) -> Mapped[int]:
        return orm_insert_sentinel()

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))

class MySubClass(MyClass):
    __tablename__ = "sub_table"

    id: Mapped[str] = mapped_column(ForeignKey("my_table.id"), primary_key=True)

class MySingleInhClass(MyClass):
    pass
```

在上面的示例中，“my_table”和“sub_table”都将有一个额外的整数列名为“_sentinel”，可以被“insertmanyvalues”功能使用，以帮助优化 ORM 使用的批量插入。  #### 配置哨兵列

在典型情况下，“insertmanyvalues”功能为了提供具有确定性行顺序的 INSERT..RETURNING 将自动从给定表的主键确定一个哨兵列，如果无法识别，则优雅地降级为“逐行”模式。作为完全**可选**的功能，为了使具有服务器生成的主键的表获得完整的“insertmanyvalues”批量性能，其默认生成函数与“哨兵”用例不兼容，其他非主键列可以被标记为“哨兵”列，假设它们满足某些要求。一个典型的例子是一个非主键`Uuid`列，具有客户端默认值，例如 Python 的`uuid.uuid4()`函数。还有一种构造方法，用于创建简单的整数列，具有面向“insertmanyvalues”用例的客户端整数计数器。

可以通过将`Column.insert_sentinel`添加到合格的列来指示哨兵列。最基本的“合格”列是一个非空、唯一的列，具有客户端默认值，例如 UUID 列如下所示：

```py
import uuid

from sqlalchemy import Column
from sqlalchemy import FetchedValue
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid

my_table = Table(
    "some_table",
    metadata,
    # assume some arbitrary server-side function generates
    # primary key values, so cannot be tracked by a bulk insert
    Column("id", String(50), server_default=FetchedValue(), primary_key=True),
    Column("data", String(50)),
    Column(
        "uniqueid",
        Uuid(),
        default=uuid.uuid4,
        nullable=False,
        unique=True,
        insert_sentinel=True,
    ),
)
```

在使用 ORM 声明性模型时，可以使用`mapped_column`构造相同的形式：

```py
import uuid

from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column

class Base(DeclarativeBase):
    pass

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))
    uniqueid: Mapped[uuid.UUID] = mapped_column(
        default=uuid.uuid4, unique=True, insert_sentinel=True
    )
```

尽管默认生成器生成的值**必须**是唯一的，但上述“哨兵”列上的实际 UNIQUE 约束，由`unique=True`参数指示，本身是可选的，如果不需要可以省略。

还有一种特殊形式的“插入哨兵”，它是一个专用的可空整数列，利用一个特殊的默认整数计数器，仅在“insertmanyvalues”操作期间使用；作为额外的行为，该列将在 SQL 语句和结果集中省略自身，并以基本透明的方式行为。但是，它确实需要在实际数据库表中物理存在。可以使用函数`insert_sentinel()`构造这种`Column`的样式：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import String
from sqlalchemy import Table
from sqlalchemy import Uuid
from sqlalchemy import insert_sentinel

Table(
    "some_table",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("data", String(50)),
    insert_sentinel("sentinel"),
)
```

当使用 ORM 声明时，提供了一种友好的与声明性兼容的`insert_sentinel()`版本，称为`orm_insert_sentinel()`，它具有在基类或混合类上使用的能力；如果使用`declared_attr()`打包，该列将应用于所有绑定到表的子类，包括在连接继承层次结构中的子类：

```py
from sqlalchemy.orm import declared_attr
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import orm_insert_sentinel

class Base(DeclarativeBase):
    @declared_attr
    def _sentinel(cls) -> Mapped[int]:
        return orm_insert_sentinel()

class MyClass(Base):
    __tablename__ = "my_table"

    id: Mapped[str] = mapped_column(primary_key=True, server_default=FetchedValue())
    data: Mapped[str] = mapped_column(String(50))

class MySubClass(MyClass):
    __tablename__ = "sub_table"

    id: Mapped[str] = mapped_column(ForeignKey("my_table.id"), primary_key=True)

class MySingleInhClass(MyClass):
    pass
```

在上述示例中，“my_table”和“sub_table”都将有一个名为“_sentinel”的额外整数列，该列可供“insertmanyvalues”功能使用，以帮助优化 ORM 使用的批量插入。

### 控制批量大小

“insertmanyvalues”的一个关键特点是，INSERT 语句的大小限制为固定的最大“values”子句数以及方言特定的固定总绑定参数数，这些参数可以同时表示在一个 INSERT 语句中。当给定的参数字典数量超过固定限制，或者当要在单个 INSERT 语句中呈现的绑定参数的总数超过固定限制（这两个固定限制是分开的）时，将在单个`Connection.execute()`调用的范围内调用多个 INSERT 语句，其中每个 INSERT 语句都容纳一部分参数字典，称为“批量”。然后，每个“批量”中表示的参数字典数量称为“批量大小”。例如，批量大小为 500 意味着每个发出的 INSERT 语句最多会插入 500 行。

能够调整批量大小可能非常重要，因为较大的批量大小对于值集本身相对较小的 INSERT 可能更有效率，而较小的批量大小可能更适用于使用非常大的值集的 INSERT，其中渲染的 SQL 大小以及一次传递的总数据大小可能受益于根据后端行为和内存约束而限制为某个大小。因此，批量大小可以在每个`Engine`以及每个语句的基础上进行配置。另一方面，参数限制是根据正在使用的数据库的已知特性固定的。

大多数后端的批处理大小默认为 1000，还有一个每方言的“最大参数数”限制因素，可能会在每个语句的基础上进一步减小批处理大小。最大参数数因方言和服务器版本而异；最大尺寸为 32700（选择与 PostgreSQL 的限制 32767 和 SQLite 的现代限制 32766 相距较远，同时为语句中的其他参数以及 DBAPI 的怪癖留出空间）。旧版本的 SQLite（3.32.0 之前）将此值设置为 999。MariaDB 没有确定的限制，但是 32700 仍然作为 SQL 消息大小的限制因素。

“批处理大小”的值可以通过`Engine`的`create_engine.insertmanyvalues_page_size`参数影响整个引擎。例如，要影响每个语句中包含最多 100 个参数集的 INSERT 语句：

```py
e = create_engine("sqlite://", insertmanyvalues_page_size=100)
```

批处理大小也可以根据每个语句使用`Connection.execution_options.insertmanyvalues_page_size`执行选项进行设置，例如每次执行：

```py
with e.begin() as conn:
    result = conn.execute(
        table.insert().returning(table.c.id),
        parameterlist,
        execution_options={"insertmanyvalues_page_size": 100},
    )
```

或者在语句本身上进行配置：

```py
stmt = (
    table.insert()
    .returning(table.c.id)
    .execution_options(insertmanyvalues_page_size=100)
)
with e.begin() as conn:
    result = conn.execute(stmt, parameterlist)
```

### 日志和事件

“insertmanyvalues”功能与 SQLAlchemy 的语句日志记录以及游标事件（如`ConnectionEvents.before_cursor_execute()`）完全集成。当参数列表被分成单独的批次时，**每个 INSERT 语句都会单独记录并传递给事件处理程序**。这与 SQLAlchemy 1.x 系列的以前版本中仅基于 psycopg2 的功能的工作方式相比是一个重大变化，以前的版本中多个 INSERT 语句的生成被隐藏在日志记录和事件之外。日志显示将截断用于可读性的长参数列表，并且还将指示每个语句的特定批次。下面的示例说明了此日志的摘录：

```py
INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[generated in 0.00177s (insertmanyvalues) 1/10 (unordered)] ('d0', 0, 0, 'd1',  ...
INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[insertmanyvalues 2/10 (unordered)] ('d100', 100, 1000, 'd101', ...

...

INSERT INTO a (data, x, y) VALUES (?, ?, ?), ... 795 characters truncated ...  (?, ?, ?), (?, ?, ?) RETURNING id
[insertmanyvalues 10/10 (unordered)] ('d900', 900, 9000, 'd901', ...
```

当非批处理模式发生时，日志将指示此情况以及 insertmanyvalues 消息：

```py
...

INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 67/78 (ordered; batch not supported)] ('d66', 66, 66)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 68/78 (ordered; batch not supported)] ('d67', 67, 67)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 69/78 (ordered; batch not supported)] ('d68', 68, 68)
INSERT INTO a (data, x, y) VALUES (?, ?, ?) RETURNING id
[insertmanyvalues 70/78 (ordered; batch not supported)] ('d69', 69, 69)

...
```

另请参阅

配置日志

### Upsert 支持

PostgreSQL、SQLite 和 MariaDB 方言提供了特定于后端的“upsert”构造`insert()`、`insert()` 和 `insert()`，它们是 `Insert` 构造的增强版本，具有额外的方法，如 `on_conflict_do_update()` 或 ``on_duplicate_key()``。当它们与 RETURNING 一起使用时，这些构造也支持“insertmanyvalues”行为，允许使用 RETURNING 进行有效的 upsert 操作。

## 引擎处理

`Engine` 指的是一个连接池，这意味着在正常情况下，当 `Engine` 对象仍然驻留在内存中时，存在着打开的数据库连接。当一个 `Engine` 被垃圾回收时，它的连接池不再被该 `Engine` 引用，假设它的连接都没有被检出，连接池及其连接也将被垃圾回收，这将关闭实际的数据库连接。但在其他情况下，`Engine` 将保持打开的数据库连接，假设它使用的是通常的默认池实现`QueuePool`。

`Engine` 通常被设计成是一个固定的组件，在应用程序的整个生命周期中建立并维护。它**不**打算每个连接都创建和销毁；相反，它是一个注册表，维护着一个连接池以及关于正在使用的数据库和 DBAPI 的配置信息，以及某种程度的每个数据库资源的内部缓存。

然而，在许多情况下，希望所有由`Engine`引用的连接资源都被完全关闭。通常不建议依赖 Python 垃圾回收来处理这些情况；相反，可以使用`Engine.dispose()`方法来显式处置`Engine`。这会处理引擎的底层连接池，并将其替换为一个空的连接池。只要在此时丢弃`Engine`并且不再使用它，它所引用的所有**已签入**连接也将被完全关闭。

调用`Engine.dispose()`的有效用例包括：

+   当程序希望释放连接池中持有的任何剩余已签入连接，并且不希望将来连接到该数据库以进行任何未来操作时。

+   当程序使用多进程或`fork()`，并且将`Engine`对象复制到子进程时，应调用`Engine.dispose()`，以便引擎在该 fork 中创建全新的数据库连接。数据库连接通常不会跨越进程边界。在这种情况下，可以使用参数`Engine.dispose.close`设置为 False。有关此用例的更多背景信息，请参阅使用连接池与多进程或 os.fork()部分。

+   在测试套件或多租户方案中，可能会创建和处置许多临时的、短暂的`Engine`对象。

当引擎被处置或垃圾回收时，**签出**的连接不会被丢弃，因为这些连接在应用程序的其他地方仍然被强引用。但是，在调用`Engine.dispose()`后，这些连接将不再与该`Engine`关联；当它们关闭时，它们将被返回到它们现在孤立的连接池中，最终会被垃圾回收，一旦所有引用它的连接也不再在任何地方被引用。由于这个过程不容易控制，强烈建议只在所有签出的连接都被签入或以其他方式与其池解除关联后才调用`Engine.dispose()`。

对于那些受到 `Engine` 对象使用连接池的负面影响的应用程序，另一种选择是完全禁用连接池。这通常只会对使用新连接产生一些性能影响，并且意味着当连接被检入时，它完全关闭并且不会保留在内存中。参见 切换池实现 以获取有关如何禁用连接池的指南。

另请参阅

连接池

在多进程或 os.fork() 中使用连接池

## 使用驱动程序 SQL 和原始 DBAPI 连接

使用 `Connection.execute()` 的介绍利用了 `text()` 构造来说明如何调用文本 SQL 语句。在使用 SQLAlchemy 时，文本 SQL 实际上更多地是个例外，而不是规范，因为核心表达语言和 ORM 都将 SQL 的文本表示抽象化了。然而，`text()` 构造本身也提供了一些文本 SQL 的抽象，它规范了绑定参数的传递方式，并且支持参数和结果集行的数据类型行为。

### 直接调用驱动程序的 SQL 字符串

对于想要直接调用传递给底层驱动程序（称为 DBAPI）的文本 SQL 而没有任何 `text()` 构造干预的用例，可以使用 `Connection.exec_driver_sql()` 方法：

```py
with engine.connect() as conn:
    conn.exec_driver_sql("SET param='bar'")
```

版本 1.4 中新增：添加了 `Connection.exec_driver_sql()` 方法。

### 直接使用 DBAPI 游标

有些情况下 SQLAlchemy 没有提供一种通用的方式来访问一些 DBAPI 函数，比如调用存储过程以及处理多个结果集。在这些情况下，直接处理原始 DBAPI 连接同样方便。

访问原始 DBAPI 连接的最常见方式是直接从已经存在的 `Connection` 对象获取它。通过 `Connection.connection` 属性来访问：

```py
connection = engine.connect()
dbapi_conn = connection.connection
```

这里的 DBAPI 连接实际上是“代理”的，就像以前的连接池一样，但这是一个在大多数情况下可以忽略的实现细节。由于此 DBAPI 连接仍包含在拥有者`Connection`对象的范围内，因此最好使用`Connection`对象进行大多数功能的操作，例如事务控制以及调用`Connection.close()`方法；如果这些操作直接在 DBAPI 连接上执行，拥有者`Connection`将不会意识到这些状态的变化。

为了克服由由拥有者`Connection`维护的 DBAPI 连接所施加的限制，还可以在不需要首先获得`Connection`的情况下获得一个 DBAPI 连接，方法是使用`Engine.raw_connection()`方法的`Engine`:

```py
dbapi_conn = engine.raw_connection()
```

这个 DBAPI 连接再次是一个“代理”形式，就像以前一样。这种代理的目的现在显而易见，因为当我们调用该连接的`.close()`方法时，DBAPI 连接通常实际上并没有关闭，而是释放回到引擎的连接池中：

```py
dbapi_conn.close()
```

虽然 SQLAlchemy 可能在将来添加更多 DBAPI 使用案例的内置模式，但由于这些情况往往很少需要，并且它们也高度依赖于所使用的 DBAPI 类型，所以无论如何，直接的 DBAPI 调用模式始终存在于那些需要的情况下。

另请参见

在使用引擎时如何获取原始 DBAPI 连接？ - 包括有关如何访问 DBAPI 连接以及在使用 asyncio 驱动程序时的“驱动程序”连接的其他详细信息。

一些用于 DBAPI 连接的示例。### 调用存储过程和用户定义的函数

SQLAlchemy 支持以几种方式调用存储过程和用户定义的函数。请注意，所有 DBAPI 都有不同的做法，因此您必须查阅底层 DBAPI 的文档，以了解与您特定用法相关的具体内容。以下示例是假设性的，可能不适用于您的底层 DBAPI。

对于具有特殊语法或参数问题的存储过程或函数，可以使用 DBAPI 级别的[callproc](https://legacy.python.org/dev/peps/pep-0249/#callproc)，具体取决于您的 DBAPI。这种模式的示例如下：

```py
connection = engine.raw_connection()
try:
    cursor_obj = connection.cursor()
    cursor_obj.callproc("my_procedure", ["x", "y", "z"])
    results = list(cursor_obj.fetchall())
    cursor_obj.close()
    connection.commit()
finally:
    connection.close()
```

注意

并非所有的 DBAPI 都使用 callproc，总体使用细节会有所不同。上面的示例仅是展示如何使用特定的 DBAPI 函数的示例。

您的 DBAPI 可能没有`callproc`的要求，*或者*可能需要以另一种模式调用存储过程或用户定义函数，例如正常的 SQLAlchemy 连接用法。一个这种用法模式的例子是，在*本文档编写时*，使用 psycopg2 DBAPI 在 PostgreSQL 数据库中执行存储过程，应该使用正常的连接用法：

```py
connection.execute("CALL my_procedure();")
```

上述示例是假设性的。在这些情况下，底层数据库不能保证支持“CALL”或“SELECT”，关键字可能会根据函数是存储过程还是用户定义函数而变化。在这些情况下，您应该查阅底层 DBAPI 和数据库文档，以确定正确的语法和模式。

### 多结果集

通过 DBAPI 游标使用[nextset](https://legacy.python.org/dev/peps/pep-0249/#nextset)方法支持多结果集：

```py
connection = engine.raw_connection()
try:
    cursor_obj = connection.cursor()
    cursor_obj.execute("select * from table1; select * from table2")
    results_one = cursor_obj.fetchall()
    cursor_obj.nextset()
    results_two = cursor_obj.fetchall()
    cursor_obj.close()
finally:
    connection.close()
```

### 直接向驱动程序调用 SQL 字符串

对于想要直接传递给底层驱动程序的文本 SQL（称为 DBAPI）而不需要任何`text()`构造干预的用例，可以使用`Connection.exec_driver_sql()`方法：

```py
with engine.connect() as conn:
    conn.exec_driver_sql("SET param='bar'")
```

新版本 1.4 中新增了`Connection.exec_driver_sql()`方法。

### 直接使用 DBAPI 游标

有些情况下，SQLAlchemy 没有提供一种通用的方式来访问一些 DBAPI 函数，例如调用存储过程以及处理多个结果集。在这些情况下，直接处理原始的 DBAPI 连接同样方便。

访问原始 DBAPI 连接的最常见方式是直接从已经存在的`Connection`对象中获取。可以使用`Connection.connection`属性来获取：

```py
connection = engine.connect()
dbapi_conn = connection.connection
```

此处的 DBAPI 连接实际上是“代理”形式，就原始连接池而言，但这是一个实现细节，在大多数情况下可以忽略。由于此 DBAPI 连接仍然包含在一个拥有者 `Connection` 对象的范围内，最好使用 `Connection` 对象进行大多数功能的操作，例如事务控制以及调用 `Connection.close()` 方法；如果这些操作直接在 DBAPI 连接上执行，则拥有者 `Connection` 将不会意识到这些状态的变化。

为了克服由拥有 `Connection` 维护的 DBAPI 连接所施加的限制，还提供了不需要先获得 `Connection` 的 DBAPI 连接，可以使用 `Engine.raw_connection()` 方法来获得 `Engine` 的原始连接：

```py
dbapi_conn = engine.raw_connection()
```

此 DBAPI 连接再次是“代理”形式，与之前的情况一样。现在这种代理的目的显而易见，因为当我们调用这个连接的 `.close()` 方法时，DBAPI 连接通常并不实际关闭，而是被 释放 回到引擎的连接池中：

```py
dbapi_conn.close()
```

虽然 SQLAlchemy 可能会在将来为更多的 DBAPI 使用情况添加内置模式，但由于这些情况往往很少需要，并且它们也高度依赖于所使用的 DBAPI 的类型，因此在任何情况下，直接的 DBAPI 调用模式始终存在于那些需要的情况下。

另请参阅

在使用 Engine 时如何获取原始 DBAPI 连接？ - 包括有关如何访问 DBAPI 连接以及使用 asyncio 驱动程序时的“驱动程序”连接的其他详细信息。

一些 DBAPI 连接的示例如下。

### 调用存储过程和用户定义的函数

SQLAlchemy 支持多种方式调用存储过程和用户定义的函数。请注意，所有的 DBAPI 都有不同的做法，所以你必须查阅你所使用的底层 DBAPI 的文档，了解与你特定用法相关的具体内容。以下示例是假设性的，可能不适用于你所使用的底层 DBAPI。

对于具有特殊语法或参数关注点的存储过程或函数，可以使用 DBAPI 级别的 [callproc](https://legacy.python.org/dev/peps/pep-0249/#callproc)，与你的 DBAPI 搭配使用。这种模式的示例是：

```py
connection = engine.raw_connection()
try:
    cursor_obj = connection.cursor()
    cursor_obj.callproc("my_procedure", ["x", "y", "z"])
    results = list(cursor_obj.fetchall())
    cursor_obj.close()
    connection.commit()
finally:
    connection.close()
```

注

并非所有的 DBAPI 都使用`callproc`，整体使用细节会有所不同。上面的示例仅说明了如何使用特定的 DBAPI 函数。

您的 DBAPI 可能没有`callproc`的要求，*或*可能需要使用另一种模式调用存储过程或用户定义函数，例如正常的 SQLAlchemy 连接使用。一个使用模式的例子是，在撰写本文档时，在 PostgreSQL 数据库中使用 psycopg2 DBAPI 执行存储过程，应该使用正常的连接使用方式：

```py
connection.execute("CALL my_procedure();")
```

上述示例是假设性的。底层数据库不能保证在这些情况下支持“CALL”或“SELECT”，关键字可能会根据函数是存储过程还是用户定义函数而变化。在这些情况下，您应该查阅底层 DBAPI 和数据库文档，以确定正确的语法和模式。

### 多结果集

从原始 DBAPI 游标使用[nextset](https://legacy.python.org/dev/peps/pep-0249/#nextset)方法可获得多结果集支持：

```py
connection = engine.raw_connection()
try:
    cursor_obj = connection.cursor()
    cursor_obj.execute("select * from table1; select * from table2")
    results_one = cursor_obj.fetchall()
    cursor_obj.nextset()
    results_two = cursor_obj.fetchall()
    cursor_obj.close()
finally:
    connection.close()
```

## 注册新方言

`create_engine()`函数调用使用 setuptools entrypoints 定位给定的方言。这些入口点可以在 setup.py 脚本中为第三方方言建立。例如，要创建一个新的方言“foodialect://”，步骤如下：

1.  创建一个名为`foodialect`的包。

1.  包应该包含一个包含方言类的模块，通常是`sqlalchemy.engine.default.DefaultDialect`的子类。在这个例子中，假设它被称为`FooDialect`，并且可以通过`foodialect.dialect`访问其模块。

1.  入口点可以在`setup.cfg`中建立如下：

    ```py
    [options.entry_points]
    sqlalchemy.dialects  =
      foodialect  =  foodialect.dialect:FooDialect
    ```

如果方言在现有的 SQLAlchemy 支持的数据库之上提供对特定 DBAPI 的支持，则可以包括数据库限定的名称。例如，如果`FooDialect`实际上是一个 MySQL 方言，入口点可以这样建立：

```py
[options.entry_points]
sqlalchemy.dialects
  mysql.foodialect  =  foodialect.dialect:FooDialect
```

然后可以通过`create_engine("mysql+foodialect://")`访问上述入口点。

### 在进程中注册方言

SQLAlchemy 还允许在当前进程中注册方言，无需单独安装。使用`register()`函数如下：

```py
from sqlalchemy.dialects import registry

registry.register("mysql.foodialect", "myapp.dialect", "MyMySQLDialect")
```

上述代码将响应`create_engine("mysql+foodialect://")`并从`myapp.dialect`模块加载`MyMySQLDialect`类。

### 在进程中注册方言

SQLAlchemy 还允许在当前进程中注册方言，无需单独安装。使用`register()`函数如下：

```py
from sqlalchemy.dialects import registry

registry.register("mysql.foodialect", "myapp.dialect", "MyMySQLDialect")
```

上述代码将响应`create_engine("mysql+foodialect://")`并从`myapp.dialect`模块加载`MyMySQLDialect`类。

## 连接/引擎 API

| 对象名称 | 描述 |
| --- | --- |
| 连接 | 提供了一个封装的 DB-API 连接的高级功能。 |
| CreateEnginePlugin | 一组旨在根据 URL 中的入口点名称增强 `Engine` 对象构造的钩子。 |
| 引擎 | 将 `Pool` 和 `Dialect` 连接在一起，提供数据库连接和行为的来源。 |
| 异常上下文 | 封装了正在进行的错误条件的信息。 |
| 嵌套事务 | 表示一个“嵌套”的或 SAVEPOINT 事务。 |
| 根事务 | 表示 `Connection` 上的“根”事务。 |
| 事务 | 表示正在进行的数据库事务。 |
| 两阶段事务 | 表示一个两阶段事务。 |

```py
class sqlalchemy.engine.Connection
```

提供了一个封装的 DB-API 连接的高级功能。

`Connection` 对象通过调用 `Engine.connect()` 方法从 `Engine` 对象获取，并提供执行 SQL 语句以及事务控制的服务。

Connection 对象 **不是** 线程安全的。虽然一个 Connection 可以通过适当同步的访问在线程之间共享，但底层的 DBAPI 连接可能不支持在线程之间的共享访问。请查阅 DBAPI 文档以了解详情。

**成员**

__init__(), begin(), begin_nested(), begin_twophase(), close(), closed, commit(), connection, default_isolation_level, detach(), exec_driver_sql(), execute(), execution_options(), get_execution_options(), get_isolation_level(), get_nested_transaction(), get_transaction(), in_nested_transaction(), in_transaction(), info, invalidate(), invalidated, rollback(), scalar(), scalars(), schema_for_object()

连接对象表示从连接池中检出的单个 DBAPI 连接。在此状态下，连接池对连接没有任何影响，包括其到期或超时状态。为了让连接池正确管理连接，应该在连接不使用时将连接返回给连接池（即`connection.close()`）。

**类签名**

类 `sqlalchemy.engine.Connection` (`sqlalchemy.engine.interfaces.ConnectionEventsTarget`, `sqlalchemy.inspection.Inspectable`)

```py
method __init__(engine: Engine, connection: PoolProxiedConnection | None = None, _has_events: bool | None = None, _allow_revalidate: bool = True, _allow_autobegin: bool = True)
```

构建一个新的连接。

```py
method begin() → RootTransaction
```

在自动开始之前开始事务。

例如：

```py
with engine.connect() as conn:
    with conn.begin() as trans:
        conn.execute(table.insert(), {"username": "sandy"})
```

返回的对象是 `RootTransaction` 的实例。该对象表示事务的“范围”，当调用 `Transaction.rollback()` 或 `Transaction.commit()` 方法时完成事务；该对象还可以作为上述示例中的上下文管理器使用。

`Connection.begin()` 方法开始一个事务，通常会在连接首次用于执行语句时开始。可能使用此方法的原因是在特定时间调用 `ConnectionEvents.begin()` 事件，或者以上下文管理块的形式组织代码，例如：

```py
with engine.connect() as conn:
    with conn.begin():
        conn.execute(...)
        conn.execute(...)

    with conn.begin():
        conn.execute(...)
        conn.execute(...)
```

上述代码在行为上与不使用 `Connection.begin()` 的以下代码基本上没有任何不同；下面的样式称为“随时提交”样式。

```py
with engine.connect() as conn:
    conn.execute(...)
    conn.execute(...)
    conn.commit()

    conn.execute(...)
    conn.execute(...)
    conn.commit()
```

从数据库角度来看，`Connection.begin()` 方法不会发出任何 SQL 或以任何方式更改底层 DBAPI 连接的状态；Python DBAPI 没有任何显式事务开始的概念。

另请参阅

处理事务和 DBAPI - 在 SQLAlchemy 统一教程 中。 

`Connection.begin_nested()` - 使用 SAVEPOINT。

`Connection.begin_twophase()` - 使用两阶段/XID 事务。

`Engine.begin()` - 可从 `Engine` 获取的上下文管理器。

```py
method begin_nested() → NestedTransaction
```

开始一个嵌套事务（即 SAVEPOINT）并返回一个控制 SAVEPOINT 范围的事务句柄。

例如：

```py
with engine.begin() as connection:
    with connection.begin_nested():
        connection.execute(table.insert(), {"username": "sandy"})
```

返回的对象是`NestedTransaction`的一个实例，其中包括事务方法`NestedTransaction.commit()`和`NestedTransaction.rollback()`；对于嵌套事务，这些方法对应于操作“RELEASE SAVEPOINT <name>”和“ROLLBACK TO SAVEPOINT <name>”。保存点的名称对于`NestedTransaction`对象是本地的，并且会自动生成。与任何其他`Transaction`一样，`NestedTransaction`可以像上面所示一样用作上下文管理器，这将“释放”或“回滚”对应于块内的操作是否成功或引发异常。

嵌套事务要求底层数据库支持 SAVEPOINT，否则行为是未定义的。SAVEPOINT 通常用于在事务中运行可能失败的操作，同时继续外部事务。例如：

```py
from sqlalchemy import exc

with engine.begin() as connection:
    trans = connection.begin_nested()
    try:
        connection.execute(table.insert(), {"username": "sandy"})
        trans.commit()
    except exc.IntegrityError:  # catch for duplicate username
        trans.rollback()  # rollback to savepoint

    # outer transaction continues
    connection.execute( ... )
```

如果在未调用`Connection.begin()`或`Engine.begin()`之前调用`Connection.begin_nested()`，则`Connection`对象将首先“自动开始”外部事务。这个外部事务可以使用“按步骤提交”样式提交，例如：

```py
with engine.connect() as connection:  # begin() wasn't called

    with connection.begin_nested(): will auto-"begin()" first
        connection.execute( ... )
    # savepoint is released

    connection.execute( ... )

    # explicitly commit outer transaction
    connection.commit()

    # can continue working with connection here
```

从版本 2.0 开始更改：`Connection.begin_nested()` 现在将参与连接的“自动开始”行为，这是 2.0 版/ 1.4 版中的“未来”风格连接的新功能。

另请参阅

`Connection.begin()`

使用 SAVEPOINT - ORM 对 SAVEPOINT 的支持

```py
method begin_twophase(xid: Any | None = None) → TwoPhaseTransaction
```

开始一个两阶段或 XA 事务并返回事务句柄。

返回的对象是`TwoPhaseTransaction`的一个实例，除了由`Transaction`提供的方法之外，还提供了一个`TwoPhaseTransaction.prepare()`方法。

参数：

**xid** – 两阶段事务 ID。如果未提供，则会生成一个随机 ID。

另请参阅

`Connection.begin()`

`Connection.begin_twophase()`

```py
method close() → None
```

关闭此`Connection`。

这将释放底层数据库资源，即内部引用的 DBAPI 连接。DBAPI 连接通常会恢复到由产生此`Connection`的`Engine`引用的连接持有`Pool`。无论是否存在与此`Connection`相关的任何`Transaction`对象，DBAPI 连接上的任何事务状态也将通过 DBAPI 连接的`rollback()`方法无条件释放。

如果存在任何事务，则这也会调用`Connection.rollback()`。

在调用`Connection.close()`之后，`Connection`将永久处于关闭状态，并且不允许进一步操作。

```py
attribute closed
```

如果此连接已关闭，则返回 True。

```py
method commit() → None
```

提交当前正在进行的事务。

如果已经启动了当前事务，则此方法会提交当前事务。如果没有启动事务，则该方法不起作用，假定连接处于非失效状态。

每当首次执行语句或调用`Connection.begin()`方法时，`Connection`上都会自动开始事务。

注意

`Connection.commit()` 方法仅对与 `Connection` 对象关联的主数据库事务起作用。它不会对从 `Connection.begin_nested()` 方法调用的 SAVEPOINT 进行操作；要控制 SAVEPOINT，请在 `Connection.begin_nested()` 方法本身返回的 `NestedTransaction` 上调用 `NestedTransaction.commit()`。

```py
attribute connection
```

此连接管理的底层 DB-API 连接。

这是一个由 SQLAlchemy 连接池代理的连接，然后具有属性 `_ConnectionFairy.dbapi_connection`，该属性指向实际的驱动程序连接。

另请参见

使用 Driver SQL 和原始 DBAPI 连接

```py
attribute default_isolation_level
```

与使用中的 `Dialect` 关联的初始连接时间隔离级别。

此值独立于 `Connection.execution_options.isolation_level` 和 `Engine.execution_options.isolation_level` 执行选项，并且由 `Dialect` 在创建第一个连接时确定，通过在发出任何其他命令之前对数据库执行 SQL 查询以获取当前隔离级别。

调用此访问器不会触发任何新的 SQL 查询。

另请参见

`Connection.get_isolation_level()` - 查看当前实际隔离级别

`create_engine.isolation_level` - 设置每个 `Engine` 隔离级别

`Connection.execution_options.isolation_level` - 设置每个 `Connection` 隔离级别

```py
method detach() → None
```

从其连接池中分离基础 DB-API 连接。

例如：

```py
with engine.connect() as conn:
    conn.detach()
    conn.execute(text("SET search_path TO schema1, schema2"))

    # work with connection

# connection is fully closed (since we used "with:", can
# also call .close())
```

此 `Connection` 实例将保持可用状态。当关闭（或者像上面那样退出上下文管理器环境）时，DB-API 连接将被彻底关闭，而不会返回到其原始池中。

此方法可用于将连接上的修改状态（例如事务隔离级别或类似状态）与应用程序的其余部分隔离开来。

```py
method exec_driver_sql(statement: str, parameters: _DBAPIAnyExecuteParams | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

在不经过任何 SQL 编译步骤的情况下，在 DBAPI 光标上直接执行字符串 SQL 语句。

这可以用来直接将任何字符串传递给正在使用的 DBAPI 的 `cursor.execute()` 方法。

参数：

+   `statement` – 要执行的语句 str。绑定参数必须使用底层 DBAPI 的 paramstyle，例如 “qmark”、“pyformat”、“format” 等。

+   `parameters` – 代表要在执行中使用的绑定参数值。格式之一：命名参数的字典、位置参数的元组，或者包含用于多次执行支持的字典或元组的列表。

返回：

一个 `CursorResult`。

例如，多个字典：

```py
conn.exec_driver_sql(
    "INSERT INTO table (id, value) VALUES (%(id)s, %(value)s)",
    [{"id":1, "value":"v1"}, {"id":2, "value":"v2"}]
)
```

单个字典：

```py
conn.exec_driver_sql(
    "INSERT INTO table (id, value) VALUES (%(id)s, %(value)s)",
    dict(id=1, value="v1")
)
```

单个元组：

```py
conn.exec_driver_sql(
    "INSERT INTO table (id, value) VALUES (?, ?)",
    (1, 'v1')
)
```

注意

`Connection.exec_driver_sql()` 方法不参与 `ConnectionEvents.before_execute()` 和 `ConnectionEvents.after_execute()` 事件。要拦截对 `Connection.exec_driver_sql()` 的调用，请使用 `ConnectionEvents.before_cursor_execute()` 和 `ConnectionEvents.after_cursor_execute()`。

另请参阅

[**PEP 249**](https://peps.python.org/pep-0249/)

```py
method execute(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

执行一个 SQL 语句构造并返回一个 `CursorResult`。

参数：

+   `statement` –

    要执行的语句。这始终是同时在 `ClauseElement` 和 `Executable` 层次结构中的对象，包括：

    +   `Select`

    +   `Insert`、`Update`、`Delete`

    +   `TextClause` 和 `TextualSelect`

    +   `DDL` 和从`ExecutableDDLElement`继承的对象

+   `parameters` – 将绑定到语句中的参数。这可以是参数名称到值的字典，也可以是可变序列（例如列表）的字典。当传递一个字典列表时，底层语句执行将使用 DBAPI `cursor.executemany()`方法。当传递单个字典时，将使用 DBAPI `cursor.execute()`方法。

+   `execution_options` – 可选的执行选项字典，将与语句执行关联。该字典可以提供`Connection.execution_options()`接受的选项子集。

返回：

一个`Result`对象。

```py
method execution_options(**opt: Any) → Connection
```

设置在执行期间生效的连接的非 SQL 选项。

此方法会**就地**修改此`Connection`；返回值是调用该方法的相同`Connection`对象。请注意，这与其他对象的`execution_options`方法的行为相反，例如`Engine.execution_options()`和`Executable.execution_options()`。其理由是许多这样的执行选项必然会修改基本 DBAPI 连接的状态，因此没有可行的方法将此类选项的效果局限于“子”连接。

2.0 版本中的更改：`Connection.execution_options()`方法，与具有此方法的其他对象不同，会就地修改连接而不创建副本。

如其他地方所讨论的，`Connection.execution_options()` 方法接受任意参数，包括用户定义的名称。所有给定的参数都可以以多种方式被消耗，包括使用 `Connection.get_execution_options()` 方法。请参见 `Executable.execution_options()` 和 `Engine.execution_options()` 中的示例。

SQLAlchemy 本身当前识别的关键字包括`Executable.execution_options()`下列出的所有内容，以及特定于`Connection`的其他内容。

参数：

+   `compiled_cache` –

    可用于：`Connection`，`Engine`。

    一个字典，在`Connection`编译子句表达式为`Compiled`对象时将被缓存。该字典将取代可能在`Engine`本身上配置的语句缓存。如果设置为`None`，缓存将被禁用，即使引擎配置了缓存大小。

    请注意，ORM 在一些操作中（包括 flush 操作）使用了自己的“compiled”缓存。ORM 内部使用的缓存优先于此处指定的缓存字典。

+   `logging_token` –

    可用于：`Connection`，`Engine`，`Executable`。

    在连接日志中添加由括号括起来的指定字符串标记，即由连接记录的日志启用的日志，即通过`create_engine.echo`标志启用的日志，或通过`logging.getLogger("sqlalchemy.engine")`记录器启用的日志。这允许每个连接或每个子引擎都有一个可用的标记，这对于调试并发连接场景非常有用。

    自版本 1.4.0b2 新增。

    另见

    设置每个连接 / 子引擎标记 - 用法示例

    `create_engine.logging_name` - 将名称添加到 Python 日志记录器对象本身使用的名称。

+   `isolation_level` –

    可用于：`Connection`、`Engine`。

    设置此 `Connection` 对象的事务隔离级别的生命周期。有效值包括那些由 `create_engine()` 方法传递给 `create_engine.isolation_level` 参数接受的字符串值。这些级别在某种程度上特定于数据库；有关有效级别，请参阅各个方言的文档。

    通过在 DBAPI 连接上发出语句，`isolation_level` 选项将应用隔离级别，并且**必然会影响原始 Connection 对象的整体**。隔离级别将保持在给定设置，直到明确更改，或者当 DBAPI 连接本身被释放到连接池时，即调用 `Connection.close()` 方法时，在这时，事件处理程序将发出附加语句以在 DBAPI 连接上恢复隔离级别更改。

    注意

    `isolation_level` 执行选项只能在调用 `Connection.begin()` 方法之前建立，并且在发出任何否则会触发“autobegin”的 SQL 语句之前，或者在调用 `Connection.commit()` 或 `Connection.rollback()` 之后直接建立。数据库无法在进行中的事务上更改隔离级别。

    注意

    如果 `Connection` 被无效化，例如通过 `Connection.invalidate()` 方法，或者发生断开连接错误，则会隐式重置 `isolation_level` 执行选项。在无效化后生成的新连接将**不会**自动重新应用所选隔离级别。

    另请参阅

    设置事务隔离级别，包括 DBAPI 自动提交

    `Connection.get_isolation_level()` - 查看当前实际级别

+   `no_parameters` –

    可用于：`Connection`, `Executable`。

    当为`True`时，如果最终参数列表或字典完全为空，则会像`cursor.execute(statement)`一样在游标上调用语句，根本不传递参数集合。某些 DBAPI，如 psycopg2 和 mysql-python，仅当存在参数时才认为百分号是有意义的；此选项允许代码生成包含百分号（和可能的其他字符）的 SQL，该 SQL 在执行时与 DBAPI 无关，或者被管道传递到稍后由命令行工具调用的脚本中。

+   `stream_results` –

    可用于：`Connection`, `Executable`。

    如果可能的话，向方言指示结果应“流式传输”而不是预先缓冲。对于后端如 PostgreSQL、MySQL 和 MariaDB，这表示使用“服务器端游标”而不是客户端游标。其他后端，如 Oracle 的后端，可能默认已使用服务器端游标。

    通常将`Connection.execution_options.stream_results`的使用与设置一定数量的行以按批次提取相结合，以便在数据库行的有效迭代同时不一次将所有结果行加载到内存中；可以在执行返回新的`Result`之后，通过`Result.yield_per()`方法在`Result`对象上配置此项。如果不使用`Result.yield_per()`，那么`Connection.execution_options.stream_results`的操作模式将使用动态大小的缓冲区，该缓冲区一次缓冲一组行，在每个批次上根据固定的增长大小增长，直到通过使用`Connection.execution_options.max_row_buffer`参数配置的限制。

    当使用 ORM 从结果中获取 ORM 映射对象时，应始终与 `Connection.execution_options.stream_results` 一起使用 `Result.yield_per()`，以便 ORM 不会一次性将所有行都获取到新的 ORM 对象中。

    对于常规用途，应优先使用 `Connection.execution_options.yield_per` 执行选项，它同时设置了 `Connection.execution_options.stream_results` 和 `Result.yield_per()`。这个选项在核心层面由 `Connection` 和 ORM `Session` 支持；后者在使用 Yield Per 获取大型结果集中有描述。

    另请参阅

    使用服务器端游标（也称为流式结果） - 关于 `Connection.execution_options.stream_results` 的背景信息

    `Connection.execution_options.max_row_buffer`

    `Connection.execution_options.yield_per`

    使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中描述了 `yield_per` 的 ORM 版本

+   `max_row_buffer` –

    可用于：`Connection`，`Executable`。在支持服务器端游标的后端使用 `Connection.execution_options.stream_results` 执行选项时，设置要使用的最大缓冲区大小。如果未指定，默认值为 1000。

    另请参阅

    `Connection.execution_options.stream_results`

    使用服务器端游标（也称为流式结果）

+   `yield_per` –

    可用于：`Connection`，`Executable`。应用的整数值将设置 `Connection.execution_options.stream_results` 执行选项，并立即调用 `Result.yield_per()`。允许与 ORM 中使用此参数时存在的等效功能。

    版本 1.4.40 中的新功能。

    另请参见

    使用服务器端游标（也称为流式结果） - 使用 Core 使用服务器端游标的背景和示例。

    Fetching Large Result Sets with Yield Per - 在 ORM 查询指南 中描述了 `yield_per` 的 ORM 版本

+   `insertmanyvalues_page_size` –

    可用于：`Connection`，`Engine`。当语句使用“insertmanyvalues”模式时，即在使用 executemany 执行时通常与 RETURNING 一起使用时，将格式化为 INSERT 语句的行数，这是一种分页形式的批量插入，用于许多后端。默认为 1000。也可以通过 `create_engine.insertmanyvalues_page_size` 参数在每个引擎上进行修改。

    版本 2.0 中的新功能。

    另请参见

    “Insert Many Values” Behavior for INSERT statements

+   `schema_translate_map` –

    可用于：`Connection`，`Engine`，`Executable`。

    将模式名称映射到模式名称的字典，将应用于将 SQL 或 DDL 表达式元素编译为字符串时遇到的每个 `Table` 的 `Table.schema` 元素；根据原始名称在映射中的存在情况转换结果模式名称。

    另请参见

    模式名称的翻译

+   `preserve_rowcount` –

    Boolean；当为 True 时，`cursor.rowcount` 属性将在结果中被无条件地存储并可通过 `CursorResult.rowcount` 属性进行访问。通常情况下，此属性仅对 UPDATE 和 DELETE 语句保留。使用此选项，可以访问 DBAPI 的 rowcount 值以用于其他类型的语句，如 INSERT 和 SELECT，只要 DBAPI 支持这些语句。有关此属性行为的说明，请参见 `CursorResult.rowcount`。

    新版本 2.0.28 中新增。

另请参见

`Engine.execution_options()`

`Executable.execution_options()`

`Connection.get_execution_options()`

ORM 执行选项 - 所有 ORM 特定执行选项的文档

```py
method get_execution_options() → _ExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

新版本 1.3 中新增。

另请参见

`Connection.execution_options()`

```py
method get_isolation_level() → Literal['SERIALIZABLE', 'REPEATABLE READ', 'READ COMMITTED', 'READ UNCOMMITTED', 'AUTOCOMMIT']
```

返回在此连接范围内数据库中**实际**存在的当前隔离级别。

此属性将对数据库执行实时 SQL 操作，以获取当前隔离级别，因此返回的值是底层 DBAPI 连接上的实际级别，无论该状态如何设置。它将是四种实际隔离模式之一：`READ UNCOMMITTED`、`READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE`。它**不会**包括 `AUTOCOMMIT` 隔离级别设置。第三方方言可能还具有其他隔离级别设置。

注意

此方法**不会报告** `AUTOCOMMIT` 隔离级别，它是独立于**实际**隔离级别的单独的 dbapi 设置。当使用 `AUTOCOMMIT` 时，数据库连接仍然具有“传统”隔离模式，通常是四个值之一 `READ UNCOMMITTED`、`READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE`。

与 `Connection.default_isolation_level` 访问器相比，后者返回在初始连接时数据库中存在的隔离级别。

另请参见

`Connection.default_isolation_level` - 查看默认级别

`create_engine.isolation_level` - 设置每个 `Engine` 的隔离级别

`Connection.execution_options.isolation_level` - 设置每个 `Connection` 的隔离级别

```py
method get_nested_transaction() → NestedTransaction | None
```

返回当前正在进行的嵌套事务（如果有）。

1.4 版中的新功能。

```py
method get_transaction() → RootTransaction | None
```

返回当前正在进行的根事务（如果有）。

1.4 版中的新功能。

```py
method in_nested_transaction() → bool
```

如果事务正在进行中，则返回 True。

```py
method in_transaction() → bool
```

如果事务正在进行中，则返回 True。

```py
attribute info
```

与此 `Connection` 所引用的底层 DBAPI 连接相关联的信息字典，允许将用户定义的数据与连接关联起来。

此处的数据将与 DBAPI 连接一起传递，包括在将其返回到连接池并在后续 `Connection` 实例中再次使用之后。

```py
method invalidate(exception: BaseException | None = None) → None
```

使与此 `Connection` 关联的底层 DBAPI 连接无效。

尝试立即关闭底层的 DBAPI 连接；但是，如果此操作失败，则会记录错误但不会引发异常。然后，无论 close() 是否成功，连接都将被丢弃。

在下一次使用时（“使用”通常意味着使用 `Connection.execute()` 方法或类似方法），此 `Connection` 将尝试使用 `Pool` 的服务作为连接源（例如，“重新连接”）来获得新的 DBAPI 连接。

当调用`Connection.invalidate()`方法时，如果正在进行事务（例如已调用`Connection.begin()`方法），则在 DBAPI 级别，与此事务相关的所有状态都会丢失，因为 DBAPI 连接被关闭。直到调用`Transaction.rollback()`方法结束`Transaction`对象为止，`Connection`将不允许重新连接进行，直到此时，任何继续使用`Connection`的尝试都会引发`InvalidRequestError`。这是为了防止应用程序意外地继续进行已丢失的事务操作，尽管事务由于无效化而丢失。

与自动无效化一样，`Connection.invalidate()`方法在连接池级别会触发`PoolEvents.invalidate()`事件。

参数：

**exception** – 一个可选的`Exception`实例，表示无效化的原因，会传递给事件处理程序和日志记录函数。

另请参阅

更多关于无效化的信息

```py
attribute invalidated
```

如果此连接被无效化，则返回 True。

但是，这并不表示连接是否在池级别被无效化。

```py
method rollback() → None
```

回滚当前正在进行的事务。

如果已经开始了当前事务，则此方法会回滚当前事务。如果没有启动事务，则此方法不起作用。如果已启动事务且连接处于无效状态，则使用此方法清除事务。

当第一次执行语句或调用`Connection.begin()`方法时，`Connection`会自动开始事务。

注意

`Connection.rollback()` 方法仅作用于与 `Connection` 对象关联的主数据库事务。它不会作用于通过 `Connection.begin_nested()` 方法调用的 SAVEPOINT；要控制 SAVEPOINT，请在 `Connection.begin_nested()` 方法本身返回的 `NestedTransaction` 上调用 `NestedTransaction.rollback()`。

```py
method scalar(statement: Executable, parameters: _CoreSingleExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → Any
```

执行 SQL 语句构造并返回一个标量对象。

此方法是在调用 `Connection.execute()` 方法后调用 `Result.scalar()` 方法的简写。参数是等效的。

返回：

代表返回的第一行的第一列的标量 Python 值。

```py
method scalars(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → ScalarResult[Any]
```

执行并返回一个标量结果集��该结果集从每行的第一列产生标量值。

此方法等同于调用 `Connection.execute()` 以接收一个 `Result` 对象，然后调用 `Result.scalars()` 方法生成一个 `ScalarResult` 实例。

返回：

一个 `ScalarResult`

版本 1.4.24 中的新功能。

```py
method schema_for_object(obj: HasSchemaAttr) → str | None
```

返回给定模式项的模式名称，考虑当前模式转换映射。

```py
class sqlalchemy.engine.CreateEnginePlugin
```

一组旨在通过 URL 中的入口点名称增强基于 `Engine` 对象构造的钩子。

`CreateEnginePlugin` 的目的是允许第三方系统应用引擎、池和方言级别的事件侦听器，而无需修改目标应用程序；相反，插件名称可以添加到数据库 URL 中。 `CreateEnginePlugin` 的目标应用程序包括：

+   连接和 SQL 性能工具，例如使用事件跟踪检出次数和/或与语句一起花费的时间

+   诸如代理之类的连接插件

一个基本的 `CreateEnginePlugin`，将一个日志记录器附加到一个 `Engine` 对象可能看起来像这样：

```py
import logging

from sqlalchemy.engine import CreateEnginePlugin
from sqlalchemy import event

class LogCursorEventsPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        # consume the parameter "log_cursor_logging_name" from the
        # URL query
        logging_name = url.query.get("log_cursor_logging_name", "log_cursor")

        self.log = logging.getLogger(logging_name)

    def update_url(self, url):
        "update the URL to one that no longer includes our parameters"
        return url.difference_update_query(["log_cursor_logging_name"])

    def engine_created(self, engine):
        "attach an event listener after the new Engine is constructed"
        event.listen(engine, "before_cursor_execute", self._log_event)

    def _log_event(
        self,
        conn,
        cursor,
        statement,
        parameters,
        context,
        executemany):

        self.log.info("Plugin logged cursor event: %s", statement)
```

插件是使用与方言类似的方式使用入口点进行注册的：

```py
entry_points={
    'sqlalchemy.plugins': [
        'log_cursor_plugin = myapp.plugins:LogCursorEventsPlugin'
    ]
```

使用上述名称的插件将从数据库 URL 中调用，如下所示：

```py
from sqlalchemy import create_engine

engine = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?"
    "plugin=log_cursor_plugin&log_cursor_logging_name=mylogger"
)
```

`plugin` URL 参数支持多个实例，因此 URL 可以指定多个插件；它们按照 URL 中指定的顺序加载：

```py
engine = create_engine(
  "mysql+pymysql://scott:tiger@localhost/test?"
  "plugin=plugin_one&plugin=plugin_twp&plugin=plugin_three")
```

插件名称也可以直接通过 `create_engine()` 的 `create_engine.plugins` 参数传递：

```py
engine = create_engine(
  "mysql+pymysql://scott:tiger@localhost/test",
  plugins=["myplugin"])
```

新版本 1.2.3 中的新增内容：可以将插件名称作为列表指定给 `create_engine()`。

插件还可以从 `URL` 对象以及传递给 `create_engine()` 调用的参数字典中消耗插件特定参数。 “消耗” 这些参数包括在插件初始化时必须删除它们，以便不将参数传递给 `Dialect` 构造函数，否则它们将引发 `ArgumentError`，因为方言不知道它们。

截至 SQLAlchemy 版本 1.4，参数应该继续直接从 `kwargs` 字典中消耗，通过使用诸如 `dict.pop` 的方法删除值。应该通过实现 `CreateEnginePlugin.update_url()` 方法消耗来自 `URL` 对象的参数，返回一个新的不含插件特定参数的 `URL` 的副本：

```py
class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        self.my_argument_one = url.query['my_argument_one']
        self.my_argument_two = url.query['my_argument_two']
        self.my_argument_three = kwargs.pop('my_argument_three', None)

    def update_url(self, url):
        return url.difference_update_query(
            ["my_argument_one", "my_argument_two"]
        )
```

类似上述示例的参数将从 `create_engine()` 调用中消耗：

```py
from sqlalchemy import create_engine

engine = create_engine(
  "mysql+pymysql://scott:tiger@localhost/test?"
  "plugin=myplugin&my_argument_one=foo&my_argument_two=bar",
  my_argument_three='bat'
)
```

自版本 1.4 起更改：`URL` 对象现在是不可变的；需要改变 `URL` 的 `CreateEnginePlugin` 应该实现新添加的 `CreateEnginePlugin.update_url()` 方法，在构造插件之后调用。

对于迁移，以以下方式构建插件，检查`CreateEnginePlugin.update_url()`方法的存在以检测正在运行的版本：

```py
class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        if hasattr(CreateEnginePlugin, "update_url"):
            # detect the 1.4 API
            self.my_argument_one = url.query['my_argument_one']
            self.my_argument_two = url.query['my_argument_two']
        else:
            # detect the 1.3 and earlier API - mutate the
            # URL directly
            self.my_argument_one = url.query.pop('my_argument_one')
            self.my_argument_two = url.query.pop('my_argument_two')

        self.my_argument_three = kwargs.pop('my_argument_three', None)

    def update_url(self, url):
        # this method is only called in the 1.4 version
        return url.difference_update_query(
            ["my_argument_one", "my_argument_two"]
        )
```

另请参阅

URL 对象现在是不可变的 - `URL`更改的概述，还包括有关`CreateEnginePlugin`的注释。

**成员**

__init__(), engine_created(), handle_dialect_kwargs(), handle_pool_kwargs(), update_url()

当引擎创建过程完成并生成`Engine`对象时，再次通过`CreateEnginePlugin.engine_created()`钩子将其传递给插件。在此钩子中，可以对引擎进行其他更改，最常涉及事件设置（例如在核心事件中定义的事件）。

```py
method __init__(url: URL, kwargs: Dict[str, Any])
```

构建一个新的`CreateEnginePlugin`。

对于每次调用`create_engine()`都会单独实例化插件对象。一个单独的`Engine`将传递给与此 URL 对应的`CreateEnginePlugin.engine_created()`方法。

参数：

+   `url` –

    `URL`对象。插件可以检查`URL`的参数。插件使用的参数应通过从`CreateEnginePlugin.update_url()`方法返回的更新后的`URL`来移除。

    自版本 1.4 起更改：`URL`对象现在是不可变的，因此需要修改`URL`对象的`CreateEnginePlugin`应实现`CreateEnginePlugin.update_url()`方法。

+   `kwargs` – 传递给`create_engine()`的关键字参数。

```py
method engine_created(engine: Engine) → None
```

当完全构建时，接收`Engine`对象。

插件可能对引擎进行其他更改，例如注册引擎或连接池事件。

```py
method handle_dialect_kwargs(dialect_cls: Type[Dialect], dialect_args: Dict[str, Any]) → None
```

解析和修改方言参数。

```py
method handle_pool_kwargs(pool_cls: Type[Pool], pool_args: Dict[str, Any]) → None
```

解析和修改池参数。

```py
method update_url(url: URL) → URL
```

更新`URL`。

应返回一个新的`URL`。通常使用此方法来消耗从`URL`中移除的配置参数，因为方言不会识别这些参数。`URL.difference_update_query()`方法可用于移除这些参数。查看`CreateEnginePlugin`的文档字符串以获取示例。

版本 1.4 中的新功能。

```py
class sqlalchemy.engine.Engine
```

将`Pool`和`Dialect`连接在一起，以提供数据库连接和行为的来源。

使用`create_engine()`函数公开实例化一个`Engine`对象。

另请参阅

引擎配置

与引擎和连接一起工作

**成员**

begin(), clear_compiled_cache(), connect(), dispose(), driver, engine, execution_options(), get_execution_options(), name, raw_connection(), update_execution_options()

**类签名**

类`sqlalchemy.engine.Engine`（`sqlalchemy.engine.interfaces.ConnectionEventsTarget`，`sqlalchemy.log.Identified`，`sqlalchemy.inspection.Inspectable`）

```py
method begin() → Iterator[Connection]
```

返回一个通过建立`Transaction`的`Connection`提供的上下文管理器。

例如：

```py
with engine.begin() as conn:
    conn.execute(
        text("insert into table (x, y, z) values (1, 2, 3)")
    )
    conn.execute(text("my_special_procedure(5)"))
```

操作成功后，`Transaction`被提交。如果出现错误，则`Transaction`被回滚。

另请参阅

`Engine.connect()` - 从`Engine`获取一个`Connection`。

`Connection.begin()` - 为特定的`Connection`启动一个`Transaction`。

```py
method clear_compiled_cache() → None
```

清除与方言相关联的编译缓存。

这仅适用于通过`create_engine.query_cache_size`参数建立的内置缓存。它不会影响通过`Connection.execution_options.compiled_cache`参数传递的任何字典缓存。

从版本 1.4 开始。

```py
method connect() → Connection
```

返回一个新的`Connection`对象。

`Connection`充当 Python 上下文管理器，因此该方法的典型用法如下：

```py
with engine.connect() as connection:
    connection.execute(text("insert into table values ('foo')"))
    connection.commit()
```

在上述情况下，块完成后，连接被“关闭”，其底层的 DBAPI 资源被返回到连接池。这也会导致任何显式开始或通过自动开始开始的事务回滚，并且如果已开始并且仍在进行，则会发出`ConnectionEvents.rollback()`事件。

另请参阅

`Engine.begin()`

```py
method dispose(close: bool = True) → None
```

处理此`Engine`使用的连接池。

旧连接池被处理后立即创建一个新的连接池。之前的连接池可以通过主动关闭该池中所有当前签入的连接或者被动地失去对其的引用但不关闭任何连接来处理。后一种策略更适用于分叉的 Python 进程中的初始化器。

参数：

**close** –

如果保持默认值`True`，则会完全关闭所有**当前已签入**的数据库连接。仍在签出的连接将**不会**被关闭，但它们将不再与此`Engine`关联，因此当它们被单独关闭时，它们将最终被关联的`Pool`进行垃圾回收并完全关闭，如果尚未在签入时关闭。

如果设置为`False`，则先前的连接池将被取消引用，否则不会以任何方式触及。

版本 1.4.33 中的新功能：添加了`Engine.dispose.close`参数，允许在子进程中替换连接池而不干扰父进程使用的连接。

另请参见

Engine 处理

在多进程或 os.fork()中使用连接池

```py
attribute driver
```

此`Engine`使用的`Dialect`的驱动程序名称。

```py
attribute engine
```

返回此`Engine`。

用于接受相同变量中的`Connection` / `Engine`对象的传统方案。

```py
method execution_options(**opt: Any) → OptionEngine
```

返回一个新的`Engine`，将提供具有给定执行选项的`Connection`对象。

返回的`Engine`与原始`Engine`相关联，因为它共享相同的连接池和其他状态：

+   新`Engine`使用的`Pool`是同一个实例。`Engine.dispose()`方法将替换父引擎的连接池实例以及此引擎的连接池实例。

+   事件监听器是“级联”的 - 意味着新的`Engine`继承父级的事件，并且新事件可以单独与新的`Engine`关联。

+   日志配置和 logging_name 从父`Engine`复制。

`Engine.execution_options()`方法的目的是实现多个`Engine`对象引用同一连接池的方案，但通过影响每个引擎的某些执行级别行为的选项来区分它们。其中一个例子是将一个`Engine`分为单独的“读取器”和“写入器”实例，其中一个`Engine`配置了较低的隔离级别设置，甚至使用“自动提交”禁用了事务。此配置的示例位于为单个 Engine 维护多个隔离级别。

另一个示例是使用自定义选项`shard_id`，该选项由事件消耗以在数据库连接上更改当前模式：

```py
from sqlalchemy import event
from sqlalchemy.engine import Engine

primary_engine = create_engine("mysql+mysqldb://")
shard1 = primary_engine.execution_options(shard_id="shard1")
shard2 = primary_engine.execution_options(shard_id="shard2")

shards = {"default": "base", "shard_1": "db1", "shard_2": "db2"}

@event.listens_for(Engine, "before_cursor_execute")
def _switch_shard(conn, cursor, stmt,
        params, context, executemany):
    shard_id = conn.get_execution_options().get('shard_id', "default")
    current_shard = conn.info.get("current_shard", None)

    if current_shard != shard_id:
        cursor.execute("use %s" % shards[shard_id])
        conn.info["current_shard"] = shard_id
```

上述示例说明了两个`Engine`对象，它们各自作为预先设置了“shard_id”执行选项的`Connection`对象的工厂。然后，`ConnectionEvents.before_cursor_execute()`事件处理程序解释此执行选项，以在语句执行之前发出 MySQL `use`语句以切换数据库，同时使用`Connection.info`字典跟踪我们已经建立的数据库。

另请参阅

`Connection.execution_options()` - 更新`Connection`对象的执行选项。

`Engine.update_execution_options()` - 就地更新给定`Engine`的执行选项。

`Engine.get_execution_options()`

```py
method get_execution_options() → _ExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

另请参阅

`Engine.execution_options()`

```py
attribute name
```

正在使用的`Engine`的`Dialect`的字符串名称。

```py
method raw_connection() → PoolProxiedConnection
```

从连接池获取“原始”DBAPI 连接。

返回的对象是底层驱动程序使用的 DBAPI 连接对象的代理版本。该对象将具有与真实 DBAPI 连接相同的所有行为，只是其`close()`方法将导致连接被返回到池中，而不是真正关闭。

当`Connection`提供的 API 不需要时，此方法提供了直接的 DBAPI 连接访问。当已经存在一个`Connection`对象时，可以使用`Connection.connection`访问器来获取 DBAPI 连接。

另请参见

使用驱动程序 SQL 和原始 DBAPI 连接

```py
method update_execution_options(**opt: Any) → None
```

更新此`Engine`的默认 execution_options 字典。

在**opt 中给定的键/值将添加到将用于所有连接的默认执行选项中。此字典的初始内容可以通过`execution_options`参数发送到`create_engine()`。

另请参见

`Connection.execution_options()`

`Engine.execution_options()`

```py
class sqlalchemy.engine.ExceptionContext
```

封装有关正在进行的错误条件的信息。

**成员**

chained_exception, connection, cursor, dialect, engine, execution_context, invalidate_pool_on_disconnect, is_disconnect, is_pre_ping, original_exception, parameters, sqlalchemy_exception, statement

此对象仅存在以传递给`DialectEvents.handle_error()`事件，支持可以在不向后不兼容的情况下扩展的接口。

```py
attribute chained_exception: BaseException | None
```

如果有的话，这是前一个处理程序在异常链中返回的异常。

如果存在，此异常将最终由 SQLAlchemy 引发，除非后续处理程序替换它。

可能为 None。

```py
attribute connection: Connection | None
```

在异常处理期间使用的 `Connection`。

该成员存在，除非在首次连接失败时。

另请参阅

`ExceptionContext.engine`

```py
attribute cursor: DBAPICursor | None
```

DBAPI 游标对象。

可能为 None。

```py
attribute dialect: Dialect
```

正在使用的 `Dialect`。

该成员在事件钩子的所有调用中都存在。

新版本 2.0 中新增。

```py
attribute engine: Engine | None
```

在异常处理期间使用的 `Engine`。

该成员在所有情况下都存在，除非在连接池“预检测”过程中处理错误时除外。

```py
attribute execution_context: ExecutionContext | None
```

正在进行的执行操作对应的 `ExecutionContext`。

该成员对于语句执行操作是存在的，但对于事务开始/结束等操作则不存在。它也不会在在 `ExecutionContext` 构造之前引发异常的情况下存在。

请注意，`ExceptionContext.statement` 和 `ExceptionContext.parameters` 成员可能代表与 `ExecutionContext` 不同的值，可能在 `ConnectionEvents.before_cursor_execute()` 事件或类似事件修改了要发送的语句/参数的情况下。

可能为 None。

```py
attribute invalidate_pool_on_disconnect: bool
```

表示在存在“断开”条件时是否应使池中的所有连接失效。

在 `DialectEvents.handle_error()` 事件范围内将此标志设置为 False 将产生如下效果：在断开连接时不会使池中的所有连接失效；只有实际上产生错误的当前连接将被使无效。

此标志的目的是为了定制断开处理方案，其中其他连接在池中的失效是基于其他条件进行的，甚至是基于每个连接的基础。

```py
attribute is_disconnect: bool
```

表示发生的异常是否代表“断开”条件。

此标志在 `DialectEvents.handle_error()` 处理程序的范围内始终为 True 或 False。

SQLAlchemy 将根据此标志来确定是否随后应使连接无效。也就是说，通过将值分配给此标志，可以通过更改此标志来调用或防止导致连接和池失效的“断开”事件。

注

使用 `create_engine.pool_pre_ping` 参数启用的池“pre_ping”处理程序在决定“ping”返回 false 时 **不** 会参考此事件，而不是收到未处理的错误。对于这种用例，可以使用基于 `engine_connect()` 的传统配方。将来的 API 允许在所有函数中更全面地定制“断开”检测机制。

```py
attribute is_pre_ping: bool
```

指示此错误是否发生在设置 `create_engine.pool_pre_ping` 为 `True` 时执行的“pre-ping”步骤中。在此模式下，`ExceptionContext.engine` 属性将为 `None`。正在使用的方言可通过 `ExceptionContext.dialect` 属性访问。

从版本 2.0.5 开始新增。

```py
attribute original_exception: BaseException
```

被捕获的异常对象。

此成员始终存在。

```py
attribute parameters: _DBAPIAnyExecuteParams | None
```

直接发射到 DBAPI 的参数集合。

可能为 None。

```py
attribute sqlalchemy_exception: StatementError | None
```

`sqlalchemy.exc.StatementError` 包装了原始内容，并且如果事件未被绕过，则会被引发。

可能为 None，因为并非所有异常类型都由 SQLAlchemy 包装。对于子类化 dbapi 的 Error 类的 DBAPI 级异常，此字段将始终存在。

```py
attribute statement: str | None
```

直接发射到 DBAPI 的字符串 SQL 语句。

可能为 None。

```py
class sqlalchemy.engine.NestedTransaction
```

表示一个‘嵌套’，或 SAVEPOINT 事务。

`NestedTransaction` 对象是通过调用 `Connection.begin_nested()` 方法来创建的 `Connection`。

当使用 `NestedTransaction` 时，“begin” / “commit” / “rollback”的语义如下：

+   “begin”操作对应于“BEGIN SAVEPOINT”命令，其中保存点被赋予此对象状态的显式名称。

+   `NestedTransaction.commit()` 方法对应于“RELEASE SAVEPOINT”操作，使用与此 `NestedTransaction` 关联的保存点标识符。

+   `NestedTransaction.rollback()` 方法对应于“ROLLBACK TO SAVEPOINT”操作，使用与此 `NestedTransaction` 关联的保存点标识符。

模仿外部事务的语义以便代码可以以一种不可知的方式处理“保存点”事务和“外部”事务的原理。

另请参阅

使用 SAVEPOINT - SAVEPOINT API 的 ORM 版本。

**成员**

close(), commit(), rollback()

**类签名**

类 `sqlalchemy.engine.NestedTransaction`（`sqlalchemy.engine.Transaction`）

```py
method close() → None
```

*继承自* `Transaction.close()` *方法的* `Transaction`

关闭此 `Transaction`。

如果此事务是开始/提交嵌套中的基本事务，则事务将回滚。 否则，该方法返回。

这用于取消一个事务，而不影响封闭事务的范围。

```py
method commit() → None
```

*继承自* `Transaction.commit()` *方法的* `Transaction`

提交这个 `Transaction`。

根据使用的事务类型，其实现可能会有所不同：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于一个 COMMIT。

+   对于 `NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用用于两阶段事务的特定于 DBAPI 的方法。

```py
method rollback() → None
```

*继承自* `Transaction.rollback()` *方法的* `Transaction`

回滚此`Transaction`。

这个实现可能根据正在使用的事务类型而变化：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于 ROLLBACK。

+   对于`NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
class sqlalchemy.engine.RootTransaction
```

在`Connection`上表示“根”事务。

这对应于正在进行的`Connection`的当前“BEGIN/COMMIT/ROLLBACK”。通过调用`Connection.begin()`方法创建`RootTransaction`，并且在其活动期间与`Connection`关联。正在使用的当前`RootTransaction`可以通过`Connection`的`Connection.get_transaction`方法访问。

在 2.0 风格中，`Connection`还采用“自动开始”行为，每当处于非事务状态的连接用于在 DBAPI 连接上发出命令时，就会创建一个新的`RootTransaction`。在 2.0 风格中使用`RootTransaction`的范围可以使用`Connection.commit()`和`Connection.rollback()`方法进行控制。

**成员**

close(), commit(), rollback()

**类签名**

类 `sqlalchemy.engine.RootTransaction`（`sqlalchemy.engine.Transaction`）

```py
method close() → None
```

*继承自* `Transaction.close()` *方法的* `Transaction`

关闭此 `Transaction`。

如果此事务是嵌套事务中的基本事务，则事务将回滚。否则，该方法将返回。

用于取消事务而不影响封闭事务范围。

```py
method commit() → None
```

*继承自* `Transaction.commit()` *方法的* `Transaction`

提交此 `Transaction`。

其实现可能会根据正在使用的事务类型而变化：

+   对于简单的数据库事务（例如 `RootTransaction`），它对应于一个 COMMIT。

+   对于 `NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
method rollback() → None
```

*继承自* `Transaction.rollback()` *方法的* `Transaction`

回滚此 `Transaction`。

其实现可能会根据正在使用的事务类型而变化：

+   对于简单的数据库事务（例如 `RootTransaction`），它对应于一个 ROLLBACK。

+   对于 `NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
class sqlalchemy.engine.Transaction
```

表示正在进行的数据库事务。

通过调用 `Connection.begin()` 方法的 `Connection` 获得 `Transaction` 对象：

```py
from sqlalchemy import create_engine
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")
connection = engine.connect()
trans = connection.begin()
connection.execute(text("insert into x (a, b) values (1, 2)"))
trans.commit()
```

该对象提供了`rollback()`和`commit()`方法，以便控制事务边界。它还实现了上下文管理器接口，以便可以使用 Python 的`with`语句与`Connection.begin()`方法一起使用：

```py
with connection.begin():
    connection.execute(text("insert into x (a, b) values (1, 2)"))
```

Transaction 对象**不**是线程安全的。

**成员**

close(), commit(), rollback()

另请参见

`Connection.begin()`

`Connection.begin_twophase()`

`Connection.begin_nested()`

**类签名**

类 `sqlalchemy.engine.Transaction` (`sqlalchemy.engine.util.TransactionalContext`)

```py
method close() → None
```

关闭此 `Transaction`。

如果此事务是嵌套的 begin/commit 中的基本事务，则事务将回滚()。否则，该方法返回。

用于取消事务而不影响封闭事务范围。

```py
method commit() → None
```

提交此 `Transaction`。

其实现可能会根据正在使用的事务类型而变化：

+   对于简单的数据库事务（例如 `RootTransaction`），它对应于 COMMIT。

+   对于 `NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
method rollback() → None
```

回滚此`Transaction`。

其实现可能会根据正在使用的事务类型而变化：

+   对于简单的数据库事务（例如 `RootTransaction`），它对应于 ROLLBACK。

+   对于 `NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于 `TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
class sqlalchemy.engine.TwoPhaseTransaction
```

表示一个两阶段事务。

可以使用`Connection.begin_twophase()`方法获取一个新的`TwoPhaseTransaction`对象。

接口与`Transaction`相同，但添加了`prepare()`方法。

**成员**

close()，commit()，prepare()，rollback()

**类签名**

类`sqlalchemy.engine.TwoPhaseTransaction`（`sqlalchemy.engine.RootTransaction`）

```py
method close() → None
```

*继承自* `Transaction.close()` *方法的* `Transaction`

关闭这个`Transaction`。

如果这个事务是嵌套的开始/提交中的基本事务，则事务将回滚()。否则，该方法返回。

这用于取消事务，而不影响封闭事务的范围。

```py
method commit() → None
```

*继承自* `Transaction.commit()` *方法的* `Transaction`

提交这个`Transaction`。

根据正在使用的事务类型，此实现可能会有所不同：

+   对于一个简单的数据库事务（例如`RootTransaction`），它对应于一个提交。

+   对于`NestedTransaction`，它对应于“RELEASE SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可以使用特定于 DBAPI 的两阶段事务方法。

```py
method prepare() → None
```

准备这个`TwoPhaseTransaction`。

在准备之后，事务可以被提交。

```py
method rollback() → None
```

*继承自* `Transaction.rollback()` *方法的* `Transaction`

回滚这个`Transaction`。

其实现可能会根据使用的事务类型而有所不同：

+   对于简单的数据库事务（例如`RootTransaction`），它对应于一个 ROLLBACK。

+   对于`NestedTransaction`，它对应于“ROLLBACK TO SAVEPOINT”操作。

+   对于`TwoPhaseTransaction`，可以使用特定于 DBAPI 的方法进行两阶段事务。

## 结果集 API

| 对象名称 | 描述 |
| --- | --- |
| ChunkedIteratorResult | 从生成迭代器的可调用对象中工作的`IteratorResult`。 |
| CursorResult | 代表来自 DBAPI 游标的状态的结果。 |
| FilterResult | 一个`Result`的包装器，返回的是除`Row`对象之外的对象，例如字典或标量对象。 |
| FrozenResult | 表示适用于缓存的“冻结”状态的`Result`对象。 |
| IteratorResult | 从 Python 迭代器中获取`Row`对象或类似行数据的`Result`。 |
| MappingResult | 一个`Result`的包装器，返回的是字典值而不是`Row`值。 |
| MergedResult | 从任意数量的`Result`对象合并而成的`Result`。 |
| Result | 代表一组数据库结果。 |
| Row | 代��单个结果行。 |
| RowMapping | 将列名和对象映射到`Row`值的`Mapping`。 |
| ScalarResult | 一个`Result`的包装器，返回的是标量值而不是`Row`值。 |
| TupleResult | 一个`Result`，其返回的是普通的 Python 元组而不是行。 |

```py
class sqlalchemy.engine.ChunkedIteratorResult
```

从生成迭代器的可调用函数中工作的`IteratorResult`。

给定的`chunks`参数是一个函数，该函数给出每个块中要返回的行数，或者为所有行返回`None`。然后，该函数应返回一个未消耗的列表迭代器，每个列表的大小为请求的大小。

该函数可以随时再次调用，在这种情况下，它应该继续从相同的结果集开始，但根据给定的块大小进行调整。

版本 1.4 中的新功能。

**成员**

yield_per()

**类签名**

类`sqlalchemy.engine.ChunkedIteratorResult`（`sqlalchemy.engine.IteratorResult`）

```py
method yield_per(num: int) → Self
```

配置行提取策略以一次提取`num`行。

当迭代结果对象或以其他方式使用返回一行的方法（如`Result.fetchone()`）时，这会影响结果的基础行为。来自底层游标或其他数据源的数据将在内存中缓冲到这么多行，并且缓冲集合将一次性输出一行或请求的行数。每次缓冲清除时，它将刷新到这么多行或如果剩余的行数较少，则刷新到剩余的行数。

`Result.yield_per()`方法通常与`Connection.execution_options.stream_results`执行选项一起使用，这将允许正在使用的数据库方言使用服务器端游标，如果 DBAPI 支持与其默认操作模式分开的特定“服务器端游标”模式。

提示

考虑使用`Connection.execution_options.yield_per`执行选项，同时设置`Connection.execution_options.stream_results`以确保使用服务器端游标，并自动调用`Result.yield_per()`方法一次性建立固定的行缓冲区大小。

ORM 操作可使用`Connection.execution_options.yield_per`执行选项，在使用 Yield Per 获取大型结果集中描述了面向`Session`的用法。仅适用于核心的版本，可与`Connection`一起使用，这是 SQLAlchemy 1.4.40 的新功能。

1.4 版本中的新功能。

参数：

**num** – 每次重新填充缓冲区时要获取的行数。如果设置为小于 1 的值，则获取下一个缓冲区的所有行。

另请参阅

使用服务器端游标（也称为流式结果） - 描述了`Result.yield_per()`的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.engine.CursorResult
```

代表来自 DBAPI 游标的状态的结果。

自版本 1.4 起更改：`CursorResult`类取代了先前的`ResultProxy`接口。这些类基于`Result`调用 API，为 SQLAlchemy 核心和 SQLAlchemy ORM 提供了更新的使用模型和调用外观。

通过`Row`类返回数据库行，该类在 DBAPI 返回的原始数据之上提供了额外的 API 功能和行为。通过诸如`Result.scalars()`方法之类的过滤器，还可以返回其他类型的对象。

另请参阅

使用 SELECT 语句 - 用于访问`CursorResult`和`Row`对象的入门材料。

**成员**

all()，close()，columns()，fetchall()，fetchmany()，fetchone()，first()，freeze()，inserted_primary_key，inserted_primary_key_rows，is_insert，keys()，last_inserted_params()，last_updated_params()，lastrow_has_defaults()，lastrowid，mappings()，merge()，one()，one_or_none()，partitions()，postfetch_cols()，prefetch_cols()，returned_defaults，returned_defaults_rows，returns_rows，rowcount，scalar()，scalar_one()，scalar_one_or_none()，scalars()，splice_horizontally()，splice_vertically()，supports_sane_multi_rowcount()，supports_sane_rowcount()，t，tuples()，unique()，yield_per()

**类签名**

类 `sqlalchemy.engine.CursorResult`（`sqlalchemy.engine.Result`）

```py
method all() → Sequence[Row[_TP]]
```

*继承自* `Result` *方法的* `Result.all()`

返回序列中的所有行。

调用后关闭结果集。后续的调用将返回一个空序列。

版本 1.4 中新增。

返回：

一系列`Row`对象。

另请参阅

使用服务器端游标（也称为流式结果） - 如何在 python 中流式处理大型结果集而不完全加载它。

```py
method close() → Any
```

关闭此`CursorResult`。

如果仍然存在，则关闭与语句执行相对应的底层 DBAPI 游标。请注意，当`CursorResult`耗尽所有可用行时，DBAPI 游标会自动释放。通常，`CursorResult.close()`是一个可选方法，除非在丢弃仍有待获取的额外行的`CursorResult`时。

在调用此方法后，再调用获取方法将不再有效，并且在后续使用中将引发`ResourceClosedError`。

另请参阅

与引擎和连接一起工作

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

*继承自* `Result.columns()` *方法的* `Result`。

建立应在每行中返回的列。

此方法既可用于限制返回的列，也可用于重新排序它们。给定的表达式列表通常是一系列整数或字符串键名。它们也可以是与给定语句结构相对应的适当的`ColumnElement`对象。

2.0 版本中的更改：由于 1.4 中存在错误，`Result.columns()` 方法的行为不正确，仅使用一个索引调用该方法将导致`Result`对象产生标量值而不是`Row`对象。在 2.0 版本中，已更正此行为，使得使用单个索引调用`Result.columns()`将产生一个继续产生`Row`对象的`Result`对象，该对象仅包含单个列。

例如：

```py
statement = select(table.c.x, table.c.y, table.c.z)
result = connection.execute(statement)

for z, y in result.columns('z', 'y'):
    # ...
```

从语句本身使用列对象的示例：

```py
for z, y in result.columns(
        statement.selected_columns.c.z,
        statement.selected_columns.c.y
):
    # ...
```

新版本中新增。

参数：

***col_expressions** – 表示要返回的列。元素可以是整数行索引、字符串列名称或相应的`ColumnElement`对象，对应于选择构造。

返回：

此`Result`对象具有给定的修改。

```py
method fetchall() → Sequence[Row[_TP]]
```

*继承自* `Result.fetchall()` *方法的* `Result`

`Result.all()` 方法的同义词。

```py
method fetchmany(size: int | None = None) → Sequence[Row[_TP]]
```

*继承自* `Result.fetchmany()` *方法的* `Result`

获取许多行。

当所有行耗尽时，返回一个空序列。

此方法提供了与 SQLAlchemy 1.x.x 的向后兼容性。

要按组获取行，请使用`Result.partitions()`方法。

返回：

一系列`Row`对象。

另请参阅

`Result.partitions()`

```py
method fetchone() → Row[_TP] | None
```

*继承自* `Result.fetchone()` *方法的* `Result`

获取一行。

当所有行耗尽时，返回 None。

此方法提供了与 SQLAlchemy 1.x.x 的向后兼容性。

要仅获取结果的第一行，请使用`Result.first()`方法。要遍历所有行，请直接遍历`Result`对象。

返回：

如果未应用任何过滤器，则为`Row`对象，如果没有剩余行，则为`None`。

```py
method first() → Row[_TP] | None
```

*继承自* `Result.first()` *方法的* `Result`

获取第一行或如果没有行存在，则为`None`。

关闭结果集并丢弃剩余行。

注意

默认情况下，此方法返回一行**row**，例如元组。要返回确切的单个标量值，即第一行的第一列，请使用`Result.scalar()`方法，或结合`Result.scalars()`和`Result.first()`。

此外，与传统的 ORM `Query.first()` 方法的行为相反，对产生此`Result`的 SQL 查询不应用限制；对于在产生行之前在内存中缓冲结果的 DBAPI 驱动程序，所有行将被发送到 Python 进程，并且除了第一行外，所有行都将被丢弃。

另请参阅

ORM 查询与核心选择统一

返回：

一个`Row`对象，如果没有剩余行，则为 None。

另请参阅

`Result.scalar()`

`Result.one()`

```py
method freeze() → FrozenResult[_TP]
```

*从* `Result.freeze()` *方法继承的* `Result`

返回一个可调用对象，调用时将产生此`Result`的副本。

返回的可调用对象是`FrozenResult`的一个实例。

这用于结果集缓存。当结果尚未消耗时，必须调用该方法，调用该方法将完全消耗结果。当从缓存中检索到`FrozenResult`时，可以调用任意次数，每次调用都会产生一个新的`Result`对象，针对其存储的行集。

另请参阅

重新执行语句 - 在 ORM 中使用示例实现结果集缓存。

```py
attribute inserted_primary_key
```

返回刚插入的行的主键。

返回值是一个`Row`对象，表示主键值的命名元组，其顺序与源`Table`中配置的主键列相同。

版本 1.4.8 中更改：- `CursorResult.inserted_primary_key`的值现在是通过`Row`类的命名元组，而不是普通元组。

此访问器仅适用于未明确指定`Insert.returning()`的单行`insert()`构造。虽然大多数后端尚不支持多行插入，但可以使用`CursorResult.inserted_primary_key_rows`访问器进行访问。

请注意，指定了服务器默认子句或以其他方式不符合“自动增量”列的主键列（请参阅`Column`中的注释），并且是使用数据库端默认生成的，将在此列表中显示为`None`，除非后端支持“返回”并且插入语句以启用“隐式返回”执行。

如果执行的语句不是编译后的表达式构造或不是 insert() 构造，则引发`InvalidRequestError`。

```py
attribute inserted_primary_key_rows
```

将`CursorResult.inserted_primary_key`的值作为包含在列表中的行返回；某些方言可能也支持多行形式。

注意

如下所示，在当前的 SQLAlchemy 版本中，当使用 psycopg2 方言时，此访问器仅有 `CursorResult.inserted_primary_key` 提供的内容之外才有用。未来版本希望将此功能推广到更多方言中。

此访问器被添加以支持提供由 Psycopg2 快速执行助手 功能实现的方言，目前**仅适用于 psycopg2 方言**，该功能允许一次插入多行而仍然保留能够返回服务器生成的主键值的行为。

+   `在使用 psycopg2 方言或其他可能在即将发布的版本中支持“快速执行多个”样式插入的方言时`：在调用 INSERT 语句时，将行列表作为`Connection.execute()`的第二个参数传递时，此访问器将提供一个包含每行插入的主键值的行列表。

+   `当使用其他尚未支持此功能的方言/后端时`：此访问器仅对`单行 INSERT 语句`有用，并返回与单个元素列表中的 `CursorResult.inserted_primary_key` 相同的信息。当执行插入语句与要插入的行的列表结合时，列表将包含每个在语句中插入的行，但对于任何服务器生成的值，它将包含 `None`。

SQLAlchemy 的未来版本将进一步泛化 psycopg2 的 “快速执行辅助” 功能以适应其他方言，从而使此访问器能够更加通用。

新版本中的新特性：1.4。

请参阅也

`CursorResult.inserted_primary_key`

```py
attribute is_insert
```

如果此`CursorResult` 是执行表达式语言编译的 `insert()` 构造的结果，则为 True。

当为 True 时，这意味着`inserted_primary_key` 属性是可访问的，假设该语句未包括用户定义的“returning”构造。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys` *的* `sqlalchemy.engine._WithKeys.keys` *方法*

返回一个可迭代的视图，该视图产生由每个`Row`所代表的字符串键。

这些键可以表示由核心语句返回的列的标签，或者由 orm 执行返回的 orm 类的名称。

该视图还可以使用 Python `in` 运算符进行键包含性测试，该测试将测试视图中表示的字符串键，以及列对象等替代键。

1.4 版本中的变更：返回一个键视图对象，而不是一个普通列表。

```py
method last_inserted_params()
```

从此执行中返回已插入参数的集合。

如果执行的语句不是编译后的表达式构造或不是 insert() 构造，则会引发`InvalidRequestError`。

```py
method last_updated_params()
```

从此执行中返回已更新参数的集合。

如果执行的语句不是编译后的表达式构造或不是 update() 构造，则会引发`InvalidRequestError`。

```py
method lastrow_has_defaults()
```

从底层的 `ExecutionContext` 返回 `lastrow_has_defaults()`。

有关详细信息，请参见 `ExecutionContext`。

```py
attribute lastrowid
```

返回 DBAPI 游标上的‘lastrowid’访问器。

这是一个 DBAPI 特定的方法，仅对支持它的后端有效，对于适当的语句。它的行为在不同的后端上不一致。

当使用 insert()表达式构造时，通常不需要使用此方法；`CursorResult.inserted_primary_key` 属性提供了新插入行的主键值的元组，而不管数据库后端如何。

```py
method mappings() → MappingResult
```

*继承自* `Result.mappings()` *方法的* `Result`

对返回的行应用映射过滤器，返回`MappingResult`的实例。

当应用此过滤器时，获取行将返回`RowMapping`对象，而不是`Row`对象。

1.4 版中的新功能。

返回：

一个新的指向此`Result`对象的`MappingResult`过滤对象。

```py
method merge(*others: Result[Any]) → MergedResult[Any]
```

将此`Result`与其他兼容的结果对象合并。

返回的对象是`MergedResult`的实例，它将由给定结果对象的迭代器组成。

新结果将使用此结果对象的元数据。随后的结果对象必须与相同的结果/游标元数据匹配，否则行为未定义。

```py
method one() → Row[_TP]
```

*继承自* `Result.one()` *方法的* `Result`

返回确切的一行或引发异常。

如果结果返回零行，则引发`NoResultFound` ，或者如果将返回多行，则引发`MultipleResultsFound` 。

注意

此方法默认返回一个**行**，例如元组。要返回确切的单个标量值，即第一行的第一列，请使用`Result.scalar_one()` 方法，或结合使用`Result.scalars()` 和 `Result.one()`。

1.4 版中的新功能。

返回：

第一个`Row`。

引发：

`MultipleResultsFound`、`NoResultFound`

另请参阅

`Result.first()`

`Result.one_or_none()`

`Result.scalar_one()`

```py
method one_or_none() → Row[_TP] | None
```

*继承自* `Result.one_or_none()` *方法的* `Result`

返回最多一个结果或引发异常。

如果结果没有行，则返回 `None`。如果返回多行，则引发 `MultipleResultsFound` 异常。

新版本 1.4 中新增。

返回：

第一个 `Row` 或者 `None`（如果没有可用行）。

异常：

`MultipleResultsFound`

另请参阅

`Result.first()`

`Result.one()`

```py
method partitions(size: int | None = None) → Iterator[Sequence[Row[_TP]]]
```

*继承自* `Result.partitions()` *方法的* `Result`

迭代给定大小的行子列表。

每个列表将具有给定大小，最后一个列表不包括在内，可能具有少量行。不会生成空列表。

当迭代器被完全消耗时，结果对象会自动关闭。

请注意，除非使用 `Connection.execution_options.stream_results` 执行选项指示驱动程序不要提前缓冲结果，否则后端驱动程序通常会提前缓冲整个结果集，如果可能的话。并非所有驱动程序都支持此选项，并且对于不支持此选项的驱动程序，该选项会被悄悄忽略。

当使用 ORM 时，通常在内存方面更有效的方法是将 `Result.partitions()` 方法与 yield_per execution option 结合使用，该选项指示 DBAPI 驱动程序在可用时使用服务器端游标，并且指示 ORM 加载内部在将结果中的某些 ORM 对象建立一定数量后才将它们输出。

新版本 1.4 中新增。

参数：

**size** – 指示每个生成的列表中应存在的最大行数。如果为 None，则使用`Result.yield_per()`方法设置的值，如果调用了该方法，或者等效的`Connection.execution_options.yield_per`执行选项。如果未设置 yield_per，则使用`Result.fetchmany()`的默认值，这可能是特定于后端的，并且定义不明确。

返回:

列表的迭代器

另请参见

使用服务器端游标（也称为流式结果）

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
method postfetch_cols()
```

从底层的`ExecutionContext`返回`postfetch_cols()`。

查看`ExecutionContext`以获取详细信息。

如果执行的语句不是编译的表达式构造或不是 insert()或 update()构造，则引发`InvalidRequestError`。

```py
method prefetch_cols()
```

从底层的`ExecutionContext`返回`prefetch_cols()`。

查看`ExecutionContext`以获取详细信息。

如果执行的语句不是编译的表达式构造或不是 insert()或 update()构造，则引发`InvalidRequestError`。

```py
attribute returned_defaults
```

返回使用`ValuesBase.return_defaults()`功能获取的默认列的值。

该值是`Row`的实例，如果未使用`ValuesBase.return_defaults()`或后端不支持 RETURNING，则为`None`。

另请参见

`ValuesBase.return_defaults()`

```py
attribute returned_defaults_rows
```

返回一个包含使用`ValuesBase.return_defaults()`功能获取的默认列值的行列表。

返回值是`Row`对象的列表。

版本 1.4 中的新功能。

```py
attribute returns_rows
```

如果此`CursorResult`返回零个或多个行，则为 True。

即，如果可以调用方法 `CursorResult.fetchone()`, `CursorResult.fetchmany()` `CursorResult.fetchall()`。

总的来说，`CursorResult.returns_rows` 的值应该与 DBAPI 游标是否具有`.description`属性始终同义，指示结果列的存在，注意，即使游标返回零行，如果发出了返回行的语句，则仍然具有`.description`。

对于针对 SELECT 语句的所有结果，以及对使用 RETURNING 的 DML 语句 INSERT/UPDATE/DELETE，此属性应为 True。对于未使用 RETURNING 的 INSERT/UPDATE/DELETE 语句，该值通常为 False，但是有一些方言特定的例外情况，例如在使用 MSSQL / pyodbc 方言时，内联发出 SELECT 以检索插入的主键值时。

```py
attribute rowcount
```

返回此结果的“rowcount”。

“rowcount”的主要目的是报告一次执行（即对单个参数集）的 UPDATE 或 DELETE 语句的 WHERE 条件匹配的行数，然后可以将其与预期更新或删除的行数进行比较，作为断言数据完整性的手段。

此属性在关闭游标之前从 DBAPI 的`cursor.rowcount`属性转移，以支持不在游标关闭后提供此值的 DBAPI。某些 DBAPI 可能还为其他类型的语句提供有意义的值，例如 INSERT 和 SELECT 语句。为了检索这些语句的`cursor.rowcount`，请将 `Connection.execution_options.preserve_rowcount` 执行选项设置为 True，这将导致在返回任何结果或关闭游标之前无条件地对`cursor.rowcount`值进行备忘，而不管语句类型如何。

对于 DBAPI 不支持特定类型的语句和/或执行的 rowcount 的情况，返回的值将是`-1`，直接从 DBAPI 传递，并且是 [**PEP 249**](https://peps.python.org/pep-0249/) 的一部分。但所有的 DBAPI 都应该支持单参数集 UPDATE 和 DELETE 语句的 rowcount。

注意

关于 `CursorResult.rowcount` 的注释：

+   此属性返回*匹配*的行数，这不一定与实际*修改*的行数相同。例如，如果 UPDATE 语句中给定的 SET 值与已经存在于行中的值相同，则对于给定行可能没有净变化。这样的行将被匹配但不会被修改。在具有两种样式的后端（例如 MySQL）上，rowcount 配置为在所有情况下返回匹配计数。

+   在默认情况下，`CursorResult.rowcount` *只有*在与 UPDATE 或 DELETE 语句结合使用时才有用，并且只适用于单组参数。对于其他类型的语句，除非使用了`Connection.execution_options.preserve_rowcount`执行选项，否则 SQLAlchemy 不会尝试预先记忆该值。请注意，与[**PEP 249**](https://peps.python.org/pep-0249/)相反，许多 DBAPI 不支持不是 UPDATE 或 DELETE 的语句的 rowcount 值，特别是当返回未完全预先缓冲的行时。不支持特定类型语句的 rowcount 的 DBAPI 应为此类语句返回值`-1`。

+   当执行带有多个参数集的单个语句（即 executemany）时，可能无法理解`CursorResult.rowcount`。大多数 DBAPI 不会跨多个参数集总和“rowcount”值，并在访问时返回`-1`。

+   当`Connection.execution_options.preserve_rowcount`执行选项设置为 True 时，SQLAlchemy 的 INSERT 语句的“插入多个值”行为功能支持正确填充`CursorResult.rowcount`。

+   使用 RETURNING 的语句可能不支持 rowcount，而返回值为`-1`。

另请参阅

从 UPDATE、DELETE 语句获取受影响的行数 - 在 SQLAlchemy 统一教程中

`Connection.execution_options.preserve_rowcount`

```py
method scalar() → Any
```

*从* `Result` *的* `Result.scalar()` *方法继承*

抓取第一行的第一列，并关闭结果集。

如果没有要抓取的行，则返回`None`。

不执行验证以测试是否存在额外的行。

调用此方法后，对象将完全关闭，例如`CursorResult.close()`方法将被调用。

返回：

一个 Python 标量值，如果没有剩余行则返回`None`。

```py
method scalar_one() → Any
```

*继承自* `Result.scalar_one()` *方法的* `Result`

返回精确一个标量结果或引发异常。

这相当于调用`Result.scalars()`然后调用`Result.one()`。

参见

`Result.one()`

`Result.scalars()`

```py
method scalar_one_or_none() → Any | None
```

*继承自* `Result.scalar_one_or_none()` *方法的* `Result`

返回精确一个标量结果或`None`。

这相当于调用`Result.scalars()`然后调用`Result.one_or_none()`。

参见

`Result.one_or_none()`

`Result.scalars()`

```py
method scalars(index: _KeyIndexType = 0) → ScalarResult[Any]
```

*继承自* `Result.scalars()` *方法的* `Result`

返回一个`ScalarResult`过滤对象，该对象将返回单个元素而不是`Row`对象。

例如：

```py
>>> result = conn.execute(text("select int_id from table"))
>>> result.scalars().all()
[1, 2, 3]
```

当从`ScalarResult`过滤对象中获取结果时，将返回由`Result`返回的单列行，而不是作为列值返回。

版本 1.4 中的新功能。

参数：

**index** – 指示从每行中获取的列的整数或行键，默认为`0`，表示第一列。

返回：

一个新的指向此`Result`对象的`ScalarResult`过滤对象。

```py
method splice_horizontally(other)
```

返回一个新的`CursorResult`，将此`CursorResult`的行与另一个`CursorResult`的行“横向拼接”在一起。

提示

此方法是为了 SQLAlchemy ORM 的利益而设计，不适用于一般用途。

“横向拼接”意味着对于第一个和第二个结果集中的每一行，都会生成一个将两行连接在一起的新行，然后成为新行。传入的`CursorResult`必须具有相同数量的行。通常期望两个结果集也来自相同的排序顺序，因为结果行根据它们在结果中的位置拼接在一起。

预期的用例是，对不同表执行多个 INSERT..RETURNING 语句（肯定需要排序），可以生成一个看起来像这两个表的 JOIN 的单个结果。

例如：

```py
r1 = connection.execute(
    users.insert().returning(
        users.c.user_name,
        users.c.user_id,
        sort_by_parameter_order=True
    ),
    user_values
)

r2 = connection.execute(
    addresses.insert().returning(
        addresses.c.address_id,
        addresses.c.address,
        addresses.c.user_id,
        sort_by_parameter_order=True
    ),
    address_values
)

rows = r1.splice_horizontally(r2).all()
assert (
    rows ==
    [
        ("john", 1, 1, "foo@bar.com", 1),
        ("jack", 2, 2, "bar@bat.com", 2),
    ]
)
```

2.0 版中的新功能。

另请参阅

`CursorResult.splice_vertically()`

```py
method splice_vertically(other)
```

返回一个新的`CursorResult`，即“纵向拼接”，即将此`CursorResult`的行与另一个`CursorResult`的行“扩展”在一起。

提示

此方法是为了 SQLAlchemy ORM 的利益而设计，不适用于一般用途。

“纵向拼接”意味着给定结果的行被附加到此游标结果的行。传入的`CursorResult`必须具有与此`CursorResult`中相同顺序的相同列列表。

2.0 版中的新功能。

另请参阅

`CursorResult.splice_horizontally()`

```py
method supports_sane_multi_rowcount()
```

从方言返回`supports_sane_multi_rowcount`。

有关背景，请参阅`CursorResult.rowcount`。

```py
method supports_sane_rowcount()
```

从方言返回`supports_sane_rowcount`。

有关背景，请参阅`CursorResult.rowcount`。

```py
attribute t
```

*继承自* `Result.t` *属性的* `Result`

对返回的行应用“类型化元组”类型过滤器。

`Result.t` 属性是调用 `Result.tuples()` 方法的同义词。

2.0 版本中的新功能。

```py
method tuples() → TupleResult[_TP]
```

*来自* `Result.tuples()` *方法的继承* `Result` *方法*

对返回的行应用“类型化元组”类型过滤。

此方法在运行时返回相同的 `Result` 对象，但标注为返回一个 `TupleResult` 对象，这将指示 [**PEP 484**](https://peps.python.org/pep-0484/) 类型工具，返回的是普通类型的 `Tuple` 实例，而不是行。这允许对返回的 `Row` 对象进行元组解包和 `__getitem__` 访问，对于那些语句本身包含类型信息的情况。

2.0 版本中的新功能。

返回：

在类型时间使用 `TupleResult` 类型。

另请参阅

`Result.t` - 更短的同义词

`Row._t` - `Row` 版本

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

*来自* `Result.unique()` *方法的继承* `Result` *方法*

对由此 `Result` 返回的对象应用唯一过滤。

当没有参数应用此过滤器时，返回的行或对象将被过滤，以便每行都是唯一的。确定此唯一性的算法默认为整个元组的 Python 散列标识。在某些情况下，可能会使用专门的每个实体散列方案，例如当使用 ORM 时，会应用一种针对返回对象的主键标识的方案。

唯一过滤器在 **所有其他过滤器之后** 应用，这意味着如果返回的列已经使用 `Result.columns()` 或 `Result.scalars()` 等方法进行了细化，唯一性将仅应用于**返回的列或列**。这不受这些方法被调用时的顺序对 `Result` 对象的影响。

唯一过滤器还改变了像 `Result.fetchmany()` 和 `Result.partitions()` 这样的方法所使用的计算方法。当使用 `Result.unique()` 时，这些方法将继续产出请求的行数或对象，在唯一化应用后。然而，这必然会影响底层游标或数据源的缓冲行为，以至于可能需要多次调用 `cursor.fetchmany()` 才能积累足够的对象以提供所请求大小的唯一集合。

参数：

**策略** - 一个应用于被迭代的行或对象的可调用函数，应返回表示行的唯一值的对象。Python 的 `set()` 用于存储这些标识。如果未传递，则使用默认的唯一性策略，该策略可能由此 `Result` 对象的源组装而成。

```py
method yield_per(num: int) → Self
```

配置行获取策略以一次获取 `num` 行。

这会影响在迭代结果对象时的底层行为，或者以其他方式使用诸如 `Result.fetchone()` 这样一次返回一行的方法时的行为。来自底层游标或其他数据源的数据将在内存中缓冲到这么多行，并且缓冲的集合然后将以一行或所请求的行数被产出。每次缓冲清除时，它将被刷新到这么多行或者如果剩余的行数更少则剩余的行数。

`Result.yield_per()` 方法通常与 `Connection.execution_options.stream_results` 执行选项一起使用，这将允许正在使用的数据库方言利用服务器端游标，如果 DBAPI 支持与其默认操作模式分离的特定“服务器端游标”模式。

提示

考虑使用 `Connection.execution_options.yield_per` 执行选项，这将同时设置 `Connection.execution_options.stream_results` 以确保使用服务器端游标，并自动调用 `Result.yield_per()` 方法以一次性建立固定的行缓冲区大小。

`Connection.execution_options.yield_per` 执行选项适用于 ORM 操作，使用 `Session` 导向的用法在 使用 Yield Per 获取大型结果集 中描述。仅适用于与 `Connection` 一起使用的 Core 版本是 SQLAlchemy 1.4.40 的新功能。

新版本 1.4 中新增。

参数：

**num** – 每次重新填充缓冲区时要获取的行数。如果设置为小于 1 的值，则获取下一个缓冲区的所有行。

另请参阅

使用服务器端游标（即流式结果） - 描述了 `Result.yield_per()` 的核心行为。

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
class sqlalchemy.engine.FilterResult
```

用于返回除 `Row` 对象之外的其他对象（例如字典或标量对象）的 `Result` 的包装器。

`FilterResult` 是其他结果 API 的常见基础，包括 `MappingResult`、`ScalarResult` 和 `AsyncResult`。

**成员**

close()，closed，yield_per()

**类签名**

类 `sqlalchemy.engine.FilterResult` (`sqlalchemy.engine.ResultInternal`)

```py
method close() → None
```

关闭此 `FilterResult`。

新版本 1.4.43 中新增。

```py
attribute closed
```

如果底层 `Result` 报告已关闭，则返回 `True`。

新版本 1.4.43 中新增。

```py
method yield_per(num: int) → Self
```

配置行获取策略，以一次获取 `num` 行。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的传递。有关使用说明，请参阅该方法的文档。

新版本 1.4.40 中新增：- 添加了 `FilterResult.yield_per()` 方法，使其在所有结果集实现中均可用。

另请参阅

使用服务器端游标（即流式结果） - 描述了 `Result.yield_per()` 的核心行为。

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中。

```py
class sqlalchemy.engine.FrozenResult
```

表示适用于缓存的 “冻结” 状态的 `Result` 对象。

任何 `Result` 对象的 `Result.freeze()` 方法返回 `FrozenResult` 对象。

每次调用 `FrozenResult` 时，从固定的数据集生成一个新的可迭代的 `Result` 对象：

```py
result = connection.execute(query)

frozen = result.freeze()

unfrozen_result_one = frozen()

for row in unfrozen_result_one:
    print(row)

unfrozen_result_two = frozen()
rows = unfrozen_result_two.all()

# ... etc
```

版本 1.4 中的新功能。

另请参见

重新执行语句 - ORM 中的示例用法，用于实现结果集缓存。

`merge_frozen_result()` - 将冻结的结果合并回 `Session` 的 ORM 函数。

**类签名**

类 `sqlalchemy.engine.FrozenResult`（`typing.Generic`）

```py
class sqlalchemy.engine.IteratorResult
```

从 Python 迭代器或类似行数据的 `Row` 对象中获取数据的 `Result`。

版本 1.4 中的新功能。

**成员**

closed

**类签名**

类 `sqlalchemy.engine.IteratorResult`（`sqlalchemy.engine.Result`）

```py
attribute closed
```

如果这个 `IteratorResult` 已经关闭，则返回 `True`。

版本 1.4.43 中的新功能。

```py
class sqlalchemy.engine.MergedResult
```

从任意数量的 `Result` 对象合并而来的 `Result`。

由 `Result.merge()` 方法返回。

版本 1.4 中的新功能。

**类签名**

类 `sqlalchemy.engine.MergedResult`（`sqlalchemy.engine.IteratorResult`）

```py
class sqlalchemy.engine.Result
```

表示一组数据库结果。

自 1.4 版本新增：`Result` 对象提供了一个完全更新的用法模型和调用门面，用于 SQLAlchemy 核心和 SQLAlchemy ORM。在核心中，它形成了 `CursorResult` 对象的基础，取代了以前的 `ResultProxy` 接口。在使用 ORM 时，通常会使用一个更高级的对象，称为 `ChunkedIteratorResult`。

注意

在 SQLAlchemy 1.4 及以上版本中，此对象用于由 `Session.execute()` 返回的 ORM 结果，该方法可以逐个返回 ORM 映射对象的实例或在类似元组的行中。请注意，`Result` 对象不会自动对实例或行进行去重，而是像旧版 `Query` 对象一样。为了在 Python 中对实例或行进行去重，请使用 `Result.unique()` 修改器方法。

另请参见

获取行 - 在 SQLAlchemy 统一教程中

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), freeze(), keys(), mappings(), merge(), one(), one_or_none(), partitions(), scalar(), scalar_one(), scalar_one_or_none(), scalars(), t, tuples(), unique(), yield_per()

**类签名**

类 `sqlalchemy.engine.Result` (`sqlalchemy.engine._WithKeys`, `sqlalchemy.engine.ResultInternal`)

```py
method all() → Sequence[Row[_TP]]
```

返回序列中的所有行。

调用后关闭结果集。后续调用将返回一个空序列。

1.4 版本新增。

返回：

一系列 `Row` 对象。

另请参见

使用服务器端游标（又名流结果） - 如何在 Python 中流式传输大型结果集而不完全加载它。

```py
method close() → None
```

关闭此`Result`。

此方法的行为是特定于实现的，并且默认情况下未实现。该方法通常应结束结果对象使用的资源，并且还应导致任何后续迭代或行提取引发`ResourceClosedError`。

新版本 1.4.27 中新增：- `.close()`先前通常不适用于所有`Result`类，而仅适用于为核心语句执行返回的`CursorResult`。由于大多数其他结果对象，即 ORM 使用的对象，无论如何都是代理一个`CursorResult`，这允许从外部门面关闭底层的游标结果，以便在 ORM 查询使用`yield_per`执行选项时关闭数据库游标而不立即用完和自动关闭。

```py
attribute closed
```

如果此`Result`报告`.closed`，则返回`True`。

新版本 1.4.43 中新增。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定每行应返回的列。

此方法可用于限制返回的列以及重新排序它们。给定的表达式列表通常是一系列整数或字符串键名。它们也可以是适当的`ColumnElement`对象，这些对象与给定的语句结构相对应。

2.0 版本更改：由于 1.4 中的一个错误，`Result.columns()`方法的行为不正确，仅使用一个索引调用该方法会导致`Result`对象产生标量值而不是`Row`对象。在 2.0 版本中，已纠正了这种行为，使得只用单个索引调用`Result.columns()`将产生一个继续生成`Row`对象的`Result`对象，该对象仅包含一个列。

例如：

```py
statement = select(table.c.x, table.c.y, table.c.z)
result = connection.execute(statement)

for z, y in result.columns('z', 'y'):
    # ...
```

使用语句本身的列对象的示例：

```py
for z, y in result.columns(
        statement.selected_columns.c.z,
        statement.selected_columns.c.y
):
    # ...
```

新版本 1.4 中新增。

参数：

***col_expressions** – 表示要返回的列。元素可以是整数行索引、字符串列名或与选择构造对应的适当`ColumnElement`对象。

返回：

使用给定的修改返回此`Result`对象。

```py
method fetchall() → Sequence[Row[_TP]]
```

`Result.all()`方法的同义词。

```py
method fetchmany(size: int | None = None) → Sequence[Row[_TP]]
```

获取多行。

当所有行都被耗尽时，返回一个空序列。

此方法是为了与 SQLAlchemy 1.x.x 向后兼容而提供的。

要按组获取行，请使用`Result.partitions()`方法。

返回：

一系列`Row`对象。

另请参见

`Result.partitions()`

```py
method fetchone() → Row[_TP] | None
```

获取一行。

当所有行都被耗尽时，返回 None。

此方法是为了与 SQLAlchemy 1.x.x 向后兼容而提供的。

要仅获取结果的第一行，请使用`Result.first()`方法。要遍历所有行，请直接遍历`Result`对象。

返回：

如果未应用任何过滤器，则为`Row`对象，否则为`None`。

```py
method first() → Row[_TP] | None
```

获取第一行或如果没有行则获取`None`。

关闭结果集并丢弃剩余行。

注意

默认情况下，此方法返回一个**行**，例如元组。要返回确切的单个标量值，即第一行的第一列，请使用`Result.scalar()`方法，或结合`Result.scalars()`和`Result.first()`。

此外，与传统 ORM `Query.first()`方法的行为相反，对生成此`Result`的 SQL 查询不应用任何限制；对于在向 Python 进程发送行之前在内存中缓冲结果的 DBAPI 驱动程序，所有行将被发送到 Python 进程，除了第一行之外的所有行将被丢弃。

另请参见

ORM 查询与核心选择统一

返回：

一个`Row`对象，如果没有行剩余则为 None。

另请参见

`Result.scalar()`

`Result.one()`

```py
method freeze() → FrozenResult[_TP]
```

返回一个可调用对象，调用时将生成此 `Result` 的副本。

返回的可调用对象是 `FrozenResult` 的实例。

这用于结果集缓存。当结果尚未使用时，必须调用该方法，并且调用该方法将完全消耗结果。当从缓存中检索到`FrozenResult`时，可以调用任意次数，它将每次针对其存储的行集生成一个新的`Result`对象。

另请参阅

重新执行语句 - 在 ORM 中的示例用法，实现结果集缓存。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys` *的* `sqlalchemy.engine._WithKeys.keys` *方法*

返回一个可迭代视图，该视图会产生每个 `Row` 所表示的字符串键。

这些键可以表示核心语句返回的列的标签，或者 orm 执行返回的 orm 类的名称。

该视图还可以使用 Python `in` 运算符进行键包含性测试，该运算符将测试视图中表示的字符串键，以及列对象等替代键。

从版本 1.4 开始更改：返回一个键视图对象，而不是一个普通列表。

```py
method mappings() → MappingResult
```

对返回的行应用映射过滤器，返回`MappingResult`的实例。

应用此过滤器后，获取行将返回 `RowMapping` 对象，而不是 `Row` 对象。

新版本中新增。

返回：

一个新的 `MappingResult` 过滤对象，引用此 `Result` 对象。

```py
method merge(*others: Result[Any]) → MergedResult[_TP]
```

将此 `Result` 与其他兼容的结果对象合并。

返回的对象是 `MergedResult` 的实例，它将由给定结果对象的迭代器组成。

新结果将使用此结果对象的元数据。随后的结果对象必须针对相同的结果/游标元数据集，否则行为是未定义的。

```py
method one() → Row[_TP]
```

返回精确地一行，或引发异常。

如果结果不返回任何行，则引发 `NoResultFound`，如果返回多行，则引发 `MultipleResultsFound`。

注意

默认情况下，此方法返回一个 **行**，例如元组。要返回确切的一个单个标量值，即第一行的第一列，请使用 `Result.scalar_one()` 方法，或者结合使用 `Result.scalars()` 和 `Result.one()`。

新版本 1.4 中新增。

返回：

第一个 `Row`。

引发：

`MultipleResultsFound`，`NoResultFound`

另请参阅

`Result.first()`

`Result.one_or_none()`

`Result.scalar_one()`

```py
method one_or_none() → Row[_TP] | None
```

返回最多一个结果或引发异常。

如果结果没有行，则返回 `None`。如果返回了多行，则引发 `MultipleResultsFound` 异常。

新版本 1.4 中新增。

返回：

第一个 `Row` 或如果没有可用行则为 `None`。

引发：

`MultipleResultsFound`

另请参阅

`Result.first()`

`Result.one()`

```py
method partitions(size: int | None = None) → Iterator[Sequence[Row[_TP]]]
```

迭代给定大小的子行列表。

每个列表将是给定大小的，除了将要生成的最后一个列表，该列表可能包含少量行。不会生成空列表。

迭代器完全被使用时，结果对象会自动关闭。

注意，除非使用 `Connection.execution_options.stream_results` 执行选项指示驱动程序尽量不要预先缓冲结果，否则后端驱动程序通常会提前缓冲整个结果。并非所有驱动程序都支持此选项，并且对于不支持此选项的驱动程序会悄悄地忽略该选项。

在使用 ORM 时，通常与使用 yield_per 执行选项 结合使用的 `Result.partitions()` 方法从内存的角度来看更有效，该选项指示 DBAPI 驱动程序在可能的情况下使用服务器端游标，并且指示 ORM 加载内部一次仅构建一定数量的 ORM 对象，然后将其传递出去。

版本 1.4 中的新功能。

参数：

**size** – 指示每个列表中应包含的最大行数。如果为 None，则使用由`Result.yield_per()`方法设置的值（如果已调用），或者使用与此相关的`Connection.execution_options.yield_per`执行选项。如果未设置 yield_per，则使用`Result.fetchmany()`的默认值，该默认值可能是特定于后端的且未定义良好的。

返回：

列表的迭代器

另请参阅

使用服务器端游标（又名流结果）

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
method scalar() → Any
```

获取第一行的第一列，并关闭结果集。

如果没有要获取的行，则返回`None`。

不执行验证以测试是否有额外的行。

调用此方法后，对象已完全关闭，例如已调用`CursorResult.close()`方法。

返回：

Python 标量值，如果没有剩余行，则返回`None`。

```py
method scalar_one() → Any
```

返回确切的一个标量结果或引发异常。

这相当于调用`Result.scalars()`然后调用`Result.one()`。

另请参阅

`Result.one()`

`Result.scalars()`

```py
method scalar_one_or_none() → Any | None
```

返回确切的一个标量结果或`None`。

这相当于调用`Result.scalars()`然后调用`Result.one_or_none()`。

另请参阅

`Result.one_or_none()`

`Result.scalars()`

```py
method scalars(index: _KeyIndexType = 0) → ScalarResult[Any]
```

返回一个将返回单个元素而不是`Row`对象的`ScalarResult`过滤对象。

例如：

```py
>>> result = conn.execute(text("select int_id from table"))
>>> result.scalars().all()
[1, 2, 3]
```

当从`ScalarResult`过滤对象中获取结果时，将返回`Result`将返回的单列行而不是作为列值的值。

版本 1.4 中的新功能。

参数：

**索引** – 指示要从每行提取的列的整数或行键，默认为`0`，表示第一列。

返回：

指向此`Result`对象的新 `ScalarResult` 过滤对象。

```py
attribute t
```

对返回的行应用“类型元组”类型过滤器。

`Result.t` 属性是调用 `Result.tuples()` 方法的同义词。

版本 2.0 中的新增功能。

```py
method tuples() → TupleResult[_TP]
```

对返回的行应用“类型元组”类型过滤器。

此方法在运行时返回相同的 `Result` 对象，但注释为返回一个 `TupleResult` 对象，该对象将指示 [**PEP 484**](https://peps.python.org/pep-0484/) 类型工具，而不是行，返回纯粹的类型化 `Tuple` 实例。这允许对 `Row` 对象进行元组解包和 `__getitem__` 访问，以便在语句本身包含类型信息的情况下进行类型化。

版本 2.0 中的新增功能。

返回：

在编写时，`TupleResult` 类型。

另请参见

`Result.t` - 更短的同义词

`Row._t` - `Row` 版本

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

对此`Result`返回的对象应用唯一过滤。

当不带参数应用此过滤器时，返回的行或对象将被过滤，以确保每行都是唯一的。确定此唯一性的算法默认为整个元组的 Python 哈希标识。在某些情况下，可能会使用专门的每个实体哈希方案，例如在使用 ORM 时，将应用一种针对返回对象的主键标识的方案。

唯一过滤器在**所有其他过滤器之后**应用，这意味着如果通过方法（如 `Result.columns()` 或 `Result.scalars()`）精细化返回的列，则唯一性仅应用于**返回的列或列**。这发生在无论这些方法被调用于 `Result` 对象的顺序如何。

唯一过滤器还改变了诸如`Result.fetchmany()`和`Result.partitions()`等方法的计算方式。当使用`Result.unique()`时，这些方法将在应用唯一性后继续产出请求的行或对象数量。然而，这必然会影响底层游标或数据源的缓冲行为，以至于可能需要多次调用`cursor.fetchmany()`才能累积足够的对象以提供请求大小的唯一集合。

参数：

**策略** - 一个可调用对象，将应用于被迭代的行或对象，应返回代表行的唯一值的对象。Python 的`set()`用于存储这些标识。如果未传递，则将使用默认的唯一性策略，该策略可能已由此`Result`对象的来源组装而成。

```py
method yield_per(num: int) → Self
```

配置行提取策略以一次提取`num`行。

当迭代结果对象或者利用诸如`Result.fetchone()`这样一次返回一行的方法时，这会影响结果的基本行为。来自底层游标或其他数据源的数据将在内存中缓冲多达这么多行，然后缓冲集合将一次产出一行或者根据请求产出多行。每次缓冲清空时，它将刷新到这么多行或者如果剩余行数较少，则刷新为剩余行数。

`Result.yield_per()`方法通常与`Connection.execution_options.stream_results`执行选项一起使用，这将允许正在使用的数据库方言利用服务器端游标，如果 DBAPI 支持与其默认操作模式分开的特定“服务器端游标”模式。

提示

考虑使用`Connection.execution_options.yield_per`执行选项，它将同时设置`Connection.execution_options.stream_results`以确保使用服务器端游标，并自动调用`Result.yield_per()`方法一次性建立固定的行缓冲区大小。

`Connection.execution_options.yield_per` 执行选项适用于 ORM 操作，具体使用方法在 使用 Yield Per 获取大型结果集 中进行了描述。与仅适用于 `Connection` 的 Core 版本是 SQLAlchemy 1.4.40 的新功能。

1.4 版本中的新功能。

参数：

**num** – 每次缓冲区重新填充时要获取的行数。如果设置为小于 1 的值，则获取下一个缓冲区的所有行。

另请参阅

使用服务器端游标（又名流结果） - 描述 `Result.yield_per()` 的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
class sqlalchemy.engine.ScalarResult
```

一个 `Result` 的包装器，返回标量值而不是 `Row` 值。

调用 `Result.scalars()` 方法获取 `ScalarResult` 对象。

`ScalarResult` 的一个特殊限制是它没有 `fetchone()` 方法；由于 `fetchone()` 的语义是 `None` 值表示没有更多的结果，这与 `ScalarResult` 不兼容，因为无法区分 `None` 是作为行值还是作为指示符。使用 `next(result)` 逐个接收值。

**成员**

all(), close(), closed, fetchall(), fetchmany(), first(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类 `sqlalchemy.engine.ScalarResult` (`sqlalchemy.engine.FilterResult`)

```py
method all() → Sequence[_R]
```

返回一个序列中的所有标量值。

等同于`Result.all()`，不同之处在于返回标量值，而不是`Row`对象。

```py
method close() → None
```

*继承自* `FilterResult.close()` *方法的* `FilterResult`

关闭此`FilterResult`。

新版本 1.4.43 中新增。

```py
attribute closed
```

*继承自* `FilterResult.closed` *属性的* `FilterResult`

如果底层`Result`报告已关闭，则返回`True`。

新版本 1.4.43 中新增。

```py
method fetchall() → Sequence[_R]
```

与`ScalarResult.all()`方法同义。

```py
method fetchmany(size: int | None = None) → Sequence[_R]
```

获取多个对象。

等同于`Result.fetchmany()`，不同之处在于返回标量值，而不是`Row`对象。

```py
method first() → _R | None
```

获取第一个对象，如果没有对象则返回`None`。

等同于`Result.first()`，不同之处在于返回标量值，而不是`Row`对象。

```py
method one() → _R
```

返回正好一个对象，否则引发异常。

等同于`Result.one()`，不同之处在于返回标量值，而不是`Row`对象。

```py
method one_or_none() → _R | None
```

返回最多一个对象，否则引发异常。

等同于`Result.one_or_none()`，不同之处在于返回标量值，而不是`Row`对象。

```py
method partitions(size: int | None = None) → Iterator[Sequence[_R]]
```

迭代给定大小的子元素列表。

等同于`Result.partitions()`，不同之处在于返回标量值，而不是`Row`对象。

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

对由此`ScalarResult`返回的对象应用唯一过滤。

查看`Result.unique()`以获取使用详情。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

配置行提取策略，一次提取`num`行。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的传递。请参阅该方法的文档以获取使用说明。

版本 1.4.40 中的新增内容：- 添加了 `FilterResult.yield_per()`，以便在所有结果集实现中可用该方法。

另请参阅

使用服务器端游标（即流式结果） - 描述了 `Result.yield_per()` 的核心行为。

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中。

```py
class sqlalchemy.engine.MappingResult
```

对返回字典值而不是 `Row` 值的 `Result` 的包装器。

调用 `Result.mappings()` 方法获取的 `MappingResult` 对象。

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), keys(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类 `sqlalchemy.engine.MappingResult` (`sqlalchemy.engine._WithKeys`, `sqlalchemy.engine.FilterResult`)

```py
method all() → Sequence[RowMapping]
```

返回序列中的所有标量值。

与 `Result.all()` 等效，只是返回的是 `RowMapping` 值，而不是 `Row` 对象。

```py
method close() → None
```

*继承自* `FilterResult.close()` *方法的* `FilterResult`

关闭此 `FilterResult`。

版本 1.4.43 中的新增内容。

```py
attribute closed
```

*继承自* `FilterResult.closed` *属性的* `FilterResult`

如果底层`Result`报告已关闭，则返回 `True`。

新版本 1.4.43 中新增。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定每行应返回的列。

```py
method fetchall() → Sequence[RowMapping]
```

`MappingResult.all()` 方法的同义词。

```py
method fetchmany(size: int | None = None) → Sequence[RowMapping]
```

获取多个对象。

等同于`Result.fetchmany()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method fetchone() → RowMapping | None
```

获取一个对象。

等同于`Result.fetchone()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method first() → RowMapping | None
```

获取第一个对象或如果不存在对象则返回 `None`。

等同于`Result.first()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys.keys` *方法的* `sqlalchemy.engine._WithKeys`

返回一个可迭代视图，该视图产生每个`Row`将表示的字符串键。

键可以表示核心语句返回的列的标签，也可以表示 orm 执行返回的 orm 类的名称。

这个视图也可以使用 Python 的 `in` 运算符进行键包含性测试，这将同时测试视图中表示的字符串键以及列对象等备用键。

1.4 版本中的变化：返回键视图对象而不是普通列表。

```py
method one() → RowMapping
```

返回一个对象或引发异常。

等同于`Result.one()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method one_or_none() → RowMapping | None
```

返回最多一个对象或引发异常。

等同于`Result.one_or_none()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method partitions(size: int | None = None) → Iterator[Sequence[RowMapping]]
```

迭代给定大小的元素子列表。

等同于`Result.partitions()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method unique(strategy: Callable[[Any], Any] | None = None) → Self
```

对由此`MappingResult`返回的对象应用唯一过滤。

有关使用详细信息，请参阅`Result.unique()`。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

配置行提取策略以一次提取`num`行。

`FilterResult.yield_per()`方法是对`Result.yield_per()`方法的传递。请参阅该方法的文档以获取使用说明。

新版本 1.4.40 中新增：- 添加了`FilterResult.yield_per()`，使得该方法在所有结果集实现中都可用

请参阅

使用服务器端游标（也称为流式结果） - 描述了`Result.yield_per()`的核心行为

使用每次产出大型结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.engine.Row
```

表示单个结果行。

`Row`对象表示数据库结果的一行。在 SQLAlchemy 1.x 系列中，它通常与`CursorResult`对象相关联，但自 SQLAlchemy 1.4 以来，也被 ORM 用于类似元组的结果。

`Row`对象尽可能地类似于 Python 命名元组。有关在行上进行映射（即字典）行为的信息，例如测试是否包含键，请参阅`Row._mapping`属性。

请参阅

使用 SELECT 语句 - 包含从 SELECT 语句中选择行的示例。

在版本 1.4 中更改：将 `RowProxy` 重命名为 `Row`。`Row` 不再是“代理”对象，它包含其中的最终数据形式，并且现在主要像一个命名元组。类似映射功能移至 `Row._mapping` 属性。有关此更改的背景，请参见 RowProxy is no longer a “proxy”; is now called Row and behaves like an enhanced named tuple。

**成员**

_asdict(), _fields, _mapping, _t, _tuple(), count, index, t, tuple()

**类签名**

类 `sqlalchemy.engine.Row` (`sqlalchemy.engine._py_row.BaseRow`, `collections.abc.Sequence`, `typing.Generic`) 的同义词。

```py
method _asdict() → Dict[str, Any]
```

返回一个将字段名映射到其对应值的新字典。

此方法类似于 Python 命名元组的 `._asdict()` 方法，通过将 `dict()` 构造函数应用于 `Row._mapping` 属性来实现。

新版本 1.4 中新增。

另请参阅

`Row._mapping`

```py
attribute _fields
```

返回由此 `Row` 表示的字符串键的元组。

键可以表示核心语句返回的列的标签或 orm 执行返回的 orm 类的名称。

此属性类似于 Python 命名元组的 `._fields` 属性。

新版本 1.4 中新增。

另请参阅

`Row._mapping`

```py
attribute _mapping
```

为此 `Row` 返回一个 `RowMapping`。

此对象为行中包含的数据提供一致的 Python 映射（即字典）接口。`Row` 本身的行为类似于命名元组。

另请参阅

`Row._fields`

新版本 1.4 中新增。

```py
attribute _t
```

`Row._tuple()` 的同义词。

新版本 2.0.19 中新增：- `Row._t` 属性取代了以前的 `Row.t` 属性，现在加下划线以避免与列名发生命名冲突，类似于 `Row` 上的其他命名元组方法��

另请参阅

`Result.t`

```py
method _tuple() → _TP
```

返回此`Row`的‘tuple’形式。

在运行时，此方法返回“self”；`Row`对象已经是一个命名元组。但是，在类型级别，如果此`Row`被类型化，那么“tuple”返回类型将是一个[**PEP 484**](https://peps.python.org/pep-0484/) `Tuple`数据类型，其中包含有关各个元素的类型信息，支持类型解包和属性访问。

新版本 2.0.19 中：- `Row._tuple()` 方法取代了以前的`Row.tuple()` 方法，现在已添加下划线以避免与列名冲突，与`Row`上的其他命名元组方法一样。

另请参阅

`Row._t` - 简写属性表示法

`Result.tuples()`

```py
attribute count
```

```py
attribute index
```

```py
attribute t
```

`Row._tuple()`的同义词。

自版本 2.0.19 起已弃用：`Row.t` 属性已弃用，推荐使用`Row._t`；所有`Row` 方法和库级属性都应以下划线开头，以避免名称冲突。请使用`Row._t`。

版本 2.0 中新增。

```py
method tuple() → _TP
```

返回此`Row`的‘tuple’形式。

自版本 2.0.19 起已弃用：`Row.tuple()` 方法已弃用，推荐使用`Row._tuple()`；所有`Row` 方法和库级属性都应以下划线开头，以避免名称冲突。请使用`Row._tuple()`。

版本 2.0 中新增。

```py
class sqlalchemy.engine.RowMapping
```

一个将列名和对象映射到`Row`值的`Mapping`。

通过`Result.mappings()`方法返回的`MappingResult`对象提供的可迭代接口，可以从`Row`通过`Row._mapping`属性获得`RowMapping`。

`RowMapping` 提供 Python 映射（即字典）访问行内容。 这包括支持测试特定键（字符串列名或对象）的包含性，以及键、值和项的迭代：

```py
for row in result:
    if 'a' in row._mapping:
        print("Column 'a': %s" % row._mapping['a'])

    print("Column b: %s" % row._mapping[table.c.b])
```

新版本 1.4 中：`RowMapping` 对象取代了以前由数据库结果行提供的类似映射的访问，现在它主要行为类似于命名元组。

**成员**

items(), keys(), values()

**类签名**

类 `sqlalchemy.engine.RowMapping` (`sqlalchemy.engine._py_row.BaseRow`, `collections.abc.Mapping`, `typing.Generic`)

```py
method items() → ROMappingItemsView
```

返回底层 `Row` 中元素的键/值元组的视图。

```py
method keys() → RMKeyView
```

返回底层 `Row` 中表示的字符串列名的‘keys’的视图。

```py
method values() → ROMappingKeysValuesView
```

返回底层 `Row` 中表示的值的视图。

```py
class sqlalchemy.engine.TupleResult
```

一个将以普通 Python 元组形式返回而不是行的 `Result`。

由于 `Row` 在任何方面都像一个元组，因此这个类仅用于类型注释，正常的 `Result` 在运行时仍然被使用。

**类签名**

类 `sqlalchemy.engine.TupleResult` (`sqlalchemy.engine.FilterResult`, `sqlalchemy.util.langhelpers.TypingOnly`)
