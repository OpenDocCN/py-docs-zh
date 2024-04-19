# 异步 I/O（asyncio）

> 原文：[`docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html`](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)

支持 Python asyncio。包括对 Core 和 ORM 使用的支持，使用与 asyncio 兼容的方言。

版本 1.4 中的新功能。

警告

请阅读 Asyncio 平台安装说明（包括 Apple M1）以获取许多平台的重要平台安装说明，包括**Apple M1 架构**。

另请参阅

Core 和 ORM 的异步 IO 支持 - 初始功能公告

异步 IO 集成 - 展示了在 asyncio 扩展中使用 Core 和 ORM 的示例脚本。

## Asyncio 平台安装说明（包括 Apple M1）

asyncio 扩展仅支持 Python 3。它还依赖于[greenlet](https://pypi.org/project/greenlet/)库。这个依赖在常见的机器平台上默认安装，包括：

```py
x86_64 aarch64 ppc64le amd64 win32
```

对于上述平台，`greenlet`已知提供预构建的 wheel 文件。对于其他平台，**greenlet 不会默认安装**；可以在[Greenlet - Download Files](https://pypi.org/project/greenlet/#files)查看当前的文件列表。请注意**有许多架构被省略，包括 Apple M1**。

要安装 SQLAlchemy 并确保`greenlet`依赖存在，无论使用什么平台，可以按照以下方式安装`[asyncio]` [setuptools extra](https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras)，这也会指示`pip`安装`greenlet`：

```py
pip install sqlalchemy[asyncio]
```

请注意，在没有预构建 wheel 文件的平台上安装`greenlet`意味着`greenlet`将从源代码构建，这要求 Python 的开发库也存在。

## 概要 - Core

对于核心用法，`create_async_engine()`函数创建一个`AsyncEngine`的实例，然后提供传统`Engine` API 的异步版本。`AsyncEngine`通过其`AsyncEngine.connect()`和`AsyncEngine.begin()`方法提供一个`AsyncConnection`，这两个方法都提供异步上下文管理器。`AsyncConnection`然后可以使用`AsyncConnection.execute()`方法来执行语句以提供一个缓冲的`Result`，或者使用`AsyncConnection.stream()`方法来提供一个流式的服务器端`AsyncResult`：

```py
>>> import asyncio

>>> from sqlalchemy import Column
>>> from sqlalchemy import MetaData
>>> from sqlalchemy import select
>>> from sqlalchemy import String
>>> from sqlalchemy import Table
>>> from sqlalchemy.ext.asyncio import create_async_engine

>>> meta = MetaData()
>>> t1 = Table("t1", meta, Column("name", String(50), primary_key=True))

>>> async def async_main() -> None:
...     engine = create_async_engine("sqlite+aiosqlite://", echo=True)
...
...     async with engine.begin() as conn:
...         await conn.run_sync(meta.drop_all)
...         await conn.run_sync(meta.create_all)
...
...         await conn.execute(
...             t1.insert(), [{"name": "some name 1"}, {"name": "some name 2"}]
...         )
...
...     async with engine.connect() as conn:
...         # select a Result, which will be delivered with buffered
...         # results
...         result = await conn.execute(select(t1).where(t1.c.name == "some name 1"))
...
...         print(result.fetchall())
...
...     # for AsyncEngine created in function scope, close and
...     # clean-up pooled connections
...     await engine.dispose()

>>> asyncio.run(async_main())
BEGIN  (implicit)
...
CREATE  TABLE  t1  (
  name  VARCHAR(50)  NOT  NULL,
  PRIMARY  KEY  (name)
)
...
INSERT  INTO  t1  (name)  VALUES  (?)
[...]  [('some name 1',),  ('some name 2',)]
COMMIT
BEGIN  (implicit)
SELECT  t1.name
FROM  t1
WHERE  t1.name  =  ?
[...]  ('some name 1',)
[('some name 1',)]
ROLLBACK 
```

上面，`AsyncConnection.run_sync()`方法可用于调用特殊的 DDL 函数，例如`MetaData.create_all()`，这些函数不包括可等待的钩子。

提示

在使用`AsyncEngine`对象的范围内调用`await`来调用`AsyncEngine.dispose()`方法是明智的，如上例中的`async_main`函数所示。这确保了连接池保持的任何连接在可等待的上下文中被正确处理。与使用阻塞 IO 不同，SQLAlchemy 无法在`__del__`或弱引用终结器等方法中正确处理这些连接，因为没有机会调用`await`。当引擎超出范围时未显式处理引擎可能导致发出到标准输出的警告，类似于`RuntimeError: Event loop is closed`的形式在垃圾回收中。

`AsyncConnection` 还提供了一个“流式” API，通过 `AsyncConnection.stream()` 方法返回一个 `AsyncResult` 对象。该结果对象使用服务器端游标并提供了一个 async/await API，比如一个异步迭代器：

```py
async with engine.connect() as conn:
    async_result = await conn.stream(select(t1))

    async for row in async_result:
        print("row: %s" % (row,))
```

## 概要 - ORM

使用 2.0 风格 查询，`AsyncSession` 类提供了完整的 ORM 功能。

在默认使用模式下，必须特别小心，以避免涉及 ORM 关系和列属性的 惰性加载 或其他已过期的属性访问；下一节 在使用 AsyncSession 时防止隐式 IO 对此进行了详细说明。

警告

一个 `AsyncSession` 实例**不能安全地用于多个并发任务**。请参阅章节 在并发任务中使用 AsyncSession 和 会话是线程安全的吗？ AsyncSession 是否安全用于共享在并发任务中？ 了解背景信息。

下面的示例演示了一个完整的示例，包括映射器和会话配置：

```py
>>> from __future__ import annotations

>>> import asyncio
>>> import datetime
>>> from typing import List

>>> from sqlalchemy import ForeignKey
>>> from sqlalchemy import func
>>> from sqlalchemy import select
>>> from sqlalchemy.ext.asyncio import AsyncAttrs
>>> from sqlalchemy.ext.asyncio import async_sessionmaker
>>> from sqlalchemy.ext.asyncio import AsyncSession
>>> from sqlalchemy.ext.asyncio import create_async_engine
>>> from sqlalchemy.orm import DeclarativeBase
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship
>>> from sqlalchemy.orm import selectinload

>>> class Base(AsyncAttrs, DeclarativeBase):
...     pass

>>> class B(Base):
...     __tablename__ = "b"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
...     data: Mapped[str]

>>> class A(Base):
...     __tablename__ = "a"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     data: Mapped[str]
...     create_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())
...     bs: Mapped[List[B]] = relationship()

>>> async def insert_objects(async_session: async_sessionmaker[AsyncSession]) -> None:
...     async with async_session() as session:
...         async with session.begin():
...             session.add_all(
...                 [
...                     A(bs=[B(data="b1"), B(data="b2")], data="a1"),
...                     A(bs=[], data="a2"),
...                     A(bs=[B(data="b3"), B(data="b4")], data="a3"),
...                 ]
...             )

>>> async def select_and_update_objects(
...     async_session: async_sessionmaker[AsyncSession],
... ) -> None:
...     async with async_session() as session:
...         stmt = select(A).order_by(A.id).options(selectinload(A.bs))
...
...         result = await session.execute(stmt)
...
...         for a in result.scalars():
...             print(a, a.data)
...             print(f"created at: {a.create_date}")
...             for b in a.bs:
...                 print(b, b.data)
...
...         result = await session.execute(select(A).order_by(A.id).limit(1))
...
...         a1 = result.scalars().one()
...
...         a1.data = "new data"
...
...         await session.commit()
...
...         # access attribute subsequent to commit; this is what
...         # expire_on_commit=False allows
...         print(a1.data)
...
...         # alternatively, AsyncAttrs may be used to access any attribute
...         # as an awaitable (new in 2.0.13)
...         for b1 in await a1.awaitable_attrs.bs:
...             print(b1, b1.data)

>>> async def async_main() -> None:
...     engine = create_async_engine("sqlite+aiosqlite://", echo=True)
...
...     # async_sessionmaker: a factory for new AsyncSession objects.
...     # expire_on_commit - don't expire objects after transaction commit
...     async_session = async_sessionmaker(engine, expire_on_commit=False)
...
...     async with engine.begin() as conn:
...         await conn.run_sync(Base.metadata.create_all)
...
...     await insert_objects(async_session)
...     await select_and_update_objects(async_session)
...
...     # for AsyncEngine created in function scope, close and
...     # clean-up pooled connections
...     await engine.dispose()

>>> asyncio.run(async_main())
BEGIN  (implicit)
...
CREATE  TABLE  a  (
  id  INTEGER  NOT  NULL,
  data  VARCHAR  NOT  NULL,
  create_date  DATETIME  DEFAULT  (CURRENT_TIMESTAMP)  NOT  NULL,
  PRIMARY  KEY  (id)
)
...
CREATE  TABLE  b  (
  id  INTEGER  NOT  NULL,
  a_id  INTEGER  NOT  NULL,
  data  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(a_id)  REFERENCES  a  (id)
)
...
COMMIT
BEGIN  (implicit)
INSERT  INTO  a  (data)  VALUES  (?)  RETURNING  id,  create_date
[...]  ('a1',)
...
INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)  RETURNING  id
[...]  (1,  'b2')
...
COMMIT
BEGIN  (implicit)
SELECT  a.id,  a.data,  a.create_date
FROM  a  ORDER  BY  a.id
[...]  ()
SELECT  b.a_id  AS  b_a_id,  b.id  AS  b_id,  b.data  AS  b_data
FROM  b
WHERE  b.a_id  IN  (?,  ?,  ?)
[...]  (1,  2,  3)
<A  object  at  ...>  a1
created  at:  ...
<B  object  at  ...>  b1
<B  object  at  ...>  b2
<A  object  at  ...>  a2
created  at:  ...
<A  object  at  ...>  a3
created  at:  ...
<B  object  at  ...>  b3
<B  object  at  ...>  b4
SELECT  a.id,  a.data,  a.create_date
FROM  a  ORDER  BY  a.id
LIMIT  ?  OFFSET  ?
[...]  (1,  0)
UPDATE  a  SET  data=?  WHERE  a.id  =  ?
[...]  ('new data',  1)
COMMIT
new  data
<B  object  at  ...>  b1
<B  object  at  ...>  b2 
```

在上面的示例中，使用可选的 `async_sessionmaker` 助手实例化了 `AsyncSession`，该助手提供了一个带有固定参数集的新 `AsyncSession` 对象的工厂，其中包括将其与特定数据库 URL 关联。然后将其传递给其他方法，在那里它可以在 Python 异步上下文管理器（即 `async with:` 语句）中使用，以便在块结束时自动关闭；这相当于调用 `AsyncSession.close()` 方法。

### 在并发任务中使用 AsyncSession

`AsyncSession` 对象是一个**可变的，有状态的对象**，代表了**正在进行的单个，有状态的数据库事务**。使用 asyncio 的并发任务，例如使用 `asyncio.gather()` 等 API，应该**每个个体任务**使用**单独的** `AsyncSession`。 

参见 Is the Session thread-safe? Is AsyncSession safe to share in concurrent tasks? 部分，了解关于 `Session` 和 `AsyncSession` 如何在并发工作负载中使用的一般描述。### 在使用 AsyncSession 时防止隐式 IO

使用传统 asyncio，应用程序需要避免发生任何可能导致 IO-on-attribute 访问的点。以下是可用于帮助此目的的技术，在前述示例中有很多技术。

+   懒加载关系、延迟列或表达式的属性，或在到期情况下被访问的属性可以利用 `AsyncAttrs` mixin。当将此 mixin 添加到特定类或更一般地添加到 Declarative `Base` 超类时，它提供一个访问器 `AsyncAttrs.awaitable_attrs`，它将任何属性作为可等待对象提供：

    ```py
    from __future__ import annotations

    from typing import List

    from sqlalchemy.ext.asyncio import AsyncAttrs
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship

    class Base(AsyncAttrs, DeclarativeBase):
        pass

    class A(Base):
        __tablename__ = "a"

        # ... rest of mapping ...

        bs: Mapped[List[B]] = relationship()

    class B(Base):
        __tablename__ = "b"

        # ... rest of mapping ...
    ```

    在不使用急加载的情况下，访问新加载的 `A` 实例上的 `A.bs` 集合通常会使用 lazy loading，为了成功，通常会向数据库发出 IO，但在 asyncio 下会失败，因为不允许隐式 IO。在没有任何先前加载操作的情况下直接访问此属性，在 asyncio 下，该属性可以作为可等待对象进行访问，指示 `AsyncAttrs.awaitable_attrs` 前缀：

    ```py
    a1 = (await session.scalars(select(A))).one()
    for b1 in await a1.awaitable_attrs.bs:
        print(b1)
    ```

    `AsyncAttrs` mixin 提供了一个简洁的外观，它覆盖了内部方法，该方法也被 `AsyncSession.run_sync()` 方法使用。

    版本 2.0.13 中的新功能。

    另请参见

    `AsyncAttrs`

+   使用 SQLAlchemy 2.0 中的 Write Only Relationships 特性，可以将集合替换为**只写集合**，这些集合永远不会隐式发出 IO，在此特性下，集合从不读取，只使用显式 SQL 调用查询。在 Asyncio Integration 部分的示例 `async_orm_writeonly.py` 中，可见一个使用 asyncio 的只写集合示例。

    当使用仅写集合时，程序的行为在关于集合方面是简单且易于预测的。然而，缺点是没有任何内置系统可以一次性加载许多这些集合，而是需要手动执行。因此，下面的许多要点涉及在使用 asyncio 时使用传统的懒加载关系时需要更加小心的具体技术。

+   如果不使用`AsyncAttrs`，关系可以声明为`lazy="raise"`，这样默认情况下它们不会尝试发出 SQL。为了加载集合，将使用 eager loading。

+   最有用的急加载策略是`selectinload()`急加载器，在前面的例子中被用来在`await session.execute()`调用的范围内急加载`A.bs`集合：

    ```py
    stmt = select(A).options(selectinload(A.bs))
    ```

+   当构建新对象时，**集合总是被分配一个默认的空集合**，比如上面的例子中的列表：

    ```py
    A(bs=[], data="a2")
    ```

    这允许在刷新`A`对象时，上述`A`对象上的`.bs`集合存在且可读；否则，当刷新`A`时，`.bs`将被卸载并在访问时引发错误。

+   `AsyncSession`是使用`Session.expire_on_commit`设置为 False 进行配置的，这样我们可以在调用`AsyncSession.commit()`之后访问对象的属性，就像在最后一行访问属性时一样：

    ```py
    # create AsyncSession with expire_on_commit=False
    async_session = AsyncSession(engine, expire_on_commit=False)

    # sessionmaker version
    async_session = async_sessionmaker(engine, expire_on_commit=False)

    async with async_session() as session:
        result = await session.execute(select(A).order_by(A.id))

        a1 = result.scalars().first()

        # commit would normally expire all attributes
        await session.commit()

        # access attribute subsequent to commit; this is what
        # expire_on_commit=False allows
        print(a1.data)
    ```

其他指导原则包括：

+   应该避免使用类似`AsyncSession.expire()`的方法，而应该使用`AsyncSession.refresh()`；**如果**绝对需要过期。通常情况下不应该需要过期，因为在使用 asyncio 时通常应该将`Session.expire_on_commit`设置为`False`。

+   使用`AsyncSession.refresh()`可以显式加载懒加载关系，**如果**所需的属性名称被显式传递给`Session.refresh.attribute_names`，例如：

    ```py
    # assume a_obj is an A that has lazy loaded A.bs collection
    a_obj = await async_session.get(A, [1])

    # force the collection to load by naming it in attribute_names
    await async_session.refresh(a_obj, ["bs"])

    # collection is present
    print(f"bs collection: {a_obj.bs}")
    ```

    当然最好一开始就使用急加载，以便无需延迟加载即可设置集合。

    2.0.4 版中新增了对`AsyncSession.refresh()`和底层`Session.refresh()`方法的支持，以强制懒加载的关系加载，如果它们在`Session.refresh.attribute_names`参数中明确命名。在之前的版本中，即使在参数中命名了关系，也会被静默跳过。

+   避免使用文档中记录的 Cascades 中的`all`级联选项，而是明确列出所需的级联特性。`all`级联选项暗示了 refresh-expire 设置，这意味着`AsyncSession.refresh()`方法将使相关对象上的属性过期，但不一定会刷新那些相关对象，假设未在`relationship()`内配置急加载，则将其保留在过期状态。

+   如果使用，应该使用适当的加载器选项来为`deferred()`列进行延迟加载，除了上面注意到的`relationship()`结构。请参阅 Limiting which Columns Load with Column Deferral 了解延迟列加载的背景信息。

+   在 Dynamic Relationship Loaders 中描述的“动态”关系加载器策略默认情况下不与 asyncio 方法兼容。只有在 Running Synchronous Methods and Functions under asyncio 中描述的`AsyncSession.run_sync()`方法内调用时，或者通过使用其`.statement`属性获取普通选择时，它才能直接使用：

    ```py
    user = await session.get(User, 42)
    addresses = (await session.scalars(user.addresses.statement)).all()
    stmt = user.addresses.statement.where(Address.email_address.startswith("patrick"))
    addresses_filter = (await session.scalars(stmt)).all()
    ```

    引入 SQLAlchemy 2.0 版本的 write only 技术完全与 asyncio 兼容，并应优先使用。

    请参阅

    “动态”关系加载器被“只写”所取代 - 迁移到 2.0 样式的注意事项

+   如果在不支持 RETURNING 的数据库（例如 MySQL 8）中使用 asyncio，那么新刷新的对象上将不会有服务器默认值，例如生成的时间戳，除非使用 `Mapper.eager_defaults` 选项。在 SQLAlchemy 2.0 中，这种行为会自动应用于像 PostgreSQL、SQLite 和 MariaDB 这样使用 RETURNING 在插入行时获取新值的后端。### 在 asyncio 下运行同步方法和函数

深度炼金术

这种方法本质上是公开了 SQLAlchemy 能够提供 asyncio 接口的机制。虽然这样做没有技术问题，但总体上这种方法可能被认为是“有争议的”，因为它违背了 asyncio 编程模型的一些核心理念，即任何可能导致 IO 调用的编程语句**必须**有一个 `await` 调用，否则程序不会明确地指出每一行可能发生 IO 的地方。这种方法并没有改变这个一般观念，只是允许一系列同步 IO 指令在函数调用范围内免除这个规则，实质上被打包成一个可等待对象。

作为在 asyncio 事件循环中集成传统 SQLAlchemy “延迟加载”的另一种方法，提供了一种名为 `AsyncSession.run_sync()` 的**可选**方法，它将在一个 greenlet 中运行任何 Python 函数，传统的同步编程概念将在到达数据库驱动程序时转换为使用 `await`。这里的一个假设方法是，一个面向 asyncio 的应用程序可以将与数据库相关的方法打包到使用 `AsyncSession.run_sync()` 调用的函数中。

修改上面的示例，如果我们不为 `A.bs` 集合使用 `selectinload()`，我们可以在一个单独的函数中完成对这些属性访问的处理：

```py
import asyncio

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

def fetch_and_update_objects(session):
  """run traditional sync-style ORM code in a function that will be
 invoked within an awaitable.

 """

    # the session object here is a traditional ORM Session.
    # all features are available here including legacy Query use.

    stmt = select(A)

    result = session.execute(stmt)
    for a1 in result.scalars():
        print(a1)

        # lazy loads
        for b1 in a1.bs:
            print(b1)

    # legacy Query use
    a1 = session.query(A).order_by(A.id).first()

    a1.data = "new data"

async def async_main():
    engine = create_async_engine(
        "postgresql+asyncpg://scott:tiger@localhost/test",
        echo=True,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)

    async with AsyncSession(engine) as session:
        async with session.begin():
            session.add_all(
                [
                    A(bs=[B(), B()], data="a1"),
                    A(bs=[B()], data="a2"),
                    A(bs=[B(), B()], data="a3"),
                ]
            )

        await session.run_sync(fetch_and_update_objects)

        await session.commit()

    # for AsyncEngine created in function scope, close and
    # clean-up pooled connections
    await engine.dispose()

asyncio.run(async_main())
```

在“sync”运行器中运行某些函数的上述方法与在类似 `gevent` 的事件驱动编程库上运行 SQLAlchemy 应用程序的应用程序有一些相似之处。区别如下：

1.  与使用 `gevent` 不同，我们可以继续使用标准的 Python asyncio 事件循环，或任何自定义事件循环，而无需集成到 `gevent` 事件循环中。

1.  没有任何“猴子补丁”。上面的示例使用了真正的 asyncio 驱动程序，底层的 SQLAlchemy 连接池也使用了 Python 内置的 `asyncio.Queue` 来池化连接。

1.  该程序可以自由地在异步/等待代码和使用同步代码的封装函数之间切换，几乎没有性能损失。没有使用“线程执行器”或任何额外的等待器或同步。

1.  底层网络驱动程序也在使用纯 Python asyncio 概念，不使用`gevent`和`eventlet`提供的第三方网络库。## 使用与异步扩展的事件

SQLAlchemy 的事件系统未直接由异步扩展暴露，这意味着目前还没有“异步”版本的 SQLAlchemy 事件处理程序。

但是，由于异步扩展包围了通常的同步 SQLAlchemy API，因此常规的“同步”风格事件处理程序可自由使用，就像没有使用 asyncio 一样。

如下所述，目前有两种策略可以注册给予 asyncio-facing APIs 的事件：

+   事件可以在实例级别（例如特定的`AsyncEngine`实例）上注册，方法是将事件与引用代理对象的`sync`属性关联起来。例如，要针对`AsyncEngine`实例注册`PoolEvents.connect()`事件，请使用其`AsyncEngine.sync_engine`属性作为目标。目标包括：

    > `AsyncEngine.sync_engine`
    > 
    > `AsyncConnection.sync_connection`
    > 
    > `AsyncConnection.sync_engine`
    > 
    > `AsyncSession.sync_session`

+   要在类级别注册事件，针对同一类型的所有实例（例如所有`AsyncSession`实例），请使用相应的同步样式类。例如，要针对`AsyncSession`类注册`SessionEvents.before_commit()`事件，请使用`Session`类作为目标。

+   要在`sessionmaker`级别注册，请使用`async_sessionmaker.sync_session_class`将显式`sessionmaker`与`async_sessionmaker`组合，并将事件与`sessionmaker`相关联。

当在异步 IO 上下文中的事件处理程序中工作时，例如`Connection`等对象将继续以通常的“同步”方式工作，而不需要`await`或`async`使用；当消息最终由异步 IO 数据库适配器接收时，调用样式将透明地转换回异步 IO 调用样式。对于传递了 DBAPI 级别连接的事件，例如`PoolEvents.connect()`，对象是一个符合 pep-249 的“连接”对象，它将同步样式调用适配为异步 IO 驱动程序。

### 带有异步引擎/会话/会话工厂的事件监听器示例

下面是一些与异步 API 构造相关的同步风格事件处理程序的示例：

+   **AsyncEngine 上的核心事件**

    在此示例中，我们将`AsyncEngine.sync_engine`属性作为`ConnectionEvents`和`PoolEvents`的目标：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.engine import Engine
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")

    # connect event on instance of Engine
    @event.listens_for(engine.sync_engine, "connect")
    def my_on_connect(dbapi_con, connection_record):
        print("New DBAPI connection:", dbapi_con)
        cursor = dbapi_con.cursor()

        # sync style API use for adapted DBAPI connection / cursor
        cursor.execute("select 'execute from event'")
        print(cursor.fetchone()[0])

    # before_execute event on all Engine instances
    @event.listens_for(Engine, "before_execute")
    def my_before_execute(
        conn,
        clauseelement,
        multiparams,
        params,
        execution_options,
    ):
        print("before execute!")

    async def go():
        async with engine.connect() as conn:
            await conn.execute(text("select 1"))
        await engine.dispose()

    asyncio.run(go())
    ```

    输出：

    ```py
    New DBAPI connection: <AdaptedConnection <asyncpg.connection.Connection object at 0x7f33f9b16960>>
    execute from event
    before execute!
    ```

+   **AsyncSession 上的 ORM 事件**

    在此示例中，我们将`AsyncSession.sync_session`作为`SessionEvents`的目标：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.ext.asyncio import AsyncSession
    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.orm import Session

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")

    session = AsyncSession(engine)

    # before_commit event on instance of Session
    @event.listens_for(session.sync_session, "before_commit")
    def my_before_commit(session):
        print("before commit!")

        # sync style API use on Session
        connection = session.connection()

        # sync style API use on Connection
        result = connection.execute(text("select 'execute from event'"))
        print(result.first())

    # after_commit event on all Session instances
    @event.listens_for(Session, "after_commit")
    def my_after_commit(session):
        print("after commit!")

    async def go():
        await session.execute(text("select 1"))
        await session.commit()

        await session.close()
        await engine.dispose()

    asyncio.run(go())
    ```

    输出：

    ```py
    before commit!
    execute from event
    after commit!
    ```

+   **async_sessionmaker 上的 ORM 事件**

    对于这种用例，我们将`sessionmaker`作为事件目标，然后使用`async_sessionmaker.sync_session_class`参数将其分配给`async_sessionmaker`：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy.ext.asyncio import async_sessionmaker
    from sqlalchemy.orm import sessionmaker

    sync_maker = sessionmaker()
    maker = async_sessionmaker(sync_session_class=sync_maker)

    @event.listens_for(sync_maker, "before_commit")
    def before_commit(session):
        print("before commit")

    async def main():
        async_session = maker()

        await async_session.commit()

    asyncio.run(main())
    ```

    输出：

    ```py
    before commit
    ```

### 在连接池和其他事件中使用仅 awaitable 的驱动程序方法

如上一节所讨论的那样，诸如`PoolEvents`之类的事件处理程序接收到一个同步风格的“DBAPI”连接，这是 SQLAlchemy asyncio 方言提供的一个包装对象，用于将底层的 asyncio“driver”连接适配成 SQLAlchemy 内部可以使用的连接。当用户定义的事件处理程序需要直接使用最终的“driver”连接，并且只使用该驱动连接上的 awaitable 方法时，就会出现一种特殊的用例。其中一个例子是 asyncpg 驱动程序提供的`.set_type_codec()`方法。

为了适应这种用例，SQLAlchemy 的`AdaptedConnection`类提供了一个方法`AdaptedConnection.run_async()`，允许在事件处理程序或其他 SQLAlchemy 内部的“同步”上下文中调用一个 awaitable 函数。这个方法直接类似于`AsyncConnection.run_sync()`方法，它允许一个同步风格的方法在 async 下运行。

应该向`AdaptedConnection.run_async()`传递一个接受内部“driver”连接作为单个参数的函数，并返回一个 awaitable，该 awaitable 将由`AdaptedConnection.run_async()`方法调用。给定的函数本身不需要声明为`async`；它完全可以是 Python 的`lambda:`，因为返回的 awaitable 值将在返回后被调用：

```py
from sqlalchemy import event
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(...)

@event.listens_for(engine.sync_engine, "connect")
def register_custom_types(dbapi_connection, *args):
    dbapi_connection.run_async(
        lambda connection: connection.set_type_codec(
            "MyCustomType",
            encoder,
            decoder,  # ...
        )
    )
```

上面，传递给`register_custom_types`事件处理程序的对象是`AdaptedConnection`的一个实例，它提供了一个类似 DBAPI 的接口，用于访问底层的仅 async 的驱动级连接对象。然后，`AdaptedConnection.run_async()`方法提供了访问底层驱动程序级连接的 awaitable 环境。

版本 1.4.30 中的新功能。

## 使用多个 asyncio 事件循环

当一个应用程序同时使用多个事件循环时，例如在罕见的情况下将 asyncio 与多线程结合使用时，当使用默认的池实现时，不应该将相同的 `AsyncEngine` 与不同的事件循环共享。

如果一个 `AsyncEngine` 从一个事件循环传递到另一个事件循环，则在重新使用之前应调用 `AsyncEngine.dispose()` 方法。未能这样做可能会导致类似于 `Task <Task pending ...> got Future attached to a different loop` 的 `RuntimeError`。

如果同一个引擎必须在不同的循环之间共享，则应配置为使用 `NullPool` 来禁用池，防止引擎重复使用任何连接：

```py
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.pool import NullPool

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@host/dbname",
    poolclass=NullPool,
)
```

## 使用 asyncio scoped session

在线程化的 SQLAlchemy 中使用的“scoped session”模式，使用适应版本称为 `async_scoped_session`，在 asyncio 中也是可用的。

提示

SQLAlchemy 通常不推荐为新开发使用“scoped”模式，因为它依赖于必须在线程或任务完成后显式清除的可变全局状态。特别是在使用 asyncio 时，直接将 `AsyncSession` 传递给需要它的可等待函数可能是一个更好的主意。

在使用 `async_scoped_session` 时，由于在 asyncio 上下文中没有“线程本地”概念，必须为构造函数提供“scopefunc”参数。下面的示例说明了使用 `asyncio.current_task()` 函数来实现此目的：

```py
from asyncio import current_task

from sqlalchemy.ext.asyncio import (
    async_scoped_session,
    async_sessionmaker,
)

async_session_factory = async_sessionmaker(
    some_async_engine,
    expire_on_commit=False,
)
AsyncScopedSession = async_scoped_session(
    async_session_factory,
    scopefunc=current_task,
)
some_async_session = AsyncScopedSession()
```

警告

`async_scoped_session` 使用的“scopefunc”在任务中被调用**任意次数**，每次访问底层 `AsyncSession` 时都会调用该函数。因此，该函数应该是**幂等**且轻量级的，并且不应尝试创建或改变任何状态，例如建立回调等。

警告

在作用域中使用 `current_task()` 作为“键”要求必须从最外层的可等待对象中调用 `async_scoped_session.remove()` 方法，以确保任务完成时从注册表中删除键，否则任务句柄和 `AsyncSession` 将继续驻留在内存中，从根本上创建了内存泄漏。请参阅以下示例，该示例说明了 `async_scoped_session.remove()` 的正确用法。

`async_scoped_session` 包含与 `scoped_session` 类似的 **代理行为**，这意味着它可以直接作为 `AsyncSession` 对待，需要注意通常需要使用 `await` 关键字，包括 `async_scoped_session.remove()` 方法：

```py
async def some_function(some_async_session, some_object):
    # use the AsyncSession directly
    some_async_session.add(some_object)

    # use the AsyncSession via the context-local proxy
    await AsyncScopedSession.commit()

    # "remove" the current proxied AsyncSession for the local context
    await AsyncScopedSession.remove()
```

新版版本 1.4.19。  ## 使用 Inspector 检查模式对象

SQLAlchemy 尚未提供 `Inspector` 的 asyncio 版本（介绍请参见 使用 Inspector 进行细粒度反射），但是可以通过利用 `AsyncConnection.run_sync()` 方法来在 asyncio 上下文中使用现有接口：

```py
import asyncio

from sqlalchemy import inspect
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost/test")

def use_inspector(conn):
    inspector = inspect(conn)
    # use the inspector
    print(inspector.get_view_names())
    # return any value to the caller
    return inspector.get_table_names()

async def async_main():
    async with engine.connect() as conn:
        tables = await conn.run_sync(use_inspector)

asyncio.run(async_main())
```

另请参见

反射数据库对象

运行时检查 API

## 引擎 API 文档

| 对象名称 | 描述 |
| --- | --- |
| async_engine_from_config(configuration[, prefix], **kwargs) | 使用配置字典创建一个新的 AsyncEngine 实例。 |
| AsyncConnection | 一个用于`Connection`的 asyncio 代理。 |
| AsyncEngine | 一个用于`Engine`的 asyncio 代理。 |
| AsyncTransaction | `Transaction`的一个 asyncio 代理。 |
| create_async_engine(url, **kw) | 创建一个新的异步引擎实例。 |
| create_async_pool_from_url(url, **kwargs) | 创建一个新的异步引擎实例。 |

```py
function sqlalchemy.ext.asyncio.create_async_engine(url: str | URL, **kw: Any) → AsyncEngine
```

创建一个新的异步引擎实例。

传递给`create_async_engine()`的参数基本与传递给`create_engine()`的参数相同。指定的方言必须是支持 asyncio 的方言，例如 asyncpg。

1.4 版的新功能。

参数：

**async_creator** –

一个异步可调用函数，返回一个驱动级别的 asyncio 连接。如果给定，该函数不应该接受任何参数，并从底层的 asyncio 数据库驱动程序返回一个新的 asyncio 连接；连接将被包装在适当的结构中，以便与`AsyncEngine`一起使用。请注意，URL 中指定的参数在此处不适用，创建函数应该使用自己的连接参数。

此参数是`create_engine()`函数的 asyncio 等效参数。

2.0.16 版的新功能。

```py
function sqlalchemy.ext.asyncio.async_engine_from_config(configuration: Dict[str, Any], prefix: str = 'sqlalchemy.', **kwargs: Any) → AsyncEngine
```

使用配置字典创建一个新的 AsyncEngine 实例。

这个函数类似于 SQLAlchemy 核心中的`engine_from_config()`函数，不同之处在于所请求的方言必须是类似于 asyncpg 这样的支持 asyncio 的方言。该函数的参数签名与`engine_from_config()`相同。

1.4.29 版的新功能。

```py
function sqlalchemy.ext.asyncio.create_async_pool_from_url(url: str | URL, **kwargs: Any) → Pool
```

创建一个新的异步引擎实例。

传递给`create_async_pool_from_url()`的参数基本与传递给`create_pool_from_url()`的参数相同。指定的方言必须是支持 asyncio 的方言，例如 asyncpg。

2.0.10 版的新功能。

```py
class sqlalchemy.ext.asyncio.AsyncEngine
```

一个`Engine`的 asyncio 代理。

`AsyncEngine`是使用`create_async_engine()`函数获取的：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("postgresql+asyncpg://user:pass@host/dbname")
```

新版本 1.4 中新增。

**成员**

begin(), clear_compiled_cache(), connect(), dialect, dispose(), driver, echo, engine, execution_options(), get_execution_options(), name, pool, raw_connection(), sync_engine, update_execution_options(), url

**类签名**

类`sqlalchemy.ext.asyncio.AsyncEngine` (`sqlalchemy.ext.asyncio.base.ProxyComparable`, `sqlalchemy.ext.asyncio.AsyncConnectable`)

```py
method begin() → AsyncIterator[AsyncConnection]
```

返回一个上下文管理器，当进入时将提供一个已建立 `AsyncTransaction` 的 `AsyncConnection`。

例如：

```py
async with async_engine.begin() as conn:
    await conn.execute(
        text("insert into table (x, y, z) values (1, 2, 3)")
    )
    await conn.execute(text("my_special_procedure(5)"))
```

```py
method clear_compiled_cache() → None
```

清除与方言关联的编译缓存。

代表`Engine`类的代理，代表`AsyncEngine`类。

这仅适用于通过`create_engine.query_cache_size`参数建立的内置缓存。它不会影响通过`Connection.execution_options.compiled_cache`参数传递的任何字典缓存。

新版本 1.4 中新增。

```py
method connect() → AsyncConnection
```

返回一个`AsyncConnection`对象。

当作为异步上下文管理器输入时，`AsyncConnection`将从底层连接池中获取数据库连接：

```py
async with async_engine.connect() as conn:
    result = await conn.execute(select(user_table))
```

`AsyncConnection`也可以通过调用其`AsyncConnection.start()`方法在上下文管理器之外启动。

```py
attribute dialect
```

代理`AsyncEngine`类的`Engine.dialect`属性。

```py
method async dispose(close: bool = True) → None
```

处置此`AsyncEngine`使用的连接池。

参数：

**关闭** –

如果将其默认值保留为`True`，则会完全关闭所有**当前已签入**的数据库连接。然而，仍在使用的连接将**不会**被关闭，但它们将不再与此`Engine`关联，因此当它们被单独关闭时，它们所关联的`Pool`最终将被垃圾回收，如果已经在签入时关闭，则将完全关闭。

如果设置为`False`，则前一个连接池将被取消引用，否则不会以任何方式触及。

另请参阅

`Engine.dispose()`

```py
attribute driver
```

此`Engine`正在使用的`Dialect`的驱动程序名称。

代理`AsyncEngine`类的`Engine`类。

```py
attribute echo
```

当为`True`时，启用此元素的日志输出。

代理`AsyncEngine`类的`Engine`类。

这将设置此元素类和对象引用的命名空间的 Python 日志级别。布尔值`True`表示将为记录器设置日志级别`logging.INFO`，而字符串值`debug`将将日志级别设置为`logging.DEBUG`。

```py
attribute engine
```

返回此`Engine`。

代理`AsyncEngine`类的`Engine`类。

用于接受同一变量内的`Connection` / `Engine`对象的传统方案。

```py
method execution_options(**opt: Any) → AsyncEngine
```

返回一个新的 `AsyncEngine`，该引擎将以给定的执行选项提供 `AsyncConnection` 对象。

代理自 `Engine.execution_options()`。请参阅该方法了解详情。

```py
method get_execution_options() → _ExecuteOptions
```

获取执行期间将生效的非 SQL 选项。

代表 `AsyncEngine` 类的 `Engine` 类的代理。

另请参阅

`Engine.execution_options()`

```py
attribute name
```

此 `Engine` 使用的 `Dialect` 的字符串名称。

代表 `AsyncEngine` 类的 `Engine` 类的代理。

```py
attribute pool
```

代表 `AsyncEngine` 类的 `Engine.pool` 属性的代理。

```py
method async raw_connection() → PoolProxiedConnection
```

从连接池返回“原始” DBAPI 连接。

另请参阅

使用 Driver SQL 和原始 DBAPI 连接

```py
attribute sync_engine: Engine
```

此 `AsyncEngine` 代理请求到同步样式的 `Engine`。

此实例可用作事件目标。

另请参阅

与 asyncio 扩展一起使用事件

```py
method update_execution_options(**opt: Any) → None
```

更新此 `Engine` 的默认执行选项字典。

代表 `AsyncEngine` 类的 `Engine` 类的代理。

**opt 中给定的键/值将添加到将用于所有连接的默认执行选项中。此字典的初始内容可以通过 `execution_options` 参数发送到 `create_engine()`。

另请参阅

`Connection.execution_options()`

`Engine.execution_options()`

```py
attribute url
```

代表`AsyncEngine`类的`Engine.url`属性的代理。

```py
class sqlalchemy.ext.asyncio.AsyncConnection
```

一个`Connection`的 asyncio 代理。

`AsyncConnection`通过`AsyncEngine.connect()`方法获取：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("postgresql+asyncpg://user:pass@host/dbname")

async with engine.connect() as conn:
    result = await conn.execute(select(table))
```

版本 1.4 中新增。

**成员**

aclose()，begin()，begin_nested()，close()，closed，commit()，connection，default_isolation_level，dialect，exec_driver_sql()，execute()，execution_options()，get_nested_transaction()，get_raw_connection()，get_transaction()，in_nested_transaction()，in_transaction()，info，invalidate()，invalidated，rollback()，run_sync()，scalar()，scalars()，start()，stream()，stream_scalars()，sync_connection，sync_engine

**类签名**

class `sqlalchemy.ext.asyncio.AsyncConnection` (`sqlalchemy.ext.asyncio.base.ProxyComparable`, `sqlalchemy.ext.asyncio.base.StartableContext`, `sqlalchemy.ext.asyncio.AsyncConnectable`)

```py
method async aclose() → None
```

`AsyncConnection.close()`的同义词。

`AsyncConnection.aclose()`名称专门用于支持 Python 标准库`@contextlib.aclosing`上下文管理器函数。

版本 2.0.20 中的新功能。

```py
method begin() → AsyncTransaction
```

在自动开始之前开始事务。

```py
method begin_nested() → AsyncTransaction
```

开始一个嵌套事务并返回事务句柄。

```py
method async close() → None
```

关闭此`AsyncConnection`。

这也会导致回滚事务（如果存在）。

```py
attribute closed
```

如果此连接已关闭，则返回 True。

代理`Connection`类，代表`AsyncConnection`类。

```py
method async commit() → None
```

提交当前正在进行的事务。

如果已启动事务，则此方法提交当前事务。如果未启动事务，则该方法不起作用，假定连接处于非失效状态。

每当首次执行语句或调用`Connection.begin()`方法时，都会自动在`Connection`上开始事务。

```py
attribute connection
```

未实现异步；调用`AsyncConnection.get_raw_connection()`。

```py
attribute default_isolation_level
```

与正在使用的`Dialect`相关联的初始连接时间隔离级别。

代理`Connection`类，代表`AsyncConnection`类。

此值独立于`Connection.execution_options.isolation_level`和`Engine.execution_options.isolation_level`执行选项，并由`Dialect`在创建第一个连接时确定，通过针对数据库执行 SQL 查询以获取当前隔离级别，然后再发出任何其他命令。

调用此访问器不会触发任何新的 SQL 查询。

另请参阅

`Connection.get_isolation_level()` - 查看当前实际隔离级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

`Connection.execution_options.isolation_level` - 设置每个`Connection`的隔离级别

```py
attribute dialect
```

代表`AsyncConnection`类的`Connection.dialect`属性的代理。

```py
method async exec_driver_sql(statement: str, parameters: _DBAPIAnyExecuteParams | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

执行驱动程序级别的 SQL 字符串并返回缓冲的`Result`。

```py
method async execute(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

执行 SQL 语句构造并返回一个缓冲的`Result`。

参数：

+   `object` –

    要执行的语句。这始终是一个同时存在于`ClauseElement`和`Executable`层次结构中的对象，包括：

    +   `Select` - `Select`操作

    +   `Insert`, `Update`, `Delete`

    +   `TextClause` 和 `TextualSelect`

    +   `DDL` 和继承自`ExecutableDDLElement`的对象

+   `parameters` – 将绑定到语句中的参数。这可以是参数名称到值的字典，也可以是可变序列（例如列表）的字典。当传递一个字典列表时，底层语句执行将使用 DBAPI `cursor.executemany()`方法。当传递单个字典时，将使用 DBAPI `cursor.execute()`方法。

+   `execution_options` – 可选的执行选项字典，将与语句执行关联。该字典可以提供`Connection.execution_options()`接受的选项的子集。

返回：

一个`Result`对象。

```py
method async execution_options(**opt: Any) → AsyncConnection
```

设置在执行期间生效的非 SQL 选项。

返回此`AsyncConnection`对象，其中添加了新选项。

有关此方法的完整详情，请参阅`Connection.execution_options()`。

```py
method get_nested_transaction() → AsyncTransaction | None
```

返回一个表示当前嵌套（保存点）事务的`AsyncTransaction`，如果有的话。

这将使用底层同步连接的`Connection.get_nested_transaction()`方法获取当前`Transaction`，然后在新的`AsyncTransaction`对象中进行代理。

新版本 1.4.0b2 中引入。

```py
method async get_raw_connection() → PoolProxiedConnection
```

返回此`AsyncConnection`正在使用的池化 DBAPI 级连接。

这是一个 SQLAlchemy 连接池代理连接，然后具有属性`_ConnectionFairy.driver_connection`，该属性引用实际的驱动程序连接。其`_ConnectionFairy.dbapi_connection`则指代一个`AdaptedConnection`实例，将驱动程序连接适配为 DBAPI 协议。

```py
method get_transaction() → AsyncTransaction | None
```

返回一个表示当前事务的`AsyncTransaction`，如果有的话。

这将使用底层同步连接的`Connection.get_transaction()`方法获取当前`Transaction`，然后在新的`AsyncTransaction`对象中进行代理。

新版本 1.4.0b2 中引入。

```py
method in_nested_transaction() → bool
```

如果事务正在进行中，则返回 True。

新版本 1.4.0b2 中引入。

```py
method in_transaction() → bool
```

如果事务正在进行中，则返回 True。

```py
attribute info
```

返回底层`Connection`的`Connection.info`字典。

此字典可自由编写，以将用户定义的状态与数据库连接关联起来。

仅当`AsyncConnection`当前已连接时才可用此属性。如果`AsyncConnection.closed`属性为`True`，则访问此属性将引发`ResourceClosedError`。 

新版本为 1.4.0b2。

```py
method async invalidate(exception: BaseException | None = None) → None
```

使与此`Connection`相关联的基础 DBAPI 连接无效。

参见方法`Connection.invalidate()`，了解此方法的详细信息。

```py
attribute invalidated
```

如果此连接已失效，则返回 True。

代理了`AsyncConnection`类的`Connection`类。

但这并不表示连接是否在池级别上失效。

```py
method async rollback() → None
```

回滚当前正在进行的事务。

如果已启动事务，则此方法将回滚当前事务。如果未启动事务，则该方法不起作用。如果已启动事务并且连接处于无效状态，则使用此方法清除事务。

当首次执行语句或调用`Connection.begin()`方法时，将自动在`Connection`上启动事务。

```py
method async run_sync(fn: ~typing.Callable[[~typing.Concatenate[~sqlalchemy.engine.base.Connection, ~_P]], ~sqlalchemy.ext.asyncio.engine._T], *arg: ~typing.~_P, **kw: ~typing.~_P) → _T
```

调用给定的同步（即非异步）可调用对象，并将同步风格的`Connection`作为第一个参数传递。

此方法允许在异步应用程序的上下文中运行传统的同步 SQLAlchemy 函数。

例如：

```py
def do_something_with_core(conn: Connection, arg1: int, arg2: str) -> str:
  '''A synchronous function that does not require awaiting

 :param conn: a Core SQLAlchemy Connection, used synchronously

 :return: an optional return value is supported

 '''
    conn.execute(
        some_table.insert().values(int_col=arg1, str_col=arg2)
    )
    return "success"

async def do_something_async(async_engine: AsyncEngine) -> None:
  '''an async function that uses awaiting'''

    async with async_engine.begin() as async_conn:
        # run do_something_with_core() with a sync-style
        # Connection, proxied into an awaitable
        return_code = await async_conn.run_sync(do_something_with_core, 5, "strval")
        print(return_code)
```

通过在一个特别的被监控的 greenlet 中运行给定的可调用对象，此方法将一直维持 asyncio 事件循环直到数据库连接。

`AsyncConnection.run_sync()`的最基本用法是调用诸如`MetaData.create_all()`之类的方法，给定需要提供给`MetaData.create_all()`作为`Connection`对象的`AsyncConnection`：

```py
# run metadata.create_all(conn) with a sync-style Connection,
# proxied into an awaitable
with async_engine.begin() as conn:
    await conn.run_sync(metadata.create_all)
```

注意

提供的可调用对象在 asyncio 事件循环内联调用，并且将在传统 IO 调用上阻塞。此可调用对象内的 IO 应仅调用进入 SQLAlchemy 的 asyncio 数据库 API，这些 API 将被正确地适应到 greenlet 上下文中。

另请参阅

`AsyncSession.run_sync()`

在 asyncio 下运行同步方法和函数

```py
method async scalar(statement: Executable, parameters: _CoreSingleExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → Any
```

执行 SQL 语句构造并返回标量对象。

此方法是在调用`Connection.execute()`方法后调用`Result.scalar()`方法的简写。参数是等效的。

返回：

代表返回的第一行的第一列的标量 Python 值。

```py
method async scalars(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → ScalarResult[Any]
```

执行 SQL 语句构造并返回标量对象。

此方法是在调用`Connection.execute()`方法后调用`Result.scalars()`方法的简写。参数是等效的。

返回：

一个`ScalarResult`对象。

版本 1.4.24 中的新功能。

```py
method async start(is_ctxmanager: bool = False) → AsyncConnection
```

在使用 Python `with:` 块之外启动此`AsyncConnection`对象的上下文。

```py
method stream(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → AsyncIterator[AsyncResult[Any]]
```

执行语句并返回一个产生`AsyncResult`对象的可等待对象。

例如：

```py
result = await conn.stream(stmt):
async for row in result:
    print(f"{row}")
```

`AsyncConnection.stream()`方法支持可选的上下文管理器用法，针对`AsyncResult`对象，如下所示：

```py
async with conn.stream(stmt) as result:
    async for row in result:
        print(f"{row}")
```

在上述模式中，即使迭代器被异常抛出中断，`AsyncResult.close()`方法也会无条件地被调用。然而，上下文管理器的使用仍然是可选的，并且该函数可以以`async with fn():`或`await fn()`的方式调用。

新增于版本 2.0.0b3：增加了上下文管理器支持

返回：

将产生一个可等待对象，该对象将生成一个`AsyncResult`对象。

另见

`AsyncConnection.stream_scalars()`

```py
method stream_scalars(statement: Executable, parameters: _CoreSingleExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → AsyncIterator[AsyncScalarResult[Any]]
```

执行语句并返回一个可等待的`AsyncScalarResult`对象。

例如：

```py
result = await conn.stream_scalars(stmt)
async for scalar in result:
    print(f"{scalar}")
```

此方法是在调用`Connection.stream()`方法后调用`AsyncResult.scalars()`方法的简写。参数是等效的。

`AsyncConnection.stream_scalars()`方法支持针对`AsyncScalarResult`对象的可选上下文管理器使用，如下所示：

```py
async with conn.stream_scalars(stmt) as result:
    async for scalar in result:
        print(f"{scalar}")
```

在上述模式中，即使迭代器被异常抛出中断，`AsyncScalarResult.close()`方法也会无条件地被调用。然而，上下文管理器的使用仍然是可选的，并且该函数可以以`async with fn():`或`await fn()`的方式调用。

新增于版本 2.0.0b3：增加了上下文管理器支持

返回：

将产生一个可等待对象，该对象将生成一个`AsyncScalarResult`对象。

新增于版本 1.4.24。

另见

`AsyncConnection.stream()`

```py
attribute sync_connection: Connection | None
```

引用同步式`Connection`指向此`AsyncConnection`的请求代理。

此实例可用作事件目标。

另见

使用 asyncio 扩展的事件

```py
attribute sync_engine: Engine
```

引用同步式`Engine`指向此`AsyncConnection`的关联，通过其基础的`Connection`。

此实例可用作事件目标。

另见

使用 asyncio 扩展的事件

```py
class sqlalchemy.ext.asyncio.AsyncTransaction
```

一个`Transaction`的 asyncio 代理。

**成员**

close(), commit(), rollback(), start()

**类签名**

类`sqlalchemy.ext.asyncio.AsyncTransaction`（`sqlalchemy.ext.asyncio.base.ProxyComparable`，`sqlalchemy.ext.asyncio.base.StartableContext`）

```py
method async close() → None
```

关闭此`AsyncTransaction`。

如果此事务是 begin/commit 嵌套中的基本事务，则事务将 rollback()。 否则，该方法返回。

此用于取消事务而不影响封闭事务范围的事务。

```py
method async commit() → None
```

提交此`AsyncTransaction`。

```py
method async rollback() → None
```

回滚此`AsyncTransaction`。

```py
method async start(is_ctxmanager: bool = False) → AsyncTransaction
```

在不使用 Python `with:`块的情况下启动此`AsyncTransaction`对象的上下文。

## 结果集 API 文档

`AsyncResult`对象是`Result`对象的异步适配版本。 仅在使用`AsyncConnection.stream()`或`AsyncSession.stream()`方法时返回结果对象，该对象位于活动数据库游标的顶部。

| 对象名称 | 描述 |
| --- | --- |
| AsyncMappingResult | 包装器，用于将`AsyncResult`返回字典值，而不是`Row`值。 |
| AsyncResult | `Result`对象的 asyncio 包装器。 |
| AsyncScalarResult | 包装器，用于将`AsyncResult`返回标量值，而不是`Row`值。 |
| AsyncTupleResult | 一个被类型化为返回普通 Python 元组而不是行的`AsyncResult`。 |

```py
class sqlalchemy.ext.asyncio.AsyncResult
```

围绕`Result`对象的 asyncio 包装器。

`AsyncResult`仅适用于使用服务器端游标的语句执行。它仅从`AsyncConnection.stream()`和`AsyncSession.stream()`方法返回。

注意

与`Result`一样，此对象用于由`AsyncSession.execute()`返回的 ORM 结果，该结果可以单独或在类似元组的行中生成 ORM 映射对象的实例。请注意，这些结果对象不会像旧的`Query`对象一样自动去重实例或行。对于实例或行的 Python 内去重，请使用`AsyncResult.unique()`修饰器方法。

版本 1.4 中的新功能。

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), freeze(), keys(), mappings(), one(), one_or_none(), partitions(), scalar(), scalar_one(), scalar_one_or_none(), scalars(), t, tuples(), unique(), yield_per()

**类签名**

类 `sqlalchemy.ext.asyncio.AsyncResult` (`sqlalchemy.engine._WithKeys`, `sqlalchemy.ext.asyncio.AsyncCommon`)

```py
method async all() → Sequence[Row[_TP]]
```

返回列表中的所有行。

在调用后关闭结果集。后续调用将返回一个空列表。

返回：

一系列 `Row` 对象的列表。

```py
method async close() → None
```

*从* `AsyncCommon.close()` *方法继承*

关闭此结果。

```py
attribute closed
```

*从* `AsyncCommon.closed` *属性继承*

代理底层结果对象的 .closed 属性（如果有），否则引发`AttributeError`。

版本 `2.0.0b3` 中的新内容。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定每行应返回的列。

有关完整行为描述，请参阅同步 SQLAlchemy API 中的 `Result.columns()`。

```py
method async fetchall() → Sequence[Row[_TP]]
```

`AsyncResult.all()` 方法的同义词。

版本 `2.0` 中的新内容。

```py
method async fetchmany(size: int | None = None) → Sequence[Row[_TP]]
```

检索多行。

当所有行都被耗尽时，返回一个空列表。

此方法是为了向后兼容 SQLAlchemy 1.x.x 提供的。

要按组检索行，请使用 `AsyncResult.partitions()` 方法。

返回：

一系列 `Row` 对象的列表。

另请参阅

`AsyncResult.partitions()`

```py
method async fetchone() → Row[_TP] | None
```

检索一行。

当所有行都被耗尽时，返回`None`。

此方法是为了向后兼容 SQLAlchemy 1.x.x 提供的。

要仅检索结果的第一行，请使用 `AsyncResult.first()` 方法。要遍历所有行，请直接迭代 `AsyncResult` 对象。

返回：

如果未应用任何过滤器，则为 `Row` 对象，否则为`None`。

```py
method async first() → Row[_TP] | None
```

检索第一行或如果不存在行则为`None`。

关闭结果集并丢弃剩余行。

注意

默认情况下，此方法返回一个**行**（例如元组）。要返回确切的单个标量值，即第一行的第一列，请使用 `AsyncResult.scalar()` 方法，或者结合 `AsyncResult.scalars()` 和 `AsyncResult.first()`。

此外，与传统 ORM `Query.first()` 方法的行为相反，对产生此`AsyncResult`的 SQL 查询不应用任何限制；对于在向 Python 进程发送行之前在内存中缓冲结果的 DBAPI 驱动程序，所有行将被发送到 Python 进程，除第一行外的所有行将被丢弃。

另请参阅

ORM 查询与核心选择统一

返回���

一个`Row`对象，如果没有剩余行则为 None。

另请参阅

`AsyncResult.scalar()`

`AsyncResult.one()`

```py
method async freeze() → FrozenResult[_TP]
```

返回一个可调用对象，当调用时将产生此`AsyncResult`的副本。

返回的可调用对象是`FrozenResult`的实例。

用于结果集缓存。当结果未被消耗时，必须在结果上调用该方法，并且调用该方法将完全消耗结果。当从缓存中检索到`FrozenResult`时，可以多次调用它，每次都会针对其存储的行集产生一个新的`Result`对象。

另请参阅

重新执行语句 - 在 ORM 中实现结果集缓存的示例用法。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys.keys` *方法的* `sqlalchemy.engine._WithKeys`

返回一个可迭代视图，该视图产生每个`Row`所代表的字符串键。

键可以表示核心语句返回的列标签或 ORM 执行返回的 orm 类的名称。

还可以使用 Python 的 `in` 运算符测试视图中是否包含键，该运算符将同时测试视图中表示的字符串键以及列对象等备用键。

1.4 版本更改：返回键视图对象而不是普通列表。

```py
method mappings() → AsyncMappingResult
```

对返回的行应用映射过滤器，返回一个`AsyncMappingResult`的实例。

当应用此过滤器时，获取行将返回`RowMapping`对象，而不是`Row`对象。

返回:

一个新的指向底层`Result`对象的`AsyncMappingResult`过滤对象。

```py
method async one() → Row[_TP]
```

返回确切的一行或引发异常。

如果结果返回没有行，则引发`NoResultFound`，如果将返回多行，则引发`MultipleResultsFound`。

注意

默认情况下，此方法返回一个**行**，例如元组。要返回确切的一个单一标量值，即第一行的第一列，请使用`AsyncResult.scalar_one()`方法，或结合`AsyncResult.scalars()`和`AsyncResult.one()`。

版本 1.4 中的新功能。

返回:

第一个`Row`。

引发:

`MultipleResultsFound`，`NoResultFound`

另请参阅

`AsyncResult.first()`

`AsyncResult.one_or_none()`

`AsyncResult.scalar_one()`

```py
method async one_or_none() → Row[_TP] | None
```

返回至多一个结果或引发异常。

如果结果没有行，则返回`None`。如果返回多行，则引发`MultipleResultsFound`。

版本 1.4 中的新功能。

返回:

第一个`Row`或如果没有可用行则为`None`。

引发:

`MultipleResultsFound`

另请参阅

`AsyncResult.first()`

`AsyncResult.one()`

```py
method async partitions(size: int | None = None) → AsyncIterator[Sequence[Row[_TP]]]
```

迭代给定大小的行子列表。

返回一个异步迭代器：

```py
async def scroll_results(connection):
    result = await connection.stream(select(users_table))

    async for partition in result.partitions(100):
        print("list of rows: %s" % partition)
```

有关完整的行为描述，请参阅同步 SQLAlchemy API 中的`Result.partitions()`。

```py
method async scalar() → Any
```

提取第一行的第一列，并关闭结果集。

如果没有要提取的行，则返回`None`。

不执行验证以测试是否存在额外的行。

调用此方法后，对象已完全关闭，例如已调用`CursorResult.close()`方法。

返回：

一个 Python 标量值，如果没有剩余行，则为`None`。

```py
method async scalar_one() → Any
```

返回确切的一个标量结果，否则引发异常。

这等同于调用`AsyncResult.scalars()`然后调用`AsyncResult.one()`。

另请参阅

`AsyncResult.one()`

`AsyncResult.scalars()`

```py
method async scalar_one_or_none() → Any | None
```

返回确切的一个标量结果或`None`。

这等同于调用`AsyncResult.scalars()`然后调用`AsyncResult.one_or_none()`。

另请参阅

`AsyncResult.one_or_none()`

`AsyncResult.scalars()`

```py
method scalars(index: _KeyIndexType = 0) → AsyncScalarResult[Any]
```

返回一个过滤对象`AsyncScalarResult`，该对象将返回单个元素而不是`Row`对象。

有关完整的行为描述，请参阅同步 SQLAlchemy API 中的`Result.scalars()`。

参数：

**index** - 整数或行键，指示要从每行提取的列，默认为`0`，表示第一列。

返回：

一个新的过滤对象`AsyncScalarResult`，该对象引用此`AsyncResult`对象。

```py
attribute t
```

对返回的行应用“typed tuple”类型过滤器。

`AsyncResult.t` 属性是调用 `AsyncResult.tuples()` 方法的同义词。

版本 2.0 新内容。

```py
method tuples() → AsyncTupleResult[_TP]
```

对返回的行应用“类型化元组”类型过滤器。

此方法在运行时返回相同的 `AsyncResult` 对象，但标注为返回 `AsyncTupleResult` 对象，这将向 [**PEP 484**](https://peps.python.org/pep-0484/) 类型工具指示，返回的是纯粹的 `Tuple` 实例而不是行。这允许对 `Row` 对象进行元组解包和 `__getitem__` 访问，对于语句本身包含类型信息的情况。

版本 2.0 新内容。

返回：

在类型工具运行时的 `AsyncTupleResult` 类型。

另请参阅

`AsyncResult.t` - 更短的同义词

`Row.t` - `Row` 版本

```py
method unique(strategy: _UniqueFilterType | None = None) → Self
```

对此 `AsyncResult` 返回的对象应用唯一过滤。

有关完整的行为描述，请参阅同步 SQLAlchemy API 中的 `Result.unique()`。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *的方法* `FilterResult`

配置行获取策略，一次获取 `num` 行。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的传递。请参阅该方法的文档以获取用法说明。

版本 1.4.40 新内容： - 添加 `FilterResult.yield_per()` 以便该方法在所有结果集实现上都可用

另请参阅

使用服务器端游标（即流式结果） - 描述 `Result.yield_per()` 的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.ext.asyncio.AsyncScalarResult
```

一个 `AsyncResult` 的包装器，返回标量值而不是 `Row` 值。

`AsyncScalarResult` 对象是通过调用 `AsyncResult.scalars()` 方法获得的。

请参阅同步 SQLAlchemy API 中的 `ScalarResult` 对象，以获取完整的行为描述。

版本 1.4 中的新功能。

**成员**

all(), close(), closed, fetchall(), fetchmany(), first(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类 `sqlalchemy.ext.asyncio.AsyncScalarResult` (`sqlalchemy.ext.asyncio.AsyncCommon`)

```py
method async all() → Sequence[_R]
```

返回列表中的所有标量值。

相当于 `AsyncResult.all()`，只是返回标量值而不是 `Row` 对象。

```py
method async close() → None
```

*继承自* `AsyncCommon` *的* `AsyncCommon.close()` *方法*

关闭此结果。

```py
attribute closed
```

*继承自* `AsyncCommon.closed` *属性的* `AsyncCommon`

代理底层结果对象的 `.closed` 属性，如果有的话，否则引发 `AttributeError`。

版本 2.0.0b3 中的新功能。

```py
method async fetchall() → Sequence[_R]
```

`AsyncScalarResult.all()` 方法的同义词。

```py
method async fetchmany(size: int | None = None) → Sequence[_R]
```

获取多个对象。

相当于 `AsyncResult.fetchmany()`，只是返回标量值而不是 `Row` 对象。

```py
method async first() → _R | None
```

获取第一个对象或 `None`（如果没有对象存在）。

等同于 `AsyncResult.first()`，但返回的是标量值，而不是 `Row` 对象。

```py
method async one() → _R
```

返回恰好一个对象或引发异常。

等同于 `AsyncResult.one()`，但返回的是标量值，而不是 `Row` 对象。

```py
method async one_or_none() → _R | None
```

返回至多一个对象或引发异常。

等同于 `AsyncResult.one_or_none()`，但返回的是标量值，而不是 `Row` 对象。

```py
method async partitions(size: int | None = None) → AsyncIterator[Sequence[_R]]
```

迭代给定大小的子列表元素。

等同于 `AsyncResult.partitions()`，但返回的是标量值，而不是 `Row` 对象。

```py
method unique(strategy: _UniqueFilterType | None = None) → Self
```

对此 `AsyncScalarResult` 返回的对象应用唯一过滤。

请参阅 `AsyncResult.unique()` 以了解使用详情。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`。

配置行提取策略，一次提取 `num` 行。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的一个转发。请参阅该方法的文档以了解使用注意事项。

自版本 1.4.40 新增：- 添加 `FilterResult.yield_per()`，以便在所有结果集实现中都可用

另请参阅

使用服务器端游标（即流式结果） - 描述了 `Result.yield_per()` 的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南 中

```py
class sqlalchemy.ext.asyncio.AsyncMappingResult
```

一个`AsyncResult`的包装器，返回字典值而不是`Row`值。

通过调用`AsyncResult.mappings()`方法获取`AsyncMappingResult`对象。

请参考同步 SQLAlchemy API 中的`MappingResult`对象，以获取完整的行为描述。

版本 1.4 中的新功能。

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), keys(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类`sqlalchemy.ext.asyncio.AsyncMappingResult` (`sqlalchemy.engine._WithKeys`, `sqlalchemy.ext.asyncio.AsyncCommon`)

```py
method async all() → Sequence[RowMapping]
```

返回列表中的所有行。

等同于`AsyncResult.all()`，只是返回`RowMapping`值，而不是`Row`对象。

```py
method async close() → None
```

*继承自* `AsyncCommon.close()` *方法的* `AsyncCommon`

关闭此结果。

```py
attribute closed
```

*继承自* `AsyncCommon.closed` *属性的* `AsyncCommon`

代理底层结果对象的.closed 属性，如果有的话，否则引发`AttributeError`。

版本 2.0.0b3 中的新功能。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定每行应返回的列。

```py
method async fetchall() → Sequence[RowMapping]
```

`AsyncMappingResult.all()` 方法的同义词。

```py
method async fetchmany(size: int | None = None) → Sequence[RowMapping]
```

获取多行。

等同于`AsyncResult.fetchmany()`，只是返回`RowMapping`值，而不是`Row`对象。

```py
method async fetchone() → RowMapping | None
```

获取一个对象。

等同于`AsyncResult.fetchone()`，只是返回`RowMapping`值，而不是`Row`对象。

```py
method async first() → RowMapping | None
```

获取第一个对象或`None`（如果不存在对象）。

等同于`AsyncResult.first()`，只是返回`RowMapping`值，而不是`Row`对象。

```py
method keys() → RMKeyView
```

*继承自*`sqlalchemy.engine._WithKeys`*的*`sqlalchemy.engine._WithKeys.keys`*方法*。

返回一个可迭代视图，该视图产生每个`Row`所代表的字符串键。

键可以表示核心语句返回的列的标签，也可以表示 orm 执行返回的 orm 类的名称。

视图还可以使用 Python 的`in`操作符进行键包含测试，该操作符将同时测试视图中表示的字符串键以及列对象等备用键。

自版本 1.4 起更改：返回的是键视图对象，而不是普通列表。

```py
method async one() → RowMapping
```

返回一个对象或引发异常。

等同于`AsyncResult.one()`，只是返回`RowMapping`值，而不是`Row`对象。

```py
method async one_or_none() → RowMapping | None
```

返回最多一个对象或引发异常。

等同于`AsyncResult.one_or_none()`，只是返回`RowMapping`值，而不是`Row`对象。

```py
method async partitions(size: int | None = None) → AsyncIterator[Sequence[RowMapping]]
```

迭代给定大小的元素子列表。

等同于`AsyncResult.partitions()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method unique(strategy: _UniqueFilterType | None = None) → Self
```

对此`AsyncMappingResult`返回的对象应用唯一过滤。

查看`AsyncResult.unique()`以获取使用详细信息。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

配置行提取策略以一次提取`num`行。

`FilterResult.yield_per()`方法是对`Result.yield_per()`方法的传递。请参阅该方法的文档以获取使用注意事项。

新版本 1.4.40 中：- 添加了`FilterResult.yield_per()`，以便该方法在所有结果集实现上都可用

另见

使用服务器端游标（即流式结果） - 描述了`Result.yield_per()`的核心行为

使用逐个提取大结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.ext.asyncio.AsyncTupleResult
```

一个`AsyncResult`，其类型为返回普通的 Python 元组而不是行。

由于`Row`在所有方面都像元组一样，所以这个类只是一个类型类，正常的`AsyncResult`仍然在运行时使用。

**类签名**

类`sqlalchemy.ext.asyncio.AsyncTupleResult` (`sqlalchemy.ext.asyncio.AsyncCommon`, `sqlalchemy.util.langhelpers.TypingOnly`)

## ORM 会话 API 文档

| 对象名称 | 描述 |
| --- | --- |
| async_object_session(实例) | 返回给定实例所属的`AsyncSession`。 |
| async_scoped_session | 提供`AsyncSession`对象的作用域管理。 |
| async_session(会话) | 返回代理给定`Session`对象的`AsyncSession`，如果有的话。 |
| async_sessionmaker | 可配置的`AsyncSession`工厂。 |
| 异步属性 | 提供所有属性的可等待访问器的混合类。 |
| AsyncSession | `Session`的 Asyncio 版本。 |
| AsyncSessionTransaction | ORM `SessionTransaction`对象的包装器。 |
| close_all_sessions() | 关闭所有`AsyncSession`会话。 |

```py
function sqlalchemy.ext.asyncio.async_object_session(instance: object) → AsyncSession | None
```

返回给定实例所属的`AsyncSession`。

此函数利用同步 API 函数`object_session`来检索引用给定实例的`Session`，然后将其链接到原始的`AsyncSession`。

如果`AsyncSession`已被垃圾回收，返回值为`None`。

此功能也可以从`InstanceState.async_session`访问器中获得。

参数:

**实例** – 一个 ORM 映射实例

返回:

一个`AsyncSession`对象，或`None`。

版本 1.4.18 中的新功能。

```py
function sqlalchemy.ext.asyncio.async_session(session: Session) → AsyncSession | None
```

返回代理给定`Session`对象的`AsyncSession`，如果有的话。

参数:

**session** – 一个`Session` 实例。

返回：

一个`AsyncSession` 实例，或 `None`。

版本 1.4.18 中的新功能。

```py
function async sqlalchemy.ext.asyncio.close_all_sessions() → None
```

关闭所有`AsyncSession` 会话。

版本 2.0.23 中的新功能。

另请参阅

`close_all_sessions()`

```py
class sqlalchemy.ext.asyncio.async_sessionmaker
```

一个可配置的`AsyncSession` 工厂。

`async_sessionmaker` 工厂的工作方式与`sessionmaker` 工厂相同，当调用时生成新的`AsyncSession` 对象，根据此处建立的配置参数创建它们。

例如：

```py
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker

async def run_some_sql(async_session: async_sessionmaker[AsyncSession]) -> None:
    async with async_session() as session:
        session.add(SomeObject(data="object"))
        session.add(SomeOtherObject(name="other object"))
        await session.commit()

async def main() -> None:
    # an AsyncEngine, which the AsyncSession will use for connection
    # resources
    engine = create_async_engine('postgresql+asyncpg://scott:tiger@localhost/')

    # create a reusable factory for new AsyncSession instances
    async_session = async_sessionmaker(engine)

    await run_some_sql(async_session)

    await engine.dispose()
```

`async_sessionmaker` 很有用，因此程序的不同部分可以使用预先建立的固定配置创建新的`AsyncSession` 对象。请注意，当不使用`async_sessionmaker` 时，也可以直接实例化`AsyncSession` 对象。

版本 2.0 中的新功能：`async_sessionmaker` 提供了一个专门用于`AsyncSession` 对象的`sessionmaker` 类，包括 pep-484 类型支持。

另请参阅

概要 - ORM - 展示示例用法

`sessionmaker` - 关于的一般概述

`sessionmaker` 架构

打开和关闭会话 - 介绍如何使用`sessionmaker` 创建会话的文本。

**成员**

__call__(), __init__(), begin(), configure()

**类签名**

类`sqlalchemy.ext.asyncio.async_sessionmaker` (`typing.Generic`)

```py
method __call__(**local_kw: Any) → _AS
```

使用此`async_sessionmaker`中建立的配置生成一个新的`AsyncSession`对象。

在 Python 中，当对象以与函数相同的方式“调用”时，会调用`__call__`方法：

```py
AsyncSession = async_sessionmaker(async_engine, expire_on_commit=False)
session = AsyncSession()  # invokes sessionmaker.__call__()
```

```py
method __init__(bind: Optional[_AsyncSessionBind] = None, *, class_: Type[_AS] = <class 'sqlalchemy.ext.asyncio.session.AsyncSession'>, autoflush: bool = True, expire_on_commit: bool = True, info: Optional[_InfoType] = None, **kw: Any)
```

构建一个新的`async_sessionmaker`。

这里的所有参数（除了`class_`）都直接对应于`Session`直接接受的参数。请查看`AsyncSession.__init__()`文档字符串以获取有关参数的更多详细信息。

```py
method begin() → _AsyncSessionContextManager[_AS]
```

生成一个上下文管理器，既提供一个新的`AsyncSession`，又提供一个提交的事务。

例如：

```py
async def main():
    Session = async_sessionmaker(some_engine)

    async with Session.begin() as session:
        session.add(some_object)

    # commits transaction, closes session
```

```py
method configure(**new_kw: Any) → None
```

重新配置此 async_sessionmaker 的参数。

例如：

```py
AsyncSession = async_sessionmaker(some_engine)

AsyncSession.configure(bind=create_async_engine('sqlite+aiosqlite://'))
```

```py
class sqlalchemy.ext.asyncio.async_scoped_session
```

提供对`AsyncSession`对象的作用域管理。

查看使用 asyncio scoped session 一节获取详细的使用说明。

在版本 1.4.19 中新增。

**成员**

__call__(), __init__(), aclose(), add(), add_all(), autoflush, begin(), begin_nested(), bind, close(), close_all(), commit(), configure(), connection(), delete(), deleted, dirty, execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_one(), identity_key(), identity_map, info, invalidate(), is_active, is_modified(), merge(), new, no_autoflush, object_session(), refresh(), remove(), reset(), rollback(), scalar(), scalars(), session_factory, stream(), stream_scalars()

**类签名**

类`sqlalchemy.ext.asyncio.async_scoped_session` (`typing.Generic`)

```py
method __call__(**kw: Any) → _AS
```

返回当前的`AsyncSession`，如果不存在则使用`scoped_session.session_factory`创建它。

参数：

****kw** – 如果不存在现有的`AsyncSession`，关键字参数将被传递给`scoped_session.session_factory`可调用对象。如果存在`AsyncSession`并且已传递关键字参数，则会引发`InvalidRequestError`。

```py
method __init__(session_factory: async_sessionmaker[_AS], scopefunc: Callable[[], Any])
```

构造一个新的`async_scoped_session`。

参数：

+   `session_factory` – 用于创建新的`AsyncSession`实例的工厂。通常情况下，但不一定，是`async_sessionmaker`的实例。

+   `scopefunc` – 定义当前范围的函数。例如，`asyncio.current_task`可能在这里很有用。

```py
method async aclose() → None
```

是`AsyncSession.close()`的一个同义词。

代理了`async_scoped_session`类，代表`AsyncSession`类。

`AsyncSession.aclose()`的名称是专门为了支持 Python 标准库中的`@contextlib.aclosing`上下文管理器函数。

在版本 2.0.20 中新增。

```py
method add(instance: object, _warn: bool = True) → None
```

将对象放入此`Session`中。

代理了`async_scoped_session`类，代表`AsyncSession`类。

代理了`Session`类，代表`AsyncSession`类。

当传递到`Session.add()`方法时处于瞬态状态的对象将移动到挂起状态，直到下一次刷新，此时它们将转移到持久状态。

当传递到`Session.add()`方法时处于分离状态的对象将直接转移到持久状态。

如果由`Session`使用的事务被回滚，当它们传递给`Session.add()`时处于瞬态状态的对象将被移回瞬态状态，并且将不再存在于此`Session`中。

另请参阅

`Session.add_all()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集合添加到此`Session`中。

代表`async_scoped_session`类，为`AsyncSession`类代理。

代表`AsyncSession`类，为`Session`类代理。

有关一般行为描述，请参阅`Session.add()`的文档。

另请参阅

`Session.add()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
attribute autoflush
```

代表`async_scoped_session`类，为`AsyncSession`类代理`Session.autoflush`属性。

代表`async_scoped_session`类，为`AsyncSession`类代理。

```py
method begin() → AsyncSessionTransaction
```

返回一个`AsyncSessionTransaction`对象。

代理类`AsyncSession`的代表类`async_scoped_session`。

当`AsyncSessionTransaction`对象进入时，底层的`Session`将执行“开始”操作：

```py
async with async_session.begin():
    # .. ORM transaction is begun
```

注意，当会话级事务开始时，通常不会发生数据库 IO，因为数据库事务是按需开始的。但是，开始块是异步的，以适应可能执行 IO 的`SessionEvents.after_transaction_create()`事件挂钩。

关于 ORM 开始的一般描述，请参阅`Session.begin()`。

```py
method begin_nested() → AsyncSessionTransaction
```

返回一个`AsyncSessionTransaction`对象，该对象将开始一个“嵌套”事务，例如 SAVEPOINT。

代理类`AsyncSession`的代表类`async_scoped_session`。

行为与`AsyncSession.begin()`的行为相同。

关于 ORM 开始嵌套的一般描述，请参阅`Session.begin_nested()`。

另见

可序列化隔离/保存点/事务 DDL（asyncio 版本） - 为了使 SAVEPOINT 正常工作，SQLite asyncio 驱动程序需要特殊的解决方法。

```py
attribute bind
```

代理类`AsyncSession.bind`属性的代表类`async_scoped_session`。

```py
method async close() → None
```

关闭此`AsyncSession`使用的事务资源和 ORM 对象。

代理类`AsyncSession`的代表类`async_scoped_session`。

另见

`Session.close()` - “close”的主要文档

关闭 - 关于`AsyncSession.close()`和`AsyncSession.reset()`语义的详细信息。

```py
async classmethod close_all() → None
```

关闭所有`AsyncSession`会话。

代表`AsyncSession`类为`async_scoped_session`类代理。

从版本 2.0 开始弃用：`AsyncSession.close_all()`方法已弃用，并将在将来的版本中移除。请参阅`close_all_sessions()`。

```py
method async commit() → None
```

提交当前进行中的事务。

代表`AsyncSession`类为`async_scoped_session`类代理。

另请参阅

`Session.commit()` - “commit”的主要文档

```py
method configure(**kwargs: Any) → None
```

重新配置此`scoped_session`使用的`sessionmaker`。

请参阅`sessionmaker.configure()`。

```py
method async connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None, **kw: Any) → AsyncConnection
```

返回一个与此`Session`对象的事务状态相对应的`AsyncConnection`对象。

代表`AsyncSession`类为`async_scoped_session`类代理。

此方法还可用于为当前事务使用的数据库连接建立执行选项。

新版本 1.4.24 中添加了传递给底层`Session.connection()`方法的**kw 参数。

另请参阅

`Session.connection()` - “connection”的主要文档

```py
method async delete(instance: object) → None
```

将实例标记为已删除。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

数据库删除操作发生在`flush()`时。

由于此操作可能需要沿未加载的关系级联，因此它是可等待的，以允许执行这些查询。

另请参阅

`Session.delete()` - 删除的主要文档

```py
attribute deleted
```

在此`Session`中标记为‘删除’的所有实例的集合

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

代表`Session`类的代理，代表`AsyncSession`类。

```py
attribute dirty
```

所有持久实例的集合被视为脏数据。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

代表`Session`类的代理，代表`AsyncSession`类。

例如：

```py
some_mapped_object in session.dirty
```

当实例被修改但未被删除时，将其视为脏数据。

请注意，此‘脏数据’计算是‘乐观的’；大多数属性设置或集合修改操作都会将实例标记为‘脏数据’并将其放入此集合中，即使属性的值没有净变化。在刷新时，将每个属性的值与先前保存的值进行比较，如果没有净变化，则不会执行任何 SQL 操作（这是一种更昂贵的操作，因此仅在刷新时执行）。

要检查实例的属性是否具有可执行的净变化，请使用`Session.is_modified()`方法。

```py
method async execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Result[Any]
```

执行语句并返回缓冲的`Result`对象。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

另请参阅

`Session.execute()` - 执行的主要文档

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

使实例上的属性过期。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

代表`Session`类的代理，代表`AsyncSession`类。

将实例的属性标记为过时。下次访问过期属性时，将向`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不管该事务之外的数据库状态如何更改。

要同时使`Session`中的所有对象过期，请使用`Session.expire_all()`。

`Session`对象的默认行为是在调用`Session.rollback()`或`Session.commit()`方法时使所有状态过期，以便为新事务加载新状态。因此，仅在当前事务中发出非 ORM SQL 语句的特定情况下调用`Session.expire()`才有意义。

参数：

+   `instance` – 需要刷新的实例。

+   `attribute_names` – 可选的字符串属性名称列表，指示要过期的属性子集。

另请参阅

刷新/过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使此会话中的所有持久实例过期。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

代表`Session`类的代理，代表`AsyncSession`类。

当下次访问持久实例上的任何属性时，将使用`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与先前在该事务中读取的值相同的值，而不考虑该事务外部数据库状态的变化。

要过期单个对象和这些对象上的单个属性，请使用`Session.expire()`。

当调用`Session.rollback()`或`Session.commit()`方法时，`Session`对象的默认行为是过期所有状态，以便为新事务加载新状态。因此，通常不需要调用`Session.expire_all()`，假设事务是隔离的。

另见

刷新/过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从此`Session`中移除实例。

代表`async_scoped_session`类的`AsyncSession`类的代理。

代表`AsyncSession`类的`Session`类的代理。

这将释放实例的所有内部引用。将根据*expunge*级联规则应用级联。

```py
method expunge_all() → None
```

从此`Session`中移除所有对象实例。

代表`async_scoped_session`类的`AsyncSession`类的代理。

代表`AsyncSession`类的`Session`类的代理。

这等同于在此`Session`中对所有对象调用`expunge(obj)`。

```py
method async flush(objects: Sequence[Any] | None = None) → None
```

将所有对象更改刷新到数据库。

代表`async_scoped_session`类的`AsyncSession`类的代理。

另见

`Session.flush()` - flush 的主要文档

```py
method async get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O | None
```

根据给定的主键标识符返回一个实例，如果找不到则返回`None`。

代理`AsyncSession`类，代表`async_scoped_session`类。

另请参阅

`Session.get()` - get 的主要文档

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, clause: ClauseElement | None = None, bind: _SessionBind | None = None, **kw: Any) → Engine | Connection
```

返回一个同步代理的`Session`绑定到的“bind”。

代理`AsyncSession`类，代表`async_scoped_session`类。

与`Session.get_bind()`方法不同，这个方法目前**不**以任何方式被`AsyncSession`使用，以解析请求的引擎。

注意

这个方法直接代理到`Session.get_bind()`方法，但目前**不**作为一个覆盖目标有用，与`Session.get_bind()`方法相反。下面的示例说明了如何实现与`AsyncSession`和`AsyncEngine`一起工作的自定义`Session.get_bind()`方案。

在自定义垂直分区中介绍的模式说明了如何将自定义绑定查找方案应用于给定一组`Engine`对象的`Session`。要为与`AsyncSession`和`AsyncEngine`对象一起使用的`Session.get_bind()`实现，继续对`Session`进行子类化，并将其应用于`AsyncSession`，使用`AsyncSession.sync_session_class`。内部方法必须继续返回`Engine`实例，可以从`AsyncEngine`使用`AsyncEngine.sync_engine`属性获取：

```py
# using example from "Custom Vertical Partitioning"

import random

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.orm import Session

# construct async engines w/ async drivers
engines = {
    'leader':create_async_engine("sqlite+aiosqlite:///leader.db"),
    'other':create_async_engine("sqlite+aiosqlite:///other.db"),
    'follower1':create_async_engine("sqlite+aiosqlite:///follower1.db"),
    'follower2':create_async_engine("sqlite+aiosqlite:///follower2.db"),
}

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None, **kw):
        # within get_bind(), return sync engines
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines['other'].sync_engine
        elif self._flushing or isinstance(clause, (Update, Delete)):
            return engines['leader'].sync_engine
        else:
            return engines[
                random.choice(['follower1','follower2'])
            ].sync_engine

# apply to AsyncSession using sync_session_class
AsyncSessionMaker = async_sessionmaker(
    sync_session_class=RoutingSession
)
```

`Session.get_bind()`方法在非异步、隐式非阻塞上下文中调用，方式与 ORM 事件钩子和通过`AsyncSession.run_sync()`调用的函数相同，因此希望在`Session.get_bind()`内运行 SQL 命令的例程可以继续使用阻塞式代码，这将在调用数据库驱动程序的 IO 点时转换为隐式异步调用。

```py
method async get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O
```

返回基于给定主键标识符的实例，如果未找到则引发异常。

代理`AsyncSession`类，代表`async_scoped_session`类。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`。

..版本新增: 2.0.22

另请参阅

`Session.get_one()` - get_one 的主要文档

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

返回标识键。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

代表`Session`类的代理，代表`AsyncSession`类。

这是`identity_key()`的别名。

```py
attribute identity_map
```

代理`Session.identity_map`属性，代表`AsyncSession`类。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

```py
attribute info
```

可由用户修改的字典。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

代表`Session`类的代理，代表`AsyncSession`类。

此字典的初始值可以使用`info`参数来填充`Session`构造函数或`sessionmaker`构造函数或工厂方法。此处的字典始终局限于此`Session`，可以独立于所有其他`Session`对象进行修改。

```py
method async invalidate() → None
```

关闭此 Session，使用连接失效。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

完整描述，请参阅`Session.invalidate()`。

```py
attribute is_active
```

如果此`Session`不处于“部分回滚”状态，则为 True。

代表`async_scoped_session`类的`AsyncSession`类的代理。

代表`AsyncSession`类的`Session`类的代理。

从版本 1.4 开始更改：`Session`不再立即开始新事务，因此当首次实例化`Session`时，此属性将为 False。

“partial rollback”状态通常表示`Session`的刷新过程失败，必须发出`Session.rollback()`方法才能完全回滚事务。

如果此`Session`根本不在事务中，则在首次使用时`Session`将自动开始，因此在这种情况下`Session.is_active`将返回 True。

否则，如果此`Session`在事务中，并且该事务尚未在内部回滚，则`Session.is_active`也将返回 True。

另请参见

“由于刷新期间的先前异常，此 Session 的事务已回滚。”（或类似）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

如果给定实例具有本地修改的属性，则返回`True`。

代表`async_scoped_session`类的`AsyncSession`类的代理。

代表`AsyncSession`类的`Session`类的代理。

此方法检索实例上每个被检测属性的历史记录，并比较当前值与先前提交的值（如果有）。

实际上，这是检查给定实例是否在`Session.dirty`集合中的更昂贵且更准确的版本；将对每个属性的净“脏”状态进行完整测试。

例如（E.g.）：

```py
return session.is_modified(someobject)
```

对此方法有一些注意事项：

+   在`Session.dirty`集合中的实例在使用此方法进行测试时可能报告为`False`。这是因为对象可能已通过属性突变接收到变更事件，从而将其放置在`Session.dirty`中，但最终状态与从数据库加载的状态相同，在这里没有净变化。

+   当应用新值时，标量属性可能没有记录先前设置的值，如果属性在应用新值时未加载或已过期，则会出现这种情况 - 在这些情况下，即使与其数据库值相比最终没有净变化，也会假定属性已更改。在大多数情况下，当发生设置事件时，SQLAlchemy 不需要“旧”值，因此如果旧值不存在，则会跳过发出 SQL 调用的费用，这基于对标量值的更新通常是需要的假设，并且在那几种情况下，它不是。与发出防御性 SELECT 相比，平均成本较低。

    当属性容器的`active_history`标志设置为`True`时，才无条件地在设置时获取“旧”值。此标志通常设置为主键属性和不是简单的多对一的标量对象引用。要为任意映射列设置此标志，请使用`column_property()`的`active_history`参数。

参数（Parameters）：

+   `instance` – 要测试挂起更改的映射实例。

+   `include_collections` – 表示是否应该在操作中包含多值集合。将其设置为`False`是一种仅检测基于本地列的属性（即标量列或多对一外键），这些属性在刷新时将导致此实例更新的方法。

```py
method async merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

复制给定实例的状态到此`AsyncSession`中的相应实例。

代理`AsyncSession`类，代表`async_scoped_session`类。

另请参见

`Session.merge()` - merge 的主要文档

```py
attribute new
```

在此`Session`中标记为“新”的所有实例的集合。

代表`async_scoped_session`类的代理，代表`AsyncSession`类。

代表`AsyncSession`类的`Session`类的代理。

```py
attribute no_autoflush
```

返回一个禁用自动刷新的上下文管理器。

代表`async_scoped_session`类的代理，代表`AsyncSession`类。

代表`AsyncSession`类的`Session`类的代理。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在`with:`块内执行的操作不会受到在查询访问时发生的刷新的影响。这在初始化涉及现有数据库查询的一系列对象时非常有用，其中未完成的对象不应立即被刷新。

```py
classmethod object_session(instance: object) → Session | None
```

返回对象所属的`Session`。

代表`async_scoped_session`类的代理，代表`AsyncSession`类。

代表`AsyncSession`类的`Session`类的代理。

这是`object_session()`的别名。

```py
method async refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

使给定实例的属性过期并刷新。

代表`async_scoped_session`类的代理，代表`AsyncSession`类。

将向数据库发出查询，并使用其当前数据库值刷新所有属性。

这是`Session.refresh()`方法的异步版本。有关所有选项的完整描述，请参阅该方法。

另请参阅

`Session.refresh()` - 刷新的主要文档

```py
method async remove() → None
```

丢弃当前的`AsyncSession`（如果存在）。

与 scoped_session 的 remove 方法不同，此方法将使用 await 等待 AsyncSession 的 close 方法。

```py
method async reset() → None
```

关闭事务资源和此 `Session` 使用的 ORM 对象，将会重置会话到其初始状态。

代理`AsyncSession` 类，代表 `async_scoped_session` 类。

新版本 2.0.22 中新增。

另请参阅

`Session.reset()` - 重置的主要文档

关闭 - 关于 `AsyncSession.close()` 和 `AsyncSession.reset()` 语义的详细信息。

```py
method async rollback() → None
```

回滚当前进行中的事务。

代理`AsyncSession` 类，代表 `async_scoped_session` 类。

另请参阅

`Session.rollback()` - 回滚的主要文档

```py
method async scalar(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行语句并返回标量结果。

代理`AsyncSession` 类，代表 `async_scoped_session` 类。

另请参阅

`Session.scalar()` - 标量的主要文档

```py
method async scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行语句并返回标量结果。

代理`AsyncSession` 类，代表 `async_scoped_session` 类。

返回：

一个 `ScalarResult` 对象

新版本 1.4.24 中新增：添加了 `AsyncSession.scalars()`

新版本 1.4.26 中新增：添加了 `async_scoped_session.scalars()`

另请参阅

`Session.scalars()` - 标量的主要文档

`AsyncSession.stream_scalars()` - 流式版本

```py
attribute session_factory: async_sessionmaker[_AS]
```

提供给 __init__ 的 session_factory 存储在此属性中，可在以后访问。当需要新的非作用域 `AsyncSession` 时，这可能会很有用。

```py
method async stream(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncResult[Any]
```

执行语句并返回流式 `AsyncResult` 对象。

代表 `async_scoped_session` 类的 `AsyncSession` 类的代理。

```py
method async stream_scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncScalarResult[Any]
```

执行语句并返回标量结果流。

代表 `async_scoped_session` 类的 `AsyncSession` 类的代理。

返回：

一个 `AsyncScalarResult` 对象

自版本 1.4.24 新增。

另请参阅

`Session.scalars()` - 标量的主要文档

`AsyncSession.scalars()` - 非流式版本

```py
class sqlalchemy.ext.asyncio.AsyncAttrs
```

混合类，为所有属性提供可等待的访问器。

例如：

```py
from __future__ import annotations

from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.ext.asyncio import AsyncAttrs
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(AsyncAttrs, DeclarativeBase):
    pass

class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[str]
    bs: Mapped[List[B]] = relationship()

class B(Base):
    __tablename__ = "b"
    id: Mapped[int] = mapped_column(primary_key=True)
    a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
    data: Mapped[str]
```

在上面的示例中，将 `AsyncAttrs` 混合类应用于声明的 `Base` 类，在所有子类中生效。此混合类为所有类添加一个新属性 `AsyncAttrs.awaitable_attrs` ，它将任何属性的值作为可等待返回。这允许访问可能受惰性加载、延迟加载或未过期加载影响的属性，以便仍然可以发出 IO：

```py
a1 = (await async_session.scalars(select(A).where(A.id == 5))).one()

# use the lazy loader on ``a1.bs`` via the ``.awaitable_attrs``
# interface, so that it may be awaited
for b1 in await a1.awaitable_attrs.bs:
    print(b1)
```

`AsyncAttrs.awaitable_attrs` 对属性执行调用，这相当于使用 `AsyncSession.run_sync()` 方法，例如：

```py
for b1 in await async_session.run_sync(lambda sess: a1.bs):
    print(b1)
```

自版本 2.0.13 新增。

**成员**

awaitable_attrs

另请参阅

在使用 AsyncSession 时防止隐式 IO

```py
attribute awaitable_attrs
```

提供所有属性的命名空间，这些属性在此对象上被封装为可等待。

例如：

```py
a1 = (await async_session.scalars(select(A).where(A.id == 5))).one()

some_attribute = await a1.awaitable_attrs.some_deferred_attribute
some_collection = await a1.awaitable_attrs.some_collection
```

```py
class sqlalchemy.ext.asyncio.AsyncSession
```

`Session` 的异步版本。

`AsyncSession`是传统`Session`实例的代理。

`AsyncSession` **不能安全地用于并发任务**。详见会话线程安全吗？AsyncSession 在并发任务中是否安全共享？。

版本 1.4 中的新功能。

要想使用自定义`Session`实现的`AsyncSession`，请参见`AsyncSession.sync_session_class`参数。

**成员**

同步会话类, __init__(), aclose(), add(), add_all(), autoflush, begin(), begin_nested(), close(), close_all(), commit(), connection(), delete(), deleted, dirty, execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_nested_transaction(), get_one(), get_transaction(), identity_key(), identity_map, in_nested_transaction(), in_transaction(), info, invalidate(), is_active, is_modified(), merge(), new, no_autoflush, object_session(), refresh(), reset(), rollback(), run_sync(), scalar(), scalars(), stream(), stream_scalars(), sync_session

**类签名**

类`sqlalchemy.ext.asyncio.AsyncSession`（`sqlalchemy.ext.asyncio.base.ReversibleProxy`）

```py
attribute sync_session_class: Type[Session] = <class 'sqlalchemy.orm.session.Session'>
```

为特定`AsyncSession`提供基础`Session`实例的类或可调用对象。

在类级别，此属性是`AsyncSession.sync_session_class`参数的默认值。`AsyncSession`的自定义子类可以覆盖此值。

在实例级别，此属性指示当前类或可调用对象，用于为此`AsyncSession`实例提供`Session`实例。

版本 1.4.24 中的新功能。

```py
method __init__(bind: _AsyncSessionBind | None = None, *, binds: Dict[_SessionBindKey, _AsyncSessionBind] | None = None, sync_session_class: Type[Session] | None = None, **kw: Any)
```

构造一个新的`AsyncSession`。

除了`sync_session_class`之外的所有参数都直接传递给`sync_session_class`可调用对象，以实例化一个新的`Session`。有关参数文档，请参阅`Session.__init__()`。

参数：

**sync_session_class** –

一个`Session`子类或其他可调用对象，将用于构造将被代理的`Session`。此参数可用于提供自定义的`Session`子类。默认为`AsyncSession.sync_session_class`类级别属性。

版本 1.4.24 中的新功能。

```py
method async aclose() → None
```

`AsyncSession.close()`的同义词。

`AsyncSession.aclose()`名称专门用于支持 Python 标准库中的`@contextlib.aclosing`上下文管理器函数。

版本 2.0.20 中的新功能。

```py
method add(instance: object, _warn: bool = True) → None
```

将对象放入此`Session`。

代表`Session`类的代理，代表`AsyncSession`类。

当传递给`Session.add()`方法的对象处于瞬态状态时，它们将转移到挂起状态，直到下一次刷新，此时它们将转移到持久状态。

当传递给`Session.add()`方法时处于分离状态的对象将直接转移到持久状态。

如果`Session`使用的事务被回滚，则当传递给`Session.add()`方法时处于瞬态的对象将被移回瞬态状态，并且将不再存在于此`Session`中。

请参见

`Session.add_all()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集合添加到此`Session`中。

代表`AsyncSession`类的`Session`类的代理。

请查看`Session.add()`的文档以获取一般行为描述。

请参见

`Session.add()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
attribute autoflush
```

代表`AsyncSession`类的`Session.autoflush`属性的代理。

```py
method begin() → AsyncSessionTransaction
```

返回一个`AsyncSessionTransaction`对象。

当`AsyncSessionTransaction`对象被输入时，底层的`Session`将执行“开始”操作：

```py
async with async_session.begin():
    # .. ORM transaction is begun
```

请注意，当会话级事务开始时，通常不会发生数据库 IO，因为数据库事务是按需开始的。但是，开始块是异步的，以适应可能执行 IO 的`SessionEvents.after_transaction_create()`事件钩子。

有关 ORM 开始的一般描述，请参阅`Session.begin()`。

```py
method begin_nested() → AsyncSessionTransaction
```

返回一个将开始“嵌套”事务（例如 SAVEPOINT）的`AsyncSessionTransaction`对象。

行为与`AsyncSession.begin()`相同。

有关 ORM 开始嵌套的一般描述，请参阅`Session.begin_nested()`。

另请参阅

可序列化隔离/保存点/事务 DDL（asyncio 版本） - 在 SQLite asyncio��动程序中为 SAVEPOINT 正常工作所需的特殊解决方法。

```py
method async close() → None
```

关闭此`AsyncSession`使用的事务资源和 ORM 对象。

另请参阅

`Session.close()` - “close”主要文档

关闭 - 关于`AsyncSession.close()`和`AsyncSession.reset()`语义的详细信息。

```py
async classmethod close_all() → None
```

关闭所有`AsyncSession`会话。

自版本 2.0 起弃用：`AsyncSession.close_all()`方法已弃用，并将在将来的版本中删除。请参考`close_all_sessions()`。

```py
method async commit() → None
```

提交当前进行中的事务。

另请参阅

`Session.commit()` - “commit”主要文档

```py
method async connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None, **kw: Any) → AsyncConnection
```

返回一个与此`Session`对象的事务状态对应的`AsyncConnection`对象。

此方法也可用于为当前事务使用的数据库连接建立执行选项。

新版本 1.4.24 中添加了**kw 参数，这些参数将传递给底层的`Session.connection()`方法。

另请参阅

`Session.connection()` - “连接”的主要文档

```py
method async delete(instance: object) → None
```

将实例标记为已删除。

数据库删除操作发生在`flush()`时。

由于此操作可能需要沿着未加载的关系进行级联，因此需要等待以便进行这些查询。

另请参阅

`Session.delete()` - 删除的主要文档

```py
attribute deleted
```

此`Session`中所有标记为“已删除”的实例的集合

代表`AsyncSession`类的`Session`类的代理。

```py
attribute dirty
```

被视为所有持久实例的脏集合。

代表`AsyncSession`类的`Session`类的代理。

例如：

```py
some_mapped_object in session.dirty
```

当实例被修改但未删除时，会被视为脏。

请注意，此“脏”计算是“乐观”的；大多数属性设置或集合修改操作都会将实例标记为“脏”，并将其放入此集合中，即使属性的值没有净变化。在 flush 时，将每个属性的值与其先前保存的值进行比较，如果没有净变化，则不会发生 SQL 操作（这是一项更昂贵的操作，因此只在 flush 时执行）。

要检查实例的属性是否具有可行的净变化，请使用`Session.is_modified()`方法。

```py
method async execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Result[Any]
```

执行语句并返回一个缓冲的`Result`对象。

另请参阅

`Session.execute()` - 执行的主要文档

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

使实例上的属性过期。

代表`AsyncSession`类的`Session`类的代理。

将实例的属性标记为过时。下次访问过期属性时，将使用`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不考虑该事务之外数据库状态的更改。

要同时使`Session`中的所有对象过期，请使用`Session.expire_all()`。

当调用`Session.rollback()`或`Session.commit()`方法时，`Session`对象的默认行为是使所有状态过期，以便为新事务加载新状态。因此，仅在当前事务中发出非 ORM SQL 语句的特定情况下调用`Session.expire()`才有意义。

参数：

+   `instance` – 需要刷新的实例。

+   `attribute_names` – 可选的字符串属性名称列表，指示要过期的属性子集。

另请参阅

刷新/过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使此会话中的所有持久实例过期。

代表`AsyncSession`类的`Session`类的代理。

当持久化实例上的任何属性下次被访问时，将使用`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不考虑该事务之外数据库状态的更改。

要使单个对象和这些对象上的单个属性过期，请使用`Session.expire()`。

当调用`Session.rollback()`或`Session.commit()`方法时，`Session`对象的默认行为是使所有状态过期，以便为新事务加载新状态。因此，通常不需要调用`Session.expire_all()`，假设事务是隔离的。

另请参阅

刷新/过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从此`Session`中删除实例。

代表`AsyncSession`类的`Session`类的代理。

这将释放对实例的所有内部引用。级联将根据*expunge*级联规则应用。

```py
method expunge_all() → None
```

从此`Session`中删除所有对象实例。

代表`AsyncSession`类的`Session`类的代理。

这等效于在此`Session`中的所有对象上调用`expunge(obj)`。

```py
method async flush(objects: Sequence[Any] | None = None) → None
```

将所有对象更改刷新到数据库。

另请参阅

`Session.flush()` - 刷新的主要文档

```py
method async get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O | None
```

根据给定的主键标识符返回一个实例，如果找不到则返回`None`。

另请参阅

`Session.get()` - get 的主要文档

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, clause: ClauseElement | None = None, bind: _SessionBind | None = None, **kw: Any) → Engine | Connection
```

返回一个“绑定”，用于同步代理的`Session`绑定。

与`Session.get_bind()`方法不同，此方法目前**未**以任何方式被此`AsyncSession`使用，以便为请求解析引擎。

注意

此方法直接代理到`Session.get_bind()`方法，但目前**不**像`Session.get_bind()`方法那样作为一个重写目标有用。下面的示例说明了如何实现与`AsyncSession`和`AsyncEngine`配合使用的自定义`Session.get_bind()`方案。

在 自定义垂直分区 中引入的模式说明了如何对给定一组 `Engine` 对象的 `Session` 应用自定义绑定查找方案。要应用相应的 `Session.get_bind()` 实现以与 `AsyncSession` 和 `AsyncEngine` 对象一起使用，继续对 `Session` 进行子类化，并使用 `AsyncSession.sync_session_class` 将其应用到 `AsyncSession`。内部方法必须继续返回 `Engine` 实例，可以使用 `AsyncEngine` 的 `AsyncEngine.sync_engine` 属性从中获取：

```py
# using example from "Custom Vertical Partitioning"

import random

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.orm import Session

# construct async engines w/ async drivers
engines = {
    'leader':create_async_engine("sqlite+aiosqlite:///leader.db"),
    'other':create_async_engine("sqlite+aiosqlite:///other.db"),
    'follower1':create_async_engine("sqlite+aiosqlite:///follower1.db"),
    'follower2':create_async_engine("sqlite+aiosqlite:///follower2.db"),
}

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None, **kw):
        # within get_bind(), return sync engines
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines['other'].sync_engine
        elif self._flushing or isinstance(clause, (Update, Delete)):
            return engines['leader'].sync_engine
        else:
            return engines[
                random.choice(['follower1','follower2'])
            ].sync_engine

# apply to AsyncSession using sync_session_class
AsyncSessionMaker = async_sessionmaker(
    sync_session_class=RoutingSession
)
```

在一个非异步、隐式非阻塞上下文中调用 `Session.get_bind()` 方法，方式与通过 `AsyncSession.run_sync()` 调用的 ORM 事件钩子和函数相同，因此希望在 `Session.get_bind()` 中运行 SQL 命令的例程可以继续使用阻塞式代码，这将在调用数据库驱动程序的 IO 时被转换为隐式异步调用。

```py
method get_nested_transaction() → AsyncSessionTransaction | None
```

返回当前正在进行的嵌套事务，如果有的话。

返回：

一个 `AsyncSessionTransaction` 对象，或 `None`。

新增于版本 1.4.18。

```py
method async get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O
```

根据给定的主键标识符返回一个实例，如果找不到则引发异常。

如果查询未选择任何行，则引发 `sqlalchemy.orm.exc.NoResultFound` 异常。

..versionadded: 2.0.22

另请参见

`Session.get_one()` - get_one 的主要文档

```py
method get_transaction() → AsyncSessionTransaction | None
```

返回当前正在进行的根事务，如果有的话。

返回：

一个 `AsyncSessionTransaction` 对象，或 `None`。

1.4.18 版本中的新功能。

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

返回标识键。

在`Session`类上代理`AsyncSession`类。

这是`identity_key()`的别名。

```py
attribute identity_map
```

代理`AsyncSession`类的`Session.identity_map`属性。

```py
method in_nested_transaction() → bool
```

如果此`Session`已开始嵌套事务（例如，SAVEPOINT），则返回 True。

在`Session`类上代理`AsyncSession`类。

1.4 版本中的新功能。

```py
method in_transaction() → bool
```

如果此`Session`已开始事务，则返回 True。

在`Session`类上代理`AsyncSession`类。

1.4 版本中的新功能。

另请参阅

`Session.is_active`的代理。

```py
attribute info
```

可以由用户修改的字典。

在`Session`类上代理`AsyncSession`类。

此字典的初始值可以使用`info`参数来填充`Session`构造函数或`sessionmaker`构造函数或工厂方法。此处的字典始终局限于此`Session`，并且可以独立于所有其他`Session`对象进行修改。

```py
method async invalidate() → None
```

使用连接失效关闭此会话。

有关完整描述，请参阅`Session.invalidate()`。

```py
attribute is_active
```

如果此`Session`不处于“部分回滚”状态，则为 True。

在`Session`类上代理`AsyncSession`类。

在 1.4 版本中更改：`Session`不再立即开始新的事务，因此在首次实例化`Session`时，此属性将为 False。

“部分回滚”状态通常表示`Session`的 flush 过程失败，并且必须发出`Session.rollback()`方法以完全回滚事务。

如果此`Session`根本不处于事务中，则在首次使用时将自动开始，因此在这种情况下`Session.is_active`将返回 True。

否则，如果此`Session`位于事务中，并且该事务尚未在内部回滚，则`Session.is_active`也将返回 True。

另请参阅

“由于在 flush 期间发生先前异常，此会话的事务已回滚。”（或类似）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

如果给定实例具有本地修改的属性，则返回`True`。

代表`Session`类的代理，代表`AsyncSession`类。

此方法检索实例上每个被检测属性的历史记录，并将当前值与其先前提交的值进行比较（如果有）。

这实际上是检查给定实例是否在`Session.dirty`集合中更昂贵和准确的版本；执行每个属性的净“脏”状态的全面测试。

例如：

```py
return session.is_modified(someobject)
```

此方法有一些注意事项：

+   在`Session.dirty`集合中的实例在使用此方法进行测试时可能报告`False`。这是因为对象可能已通过属性变化接收到更改事件，从而将其放置在`Session.dirty`中，但最终状态与从数据库加载的状态相同，在此处没有净变化。

+   当新值被应用时，标量属性可能没有记录先前设置的值，如果在接收到新值时未加载或过期，则在这些情况下，假设属性具有更改，即使最终对其数据库值没有净更改也是如此。在大多数情况下，当发生设置事件时，SQLAlchemy 不需要“旧”值，因此如果旧值不存在，则跳过 SQL 调用的开销，这基于假设标量值的 UPDATE 通常是必需的，并且在极少数情况下，当它不是时，平均成本比发出防御性 SELECT 更低。

    仅当属性容器的`active_history`标志设置为`True`时，才无条件地获取“旧”值。此标志通常设置为主键属性和不是简单多对一的标量对象引用。要为任意映射列设置此标志，请使用`column_property()`的`active_history`参数。

参数：

+   `instance` – 要测试是否存在待处理更改的映射实例。

+   `include_collections` – 指示是否应该在操作中包含多值集合。将其设置为`False`是一种检测仅基于本地列的属性（即标量列或多对一外键），这些属性会导致此实例在 flush 时进行更新的方法。

```py
method async merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

将给定实例的状态复制到此`AsyncSession`中的相应实例。

另请参见

`Session.merge()` - merge 的主要文档

```py
attribute new
```

在此`Session`中标记为“new”的所有实例的集合。

代理`Session`类，代表`AsyncSession`类。

```py
attribute no_autoflush
```

返回一个上下文管理器，用于禁用自动 flush。

代理`Session`类，代表`AsyncSession`类。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在`with:`块中进行的操作将不受查询访问时发生的 flush 的影响。这在初始化一系列涉及现有数据库查询的对象时很有用，其中未完成的对象不应立即被 flush。

```py
classmethod object_session(instance: object) → Session | None
```

返回对象所属的`Session`。

代理`Session`类，代表`AsyncSession`类。

这是 `object_session()` 的别名。

```py
method async refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

使给定实例的属性过期并刷新。

将向数据库发出查询，并刷新所有属性为其当前数据库值。

这是 `Session.refresh()` 方法的异步版本。有关所有选项的完整描述，请参阅该方法。

参见

`Session.refresh()` - refresh 的主要文档

```py
method async reset() → None
```

关闭事务资源和此 `Session` 使用的 ORM 对象，将会重置会话到初始状态。

新版本 2.0.22 中新增。

参见

`Session.reset()` - “reset” 的主要文档

关闭 - 关于 `AsyncSession.close()` 和 `AsyncSession.reset()` 语义的详细信息。

```py
method async rollback() → None
```

回滚当前进行中的事务。

参见

`Session.rollback()` - “rollback” 的主要文档

```py
method async run_sync(fn: ~typing.Callable[[~typing.Concatenate[~sqlalchemy.orm.session.Session, ~_P]], ~sqlalchemy.ext.asyncio.session._T], *arg: ~typing.~_P, **kw: ~typing.~_P) → _T
```

调用给定的同步（即非异步）可调用对象，并将同步式 `Session` 作为第一个参数传递。

该方法允许在 asyncio 应用程序的上下文中运行传统的同步 SQLAlchemy 函数。

例如：

```py
def some_business_method(session: Session, param: str) -> str:
  '''A synchronous function that does not require awaiting

 :param session: a SQLAlchemy Session, used synchronously

 :return: an optional return value is supported

 '''
    session.add(MyObject(param=param))
    session.flush()
    return "success"

async def do_something_async(async_engine: AsyncEngine) -> None:
  '''an async function that uses awaiting'''

    with AsyncSession(async_engine) as async_session:
        # run some_business_method() with a sync-style
        # Session, proxied into an awaitable
        return_code = await async_session.run_sync(some_business_method, param="param1")
        print(return_code)
```

该方法通过在特别检测的绿色线程中运行给定的可调用对象，从而一直维持 asyncio 事件循环与数据库连接。

提示

在 asyncio 事件循环中内联调用提供的可调用对象，并将在传统 IO 调用上阻塞。此可调用对象内的 IO 应仅调用 SQLAlchemy 的 asyncio 数据库 API，这些 API 将正确适应绿色线程上下文。

参见

`AsyncAttrs` - 为 ORM 映射类提供类似功能的混入，以便每个属性更简洁地提供类似功能

`AsyncConnection.run_sync()`

在 asyncio 下运行同步方法和函数

```py
method async scalar(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行语句并返回标量结果。

参见

`Session.scalar()` - scalar 的主要文档

```py
method async scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行语句并返回标量结果。

返回：

一个`ScalarResult`对象

新版本 1.4.24 中新增了`AsyncSession.scalars()`

新版本 1.4.26 中新增了`async_scoped_session.scalars()`

另请参阅

`Session.scalars()` - 标量的主要文档

`AsyncSession.stream_scalars()` - 流式版本

```py
method async stream(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncResult[Any]
```

执行语句并返回流式`AsyncResult`对象。

```py
method async stream_scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncScalarResult[Any]
```

执行语句并返回标量结果流。

返回：

一个`AsyncScalarResult`对象

新版本 1.4.24 中新增。

另请参阅

`Session.scalars()` - 标量的主要文档

`AsyncSession.scalars()` - 非流式版本

```py
attribute sync_session: Session
```

引用底层的`Session`，此`AsyncSession`代理请求。

此实例可用作事件目标。

另请参阅

使用 asyncio 扩展的事件

```py
class sqlalchemy.ext.asyncio.AsyncSessionTransaction
```

用于 ORM `SessionTransaction` 对象的包装器。

此对象提供了一个用于返回`AsyncSession.begin()`的事务持有对象。

该对象支持对`AsyncSessionTransaction.commit()`和`AsyncSessionTransaction.rollback()`的显式调用，以及作为异步上下文管理器的使用。

新版本 1.4 中新增。

**成员**

commit(), rollback()

**类签名**

类`sqlalchemy.ext.asyncio.AsyncSessionTransaction` (`sqlalchemy.ext.asyncio.base.ReversibleProxy`, `sqlalchemy.ext.asyncio.base.StartableContext`)

```py
method async commit() → None
```

提交这个`AsyncTransaction`。

```py
method async rollback() → None
```

回滚这个`AsyncTransaction`。

## Asyncio 平台安装说明（包括 Apple M1）

asyncio 扩展仅支持 Python 3。它还依赖于[greenlet](https://pypi.org/project/greenlet/)库。这个依赖默认安装在常见的机器平���上，包括：

```py
x86_64 aarch64 ppc64le amd64 win32
```

对于上述平台，已知`greenlet`提供预构建的 wheel 文件。对于其他平台，**默认情况下不安装 greenlet**；可以在[Greenlet - Download Files](https://pypi.org/project/greenlet/#files)查看当前的 greenlet 文件列表。请注意**有许多架构被省略，包括 Apple M1**。

要安装 SQLAlchemy 并确保 `greenlet` 依赖存在，无论使用何种平台，可以按照以下方式安装 `[asyncio]` [setuptools extra](https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras)，这也会指示 `pip` 安装 `greenlet`:

```py
pip install sqlalchemy[asyncio]
```

请注意，在没有预构建 wheel 文件的平台上安装 `greenlet` 意味着 `greenlet` 将从源代码构建，这要求 Python 的开发库也存在。

## 概要 - 核心

对于核心用途，`create_async_engine()` 函数创建一个`AsyncEngine`实例，然后提供传统`Engine` API 的异步版本。`AsyncEngine`通过其`AsyncEngine.connect()`和`AsyncEngine.begin()`方法提供一个`AsyncConnection`，两者都提供异步上下文管理器。`AsyncConnection`然后可以使用`AsyncConnection.execute()`方法来执行语句，以提供一个缓冲的`Result`，或者使用`AsyncConnection.stream()`方法来提供一个流式的服务器端`AsyncResult`:

```py
>>> import asyncio

>>> from sqlalchemy import Column
>>> from sqlalchemy import MetaData
>>> from sqlalchemy import select
>>> from sqlalchemy import String
>>> from sqlalchemy import Table
>>> from sqlalchemy.ext.asyncio import create_async_engine

>>> meta = MetaData()
>>> t1 = Table("t1", meta, Column("name", String(50), primary_key=True))

>>> async def async_main() -> None:
...     engine = create_async_engine("sqlite+aiosqlite://", echo=True)
...
...     async with engine.begin() as conn:
...         await conn.run_sync(meta.drop_all)
...         await conn.run_sync(meta.create_all)
...
...         await conn.execute(
...             t1.insert(), [{"name": "some name 1"}, {"name": "some name 2"}]
...         )
...
...     async with engine.connect() as conn:
...         # select a Result, which will be delivered with buffered
...         # results
...         result = await conn.execute(select(t1).where(t1.c.name == "some name 1"))
...
...         print(result.fetchall())
...
...     # for AsyncEngine created in function scope, close and
...     # clean-up pooled connections
...     await engine.dispose()

>>> asyncio.run(async_main())
BEGIN  (implicit)
...
CREATE  TABLE  t1  (
  name  VARCHAR(50)  NOT  NULL,
  PRIMARY  KEY  (name)
)
...
INSERT  INTO  t1  (name)  VALUES  (?)
[...]  [('some name 1',),  ('some name 2',)]
COMMIT
BEGIN  (implicit)
SELECT  t1.name
FROM  t1
WHERE  t1.name  =  ?
[...]  ('some name 1',)
[('some name 1',)]
ROLLBACK 
```

上文提到的`AsyncConnection.run_sync()`方法可用于调用特殊的 DDL 函数，例如`MetaData.create_all()`，这些函数不包括可等待的挂钩。

提示

在使用`AsyncEngine`对象的范围超出上下文并被垃圾收集时，建议使用`await`调用`AsyncEngine.dispose()`方法，如上例中的`async_main`函数所示。这确保了连接池保持的任何连接都将在可等待的上下文中正确处理。与使用阻塞 IO 不同，SQLAlchemy 无法在像`__del__`或 weakref finalizer 之类的方法中正确处理这些连接，因为没有机会调用`await`。当引擎超出范围时未显式处理时，可能会导致发出到标准输出的警告，形式类似于垃圾收集中的`RuntimeError: Event loop is closed`。

`AsyncConnection`还通过`AsyncConnection.stream()`方法提供了一个“流式”API，返回一个`AsyncResult`对象。此结果对象使用服务器端游标并提供异步/等待 API，例如异步迭代器：

```py
async with engine.connect() as conn:
    async_result = await conn.stream(select(t1))

    async for row in async_result:
        print("row: %s" % (row,))
```

## 概述 - ORM

使用 2.0 风格查询，`AsyncSession`类提供完整的 ORM 功能。

在默认使用模式下，必须特别小心避免涉及 ORM 关系和列属性的延迟加载或其他已过期属性访问；下一节在使用 AsyncSession 时防止隐式 IO 详细说明了这一点。

警告

单个`AsyncSession`实例**不适合在多个并发任务中使用**。有关背景信息，请参阅使用 AsyncSession 处理并发任务和会话线程安全吗？AsyncSession 在并发任务中是否安全共享？部分。

下面的示例说明了包括映射器和会话配置在内的完整示例：

```py
>>> from __future__ import annotations

>>> import asyncio
>>> import datetime
>>> from typing import List

>>> from sqlalchemy import ForeignKey
>>> from sqlalchemy import func
>>> from sqlalchemy import select
>>> from sqlalchemy.ext.asyncio import AsyncAttrs
>>> from sqlalchemy.ext.asyncio import async_sessionmaker
>>> from sqlalchemy.ext.asyncio import AsyncSession
>>> from sqlalchemy.ext.asyncio import create_async_engine
>>> from sqlalchemy.orm import DeclarativeBase
>>> from sqlalchemy.orm import Mapped
>>> from sqlalchemy.orm import mapped_column
>>> from sqlalchemy.orm import relationship
>>> from sqlalchemy.orm import selectinload

>>> class Base(AsyncAttrs, DeclarativeBase):
...     pass

>>> class B(Base):
...     __tablename__ = "b"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
...     data: Mapped[str]

>>> class A(Base):
...     __tablename__ = "a"
...
...     id: Mapped[int] = mapped_column(primary_key=True)
...     data: Mapped[str]
...     create_date: Mapped[datetime.datetime] = mapped_column(server_default=func.now())
...     bs: Mapped[List[B]] = relationship()

>>> async def insert_objects(async_session: async_sessionmaker[AsyncSession]) -> None:
...     async with async_session() as session:
...         async with session.begin():
...             session.add_all(
...                 [
...                     A(bs=[B(data="b1"), B(data="b2")], data="a1"),
...                     A(bs=[], data="a2"),
...                     A(bs=[B(data="b3"), B(data="b4")], data="a3"),
...                 ]
...             )

>>> async def select_and_update_objects(
...     async_session: async_sessionmaker[AsyncSession],
... ) -> None:
...     async with async_session() as session:
...         stmt = select(A).order_by(A.id).options(selectinload(A.bs))
...
...         result = await session.execute(stmt)
...
...         for a in result.scalars():
...             print(a, a.data)
...             print(f"created at: {a.create_date}")
...             for b in a.bs:
...                 print(b, b.data)
...
...         result = await session.execute(select(A).order_by(A.id).limit(1))
...
...         a1 = result.scalars().one()
...
...         a1.data = "new data"
...
...         await session.commit()
...
...         # access attribute subsequent to commit; this is what
...         # expire_on_commit=False allows
...         print(a1.data)
...
...         # alternatively, AsyncAttrs may be used to access any attribute
...         # as an awaitable (new in 2.0.13)
...         for b1 in await a1.awaitable_attrs.bs:
...             print(b1, b1.data)

>>> async def async_main() -> None:
...     engine = create_async_engine("sqlite+aiosqlite://", echo=True)
...
...     # async_sessionmaker: a factory for new AsyncSession objects.
...     # expire_on_commit - don't expire objects after transaction commit
...     async_session = async_sessionmaker(engine, expire_on_commit=False)
...
...     async with engine.begin() as conn:
...         await conn.run_sync(Base.metadata.create_all)
...
...     await insert_objects(async_session)
...     await select_and_update_objects(async_session)
...
...     # for AsyncEngine created in function scope, close and
...     # clean-up pooled connections
...     await engine.dispose()

>>> asyncio.run(async_main())
BEGIN  (implicit)
...
CREATE  TABLE  a  (
  id  INTEGER  NOT  NULL,
  data  VARCHAR  NOT  NULL,
  create_date  DATETIME  DEFAULT  (CURRENT_TIMESTAMP)  NOT  NULL,
  PRIMARY  KEY  (id)
)
...
CREATE  TABLE  b  (
  id  INTEGER  NOT  NULL,
  a_id  INTEGER  NOT  NULL,
  data  VARCHAR  NOT  NULL,
  PRIMARY  KEY  (id),
  FOREIGN  KEY(a_id)  REFERENCES  a  (id)
)
...
COMMIT
BEGIN  (implicit)
INSERT  INTO  a  (data)  VALUES  (?)  RETURNING  id,  create_date
[...]  ('a1',)
...
INSERT  INTO  b  (a_id,  data)  VALUES  (?,  ?)  RETURNING  id
[...]  (1,  'b2')
...
COMMIT
BEGIN  (implicit)
SELECT  a.id,  a.data,  a.create_date
FROM  a  ORDER  BY  a.id
[...]  ()
SELECT  b.a_id  AS  b_a_id,  b.id  AS  b_id,  b.data  AS  b_data
FROM  b
WHERE  b.a_id  IN  (?,  ?,  ?)
[...]  (1,  2,  3)
<A  object  at  ...>  a1
created  at:  ...
<B  object  at  ...>  b1
<B  object  at  ...>  b2
<A  object  at  ...>  a2
created  at:  ...
<A  object  at  ...>  a3
created  at:  ...
<B  object  at  ...>  b3
<B  object  at  ...>  b4
SELECT  a.id,  a.data,  a.create_date
FROM  a  ORDER  BY  a.id
LIMIT  ?  OFFSET  ?
[...]  (1,  0)
UPDATE  a  SET  data=?  WHERE  a.id  =  ?
[...]  ('new data',  1)
COMMIT
new  data
<B  object  at  ...>  b1
<B  object  at  ...>  b2 
```

在上面的示例中，使用可选的 `async_sessionmaker` 助手来实例化 `AsyncSession`，该助手提供了一个新 `AsyncSession` 对象的工厂，并带有一组固定的参数，其中包括将其与特定数据库 URL 关联起来的 `AsyncEngine`。然后将其传递给其他方法，在这些方法中，它可能会在 Python 异步上下文管理器中使用（即 `async with:` 语句），以便在块结束时自动关闭；这相当于调用 `AsyncSession.close()` 方法。

### 使用 AsyncSession 处理并发任务

`AsyncSession` 对象是一个**可变的、有状态的对象**，表示正在进行的**单个、有状态的数据库事务**。使用 asyncio 的并发任务，例如使用 `asyncio.gather()` 等 API，应该为每个单独的任务使用**单独的** `AsyncSession`。

有关在并发工作负载中如何使用 `Session` 和 `AsyncSession` 的一般描述，请参阅 Session 线程安全吗？AsyncSession 在并发任务中共享是否安全？ 部分。### 使用 AsyncSession 时防止隐式 IO

使用传统的 asyncio，应用程序需要避免出现任何可能发生 IO-on-attribute 访问的点。可以用以下技术来帮助解决这个问题，其中许多技术在前面的示例中有所体现。

+   惰性加载关系、延迟列或表达式的属性，或者在过期情况下访问的属性，可以利用 `AsyncAttrs` 混合类。当将此混合类添加到特定类或更一般地添加到声明性的 `Base` 超类时，它提供了一个访问器 `AsyncAttrs.awaitable_attrs`，它将任何属性作为可等待对象传递：

    ```py
    from __future__ import annotations

    from typing import List

    from sqlalchemy.ext.asyncio import AsyncAttrs
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship

    class Base(AsyncAttrs, DeclarativeBase):
        pass

    class A(Base):
        __tablename__ = "a"

        # ... rest of mapping ...

        bs: Mapped[List[B]] = relationship()

    class B(Base):
        __tablename__ = "b"

        # ... rest of mapping ...
    ```

    在不使用急切加载时，访问新加载的`A`实例上的`A.bs`集合通常会使用延迟加载，为了成功，通常会向数据库发出 IO，这在 asyncio 下会失败，因为不允许隐式 IO。在没有任何先前加载操作的情况下，在 asyncio 下直接访问此属性，该属性可以作为可等待对象访问，通过指定`AsyncAttrs.awaitable_attrs`前缀：

    ```py
    a1 = (await session.scalars(select(A))).one()
    for b1 in await a1.awaitable_attrs.bs:
        print(b1)
    ```

    `AsyncAttrs` mixin 提供了一个简洁的外观，覆盖了内部方法，该方法也被`AsyncSession.run_sync()`方法使用。

    版本 2.0.13 中的新功能。

    另请参见

    `AsyncAttrs`

+   集合可以被替换为**只写集合**，这些集合永远不会隐式发出 IO，通过在 SQLAlchemy 2.0 中使用 Write Only Relationships 功能。使用此功能，集合永远不会被读取，只能通过显式 SQL 调用进行查询。查看 Asyncio Integration 部分中的示例`async_orm_writeonly.py`，展示了如何在 asyncio 中使用只写集合。

    当使用只写集合时，程序的行为在处理集合方面是简单且易于预测的。然而，缺点是没有任何内置系统可以一次性加载这些集合中的许多，而需要手动执行。因此，下面的许多要点涉及在使用传统的延迟加载关系与 asyncio 时需要更加小心的具体技术。

+   如果不使用`AsyncAttrs`，关系可以声明为`lazy="raise"`，这样默认情况下它们不会尝试发出 SQL。为了加载集合，将使用 eager loading。

+   最有用的急切加载策略是`selectinload()`急切加载器，在前面的示例中被用来在`await session.execute()`调用的范围内急切加载`A.bs`集合：

    ```py
    stmt = select(A).options(selectinload(A.bs))
    ```

+   当构建新对象时，**集合总是被分配一个默认的空集合**，比如上面的示例中的列表：

    ```py
    A(bs=[], data="a2")
    ```

    这使得在刷新`A`对象时，上述`A`对象上的`.bs`集合可以存在且可读；否则，当刷新`A`时，`.bs`将会被卸载，并在访问时引发错误。

+   `AsyncSession`配置为使用`Session.expire_on_commit`设置为 False，以便我们可以在调用`AsyncSession.commit()`后访问对象上的属性，就像在最后一行访问属性的地方一样：

    ```py
    # create AsyncSession with expire_on_commit=False
    async_session = AsyncSession(engine, expire_on_commit=False)

    # sessionmaker version
    async_session = async_sessionmaker(engine, expire_on_commit=False)

    async with async_session() as session:
        result = await session.execute(select(A).order_by(A.id))

        a1 = result.scalars().first()

        # commit would normally expire all attributes
        await session.commit()

        # access attribute subsequent to commit; this is what
        # expire_on_commit=False allows
        print(a1.data)
    ```

其他指南包括：

+   应该避免使用类似`AsyncSession.expire()`的方法，而应该使用`AsyncSession.refresh()`；**如果**绝对需要过期。当使用 asyncio 时，通常不需要过期，因为`Session.expire_on_commit`应该通常设置为`False`。

+   可以在 asyncio 下显式加载延迟加载的关系，使用`AsyncSession.refresh()`，**如果**所需的属性名称明确传递给`Session.refresh.attribute_names`，例如：

    ```py
    # assume a_obj is an A that has lazy loaded A.bs collection
    a_obj = await async_session.get(A, [1])

    # force the collection to load by naming it in attribute_names
    await async_session.refresh(a_obj, ["bs"])

    # collection is present
    print(f"bs collection: {a_obj.bs}")
    ```

    当然，最好在一开始就使用急加载，以便在不需要延迟加载的情况下已经设置好集合。

    版本 2.0.4 中的新功能：增加了对`AsyncSession.refresh()`和底层`Session.refresh()`方法的支持，以强制加载延迟加载的关系，如果它们在`Session.refresh.attribute_names`参数中明确命名。在先前的版本中，即使在参数中命名，关系也会被静默跳过。

+   避免使用 Cascades 中记录的`all`级联选项，而是明确列出所需的级联特性。`all`级联选项暗示了 refresh-expire 设置，这意味着`AsyncSession.refresh()`方法将使相关对象的属性过期，但不一定刷新这些相关对象，假设未在`relationship()`中配置急加载，将它们保持在过期状态。

+   如果在`deferred()`列中使用了适当的加载选项，应该使用适当的加载选项，此外还应该注意`relationship()`结构。请参见限制使用延迟列加载的列以获取关于延迟列加载的背景信息。

+   在动态关系加载器一节描述的“动态”关系加载器策略在默认情况下与 asyncio 方法不兼容。它只能在`AsyncSession.run_sync()`方法中直接调用，或者通过使用其`.statement`属性获取普通 select：

    ```py
    user = await session.get(User, 42)
    addresses = (await session.scalars(user.addresses.statement)).all()
    stmt = user.addresses.statement.where(Address.email_address.startswith("patrick"))
    addresses_filter = (await session.scalars(stmt)).all()
    ```

    只写技术，在 SQLAlchemy 的 2.0 版本中引入，完全兼容 asyncio，并应该优先使用。

    另请参阅

    “动态”关系加载器被“只写”取代 - 迁移到 2.0 风格的注意事项

+   如果在与不支持 RETURNING 的数据库（例如 MySQL 8）一起使用 asyncio，那么在刷新的新对象上将不会有服务器默认值，例如生成的时间戳，除非使用了`Mapper.eager_defaults`选项。在 SQLAlchemy 2.0 中，这种行为自动应用于像 PostgreSQL、SQLite 和 MariaDB 这样使用 RETURNING 来在插入行时获取新值的后端。### 在 asyncio 下运行同步方法和函数

深度炼金术

这种方法本质上是公开了 SQLAlchemy 能够首先提供 asyncio 接口的机制。虽然在技术上没有任何问题，但总的来说，这种方法可能被认为是“有争议的”，因为它违背了 asyncio 编程模型的一些核心理念，即任何可能导致 IO 被调用的编程语句**必须**有一个`await`调用，以防程序不明确地指明每一行可能发生 IO 的位置。这种方法并没有改变这个一般的想法，只是允许一系列同步 IO 指令在函数调用的范围内豁免这个规则，基本上被捆绑成一个可等待的。

作为在 asyncio 事件循环中集成传统 SQLAlchemy “延迟加载” 的替代方法，提供了一个名为`AsyncSession.run_sync()`的**可选**方法，它将运行任何 Python 函数在一个 greenlet 中，传统的同步编程概念将被转换为在到达数据库驱动程序时使用`await`。这里的一个假设方法是一个面向 asyncio 的应用程序可以将与数据库相关的方法打包成函数，这些函数使用`AsyncSession.run_sync()`调用。

修改上面的示例，如果我们不使用`selectinload()`来加载`A.bs`集合，我们可以在一个单独的函数中完成对这些属性访问的处理：

```py
import asyncio

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

def fetch_and_update_objects(session):
  """run traditional sync-style ORM code in a function that will be
 invoked within an awaitable.

 """

    # the session object here is a traditional ORM Session.
    # all features are available here including legacy Query use.

    stmt = select(A)

    result = session.execute(stmt)
    for a1 in result.scalars():
        print(a1)

        # lazy loads
        for b1 in a1.bs:
            print(b1)

    # legacy Query use
    a1 = session.query(A).order_by(A.id).first()

    a1.data = "new data"

async def async_main():
    engine = create_async_engine(
        "postgresql+asyncpg://scott:tiger@localhost/test",
        echo=True,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)

    async with AsyncSession(engine) as session:
        async with session.begin():
            session.add_all(
                [
                    A(bs=[B(), B()], data="a1"),
                    A(bs=[B()], data="a2"),
                    A(bs=[B(), B()], data="a3"),
                ]
            )

        await session.run_sync(fetch_and_update_objects)

        await session.commit()

    # for AsyncEngine created in function scope, close and
    # clean-up pooled connections
    await engine.dispose()

asyncio.run(async_main())
```

在一个应用程序中运行某些函数在“同步”运行器中的上述方法与在一个基于事件的编程库（如`gevent`）上运行 SQLAlchemy 应用程序有一些相似之处。区别如下：

1.  与使用`gevent`不同，我们可以继续使用标准的 Python asyncio 事件循环，或任何自定义事件循环，而无需集成到`gevent`事件循环中。

1.  完全没有“monkeypatching”。上面的示例使用了一个真正的 asyncio 驱动程序，底层的 SQLAlchemy 连接池也使用 Python 内置的`asyncio.Queue`来池化连接。

1.  程序可以自由地在 async/await 代码和使用同步代码的包含函数之间切换，几乎没有性能损失。没有“线程执行器”或任何额外的等待器或同步在使用。

1.  底层网络驱动程序也使用纯 Python asyncio 概念，不使用`gevent`和`eventlet`提供的第三方网络库。### 与并发任务一起使用 AsyncSession

`AsyncSession`对象是一个**可变的、有状态的对象**，代表着**正在进行的单个、有状态的数据库事务**。使用 asyncio 进行并发任务，例如使用`asyncio.gather()`等 API，应该为每个单独的任务使用一个**独立的**`AsyncSession`。

请参阅会话是否线程安全？AsyncSession 是否可以在并发任务中共享？部分，了解关于`Session`和`AsyncSession`在处理并发工作负载时应如何使用的一般描述。

### 使用 AsyncSession 时防止隐式 IO

使用传统的 asyncio，应用程序需要避免发生可能导致 IO-on-attribute 访问的任何点。下面列出的技术可以帮助实现这一点，其中许多在前面的示例中有所说明。

+   延迟加载关系、延迟列或表达式，或在过期情况下访问的属性可以利用`AsyncAttrs` mixin。当将此 mixin 添加到特定类或更一般地添加到 Declarative `Base`超类时，会提供一个访问器`AsyncAttrs.awaitable_attrs`，将任何属性作为可等待对象传递：

    ```py
    from __future__ import annotations

    from typing import List

    from sqlalchemy.ext.asyncio import AsyncAttrs
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import relationship

    class Base(AsyncAttrs, DeclarativeBase):
        pass

    class A(Base):
        __tablename__ = "a"

        # ... rest of mapping ...

        bs: Mapped[List[B]] = relationship()

    class B(Base):
        __tablename__ = "b"

        # ... rest of mapping ...
    ```

    在不使用急加载时，访问新加载实例`A`上的`A.bs`集合通常会使用延迟加载，为了成功，通常会向数据库发出 IO 请求，而在 asyncio 下会失败，因为不允许隐式 IO。要在 asyncio 下直接访问此属性而不需要任何先前的加载操作，可以通过指定`AsyncAttrs.awaitable_attrs`前缀将属性访问为可等待对象：

    ```py
    a1 = (await session.scalars(select(A))).one()
    for b1 in await a1.awaitable_attrs.bs:
        print(b1)
    ```

    `AsyncAttrs` mixin 提供了一个简洁的外观，也是`AsyncSession.run_sync()`方法使用的内部方法。

    版本 2.0.13 中的新功能。

    另请参见

    `AsyncAttrs`

+   集合可以被**只写集合**替换，永远不会隐式发出 IO，通过在 SQLAlchemy 2.0 中使用只写关系功能。使用此功能，集合永远不会被读取，只能使用显式 SQL 调用进行查询。请参见 Asyncio Integration 部分中的示例`async_orm_writeonly.py`，演示了在 asyncio 中使用只写集合的示例。

    当使用只写集合时，程序在处理集合方面的行为简单且易于预测。然而，缺点是没有任何内置系统可以一次性加载许多这些集合，而是需要手动执行。因此，下面的许多要点涉及在使用传统的延迟加载关系与 asyncio 时需要更加小心的具体技术。

+   如果不使用`AsyncAttrs`，可以使用`lazy="raise"`声明关系，这样默认情况下它们不会尝试发出 SQL。为了加载集合，将使用急加载。

+   最有用的急加载策略是`selectinload()`急加载器，在前面的示例中用于在`await session.execute()`调用范围内急加载`A.bs`集合：

    ```py
    stmt = select(A).options(selectinload(A.bs))
    ```

+   在构建新对象时，**集合总是分配一个默认的空集合**，例如上面示例中的列表：

    ```py
    A(bs=[], data="a2")
    ```

    当`A`对象被刷新时，允许上述`A`对象上的`.bs`集合存在并可读；否则，当`A`被刷新时，`.bs`将被卸载并在访问时引发错误。

+   `AsyncSession`配置为使用`Session.expire_on_commit`设置为 False，这样我们可以在调用`AsyncSession.commit()`后访问对象的属性，就像在最后一行访问属性时一样：

    ```py
    # create AsyncSession with expire_on_commit=False
    async_session = AsyncSession(engine, expire_on_commit=False)

    # sessionmaker version
    async_session = async_sessionmaker(engine, expire_on_commit=False)

    async with async_session() as session:
        result = await session.execute(select(A).order_by(A.id))

        a1 = result.scalars().first()

        # commit would normally expire all attributes
        await session.commit()

        # access attribute subsequent to commit; this is what
        # expire_on_commit=False allows
        print(a1.data)
    ```

其他指南包括：

+   像`AsyncSession.expire()`这样的方法应该避免使用，而应该使用`AsyncSession.refresh()`；**如果**绝对需要过期。通常情况下不应该需要过期，因为在使用 asyncio 时通常应将`Session.expire_on_commit`设置为`False`。

+   使用`AsyncSession.refresh()`可以显式加载懒加载关系**在 asyncio 下**，**如果**需要显式传递所需的属性名称给`Session.refresh.attribute_names`，例如：

    ```py
    # assume a_obj is an A that has lazy loaded A.bs collection
    a_obj = await async_session.get(A, [1])

    # force the collection to load by naming it in attribute_names
    await async_session.refresh(a_obj, ["bs"])

    # collection is present
    print(f"bs collection: {a_obj.bs}")
    ```

    当然最好一开始就使用急加载，以便在不需要懒加载的情况下已经设置好集合。

    2.0.4 版本中新增：增加了对`AsyncSession.refresh()`和底层的`Session.refresh()`方法的支持，以强制延迟加载的关系加载，如果它们在`Session.refresh.attribute_names`参数中明确命名。在先前的版本中，即使在参数中命名，关系也会被静默跳过。

+   避免使用 Cascades 文档中记录的`all`级联选项，而是明确列出所需的级联特性。`all`级联选项暗示了 refresh-expire 设置，这意味着`AsyncSession.refresh()`方法将使相关对象的属性过期，但不一定会刷新那些相关对象，假设未在`relationship()`中配置急加载，它们将保持在过期状态。

+   如果使用`deferred()`列，应该使用适当的加载器选项，除了如上所述的`relationship()`构造。有关延迟列加载的背景，请参阅限制使用列延迟加载。

+   在默认情况下，“动态”关系加载策略在动态关系加载器中描述，与 asyncio 方法不兼容。只有在`AsyncSession.run_sync()`方法中调用，或者通过使用其`.statement`属性获取正常的 select 语句，才能直接使用它：

    ```py
    user = await session.get(User, 42)
    addresses = (await session.scalars(user.addresses.statement)).all()
    stmt = user.addresses.statement.where(Address.email_address.startswith("patrick"))
    addresses_filter = (await session.scalars(stmt)).all()
    ```

    仅写入技术，在 SQLAlchemy 的 2.0 版本中引入，与 asyncio 完全兼容，应该优先考虑使用。

    另请参见

    “动态”关系加载器被“仅写入”取代 - 迁移到 2.0 风格的注意事项

+   如果在使用 asyncio 与不支持 RETURNING 的数据库（例如 MySQL 8）时，服务器默认值（例如生成的时间戳）将不会在新刷新的对象上可用，除非使用了`Mapper.eager_defaults` 选项。在 SQLAlchemy 2.0 中，这种行为会自动应用于像 PostgreSQL、SQLite 和 MariaDB 这样使用 RETURNING 在插入行时获取新值的后端。

### 在 asyncio 下运行同步方法和函数

深度合成

这种方法实质上是公开了 SQLAlchemy 能够在第一时间提供 asyncio 接口的机制。虽然这样做没有技术问题，但总的来说，这种方法可能被认为是“有争议的”，因为它违背了 asyncio 编程模型的一些核心理念，即任何可能导致 IO 调用的编程语句**必须**具有一个 `await` 调用，否则程序在 IO 可能发生的每一行都不会明确地表明。这种方法并没有改变这个一般性的想法，除了它允许一系列同步 IO 指令在函数调用的范围内被豁免这个规则，本质上被捆绑成一个可等待的。

作为在 asyncio 事件循环中集成传统 SQLAlchemy “懒加载”的另一种替代方法，提供了一种称为`AsyncSession.run_sync()` 的**可选**方法，该方法将在一个 greenlet 内运行任何 Python 函数，其中当它们到达数据库驱动程序时，传统的同步编程概念将被转换为使用 `await`。这里的一个假设性方法是，一个以 asyncio 为导向的应用程序可以将数据库相关方法打包成函数，这些函数将使用`AsyncSession.run_sync()` 被调用。

修改上面的示例，如果我们不使用`selectinload()` 来处理 `A.bs` 集合，我们可以在一个单独的函数内完成对这些属性访问的处理：

```py
import asyncio

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

def fetch_and_update_objects(session):
  """run traditional sync-style ORM code in a function that will be
 invoked within an awaitable.

 """

    # the session object here is a traditional ORM Session.
    # all features are available here including legacy Query use.

    stmt = select(A)

    result = session.execute(stmt)
    for a1 in result.scalars():
        print(a1)

        # lazy loads
        for b1 in a1.bs:
            print(b1)

    # legacy Query use
    a1 = session.query(A).order_by(A.id).first()

    a1.data = "new data"

async def async_main():
    engine = create_async_engine(
        "postgresql+asyncpg://scott:tiger@localhost/test",
        echo=True,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
        await conn.run_sync(Base.metadata.create_all)

    async with AsyncSession(engine) as session:
        async with session.begin():
            session.add_all(
                [
                    A(bs=[B(), B()], data="a1"),
                    A(bs=[B()], data="a2"),
                    A(bs=[B(), B()], data="a3"),
                ]
            )

        await session.run_sync(fetch_and_update_objects)

        await session.commit()

    # for AsyncEngine created in function scope, close and
    # clean-up pooled connections
    await engine.dispose()

asyncio.run(async_main())
```

在“同步”运行器中运行某些函数的上述方法与在基于事件的编程库（例如 `gevent`）上运行 SQLAlchemy 应用程序有一些相似之处。区别如下：

1.  与使用 `gevent` 不同，我们可以继续使用标准的 Python asyncio 事件循环，或者任何自定义的事件循环，而无需将其集成到 `gevent` 事件循环中。

1.  完全没有“猴子补丁”。上面的示例利用了一个真正的 asyncio 驱动程序，底层的 SQLAlchemy 连接池也使用了 Python 内置的 `asyncio.Queue` 来池化连接。

1.  程序可以自由在 async/await 代码和使用同步代码的包含函数之间切换，几乎没有性能损失。不使用“线程执行器”或任何额外的等待器或同步。

1.  底层网络驱动程序也使用纯 Python asyncio 概念，不使用`gevent`和`eventlet`等第三方网络库。

## 使用 asyncio 扩展与事件

SQLAlchemy 事件系统不会直接暴露给 asyncio 扩展，这意味着尚未有 SQLAlchemy 事件处理程序的“异步”版本。

然而，由于 asyncio 扩展包围了通常的同步 SQLAlchemy API，因此常规的“同步”风格事件处理程序可以自由使用，就像没有使用 asyncio 一样。

如下所述，目前有两种注册事件的策略，针对面向 asyncio 的 API：

+   事件可以在实例级别注册（例如特定的`AsyncEngine`实例），通过将事件与引用代理对象的`sync`属性关联。例如，要针对`AsyncEngine`实例注册`PoolEvents.connect()`事件，请使用其`AsyncEngine.sync_engine`属性作为目标。目标包括：

    > `AsyncEngine.sync_engine`
    > 
    > `AsyncConnection.sync_connection`
    > 
    > `AsyncConnection.sync_engine`
    > 
    > `AsyncSession.sync_session`

+   要在类级别注册事件，针对同一类型的所有实例（例如所有`AsyncSession`实例），请使用相应的同步风格类。例如，要针对`AsyncSession`类注册`SessionEvents.before_commit()`事件，请将`Session`类作为目标。

+   要在`sessionmaker`级别注册，结合一个明确的`sessionmaker`和一个`async_sessionmaker`，使用`async_sessionmaker.sync_session_class`，并将事件与`sessionmaker`关联。

在异步上下文中工作的事件处理程序中，像`Connection`这样的对象继续以通常的“同步”方式工作，而不需要`await`或`async`的使用；当消息最终被异步数据库适配器接收时，调用风格会透明地转换回异步调用风格。对于传递了 DBAPI 级别连接的事件，例如`PoolEvents.connect()`，该对象是一个符合 pep-249 的“连接”对象，它将同步样式调用转换为异步驱动程序。

### 具有异步引擎/会话/会话制造器的事件监听器示例

一些与面向异步 API 构造相关的同步样式事件处理程序示例如下：

+   **在 AsyncEngine 上的核心事件**

    在这个例子中，我们访问`AsyncEngine.sync_engine`属性，作为`ConnectionEvents`和`PoolEvents`的目标：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.engine import Engine
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")

    # connect event on instance of Engine
    @event.listens_for(engine.sync_engine, "connect")
    def my_on_connect(dbapi_con, connection_record):
        print("New DBAPI connection:", dbapi_con)
        cursor = dbapi_con.cursor()

        # sync style API use for adapted DBAPI connection / cursor
        cursor.execute("select 'execute from event'")
        print(cursor.fetchone()[0])

    # before_execute event on all Engine instances
    @event.listens_for(Engine, "before_execute")
    def my_before_execute(
        conn,
        clauseelement,
        multiparams,
        params,
        execution_options,
    ):
        print("before execute!")

    async def go():
        async with engine.connect() as conn:
            await conn.execute(text("select 1"))
        await engine.dispose()

    asyncio.run(go())
    ```

    输出：

    ```py
    New DBAPI connection: <AdaptedConnection <asyncpg.connection.Connection object at 0x7f33f9b16960>>
    execute from event
    before execute!
    ```

+   **在 AsyncSession 上的 ORM 事件**

    在这个例子中，我们访问`AsyncSession.sync_session`作为`SessionEvents`的目标：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.ext.asyncio import AsyncSession
    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.orm import Session

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")

    session = AsyncSession(engine)

    # before_commit event on instance of Session
    @event.listens_for(session.sync_session, "before_commit")
    def my_before_commit(session):
        print("before commit!")

        # sync style API use on Session
        connection = session.connection()

        # sync style API use on Connection
        result = connection.execute(text("select 'execute from event'"))
        print(result.first())

    # after_commit event on all Session instances
    @event.listens_for(Session, "after_commit")
    def my_after_commit(session):
        print("after commit!")

    async def go():
        await session.execute(text("select 1"))
        await session.commit()

        await session.close()
        await engine.dispose()

    asyncio.run(go())
    ```

    输出：

    ```py
    before commit!
    execute from event
    after commit!
    ```

+   **在 async_sessionmaker 上的 ORM 事件**

    对于这种用例，我们将`sessionmaker`作为事件目标，然后将其分配给`async_sessionmaker`，使用`async_sessionmaker.sync_session_class`参数：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy.ext.asyncio import async_sessionmaker
    from sqlalchemy.orm import sessionmaker

    sync_maker = sessionmaker()
    maker = async_sessionmaker(sync_session_class=sync_maker)

    @event.listens_for(sync_maker, "before_commit")
    def before_commit(session):
        print("before commit")

    async def main():
        async_session = maker()

        await async_session.commit()

    asyncio.run(main())
    ```

    输出：

    ```py
    before commit
    ```

### 在连接池和其他事件中使用仅可等待的驱动程序方法

如上节所述，事件处理程序（例如那些围绕 `PoolEvents` 的事件处理程序）接收到一个同步风格的“DBAPI”连接，这是由 SQLAlchemy asyncio 方言提供的包装对象，用于将底层 asyncio “驱动程序”连接适配成可以被 SQLAlchemy 内部使用的对象。当用户定义的实现需要直接使用最终的“驱动程序”连接，并在该驱动程序连接上使用仅可等待方法时，就会出现特殊的用例。其中一个例子是 asyncpg 驱动程序提供的 `.set_type_codec()` 方法。

为了适应这种用例，SQLAlchemy 的 `AdaptedConnection` 类提供了一个方法 `AdaptedConnection.run_async()`，允许在事件处理程序或其他 SQLAlchemy 内部的“同步”上下文中调用可等待函数。这个方法直接对应于 `AsyncConnection.run_sync()` 方法，后者允许在异步环境中运行同步风格的方法。

`AdaptedConnection.run_async()` 应该传递一个函数，该函数将接受内部的“驱动程序”连接作为单个参数，并返回一个可等待对象，该对象将由 `AdaptedConnection.run_async()` 方法调用。给定的函数本身不需要声明为 `async`；它可以是一个 Python 的 `lambda:`，因为返回的可等待值将在返回后被调用：

```py
from sqlalchemy import event
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(...)

@event.listens_for(engine.sync_engine, "connect")
def register_custom_types(dbapi_connection, *args):
    dbapi_connection.run_async(
        lambda connection: connection.set_type_codec(
            "MyCustomType",
            encoder,
            decoder,  # ...
        )
    )
```

在上面的例子中，传递给 `register_custom_types` 事件处理程序的对象是 `AdaptedConnection` 的一个实例，它提供了对底层仅支持异步驱动程序级连接对象的类似 DBAPI 的接口。然后，`AdaptedConnection.run_async()` 方法提供了对一个可等待环境的访问，在该环境中可以对底层驱动程序级连接进行操作。

版本 1.4.30 中的新功能。

### 异步引擎 / 会话 / 会话工厂的事件监听器示例

下面给出了一些与异步 API 构造相关的同步风格事件处理程序的示例：

+   **异步引擎的核心事件**

    在这个例子中，我们将`AsyncEngine.sync_engine`属性作为`ConnectionEvents`和`PoolEvents`的目标：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.engine import Engine
    from sqlalchemy.ext.asyncio import create_async_engine

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")

    # connect event on instance of Engine
    @event.listens_for(engine.sync_engine, "connect")
    def my_on_connect(dbapi_con, connection_record):
        print("New DBAPI connection:", dbapi_con)
        cursor = dbapi_con.cursor()

        # sync style API use for adapted DBAPI connection / cursor
        cursor.execute("select 'execute from event'")
        print(cursor.fetchone()[0])

    # before_execute event on all Engine instances
    @event.listens_for(Engine, "before_execute")
    def my_before_execute(
        conn,
        clauseelement,
        multiparams,
        params,
        execution_options,
    ):
        print("before execute!")

    async def go():
        async with engine.connect() as conn:
            await conn.execute(text("select 1"))
        await engine.dispose()

    asyncio.run(go())
    ```

    输出：

    ```py
    New DBAPI connection: <AdaptedConnection <asyncpg.connection.Connection object at 0x7f33f9b16960>>
    execute from event
    before execute!
    ```

+   **AsyncSession 上的 ORM 事件**

    在这个例子中，我们将`AsyncSession.sync_session`作为`SessionEvents`的目标：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy import text
    from sqlalchemy.ext.asyncio import AsyncSession
    from sqlalchemy.ext.asyncio import create_async_engine
    from sqlalchemy.orm import Session

    engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost:5432/test")

    session = AsyncSession(engine)

    # before_commit event on instance of Session
    @event.listens_for(session.sync_session, "before_commit")
    def my_before_commit(session):
        print("before commit!")

        # sync style API use on Session
        connection = session.connection()

        # sync style API use on Connection
        result = connection.execute(text("select 'execute from event'"))
        print(result.first())

    # after_commit event on all Session instances
    @event.listens_for(Session, "after_commit")
    def my_after_commit(session):
        print("after commit!")

    async def go():
        await session.execute(text("select 1"))
        await session.commit()

        await session.close()
        await engine.dispose()

    asyncio.run(go())
    ```

    输出：

    ```py
    before commit!
    execute from event
    after commit!
    ```

+   **异步会话工厂上的 ORM 事件**

    对于这种用例，我们将`sessionmaker`作为事件目标，然后使用`async_sessionmaker`并使用`async_sessionmaker.sync_session_class`参数进行赋值：

    ```py
    import asyncio

    from sqlalchemy import event
    from sqlalchemy.ext.asyncio import async_sessionmaker
    from sqlalchemy.orm import sessionmaker

    sync_maker = sessionmaker()
    maker = async_sessionmaker(sync_session_class=sync_maker)

    @event.listens_for(sync_maker, "before_commit")
    def before_commit(session):
        print("before commit")

    async def main():
        async_session = maker()

        await async_session.commit()

    asyncio.run(main())
    ```

    输出：

    ```py
    before commit
    ```

### 在连接池和其他事件中使用仅可等待的驱动程序方法

如上节所述，事件处理程序（例如围绕`PoolEvents`事件处理程序定位的事件处理程序）接收到一个同步风格的“DBAPI”连接，这是 SQLAlchemy asyncio 方言提供的包装对象，用于将底层的 asyncio“driver”连接适配为 SQLAlchemy 内部可以使用的连接。当用户定义的实现需要直接使用最终的“driver”连接时，使用该驱动连接上的仅可等待方法时会出现特殊的用例。一个这样的例子是 asyncpg 驱动程序提供的`.set_type_codec()`方法。

为了适应这种用例，SQLAlchemy 的`AdaptedConnection`类提供了一个方法`AdaptedConnection.run_async()`，允许在事件处理程序或其他 SQLAlchemy 内部的“同步”上下文中调用可等待函数。这个方法直接类似于`AsyncConnection.run_sync()`方法，允许同步风格的方法在异步下运行。

`AdaptedConnection.run_async()`应该传递一个接受最内层的“driver”连接作为单个参数的函数，并返回一个由`AdaptedConnection.run_async()`方法调用的可等待对象。给定函数本身不需要声明为`async`；它完全可以是一个 Python 的`lambda:`，因为返回的可等待值将在返回后被调用：

```py
from sqlalchemy import event
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine(...)

@event.listens_for(engine.sync_engine, "connect")
def register_custom_types(dbapi_connection, *args):
    dbapi_connection.run_async(
        lambda connection: connection.set_type_codec(
            "MyCustomType",
            encoder,
            decoder,  # ...
        )
    )
```

上面，传递给`register_custom_types`事件处理程序的对象是`AdaptedConnection`的一个实例，它提供了对底层仅异步驱动程序级连接对象的类似 DBAPI 的接口。然后，`AdaptedConnection.run_async()`方法提供了对可等待环境的访问，其中底层驱动程序级连接可以被操作。

1.4.30 版本中新增。

## 使用多个 asyncio 事件循环

使用多个事件循环的应用程序，例如在将 asyncio 与多线程结合的不常见情况下，在使用默认池实现时不应该将同一个`AsyncEngine`与不同的事件循环共享。

如果从一个事件循环传递`AsyncEngine`到另一个事件循环，应该在它被重新使用于新的事件循环之前调用`AsyncEngine.dispose()`方法。未这样做可能会导致类似于`Task <Task pending ...> got Future attached to a different loop`的`RuntimeError`。

如果同一个引擎必须在不同的循环之间共享，应该配置为禁用连接池，使用`NullPool`，防止引擎使用任何连接超过一次：

```py
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.pool import NullPool

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@host/dbname",
    poolclass=NullPool,
)
```

## 使用 asyncio scoped session

在使用带有`scoped_session`对象的线程化 SQLAlchemy 中使用的“scoped session”模式也可在 asyncio 中使用，使用一个名为`async_scoped_session`的调整版本。

小贴士

SQLAlchemy 通常不建议在新开发中使用“scoped”模式，因为它依赖于可变的全局状态，当线程或任务内的工作完成时，必须明确地将其销毁。特别是在使用 asyncio 时，直接将`AsyncSession`传递给需要它的可等待函数可能是一个更好的主意。

当使用`async_scoped_session`时，由于 asyncio 上下文中没有“线程本地”概念，必须向构造函数提供“scopefunc”参数。下面的示例演示了使用`asyncio.current_task()`函数来实现这一目的：

```py
from asyncio import current_task

from sqlalchemy.ext.asyncio import (
    async_scoped_session,
    async_sessionmaker,
)

async_session_factory = async_sessionmaker(
    some_async_engine,
    expire_on_commit=False,
)
AsyncScopedSession = async_scoped_session(
    async_session_factory,
    scopefunc=current_task,
)
some_async_session = AsyncScopedSession()
```

警告

使用`async_scoped_session`中的“scopefunc”在任务中被**任意次**调用，每次访问底层的`AsyncSession`时都会被调用。因此，该函数应该是**幂等的**和轻量级的，并且不应尝试创建或改变任何状态，比如建立回调等。

警告

在作用于范围的“key”中使用`current_task()`要求在最外层可等待内调用`async_scoped_session.remove()`方法，以确保在任务完成时从注册表中移除该键，否则任务句柄以及`AsyncSession`将仍然保留在内存中，实质上创建了内存泄漏。请参阅以下示例，演示了`async_scoped_session.remove()`的正确使用方法。

`async_scoped_session`包括与`scoped_session`类似的**代理行为**，这意味着它可以直接被视为`AsyncSession`，需要注意的是，通常需要使用`await`关键字，包括`async_scoped_session.remove()`方法：

```py
async def some_function(some_async_session, some_object):
    # use the AsyncSession directly
    some_async_session.add(some_object)

    # use the AsyncSession via the context-local proxy
    await AsyncScopedSession.commit()

    # "remove" the current proxied AsyncSession for the local context
    await AsyncScopedSession.remove()
```

自 1.4.19 版本新增。

## 使用检视器检视模式对象

SQLAlchemy 目前尚未提供`Inspector`的 asyncio 版本（在使用 Inspector 进行细粒度反射中介绍），但是现有的接口可以通过利用`AsyncConnection.run_sync()`方法的`AsyncConnection`在 asyncio 上下文中使用：

```py
import asyncio

from sqlalchemy import inspect
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine("postgresql+asyncpg://scott:tiger@localhost/test")

def use_inspector(conn):
    inspector = inspect(conn)
    # use the inspector
    print(inspector.get_view_names())
    # return any value to the caller
    return inspector.get_table_names()

async def async_main():
    async with engine.connect() as conn:
        tables = await conn.run_sync(use_inspector)

asyncio.run(async_main())
```

另请参阅

反射数据库对象

运行时检查 API

## 引擎 API 文档

| 对象名称 | 描述 |
| --- | --- |
| async_engine_from_config(configuration[, prefix], **kwargs) | 使用配置字典创建一个新的 AsyncEngine 实例。 |
| AsyncConnection | 用于`Connection`的 asyncio 代理。 |
| AsyncEngine | 用于`Engine`的 asyncio 代理。 |
| AsyncTransaction | 用于`Transaction`的 asyncio 代理。 |
| create_async_engine(url, **kw) | 创建一个新的异步引擎实例。 |
| create_async_pool_from_url(url, **kwargs) | 创建一个新的异步引擎实例。 |

```py
function sqlalchemy.ext.asyncio.create_async_engine(url: str | URL, **kw: Any) → AsyncEngine
```

创建一个新的异���引擎实例。

传递给`create_async_engine()`的参数与传递给`create_engine()`函数的参数基本相同。指定的方言必须是一个 asyncio 兼容的方言，如 asyncpg。

版本 1.4 中的新功能。

参数：

**async_creator** –

一个异步可调用函数，返回一个驱动级别的 asyncio 连接。如果提供了该函数，它不应该带任何参数，并且应该从底层 asyncio 数据库驱动程序返回一个新的 asyncio 连接；该连接将被包装在适当的结构中，以便与`AsyncEngine`一起使用。请注意，URL 中指定的参数在此处不适用，创建函数应该使用自己的连接参数。

这个参数是 `create_engine()` 函数的 asyncio 等效参数。

版本 2.0.16 中新增。

```py
function sqlalchemy.ext.asyncio.async_engine_from_config(configuration: Dict[str, Any], prefix: str = 'sqlalchemy.', **kwargs: Any) → AsyncEngine
```

使用配置字典创建一个新的 AsyncEngine 实例。

这个函数类似于 SQLAlchemy 核心中的 `engine_from_config()` 函数，不同之处在于请求的方言必须是类似于 asyncpg 这样的 asyncio 兼容方言。函数的参数签名与 `engine_from_config()` 完全相同。

版本 1.4.29 中新增。

```py
function sqlalchemy.ext.asyncio.create_async_pool_from_url(url: str | URL, **kwargs: Any) → Pool
```

创建一个新的异步引擎实例。

传递给 `create_async_pool_from_url()` 的参数大部分与传递给 `create_pool_from_url()` 函数的参数相同。指定的方言必须是类似于 asyncpg 这样的 asyncio 兼容方言。

版本 2.0.10 中新增。

```py
class sqlalchemy.ext.asyncio.AsyncEngine
```

一个 `Engine` 的 asyncio 代理。

`AsyncEngine` 是通过 `create_async_engine()` 函数获取的：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("postgresql+asyncpg://user:pass@host/dbname")
```

版本 1.4 中新增。

**成员**

begin(), clear_compiled_cache(), connect(), dialect, dispose(), driver, echo, engine, execution_options(), get_execution_options(), name, pool, raw_connection(), sync_engine, update_execution_options(), url

**类签名**

类`sqlalchemy.ext.asyncio.AsyncEngine`（`sqlalchemy.ext.asyncio.base.ProxyComparable`，`sqlalchemy.ext.asyncio.AsyncConnectable`）

```py
method begin() → AsyncIterator[AsyncConnection]
```

返回一个上下文管理器，当进入时将提供一个已建立`AsyncTransaction`的`AsyncConnection`。

例如：

```py
async with async_engine.begin() as conn:
    await conn.execute(
        text("insert into table (x, y, z) values (1, 2, 3)")
    )
    await conn.execute(text("my_special_procedure(5)"))
```

```py
method clear_compiled_cache() → None
```

清除与方言关联的已编译缓存。

代表`AsyncEngine`类的`Engine`类。

这仅适用于通过`create_engine.query_cache_size`参数建立的内置缓存。它不会影响通过`Connection.execution_options.compiled_cache`参数传递的任何字典缓存。

版本 1.4 中的新功能。

```py
method connect() → AsyncConnection
```

返回一个`AsyncConnection`对象。

当作为异步上下文管理器输入时，`AsyncConnection`将从底层连接池中获取数据库连接：

```py
async with async_engine.connect() as conn:
    result = await conn.execute(select(user_table))
```

也可以通过调用其`AsyncConnection.start()`方法在上下文管理器之外启动`AsyncConnection`。

```py
attribute dialect
```

代表`AsyncEngine`类的`Engine.dialect`属性的代理。

```py
method async dispose(close: bool = True) → None
```

处置此`AsyncEngine`使用的连接池。

参数：

**close** –

如果保持默认值`True`，则会完全关闭所有**当前已签入**的数据库连接。仍在签出的连接将**不会**被关闭，但它们将不再与此`Engine`关联，因此当它们被单独关闭时，它们将最终被垃圾回收，并且如果尚未在签入时关闭，则它们将被完全关闭。

如果设置为`False`，则先前的连接池将被取消引用，否则不会以任何方式触及。

另请参见

`Engine.dispose()`

```py
attribute driver
```

此`Engine`正在使用的`Dialect`的驱动程序名称。

代理`Engine`类，代表`AsyncEngine`类。

```py
attribute echo
```

当为`True`时，启用此元素的日志输出。

代理`Engine`类，代表`AsyncEngine`类。

这将设置此元素的类和对象引用的 Python 日志级别的效果。布尔值`True`表示将为记录器设置日志级别`logging.INFO`，而字符串值`debug`将将日志级别设置为`logging.DEBUG`。

```py
attribute engine
```

返回此`Engine`。

代理`Engine`类，代表`AsyncEngine`类。

用于接受相同变量中的`Connection` / `Engine`对象的传统方案。

```py
method execution_options(**opt: Any) → AsyncEngine
```

返回一个新的`AsyncEngine`，将提供具有给定执行选项的`AsyncConnection`对象。

从`Engine.execution_options()`代理。有关详细信息，请参阅该方法。

```py
method get_execution_options() → _ExecuteOptions
```

获取在执行期间生效的非 SQL 选项。

代理`Engine`类，代表`AsyncEngine`类。

另请参阅

`Engine.execution_options()`

```py
attribute name
```

此`Engine`正在使用的`Dialect`的字符串名称。

代理`Engine`类，代表`AsyncEngine`类。

```py
attribute pool
```

代理`Engine.pool`属性，代表`AsyncEngine`类。

```py
method async raw_connection() → PoolProxiedConnection
```

从连接池返回“原始”DBAPI 连接。

另请参阅

与 Driver SQL 和原始 DBAPI 连接一起工作

```py
attribute sync_engine: Engine
```

此`AsyncEngine`代理请求的同步式`Engine`的引用。

此实例可用作事件目标。

另请参阅

使用 asyncio 扩展处理事件

```py
method update_execution_options(**opt: Any) → None
```

更新此`Engine`的默认执行选项字典。

代理`AsyncEngine`类的`Engine`类。

**opt 中给定的键/值将添加到将用于所有连接的默认执行选项中。此字典的初始内容可以通过`execution_options`参数发送到`create_engine()`。

另请参阅

`Connection.execution_options()`

`Engine.execution_options()`

```py
attribute url
```

代理`AsyncEngine`类的`Engine.url`属性。

```py
class sqlalchemy.ext.asyncio.AsyncConnection
```

一个`Connection`的 asyncio 代理。

使用`AsyncEngine.connect()`方法获取`AsyncConnection`:

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("postgresql+asyncpg://user:pass@host/dbname")

async with engine.connect() as conn:
    result = await conn.execute(select(table))
```

版本 1.4 中的新功能。

**成员**

aclose(), begin(), begin_nested(), close(), closed, commit(), connection, default_isolation_level, dialect, exec_driver_sql(), execute(), execution_options(), get_nested_transaction(), get_raw_connection(), get_transaction(), in_nested_transaction(), in_transaction(), info, invalidate(), invalidated, rollback(), run_sync(), scalar(), scalars(), start(), stream(), stream_scalars(), sync_connection, sync_engine

**类签名**

类`sqlalchemy.ext.asyncio.AsyncConnection` (`sqlalchemy.ext.asyncio.base.ProxyComparable`, `sqlalchemy.ext.asyncio.base.StartableContext`, `sqlalchemy.ext.asyncio.AsyncConnectable`)

```py
method async aclose() → None
```

`AsyncConnection.close()`的同义词。

`AsyncConnection.aclose()` 名称特别支持 Python 标准库 `@contextlib.aclosing` 上下文管理器函数。

在 2.0.20 版本中新增。

```py
method begin() → AsyncTransaction
```

在自动开始之前开始一个事务。

```py
method begin_nested() → AsyncTransaction
```

开始一个嵌套事务并返回一个事务句柄。

```py
method async close() → None
```

关闭这个`AsyncConnection`。

这也会导致事务回滚（如果存在的话）。

```py
attribute closed
```

返回 `True` 如果此连接已关闭。

代表`AsyncConnection`类的`Connection`类的代理。

```py
method async commit() → None
```

提交当前正在进行的事务。

如果已启动事务，则此方法会提交当前事务。如果未启动事务，则该方法不起作用，假定连接处于非失效状态。

当首次执行语句或调用 `Connection.begin()` 方法时，`Connection`自动开始事务。

```py
attribute connection
```

未实现异步; 调用 `AsyncConnection.get_raw_connection()`。

```py
attribute default_isolation_level
```

与正在使用的`Dialect`相关联的初始连接时间隔离级别。

代表`AsyncConnection`类的`Connection`类的代理。

此值与`Connection.execution_options.isolation_level`和`Engine.execution_options.isolation_level`执行选项无关，并且由`Dialect`在创建第一个连接时确定，通过对数据库执行 SQL 查询以获取当前隔离级别，然后在发出任何其他命令之前。 

调用此访问器不会触发任何新的 SQL 查询。

另请参阅

`Connection.get_isolation_level()` - 查看当前实际隔离级别

`create_engine.isolation_level` - 设置每个`Engine`的隔离级别

`Connection.execution_options.isolation_level` - 设置每个`Connection`的隔离级别

```py
attribute dialect
```

代表`AsyncConnection`类的`Connection.dialect`属性的代理。

```py
method async exec_driver_sql(statement: str, parameters: _DBAPIAnyExecuteParams | None = None, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

执行驱动程序级别的 SQL 字符串并返回缓冲的`Result`。

```py
method async execute(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → CursorResult[Any]
```

执行 SQL 语句构造并返回一个缓冲的`Result`。

参数：

+   `object` –

    要执行的语句。这始终是同时在`ClauseElement`和`Executable`层次结构中的对象，包括：

    +   `Select`

    +   `Insert`, `Update`, `Delete`

    +   `TextClause`和`TextualSelect`

    +   `DDL`和从`ExecutableDDLElement`继承的对象

+   `parameters` – 将绑定到语句中的参数。这可以是参数名称到值的字典，也可以是可变序列（例如列表）的字典。当传递一个字典列表时，底层语句执行将使用 DBAPI `cursor.executemany()` 方法。当传递单个字典时，将使用 DBAPI `cursor.execute()` 方法。

+   `execution_options` – 可选的执行选项字典，将与语句执行关联。此字典可以提供`Connection.execution_options()`接受的选项子集。

返回：

一个`Result`对象。

```py
method async execution_options(**opt: Any) → AsyncConnection
```

为连接设置在执行期间生效的非 SQL 选项。

这返回带有新选项的此`AsyncConnection`对象。

有关此方法的详细信息，请参阅`Connection.execution_options()`。

```py
method get_nested_transaction() → AsyncTransaction | None
```

如果存在当前嵌套（保存点）事务，则返回表示当前嵌套（保存点）事务的`AsyncTransaction`。

这利用了底层同步连接的`Connection.get_nested_transaction()`方法来获取当前`Transaction`，然后在新的`AsyncTransaction`对象中进行代理。

新功能在版本 1.4.0b2 中新增。

```py
method async get_raw_connection() → PoolProxiedConnection
```

返回由此`AsyncConnection`使用的汇总的 DBAPI 级连接。

这是一个由 SQLAlchemy 连接池代理的连接，然后具有属性 `_ConnectionFairy.driver_connection`，该属性引用实际的驱动程序连接。其 `_ConnectionFairy.dbapi_connection` 引用的是将驱动程序连接适配到 DBAPI 协议的`AdaptedConnection`实例。

```py
method get_transaction() → AsyncTransaction | None
```

如果存在当前事务，则返回表示当前事务的`AsyncTransaction`。

这利用了底层同步连接的`Connection.get_transaction()`方法来获取当前`Transaction`，然后在新的`AsyncTransaction`对象中进行代理。

新功能在版本 1.4.0b2 中新增。

```py
method in_nested_transaction() → bool
```

如果事务正在进行中，则返回 True。

新功能在版本 1.4.0b2 中新增。

```py
method in_transaction() → bool
```

如果事务正在进行中，则返回 True。

```py
attribute info
```

返回底层`Connection`的`Connection.info`字典。

此字典可以自由写入，用于关联与数据库连接相关的用户定义状态。

仅当`AsyncConnection`当前连接时才可用此属性。如果`AsyncConnection.closed`属性为`True`，则访问此属性将引发`ResourceClosedError`。

在版本 1.4.0b2 中新增。

```py
method async invalidate(exception: BaseException | None = None) → None
```

使与此`Connection`相关联的底层 DBAPI 连接失效。

有关此方法的详细信息，请参阅`Connection.invalidate()`方法。

```py
attribute invalidated
```

如果此连接被使无效，则返回 True。

代理`AsyncConnection`类的`Connection`类。

这并不表示连接是否在池级别被使无效。

```py
method async rollback() → None
```

回滚当前正在进行的事务。

如果已启动事务，则此方法将回滚当前事务。如果未启动事务，则该方法不起作用。如果已启动事务且连接处于无效状态，则使用此方法清除事务。

每当首次执行语句或调用`Connection.begin()`方法时，`Connection`上都会自动开始事务。

```py
method async run_sync(fn: ~typing.Callable[[~typing.Concatenate[~sqlalchemy.engine.base.Connection, ~_P]], ~sqlalchemy.ext.asyncio.engine._T], *arg: ~typing.~_P, **kw: ~typing.~_P) → _T
```

调用给定的同步（即非异步）可调用对象，将同步风格的`Connection`作为第一个参数传递。

此方法允许传统的同步 SQLAlchemy 函数在异步应用程序的上下文中运行。

例如：

```py
def do_something_with_core(conn: Connection, arg1: int, arg2: str) -> str:
  '''A synchronous function that does not require awaiting

 :param conn: a Core SQLAlchemy Connection, used synchronously

 :return: an optional return value is supported

 '''
    conn.execute(
        some_table.insert().values(int_col=arg1, str_col=arg2)
    )
    return "success"

async def do_something_async(async_engine: AsyncEngine) -> None:
  '''an async function that uses awaiting'''

    async with async_engine.begin() as async_conn:
        # run do_something_with_core() with a sync-style
        # Connection, proxied into an awaitable
        return_code = await async_conn.run_sync(do_something_with_core, 5, "strval")
        print(return_code)
```

通过在特别调试的 greenlet 中运行给定的可调用对象，此方法将一直保持异步事件循环直到数据库连接。

`AsyncConnection.run_sync()`的最基本用法是调用诸如`MetaData.create_all()`之类的方法，给定一个需要提供给`MetaData.create_all()`的`AsyncConnection`对象作为`Connection`对象：

```py
# run metadata.create_all(conn) with a sync-style Connection,
# proxied into an awaitable
with async_engine.begin() as conn:
    await conn.run_sync(metadata.create_all)
```

注意

提供的可调用对象在 asyncio 事件循环中内联调用，并将阻塞传统 IO 调用。此可调用对象中的 IO 应仅调用到 SQLAlchemy 的 asyncio 数据库 API，这些 API 将被正确地适配到 greenlet 上下文。

另请参阅

`AsyncSession.run_sync()`

在 asyncio 下运行同步方法和函数

```py
method async scalar(statement: Executable, parameters: _CoreSingleExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → Any
```

执行 SQL 语句构造并返回标量对象。

此方法是在调用`Connection.execute()`方法后调用`Result.scalar()`方法的简写。参数是等效的。

返回：

代表返回的第一行的第一列的标量 Python 值。

```py
method async scalars(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → ScalarResult[Any]
```

执行 SQL 语句构造并返回标量对象。

此方法是在调用`Connection.execute()`方法后调用`Result.scalars()`方法的简写。参数是等效的。

返回：

一个`ScalarResult`对象。

新版本 1.4.24 中新增。

```py
method async start(is_ctxmanager: bool = False) → AsyncConnection
```

在使用 Python 的`with:`块之外启动此`AsyncConnection`对象的上下文。

```py
method stream(statement: Executable, parameters: _CoreAnyExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → AsyncIterator[AsyncResult[Any]]
```

执行语句并返回一个等待可迭代的`AsyncResult`对象。

例如：

```py
result = await conn.stream(stmt):
async for row in result:
    print(f"{row}")
```

`AsyncConnection.stream()`方法支持可选的上下文管理器用法，针对`AsyncResult`对象，如下所示：

```py
async with conn.stream(stmt) as result:
    async for row in result:
        print(f"{row}")
```

在上述模式中，`AsyncResult.close()`方法无条件调用，即使迭代器被异常中断。但上下文管理器的使用仍然是可选的，函数可以以`async with fn():`或`await fn()`的方式调用。

新版本 2.0.0b3 中：添加了上下文管理器支持

返回：

一个可等待对象，将产生一个`AsyncResult`对象。

另请参阅

`AsyncConnection.stream_scalars()`

```py
method stream_scalars(statement: Executable, parameters: _CoreSingleExecuteParams | None = None, *, execution_options: CoreExecuteOptionsParameter | None = None) → AsyncIterator[AsyncScalarResult[Any]]
```

执行语句并返回一个可等待对象，产生一个`AsyncScalarResult`对象。

例如：

```py
result = await conn.stream_scalars(stmt)
async for scalar in result:
    print(f"{scalar}")
```

此方法是在调用`Connection.stream()`方法后调用`AsyncResult.scalars()`方法的简写。参数是等效的。

`AsyncConnection.stream_scalars()`方法支持可选的上下文管理器使用，针对`AsyncScalarResult`对象，如下所示：

```py
async with conn.stream_scalars(stmt) as result:
    async for scalar in result:
        print(f"{scalar}")
```

在上述模式中，`AsyncScalarResult.close()`方法无条件调用，即使迭代器被异常中断。但上下文管理器的使用仍然是可选的，函数可以以`async with fn():`或`await fn()`的方式调用。

新版本 2.0.0b3 中：添加了上下文管理器支持

返回：

一个可等待��象，将产生一个`AsyncScalarResult`对象。

新版本 1.4.24 中。

另请参阅

`AsyncConnection.stream()`

```py
attribute sync_connection: Connection | None
```

引用与其关联的同步式`Connection`的`AsyncConnection`。

此实例可用作事件目标。

另请参阅

使用 asyncio 扩展处理事件

```py
attribute sync_engine: Engine
```

引用与其关联的同步式`Engine`的`AsyncConnection`。

此实例可用作事件目标。

另请参阅

使用 asyncio 扩展处理事件

```py
class sqlalchemy.ext.asyncio.AsyncTransaction
```

一个 `Transaction` 的 asyncio 代理。

**成员**

close(), commit(), rollback(), start()

**类签名**

类 `sqlalchemy.ext.asyncio.AsyncTransaction` (`sqlalchemy.ext.asyncio.base.ProxyComparable`, `sqlalchemy.ext.asyncio.base.StartableContext`)

```py
method async close() → None
```

关闭此 `AsyncTransaction`。

如果此事务是嵌套在 begin/commit 中的基本事务，则事务将回滚。否则，该方法返回。

用于取消事务而不影响封闭事务范围。

```py
method async commit() → None
```

提交此 `AsyncTransaction`。

```py
method async rollback() → None
```

回滚此 `AsyncTransaction`。

```py
method async start(is_ctxmanager: bool = False) → AsyncTransaction
```

在不使用 Python `with:` 块的情况下启动此 `AsyncTransaction` 对象的上下文。

## 结果集 API 文档

`AsyncResult` 对象是 `Result` 对象的异步适配版本。仅在使用 `AsyncConnection.stream()` 或 `AsyncSession.stream()` 方法时返回，该方法返回一个位于活动数据库游标之上的结果对象。

| 对象名称 | 描述 |
| --- | --- |
| AsyncMappingResult | 用于返回字典值而不是 `Row` 值的 `AsyncResult` 的包装器。 |
| AsyncResult | 一个围绕 `Result` 对象的 asyncio 包装器。 |
| AsyncScalarResult | 用于返回标量值而不是 `Row` 值的 `AsyncResult` 的包装器。 |
| AsyncTupleResult | 一个被类型化为返回普通 Python 元组而不是行的`AsyncResult`。 |

```py
class sqlalchemy.ext.asyncio.AsyncResult
```

一个围绕`Result`对象的 asyncio 包装��。

`AsyncResult`仅适用于使用服务器端游标的语句执行。它仅从`AsyncConnection.stream()`和`AsyncSession.stream()`方法返回。

注意

与`Result`相同，此对象用于由`AsyncSession.execute()`返回的 ORM 结果，可以单独返回 ORM 映射对象的实例或在类似元组的行中返回。请注意，这些结果对象不会像传统的`Query`对象一样自动去重实例或行。要在 Python 中去重实例或行，请使用`AsyncResult.unique()`修改器方法。

版本 1.4 中的新功能。

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), freeze(), keys(), mappings(), one(), one_or_none(), partitions(), scalar(), scalar_one(), scalar_one_or_none(), scalars(), t, tuples(), unique(), yield_per()

**类签名**

类`sqlalchemy.ext.asyncio.AsyncResult`（`sqlalchemy.engine._WithKeys`，`sqlalchemy.ext.asyncio.AsyncCommon`）

```py
method async all() → Sequence[Row[_TP]]
```

返回所有行的列表。

在调用后关闭结果集。后续调用将返回空列表。

返回：

一列`Row`对象的列表。

```py
method async close() → None
```

*继承自* `AsyncCommon.close()` *方法的* `AsyncCommon`

关闭此结果。

```py
attribute closed
```

*继承自* `AsyncCommon.closed` *属性的* `AsyncCommon`

代理底层结果对象的`.closed`属性，如果有的话，否则引发`AttributeError`。

2.0.0b3 版本中的新功能。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定每行应返回的列。

有关完整的行为描述，请参阅同步 SQLAlchemy API 中的`Result.columns()`。

```py
method async fetchall() → Sequence[Row[_TP]]
```

`AsyncResult.all()`方法的同义词。

2.0 版本中的新功能。

```py
method async fetchmany(size: int | None = None) → Sequence[Row[_TP]]
```

获取多行。

当所有行都用尽时，返回空列表。

该方法是为了与 SQLAlchemy 1.x.x 向后兼容而提供的。

要以分组方式获取行，请使用`AsyncResult.partitions()`方法。

返回：

一列`Row`对象的列表。

另请参阅

`AsyncResult.partitions()`

```py
method async fetchone() → Row[_TP] | None
```

获取一行。

当所有行都用尽时，返回`None`。

该方法是为了与 SQLAlchemy 1.x.x 向后兼容而提供的。

仅获取结果的第一行，请使用`AsyncResult.first()`方法。要遍历所有行，请直接迭代`AsyncResult`对象。

返回：

如果未应用任何过滤器，则为`Row`对象，如果没有剩余行则为`None`。

```py
method async first() → Row[_TP] | None
```

获取第一行或如果不存在行则获取`None`。

关闭结果集并丢弃剩余行。

注意

默认情况下，此方法返回一个**行**，例如元组。要返回确切的单个标量值，即第一行的第一列，请使用`AsyncResult.scalar()`方法，或者结合`AsyncResult.scalars()`和`AsyncResult.first()`。

此外，与传统 ORM `Query.first()` 方法的行为相反，对于调用以产生此 `AsyncResult` 的 SQL 查询不会应用任何限制；对于在生成行之前在内存中缓冲结果的 DBAPI 驱动程序，所有行都将发送到 Python 进程，除了第一行之外的所有行都将被丢弃。

另请参阅

ORM Query Unified with Core Select

返回：

`Row` 对象，如果没有剩余行则为 None。

另请参阅

`AsyncResult.scalar()`

`AsyncResult.one()`

```py
method async freeze() → FrozenResult[_TP]
```

返回一个可调用对象，当调用时将产生此`AsyncResult`的副本。

返回的可调用对象是`FrozenResult`的一个实例。

这用于结果集缓存。 当结果未被消耗时必须调用该方法，并且调用该方法将完全消耗结果。 当从缓存中检索到`FrozenResult`时，可以任意多次调用它，每次都会针对其存储的行集产生一个新的`Result`对象。

另请参阅

重新执行语句 - 在 ORM 中实现结果集缓存的示例用法。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys` *的* `sqlalchemy.engine._WithKeys.keys` *方法*

返回一个可迭代的视图，该视图生成每个`Row`表示的字符串键。

这些键可以表示核心语句返回的列的标签，或者 orm 执行返回的 orm 类的名称。

还可以使用 Python `in` 运算符测试视图是否包含键，该运算符将测试视图中表示的字符串键，以及列对象等替代键。

从版本 1.4 开始更改：返回的是一个键视图对象，而不是一个普通列表。

```py
method mappings() → AsyncMappingResult
```

对返回的行应用映射过滤器，返回 `AsyncMappingResult` 的一个实例。

应用此过滤器时，获取行将返回 `RowMapping` 对象而不是 `Row` 对象。

返回：

指向底层 `Result` 对象的新 `AsyncMappingResult` 过滤对象。

```py
method async one() → Row[_TP]
```

返回确切的一行或引发异常。

如果结果没有行则引发 `NoResultFound`，如果返回多行则引发 `MultipleResultsFound`。

注意

此方法默认返回一个**行**，例如元组。要返回确切的一个标量值，即第一行的第一列，请使用 `AsyncResult.scalar_one()` 方法，或者组合 `AsyncResult.scalars()` 和 `AsyncResult.one()`。

版本 1.4 中的新功能。

返回：

第一个`Row`。

引发：

`MultipleResultsFound`，`NoResultFound`

另请参阅

`AsyncResult.first()`

`AsyncResult.one_or_none()`

`AsyncResult.scalar_one()`

```py
method async one_or_none() → Row[_TP] | None
```

返回至多一个结果或引发异常。

如果结果没有行则返回 `None`。如果返回多行则引发 `MultipleResultsFound`。

版本 1.4 中的新功能。

返回：

第一个`Row`或如果没有行可用则为`None`。

引发：

`MultipleResultsFound`

另请参阅

`AsyncResult.first()`

`AsyncResult.one()`

```py
method async partitions(size: int | None = None) → AsyncIterator[Sequence[Row[_TP]]]
```

遍历给定大小的行子列表。

返回一个异步迭代器：

```py
async def scroll_results(connection):
    result = await connection.stream(select(users_table))

    async for partition in result.partitions(100):
        print("list of rows: %s" % partition)
```

参考完整的行为描述中的`Result.partitions()`同步 SQLAlchemy API。

```py
method async scalar() → Any
```

获取第一行的第一列，并关闭结果集。

如果没有要获取的行，则返回`None`。

不执行任何验证来测试是否有额外的行剩余。

调用此方法后，对象已完全关闭，例如已调用`CursorResult.close()`方法。

返回：

一个 Python 标量值，如果没有剩余行，则为`None`。

```py
method async scalar_one() → Any
```

返回确切的标量结果或引发异常。

这相当于调用`AsyncResult.scalars()`然后调用`AsyncResult.one()`。

另请参阅

`AsyncResult.one()`

`AsyncResult.scalars()`

```py
method async scalar_one_or_none() → Any | None
```

返回确切的标量结果或`None`。

这相当于调用`AsyncResult.scalars()`然后调用`AsyncResult.one_or_none()`。

另请参阅

`AsyncResult.one_or_none()`

`AsyncResult.scalars()`

```py
method scalars(index: _KeyIndexType = 0) → AsyncScalarResult[Any]
```

返回一个`AsyncScalarResult`过滤对象，该对象将返回单个元素而不是`Row`对象。

参考完整的行为描述中的`Result.scalars()`同步 SQLAlchemy API。

参数：

**index** – 指示要从每行中提取的列的整数或行键，默认为`0`，表示第一列。

返回：

一个新的`AsyncScalarResult`过滤对象，指的是此`AsyncResult`对象。

```py
attribute t
```

对返回的行应用“typed tuple”类型过滤器。

`AsyncResult.t` 属性是调用 `AsyncResult.tuples()` 方法的同义词。

新版本 2.0。

```py
method tuples() → AsyncTupleResult[_TP]
```

对返回的行应用“类型化元组”类型过滤器。

此方法在运行时返回相同的 `AsyncResult` 对象，但注释为返回一个 `AsyncTupleResult` 对象，该对象将指示给 [**PEP 484**](https://peps.python.org/pep-0484/) 类型工具以提示普通的类型化 `Tuple` 实例而不是行。这允许对 `Row` 对象进行元组解包和 `__getitem__` 访问进行类型化，对于语句本身包含了类型信息的情况。

新版本 2.0。

返回：

在编写时为 `AsyncTupleResult` 类型。

另请参阅

`AsyncResult.t` - 更短的同义词

`Row.t` - `Row` 版本

```py
method unique(strategy: _UniqueFilterType | None = None) → Self
```

对由此 `AsyncResult` 返回的对象应用唯一过滤。

请参阅同步 SQLAlchemy API 中的 `Result.unique()`，获取完整的行为描述。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

配置行提取策略，一次提取 `num` 行。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的透传。请参阅该方法的文档以获取用法说明。

新版本 1.4.40 中新增：- 添加了 `FilterResult.yield_per()` 方法，使该方法在所有结果集实现中都可用。

另请参阅

使用服务器端游标（即流式结果） - 描述 `Result.yield_per()` 的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.ext.asyncio.AsyncScalarResult
```

用于返回标量值而不是`Row`值的`AsyncResult`的包装器。

通过调用`AsyncResult.scalars()`方法获取`AsyncScalarResult`对象。

参考同步 SQLAlchemy API 中的`ScalarResult`对象以获取完整的行为描述。

在版本 1.4 中新增。

**成员**

all(), close(), closed, fetchall(), fetchmany(), first(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类`sqlalchemy.ext.asyncio.AsyncScalarResult`（`sqlalchemy.ext.asyncio.AsyncCommon`）

```py
method async all() → Sequence[_R]
```

返回列表中的所有标量值。

等同于`AsyncResult.all()`，只是返回标量值，而不是`Row`对象。

```py
method async close() → None
```

*继承自* `AsyncCommon` *的* `close()` *方法*

关闭此结果。

```py
attribute closed
```

*继承自* `AsyncCommon` *的* `closed` *属性*

代理底层结果对象的`.closed`属性，如果没有则引发`AttributeError`。

在版本 2.0.0b3 中新增。

```py
method async fetchall() → Sequence[_R]
```

`AsyncScalarResult.all()`方法的同义词。

```py
method async fetchmany(size: int | None = None) → Sequence[_R]
```

获取多个对象。

等同于`AsyncResult.fetchmany()`，只是返回标量值，而不是`Row`对象。

```py
method async first() → _R | None
```

获取第一个对象或如果没有对象则返回`None`。

等效于`AsyncResult.first()`，但返回的是标量值，而不是`Row`对象。

```py
method async one() → _R
```

返回一个对象或引发异常。

等效于`AsyncResult.one()`，但返回的是标量值，而不是`Row`对象。

```py
method async one_or_none() → _R | None
```

返回至多一个对象或引发异常。

等效于`AsyncResult.one_or_none()`，但返回的是标量值，而不是`Row`对象。

```py
method async partitions(size: int | None = None) → AsyncIterator[Sequence[_R]]
```

遍历给定大小的子元素子列表。

等效于`AsyncResult.partitions()`，但返回的是标量值，而不是`Row`对象。

```py
method unique(strategy: _UniqueFilterType | None = None) → Self
```

将唯一性过滤应用于此`AsyncScalarResult`返回的对象。

有关使用详细信息，请参阅`AsyncResult.unique()`。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

配置行提取策略，一次提取`num`行。

`FilterResult.yield_per()` 方法是对 `Result.yield_per()` 方法的传递。有关使用说明，请参阅该方法的文档。

1.4.40 版本中的新功能：- 添加了`FilterResult.yield_per()`，以便该方法在所有结果集实现上都可用。

另请参阅

使用服务器端游标（即流式结果） - 描述了`Result.yield_per()`的核心行为。

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.ext.asyncio.AsyncMappingResult
```

一个 `AsyncResult` 的包装器，返回的是字典值，而不是 `Row` 值。

调用 `AsyncResult.mappings()` 方法会获取 `AsyncMappingResult` 对象。

有关完整行为描述，请参考同步 SQLAlchemy API 中的 `MappingResult` 对象。

在版本 1.4 中引入的新功能。

**成员**

all(), close(), closed, columns(), fetchall(), fetchmany(), fetchone(), first(), keys(), one(), one_or_none(), partitions(), unique(), yield_per()

**类签名**

类 `sqlalchemy.ext.asyncio.AsyncMappingResult` (`sqlalchemy.engine._WithKeys`, `sqlalchemy.ext.asyncio.AsyncCommon`)

```py
method async all() → Sequence[RowMapping]
```

返回一个包含所有行的列表。

等同于 `AsyncResult.all()`，不同之处在于返回的是 `RowMapping` 值，而不是 `Row` 对象。

```py
method async close() → None
```

*继承自* `AsyncCommon.close()` *方法的* `AsyncCommon`

关闭此结果。

```py
attribute closed
```

*继承自* `AsyncCommon.closed` *属性的* `AsyncCommon`

代理底层结果对象的 `.closed` 属性，如果有的话，否则会引发 `AttributeError`。

新功能在版本 2.0.0b3 中引入。

```py
method columns(*col_expressions: _KeyIndexType) → Self
```

确定每行中应返回的列。

```py
method async fetchall() → Sequence[RowMapping]
```

`AsyncMappingResult.all()` 方法的同义词。

```py
method async fetchmany(size: int | None = None) → Sequence[RowMapping]
```

获取多行。

等同于`AsyncResult.fetchmany()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method async fetchone() → RowMapping | None
```

获取一个对象。

等同于`AsyncResult.fetchone()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method async first() → RowMapping | None
```

获取第一个对象或`None`（如果没有对象存在）。

等同于`AsyncResult.first()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method keys() → RMKeyView
```

*继承自* `sqlalchemy.engine._WithKeys.keys` *方法的* `sqlalchemy.engine._WithKeys`

返回一个可迭代的视图，该视图会产生每个`Row`所代表的字符串键。

这些键可以表示核心语句返回的列的标签，或者 ORM 执行返回的 ORM 类的名称。

该视图还可以使用 Python 的`in`运算符进行键包含性测试，该测试将同时测试视图中表示的字符串键，以及列对象等备用键。

从版本 1.4 开始更改：返回一个键视图对象，而不是一个普通列表。

```py
method async one() → RowMapping
```

返回确切的一个对象或引发异常。

等同于`AsyncResult.one()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method async one_or_none() → RowMapping | None
```

返回至多一个对象或引发异常。

等同于`AsyncResult.one_or_none()`，但返回的是`RowMapping`值，而不是`Row`对象。

```py
method async partitions(size: int | None = None) → AsyncIterator[Sequence[RowMapping]]
```

遍历给定大小的子列表元素。

等效于`AsyncResult.partitions()`，不同之处在于返回的是`RowMapping`值，而不是`Row`对象。

```py
method unique(strategy: _UniqueFilterType | None = None) → Self
```

对由此`AsyncMappingResult`返回的对象应用唯一过滤。

查看`AsyncResult.unique()`以获取使用详情。

```py
method yield_per(num: int) → Self
```

*继承自* `FilterResult.yield_per()` *方法的* `FilterResult`

配置行提取策略，一次提取`num`行。

`FilterResult.yield_per()` 方法是对`Result.yield_per()` 方法的一个传递。查看该方法的文档以获取使用说明。

新版本 1.4.40 中新增：- 添加`FilterResult.yield_per()`，以便该方法在所有结果集实现上都可用

另请参阅

使用服务器端游标（也称为流式结果） - 描述了`Result.yield_per()`的核心行为

使用 Yield Per 获取大型结果集 - 在 ORM 查询指南中

```py
class sqlalchemy.ext.asyncio.AsyncTupleResult
```

一个`AsyncResult`，其类型为返回普通的 Python 元组而不是行。

由于`Row`在所有方面都像一个元组，因此这个类只是一个类型类，运行时仍然使用常规的`AsyncResult`。

**类签名**

类`sqlalchemy.ext.asyncio.AsyncTupleResult`（`sqlalchemy.ext.asyncio.AsyncCommon`，`sqlalchemy.util.langhelpers.TypingOnly`）

## ORM 会话 API 文档

| 对象名称 | 描述 |
| --- | --- |
| async_object_session(实例) | 返回给定实例所属的`AsyncSession`。 |
| async_scoped_session | 提供`AsyncSession`对象的作用域管理。 |
| async_session(会话) | 返回代理给定`Session`对象的`AsyncSession`，如果有的话。 |
| async_sessionmaker | 可配置的`AsyncSession`工厂。 |
| 异步属性 | 提供所有属性的可等待访问器的混合类。 |
| AsyncSession | `Session`的 Asyncio 版本。 |
| AsyncSessionTransaction | ORM `SessionTransaction`对象的包装器。 |
| close_all_sessions() | 关闭所有`AsyncSession`会话。 |

```py
function sqlalchemy.ext.asyncio.async_object_session(instance: object) → AsyncSession | None
```

返回给定实例所属的`AsyncSession`。

此函数利用同步 API 函数`object_session`来检索引用给定实例的`Session`，然后将其链接到原始的`AsyncSession`。

如果`AsyncSession`已被垃圾回收，返回值为`None`。

此功能也可以从`InstanceState.async_session`访问器中使用。

参数：

**实例** – 一个 ORM 映射实例

返回：

一个`AsyncSession`对象，或`None`。

版本 1.4.18 中新增。

```py
function sqlalchemy.ext.asyncio.async_session(session: Session) → AsyncSession | None
```

返回代理给定`Session`对象的`AsyncSession`，如果有的话。

参数：

**会话** – 一个`Session`实例。

返回：

一个`AsyncSession`实例，或`None`。

版本 1.4.18 中的新功能。

```py
function async sqlalchemy.ext.asyncio.close_all_sessions() → None
```

关闭所有`AsyncSession`会话。

版本 2.0.23 中的新功能。

另请参阅

`close_all_sessions()`

```py
class sqlalchemy.ext.asyncio.async_sessionmaker
```

一个可配置的`AsyncSession`工厂。

`async_sessionmaker`工厂的工作方式与`sessionmaker`工厂相同，当调用时生成新的`AsyncSession`对象，根据此处建立的配置参数创建它们。

例如：

```py
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.ext.asyncio import async_sessionmaker

async def run_some_sql(async_session: async_sessionmaker[AsyncSession]) -> None:
    async with async_session() as session:
        session.add(SomeObject(data="object"))
        session.add(SomeOtherObject(name="other object"))
        await session.commit()

async def main() -> None:
    # an AsyncEngine, which the AsyncSession will use for connection
    # resources
    engine = create_async_engine('postgresql+asyncpg://scott:tiger@localhost/')

    # create a reusable factory for new AsyncSession instances
    async_session = async_sessionmaker(engine)

    await run_some_sql(async_session)

    await engine.dispose()
```

`async_sessionmaker`很有用，因此程序的不同部分可以使用预先建立的固定配置创建新的`AsyncSession`对象。请注意，当不使用`async_sessionmaker`时，也可以直接实例化`AsyncSession`对象。

版本 2.0 中的新功能：`async_sessionmaker`提供了一个专门用于`AsyncSession`对象的`sessionmaker`类，包括 pep-484 类型支持。

另请参阅

概要 - ORM - 显示示例用法

`sessionmaker` - 一般概述

`sessionmaker`架构

打开和关闭会话 - 创建会话的入门文本，使用`sessionmaker`。

**成员**

__call__(), __init__(), begin(), configure()

**类签名**

类`sqlalchemy.ext.asyncio.async_sessionmaker` (`typing.Generic`)

```py
method __call__(**local_kw: Any) → _AS
```

使用此`async_sessionmaker`中建立的配置生成一个新的`AsyncSession`对象。

在 Python 中，当对象“被调用”时，将调用`__call__`方法，方式与函数相同：

```py
AsyncSession = async_sessionmaker(async_engine, expire_on_commit=False)
session = AsyncSession()  # invokes sessionmaker.__call__()
```

```py
method __init__(bind: Optional[_AsyncSessionBind] = None, *, class_: Type[_AS] = <class 'sqlalchemy.ext.asyncio.session.AsyncSession'>, autoflush: bool = True, expire_on_commit: bool = True, info: Optional[_InfoType] = None, **kw: Any)
```

构建一个新的`async_sessionmaker`。

这里的所有参数（除了`class_`）都对应于`Session`直接接受的参数。有关参数的更多详细信息，请参阅`AsyncSession.__init__()`文档字符串。

```py
method begin() → _AsyncSessionContextManager[_AS]
```

生成一个上下文管理器，既提供一个新的`AsyncSession`，又提供一个提交事务。

例如：

```py
async def main():
    Session = async_sessionmaker(some_engine)

    async with Session.begin() as session:
        session.add(some_object)

    # commits transaction, closes session
```

```py
method configure(**new_kw: Any) → None
```

(Re)配置此`async_sessionmaker`的参数。

例如：

```py
AsyncSession = async_sessionmaker(some_engine)

AsyncSession.configure(bind=create_async_engine('sqlite+aiosqlite://'))
```

```py
class sqlalchemy.ext.asyncio.async_scoped_session
```

提供对`AsyncSession`对象的作用域管理。

查看使用 asyncio scoped session 部分以获取详细用法。

自版本 1.4.19 起新增。

**成员**

__call__(), __init__(), aclose(), add(), add_all(), autoflush, begin(), begin_nested(), bind, close(), close_all(), commit(), configure(), connection(), delete(), deleted, dirty, execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_one(), identity_key(), identity_map, info, invalidate(), is_active, is_modified(), merge(), new, no_autoflush, object_session(), refresh(), remove(), reset(), rollback(), scalar(), scalars(), session_factory, stream(), stream_scalars()

**类签名**

类`sqlalchemy.ext.asyncio.async_scoped_session`（`typing.Generic`）

```py
method __call__(**kw: Any) → _AS
```

返回当前的`AsyncSession`，如果不存在则使用`scoped_session.session_factory`创建它。

参数：

****kw** – 如果不存在现有的`AsyncSession`，关键字参数将传递给`scoped_session.session_factory`可调用对象。如果存在`AsyncSession`并且传递了关键字参数，则会引发`InvalidRequestError`。

```py
method __init__(session_factory: async_sessionmaker[_AS], scopefunc: Callable[[], Any])
```

构建一个新的`async_scoped_session`。

参数：

+   `session_factory` – 用于创建新的`AsyncSession`实例的工厂。通常情况下，但不一定，是`async_sessionmaker`的一个实例。

+   `scopefunc` – 定义当前范围的函数。例如`asyncio.current_task`可能在这里有用。

```py
method async aclose() → None
```

一个`AsyncSession.close()`的同义词。

代表`async_scoped_session`类的`AsyncSession`类。

`AsyncSession.aclose()`名称专门用于支持 Python 标准库的`@contextlib.aclosing`上下文管理器函数。

在版本 2.0.20 中新增。

```py
method add(instance: object, _warn: bool = True) → None
```

将一个对象放入这个`Session`。

代表`async_scoped_session`类的`AsyncSession`类。

代表`AsyncSession`类的`Session`类。

当传递给`Session.add()`方法时处于瞬态状态的对象将移动到挂起状态，直到下一次刷新，在此时它们将移动到持久状态。

当传递给`Session.add()`方法时处于分离状态的对象将直接移动到持久状态。

如果由`Session`使用的事务被回滚，则传递给`Session.add()`时处于瞬态的对象将被移回瞬态状态，并且不再存在于此`Session`中。

另见

`Session.add_all()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集合添加到这个`Session`中。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

代表`Session`类的代理，代表`AsyncSession`类。

参见`Session.add()`的文档以获取一般行为描述。

另见

`Session.add()`

添加新项目或现有项目 - 在使用会话的基础知识

```py
attribute autoflush
```

代表`AsyncSession`类的`Session.autoflush`属性的代理。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

```py
method begin() → AsyncSessionTransaction
```

返回一个`AsyncSessionTransaction`对象。

代理`async_scoped_session`类的`AsyncSession`类。

当进入`AsyncSessionTransaction`对象时，底层的`Session`将执行“begin”操作：

```py
async with async_session.begin():
    # .. ORM transaction is begun
```

请注意，当会话级事务开始时，通常不会发生数据库 IO，因为数据库事务是按需开始的。但是，`begin`块是异步的，以适应可能执行 IO 的`SessionEvents.after_transaction_create()`事件钩子。

有关 ORM 开始的一般描述，请参阅`Session.begin()`。

```py
method begin_nested() → AsyncSessionTransaction
```

返回一个将开始“嵌套”事务（例如 SAVEPOINT）的`AsyncSessionTransaction`对象。

代理`async_scoped_session`类的`AsyncSession`类。

行为与`AsyncSession.begin()`相同。

有关 ORM 嵌套开始的一般描述，请参阅`Session.begin_nested()`。

另请参阅

可序列化隔离/保存点/事务 DDL（asyncio 版本） - 在 SQLite asyncio 驱动程序中为了使 SAVEPOINT 正常工作而需要特殊的解决方法。

```py
attribute bind
```

代理`async_scoped_session`类的`AsyncSession.bind`属性。

```py
method async close() → None
```

关闭此`AsyncSession`使用的事务资源和 ORM 对象。

代理`async_scoped_session`类的`AsyncSession`类。

另请参阅

`Session.close()` - “close”的主要文档

关闭 - 关于 `AsyncSession.close()` 和 `AsyncSession.reset()` 语义的详细信息。

```py
async classmethod close_all() → None
```

关闭所有 `AsyncSession` 会话。

代表 `async_scoped_session` 类，为 `AsyncSession` 类做代理。

自版本 2.0 弃用：`AsyncSession.close_all()` 方法已弃用，并将在将来的版本中删除。请参考 `close_all_sessions()`。

```py
method async commit() → None
```

提交当前进行中的事务。

代表 `async_scoped_session` 类，为 `AsyncSession` 类做代理。

另请参阅

`Session.commit()` - “commit” 的主要文档

```py
method configure(**kwargs: Any) → None
```

重新配置由此 `scoped_session` 使用的 `sessionmaker`。

参见 `sessionmaker.configure()`。

```py
method async connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None, **kw: Any) → AsyncConnection
```

返回与此 `Session` 对象的事务状态对应的 `AsyncConnection` 对象。

代表 `async_scoped_session` 类，为 `AsyncSession` 类做代理。

此方法还可用于为当前事务所使用的数据库连接建立执行选项。

新功能，版本 1.4.24：添加了传递给底层 `Session.connection()` 方法的 **kw 参数。

另请参阅

`Session.connection()` - “connection” 的主要文档

```py
method async delete(instance: object) → None
```

将一个实例标记为已删除。

代表`AsyncSession`类的`async_scoped_session`类的代理。

数据库删除操作发生在`flush()`时。

由于此操作可能需要沿着未加载的关系级联，因此它是可等待的，以允许进行这些查询。

另请参阅

`Session.delete()` - 删除的主要文档

```py
attribute deleted
```

在此`Session`中标记为“已删除”的所有实例的集合。

代表`AsyncSession`类的`async_scoped_session`类的代理。

代表`Session`类的`AsyncSession`类的代理。

```py
attribute dirty
```

被认为是脏的所有持久实例的集合。

代表`AsyncSession`类的`async_scoped_session`类的代理。

代表`Session`类的`AsyncSession`类的代理。

例如：

```py
some_mapped_object in session.dirty
```

当实例被修改但未被删除时，被视为脏。

请注意，此“脏”计算是“乐观的”；大多数属性设置或集合修改操作都会将实例标记为“脏”，并将其放入此集合中，即使属性的值没有净变化。在 flush 时，将每个属性的值与其先前保存的值进行比较，如果没有净变化，将不会发生 SQL 操作（这是一种更昂贵的操作，因此仅在 flush 时执行）。

要检查实例的属性是否有可操作的净变化，请使用`Session.is_modified()`方法。

```py
method async execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Result[Any]
```

执行语句并返回一个缓冲的`Result`对象。

代表`AsyncSession`类的`async_scoped_session`类的代理。

另请参阅

`Session.execute()` - 执行的主要文档

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

使实例的属性过期。

代表`async_scoped_session`类的代理，代表`AsyncSession`类。

代表`AsyncSession`类的`Session`类的代理。

将实例的属性标记为过时。下次访问过期属性时，将向`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的值相同的值，而不管该事务之外的数据库状态发生的变化。

要同时使`Session`中的所有对象过期，请使用`Session.expire_all()`。

当调用`Session.rollback()`或`Session.commit()`方法时，`Session`对象的默认行为是使所有状态过期，以便为新事务加载新状态。因此，仅在当前事务中发出非 ORM SQL 语句的特定情况下调用`Session.expire()`才有意义。

参数：

+   `instance` – 要刷新的实例。

+   `attribute_names` – 可选的字符串属性名称列表，指示要过期的属性子集。

另请参阅

刷新/过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使此 Session 中的所有持久实例过期。

代表`async_scoped_session`类的代理，代表`AsyncSession`类。

代表`AsyncSession`类的代理，代表`Session`类。

当持久实例上的任何属性下次被访问时，将使用`Session`对象的当前事务上下文发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与之前在同一事务中读取的相同值，而不管该事务之外的数据库状态的变化。

要使单个对象和这些对象上的单个属性过期，请使用`Session.expire()`。

`Session`对象的默认行为是在调用`Session.rollback()`或`Session.commit()`方法时使所有状态过期，以便为新事务加载新状态。因此，通常不需要调用`Session.expire_all()`，假设事务是隔离的。

另请参阅

刷新/过期 - 入门材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从此`Session`中删除实例。

代表`AsyncSession`类的代理类，代表`async_scoped_session`类。

代表`Session`类的代理类，代表`AsyncSession`类。

这将释放对实例的所有内部引用。将根据*expunge*级联规则应用级联。

```py
method expunge_all() → None
```

从此`Session`中删除所有对象实例。

代表`AsyncSession`类的代理类，代表`async_scoped_session`类。

代表`Session`类的代理类，代表`AsyncSession`类。

这相当于在此`Session`中的所有对象上调用`expunge(obj)`。

```py
method async flush(objects: Sequence[Any] | None = None) → None
```

将所有对象更改刷新到数据库。

代表`AsyncSession`类的代理类，代表`async_scoped_session`类。

另请参阅

`Session.flush()` - flush 的主要文档

```py
method async get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O | None
```

根据给定的主键标识符返回一个实例，如果找不到则返回`None`。

代表`async_scoped_session`类的`AsyncSession`类的代理。

另请参阅

`Session.get()` - get 的主要文档

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, clause: ClauseElement | None = None, bind: _SessionBind | None = None, **kw: Any) → Engine | Connection
```

返回一个“绑定”，将同步代理的`Session`绑定到其中。

代表`async_scoped_session`类的`AsyncSession`类的代理。

与`Session.get_bind()`方法不同，此方法目前**不**以任何方式被`AsyncSession`使用，以解析请求的引擎。

注意

此方法直接代理到`Session.get_bind()`方法，但目前**不**作为覆盖目标有用，与`Session.get_bind()`方法相比。下面的示例说明了如何实现与`AsyncSession`和`AsyncEngine`配合使用的自定义`Session.get_bind()`方案。

在自定义垂直分区中介绍的模式说明了如何将自定义绑定查找方案应用于给定一组`Engine`对象的`Session`。要为与`AsyncSession`和`AsyncEngine`对象一起使用的`Session.get_bind()`实现，继续对`Session`进行子类化，并使用`AsyncSession.sync_session_class`将其应用于`AsyncSession`。内部方法必须继续返回`Engine`实例，可以从`AsyncEngine`使用`AsyncEngine.sync_engine`属性获取：

```py
# using example from "Custom Vertical Partitioning"

import random

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.orm import Session

# construct async engines w/ async drivers
engines = {
    'leader':create_async_engine("sqlite+aiosqlite:///leader.db"),
    'other':create_async_engine("sqlite+aiosqlite:///other.db"),
    'follower1':create_async_engine("sqlite+aiosqlite:///follower1.db"),
    'follower2':create_async_engine("sqlite+aiosqlite:///follower2.db"),
}

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None, **kw):
        # within get_bind(), return sync engines
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines['other'].sync_engine
        elif self._flushing or isinstance(clause, (Update, Delete)):
            return engines['leader'].sync_engine
        else:
            return engines[
                random.choice(['follower1','follower2'])
            ].sync_engine

# apply to AsyncSession using sync_session_class
AsyncSessionMaker = async_sessionmaker(
    sync_session_class=RoutingSession
)
```

`Session.get_bind()`方法在非异步、隐式非阻塞的上下文中调用，方式与 ORM 事件钩子和通过`AsyncSession.run_sync()`调用的函数相同，因此希望在`Session.get_bind()`内运行 SQL 命令的例程可以继续使用阻塞式代码，这将在调用数据库驱动程序的 IO 时转换为隐式异步调用。

```py
method async get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O
```

根据给定的主键标识符返回一个实例，如果未找到则引发异常。

代理`async_scoped_session`类的`AsyncSession`类。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`异常。

..versionadded: 2.0.22

另请参阅

`Session.get_one()` - get_one 的主要文档

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

返���一个标识键。

代理`AsyncSession`类，代表`async_scoped_session`类。

代理`Session`类，代表`AsyncSession`类。

这是`identity_key()`的别名。

```py
attribute identity_map
```

代理`Session.identity_map`属性，代表`AsyncSession`类。

代理`AsyncSession`类，代表`async_scoped_session`类。

```py
attribute info
```

用户可修改的字典。

代理`AsyncSession`类，代表`async_scoped_session`类。

代理`Session`类，代表`AsyncSession`类。

可以使用`info`参数来填充此字典的初始值，该参数可以传递给`Session`构造函数或`sessionmaker`构造函数或工厂方法。此处的字典始终局限于此`Session`，并且可以独立于所有其他`Session`对象进行修改。

```py
method async invalidate() → None
```

使用连接失效关闭此 Session。

代理`AsyncSession`类，代表`async_scoped_session`类。

完整描述，请参阅`Session.invalidate()`。

```py
attribute is_active
```

如果此`Session`不处于“部分回滚”状态，则为 True。

代理`AsyncSession`类，代表`async_scoped_session`类。

代理`Session`类，代表`AsyncSession`类。

版本 1.4 中更改：`Session`不再立即开始新事务，因此在首次实例化`Session`时，此属性将为 False。

“部分回滚”状态通常表示`Session`的刷新过程失败，必须发出`Session.rollback()`方法以完全回滚事务。

如果此`Session`根本不在事务中，则首次使用时`Session`将自动开始，因此在这种情况下，`Session.is_active`将返回 True。

否则，如果此`Session`在事务中，并且该事务在内部未回滚，则`Session.is_active`也将返回 True。

另请参阅

“此会话的事务由于在刷新期间发生的先前异常而回滚。”（或类似）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

如果给定的实例具有本地修改的属性，则返回`True`。

代理`AsyncSession`类，代表`async_scoped_session`类。

代理`Session`类，代表`AsyncSession`类。

此方法检索实例上每个受检属性的历史记录，并将当前值与先前提交的值进行比较（如果有）。

这实际上是检查`Session.dirty` 集合中是否存在给定实例的更昂贵和准确的版本；对每个属性的净“脏”状态进行全面测试。

例如：

```py
return session.is_modified(someobject)
```

此方法有一些注意事项：

+   在 `Session.dirty` 集合中存在的实例在使用此方法进行测试时可能会报告 `False`。这是因为对象可能已通过属性突变接收到更改事件，从而将其放入 `Session.dirty`，但最终状态与从数据库加载的状态相同，因此在此处没有净变化。

+   标量属性在新值应用时可能没有记录先前设置的值，如果属性在收到新值时未加载或已过期，则假定属性发生了更改，即使最终与数据库值没有净变化。在大多数情况下，当发生设置事件时，SQLAlchemy 不需要“旧”值，因此如果旧值不存在，则会跳过 SQL 调用的开销，这是基于标量值通常需要进行更新的假设，而在极少数情况下，与发出防御性 SELECT 相比，平均成本更低。

    仅当属性容器的 `active_history` 标志设置为 `True` 时，才无条件地在设置时获取“旧”值。此标志通常设置为主键属性和不是简单的一对多的标量对象引用。要为任意映射列设置此标志，请使用 `column_property()` 的 `active_history` 参数。

参数：

+   `instance` – 要测试是否存在待处理更改的映射实例。

+   `include_collections` – 指示是否应在操作中包含多值集合。将其设置为 `False` 是一种检测仅基于本地列的属性（即标量列或一对多外键）的方法，这将导致在刷新时对此实例进行更新。

```py
method async merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

将给定实例的状态复制到此 `AsyncSession` 中的相应实例。

代表 `async_scoped_session` 类的 `AsyncSession` 类的代理。

另请参见

`Session.merge()` - 合并的主要文档

```py
attribute new
```

在此 `Session` 中标记为“新”的所有实例的集合。

代表`async_scoped_session`类，为`AsyncSession`类提供代理。

代表`AsyncSession`类，为`Session`类提供代理。

```py
attribute no_autoflush
```

返回一个上下文管理器，用于禁用自动刷新。

代表`async_scoped_session`类，为`AsyncSession`类提供代理。

代表`AsyncSession`类，为`Session`类提供代理。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在`with:`块中进行的操作不会受到查询访问时的刷新影响。这在初始化涉及现有数据库查询的一系列对象时很有用，其中未完成的对象不应立即被刷新。

```py
classmethod object_session(instance: object) → Session | None
```

返回对象所属的`Session`。

代表`async_scoped_session`类，为`AsyncSession`类提供代理。

代表`AsyncSession`类，为`Session`类提供代理。

这是`object_session()`的别名。

```py
method async refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

使给定实例上的属性过期并刷新。

代表`async_scoped_session`类，为`AsyncSession`类提供代理。

将向数据库发出查询，并刷新所有属性以获取其当前数据库值。

这是`Session.refresh()`方法的异步版本。查看该方法以获取所有选项的完整描述。

请参阅

`Session.refresh()` - 刷新的主要文档

```py
method async remove() → None
```

处理当前的`AsyncSession`，如果存在的话。

不同于 scoped_session 的 remove 方法，此方法将使用 await 等待 AsyncSession 的 close 方法。

```py
method async reset() → None
```

关闭此 `Session` 使用的事务资源和 ORM 对象，将会将 session 重置为其初始状态。

代表 `async_scoped_session` 类的 `AsyncSession` 类的代理。

新版本 2.0.22。

请参见

`Session.reset()` - “reset” 的主要文档

关闭 - 关于 `AsyncSession.close()` 和 `AsyncSession.reset()` 语义的详细信息。

```py
method async rollback() → None
```

回滚当前进行中的事务。

代表 `async_scoped_session` 类的 `AsyncSession` 类的代理。

请参见

`Session.rollback()` - “rollback” 的主要文档

```py
method async scalar(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行语句并返回标量结果。

代表 `async_scoped_session` 类的 `AsyncSession` 类的代理。

请参见

`Session.scalar()` - scalar 的主要文档

```py
method async scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行语句并返回标量结果。

代表 `async_scoped_session` 类的 `AsyncSession` 类的代理。

返回：

一个 `ScalarResult` 对象

新版本 1.4.24 中新增了 `AsyncSession.scalars()`

新版本 1.4.26 中新增了 `async_scoped_session.scalars()`

请参见

`Session.scalars()` - scalars 的主要文档

`AsyncSession.stream_scalars()` - 流式版本

```py
attribute session_factory: async_sessionmaker[_AS]
```

提供给`__init__`的 session_factory 存储在此属性中，稍后可以访问。当需要新的非作用域`AsyncSession`时，这可能很有用。

```py
method async stream(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncResult[Any]
```

执行语句并返回流式`AsyncResult`对象。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

```py
method async stream_scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncScalarResult[Any]
```

执行语句并返回标量结果流。

代表`AsyncSession`类的代理，代表`async_scoped_session`类。

返回：

一个`AsyncScalarResult`对象

版本 1.4.24 中的新功能。

另请参阅

`Session.scalars()` - 标量的主要文档

`AsyncSession.scalars()` - 非流式版本

```py
class sqlalchemy.ext.asyncio.AsyncAttrs
```

提供所有属性的可等待访问器的 mixin 类。

例如：

```py
from __future__ import annotations

from typing import List

from sqlalchemy import ForeignKey
from sqlalchemy import func
from sqlalchemy.ext.asyncio import AsyncAttrs
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import relationship

class Base(AsyncAttrs, DeclarativeBase):
    pass

class A(Base):
    __tablename__ = "a"

    id: Mapped[int] = mapped_column(primary_key=True)
    data: Mapped[str]
    bs: Mapped[List[B]] = relationship()

class B(Base):
    __tablename__ = "b"
    id: Mapped[int] = mapped_column(primary_key=True)
    a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
    data: Mapped[str]
```

在上面的示例中，`AsyncAttrs` mixin 应用于声明性`Base`类，对所有子类生效。此 mixin 为所有类添加了一个新属性`AsyncAttrs.awaitable_attrs`，它将任何属性的值作为可等待对象返回。这允��访问可能受惰性加载或延迟/未过期加载影响的属性，以便仍然可以发出 IO：

```py
a1 = (await async_session.scalars(select(A).where(A.id == 5))).one()

# use the lazy loader on ``a1.bs`` via the ``.awaitable_attrs``
# interface, so that it may be awaited
for b1 in await a1.awaitable_attrs.bs:
    print(b1)
```

`AsyncAttrs.awaitable_attrs` 执行针对属性的调用，大致相当于使用`AsyncSession.run_sync()`方法，例如：

```py
for b1 in await async_session.run_sync(lambda sess: a1.bs):
    print(b1)
```

版本 2.0.13 中的新功能。

**成员**

awaitable_attrs

另请参阅

在使用 AsyncSession 时防止隐式 IO

```py
attribute awaitable_attrs
```

提供一个将此对象上的所有属性命名空间包装为可等待对象的方法。

例如：

```py
a1 = (await async_session.scalars(select(A).where(A.id == 5))).one()

some_attribute = await a1.awaitable_attrs.some_deferred_attribute
some_collection = await a1.awaitable_attrs.some_collection
```

```py
class sqlalchemy.ext.asyncio.AsyncSession
```

`Session`的 Asyncio 版本。

`AsyncSession`是传统`Session`实例的代理。

`AsyncSession` **不适合在并发任务中使用**。请参阅 Session 线程安全吗？AsyncSession 在并发任务中安全共享吗？了解背景。

版本 1.4 中的新功能。

要在自定义`Session`实现中使用`AsyncSession`，请查看`AsyncSession.sync_session_class`参数。

**成员**

sync_session_class, __init__(), aclose(), add(), add_all(), autoflush, begin(), begin_nested(), close(), close_all(), commit(), connection(), delete(), deleted, dirty, execute(), expire(), expire_all(), expunge(), expunge_all(), flush(), get(), get_bind(), get_nested_transaction(), get_one(), get_transaction(), identity_key(), identity_map, in_nested_transaction(), in_transaction(), info, invalidate(), is_active, is_modified(), merge(), new, no_autoflush, object_session(), refresh(), reset(), rollback(), run_sync(), scalar(), scalars(), stream(), stream_scalars(), sync_session

**类签名**

类`sqlalchemy.ext.asyncio.AsyncSession`（`sqlalchemy.ext.asyncio.base.ReversibleProxy`）

```py
attribute sync_session_class: Type[Session] = <class 'sqlalchemy.orm.session.Session'>
```

为特定`AsyncSession`提供底层`Session`实例的类或可调用对象。

在类级别，此属性是`AsyncSession.sync_session_class`参数的默认值。`AsyncSession`的自定义子类可以覆盖此值。

在实例级别，此属性指示当前类或可调用对象，用于为此`AsyncSession`实例提供`Session`实例。

版本 1.4.24 中的新功能。

```py
method __init__(bind: _AsyncSessionBind | None = None, *, binds: Dict[_SessionBindKey, _AsyncSessionBind] | None = None, sync_session_class: Type[Session] | None = None, **kw: Any)
```

构建一个新的`AsyncSession`。

除了`sync_session_class`之外的所有参数都直接传递给`sync_session_class`可调用对象，以实例化一个新的`Session`。请参考`Session.__init__()`获取参数文档。

参数：

**sync_session_class** –

一个`Session`子类或其他可调用对象，用于构建将被代理的`Session`。此参数可用于提供自定义的`Session`子类。默认为`AsyncSession.sync_session_class`类级属性。

版本 1.4.24 中的新功能。

```py
method async aclose() → None
```

`AsyncSession.close()`的同义词。

`AsyncSession.aclose()`名称专门支持 Python 标准库的`@contextlib.aclosing`上下文管理器函数。

版本 2.0.20 中的新功能。

```py
method add(instance: object, _warn: bool = True) → None
```

将一个对象放入此`Session`。

代理`AsyncSession`类的`Session`类。

传递给`Session.add()`方法时处于瞬态状态的对象将移动到挂起状态，直到下一次刷新，然后它们将移动到持久状态。

传递给`Session.add()`方法时处于分离状态的对象将直接移动到持久状态。

如果`Session`使用的事务被回滚，则当它们被传递给`Session.add()`时处于瞬态的对象将被移回到瞬态状态，并且将不再存在于此`Session`中。

另请参阅

`Session.add_all()`

添加新项目或现有项目 - 在使用会话基础知识

```py
method add_all(instances: Iterable[object]) → None
```

将给定的实例集合添加到这个`Session`中。

代理`AsyncSession`类的`Session`类。

有关一般行为描述，请参阅`Session.add()`的文档。

另请参阅

`Session.add()`

添加新项目或现有项目 - 在使用会话基础知识

```py
attribute autoflush
```

代理`AsyncSession`类的`Session.autoflush`属性。

```py
method begin() → AsyncSessionTransaction
```

返回一个`AsyncSessionTransaction`对象。

当进入`AsyncSessionTransaction`对象时，底层的`Session`将执行“开始”操作：

```py
async with async_session.begin():
    # .. ORM transaction is begun
```

请注意，当会话级事务开始时，通常不会发生数据库 IO，因为数据库事务在按需基础上开始。但是，开始块是异步的，以适应可能执行 IO 的`SessionEvents.after_transaction_create()`事件钩子。

有关 ORM 开始的一般描述，请参见`Session.begin()`。

```py
method begin_nested() → AsyncSessionTransaction
```

返回一个`AsyncSessionTransaction`对象，该对象将开始一个“嵌套”事务，例如 SAVEPOINT。

行为与`AsyncSession.begin()`相同。

有关 ORM 开始嵌套的一般描述，请参见`Session.begin_nested()`。

另请参见

可序列化隔离/保存点/事务性 DDL（asyncio 版本） - 针对 SQLite asyncio 驱动程序，需要特殊的解决方法才能正确使用 SAVEPOINT。

```py
method async close() → None
```

关闭此`AsyncSession`使用的事务资源和 ORM 对象。

另请参见

`Session.close()` - 关于“close”的主要文档

关闭 - 关于`AsyncSession.close()`和`AsyncSession.reset()`语义的详细信息。

```py
async classmethod close_all() → None
```

关闭所有`AsyncSession`会话。

自 2.0 版本弃用：`AsyncSession.close_all()`方法已弃用，并将在以后的版本中删除。请参考`close_all_sessions()`。

```py
method async commit() → None
```

提交当前进行中的事务。

另请参见

`Session.commit()` - 关于“commit”的主要文档

```py
method async connection(bind_arguments: _BindArguments | None = None, execution_options: CoreExecuteOptionsParameter | None = None, **kw: Any) → AsyncConnection
```

返回一个`AsyncConnection`对象，对应于此`Session`对象的事务状态。

此方法还可用于为当前事务使用的数据库连接建立执行选项。

版本 1.4.24 中的新内容：添加传递给底层`Session.connection()`方法的**kw 参数。

另请参阅

`Session.connection()` - “连接”主要文档

```py
method async delete(instance: object) → None
```

将实例标记为已删除。

数据库删除操作在`flush()`时发生。

由于此操作可能需要沿着未加载的关系级联，因此它是可等待的，以允许执行这些查询。

另请参阅

`Session.delete()` - 删除的主要文档

```py
attribute deleted
```

在此`Session`中标记为“已删除”的所有实例集合

代表`AsyncSession`类的`Session`类的代理。

```py
attribute dirty
```

被视为脏的所有持久实例集合。

代表`AsyncSession`类的`Session`类的代理。

例如：

```py
some_mapped_object in session.dirty
```

当实例被修改但未删除时，被视为脏。

请注意，此“脏”计算是“乐观”的；大多数属性设置或集合修改操作都会将实例标记为“脏”并将其放入此集合中，即使属性值没有净变化。在刷新时，将每个属性的值与其先前保存的值进行比较，如果没有净变化，则不会发生任何 SQL 操作（这是一种更昂贵的操作，因此仅在刷新时执行）。

要检查实例的属性是否具有可操作的净变化，请使用`Session.is_modified()`方法。

```py
method async execute(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Result[Any]
```

执行语句并返回缓冲的`Result`对象。

另请参阅

`Session.execute()` - 执行的主要文档

```py
method expire(instance: object, attribute_names: Iterable[str] | None = None) → None
```

使实例的属性过期。

代表`AsyncSession`类的`Session`类的代理。

标记实例的属性为过时。下次访问过期的属性时，将在`Session`对象的当前事务上下文中发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与先前在同一事务中读取的相同值，而不考虑该事务之外的数据库状态的变化。

要同时使`Session`中的所有对象过期，请使用 `Session.expire_all()`。

`Session`对象的默认行为是在调用 `Session.rollback()` 或 `Session.commit()` 方法时使所有状态过期，以便为新事务加载新状态。因此，仅在当前事务中发出了非 ORM SQL 语句的特定情况下调用 `Session.expire()` 才有意义。

参数：

+   `instance` – 要刷新的实例。

+   `attribute_names` – 表示要过期的属性子集的可选字符串属性名称列表。

另请参阅

刷新 / 过期 - 介绍性材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expire_all() → None
```

使此会话中的所有持久化实例过期。

代表`AsyncSession`类的`Session`类的代理。

下次访问持久实例的任何属性时，将在`Session`对象的当前事务上下文中发出查询，以加载给定实例的所有过期属性。请注意，高度隔离的事务将返回与先前在同一事务中读取的相同值，而不考虑该事务之外的数据库状态的变化。

要使单个对象和这些对象上的单个属性过期，请使用 `Session.expire()`。

`Session` 对象的默认行为是在调用 `Session.rollback()` 或 `Session.commit()` 方法时使所有状态过期，以便为新事务加载新状态。因此，通常情况下不需要调用 `Session.expire_all()`，假设事务是隔离的。

另请参阅

刷新/到期 - 介绍材料

`Session.expire()`

`Session.refresh()`

`Query.populate_existing()`

```py
method expunge(instance: object) → None
```

从此`Session`中删除实例。

代理给`AsyncSession`类的`Session`类。

这将释放对实例的所有内部引用。级联将根据*expunge*级联规则应用。

```py
method expunge_all() → None
```

从此`Session`中删除所有对象实例。

代理给`AsyncSession`类的`Session`类。

这相当于在此`Session`中调用`expunge(obj)`来删除所有对象。

```py
method async flush(objects: Sequence[Any] | None = None) → None
```

将所有对象更改刷新到数据库。

另请参见

`Session.flush()` - 刷新的主要文档

```py
method async get(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O | None
```

根据给定的主键标识符返回一个实例，如果未找到则返回`None`。

另请参见

`Session.get()` - 获取的主要文档

```py
method get_bind(mapper: _EntityBindKey[_O] | None = None, clause: ClauseElement | None = None, bind: _SessionBind | None = None, **kw: Any) → Engine | Connection
```

返回同步代理的“bind”，该绑定绑定到的`Session`。

与`Session.get_bind()`方法不同，此方法目前**不**以任何方式由此`AsyncSession`使用以解析请求的引擎。

注意

此方法直接代理到`Session.get_bind()`方法，但目前**不**像`Session.get_bind()`方法那样有用作为一个覆盖目标。下面的示例说明了如何实现与`AsyncSession`和`AsyncEngine`配合工作的自定义`Session.get_bind()`方案。

自定义垂直分区介绍的模式说明了如何对给定一组`Engine`对象应用自定义绑定查找方案到一个`Session`。要为`AsyncSession`和`AsyncEngine`对象应用相应的`Session.get_bind()`实现，继续对`Session`进行子类化，并使用`AsyncSession.sync_session_class`将其应用于`AsyncSession`。内部方法必须继续返回`Engine`实例，可以从`AsyncEngine`使用`AsyncEngine.sync_engine`属性获取：

```py
# using example from "Custom Vertical Partitioning"

import random

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy.ext.asyncio import async_sessionmaker
from sqlalchemy.orm import Session

# construct async engines w/ async drivers
engines = {
    'leader':create_async_engine("sqlite+aiosqlite:///leader.db"),
    'other':create_async_engine("sqlite+aiosqlite:///other.db"),
    'follower1':create_async_engine("sqlite+aiosqlite:///follower1.db"),
    'follower2':create_async_engine("sqlite+aiosqlite:///follower2.db"),
}

class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None, **kw):
        # within get_bind(), return sync engines
        if mapper and issubclass(mapper.class_, MyOtherClass):
            return engines['other'].sync_engine
        elif self._flushing or isinstance(clause, (Update, Delete)):
            return engines['leader'].sync_engine
        else:
            return engines[
                random.choice(['follower1','follower2'])
            ].sync_engine

# apply to AsyncSession using sync_session_class
AsyncSessionMaker = async_sessionmaker(
    sync_session_class=RoutingSession
)
```

`Session.get_bind()`方法在非异步、隐式非阻塞的上下文中被调用，方式与 ORM 事件钩子和通过`AsyncSession.run_sync()`调用的函数相同，因此希望在`Session.get_bind()`内运行 SQL 命令的例程可以继续使用阻塞式代码，这将在调用数据库驱动程序的 IO 点隐式转换为异步调用。

```py
method get_nested_transaction() → AsyncSessionTransaction | None
```

返回当前正在进行的嵌套事务，如果有的话。

返回：

一个`AsyncSessionTransaction`对象，或`None`。

版本 1.4.18 中的新功能。

```py
method async get_one(entity: _EntityBindKey[_O], ident: _PKIdentityArgument, *, options: Sequence[ORMOption] | None = None, populate_existing: bool = False, with_for_update: ForUpdateParameter = None, identity_token: Any | None = None, execution_options: OrmExecuteOptionsParameter = {}) → _O
```

根据给定的主键标识符返回一个实例，如果未找到则引发异常。

如果查询未选择任何行，则引发`sqlalchemy.orm.exc.NoResultFound`。

..versionadded: 2.0.22

另请参阅

`Session.get_one()` - get_one 的主要文档

```py
method get_transaction() → AsyncSessionTransaction | None
```

返回当前正在进行的根事务，如果有的话。

返回：

一个`AsyncSessionTransaction`对象，或`None`。

版本 1.4.18 中的新功能。

```py
classmethod identity_key(class_: Type[Any] | None = None, ident: Any | Tuple[Any, ...] = None, *, instance: Any | None = None, row: Row[Any] | RowMapping | None = None, identity_token: Any | None = None) → _IdentityKeyType[Any]
```

返回标识键。

代表`AsyncSession`类的`Session`类的代理。

这��`identity_key()`的别名。

```py
attribute identity_map
```

代表`AsyncSession`类的`Session.identity_map`属性的代理。

```py
method in_nested_transaction() → bool
```

如果此`Session`已开始嵌套事务（例如 SAVEPOINT），则返回 True。

代表`AsyncSession`类的`Session`类的代理。

版本 1.4 中的新功能。

```py
method in_transaction() → bool
```

如果此`Session`已开始事务，则返回 True。

代表`AsyncSession`类的`Session`类的代理。

版本 1.4 中的新功能。

另请参阅

`Session.is_active`

```py
attribute info
```

可由用户修改的字典。

代表`AsyncSession`类的`Session`类的代理。

可以使用`info`参数来填充此字典的初始值，该参数可用于`Session`构造函数或`sessionmaker`构造函数或工厂方法。此处的字典始终局限于此`Session`，并且可以独立于所有其他`Session`对象进行修改。

```py
method async invalidate() → None
```

使用连接失效关闭此 Session。

有关完整描述，请参阅`Session.invalidate()`。

```py
attribute is_active
```

如果此`Session`不处于“部分回滚”状态，则返回 True。

代表`AsyncSession`类的`Session`类的代理。

在 1.4 版本中更改：`Session`不再立即开始新事务，因此当首次实例化`Session`时，此属性将为 False。

“部分回滚”状态通常表示`Session`的刷新过程失败，并且必须发出`Session.rollback()`方法以完全回滚事务。

如果此`Session`根本不在事务中，则在首次使用时，`Session`将自动开始，因此在这种情况下，`Session.is_active`将返回 True。

否则，如果此`Session`在事务中，并且该事务尚未在内部回滚，则`Session.is_active`也将返回 True。

另请参见

“由于刷新期间的先前异常，此会话的事务已回滚。”（或类似）

`Session.in_transaction()`

```py
method is_modified(instance: object, include_collections: bool = True) → bool
```

如果给定实例具有本地修改的属性，则返回`True`。

代表`AsyncSession`类的`Session`类的代理。

此方法检索实例上每个被检测属性的历史，并将当前值与先前提交的值进行比较（如果有的话）。

这实际上是检查给定实例是否在`Session.dirty`集合中的更昂贵和准确的版本；对每个属性的净“脏”状态进行了全面测试。

例如：

```py
return session.is_modified(someobject)
```

此方法有一些注意事项：

+   在`Session.dirty`集合中存在的实例在使用此方法进行测试时可能会报告`False`。这是因为对象可能已通过属性突变接收到更改事件，从而将其放置在`Session.dirty`中，但最终状态与从数据库加载的状态相同，在这里没有净变化。

+   当新值被应用时，如果属性未加载或已过期，则标量属性可能没有记录先前设置的值 - 在这些情况下，即使最终没有对其数据库值进行净更改，也假定属性已更改。大多数情况下，当发生设置事件时，SQLAlchemy 不需要“旧”值，因此如果旧值不存在，则会跳过发出 SQL 调用的开销，这是基于标量值通常需要进行更新的假设，并且在极少数情况下，与发出防御性 SELECT 相比，平均成本更低。

    仅当属性容器的 `active_history` 标志设置为 `True` 时，才会无条件地在设置时获取“旧”值。通常为主键属性和不是简单一对多的标量对象引用设置此标志。要为任意映射列设置此标志，请使用 `column_property()` 中的 `active_history` 参数。

参数：

+   `instance` – 要测试待处理更改的映射实例。

+   `include_collections` – 指示是否应在操作中包含多值集合。将其设置为 `False` 是一种检测仅基于本地列的属性（即标量列或一对多外键）的方法，这些属性在刷新时会导致此实例进行更新。

```py
method async merge(instance: _O, *, load: bool = True, options: Sequence[ORMOption] | None = None) → _O
```

将给定实例的状态复制到此 `AsyncSession` 中的相应实例。

另请参阅

`Session.merge()` - merge 的主要文档

```py
attribute new
```

在此 `Session` 中标记为“新”的所有实例的集合。

代理 `AsyncSession` 类的 `Session` 类。

```py
attribute no_autoflush
```

返回一个禁用自动刷新的上下文管理器。

代理 `AsyncSession` 类的 `Session` 类。

例如：

```py
with session.no_autoflush:

    some_object = SomeClass()
    session.add(some_object)
    # won't autoflush
    some_object.related_thing = session.query(SomeRelated).first()
```

在 `with:` 块中进行的操作不会受到在查询访问时发生的刷新的影响。这在初始化涉及现有数据库查询的一系列对象时很有用，其中尚未完成的对象不应立即刷新。

```py
classmethod object_session(instance: object) → Session | None
```

返回对象所属的 `Session`。

代理 `AsyncSession` 类的 `Session` 类。

这是 `object_session()` 的别名。

```py
method async refresh(instance: object, attribute_names: Iterable[str] | None = None, with_for_update: ForUpdateParameter = None) → None
```

使给定实例上的属性过期并刷新。

将向数据库发出查询，并刷新所有属性以获取其当前数据库值。

这是 `Session.refresh()` 方法的异步版本。请参阅该方法以获取所有选项的完整描述。

另请参阅

`Session.refresh()` - ��新的主要文档

```py
method async reset() → None
```

关闭此 `Session` 使用的事务资源和 ORM 对象，将会重置会话到其初始状态。

新版本 2.0.22 中新增。

另请参阅

`Session.reset()` - “reset” 的主要文档

关闭 - 关于 `AsyncSession.close()` 和 `AsyncSession.reset()` 语义的详细信息。

```py
method async rollback() → None
```

回滚当前进行中的事务。

另请参阅

`Session.rollback()` - “rollback”的主要文档

```py
method async run_sync(fn: ~typing.Callable[[~typing.Concatenate[~sqlalchemy.orm.session.Session, ~_P]], ~sqlalchemy.ext.asyncio.session._T], *arg: ~typing.~_P, **kw: ~typing.~_P) → _T
```

调用给定的同步（即非异步）可调用对象，将同步风格的 `Session` 作为第一个参数传递。

该方法允许传统同步的 SQLAlchemy 函数在 asyncio 应用程序的上下文中运行。

例如：

```py
def some_business_method(session: Session, param: str) -> str:
  '''A synchronous function that does not require awaiting

 :param session: a SQLAlchemy Session, used synchronously

 :return: an optional return value is supported

 '''
    session.add(MyObject(param=param))
    session.flush()
    return "success"

async def do_something_async(async_engine: AsyncEngine) -> None:
  '''an async function that uses awaiting'''

    with AsyncSession(async_engine) as async_session:
        # run some_business_method() with a sync-style
        # Session, proxied into an awaitable
        return_code = await async_session.run_sync(some_business_method, param="param1")
        print(return_code)
```

该方法通过在特别调试的 greenlet 中运行给定的可调用对象，一直将 asyncio 事件循环传递到数据库连接。

提示

提供的可调用对象在 asyncio 事件循环中内联调用，并将在传统 IO 调用上阻塞。此可调用对象内的 IO 应该仅调用 SQLAlchemy 的 asyncio 数据库 API，这些 API 将被正确地适配到 greenlet 上下文。

另请参阅

`AsyncAttrs` - 为 ORM 映射类提供每个属性基础上更简洁的类似功能的混合类

`AsyncConnection.run_sync()`

在 asyncio 下运行同步方法和函数

```py
method async scalar(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → Any
```

执行语句并返回标量结果。

另请参阅

`Session.scalar()` - 标量的主要文档

```py
method async scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → ScalarResult[Any]
```

执行语句并返回标量结果。

返回：

一个 `ScalarResult` 对象

新版本 1.4.24 中新增 `AsyncSession.scalars()`

新版本 1.4.26 中新增 `async_scoped_session.scalars()`

另请参阅

`Session.scalars()` - 标量的主要文档

`AsyncSession.stream_scalars()` - 流式版本

```py
method async stream(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncResult[Any]
```

执行语句并返回流式 `AsyncResult` 对象。

```py
method async stream_scalars(statement: Executable, params: _CoreAnyExecuteParams | None = None, *, execution_options: OrmExecuteOptionsParameter = {}, bind_arguments: _BindArguments | None = None, **kw: Any) → AsyncScalarResult[Any]
```

执行语句并返回标量结果流。

返回:

一个 `AsyncScalarResult` 对象

新版本 1.4.24。

另请参阅

`Session.scalars()` - 标量的主要文档

`AsyncSession.scalars()` - 非流式版本

```py
attribute sync_session: Session
```

指向此 `AsyncSession` 代理请求的基础 `Session` 的引用。

此实例可用作事件目标。

另请参阅

使用 asyncio 扩展处理事件

```py
class sqlalchemy.ext.asyncio.AsyncSessionTransaction
```

用于 ORM `SessionTransaction` 对象的包装器。

提供此对象以便返回一个用于 `AsyncSession.begin()` 的事务持有对象。

该对象支持对 `AsyncSessionTransaction.commit()` 和 `AsyncSessionTransaction.rollback()` 的显式调用，以及作为异步上下文管理器使用。

新版本 1.4。

**成员**

commit(), rollback()

**类签名**

类 `sqlalchemy.ext.asyncio.AsyncSessionTransaction` (`sqlalchemy.ext.asyncio.base.ReversibleProxy`, `sqlalchemy.ext.asyncio.base.StartableContext`)

```py
method async commit() → None
```

提交此`AsyncTransaction`。

```py
method async rollback() → None
```

回滚此`AsyncTransaction`。
