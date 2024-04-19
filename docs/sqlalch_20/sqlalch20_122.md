# 错误消息

> 原文：[`docs.sqlalchemy.org/en/20/errors.html`](https://docs.sqlalchemy.org/en/20/errors.html)

本节列出了 SQLAlchemy 引发或发出的常见错误消息和警告的描述和背景。

SQLAlchemy 通常在 SQLAlchemy 特定的异常类的上下文中引发错误。有关这些类的详细信息，请参见核心异常和 ORM 异常。

SQLAlchemy 错误大致可分为两类，即**编程时错误**和**运行时错误**。编程时错误是由于函数或方法使用不正确的参数而引发的，或者来自于无法解析的其他配置方法，例如无法解析的映射器配置。编程时错误通常是即时且确定的。另一方面，运行时错误表示程序运行时响应某些随机条件发生的失败，例如数据库连接耗尽或发生某些数据相关问题。运行时错误更可能出现在正在运行的应用程序的日志中，因为程序在遇到这些状态时会对负载和遇到的数据做出响应。

由于运行时错误不容易重现，并且通常发生在程序运行时对某些任意条件的响应中，它们更难以调试，也会影响到已经投入生产的程序。

在本节中，目标是尝试提供关于一些最常见的运行时错误以及编程时错误的背景信息。

## 连接和事务

### 队列池大小 <x> 超出 <y> 达到，连接超时，超时 <z>

这可能是最常见的运行时错误，直接涉及到应用程序的工作负载超过了一个配置的限制，这个限制通常适用于几乎所有的 SQLAlchemy 应用程序。

以下要点总结了此错误的含义，从大多数 SQLAlchemy 用户应该已经熟悉的最基本的要点开始。

+   **SQLAlchemy 引擎对象默认使用一个连接池** - 这意味着当一个`Engine`对象使用一个 SQL 数据库连接资源，并且然后释放该资源时，数据库连接本身保持连接到数据库，并返回到一个内部队列，可以再次使用。即使代码似乎已经结束了与数据库的对话，在许多情况下，应用程序仍将保持一定数量的数据库连接，直到应用程序结束或池明确释放为止。

+   由于池的存在，当应用程序使用 SQL 数据库连接时，通常是从使用`Engine.connect()`或使用 ORM`Session`进行查询时，此活动不一定会在获取连接对象时立即建立到数据库的新连接；它反而会向连接池查询连接，该连接池通常会从池中检索一个现有的连接以供重用。如果没有可用连接，则池将创建一个新的数据库连接，但仅当池未超过配置的容量时。

+   在大多数情况下使用的默认池被称为`QueuePool`。当您请求此池提供连接并且没有可用连接时，它会创建一个新连接**如果当前使用的连接总数小于配置的值**。这个值等于**池大小加上最大溢出**。这意味着如果您已将引擎配置为：

    ```py
    engine = create_engine("mysql+mysqldb://u:p@host/db", pool_size=10, max_overflow=20)
    ```

    上述`Engine`将允许**最多 30 个连接**在任何时候使用，不包括从引擎分离或失效的连接。如果一个新连接的请求到达，而应用程序的其他部分已经使用了 30 个连接，连接池将在固定时间内阻塞，然后超时并引发此错误消息。

    为了允许一次使用更多的连接，可以使用传递给`create_engine()`函数的`create_engine.pool_size`和`create_engine.max_overflow`参数来调整池。等待连接可用的超时时间通过`create_engine.pool_timeout`参数进行配置。

+   通过将`create_engine.max_overflow`设置为值“-1”，可以配置池具有无限的溢出。使用此设置，池仍然会维护一组固定的连接，但如果没有可用连接，则绝对会创建一个新连接，而不会阻塞。

    然而，当以这种方式运行时，如果应用程序存在使用所有可用连接资源的问题，最终会达到数据库本身可用连接的配置限制，这将再次返回一个错误。更严重的是，当应用程序耗尽连接数据库的连接时，通常会在失败之前使用大量资源，并且还可能干扰依赖于能够连接到数据库的其他应用程序和数据库状态机制。

    鉴于上述情况，可以将连接池视为连接使用的**安全阀**，为防止恶意应用程序导致整个数据库对所有其他应用程序不可用提供了关键的保护层。在收到此错误消息时，最好修复使用过多连接的问题和/或适当配置限制，而不是允许无限溢出，因为这实际上并不能解决潜在的问题。

什么导致应用程序使用完所有可用的连接？

+   **应用程序正在处理基于池配置值的太多并发请求以执行工作** - 这是最直接的原因。如果您有一个在允许 30 个并发线程的线程池中运行的应用程序，并且每个线程使用一个连接，如果您的池未配置为允许至少同时检出 30 个连接，那么一旦您的应用程序接收到足够的并发请求，您将收到此错误。解决方案是提高池的限制或降低并发线程数。

+   **应用程序未将连接返回到池中** - 这是下一个最常见的原因，即应用程序正在使用连接池，但程序未能释放这些连接，而是将它们保持打开状态。连接池以及 ORM `Session` 确实具有逻辑，以便当会话和/或连接对象被垃圾收集时，会导致底层连接资源被释放，但是不能依赖此行为及时释放资源。

    造成这种情况的常见原因是应用程序使用 ORM 会话，但在完成涉及该会话的工作后未调用 `Session.close()`。解决方法是确保 ORM 会话（如果使用 ORM）或引擎绑定的`Connection`对象（如果使用 Core）在完成工作后明确关闭，可以通过适当的`.close()`方法或使用可用的上下文管理器之一（例如，“with:”语句）来正确释放资源。

+   **应用程序试图运行长时间事务** - 数据库事务是非常昂贵的资源，**永远不应保持空闲以等待某个事件发生**。如果应用程序正在等待用户按下按钮，或者等待长时间运行的作业队列中的结果，或者保持持久连接以向浏览器发送请求，**不要在整个时间内保持数据库事务处于打开状态**。当应用程序需要与数据库交互并与事件交互时，在该点打开一个短暂的事务，然后关闭它。

+   **应用程序发生死锁** - 也是此错误的常见原因，更难以理解，如果应用程序由于应用程序端或数据库端的死锁而无法完成对连接的使用，则应用程序可能会使用完所有可用连接，从而导致附加请求接收到此错误。造成死锁的原因包括：

    +   当使用隐式异步系统（如 gevent 或 eventlet）时，如果未正确地对所有套接字库和驱动程序进行猴子补丁，或者对所有猴子补丁驱动程序方法的覆盖不完全，或者在异步系统用于 CPU 绑定的工作负载并且使用数据库资源的 greenlets 等待时间过长时，可能会出现问题。通常情况下，隐式或显式的异步编程框架对于绝大多数关系型数据库操作来说通常不是必要的或合适的；如果应用程序必须在某些功能区域使用异步系统，则最好是数据库导向型业务方法在传统线程内运行，而将消息传递给应用程序的异步部分。

    +   数据库端的死锁，例如行相互死锁

    +   线程错误，例如互相死锁的互斥体，或者在同一线程中调用已锁定的互斥体

请记住，使用连接池的另一种选择是完全关闭连接池。有关此问题的背景，请参阅切换池实现一节。然而，要注意，当发生此错误消息时，这总是由于应用程序本身的问题更大；池只是帮助更早地揭示问题。

请参阅

连接池

与引擎和连接一起工作 ### Pool 类不能与 asyncio 引擎一起使用（反之亦然）

`QueuePool`池类在内部使用`thread.Lock`对象，与 asyncio 不兼容。如果使用`create_async_engine()`函数创建`AsyncEngine`，则适当的队列池类是`AsyncAdaptedQueuePool`，它会自动使用，无需指定。

除了`AsyncAdaptedQueuePool`之外，`NullPool`和`StaticPool`池类不使用锁，并且也适用于与异步引擎一起使用。

在极少数情况下，如果使用`create_engine()`函数明确指定`AsyncAdaptedQueuePool`池类，则也会引发此错误。

另请参阅

连接池  ### 在无效事务回滚之前无法重新连接。请在继续之前完全回滚()

此错误条件指的是`Connection`被使无效，无论是由于数据库断开连接检测还是由于显式调用`Connection.invalidate()`，但仍然存在一个事务，该事务是由`Connection.begin()`方法显式启动，或者由于连接在发出任何 SQL 语句时自动开始事务，如 SQLAlchemy 2.x 系列中发生的情况。当连接被使无效时，任何正在进行的`Transaction`现在处于无效状态，必须显式回滚以将其从`Connection`中移除。  ## DBAPI 错误

Python 数据库 API，或者 DBAPI，是一个数据库驱动程序的规范，可以在[Pep-249](https://www.python.org/dev/peps/pep-0249/)找到。这个 API 指定了一组异常类，适应了数据库的所有故障模式。

SQLAlchemy 不直接生成这些异常。相反，它们被从数据库驱动程序拦截并由 SQLAlchemy 提供的异常 `DBAPIError` 包装，但异常中的消息 **由驱动程序生成，而非 SQLAlchemy**。

### InterfaceError

与数据库本身而非数据库接口相关的错误引发的异常。

此错误是 DBAPI 错误，源自于数据库驱动程序（DBAPI），而非 SQLAlchemy 本身。

`InterfaceError` 有时会由驱动程序在数据库连接被断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参阅 处理断开连接 部分。  ### DatabaseError

与数据库本身而非接口或传递的数据相关的错误引发的异常。

此错误是 DBAPI 错误，源自于数据库驱动程序（DBAPI），而非 SQLAlchemy 本身。  ### DataError

由于处理数据的问题而引发的错误，例如除以零、数值超出范围等。

此错误是 DBAPI 错误，源自于数据库驱动程序（DBAPI），而非 SQLAlchemy 本身。  ### OperationalError

数据库操作中出现的与程序员控制无关的错误引发的异常，例如出现意外断开连接、找不到数据源名称、无法处理事务、在处理过程中发生内存分配错误等。

此错误是 DBAPI 错误，源自于数据库驱动程序（DBAPI），而非 SQLAlchemy 本身。

在数据库连接被断开或无法连接到数据库的情况下，`OperationalError` 是驱动程序中最常见（但不是唯一）使用的错误类。有关如何处理此问题的提示，请参阅 处理断开连接 部分。  ### IntegrityError

数据库的关系完整性受到影响时引发的异常，例如外键检查失败。

此错误是 DBAPI 错误，源自于数据库驱动程序（DBAPI），而非 SQLAlchemy 本身。  ### InternalError

数据库遇到内部错误时引发的异常，例如游标不再有效、事务不同步等。

此错误是 DBAPI 错误，源自于数据库驱动程序（DBAPI），而非 SQLAlchemy 本身。

`InternalError` 有时会由驱动程序在数据库连接被断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参阅 处理断开连接 部分。  ### ProgrammingError

引发编程错误的异常，例如找不到表或已存在，SQL 语句中的语法错误，指定的参数数量错误等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`ProgrammingError`有时由驱动程序引发，原因是数据库连接被断开，或者无法连接到数据库。有关如何处理此问题的提示，请参见处理断开连接部分。  ### NotSupportedError

当方法或数据库 API 使用数据库不支持的情况下引发异常，例如在不支持事务或已关闭事务的连接上请求`.rollback()`。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

## SQL 表达语言

### 对象不会产生缓存键，性能影响

自 SQLAlchemy 版本 1.4 起，包括 SQL 编译缓存机制在内，将允许 Core 和 ORM SQL 结构缓存其字符串形式，以及用于从语句中提取结果的其他结构信息，从而在下次使用另一个结构等效构造时跳过相对昂贵的字符串编译过程。此系统依赖于为所有 SQL 构造实现的功能，包括对象，如 `Column`、`select()` 和 `TypeEngine` 对象，以生成完全代表其状态的**缓存键**，以影响 SQL 编译过程。 

如果问题中的警告涉及到广泛使用的对象，例如 `Column` 对象，并且显示出影响大多数发出的 SQL 结构的情况（使用估计缓存性能使用日志中描述的估算技术），以至于缓存通常不会为应用程序启用，这将对性能产生负面影响，并且在某些情况下，与以前的 SQLAlchemy 版本相比，实际上可能会产生**性能降低**。为什么升级到 1.4 和/或 2.x 后我的应用程序变慢了？ FAQ 对此进行了额外详细的介绍。

#### 如果存在任何疑问，缓存会自行禁用

缓存依赖于能够生成准确表示语句**完整结构**的缓存键以一致的方式。如果特定的 SQL 结构（或类型）没有适当的指令，允许其生成正确的缓存键，则不能安全地启用缓存：

+   缓存键必须表示**完整的结构**：如果两个单独的结构实例的使用可能导致渲染不同的 SQL，则使用不捕捉第一个和第二个元素之间不同之处的缓存键缓存该元素的 SQL 会导致为第二个实例缓存和渲染错误的 SQL。

+   缓存键必须是**一致的**：如果某个结构代表的状态每次都会更改，比如文字值，为每个实例生成唯一的 SQL，那么这个结构也不适合缓存，因为重复使用该结构会很快填满语句缓存，其中包含可能不会再次使用的唯一 SQL 字符串，从而达不到缓存的目的。

出于上述两个原因，SQLAlchemy 的缓存系统对于决定是否缓存与对象对应的 SQL **非常谨慎**。

#### 缓存的断言属性

基于以下标准发出警告。有关每个标准的详细信息，请参阅 为什么在升级到 1.4 和/或 2.x 后我的应用程序变慢了？ 部分。

+   `Dialect` 本身（即由我们传递给 `create_engine()` 的 URL 的第一部分指定的模块，如 `postgresql+psycopg2://`）必须指示已经审查并测试以正确支持缓存，这由 `Dialect.supports_statement_cache` 属性设置为 `True` 来表示。在使用第三方方言时，请与方言的维护者协商，以便他们可以遵循 确保可以启用缓存的步骤 并发布新版本。

+   第三方或用户定义的类型，其继承自`TypeDecorator`或`UserDefinedType`必须在其定义中包含`ExternalType.cache_ok`属性，包括所有派生的子类，遵循`ExternalType.cache_ok`的文档字符串中描述的指南。如前所述，如果这些数据类型是从第三方库导入的，请与该库的维护者联系，以便他们提供必要的更改并发布新版本。

+   第三方或用户定义的 SQL 构造，它们从诸如`ClauseElement`、`Column`、`Insert` 等类继承，包括简单的子类以及设计用于与自定义 SQL 构造和编译扩展一起使用的构造，通常应包括`HasCacheKey.inherit_cache` 属性设置为 `True` 或 `False`，根据构造的设计而定，遵循启用自定义构造的缓存支持中描述的指南。

参见

使用日志估算缓存性能 - 关于观察缓存行为和效率的背景信息

升级到 1.4 和/或 2.x 后，为什么我的应用变慢了？ - 在常见问题解答部分  ### Compiler StrSQLCompiler 无法呈现 `<element type>` 类型的元素

当尝试对包含不是默认编译的元素的 SQL 表达式构造进行字符串化时，通常会发生此错误；在这种情况下，错误将针对`StrSQLCompiler`类。在较少见的情况下，当使用错误类型的 SQL 表达式与特定类型的数据库后端时，也可能发生这种情况；在这些情况下，将命名其他类型的 SQL 编译器类，例如 `SQLCompiler` 或 `sqlalchemy.dialects.postgresql.PGCompiler`。下面的指南更具体地针对“字符串化”用例，但也描述了一般背景。

通常，核心 SQL 结构或 ORM `Query` 对象可以直接字符串化，例如我们使用 `print()`：

```py
>>> from sqlalchemy import column
>>> print(column("x") == 5)
x  =  :x_1 
```

当上述 SQL 表达式被字符串化时，会使用`StrSQLCompiler` 编译器类，这是一个特殊的语句编译器，当一个结构被字符串化而没有任何特定于方言的信息时会被调用。

然而，有许多结构是特定于某种特定类型的数据库方言的，对于这些结构，`StrSQLCompiler` 并不知道如何转换成字符串，例如 PostgreSQL 的“插入冲突” 结构：

```py
>>> from sqlalchemy.dialects.postgresql import insert
>>> from sqlalchemy import table, column
>>> my_table = table("my_table", column("x"), column("y"))
>>> insert_stmt = insert(my_table).values(x="foo")
>>> insert_stmt = insert_stmt.on_conflict_do_nothing(index_elements=["y"])
>>> print(insert_stmt)
Traceback (most recent call last):

...

sqlalchemy.exc.UnsupportedCompilationError:
Compiler <sqlalchemy.sql.compiler.StrSQLCompiler object at 0x7f04fc17e320>
can't render element of type
<class 'sqlalchemy.dialects.postgresql.dml.OnConflictDoNothing'>
```

为了字符串化特定于特定后端的结构，必须使用 `ClauseElement.compile()` 方法，传递一个 `Engine` 或一个 `Dialect` 对象，这将调用正确的编译器。 下面我们使用 PostgreSQL 方言：

```py
>>> from sqlalchemy.dialects import postgresql
>>> print(insert_stmt.compile(dialect=postgresql.dialect()))
INSERT  INTO  my_table  (x)  VALUES  (%(x)s)  ON  CONFLICT  (y)  DO  NOTHING 
```

对于 ORM `Query` 对象，可以使用 `Query.statement` 访问器访问语句：

```py
statement = query.statement
print(statement.compile(dialect=postgresql.dialect()))
```

请查看下面的常见问题解答链接，了解有关直接字符串化/编译 SQL 元素的额外细节。

另请参阅

如何将 SQL 表达式渲染为字符串，可能包含内联的绑定参数？

### TypeError: <operator> 不支持在 ‘ColumnProperty’ 和 <something> 实例之间的操作

这经常发生在尝试在 SQL 表达式的上下文中使用`column_property()` 或 `deferred()` 对象时，通常在声明性语句中，例如：

```py
class Bar(Base):
    __tablename__ = "bar"

    id = Column(Integer, primary_key=True)
    cprop = deferred(Column(Integer))

    __table_args__ = (CheckConstraint(cprop > 5),)
```

在上面的例子中，在映射之前内联使用了 `cprop` 属性，但是这个 `cprop` 属性不是一个`Column`，而是一个`ColumnProperty`，这是一个临时对象，因此不具备 `Column` 对象或 `InstrumentedAttribute` 对象的全部功能，后者将在声明过程完成后映射到 `Bar` 类上。

虽然 `ColumnProperty` 确实有一个 `__clause_element__()` 方法，允许它在某些基于列的上下文中工作，但是它不能在上述开放式比较上下文中工作，因为它没有 Python `__eq__()` 方法，该方法将允许它将对数字 “5” 的比较解释为 SQL 表达式而不是常规的 Python 比较。

解决方法是直接访问 `Column`，使用 `ColumnProperty.expression` 属性：

```py
class Bar(Base):
    __tablename__ = "bar"

    id = Column(Integer, primary_key=True)
    cprop = deferred(Column(Integer))

    __table_args__ = (CheckConstraint(cprop.expression > 5),)
```

### 绑定参数 <x>（在参数组 <y> 中）需要一个值。

当语句使用 `bindparam()` 而在执行语句时未提供值时，就会发生此错误：

```py
stmt = select(table.c.column).where(table.c.id == bindparam("my_param"))

result = conn.execute(stmt)
```

在上面，未提供参数 “my_param” 的值。正确的方法是提供一个值：

```py
result = conn.execute(stmt, {"my_param": 12})
```

当消息采用“需要参数组 <y> 中的绑定参数 <x> 的值”形式时，消息是指向 “executemany” 执行方式。在这种情况下，语句通常是 INSERT、UPDATE 或 DELETE，并传递了参数列表。在这种格式中，语句可以动态生成，以包括参数列表中的每个参数的参数位置，其中它将使用 **第一组参数** 来确定这些参数应该是什么。

例如，下面的语句是基于第一个参数集设置为需要参数 “a”、“b” 和 “c” 而计算的 - 这些名称确定了语句的最终字符串格式，该格式将用于列表中的每组参数。由于第二个条目不包含 “b”，因此会生成此错误：

```py
m = MetaData()
t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))

e.execute(
    t.insert(),
    [
        {"a": 1, "b": 2, "c": 3},
        {"a": 2, "c": 4},
        {"a": 3, "b": 4, "c": 5},
    ],
)
```

```py
sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError)
A value is required for bind parameter 'b', in parameter group 1
[SQL: u'INSERT INTO t (a, b, c) VALUES (?, ?, ?)']
[parameters: [{'a': 1, 'c': 3, 'b': 2}, {'a': 2, 'c': 4}, {'a': 3, 'c': 5, 'b': 4}]]
```

由于需要 “b”，因此将其传递为 `None`，以便 INSERT 可以继续进行：

```py
e.execute(
    t.insert(),
    [
        {"a": 1, "b": 2, "c": 3},
        {"a": 2, "b": None, "c": 4},
        {"a": 3, "b": 4, "c": 5},
    ],
)
```

另请参阅

发送参数  ### 预期的 FROM 子句，却收到了 Select。要创建 FROM 子句，请使用 `.subquery()` 方法。

这指的是 SQLAlchemy 1.4 中的一个更改，即由`select()`等函数生成的 SELECT 语句，但也包括联合和文本 SELECT 表达式等，不再被视为`FromClause`对象，不能直接放在另一个 SELECT 语句的 FROM 子句中，而必须首先将它们包装在`Subquery`中。这是 Core 中的一个重大概念变化，完整的原因讨论在不再将 SELECT 语句隐式视为 FROM 子句中。

给出一个示例如下：

```py
m = MetaData()
t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))
stmt = select(t)
```

在上面，`stmt`代表一个 SELECT 语句。当我们想直接将`stmt`作为另一个 SELECT 语句中的 FROM 子句使用时，就会产生错误，比如如果我们尝试从中选择：

```py
new_stmt_1 = select(stmt)
```

或者如果我们想在 FROM 子句中使用它，比如在 JOIN 中：

```py
new_stmt_2 = select(some_table).select_from(some_table.join(stmt))
```

在 SQLAlchemy 的早期版本中，在另一个 SELECT 语句中使用 SELECT 会产生一个带括号的无名称子查询。在大多数情况下，这种 SQL 形式并不是很有用，因为像 MySQL 和 PostgreSQL 这样的数据库要求 FROM 子句中的子查询具有命名别名，这意味着使用`SelectBase.alias()`方法或者从 1.4 版本开始使用`SelectBase.subquery()`方法来生成这个别名。在其他数据库中，为子查询命名仍然更清晰，以解决子查询内部列名的任何歧义。

除了上述实际原因外，还有许多其他与 SQLAlchemy 相关的原因导致进行了更改。因此，上述两个语句的正确形式要求使用`SelectBase.subquery()`：

```py
subq = stmt.subquery()

new_stmt_1 = select(subq)

new_stmt_2 = select(some_table).select_from(some_table.join(subq))
```

另请参阅

不再将 SELECT 语句隐式视为 FROM 子句  ### 为原始 clauseelement 自动生成别名

从版本 1.4.26 开始新增。

此废弃警告指的是一个非常古老且可能不太熟知的模式，适用于旧版 `Query.join()` 方法以及 2.0 风格 `Select.join()` 方法，其中可以根据 `relationship()` 来说明连接，但是目标是映射到的 `Table` 或其他 Core 可选择对象，而不是 ORM 实体，如映射的类或`aliased()`构造：

```py
a1 = Address.__table__

q = (
    s.query(User)
    .join(a1, User.addresses)
    .filter(Address.email_address == "ed@foo.com")
    .all()
)
```

上述模式还允许使用任意可选择的对象，例如 Core `Join` 或 `Alias` 对象，但是这个元素没有自动适应，这意味着必须直接引用 Core 元素：

```py
a1 = Address.__table__.alias()

q = (
    s.query(User)
    .join(a1, User.addresses)
    .filter(a1.c.email_address == "ed@foo.com")
    .all()
)
```

指定连接目标的正确方式始终是使用映射的类本身或一个`aliased`对象，在后一种情况下，使用 `PropComparator.of_type()`修饰符设置别名：

```py
# normal join to relationship entity
q = s.query(User).join(User.addresses).filter(Address.email_address == "ed@foo.com")

# name Address target explicitly, not necessary but legal
q = (
    s.query(User)
    .join(Address, User.addresses)
    .filter(Address.email_address == "ed@foo.com")
)
```

加入到一个别名：

```py
from sqlalchemy.orm import aliased

a1 = aliased(Address)

# of_type() form; recommended
q = (
    s.query(User)
    .join(User.addresses.of_type(a1))
    .filter(a1.email_address == "ed@foo.com")
)

# target, onclause form
q = s.query(User).join(a1, User.addresses).filter(a1.email_address == "ed@foo.com")
```  ### 由于重叠的表而自动生成别名

自版本 1.4.26 新增。

当使用涉及加入表继承的映射进行查询时，通常会生成此警告。问题在于，在两个具有共同基表的加入继承模型之间进行连接时，不能形成适当的 SQL JOIN 而不对其中一侧应用别名；SQLAlchemy 将别名应用于连接的右侧。例如，给定一个加入继承映射如下：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    manager_id = Column(ForeignKey("manager.id"))
    name = Column(String(50))
    type = Column(String(50))

    reports_to = relationship("Manager", foreign_keys=manager_id)

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "inherit_condition": id == Employee.id,
    }
```

上述映射包括`Employee`和`Manager`类之间的关系。由于这两个类都使用了“employee”数据库表，从 SQL 的角度来看，这是一种自引用关系。如果我们想要使用连接从`Employee`和`Manager`模型中查询，那么在 SQL 层面上，“employee”表需要在查询中包含两次，这意味着它必须被别名化。当我们使用 SQLAlchemy ORM 创建这样一个连接时，得到的 SQL 如下所示：

```py
>>> stmt = select(Employee, Manager).join(Employee.reports_to)
>>> print(stmt)
SELECT  employee.id,  employee.manager_id,  employee.name,
employee.type,  manager_1.id  AS  id_1,  employee_1.id  AS  id_2,
employee_1.manager_id  AS  manager_id_1,  employee_1.name  AS  name_1,
employee_1.type  AS  type_1
FROM  employee  JOIN
(employee  AS  employee_1  JOIN  manager  AS  manager_1  ON  manager_1.id  =  employee_1.id)
ON  manager_1.id  =  employee.manager_id 
```

在上面的 SQL 语句中，选择了`employee`表，代表了查询中的`Employee`实体。然后连接到`employee AS employee_1 JOIN manager AS manager_1`的右嵌套连接，其中`employee`表再次出现，但作为一个匿名别名`employee_1`。这就是警告消息所指的“自动生成别名”。

当 SQLAlchemy 加载包含`Employee`和`Manager`对象的 ORM 行时，ORM 必须将来自上述`employee_1`和`manager_1`表别名的行适应为未别名化的`Manager`类的行。这个过程在内部是复杂的，并且不支持所有 API 特性，特别是当尝试在比这里展示的更深度嵌套的查询中使用`contains_eager()`等急加载特性时。由于这种模式对于更复杂的情况不可靠，并涉及难以预测和遵循的隐式决策，因此会发出警告，并且这种模式可能被视为传统特性。编写此查询的更好方式是使用适用于任何其他自引用关系的相同模式，即显式使用`aliased()`构造。对于连接继承和其他基于连接的映射，通常希望添加使用`aliased.flat`参数，这将允许通过将别名应用于连接中的各个表来对两个或更多表进行连接别名化，而不是将连接嵌入到新的子查询中：

```py
>>> from sqlalchemy.orm import aliased
>>> manager_alias = aliased(Manager, flat=True)
>>> stmt = select(Employee, manager_alias).join(Employee.reports_to.of_type(manager_alias))
>>> print(stmt)
SELECT  employee.id,  employee.manager_id,  employee.name,
employee.type,  manager_1.id  AS  id_1,  employee_1.id  AS  id_2,
employee_1.manager_id  AS  manager_id_1,  employee_1.name  AS  name_1,
employee_1.type  AS  type_1
FROM  employee  JOIN
(employee  AS  employee_1  JOIN  manager  AS  manager_1  ON  manager_1.id  =  employee_1.id)
ON  manager_1.id  =  employee.manager_id 
```

如果我们想要使用`contains_eager()`来填充`reports_to`属性，我们将引用别名：

```py
>>> stmt = (
...     select(Employee)
...     .join(Employee.reports_to.of_type(manager_alias))
...     .options(contains_eager(Employee.reports_to.of_type(manager_alias)))
... )
```

在某些更嵌套的情况下，如果不使用显式的`aliased()`对象，`contains_eager()`选项可能无法获得足够的上下文来确定从哪里获取数据，特别是在 ORM 在非常嵌套的上下文中“自动别名”时。因此，最好不要依赖这个特性，而是尽可能将 SQL 构造明确化。

## 对象关系映射

### IllegalStateChangeError 和并发异常

SQLAlchemy 2.0 引入了一个新系统，详见会话在检测到非法并发或重入访问时主动引发，该系统主动检测在单个 `Session` 对象的实例以及其扩展的 `AsyncSession` 代理对象上调用并发方法。这些并发访问调用通常会发生在单个 `Session` 实例在多个并发线程之间共享而没有进行同步访问时，或者类似地，当单个 `AsyncSession` 实例在多个并发任务之间共享时（例如使用 `asyncio.gather()` 这样的函数）。这些使用模式不是这些对象的适当用法，在没有 SQLAlchemy 实现的主动警告系统的情况下，仍然会在对象内部产生无效状态，从而产生难以调试的错误，包括数据库连接本身的驱动程序级错误。

`Session` 和 `AsyncSession` 的实例都是**可变、有状态的对象，没有内置的方法调用同步**，并且代表着一次单一的数据库事务，该事务在一次特定的 `Engine` 或 `AsyncEngine` 绑定的数据库连接上进行（请注意，这些对象都支持同时绑定到多个引擎，但在这种情况下，在事务范围内仍然只会有一个连接与引擎相关）。单个数据库事务不是并发 SQL 命令的适当目标；相反，运行并发数据库操作的应用程序应该使用并发事务。因此，对于这些对象，适当的模式是每个线程一个 `Session` 或每个任务一个 `AsyncSession`。

有关并发的更多背景信息，请参阅会话是否线程安全？AsyncSession 是否可以在并发任务中共享？一节。 ### 父实例 <x> 未绑定到会话；（延迟加载/延迟加载/刷新等）操作无法继续

这很可能是处理 ORM 时最常见的错误消息，并且它是由 ORM 广泛使用的一种技术的性质引起的，这种技术称为延迟加载。延迟加载是一种常见的对象关系模式，其中由 ORM 持久化的对象维护了与数据库本身的代理，以便当访问对象上的各种属性时，可以*延迟*从数据库中检索其值。这种方法的优点是可以从数据库中检索对象而不必一次加载其所有属性或相关数据，而只能在那时提供所请求的数据。其主要缺点基本上是优点的镜像，即如果正在加载大量对象，这些对象在所有情况下都需要某一组数据，则逐步加载该额外数据是一种浪费。

对于懒加载的另一个警告，除了通常的效率问题之外，还有一个要注意的是，为了进行懒加载，对象必须**保持与会话相关联**，以便能够检索其状态。这个错误消息意味着一个对象已经与其`Session`解除关联，并且被要求从数据库中懒加载数据。

对象变为分离状态的最常见原因是会话本身已关闭，通常是通过`Session.close()`方法关闭的。然后，对象将继续存在以供进一步访问，这在 Web 应用程序中非常常见，其中它们被传递到服务器端模板引擎，并被要求加载更多属性。

减轻这个错误的方法是通过以下技术：

+   **尽量不要有分离的对象；不要过早关闭会话** - 通常，应用程序会在将相关对象传递给其他系统之前关闭事务，然后由于此错误而失败。有时，事务不需要那么快关闭；一个例子是 Web 应用在渲染视图之前关闭了事务。这通常是以“正确性”的名义而做的，但可能被视为“封装”的误用，因为此术语指的是代码组织，而不是实际操作。使用 ORM 对象的模板正在使用[代理模式](https://en.wikipedia.org/wiki/Proxy_pattern)，它将数据库逻辑封装在调用者之外。如果`Session`可以保持打开状态直到对象的生命周期结束，那么这是最佳方法。

+   **否则，将需要的所有内容一次性加载** - 通常不可能保持事务处于打开状态，特别是在需要将对象传递给其他无法在同一上下文中运行的系统的更复杂的应用程序中。在这种情况下，应用程序应准备处理分离对象，并应尽量恰当地使用急加载来确保对象一开始就拥有所需的内容。

+   **并且重要的是，将 expire_on_commit 设置为 False** - 当使用分离的对象时，对象需要重新加载数据的最常见原因是因为它们在上次调用 `Session.commit()` 时过期了。当处理分离的对象时，不应使用此过期；因此，`Session.expire_on_commit` 参数应设置为 `False`。通过防止对象在事务外过期，加载的数据将保持存在，并且在访问该数据时不会产生额外的延迟加载。

    `Session.rollback()` 方法无条件地使 `Session` 中的所有内容过期，并且在非错误情况下也应避免使用。

    另请参阅

    关系加载技术 - 关于急加载和其他基于关系的加载技术的详细文档

    提交 - 有关会话提交的背景

    刷新 / 过期 - 属性过期的背景 ### 此 Session 的事务由于在 flush 过程中出现先前的异常而被回滚

`Session` 的 flush 过程，在遇到错误时会回滚数据库事务，以保持内部一致性。但是，一旦发生这种情况，会话的事务现在处于“不活动”状态，必须由调用方显式地回滚，就像如果没有发生失败，则必须显式地提交一样。

当使用 ORM 时，这是一个常见错误，通常适用于尚未正确围绕其`Session`操作进行“框架化”的应用程序。更多详细信息请参阅 FAQ 中的“由于刷新期间的先前异常，此会话的事务已被回滚。”（或类似）。### 对于关系<relationship>，delete-orphan 级联通常仅在一对多关系的“一”侧上配置，而不在多对一或多对多关系的“多”侧上配置。

当在多对一或多对多关系上设置“delete-orphan”级联时，就会出现这个错误，例如：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    # this will emit the error message when the mapper
    # configuration step occurs
    a = relationship("A", back_populates="bs", cascade="all, delete-orphan")

configure_mappers()
```

上面，对`B.a`上的“delete-orphan”设置表示的意图是，当引用特定`A`的每个`B`对象被删除时，该`A`也应该被删除。也就是说，它表达了被删除的“孤儿”将是一个`A`对象，并且当引用它的每个`B`被删除时，它就成为一个“孤儿”。

“delete-orphan”级联模型不支持这一功能。“孤儿”考虑仅在删除一个对象时进行，然后该对象将引用零个或多个现在由此单个删除“孤儿化”的对象，这将导致这些对象也被删除。换句话说，它仅设计用于跟踪基于删除一个且仅一个“父”对象每个孤儿的创建，这是一对多关系中的自然情况，其中在“一”侧的对象的删除导致“多”侧的相关项目随后被删除。

为了支持这一功能，上述映射将级联设置放在一对多的一侧，看起来像是：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a", cascade="all, delete-orphan")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship("A", back_populates="bs")
```

其中表达的意图是，当删除一个`A`时，它所引用的所有`B`对象也被删除。

然后错误消息继续建议使用`relationship.single_parent`标志。该标志可用于强制执行一个关系，该关系能够让许多对象引用特定对象，实际上每次只会有**一个**对象引用它。它用于传统或其他不太理想的数据库模式，其中外键关系暗示“多”集合，但实际上只有一个对象会引用给定目标对象。这种不常见的情况可以通过上面的示例来演示：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship(
        "A",
        back_populates="bs",
        single_parent=True,
        cascade="all, delete-orphan",
    )
```

上述配置将安装一个验证器，该验证器将强制执行在`B.a`关系的范围内，只能有一个`B`与一个`A`关联：

```py
>>> b1 = B()
>>> b2 = B()
>>> a1 = A()
>>> b1.a = a1
>>> b2.a = a1
sqlalchemy.exc.InvalidRequestError: Instance <A at 0x7eff44359350> is
already associated with an instance of <class '__main__.B'> via its
B.a attribute, and is only allowed a single parent.
```

请注意，此验证器的范围有限，并且不会阻止通过其他方向创建多个“父对象”。例如，它不会检测到关于`A.bs`的相同设置：

```py
>>> a1.bs = [b1, b2]
>>> session.add_all([a1, b1, b2])
>>> session.commit()
INSERT  INTO  a  DEFAULT  VALUES
()
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,)
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,) 
```

然而，事情不会按预期进行，因为“delete-orphan”级联将继续按照**单个**主导对象的术语工作，这意味着如果我们删除`B`对象中的**任意一个**，`A`就会被删除。另一个`B`还会留下，ORM 通常足够智能以将外键属性设置为 NULL，但这通常不是预期的结果：

```py
>>> session.delete(b1)
>>> session.commit()
UPDATE  b  SET  a_id=?  WHERE  b.id  =  ?
(None,  2)
DELETE  FROM  b  WHERE  b.id  =  ?
(1,)
DELETE  FROM  a  WHERE  a.id  =  ?
(1,)
COMMIT 
```

对于上述所有示例，类似的逻辑也适用于多对多关系的微积分；如果一个多对多关系在一侧设置了 single_parent=True，那么该侧可以使用“delete-orphan”级联，但这很可能不是实际想要的，因为多对多关系的目的是使得可以有许多对象引用任一方向的对象。

总的来说，“delete-orphan”级联通常应用于一对多关系的“一”侧，以便删除“多”侧的对象，而不是反过来。

在 1.3.18 版本中更改：当在多对一或多对多关系上使用“delete-orphan”错误消息时，已更新为更具描述性的文本。

另请参阅

级联

delete-orphan

实例<instance>已通过其<attribute>属性与<instance>的实例关联，并且仅允许有一个单独的父对象。  ### 实例<instance>已通过其<attribute>属性与<instance>的实例关联，并且仅允许有一个单独的父对象。

当 `relationship.single_parent` 标志被使用，并且一个对象同时被指定为多个对象的“父对象”时，会发出此错误。

鉴于以下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship(
        "A",
        single_parent=True,
        cascade="all, delete-orphan",
    )
```

意图表明，不会有多于一个`B`对象同时引用特定的`A`对象：

```py
>>> b1 = B()
>>> b2 = B()
>>> a1 = A()
>>> b1.a = a1
>>> b2.a = a1
sqlalchemy.exc.InvalidRequestError: Instance <A at 0x7eff44359350> is
already associated with an instance of <class '__main__.B'> via its
B.a attribute, and is only allowed a single parent.
```

当此错误出现意外时，通常是因为在对对于关系<relationship>，delete-orphan 级联通常仅在一对多关系的“一”侧配置，并不在多对一或多对多关系的“多”侧上。描述的错误消息做出响应时应用了`relationship.single_parent`标志，而实际问题是对“delete-orphan”级联设置的误解。有关详细信息，请参阅该消息。

另请参阅

对于关系<relationship>，删除孤儿级联通常仅在一对多关系的“一”方配置，并不在多对一或多对多关系的“多”方配置。  ### 关系 X 将列 Q 复制到列 P，与关系‘Y’冲突

此警告指的是在刷新时两个或多个关系将写入相同列的情况，但 ORM 没有任何手段来协调这些关系。根据具体情况，解决方案可能是两个关系需要使用`relationship.back_populates`相互引用，或者一个或多个关系应该配置为`relationship.viewonly`以防止冲突写入，有时配置是完全有意的，应该配置`relationship.overlaps`以消除每个警告。

对于缺少`relationship.back_populates`的典型示例，给定以下映射：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    children = relationship("Child")

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))
    parent = relationship("Parent")
```

上述映射将生成警告：

```py
SAWarning: relationship 'Child.parent' will copy column parent.id to column child.parent_id,
which conflicts with relationship(s): 'Parent.children' (copies parent.id to child.parent_id).
```

关系`Child.parent`和`Parent.children`似乎存在冲突。解决方案是应用`relationship.back_populates`:

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))
    parent = relationship("Parent", back_populates="children")
```

对于更自定义的关系，在“重叠”情况可能是有意的且无法解决的情况下，`relationship.overlaps`参数可以指定不应发生警告的关系名称。这通常发生在对同一底层表的两个或多个关系具有自定义`relationship.primaryjoin`条件以限制每种情况下相关项目的情况：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    c1 = relationship(
        "Child",
        primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 0)",
        backref="parent",
        overlaps="c2, parent",
    )
    c2 = relationship(
        "Child",
        primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 1)",
        overlaps="c1, parent",
    )

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))

    flag = Column(Integer)
```

在上述示例中，ORM 将知道`Parent.c1`、`Parent.c2`和`Child.parent`之间的重叠是有意的。  ### 对象无法转换为‘persistent’状态，因为此标识映射不再有效。

版本 1.4.26 中的新功能。

这条消息是为了处理以下情况而添加的：在原始`Session`关闭后，或者在调用其`Session.expunge_all()`方法后，迭代可能会产生 ORM 对象的`Result`对象。当一个`Session`一次性地移除所有对象时，该`Session`使用的内部标识映射将被替换为一个新的映射，并且原始映射将被丢弃。一个未被使用和未被缓冲的`Result`对象将在内部保留对该现在被丢弃的标识映射的引用。因此，当消耗了`Result`时，将无法将要产生的对象与该`Session`关联起来。这种安排是有意的，因为通常不建议在创建它的事务上下文之外迭代未缓冲的`Result`对象：

```py
# context manager creates new Session
with Session(engine) as session_obj:
    result = sess.execute(select(User).where(User.id == 7))

# context manager is closed, so session_obj above is closed, identity
# map is replaced

# iterating the result object can't associate the object with the
# Session, raises this error.
user = result.first()
```

使用`asyncio` ORM 扩展时，上述情况通常**不会**发生，因为当`AsyncSession`返回同步风格的`Result`时，结果在执行语句时已经预先缓冲。这样可以允许次要的急切加载器调用而无需额外的`await`调用。

要在上述情况下像`asyncio`扩展一样预先缓冲结果，可以使用`prebuffer_rows`执行选项，如下所示：

```py
# context manager creates new Session
with Session(engine) as session_obj:
    # result internally pre-fetches all objects
    result = sess.execute(
        select(User).where(User.id == 7), execution_options={"prebuffer_rows": True}
    )

# context manager is closed, so session_obj above is closed, identity
# map is replaced

# pre-buffered objects are returned
user = result.first()

# however they are detached from the session, which has been closed
assert inspect(user).detached
assert inspect(user).session is None
```

在上述代码块中，所选的 ORM 对象完全在`session_obj`块内生成，与`session_obj`关联，并在`Result`对象中缓冲以供迭代。在块外，`session_obj`被关闭并且移除了这些 ORM 对象。迭代`Result`对象将产生这些 ORM 对象，但是由于它们的来源`Session`已经移除了它们，它们将以分离状态提供。

注意

上面对“预缓冲”与“非缓冲” `Result` 对象的引用是指 ORM 将来自 DBAPI 的原始数据库行转换为 ORM 对象的过程。这并不意味着底层的 `cursor` 对象本身是否被缓冲，表示来自 DBAPI 的待处理结果，它本身是被缓冲的还是非缓冲的，因为这本质上是一个更低层的缓冲。关于 `cursor` 结果本身的缓冲，请参阅使用服务器端游标（也称为流结果）部分。  ### 类型注释无法解释为注释的声明性表单

SQLAlchemy 2.0 引入了一个新的注释声明式表声明系统，该系统从运行时类定义中的 [**PEP 484**](https://peps.python.org/pep-0484/) 注释中派生 ORM 映射属性信息。此形式的要求是所有 ORM 注释必须使用称为 `Mapped` 的通用容器进行正确注释。包括显式 [**PEP 484**](https://peps.python.org/pep-0484/) 类型注释的遗留 SQLAlchemy 映射，例如使用 遗留 Mypy 扩展进行类型支持的映射，可能包括诸如 `relationship()` 的指令，不包括此通用容器。

为了解决此问题，可以将类标记为 `__allow_unmapped__` 布尔属性，直到它们完全迁移到 2.0 语法。请参阅迁移说明，例如 迁移到 2.0 的第六步 - 向显式类型的 ORM 模型添加 __allow_unmapped__ 的示例。

另请参阅

迁移到 2.0 的第六步 - 向显式类型的 ORM 模型添加 __allow_unmapped__ - 在 SQLAlchemy 2.0 - 主要迁移指南 文档中  ### 当将 <cls> 转换为数据类时，属性(s) 源自非数据类的父类 <cls>。

当使用描述在任何 mixin 类或抽象基类中的 SQLAlchemy ORM 映射数据类特性，该特性本身并未声明为数据类，例如下面的示例所示时，会发生此警告：

```py
from __future__ import annotations

import inspect
from typing import Optional
from uuid import uuid4

from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Mixin:
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

class Base(DeclarativeBase, MappedAsDataclass):
    pass

class User(Base, Mixin):
    __tablename__ = "sys_user"

    uid: Mapped[str] = mapped_column(
        String(50), init=False, default_factory=uuid4, primary_key=True
    )
    username: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column()
```

由于 `Mixin` 本身不是从 `MappedAsDataclass` 扩展的，因此会生成以下警告：

```py
SADeprecationWarning: When transforming <class '__main__.User'> to a
dataclass, attribute(s) "create_user", "update_user" originates from
superclass <class
'__main__.Mixin'>, which is not a dataclass. This usage is deprecated and
will raise an error in SQLAlchemy 2.1\. When declaring SQLAlchemy
Declarative Dataclasses, ensure that all mixin classes and other
superclasses which include attributes are also a subclass of
MappedAsDataclass.
```

修复方法是在 `Mixin` 的签名中添加 `MappedAsDataclass`：

```py
class Mixin(MappedAsDataclass):
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)
```

Python 的[**PEP 681**](https://peps.python.org/pep-0681/)规范不支持在数据类的超类上声明的属性，这些超类本身不是数据类；根据 Python 数据类的行为，这些字段将被忽略，如下例所示：

```py
from dataclasses import dataclass
from dataclasses import field
import inspect
from typing import Optional
from uuid import uuid4

class Mixin:
    create_user: int
    update_user: Optional[int] = field(default=None)

@dataclass
class User(Mixin):
    uid: str = field(init=False, default_factory=lambda: str(uuid4()))
    username: str
    password: str
    email: str
```

上面，`User`类将不会在其构造函数中包含`create_user`，也不会尝试将`update_user`解释为数据类属性。这是因为`Mixin`不是数据类。

SQLAlchemy 2.0 系列中的数据类功能未正确遵守这一行为；相反，非数据类混合类和超类上的属性将被视为最终数据类配置的一部分。然而，像 Pyright 和 Mypy 这样的类型检查器不会将这些字段视为数据类构造函数的一部分，因为根据[**PEP 681**](https://peps.python.org/pep-0681/)它们应该被忽略。否则，由于它们的存在是模棱两可的，SQLAlchemy 2.1 将要求在数据类层次结构中具有 SQLAlchemy 映射属性的混合类本身必须是数据类。### 创建<类名>数据类时遇到的 Python 数据类错误

当使用`MappedAsDataclass`混合类或`registry.mapped_as_dataclass()`装饰器时，SQLAlchemy 利用 Python 标准库中的实际[Python 数据类](https://docs.python.org/3/library/dataclasses.html)模块，以将数据类行为应用于目标类。此 API 有其自己的错误场景，其中大部分涉及在用户定义的类上构建`__init__()`方法；在类上声明的属性的顺序，以及[在超类上](https://docs.python.org/3/library/dataclasses.html#inheritance)的顺序决定了`__init__()`方法将如何构建，还有特定规则规定了属性的组织方式以及它们应如何使用参数如`init=False`、`kw_only=True`等。**SQLAlchemy 不控制或实现这些规则**。因此，对于这种类型的错误，请参考[Python 数据类](https://docs.python.org/3/library/dataclasses.html)文档，特别注意应用于[继承](https://docs.python.org/3/library/dataclasses.html#inheritance)的规则。

另请参见

声明式数据类映射 - SQLAlchemy 数据类文档

[Python 数据类](https://docs.python.org/3/library/dataclasses.html) - 在 python.org 网站上

[继承](https://docs.python.org/3/library/dataclasses.html#inheritance) - 在 python.org 网站上  ### 按主键进行逐行 ORM 批量更新需要记录包含主键值

在不提供给定记录中的主键值的情况下使用 ORM 批量更新 by Primary Key 功能时会发生此错误，例如：

```py
>>> session.execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
```

在上述情况中，参数字典列表的存在与使用`Session`来执行 ORM 启用的 UPDATE 语句会自动使用基于主键的 ORM 批量更新，该方法期望参数字典包含主键值，例如：

```py
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...         {"id": 5, "fullname": "Eugene H. Krabs"},
...     ],
... )
```

要在不提供每个记录的主键值的情况下调用 UPDATE 语句，请使用`Session.connection()`来获取当前`Connection`，然后使用该连接进行调用：

```py
>>> session.connection().execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
```

另请参阅

ORM 批量更新 by Primary Key

针对具有多个参数集的 UPDATE 语句禁用基于主键的 ORM 批量更新

## 异步 I/O 异常

### 需要等待

SQLAlchemy 的异步模式需要使用异步驱动程序连接到数据库。尝试使用不兼容的 DBAPI 的情况下，通常会引发此错误。

另请参阅

异步 I/O（asyncio）  ### 丢失 Greenlet

对异步 DBAPI 的调用是在通常由 SQLAlchemy AsyncIO 代理类设置的 greenlet spawn 上下文之外启动的。通常情况下，当尝试在意料之外的位置进行 IO 时，会发生此错误，使用的调用模式不直接提供使用`await`关键字的情况。在使用 ORM 时，几乎总是由于使用了延迟加载，在不经过额外步骤和/或使用成功所需的替代加载器模式的情况下，不直接支持 asyncio。

另请参阅

在使用 AsyncSession 时防止隐式 IO - 涵盖了大多数可能出现此问题的 ORM 方案以及如何进行缓解，包括与延迟加载场景一起使用的特定模式。 ### 无可用检查

直接在`AsyncConnection`或`AsyncEngine`对象上直接使用`inspect()`函数目前不受支持，因为尚未提供`Inspector`对象的可等待形式。相反，通过使用`inspect()`函数获取对象，使其引用`AsyncConnection`对象的底层`AsyncConnection.sync_connection`属性；然后通过使用`AsyncConnection.run_sync()`方法以及执行所需操作的自定义函数以“同步”调用样式使用`Inspector`：

```py
async def async_main():
    async with engine.connect() as conn:
        tables = await conn.run_sync(
            lambda sync_conn: inspect(sync_conn).get_table_names()
        )
```

另请参阅

使用检查器检查模式对象 - 使用`inspect()`与 asyncio 扩展的其他示例。

## Core 异常类

查看 Core Exceptions 以获取 Core 异常类。

## ORM 异常类

查看 ORM Exceptions 以获取 ORM 异常类。

## 遗留异常

本节中的异常不是由当前 SQLAlchemy 版本生成的，但在此提供以适应异常消息超链接。

### 在 SQLAlchemy 2.0 中，<某个函数>将不再<某事>。

SQLAlchemy 2.0 代表了 Core 和 ORM 组件中许多关键 SQLAlchemy 使用模式的重大转变。2.0 版本的目标是对 SQLAlchemy 自其早期开始以来的一些最基本假设进行轻微调整，并提供一个新的简化使用模型，希望在 Core 和 ORM 组件之间更加简约一致，同时更具能力。

在 SQLAlchemy 2.0 - 主要迁移指南中介绍的 SQLAlchemy 2.0 项目包含了一个综合的未来兼容性系统，该系统集成到了 SQLAlchemy 1.4 系列中，以便应用程序能够明确、清晰地、逐步地升级路径，将应用程序迁移到完全兼容 2.0 版本。`RemovedIn20Warning`弃用警告是该系统的基础，用于提供关于现有代码库中需要修改的行为的指导。如何启用此警告的概述在 SQLAlchemy 2.0 弃用模式中。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南 - 从 1.x 系列升级流程的概述，以及 SQLAlchemy 2.0 的当前目标和进展。

SQLAlchemy 2.0 弃用模式 - 如何在 SQLAlchemy 1.4 中使用“2.0 弃用模式”的具体指南。 ### 对象正在通过反向引用级联合并到一个 Session

本消息指的是 SQLAlchemy 中在 2.0 版本中删除的“反向引用级联”行为。这指的是将对象添加到`Session`中的操作，因为该会话中已经存在的另一个对象与之关联。由于这种行为被证明比有用更令人困惑，因此添加了`relationship.cascade_backrefs`和`backref.cascade_backrefs`参数，可以将其设置为`False`以禁用它，在 SQLAlchemy 2.0 中完全删除了“级联反向引用”行为。

对于较旧的 SQLAlchemy 版本，在当前使用`relationship.backref`字符串参数配置的反向引用上设置`relationship.cascade_backrefs`为`False`，必须首先使用`backref()`函数声明该反向引用，以便传递`backref.cascade_backrefs`参数。

或者，可以通过在“未来”模式下使用`Session`来完全关闭整个“级联反向引用”行为，通过将`True`传递给`Session.future`参数。

另请参阅

cascade_backrefs 行为在 2.0 中被弃用以移除 - 关于 SQLAlchemy 2.0 变更的背景。### select() 构造以“旧”模式创建；关键字参数等。

`select()` 构造已在 SQLAlchemy 1.4 中更新，以支持在 SQLAlchemy 2.0 中标准的新调用风格。为了向后兼容在 1.4 系列内，该构造接受“旧”风格和“新”风格的参数。

“新”风格的特性是列和表达式只能按位置传递给`select()` 构造；对象的任何其他修饰符都必须使用后续的方法链传递：

```py
# this is the way to do it going forward
stmt = select(table1.c.myid).where(table1.c.myid == table2.c.otherid)
```

对比之下，在 SQLAlchemy 的旧形式中，`select()` 在像 `Select.where()` 这样的方法甚至添加之前，可能是这样的：

```py
# this is how it was documented in original SQLAlchemy versions
# many years ago
stmt = select([table1.c.myid], whereclause=table1.c.myid == table2.c.otherid)
```

或者甚至“whereclause”会按位置传递：

```py
# this is also how it was documented in original SQLAlchemy versions
# many years ago
stmt = select([table1.c.myid], table1.c.myid == table2.c.otherid)
```

近年来，大多数叙述性文档已经删除了接受的“whereclause”和其他参数，导致调用风格更像是作为列参数传递的列列表，但没有进一步的参数：

```py
# this is how it's been documented since around version 1.0 or so
stmt = select([table1.c.myid]).where(table1.c.myid == table2.c.otherid)
```

select() 不再接受多样的构造参数，列仅按位置传递 中的文档描述了这一变更的 2.0 迁移。

另请参阅

select() 不再接受多样的构造参数，列仅按位置传递

SQLAlchemy 2.0 - 主要迁移指南  ### 通过旧绑定的元数据找到了一个绑定，但由于该 Session 设置了 future=True，因此该绑定被忽略。

直到 SQLAlchemy 1.4，都存在“bound metadata”的概念；从 SQLAlchemy 2.0 开始已经移除。

此错误指的是`MetaData.bind`参数，该参数位于`MetaData`对象上，该对象又允许 ORM `Session`将特定的映射类与`Engine`关联起来。在 SQLAlchemy 2.0 中，`Session`必须直接链接到每个`Engine`。也就是说，不是实例化`Session`或`sessionmaker`而不带任何参数，并将`Engine`与`MetaData`关联起来：

```py
engine = create_engine("sqlite://")
Session = sessionmaker()
metadata_obj = MetaData(bind=engine)
Base = declarative_base(metadata=metadata_obj)

class MyClass(Base): ...

session = Session()
session.add(MyClass())
session.commit()
```

`Engine`必须直接与`sessionmaker`或`Session`关联。`MetaData`对象不应再关联任何引擎：

```py
engine = create_engine("sqlite://")
Session = sessionmaker(engine)
Base = declarative_base()

class MyClass(Base): ...

session = Session()
session.add(MyClass())
session.commit()
```

在 SQLAlchemy 1.4 中，当`Session.future`标志设置在`sessionmaker`或`Session`上时，将启用此 2.0 风格行为。 ### 此编译对象未绑定到任何 Engine 或 Connection

此错误指的是“绑定的元数据”概念，这是仅存在于 1.x 版本的传统 SQLAlchemy 模式。当直接在未关联任何`Engine`的 Core 表达式对象上调用`Executable.execute()`方法时，会出现此问题：

```py
metadata_obj = MetaData()
table = Table("t", metadata_obj, Column("q", Integer))

stmt = select(table)
result = stmt.execute()  # <--- raises
```

逻辑期望的是`MetaData`对象已经**绑定**到一个`Engine`上：

```py
engine = create_engine("mysql+pymysql://user:pass@host/db")
metadata_obj = MetaData(bind=engine)
```

在上述情况下，任何从`Table`派生的语句，又从`MetaData`派生的语句将隐式地利用给定的`Engine`来调用该语句。

请注意，绑定元数据的概念**在 SQLAlchemy 2.0 中不存在**。调用语句的正确方式是通过`Connection.execute()`方法的`Connection`：

```py
with engine.connect() as conn:
    result = conn.execute(stmt)
```

在使用 ORM 时，可以通过`Session`获得类似的功能：

```py
result = session.execute(stmt)
```

另请参阅

语句执行基础知识  ### 此连接处于非活动事务状态。请在继续之前完全回滚()

该错误条件是在 SQLAlchemy 版本 1.4 中添加的，不适用于 SQLAlchemy 2.0\. 该错误是指将`Connection`置于使用诸如`Connection.begin()`之类的方法创建的事务中，然后在该范围内创建进一步的“标记”事务；然后使用`Transaction.rollback()`回滚或使用`Transaction.close()`关闭“标记”事务，但是外部事务仍然处于“非活动”状态，必须回滚。

这种模式看起来像：

```py
engine = create_engine(...)

connection = engine.connect()
transaction1 = connection.begin()

# this is a "sub" or "marker" transaction, a logical nesting
# structure based on "real" transaction transaction1
transaction2 = connection.begin()
transaction2.rollback()

# transaction1 is still present and needs explicit rollback,
# so this will raise
connection.execute(text("select 1"))
```

上述中，`transaction2` 是一个“标记”事务，表示事务在外部事务内的逻辑嵌套；虽然内部事务可以通过其 rollback() 方法回滚整个事务，但其 commit() 方法除了关闭“标记”事务的范围外，没有其他效果。调用 `transaction2.rollback()` 的效果是**取消激活** transaction1，这意味着它在数据库级别上实际上已回滚，但仍然存在以适应事务的一致嵌套模式。

正确的解决方法是确保外部事务也已回滚：

```py
transaction1.rollback()
```

这种模式在 Core 中不常用。在 ORM 中，可能会出现类似的问题，这是 ORM 的“逻辑”事务结构的产物；这在“此会话的事务由于刷新期间的先前异常而被回滚。”（或类似内容）的常见问题解答条目中有描述。

SQLAlchemy 2.0 中已删除“子事务”模式，因此不再提供此特定编程模式，从而防止出现此错误消息。

## 连接和事务

### 队列池大小限制<x>已达到溢出<y>，连接超时，超时<z>

这可能是最常见的运行时错误，因为它直接涉及应用程序的工作负载超过配置限制，这通常适用于几乎所有 SQLAlchemy 应用程序。

以下几点总结了这个错误的含义，从大多数 SQLAlchemy 用户应该已经熟悉的最基本点开始。

+   **SQLAlchemy 引擎对象默认使用连接池** - 这意味着当使用`Engine`对象的 SQL 数据库连接资源，并且释放该资源后，数据库连接本身仍保持连接状态，并返回到内部队列，可以再次使用。尽管代码可能看起来已经结束了与数据库的交互，但在许多情况下，应用程序仍会保持一定数量的数据库连接，直到应用程序结束或显式处理池。

+   由于连接池，当应用程序使用 SQL 数据库连接时，通常是通过使用`Engine.connect()`或使用 ORM `Session`进行查询时，此活动并不一定在获取连接对象时立即建立新连接到数据库；相反，它会向连接池查询连接，该连接池通常会检索一个现有连接以供重复使用。如果没有可用连接，连接池将创建一个新的数据库连接，但前提是池未超过配置容量。

+   大多数情况下使用的默认池称为`QueuePool`。当您要求此池提供连接但没有可用连接时，它将创建一个新连接**如果当前连接总数小于配置值**。该值等于**池大小加上最大溢出**。这意味着如果您已将引擎配置为：

    ```py
    engine = create_engine("mysql+mysqldb://u:p@host/db", pool_size=10, max_overflow=20)
    ```

    上述`Engine`将允许**最多同时有 30 个连接**在使用中，不包括从引擎分离或失效的连接。如果新连接请求到达并且应用程序的其他部分已经使用了 30 个连接，连接池将阻塞一段固定时间，然后超时并引发此错误消息。

    为了允许更多的连接同时使用，可以使用传递给`create_engine()`函数的`create_engine.pool_size`和`create_engine.max_overflow`参数来调整池。等待连接可用的超时时间是使用`create_engine.pool_timeout`参数配置的。

+   可以通过将`create_engine.max_overflow`设置为值“-1”来配置池以具有无限溢出。使用此设置，池仍将维护一组固定的连接池，但如果没有可用的连接，则永远不会阻止新连接的请求；相反，如果没有可用的连接，它将无条件地建立一个新连接。

    然而，以这种方式运行时，如果应用程序存在使用所有可用连接资源的问题，它最终将达到数据库本身可用连接的配置限制，这将再次返回错误。更严重的是，当应用程序耗尽数据库连接时，通常会在失败之前使用大量资源，并且还可能干扰其他依赖于能够连接到数据库的应用程序和数据库状态机制。

    根据上述情况，连接池可以被视为连接使用的**安全阀**，为防止一个恶意应用导致整个数据库对所有其他应用不可用提供了至关重要的保护层。当收到此错误消息时，最好修复使用过多连接和/或适当配置限制的问题，而不是允许无限溢出，因为这实际上并不能解决潜在问题。

应用程序耗尽所有可用连接的原因是什么？

+   **应用程序正在处理过多的并发请求，以根据池的配置值进行工作** - 这是最直接的原因。如果您的应用程序在允许 30 个并发线程的线程池中运行，并且每个线程使用一个连接，则如果您的池未配置为允许同时至少检出 30 个连接，一旦您的应用程序接收到足够的并发请求，您将会收到此错误。解决方法是提高池的限制或降低并发线程数。

+   **应用程序没有将连接归还到池中** - 这是接下来最常见的原因，即应用程序正在使用连接池，但程序未能将这些连接释放并且仍然保持打开状态。连接池以及 ORM `Session` 都有逻辑，当会话和/或连接对象被垃圾回收时，底层连接资源将被释放，但不能依赖这种行为及时释放资源。

    这种情况发生的常见原因是应用程序使用 ORM 会话但在完成使用该会话的工作后没有调用 `Session.close()`。解决方案是确保在完成工作时显式关闭 ORM 会话（如果使用 ORM）或引擎绑定的 `Connection` 对象（如果使用 Core），通过适当的 `.close()` 方法或使用其中一个可用的上下文管理器（例如，“with:”语句）来正确释放资源。

+   **应用程序试图运行长时间事务** - 数据库事务是一种非常昂贵的资源，绝对**不能让其空闲等待某些事件发生**。如果一个应用程序正在等待用户按下按钮，或者等待长时间运行的作业队列中的结果，或者正在保持与浏览器的持久连接打开，请**不要在整个时间段保持数据库事务处于打开状态**。当应用程序需要与数据库交互并处理事件时，在那一点上打开一个短暂的事务，然后关闭它。

+   **应用程序发生死锁** - 这也是这个错误的常见原因之一，也更难以理解，如果一个应用程序无法完成其对连接的使用，无论是由于应用程序端还是数据库端的死锁，应用程序都会使用完所有可用的连接，这样会导致其他请求收到这个错误。死锁的原因包括：

    +   如果使用隐式异步系统（例如 gevent 或 eventlet）而没有正确地 monkeypatch 所有的 socket 库和驱动程序，或者在没有完全覆盖所有被 monkeypatch 的驱动程序方法的情况下存在漏洞，或者更少见的情况是异步系统正在用于 CPU 密集型工作负载且 greenlets 使用数据库资源的等待时间过长。对于绝大多数关系型数据库操作来说，隐式或显式的异步编程框架通常是不必要或不合适的；如果应用程序必须在某些功能区域使用异步系统，最好是数据库导向型业务方法在传统线程中运行，然后将消息传递给应用程序的异步部分。

    +   数据库端死锁，例如行之间相互死锁

    +   线程错误，例如互相死锁的互斥锁，或者在同一线程中调用已锁定的互斥锁

请记住，使用池的另一种选择是完全关闭池。请参阅切换池实现部分以了解相关背景信息。但是，请注意，当出现此错误消息时，**总是**由应用程序本身的更大问题引起；池只是帮助更快地暴露问题。

另请参见

连接池

使用引擎和连接  ### 池类不能与 asyncio 引擎一起使用（反之亦然）

`QueuePool`池类在内部使用`thread.Lock`对象，并且与 asyncio 不兼容。如果使用`create_async_engine()`函数创建`AsyncEngine`，则适当的队列池类是`AsyncAdaptedQueuePool`，它会自动使用，无需指定。

除了`AsyncAdaptedQueuePool`之外，`NullPool`和`StaticPool`池类不使用锁，并且也适用于与异步引擎一起使用。

在极少数情况下，如果使用`create_engine()`函数显式指定了`AsyncAdaptedQueuePool`池类，则也会引发此错误。

另请参见

连接池  ### 在无效事务回滚之前无法重新连接。请在继续之前完全回滚()

这个错误条件指的是当一个`Connection`无效时，无论是因为数据库断开检测还是因为显式调用`Connection.invalidate()`，但仍然存在一个事务，该事务由`Connection.begin()`方法明确启动，或者由于连接在发出任何 SQL 语句时自动开始一个事务，如 SQLAlchemy 2.x 系列中发生的情况。当连接无效时，任何正在进行中的`Transaction`现在处于无效状态，必须明确回滚才能将其从`Connection`中移除。### QueuePool 大小为<x>的限制溢出<y>，连接超时，超时<z>

这可能是最常见的运行时错误，因为它直接涉及应用程序的工作负载超过了配置的限制，这个限制通常适用于几乎所有的 SQLAlchemy 应用程序。

以下几点总结了这个错误的含义，从大多数 SQLAlchemy 用户应该已经熟悉的最基本的点开始。

+   **SQLAlchemy Engine 对象默认使用连接池** - 这意味着当一个`Engine`对象的 SQL 数据库连接资源被使用，并且释放了该资源后，数据库连接本身仍然保持连接状态，并返回到一个内部队列中，可以再次使用。尽管代码可能看起来已经结束了与数据库的交互，但在许多情况下，应用程序仍将保持一定数量的数据库连接，直到应用程序结束或显式释放池为止。

+   由于连接池，当应用程序使用 SQL 数据库连接时，通常是通过使用`Engine.connect()`或通过使用 ORM `Session`进行查询时，这个活动并不一定在获取连接对象时立即建立到数据库的新连接；它实际上是向连接池查询连接，这通常会从池中检索一个现有的连接以便重新使用。如果没有可用的连接，池将创建一个新的数据库连接，但只有在池没有超过配置容量时才会这样做。

+   在大多数情况下使用的默认池称为`QueuePool`。当您要求此池提供连接，而没有可用连接时，它将创建一个新连接**如果当前使用的连接总数少于配置值**。这个值等于**池大小加上最大溢出**。这意味着如果您已将引擎配置为：

    ```py
    engine = create_engine("mysql+mysqldb://u:p@host/db", pool_size=10, max_overflow=20)
    ```

    上述`Engine`允许**最多 30 个连接**同时存在，不包括已从引擎分离或失效的连接。如果请求新连接，并且应用程序的其他部分已使用 30 个连接，连接池将阻塞一段固定时间，然后超时并引发此错误消息。

    为了允许更多连接同时使用，池可以使用传递给`create_engine()`函数的`create_engine.pool_size`和`create_engine.max_overflow`参数进行调整。等待可用连接的超时时间由`create_engine.pool_timeout`参数配置。

+   通过将`create_engine.max_overflow`设置为值“-1”，可以配置池具有无限溢出。使用此设置，池仍将维护一组固定的连接，但如果请求新连接时没有可用连接，它将无条件地创建一个新连接。

    然而，在这种方式下运行时，如果应用程序存在使用所有可用连接资源的问题，则最终会达到数据库本身可用连接的配置限制，这将再次返回错误。更严重的是，当应用程序耗尽数据库连接时，通常会在失败之前使用大量资源，并且还可能干扰其他依赖于能够连接到数据库的应用程序和数据库状态机制。

    鉴于上述情况，连接池可以被视为连接使用的**安全阀**，提供了对于恶意应用程序使整个数据库对于所有其他应用程序不可用的关键保护层。当收到此错误消息时，最好修复使用太多连接和/或适当配置限制的问题，而不是允许无限溢出，这实际上并没有解决潜在问题。

什么原因会导致应用程序耗尽所有可用的连接？

+   **应用程序正在处理太多并发请求以执行基于池配置的工作** - 这是最直接的原因。如果您有一个运行在允许 30 个并发线程的线程池中的应用程序，每个线程使用一个连接，并且如果您的池没有配置为允许至少同时检出 30 个连接，则在您的应用程序接收到足够的并发请求时，您将收到此错误。解决方法是提高池的限制或降低并发线程的数量。

+   **应用程序未将连接返回到池中** - 这是接下来最常见的原因，即应用程序正在使用连接池，但程序未能释放这些连接，而是保持它们处于打开状态。连接池以及 ORM `Session`确实具有逻辑，使得当会话和/或连接对象被垃圾回收时，导致底层连接资源被释放，但不能依赖此行为及时释放资源。

    这种情况经常发生的原因是应用程序使用 ORM 会话，但在完成涉及该会话的工作后未调用`Session.close()`。解决方法是确保 ORM 会话（如果使用 ORM）或绑定到引擎的`Connection`对象（如果使用 Core）在完成工作后明确关闭，可以通过适当的`.close()`方法或使用其中一个可用的上下文管理器（例如“with:”语句）来正确释放资源。

+   **应用程序正试图运行长时间事务** - 数据库事务是一种非常昂贵的资源，不应该**空闲等待某个事件发生**。如果应用程序正在等待用户按下按钮，或者等待长时间作业队列的结果，或者保持持久连接打开以与浏览器交互，**不要保持数据库事务始终处于打开状态**。当应用程序需要与数据库一起工作并与事件交互时，在那一点上打开一个短暂的事务，然后关闭它。

+   **应用程序发生死锁** - 这也是此错误的常见原因，更难以理解。如果应用程序由于应用程序端或数据库端的死锁而无法完成其对连接的使用，那么应用程序可能会耗尽所有可用连接，然后导致其他请求收到此错误。死锁的原因包括：

    +   如果没有正确地对所有套接字库和驱动程序进行猴子补丁，或者对所有猴子补丁驱动程序方法的覆盖不完全，或者在异步系统正在用于 CPU 密集型工作负载并且使用数据库资源的绿色线程等待太长时间时，则使用隐式异步系统，例如 gevent 或 eventlet，会出现问题。通常，隐式或显式异步编程框架通常对绝大多数关系数据库操作都不是必要的或合适的；如果应用程序必须对某些功能区域使用异步系统，则最好是数据库导向的业务方法在传统线程中运行，然后将消息传递到应用程序的异步部分。

    +   数据库端发生死锁，例如行相互死锁

    +   线程错误，例如互斥体在相互死锁，或在同一线程中调用已锁定的互斥体

请记住，除了使用池化技术的替代方法是完全关闭池化技术。请参阅切换池实现部分了解背景信息。但是，请注意，当出现此错误消息时，通常是由于应用程序本身存在更大的问题；池仅帮助更早地暴露问题。

另请参阅

连接池

与引擎和连接一起工作

### 无法将 Pool 类与 asyncio 引擎一起使用（反之亦然）

`QueuePool` 池类在内部使用 `thread.Lock` 对象，并且与 asyncio 不兼容。如果使用 `create_async_engine()` 函数创建 `AsyncEngine`，则适当的队列池类是 `AsyncAdaptedQueuePool`，它会自动使用，无需指定。

除了 `AsyncAdaptedQueuePool`，`NullPool` 和 `StaticPool` 池类不使用锁，并且也适用于与异步引擎一起使用。

在极少数情况下，如果使用 `create_engine()` 函数明确指定 `AsyncAdaptedQueuePool` 池类，则还会引发此错误。

另请参阅

连接池

### 无法重新连接直到无效事务被完全回滚。请在继续之前完全回滚()

此错误条件指的是 `Connection` 被作废的情况，可能是由于检测到数据库断开连接或由于显式调用 `Connection.invalidate()`，但仍然存在已经由 `Connection.begin()` 方法显式启动的事务，或者由于连接在发出任何 SQL 语句时自动开始事务，这在 SQLAlchemy 的 2.x 系列中发生。当连接被作废时，任何正在进行的 `Transaction` 现在处于无效状态，必须显式回滚以将其从 `Connection` 中移除。

## DBAPI 错误

Python 数据库 API，或 DBAPI，是一种用于数据库驱动程序的规范，可以在 [Pep-249](https://www.python.org/dev/peps/pep-0249/) 找到。此 API 指定了一组异常类，以适应数据库的所有故障模式。

SQLAlchemy 不会直接生成这些异常。相反，它们是从数据库驱动程序拦截并由 SQLAlchemy 提供的异常 `DBAPIError` 包装的，但异常中的消息是**由驱动程序生成的，而不是 SQLAlchemy**。

### InterfaceError

与数据库接口而不是数据库本身相关的错误引发的异常。

此错误是 DBAPI 错误 ，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`InterfaceError` 有时会由驱动程序在数据库连接断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参阅 处理断开连接 部分。  ### DatabaseError

与数据库本身相关而不是接口或传递的数据的错误引发的异常。

此错误是 DBAPI 错误 ，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。  ### DataError

由于处理的数据问题引发的异常，例如除以零，数值超出范围等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。  ### OperationalError

与数据库操作相关的错误引发的异常，不一定在程序员控制之下，例如出现意外断开连接，找不到数据源名称，无法处理事务，处理过程中发生内存分配错误等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`OperationalError` 是由驱动程序在数据库连接断开或无法连接到数据库的情况下使用的最常见（但不是唯一）错误类别。有关如何处理此问题的提示，请参阅处理断开连接部分。  ### IntegrityError

当数据库的关系完整性受到影响时引发的异常，例如外键检查失败。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。  ### InternalError

当数据库遇到内部错误时引发的异常，例如游标不再有效，事务不同步等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`InternalError` 有时会由驱动程序在数据库连接断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参阅处理断开连接部分。  ### ProgrammingError

由于编程错误引发的异常，例如未找到表或已存在，SQL 语句中的语法错误，指定的参数数量错误等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`ProgrammingError` 有时会由驱动程序在数据库连接断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参阅处理断开连接部分。  ### NotSupportedError

当使用数据库不支持的方法或数据库 API 时引发的异常，例如在不支持事务或已关闭事务的连接上请求`.rollback()`。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。  ### InterfaceError

与数据库本身而不是数据库接口相关的错误引发的异常。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`InterfaceError`有时由驱动程序在数据库连接断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参见处理断开连接部分。

### DatabaseError

由于与数据库本身相关的错误而引发的异常，而不是与传递的接口或数据相关。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

### DataError

由于处理数据的问题（例如除零，数值超出范围等）引发的错误。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

### OperationalError

由于与数据库操作相关的错误而引发的异常，不一定在程序员的控制之下，例如发生意外断开连接，数据源名称未找到，无法处理事务，处理过程中发生内存分配错误等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`OperationalError`是驱动程序在数据库连接断开或无法连接到数据库的情况下最常见（但不是唯一）使用的错误类。有关如何处理此问题的提示，请参见处理断开连接部分。

### IntegrityError

当数据库的关系完整性受到影响时引发异常，例如外键检查失败。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

### InternalError

当数据库遇到内部错误时引发异常，例如游标不再有效，事务不同步等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`InternalError`有时由驱动程序在数据库连接断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参见处理断开连接部分。

### ProgrammingError

由于编程错误而引发的异常，例如表未找到或已存在，在 SQL 语句中存在语法错误，指定的参数数量错误等。

此错误是 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

`ProgrammingError`有时由驱动程序在数据库连接断开或无法连接到数据库的情况下引发。有关如何处理此问题的提示，请参见处理断开连接部分。

### NotSupportedError

在使用数据库不支持的方法或数据库 API 时引发异常，例如在不支持事务或已关闭事务的连接上请求 .rollback()。

这个错误是一个 DBAPI 错误，源自数据库驱动程序（DBAPI），而不是 SQLAlchemy 本身。

## SQL 表达语言

### 对象不会生成缓存键，性能影响

SQLAlchemy 从版本 1.4 开始包含一个 SQL 编译缓存设施，它允许 Core 和 ORM SQL 构造缓存它们的字符串形式，以及用于从语句中获取结果的其他结构信息，当下次使用另一个结构上等效的构造时，可以跳过相对昂贵的字符串编译过程。该系统依赖于为所有 SQL 构造实现的功能，包括对象如`Column`、`select()`和`TypeEngine`对象，以生成完全代表它们状态的**缓存键**，以影响 SQL 编译过程。

如果问题中的警告涉及到广泛使用的对象，如`Column`对象，并且显示影响到发出的大多数 SQL 构造（使用估算缓存性能使用日志记录描述的估算技术），以至于缓存通常不会为应用程序启用，这将对性能产生负面影响，并且在某些情况下，与之前的 SQLAlchemy 版本相比，实际上会产生**性能下降**。在为什么升级到 1.4 和/或 2.x 后我的应用程序变慢？的常见问题解答中详细介绍了这一点。

#### 如果有任何疑问，缓存会自行禁用。

缓存依赖于能够以**一致**的方式生成准确代表语句**完整结构**的缓存键。如果特定的 SQL 构造（或类型）没有适当的指令，允许它生成正确的缓存键，那么不能安全地启用缓存：

+   缓存键必须代表**完整结构**：如果使用两个不同实例的构造可能导致渲染不同 SQL，那么针对第一个元素使用不捕捉第一和第二元素之间不同之处的缓存键缓存 SQL，将导致第二个实例渲染出错误的 SQL。

+   缓存键必须是**一致的**：如果一个构造表示每次都会改变的状态，比如字面值，为每个实例生成唯一的 SQL，那么这个构造也不安全可缓存，因为重复使用构造将快速填满语句缓存，其中包含可能不会再次使用的唯一 SQL 字符串，从而破坏了缓存的目的。

由于上述两个原因，SQLAlchemy 的缓存系统对于决定是否缓存与对象对应的 SQL 是**非常保守**的。

#### 缓存断言属性

根据以下标准发出警告。有关每个标准的详细信息，请参见 为什么升级到 1.4 和/或 2.x 后我的应用程序变慢了？。

+   `Dialect` 本身（即由我们传递给 `create_engine()` 的 URL 的第一部分指定的模块，如 `postgresql+psycopg2://`），必须指示已经审查和测试以正确支持缓存，这由 `Dialect.supports_statement_cache` 属性设置为 `True` 来表示。在使用第三方方言时，请咨询方言的维护人员，以便他们遵循确保可以启用缓存的步骤并发布新版本。

+   第三方或用户定义的类型，它们继承自`TypeDecorator` 或 `UserDefinedType`，必须在其定义中包含 `ExternalType.cache_ok` 属性，包括所有派生子类，遵循`ExternalType.cache_ok`的文档字符串中描述的准则。与以前一样，如果这些数据类型是从第三方库导入的，请咨询该库的维护人员，以便他们提供必要的更改并发布新版本。

+   第三方或用户定义的 SQL 构造，从类似 `ClauseElement`、`Column`、`Insert` 等类继承，包括简单的子类以及设计用于与自定义 SQL 构造和编译扩展一起工作的那些，通常应包括 `HasCacheKey.inherit_cache` 属性设置为 `True` 或 `False`，根据构造的设计，遵循在为自定义构造启用缓存支持中描述的准则。

另请参阅

使用日志估算缓存性能 - 关于观察缓存行为和效率的背景知识

为什么升级到 1.4 和/或 2.x 后我的应用程序变慢了？ - 在常见问题解答部分  ### Compiler StrSQLCompiler 无法渲染类型为 <element type> 的元素

这个错误通常发生在尝试将包含不属于默认编译的元素的 SQL 表达式构造转换为字符串时；在这种情况下，错误将针对 `StrSQLCompiler` 类。在较少见的情况下，当使用错误类型的 SQL 表达式与特定类型的数据库后端一起使用时，也会发生这种情况；在这些情况下，将命名其他类型的 SQL 编译器类，如 `SQLCompiler` 或 `sqlalchemy.dialects.postgresql.PGCompiler`。以下指导更具体地针对“字符串化”用例，但也描述了一般背景。

通常，核心 SQL 构造或 ORM `Query` 对象可以直接转换为字符串，例如当我们使用 `print()` 时：

```py
>>> from sqlalchemy import column
>>> print(column("x") == 5)
x  =  :x_1 
```

当上述 SQL 表达式被转换为字符串时，将使用 `StrSQLCompiler` 编译器类，这是一个特殊的语句编译器，当一个构造被转换为字符串时，没有任何特定于方言的信息时会被调用。

然而，有许多构造是特定于某种数据库方言的，对于这些构造，`StrSQLCompiler` 不知道如何转换为字符串，例如 PostgreSQL 的 “insert on conflict” 构造：

```py
>>> from sqlalchemy.dialects.postgresql import insert
>>> from sqlalchemy import table, column
>>> my_table = table("my_table", column("x"), column("y"))
>>> insert_stmt = insert(my_table).values(x="foo")
>>> insert_stmt = insert_stmt.on_conflict_do_nothing(index_elements=["y"])
>>> print(insert_stmt)
Traceback (most recent call last):

...

sqlalchemy.exc.UnsupportedCompilationError:
Compiler <sqlalchemy.sql.compiler.StrSQLCompiler object at 0x7f04fc17e320>
can't render element of type
<class 'sqlalchemy.dialects.postgresql.dml.OnConflictDoNothing'>
```

为了将特定于特定后端的结构字符串化，必须使用`ClauseElement.compile()`方法，传递一个`Engine`或一个`Dialect`对象，该对象将调用正确的编译器。下面我们使用了一个 PostgreSQL 方言：

```py
>>> from sqlalchemy.dialects import postgresql
>>> print(insert_stmt.compile(dialect=postgresql.dialect()))
INSERT  INTO  my_table  (x)  VALUES  (%(x)s)  ON  CONFLICT  (y)  DO  NOTHING 
```

对于 ORM `Query` 对象，可以使用 `Query.statement` 访问器访问语句：

```py
statement = query.statement
print(statement.compile(dialect=postgresql.dialect()))
```

请查看下方的常见问题解答链接，了解关于直接字符串化/编译 SQL 元素的额外细节。

请参阅

如何将 SQL 表达式呈现为字符串，可能会内联绑定参数？

### TypeError: `<operator>`不支持‘ColumnProperty’和`<something>`实例之间的操作

当尝试在 SQL 表达式的上下文中使用 `column_property()` 或 `deferred()` 对象时，通常会发生这种情况，通常是在声明性的上下文中，如下所示：

```py
class Bar(Base):
    __tablename__ = "bar"

    id = Column(Integer, primary_key=True)
    cprop = deferred(Column(Integer))

    __table_args__ = (CheckConstraint(cprop > 5),)
```

上面，`cprop` 属性在映射之前内联使用，但是这个 `cprop` 属性不是一个`Column`，它是一个`ColumnProperty`，这是一个临时对象，因此它没有 `Column` 对象或 `InstrumentedAttribute` 对象的全部功能，一旦声明过程完成，它将被映射到 `Bar` 类上。

虽然 `ColumnProperty` 有一个 `__clause_element__()` 方法，它允许它在某些面向列的上下文中工作，但它不能在上面示例中所示的开放式比较上下文中工作，因为它没有 Python `__eq__()` 方法，该方法允许它将与数字“5”的比较解释为 SQL 表达式而不是常规 Python 比较。

解决方案是直接使用`Column`访问`ColumnProperty.expression`属性：

```py
class Bar(Base):
    __tablename__ = "bar"

    id = Column(Integer, primary_key=True)
    cprop = deferred(Column(Integer))

    __table_args__ = (CheckConstraint(cprop.expression > 5),)
```

### 绑定参数<x>（在参数组<y>中）需要一个值

当语句隐式或显式地使用`bindparam()`，并且在执行语句时没有提供值时，会发生此错误：

```py
stmt = select(table.c.column).where(table.c.id == bindparam("my_param"))

result = conn.execute(stmt)
```

在上述示例中，没有为参数“my_param”提供值。正确的方法是提供一个值：

```py
result = conn.execute(stmt, {"my_param": 12})
```

当消息采用“在参数组< y >中需要为绑定参数< x >提供值”的形式时，消息是指执行的“executemany”风格。在这种情况下，语句通常是 INSERT、UPDATE 或 DELETE，并且正在传递参数列表。在这种格式中，语句可以动态生成，以包含参数列表中提供的每个参数的参数位置，它将使用**第一组参数**来确定这些参数应该是什么。

例如，下面的语句是基于第一个参数集计算的，需要参数“a”、“b”和“c” - 这些名称确定了语句的最终字符串格式，该格式将用于列表中的每个参数集。由于第二个条目不包含“b”，因此会生成此错误：

```py
m = MetaData()
t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))

e.execute(
    t.insert(),
    [
        {"a": 1, "b": 2, "c": 3},
        {"a": 2, "c": 4},
        {"a": 3, "b": 4, "c": 5},
    ],
)
```

```py
sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError)
A value is required for bind parameter 'b', in parameter group 1
[SQL: u'INSERT INTO t (a, b, c) VALUES (?, ?, ?)']
[parameters: [{'a': 1, 'c': 3, 'b': 2}, {'a': 2, 'c': 4}, {'a': 3, 'c': 5, 'b': 4}]]
```

由于“b”是必需的，因此将其传递为`None`，以便进行 INSERT 操作：

```py
e.execute(
    t.insert(),
    [
        {"a": 1, "b": 2, "c": 3},
        {"a": 2, "b": None, "c": 4},
        {"a": 3, "b": 4, "c": 5},
    ],
)
```

请参阅

发送参数 ### 期望的 FROM 子句，得到 Select。要创建 FROM 子句，请使用 .subquery() 方法

这是 SQLAlchemy 1.4 所做的更改，其中通过诸如`select()`之类的函数生成的 SELECT 语句，但也包括联合和文本 SELECT 表达式等内容不再被视为`FromClause`对象，并且不能直接放置在另一个 SELECT 语句的 FROM 子句中，而不是首先将它们包装在`Subquery`中。这是核心中的一个重大概念性变化，完整的原理在 SELECT 语句不再隐式视为 FROM 子句中讨论。

给出一个示例：

```py
m = MetaData()
t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))
stmt = select(t)
```

在上述示例中，`stmt`表示一个 SELECT 语句。当我们想要直接将`stmt`作为另一个 SELECT 语句的 FROM 子句时，比如如果我们试图从中选择：

```py
new_stmt_1 = select(stmt)
```

或者如果我们想在 FROM 子句中使用它，比如在 JOIN 中：

```py
new_stmt_2 = select(some_table).select_from(some_table.join(stmt))
```

在 SQLAlchemy 的早期版本中，使用一个 SELECT 语句在另一个 SELECT 语句内会产生一个有括号的无名称子查询。在大多数情况下，这种形式的 SQL 不是很有用，因为像 MySQL 和 PostgreSQL 这样的数据库要求 FROM 子句中的子查询具有命名别名，这意味着需要使用`SelectBase.alias()`方法或者从 1.4 版本开始使用`SelectBase.subquery()`方法来产生这个。在其他数据库中，子查询有一个名称来解析子查询内部列名的任何歧义仍然更清晰。

除了上述实际原因外，还有很多其他与 SQLAlchemy 相关的原因导致进行此更改。因此，上述两个语句的正确形式要求使用`SelectBase.subquery()`：

```py
subq = stmt.subquery()

new_stmt_1 = select(subq)

new_stmt_2 = select(some_table).select_from(some_table.join(subq))
```

另请参阅

SELECT 语句不再隐式地被视为 FROM 子句  ### 为原始 clauseelement 自动生成别名

版本 1.4.26 中新增。

这个弃用警告指的是一个非常古老且可能不太为人所知的模式，适用于传统的`Query.join()`方法以及 2.0 风格的`Select.join()`方法，其中联接可以根据`relationship()`来表示，但目标是将类映射到的`Table`或其他核心可选择项，而不是 ORM 实体，比如映射类或`aliased()`构造：

```py
a1 = Address.__table__

q = (
    s.query(User)
    .join(a1, User.addresses)
    .filter(Address.email_address == "ed@foo.com")
    .all()
)
```

上述模式还允许任意可选择项，比如核心`Join`或`Alias`对象，但是没有对此元素的自动适应，这意味着需要直接引用核心元素：

```py
a1 = Address.__table__.alias()

q = (
    s.query(User)
    .join(a1, User.addresses)
    .filter(a1.c.email_address == "ed@foo.com")
    .all()
)
```

指定联接目标的正确方法始终是使用映射类本身或一个`aliased`对象，后者使用`PropComparator.of_type()`修饰符来设置一个别名：

```py
# normal join to relationship entity
q = s.query(User).join(User.addresses).filter(Address.email_address == "ed@foo.com")

# name Address target explicitly, not necessary but legal
q = (
    s.query(User)
    .join(Address, User.addresses)
    .filter(Address.email_address == "ed@foo.com")
)
```

加入到别名：

```py
from sqlalchemy.orm import aliased

a1 = aliased(Address)

# of_type() form; recommended
q = (
    s.query(User)
    .join(User.addresses.of_type(a1))
    .filter(a1.email_address == "ed@foo.com")
)

# target, onclause form
q = s.query(User).join(a1, User.addresses).filter(a1.email_address == "ed@foo.com")
```  ### 由于重叠表而自动生成别名

新版本为 1.4.26。

此警告通常是在使用`Select.join()`方法或传统的`Query.join()`方法进行查询时生成的，其中涉及到涉及连接表继承的映射。问题在于，在两个共享共同基表的连接继承模型之间进行连接时，如果不对其中一个或另一个应用别名，就无法形成两个实体之间的适当 SQL JOIN；SQLAlchemy 将别名应用于连接的右侧。例如，考虑到连接继承映射：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    manager_id = Column(ForeignKey("manager.id"))
    name = Column(String(50))
    type = Column(String(50))

    reports_to = relationship("Manager", foreign_keys=manager_id)

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "inherit_condition": id == Employee.id,
    }
```

上述映射包括`Employee`和`Manager`类之间的关系。由于两个类都使用“employee”数据库表，从 SQL 的角度来看，这是一种自引用关系。如果我们想要使用连接从`Employee`和`Manager`模型中查询，SQL 级别上“employee”表需要在查询中包含两次，这意味着它必须被别名化。当我们使用 SQLAlchemy ORM 创建这样的连接时，我们得到的 SQL 看起来像下面这样：

```py
>>> stmt = select(Employee, Manager).join(Employee.reports_to)
>>> print(stmt)
SELECT  employee.id,  employee.manager_id,  employee.name,
employee.type,  manager_1.id  AS  id_1,  employee_1.id  AS  id_2,
employee_1.manager_id  AS  manager_id_1,  employee_1.name  AS  name_1,
employee_1.type  AS  type_1
FROM  employee  JOIN
(employee  AS  employee_1  JOIN  manager  AS  manager_1  ON  manager_1.id  =  employee_1.id)
ON  manager_1.id  =  employee.manager_id 
```

上面的 SQL 从`employee`表中选择，表示查询中的`Employee`实体。然后加入到`employee AS employee_1 JOIN manager AS manager_1`的右嵌套连接，其中再次声明了`employee`表，但作为匿名别名`employee_1`。这就是警告消息所指的“自动生成别名”。

当 SQLAlchemy 加载包含`Employee`和`Manager`对象的 ORM 行时，ORM 必须将来自上述`employee_1`和`manager_1`表别名的行适应为未别名化的`Manager`类的行。这个过程在内部是复杂的，并且不支持所有 API 功能，特别是当尝试在比这里展示的更深度嵌套查询中使用急加载功能时，如`contains_eager()`。由于该模式对于更复杂的情况不可靠，并涉及难以预测和遵循的隐式决策，因此会发出警告，并且该模式可能被视为传统功能。编写此查询的更好方法是使用适用于任何其他自引用关系的相同模式，即显式使用`aliased()`构造。对于联接继承和其他基于联接的映射，通常希望添加使用`aliased.flat`参数，这将允许通过将别名应用于联接中的各个表来对两个或更多表进行联接别名化，而不是将联接嵌入到新的子查询中：

```py
>>> from sqlalchemy.orm import aliased
>>> manager_alias = aliased(Manager, flat=True)
>>> stmt = select(Employee, manager_alias).join(Employee.reports_to.of_type(manager_alias))
>>> print(stmt)
SELECT  employee.id,  employee.manager_id,  employee.name,
employee.type,  manager_1.id  AS  id_1,  employee_1.id  AS  id_2,
employee_1.manager_id  AS  manager_id_1,  employee_1.name  AS  name_1,
employee_1.type  AS  type_1
FROM  employee  JOIN
(employee  AS  employee_1  JOIN  manager  AS  manager_1  ON  manager_1.id  =  employee_1.id)
ON  manager_1.id  =  employee.manager_id 
```

如果我们想要使用`contains_eager()`来填充`reports_to`属性，我们将引用别名：

```py
>>> stmt = (
...     select(Employee)
...     .join(Employee.reports_to.of_type(manager_alias))
...     .options(contains_eager(Employee.reports_to.of_type(manager_alias)))
... )
```

在某些更嵌套的情况下，如果没有使用显式的`aliased()`对象，在 ORM 在非常嵌套的上下文中“自动别名化”的情况下，`contains_eager()`选项可能没有足够的上下文来知道从哪里获取其数据。因此，最好不要依赖此功能，而是尽可能保持 SQL 构造尽可能明确。###对象不会生成缓存键，性能影响

截至版本 1.4，SQLAlchemy 包括一个 SQL 编译缓存设施，它允许 Core 和 ORM SQL 构造缓存它们的字符串形式，以及用于从语句中获取结果的其他结构信息，这样当下次使用另一个结构等效的构造时，就可以跳过相对昂贵的字符串编译过程。该系统依赖于为所有 SQL 构造实现的功能，包括诸如`Column`、`select()`和`TypeEngine`对象等对象，以生成完全代表它们状态的**缓存键**，以至于影响 SQL 编译过程。

如果警告涉及到广泛使用的对象，比如`Column`对象，并且显示为影响到大部分发出的 SQL 构造（使用通过日志估算缓存性能描述的估算技术）以至于缓存通常不会为应用程序启用，这将对性能产生负面影响，并且在某些情况下，与之前的 SQLAlchemy 版本相比实际上会产生**性能下降**。在为什么升级到 1.4 和/或 2.x 后我的应用程序变慢了？的常见问题解答中详细介绍了这一点。

#### 如果存在任何疑问，缓存会自行禁用

缓存依赖于能够生成一个缓存键，以一种**一致的**方式准确地表示语句的**完整结构**。如果某个特定的 SQL 构造（或类型）没有适当的指令来生成正确的缓存键，那么就不能安全地启用缓存：

+   缓存键必须表示**完整的结构**：如果两个单独实例的使用可能导致呈现不同 SQL，则针对第一个元素使用一个不捕获第一和第二个元素之间不同之处的缓存键缓存 SQL，将导致为第二个实例缓存并呈现不正确的 SQL。

+   缓存键必须是**一致的**：如果一个构造代表每次都会更改的状态，比如文字值，为每个实例产生唯一的 SQL，那么这个构造也不安全可以缓存，因为重复使用这个构造将很快填满语句缓存，里面包含的唯一 SQL 字符串可能不会再次使用，从而使缓存失去了意义。

由于上述两个原因，SQLAlchemy 的缓存系统在决定是否缓存与对象对应的 SQL 时**极其保守**。

#### 缓存的断言属性

根据以下标准发出警告。有关每个标准的更多详细信息，请参阅升级到 1.4 和/或 2.x 后为什么我的应用程序变慢？部分。

+   `Dialect`本身（即由我们传递给`create_engine()`的 URL 的第一部分指定的模块，如`postgresql+psycopg2://`），必须指示已经审查并测试以正确支持缓存，这由`Dialect.supports_statement_cache`属性设置为`True`来指示。在使用第三方方言时，请与方言的维护者协商，以便他们遵循确保可以启用缓存的步骤并发布新版本。

+   第三方或用户定义的类型，继承自`TypeDecorator`或`UserDefinedType`，必须在其定义中包含`ExternalType.cache_ok`属性，包括所有派生子类，在`ExternalType.cache_ok`的文档字符串中描述的指南中遵循。同样，如果这些数据类型是从第三方库导入的，请与该库的维护者协商，以便他们提供必要的更改并发布新版本。

+   第三方或用户定义的 SQL 构造，从诸如`ClauseElement`、`Column`、`Insert`等类继承，包括简单的子类以及设计用于与自定义 SQL 构造和编译扩展一起工作的子类，通常应包含`HasCacheKey.inherit_cache`属性设置为`True`或`False`，根据构造的设计，在为自定义构造启用缓存支持中描述的指南。 

另请参阅

使用日志估算缓存性能 - 观察缓存行为和效率的背景

为什么升级到 1.4 和/或 2.x 后我的应用程序变慢了？ - 在常见问题部分

#### 如果有任何疑问，缓存会自行禁用

缓存依赖于能够生成准确代表语句的**完整结构**的缓存键，以一致的方式。如果特定的 SQL 结构（或类型）没有适当的指令来允许其生成正确的缓存键，则不能安全地启用缓存：

+   缓存键必须代表**完整结构**：如果使用两个不同实例的结构可能导致渲染不同的 SQL，则使用不捕获第一个和第二个元素之间不同之处的缓存键对第一个元素的 SQL 进行缓存将导致第二个实例的 SQL 被错误地缓存和渲染。

+   缓存键必须是**一致的**：如果一个结构代表每次都会改变的状态，比如字面值，为每个实例生成唯一的 SQL，那么这个结构也不能安全地缓存，因为对该结构的重复使用将迅速用唯一的 SQL 字符串填满语句缓存，这些字符串可能不会再次使用，从而打破缓存的目的。

由于上述两个原因，SQLAlchemy 的缓存系统对于决定是否缓存与对象对应的 SQL 是**极端保守的**。

#### 缓存的断言属性

根据以下标准发出警告。有关每个标准的更多详细信息，请参见 为什么升级到 1.4 和/或 2.x 后我的应用程序变慢了？一节。

+   `Dialect` 本身（即我们传递给 `create_engine()` 的 URL 的第一部分指定的模块，如 `postgresql+psycopg2://`），必须指示它已经经过审查和测试，以正确支持缓存，这由 `Dialect.supports_statement_cache` 属性设置为 `True` 来表示。使用第三方方言时，请咨询方言的维护者，以便他们可以按照确保可以启用缓存的步骤进行操作，并发布一个新的版本。

+   从 `TypeDecorator` 或 `UserDefinedType` 继承的第三方或用户定义的类型必须在其定义中包含 `ExternalType.cache_ok` 属性，包括所有派生子类，在外部类型缓存支持 的文档字符串中描述的准则。与以前一样，如果这些数据类型是从第三方库导入的，请咨询该库的维护者，以便他们提供必要的更改并发布新版本。

+   第三方或用户定义的 SQL 构造，它们从类中子类化，如`ClauseElement`，`Column`，`Insert` 等，包括简单的子类以及那些设计用于与 自定义 SQL 构造和编译扩展一起工作的子类，通常应该将`HasCacheKey.inherit_cache` 属性设置为 `True` 或 `False`，根据构造的设计，遵循在 启用自定义构造的缓存支持 中描述的准则。

参见

使用日志估算缓存性能 - 观察缓存行为和效率的背景知识

为什么我的应用程序在升级到 1.4 和/或 2.x 后变慢？ - 在常见问题解答部分

### 编译器 StrSQLCompiler 无法渲染类型为 <element type> 的元素

当尝试将包含不属于默认编译的元素的 SQL 表达式构造进行字符串化时，通常会发生此错误；在这种情况下，错误将针对`StrSQLCompiler` 类。在较少见的情况下，当使用错误类型的 SQL 表达式与特定类型的数据库后端时，也可能发生这种情况；在这些情况下，将命名其他类型的 SQL 编译器类，例如 `SQLCompiler` 或 `sqlalchemy.dialects.postgresql.PGCompiler`。下面的指导更具体地针对“字符串化”用例，但也描述了一般背景。

通常，Core SQL 构造或 ORM `Query`对象可以直接转换为字符串，比如当我们使用`print()`时：

```py
>>> from sqlalchemy import column
>>> print(column("x") == 5)
x  =  :x_1 
```

当上述 SQL 表达式被字符串化时，将使用`StrSQLCompiler` 编译器类，这是一个特殊的语句编译器，当一个构造在没有任何特定于方言的信息的情况下被字符串化时会被调用。

然而，有许多构造是特定于某种数据库方言的，对于这些构造，`StrSQLCompiler` 不知道如何转换为字符串，比如 PostgreSQL 的“insert on conflict”构造：

```py
>>> from sqlalchemy.dialects.postgresql import insert
>>> from sqlalchemy import table, column
>>> my_table = table("my_table", column("x"), column("y"))
>>> insert_stmt = insert(my_table).values(x="foo")
>>> insert_stmt = insert_stmt.on_conflict_do_nothing(index_elements=["y"])
>>> print(insert_stmt)
Traceback (most recent call last):

...

sqlalchemy.exc.UnsupportedCompilationError:
Compiler <sqlalchemy.sql.compiler.StrSQLCompiler object at 0x7f04fc17e320>
can't render element of type
<class 'sqlalchemy.dialects.postgresql.dml.OnConflictDoNothing'>
```

为了将特定于特定后端的构造转换为字符串，必须使用`ClauseElement.compile()` 方法，传递一个 `Engine` 或一个 `Dialect` 对象，这将调用正确的编译器。下面我们使用一个 PostgreSQL 方言：

```py
>>> from sqlalchemy.dialects import postgresql
>>> print(insert_stmt.compile(dialect=postgresql.dialect()))
INSERT  INTO  my_table  (x)  VALUES  (%(x)s)  ON  CONFLICT  (y)  DO  NOTHING 
```

对于 ORM `Query` 对象，可以通过`Query.statement`访问器访问语句：

```py
statement = query.statement
print(statement.compile(dialect=postgresql.dialect()))
```

请查看下面的 FAQ 链接，了解有关 SQL 元素的直接字符串化/编译的更多详细信息。

另请参阅

如何将 SQL 表达式呈现为字符串，可能包含内联的绑定参数？

### TypeError: <operator> not supported between instances of ‘ColumnProperty’ and <something>

当尝试在 SQL 表达式的上下文中使用`column_property()`或`deferred()`对象时，通常在声明中会出现这种情况：

```py
class Bar(Base):
    __tablename__ = "bar"

    id = Column(Integer, primary_key=True)
    cprop = deferred(Column(Integer))

    __table_args__ = (CheckConstraint(cprop > 5),)
```

在上面的例子中，在映射之前内联使用了`cprop`属性，但是这个`cprop`属性不是一个`Column`，它是一个`ColumnProperty`，这是一个临时对象，因此不具备`Column`对象或`InstrumentedAttribute`对象的全部功能，一旦声明过程完成，它将映射到`Bar`类上。

虽然`ColumnProperty`确实具有`__clause_element__()`方法，允许它在某些面向列的上下文中工作，但是它无法在开放式比较上下文中工作，如上所示，因为它没有 Python `__eq__()` 方法，该方法将允许它将对数字“5”的比较解释为 SQL 表达式而不是常规的 Python 比较。

解决方案是直接访问`Column`，使用`ColumnProperty.expression` 属性：

```py
class Bar(Base):
    __tablename__ = "bar"

    id = Column(Integer, primary_key=True)
    cprop = deferred(Column(Integer))

    __table_args__ = (CheckConstraint(cprop.expression > 5),)
```

### 绑定参数<x>（在参数组<y>中）需要值

当语句在执行时使用`bindparam()` 时，如果未显式或隐式地提供值，则会出现此错误：

```py
stmt = select(table.c.column).where(table.c.id == bindparam("my_param"))

result = conn.execute(stmt)
```

上述情况下，未为参数 “my_param” 提供任何值。正确的方法是提供一个值：

```py
result = conn.execute(stmt, {"my_param": 12})
```

当消息采用“在参数组<y>中需要绑定参数<x>的值”的形式时，消息是指“executemany”执行风格。在这种情况下，语句通常是 INSERT、UPDATE 或 DELETE，并且正在传递参数列表。在此格式中，语句可以动态生成，以包括参数列表中提供的每个参数的参数位置，其中它将使用**第一组参数**来确定这些参数应该是什么。

例如，以下语句是基于第一个参数集计算的，要求参数 “a”、“b” 和 “c” - 这些名称确定语句的最终字符串格式，该格式将用于列表中每个参数集的参数。由于第二个条目不包含 “b”，因此会生成此错误：

```py
m = MetaData()
t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))

e.execute(
    t.insert(),
    [
        {"a": 1, "b": 2, "c": 3},
        {"a": 2, "c": 4},
        {"a": 3, "b": 4, "c": 5},
    ],
)
```

```py
sqlalchemy.exc.StatementError: (sqlalchemy.exc.InvalidRequestError)
A value is required for bind parameter 'b', in parameter group 1
[SQL: u'INSERT INTO t (a, b, c) VALUES (?, ?, ?)']
[parameters: [{'a': 1, 'c': 3, 'b': 2}, {'a': 2, 'c': 4}, {'a': 3, 'c': 5, 'b': 4}]]
```

由于“b”是必需的，因此将其传递为 `None`，以便 INSERT 可以继续进行：

```py
e.execute(
    t.insert(),
    [
        {"a": 1, "b": 2, "c": 3},
        {"a": 2, "b": None, "c": 4},
        {"a": 3, "b": 4, "c": 5},
    ],
)
```

另请参阅

发送参数

### 期望 FROM 子句，但是得到了 Select。要创建 FROM 子句，请使用 `.subquery()` 方法

这指的是 SQLAlchemy 1.4 中的一项更改，根据此更改，由 `select()` 等函数生成的 SELECT 语句，但也包括联合和文本型 SELECT 表达式，不再被视为 `FromClause` 对象，而且不能直接放在另一个 SELECT 语句的 FROM 子句中，而必须首先将它们包装在 `Subquery` 中。这是 Core 中的一个重大概念变更，完整的解释在 不再将 SELECT 语句隐式视为 FROM 子句 中讨论。

举个例子：

```py
m = MetaData()
t = Table("t", m, Column("a", Integer), Column("b", Integer), Column("c", Integer))
stmt = select(t)
```

在上述中，`stmt` 表示一个 SELECT 语句。当我们想要直接将 `stmt` 用作另一个 SELECT 的 FROM 子句时，比如我们试图从中选择时，会产生错误：

```py
new_stmt_1 = select(stmt)
```

或者，如果我们想在 FROM 子句中使用它，比如在 JOIN 中：

```py
new_stmt_2 = select(some_table).select_from(some_table.join(stmt))
```

在之前的 SQLAlchemy 版本中，使用一个 SELECT 嵌套在另一个 SELECT 中会产生一个带括号的、未命名的子查询。在大多数情况下，这种 SQL 形式并不是很有用，因为像 MySQL 和 PostgreSQL 这样的数据库要求 FROM 子句中的子查询具有命名别名，这意味着需要使用 `SelectBase.alias()` 方法，或者从 1.4 开始使用 `SelectBase.subquery()` 方法来实现这一点。在其他数据库中，为子查询命名仍然更清晰，以解决在子查询内部对列名的未来引用可能产生的任何歧义。

除了上述实际原因外，还有许多其他基于 SQLAlchemy 的原因导致了这一更改的进行。因此，上述两个语句的正确形式要求使用 `SelectBase.subquery()`：

```py
subq = stmt.subquery()

new_stmt_1 = select(subq)

new_stmt_2 = select(some_table).select_from(some_table.join(subq))
```

另请参阅

不再将 SELECT 语句隐式视为 FROM 子句

### 自动为原始 clauseelement 生成别名

自 1.4.26 版开始新加入的功能。

此弃用警告是针对非常古老且可能不为人知的模式的，该模式适用于遗留的`Query.join()`方法以及 2.0 样式 `Select.join()`方法，其中可以根据`relationship()`指定连接，但目标是`Table`或其他映射到类的 Core 可选择对象，而不是 ORM 实体，如映射类或`aliased()`构造：

```py
a1 = Address.__table__

q = (
    s.query(User)
    .join(a1, User.addresses)
    .filter(Address.email_address == "ed@foo.com")
    .all()
)
```

以上模式还允许任意可选择对象，例如 Core `Join`或`Alias`对象，但是没有此元素的自动适应，这意味着必须直接引用 Core 元素：

```py
a1 = Address.__table__.alias()

q = (
    s.query(User)
    .join(a1, User.addresses)
    .filter(a1.c.email_address == "ed@foo.com")
    .all()
)
```

指定连接目标的正确方式始终是使用映射类本身或一个`aliased`对象，后者使用`PropComparator.of_type()`修饰符来设置别名：

```py
# normal join to relationship entity
q = s.query(User).join(User.addresses).filter(Address.email_address == "ed@foo.com")

# name Address target explicitly, not necessary but legal
q = (
    s.query(User)
    .join(Address, User.addresses)
    .filter(Address.email_address == "ed@foo.com")
)
```

连接到别名：

```py
from sqlalchemy.orm import aliased

a1 = aliased(Address)

# of_type() form; recommended
q = (
    s.query(User)
    .join(User.addresses.of_type(a1))
    .filter(a1.email_address == "ed@foo.com")
)

# target, onclause form
q = s.query(User).join(a1, User.addresses).filter(a1.email_address == "ed@foo.com")
```

### 由于表重叠而自动生成别名

自 1.4.26 版新增。

当使用`Select.join()`方法或遗留的`Query.join()`方法查询涉及联合表继承的映射时，通常会生成此警告。问题在于，当在两个共享公共基表的联合继承模型之间进行连接时，如果不对其中一侧应用别名，则无法形成两个实体之间的适当 SQL JOIN；SQLAlchemy 对连接的右侧应用了别名。例如，给定一个联合继承映射：

```py
class Employee(Base):
    __tablename__ = "employee"
    id = Column(Integer, primary_key=True)
    manager_id = Column(ForeignKey("manager.id"))
    name = Column(String(50))
    type = Column(String(50))

    reports_to = relationship("Manager", foreign_keys=manager_id)

    __mapper_args__ = {
        "polymorphic_identity": "employee",
        "polymorphic_on": type,
    }

class Manager(Employee):
    __tablename__ = "manager"
    id = Column(Integer, ForeignKey("employee.id"), primary_key=True)

    __mapper_args__ = {
        "polymorphic_identity": "manager",
        "inherit_condition": id == Employee.id,
    }
```

以上映射包括`Employee`和`Manager`类之间的关系。由于这两个类都使用“employee”数据库表，从 SQL 角度来看，这是一个自引用关系。如果我们想要使用连接从`Employee`和`Manager`模型查询，那么在 SQL 级别上，“employee”表需要在查询中出现两次，这意味着必须给它起个别名。当我们使用 SQLAlchemy ORM 创建这样的连接时，得到的 SQL 如下所示：

```py
>>> stmt = select(Employee, Manager).join(Employee.reports_to)
>>> print(stmt)
SELECT  employee.id,  employee.manager_id,  employee.name,
employee.type,  manager_1.id  AS  id_1,  employee_1.id  AS  id_2,
employee_1.manager_id  AS  manager_id_1,  employee_1.name  AS  name_1,
employee_1.type  AS  type_1
FROM  employee  JOIN
(employee  AS  employee_1  JOIN  manager  AS  manager_1  ON  manager_1.id  =  employee_1.id)
ON  manager_1.id  =  employee.manager_id 
```

在上面，SQL 从 `employee` 表中选择，表示查询中的 `Employee` 实体。然后加入到一个右嵌套连接 `employee AS employee_1 JOIN manager AS manager_1`，其中 `employee` 表再次出现，但是作为一个匿名别名 `employee_1`。这就是警告消息所指的‘自动生成别名’。

当 SQLAlchemy 加载包含一个 `Employee` 和一个 `Manager` 对象的 ORM 行时，ORM 必须将来自上面的 `employee_1` 和 `manager_1` 表别名的行适配到未别名化的 `Manager` 类中。这个过程内部复杂，并且不能适应所有 API 特性，尤其是当尝试使用比这里显示的更深度嵌套的查询时，如 `contains_eager()` 等急切加载特性。由于该模式对于更复杂的场景不可靠，并涉及难以预测和遵循的隐式决策，因此会发出警告，并且该模式可能被视为一种传统特性。编写此查询的更好方法是使用适用于任何其他自引用关系的相同模式，即显式使用 `aliased()` 构造。对于连接继承和其他基于连接的映射，通常希望添加使用 `aliased.flat` 参数的使用，这将允许通过将别名应用于连接中的各个表来对两个或多个表进行 JOIN，而不是将连接嵌入到新的子查询中：

```py
>>> from sqlalchemy.orm import aliased
>>> manager_alias = aliased(Manager, flat=True)
>>> stmt = select(Employee, manager_alias).join(Employee.reports_to.of_type(manager_alias))
>>> print(stmt)
SELECT  employee.id,  employee.manager_id,  employee.name,
employee.type,  manager_1.id  AS  id_1,  employee_1.id  AS  id_2,
employee_1.manager_id  AS  manager_id_1,  employee_1.name  AS  name_1,
employee_1.type  AS  type_1
FROM  employee  JOIN
(employee  AS  employee_1  JOIN  manager  AS  manager_1  ON  manager_1.id  =  employee_1.id)
ON  manager_1.id  =  employee.manager_id 
```

如果我们想要使用 `contains_eager()` 来填充 `reports_to` 属性，我们引用别名：

```py
>>> stmt = (
...     select(Employee)
...     .join(Employee.reports_to.of_type(manager_alias))
...     .options(contains_eager(Employee.reports_to.of_type(manager_alias)))
... )
```

在某些更嵌套的情况下，如果 ORM 在非常嵌套的上下文中“自动别名”，则不使用显式 `aliased()` 对象，`contains_eager()` 选项没有足够的上下文来知道从哪里获取其数据。因此，最好不要依赖此功能，而是尽可能保持 SQL 构造的显式性。

## 对象关系映射

### IllegalStateChangeError 和并发异常

SQLAlchemy 2.0 引入了一个新系统，描述在会话主动引发非法并发或重入访问时，该系统主动检测在`Session`对象的单个实例上调用并发方法以及通过扩展`AsyncSession`代理对象。这些并发访问调用通常，尽管不是专门，会在单个`Session`实例在多个并发线程之间共享时发生，而没有进行同步访问，或者类似地，当单个`AsyncSession`实例在多个并发任务之间共享时（例如在使用`asyncio.gather()`等函数时）。这些使用模式不是这些对象的适当使用方式，如果没有 SQLAlchemy 实现的主动警告系统，仍然会在对象内产生无效状态，导致难以调试的错误，包括数据库连接本身的驱动程序级错误。

`Session`和`AsyncSession`的实例是**可变的、有状态的对象，没有内置的方法调用同步**，并代表一次性数据库事务，一次只能连接一个特定的`Engine`或`AsyncEngine`（请注意，这些对象都支持同时绑定到多个引擎，但在这种情况下，在事务范围内仍然只会有一个连接与引擎相关）。单个数据库事务不是并发 SQL 命令的适当目标；相反，运行并发数据库操作的应用程序应该使用并发事务。因此，对于这些对象，适当的模式是每个线程一个`Session`，或每个任务一个`AsyncSession`。

有关并发性的更多背景信息，请参阅会话是否线程安全？AsyncSession 在并发任务中是否安全共享？部分。### 父实例<x>未绑定到会话；（延迟加载/延迟加载/刷新等）操作无法继续

这可能是处理 ORM 时最常见的错误消息，它是由于 ORM 广泛使用的一种称为延迟加载的技术的性质造成的。延迟加载是一种常见的对象关系模式，其中由 ORM 持久化的对象维护与数据库本身的代理，以便当访问对象的各种属性时，可以从数据库中*惰性*检索它们的值。这种方法的优点是可以从数据库中检索对象而无需一次性加载所有属性或相关数据，而只需在那个时间点传递请求的数据即可。主要缺点基本上是优点的镜像，即如果加载了许多对象，这些对象在所有情况下都需要某组数据，则逐步加载该附加数据是浪费的。

延迟加载的另一个警告是，为了使延迟加载继续进行，对象必须**保持与 Session 关联**，以便能够检索其状态。此错误消息意味着对象已从其`Session`中解除关联，并且正在被要求从数据库中惰性加载数据。

对象与其`Session`分离的最常见原因是会话本身被关闭，通常是通过`Session.close()`方法。这些对象将继续存在，很常见地在 web 应用程序中被访问，它们被传递到服务器端模板引擎，并被要求加载更多它们无法加载的属性。

减轻此错误的方法是通过以下技术：

+   **尽量不要有分离的对象；不要过早关闭会话** - 通常，应用程序会在将相关对象传递给其他系统之前关闭事务，然后由于此错误而失败。有时事务不需要那么快关闭；一个例子是 web 应用程序在渲染视图之前关闭事务。这通常是以“正确性”的名义来做的，但可能被视为“封装”的错误应用，因为该术语指的是代码组织，而不是实际操作。使用 ORM 对象的模板正在使用[代理模式](https://en.wikipedia.org/wiki/Proxy_pattern)，该模式将数据库逻辑封装在调用者之外。如果`Session`可以保持打开，直到对象的寿命结束，这是最佳方法。

+   **否则，加载所有所需内容** - 很多时候不可能保持事务开启，特别是在需要将对象传递给无法在相同上下文中运行的其他系统的更复杂的应用程序中。在这种情况下，应用程序应准备处理分离对象，并应尽量适当地使用急切加载来确保对象在一开始就拥有所需内容。

+   **而且，重要的是，将 expire_on_commit 设置为 False** - 在使用分离对象时，对象需要重新加载数据的最常见原因是因为它们在上一次调用`Session.commit()`时被标记为过期。在处理分离对象时不应该使用这种过期机制；因此，`Session.expire_on_commit`参数应设置为`False`。通过防止对象在事务外部过期，加载的数据将保持存在，并且在访问数据时不会产生额外的延迟加载。

    还要注意，`Session.rollback()`方法会无条件地使`Session`中的所有内容过期，并且在非错误情况下也应该避免使用。

    另请参阅

    关系加载技术 - 关于急切加载和其他基于关系的加载技术的详细文档

    提交 - 有关会话提交的背景信息

    刷新/过期 - 属性过期的背景信息  ### 由于在刷新过程中发生了先前的异常，此会话的事务已被回滚

`Session`的刷新过程，描述在刷新中，如果遇到错误，将回滚数据库事务，以保持内部一致性。然而，一旦发生这种情况，会话的事务现在是“不活动的”，必须由调用应用程序显式回滚，就像如果没有发生故障，否则需要显式提交一样。

在使用 ORM 时，这是一个常见的错误，通常适用于尚未正确围绕其`Session`操作进行“框架化”的应用程序。更多详细信息请参阅常见问题解答中的“由于刷新期间的先前异常，此会话的事务已被回滚。”或类似问题。###对于关系<relationship>，delete-orphan 级联通常仅配置在一对多关系的“一”侧，而不是多对一或多对多关系的“多”侧。

当“delete-orphan”级联设置在多对一或多对多关系上时，会引发此错误，例如：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    # this will emit the error message when the mapper
    # configuration step occurs
    a = relationship("A", back_populates="bs", cascade="all, delete-orphan")

configure_mappers()
```

上面的“delete-orphan”设置在`B.a`上表示的意图是，当每个引用特定`A`的`B`对象被删除时，那么`A`也应该被删除。也就是说，它表达了正在被删除的“孤立”将是一个`A`对象，当每个引用它的`B`都被删除时，它变成了“孤立”。

“delete-orphan”级联模型不支持此功能。只有在单个对象被删除的情况下才考虑“孤立”问题，这个对象随后会引用零个或多个现在由此单个删除而“孤立”的对象，这将导致这些对象也被删除。换句话说，它只设计用于跟踪基于“父”对象的单个删除而创建“孤立”对象的情况，这是一个自然的情况，即一对多关系中的一个对象的删除会导致“多”侧上的相关项目的后续删除。

支持此功能的上述映射将级联设置放置在一对多的一侧，如下所示：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a", cascade="all, delete-orphan")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship("A", back_populates="bs")
```

当表达意图时，即当删除一个`A`时，所有它所指向的`B`对象也被删除。

错误消息然后继续建议使用`relationship.single_parent`标志。该标志可用于强制将能够有多个对象引用特定对象的关系实际上只有一个对象在某一时间引用它。它用于遗留或其他不太理想的数据库模式，在这些模式中，外键关系表明存在“多”集合，但实际上在任何时间只有一个对象会引用给定目标对象。可以通过上述示例来演示这种不常见的情况如下：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship(
        "A",
        back_populates="bs",
        single_parent=True,
        cascade="all, delete-orphan",
    )
```

上述配置将安装一个验证器，该验证器将强制执行在`B.a`关系的范围内只能将一个`B`与一个`A`关联起来的规则：

```py
>>> b1 = B()
>>> b2 = B()
>>> a1 = A()
>>> b1.a = a1
>>> b2.a = a1
sqlalchemy.exc.InvalidRequestError: Instance <A at 0x7eff44359350> is
already associated with an instance of <class '__main__.B'> via its
B.a attribute, and is only allowed a single parent.
```

请注意，此验证器的范围有限，并且无法阻止通过其他方向创建多个“父”对象。例如，它不会检测到与`A.bs`相同的设置：

```py
>>> a1.bs = [b1, b2]
>>> session.add_all([a1, b1, b2])
>>> session.commit()
INSERT  INTO  a  DEFAULT  VALUES
()
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,)
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,) 
```

然而，后续事情将不会如预期那样进行，因为“delete-orphan”级联将继续按照单个主要对象的术语工作，这意味着如果我们删除任一`B`对象，`A`将被删除。另一个`B`仍然存在，虽然 ORM 通常足够聪明以将外键属性设置为 NULL，但这通常不是所期望的：

```py
>>> session.delete(b1)
>>> session.commit()
UPDATE  b  SET  a_id=?  WHERE  b.id  =  ?
(None,  2)
DELETE  FROM  b  WHERE  b.id  =  ?
(1,)
DELETE  FROM  a  WHERE  a.id  =  ?
(1,)
COMMIT 
```

对于上述所有示例，类似的逻辑适用于多对多关系的计算；如果多对多关系在一侧设置了 single_parent=True，则该侧可以使用“delete-orphan”级联，但这几乎不可能是某人实际想要的，因为多对多关系的目的是可以有许多对象引用任一方向的对象。

总的来说，“delete-orphan”级联通常应用于一对多关系的“一”侧，以便删除“多”侧的对象，而不是相反。

从版本 1.3.18 开始更改：当在一对多或多对多关系上使用“delete-orphan”时，错误消息的文本已更新为更具描述性。

另请参阅

级联

delete-orphan

实例 <instance> 已通过其 <attribute> 属性与实例 <instance> 关联，且仅允许有一个父实例。  ### 实例 <instance> 已通过其 <attribute> 属性与实例 <instance> 关联，且仅允许有一个父实例。

当使用`relationship.single_parent`标志，并且一次分配多个对象作为对象的“父”时，会发出此错误。

给定以下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship(
        "A",
        single_parent=True,
        cascade="all, delete-orphan",
    )
```

意图指示一次最多只能有一个`B`对象引用特定的`A`对象：

```py
>>> b1 = B()
>>> b2 = B()
>>> a1 = A()
>>> b1.a = a1
>>> b2.a = a1
sqlalchemy.exc.InvalidRequestError: Instance <A at 0x7eff44359350> is
already associated with an instance of <class '__main__.B'> via its
B.a attribute, and is only allowed a single parent.
```

当此错误意外发生时，通常是因为在响应于对于关系 <relationship>，delete-orphan 级联通常仅在一对多关系的“一”侧上配置，而不在多对一或多对多关系的“多”侧上配置。描述的错误消息时，应用了`relationship.single_parent`标志，而实际问题是对“delete-orphan”级联设置的误解。有关详细信息，请参阅该消息。

另请参阅

对于关系<relationship>，删除孤立节点级联通常仅在一对多关系的“一”侧上配置，并不在多对一或多对多关系的“多”侧上配置。  ### 关系 X 将列 Q 复制到列 P，与关系‘Y’存在冲突。

此警告是指当两个或更多关系在 flush 时将数据写入相同列，但 ORM 没有任何协调这些关系的方式时发生的情况。根据具体情况，解决方案可能是两个关系需要使用`relationship.back_populates`相互引用，或者一个或多个关系应该配置为`relationship.viewonly`以防止冲突写入，或者有时配置是完全有意为之的，并应该配置`relationship.overlaps`来抑制每个警告。

对于典型示例，缺少`relationship.back_populates`的情况，给定以下映射：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    children = relationship("Child")

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))
    parent = relationship("Parent")
```

上述映射将生成警告：

```py
SAWarning: relationship 'Child.parent' will copy column parent.id to column child.parent_id,
which conflicts with relationship(s): 'Parent.children' (copies parent.id to child.parent_id).
```

关系`Child.parent`和`Parent.children`似乎存在冲突。解决方案是应用`relationship.back_populates`：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))
    parent = relationship("Parent", back_populates="children")
```

对于更加定制化的关系，在“重叠”情况可能是有意为之且无法解决时，`relationship.overlaps`参数可以指定不应该触发警告的关系名称。这通常发生在两个或更多关系指向相同基础表的情况下，这些关系包括自定义的`relationship.primaryjoin`条件，限制了每种情况下的相关项：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    c1 = relationship(
        "Child",
        primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 0)",
        backref="parent",
        overlaps="c2, parent",
    )
    c2 = relationship(
        "Child",
        primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 1)",
        overlaps="c1, parent",
    )

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))

    flag = Column(Integer)
```

在上述情况下，ORM 将知道`Parent.c1`、`Parent.c2`和`Child.parent`之间的重叠是有意为之的。### 对象无法转换为“持久”状态，因为此标识映射不再有效。

新版本 1.4.26 中新增。

此消息添加是为了适应以下情况：在原始`Session`关闭后或者已调用其`Session.expunge_all()`方法后，迭代将产生 ORM 对象的`Result`对象。当一个`Session`一次性清除所有对象时，该`Session`使用的内部身份映射将被替换为一个新的，并且原始的将被丢弃。一个未消耗且未缓冲的`Result`对象将在内部保持对该现在已丢弃的身份映射的引用。因此，当消耗`Result`时，将要产生的对象无法与该`Session`相关联。这种安排是有意设计的，因为通常不建议在创建它的事务上下文之外迭代未缓冲的`Result`对象：

```py
# context manager creates new Session
with Session(engine) as session_obj:
    result = sess.execute(select(User).where(User.id == 7))

# context manager is closed, so session_obj above is closed, identity
# map is replaced

# iterating the result object can't associate the object with the
# Session, raises this error.
user = result.first()
```

使用`asyncio` ORM 扩展时，通常不会发生上述情况，因为当`AsyncSession`返回一个同步风格的`Result`时，结果在语句执行时已经被预先缓冲。这样做是为了允许次级的急切加载器在不需要额外的`await`调用的情况下调用。

在上述情况下使用常规`Session`来预缓冲结果，可以像`asyncio`扩展一样使用`prebuffer_rows`执行选项，如下所示：

```py
# context manager creates new Session
with Session(engine) as session_obj:
    # result internally pre-fetches all objects
    result = sess.execute(
        select(User).where(User.id == 7), execution_options={"prebuffer_rows": True}
    )

# context manager is closed, so session_obj above is closed, identity
# map is replaced

# pre-buffered objects are returned
user = result.first()

# however they are detached from the session, which has been closed
assert inspect(user).detached
assert inspect(user).session is None
```

在上面，所选的 ORM 对象完全在`session_obj`块中生成，与`session_obj`关联并在`Result`对象中缓冲以进行迭代。在块外，`session_obj`被关闭并且清除这些 ORM 对象。迭代`Result`对象将产生这些 ORM 对象，但是由于它们的来源`Session`已将它们清除，它们将以分离状态交付。

注意

上面提到的 “预缓冲” vs. “非缓冲” `Result` 对象是指 ORM 将来自 DBAPI 的传入原始数据库行转换为 ORM 对象的过程。它不意味着底层的 `cursor` 对象本身，它表示来自 DBAPI 的待处理结果，是缓冲的还是非缓冲的，因为这实际上是一个更低层的缓冲。有关缓冲 `cursor` 结果本身的背景，请参阅 使用服务器端游标（也称为流式结果） 部分。 ### 无法解释注解式声明表形式的类型注解

SQLAlchemy 2.0 引入了一种新的注解式声明表声明系统，它从类定义中的 [**PEP 484**](https://peps.python.org/pep-0484/) 注解在运行时派生 ORM 映射属性信息。这种形式的要求是，所有的 ORM 注解都必须使用一个称为 `Mapped` 的通用容器才能正确注解。包括显式 [**PEP 484**](https://peps.python.org/pep-0484/) 类型注解的传统 SQLAlchemy 映射，例如使用 旧版 Mypy 扩展 进行类型支持的映射，可能包含诸如 `relationship()` 之类的指令，这些指令不包括这个通用容器。

要解决此问题，可以在类中添加 `__allow_unmapped__` 布尔属性，直到它们可以完全迁移到 2.0 语法。参见 迁移到 2.0 步骤六 - 为明确定义的 ORM 模型添加 __allow_unmapped__ 的迁移说明中的示例。

另请参阅

迁移到 2.0 步骤六 - 为明确定义的 ORM 模型添加 __allow_unmapped__ - 在 SQLAlchemy 2.0 - 主要迁移指南 文档中 ### 当将 <cls> 转换为数据类时，属性(s) 来自不是数据类的超类 <cls>。

当使用在 声明式数据类映射 中描述的 SQLAlchemy ORM 映射数据类功能与任何未本身声明为数据类的 mixin 类或抽象基类一起使用时（例如下面的示例）会出现此警告：

```py
from __future__ import annotations

import inspect
from typing import Optional
from uuid import uuid4

from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Mixin:
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

class Base(DeclarativeBase, MappedAsDataclass):
    pass

class User(Base, Mixin):
    __tablename__ = "sys_user"

    uid: Mapped[str] = mapped_column(
        String(50), init=False, default_factory=uuid4, primary_key=True
    )
    username: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column()
```

由于 `Mixin` 本身不扩展自 `MappedAsDataclass`，因此会生成以下警告：

```py
SADeprecationWarning: When transforming <class '__main__.User'> to a
dataclass, attribute(s) "create_user", "update_user" originates from
superclass <class
'__main__.Mixin'>, which is not a dataclass. This usage is deprecated and
will raise an error in SQLAlchemy 2.1\. When declaring SQLAlchemy
Declarative Dataclasses, ensure that all mixin classes and other
superclasses which include attributes are also a subclass of
MappedAsDataclass.
```

解决方法是在 `Mixin` 的签名中也添加 `MappedAsDataclass`：

```py
class Mixin(MappedAsDataclass):
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)
```

Python 的 [**PEP 681**](https://peps.python.org/pep-0681/) 规范不适用于声明在不是 dataclasses 的 dataclasses 超类上的属性；根据 Python dataclasses 的行为，这样的字段将被忽略，如以下示例所示：

```py
from dataclasses import dataclass
from dataclasses import field
import inspect
from typing import Optional
from uuid import uuid4

class Mixin:
    create_user: int
    update_user: Optional[int] = field(default=None)

@dataclass
class User(Mixin):
    uid: str = field(init=False, default_factory=lambda: str(uuid4()))
    username: str
    password: str
    email: str
```

上述 `User` 类将不会在其构造函数中包含 `create_user`，也不会尝试将 `update_user` 解释为 dataclass 属性。这是因为 `Mixin` 不是一个 dataclass。

SQLAlchemy 2.0 系列中的 dataclasses 功能未正确遵守此行为；相反，非 dataclass 混合类和超类上的属性被视为最终 dataclass 配置的一部分。然而，像 Pyright 和 Mypy 这样的类型检查器不会将这些字段视为 dataclass 构造函数的一部分，因为根据 [**PEP 681**](https://peps.python.org/pep-0681/)，它们应该被忽略。由于否则存在歧义，因此 SQLAlchemy 2.1 将要求在 dataclass 层次结构中具有 SQLAlchemy 映射属性的混合类本身必须是 dataclasses。  ### 创建 <classname> 的 dataclass 时遇到的 Python dataclasses 错误

当使用 `MappedAsDataclass` 混合类或 `registry.mapped_as_dataclass()` 装饰器时，SQLAlchemy 使用实际的 [Python dataclasses](https://docs.python.org/3/library/dataclasses.html) 模块，该模块位于 Python 标准库中，以将 dataclass 行为应用于目标类。此 API 具有自己的错误场景，其中大部分涉及在用户定义的类上构建 `__init__()` 方法；在类上声明的属性的顺序，以及[在超类上](https://docs.python.org/3/library/dataclasses.html#inheritance)的顺序决定了 `__init__()` 方法将如何构建，并且有特定规则规定了属性的组织方式以及它们应该如何使用参数，如 `init=False`，`kw_only=True` 等。**SQLAlchemy 不控制或实现这些规则**。因此，对于这种类型的错误，请参考 [Python dataclasses](https://docs.python.org/3/library/dataclasses.html) 文档，特别注意应用于[继承](https://docs.python.org/3/library/dataclasses.html#inheritance)的规则。

另请参阅

声明性 Dataclass 映射 - SQLAlchemy dataclasses 文档

[Python dataclasses](https://docs.python.org/3/library/dataclasses.html) - 在 python.org 网站上

[继承](https://docs.python.org/3/library/dataclasses.html#inheritance) - 在 python.org 网站上

在使用 ORM 通过主键进行批量更新功能时，如果在给定的记录中没有提供主键值，则会出现此错误，例如：

```py
>>> session.execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
```

上述情况下，参数字典列表的存在结合使用`Session`执行 ORM 启用的 UPDATE 语句将自动使用 ORM 通过主键进行批量更新，该批量更新期望参数字典包括主键值，例如：

```py
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...         {"id": 5, "fullname": "Eugene H. Krabs"},
...     ],
... )
```

要在不提供每个记录的主键值的情况下调用 UPDATE 语句，请使用`Session.connection()`来获取当前的`Connection`，然后使用它调用：

```py
>>> session.connection().execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
```

另请参阅

ORM 通过主键进行批量更新

禁用通过主键进行批量 ORM 更新以使用多个参数集的 UPDATE 语句 ### 非法状态更改错误和并发异常

SQLAlchemy 2.0 引入了一个新系统，描述在检测到非法并发或重新进入访问时，会主动引发会话，该系统主动检测在`Session`对象的个别实例上以及通过扩展`AsyncSession`代理对象调用并发方法时的情况。这些并发访问调用通常，但不仅仅，会发生在单个`Session`实例在多个并发线程之间共享时，而没有同步这样的访问，或者类似地，当单个`AsyncSession`实例在多个并发任务之间共享时（例如使用`asyncio.gather()`函数）。这些使用模式不是这些对象的适当使用方式，如果没有 SQLAlchemy 实现的主动警告系统，否则仍然会在对象内部产生无效状态，从而产生难以调试的错误，包括在数据库连接本身上的驱动程序级错误。

`Session`和`AsyncSession`的实例是**可变的、有状态的对象，没有内置的方法调用同步**，并且代表一次**单一的持续数据库事务**，一次只能在一个特定的`Engine`或`AsyncEngine`上绑定的数据库连接（请注意，这些对象都支持同时绑定到多个引擎，但在这种情况下，在事务范围内仍然只有一个连接在运行）。单个数据库事务不是并发 SQL 命令的适当目标；相反，运行并发数据库操作的应用程序应该使用并发事务。因此，对于这些对象，适当的模式是每个线程一个`Session`，或每个任务一个`AsyncSession`。

有关并发性的更多背景信息，请参阅会话是否线程安全？AsyncSession 是否安全可在并发任务中共享？部分。

### 父实例 <x> 未绑定到会话；（延迟加载/延迟加载/刷新等）操作无法继续

这很可能是处理 ORM 时最常见的错误消息，它是由 ORM 广泛使用的一种技术的性质导致的，这种技术被称为延迟加载。延迟加载是一种常见的对象关系模式，其中由 ORM 持久化的对象维护一个代理到数据库本身，因此当访问对象上的各种属性时，它们的值可能会被*惰性地*从数据库中检索出来。这种方法的优势在于可以从数据库中检索对象，而无需一次加载所有属性或相关数据，而只需在请求时传递所需的数据。主要的缺点基本上是优势的镜像，即如果正在加载许多需要在所有情况下都需要一组数据的对象，逐步加载额外数据是浪费的。

延迟加载的另一个警告是，为了使延迟加载继续进行，对象必须**保持与会话关联**，以便能够检索其状态。此错误消息意味着一个对象已经与其`Session`解除关联，并且正在被要求从数据库中延迟加载数据。

对象从其 `Session` 分离的最常见原因是会话本身被关闭，通常是通过 `Session.close()` 方法。然后，这些对象将继续存在，被进一步访问，往往是在 Web 应用程序中，在那里它们被传递给服务器端模板引擎，并要求获取它们无法加载的进一步属性。

对这个错误的缓解是通过这些技术：

+   **尽量避免分离对象；不要过早关闭会话** - 通常，应用程序会在将相关对象传递给其他系统之前关闭事务，但由于这个错误而失败。有时，事务不需要那么快关闭；一个例子是 Web 应用在视图呈现之前关闭事务。这通常是以“正确性”的名义而完成的，但可能被视为对“封装”的错误应用，因为此术语指的是代码组织，而不是实际操作。使用 ORM 对象的模板正在使用[代理模式](https://zh.wikipedia.org/wiki/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F)来保持数据库逻辑与调用者的封装。如果`Session`可以保持打开状态，直到对象的生命周期结束，这是最佳方法。

+   **否则，加载所有需要的内容** - 很多时候是不可能保持事务处于打开状态的，特别是在需要将对象传递给其他系统的更复杂的应用程序中，即使它们在同一个进程中也无法运行在相同的上下文中。在这种情况下，应用程序应准备处理分离的对象，并应尽量适当地使用急切加载以确保对象从一开始就拥有所需内容。

+   **而且，重要的是，将 expire_on_commit 设置为 False** - 当使用分离对象时，对象需要重新加载数据的最常见原因是因为它们从上一次调用 `Session.commit()` 被标记为过期。在处理分离对象时不应使用此过期；因此应将 `Session.expire_on_commit` 参数设置为`False`。通过防止对象在事务外部过期，已加载的数据将保持存在，并且在访问该数据时不会产生额外的延迟加载。

    `Session.rollback()` 方法会无条件地使 `Session` 中的所有内容过期，因此在非错误情况下也应避免使用。

    另请参阅

    关系加载技术 - 关于急加载和其他面向关系的加载技术的详细文档

    提交 - 会话提交的背景介绍

    刷新/过期 - 属性过期的背景介绍

### 由于刷新期间的先前异常，此会话的事务已回滚

`Session` 的刷新过程在遇到错误时会回滚数据库事务，以保持内部一致性。然而，一旦发生这种情况，会话的事务现在处于 “不活动” 状态，并且必须由调用应用程序显式地回滚，就像如果没有发生故障时需要显式提交一样。

当使用 ORM 时，这是一个常见的错误，通常适用于尚未在其 `Session` 操作周围正确设置 “框架”的应用程序。更多详细信息请参阅“由于刷新期间的先前异常，此会话的事务已回滚。”（或类似内容）的常见问题。

### 对于关系 <relationship>，只有在一对多关系的“一”端才通常配置了 delete-orphan 级联，而不是在多对一或多对多关系的“多”端。

当在多对一或多对多关系上设置了 “delete-orphan” 级联 时会出现此错误，例如：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    # this will emit the error message when the mapper
    # configuration step occurs
    a = relationship("A", back_populates="bs", cascade="all, delete-orphan")

configure_mappers()
```

上面的 `B.a` 上的 “delete-orphan” 设置表明了这样一个意图，即当指向特定 `A` 的每个 `B` 对象都被删除时，该 `A` 也应该被删除。也就是说，它表达了被删除的 “孤立” 对象将是一个 `A` 对象，并且当指向它的每个 `B` 都被删除时，它就成为了一个 “孤立” 对象。

“delete-orphan”级联模型不支持此功能。 “孤儿”考虑仅在单个对象的删除方面进行，然后引用零个或多个由此单个删除“孤儿”对象的对象，这将导致这些对象也被删除。换句话说，它仅设计为基于删除每个孤儿的一个且仅一个“父”对象的创建，“父”对象在一对多关系中的自然情况下导致“多”侧的相关项目随后被删除。

为支持此功能的上述映射将在一对多关系的一侧放置级联设置，如下所示：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a", cascade="all, delete-orphan")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship("A", back_populates="bs")
```

当表达出当删除一个`A`时，所有它所引用的`B`对象也将被删除的意图时。

错误消息随后建议使用`relationship.single_parent`标志。此标志可用于强制执行一个关系，该关系可以让多个对象引用特定对象，但实际上一次只能有**一个**对象引用它。它用于传统或其他不太理想的数据库模式，其中外键关系暗示“多”集合，但实际上只有一个对象会引用给定目标对象。这种不常见的情况可以如上例所示进行演示：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

    bs = relationship("B", back_populates="a")

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship(
        "A",
        back_populates="bs",
        single_parent=True,
        cascade="all, delete-orphan",
    )
```

上述配置将安装一个验证器，该验证器将强制执行在`B.a`关系的范围内一次只能关联一个`B`与一个`A`：

```py
>>> b1 = B()
>>> b2 = B()
>>> a1 = A()
>>> b1.a = a1
>>> b2.a = a1
sqlalchemy.exc.InvalidRequestError: Instance <A at 0x7eff44359350> is
already associated with an instance of <class '__main__.B'> via its
B.a attribute, and is only allowed a single parent.
```

请注意，此验证器的范围有限，并不会阻止通过其他方向创建多个“父级”。例如，它不会检测到关于`A.bs`的相同设置：

```py
>>> a1.bs = [b1, b2]
>>> session.add_all([a1, b1, b2])
>>> session.commit()
INSERT  INTO  a  DEFAULT  VALUES
()
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,)
INSERT  INTO  b  (a_id)  VALUES  (?)
(1,) 
```

然而，事情不会按预期进行，因为“delete-orphan”级联将继续按照**单个**主要对象的方式工作，这意味着如果我们删除其中一个`B`对象，`A`将被删除。另一个`B`仍然存在，ORM 通常会足够聪明地将外键属性设置为 NULL，但这通常不是期望的结果：

```py
>>> session.delete(b1)
>>> session.commit()
UPDATE  b  SET  a_id=?  WHERE  b.id  =  ?
(None,  2)
DELETE  FROM  b  WHERE  b.id  =  ?
(1,)
DELETE  FROM  a  WHERE  a.id  =  ?
(1,)
COMMIT 
```

对于上述所有示例，类似的逻辑也适用于多对多关系的计算；如果多对多关系在一侧设置了 single_parent=True，则该侧可以使用“delete-orphan”级联，但这很不可能是某人实际想要的，因为多对多关系的目的是让可以有许多对象相互引用。

通常，“delete-orphan”级联通常应用于一对多关系的“一”侧，以便删除“多”侧的对象，而不是相反。

1.3.18 版本中的更改：当在多对一或多对多关系上使用“delete-orphan”时，错误消息的文本已更新为更详细的描述。

另请参阅

级联

delete-orphan

实例<instance>已通过其<attribute>属性与<instance>的实例关联，并且只允许一个父级。

### 实例<instance>已通过其<attribute>属性与<instance>的实例关联，并且只允许一个父级。

当使用`relationship.single_parent`标志，并且同时为一个对象分配了多个“父级”对象时，会发出此错误。

给定以下映射：

```py
class A(Base):
    __tablename__ = "a"

    id = Column(Integer, primary_key=True)

class B(Base):
    __tablename__ = "b"
    id = Column(Integer, primary_key=True)
    a_id = Column(ForeignKey("a.id"))

    a = relationship(
        "A",
        single_parent=True,
        cascade="all, delete-orphan",
    )
```

意图指示不超过一个`B`对象可以同时引用特定的`A`对象：

```py
>>> b1 = B()
>>> b2 = B()
>>> a1 = A()
>>> b1.a = a1
>>> b2.a = a1
sqlalchemy.exc.InvalidRequestError: Instance <A at 0x7eff44359350> is
already associated with an instance of <class '__main__.B'> via its
B.a attribute, and is only allowed a single parent.
```

当这种错误出现时，通常是因为在错误消息中描述的错误消息响应中应用了`relationship.single_parent`标志，实际上问题是对“delete-orphan”级联设置的误解。请参阅该消息以了解详情。

另请参阅

对于关系<relationship>，delete-orphan 级联通常仅在一对多关系的“one”端上配置，并且不在多对一或多对多关系的“many”端上配置。

### 关系 X 将列 Q 复制到列 P，与关系‘Y’冲突

此警告是指当两个或更多关系将数据写入相同的列时，但 ORM 没有任何协调这些关系的方式时。根据具体情况，解决方案可能是两个关系需要彼此引用，使用`relationship.back_populates`，或者一个或多个关系应该配置为`relationship.viewonly`以防止冲突的写入，有时配置是完全有意的，应该配置`relationship.overlaps`以使每个警告静音。

对于典型的缺少`relationship.back_populates`的示例，给定以下映射：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    children = relationship("Child")

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))
    parent = relationship("Parent")
```

上述映射将生成警告：

```py
SAWarning: relationship 'Child.parent' will copy column parent.id to column child.parent_id,
which conflicts with relationship(s): 'Parent.children' (copies parent.id to child.parent_id).
```

关系`Child.parent`和`Parent.children`似乎存在冲突。解决方案是应用`relationship.back_populates`：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))
    parent = relationship("Parent", back_populates="children")
```

对于更自定义的关系，其中“重叠”情况可能是有意的并且无法解决的情况，`relationship.overlaps`参数可以指定不应触发警告的关系名称。这通常发生在对同一基础表的两个或多个关系中，这些关系包括限制每种情况中相关项的自定义`relationship.primaryjoin`条件：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)
    c1 = relationship(
        "Child",
        primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 0)",
        backref="parent",
        overlaps="c2, parent",
    )
    c2 = relationship(
        "Child",
        primaryjoin="and_(Parent.id == Child.parent_id, Child.flag == 1)",
        overlaps="c1, parent",
    )

class Child(Base):
    __tablename__ = "child"
    id = Column(Integer, primary_key=True)
    parent_id = Column(ForeignKey("parent.id"))

    flag = Column(Integer)
```

在上述情况下，ORM 将知道`Parent.c1`、`Parent.c2`和`Child.parent`之间的重叠是有意的。

### 对象无法转换为‘持久’状态，因为此标识映射不再有效。

自版本 1.4.26 新增。

添加此消息是为了适应以下情况：当迭代一个在原始`Session`关闭后或在其上调用`Session.expunge_all()`方法后仍会产生 ORM 对象的`Result`对象时。当一个`Session`一次性删除所有对象时，该`Session`使用的内部标识映射将被替换为新的，并且原始映射将被丢弃。一个未使用且未缓冲的`Result`对象将在内部维护对该现在被丢弃的标识映射的引用。因此，当消耗了`Result`时，将要产生的对象无法与该`Session`关联。这种安排是有意设计的，因为通常不建议在创建它的事务上下文之外迭代未缓冲的`Result`对象：

```py
# context manager creates new Session
with Session(engine) as session_obj:
    result = sess.execute(select(User).where(User.id == 7))

# context manager is closed, so session_obj above is closed, identity
# map is replaced

# iterating the result object can't associate the object with the
# Session, raises this error.
user = result.first()
```

使用 `asyncio` ORM 扩展时，通常不会出现上述情况，因为当 `AsyncSession` 返回一个同步风格的 `Result` 时，结果在语句执行时已经被预先缓冲。这样做是为了允许次级急切加载器在不需要额外的 `await` 调用的情况下调用。

若要在上述情况下像 `asyncio` 扩展一样预先缓冲结果，可以使用 `prebuffer_rows` 执行选项如下所示：

```py
# context manager creates new Session
with Session(engine) as session_obj:
    # result internally pre-fetches all objects
    result = sess.execute(
        select(User).where(User.id == 7), execution_options={"prebuffer_rows": True}
    )

# context manager is closed, so session_obj above is closed, identity
# map is replaced

# pre-buffered objects are returned
user = result.first()

# however they are detached from the session, which has been closed
assert inspect(user).detached
assert inspect(user).session is None
```

在上面的例子中，所选的 ORM 对象完全在 `session_obj` 块内生成，与 `session_obj` 关联并在 `Result` 对象内缓冲以供迭代。在块外，`session_obj` 被关闭并清除这些 ORM 对象。迭代 `Result` 对象将产生这些 ORM 对象，但是由于它们的来源 `Session` 已将它们清除，它们将以 分离 状态传递。

注意

上文提到的“预缓冲”与“未缓冲”的 `Result` 对象指的是 ORM 将传入的原始数据库行从 DBAPI 转换为 ORM 对象的过程。这并不意味着底层的 `cursor` 对象本身，它代表了来自 DBAPI 的待处理结果，是缓冲的还是非缓冲的，因为这本质上是一个更低层次的缓冲。有关 `cursor` 结果本身的缓冲背景，请参阅 使用服务器端游标 (即流式结果) 部分。

### 无法解释注释的声明式表格形式的类型注释

SQLAlchemy 2.0 引入了一个新的 注释式声明表 声明系统，它会在运行时从类定义中的 [**PEP 484**](https://peps.python.org/pep-0484/) 注释中派生 ORM 映射属性信息。这种形式的要求是，所有 ORM 注释都必须使用一个名为`Mapped`的通用容器才能正确注释。包含显式 [**PEP 484**](https://peps.python.org/pep-0484/) 类型注释的传统 SQLAlchemy 映射，例如那些使用 传统 Mypy 扩展 进行类型支持的映射，可能包含不包括此通用容器的诸如`relationship()`之类的指令。

要解决此问题，可以将类标记为`__allow_unmapped__`布尔属性，直到它们可以完全迁移到 2.0 语法。请参阅迁移到 2.0 第六步 - 向显式类型化的 ORM 模型添加 __allow_unmapped__ 的迁移说明以获取示例。

另请参阅

迁移到 2.0 第六步 - 向显式类型化的 ORM 模型添加 __allow_unmapped__ - 在 SQLAlchemy 2.0 - 主要迁移指南 文档中

### 当将<cls>转换为数据类时，属性源自于不是数据类的超类<cls>。

当与任何不是自身声明为数据类的混入类或抽象基类一起使用 SQLAlchemy ORM 映射数据类功能时，会出现此警告，例如下面的示例:

```py
from __future__ import annotations

import inspect
from typing import Optional
from uuid import uuid4

from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase
from sqlalchemy.orm import Mapped
from sqlalchemy.orm import mapped_column
from sqlalchemy.orm import MappedAsDataclass

class Mixin:
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)

class Base(DeclarativeBase, MappedAsDataclass):
    pass

class User(Base, Mixin):
    __tablename__ = "sys_user"

    uid: Mapped[str] = mapped_column(
        String(50), init=False, default_factory=uuid4, primary_key=True
    )
    username: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column()
```

在上述情况下，由于`Mixin`本身不是扩展自`MappedAsDataclass`，因此会生成以下警告:

```py
SADeprecationWarning: When transforming <class '__main__.User'> to a
dataclass, attribute(s) "create_user", "update_user" originates from
superclass <class
'__main__.Mixin'>, which is not a dataclass. This usage is deprecated and
will raise an error in SQLAlchemy 2.1\. When declaring SQLAlchemy
Declarative Dataclasses, ensure that all mixin classes and other
superclasses which include attributes are also a subclass of
MappedAsDataclass.
```

修复方法是在`Mixin`的签名中也添加`MappedAsDataclass`:

```py
class Mixin(MappedAsDataclass):
    create_user: Mapped[int] = mapped_column()
    update_user: Mapped[Optional[int]] = mapped_column(default=None, init=False)
```

Python 的 [**PEP 681**](https://peps.python.org/pep-0681/) 规范不包含不是数据类本身的数据类超类上声明的属性; 根据 Python 数据类的行为，这些字段会被忽略，如下例所示:

```py
from dataclasses import dataclass
from dataclasses import field
import inspect
from typing import Optional
from uuid import uuid4

class Mixin:
    create_user: int
    update_user: Optional[int] = field(default=None)

@dataclass
class User(Mixin):
    uid: str = field(init=False, default_factory=lambda: str(uuid4()))
    username: str
    password: str
    email: str
```

在上述情况下，`User`类将不会在其构造函数中包含`create_user`，也不会尝试将`update_user`解释为数据类属性。这是因为`Mixin`不是数据类。

SQLAlchemy 2.0 系列中的数据类功能未正确遵守此行为；相反，非数据类混合类和超类上的属性被视为最终数据类配置的一部分。但是像 Pyright 和 Mypy 这样的类型检查器不会将这些字段视为数据类构造函数的一部分，因为根据[**PEP 681**](https://peps.python.org/pep-0681/)，它们应该被忽略。由于否则它们的存在是模棱两可的，因此 SQLAlchemy 2.1 将要求在数据类层次结构中具有 SQLAlchemy 映射属性的混合类本身必须是数据类。

### 创建<dataclass>类时遇到的 Python 数据类错误

当使用`MappedAsDataclass`混合类或`registry.mapped_as_dataclass()`装饰器时，SQLAlchemy 利用 Python 标准库中实际的[Python 数据类](https://docs.python.org/3/library/dataclasses.html)模块，以将数据类行为应用于目标类。此 API 具有自己的错误场景，其中大多数涉及在用户定义的类上构建`__init__()`方法；在类上声明的属性的顺序，以及[在超类上](https://docs.python.org/3/library/dataclasses.html#inheritance)声明的属性，决定了`__init__()`方法将如何构建，并且有特定规则规定了属性的组织方式以及它们应如何使用参数，如`init=False`，`kw_only=True`等。**SQLAlchemy 不控制或实现这些规则**。因此，对于这种类型的错误，请参阅[Python 数据类](https://docs.python.org/3/library/dataclasses.html)文档，特别注意应用于[继承](https://docs.python.org/3/library/dataclasses.html#inheritance)的规则。

另请参阅

声明式数据类映射 - SQLAlchemy 数据类文档

[Python 数据类](https://docs.python.org/3/library/dataclasses.html) - 在 python.org 网站上

[继承](https://docs.python.org/3/library/dataclasses.html#inheritance) - 在 python.org 网站上

### 按主键进行每行 ORM 批量更新要求记录包含主键值

当在给定记录中使用 ORM 按主键批量更新功能而未提供主键值时，将出现此错误，例如：

```py
>>> session.execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
```

上述，将参数字典列表与使用`Session`执行 ORM 启用的 UPDATE 语句结合使用将自动使用按主键进行 ORM 批量更新，该功能期望参数字典包含主键值，例如：

```py
>>> session.execute(
...     update(User),
...     [
...         {"id": 1, "fullname": "Spongebob Squarepants"},
...         {"id": 3, "fullname": "Patrick Star"},
...         {"id": 5, "fullname": "Eugene H. Krabs"},
...     ],
... )
```

若要在不提供每条记录主键值的情况下调用 UPDATE 语句，请使用 `Session.connection()` 获取当前的 `Connection`，然后使用它进行调用：

```py
>>> session.connection().execute(
...     update(User).where(User.name == bindparam("u_name")),
...     [
...         {"u_name": "spongebob", "fullname": "Spongebob Squarepants"},
...         {"u_name": "patrick", "fullname": "Patrick Star"},
...     ],
... )
```

另请参阅

按主键进行 ORM 批量更新

禁用批量 ORM 按主键更新，以及包含多个参数集的 UPDATE 语句

## AsyncIO 异常

### 等待所需

SQLAlchemy 异步模式要求使用异步驱动程序连接到数据库。尝试使用不兼容的 DBAPI 的异步版本与 SQLAlchemy 的异步版本一起使用时通常会引发此错误。

另请参阅

异步 I/O (asyncio)  ### 缺少 Greenlet

在没有创建 SQLAlchemy AsyncIO 代理类设置的协程生成上下文之外启动异步 DBAPI 调用时，通常会引发此错误。通常情况下，当在意外位置尝试进行 IO 操作时，使用了不直接提供 `await` 关键字的调用模式时会发生此错误。在使用 ORM 时，几乎总是由于使用了延迟加载，这在 asyncio 中不直接支持，需要采取额外步骤和/或替代加载器模式才能成功使用。

另请参阅

使用 AsyncSession 时防止隐式 IO - 涵盖了大多数可能发生此问题的 ORM 方案以及如何缓解这个问题，包括在懒加载情况下使用的特定模式。  ### 无可用检查

当前不支持直接在 `AsyncConnection` 或 `AsyncEngine` 对象上直接使用 `inspect()` 函数，因为尚未提供 `Inspector` 对象的可等待形式。相反，该对象是通过使用 `inspect()` 函数获取的，以一种方式，使其引用 `AsyncConnection` 对象的底层 `AsyncConnection.sync_connection` 属性；然后，通过使用 `AsyncConnection.run_sync()` 方法以及执行所需操作的自定义函数，以“同步”调用样式使用 `Inspector`：

```py
async def async_main():
    async with engine.connect() as conn:
        tables = await conn.run_sync(
            lambda sync_conn: inspect(sync_conn).get_table_names()
        )
```

另请参阅

使用 Inspector 检查模式对象 - 使用 asyncio 扩展的 `inspect()` 的附加示例。### 必须等待

SQLAlchemy 的异步模式需要使用异步驱动程序连接到数据库。当尝试使用不兼容的 DBAPI 时，通常会引发此错误。

另请参阅

异步 I/O (asyncio)

### 缺少 Greenlet

尝试在由 SQLAlchemy AsyncIO 代理类设置的 greenlet spawn 上下文之外启动异步 DBAPI 调用时会引发此错误。通常，当在意外位置尝试进行 IO 操作时，使用不直接提供 `await` 关键字的调用模式会发生此错误。在使用 ORM 时，这几乎总是由于使用 懒加载，在 asyncio 中，需要通过额外的步骤和/或替代加载程序模式才能成功使用。

另请参阅

在使用 AsyncSession 时预防隐式 IO - 涵盖了大多数可能出现此问题的 ORM 方案以及如何缓解，包括在懒加载场景中使用的特定模式。

### 无可用检查

直接在 `AsyncConnection` 或 `AsyncEngine` 对象上使用 `inspect()` 函数目前不受支持，因为尚未提供 `Inspector` 对象的可等待形式。相反，通过以获取 `AsyncConnection` 对象的基础 `AsyncConnection.sync_connection` 属性的方式获取该对象；然后使用 `Inspector` 通过使用 `AsyncConnection.run_sync()` 方法以及执行所需操作的自定义函数来进行 “同步” 调用：

```py
async def async_main():
    async with engine.connect() as conn:
        tables = await conn.run_sync(
            lambda sync_conn: inspect(sync_conn).get_table_names()
        )
```

另请参阅

使用 Inspector 检查模式对象 - 使用 asyncio 扩展与 `inspect()` 的其他示例。

## 核心异常类

查看 核心异常 以获取核心异常类。

## ORM 异常类

查看 ORM 异常 以获取 ORM 异常类。

## 旧版本异常

本节中的异常不是由当前的 SQLAlchemy 版本生成的，但提供了这些异常以适应异常消息的超链接。

### 在 SQLAlchemy 2.0 中，<某个函数> 将不再 <某事>

SQLAlchemy 2.0 对于核心和 ORM 组件中的许多关键 SQLAlchemy 使用模式都表示了一个重大转变。2.0 发布的目标是在 SQLAlchemy 自早期开始以来的一些最基本的假设中进行轻微调整，并提供一个新的简化使用模型，希望它在核心和 ORM 组件之间更加简约一致，并更加强大。

在 SQLAlchemy 2.0 - 主要迁移指南中介绍的 SQLAlchemy 2.0 项目包含了一个全面的未来兼容系统，该系统已集成到 SQLAlchemy 1.4 系列中，因此应用程序将具有明确、无歧义和逐步的升级路径，以将应用程序迁移到完全兼容 2.0 的状态。`RemovedIn20Warning`废弃警告是该系统的基础，提供了关于现有代码库中需要修改的行为的指导。如何启用此警告的概述在 SQLAlchemy 2.0 Deprecations Mode 中。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南 - 从 1.x 系列升级过程的概述，以及 SQLAlchemy 2.0 的当前目标和进展。

SQLAlchemy 2.0 Deprecations Mode - 关于如何在 SQLAlchemy 1.4 中使用“2.0 废弃模式”的具体指南。### 对象正在被合并到会话中，沿着反向引用级联。

此消息指的是 SQLAlchemy 的“backref cascade”行为，在版本 2.0 中已删除。这指的是将对象添加到`Session`中，因为该会话中已经存在的另一个对象与之关联。由于这种行为被证明比有用更令人困惑，因此添加了`relationship.cascade_backrefs`和`backref.cascade_backrefs`参数，可以将其设置为`False`以禁用它，在 SQLAlchemy 2.0 中完全删除了“cascade backrefs”行为。

对于较旧的 SQLAlchemy 版本，要在当前使用`relationship.backref`字符串参数配置的反向引用上设置`relationship.cascade_backrefs`为`False`，必须首先使用`backref()`函数声明反向引用，以便可以传递`backref.cascade_backrefs`参数。

或者，可以通过在“未来”模式下使用`Session`，通过为`Session.future`参数传递`True`来全面关闭“cascade backrefs”行为。

另请参阅

cascade_backrefs 行为在 2.0 中已弃用 - SQLAlchemy 2.0 的变更背景。  ### 创建在“传统”模式下的 select() 构造；关键字参数等。

从 SQLAlchemy 1.4 开始，`select()` 构造已经更新为支持 SQLAlchemy 2.0 中标准的新调用风格。为了在 1.4 系列内保持向后兼容性，该构造在“传统”风格以及“新”风格下都接受参数。

“新”风格的特点是，列和表达式只传递给 `select()` 构造；对象的任何其他修饰符必须通过后续的方法链传递：

```py
# this is the way to do it going forward
stmt = select(table1.c.myid).where(table1.c.myid == table2.c.otherid)
```

作为对比，在 SQLAlchemy 的传统形式中，像 `Select.where()` 这样的方法甚至还未添加之前，`select()` 会是这样的：

```py
# this is how it was documented in original SQLAlchemy versions
# many years ago
stmt = select([table1.c.myid], whereclause=table1.c.myid == table2.c.otherid)
```

或者甚至，“whereclause”会被按位置传递：

```py
# this is also how it was documented in original SQLAlchemy versions
# many years ago
stmt = select([table1.c.myid], table1.c.myid == table2.c.otherid)
```

多年来，大多数叙述性文档中已经删除了接受的额外“whereclause”和其他参数，导致了一种最为熟悉的调用风格，即将列参数作为列表传递，但没有进一步的参数：

```py
# this is how it's been documented since around version 1.0 or so
stmt = select([table1.c.myid]).where(table1.c.myid == table2.c.otherid)
```

在 select() 不再接受多样化的构造函数参数，列是按位置传递的 文档中以 2.0 迁移 的术语描述了这一变更。

另请参阅

select() 不再接受多样化的构造函数参数，列是按位置传递的

SQLAlchemy 2.0 - 重大迁移指南  ### 通过传递 future=True 到 Session 上，将会忽略通过传统绑定的元数据所定位的绑定。

“绑定元数据”的概念一直存在直到 SQLAlchemy 1.4；截至 SQLAlchemy 2.0，它已被移除。

此错误指的是`MetaData.bind`参数，它在 ORM `Session`中允许将特定映射类与`Engine`相关联的`MetaData`对象上。在 SQLAlchemy 2.0 中，`Session`必须直接链接到每个`Engine`上。也就是说，不能再不带任何参数实例化`Session`或`sessionmaker`，并将`Engine`与`MetaData`相关联：

```py
engine = create_engine("sqlite://")
Session = sessionmaker()
metadata_obj = MetaData(bind=engine)
Base = declarative_base(metadata=metadata_obj)

class MyClass(Base): ...

session = Session()
session.add(MyClass())
session.commit()
```

`Engine`必须直接与`sessionmaker`或`Session`相关联。`MetaData`对象不应再与任何引擎相关联：

```py
engine = create_engine("sqlite://")
Session = sessionmaker(engine)
Base = declarative_base()

class MyClass(Base): ...

session = Session()
session.add(MyClass())
session.commit()
```

在 SQLAlchemy 1.4 中，当在`sessionmaker`或`Session`上设置`Session.future`标志时，启用此 2.0 样式行为。###此编译对象未绑定到任何引擎或连接

此错误涉及到“绑定元数据”的概念，这是仅存在于 1.x 版本中的传统 SQLAlchemy 模式。当直接从未与任何`Engine`相关联的 Core 表达式对象上调用`Executable.execute()`方法时会发生此问题：

```py
metadata_obj = MetaData()
table = Table("t", metadata_obj, Column("q", Integer))

stmt = select(table)
result = stmt.execute()  # <--- raises
```

逻辑预期的是`MetaData`对象已经**绑定**到`Engine`：

```py
engine = create_engine("mysql+pymysql://user:pass@host/db")
metadata_obj = MetaData(bind=engine)
```

在上述情况下，从`Table`派生的任何语句将隐式使用给定的`Engine`来调用该语句。

请注意，绑定元数据的概念**在 SQLAlchemy 2.0 中不存在**。调用语句的正确方式是通过`Connection.execute()`方法的`Connection`：

```py
with engine.connect() as conn:
    result = conn.execute(stmt)
```

在使用 ORM 时，通过`Session`也可以使用类似的功能：

```py
result = session.execute(stmt)
```

另请参阅

语句执行基础知识### 此连接处于非活动事务状态。请在继续之前完全回滚()

此错误条件已添加到 SQLAlchemy 自版本 1.4 起，不适用于 SQLAlchemy 2.0。该错误指的是将`Connection`放入事务中，使用类似`Connection.begin()`的方法创建一个进一步的“标记”事务；然后使用`Transaction.rollback()`回滚或使用`Transaction.close()`关闭“标记”事务，但外部事务仍处于“非活动”状态，必须回滚。

该模式如下：

```py
engine = create_engine(...)

connection = engine.connect()
transaction1 = connection.begin()

# this is a "sub" or "marker" transaction, a logical nesting
# structure based on "real" transaction transaction1
transaction2 = connection.begin()
transaction2.rollback()

# transaction1 is still present and needs explicit rollback,
# so this will raise
connection.execute(text("select 1"))
```

上面，`transaction2`是一个“标记”事务，表示在外部事务内部的事务逻辑嵌套；虽然内部事务可以通过其 rollback()方法回滚整个事务，但其 commit()方法除了关闭“标记”事务本身的范围外，没有任何效果。调用`transaction2.rollback()`的效果是**停用**transaction1，这意味着它在数据库级别上基本上被回滚，但仍然存在以适应一致的事务嵌套模式。

正确的解决方法是确保外部事务也被回滚：

```py
transaction1.rollback()
```

此模式在 Core 中并不常用。在 ORM 中，可能会出现类似的问题，这是 ORM 的“逻辑”事务结构的产物；这在 FAQ 条目中有描述“由于刷新期间的先前异常，此会话的事务已回滚。”（或类似）。

在 SQLAlchemy 2.0 中，已删除“子事务”模式，因此这种特定的编程模式不再可用，从而避免了这个错误消息。### 在 SQLAlchemy 2.0 中，<某个函数>将不再<某事>

SQLAlchemy 2.0 对于核心和 ORM 组件中的许多关键 SQLAlchemy 使用模式都表示了一个重大转变。2.0 版本的目标是在 SQLAlchemy 从一开始的基本假设中进行一些轻微调整，并提供一个新的简化的使用模型，希望在核心和 ORM 组件之间更加一致和简约，并且更具有能力。

在 SQLAlchemy 2.0 - 主要迁移指南 中介绍的 SQLAlchemy 2.0 项目包括一个综合的未来兼容性系统，该系统集成到 SQLAlchemy 1.4 系列中，以便应用程序能够清晰、明确地、逐步地升级到完全兼容 2.0 版本。`RemovedIn20Warning` 弃用警告是这个系统的基础，它提供了对现有代码库中需要修改的行为的指导。关于如何启用此警告的概述在 SQLAlchemy 2.0 弃用模式 中。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南 - 从 1.x 系列的升级过程的概述，以及 SQLAlchemy 2.0 的当前目标和进展。

SQLAlchemy 2.0 弃用模式 - 如何在 SQLAlchemy 1.4 中使用“2.0 弃用模式”的具体指南。

### 对象正被合并到会话中，沿着反向引用级联

此消息指的是 SQLAlchemy 的“backref cascade”行为，在 2.0 版本中已删除。这是指对象作为已经存在于该会话中的另一个对象的关联而被添加到 `Session` 中的操作。由于这种行为显示出比有帮助更加令人困惑，添加了 `relationship.cascade_backrefs` 和 `backref.cascade_backrefs` 参数，可以将其设置为 `False` 以禁用它，并且在 SQLAlchemy 2.0 中已完全删除“级联反向引用”行为。

对于较旧的 SQLAlchemy 版本，要在当前使用 `relationship.backref` 字符串参数配置的反向引用上将 `relationship.cascade_backrefs` 设置为 `False`，必须首先使用 `backref()` 函数声明反向引用，以便传递 `backref.cascade_backrefs` 参数。

或者，可以通过在“未来”模式下使用 `Session`，将整个“级联反向引用”行为全部关闭，通过为 `Session.future` 参数传递 `True`。

另请参阅

级联反向引用行为在 2.0 中已弃用 - SQLAlchemy 2.0 变更的背景。

### 以“传统”模式创建的 select() 构造；关键字参数等。

`select()` 构造已在 SQLAlchemy 1.4 中更新，以支持在 SQLAlchemy 2.0 中标准的新调用风格。为了向后兼容 1.4 系列，该构造接受“传统”风格和“新”风格的参数。

“新”风格的特点是列和表达式仅以位置方式传递给 `select()` 构造；对象的任何其他修饰符必须使用后续方法链接传递：

```py
# this is the way to do it going forward
stmt = select(table1.c.myid).where(table1.c.myid == table2.c.otherid)
```

作为对比，在 SQLAlchemy 的传统形式中，即使在添加像 `Select.where()` 这样的方法之前，`select()` 会像这样：

```py
# this is how it was documented in original SQLAlchemy versions
# many years ago
stmt = select([table1.c.myid], whereclause=table1.c.myid == table2.c.otherid)
```

或者甚至“whereclause”将以位置方式传���：

```py
# this is also how it was documented in original SQLAlchemy versions
# many years ago
stmt = select([table1.c.myid], table1.c.myid == table2.c.otherid)
```

多年来，大多数叙述性文档中接受的额外“whereclause”和其他参数已被移除，导致调用风格最为熟悉的是作为列参数传递的列表，但没有其他参数：

```py
# this is how it's been documented since around version 1.0 or so
stmt = select([table1.c.myid]).where(table1.c.myid == table2.c.otherid)
```

select() no longer accepts varied constructor arguments, columns are passed positionally 中的文档描述了这一变化，涉及 2.0 迁移。

另请参阅

select() no longer accepts varied constructor arguments, columns are passed positionally

SQLAlchemy 2.0 - 重大迁移指南

### 通过传统的绑定元数据找到了一个绑定，但由于此会话设置了 future=True，因此会忽略此绑定。

“绑定元数据”的概念一直存在于 SQLAlchemy 1.4 之前；从 SQLAlchemy 2.0 开始已将其删除。

此错误指的是`MetaData.bind`参数，该参数位于`MetaData`对象上，该对象允许像 ORM `Session`这样的对象将特定的映射类与`Engine`关联起来。在 SQLAlchemy 2.0 中，`Session`必须直接与每个`Engine`关联。也就是说，不要实例化`Session`或`sessionmaker`而不带任何参数，并将`Engine`与`MetaData`关联：

```py
engine = create_engine("sqlite://")
Session = sessionmaker()
metadata_obj = MetaData(bind=engine)
Base = declarative_base(metadata=metadata_obj)

class MyClass(Base): ...

session = Session()
session.add(MyClass())
session.commit()
```

相反，`Engine`必须直接与`sessionmaker`或`Session`关联。`MetaData`对象不应再与任何引擎相关联：

```py
engine = create_engine("sqlite://")
Session = sessionmaker(engine)
Base = declarative_base()

class MyClass(Base): ...

session = Session()
session.add(MyClass())
session.commit()
```

在 SQLAlchemy 1.4 中，当在`sessionmaker`或`Session`上设置了`Session.future`标志时，将启用此 2.0 样式行为。

### 此 Compiled 对象未绑定到任何 Engine 或 Connection

此错误指的是“绑定元数据”的概念，这是一个仅在 1.x 版本中存在的传统 SQLAlchemy 模式。当直接从未与任何`Engine`相关联的 Core 表达式对象上调用`Executable.execute()`方法时，就会出现此问题：

```py
metadata_obj = MetaData()
table = Table("t", metadata_obj, Column("q", Integer))

stmt = select(table)
result = stmt.execute()  # <--- raises
```

逻辑期望的是`MetaData`对象已经与`Engine`**绑定**：

```py
engine = create_engine("mysql+pymysql://user:pass@host/db")
metadata_obj = MetaData(bind=engine)
```

在上述情况中，任何从`Table`派生的语句，其又派生自`MetaData`的语句，将隐式使用给定的`Engine`来调用该语句。

请注意，在 SQLAlchemy 2.0 中**不存在**绑定元数据的概念。调用语句的正确方式是通过`Connection.execute()`方法的一个`Connection`：

```py
with engine.connect() as conn:
    result = conn.execute(stmt)
```

当使用 ORM 时，可以通过`Session`提供类似的功能：

```py
result = session.execute(stmt)
```

请参阅

语句执行基础

### 此连接处于非活动事务状态。请在继续之前完全 rollback()。

此错误条件已添加到 SQLAlchemy 自 1.4 版本以来，并且不适用于 SQLAlchemy 2.0。该错误是指将`Connection`放入事务中，使用类似`Connection.begin()`的方法，然后在该范围内创建一个进一步的“标记”事务；然后使用`Transaction.rollback()`回滚“标记”事务，或使用`Transaction.close()`关闭它，但是外部事务仍然以“非活动”状态存在，必须回滚。

模式如下：

```py
engine = create_engine(...)

connection = engine.connect()
transaction1 = connection.begin()

# this is a "sub" or "marker" transaction, a logical nesting
# structure based on "real" transaction transaction1
transaction2 = connection.begin()
transaction2.rollback()

# transaction1 is still present and needs explicit rollback,
# so this will raise
connection.execute(text("select 1"))
```

在上述代码中，`transaction2` 是一个“标记”事务，它表示外部事务内部的逻辑嵌套；而内部事务可以通过其 rollback()方法回滚整个事务，但是其 commit()方法除了关闭“标记”事务本身的范围外，并不产生任何效果。调用`transaction2.rollback()`的效果是**停用**transaction1，这意味着它在数据库级别上基本上已被回滚，但仍然存在以适应一致的事务嵌套模式。

正确的解决方法是确保外部事务也被回滚：

```py
transaction1.rollback()
```

此模式在核心中不常用。在 ORM 中，可能会出现类似的问题，这是 ORM 的“逻辑”事务结构的产物；这在常见问题解答条目中有描述：“此会话的事务由于刷新期间的先前异常而已被回滚。”（或类似）。

“子事务”模式在 SQLAlchemy 2.0 中被移除，因此这种特定的编程模式不再可用，从而防止了这个错误消息的出现。
