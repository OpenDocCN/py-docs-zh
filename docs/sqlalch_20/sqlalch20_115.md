# 连接 / 引擎

> 原文：[`docs.sqlalchemy.org/en/20/faq/connections.html`](https://docs.sqlalchemy.org/en/20/faq/connections.html)

+   我如何配置日志记录？

+   我如何池化数据库连接？我的连接被池化了吗？

+   我如何传递自定义连接参数给我的数据库 API？

+   “MySQL 服务器已断开连接”

+   “命令不同步；您现在无法运行此命令” / “此结果对象不返回行。它已被自动关闭”

+   如何自动“重试”语句执行？

    +   使用 DBAPI 自动提交允许透明重连的只读版本

+   为什么 SQLAlchemy 发出那么多回滚？

    +   我正在使用 MyISAM - 如何关闭它？

    +   我正在使用 SQL Server - 如何将那些回滚变成提交？

+   我正在使用 SQLite 数据库的多个连接（通常用于测试事务操作），但我的测试程序不起作用！

+   在使用引擎时如何获取原始 DBAPI 连接？

    +   访问 asyncio 驱动程序的底层连接

+   如何在 Python 多进程或 os.fork() 中使用引擎 / 连接 / 会话？

## 我如何配置日志记录？

参见 配置日志记录。

## 我如何池化数据库连接？我的连接被池化了吗？

SQLAlchemy 在大多数情况下会自动执行应用程序级别的连接池。对于所有包含的方言（除了在使用“内存”数据库时的 SQLite 外），`Engine` 对象都指向 `QueuePool` 作为连接的来源。

更多细节，请参阅 引擎配置 和 连接池。

## 我如何传递自定义连接参数给我的数据库 API？

`create_engine()` 调用可以通过 `connect_args` 关键字参数直接接受附加参数：

```py
e = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/test", connect_args={"encoding": "utf8"}
)
```

或者对于基本的字符串和整数参数，它们通常可以在 URL 的查询字符串中指定：

```py
e = create_engine("mysql+mysqldb://scott:tiger@localhost/test?encoding=utf8")
```

另请参见

自定义 DBAPI connect() 参数 / 连接时例程

## “MySQL 服务器已断开连接”

此错误的主要原因是 MySQL 连接已超时并已被服务器关闭。 MySQL 服务器会关闭空闲了一段时间（默认为八小时）的连接。 为了适应此情况，可立即设置为启用 `create_engine.pool_recycle` 设置，这将确保比一定时间旧的连接在下次检出时将被丢弃并替换为新连接。

对于更一般的情况，如适应数据库重新启动和由于网络问题而导致的临时连接丢失，池中的连接可能会在响应更广泛的断开连接检测技术时进行回收利用。 章节 处理断开连接 提供了关于“悲观”（例如预检）和“乐观”（例如优雅恢复）技术的背景。 现代 SQLAlchemy 倾向于采用“悲观”方法。

另请参见

处理断开连接

## “命令不同步；您现在无法运行此命令” / “此结果对象不返回行。 它已被自动关闭”

MySQL 驱动程序存在一类失败模式，其中与服务器的连接状态处于无效状态。 通常，当再次使用连接时，将出现这两种错误消息之一。 原因是服务器的状态已更改为客户端库不期望的状态，因此当客户端库在连接上发出新语句时，服务器不会如预期地响应。

在 SQLAlchemy 中，由于数据库连接是池化的，连接上的消息不同步的问题变得更加重要，因为当操作失败时，如果连接本身处于不可用状态，如果它再次返回到连接池中，那么在再次检出时将会发生故障。 对此问题的缓解措施是当发生这种故障模式时连接被**作废**，以便底层 MySQL 数据库连接被丢弃。 对于许多已知的故障模式，此作废会自动发生，也可以通过 `Connection.invalidate()` 方法显式调用。

在此类别中还存在第二类故障模式，其中上下文管理器（例如`with session.begin_nested():`）希望在发生错误时“回滚”事务； 但是在某些连接的故障模式中，回滚本身（也可以是 RELEASE SAVEPOINT 操作）也会失败，导致误导性的堆栈跟踪。

最初，此错误的原因相当简单，它意味着多线程程序从多个线程调用单个连接上的命令。 这适用于原始的“MySQLdb”本机 C 驱动程序，这几乎是唯一使用的驱动程序。 但是，随着纯 Python 驱动程序（如 PyMySQL 和 MySQL-connector-Python）的引入，以及诸如 gevent/eventlet、多处理（通常与 Celery 一起使用）等工具的增加使用，已知有一整套因素会导致这个问题，其中一些因素已经在 SQLAlchemy 的不同版本中得到改进，但其他因素是无法避免的：

+   **在线程之间共享连接** - 这是这类错误发生的最初原因。 程序在同一时间在两个或多个线程中使用同一个连接，这意味着多组消息在连接上混合在一起，将服务器端会话置于客户端不再知道如何解释的状态。 但是，如今通常更有可能出现其他原因。

+   **在进程之间共享连接的文件句柄** - 这通常发生在程序使用`os.fork()`生成新进程时，父进程中存在的 TCP 连接被共享到一个或多个子进程。 由于多个进程现在向本质上是相同文件句柄的服务器发送消息，因此服务器接收到交错的消息并破坏连接的状态。

    如果程序使用 Python 的“multiprocessing”模块，并使用在父进程中创建的`Engine`，则此场景可能非常容易发生。 使用工具如 Celery 时通常会使用“multiprocessing”。 正确的方法应该是在子进程首次启动时生成一个新的`Engine`，丢弃从父进程传递下来的任何`Engine`； 或者，从父进程继承的`Engine`可以通过调用`Engine.dispose()`来处理其内部连接池。

+   **使用 Greenlet Monkeypatching w/ Exits** - 当使用像 gevent 或 eventlet 这样的库对 Python 网络 API 进行 monkeypatch 时，像 PyMySQL 这样的库现在以异步模式运行，即使它们并没有明确针对这种模型开发。一个常见问题是 greenthread 被中断，通常是由于应用程序中的超时逻辑。这导致引发`GreenletExit`异常，并且纯 Python MySQL 驱动程序被中断了其工作，可能是正在接收来自服务器的响应或准备以其他方式重置连接状态。当异常中断所有这些工作时，客户端和服务器之间的对话现在不同步，后续使用连接可能会失败。从版本 1.1.0 开始，SQLAlchemy 知道如何防范这种情况，如果数据库操作被所谓的“退出异常”中断，其中包括`GreenletExit`和任何不是`Exception`的 Python `BaseException`的子类，连接将被作废。

+   **回滚/SAVEPOINT 释放失败** - 某些类别的错误导致连接在事务上下文中无法使用，以及在“SAVEPOINT”块中操作时。在这些情况下，连接上的失败使任何 SAVEPOINT 不再存在，但当 SQLAlchemy 或应用程序尝试“回滚”此保存点时，“RELEASE SAVEPOINT”操作失败，通常会显示类似“savepoint does not exist”的消息。在这种情况下，在 Python 3 下会输出一系列异常，其中最终的错误“原因”也将被显示。在 Python 2 下，没有“链接”异常，但是最近的 SQLAlchemy 版本将尝试发出警告，说明原始失败原因，同时仍会抛出立即错误，即 ROLLBACK 的失败。## 如何自动“重试”语句执行？

文档部分处理断开连接讨论了对已自上次检查特定连接以来已断开的连接可用的策略。在这方面最现代的功能是`create_engine.pre_ping`参数，它允许在从池中检索数据库连接时发出“ping”，如果当前连接已断开，则重新连接。

需要注意的是，“ping” 仅在连接实际用于操作之前发出。一旦连接交付给调用方，根据 Python DBAPI 规范，它现在将受到 **自动启动** 操作的影响，这意味着当首次使用连接时将自动开始一个新事务，该事务将在后续语句中保持有效，直到调用 DBAPI 级别的 `connection.commit()` 或 `connection.rollback()` 方法。

在 SQLAlchemy 的现代用法中，一系列 SQL 语句始终在这个事务状态下调用，假设未启用 DBAPI 自动提交模式（关于此后面会有更多介绍），这意味着没有单个语句会自动提交；如果操作失败，当前事务中所有语句的效果将丢失。

这对于“重试”语句的概念意味着在默认情况下，当连接丢失时，**整个事务都将丢失**。数据库无法“重新连接和重试”并继续之前的操作，因为数据已经丢失。因此，SQLAlchemy 没有一个在事务中途重新连接的透明“重连”功能。处理中途断开连接的操作的标准方法是**从事务开始处重新尝试整个操作**，通常通过使用一个自定义的 Python 装饰器，该装饰器会多次“重试”特定函数直到成功，或以其他方式设计应用程序以使其能够抵御因事务断开而导致操作失败。

还有一个扩展概念，可以跟踪事务中执行的所有语句，然后在新事务中重播它们以近似“重试”操作。SQLAlchemy 的 事件系统 确实允许构建这样一个系统，但这种方法通常也不太有用，因为无法保证这些 DML 语句是否针对相同的状态进行操作，一旦事务结束，新事务中的数据库状态可能完全不同。在事务操作开始和提交的地方显式地构建“重试”到应用程序中仍然是更好的方法，因为应用程序级别的事务方法最了解如何重新运行它们的步骤。

否则，如果 SQLAlchemy 提供了一个在事务中途自动且悄无声息地“重新连接”连接的功能，那么效果将是数据被悄无声息地丢失。通过试图隐藏问题，SQLAlchemy 将使情况变得更糟。

但是，如果我们 **不** 使用事务，那么就会有更多的选项可用，下一节将描述这些选项。

### 使用 DBAPI 自动提交允许透明重新连接的只读版本

在说明不具有透明重新连接机制的基础上，上一节假设应用实际上正在使用 DBAPI 级别的事务。由于大多数 DBAPI 现在提供了本机的“自动提交”设置，我们可以利用这些特性为**只读，仅自动提交操作**提供有限的透明重新连接形式。可以将透明语句重试应用于 DBAPI 的`cursor.execute()`方法，但是仍然不安全应用于 DBAPI 的`cursor.executemany()`方法，因为语句可能已经消耗了给定参数的任何部分。

警告

不应将以下方案用于编写数据的操作。用户应该仔细阅读并了解该方案的工作原理，并在将该方案投入生产使用之前对特定的目标 DBAPI 驱动程序非常仔细地测试故障模式。重试机制不能保证在所有情况下防止断开连接错误。

可以通过利用`DialectEvents.do_execute()`和`DialectEvents.do_execute_no_params()`钩子向 DBAPI 级别的`cursor.execute()`方法应用简单的重试机制，该机制将能够在语句执行期间拦截断开连接。它将**不会**拦截在结果集获取操作期间的连接失败，对于那些不完全缓冲结果集的 DBAPI。该方案要求数据库支持 DBAPI 级别的自动提交，并且**不能保证**适用于特定的后端。提供了一个名为`reconnecting_engine()`的单一函数，它将事件钩子应用于给定的`Engine`对象，返回一个总是自动提交的版本，该版本支持 DBAPI 级别的自动提交。连接将在单参数和无参数语句执行时自动重新连接：

```py
import time

from sqlalchemy import event

def reconnecting_engine(engine, num_retries, retry_interval):
    def _run_with_retries(fn, context, cursor_obj, statement, *arg, **kw):
        for retry in range(num_retries + 1):
            try:
                fn(cursor_obj, statement, context=context, *arg)
            except engine.dialect.dbapi.Error as raw_dbapi_err:
                connection = context.root_connection
                if engine.dialect.is_disconnect(raw_dbapi_err, connection, cursor_obj):
                    if retry > num_retries:
                        raise
                    engine.logger.error(
                        "disconnection error, retrying operation",
                        exc_info=True,
                    )
                    connection.invalidate()

                    # use SQLAlchemy 2.0 API if available
                    if hasattr(connection, "rollback"):
                        connection.rollback()
                    else:
                        trans = connection.get_transaction()
                        if trans:
                            trans.rollback()

                    time.sleep(retry_interval)
                    context.cursor = cursor_obj = connection.connection.cursor()
                else:
                    raise
            else:
                return True

    e = engine.execution_options(isolation_level="AUTOCOMMIT")

    @event.listens_for(e, "do_execute_no_params")
    def do_execute_no_params(cursor_obj, statement, context):
        return _run_with_retries(
            context.dialect.do_execute_no_params, context, cursor_obj, statement
        )

    @event.listens_for(e, "do_execute")
    def do_execute(cursor_obj, statement, parameters, context):
        return _run_with_retries(
            context.dialect.do_execute, context, cursor_obj, statement, parameters
        )

    return e
```

给定上述方案，可以使用以下概念验证脚本演示事务中的重新连接。运行一次后，它将每五秒向数据库发出一个`SELECT 1`语句：

```py
from sqlalchemy import create_engine
from sqlalchemy import select

if __name__ == "__main__":
    engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True)

    def do_a_thing(engine):
        with engine.begin() as conn:
            while True:
                print("ping: %s" % conn.execute(select([1])).scalar())
                time.sleep(5)

    e = reconnecting_engine(
        create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True),
        num_retries=5,
        retry_interval=2,
    )

    do_a_thing(e)
```

在脚本运行时重新启动数据库，以演示透明重新连接操作：

```py
$ python reconnect_test.py
ping: 1
ping: 1
disconnection error, retrying operation
Traceback (most recent call last):
  ...
MySQLdb._exceptions.OperationalError: (2006, 'MySQL server has gone away')
2020-10-19 16:16:22,624 INFO sqlalchemy.pool.impl.QueuePool Invalidate connection <_mysql.connection open to 'localhost' at 0xf59240>
ping: 1
ping: 1
...
```

上述方案已针对 SQLAlchemy 1.4 进行了测试。

## 为什么 SQLAlchemy 会发出那么多的 ROLLBACK？

SQLAlchemy 目前假定 DBAPI 连接处于“非自动提交”模式 - 这是 Python 数据库 API 的默认行为，这意味着必须假定事务始终在进行中。当连接返回时，连接池会发出`connection.rollback()`。这样可以释放连接上剩余的任何事务资源。在像 PostgreSQL 或 MSSQL 这样的数据库中，表资源会被积极锁定，这一点至关重要，以防止行和表在不再使用的连接中保持锁定。否则应用程序可能会挂起。然而，这不仅仅是为了锁定，对于任何具有任何类型事务隔离的数据库，包括具有 InnoDB 的 MySQL，这同样至关重要。如果任何连接仍在旧事务中，那么该连接返回的数据将是过时的，如果在隔离中已经在该连接上查询了该数据。有关为什么甚至在 MySQL 上可能看到过时数据的背景，请参阅[`dev.mysql.com/doc/refman/5.1/en/innodb-transaction-model.html`](https://dev.mysql.com/doc/refman/5.1/en/innodb-transaction-model.html)

### 我在 MyISAM 上 - 如何关闭它？

连接池的连接返回行为的行为可以使用`reset_on_return`进行配置：

```py
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/myisam_database",
    pool=QueuePool(reset_on_return=False),
)
```

### 我在 SQL Server 上 - 如何将那些 ROLLBACKs 转换为 COMMITs？

`reset_on_return`接受`commit`，`rollback`的值，以及`True`，`False`和`None`。设置为`commit`将导致任何连接返回到池时进行 COMMIT：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mydsn", pool=QueuePool(reset_on_return="commit")
)
```

## 我正在使用 SQLite 数据库的多个连接（通常用于测试事务操作），但我的测试程序不起作用！

如果使用 SQLite 的`:memory:`数据库，默认连接池是`SingletonThreadPool`，每个线程保持一个 SQLite 连接。因此，在同一线程中使用两个连接实际上是相同的 SQLite 连接。确保您不使用`:memory:`数据库，以便引擎将使用`QueuePool`（当前 SQLAlchemy 版本中非内存数据库的默认值）。

另请参见

线程/池行为 - 有关 PySQLite 行为的信息。

## 当使用引擎时，如何访问原始 DBAPI 连接？

使用常规的 SA 引擎级 Connection，您可以通过`Connection.connection`属性在`Connection`上获取一个经过池代理的 DBAPI 连接版本，并且对于真正的 DBAPI 连接，您可以在其上调用`PoolProxiedConnection.dbapi_connection`属性。在常规的同步驱动程序中，通常不需要访问非经过池代���的 DBAPI 连接，因为所有方法都经过代理：

```py
engine = create_engine(...)
conn = engine.connect()

# pep-249 style PoolProxiedConnection (historically called a "connection fairy")
connection_fairy = conn.connection

# typically to run statements one would get a cursor() from this
# object
cursor_obj = connection_fairy.cursor()
# ... work with cursor_obj

# to bypass "connection_fairy", such as to set attributes on the
# unproxied pep-249 DBAPI connection, use .dbapi_connection
raw_dbapi_connection = connection_fairy.dbapi_connection

# the same thing is available as .driver_connection (more on this
# in the next section)
also_raw_dbapi_connection = connection_fairy.driver_connection
```

自版本 1.4.24 起发生了变化：添加了`PoolProxiedConnection.dbapi_connection`属性，取代了先前的`PoolProxiedConnection.connection`属性，但该属性仍然可用；该属性始终提供一个符合 pep-249 同步风格的连接对象。还添加了`PoolProxiedConnection.driver_connection`属性，它将始终引用真实的驱动程序级连接，无论它展示了什么 API。

### 访问 asyncio 驱动程序的基础连接

当使用 asyncio 驱动程序时，上述方案有两个变化。首先是当使用`AsyncConnection`时，必须使用可等待方法`AsyncConnection.get_raw_connection()`来访问`PoolProxiedConnection`。在这种情况下返回的`PoolProxiedConnection`保留了一个同步风格的 pep-249 使用模式，而`PoolProxiedConnection.dbapi_connection`属性指的是一个将 asyncio 连接适配为同步风格 pep-249 API 的 SQLAlchemy 适配连接对象，换句话说，在使用 asyncio 驱动程序时会有*两层*代理。实际的 asyncio 连接可以从`driver_connection`属性中获取。以 asyncio 方式重新表述前面的示例如下：

```py
async def main():
    engine = create_async_engine(...)
    conn = await engine.connect()

    # pep-249 style ConnectionFairy connection pool proxy object
    # presents a sync interface
    connection_fairy = await conn.get_raw_connection()

    # beneath that proxy is a second proxy which adapts the
    # asyncio driver into a pep-249 connection object, accessible
    # via .dbapi_connection as is the same with a sync API
    sqla_sync_conn = connection_fairy.dbapi_connection

    # the really-real innermost driver connection is available
    # from the .driver_connection attribute
    raw_asyncio_connection = connection_fairy.driver_connection

    # work with raw asyncio connection
    result = await raw_asyncio_connection.execute(...)
```

从版本 1.4.24 开始更改：添加了`PoolProxiedConnection.dbapi_connection`和`PoolProxiedConnection.driver_connection`属性，以允许通过一致的接口访问 pep-249 连接、pep-249 适配层和底层驱动程序连接。

使用 asyncio 驱动程序时，上述“DBAPI”连接实际上是一个经过 SQLAlchemy 适配的连接形式，它呈现同步风格的 pep-249 风格 API。要访问实际的 asyncio 驱动程序连接，可以通过`PoolProxiedConnection`的`PoolProxiedConnection.driver_connection`属性进行访问。对于标准的 pep-249 驱动程序，`PoolProxiedConnection.dbapi_connection`和`PoolProxiedConnection.driver_connection`是同义词。

在将连接返回到池之前，您必须确保将任何隔离级别设置或其他特定操作设置恢复为正常状态。

作为恢复设置的替代方法，您可以在`Connection`或代理连接上调用`Connection.detach()`方法，这将使连接与池解除关联，从而在调用`Connection.close()`时关闭并丢弃连接：

```py
conn = engine.connect()
conn.detach()  # detaches the DBAPI connection from the connection pool
conn.connection.<go nuts>
conn.close()  # connection is closed for real, the pool replaces it with a new connection
```

## 如何在 Python 多进程或 os.fork()中使用引擎/连接/会话？

这在使用连接池与多进程或 os.fork()一节中有所涉及。

## 如何配置日志记录？

参见配置日志记录。

## 如何池化数据库连接？我的连接是否被池化了？

SQLAlchemy 在大多数情况下会自动执行应用程序级别的连接池。对于所有包含的方言（除了使用“内存”数据库的 SQLite），`Engine` 对象指的是一个 `QueuePool` 作为连接的来源。

更多详细信息，请参阅 引擎配置 和 连接池。

## 如何向我的数据库 API 传递自定义连接参数？

`create_engine()` 调用可以通过 `connect_args` 关键字参数直接接受附加参数：

```py
e = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/test", connect_args={"encoding": "utf8"}
)
```

或者对于基本的字符串和整数参数，它们通常可以在 URL 的查询字符串中指定：

```py
e = create_engine("mysql+mysqldb://scott:tiger@localhost/test?encoding=utf8")
```

另请参见

自定义 DBAPI connect() 参数 / 连接时例程

## “MySQL 服务器已关闭连接”

此错误的主要原因是 MySQL 连接已超时并已被服务器关闭。MySQL 服务器会关闭空闲一段时间（默认为八小时）的连接。为了适应这一点，立即设置是启用 `create_engine.pool_recycle` 设置，这将确保超过一定秒数的连接在下次检出时被丢弃并替换为新连接。

对于更一般的情况，即适应数据库重新启动和由于网络问题导致的临时连接丢失，池中的连接可能会根据更广义的断开连接检测技术进行回收。章节 处理断开连接 提供了关于“悲观”（例如，预先 ping）和“乐观”（例如，优雅恢复）技术的背景。现代 SQLAlchemy 倾向于采用“悲观”方法。

另请参见

处理断开连接

## “命令不同步；您现在无法运行此命令” / “此结果对象不返回行。它已被自动关闭”

MySQL 驱动程序存在一类相当广泛的故障模式，其中与服务器的连接状态处于无效状态。通常情况下，当再次使用连接时，将出现以下两个错误消息之一。原因是因为服务器的状态已更改为客户端库不期望的状态，因此当客户端库在连接上发出新语句时，服务器不会如预期地响应。

在 SQLAlchemy 中，由于数据库连接是池化的，连接上的消息不同步的问题变得更加重要，因为当一个操作失败时，如果连接本身处于不可用状态，如果它重新进入连接池，当再次检出时将发生故障。对于这个问题的缓解措施是，当出现这种故障模式时，连接被**作废**，以便底层数据库连接到 MySQL 被丢弃。这种作废对于许多已知的故障模式会自动发生，也可以通过`Connection.invalidate()`方法显式调用。

在这个类别中还有第二类故障模式，其中上下文管理器（如`with session.begin_nested():`）在发生错误时希望“回滚”事务；然而在某些连接的故障模式中，回滚本身（也可以是一个 RELEASE SAVEPOINT 操作）也会失败，导致误导性的堆栈跟踪。

最初，这种错误的原因通常很简单，意味着一个多线程程序从多个线程调用单个连接上的命令。这适用于最初几乎是唯一使用的原始“MySQLdb”本机 C 驱动程序。然而，随着纯 Python 驱动程序（如 PyMySQL 和 MySQL-connector-Python）的引入，以及诸如 gevent/eventlet、多进程（通常与 Celery 一起使用）等工具的增加使用，已知存在一整套因素会导致这个问题，其中一些已经在 SQLAlchemy 版本中得到改进，但另一些是不可避免的：

+   **在线程之间共享连接** - 这是这类错误发生的最初原因。程序在两个或多个线程中同时使用相同的连接，意味着多组消息在连接上混在一起，使得服务器端会话进入一个客户端不再知道如何解释的状态。然而，今天通常更可能出现其他原因。

+   **在进程之间共享连接的文件句柄** - 这通常发生在程序使用`os.fork()`生成新进程时，父进程中存在的 TCP 连接被共享到一个或多个子进程中。由于多个进程现在向基本上相同的文件句柄发送消息，服务器接收到交错的消息并破坏连接的状态。

    如果程序使用 Python 的“multiprocessing”模块，并且使用了在父进程中创建的 `Engine`，则可能会很容易发生此情况。在使用 Celery 等工具时，使用“multiprocessing”是很常见的。正确的方法应该是在子进程第一次启动时生成一个新的 `Engine`，丢弃从父进程继承下来的任何 `Engine`；或者，从父进程继承的 `Engine` 可以通过调用 `Engine.dispose()` 来处理其内部的连接池。

+   **使用 Exit 的 Greenlet Monkeypatching** - 当使用类似 gevent 或 eventlet 的库对 Python 网络 API 进行 monkeypatch 时，像 PyMySQL 这样的库现在以异步模式运行，即使它们并没有明确针对此模型开发。一个常见问题是 greenthread 被中断，通常是由于应用程序中的超时逻辑。这导致引发`GreenletExit`异常，并且纯 Python MySQL 驱动程序被中断了其工作，可能是正在接收来自服务器的响应或准备重新设置连接的状态。当异常中断了所有这些工作时，客户端和服务器之间的对话现在不再同步，连接的后续使用可能会失败。截至版本 1.1.0，SQLAlchemy 知道如何防范这种情况，因为如果数据库操作被所谓的“退出异常”中断，这包括`GreenletExit`和任何不是也是`Exception`子类的 Python `BaseException`的子类，则连接将无效。

+   **回滚 / SAVEPOINT 释放失败** - 某些类别的错误会导致连接在事务上下文中无法使用，以及在“SAVEPOINT”块中操作时无法使用。在这些情况下，连接上的故障使任何 SAVEPOINT 都不再存在，然而当 SQLAlchemy 或应用程序尝试“回滚”此 savepoint 时，“RELEASE SAVEPOINT”操作会失败，通常会出现“savepoint 不存在”的消息。在这种情况下，在 Python 3 下将输出一系列异常，其中最终的错误“原因”也将被显示出来。在 Python 2 下，没有“链接”异常，但是 SQLAlchemy 的最新版本将尝试发出警告，说明原始故障原因，同时仍然抛出 ROLLBACK 失败的立即错误。

## 如何自动“重试”语句执行？

文档部分 处理断开连接 讨论了对已经断开连接的池化连接可用的策略。在这方面最现代的特性是 `create_engine.pre_ping` 参数，它允许在从池中检索数据库连接时发出“ping”，如果当前连接已断开，则重新连接。

需要注意的是，此“ping”仅在连接实际用于操作之前发出。一旦连接被提供给调用者，根据 Python DBAPI 规范，它现在已经受到**autobegin**操作的影响，这意味着当首次使用时，它将自动开始一个新事务，该事务在后续语句中仍然有效，直到调用 DBAPI 级别的 `connection.commit()` 或 `connection.rollback()` 方法。

在现代使用 SQLAlchemy 中，一系列 SQL 语句总是在事务状态下调用，假设未启用 DBAPI 自动提交模式（下一节将详细介绍），这意味着没有单个语句会自动提交；如果操作失败，当前事务内所有语句的影响都将丢失。

对于“重试”语句的含义是，默认情况下，当连接丢失时，**整个事务都将丢失**。数据库无法以有用的方式“重新连接和重试”，并继续上次执行的位置，因为数据已经丢失。因此，SQLAlchemy 没有一个能在事务进行中工作时透明地进行“重新连接”的功能，以处理数据库连接在使用过程中断开的情况。处理中途断开连接的规范方法是**从事务开始处重试整个操作**，通常通过使用自定义 Python 装饰器多次“重试”特定函数直到成功，或者以其他方式设计应用程序，使其能够抵御事务被中断而导致操作失败的情况。

还有一个概念，即扩展程序可以跟踪事务中已经执行的所有语句，然后在新事务中重新执行它们，以近似实现“重试”操作。SQLAlchemy 的事件系统确实允许构建这样一个系统，但这种方法通常也不实用，因为没有办法保证这些 DML 语句将针对相同的状态进行操作，一旦事务结束，数据库在新事务中的状态可能会完全不同。在事务操作开始和提交的点明确将“重试”架构化到应用程序中仍然是更好的方法，因为应用程序级别的事务方法最了解如何重新运行它们的步骤。

否则，如果 SQLAlchemy 提供了一个透明且静默地在事务中重新连接连接的功能，则效果将是数据被静默丢失。通过试图隐藏问题，SQLAlchemy 将使情况变得更糟。

然而，如果我们**不**使用事务，则会有更多的选择，如下一节所述。

### 使用 DBAPI 自动提交允许只读版本的透明重新连接

由于没有透明的重新连接机制的理由已经说明，上一节建立在这样一个假设之上，即应用程序实际上正在使用 DBAPI 级别的事务。由于大多数 DBAPI 现在提供了本地的“自动提交”设置，我们可以利用这些特性来为**只读、自动提交的操作**提供有限形式的透明重新连接。可以将透明的语句重试应用于 DBAPI 的`cursor.execute()`方法，但仍然不安全应用于 DBAPI 的`cursor.executemany()`方法，因为该语句可能已经消耗了给定参数的任何部分。

警告

下面的方法**不**应用于写入数据的操作。用户应该仔细阅读和理解该方法的工作原理，并仔细针对具体目标的 DBAPI 驱动程序测试故障模式，然后再在生产中使用该方法。重试机制不能保证在所有情况下都防止断开连接错误。

可以通过利用`DialectEvents.do_execute()`和`DialectEvents.do_execute_no_params()`钩子来应用于 DBAPI 级别的`cursor.execute()`方法的简单重试机制，这将能够在语句执行期间拦截断开连接。对于那些不完全缓冲结果集的 DBAPI，它**不会**拦截在结果集获取操作期间的连接故障。该配方要求数据库支持 DBAPI 级别的自动提交，并且对于特定后端**不能保证**。提供了一个名为`reconnecting_engine()`的单个函数，它将事件钩子应用于给定的`Engine`对象，返回一个始终自动提交的版本，该版本启用了 DBAPI 级别的自动提交。连接将透明地重新连接以进行单参数和无参数语句执行：

```py
import time

from sqlalchemy import event

def reconnecting_engine(engine, num_retries, retry_interval):
    def _run_with_retries(fn, context, cursor_obj, statement, *arg, **kw):
        for retry in range(num_retries + 1):
            try:
                fn(cursor_obj, statement, context=context, *arg)
            except engine.dialect.dbapi.Error as raw_dbapi_err:
                connection = context.root_connection
                if engine.dialect.is_disconnect(raw_dbapi_err, connection, cursor_obj):
                    if retry > num_retries:
                        raise
                    engine.logger.error(
                        "disconnection error, retrying operation",
                        exc_info=True,
                    )
                    connection.invalidate()

                    # use SQLAlchemy 2.0 API if available
                    if hasattr(connection, "rollback"):
                        connection.rollback()
                    else:
                        trans = connection.get_transaction()
                        if trans:
                            trans.rollback()

                    time.sleep(retry_interval)
                    context.cursor = cursor_obj = connection.connection.cursor()
                else:
                    raise
            else:
                return True

    e = engine.execution_options(isolation_level="AUTOCOMMIT")

    @event.listens_for(e, "do_execute_no_params")
    def do_execute_no_params(cursor_obj, statement, context):
        return _run_with_retries(
            context.dialect.do_execute_no_params, context, cursor_obj, statement
        )

    @event.listens_for(e, "do_execute")
    def do_execute(cursor_obj, statement, parameters, context):
        return _run_with_retries(
            context.dialect.do_execute, context, cursor_obj, statement, parameters
        )

    return e
```

给定上述配方，可以使用以下概念验证脚本演示事务中的重新连接。运行后，它将每五秒向数据库发出一个`SELECT 1`语句：

```py
from sqlalchemy import create_engine
from sqlalchemy import select

if __name__ == "__main__":
    engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True)

    def do_a_thing(engine):
        with engine.begin() as conn:
            while True:
                print("ping: %s" % conn.execute(select([1])).scalar())
                time.sleep(5)

    e = reconnecting_engine(
        create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True),
        num_retries=5,
        retry_interval=2,
    )

    do_a_thing(e)
```

在脚本运行时重新启动数据库以演示透明重连接操作：

```py
$ python reconnect_test.py
ping: 1
ping: 1
disconnection error, retrying operation
Traceback (most recent call last):
  ...
MySQLdb._exceptions.OperationalError: (2006, 'MySQL server has gone away')
2020-10-19 16:16:22,624 INFO sqlalchemy.pool.impl.QueuePool Invalidate connection <_mysql.connection open to 'localhost' at 0xf59240>
ping: 1
ping: 1
...
```

上述配方已在 SQLAlchemy 1.4 中进行了测试。### 使用 DBAPI 自动提交允许透明重连接的只读版本

在未说明透明重连接机制的理由的情况下，前一节基于这样一种假设，即应用程序实际上正在使用 DBAPI 级别的事务。由于大多数 DBAPI 现在提供本地“自动提交”设置，我们可以利用这些特性为**只读，仅自动提交操作**提供一种有限形式的透明重连接。透明语句重试可以应用于 DBAPI 的`cursor.execute()`方法，但是仍然不安全应用于 DBAPI 的`cursor.executemany()`方法，因为该语句可能已经消耗了给定参数的任何部分。

警告

不应将以下配方用于写入数据的操作。用户应仔细阅读和理解配方的工作原理，并在生产使用此配方之前针对特定的 DBAPI 驱动程序非常仔细地测试故障模式。重试机制不能保证在所有情况下防止断开连接错误。

可以通过使用`DialectEvents.do_execute()`和`DialectEvents.do_execute_no_params()`钩子向 DBAPI 级别的 `cursor.execute()` 方法应用简单的重试机制，这些钩子将能够在语句执行期间拦截断开连接。对于那些不完全缓冲结果集的 DBAPI，它将**不会**拦截结果集获取操作期间的连接故障。该方案要求数据库支持 DBAPI 级别的自动提交，并且**不能保证**适用于特定的后端。提供了一个名为 `reconnecting_engine()` 的单个函数，它将事件钩子应用于给定的 `Engine` 对象，返回一个始终启用 DBAPI 级别自动提交的版本。连接将自动重新连接以用于单参数和无参数语句执行：

```py
import time

from sqlalchemy import event

def reconnecting_engine(engine, num_retries, retry_interval):
    def _run_with_retries(fn, context, cursor_obj, statement, *arg, **kw):
        for retry in range(num_retries + 1):
            try:
                fn(cursor_obj, statement, context=context, *arg)
            except engine.dialect.dbapi.Error as raw_dbapi_err:
                connection = context.root_connection
                if engine.dialect.is_disconnect(raw_dbapi_err, connection, cursor_obj):
                    if retry > num_retries:
                        raise
                    engine.logger.error(
                        "disconnection error, retrying operation",
                        exc_info=True,
                    )
                    connection.invalidate()

                    # use SQLAlchemy 2.0 API if available
                    if hasattr(connection, "rollback"):
                        connection.rollback()
                    else:
                        trans = connection.get_transaction()
                        if trans:
                            trans.rollback()

                    time.sleep(retry_interval)
                    context.cursor = cursor_obj = connection.connection.cursor()
                else:
                    raise
            else:
                return True

    e = engine.execution_options(isolation_level="AUTOCOMMIT")

    @event.listens_for(e, "do_execute_no_params")
    def do_execute_no_params(cursor_obj, statement, context):
        return _run_with_retries(
            context.dialect.do_execute_no_params, context, cursor_obj, statement
        )

    @event.listens_for(e, "do_execute")
    def do_execute(cursor_obj, statement, parameters, context):
        return _run_with_retries(
            context.dialect.do_execute, context, cursor_obj, statement, parameters
        )

    return e
```

根据上述方案，可以使用以下概念证明脚本演示事务中重新连接。运行一次后，它将每五秒向数据库发出一个`SELECT 1`语句：

```py
from sqlalchemy import create_engine
from sqlalchemy import select

if __name__ == "__main__":
    engine = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True)

    def do_a_thing(engine):
        with engine.begin() as conn:
            while True:
                print("ping: %s" % conn.execute(select([1])).scalar())
                time.sleep(5)

    e = reconnecting_engine(
        create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo_pool=True),
        num_retries=5,
        retry_interval=2,
    )

    do_a_thing(e)
```

在脚本运行时重新启动数据库以演示透明的重新连接操作：

```py
$ python reconnect_test.py
ping: 1
ping: 1
disconnection error, retrying operation
Traceback (most recent call last):
  ...
MySQLdb._exceptions.OperationalError: (2006, 'MySQL server has gone away')
2020-10-19 16:16:22,624 INFO sqlalchemy.pool.impl.QueuePool Invalidate connection <_mysql.connection open to 'localhost' at 0xf59240>
ping: 1
ping: 1
...
```

上述方案已经在 SQLAlchemy 1.4 上进行了测试。

## 为什么 SQLAlchemy 发出了那么多个 ROLLBACK？

SQLAlchemy 目前假设 DBAPI 连接处于“非自动提交”模式 - 这是 Python 数据库 API 的默认行为，这意味着必须假定事务始终在进行中。连接池在连接返回时发出 `connection.rollback()`。这是为了释放连接上仍然存在的任何事务资源。在像 PostgreSQL 或 MSSQL 这样的数据库上，表资源被积极地锁定，这一点至关重要，以确保行和表不会在不再使用的连接中保持锁定状态。否则，应用程序可能会挂起。然而，这不仅仅是为了锁定，并且在具有任何类型的事务隔离的任何数据库上同样关键，包括具有 InnoDB 的 MySQL。如果在隔离内在连接上已经查询了该数据，任何仍然处于旧事务中的连接将返回陈旧的数据。有关为什么即使在 MySQL 上也可能看到陈旧数据的背景，请参阅[`dev.mysql.com/doc/refman/5.1/en/innodb-transaction-model.html`](https://dev.mysql.com/doc/refman/5.1/en/innodb-transaction-model.html)

### 我使用的是 MyISAM - 如何关闭它？

连接池的连接返回行为的行为可以使用 `reset_on_return` 进行配置：

```py
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/myisam_database",
    pool=QueuePool(reset_on_return=False),
)
```

### 我使用的是 SQL Server - 如何将那些 ROLLBACKs 转换为 COMMITs？

`reset_on_return` 接受值 `commit`、`rollback`，除了 `True`、`False` 和 `None`。设置为 `commit` 将导致任何连接返回到池时进行 COMMIT：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mydsn", pool=QueuePool(reset_on_return="commit")
)
```

### 我正在使用 MyISAM - 如何关闭它？

可以使用 `reset_on_return` 配置连接池的连接返回行为：

```py
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/myisam_database",
    pool=QueuePool(reset_on_return=False),
)
```

### 我正在使用 SQL Server - 如何将那些 ROLLBACKs 转换为 COMMITs？

`reset_on_return` 接受值 `commit`、`rollback`，除了 `True`、`False` 和 `None`。设置为 `commit` 将导致任何连接返回到池时进行 COMMIT：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mydsn", pool=QueuePool(reset_on_return="commit")
)
```

## 我正在使用 SQLite 数据库的多个连接（通常用于测试事务操作），但我的测试程序不起作用！

如果使用 SQLite 的 `:memory:` 数据库，默认连接池是 `SingletonThreadPool`，它每个线程维护一个 SQLite 连接。因此，在同一线程中使用两个连接实际上是相同的 SQLite 连接。确保您不是使用 `:memory:` 数据库，以便引擎将使用 `QueuePool`（当前 SQLAlchemy 版本中非内存数据库的默认值）。

另请参阅

线程/池行为 - 有关 PySQLite 行为的信息。

## 在使用 Engine 时如何访问原始的 DBAPI 连接？

使用常规的 SA 引擎级 Connection，您可以通过 `Connection.connection` 属性获取到一个池代理版本的 DBAPI 连接，并且对于真正的 DBAPI 连接，您可以在此调用 `PoolProxiedConnection.dbapi_connection` 属性。在常规的同步驱动程序中，通常不需要访问非池代理的 DBAPI 连接，因为所有方法都是通过代理的：

```py
engine = create_engine(...)
conn = engine.connect()

# pep-249 style PoolProxiedConnection (historically called a "connection fairy")
connection_fairy = conn.connection

# typically to run statements one would get a cursor() from this
# object
cursor_obj = connection_fairy.cursor()
# ... work with cursor_obj

# to bypass "connection_fairy", such as to set attributes on the
# unproxied pep-249 DBAPI connection, use .dbapi_connection
raw_dbapi_connection = connection_fairy.dbapi_connection

# the same thing is available as .driver_connection (more on this
# in the next section)
also_raw_dbapi_connection = connection_fairy.driver_connection
```

在版本 1.4.24 中更改：添加了 `PoolProxiedConnection.dbapi_connection` 属性，它取代了以前的 `PoolProxiedConnection.connection` 属性，后者仍然可用；此属性始终提供 pep-249 同步风格的连接对象。还添加了 `PoolProxiedConnection.driver_connection` 属性，它将始终引用真正的驱动程序级连接，无论它呈现什么 API。

### 访问 asyncio 驱动程序的底层连接

在使用 asyncio 驱动程序时，上述方案有两个变化。首先是在使用`AsyncConnection`时，必须使用可等待方法`AsyncConnection.get_raw_connection()`来访问`PoolProxiedConnection`。在这种情况下返回的`PoolProxiedConnection`保留了同步风格的 pep-249 使用模式，而`PoolProxiedConnection.dbapi_connection`属性指的是一个将 asyncio 连接适配为同步风格 pep-249 API 的 SQLAlchemy 适配连接对象，换句话说，在使用 asyncio 驱动程序时存在*两层*代理。实际的 asyncio 连接可以从`driver_connection`属性中获取。将上述示例重新阐述为 asyncio 的形式如下：

```py
async def main():
    engine = create_async_engine(...)
    conn = await engine.connect()

    # pep-249 style ConnectionFairy connection pool proxy object
    # presents a sync interface
    connection_fairy = await conn.get_raw_connection()

    # beneath that proxy is a second proxy which adapts the
    # asyncio driver into a pep-249 connection object, accessible
    # via .dbapi_connection as is the same with a sync API
    sqla_sync_conn = connection_fairy.dbapi_connection

    # the really-real innermost driver connection is available
    # from the .driver_connection attribute
    raw_asyncio_connection = connection_fairy.driver_connection

    # work with raw asyncio connection
    result = await raw_asyncio_connection.execute(...)
```

自版本 1.4.24 起更改：添加了`PoolProxiedConnection.dbapi_connection`和`PoolProxiedConnection.driver_connection`属性，以允许通过一致的接口访问 pep-249 连接、pep-249 适配层和底层驱动程序连接。

在使用 asyncio 驱动程序时，上述“DBAPI”连接实际上是 SQLAlchemy 适配的连接形式，它呈现了同步风格的 pep-249 风格 API。要访问实际的 asyncio 驱动程序连接，可以通过`PoolProxiedConnection.driver_connection`属性来访问`PoolProxiedConnection`。对于标准的 pep-249 驱动程序，`PoolProxiedConnection.dbapi_connection`和`PoolProxiedConnection.driver_connection`是同义词。

在将连接返回到池之前，必须确保将连接上的任何隔离级别设置或其他操作特定设置恢复为正常状态。

作为恢复设置的替代方案，您可以在 `Connection` 或代理连接上调用 `Connection.detach()` 方法，这将使连接与池解除关联，从而在调用 `Connection.close()` 时关闭和丢弃它：

```py
conn = engine.connect()
conn.detach()  # detaches the DBAPI connection from the connection pool
conn.connection.<go nuts>
conn.close()  # connection is closed for real, the pool replaces it with a new connection
```

### 使用 asyncio 驱动程序访问底层连接

当使用 asyncio 驱动程序时，对上述方案有两个变化。首先是当使用 `AsyncConnection` 时，必须使用可等待方法 `AsyncConnection.get_raw_connection()` 访问 `PoolProxiedConnection`。在这种情况下返回的 `PoolProxiedConnection` 保留了同步样式 pep-249 使用模式，并且 `PoolProxiedConnection.dbapi_connection` 属性指向一个 SQLAlchemy 适配的连接对象，将 asyncio 连接适配为同步样式 pep-249 API，换句话说，当使用 asyncio 驱动程序时会有*两层*代理。实际的 asyncio 连接可以从 `driver_connection` 属性获得。在 asyncio 方面重新表述上一个示例如下：

```py
async def main():
    engine = create_async_engine(...)
    conn = await engine.connect()

    # pep-249 style ConnectionFairy connection pool proxy object
    # presents a sync interface
    connection_fairy = await conn.get_raw_connection()

    # beneath that proxy is a second proxy which adapts the
    # asyncio driver into a pep-249 connection object, accessible
    # via .dbapi_connection as is the same with a sync API
    sqla_sync_conn = connection_fairy.dbapi_connection

    # the really-real innermost driver connection is available
    # from the .driver_connection attribute
    raw_asyncio_connection = connection_fairy.driver_connection

    # work with raw asyncio connection
    result = await raw_asyncio_connection.execute(...)
```

从版本 1.4.24 开始更改：添加了 `PoolProxiedConnection.dbapi_connection` 和 `PoolProxiedConnection.driver_connection` 属性，以允许使用一致的接口访问 pep-249 连接、pep-249 适配层和底层驱动程序连接。

在使用 asyncio 驱动程序时，上述“DBAPI”连接实际上是一个经过 SQLAlchemy 适配的连接形式，它呈现了一个同步风格的 pep-249 风格 API。要访问实际的 asyncio 驱动程序连接，它将呈现所使用驱动程序的原始 asyncio API，可以通过`PoolProxiedConnection`的`PoolProxiedConnection.driver_connection`属性进行访问。对于标准的 pep-249 驱动程序，`PoolProxiedConnection.dbapi_connection` 和 `PoolProxiedConnection.driver_connection` 是同义词。

在将连接返回到池之前，您必须确保将任何隔离级别设置或其他特定操作设置恢复为正常状态。

作为恢复设置的替代方案，您可以在`Connection`或代理连接上调用`Connection.detach()`方法，这将使连接与池解除关联，从而在调用`Connection.close()`时关闭并丢弃连接：

```py
conn = engine.connect()
conn.detach()  # detaches the DBAPI connection from the connection pool
conn.connection.<go nuts>
conn.close()  # connection is closed for real, the pool replaces it with a new connection
```

## 我如何在 Python 多进程或 os.fork() 中使用引擎/连接/会话？

这在使用连接池与多进程或 os.fork()一节中有详细介绍。
