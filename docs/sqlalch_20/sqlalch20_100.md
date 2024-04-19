# 连接池

> 原文：[`docs.sqlalchemy.org/en/20/core/pooling.html`](https://docs.sqlalchemy.org/en/20/core/pooling.html)

连接池是一种标准技术，用于在内存中维护长时间运行的连接以进行有效重用，并为应用程序可能同时使用的连接总数提供管理。

特别是对于服务器端 Web 应用程序，连接池是在内存中维护一组活动数据库连接并在请求之间重用的标准方式。

SQLAlchemy 包含几种连接池实现，它们与`Engine`集成。它们也可以直接用于希望为其他普通 DBAPI 方法添加连接池的应用程序。

## 连接池配置

`create_engine()` 函数返回的 `Engine` 大多数情况下都已集成了一个 `QueuePool`，预先配置了合理的池默认值。如果你只是想学习如何启用连接池 - 恭喜！你已经完成了。

最常见的 `QueuePool` 调整参数可以直接作为关键字参数传递给 `create_engine()`：`pool_size`、`max_overflow`、`pool_recycle` 和 `pool_timeout`。例如：

```py
engine = create_engine(
    "postgresql+psycopg2://me@localhost/mydb", pool_size=20, max_overflow=0
)
```

所有 SQLAlchemy 连接池实现的共同点是它们都不会“预先创建”连接 - 所有实现都会等待首次使用之前才创建连接。在那时，如果没有额外的并发检出请求需要更多连接，就不会创建额外的连接。这就是为什么 `create_engine()` 默认使用大小为五的 `QueuePool` 是完全可以的，而不管应用程序是否真的需要排队五个连接 - 只有当应用程序实际上同时使用五个连接时，池才会增长到该大小，这种使用小池的行为是完全合适的默认行为。

注意

`QueuePool`类**不兼容 asyncio**。当使用`create_async_engine`创建`AsyncEngine`实例时，将使用`AsyncAdaptedQueuePool`类，该类使用与 asyncio 兼容的队列实现。

## 切换池实现

使用不同类型的池与`create_engine()`的通常方法是使用`poolclass`参数。此参数接受从`sqlalchemy.pool`模块导入的类，并为您处理构建池的详细信息。这里的一个常见用例是禁用连接池，可以通过使用`NullPool`实现来实现：

```py
from sqlalchemy.pool import NullPool

engine = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test", poolclass=NullPool
)
```

## 使用自定义连接函数

请参阅自定义 DBAPI connect()参数 / on-connect routines 一节，了解各种连接定制例程。

## 构建池

要单独使用`Pool`，则`creator`函数是唯一需要的参数，并首先传递，然后是任何其他选项：

```py
import sqlalchemy.pool as pool
import psycopg2

def getconn():
    c = psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")
    return c

mypool = pool.QueuePool(getconn, max_overflow=10, pool_size=5)
```

可以使用`Pool.connect()`函数从池中获取 DBAPI 连接。此方法的返回值是一个包含在透明代理中的 DBAPI 连接：

```py
# get a connection
conn = mypool.connect()

# use it
cursor_obj = conn.cursor()
cursor_obj.execute("select foo")
```

透明代理的目的是拦截`close()`调用，这样，DBAPI 连接不会关闭，而是返回到池中：

```py
# "close" the connection.  Returns
# it to the pool.
conn.close()
```

当代理被垃圾回收时，它还将其包含的 DBAPI 连接返回到池中，尽管在 Python 中并非确定性地立即发生这种情况（尽管在 cPython 中通常是这样）。然而，不建议使用此用法，特别是不支持与 asyncio DBAPI 驱动程序一起使用。

## 返回时重置

池包括“返回时重置”行为，当连接返回到池时，将调用 DBAPI 连接的`rollback()`方法。这样做是为了从连接中删除任何现有的事务状态，这不仅包括未提交的数据，还包括表和行锁。对于大多数 DBAPIs，调用`rollback()`是廉价的，如果 DBAPI 已经完成了一个事务，则该方法应该是无操作的。

### 禁用非事务连接的返回时重置

对于一些特定情况下`rollback()`不起作用的情况，例如使用配置为 autocommit 或使用没有 ACID 功能的数据库（如 MySQL 的 MyISAM 引擎）的连接时，可以禁用归还时重置行为，通常出于性能原因。可以通过使用`Pool.reset_on_return`参数来实现，该参数也可以从`create_engine()`中使用`create_engine.pool_reset_on_return`传递值为`None`来实现。下面的示例中演示了这一点，结合了`AUTOCOMMIT`的`create_engine.isolation_level`参数设置：

```py
non_acid_engine = create_engine(
    "mysql://scott:tiger@host/db",
    pool_reset_on_return=None,
    isolation_level="AUTOCOMMIT",
)
```

上述引擎在连接返回到池中时实际上不会执行回滚操作；由于启用了 AUTOCOMMIT，驱动程序也不会执行任何 BEGIN 操作。

### 自定义归还时重置方案

仅包含单个`rollback()`的“归还时重置”对于某些用例可能不足够；特别是，使用临时表的应用程序可能希望在连接归还时自动删除这些表。一些（但并非所有）后端包括可以在数据库连接范围内“重置”这些表的功能，这可能是连接池重置的理想行为。其他服务器资源，如准备好的语句句柄和服务器端语句缓存，可能会在归还过程之后持续存在，具体取决于具体情况是否希望这样。同样，一些（但再次并非所有）后端可能提供一种重置此状态的方法。已知具有此类重置方案的两个 SQLAlchemy 包含的方言包括 Microsoft SQL Server，其中通常使用一个名为`sp_reset_connection`的未记录但广为人知的存储过程，以及 PostgreSQL，后者具有一系列良好记录的命令，包括`DISCARD`、`RESET`、`DEALLOCATE`和`UNLISTEN`。

以下示例说明了如何使用 `PoolEvents.reset()` 事件钩子将返回时的重置替换为 Microsoft SQL Server 的 `sp_reset_connection` 存储过程。`create_engine.pool_reset_on_return` 参数设置为 `None`，以便自定义方案完全替换默认行为。自定义钩子实现在任何情况下调用 `.rollback()`，因为通常重要的是 DBAPI 自己的提交/回滚跟事务状态保持一致：

```py
from sqlalchemy import create_engine
from sqlalchemy import event

mssql_engine = create_engine(
    "mssql+pyodbc://scott:tiger⁵HHH@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
    # disable default reset-on-return scheme
    pool_reset_on_return=None,
)

@event.listens_for(mssql_engine, "reset")
def _reset_mssql(dbapi_connection, connection_record, reset_state):
    if not reset_state.terminate_only:
        dbapi_connection.execute("{call sys.sp_reset_connection}")

    # so that the DBAPI itself knows that the connection has been
    # reset
    dbapi_connection.rollback()
```

自版本 2.0.0b3 起进行了更改：在 `PoolEvents.reset()` 事件中添加了额外的状态参数，并且确保该事件在所有“重置”发生时都被调用，以便作为自定义“重置”处理程序的适当位置。之前使用 `PoolEvents.checkin()` 处理程序的方案仍然可用。

另请参阅

+   用于连接池的临时表/资源重置 - 在 Microsoft SQL Server 文档中

+   用于连接池的临时表/资源重置 - 在 PostgreSQL 文档中

### 记录返回时重置事件

记录池事件，包括返回时重置，可以将其设置为 `logging.DEBUG` 日志级别以及 `sqlalchemy.pool` 记录器，或者在使用 `create_engine()` 时通过将 `create_engine.echo_pool` 设置为 `"debug"` 来设置：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine("postgresql://scott:tiger@localhost/test", echo_pool="debug")
```

上述池将显示详细的日志，包括返回时的重置：

```py
>>> c1 = engine.connect()
DEBUG sqlalchemy.pool.impl.QueuePool Created new connection <connection object ...>
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> checked out from pool
>>> c1.close()
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> being returned to pool
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> rollback-on-return
```

## 池事件

连接池支持事件接口，允许在第一次连接、每次新连接、以及连接的签出和签入时执行钩子。详情请参阅 `PoolEvents`。

## 处理断开连接

连接池具有刷新单个连接以及其整套连接的能力，将先前池化的连接设置为“无效”。常见用例是在数据库服务器重新启动时允许连接池优雅地恢复，并且所有先前建立的连接都不再可用。有两种方法可以做到这一点。

### 断开连接处理 - 悲观

悲观方法是指在每次连接池检出时发出 SQL 连接上的测试语句，以测试数据库连接是否仍然可行。该实现是方言特定的，并且利用特定于 DBAPI 的 ping 方法，或者使用简单的 SQL 语句如“SELECT 1”，以便测试连接的活动性。

该方法会在连接检出过程中增加一小部分额外开销，但除此之外，它是完全消除因连接池中的过期连接而导致数据库错误的最简单和可靠的方法。调用应用程序无需担心组织操作以从池中恢复过期连接。

可以通过使用`Pool.pre_ping`参数来实现对连接的悲观检测，该参数可通过`create_engine()`的`create_engine.pool_pre_ping`参数获得：

```py
engine = create_engine("mysql+pymysql://user:pw@host/db", pool_pre_ping=True)
```

“预 ping”功能根据每个方言的基础，通过调用特定于 DBAPI 的“ping”方法，或者如果不可用，则发出与“SELECT 1”等效的 SQL，捕获任何错误并将错误检测为“断开”情况。如果 ping/错误检查确定连接不可用，则连接将立即被重新使用，并且所有比当前时间更早的其他池连接都将无效，以便下次检出时它们也将在使用前被重新使用。

如果数据库在“预 ping”运行时仍然不可用，则初始连接将失败，并且无法连接的错误将正常传播。在数据库可用于连接但无法响应“ping”的情况下，将在放弃之前尝试最多三次“pre_ping”，并传播最后收到的数据库错误。

需要特别注意的是，预检测方法**不适用于事务中断开连接或其他 SQL 操作**的情况。如果数据库在事务进行中变得不可用，则事务将丢失并引发数据库错误。虽然`Connection`对象会检测到“断开连接”情况并重新使用连接以及在此情况发生时使其余连接池失效，但引发异常的单个操作将丢失，并且由应用程序来放弃操作或重新尝试整个事务。如果引擎使用 DBAPI 级别的自动提交连接配置，如设置事务隔离级别，包括 DBAPI 自动提交，则可能会使用事件在操作中透明地重新连接。有关示例，请参阅如何“自动重试”语句执行？。

对于使用“SELECT 1”并捕获错误以检测断开连接的方言，可以使用`DialectEvents.handle_error()`钩子为新的后端特定错误消息增加断开连接测试。

#### 自定义 / 传统悲观 Ping

在`create_engine.pool_pre_ping`添加之前，历史上一直使用`ConnectionEvents.engine_connect()`引擎事件手动执行“预检测”方法。下面是最常见的方法，供参考，以防应用程序已经使用此方法，或者需要特殊行为：

```py
from sqlalchemy import exc
from sqlalchemy import event
from sqlalchemy import select

some_engine = create_engine(...)

@event.listens_for(some_engine, "engine_connect")
def ping_connection(connection, branch):
    if branch:
        # this parameter is always False as of SQLAlchemy 2.0,
        # but is still accepted by the event hook.  In 1.x versions
        # of SQLAlchemy, "branched" connections should be skipped.
        return

    try:
        # run a SELECT 1\.   use a core select() so that
        # the SELECT of a scalar value without a table is
        # appropriately formatted for the backend
        connection.scalar(select(1))
    except exc.DBAPIError as err:
        # catch SQLAlchemy's DBAPIError, which is a wrapper
        # for the DBAPI's exception.  It includes a .connection_invalidated
        # attribute which specifies if this connection is a "disconnect"
        # condition, which is based on inspection of the original exception
        # by the dialect in use.
        if err.connection_invalidated:
            # run the same SELECT again - the connection will re-validate
            # itself and establish a new connection.  The disconnect detection
            # here also causes the whole connection pool to be invalidated
            # so that all stale connections are discarded.
            connection.scalar(select(1))
        else:
            raise
```

以上方法的优点在于，我们利用了 SQLAlchemy 检测那些已知指示“断开连接”情况的 DBAPI 异常的设施，以及`Engine`对象在此情况发生时正确使当前连接池失效并允许当前`Connection`重新验证到新的 DBAPI 连接。

### 断开连接处理 - 乐观

当不采用悲观处理时，以及当数据库在事务中使用连接期间关闭和/或重新启动时，处理陈旧/关闭连接的另一种方法是让 SQLAlchemy 在发生断开连接时处理它们，在这时，池中的所有连接都被标记为无效，这意味着它们被认为是陈旧的，并将在下次检出时刷新。此行为假定`Pool`与`Engine`一起使用。`Engine`具有可以检测到断开连接事件并自动刷新池的逻辑。

当`Connection`尝试使用 DBAPI 连接，并且引发与“断开连接”事件相对应的异常时，连接将被标记为无效。然后，`Connection`调用`Pool.recreate()`方法，有效地使所有当前未检出的连接无效，以便在下次检出时用新连接替换它们。下面的代码示例说明了这个流程：

```py
from sqlalchemy import create_engine, exc

e = create_engine(...)
c = e.connect()

try:
    # suppose the database has been restarted.
    c.execute(text("SELECT * FROM table"))
    c.close()
except exc.DBAPIError as e:
    # an exception is raised, Connection is invalidated.
    if e.connection_invalidated:
        print("Connection was invalidated!")

# after the invalidate event, a new connection
# starts with a new Pool
c = e.connect()
c.execute(text("SELECT * FROM table"))
```

上面的示例说明，在检测到断开连接事件后，无需任何特殊干预即可刷新池，池会继续正常运行。但是，对于每个在数据库不可用事件发生时处于使用状态的连接，都会引发一个数据库异常。在使用 ORM 会话的典型 Web 应用程序中，上述条件将对应于请求失败并出现 500 错误，然后 Web 应用程序在那之后正常继续。因此，该方法是“乐观”的，因为不会预期频繁的数据库重启。

#### 设置池回收

可以增强“乐观”方法的附加设置是设置池回收参数。此参数防止池使用已经过一定时期的特定连接，并且适用于自动在一段时间后关闭失效连接的数据库后端，例如 MySQL：

```py
from sqlalchemy import create_engine

e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", pool_recycle=3600)
```

以上，任何已打开超过一小时的 DBAPI 连接将在下次检出时被标记为无效并替换。请注意，这种无效化**仅**发生在检出时 - 不会发生在任何处于已检出状态的连接上。`pool_recycle`是`Pool`本身的一个函数，独立于是否正在使用`Engine`。### 更多关于无效化的内容

`Pool`提供了“连接失效”服务，允许显式无效连接以及响应确定使连接无法使用的条件自动无效连接。

“失效”意味着特定的 DBAPI 连接从池中移除并丢弃。如果不清楚连接本身是否已关闭，则会调用此连接的`.close()`方法，但是如果此方法失败，则会记录异常但操作仍将继续。

当使用`Engine`时，`Connection.invalidate()`方法是显式无效的通常入口点。导致 DBAPI 连接失效的其他条件包括：

+   当调用诸如`connection.execute()`之类的方法时引发 DBAPI 异常，比如`OperationalError`，则被检测为所谓的“断开连接”条件。由于 Python DBAPI 没有提供用于确定异常性质的标准系统，因此所有的 SQLAlchemy 方言都包括一个名为`is_disconnect()`的系统，该系统将检查异常对象的内容，包括字符串消息和其中包含的任何潜在错误代码，以确定此异常是否表明连接不再可用。如果是这种情况，则调用`_ConnectionFairy.invalidate()`方法，然后丢弃 DBAPI 连接。

+   当连接返回到池中，并且调用连接的`connection.rollback()`或`connection.commit()`方法，根据池的“重置返回”行为，抛出异常。将尝试最终调用`.close()`关闭连接，然后丢弃它。

+   当实现`PoolEvents.checkout()`的监听器引发`DisconnectionError`异常时，表示连接无法使用，需要进行新的连接尝试。

所有发生的失效都将调用`PoolEvents.invalidate()`事件。  ### 支持断开连接情况的新数据库错误代码

SQLAlchemy 方言每个都包含一个名为 `is_disconnect()` 的例程，当遇到 DBAPI 异常时会调用它。DBAPI 异常对象被传递到这个方法，在那里方言特定的启发法则将确定接收到的错误代码是否表明数据库连接已被“断开”，或者处于其他不可用状态，这表明它应该被回收利用。在这里应用的启发法则可以使用 `DialectEvents.handle_error()` 事件钩子进行定制，该事件钩子通常通过所属的 `Engine` 对象建立。使用这个钩子，发生的所有错误都将传递一个称为 `ExceptionContext` 的上下文对象。自定义事件钩子可以控制是否应该将特定错误视为“断开”情况，以及是否应该导致整个连接池无效。

例如，为了添加支持将 Oracle 错误代码 `DPY-1001` 和 `DPY-4011` 视为断开代码进行处理，可以在创建之后向引擎应用一个事件处理程序：

```py
import re

from sqlalchemy import create_engine

engine = create_engine("oracle://scott:tiger@dnsname")

@event.listens_for(engine, "handle_error")
def handle_exception(context: ExceptionContext) -> None:
    if not context.is_disconnect and re.match(
        r"^(?:DPI-1001|DPI-4011)", str(context.original_exception)
    ):
        context.is_disconnect = True

    return None
```

上述错误处理函数将为所有 Oracle 错误被引发时调用，包括那些在使用 池预 ping 功能时捕获的错误，用于依赖于断开错误处理的后端（在 2.0 中新增）。

另请参见

`DialectEvents.handle_error()`  ## 使用 FIFO vs. LIFO

`QueuePool` 类包含一个名为 `QueuePool.use_lifo` 的标志，该标志也可以通过 `create_engine()` 中的标志 `create_engine.pool_use_lifo` 进行访问。将此标志设置为 `True` 会导致池的“队列”行为变为“堆栈”行为，例如，返回到池的最后一个连接将在下一次请求时首先使用。与池的先入先出长期行为相反，即产生池中每个连接的循环效果，LIFO 模式允许多余的连接在池中保持空闲，从而允许服务器端超时方案关闭这些连接。FIFO 和 LIFO 之间的区别基本上是池是否在空闲期间保持完整的连接集：

```py
engine = create_engine("postgreql://", pool_use_lifo=True, pool_pre_ping=True)
```

上面，我们还使用 `create_engine.pool_pre_ping` 标志，以便服务器端关闭的连接能够被连接池优雅地处理，并替换为新连接。

注意该标志仅适用于 `QueuePool` 使用。

在版本 1.3 中新增。

另请参阅

处理断开连接  ## 使用连接池与多进程或 os.fork()

当使用连接池时，以及当使用通过 `create_engine()` 创建的 `Engine` 时，至关重要的是，池化的连接**不会共享到一个分叉的进程**。TCP 连接被表示为文件描述符，通常跨越进程边界工作，这意味着这将导致两个或更多完全独立的 Python 解释器状态代表的文件描述符被并发访问。

根据驱动程序和操作系统的具体情况，此处出现的问题范围从无法工作的连接到被多个进程同时使用的套接字连接，导致消息传递中断（后一种情况通常最常见）。

SQLAlchemy `Engine` 对象指的是一组现有数据库连接的连接池。因此，当这个对象被复制到子进程时，目标是确保没有数据库连接被传递过去。有四种常用的方法：

1.  使用 `NullPool` 禁用连接池。这是最简单的、一次性系统，防止 `Engine` 多次使用任何连接：

    ```py
    from sqlalchemy.pool import NullPool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname", poolclass=NullPool)
    ```

1.  在子进程的初始化阶段，对任何给定的 `Engine` 调用 `Engine.dispose()`，传递 `Engine.dispose.close` 参数值为 `False`。这样新进程就不会触及父进程的任何连接，而是开始使用新连接。**这是推荐的方法**：

    ```py
    from multiprocessing import Pool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname")

    def run_in_process(some_data_record):
        with engine.connect() as conn:
            conn.execute(text("..."))

    def initializer():
      """ensure the parent proc's database connections are not touched
     in the new connection pool"""
        engine.dispose(close=False)

    with Pool(10, initializer=initializer) as p:
        p.map(run_in_process, data)
    ```

    在版本 1.4.33 中新增：添加了 `Engine.dispose.close` 参数，允许在子进程中替换连接池而不会干扰父进程使用的连接。

1.  在创建子进程之前直接调用`Engine.dispose()`。这也将导致子进程以新的连接池启动，同时确保父连接不会传递给子进程：

    ```py
    engine = create_engine("mysql://user:pass@host/dbname")

    def run_in_process():
        with engine.connect() as conn:
            conn.execute(text("..."))

    # before process starts, ensure engine.dispose() is called
    engine.dispose()
    p = Process(target=run_in_process)
    p.start()
    ```

1.  可以应用于连接池的事件处理程序来测试跨进程边界共享的连接，并使其失效。

    ```py
    from sqlalchemy import event
    from sqlalchemy import exc
    import os

    engine = create_engine("...")

    @event.listens_for(engine, "connect")
    def connect(dbapi_connection, connection_record):
        connection_record.info["pid"] = os.getpid()

    @event.listens_for(engine, "checkout")
    def checkout(dbapi_connection, connection_record, connection_proxy):
        pid = os.getpid()
        if connection_record.info["pid"] != pid:
            connection_record.dbapi_connection = connection_proxy.dbapi_connection = None
            raise exc.DisconnectionError(
                "Connection record belongs to pid %s, "
                "attempting to check out in pid %s" % (connection_record.info["pid"], pid)
            )
    ```

    在上述示例中，我们使用了类似于 Disconnect Handling - Pessimistic 中描述的方法来处理在不同父进程中起源的 DBAPI 连接，将其视为“无效”连接，迫使池回收连接记录以建立新连接。

上述策略将适应共享在进程之间的`Engine`的情况。但仅凭上述步骤尚不足以处理跨进程边界共享特定`Connection`的情况；最好将特定`Connection`的范围保持在单个进程（和线程）内。此外，不支持直接跨进程边界共享任何正在进行的事务状态，例如已开始事务并引用活动`Connection`实例的 ORM `Session`对象；同样，最好在新进程中创建新的`Session`对象。

## 直接使用池实例

可以直接使用池实现而不需要引擎。这可用于只希望使用池行为而不需要所有其他 SQLAlchemy 功能的应用程序。在下面的示例中，使用`create_pool_from_url()`获取`MySQLdb`方言的默认池：

```py
from sqlalchemy import create_pool_from_url

my_pool = create_pool_from_url(
    "mysql+mysqldb://", max_overflow=5, pool_size=5, pre_ping=True
)

con = my_pool.connect()
# use the connection
...
# then close it
con.close()
```

如果未指定要创建的池的类型，则将使用方言的默认池。要直接指定它，可以使用`poolclass`参数，就像以下示例中一样：

```py
from sqlalchemy import create_pool_from_url
from sqlalchemy import NullPool

my_pool = create_pool_from_url("mysql+mysqldb://", poolclass=NullPool)
```

## API 文档 - 可用的池实现

| 对象名称 | 描述 |
| --- | --- |
| _ConnectionFairy | 代理一个 DBAPI 连接并提供对解除引用的支持。 |
| _ConnectionRecord | 维护连接池中的位置，引用一个池化连接。 |
| AssertionPool | 允许同时最多检出一个连接的`Pool`。 |
| AsyncAdaptedQueuePool | `QueuePool`的一个与 asyncio 兼容的版本。 |
| ConnectionPoolEntry | 代表`Pool`实例上的单个数据库连接的对象的接口。 |
| ManagesConnection | 两个连接管理接口`PoolProxiedConnection`和`ConnectionPoolEntry`的通用基类。 |
| NullPool | 不池化连接的连接池。 |
| Pool | 连接池的抽象基类。 |
| PoolProxiedConnection | 一个用于[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 连接的类似连接适配器，包括特定于`Pool`实现的附加方法。 |
| QueuePool | 对打开连接数量施加限制的`Pool`。 |
| SingletonThreadPool | 一个每个线程维护一个连接的连接池。 |
| StaticPool | 一个连接池，用于所有请求的一个连接。 |

```py
class sqlalchemy.pool.Pool
```

连接池的抽象基类。

**成员**

__init__(), connect(), dispose(), recreate()

**类签名**

类`sqlalchemy.pool.Pool` (`sqlalchemy.log.Identified`, `sqlalchemy.event.registry.EventTarget`)

```py
method __init__(creator: _CreatorFnType | _CreatorWRecFnType, recycle: int = -1, echo: log._EchoFlagType = None, logging_name: str | None = None, reset_on_return: _ResetStyleArgType = True, events: List[Tuple[_ListenerFnType, str]] | None = None, dialect: _ConnDialect | Dialect | None = None, pre_ping: bool = False, _dispatch: _DispatchCommon[Pool] | None = None)
```

构造一个连接池。

参数：

+   `creator` – 一个可调用的函数，返回一个 DB-API 连接对象。该函数将使用参数调用。

+   `recycle` – 如果设置为除-1 之外的值，连接回收之间的秒数，这意味着在签出时，如果超过此超时，则连接将被关闭并替换为新打开的连接。默认为-1。

+   `logging_name` – 将在“sqlalchemy.pool”记录器中生成的日志记录的“name”字段中使用的字符串标识符。默认为对象的 id 的十六进制字符串。

+   `echo` –

    如果为 True，连接池将记录信息输出，例如当连接无效时以及当连接被回收到默认日志处理程序时，该处理程序默认为`sys.stdout`输出。如果设置为字符串`"debug"`，日志将包括池的签出和签入。

    `Pool.echo`参数也可以通过在`create_engine()`调用中使用`create_engine.echo_pool`参数进行设置。

    另请参阅

    配置日志记录 - 关于如何配置日志记录的更多详细信息。

+   `reset_on_return` –

    确定在连接被返回到池中时需要采取的步骤，这些步骤不会被`Connection`以外的方式处理。可通过`create_engine()`中的`create_engine.pool_reset_on_return`参数获得。

    `Pool.reset_on_return`可以具有以下任一值：

    +   `"rollback"` - 在连接上调用 rollback()，以释放锁定和事务资源。这是默认值。绝大多数用例应该保持此值设置。

    +   `"commit"` - 在连接上调用 commit()，以释放锁定和事务资源。如果发出了 commit，这里可能对缓存查询计划的数据库（如 Microsoft SQL Server）是有利的。但是，这个值比‘rollback’更危险，因为事务上的任何数据更改都会无条件提交。

    +   `None` - 在连接上不执行任何操作。如果数据库/DBAPI 始终以纯“自动提交”模式工作，或者如果使用`PoolEvents.reset()`事件处理程序建立了自定义重置处理程序，则此设置可能是合适的。

    +   `True` - 与‘rollback’相同，这是为了向后兼容而存在的。

    +   `False` - 与 None 相同，这是为了向后兼容而存在的。

    要进一步定制重置操作，可以使用`PoolEvents.reset()`事件钩子，该钩子可以在重置时执行任何所需的连接活动。

    另请参阅

    重置操作

    `PoolEvents.reset()`

+   `events` – 一个 2 元组列表，每个元组的形式为`(callable, target)`，将在构造时传递给`listen()`。提供此处是为了在应用方言级别的监听器之前，可以通过`create_engine()`分配事件监听器。

+   `dialect` – 一个将负责在 DBAPI 连接上调用 rollback()、close()或 commit()的`Dialect`。如果省略，则使用内置的“存根”方言。使用`create_engine()`的应用程序不应使用此参数，因为它由引擎创建策略处理。

+   `pre_ping` –

    如果为 True，池将在检出连接时发出“ping”（通常为“SELECT 1”，但是与方言有关）来测试连接是否存活。如果没有，连接将被透明地重新连接，并在成功后，此时间戳之前建立的所有其他池化连接将无效。需要传递方言以解释断开连接错误。

    从 1.2 版本开始新增。

```py
method connect() → PoolProxiedConnection
```

从池中返回一个 DBAPI 连接。

连接被仪器化，这样当调用其`close()`方法时，连接将会返回到池中。

```py
method dispose() → None
```

处置此池。

这种方法会使得已经检出的连接保持打开状态，因为它只影响处于池中空闲的连接。

另请参阅

`Pool.recreate()`

```py
method recreate() → Pool
```

返回一个新的与此相同类的`Pool`，并配置相同的创建参数。

此方法与`dispose()`结合使用，以关闭整个`Pool`并在其位置创建一个新的。

```py
class sqlalchemy.pool.QueuePool
```

对打开连接数量施加限制的`Pool`。

`QueuePool` 是除了带有`:memory:`数据库的 SQLite 外，所有`Engine`对象的默认池化实现。

`QueuePool` 类**不兼容**于 asyncio 和`create_async_engine()`。当使用`create_async_engine()`时，如果没有指定其他类型的池，将自动使用`AsyncAdaptedQueuePool`类。

另请参阅

`AsyncAdaptedQueuePool`

**成员**

__init__(), dispose(), recreate()

**类签名**

类 `sqlalchemy.pool.QueuePool` (`sqlalchemy.pool.base.Pool`)

```py
method __init__(creator: _CreatorFnType | _CreatorWRecFnType, pool_size: int = 5, max_overflow: int = 10, timeout: float = 30.0, use_lifo: bool = False, **kw: Any)
```

构建一个 QueuePool。

参数：

+   `creator` – 一个可调用函数，返回一个与`Pool.creator`相同的 DB-API 连接对象。

+   `pool_size` – 要维护的池的大小，默认为 5。这是池中将持续保留的连接数的最大值。注意，池开始时没有连接；一旦请求了这么多连接，这么多连接就会保留下来。`pool_size` 可以设置为 0 表示没有大小限制；要禁用池，请使用 `NullPool`。

+   `max_overflow` – 池的最大溢出大小。当检出的连接数量达到池大小设置的大小时，将返回额外的连接，直到达到此限制为止。当这些额外的连接返回到池时，它们将被断开并丢弃。因此，池允许的同时连接总数为 pool_size + max_overflow，池允许的“休眠”连接总数为 pool_size。max_overflow 可以设置为 -1 表示没有溢出限制；并发连接的总数不受限制。默认为 10。

+   `timeout` – 在放弃返回连接之前等待的秒数。默认为 30.0。这可以是一个浮点数，但受 Python 时间函数的限制，可能不可靠达到十毫秒的级别。

+   `use_lifo` –

    使用 LIFO（后进先出）而不是 FIFO（先进先出）来检索连接。使用 LIFO，服务器端的超时方案可以减少在非高峰使用期间使用的连接数。在计划服务器端超时时，请确保使用了重新循环或预先 ping 策略以优雅地处理过时的连接。

    版本 1.3 中的新功能。

    另请参见

    使用 FIFO vs. LIFO

    处理断开连接

+   `**kw` – 其他关键字参数，包括`Pool.recycle`、`Pool.echo`、`Pool.reset_on_return` 和其他参数，都将传递给 `Pool` 构造函数。

```py
method dispose() → None
```

处置此池。

此方法使得可能存在检出的连接保持打开状态，因为它只影响池中空闲的连接。

亦参见

`Pool.recreate()`

```py
method recreate() → QueuePool
```

返回一个新的`Pool`，与当前的池具有相同的类，并配置了相同的创建参数。

此方法与 `dispose()` 结合使用，关闭整个 `Pool` 并在其位置创建一个新的。

```py
class sqlalchemy.pool.AsyncAdaptedQueuePool
```

`QueuePool` 的 asyncio 兼容版本。

当从 `create_async_engine()` 生成的 `AsyncEngine` 引擎时，默认使用此池。它使用不使用 `threading.Lock` 的 asyncio 兼容队列实现。

`AsyncAdaptedQueuePool` 的参数和操作与 `QueuePool` 相同。

**类签名**

类 `sqlalchemy.pool.AsyncAdaptedQueuePool` (`sqlalchemy.pool.impl.QueuePool`)

```py
class sqlalchemy.pool.SingletonThreadPool
```

每个线程维护一个连接的池。

每个线程维护一个连接，从不将连接移动到创建它的线程之外的线程中。

警告

`SingletonThreadPool` 将在存在超过 `pool_size` 设置的任意连接时调用 `.close()`，例如，如果使用的唯一 **线程标识** 多于 `pool_size` 指定的数量。此清理是非确定性的，并且不受连接是否正在使用与线程标识相关联的影响。

在未来的版本中，`SingletonThreadPool` 可能会改进，但在目前的状态下，通常仅用于使用 SQLite `:memory:` 数据库的测试场景，并不建议用于生产环境。

`SingletonThreadPool` 类**不兼容** asyncio 和 `create_async_engine()`。

选项与 `Pool` 相同，还包括：

参数：

**pool_size** – 一次性维护连接的线程数。默认为五。

当使用基于内存的数据库时，SQLite 方言会自动使用 `SingletonThreadPool`。参见 SQLite。

**成员**

connect(), dispose(), recreate()

**类签名**

类 `sqlalchemy.pool.SingletonThreadPool` (`sqlalchemy.pool.base.Pool`)

```py
method connect() → PoolProxiedConnection
```

从池中返回一个 DBAPI 连接。

连接被仪器化，以便在调用其 `close()` 方法时，连接将返回到池中。

```py
method dispose() → None
```

处理此池。

```py
method recreate() → SingletonThreadPool
```

返回一个新的 `Pool`，与此相同类的并配置有相同的创建参数。

该方法与 `dispose()` 结合使用，关闭整个 `Pool` 并在其位置创建一个新的。

```py
class sqlalchemy.pool.AssertionPool
```

允许在任何给定时间最多只有一个已签出连接的 `Pool`。

如果同时签出了多个连接，则会引发异常。 用于调试使用比预期更多的连接的代码。

`AssertionPool` 类 **与** asyncio 兼容，`create_async_engine()`。

**成员**

dispose(), recreate()

**类签名**

类 `sqlalchemy.pool.AssertionPool` (`sqlalchemy.pool.base.Pool`)

```py
method dispose() → None
```

处理此池。

此方法使得可能保持已签出连接处于打开状态，因为它仅影响池中处于空闲状态的连接。

另请参阅

`Pool.recreate()`

```py
method recreate() → AssertionPool
```

返回一个新的 `Pool`，与此相同类的并配置有相同的创建参数。

该方法与 `dispose()` 结合使用，关闭整个 `Pool` 并在其位置创建一个新的。

```py
class sqlalchemy.pool.NullPool
```

不池化连接的 Pool。

相反，它会为每个连接的打开/关闭字面上打开并关闭底层的 DB-API 连接。

此 Pool 实现不支持与重新连接相关的函数，如 `recycle` 和连接失效，因为没有持续保留连接。

`NullPool`类与 asyncio 和`create_async_engine()`兼容。

**成员**

dispose(), recreate()

**类签名**

类`sqlalchemy.pool.NullPool`（`sqlalchemy.pool.base.Pool`）

```py
method dispose() → None
```

处置此池。

此方法使得可能存在检出的连接仍然保持打开状态，因为它只影响池中处于空闲状态的连接。

另请参阅

`Pool.recreate()`

```py
method recreate() → NullPool
```

返回一个新的`Pool`，与此相同类别的，并配置有相同的创建参数。

此方法与`dispose()`一起使用，以关闭整个`Pool`并创建一个新的。

```py
class sqlalchemy.pool.StaticPool
```

一种仅包含一个连接的池，用于所有请求。

重新连接相关的函数，如`recycle`和连接失效（也用于支持自动重新连接），目前只支持部分，并且可能不会产生良好的结果。

`StaticPool`类与 asyncio 和`create_async_engine()`兼容。

**成员**

dispose(), recreate()

**类签名**

类`sqlalchemy.pool.StaticPool`（`sqlalchemy.pool.base.Pool`）

```py
method dispose() → None
```

处置此池。

此方法使得可能存在检出的连接仍然保持打开状态，因为它只影响池中处于空闲状态的连接。

另请参阅

`Pool.recreate()`

```py
method recreate() → StaticPool
```

返回一个新的`Pool`，与此相同类别的，并配置有相同的创建参数。

此方法与`dispose()`一起使用，以关闭整个`Pool`并创建一个新的。

```py
class sqlalchemy.pool.ManagesConnection
```

两个连接管理接口`PoolProxiedConnection`和`ConnectionPoolEntry`的通用基类。

这两个对象通常通过连接池事件钩子在公共 API 中公开，文档位于`PoolEvents`。

**成员**

dbapi_connection, driver_connection, info, invalidate(), record_info

版本 2.0 中新增。

```py
attribute dbapi_connection: DBAPIConnection | None
```

被跟踪的实际 DBAPI 连接的引用。

这是一个符合[**PEP 249**](https://peps.python.org/pep-0249/)的对象，对于传统的同步式方言，由正在使用的第三方 DBAPI 实现提供。对于 asyncio 方言，实现通常是由 SQLAlchemy 方言本身提供的适配器对象；底层 asyncio 对象可通过`ManagesConnection.driver_connection`属性获得。

SQLAlchemy 对 DBAPI 连接的接口基于`DBAPIConnection`协议对象。

另请参阅

`ManagesConnection.driver_connection`

在使用 Engine 时如何获取原始的 DBAPI 连接？

```py
attribute driver_connection: Any | None
```

“驱动级别”连接对象由 Python DBAPI 或数据库驱动程序使用。

对于传统的[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 实现，此对象将与`ManagesConnection.dbapi_connection`的对象相同。对于 asyncio 数据库驱动程序，这将是该驱动程序使用的最终“连接”对象，例如不具有标准 pep-249 方法的`asyncpg.Connection`对象。

版本 1.4.24 中新增。

另请参阅

`ManagesConnection.dbapi_connection`

在使用 Engine 时如何获取原始的 DBAPI 连接？

```py
attribute info
```

与此`ManagesConnection`实例引用的底层 DBAPI 连接相关联的信息字典，允许将用户定义的数据与连接关联。

此字典中的数据在 DBAPI 连接本身的生命周期内是持久的，包括池中的检入和检出。当连接无效并被新连接替换时，此字典将被清除。

对于未与`ConnectionPoolEntry`关联的`PoolProxiedConnection`实例，例如如果它被分离，该属性返回一个仅限于该`ConnectionPoolEntry`的字典。因此，`ManagesConnection.info`属性将始终提供一个 Python 字典。

另请参见

`ManagesConnection.record_info`

```py
method invalidate(e: BaseException | None = None, soft: bool = False) → None
```

将托管连接标记为失效。

参数：

+   `e` – 表示连接失效原因的异常对象。

+   `soft` – 如果为 True，则连接不会关闭；相反，此连接将在下次检出时被回收。

另请参见

更多关于失效的信息

```py
attribute record_info
```

与此`ManagesConnection`关联的持久信息字典。

与`ManagesConnection.info`字典不同，此字典的生命周期与拥有它的`ConnectionPoolEntry`相同；因此，此字典将在重新连接和特定连接池条目的连接失效时持续存在。

对于未与`ConnectionPoolEntry`关联的`PoolProxiedConnection`实例，例如如果它被分离，该属性返回 None。与永远不为 None 的`ManagesConnection.info`字典形成对比。

另请参见

`ManagesConnection.info`

```py
class sqlalchemy.pool.ConnectionPoolEntry
```

代表`Pool`实例维护单个数据库连接的对象的接口。

`ConnectionPoolEntry` 对象表示池中特定连接的长期维护，包括使该连接过期或失效以被替换为新连接，而这个新连接将继续由相同的 `ConnectionPoolEntry` 实例维护。与 `PoolProxiedConnection` 相比，后者是短期的，每次检出的连接管理器，这个对象的生命周期为连接池中的特定“槽”。

`ConnectionPoolEntry` 对象在被传递到连接池事件钩子（例如 `PoolEvents.connect()` 和 `PoolEvents.checkout()`）时，主要对外公开。

新版本 2.0 中：`ConnectionPoolEntry` 为 `_ConnectionRecord` 内部类提供了公共界面。

**成员**

close(), dbapi_connection, driver_connection, in_use, info, invalidate(), record_info

**类签名**

class `sqlalchemy.pool.ConnectionPoolEntry` (`sqlalchemy.pool.base.ManagesConnection`)

```py
method close() → None
```

关闭由此连接池条目管理的 DBAPI 连接。

```py
attribute dbapi_connection: DBAPIConnection | None
```

跟踪的实际 DBAPI 连接的引用。

这是一个符合[**PEP 249**](https://peps.python.org/pep-0249/)标准的对象，对于传统的同步式方言，由正在使用的第三方 DBAPI 实现提供。对于 asyncio 方言，该实现通常是由 SQLAlchemy 方言本身提供的适配器对象；底层 asyncio 对象可通过 `ManagesConnection.driver_connection` 属性访问。

SQLAlchemy 对 DBAPI 连接的接口基于 `DBAPIConnection` 协议对象

另请参阅

`ManagesConnection.driver_connection`

当使用 Engine 时，我如何访问原始的 DBAPI 连接？

```py
attribute driver_connection: Any | None
```

Python DBAPI 或数据库驱动程序使用的“驱动程序级别”连接对象。

对于传统的[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 实现，这个对象将与`ManagesConnection.dbapi_connection`的对象相同。对于 asyncio 数据库驱动程序，这将是该驱动程序使用的最终的“连接”对象，例如`asyncpg.Connection`对象，该对象不具有标准的 pep-249 方法。

从版本 1.4.24 开始新增。

另请参阅

`ManagesConnection.dbapi_connection`

当使用 Engine 时，我如何访问原始的 DBAPI 连接？

```py
attribute in_use
```

如果当前检出了连接，则返回 True。

```py
attribute info
```

*继承自* `ManagesConnection` *的* `ManagesConnection.info` *属性。*

与由此 `ManagesConnection` 实例引用的底层 DBAPI 连接关联的信息字典，允许将用户定义的数据与连接关联起来。

此字典中的数据在 DBAPI 连接本身的生命周期内是持久的，包括池中的签入和签出。当连接被使无效并替换为新连接时，此字典将被清除。

对于未与`ConnectionPoolEntry`关联的`PoolProxiedConnection`实例，例如，如果它被分离，该属性将返回一个仅属于该`ConnectionPoolEntry`的字典。因此，`ManagesConnection.info` 属性将始终提供一个 Python 字典。

另请参阅

`ManagesConnection.record_info`

```py
method invalidate(e: BaseException | None = None, soft: bool = False) → None
```

*继承自* `ManagesConnection` *的* `ManagesConnection.invalidate()` *方法。*

将管理的连接标记为无效。

参数：

+   `e` – 表示使无效的原因的异常对象。

+   `soft` – 如果为 True，则连接不会关闭；相反，此连接将在下次检出时被回收。

另请参阅

关于使无效

```py
attribute record_info
```

*继承自* `ManagesConnection` *的* `ManagesConnection.record_info` *属性*

与此`ManagesConnection`关联的持久信息字典。

与`ManagesConnection.info`字典不同，此字典的生命周期与拥有它的`ConnectionPoolEntry`的生命周期相同；因此，对于连接池中特定条目的重新连接和连接失效，此字典将持续存在。

对于与`ConnectionPoolEntry`不相关的`PoolProxiedConnection`实例，例如如果它被分离，该属性返回 None。与永不为 None 的`ManagesConnection.info`字典形成对比。

另请参阅

`ManagesConnection.info`

```py
class sqlalchemy.pool.PoolProxiedConnection
```

一个类似连接适配器，用于[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 连接，其中包括特定于`Pool`实现的附加方法。

`PoolProxiedConnection`是内部`_ConnectionFairy`实现对象的公共接口；熟悉`_ConnectionFairy`的用户可以将此对象视为等效。

版本 2.0 中新增：`PoolProxiedConnection`为`_ConnectionFairy`内部类提供了公共接口。

**成员**

close(), dbapi_connection, detach(), driver_connection, info, invalidate(), is_detached, is_valid, record_info

**类签名**

类`sqlalchemy.pool.PoolProxiedConnection`（`sqlalchemy.pool.base.ManagesConnection`）

```py
method close() → None
```

释放此连接回连接池。

`PoolProxiedConnection.close()`方法遮蔽了[**PEP 249**](https://peps.python.org/pep-0249/)的`.close()`方法，改变其行为以将代理连接释放回连接池。

释放到池中后，连接是否保持“打开”并在 Python 进程中保留，还是实际关闭并从 Python 进程中移除，取决于正在使用的池实现及其配置和当前状态。

```py
attribute dbapi_connection: DBAPIConnection | None
```

正在跟踪的实际 DBAPI 连接的引用。

这是一个符合[**PEP 249**](https://peps.python.org/pep-0249/)的对象，对于传统的同步式方言，由正在使用的第三方 DBAPI 实现提供。对于异步方言，实现通常是由 SQLAlchemy 方言本身提供的适配器对象；底层的异步对象可通过`ManagesConnection.driver_connection`属性获得。

SQLAlchemy 对 DBAPI 连接的接口基于`DBAPIConnection`协议对象

另请参阅

`ManagesConnection.driver_connection`

在使用引擎时如何访问原始的 DBAPI 连接？

```py
method detach() → None
```

将此连接与其池分离。

这意味着当关闭时，连接将不再返回到池中，而是被实际关闭。相关的`ConnectionPoolEntry`与此 DBAPI 连接解除关联。

请注意，在分离后，池实现施加的任何整体连接限制约束可能会被违反，因为分离的连接已从池的知识和控制中移除。

```py
attribute driver_connection: Any | None
```

Python DBAPI 或数据库驱动程序使用的“驱动级”连接对象。

对于传统的[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 实现，此对象将与`ManagesConnection.dbapi_connection`的对象相同。对于异步数据库驱动程序，这将是该驱动程序使用的最终“连接”对象，例如`asyncpg.Connection`对象，该对象不具有标准的 pep-249 方法。

版本 1.4.24 中的新功能。

另请参阅

`ManagesConnection.dbapi_connection`

在使用引擎时如何获取原始 DBAPI 连接？

```py
attribute info
```

*继承自* `ManagesConnection` *的* `ManagesConnection.info` *属性*。

与此`ManagesConnection`实例引用的底层 DBAPI 连接相关联的信息字典，允许将用户定义的数据与连接相关联。

此字典中的数据对于 DBAPI 连接本身的生命周期是持久的，包括池的签入和签出。当连接被使无效并替换为新连接时，此字典将被清除。

对于与`ConnectionPoolEntry`不关联的`PoolProxiedConnection`实例，例如如果它被分离，则该属性返回一个与该`ConnectionPoolEntry`局部相关的字典。因此，`ManagesConnection.info`属性始终提供 Python 字典。

另请参阅

`ManagesConnection.record_info`

```py
method invalidate(e: BaseException | None = None, soft: bool = False) → None
```

*继承自* `ManagesConnection` *的* `ManagesConnection.invalidate()` *方法*。

将受管理的连接标记为无效。

参数：

+   `e` – 指示无效原因的异常对象。

+   `soft` – 如果为 True，则不会关闭连接；相反，此连接将在下次检出时被回收。

另请参阅

更多关于失效的信息

```py
attribute is_detached
```

如果此`PoolProxiedConnection`从其池中分离，则返回 True。

```py
attribute is_valid
```

如果此`PoolProxiedConnection`仍然引用活动的 DBAPI 连接，则返回 True。

```py
attribute record_info
```

*继承自* `ManagesConnection` *的* `ManagesConnection.record_info` *属性*。

与此`ManagesConnection`相关联的持久信息字典。

与`ManagesConnection.info`字典不同，此字典的生命周期与拥有它的`ConnectionPoolEntry`相同；因此，对于连接池中特定条目的重新连接和连接失效，此字典将持久存在。

对于不与`ConnectionPoolEntry`相关联的`PoolProxiedConnection`实例（例如如果它已分离），该属性返回 None。与永不为 None 的`ManagesConnection.info`字典形成对比。

另请参阅

`ManagesConnection.info`

```py
class sqlalchemy.pool._ConnectionFairy
```

代理一个 DBAPI 连接并提供解引用支持。

此为`Pool`实现内部使用的对象，为由该`Pool`提供的 DBAPI 连接提供上下文管理。该类的公共接口由`PoolProxiedConnection`类描述。请参阅该类获取公共 API 详细信息。

名称“fairy”灵感来源于`_ConnectionFairy`对象的生命周期是短暂的，因为它仅在从池中检出特定 DBAPI 连接的长度内存在，并且作为透明代理，它大部分时间是不可见的。

另请参阅

`PoolProxiedConnection`

`ConnectionPoolEntry`

**类签名**

类 `sqlalchemy.pool._ConnectionFairy` (`sqlalchemy.pool.base.PoolProxiedConnection`)

```py
class sqlalchemy.pool._ConnectionRecord
```

维护连接池中引用的一个连接的位置。

此为`Pool`实现内部使用的对象，为由该`Pool`维护的 DBAPI 连接提供上下文管理。该类的公共接口由`ConnectionPoolEntry`类描述。请参阅该类获取公共 API 详细信息。

另请参阅

`ConnectionPoolEntry`

`PoolProxiedConnection`

**类签名**

类 `sqlalchemy.pool._ConnectionRecord` (`sqlalchemy.pool.base.ConnectionPoolEntry`)

## 连接池配置

大多数情况下，`create_engine()` 函数返回的 `Engine` 已经集成了一个预先配置合理的 `QueuePool`。如果你只是阅读本节来学习如何启用连接池 - 恭喜！你已经完成了。

最常见的 `QueuePool` 调优参数可以直接作为关键字参数传递给 `create_engine()`：`pool_size`、`max_overflow`、`pool_recycle` 和 `pool_timeout`。例如：

```py
engine = create_engine(
    "postgresql+psycopg2://me@localhost/mydb", pool_size=20, max_overflow=0
)
```

所有 SQLAlchemy 连接池实现都有一个共同点，即它们都不会“预先创建”连接 - 所有实现都会在首次使用之前等待创建连接。在那时，如果没有额外的并发检出请求要求更多的连接，就不会创建额外的连接。这就是为什么 `create_engine()` 默认使用一个大小为五的 `QueuePool` 是完全可以的，而不用考虑应用程序是否真的需要排队五个连接 - 只有在应用程序实际上同时使用了五个连接时，池才会增长到这个大小，此时使用一个小池是完全适当的默认行为。

注意

`QueuePool` 类 **不兼容 asyncio**。当使用 `create_async_engine` 创建一个 `AsyncEngine` 实例时，会使用 `AsyncAdaptedQueuePool` 类，该类使用了一个兼容 asyncio 的队列实现。

## 切换连接池实现

使用不同类型的池与`create_engine()`的通常方法是使用`poolclass`参数。该参数接受从`sqlalchemy.pool`模块导入的类，并为您处理构建池的详细信息。一个常见的用例是禁用连接池，这可以通过使用`NullPool`实现来实现：

```py
from sqlalchemy.pool import NullPool

engine = create_engine(
    "postgresql+psycopg2://scott:tiger@localhost/test", poolclass=NullPool
)
```

## 使用自定义连接函数

参见自定义 DBAPI connect()参数/连接时例程部分，了解各种连接自定义例程的情况。

## 构造池

要单独使用`Pool`，则`creator`函数是唯一需要的参数，并且首先传递，然后是任何其他选项：

```py
import sqlalchemy.pool as pool
import psycopg2

def getconn():
    c = psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")
    return c

mypool = pool.QueuePool(getconn, max_overflow=10, pool_size=5)
```

然后可以使用`Pool.connect()`函数从池中获取 DBAPI 连接。此方法的返回值是包含在透明代理中的 DBAPI 连接：

```py
# get a connection
conn = mypool.connect()

# use it
cursor_obj = conn.cursor()
cursor_obj.execute("select foo")
```

透明代理的目的是拦截`close()`调用，以便将 DBAPI 连接返回到池中：

```py
# "close" the connection.  Returns
# it to the pool.
conn.close()
```

当代理被垃圾回收时，它也会将其包含的 DBAPI 连接返回到池中，尽管在 Python 中并不确定这是否立即发生（尽管在 cPython 中是典型的）。然而，不建议这样使用，特别是不支持使用 asyncio DBAPI 驱动程序。

## 返回时重置

池包含“返回时重置”行为，当连接返回到池中时，将调用 DBAPI 连接的`rollback()`方法。这样做是为了从连接中移除任何现有的事务状态，这不仅包括未提交的数据，还包括表和行锁。对于大多数 DBAPI，调用`rollback()`是廉价的，如果 DBAPI 已经完成了一个事务，那么该方法应该是一个空操作。

### 对于非事务连接禁用返回时重置

对于非常特定的情况，其中 `rollback()` 不实用，例如当使用配置为 自动提交 或者使用没有 ACID 能力的数据库（如 MySQL 的 MyISAM 引擎）的连接时，可以禁用返回时重置行为，这通常是出于性能考虑。可以通过使用 `Pool` 的 `Pool.reset_on_return` 参数来影响，该参数也可以从 `create_engine()` 中使用 `create_engine.pool_reset_on_return` 进行设置，传递一个值为 `None`。下面的示例中进行了说明，结合了 `AUTOCOMMIT` 的 `create_engine.isolation_level` 参数设置：

```py
non_acid_engine = create_engine(
    "mysql://scott:tiger@host/db",
    pool_reset_on_return=None,
    isolation_level="AUTOCOMMIT",
)
```

上述引擎在连接返回到池中时实际上不会执行回滚操作；由于启用了 AUTOCOMMIT，驱动程序也不会执行任何 BEGIN 操作。

### 自定义的返回重置方案

仅由单个 `rollback()` 组成的“返回重置”对于某些用例可能不够；特别是，使用临时表的应用程序可能希望这些表在连接签入时自动删除。一些（但并非全部）后端包含可以在数据库连接范围内“重置”这些表的功能，这可能是连接池重置的一种理想行为。其他服务器资源，例如准备好的语句句柄和服务器端语句缓存，可能会在签入过程之后持续存在，具体取决于具体情况是否希望如此。同样，一些（但再次并非全部）后端可能提供了重置此状态的方法。已知具有此类重置方案的两个 SQLAlchemy 包含的方言包括 Microsoft SQL Server，其中通常使用一个名为 `sp_reset_connection` 的未记录但广为人知的存储过程，以及 PostgreSQL，它有一系列命令包括 `DISCARD` `RESET`、`DEALLOCATE` 和 `UNLISTEN`。

以下示例说明了如何使用`PoolEvents.reset()`事件钩子，在返回时用 Microsoft SQL Server 的`sp_reset_connection`存储过程替换重置。`create_engine.pool_reset_on_return`参数设置为`None`，以便完全替换默认行为。自定义钩子实现在任何情况下都调用`.rollback()`，因为通常重要的是 DBAPI 自己的提交/回滚跟踪与事务状态保持一致：

```py
from sqlalchemy import create_engine
from sqlalchemy import event

mssql_engine = create_engine(
    "mssql+pyodbc://scott:tiger⁵HHH@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
    # disable default reset-on-return scheme
    pool_reset_on_return=None,
)

@event.listens_for(mssql_engine, "reset")
def _reset_mssql(dbapi_connection, connection_record, reset_state):
    if not reset_state.terminate_only:
        dbapi_connection.execute("{call sys.sp_reset_connection}")

    # so that the DBAPI itself knows that the connection has been
    # reset
    dbapi_connection.rollback()
```

在版本 2.0.0b3 中更改：为`PoolEvents.reset()`事件添加了额外的状态参数，并另外确保事件对所有“重置”事件都被调用，因此它适用于自定义“重置”处理程序的地方。之前使用`PoolEvents.checkin()`处理程序的方案仍然可用。

另请参阅

+   连接池临时表/资源重置 - 在 Microsoft SQL Server 文档中

+   连接池临时表/资源重置 - 在 PostgreSQL 文档中

### 记录返回时的重置事件

对包括返回时的重置在内的池事件进行记录可以设置为`logging.DEBUG`日志级别，以及`sqlalchemy.pool`记录器，或者在使用`create_engine()`时将`create_engine.echo_pool`设置为`"debug"`：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine("postgresql://scott:tiger@localhost/test", echo_pool="debug")
```

以上池将显示详细的日志，包括返回时的重置：

```py
>>> c1 = engine.connect()
DEBUG sqlalchemy.pool.impl.QueuePool Created new connection <connection object ...>
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> checked out from pool
>>> c1.close()
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> being returned to pool
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> rollback-on-return
```

### 对非事务连接禁用返回时的重置

对于一些特定情况下`rollback()`不适用的情况，比如在使用配置为自动提交或者在使用没有 ACID 功能的数据库，比如 MySQL 的 MyISAM 引擎时，可以禁用返回时的重置行为，通常出于性能原因。可以通过使用`Pool.reset_on_return`参数来实现，该参数也可以从`create_engine()`中获取，作为`create_engine.pool_reset_on_return`，传递一个值为`None`。下面的示例中演示了这一点，结合了`AUTOCOMMIT`的`create_engine.isolation_level`参数设置：

```py
non_acid_engine = create_engine(
    "mysql://scott:tiger@host/db",
    pool_reset_on_return=None,
    isolation_level="AUTOCOMMIT",
)
```

由于启用了 AUTOCOMMIT，上述引擎在连接返回到池时实际上不会执行 ROLLBACK 操作；由于驱动程序也不会执行任何 BEGIN 操作。

### 自定义返回时重置方案

对于一些使用临时表的应用程序，仅由一个`rollback()`组成的“返回时重置”可能不足够；特别是，使用临时表的应用程序可能希望在连接检入时自动删除这些表。一些（但并非所有）后端包括可以在数据库连接范围内“重置”这些表的功能，这可能是连接池重置的一种理想行为。其他服务器资源，如准备好的语句句柄和服务器端语句缓存，可能会在检入过程之后持续存在，具体取决于具体情况是否希望这样。同样，一些（但再次不是所有）后端可能提供一种重置此状态的方法。已知具有此类重置方案的两个 SQLAlchemy 包含的方言包括 Microsoft SQL Server，其中通常使用一个名为`sp_reset_connection`的未记录但广为人知的存储过程，以及 PostgreSQL，后者有一系列良好记录的命令，包括`DISCARD` `RESET`，`DEALLOCATE`和`UNLISTEN`。

以下示例说明了如何使用 `PoolEvents.reset()` 事件钩子将返回时的重置替换为 Microsoft SQL Server 的 `sp_reset_connection` 存储过程。 `create_engine.pool_reset_on_return` 参数设置为 `None`，以便自定义方案完全替换默认行为。自定义钩子实现在任何情况下都调用 `.rollback()`，因为通常重要的是 DBAPI 自己的提交/回滚跟踪与事务的状态保持一致：

```py
from sqlalchemy import create_engine
from sqlalchemy import event

mssql_engine = create_engine(
    "mssql+pyodbc://scott:tiger⁵HHH@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
    # disable default reset-on-return scheme
    pool_reset_on_return=None,
)

@event.listens_for(mssql_engine, "reset")
def _reset_mssql(dbapi_connection, connection_record, reset_state):
    if not reset_state.terminate_only:
        dbapi_connection.execute("{call sys.sp_reset_connection}")

    # so that the DBAPI itself knows that the connection has been
    # reset
    dbapi_connection.rollback()
```

从版本 2.0.0b3 中更改：为 `PoolEvents.reset()` 事件添加了额外的状态参数，并确保该事件被调用以进行所有“重置”发生，因此它适用于自定义“重置”处理程序的位置。以前使用 `PoolEvents.checkin()` 处理程序的方案仍然可用。

请参阅

+   临时表 / 资源重置以进行连接池 - 在 Microsoft SQL Server 文档中

+   临时表 / 资源重置以进行连接池 - 在 PostgreSQL 文档中

### 记录返回时的重置事件

可以将池事件的日志设置为 `logging.DEBUG` 日志级别，以及 `sqlalchemy.pool` 记录器，或者在使用 `create_engine()` 时将 `create_engine.echo_pool` 设置为 `"debug"`，以记录包括返回时的重置在内的池事件：

```py
>>> from sqlalchemy import create_engine
>>> engine = create_engine("postgresql://scott:tiger@localhost/test", echo_pool="debug")
```

上述连接池将显示详细的日志，包括返回时的重置：

```py
>>> c1 = engine.connect()
DEBUG sqlalchemy.pool.impl.QueuePool Created new connection <connection object ...>
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> checked out from pool
>>> c1.close()
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> being returned to pool
DEBUG sqlalchemy.pool.impl.QueuePool Connection <connection object ...> rollback-on-return
```

## 池事件

连接池支持事件接口，允许在第一次连接时、每次新连接时以及连接的签入和签出时执行钩子。有关详细信息，请参阅 `PoolEvents`。

## 处理断开连接

连接池具有刷新单个连接以及其整个连接集的能力，将先前池化的连接设置为“无效”。一个常见的用例是当数据库服务器重新启动时，连接池能够优雅地恢复，并且所有先前建立的连接都不再可用。有两种方法可以实现这一点。

### 断开连接处理 - 悲观

悲观方法是指在每次连接池检出时在 SQL 连接上发出测试语句，以测试数据库连接是否仍然可用。该实现是方言特定的，并且利用了 DBAPI 特定的 ping 方法，或者使用简单的 SQL 语句如“SELECT 1”，以便测试连接的活性。

此方法在连接检出过程中增加了一点开销，但否则是完全消除由于过期的池化连接而导致的数据库错误的最简单可靠方法。调用应用程序无需担心组织操作以便能够从池中检出的过期连接中恢复。

通过使用 `Pool.pre_ping` 参数，可以在每次连接池检出时对连接进行悲观测试，此参数可从 `create_engine()` 中通过 `create_engine.pool_pre_ping` 参数获得：

```py
engine = create_engine("mysql+pymysql://user:pw@host/db", pool_pre_ping=True)
```

“预连接测试”功能在每种方言上都是基于每个数据库适配器（DBAPI）特定的“ping”方法运行，如果不可用，则会发出与“SELECT 1”等效的 SQL，并捕获任何错误，并将错误检测为“断开”情况。如果 ping / 错误检查确定连接不可用，则连接将立即被回收，并且所有其他比当前时间更早的池化连接都将被作废，以便在下次检出它们时，它们也将在使用之前被回收。

如果数据库在“预连接测试”运行时仍然不可用，则初始连接将失败，并且将正常传播连接失败的错误。在数据库可用于连接但无法响应“ping”的情况下，“pre_ping”将尝试最多三次，然后放弃，并传播最后收到的数据库错误。

需要注意的是，预先 ping 的方法**不适用于在事务或其他 SQL 操作中断开连接的情况**。如果数据库在事务进行中变得不可用，则事务将丢失并引发数据库错误。虽然`Connection`对象将检测“断开”情况并在此条件发生时重新使用连接并使其余连接池无效，但引发异常的个别操作将丢失，应用程序需要放弃该操作或重新尝试整个事务。如果引擎使用 DBAPI 级别的自动提交连接进行配置，如设置包括 DBAPI 自动提交的事务隔离级别，则连接**可能**会在操作中透明地重新连接使用事件。有关示例，请参阅如何自动“重试”语句执行？部分。

对于使用“SELECT 1”并捕获错误以检测断开连接的方言，可以使用`DialectEvents.handle_error()`钩子来增强新的特定于后端的错误消息的断开连接测试。

#### 自定义 / 传统悲观 Ping

在添加`create_engine.pool_pre_ping`之前，历史上一直通过`ConnectionEvents.engine_connect()`引擎事件手动执行“预先 ping”方法。以下是最常见的配方，供参考，以防应用程序已经使用此类配方，或者需要特殊行为：

```py
from sqlalchemy import exc
from sqlalchemy import event
from sqlalchemy import select

some_engine = create_engine(...)

@event.listens_for(some_engine, "engine_connect")
def ping_connection(connection, branch):
    if branch:
        # this parameter is always False as of SQLAlchemy 2.0,
        # but is still accepted by the event hook.  In 1.x versions
        # of SQLAlchemy, "branched" connections should be skipped.
        return

    try:
        # run a SELECT 1\.   use a core select() so that
        # the SELECT of a scalar value without a table is
        # appropriately formatted for the backend
        connection.scalar(select(1))
    except exc.DBAPIError as err:
        # catch SQLAlchemy's DBAPIError, which is a wrapper
        # for the DBAPI's exception.  It includes a .connection_invalidated
        # attribute which specifies if this connection is a "disconnect"
        # condition, which is based on inspection of the original exception
        # by the dialect in use.
        if err.connection_invalidated:
            # run the same SELECT again - the connection will re-validate
            # itself and establish a new connection.  The disconnect detection
            # here also causes the whole connection pool to be invalidated
            # so that all stale connections are discarded.
            connection.scalar(select(1))
        else:
            raise
```

上述配方的优点在于我们利用了 SQLAlchemy 用于检测那些已知指示“断开”情况的 DBAPI 异常以及`Engine`对象在此条件发生时正确使当前连接池无效并允许当前`Connection`重新验证到新的 DBAPI 连接的能力。

### 断开连接处理 - 乐观

当不使用悲观处理时，以及当数据库在事务中的连接期间关闭和/或重新启动时，处理陈旧/关闭连接的另一种方法是让 SQLAlchemy 在发生断开连接时处理，此时池中的所有连接都将被作废，意味着它们被认为是陈旧的，并将在下次检出时刷新。此行为假定`Pool`与`Engine`一起使用。`Engine`具有可以检测断开连接事件并自动刷新池的逻辑。

当`Connection`尝试使用 DBAPI 连接，并引发与“断开”事件对应的异常时，连接将被作废。然后，`Connection`调用`Pool.recreate()`方法，有效地作废所有当前未检出的连接，以便在下次检出时用新连接替换。下面的代码示例说明了这个流程：

```py
from sqlalchemy import create_engine, exc

e = create_engine(...)
c = e.connect()

try:
    # suppose the database has been restarted.
    c.execute(text("SELECT * FROM table"))
    c.close()
except exc.DBAPIError as e:
    # an exception is raised, Connection is invalidated.
    if e.connection_invalidated:
        print("Connection was invalidated!")

# after the invalidate event, a new connection
# starts with a new Pool
c = e.connect()
c.execute(text("SELECT * FROM table"))
```

上面的示例说明了在检测到断开连接事件后，刷新池不需要任何特殊干预，池在此后会正常运行。然而，在数据库不可用事件发生时，每个正在使用的连接会引发一个数据库异常。在使用 ORM 会话的典型 Web 应用程序中，上述情况将对应于一个请求失败并显示 500 错误，然后 Web 应用程序在此之后会正常继续。因此，这种方法是“乐观”的，不预期频繁的数据库重启。

#### 设置池回收

可以增强“乐观”方法的另一个设置是设置池回收参数。该参数防止池使用已经过一定年龄的特定连接，并适用于数据库后端，例如 MySQL，在特定时间后自动关闭陈旧连接：

```py
from sqlalchemy import create_engine

e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", pool_recycle=3600)
```

在上述情况下，任何已经打开超过一小时的 DBAPI 连接将在下次检出时被作废并替换。请注意，作废**仅**发生在检出时 - 而不是在任何处于已检出状态的连接上。`pool_recycle`是`Pool`本身的一个函数，与是否使用`Engine`无关。### 更多关于作废的信息

`Pool`提供了“连接失效”服务，允许对连接进行显式失效以及在确定使连接无法使用的条件下自动失效。

“失效”意味着特定的 DBAPI 连接被从池中移除并丢弃。如果不清楚连接本身是否可能已关闭，则会调用此连接的`.close()`方法，但是如果此方法失败，则会记录异常，但操作仍将继续。

当使用`Engine`时，`Connection.invalidate()`方法是显式失效的常规入口点。导致 DBAPI 连接失效的其他条件包括：

+   当调用诸如`connection.execute()`之类的方法时引发的 DBAPI 异常，例如`OperationalError`被检测为指示所谓的“断开”条件。由于 Python DBAPI 没有用于确定异常性质的标准系统，所有 SQLAlchemy 方言都包含一个称为`is_disconnect()`的系统，它将检查异常对象的内容，包括字符串消息以及其中包含的任何潜在错误代码，以确定此异常是否指示连接不再可用。如果是这种情况，则调用`_ConnectionFairy.invalidate()`方法，然后丢弃 DBAPI 连接。

+   当连接被返回到池中，并且调用`connection.rollback()`或`connection.commit()`方法时，根据池的“返回时重置”行为，抛出异常。最后尝试调用连接的`.close()`方法，然后将其丢弃。

+   当实现`PoolEvents.checkout()`的监听器引发`DisconnectionError`异常时，表示连接将无法使用，需要进行新的连接尝试。

所有发生的失效都将调用`PoolEvents.invalidate()`事件。### 支持断开场景的新数据库错误代码

每个 SQLAlchemy 方言都包括一个名为 `is_disconnect()` 的例程，每当遇到 DBAPI 异常时就会调用它。DBAPI 异常对象被传递给此方法，方言特定的启发式将确定接收到的错误代码是否指示数据库连接已“断开”，或者处于无法使用的状态，这表明应该对其进行回收。这里应用的启发式可以通过 `DialectEvents.handle_error()` 事件钩子进行自定义，该钩子通常通过拥有的 `Engine` 对象进行建立。使用此钩子，所有发生的错误都将传递一个称为 `ExceptionContext` 的上下文对象。自定义事件钩子可以控制特定错误是否应被视为“断开”情况，以及此断开是否应导致整个连接池无效化。

例如，要添加支持将 Oracle 错误代码 `DPY-1001` 和 `DPY-4011` 视为已处理的断开代码，请在创建引擎后应用事件处理程序：

```py
import re

from sqlalchemy import create_engine

engine = create_engine("oracle://scott:tiger@dnsname")

@event.listens_for(engine, "handle_error")
def handle_exception(context: ExceptionContext) -> None:
    if not context.is_disconnect and re.match(
        r"^(?:DPI-1001|DPI-4011)", str(context.original_exception)
    ):
        context.is_disconnect = True

    return None
```

上述错误处理函数将被调用以处理所有抛出的 Oracle 错误，包括在使用 pool pre ping 功能时捕获的那些依赖于断开连接错误处理的后端（2.0 中新增）。

参见

`DialectEvents.handle_error()`  ### 断开连接处理 - 悲观

悲观的方法是指在每次连接池检出时发出一个测试语句，以测试数据库连接是否仍然可用。该实现是方言特定的，可以使用特定于 DBAPI 的 ping 方法，也可以使用简单的 SQL 语句如“SELECT 1”来测试连接的活动性。

这种方法会给连接检出过程增加一点开销，但是除此之外，它是完全消除由于过时的连接池连接而导致数据库错误的最简单和可靠的方法。调用应用程序不需要关心组织操作以便从连接池中检出过时的连接。

通过使用 `create_engine()` 的 `create_engine.pool_pre_ping` 参数，可以通过使用 `Pool.pre_ping` 参数来实现在检出时对连接进行悲观测试：

```py
engine = create_engine("mysql+pymysql://user:pw@host/db", pool_pre_ping=True)
```

“预连接”功能是在每个方言基础上运行的，通过调用特定于 DBAPI 的“ping”方法，或者如果不可用，则会发出等效于“SELECT 1”的 SQL，捕获任何错误并将错误检测为“断开”情况。如果 ping / 错误检查确定连接不可用，则连接将立即被回收，并且所有比当前时间更旧的池化连接都将被作废，以便下次检出时，在使用之前也将被回收。

如果在“预 ping”运行时数据库仍然不可用，则初始连接将失败，并且连接失败的错误将正常传播。在数据库可用于连接但无法响应“ping”的情况下，将尝试最多三次“预 ping”，然后放弃，传播上次收到的数据库错误。

需要注意的是，预连接方法**不适用于事务中断开的连接或其他 SQL 操作**。如果数据库在事务进行中变得不可用，则事务将丢失并引发数据库错误。虽然 `Connection` 对象会检测到“断开”情况并在发生此情况时回收连接以及使其余连接池无效，但引发异常的个别操作将丢失，由应用程序来放弃操作或重新尝试整个事务。如果引擎使用 DBAPI 级别的自动提交连接进行配置，如 设置事务隔离级别，包括 DBAPI 自动提交，则可以使用事件在操作中透明地重新连接。有关示例，请参阅 如何“自动重试”语句执行？ 节。

对于使用“SELECT 1”并捕获错误以检测断开的方言，可以使用 `DialectEvents.handle_error()` 钩子来增加新的后端特定错误消息的断开测试。

#### 自定义 / 传统悲观的预连接

在添加 `create_engine.pool_pre_ping` 之前，历史上“预 ping”方法是通过使用 `ConnectionEvents.engine_connect()` 引擎事件手动执行的。以下是最常见的配方，供参考，以防应用程序已经使用了这样的配方，或者需要特殊的行为：

```py
from sqlalchemy import exc
from sqlalchemy import event
from sqlalchemy import select

some_engine = create_engine(...)

@event.listens_for(some_engine, "engine_connect")
def ping_connection(connection, branch):
    if branch:
        # this parameter is always False as of SQLAlchemy 2.0,
        # but is still accepted by the event hook.  In 1.x versions
        # of SQLAlchemy, "branched" connections should be skipped.
        return

    try:
        # run a SELECT 1\.   use a core select() so that
        # the SELECT of a scalar value without a table is
        # appropriately formatted for the backend
        connection.scalar(select(1))
    except exc.DBAPIError as err:
        # catch SQLAlchemy's DBAPIError, which is a wrapper
        # for the DBAPI's exception.  It includes a .connection_invalidated
        # attribute which specifies if this connection is a "disconnect"
        # condition, which is based on inspection of the original exception
        # by the dialect in use.
        if err.connection_invalidated:
            # run the same SELECT again - the connection will re-validate
            # itself and establish a new connection.  The disconnect detection
            # here also causes the whole connection pool to be invalidated
            # so that all stale connections are discarded.
            connection.scalar(select(1))
        else:
            raise
```

上述配方的优点是我们利用 SQLAlchemy 的设施来检测那些已知指示“断开连接”情况的 DBAPI 异常，以及 `Engine` 对象在此条件发生时正确地使当前连接池无效并允许当前 `Connection` 重新验证到新的 DBAPI 连接。  #### 自定义/遗留悲观 ping

在添加 `create_engine.pool_pre_ping` 之前，"预先 ping" 方法的历史记录通常是使用 `ConnectionEvents.engine_connect()` 引擎事件手动执行的。下面是最常见的配方，供参考，以防应用程序已经使用此类配方，或者需要特殊的行为：

```py
from sqlalchemy import exc
from sqlalchemy import event
from sqlalchemy import select

some_engine = create_engine(...)

@event.listens_for(some_engine, "engine_connect")
def ping_connection(connection, branch):
    if branch:
        # this parameter is always False as of SQLAlchemy 2.0,
        # but is still accepted by the event hook.  In 1.x versions
        # of SQLAlchemy, "branched" connections should be skipped.
        return

    try:
        # run a SELECT 1\.   use a core select() so that
        # the SELECT of a scalar value without a table is
        # appropriately formatted for the backend
        connection.scalar(select(1))
    except exc.DBAPIError as err:
        # catch SQLAlchemy's DBAPIError, which is a wrapper
        # for the DBAPI's exception.  It includes a .connection_invalidated
        # attribute which specifies if this connection is a "disconnect"
        # condition, which is based on inspection of the original exception
        # by the dialect in use.
        if err.connection_invalidated:
            # run the same SELECT again - the connection will re-validate
            # itself and establish a new connection.  The disconnect detection
            # here also causes the whole connection pool to be invalidated
            # so that all stale connections are discarded.
            connection.scalar(select(1))
        else:
            raise
```

上述配方的优点是我们利用 SQLAlchemy 的设施来检测那些已知指示“断开连接”情况的 DBAPI 异常，以及 `Engine` 对象在此条件发生时正确地使当前连接池无效并允许当前 `Connection` 重新验证到新的 DBAPI 连接。

### 断开处理 - 乐观

当不使用悲观处理，并且在事务中连接使用期间数据库关闭和/或重新启动时，处理陈旧/关闭连接的另一种方法是让 SQLAlchemy 在断开连接时处理，此时池中的所有连接都将被作废，意味着它们被假定为陈旧的，并将在下次检出时刷新。此行为假定与 `Pool` 一起使用 `Engine`。`Engine` 具有可以检测到断开连接事件并自动刷新池的逻辑。

当 `Connection` 尝试使用 DBAPI 连接，并引发与“断开”事件对应的异常时，连接将被作废。然后，`Connection` 调用 `Pool.recreate()` 方法，有效地作废所有当前未检出的连接，以便在下一次检出时用新连接替换它们。下面的代码示例说明了这个流程：

```py
from sqlalchemy import create_engine, exc

e = create_engine(...)
c = e.connect()

try:
    # suppose the database has been restarted.
    c.execute(text("SELECT * FROM table"))
    c.close()
except exc.DBAPIError as e:
    # an exception is raised, Connection is invalidated.
    if e.connection_invalidated:
        print("Connection was invalidated!")

# after the invalidate event, a new connection
# starts with a new Pool
c = e.connect()
c.execute(text("SELECT * FROM table"))
```

上面的示例说明，在检测到断开连接事件后，不需要任何特殊干预来刷新池，池在此后会正常运行。然而，在数据库不可用事件发生时，每个正在使用的连接都会引发一个数据库异常。在使用 ORM 会话的典型 Web 应用程序中，上述情况将对应于一个请求失败并返回 500 错误，然后 Web 应用程序在此之后会正常继续运行。因此，这种方法是“乐观的”，不预期频繁地重启数据库。

#### 设置池回收

可以增强“乐观”方法的另一个设置是设置池回收参数。该参数防止池使用已经存在一段时间的特定连接，适用于数据库后端（如 MySQL），该后端在一段特定时间后会自动关闭已经过时的连接：

```py
from sqlalchemy import create_engine

e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", pool_recycle=3600)
```

在上述设置中，任何已经打开超过一小时的 DBAPI 连接将在下一次检出时被作废并替换。请注意，作废仅在检出时发生 - 不会作用于任何处于已检出状态的连接。`pool_recycle` 是 `Pool` 本身的一个函数，与是否使用 `Engine` 无关。  #### 设置池回收

可以增强“乐观”方法的另一个设置是设置池回收参数。该参数防止池使用已经存在一段时间的特定连接，适用于数据库后端（如 MySQL），该后端在一段特定时间后会自动关闭已经过时的连接：

```py
from sqlalchemy import create_engine

e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", pool_recycle=3600)
```

在上述设置中，任何已经打开超过一小时的 DBAPI 连接将在下一次检出时被作废并替换。请注意，作废仅在检出时发生 - 不会作用于任何处于已检出状态的连接。`pool_recycle` 是 `Pool` 本身的一个函数，与是否使用 `Engine` 无关。

### 更多关于作废的内容

`Pool`提供“连接失效”服务，允许显式失效连接以及在确定使连接不可用的条件下自动失效。

“失效”意味着特定的 DBAPI 连接从池中移除并丢弃。如果不清楚连接本身是否关闭，则在此连接上调用`.close()`方法，但如果此方法失败，则记录异常但操作仍继续。

在使用`Engine`时，`Connection.invalidate()`方法通常是显式失效的入口点。其他可能使 DBAPI 连接失效的条件包括：

+   例如`OperationalError`这样的 DBAPI 异常，在调用`connection.execute()`等方法时引发，被检测为所谓的“断开”条件。由于 Python DBAPI 没有确定异常性质的标准系统，所有 SQLAlchemy 方言都包括一个名为`is_disconnect()`的系统，它将检查异常对象的内容，包括字符串消息和其中包含的任何潜在错误代码，以确定此异常是否指示连接不再可用。如果是这种情况，将调用`_ConnectionFairy.invalidate()`方法，然后丢弃 DBAPI 连接。

+   当连接返回到池中，并调用`connection.rollback()`或`connection.commit()`方法时，根据池的“返回时重置”行为，抛出异常。将最后尝试在连接上调用`.close()`方法，然后将其丢弃。

+   当实现`PoolEvents.checkout()`的监听器引发`DisconnectionError`异常时，表示连接将无法使用，需要进行新的连接尝试。

所有发生的失效将调用`PoolEvents.invalidate()`事件。

### 支持断开情景的新数据库错误代码

SQLAlchemy 方言（dialects）中都包含一个名为 `is_disconnect()` 的程序，当遇到 DBAPI 异常时会调用该程序。DBAPI 异常对象会传递给这个方法，在这里，方言特定的启发法则将确定接收到的错误代码是否指示数据库连接已被“断开”，或者处于其他无法使用的状态，表明应该重新使用该连接。此处应用的启发式方法可以使用 `DialectEvents.handle_error()` 事件钩子进行自定义，通常是通过所属的 `Engine` 对象建立的。使用此钩子，所有发生的错误都会传递一个称为 `ExceptionContext` 的上下文对象。自定义事件钩子可以控制特定错误是否应该被视为“断开”情况，以及此断开是否应该导致整个连接池无效化。

例如，要添加支持将 Oracle 错误代码 `DPY-1001` 和 `DPY-4011` 视为断开代码进行处理，需要在创建引擎后应用一个事件处理程序：

```py
import re

from sqlalchemy import create_engine

engine = create_engine("oracle://scott:tiger@dnsname")

@event.listens_for(engine, "handle_error")
def handle_exception(context: ExceptionContext) -> None:
    if not context.is_disconnect and re.match(
        r"^(?:DPI-1001|DPI-4011)", str(context.original_exception)
    ):
        context.is_disconnect = True

    return None
```

上述错误处理函数将对所有引发的 Oracle 错误进行调用，包括在使用 pool pre ping 特性时捕获的错误，用于依赖于断开连接错误处理的后端（在 2.0 版本中新增）。

另见

`DialectEvents.handle_error()`

## 使用 FIFO vs. LIFO

`QueuePool` 类特征一个名为 `QueuePool.use_lifo` 的标志，也可以通过标志 `create_engine.pool_use_lifo` 在 `create_engine()` 中访问。将此标志设置为 `True` 会导致池的“队列”行为变为“堆栈”，例如，返回到池中的最后一个连接将在下一次请求中首先被使用。与池的先入先出长期行为相反，后者产生一个轮转效果，依次使用池中的每个连接，LIFO 模式允许多余的连接在池中保持空闲，从而允许服务器端超时方案关闭这些连接。FIFO 和 LIFO 的区别基本上是池是否在空闲期间保持一组完整的连接准备就绪：

```py
engine = create_engine("postgreql://", pool_use_lifo=True, pool_pre_ping=True)
```

此外，我们还使用了`create_engine.pool_pre_ping`标志，以便由服务器端关闭的连接被连接池优雅地处理，并替换为新连接。

注意，此标志仅适用于`QueuePool`的使用。

版本 1.3 中的新功能。

另请参阅

处理断开连接

## 使用连接池与多进程或 os.fork()

在使用连接池时（通过`create_engine()`创建的`Engine`），至关重要的是，**不要共享池化的连接给分叉的进程**。 TCP 连接表示为文件描述符，通常跨进程边界工作，这意味着这将导致在两个或更多完全独立的 Python 解释器状态的代表性之间并发访问文件描述符。

根据驱动程序和操作系统的具体情况，此处出现的问题范围从不起作用的连接到被多个进程同时使用的套接字连接，导致消息中断（后者通常是最常见的情况）。

SQLAlchemy `Engine`对象指的是现有数据库连接的连接池。因此，当此对象被复制到子进程时，目标是确保不会携带任何数据库连接。有四种一般方法来解决这个问题：

1.  使用`NullPool`禁用连接池。这是最简单的、一次性的系统，防止`Engine`重复使用任何连接：

    ```py
    from sqlalchemy.pool import NullPool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname", poolclass=NullPool)
    ```

1.  在子进程的初始化阶段调用`Engine.dispose()`，传递值为`False`的`Engine.dispose.close`参数。这样新进程就不会触及任何父进程的连接，而是会以新的连接开始。**这是推荐的方法**：

    ```py
    from multiprocessing import Pool

    engine = create_engine("mysql+mysqldb://user:pass@host/dbname")

    def run_in_process(some_data_record):
        with engine.connect() as conn:
            conn.execute(text("..."))

    def initializer():
      """ensure the parent proc's database connections are not touched
     in the new connection pool"""
        engine.dispose(close=False)

    with Pool(10, initializer=initializer) as p:
        p.map(run_in_process, data)
    ```

    版本 1.4.33 中的新功能：添加了`Engine.dispose.close`参数，以允许在子进程中替换连接池，而不会干扰父进程使用的连接。

1.  在创建子进程之前**直接调用**`Engine.dispose()`。这也将导致子进程以新的连接池启动，同时确保父连接不会传递给子进程：

    ```py
    engine = create_engine("mysql://user:pass@host/dbname")

    def run_in_process():
        with engine.connect() as conn:
            conn.execute(text("..."))

    # before process starts, ensure engine.dispose() is called
    engine.dispose()
    p = Process(target=run_in_process)
    p.start()
    ```

1.  可以向连接池应用事件处理程序，以测试跨进程边界共享的连接，并使其无效：

    ```py
    from sqlalchemy import event
    from sqlalchemy import exc
    import os

    engine = create_engine("...")

    @event.listens_for(engine, "connect")
    def connect(dbapi_connection, connection_record):
        connection_record.info["pid"] = os.getpid()

    @event.listens_for(engine, "checkout")
    def checkout(dbapi_connection, connection_record, connection_proxy):
        pid = os.getpid()
        if connection_record.info["pid"] != pid:
            connection_record.dbapi_connection = connection_proxy.dbapi_connection = None
            raise exc.DisconnectionError(
                "Connection record belongs to pid %s, "
                "attempting to check out in pid %s" % (connection_record.info["pid"], pid)
            )
    ```

    在上面，我们使用了类似于断开处理 - 悲观中描述的方法来处理在不同父进程中起源的 DBAPI 连接，将其视为“无效”连接，强制池回收连接记录以建立新连接。

上述策略将适用于在进程之间共享`Engine`的情况。仅仅上述步骤并不足以处理在进程边界共享特定`Connection`的情况；最好将特定`Connection`的范围局限于单个进程（和线程）。直接跨进程共享任何类型的进行中的事务状态，比如已开始事务并引用活动`Connection`实例的 ORM `Session`对象，也不受支持；最好在新进程中创建新的`Session`对象。

## 直接使用池实例

可以直接使用池实现而不需要引擎。这可用于只希望使用池行为而不需要所有其他 SQLAlchemy 功能的应用程序。在下面的示例中，使用`create_pool_from_url()`获取`MySQLdb`方言的默认池：

```py
from sqlalchemy import create_pool_from_url

my_pool = create_pool_from_url(
    "mysql+mysqldb://", max_overflow=5, pool_size=5, pre_ping=True
)

con = my_pool.connect()
# use the connection
...
# then close it
con.close()
```

如果未指定要创建的池的类型，则将使用该方言的默认池。可以直接指定`poolclass`参数，如下例所示：

```py
from sqlalchemy import create_pool_from_url
from sqlalchemy import NullPool

my_pool = create_pool_from_url("mysql+mysqldb://", poolclass=NullPool)
```

## API 文档 - 可用的池实现

| 对象名称 | 描述 |
| --- | --- |
| _ConnectionFairy | 代理一个 DBAPI 连接，并提供返回引用支持。 |
| _ConnectionRecord | 维护连接池中的位置，引用一个池化连接。 |
| AssertionPool | 允许每次最多只有一个已签出连接的`Pool`。 |
| 异步适配队列池 | `队列池`的一个适用于 asyncio 的版本。 |
| 连接池条目 | 代表`池`实例的个别数据库连接的对象的接口。 |
| 管理连接 | 用于两个连接管理接口`池代理连接`和`连接池条目`的通用基类。 |
| 空池 | 不对连接进行池化的池。 |
| 池 | 连接池的抽象基类。 |
| 池代理连接 | 用于[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 连接的类似连接的适配器，其中包括特定于`池`实现的附加方法。 |
| 队列池 | 对打开连接数量施加限制的`池`。 |
| 单例线程池 | 每个线程维护一个连接的池。 |
| 静态池 | 用于所有请求的恰好一个连接的池。 |

```py
class sqlalchemy.pool.Pool
```

连接池的抽象基类。

**成员**

__init__(), connect(), dispose(), recreate()

**类签名**

类`sqlalchemy.pool.Pool` (`sqlalchemy.log.Identified`, `sqlalchemy.event.registry.EventTarget`)的签名

```py
method __init__(creator: _CreatorFnType | _CreatorWRecFnType, recycle: int = -1, echo: log._EchoFlagType = None, logging_name: str | None = None, reset_on_return: _ResetStyleArgType = True, events: List[Tuple[_ListenerFnType, str]] | None = None, dialect: _ConnDialect | Dialect | None = None, pre_ping: bool = False, _dispatch: _DispatchCommon[Pool] | None = None)
```

构建一个池。

参数：

+   `creator` – 一个可调用的函数，返回一个 DB-API 连接对象。该函数将带有参数调用。

+   `recycle` – 如果设置为除-1 以外的值，则连接回收之间的秒数，这意味着在检出时，如果超过此超时，则连接将被关闭并替换为新打开的连接。默认为-1。

+   `logging_name` – 用于“sqlalchemy.pool”记录器中生成的日志记录的“name”字段的字符串标识符。默认为对象 id 的十六进制字符串。

+   `echo` –

    如果为真，则连接池将记录信息输出，例如当连接失效时以及当连接被回收时，默认日志处理程序为`sys.stdout`。如果设置为字符串`"debug"`，日志将包括池检出和检入。

    `Pool.echo` 参数也可以通过使用 `create_engine.echo_pool` 参数从 `create_engine()` 调用中设置。

    另请参阅

    配置日志记录 - 如何配置日志记录的更多详细信息。

+   `reset_on_return` –

    确定在将连接返回到池中时执行的步骤，这些步骤否则不会由 `Connection` 处理。可以通过 `create_engine()` 通过 `create_engine.pool_reset_on_return` 参数来使用。

    `Pool.reset_on_return` 可以有以下任何值：

    +   `"rollback"` - 在连接上调用 rollback()，释放锁和事务资源。这是默认值。绝大多数情况下应该保持此值不变。

    +   `"commit"` - 在连接上调用 commit()，释放锁和事务资源。在某些情况下，如微软 SQL Server，如果发出了 commit，则可能需要提交。然而，这个值比 ‘rollback’ 更危险，因为任何存在于事务中的数据更改都会无条件地提交。

    +   `None` - 在连接上不执行任何操作。如果数据库/DBAPI 在所有时刻都以纯“自动提交”模式工作，或者使用 `PoolEvents.reset()` 事件处理程序建立了自定义重置处理程序，则此设置可能是合适的。

    +   `True` - 与 ‘rollback’ 相同，这是为了向后兼容而存在的。

    +   `False` - 与 None 相同，这是为了向后兼容而存在的。

    为了进一步定制返回时的重置，可以使用 `PoolEvents.reset()` 事件钩子，该钩子可以在重置时执行任何所需的连接活动。

    另请参阅

    返回时重置

    `PoolEvents.reset()`

+   `events` – 一个 2-元组列表，每个元组的形式为 `(callable, target)`，将在构造时传递给 `listen()`。此处提供是为了在应用方言级别的监听器之前，可以通过 `create_engine()` 分配事件监听器。

+   `dialect` – 一个处理 DBAPI 连接的回滚（rollback()）、关闭（close()）或提交（commit()）工作的 `Dialect`。如果省略，将使用内置的“存根”方言。使用 `create_engine()` 的应用程序不应使用此参数，因为它由引擎创建策略处理。

+   `pre_ping` –

    如果为 True，则池将在检出连接时发出“ping”（通常为“SELECT 1”，但是是特定于方言的），以测试连接是否活动。如果不活动，则连接将被透明地重新连接，并在成功后，所有在该时间戳之前建立的其他池连接将无效。还需要传递一个方言以解释断开连接错误。

    1.2 版本中新增。

```py
method connect() → PoolProxiedConnection
```

从池中返回一个 DBAPI 连接。

连接被检测工具检测，以便在调用其 `close()` 方法时，连接将被返回到池中。

```py
method dispose() → None
```

处置此池。

此方法可能导致仍处于检出状态的连接保持打开状态，因为它仅影响池中处于空闲状态的连接。

另请参见

`Pool.recreate()`

```py
method recreate() → Pool
```

返回一个新的 `Pool`，与此相同类的池，并配置相同的创建参数。

此方法与 `dispose()` 结合使用，用于关闭整个 `Pool` 并在其位置创建一个新的 Pool。

```py
class sqlalchemy.pool.QueuePool
```

施加对打开连接数量的限制的 `Pool`。

`QueuePool` 是除了 SQLite 的 `:memory:` 数据库之外，所有 `Engine` 对象的默认池实现。

`QueuePool` 类与 asyncio 不兼容，并且 `create_async_engine()`。当使用 `create_async_engine()` 时，如果没有指定其他类型的池，则会自动使用 `AsyncAdaptedQueuePool` 类。

另请参见

`AsyncAdaptedQueuePool`

**成员**

__init__(), dispose(), recreate()

**类签名**

类`sqlalchemy.pool.QueuePool`（`sqlalchemy.pool.base.Pool`）

```py
method __init__(creator: _CreatorFnType | _CreatorWRecFnType, pool_size: int = 5, max_overflow: int = 10, timeout: float = 30.0, use_lifo: bool = False, **kw: Any)
```

构造一个 QueuePool。

参数：

+   `creator` – 一个可调用函数，返回一个与`Pool.creator`相同的 DB-API 连接对象。

+   `pool_size` – 要维护的池的大小，默认为 5。这是将持续保留在池中的最大连接数。请注意，池开始时没有连接；一旦请求了这个数量的连接，这个数量的连接将保持不变。`pool_size` 可设置为 0，表示没有大小限制；要禁用池化，请使用 `NullPool`。

+   `max_overflow` – 池的最大溢出大小。当已签出连接的数量达到 pool_size 中设置的大小时，将返回额外的连接，直到达到此限制为止。当这些额外的连接返回到池中时，它们将被断开并丢弃。因此，池允许的同时连接数是 pool_size + max_overflow，池允许的“睡眠”连接总数是 pool_size。max_overflow 可设置为-1，表示无溢出限制；不会对并发连接的总数设置限制。默认为 10。

+   `timeout` – 在放弃返回连接之前等待的秒数。默认为 30.0。这可以是一个浮点数，但受 Python 时间函数的限制，可能不可靠，精度在几十毫秒内。

+   `use_lifo` –

    在检索连接时使用 LIFO（后进先出）而不是 FIFO（先进先出）。使用 LIFO，服务器端的超时方案可以在非高峰使用期间减少使用的连接数量。在规划服务器端超时时，请确保使用回收或预检查策略来优雅地处理陈旧的连接。

    新功能，版本 1.3。

    另请参阅

    使用 FIFO vs. LIFO

    处理断开连接

+   `**kw` – 其他关键字参数，包括`Pool.recycle`、`Pool.echo`、`Pool.reset_on_return`等，将传递给`Pool`构造函数。

```py
method dispose() → None
```

释放此池。

此方法可能导致已签出的连接保持打开状态，因为它只影响池中处于空闲状态的连接。

另请参阅

`Pool.recreate()`

```py
method recreate() → QueuePool
```

返回一个新的`Pool`，与此对象相同类别的对象，并配置相同的创建参数。

此方法与 `dispose()` 结合使用，以关闭整个 `Pool` 并在其位置创建一个新的池。

```py
class sqlalchemy.pool.AsyncAdaptedQueuePool
```

`QueuePool` 的一个与 asyncio 兼容的版本。

当使用从 `create_async_engine()` 生成的 `AsyncEngine` 引擎时，默认使用此池。它使用了一个与 asyncio 兼容的队列实现，不使用 `threading.Lock`。

`AsyncAdaptedQueuePool` 的参数和操作与 `QueuePool` 相同。

**类签名**

class `sqlalchemy.pool.AsyncAdaptedQueuePool` (`sqlalchemy.pool.impl.QueuePool`)

```py
class sqlalchemy.pool.SingletonThreadPool
```

每个线程维护一个连接的池。

每个线程维护一个连接，永远不会将连接移动到其创建的线程之外。

警告

`SingletonThreadPool` 将对超出 `pool_size` 大小设置的任意连接调用 `.close()`，例如，如果使用的唯一 **线程标识** 大于 `pool_size` 所指定的数量。此清理是非确定性的，并且不会因连接是否与这些线程标识关联并当前正在使用而受到影响。

`SingletonThreadPool` 在未来的版本中可能会得到改进，但在当前状态下，它通常仅用于使用 SQLite 的 `:memory:` 数据库的测试场景，并不建议用于生产环境。

`SingletonThreadPool` 类 **不兼容** asyncio 和 `create_async_engine()`。

选项与 `Pool` 相同，以及：

参数：

**pool_size** – 同时维护连接的线程数。默认为五。

当使用基于内存的数据库时，SQLite 方言会自动使用 `SingletonThreadPool`。请参阅 SQLite。

**成员**

connect(), dispose(), recreate()

**类签名**

类`sqlalchemy.pool.SingletonThreadPool`（`sqlalchemy.pool.base.Pool`）

```py
method connect() → PoolProxiedConnection
```

从池中返回一个 DBAPI 连接。

连接被仪器化，这样当调用其`close()`方法时，连接将被返回到池中。

```py
method dispose() → None
```

释放此池。

```py
method recreate() → SingletonThreadPool
```

返回一个新的`Pool`，与此相同类别，并使用相同的创建参数配置。

此方法与`dispose()`结合使用，关闭整个`Pool`并创建一个新的替代品。

```py
class sqlalchemy.pool.AssertionPool
```

一个允许在任何给定时间最多有一个已检出连接的`Pool`。

如果同时检出了多个连接，则会引发异常。 对于调试使用比期望的连接更多的代码很有用。

`AssertionPool`类与 asyncio 和`create_async_engine()` **兼容**。

**成员**

dispose(), recreate()

**类签名**

类`sqlalchemy.pool.AssertionPool`（`sqlalchemy.pool.base.Pool`）

```py
method dispose() → None
```

释放此池。

此方法使得已检出连接保持打开的可能性，因为它仅影响池中处于空闲状态的连接。

另请参阅

`Pool.recreate()`

```py
method recreate() → AssertionPool
```

返回一个新的`Pool`，与此相同类别，并使用相同的创建参数配置。

此方法与`dispose()`结合使用，关闭整个`Pool`并创建一个新的替代品。

```py
class sqlalchemy.pool.NullPool
```

不池化连接的池。

反而，它会逐个连接地打开和关闭底层的 DB-API 连接。

此池实现不支持与重新连接相关的函数，如`recycle`和连接失效，因为没有连接持续存在。

`NullPool`类**与**asyncio 和`create_async_engine()`兼容。

**成员**

dispose(), recreate()

**类签名**

类`sqlalchemy.pool.NullPool`（`sqlalchemy.pool.base.Pool`）

```py
method dispose() → None
```

处置此池。

这种方法留下了已检出连接保持打开的可能性，因为它只影响池中处于空闲状态的连接。

另见

`Pool.recreate()`

```py
method recreate() → NullPool
```

返回一个新的`Pool`，与此相同的类，并配置相同的创建参数。

此方法与`dispose()`一起使用，以关闭整个`Pool`并在其位置创建一个新的。

```py
class sqlalchemy.pool.StaticPool
```

一个连接池，用于所有请求。

重新连接相关的函数，如`recycle`和连接失效（也用于支持自动重新连接），目前仅部分支持，可能不会产生良好的结果。

`StaticPool`类**与**asyncio 和`create_async_engine()`兼容。

**成员**

dispose(), recreate()

**类签名**

类`sqlalchemy.pool.StaticPool`（`sqlalchemy.pool.base.Pool`）

```py
method dispose() → None
```

处置此池。

这种方法留下了已检出连接保持打开的可能性，因为它只影响池中处于空闲状态的连接。

另见

`Pool.recreate()`

```py
method recreate() → StaticPool
```

返回一个新的`Pool`，与此相同的类，并配置相同的创建参数。

此方法与`dispose()`一起使用，以关闭整个`Pool`并在其位置创建一个新的。

```py
class sqlalchemy.pool.ManagesConnection
```

两个连接管理接口的共同基类`PoolProxiedConnection`和`ConnectionPoolEntry`。

这两个对象通常通过连接池事件钩子在公共 API 中公开，详见 `PoolEvents`。

**成员**

dbapi_connection, driver_connection, info, invalidate(), record_info

新版本 2.0 中新增。

```py
attribute dbapi_connection: DBAPIConnection | None
```

跟踪的实际 DBAPI 连接的引用。

这是一个[**PEP 249**](https://peps.python.org/pep-0249/)-兼容对象，对于传统的同步式方言，由使用的第三方 DBAPI 实现提供。对于 asyncio 方言，实现通常是 SQLAlchemy 方言本身提供的适配器对象；底层 asyncio 对象可通过 `ManagesConnection.driver_connection` 属性获得。

SQLAlchemy 对 DBAPI 连接的接口基于 `DBAPIConnection` 协议对象

另请参阅

`ManagesConnection.driver_connection`

在使用引擎时如何获取原始 DBAPI 连接？

```py
attribute driver_connection: Any | None
```

Python DBAPI 或数据库驱动程序中使用的“驱动程序级别”连接对象。

对于传统的[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 实现，此对象将与 `ManagesConnection.dbapi_connection` 的对象相同。对于 asyncio 数据库驱动程序，这将是该驱动程序使用的最终“连接”对象，例如 `asyncpg.Connection` 对象，该对象不会具有标准的 pep-249 方法。

新版本 1.4.24 中新增。

另请参阅

`ManagesConnection.dbapi_connection`

在使用引擎时如何获取原始 DBAPI 连接？

```py
attribute info
```

与此 `ManagesConnection` 实例引用的底层 DBAPI 连接相关联的信息字典，允许将用户定义的数据与连接关联起来。

此字典中的数据在 DBAPI 连接本身的生命周期内是持久的，包括池检入和检出期间。当连接无效并替换为新连接时，此字典将被清除。

对于一个未关联`ConnectionPoolEntry`的`PoolProxiedConnection`实例，例如如果它被分离，该属性返回一个仅限于该`ConnectionPoolEntry`的字典。因此，`ManagesConnection.info`属性将始终提供一个 Python 字典。

另请参阅

`ManagesConnection.record_info`

```py
method invalidate(e: BaseException | None = None, soft: bool = False) → None
```

将受管连接标记为无效。

参数：

+   `e` – 指示失效原因的异常对象。

+   `soft` – 如果为 True，则连接不会关闭；相反，此连接将在下次检出时被回收。

另请参阅

更多关于失效化的信息

```py
attribute record_info
```

与此`ManagesConnection`关联的持久信息字典。

与`ManagesConnection.info`字典不同，此字典的生命周期与拥有它的`ConnectionPoolEntry`相同；因此，此字典将在重新连接和特定连接池条目的失效化过程中持续存在。

对于一个未关联`ConnectionPoolEntry`的`PoolProxiedConnection`实例，例如如果它被分离，该属性返回 None。与永不为 None 的`ManagesConnection.info`字典形成对比。

另请参阅

`ManagesConnection.info`

```py
class sqlalchemy.pool.ConnectionPoolEntry
```

代表在`Pool`实例上维护单个数据库连接的对象的接口。

`ConnectionPoolEntry` 对象表示池中特定连接的长期维护，包括使该连接过期或无效以将其替换为新连接，这将继续由同一`ConnectionPoolEntry` 实例维护。与`PoolProxiedConnection` 相比，后者是短期的，每次检出的连接管理器，该对象的寿命为连接池中特定“槽位”的寿命。

当交付给连接池事件钩子时，`ConnectionPoolEntry` 对象主要对公共 API 代码可见，例如`PoolEvents.connect()` 和 `PoolEvents.checkout()`。

2.0 版新功能：`ConnectionPoolEntry` 为`_ConnectionRecord` 内部类提供了公共界面。

**成员**

close(), dbapi_connection, driver_connection, in_use, info, invalidate(), record_info

**类签名**

类`sqlalchemy.pool.ConnectionPoolEntry` (`sqlalchemy.pool.base.ManagesConnection`)

```py
method close() → None
```

关闭此连接池条目管理的 DBAPI 连接。

```py
attribute dbapi_connection: DBAPIConnection | None
```

对正在跟踪的实际 DBAPI 连接的引用。

这是一个[**PEP 249**](https://peps.python.org/pep-0249/)-兼容对象，对于传统的同步样式方言，由使用的第三方 DBAPI 实现提供。对于 asyncio 方言，实现通常是 SQLAlchemy 方言本身提供的适配器对象；基础 asyncio 对象可通过`ManagesConnection.driver_connection` 属性获得。

SQLAlchemy 对 DBAPI 连接的接口基于`DBAPIConnection` 协议对象

另请参阅

`ManagesConnection.driver_connection`

当使用引擎时，如何获取原始的 DBAPI 连接？

```py
attribute driver_connection: Any | None
```

由 Python DBAPI 或数据库驱动程序使用的“驱动程序级”连接对象。

对于传统的[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 实现，该对象将与 `ManagesConnection.dbapi_connection` 相同。对于异步数据库驱动程序，这将是该驱动程序使用的最终“连接”对象，例如 `asyncpg.Connection` 对象，该对象不具有标准的 pep-249 方法。

新版本 1.4.24 中的新增内容。

另请参阅

`ManagesConnection.dbapi_connection`

当使用引擎时，如何获取原始的 DBAPI 连接？

```py
attribute in_use
```

如果连接当前正在被检出，则返回 True

```py
attribute info
```

*继承自* `ManagesConnection.info` *属性*

与此 `ManagesConnection` 实例引用的底层 DBAPI 连接关联的信息字典，允许将用户定义的数据与连接关联起来。

此字典中的数据对于 DBAPI 连接本身的生命周期是持久的，包括池中的检入和检出。当连接被失效并替换为新连接时，该字典将被清除。

对于不与 `ConnectionPoolEntry` 关联的 `PoolProxiedConnection` 实例，例如如果它被分离了，该属性将返回一个局部于该 `ConnectionPoolEntry` 的字典。因此，`ManagesConnection.info` 属性将始终提供一个 Python 字典。

另请参阅

`ManagesConnection.record_info`

```py
method invalidate(e: BaseException | None = None, soft: bool = False) → None
```

*继承自* `ManagesConnection.invalidate()` *方法*

将受管理的连接标记为失效。

参数：

+   `e` – 表示失效原因的异常对象。

+   `soft` – 如果为 True，则不会关闭连接；相反，该连接将在下次检出时被回收。

另请参阅

更多关于失效化的内容

```py
attribute record_info
```

*继承自* `ManagesConnection` *的* `ManagesConnection.record_info` *属性*

与此`ManagesConnection`关联的持久信息字典。

与`ManagesConnection.info`字典不同，此字典的生命周期与拥有它的`ConnectionPoolEntry`相同；因此，对于连接池中特定条目的重新连接和连接失效，此字典将持续存在。

对于未与`ConnectionPoolEntry`关联的`PoolProxiedConnection`实例，例如如果它被分离，该属性返回 None。与永不为 None 的`ManagesConnection.info`字典形成对比。

另请参阅

`ManagesConnection.info`

```py
class sqlalchemy.pool.PoolProxiedConnection
```

用于[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 连接的类似连接适配器，包括特定于`Pool`实现的附加方法。

`PoolProxiedConnection`是内部`_ConnectionFairy`实现对象的公共接口；熟悉`_ConnectionFairy`的用户可以将此对象视为等效。

新版本 2.0 中：`PoolProxiedConnection`提供了`_ConnectionFairy`内部类的公共接口。

**成员**

close(), dbapi_connection, detach(), driver_connection, info, invalidate(), is_detached, is_valid, record_info

**类签名**

类`sqlalchemy.pool.PoolProxiedConnection` (`sqlalchemy.pool.base.ManagesConnection`)

```py
method close() → None
```

将此连接释放回到池中。

`PoolProxiedConnection.close()` 方法覆盖了[**PEP 249**](https://peps.python.org/pep-0249/)的`.close()`方法，改变了其行为，使其释放代理连接返回到连接池。

将连接释放到池中后，连接在 Python 进程中是否保持“打开”并保留在池中，还是实际关闭并从 Python 进程中删除，取决于正在使用的池实现及其配置和当前状态。

```py
attribute dbapi_connection: DBAPIConnection | None
```

对被跟踪的实际 DBAPI 连接的引用。

这是一个[**PEP 249**](https://peps.python.org/pep-0249/)兼容对象，对于传统的同步风格方言，由使用的第三方 DBAPI 实现提供。对于 asyncio 方言，实现通常是 SQLAlchemy 方言本身提供的适配器对象；底层的 asyncio 对象可通过`ManagesConnection.driver_connection`属性访问。

SQLAlchemy 的 DBAPI 连接接口基于`DBAPIConnection`协议对象

另请参阅

`ManagesConnection.driver_connection`

在使用 Engine 时，如何获取原始的 DBAPI 连接？

```py
method detach() → None
```

将此连接与其连接池分离。

这意味着当关闭连接时，连接将不再返回到池中，而是被实际关闭。关联的`ConnectionPoolEntry`与此 DBAPI 连接解除关联。

请注意，在分离后，由池实现强加的任何整体连接限制约束可能会被违反，因为分离的连接从池的知识和控制中移除。

```py
attribute driver_connection: Any | None
```

Python DBAPI 或数据库驱动程序使用的“驱动程序级别”的连接对象。

对于传统的[**PEP 249**](https://peps.python.org/pep-0249/) DBAPI 实现，该对象将与`ManagesConnection.dbapi_connection`的对象相同。对于一个 asyncio 数据库驱动程序，这将是该驱动程序使用的最终的“连接”对象，例如`asyncpg.Connection`对象，它不会具有标准的 pep-249 方法。

版本 1.4.24 中的新功能。

另请参阅

`ManagesConnection.dbapi_connection`

在使用 Engine 时如何获取原始的 DBAPI 连接？

```py
attribute info
```

*继承自* `ManagesConnection` *的* `ManagesConnection.info` *属性*

与此`ManagesConnection`实例引用的底层 DBAPI 连接相关联的信息字典，允许将用户定义的数据与连接关联起来。

这个字典中的数据在整个 DBAPI 连接的生命周期内是持久的，包括连接池的签入和签出。当连接失效并被新连接替换时，该字典将被清除。

对于不与`ConnectionPoolEntry`关联的`PoolProxiedConnection`实例，例如如果它被分离，该属性返回一个仅限于该`ConnectionPoolEntry`的字典。因此，`ManagesConnection.info`属性将始终提供一个 Python 字典。

另请参阅

`ManagesConnection.record_info`

```py
method invalidate(e: BaseException | None = None, soft: bool = False) → None
```

*继承自* `ManagesConnection` *的* `ManagesConnection.invalidate()` *方法*

将托管连接标记为失效。

参数：

+   `e` – 一个表示失效原因的异常对象。

+   `soft` – 如果为 True，则连接不会关闭；相反，此连接将在下次签出时被回收。

另请参阅

更多关于失效的信息

```py
attribute is_detached
```

如果此`PoolProxiedConnection`已从其池中分离，则返回 True。

```py
attribute is_valid
```

如果此`PoolProxiedConnection`仍指向活动的 DBAPI 连接，则返回 True。

```py
attribute record_info
```

*继承自* `ManagesConnection` *的* `ManagesConnection.record_info` *属性*

与此`ManagesConnection`相关联的持久信息字典。

与`ManagesConnection.info`字典不同，此字典的生命周期是由拥有它的`ConnectionPoolEntry`决定的；因此，这个字典将在连接池中的特定条目的重新连接和连接失效时保持不变。

对于未关联到`ConnectionPoolEntry`的`PoolProxiedConnection`实例，例如如果它是分离的，则该属性返回 None。与永远不会为 None 的`ManagesConnection.info`字典相比。

另请参阅

`ManagesConnection.info`

```py
class sqlalchemy.pool._ConnectionFairy
```

代理 DBAPI 连接并提供解引用支持。

这是`Pool`实现内部使用的对象，用于为该`Pool`提供上下文管理，以由该`Pool`提供的 DBAPI 连接。该类的公共接口由`PoolProxiedConnection`类描述。请参阅该类以获取公共 API 详细信息。

名称“fairy”灵感来自于`_ConnectionFairy`对象的生命周期是短暂的，因为它仅在从池中检出的特定 DBAPI 连接的长度内存在，并且作为透明代理，它大部分时间是不可见的。

另请参阅

`PoolProxiedConnection`

`ConnectionPoolEntry`

**类签名**

类 `sqlalchemy.pool._ConnectionFairy` (`sqlalchemy.pool.base.PoolProxiedConnection`)

```py
class sqlalchemy.pool._ConnectionRecord
```

维护连接池中引用池化连接的位置。

这是`Pool`实现内部使用的对象，用于为该`Pool`维护的 DBAPI 连接提供上下文管理。该类的公共接口由`ConnectionPoolEntry`类描述。请参阅该类以获取公共 API 详细信息。

另请参阅

`ConnectionPoolEntry`

`PoolProxiedConnection`

**类签名**

类 `sqlalchemy.pool._ConnectionRecord`（`sqlalchemy.pool.base.ConnectionPoolEntry`）
