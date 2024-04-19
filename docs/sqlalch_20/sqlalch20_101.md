# 核心事件

> 原文：[`docs.sqlalchemy.org/en/20/core/events.html`](https://docs.sqlalchemy.org/en/20/core/events.html)

本节描述了 SQLAlchemy Core 中提供的事件接口。有关事件监听 API 的介绍，请参阅 Events。ORM 事件在 ORM Events 中描述。

| 对象名称 | 描述 |
| --- | --- |
| Events | 为特定目标类型定义事件监听函数。 |

```py
class sqlalchemy.event.base.Events
```

为特定目标类型定义事件监听函数。

**成员**

dispatch

**类签名**

类`sqlalchemy.event.Events` (`sqlalchemy.event._HasEventsDispatch`)

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.EventsDispatch object>
```

参考 _Dispatch class。

对抗 _Dispatch._events

## 连接池事件

| 对象名称 | 描述 |
| --- | --- |
| PoolEvents | `Pool`的可用事件。 |
| PoolResetState | 描述传递给`PoolEvents.reset()`连接池事件的 DBAPI 连接的状态。 |

```py
class sqlalchemy.events.PoolEvents
```

`Pool`的可用事件。

这里的方法定义了事件的名称以及传递给监听器函数的成员的名称。

例如:

```py
from sqlalchemy import event

def my_on_checkout(dbapi_conn, connection_rec, connection_proxy):
    "handle an on checkout event"

event.listen(Pool, 'checkout', my_on_checkout)
```

除了接受`Pool`类和`Pool`实例外，`PoolEvents`还接受`Engine`对象和`Engine`类作为目标，这将被解析为给定引擎的`.pool`属性或`Pool`类：

```py
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

# will associate with engine.pool
event.listen(engine, 'checkout', my_on_checkout)
```

**成员**

checkin(), checkout(), close(), close_detached(), connect(), detach(), dispatch, first_connect(), invalidate(), reset(), soft_invalidate()

**类签名**

类`sqlalchemy.events.PoolEvents` (`sqlalchemy.event.Events`)

```py
method checkin(dbapi_connection: DBAPIConnection | None, connection_record: ConnectionPoolEntry) → None
```

当连接返回到池时调用。

示例参数形式:

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'checkin')
def receive_checkin(dbapi_connection, connection_record):
    "listen for the 'checkin' event"

    # ... (event handling logic) ...
```

请注意，连接可能已关闭，并且如果连接已失效，则可能为 None。对于分离连接，不会调用`checkin`（它们不会返回到池中）。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection`属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
method checkout(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, connection_proxy: PoolProxiedConnection) → None
```

当从连接池中检索到连接时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'checkout')
def receive_checkout(dbapi_connection, connection_record, connection_proxy):
    "listen for the 'checkout' event"

    # ... (event handling logic) ...
```

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection`属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

+   `connection_proxy` – `PoolProxiedConnection`对象，将代理 DBAPI 连接的公共接口，直到检出结束。

如果引发`DisconnectionError`，当前连接将被处理并检索到一个新的连接。所有检出监听器的处理将中止，并使用新连接重新启动。

另请参见

`ConnectionEvents.engine_connect()` - 一个类似的事件，发生在创建新的`Connection`时。

```py
method close(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

当关闭 DBAPI 连接时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'close')
def receive_close(dbapi_connection, connection_record):
    "listen for the 'close' event"

    # ... (event handling logic) ...
```

事件在关闭发生之前发出。

连接关闭可能会失败；通常是因为连接已经关闭。如果关闭操作失败，连接将被丢弃。

`close()`事件对应于仍与池相关联的连接。要拦截分离连接的关闭事件，请使用`close_detached()`。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection`属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
method close_detached(dbapi_connection: DBAPIConnection) → None
```

当分离的 DBAPI 连接关闭时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'close_detached')
def receive_close_detached(dbapi_connection):
    "listen for the 'close_detached' event"

    # ... (event handling logic) ...
```

在关闭发生之前发出事件。

连接的关闭可能失败；通常是因为连接已关闭。如果关闭操作失败，则连接将被丢弃。

参数：

**dbapi_connection** – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

```py
method connect(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

在给定的 `Pool` 中首次创建特定 DBAPI 连接时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'connect')
def receive_connect(dbapi_connection, connection_record):
    "listen for the 'connect' event"

    # ... (event handling logic) ...
```

此事件允许捕获在使用 DBAPI 模块级别的 `.connect()` 方法产生新的 DBAPI 连接之后的直接点。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
method detach(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

当一个 DBAPI 连接从池中“分离”时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'detach')
def receive_detach(dbapi_connection, connection_record):
    "listen for the 'detach' event"

    # ... (event handling logic) ...
```

此事件在分离发生后发出。连接不再与给定的连接记录关联。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.PoolEventsDispatch object>
```

回到 _Dispatch 类的引用。

双向针对 _Dispatch._events

```py
method first_connect(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

当第一次从特定的 `Pool` 中检出 DBAPI 连接时调用一次。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'first_connect')
def receive_first_connect(dbapi_connection, connection_record):
    "listen for the 'first_connect' event"

    # ... (event handling logic) ...
```

`PoolEvents.first_connect()` 的理由是根据所有连接使用的设置确定有关特定系列数据库连接的信息。由于特定的 `Pool` 引用单个“创建者”函数（在 `Engine` 中引用 URL 和连接选项使用），通常可以假定关于单个连接的观察结果对所有后续连接都是有效的，例如数据库版本，服务器和客户端编码设置，排序规则设置等等。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection`属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
method invalidate(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, exception: BaseException | None) → None
```

在要“失效”DBAPI 连接时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'invalidate')
def receive_invalidate(dbapi_connection, connection_record, exception):
    "listen for the 'invalidate' event"

    # ... (event handling logic) ...
```

每当调用`ConnectionPoolEntry.invalidate()`方法时，无论是通过 API 使用还是通过“自动失效”，都会触发此事件，而且没有`soft`标志。

事件发生在对连接调用`.close()`的最终尝试之前。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection`属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

+   `exception` – 对应于此失效原因的异常对象，如果有的话。可能为`None`。

另请参阅

更多关于失效的信息

```py
method reset(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, reset_state: PoolResetState) → None
```

在池化连接发生“重置”操作之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'reset')
def receive_reset(dbapi_connection, connection_record, reset_state):
    "listen for the 'reset' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-2.0, will be removed in a future release)
@event.listens_for(SomeEngineOrPool, 'reset')
def receive_reset(dbapi_connection, connection_record):
    "listen for the 'reset' event"

    # ... (event handling logic) ...
```

在 2.0 版本中更改：`PoolEvents.reset()`事件现在接受参数`PoolEvents.reset.dbapi_connection`, `PoolEvents.reset.connection_record`, `PoolEvents.reset.reset_state`。支持接受先前参数签名的监听器函数将在将来的版本中删除。

此事件表示在将 DBAPI 连接返回到池中或丢弃之前调用`rollback()`方法时发生。可以使用此事件钩子实现自定义“重置”策略，也可以结合使用`Pool.reset_on_return`参数禁用默认的“重置”行为。

`PoolEvents.reset()` 和 `PoolEvents.checkin()` 事件的主要区别在于 `PoolEvents.reset()` 不仅适用于将返回池的池化连接，还适用于使用 `Connection.detach()` 方法分离的连接以及由于连接在被检入之前发生垃圾回收而被丢弃的 asyncio 连接。

请注意，**不会**为使用 `Connection.invalidate()` 使无效的连接调用此事件。这些事件可以通过 `PoolEvents.soft_invalidate()` 和 `PoolEvents.invalidate()` 事件钩子拦截，并且所有“连接关闭”事件可以通过 `PoolEvents.close()` 拦截。

`PoolEvents.reset()` 事件通常紧跟着 `PoolEvents.checkin()` 事件，在连接在重置后立即被丢弃的情况下除外。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的 `ConnectionPoolEntry`。

+   `reset_state` –

    `PoolResetState` 实例，提供有关正在重置连接的情况的信息。

    从版本 2.0 开始新增。

另请参见

返回时重置

`ConnectionEvents.rollback()`

`ConnectionEvents.commit()`

```py
method soft_invalidate(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, exception: BaseException | None) → None
```

当要“软使无效” DBAPI 连接时调用此事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'soft_invalidate')
def receive_soft_invalidate(dbapi_connection, connection_record, exception):
    "listen for the 'soft_invalidate' event"

    # ... (event handling logic) ...
```

每当使用 `soft` 标志调用 `ConnectionPoolEntry.invalidate()` 方法时，都会调用此事件。

软失效是指跟踪此连接的连接记录将在当前连接签入后强制重新连接。它不会在调用它的点主动关闭 dbapi_connection。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

+   `exception` – 无效原因对应的异常对象，如果没有则可能是 `None`。

```py
class sqlalchemy.events.PoolResetState
```

描述 DBAPI 连接在传递给`PoolEvents.reset()`连接池事件时的状态。

**成员**

asyncio_safe, terminate_only, transaction_was_reset

在版本 2.0.0b3 中新增。

```py
attribute asyncio_safe: bool
```

指示重置操作是否发生在期望在 asyncio 应用程序中存在的封闭事件循环范围内。

如果连接正在被垃圾回收，则为 False。

```py
attribute terminate_only: bool
```

指示连接是否立即终止并且不被签入到池中。

这发生在失效的连接以及未被调用代码清理地处理的 asyncio 连接，而是被垃圾回收时。在后一种情况下，不能在垃圾回收中安全地运行 asyncio 连接上的操作，因为不一定存在事件循环。

```py
attribute transaction_was_reset: bool
```

指示 DBAPI 连接上的事务是否已经由`Connection`对象实质上“重置”。

如果`Connection`上有事务状态，并且然后没有使用`Connection.rollback()`或`Connection.commit()`方法关闭事务；相反，事务在`Connection.close()`方法内联关闭，因此在到达此事件时保证保持不存在。

## SQL 执行和连接事件

| 对象名称 | 描述 |
| --- | --- |
| ConnectionEvents | `Connection`和`Engine`的可用事件。 |
| 方言事件 | 用于执行替换函数的事件接口。 |

```py
class sqlalchemy.events.ConnectionEvents
```

`Connection`和`Engine`的可用事件。

这里的方法定义了事件的名称以及传递给监听器函数的成员的名称。

事件监听器可以与任何`Connection`或`Engine`类或实例相关联，例如一个`Engine`，例如：

```py
from sqlalchemy import event, create_engine

def before_cursor_execute(conn, cursor, statement, parameters, context,
                                                executemany):
    log.info("Received statement: %s", statement)

engine = create_engine('postgresql+psycopg2://scott:tiger@localhost/test')
event.listen(engine, "before_cursor_execute", before_cursor_execute)
```

或者使用特定的`Connection`:

```py
with engine.begin() as conn:
    @event.listens_for(conn, 'before_cursor_execute')
    def before_cursor_execute(conn, cursor, statement, parameters,
                                    context, executemany):
        log.info("Received statement: %s", statement)
```

当方法使用语句参数调用时，例如在`after_cursor_execute()`或`before_cursor_execute()`中，语句是准备发送到连接的 DBAPI `cursor`的确切 SQL 字符串`Dialect`。 

`before_execute()`和`before_cursor_execute()`事件也可以使用`retval=True`标志来建立，这允许修改发送到数据库的语句和参数。`before_cursor_execute()`事件在这里特别有用，可以添加临时字符串转换，例如注释，以适用于所有执行：

```py
from sqlalchemy.engine import Engine
from sqlalchemy import event

@event.listens_for(Engine, "before_cursor_execute", retval=True)
def comment_sql_calls(conn, cursor, statement, parameters,
                                    context, executemany):
    statement = statement + " -- some comment"
    return statement, parameters
```

注意

`ConnectionEvents` 可以建立在任何组合的 `Engine`、`Connection`，以及这些类的实例上。对于给定的 `Connection` 实例，所有四个范围的事件都会触发。但是，出于性能原因，`Connection` 对象在实例化时确定其父 `Engine` 是否已经建立了事件侦听器。在依赖的 `Connection` 实例实例化后，向 `Engine` 类或 `Engine` 实例添加的事件侦听器通常不会对该 `Connection` 实例可用。而是，新添加的侦听器将对在父 `Engine` 类或实例上建立这些事件侦听器之后创建的 `Connection` 实例产生影响。

参数：

**retval=False** – 仅适用于 `before_execute()` 和 `before_cursor_execute()` 事件。当为 True 时，用户定义的事件函数必须有一个返回值，即替换给定语句和参数的参数元组。有关特定返回参数的描述，请参见这些方法。

**成员**

after_cursor_execute(), after_execute(), before_cursor_execute(), before_execute(), begin(), begin_twophase(), commit(), commit_twophase(), dispatch, engine_connect(), engine_disposed(), prepare_twophase(), release_savepoint(), rollback(), rollback_savepoint(), rollback_twophase(), savepoint(), set_connection_execution_options(), set_engine_execution_options()

**类签名**

类`sqlalchemy.events.ConnectionEvents` (`sqlalchemy.event.Events`)

```py
method after_cursor_execute(conn: Connection, cursor: DBAPICursor, statement: str, parameters: _DBAPIAnyExecuteParams, context: ExecutionContext | None, executemany: bool) → None
```

在执行后拦截低级游标 execute() 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'after_cursor_execute')
def receive_after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    "listen for the 'after_cursor_execute' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `cursor` – DBAPI 游标对象。如果语句是一个 SELECT，将会有待处理的结果，但不应该消耗这些结果，因为它们会被`CursorResult`需要。

+   `statement` – 字符串 SQL 语句，就像传递给 DBAPI 的一样

+   `parameters` – 字典、元组或传递给 DBAPI 游标的`execute()`或`executemany()`方法的参数列表。在某些情况下可能为`None`。

+   `context` – 使用的 `ExecutionContext` 对象。可能为 `None`。

+   `executemany` – 布尔值，如果为 `True`，则这是一个 `executemany()` 调用，如果为 `False`，则这是一个 `execute()` 调用。

```py
method after_execute(conn: Connection, clauseelement: Executable, multiparams: _CoreMultiExecuteParams, params: _CoreSingleExecuteParams, execution_options: _ExecuteOptions, result: Result[Any]) → None
```

在执行后拦截高级 execute() 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'after_execute')
def receive_after_execute(conn, clauseelement, multiparams, params, execution_options, result):
    "listen for the 'after_execute' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-1.4, will be removed in a future release)
@event.listens_for(SomeEngine, 'after_execute')
def receive_after_execute(conn, clauseelement, multiparams, params, result):
    "listen for the 'after_execute' event"

    # ... (event handling logic) ...
```

1.4 版本变更：`ConnectionEvents.after_execute()` 事件现在接受参数 `ConnectionEvents.after_execute.conn`, `ConnectionEvents.after_execute.clauseelement`, `ConnectionEvents.after_execute.multiparams`, `ConnectionEvents.after_execute.params`, `ConnectionEvents.after_execute.execution_options`, `ConnectionEvents.after_execute.result`。将来版本将删除对接受前述“已弃用”参数签名的监听器函数的支持。

参数：

+   `conn` – `Connection` 对象

+   `clauseelement` – SQL 表达式构造，`Compiled` 实例或传递给 `Connection.execute()` 的字符串语句。

+   `multiparams` – 多个参数集，一个字典列表。

+   `params` – 单个参数集，一个字典。

+   `execution_options` –

    传递给语句的执行选项字典，如果有的话。这是将要使用的所有选项的合并，包括语句的选项、连接的选项以及传递给方法本身的用于执行 2.0 风格的选项。

+   `result` – 执行生成的 `CursorResult`。

```py
method before_cursor_execute(conn: Connection, cursor: DBAPICursor, statement: str, parameters: _DBAPIAnyExecuteParams, context: ExecutionContext | None, executemany: bool) → Tuple[str, _DBAPIAnyExecuteParams] | None
```

在执行之前拦截低级别游标 execute() 事件，接收要针对游标调用的字符串 SQL 语句和特定于 DBAPI 的参数列表。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'before_cursor_execute')
def receive_before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    "listen for the 'before_cursor_execute' event"

    # ... (event handling logic) ...
```

此事件既可用于记录，也可用于对 SQL 字符串进行后期修改。对于除了特定于目标后端的参数修改之外的参数修改，它不太理想。

可以选择使用 `retval=True` 标志建立此事件。在这种情况下，应返回 `statement` 和 `parameters` 参数作为两个元组：

```py
@event.listens_for(Engine, "before_cursor_execute", retval=True)
def before_cursor_execute(conn, cursor, statement,
                parameters, context, executemany):
    # do something with statement, parameters
    return statement, parameters
```

参见 `ConnectionEvents` 中的示例。

参数：

+   `conn` – `Connection` 对象

+   `cursor` – DBAPI 游标对象

+   `statement` – 字符串 SQL 语句，如传递给 DBAPI

+   `parameters` – 字典、元组或传递给 DBAPI `cursor` 的 `execute()` 或 `executemany()` 方法的参数列表。在某些情况下可能为 `None`。

+   `context` – 正在使用的 `ExecutionContext` 对象。可能为 `None`。

+   `executemany` – 布尔值，如果为 `True`，则为 `executemany()` 调用，如果为 `False`，则为 `execute()` 调用。

另请参阅

`before_execute()`

`after_cursor_execute()`

```py
method before_execute(conn: Connection, clauseelement: Executable, multiparams: _CoreMultiExecuteParams, params: _CoreSingleExecuteParams, execution_options: _ExecuteOptions) → Tuple[Executable, _CoreMultiExecuteParams, _CoreSingleExecuteParams] | None
```

拦截高级别的 `execute()` 事件，在渲染为 SQL 之前接收未编译的 SQL 构造和其他对象。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'before_execute')
def receive_before_execute(conn, clauseelement, multiparams, params, execution_options):
    "listen for the 'before_execute' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-1.4, will be removed in a future release)
@event.listens_for(SomeEngine, 'before_execute')
def receive_before_execute(conn, clauseelement, multiparams, params):
    "listen for the 'before_execute' event"

    # ... (event handling logic) ...
```

在 1.4 版本中更改：`ConnectionEvents.before_execute()` 事件现在接受参数 `ConnectionEvents.before_execute.conn`、`ConnectionEvents.before_execute.clauseelement`、`ConnectionEvents.before_execute.multiparams`、`ConnectionEvents.before_execute.params`、`ConnectionEvents.before_execute.execution_options` 的支持。在未来版本中，将删除接受前述“已弃用”参数签名的侦听器函数的支持。

此事件对于调试 SQL 编译问题以及数据库发送的参数的早期操作非常有用，因为此处的参数列表将以一致的格式呈现。

此事件可以选择使用 `retval=True` 标志来建立。在这种情况下，应将 `clauseelement`、`multiparams` 和 `params` 参数作为三元组返回：

```py
@event.listens_for(Engine, "before_execute", retval=True)
def before_execute(conn, clauseelement, multiparams, params):
    # do something with clauseelement, multiparams, params
    return clauseelement, multiparams, params
```

参数：

+   `conn` – `Connection` 对象

+   `clauseelement` – SQL 表达式构造，`Compiled` 实例，或传递给 `Connection.execute()` 的字符串语句。

+   `multiparams` – 多个参数集，字典列表。

+   `params` – 单个参数集，一个字典。

+   `execution_options` –

    执行选项字典，与语句一起传递，如果有的话。这是将要使用的所有选项的合并，包括语句的选项、连接的选项以及传递给方法本身的选项，用于 2.0 风格的执行。

也请参阅

`before_cursor_execute()`

```py
method begin(conn: Connection) → None
```

拦截 `begin()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'begin')
def receive_begin(conn):
    "listen for the 'begin' event"

    # ... (event handling logic) ...
```

参数：

**conn** – `Connection` 对象

```py
method begin_twophase(conn: Connection, xid: Any) → None
```

拦截 `begin_twophase()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'begin_twophase')
def receive_begin_twophase(conn, xid):
    "listen for the 'begin_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

```py
method commit(conn: Connection) → None
```

拦截由 `Transaction` 发起的提交事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'commit')
def receive_commit(conn):
    "listen for the 'commit' event"

    # ... (event handling logic) ...
```

注意，如果 `reset_on_return` 标志设置为 `'commit'`，则 `Pool` 也可能在归还时 “自动提交” DBAPI 连接。要拦截此提交，请使用 `PoolEvents.reset()` 钩子。

参数：

**conn** – `Connection` 对象

```py
method commit_twophase(conn: Connection, xid: Any, is_prepared: bool) → None
```

拦截 `commit_twophase()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'commit_twophase')
def receive_commit_twophase(conn, xid, is_prepared):
    "listen for the 'commit_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

+   `is_prepared` – 布尔值，指示是否调用了 `TwoPhaseTransaction.prepare()`。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.ConnectionEventsDispatch object>
```

回溯到 _Dispatch 类。

双向 _Dispatch._events

```py
method engine_connect(conn: Connection) → None
```

拦截新建 `Connection`。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'engine_connect')
def receive_engine_connect(conn):
    "listen for the 'engine_connect' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-2.0, will be removed in a future release)
@event.listens_for(SomeEngine, 'engine_connect')
def receive_engine_connect(conn, branch):
    "listen for the 'engine_connect' event"

    # ... (event handling logic) ...
```

从版本 2.0 开始更改：`ConnectionEvents.engine_connect()` 事件现在接受参数 `ConnectionEvents.engine_connect.conn`。将来的版本中将删除对接受上述“弃用”之前参数签名的侦听器函数的支持。

通常，此事件是直接调用 `Engine.connect()` 方法的直接结果。

它与 `PoolEvents.connect()` 方法不同，后者是指在 DBAPI 级别对数据库的实际连接；DBAPI 连接可能会被池化并重复使用多次。相比之下，此事件仅与在此类 DBAPI 连接周围生成更高级别的 `Connection` 包装器有关。

它还与 `PoolEvents.checkout()` 事件不同，后者特定于 `Connection` 对象，而不是 `PoolEvents.checkout()` 处理的 DBAPI 连接，尽管该 DBAPI 连接可以通过 `Connection.connection` 属性在此处获得。但请注意，如果 `Connection` 无效并重新建立，则单个 `Connection` 对象的生命周期中实际上可以有多个 `PoolEvents.checkout()` 事件。

参数：

**conn** – `Connection` 对象。

另请参阅

`PoolEvents.checkout()` 单个 DBAPI 连接的低级别池检出事件

```py
method engine_disposed(engine: Engine) → None
```

拦截 `Engine.dispose()` 方法被调用的情况。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'engine_disposed')
def receive_engine_disposed(engine):
    "listen for the 'engine_disposed' event"

    # ... (event handling logic) ...
```

`Engine.dispose()` 方法指示引擎“处理”它的连接池（例如 `Pool`），并用新的连接池替换它。处理旧连接池的效果是关闭现有的已检入连接。新连接池在首次使用之前不会建立任何新连接。

这个事件可用于指示应该清理与 `Engine` 相关的资源，但要注意，`Engine` 仍然可以用于新请求，此时它将重新获取连接资源。

```py
method prepare_twophase(conn: Connection, xid: Any) → None
```

拦截 `prepare_twophase()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'prepare_twophase')
def receive_prepare_twophase(conn, xid):
    "listen for the 'prepare_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

```py
method release_savepoint(conn: Connection, name: str, context: None) → None
```

拦截 `release_savepoint()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'release_savepoint')
def receive_release_savepoint(conn, name, context):
    "listen for the 'release_savepoint' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `name` – 用于保存点的指定名称。

+   `context` – 未使用

```py
method rollback(conn: Connection) → None
```

拦截由 `Transaction` 启动的 `rollback()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'rollback')
def receive_rollback(conn):
    "listen for the 'rollback' event"

    # ... (event handling logic) ...
```

请注意，如果 `reset_on_return` 标志设置为其默认值 `'rollback'`，`Pool` 在归还时也会“自动回滚” DBAPI 连接。要拦截此回滚，请使用 `PoolEvents.reset()` 钩子。

参数：

**conn** – `Connection` 对象

另请参阅

`PoolEvents.reset()`

```py
method rollback_savepoint(conn: Connection, name: str, context: None) → None
```

拦截 `rollback_savepoint()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'rollback_savepoint')
def receive_rollback_savepoint(conn, name, context):
    "listen for the 'rollback_savepoint' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `name` – 用于保存点的指定名称。

+   `context` – 未使用

```py
method rollback_twophase(conn: Connection, xid: Any, is_prepared: bool) → None
```

拦截 `rollback_twophase()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'rollback_twophase')
def receive_rollback_twophase(conn, xid, is_prepared):
    "listen for the 'rollback_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

+   `is_prepared` – 布尔值，指示是否调用了 `TwoPhaseTransaction.prepare()`。

```py
method savepoint(conn: Connection, name: str) → None
```

拦截 `savepoint()` 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'savepoint')
def receive_savepoint(conn, name):
    "listen for the 'savepoint' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `name` – 用于保存点的指定名称。

```py
method set_connection_execution_options(conn: Connection, opts: Dict[str, Any]) → None
```

拦截 `Connection.execution_options()` 方法调用时。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'set_connection_execution_options')
def receive_set_connection_execution_options(conn, opts):
    "listen for the 'set_connection_execution_options' event"

    # ... (event handling logic) ...
```

此方法在新的 `Connection` 生成后调用，具有新更新的执行选项集，但在 `Dialect` 对这些新选项之前。

请注意，当从其父`Engine`继承执行选项的新`Connection`被生成时，不会调用此方法；要拦截此条件，请使用`ConnectionEvents.engine_connect()`事件。

参数：

+   `conn` – 新复制的`Connection`对象

+   `opts` –

    传递给`Connection.execution_options()`方法的选项字典。此字典可以就地修改，以影响最终生效的选项。

    版本 2.0 中的新内容：`opts`字典可以就地修改。

另请参阅

`ConnectionEvents.set_engine_execution_options()` - 当调用`Engine.execution_options()`时调用的事件。

```py
method set_engine_execution_options(engine: Engine, opts: Dict[str, Any]) → None
```

当调用`Engine.execution_options()`方法时拦截。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'set_engine_execution_options')
def receive_set_engine_execution_options(engine, opts):
    "listen for the 'set_engine_execution_options' event"

    # ... (event handling logic) ...
```

`Engine.execution_options()`方法生成`Engine`的浅拷贝，其中存储了新的选项。这个新的`Engine`被传递到这里。此方法的一个特定应用是将`ConnectionEvents.engine_connect()`事件处理程序添加到给定的`Engine`上，该处理程序将执行一些针对这些执行选项的特定于每个`Connection`的任务。

参数：

+   `conn` – 新复制的`Engine`对象

+   `opts` –

    传递给`Connection.execution_options()`方法的选项字典。此字典可以就地修改，以影响最终生效的选项。

    版本 2.0 中的新内容：`opts`字典可以就地修改。

另请参阅

`ConnectionEvents.set_connection_execution_options()` - 当调用`Connection.execution_options()`时调用的事件。

```py
class sqlalchemy.events.DialectEvents
```

用于执行替换函数的事件接口。

这些事件允许直接检测和替换与 DBAPI 交互的关键方言函数。

注意

`DialectEvents` 钩子应被视为**半公开**和实验性质的。这些钩子不适用于一般情况，并且仅适用于那些需要将复杂的 DBAPI 机制重新注入到现有方言中的情况。对于一般用途的语句拦截事件，请使用`ConnectionEvents` 接口。

另请参阅

`ConnectionEvents.before_cursor_execute()`

`ConnectionEvents.before_execute()`

`ConnectionEvents.after_cursor_execute()`

`ConnectionEvents.after_execute()`

**成员**

dispatch, do_connect(), do_execute(), do_execute_no_params(), do_executemany(), do_setinputsizes(), handle_error()

**类签名**

类`sqlalchemy.events.DialectEvents` (`sqlalchemy.event.Events`) 

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.DialectEventsDispatch object>
```

参考 _Dispatch 类。

双向对 _Dispatch._events

```py
method do_connect(dialect: Dialect, conn_rec: ConnectionPoolEntry, cargs: Tuple[Any, ...], cparams: Dict[str, Any]) → DBAPIConnection | None
```

在建立连接之前接收连接参数。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_connect')
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    "listen for the 'do_connect' event"

    # ... (event handling logic) ...
```

这个事件很有用，因为它允许处理程序操纵控制 DBAPI `connect()` 函数将如何调用的 `cargs` 和/或 `cparams` 集合。`cargs`将始终是一个可以原位变异的 Python 列表，而`cparams`是一个也可以变异的 Python 字典：

```py
e = create_engine("postgresql+psycopg2://user@host/dbname")

@event.listens_for(e, 'do_connect')
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    cparams["password"] = "some_password"
```

事件钩子也可用于完全覆盖`connect()`的调用，方法是返回一个非`None`的 DBAPI 连接对象：

```py
e = create_engine("postgresql+psycopg2://user@host/dbname")

@event.listens_for(e, 'do_connect')
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    return psycopg2.connect(*cargs, **cparams)
```

另请参阅

自定义 DBAPI connect() 参数 / 在连接时运行的程序

```py
method do_execute(cursor: DBAPICursor, statement: str, parameters: _DBAPISingleExecuteParams, context: ExecutionContext) → Literal[True] | None
```

接收一个游标以调用 execute()。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_execute')
def receive_do_execute(cursor, statement, parameters, context):
    "listen for the 'do_execute' event"

    # ... (event handling logic) ...
```

返回 True 值以阻止进一步调用事件，并指示游标执行已在事件处理程序中发生。

```py
method do_execute_no_params(cursor: DBAPICursor, statement: str, context: ExecutionContext) → Literal[True] | None
```

接收一个游标以调用没有参数的 execute()。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_execute_no_params')
def receive_do_execute_no_params(cursor, statement, context):
    "listen for the 'do_execute_no_params' event"

    # ... (event handling logic) ...
```

返回 True 值以阻止进一步调用事件，并指示游标执行已在事件处理程序中发生。

```py
method do_executemany(cursor: DBAPICursor, statement: str, parameters: _DBAPIMultiExecuteParams, context: ExecutionContext) → Literal[True] | None
```

接收一个游标以调用 executemany()。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_executemany')
def receive_do_executemany(cursor, statement, parameters, context):
    "listen for the 'do_executemany' event"

    # ... (event handling logic) ...
```

返回 True 值以阻止进一步调用事件，并指示游标执行已在事件处理程序中发生。

```py
method do_setinputsizes(inputsizes: Dict[BindParameter[Any], Any], cursor: DBAPICursor, statement: str, parameters: _DBAPIAnyExecuteParams, context: ExecutionContext) → None
```

接收可供修改的 setinputsizes 字典。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_setinputsizes')
def receive_do_setinputsizes(inputsizes, cursor, statement, parameters, context):
    "listen for the 'do_setinputsizes' event"

    # ... (event handling logic) ...
```

当方言使用 DBAPI `cursor.setinputsizes()` 方法传递关于特定语句参数绑定的信息时，会触发此事件。给定的 `inputsizes` 字典将包含`BindParameter` 对象作为键，链接到特定于 DBAPI 的类型对象作为值；对于未绑定的参数，它们将以 `None` 作为值添加到字典中，这意味着该参数将不会包含在最终的 setinputsizes 调用中。该事件可用于检查和/或记录被绑定的数据类型，并直接修改字典。可以向该字典中添加、修改或删除参数。调用者通常希望检查给定绑定对象的 `BindParameter.type` 属性，以便对 DBAPI 对象做出决策。

事件之后，`inputsizes` 字典将转换为适当的数据结构以传递给 `cursor.setinputsizes`；对于位置绑定参数执行样式，转换为列表；对于命名绑定参数执行样式，转换为字符串参数键到 DBAPI 类型对象的字典。

setinputsizes 钩子整体上仅用于包含标志 `use_setinputsizes=True` 的方言。使用此标志的方言包括 cx_Oracle、pg8000、asyncpg 和 pyodbc 方言。

注意

与 pyodbc 一起使用时，必须向方言传递 `use_setinputsizes` 标志，例如：

```py
create_engine("mssql+pyodbc://...", use_setinputsizes=True)
```

另请参阅

Setinputsizes 支持

新版本 1.2.9 中新增。

另请参阅

使用 setinputsizes 对 cx_Oracle 数据绑定性能进行细粒度控制

```py
method handle_error(exception_context: ExceptionContext) → BaseException | None
```

拦截`Dialect` 典型但不限于在 `Connection` 范围内发出的异常。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'handle_error')
def receive_handle_error(exception_context):
    "listen for the 'handle_error' event"

    # ... (event handling logic) ...
```

从版本 2.0 开始更改：`DialectEvents.handle_error()`事件已移至`DialectEvents`类中，从`ConnectionEvents`类中移除，以便它还可以参与使用`create_engine.pool_pre_ping`参数配置的“预连接”操作。该事件仍通过使用`Engine`作为事件目标来注册，但请注意，不再支持将`Connection`用作`DialectEvents.handle_error()`的事件目标。

这包括由 DBAPI 发出的所有异常，以及 SQLAlchemy 语句调用过程中的其他区域，包括编码错误和其他语句验证错误。调用事件的其他区域包括事务开始和结束、结果行获取、游标创建。

请注意，`handle_error()`可能随时支持新类型的异常和新的调用场景。使用此事件的代码必须预期在次要版本中存在新的调用模式。

为了支持对应于异常的广泛成员的各种情况，并允许在不向后兼容的情况下扩展事件，所接收的唯一参数是一个`ExceptionContext`的实例。此对象包含表示异常详细信息的数据成员。

此钩子支持的用例包括：

+   仅用于日志记录和调试目的的只读低级别异常处理

+   建立 DBAPI 连接错误消息是否指示需要重新连接数据库连接，包括某些方言使用的“pre_ping”处理程序

+   在响应特定异常时建立或禁用连接或拥有连接池是否无效或过期

+   异常重写

在失败的操作的游标（如果有）仍处于打开和可访问状态时调用该钩子。可以在此游标上调用特殊的清理操作；SQLAlchemy 将尝试在调用此钩子后关闭此游标。

从 SQLAlchemy 2.0 开始，使用 `create_engine.pool_pre_ping` 参数启用的“pre_ping”处理程序也将参与 `handle_error()` 过程，对于那些依赖断开连接代码来检测数据库活动性的方言。请注意，一些方言，如 psycopg、psycopg2 和大多数 MySQL 方言，使用由 DBAPI 提供的本地 `ping()` 方法，该方法不使用断开连接代码。

在版本 2.0.0 中进行了更改：`DialectEvents.handle_error()` 事件钩子参与连接池“预先 ping”操作。在此使用中，`ExceptionContext.engine` 属性将为 `None`，但是正在使用的 `Dialect` 可通过 `ExceptionContext.dialect` 属性始终可用。

在版本 2.0.5 中进行了更改：添加了 `ExceptionContext.is_pre_ping` 属性，当在连接池的“预先 ping”操作中触发 `DialectEvents.handle_error()` 事件钩子时，该属性将设置为 `True`。

在版本 2.0.5 中进行了更改：修复了一个问题，允许 PostgreSQL `psycopg` 和 `psycopg2` 驱动程序以及所有 MySQL 驱动程序在连接池“预先 ping”操作期间正确参与 `DialectEvents.handle_error()` 事件钩子；此前，这些驱动程序的实现对这些驱动程序而言是无效的。

处理程序函数有两个选项来将 SQLAlchemy 构造的异常替换为用户定义的异常。它可以直接引发此新异常，此时所有后续事件监听器都将被绕过，并且异常将在适当的清理完成后被引发：

```py
@event.listens_for(Engine, "handle_error")
def handle_exception(context):
    if isinstance(context.original_exception,
        psycopg2.OperationalError) and \
        "failed" in str(context.original_exception):
        raise MySpecialException("failed operation")
```

警告

因为 `DialectEvents.handle_error()` 事件专门提供了将异常重新抛出为失败语句引发的最终异常的方法，如果用户定义的事件处理程序本身失败并引发意外异常，则堆栈跟踪将会误导！建议在这里小心编码，并在发生意外异常时使用日志记录和/或内联调试。

或者，可以使用“链接”样式的事件处理，通过使用`retval=True`修饰符配置处理程序，并从函数返回新的异常实例来使用。在这种情况下，事件处理将继续到下一个处理程序。可以使用`ExceptionContext.chained_exception`获取“链接”异常：

```py
@event.listens_for(Engine, "handle_error", retval=True)
def handle_exception(context):
    if context.chained_exception is not None and \
        "special" in context.chained_exception.message:
        return MySpecialException("failed",
            cause=context.chained_exception)
```

返回`None`的处理程序可以在链中使用；当处理程序返回`None`时，如果有的话，前一个异常实例将保持为传递给下一个处理程序的当前异常。

当引发或返回自定义异常时，SQLAlchemy 将直接引发此新异常，不会被任何 SQLAlchemy 对象包装。如果异常不是`sqlalchemy.exc.StatementError`的子类，某些功能可能不可用；目前包括 ORM 在自动刷新过程中引发异常时添加有关“自动刷新”的详细提示的功能。

参数：

**context** – 一个`ExceptionContext`对象。有关所有可用成员的详细信息，请参阅此类。

另请参阅

支持断开场景下的新数据库错误代码

## 模式事件

| 对象名称 | 描述 |
| --- | --- |
| DDLEvents | 为模式对象定义事件监听器，即`SchemaItem`和其他`SchemaEventTarget`子类，包括`MetaData`、`Table`、`Column`等。 |
| SchemaEventTarget | 用于`DDLEvents`事件的目标元素的基类。 |

```py
class sqlalchemy.events.DDLEvents
```

为模式对象定义事件监听器，即`SchemaItem`和其他`SchemaEventTarget`子类，包括`MetaData`、`Table`、`Column`等。

**创建/删除事件**

当 CREATE 和 DROP 命令发送到数据库时发出的事件。此类别中的事件挂钩包括`DDLEvents.before_create()`，`DDLEvents.after_create()`，`DDLEvents.before_drop()`和`DDLEvents.after_drop()`。

当使用模式级方法（例如`MetaData.create_all()`和`MetaData.drop_all()`）时，会发出这些事件。还包括每个对象的 create/drop 方法，如`Table.create()`，`Table.drop()`，`Index.create()`，以及特定于方言的方法，如`ENUM.create()`。

新功能 2.0 版中：`DDLEvents`事件挂钩现在适用于非表对象，包括约束、索引和特定于方言的模式类型。

事件挂钩可以直接附加到`Table`对象或`MetaData`集合，以及任何可通过单独的 SQL 命令创建和删除的`SchemaItem`类或对象。此类包括`Index`，`Sequence`以及特定于方言的类，例如`ENUM`。

使用`DDLEvents.after_create()`事件的示例，其中自定义事件挂钩将在发送`CREATE TABLE`后在当前连接上发出`ALTER TABLE`命令：

```py
from sqlalchemy import create_engine
from sqlalchemy import event
from sqlalchemy import Table, Column, Metadata, Integer

m = MetaData()
some_table = Table('some_table', m, Column('data', Integer))

@event.listens_for(some_table, "after_create")
def after_create(target, connection, **kw):
    connection.execute(text(
        "ALTER TABLE %s SET name=foo_%s" % (target.name, target.name)
    ))

some_engine = create_engine("postgresql://scott:tiger@host/test")

# will emit "CREATE TABLE some_table" as well as the above
# "ALTER TABLE" statement afterwards
m.create_all(some_engine)
```

约束对象，如`ForeignKeyConstraint`、`UniqueConstraint`、`CheckConstraint`，也可以订阅这些事件，但通常不会产生事件，因为这些对象通常是内联渲染在一个包含的`CREATE TABLE`语句中，并且在`DROP TABLE`语句中隐式地被删除。

对于`Index`构造，事件钩子将被触发为`CREATE INDEX`，但是当删除表时，SQLAlchemy 通常不会发出`DROP INDEX`，因为这再次隐含在`DROP TABLE`语句中。

新版本 2.0 中：支持`SchemaItem`对象的创建/删除事件从其先前对`MetaData`和`Table`的支持扩展到还包括`Constraint`和所有子类、`Index`、`Sequence`以及一些与类型相关的构造，比如`ENUM`。

注意

这些事件钩子仅在 SQLAlchemy 的创建/删除方法范围内触发；它们不一定受到诸如[alembic](https://alembic.sqlalchemy.org)之类的工具的支持。

**附加事件**

附加事件提供了自定义行为的机会，每当一个子模式元素与父元素相关联时，比如当一个`Column`与其`Table`相关联时，当一个`ForeignKeyConstraint`与一个`Table`相关联时等。这些事件包括`DDLEvents.before_parent_attach()`和`DDLEvents.after_parent_attach()`。

**反射事件**

`DDLEvents.column_reflect()`事件用于拦截和修改数据库表反射进行时的数据库列的 Python 中定义。

**与通用 DDL 一起使用**

DDL 事件与`DDL`类和 DDL 子句结构的`ExecutableDDLElement`层次密切集成，它们本身适合作为侦听器可调用：

```py
from sqlalchemy import DDL
event.listen(
    some_table,
    "after_create",
    DDL("ALTER TABLE %(table)s SET name=foo_%(table)s")
)
```

**事件传播到 MetaData 复制**

对于所有`DDLEvent`事件，`propagate=True`关键字参数将确保给定的事件处理程序传播到对象的副本中，当使用`Table.to_metadata()`方法时会生成这些副本：

```py
from sqlalchemy import DDL

metadata = MetaData()
some_table = Table("some_table", metadata, Column("data", Integer))

event.listen(
    some_table,
    "after_create",
    DDL("ALTER TABLE %(table)s SET name=foo_%(table)s"),
    propagate=True
)

new_metadata = MetaData()
new_table = some_table.to_metadata(new_metadata)
```

上述`DDL`对象将与`some_table`和`new_table``Table`对象的`DDLEvents.after_create()`事件相关联。

另请参阅

事件

`ExecutableDDLElement`

`DDL`

控制 DDL 序列

**成员**

after_create()、after_drop()、after_parent_attach()、before_create()、before_drop()、before_parent_attach()、column_reflect()、dispatch

**类签名**

类`sqlalchemy.events.DDLEvents`（`sqlalchemy.event.Events`）

```py
method after_create(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在发出 CREATE 语句后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'after_create')
def receive_after_create(target, connection, **kw):
    "listen for the 'after_create' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    事件目标，如`MetaData`或`Table`，但也包括所有创建/删除对象，如`Index`、`Sequence`等。

    版本 2.0 新增：添加对所有`SchemaItem`对象的支持。

+   `connection` – 发出 CREATE 语句或语句的`Connection`。

+   `**kw` – 与事件相关的附加关键字参数。此字典的内容可能会在不同版本之间变化，并包括在元数据级事件中生成的表列表、checkfirst 标志以及内部事件使用的其他元素。

`listen()` 还接受`propagate=True`修饰符，用于此事件；当为 True 时，监听器函数将为目标对象的任何副本建立，即在使用`Table.to_metadata()` 生成的那些副本。

```py
method after_drop(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在发出 DROP 语句后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'after_drop')
def receive_after_drop(target, connection, **kw):
    "listen for the 'after_drop' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    事件目标的`SchemaObject`，例如`MetaData`或`Table`，但也包括所有 create/drop 对象，如`Index`、`Sequence` 等对象。

    新版本 2.0 中添加了对所有`SchemaItem`对象的支持。

+   `connection` – 发出 DROP 语句或语句的`Connection`。

+   `**kw` – 与事件相关的附加关键字参数。此字典的内容可能会在不同版本之间变化，并包括在元数据级事件中生成的表列表、checkfirst 标志以及内部事件使用的其他元素。

`listen()` 还接受`propagate=True`修饰符，用于此事件；当为 True 时，监听器函数将为目标对象的任何副本建立，即在使用`Table.to_metadata()` 生成的那些副本。

```py
method after_parent_attach(target: SchemaEventTarget, parent: SchemaItem) → None
```

在`SchemaItem`与父`SchemaItem`关联之后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'after_parent_attach')
def receive_after_parent_attach(target, parent):
    "listen for the 'after_parent_attach' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 目标对象

+   `parent` – 将目标附加到的父级。

`listen()`还接受`propagate=True`修饰符用于此事件；当为 True 时，监听函数将为目标对象的任何副本建立，即在使用`Table.to_metadata()`时生成的那些副本。

```py
method before_create(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在生成 CREATE 语句之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'before_create')
def receive_before_create(target, connection, **kw):
    "listen for the 'before_create' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    `SchemaObject`，比如`MetaData`或`Table`，还包括所有的创建/删除对象，比如`Index`、`Sequence`等，这些对象是事件的目标。

    2.0 版新增：对所有`SchemaItem`对象的支持已添加。

+   `connection` – 将发出 CREATE 语句或语句的`Connection`。

+   `**kw` – 与事件相关的其他关键字参数。此字典的内容可能会在不同版本之间变化，并包括元数据级事件生成的表列表、checkfirst 标志和内部事件使用的其他元素。

`listen()`接受`propagate=True`修饰符用于此事件；当为 True 时，监听函数将为目标对象的任何副本建立，即在使用`Table.to_metadata()`时生成的那些副本。

`listen()`接受`insert=True`修饰符用于此事件；当为 True 时，监听函数将被添加到内部事件列表的开头，并在未传递此参数的已注册监听函数之前执行。

```py
method before_drop(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在生成 DROP 语句之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'before_drop')
def receive_before_drop(target, connection, **kw):
    "listen for the 'before_drop' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    `SchemaObject`，比如`MetaData`或`Table`，还包括所有的创建/删除对象，比如`Index`、`Sequence`等，这些对象是事件的目标。

    2.0 版新增：对所有`SchemaItem`对象的支持已添加。

+   `connection` – 发出 DROP 语句的`Connection`。

+   `**kw` – 与事件相关的附加关键字参数。此字典的内容可能因发布而异，包括用于元数据级事件生成表的表列表，checkfirst 标志和内部事件使用的其他元素。

`listen()`也接受`propagate=True`修饰符用于此事件；当为 True 时，监听器函数将为目标对象的任何副本建立，即在使用`Table.to_metadata()`生成的那些副本。

```py
method before_parent_attach(target: SchemaEventTarget, parent: SchemaItem) → None
```

在将`SchemaItem`与父`SchemaItem`关联之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'before_parent_attach')
def receive_before_parent_attach(target, parent):
    "listen for the 'before_parent_attach' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 目标对象

+   `parent` – 将目标附加到的父对象。

`listen()`也接受`propagate=True`修饰符用于此事件；当为 True 时，监听器函数将为目标对象的任何副本建立，即在使用`Table.to_metadata()`生成的那些副本。

```py
method column_reflect(inspector: Inspector, table: Table, column_info: ReflectedColumn) → None
```

在对反射的`Table`检索每个‘列信息’单元时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'column_reflect')
def receive_column_reflect(inspector, table, column_info):
    "listen for the 'column_reflect' event"

    # ... (event handling logic) ...
```

此事件最容易通过将其应用于特定的`MetaData`实例来使用，在该实例中，它将对该`MetaData`中的所有`Table`对象产生影响，这些对象在反射时进行。

```py
metadata = MetaData()

@event.listens_for(metadata, 'column_reflect')
def receive_column_reflect(inspector, table, column_info):
    # receives for all Table objects that are reflected
    # under this MetaData

# will use the above event hook
my_table = Table("my_table", metadata, autoload_with=some_engine)
```

新版本 1.4.0b2 中：`DDLEvents.column_reflect()`钩子现在也可以应用于`MetaData`对象，以及它将对与目标`MetaData`关联的所有`Table`对象产生影响的`MetaData`类本身。

它也可以应用于整个`Table`类：

```py
from sqlalchemy import Table

@event.listens_for(Table, 'column_reflect')
def receive_column_reflect(inspector, table, column_info):
    # receives for all Table objects that are reflected
```

它也可以应用到正在使用`Table.listeners`参数反射的特定`Table`上：

```py
t1 = Table(
    "my_table",
    autoload_with=some_engine,
    listeners=[
        ('column_reflect', receive_column_reflect)
    ]
)
```

由方言返回的列信息字典会被传递，并且可以被修改。该字典是由`Inspector.get_columns()`返回的列表中的每个元素返回的：

> +   `name` - 列的名称，应用于`Column.name`参数
> +   
> +   `type` - 该列的类型，应该是`TypeEngine`的一个实例，应用于`Column.type`参数
> +   
> +   `nullable` - 如果列是 NULL 或 NOT NULL 的布尔标志，应用于`Column.nullable`参数
> +   
> +   `default` - 列的服务器默认值。通常将其指定为纯字符串 SQL 表达式，但事件也可以传递一个`FetchedValue`、`DefaultClause`或`text()`对象。应用于`Column.server_default`参数

在对该字典执行任何操作之前调用事件，并且内容可以被修改；以下其他键可以被添加到字典中以进一步修改如何构造`Column`：

> +   `key` - 将用于在`.c`集合中访问此`Column`的字符串键；将应用于`Column.key`参数。也用于 ORM 映射。参见从反射表自动命名列方案一节的示例。
> +   
> +   `quote` - 强制或取消强制对列名称进行引用；应用于`Column.quote`参数。
> +   
> +   `info` - 一个包含任意数据的字典，用于跟踪`Column`，应用于`Column.info`参数。

`listen()`还接受`propagate=True`修改器用于此事件；当为 True 时，监听器函数将为目标对象的任何副本建立，即使用`Table.to_metadata()`生成的那些副本。

另请参阅

从反射表自动命名方案 - 在 ORM 映射文档中

拦截列定义 - 在 Automap 文档中

反映与数据库无关的类型 - 在反射数据库对象文档中

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.DDLEventsDispatch object>
```

回引到 _Dispatch 类。

双向与 _Dispatch._events 相对

```py
class sqlalchemy.events.SchemaEventTarget
```

作为`DDLEvents`事件目标的元素的基类。

这包括`SchemaItem`以及`SchemaType`。

**类签名**

类`sqlalchemy.events.SchemaEventTarget`（`sqlalchemy.event.registry.EventTarget`）

## 连接池事件

| 对象名称 | 描述 |
| --- | --- |
| PoolEvents | `Pool`的可用事件。 |
| PoolResetState | 描述了一个 DBAPI 连接在传递给`PoolEvents.reset()`连接池事件时的状态。 |

```py
class sqlalchemy.events.PoolEvents
```

`Pool`的可用事件。

这里的方法定义了一个事件的名称以及传递给监听器函数的成员的名称。

例如：

```py
from sqlalchemy import event

def my_on_checkout(dbapi_conn, connection_rec, connection_proxy):
    "handle an on checkout event"

event.listen(Pool, 'checkout', my_on_checkout)
```

除了接受 `Pool` 类和 `Pool` 实例外，`PoolEvents` 还接受 `Engine` 对象和 `Engine` 类作为目标，这将解析为给定引擎的 `.pool` 属性或 `Pool` 类：

```py
engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

# will associate with engine.pool
event.listen(engine, 'checkout', my_on_checkout)
```

**成员**

checkin(), checkout(), close(), close_detached(), connect(), detach(), dispatch, first_connect(), invalidate(), reset(), soft_invalidate()

**类签名**

class `sqlalchemy.events.PoolEvents` (`sqlalchemy.event.Events`)

```py
method checkin(dbapi_connection: DBAPIConnection | None, connection_record: ConnectionPoolEntry) → None
```

当连接返回到池中时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'checkin')
def receive_checkin(dbapi_connection, connection_record):
    "listen for the 'checkin' event"

    # ... (event handling logic) ...
```

注意连接可能会关闭，并且如果连接已失效，则可能为 None。对于已分离的连接，不会调用 `checkin`。（它们不会返回到池中。）

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的 `ConnectionPoolEntry`。

```py
method checkout(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, connection_proxy: PoolProxiedConnection) → None
```

当从池中检索到连接时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'checkout')
def receive_checkout(dbapi_connection, connection_record, connection_proxy):
    "listen for the 'checkout' event"

    # ... (event handling logic) ...
```

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的 `ConnectionPoolEntry`。

+   `connection_proxy` – `PoolProxiedConnection` 对象，它将在检出的生命周期内代理 DBAPI 连接的公共接口。

如果引发`DisconnectionError`，当前连接将被释放并重新获取新连接。所有检出监听器的处理将中止，并使用新连接重新启动。

另请参阅

`ConnectionEvents.engine_connect()` - 在创建新的`Connection`时发生的类似事件。

```py
method close(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

当 DBAPI 连接关闭时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'close')
def receive_close(dbapi_connection, connection_record):
    "listen for the 'close' event"

    # ... (event handling logic) ...
```

事件在关闭发生之前发出。

连接关闭可能会失败；通常是因为连接已经关闭。如果关闭操作失败，则连接将被丢弃。

`close()` 事件对应于仍与池相关联的连接。要拦截分离连接的关闭事件，请使用`close_detached()`。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
method close_detached(dbapi_connection: DBAPIConnection) → None
```

当分离的 DBAPI 连接关闭时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'close_detached')
def receive_close_detached(dbapi_connection):
    "listen for the 'close_detached' event"

    # ... (event handling logic) ...
```

事件在关闭发生之前发出。

连接关闭可能会失败；通常是因为连接已经关闭。如果关闭操作失败，则连接将被丢弃。

参数：

**dbapi_connection** – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

```py
method connect(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

在为给定的`Pool`首次创建特定的 DBAPI 连接时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'connect')
def receive_connect(dbapi_connection, connection_record):
    "listen for the 'connect' event"

    # ... (event handling logic) ...
```

此事件允许捕获直接在使用 DBAPI 模块级别的`.connect()`方法以产生新的 DBAPI 连接之后的点。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
method detach(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

当 DBAPI 连接与池“分离”时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'detach')
def receive_detach(dbapi_connection, connection_record):
    "listen for the 'detach' event"

    # ... (event handling logic) ...
```

此事件在分离后发生。连接不再与给定的连接记录关联。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.PoolEventsDispatch object>
```

回溯到 _Dispatch 类。

与 _Dispatch._events 双向对应

```py
method first_connect(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry) → None
```

第一次从特定`Pool`检出 DBAPI 连接时仅调用一次此事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'first_connect')
def receive_first_connect(dbapi_connection, connection_record):
    "listen for the 'first_connect' event"

    # ... (event handling logic) ...
```

`PoolEvents.first_connect()` 的原因是基于所有连接使用的设置来确定关于特定一系列数据库连接的信息。由于特定 `Pool` 指的是单个“创建者”函数（在 `Engine` 方面指的是使用的 URL 和连接选项），因此通常可以对单个连接进行观察，可以安全地假定关于所有后续连接都有效，例如数据库版本、服务器和客户端编码设置、排序设置等。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

```py
method invalidate(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, exception: BaseException | None) → None
```

当 DBAPI 连接被“失效”时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'invalidate')
def receive_invalidate(dbapi_connection, connection_record, exception):
    "listen for the 'invalidate' event"

    # ... (event handling logic) ...
```

每次调用 `ConnectionPoolEntry.invalidate()` 方法时都会触发此事件，无论是通过 API 使用还是通过“自动失效”触发，不带 `soft` 标志。

在最终尝试调用连接的`.close()`之前发生此事件。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection` 属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

+   `exception` – 这次失效的原因对应的异常对象，如果有的话。可能为 `None`。

另请参阅

更多关于失效的信息

```py
method reset(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, reset_state: PoolResetState) → None
```

在池化连接发生“重置”操作之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'reset')
def receive_reset(dbapi_connection, connection_record, reset_state):
    "listen for the 'reset' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-2.0, will be removed in a future release)
@event.listens_for(SomeEngineOrPool, 'reset')
def receive_reset(dbapi_connection, connection_record):
    "listen for the 'reset' event"

    # ... (event handling logic) ...
```

从 2.0 版本开始更改：`PoolEvents.reset()` 事件现在接受参数 `PoolEvents.reset.dbapi_connection`、`PoolEvents.reset.connection_record`、`PoolEvents.reset.reset_state`。将在将来的版本中删除接受上述“已弃用”先前参数签名的侦听器函数的支持。

此事件表示在 DBAPI 连接上调用 `rollback()` 方法之前返回到池中或丢弃时发生。可以使用此事件钩子实现自定义“重置”策略，也可以通过使用 `Pool.reset_on_return` 参数禁用默认的“重置”行为。

`PoolEvents.reset()` 和 `PoolEvents.checkin()` 事件的主要区别在于 `PoolEvents.reset()` 不仅适用于被返回到池中的池化连接，还适用于使用 `Connection.detach()` 方法分离的连接以及由于在连接被检入之前进行垃圾回收而被丢弃的 asyncio 连接。

注意，并非所有使用 `Connection.invalidate()` 使连接无效的事件都会被触发。这些事件可以通过 `PoolEvents.soft_invalidate()` 和 `PoolEvents.invalidate()` 事件钩子拦截，所有的“连接关闭”事件都可以通过 `PoolEvents.close()` 拦截。

`PoolEvents.reset()` 事件通常在 `PoolEvents.checkin()` 事件之后发生，除了那些在重置后立即丢弃连接的情况。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection`属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

+   `reset_state` –

    `PoolResetState`实例，提供关于正在重置连接的情况的信息。

    新版本为 2.0。

另请参阅

返回时重置

`ConnectionEvents.rollback()`

`ConnectionEvents.commit()`

```py
method soft_invalidate(dbapi_connection: DBAPIConnection, connection_record: ConnectionPoolEntry, exception: BaseException | None) → None
```

当 DBAPI 连接要“软无效化”时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngineOrPool, 'soft_invalidate')
def receive_soft_invalidate(dbapi_connection, connection_record, exception):
    "listen for the 'soft_invalidate' event"

    # ... (event handling logic) ...
```

每当调用`ConnectionPoolEntry.invalidate()`方法时，都会发生此事件，带有`soft`标志。

软无效化指的是在当前连接被检入后，跟踪此连接的连接记录将强制重新连接。在调用时，它不会主动关闭 dbapi_connection。

参数：

+   `dbapi_connection` – 一个 DBAPI 连接。`ConnectionPoolEntry.dbapi_connection`属性。

+   `connection_record` – 管理 DBAPI 连接的`ConnectionPoolEntry`。

+   `exception` – 如果有原因导致无效化，则对应此无效化的异常对象。可能为`None`。

```py
class sqlalchemy.events.PoolResetState
```

描述了 DBAPI 连接在传递给`PoolEvents.reset()`连接池事件时的状态。

**成员**

asyncio_safe，terminate_only，transaction_was_reset

新版本为 2.0.0b3。

```py
attribute asyncio_safe: bool
```

指示重置操作是否发生在期望存在封闭事件循环的范围内的情况下，用于 asyncio 应用程序。

在连接被垃圾收集时，将为 False。

```py
attribute terminate_only: bool
```

指示连接是否立即终止并且不被检入到池中。

这发生在无效的连接上，以及未被调用代码清理处理而被垃圾收集的 asyncio 连接上。在后一种情况下，在垃圾收集中无法安全地运行 asyncio 连接操作，因为不一定存在事件循环。

```py
attribute transaction_was_reset: bool
```

表示如果 DBAPI 连接上的事务已经被`Connection`对象“重置”。

如果`Connection`具有事务状态，并且该状态未使用`Connection.rollback()`或`Connection.commit()`方法关闭；相反，事务在`Connection.close()`方法中内联关闭，因此在达到此事件时保证不会存在。

## SQL 执行和连接事件

| 对象名称 | 描述 |
| --- | --- |
| ConnectionEvents | 对于`Connection`和`Engine`可用的事件。 |
| DialectEvents | 用于执行替换函数的事件接口。 |

```py
class sqlalchemy.events.ConnectionEvents
```

对于`Connection`和`Engine`可用的事件。

这里的方法定义了事件的名称以及传递给监听器函数的成员的名称。

事件监听器可以与任何`Connection`或`Engine`类或实例相关联，例如一个`Engine`，例如：

```py
from sqlalchemy import event, create_engine

def before_cursor_execute(conn, cursor, statement, parameters, context,
                                                executemany):
    log.info("Received statement: %s", statement)

engine = create_engine('postgresql+psycopg2://scott:tiger@localhost/test')
event.listen(engine, "before_cursor_execute", before_cursor_execute)
```

或使用特定的`Connection`：

```py
with engine.begin() as conn:
    @event.listens_for(conn, 'before_cursor_execute')
    def before_cursor_execute(conn, cursor, statement, parameters,
                                    context, executemany):
        log.info("Received statement: %s", statement)
```

当方法被带有语句参数调用时，例如在`after_cursor_execute()`或`before_cursor_execute()`中，语句是传输到连接的 DBAPI`cursor`中准备的确切 SQL 字符串，该连接的`Dialect`。

`before_execute()`和`before_cursor_execute()`事件也可以使用`retval=True`标志来建立，这允许修改要发送到数据库的语句和参数。 `before_cursor_execute()`事件在此处特别有用，以添加特定的字符串转换，如注释，到所有执行中：

```py
from sqlalchemy.engine import Engine
from sqlalchemy import event

@event.listens_for(Engine, "before_cursor_execute", retval=True)
def comment_sql_calls(conn, cursor, statement, parameters,
                                    context, executemany):
    statement = statement + " -- some comment"
    return statement, parameters
```

注意

`ConnectionEvents`可以建立在任何组合的`Engine`、`Connection`，以及这些类的实例上。 对于给定的`Connection`实例，所有四个作用域的事件都将触发。 但是，出于性能原因，`Connection`对象在实例化时确定其父`Engine`是否已建立事件侦听器。 在依赖的`Connection`实例实例化之后，添加到`Engine`类或`Engine`实例的事件侦听器通常不会在该`Connection`实例上可用。 新添加的侦听器将取代对父`Engine`类或实例建立事件侦听器后创建的`Connection`实例。

参数：

**retval=False** – 仅适用于`before_execute()`和`before_cursor_execute()`事件。 当值为 True 时，用户定义的事件函数必须有一个返回值，即替换给定语句和参数的参数元组。 查看这些方法以获取特定返回参数的描述。

**成员**

after_cursor_execute(), after_execute(), before_cursor_execute(), before_execute(), begin(), begin_twophase(), commit(), commit_twophase(), dispatch, engine_connect(), engine_disposed(), prepare_twophase(), release_savepoint(), rollback(), rollback_savepoint(), rollback_twophase(), savepoint(), set_connection_execution_options(), set_engine_execution_options()

**类签名**

类`sqlalchemy.events.ConnectionEvents` (`sqlalchemy.event.Events`)

```py
method after_cursor_execute(conn: Connection, cursor: DBAPICursor, statement: str, parameters: _DBAPIAnyExecuteParams, context: ExecutionContext | None, executemany: bool) → None
```

在执行后拦截低级游标`execute()`事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'after_cursor_execute')
def receive_after_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    "listen for the 'after_cursor_execute' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection`对象

+   `cursor` – DBAPI 游标对象。如果语句是 SELECT，则将有待处理的结果，但不应消耗这些结果，因为它们将被`CursorResult`所需。

+   `statement` – 字符串 SQL 语句，如传递给 DBAPI

+   `parameters` – 字典、元组或传递给 DBAPI `cursor`的`execute()`或`executemany()`方法的参数列表。在某些情况下可能为`None`。

+   `context` – 正在使用的`ExecutionContext`对象。可能为`None`。

+   `executemany` – 布尔值，如果为`True`，则这是一个`executemany()`调用，如果为`False`，则这是一个`execute()`调用。

```py
method after_execute(conn: Connection, clauseelement: Executable, multiparams: _CoreMultiExecuteParams, params: _CoreSingleExecuteParams, execution_options: _ExecuteOptions, result: Result[Any]) → None
```

在执行后拦截高级`execute()`事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'after_execute')
def receive_after_execute(conn, clauseelement, multiparams, params, execution_options, result):
    "listen for the 'after_execute' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-1.4, will be removed in a future release)
@event.listens_for(SomeEngine, 'after_execute')
def receive_after_execute(conn, clauseelement, multiparams, params, result):
    "listen for the 'after_execute' event"

    # ... (event handling logic) ...
```

从版本 1.4 开始更改: `ConnectionEvents.after_execute()` 事件现在接受参数 `ConnectionEvents.after_execute.conn`, `ConnectionEvents.after_execute.clauseelement`, `ConnectionEvents.after_execute.multiparams`, `ConnectionEvents.after_execute.params`, `ConnectionEvents.after_execute.execution_options`, `ConnectionEvents.after_execute.result`。支持接受先前参数签名的监听器函数将在未来的版本中删除。

参数:

+   `conn` – `Connection` 对象

+   `clauseelement` – SQL 表达式构造，`Compiled` 实例，或传递给 `Connection.execute()` 的字符串语句。

+   `multiparams` – 多个参数集，一个字典列表。

+   `params` – 单个参数集，一个字典。

+   `execution_options` –

    传递给语句的执行选项字典，如果有的话。这是将要使用的所有选项的合并，包括语句、连接和传递给方法本身的那些选项，用于执行 2.0 风格。

+   `result` – 执行生成的 `CursorResult`。

```py
method before_cursor_execute(conn: Connection, cursor: DBAPICursor, statement: str, parameters: _DBAPIAnyExecuteParams, context: ExecutionContext | None, executemany: bool) → Tuple[str, _DBAPIAnyExecuteParams] | None
```

在执行之前拦截低级别的游标 execute() 事件，接收要针对游标调用的字符串 SQL 语句和 DBAPI 特定参数列表。

示��参数形式:

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'before_cursor_execute')
def receive_before_cursor_execute(conn, cursor, statement, parameters, context, executemany):
    "listen for the 'before_cursor_execute' event"

    # ... (event handling logic) ...
```

这个事件非常适合用于日志记录以及对 SQL 字符串的后期修改。对于那些特定于目标后端的参数修改来说，它不太理想。

这个事件可以选择使用 `retval=True` 标志来建立。在这种情况下，`statement` 和 `parameters` 参数应该作为一个二元组返回:

```py
@event.listens_for(Engine, "before_cursor_execute", retval=True)
def before_cursor_execute(conn, cursor, statement,
                parameters, context, executemany):
    # do something with statement, parameters
    return statement, parameters
```

参见 `ConnectionEvents` 中的示例。

参数:

+   `conn` – `Connection` 对象

+   `cursor` – DBAPI 游标对象

+   `statement` – 要传递给 DBAPI 的字符串 SQL 语句

+   `parameters` – 正在传递给 DBAPI `cursor` 的 `execute()` 或 `executemany()` 方法的参数的字典、元组或列表。在某些情况下可能为 `None`。

+   `context` – `ExecutionContext` 对象正在使用。可能为 `None`。

+   `executemany` – 布尔值，如果为 `True`，则这是一个 `executemany()` 调用，如果为 `False`，则这是一个 `execute()` 调用。

另请参阅

`before_execute()`

`after_cursor_execute()`

```py
method before_execute(conn: Connection, clauseelement: Executable, multiparams: _CoreMultiExecuteParams, params: _CoreSingleExecuteParams, execution_options: _ExecuteOptions) → Tuple[Executable, _CoreMultiExecuteParams, _CoreSingleExecuteParams] | None
```

拦截高级 execute() 事件，接收未编译的 SQL 构造和其他对象，在渲染成 SQL 之前。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'before_execute')
def receive_before_execute(conn, clauseelement, multiparams, params, execution_options):
    "listen for the 'before_execute' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-1.4, will be removed in a future release)
@event.listens_for(SomeEngine, 'before_execute')
def receive_before_execute(conn, clauseelement, multiparams, params):
    "listen for the 'before_execute' event"

    # ... (event handling logic) ...
```

自 1.4 版更改：`ConnectionEvents.before_execute()` 事件现在接受参数 `ConnectionEvents.before_execute.conn`、`ConnectionEvents.before_execute.clauseelement`、`ConnectionEvents.before_execute.multiparams`、`ConnectionEvents.before_execute.params`、`ConnectionEvents.before_execute.execution_options`。对于接受上述先前参数签名的监听函数，将在将来的版本中移除。

此事件非常适用于调试 SQL 编译问题以及对发送到数据库的参数进行早期处理，因为此处的参数列表将保持一致的格式。

此事件可以选择使用 `retval=True` 标志来建立。在这种情况下，`clauseelement`、`multiparams` 和 `params` 参数应作为三元组返回：

```py
@event.listens_for(Engine, "before_execute", retval=True)
def before_execute(conn, clauseelement, multiparams, params):
    # do something with clauseelement, multiparams, params
    return clauseelement, multiparams, params
```

参数：

+   `conn` – `Connection` 对象

+   `clauseelement` – SQL 表达式构造，`Compiled` 实例，或传递给`Connection.execute()`的字符串语句。

+   `multiparams` – 多参数集合，一个字典列表。

+   `params` – 单参数集合，一个字典。

+   `execution_options` –

    执行选项字典随语句一起传递，如果有的话。这是将被使用的所有选项的合并，包括语句、连接和传递给方法本身的 2.0 执行风格的选项。

另请参阅

`before_cursor_execute()`

```py
method begin(conn: Connection) → None
```

拦截`begin()`事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'begin')
def receive_begin(conn):
    "listen for the 'begin' event"

    # ... (event handling logic) ...
```

参数：

**conn** – `Connection` 对象

```py
method begin_twophase(conn: Connection, xid: Any) → None
```

拦截`begin_twophase()`事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'begin_twophase')
def receive_begin_twophase(conn, xid):
    "listen for the 'begin_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

```py
method commit(conn: Connection) → None
```

拦截由`Transaction`发起的`commit()`事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'commit')
def receive_commit(conn):
    "listen for the 'commit' event"

    # ... (event handling logic) ...
```

请注意，如果`reset_on_return`标志设置为值`'commit'`，`Pool`也可能在检入时“自动提交”DBAPI 连接。要拦截此提交，请使用`PoolEvents.reset()`钩子。

参数：

**conn** – `Connection` 对象

```py
method commit_twophase(conn: Connection, xid: Any, is_prepared: bool) → None
```

拦截`commit_twophase()`事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'commit_twophase')
def receive_commit_twophase(conn, xid, is_prepared):
    "listen for the 'commit_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

+   `is_prepared` – 布尔值，指示是否调用了`TwoPhaseTransaction.prepare()`。

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.ConnectionEventsDispatch object>
```

引用回 _Dispatch 类。

双向针对 _Dispatch._events

```py
method engine_connect(conn: Connection) → None
```

拦截创建新的`Connection`。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'engine_connect')
def receive_engine_connect(conn):
    "listen for the 'engine_connect' event"

    # ... (event handling logic) ...

# DEPRECATED calling style (pre-2.0, will be removed in a future release)
@event.listens_for(SomeEngine, 'engine_connect')
def receive_engine_connect(conn, branch):
    "listen for the 'engine_connect' event"

    # ... (event handling logic) ...
```

2.0 版中的变化：`ConnectionEvents.engine_connect()`事件现在接受参数`ConnectionEvents.engine_connect.conn`。支持接受上面列出的“已弃用”的先前参数签名的监听器函数将在将来的版本中移除。

该事件通常作为调用`Engine.connect()`方法的直接结果。

它与`PoolEvents.connect()`方法不同，后者指的是在 DBAPI 级别实际连接到数据库；DBAPI 连接可能会被池化并重复使用于许多操作。相比之下，此事件仅指生产一个围绕此类 DBAPI 连接的更高级别`Connection`包装器。

它还与`PoolEvents.checkout()`事件不同，后者特定于`Connection`对象，而不是`PoolEvents.checkout()`处理的 DBAPI 连接，尽管此 DBAPI 连接可通过`Connection.connection`属性在此处获取。但请注意，在单个`Connection`对象的生命周期内，实际上可以有多个`PoolEvents.checkout()`事件，如果该`Connection`被使无效并重新建立。

参数：

**conn** – `Connection` 对象。

另请参见

`PoolEvents.checkout()` 是针对单个 DBAPI 连接的低级别池检出事件。

```py
method engine_disposed(engine: Engine) → None
```

拦截`Engine.dispose()`方法调用时。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'engine_disposed')
def receive_engine_disposed(engine):
    "listen for the 'engine_disposed' event"

    # ... (event handling logic) ...
```

`Engine.dispose()`方法指示引擎“处理”其连接池（例如`Pool`)，并用新的替换它。处理旧池的效果是关闭现有的已检入连接。新池在首次使用之前不会建立任何新连接。

可以使用此事件指示应清理与`Engine`相关的资源，需要注意的是`Engine`仍然可以用于新请求，此时会重新获取连接资源。

```py
method prepare_twophase(conn: Connection, xid: Any) → None
```

拦截 prepare_twophase() 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'prepare_twophase')
def receive_prepare_twophase(conn, xid):
    "listen for the 'prepare_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

```py
method release_savepoint(conn: Connection, name: str, context: None) → None
```

拦截 release_savepoint() 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'release_savepoint')
def receive_release_savepoint(conn, name, context):
    "listen for the 'release_savepoint' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `name` – 用于保存点的指定名称。

+   `context` – 未使用

```py
method rollback(conn: Connection) → None
```

拦截 rollback() 事件，由 `Transaction` 发起。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'rollback')
def receive_rollback(conn):
    "listen for the 'rollback' event"

    # ... (event handling logic) ...
```

注意，如果 `reset_on_return` 标志设置为其默认值 `'rollback'`，则 `Pool` 也会在归还时“自动回滚” DBAPI 连接。要拦截此回滚，请使用 `PoolEvents.reset()` 钩子。

参数：

**conn** – `Connection` 对象

另请参阅

`PoolEvents.reset()`

```py
method rollback_savepoint(conn: Connection, name: str, context: None) → None
```

拦截 rollback_savepoint() 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'rollback_savepoint')
def receive_rollback_savepoint(conn, name, context):
    "listen for the 'rollback_savepoint' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `name` – 用于保存点的指定名称。

+   `context` – 未使用

```py
method rollback_twophase(conn: Connection, xid: Any, is_prepared: bool) → None
```

拦截 rollback_twophase() 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'rollback_twophase')
def receive_rollback_twophase(conn, xid, is_prepared):
    "listen for the 'rollback_twophase' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `xid` – 两阶段 XID 标识符

+   `is_prepared` – 布尔值，指示是否调用了 `TwoPhaseTransaction.prepare()`。

```py
method savepoint(conn: Connection, name: str) → None
```

拦截 savepoint() 事件。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'savepoint')
def receive_savepoint(conn, name):
    "listen for the 'savepoint' event"

    # ... (event handling logic) ...
```

参数：

+   `conn` – `Connection` 对象

+   `name` – 用于保存点的指定名称。

```py
method set_connection_execution_options(conn: Connection, opts: Dict[str, Any]) → None
```

拦截 `Connection.execution_options()` 方法调用时。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'set_connection_execution_options')
def receive_set_connection_execution_options(conn, opts):
    "listen for the 'set_connection_execution_options' event"

    # ... (event handling logic) ...
```

此方法在新的 `Connection` 生成后被调用，带有新更新的执行选项集合，但在 `Dialect` 对这些新选项采取任何操作之前。

请注意，当从其父`Engine`继承执行选项的新`Connection`生成时，不会调用此方法；要拦截此条件，请使用`ConnectionEvents.engine_connect()`事件。

参数：

+   `conn` – 新复制的`Connection`对象

+   `opts` –

    传递给`Connection.execution_options()`方法的选项字典。此字典可以就地修改以影响最终生效的选项。

    2.0 版中的新功能：`opts`字典可以就地修改。

另请参阅

`ConnectionEvents.set_engine_execution_options()` - 当调用`Engine.execution_options()`时触发的事件。

```py
method set_engine_execution_options(engine: Engine, opts: Dict[str, Any]) → None
```

当调用`Engine.execution_options()`方法时拦截。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'set_engine_execution_options')
def receive_set_engine_execution_options(engine, opts):
    "listen for the 'set_engine_execution_options' event"

    # ... (event handling logic) ...
```

`Engine.execution_options()`方法会生成一个存储新选项的`Engine`的浅拷贝。这个新的`Engine`会传递到这里。这个方法的一个特定应用是向给定的`Engine`添加一个`ConnectionEvents.engine_connect()`事件处理程序，该处理程序将执行一些特定于这些执行选项的每个`Connection`任务。

参数：

+   `conn` – 新复制的`Engine`对象

+   `opts` –

    传递给`Connection.execution_options()`方法的选项字典。此字典可以就地修改以影响最终生效的选项。

    2.0 版中的新功能：`opts`字典可以就地修改。

另请参阅

`ConnectionEvents.set_connection_execution_options()` - 当调用 `Connection.execution_options()` 时调用的事件。

```py
class sqlalchemy.events.DialectEvents
```

执行替换函数的事件接口。

这些事件允许直接对与 DBAPI 交互的关键方言函数进行检测和替换。

注意

`DialectEvents` 钩子应被视为**半公开**和实验性质。这些钩子不适用于一般用途，仅适用于需要将复杂的 DBAPI 机制重新注入现有方言的情况。对于一般用途的语句拦截事件，请使用 `ConnectionEvents` 接口。

另请参阅

`ConnectionEvents.before_cursor_execute()`

`ConnectionEvents.before_execute()`

`ConnectionEvents.after_cursor_execute()`

`ConnectionEvents.after_execute()`

**成员**

dispatch, do_connect(), do_execute(), do_execute_no_params(), do_executemany(), do_setinputsizes(), handle_error()

**类签名**

类 `sqlalchemy.events.DialectEvents` (`sqlalchemy.event.Events`)

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.DialectEventsDispatch object>
```

参考回到 _Dispatch 类。

双向对抗 _Dispatch._events

```py
method do_connect(dialect: Dialect, conn_rec: ConnectionPoolEntry, cargs: Tuple[Any, ...], cparams: Dict[str, Any]) → DBAPIConnection | None
```

在建立连接之前接收连接参数。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_connect')
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    "listen for the 'do_connect' event"

    # ... (event handling logic) ...
```

这个事件非常有用，因为它允许处理程序操作控制如何调用 DBAPI `connect()` 函数的 `cargs` 和/或 `cparams` 集合。`cargs` 将始终是一个可以原地变异的 Python 列表，`cparams` 是一个也可以被变异的 Python 字典：

```py
e = create_engine("postgresql+psycopg2://user@host/dbname")

@event.listens_for(e, 'do_connect')
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    cparams["password"] = "some_password"
```

事件钩子也可以完全覆盖对 `connect()` 的调用，通过返回一个非 `None` 的 DBAPI 连接对象：

```py
e = create_engine("postgresql+psycopg2://user@host/dbname")

@event.listens_for(e, 'do_connect')
def receive_do_connect(dialect, conn_rec, cargs, cparams):
    return psycopg2.connect(*cargs, **cparams)
```

另请参阅

自定义 DBAPI connect() 参数 / on-connect routines

```py
method do_execute(cursor: DBAPICursor, statement: str, parameters: _DBAPISingleExecuteParams, context: ExecutionContext) → Literal[True] | None
```

接收一个游标以调用 `execute()`。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_execute')
def receive_do_execute(cursor, statement, parameters, context):
    "listen for the 'do_execute' event"

    # ... (event handling logic) ...
```

返回 True 以阻止进一步调用事件，并指示光标执行已经在事件处理程序中发生。

```py
method do_execute_no_params(cursor: DBAPICursor, statement: str, context: ExecutionContext) → Literal[True] | None
```

接收一个游标以调用没有参数的 `execute()`。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_execute_no_params')
def receive_do_execute_no_params(cursor, statement, context):
    "listen for the 'do_execute_no_params' event"

    # ... (event handling logic) ...
```

返回 True 以阻止进一步调用事件，并指示光标执行已经在事件处理程序中发生。

```py
method do_executemany(cursor: DBAPICursor, statement: str, parameters: _DBAPIMultiExecuteParams, context: ExecutionContext) → Literal[True] | None
```

接收一个游标以调用 `executemany()`。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_executemany')
def receive_do_executemany(cursor, statement, parameters, context):
    "listen for the 'do_executemany' event"

    # ... (event handling logic) ...
```

返回 True 以阻止进一步调用事件，并指示光标执行已经在事件处理程序中发生。

```py
method do_setinputsizes(inputsizes: Dict[BindParameter[Any], Any], cursor: DBAPICursor, statement: str, parameters: _DBAPIAnyExecuteParams, context: ExecutionContext) → None
```

接收 setinputsizes 字典以进行可能的修改。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'do_setinputsizes')
def receive_do_setinputsizes(inputsizes, cursor, statement, parameters, context):
    "listen for the 'do_setinputsizes' event"

    # ... (event handling logic) ...
```

在方言使用 DBAPI `cursor.setinputsizes()` 方法传递有关特定语句的参数绑定的情况下，将发出此事件。给定的 `inputsizes` 字典将包含 `BindParameter` 对象作为键，链接到 DBAPI 特定类型对象作为值；对于未绑定的参数，将使用 `None` 作为值将其添加到字典中，这意味着该参数不会包含在最终的 setinputsizes 调用中。可以使用此事件来检查和/或记录正在绑定的数据类型，以及直接修改字典。可以向此字典添加、修改或删除参数。调用者通常会想要检查给定绑定对象的 `BindParameter.type` 属性，以便对 DBAPI 对象做出决策。

在事件之后，`inputsizes` 字典将转换为适当的数据结构，以传递给 `cursor.setinputsizes`；对于位置绑定参数执行样式，将转换为列表，对于命名绑定参数执行样式，将转换为字符串参数键到 DBAPI 类型对象的字典。

`setinputsizes` 钩子通常仅在包括标志 `use_setinputsizes=True` 的方言中使用。使用此功能的方言包括 cx_Oracle、pg8000、asyncpg 和 pyodbc 方言。

注意

对于使用 pyodbc，必须将 `use_setinputsizes` 标志传递给方言，例如：

```py
create_engine("mssql+pyodbc://...", use_setinputsizes=True)
```

另请参阅

setinputsizes 支持

新版本 1.2.9 中的新功能。

另请参阅

通过 setinputsizes 对 cx_Oracle 数据绑定性能进行细粒度控制

```py
method handle_error(exception_context: ExceptionContext) → BaseException | None
```

拦截由 `Dialect` 处理的所有异常，通常但不限于在 `Connection` 范围内发出的异常。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeEngine, 'handle_error')
def receive_handle_error(exception_context):
    "listen for the 'handle_error' event"

    # ... (event handling logic) ...
```

从版本 2.0 起更改：`DialectEvents.handle_error()` 事件已移至 `DialectEvents` 类，从 `ConnectionEvents` 类移动，以便它还可以参与由 `create_engine.pool_pre_ping` 参数配置的“pre ping”操作。该事件仍通过使用 `Engine` 作为事件目标进行注册，但请注意，不再支持将 `Connection` 用作 `DialectEvents.handle_error()` 的事件目标。

这包括由 DBAPI 发出的所有异常以及 SQLAlchemy 的语句调用过程中，包括编码错误和其他语句验证错误。调用事件的其他区域包括事务开始和结束，结果行获取，游标创建。

请注意，`handle_error()` 可能在 *任何时候* 支持新类型的异常和新的调用场景。使用此事件的代码必须预期在次要版本中存在新的调用模式。

为了支持对应于异常的各种成员，并且允许事件的可扩展性而不会导致向后不兼容，唯一接收的参数是`ExceptionContext`的实例。此对象包含表示异常详细信息的数据成员。

此钩子支持的用例包括：

+   用于记录和调试目的的只读低级异常处理

+   确定 DBAPI 连接错误消息是否表明需要重新连接数据库，包括一些方言中使用的“pre_ping”处理程序

+   确定或禁用特定异常响应中连接或拥有的连接池是否无效或过期

+   异常重写

在失败操作的游标（如果有）仍然打开且可访问时调用该钩子。可以在此游标上调用特殊的清理操作；SQLAlchemy 将在调用此钩子后尝试关闭此游标。

自 SQLAlchemy 2.0 起，使用 `create_engine.pool_pre_ping` 参数启用的“pre_ping”处理程序也将参与 `handle_error()` 过程，**对于依赖于断开连接代码来检测数据库存活性的那些方言**。请注意，某些方言，如 psycopg、psycopg2 和大多数 MySQL 方言，使用由 DBAPI 提供的本机 `ping()` 方法，该方法不使用断开连接代码。

在 2.0.0 版本中进行了更改：`DialectEvents.handle_error()` 事件钩子参与连接池“预检”操作。在此用法中，`ExceptionContext.engine` 属性将为 `None`，但是正在使用的 `Dialect` 始终可通过 `ExceptionContext.dialect` 属性获得。

在 2.0.5 版本中进行了更改：添加了 `ExceptionContext.is_pre_ping` 属性，当在连接池预检操作中触发 `DialectEvents.handle_error()` 事件钩子时，该属性将设置为 `True`。

在 2.0.5 版本中进行了更改：修复了一个问题，允许 PostgreSQL `psycopg` 和 `psycopg2` 驱动程序，以及所有 MySQL 驱动程序，在连接池“预检”操作期间正确参与 `DialectEvents.handle_error()` 事件钩子；先前，这些驱动程序的实现是不工作的。

处理程序函数有两个选项，可以将 SQLAlchemy 构造的异常替换为用户定义的异常。它可以直接引发这个新异常，这样所有后续的事件监听器都会被绕过，并且在适当的清理后引发异常：

```py
@event.listens_for(Engine, "handle_error")
def handle_exception(context):
    if isinstance(context.original_exception,
        psycopg2.OperationalError) and \
        "failed" in str(context.original_exception):
        raise MySpecialException("failed operation")
```

警告

因为 `DialectEvents.handle_error()` 事件特别提供了将异常重新抛出为失败语句引发的最终异常，如果用户定义的事件处理程序本身失败并抛出意外异常，则**堆栈跟踪将是误导性的**；堆栈跟踪可能不会显示实际失败的代码行！建议在此处小心编码，并在发生意外异常时使用日志记录和/或内联调试。

或者，可以使用“链接”风格的事件处理，通过使用`retval=True`修饰符配置处理程序，并从函数返回新的异常实例。在这种情况下，事件处理将继续到下一个处理程序。可使用`ExceptionContext.chained_exception`获取“链接”异常：

```py
@event.listens_for(Engine, "handle_error", retval=True)
def handle_exception(context):
    if context.chained_exception is not None and \
        "special" in context.chained_exception.message:
        return MySpecialException("failed",
            cause=context.chained_exception)
```

返回`None`的处理程序可以在链中使用；当处理程序返回`None`时，如果有的话，前一个异常实例将保持为传递给下一个处理程序的当前异常。

当引发或返回自定义异常时，SQLAlchemy 将原样引发此新异常，不会被任何 SQLAlchemy 对象包装。如果异常不是`sqlalchemy.exc.StatementError`的子类，则某些功能可能不可用；目前包括 ORM 在自动刷新过程中引发异常时添加有关“自动刷新”的详细提示的功能。

参数：

**context** – 一个`ExceptionContext`对象。有关所有可用成员的详细信息，请参阅此类。

另请参见

支持断开连接场景的新数据库错误代码

## 模式事件

| 对象名称 | 描述 |
| --- | --- |
| [DDL 事件](https://wiki.example.org/sqlalchemy.events.DDLEvents) | 为模式对象定义事件监听器，即`SchemaItem`和其他`SchemaEventTarget`子类，包括`MetaData`、`Table`、`Column`等。 |
| SchemaEventTarget | 用于`DDL 事件`事件目标的元素的基类。 |

```py
class sqlalchemy.events.DDLEvents
```

为模式对象定义事件监听器，即`SchemaItem`和其他`SchemaEventTarget`子类，包括`MetaData`、`Table`、`Column`等。

**创建/删除事件**

在将 CREATE 和 DROP 命令发送到数据库时发出的事件。此类别中的事件钩子包括`DDLEvents.before_create()`、`DDLEvents.after_create()`、`DDLEvents.before_drop()`和`DDLEvents.after_drop()`。

当使用架构级别方法时，例如`MetaData.create_all()`和`MetaData.drop_all()`时会发出这些事件。还包括每个对象的创建/删除方法，例如`Table.create()`、`Table.drop()`、`Index.create()`，以及方言特定的方法，例如`ENUM.create()`。

版本 2.0 中的新功能：`DDLEvents`事件钩子现在适用于非表对象，包括约束、索引和方言特定的架构类型。

事件钩子可以直接附加到`Table`对象或`MetaData`集合，以及任何可以使用独立的 SQL 命令单独创建和删除的`SchemaItem`类或对象。这样的类包括`Index`、`Sequence`以及方言特定的类，例如`ENUM`。

使用`DDLEvents.after_create()`事件的示例，在当前连接上自定义事件钩子将在发出`CREATE TABLE`后发出`ALTER TABLE`命令：

```py
from sqlalchemy import create_engine
from sqlalchemy import event
from sqlalchemy import Table, Column, Metadata, Integer

m = MetaData()
some_table = Table('some_table', m, Column('data', Integer))

@event.listens_for(some_table, "after_create")
def after_create(target, connection, **kw):
    connection.execute(text(
        "ALTER TABLE %s SET name=foo_%s" % (target.name, target.name)
    ))

some_engine = create_engine("postgresql://scott:tiger@host/test")

# will emit "CREATE TABLE some_table" as well as the above
# "ALTER TABLE" statement afterwards
m.create_all(some_engine)
```

诸如`ForeignKeyConstraint`、`UniqueConstraint`、`CheckConstraint`等约束对象也可以订阅这些事件，但是它们通常不会产生事件，因为这些对象通常被嵌入在包含的`CREATE TABLE`语句中，并且隐含地从`DROP TABLE`语句中删除。

对于`Index`构造，当执行`CREATE INDEX`时会触发事件钩子，但是当删除表时 SQLAlchemy 通常不会触发`DROP INDEX`，因为这在`DROP TABLE`语句中是隐含的。

从版本 2.0 开始：对于`SchemaItem`对象的支持已经从其之前对`MetaData`和`Table`的支持扩展到还包括`Constraint`及其所有子类、`Index`、`Sequence`以及一些与类型相关的构造，例如`ENUM`。

注意

这些事件钩子只在 SQLAlchemy 的 create/drop 方法范围内触发；并不一定被诸如[alembic](https://alembic.sqlalchemy.org)等工具支持。

**附加事件**

附加事件用于在子模式元素与父元素关联时自定义行为，例如当`Column`与其`Table`关联，当`ForeignKeyConstraint`与`Table`关联等。这些事件包括`DDLEvents.before_parent_attach()`和`DDLEvents.after_parent_attach()`。

**反射事件**

`DDLEvents.column_reflect()` 事件用于拦截和修改数据库表反射进行时的数据库列的 Python 定义。

**与通用 DDL 一起使用**

DDL 事件与`DDL`类和 DDL 子句构造的`ExecutableDDLElement`层次结构密切集成，它们本身适用于监听器可调用：

```py
from sqlalchemy import DDL
event.listen(
    some_table,
    "after_create",
    DDL("ALTER TABLE %(table)s SET name=foo_%(table)s")
)
```

**事件传播到 MetaData 副本**

对于所有`DDLEvent`事件，`propagate=True`关键字参数将确保给定事件处理程序传播到对象的副本，当使用`Table.to_metadata()`方法时会创建这些副本：

```py
from sqlalchemy import DDL

metadata = MetaData()
some_table = Table("some_table", metadata, Column("data", Integer))

event.listen(
    some_table,
    "after_create",
    DDL("ALTER TABLE %(table)s SET name=foo_%(table)s"),
    propagate=True
)

new_metadata = MetaData()
new_table = some_table.to_metadata(new_metadata)
```

上述`DDL`对象将与`some_table`和`new_table``Table`对象的`DDLEvents.after_create()`事件相关联。

另请参阅

事件

`ExecutableDDLElement`

`DDL`

控制 DDL 序列

**成员**

after_create(), after_drop(), after_parent_attach(), before_create(), before_drop(), before_parent_attach(), column_reflect(), dispatch

**类签名**

类`sqlalchemy.events.DDLEvents`（`sqlalchemy.event.Events`）

```py
method after_create(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在发出 CREATE 语句后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'after_create')
def receive_after_create(target, connection, **kw):
    "listen for the 'after_create' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    `SchemaObject`，比如`MetaData`或`Table`，但也包括所有创建/删除对象，如`Index`、`Sequence`等，这些对象是事件的目标。

    新版本 2.0 中：添加了对所有`SchemaItem`对象的支持。

+   `connection` – `Connection`，在其中发出了 CREATE 语句或语句。

+   `**kw` – 与事件相关的额外关键字参数。此字典的内容可能在不同版本之间变化，并包括在元数据级事件中生成的表列表、checkfirst 标志和其他内部事件使用的元素。

`listen()` 也接受 `propagate=True` 修饰符用于此事件；当为 True 时，监听器函数将被建立为目标对象的任何副本，即在使用 `Table.to_metadata()` 生成的那些副本。

```py
method after_drop(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在发出 DROP 语句后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'after_drop')
def receive_after_drop(target, connection, **kw):
    "listen for the 'after_drop' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    `SchemaObject`，如 `MetaData` 或 `Table`，但也包括所有创建/删除对象，如 `Index`、`Sequence` 等，是事件的目标对象。

    新版本 2.0 中：添加了对所有 `SchemaItem` 对象的支持。

+   `connection` – `Connection`，在其中发出了 DROP 语句或语句。

+   `**kw` – 与事件相关的额外关键字参数。此字典的内容可能在不同版本之间变化，并包括在元数据级事件中生成的表列表、checkfirst 标志和其他内部事件使用的元素。

`listen()` 也接受 `propagate=True` 修饰符用于此事件；当为 True 时，监听器函数将被建立为目标对象的任何副本，即在使用 `Table.to_metadata()` 生成的那些副本。

```py
method after_parent_attach(target: SchemaEventTarget, parent: SchemaItem) → None
```

在将 `SchemaItem` 与父 `SchemaItem` 关联后调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'after_parent_attach')
def receive_after_parent_attach(target, parent):
    "listen for the 'after_parent_attach' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 目标对象

+   `parent` – 要将目标附加到的父级。

`listen()` 还接受 `propagate=True` 修改器用于此事件；当为 True 时，监听器函数将为目标对象的任何副本建立，即当使用 `Table.to_metadata()` 生成这些副本时。

```py
method before_create(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在发出 CREATE 语句之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'before_create')
def receive_before_create(target, connection, **kw):
    "listen for the 'before_create' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    `SchemaObject`，如 `MetaData` 或 `Table`，但也包括所有创建/删除对象，如 `Index`、`Sequence` 等，是事件目标的对象。

    版本 2.0 新增支持所有 `SchemaItem` 对象。

+   `connection` – 将发出 CREATE 语句的 `Connection`。

+   `**kw` – 与事件相关的额外关键字参数。该字典的内容可能因版本而异，包括生成元数据级事件的表列表、checkfirst 标志和其他内部事件使用的元素。

`listen()` 接受 `propagate=True` 修改器用于此事件；当为 True 时，监听器函数将为目标对象的任何副本建立，即当使用 `Table.to_metadata()` 生成这些副本时。

`listen()` 接受 `insert=True` 修改器用于此事件；当为 True 时，监听器函数将在发现时被添加到内部事件列表之前，并在不传递此参数的已注册监听器函数之前执行。

```py
method before_drop(target: SchemaEventTarget, connection: Connection, **kw: Any) → None
```

在发出 DROP 语句之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'before_drop')
def receive_before_drop(target, connection, **kw):
    "listen for the 'before_drop' event"

    # ... (event handling logic) ...
```

参数：

+   `target` –

    `SchemaObject`，如 `MetaData` 或 `Table`，但也包括所有创建/删除对象，如 `Index`、`Sequence` 等，是事件目标的对象。

    版本 2.0 新增支持所有 `SchemaItem` 对象。

+   `connection` – 将发出 DROP 语句或语句的`Connection`。

+   `**kw` – 与事件相关的其他关键字参数。此字典的内容可能在不同版本中有所变化，包括为元数据级事件生成的表列表、checkfirst 标志以及内部事件使用的其他元素。

`listen()`也接受`propagate=True`修饰符用于此事件；当为 True 时，监听函数将为目标对象的任何副本建立，即当使用`Table.to_metadata()`生成副本时。

```py
method before_parent_attach(target: SchemaEventTarget, parent: SchemaItem) → None
```

在将`SchemaItem`与父`SchemaItem`关联之前调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'before_parent_attach')
def receive_before_parent_attach(target, parent):
    "listen for the 'before_parent_attach' event"

    # ... (event handling logic) ...
```

参数：

+   `target` – 目标对象

+   `parent` – 要将目标附加到的父级。

`listen()`也接受`propagate=True`修饰符用于此事件；当为 True 时，监听函数将为目标对象的任何副本建立，即当使用`Table.to_metadata()`生成副本时。

```py
method column_reflect(inspector: Inspector, table: Table, column_info: ReflectedColumn) → None
```

在反射`Table`时检索每个‘column info’单元时调用。

示例参数形式：

```py
from sqlalchemy import event

@event.listens_for(SomeSchemaClassOrObject, 'column_reflect')
def receive_column_reflect(inspector, table, column_info):
    "listen for the 'column_reflect' event"

    # ... (event handling logic) ...
```

此事件最容易通过将其应用于特定的`MetaData`实例来使用，在那里它将对该`MetaData`中的所有`Table`对象产生影响：

```py
metadata = MetaData()

@event.listens_for(metadata, 'column_reflect')
def receive_column_reflect(inspector, table, column_info):
    # receives for all Table objects that are reflected
    # under this MetaData

# will use the above event hook
my_table = Table("my_table", metadata, autoload_with=some_engine)
```

新版本 1.4.0b2 中：`DDLEvents.column_reflect()`钩子现在也可以应用于`MetaData`对象以及`MetaData`类本身，它将对与目标`MetaData`关联的所有`Table`对象进行操作。

它也可以应用于整个`Table`类：

```py
from sqlalchemy import Table

@event.listens_for(Table, 'column_reflect')
def receive_column_reflect(inspector, table, column_info):
    # receives for all Table objects that are reflected
```

它还可以应用到使用`Table.listeners`参数反射的特定`Table`上：

```py
t1 = Table(
    "my_table",
    autoload_with=some_engine,
    listeners=[
        ('column_reflect', receive_column_reflect)
    ]
)
```

通过方言返回的列信息字典被传递，并且可以被修改。字典是由`Inspector.get_columns()`返回的列表中的每个元素返回的：

> +   `name` - 列的名称，应用于`Column.name`参数。
> +   
> +   `type` - 此列的类型，应该是`TypeEngine`的实例，应用于`Column.type`参数。
> +   
> +   `nullable` - 如果列为 NULL 或 NOT NULL 的布尔标志，应用于`Column.nullable`参数。
> +   
> +   `default` - 列的服务器默认值。通常指定为简单的字符串 SQL 表达式，但事件也可以传递一个`FetchedValue`、`DefaultClause`或`text()`对象。应用于`Column.server_default`参数。

在对此字典执行任何操作之前调用事件，并且内容可以被修改；以下附加键可以添加到字典中以进一步修改如何构造`Column`：

> +   `key` - 将用于在`.c`集合中访问此`Column`的字符串键；将应用于`Column.key`参数。也用于 ORM 映射。请参阅从反射表自动命名列方案章节的示例。
> +   
> +   `quote` - 强制或取消对列名进行引用；应用于`Column.quote`参数。
> +   
> +   `info` - 一个任意数据的字典，用于跟踪 `Column`，应用于 `Column.info` 参数。

`listen()` 也接受 `propagate=True` 修改器以用于此事件；当为 True 时，监听器函数将被建立为目标对象的任何副本，即在使用 `Table.to_metadata()` 生成的副本。

另请参阅

从反射表自动命名方案 - 在 ORM 映射文档中

拦截列定义 - 在 Automap 文档中

使用数据库无关类型反射 - 在反射数据库对象文档中

```py
attribute dispatch: _Dispatch[_ET] = <sqlalchemy.event.base.DDLEventsDispatch object>
```

回到 _Dispatch 类的引用。

双向的反对 _Dispatch._events

```py
class sqlalchemy.events.SchemaEventTarget
```

是`DDLEvents`事件目标元素的基类。

这包括 `SchemaItem` 以及 `SchemaType`。

**类签名**

类`sqlalchemy.events.SchemaEventTarget` (`sqlalchemy.event.registry.EventTarget`)
