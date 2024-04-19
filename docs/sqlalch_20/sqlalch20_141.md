# SQLAlchemy 1.4 有什么新特性？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_14.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_14.html)

关于本文档

本文描述了 SQLAlchemy 版本 1.3 和 SQLAlchemy 版本 1.4 之间的变化。

版本 1.4 的重点与其他 SQLAlchemy 版本不同，它在很多方面试图作为潜在的迁移点，用于当前计划发布的 SQLAlchemy 2.0 中更为重大的一系列 API 更改。SQLAlchemy 2.0 的重点是现代化和精简的 API，删除了长期以来被不鼓励的许多使用模式，并将 SQLAlchemy 中的最佳思想作为一流 API 功能，目标是 API 的使用方式更加明确，以及删除一系列隐式行为和很少使用的 API 标志，这些都会使内部复杂化并阻碍性能。

有关 SQLAlchemy 2.0 的当前状态，请参阅 SQLAlchemy 2.0 - 主要迁移指南。

## 主要 API 更改和功能 - 通用

### Python 3.6 是最低要求的 Python 3 版本；仍支持 Python 2.7

由于 Python 3.5 在 2020 年 9 月已达到生命周期终点，因此 SQLAlchemy 1.4 现在将版本 3.6 作为最低要求的 Python 3 版本。Python 2.7 仍然受支持，但是 SQLAlchemy 1.4 系列将是最后一个支持 Python 2 的系列。### ORM 查询在内部与 select、update、delete 统一；2.0 风格的执行可用

对于 SQLAlchemy 2.0 和本质上也是 1.4 版本的最大概念变化是，在 Core 中的`Select`构造和 ORM 中的`Query`对象之间的巨大分离已被移除，以及在它们与`Update`和`Delete`之间的`Query.update()`和`Query.delete()`方法之间的分离。

关于`Select`和`Query`，这两个对象在许多版本中具有类似的、大部分重叠的 API，甚至有一些能够在两者之间切换的能力，但在使用模式和行为上仍然有很大的不同。这一历史背景是，`Query`对象是为了克服`Select`对象的缺点而引入的，后者曾经是 ORM 对象查询的核心，只是它们必须以`Table`元数据的形式进行查询。然而，`Query`只有一个简单的接口来加载对象，只有在许多主要版本的发布过程中，它最终才获得了大部分`Select`对象的灵活性，这导致这两个对象变得非常相似，但仍然在很大程度上不兼容。

在版本 1.4 中，所有核心和 ORM SELECT 语句都直接从`Select`对象呈现；当使用`Query`对象时，在语句调用时，它会将其状态复制到一个`Select`对象中，然后使用 2.0 风格执行。未来，`Query`对象将仅成为传统，应用程序将被鼓励转向 2.0 风格执行，允许核心构造自由地针对 ORM 实体使用：

```py
with Session(engine, future=True) as sess:
    stmt = (
        select(User)
        .where(User.name == "sandy")
        .join(User.addresses)
        .where(Address.email_address.like("%gmail%"))
    )

    result = sess.execute(stmt)

    for user in result.scalars():
        print(user)
```

以上示例的注意事项：

+   `Session`和`sessionmaker`对象现在具有完整的上下文管理器（即`with:`语句）功能；请参阅打开和关闭会话的修订文档以获取示例。

+   在 1.4 系列中，所有 2.0 风格的 ORM 调用都使用一个包含`Session`的对象，其中包括设置为`True`的`Session.future`标志；这个标志表示`Session`应该具有 2.0 风格的行为，其中包括 ORM 查询可以从`execute`中调用，以及一些事务特性的变化。在 2.0 版本中，这个标志将始终为`True`。

+   `select()`构造不再需要在列子句周围加括号；有关此改进的背景，请参阅 select(), case()现在接受位置表达式。

+   `select()` / `Select`对象具有一个`Select.join()`方法，其行为类似于`Query`的方法，甚至可以容纳 ORM 关系属性（而不会破坏 Core 和 ORM 之间的分离！）- 有关此内容，请参阅 select().join()和 outerjoin()向当前查询添加 JOIN 条件，而不是创建子查询。

+   与 ORM 实体一起工作并预计返回 ORM 结果的语句是使用`Session.execute()`来调用的。查看查询以获取入门指南。另请参阅 ORM Session.execute()在所有情况下使用“future”风格结果集中的以下注意事项。

+   返回一个`Result`对象，而不是一个普通列表，这本身是以前`ResultProxy`对象的一个更复杂的版本；这个对象现在用于 Core 和 ORM 结果。有关此信息，请参阅新的 Result 对象，RowProxy 不再是“代理”；现在称为 Row，并且行为类似于增强的命名元组，以及 Query 返回的“KeyedTuple”对象被 Row 替换。

在整个 SQLAlchemy 的文档中，将会有许多关于 1.x 风格 和 2.0 风格 执行的引用。这是为了区分这两种查询风格，并尝试在前进过程中向前文档化新的调用风格。在 SQLAlchemy 2.0 中，虽然 `Query` 对象可能仍然保留为遗留构造，但它将不再在大多数文档中显示。

类似的调整已经针对“批量更新和删除”进行了，以便 Core 的 `update()` 和 `delete()` 可用于批量操作。如下所示的批量更新：

```py
session.query(User).filter(User.name == "sandy").update(
    {"password": "foobar"}, synchronize_session="fetch"
)
```

可以以 2.0 风格（事实上，上述代码在内部以此方式运行）实现如下：

```py
with Session(engine, future=True) as sess:
    stmt = (
        update(User)
        .where(User.name == "sandy")
        .values(password="foobar")
        .execution_options(synchronize_session="fetch")
    )

    sess.execute(stmt)
```

请注意使用 `Executable.execution_options()` 方法传递 ORM 相关选项。现在“执行选项”的使用在 Core 和 ORM 中更为普遍，并且许多来自 `Query` 的 ORM 相关方法现在被实现为执行选项（请参阅 `Query.execution_options()` 查看一些示例）。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南

[#5159](https://www.sqlalchemy.org/trac/ticket/5159)  ### ORM `Session.execute()` 在所有情况下都使用“future”风格的 `Result` 集

如 RowProxy 现在不再是“代理”；现在被称为 Row 并且像增强型命名元组一样行为 所述，当与一个包含 `create_engine.future` 参数设置为 `True` 的 `Engine` 一起使用时，`Result` 和 `Row` 对象现在具有“命名元组”行为。这些“命名元组”行特别包括一种行为变化，即使用 `in` 的 Python 包含表达式，如下所示：

```py
>>> engine = create_engine("...", future=True)
>>> conn = engine.connect()
>>> row = conn.execute.first()
>>> "name" in row
True
```

上述包含测试将使用 **值包含**，而不是 **键包含**；`row` 需要具有“name”的 **value** 才能返回 `True`。

在 SQLAlchemy 1.4 版本下，当`create_engine.future`参数设置为`False`时，将返回遗留风格的`LegacyRow`对象，其具有之前 SQLAlchemy 版本的部分命名元组行为，其中包含性检查仍然使用键包含；如果行中有名为“name”的**列**，则`"name" in row`将返回 True，而不是一个值。

当使用`Session.execute()`时，完整的命名元组样式被**无条件**启用，意味着`"name" in row`将使用**值包含**作为测试，而**不是**键包含。这是为了适应`Session.execute()`现在返回一个`Result`，该结果还适应 ORM 结果，即使是由`Query.all()`返回的遗留 ORM 结果行也使用值包含。

这是从 SQLAlchemy 1.3 到 1.4 的行为变化。要继续接收键包含集合，请使用`Result.mappings()`方法接收返回行的`MappingResult`作为字典：

```py
for dict_row in session.execute(text("select id from table")).mappings():
    assert "id" in dict_row
```  ### 透明 SQL 编译缓存添加到 Core，ORM 中的所有 DQL，DML 语句

这是单个 SQLAlchemy 版本中最广泛的变化之一，经过数月的重新组织和重构，从 Core 的基础到 ORM，现在允许大多数涉及从用户构造的语句生成 SQL 字符串和相关语句元数据的 Python 计算被缓存在内存中，因此对于相同的语句构造的后续调用将使用 35-60%更少的 CPU 资源。

此缓存不仅限于构造 SQL 字符串，还包括构造将 SQL 结构链接到结果集的结果获取结构，而在 ORM 中，它还包括适应 ORM 启用的属性加载程序、关系急加载程序和其他选项，以及每次 ORM 查询试图运行并从结果集构造 ORM 对象时必须构建的对象构造例程。

为了介绍该功能的一般概念，给出来自 Performance 套件的代码如下，它将调用一个非常简单的查询“n”次，默认值为 n=10000。查询仅返回一行，因为我们要减少的开销是**许多小查询**的开销。对于返回许多行的查询，优化并不那么显著：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    result = session.query(Customer).filter(Customer.id == id_).one()
```

在运行 Linux 的 Dell XPS13 上的 SQLAlchemy 1.3 版本中，此示例完成如下：

```py
test_orm_query : (10000 iterations); total time 3.440652 sec
```

在 1.4 版本中，上述代码无需修改即可完成：

```py
test_orm_query : (10000 iterations); total time 2.367934 sec
```

这个第一个测试表明，当使用缓存时，常规 ORM 查询在许多迭代中可以运行得**快 30%**。

该功能的第二个变体是可选使用 Python lambda 来延迟查询本身的构建。这是“Baked Query”扩展所使用的方法的更复杂变体，该扩展是在 1.0.0 版本中引入的。 “lambda”功能可以以与烘焙查询非常相似的方式使用，只是它以一种临时方式适用于任何 SQL 构造。它还包括扫描每次调用 lambda 以查找在每次调用时更改的绑定文字值的能力，以及对其他构造的更改，例如每次查询来自不同实体或列，同时仍然无需每次运行实际代码。

使用这个 API 如下所示：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    stmt = lambda_stmt(lambda: future_select(Customer))
    stmt += lambda s: s.where(Customer.id == id_)
    session.execute(stmt).scalar_one()
```

上述代码完成：

```py
test_orm_query_newstyle_w_lambdas : (10000 iterations); total time 1.247092 sec
```

这个测试表明，使用较新的“select()”风格的 ORM 查询，结合完整的“baked”风格调用，可以在许多迭代中运行得**快 60%**，并且提供与现在被本地缓存系统取代的烘焙查询系统大致相同的性能。

新系统利用现有的`Connection.execution_options.compiled_cache`执行选项，并直接向`Engine`添加了一个缓存，该缓存使用`Engine.query_cache_size`参数进行配置。

1.4 版本中的 API 和行为变化的重要部分是为了支持这一新功能。

另请参阅

SQL 编译缓存

[#4639](https://www.sqlalchemy.org/trac/ticket/4639) [#5380](https://www.sqlalchemy.org/trac/ticket/5380) [#4645](https://www.sqlalchemy.org/trac/ticket/4645) [#4808](https://www.sqlalchemy.org/trac/ticket/4808) [#5004](https://www.sqlalchemy.org/trac/ticket/5004)  ### 声明式现在已经与新功能整合到 ORM 中

大约十年后，`sqlalchemy.ext.declarative`包现在已经整合到`sqlalchemy.orm`命名空间中，除了保留为声明式扩展的声明式“extension”类。

新添加到`sqlalchemy.orm`的类包括：

+   `registry` - 一个新类，取代了“declarative base”类的角色，作为映射类的注册表，可以通过字符串名称在`relationship()`调用中引用，并且不受任何特定类映射样式的限制。

+   `declarative_base()` - 这是在声明系统跨度期间一直在使用的相同声明基类，只是现在在内部引用了一个`registry`对象，并由`registry.generate_base()`方法实现，可以直接从`registry`调用。`declarative_base()`函数会自动创建此注册表，因此对现有代码没有影响。当启用 2.0 deprecations mode 时，`sqlalchemy.ext.declarative.declarative_base`名称仍然存在，发出 2.0 弃用警告。

+   `declared_attr()` - 同样是“declared attr”函数调用现在成为`sqlalchemy.orm`的一部分。当启用 2.0 deprecations mode 时，`sqlalchemy.ext.declarative.declared_attr`名称仍然存在，发出 2.0 弃用警告。

+   其他名称移至`sqlalchemy.orm`，包括`has_inherited_table()`，`synonym_for()`，`DeclarativeMeta`，`as_declarative()`。

另外，`instrument_declarative()`函数已被弃用，被`registry.map_declaratively()`取代。`ConcreteBase`、`AbstractConcreteBase`和`DeferredReflection`类仍然作为声明性扩展包中的扩展。

映射样式现在已经组织起来，它们都从`registry`对象扩展，并分为以下几类：

+   声明性映射

    +   使用`declarative_base()`基类与元类

        +   具有 mapped_column()的声明性表

        +   命令式表（又名“混合表”）

    +   使用`registry.mapped()`声明性装饰器

        +   声明性表

        +   命令式表（混合）

            +   将 ORM 映射应用于现有数据类（传统数据类用法）

+   命令式（又名“经典”映射）

    +   使用`registry.map_imperatively()`

        +   使用命令式映射映射预先存在的数据类

现有的经典映射函数`sqlalchemy.orm.mapper()`仍然存在，但不建议直接调用`sqlalchemy.orm.mapper()`；新的`registry.map_imperatively()`方法现在通过`sqlalchemy.orm.registry()`路由请求，以便与其他声明性映射明确集成。

这种新方法与第三方类仪器系统相互操作，这些系统必须在映射过程之前对类进行必要的操作，允许声明性映射通过装饰器而不是声明性基础工作，以便像[dataclasses](https://docs.python.org/3/library/dataclasses.html)和[attrs](https://pypi.org/project/attrs/)这样的包可以与声明性映射一起使用，除了与经典映射一起使用。

声明文档现在已完全集成到 ORM 映射器配置文档中，并包括对所有样式映射的示例，组织到一个地方。请参阅 ORM 映射类概述部分，开始新的重新组织的文档。

另请参阅

ORM 映射类概述

Python Dataclasses、attrs 支持声明性、命令式映射

[#5508](https://www.sqlalchemy.org/trac/ticket/5508)  ### 使用 Python Dataclasses、attrs 支持声明性、命令式映射

随着声明性现在已经与新特性集成到 ORM 中的新装饰器样式，`Mapper`现在明确地了解 Python 的`dataclasses`模块，并将识别配置为此方式的属性，并继续映射它们，而不是像以前那样跳过它们。对于`attrs`模块，`attrs`已经从类中删除了自己的属性，因此已经与 SQLAlchemy 的经典映射兼容。通过添加`registry.mapped()`装饰器，两种属性系统现在也可以与声明性映射互操作。

另请参阅

将 ORM 映射应用于现有的数据类（传统数据类使用）

使用命令式映射映射预先存在的数据类

[#5027](https://www.sqlalchemy.org/trac/ticket/5027)  ### 核心和 ORM 的异步 IO 支持

SQLAlchemy 现在支持使用全新的异步 IO 前端接口来使用 Python `asyncio`兼容的数据库驱动程序，用于 Core 使用的`Connection`以及用于 ORM 使用的`Session`，使用`AsyncConnection`和`AsyncSession`对象。

注意

新的 asyncio 功能应该被视为**alpha 级别**，适用于 SQLAlchemy 1.4 的初始版本。这是一些使用了一些以前不熟悉的编程技术的全新东西。

初始支持的数据库 API 是 asyncpg 用于 PostgreSQL 的 asyncio 驱动程序。

SQLAlchemy 的内部功能完全集成了[greenlet](https://greenlet.readthedocs.io/en/latest/)库，以便调整执行流程，从数据库驱动程序向最终用户 API 传播 asyncio `await` 关键字，该 API 具有 `async` 方法。使用这种方法，asyncpg 驱动程序在 SQLAlchemy 的测试套件中完全可用，并且与大多数 psycopg2 功能兼容。这种方法经过了 greenlet 项目的开发人员的审查和改进，对此 SQLAlchemy 表示感激。

用户面向的 `async` API 本身侧重于 IO 导向的方法，如`AsyncEngine.connect()`和`AsyncConnection.execute()`。新的 Core 结构严格支持 2.0 风格的使用方式；这意味着所有语句必须在给定连接对象的情况下调用，即`AsyncConnection`。

在 ORM 中，支持 2.0 风格的查询执行，使用`select()`结构与`AsyncSession.execute()`结合使用；传统的`Query`对象本身不受`AsyncSession`类支持。

ORM 功能，如延迟加载相关属性以及过期属性的取消，根据定义在传统的 asyncio 编程模型中是不允许的，因为它们表示会在 Python `getattr()` 操作的范围内隐式运行的 IO 操作。为了克服这一点，**传统**的 asyncio 应用程序应该巧妙地利用 eager loading 技术，并放弃使用诸如 expire on commit 之类的功能，以便不需要这样的加载。

对于选择与传统**决裂**的 asyncio 应用程序开发人员，新的 API 提供了一个**严格可选的功能**，使希望利用此类 ORM 功能的应用程序可以选择将与数据库相关的代码组织成函数，然后使用 `AsyncSession.run_sync()` 方法在 greenlets 中运行。查看 Asyncio Integration 中的 `greenlet_orm.py` 示例以进行演示。

还提供了对异步游标的支持，使用新方法 `AsyncConnection.stream()` 和 `AsyncSession.stream()`，支持一个新的 `AsyncResult` 对象，该对象本身提供了常见方法的可等待版本，如 `AsyncResult.all()` 和 `AsyncResult.fetchmany()`。核心和 ORM 都与传统 SQLAlchemy 中“服务器端游标”的使用对应的功能集成在一起。

另请参阅

异步 I/O（asyncio）

Asyncio Integration

[#3414](https://www.sqlalchemy.org/trac/ticket/3414)  ### 许多核心和 ORM 语句对象现在在编译阶段执行大部分构建和验证操作

1.4 系列中的一个重要举措是接近核心 SQL 语句和 ORM 查询的模型，以实现高效、可缓存的语句创建和编译模型，其中编译步骤将被缓存，基于创建的语句对象生成的缓存键，该对象本身为每次使用新创建。为实现这一目标，特别是在构建语句时发生的大部分 Python 计算，特别是 ORM `Query` 和 `select()` 构造在用于调用 ORM 查询时，正在移动到语句的编译阶段中，该阶段仅在调用语句后发生，且仅在语句的编译形式尚未被缓存时才会发生。

从最终用户的角度来看，这意味着基于传递给对象的参数可能引发的某些错误消息将不再立即引发，而是仅在首次调用语句时发生。这些条件始终是结构性的，而不是数据驱动的，因此不会因为缓存语句而错过这种条件。

属于此类别的错误条件包括：

+   当构造`_selectable.CompoundSelect`（例如 UNION，EXCEPT 等）并且传递的 SELECT 语句列数不同时，现在会引发`CompileError`；以前，在语句构造时会立即引发`ArgumentError`。

+   调用`Query.join()`时可能出现的各种错误条件将在语句编译时进行评估，而不是在首次调用方法时。

可能发生变化的其他事情涉及直接操作`Query`对象：

+   调用`Query.statement`访问器时行为可能略有不同。返回的`Select`对象现在是与`Query`中存在的相同状态的直接副本，而不执行任何 ORM 特定的编译（这意味着速度大大提高）。但是，该`Select`将不具有与 1.3 版本中相同的内部状态，包括如果在`Query`中未明确声明，则明确拼写出 FROM 子句。这意味着依赖于操作此`Select`语句的代码，例如调用`Select.with_only_columns()`方法，可能需要适应 FROM 子句。

另请参见

透明 SQL 编译缓存添加到 Core，ORM 中的所有 DQL，DML 语句  ### 修复了内部导入约定，使代码检查工具可以正常工作

SQLAlchemy 长期以来一直使用参数注入装饰器来帮助解决相互依赖的模块导入，就像这样：

```py
@util.dependency_for("sqlalchemy.sql.dml")
def insert(self, dml, *args, **kw): ...
```

上述函数将被重写，不再在外部具有`dml`参数。这会让代码检查工具看到函数缺少参数而感到困惑。已经内部实现了一种新方法，使函数的签名不再被修改，而是在函数内部获取模块对象。

[#4656](https://www.sqlalchemy.org/trac/ticket/4656)

[#4689](https://www.sqlalchemy.org/trac/ticket/4689)  ### 支持 SQL 正则表达式操作符

期待已久的功能是为数据库正则表达式操作符添加基本支持，以补充`ColumnOperators.like()`和`ColumnOperators.match()`操作套件。新功能包括`ColumnOperators.regexp_match()`实现了类似正则表达式匹配的功能，以及`ColumnOperators.regexp_replace()`实现了正则表达式字符串替换功能。

支持的后端包括 SQLite、PostgreSQL、MySQL / MariaDB 和 Oracle。SQLite 后端仅支持“regexp_match”而不支持“regexp_replace”。

正则表达式语法和标志**不是通用于所有后端**。未来的功能将允许一次指定多个正则表达式语法，以便在不同后端之间动态切换。

对于 SQLite，Python 的`re.search()`函数没有额外的参数被确定为实现。

另请参阅

`ColumnOperators.regexp_match()`

`ColumnOperators.regexp_replace()`

正则表达式支持 - SQLite 实现注意事项

[#1390](https://www.sqlalchemy.org/trac/ticket/1390)  ### SQLAlchemy 2.0 弃用模式

1.4 版本的主要目标之一是提供一个“过渡”版本，以便应用程序可以逐渐迁移到 SQLAlchemy 2.0。为此，1.4 版本的一个主要特性是“2.0 弃用模式”，这是一系列针对每个可检测到的 API 模式发出的弃用警告，在 2.0 版本中将以不同方式工作。所有警告都使用`RemovedIn20Warning`类。由于这些警告影响到包括`select()` 和`Engine` 构造在内的基础模式，即使是简单的应用程序也可能生成大量警告，直到适当的 API 更改完成。因此，警告模式默认关闭，直到开发人员启用环境变量`SQLALCHEMY_WARN_20=1`。

要全面了解如何使用 2.0 弃用模式，请参阅迁移到 2.0 步骤二 - 打开 RemovedIn20Warnings。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南

迁移到 2.0 步骤二 - 打开 RemovedIn20Warnings

## API 和行为变化 - 核心

### SELECT 语句不再隐式地被视为 FROM 子句

这个变化是多年来 SQLAlchemy 中较大的概念性变化之一，但希望最终用户的影响相对较小，因为这个变化更符合像 MySQL 和 PostgreSQL 这样的数据库实际需要。

最直接显著的影响是，一个`select()` 现在不能直接嵌套在另一个`select()` 中，而需要显式地先将内部的`select()` 转换为子查询。这在历史上是通过使用`SelectBase.alias()` 方法来实现的，该方法仍然存在，但更适合使用一个新方法`SelectBase.subquery()`；两种方法都是做同样的事情。现在返回的对象是`Subquery`，它与`Alias`对象非常相似，并共享一个共同的基类`AliasedReturnsRows`。

换句话说，现在会引发：

```py
stmt1 = select(user.c.id, user.c.name)
stmt2 = select(addresses, stmt1).select_from(addresses.join(stmt1))
```

引发：

```py
sqlalchemy.exc.ArgumentError: Column expression or FROM clause expected,
got <...Select object ...>. To create a FROM clause from a <class
'sqlalchemy.sql.selectable.Select'> object, use the .subquery() method.
```

正确的调用形式应该是（还要注意 select()不再需要括号）：

```py
sq1 = select(user.c.id, user.c.name).subquery()
stmt2 = select(addresses, sq1).select_from(addresses.join(sq1))
```

注意`SelectBase.subquery()`方法本质上等同于使用`SelectBase.alias()`方法。

这一变化的理由如下：

+   为了支持`Select`与`Query`的统一，`Select`对象需要具有实际添加 JOIN 条件到现有 FROM 子句的`Select.join()`和`Select.outerjoin()`方法，这正是用户一直期望它做的事情。先前的行为是，必须与`FromClause`一致，它会生成一个无名子查询，然后 JOIN 到它，这是一个完全没有用的功能，只会让那些不幸尝试的用户感到困惑。这一变化在 select().join() and outerjoin() add JOIN criteria to the current query, rather than creating a subquery 中讨论。

+   在另一个 SELECT 的 FROM 子句中包含 SELECT 而不先创建别名或子查询的行为将创建一个无名子查询。虽然标准 SQL 确实支持这种语法，但实际上大多数数据库都会拒绝它。例如，MySQL 和 PostgreSQL 都明确拒绝使用无名子查询：

    ```py
    #  MySQL  /  MariaDB:

    MariaDB  [(none)]>  select  *  from  (select  1);
    ERROR  1248  (42000):  Every  derived  table  must  have  its  own  alias

    #  PostgreSQL:

    test=>  select  *  from  (select  1);
    ERROR:  subquery  in  FROM  must  have  an  alias
    LINE  1:  select  *  from  (select  1);
      ^
    HINT:  For  example,  FROM  (SELECT  ...)  [AS]  foo.
    ```

    像 SQLite 这样的数据库接受它们，但通常情况下，从这样的子查询产生的名称太模糊，无法使用：

    ```py
    sqlite>  CREATE  TABLE  a(id  integer);
    sqlite>  CREATE  TABLE  b(id  integer);
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  ON  a.id=id;
    Error:  ambiguous  column  name:  id
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  ON  a.id=b.id;
    Error:  no  such  column:  b.id

    #  use  a  name
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  AS  anon_1  ON  a.id=anon_1.id;
    ```

由于`SelectBase`对象不再是`FromClause`对象，因此像`.c`属性和`.select()`方法这样的属性现在已被弃用，因为它们暗示着隐式生成子查询。`.join()`和`.outerjoin()`方法现在被重新用于在现有查询中添加 JOIN 条件，类似于`Query.join()`的方式，这正是用户一直期望这些方法做的事情。

在`.c`属性的位置，添加了一个新属性`SelectBase.selected_columns`。这个属性解析为一个列集合，大多数人希望`.c`做的事情（但实际上不是），即引用 SELECT 语句的列子句中的列。一个常见的初学者错误是以下代码：

```py
stmt = select(users)
stmt = stmt.where(stmt.c.name == "foo")
```

上述代码看起来很直观，似乎会生成“SELECT * FROM users WHERE name=’foo’”，然而，经验丰富的 SQLAlchemy 用户会意识到，实际上它生成了一个无用的子查询，类似于“SELECT * FROM (SELECT * FROM users) WHERE name=’foo’”。

然而，新的`SelectBase.selected_columns`属性确实适用于上述用例，因为在上述情况下，它直接链接到`users.c`集合中存在的列：

```py
stmt = select(users)
stmt = stmt.where(stmt.selected_columns.name == "foo")
```

[#4617](https://www.sqlalchemy.org/trac/ticket/4617)  ### select().join()和 outerjoin()将 JOIN 条件添加到当前查询，而不是创建子查询

为了实现 2.0 风格对`Select`的使用，特别是统一`Query`和`Select`的目标，关键是有一个工作的`Select.join()`方法，其行为类似于`Query.join()`方法，向现有 SELECT 的 FROM 子句添加额外条目，然后返回新的`Select`对象以进行进一步修改，而不是将对象包装在未命名的子查询中并从该子查询返回 JOIN，这种行为对用户来说一直是几乎无用和完全误导的。

为了实现这一点，不再将 SELECT 语句隐式视为 FROM 子句首先实现了这一点，将`Select`从必须是`FromClause`中分离出来；这消除了`Select.join()`需要返回一个`Join`对象而不是包含新 JOIN 的 FROM 子句的新版本`Select`对象的要求。

从那时起，由于`Select.join()`和`Select.outerjoin()`具有现有行为，最初的计划是这些方法将被弃用，并且这些方法的新“有用”版本将在一个备用的“未来”`Select`对象上作为单独的导入可用。

然而，在与这个特定代码库一段时间后，决定有两种不同类型的`Select`对象漂浮在周围，每个对象的行为几乎相同，只是某些方法的行为略有不同，这将比简单地改变这两种方法的行为更具误导性和不便，因为`Select.join()` 和 `Select.outerjoin()` 的现有行为基本上从未被使用，只会引起混乱。

因此，决定在这个领域做出**严格的行为改变**，而不是等待另一年并在此期间拥有更尴尬的 API，考虑到当前行为是多么无用，新行为将会是多么极其有用和重要。SQLAlchemy 开发人员并不轻易做出像这样完全破坏性的改变，然而这是一个非常特殊的情况，以前的这些方法实现几乎不太可能被使用；正如在 SELECT 语句不再隐式视为 FROM 子句 中所指出的，主要数据库如 MySQL 和 PostgreSQL 在任何情况下都不允许未命名的子查询，并且从语法角度来看，从未命名的子查询进行 JOIN 几乎是不可能有用的，因为很难明确地引用其中的列。

使用新的实现方式，`Select.join()` 和 `Select.outerjoin()` 现在的行为与 `Query.join()` 非常相似，通过匹配左实体来向现有语句添加 JOIN 条件：

```py
stmt = select(user_table).join(
    addresses_table, user_table.c.id == addresses_table.c.user_id
)
```

产生：

```py
SELECT  user.id,  user.name  FROM  user  JOIN  address  ON  user.id=address.user_id
```

与 `Join` 一样，如果可行，ON 子句将自动确定：

```py
stmt = select(user_table).join(addresses_table)
```

当在语句中使用 ORM 实体时，这基本上是使用 2.0 风格 调用构建 ORM 查询的方式。ORM 实体将在语句内部分配一个“插件”，以便在将语句编译成 SQL 字符串时发生 ORM 相关的编译规则。更直接地说，`Select.join()` 方法可以适应 ORM 关系，而不会破坏 Core 和 ORM 内部之间的严格分离：

```py
stmt = select(User).join(User.addresses)
```

另一个新方法`Select.join_from()`也被添加，它允许更容易地一次性指定连接的左侧和右侧：

```py
stmt = select(Address.email_address, User.name).join_from(User, Address)
```

产生：

```py
SELECT  address.email_address,  user.name  FROM  user  JOIN  address  ON  user.id  ==  address.user_id
```  ### URL 对象现在是不可变的

`URL`对象已经被正式规范化，现在它呈现为一个带有固定数量字段的不可变的`namedtuple`。此外，由`URL.query`属性表示的字典也是一个不可变映射。变异`URL`对象不是一个正式支持或记录的用例，这导致了一些开放式用例，使得很难拦截不正确的用法，最常见的是变异`URL.query`字典以包含非字符串元素。它还导致了在一个基本数据对象中允许可变性的所有常见问题，即不希望的变异泄漏到未预期 URL 会发生变化的代码中。最后，`namedtuple` 的设计灵感来自 Python 的`urllib.parse.urlparse()`，它将解析后的对象作为一个命名元组返回。

决定彻底更改 API 的基础是根据一个计算，权衡了无法实现逐步废弃路径（这将涉及更改`URL.query`字典为一个特殊字典，当调用任何标准库变异方法时会发出废弃警告，此外，当字典保存任何元素列表时，列表也必须在变异时发出废弃警告）与项目已经在第一次变异`URL`对象的不太可能使用案例相比，以及像[#5341](https://www.sqlalchemy.org/trac/ticket/5341)这样的小变化在任何情况下都会造成向后不兼容性。对于变异`URL`对象的主要案例是在`CreateEnginePlugin`扩展点内解析插件参数，这本身是一个相当新的添加，根据 Github 代码搜索的结果，有两个仓库在使用，但实际上都没有变异 URL 对象。

`URL`对象现在提供了检查和生成新的`URL`对象的丰富接口。创建`URL`对象的现有机制，即`make_url()`函数，保持不变：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
```

对于编程构造，如果代码可能直接使用`URL`构造函数或`__init__`方法，如果参数作为关键字参数而不是精确的 7 元组传递，将收到弃用警告。现在可以通过`URL.create()`方法使用关键字样式的构造函数：

```py
>>> from sqlalchemy.engine import URL
>>> url = URL.create("postgresql", "user", "pass", host="host", database="dbname")
>>> str(url)
'postgresql://user:pass@host/dbname'
```

通常可以使用`URL.set()`方法更改字段，该方法返回一个应用更改后的新`URL`对象：

```py
>>> mysql_url = url.set(drivername="mysql+pymysql")
>>> str(mysql_url)
'mysql+pymysql://user:pass@host/dbname'
```

要更改`URL.query`字典的内容，可以使用诸如`URL.update_query_dict()`之类的方法：

```py
>>> url.update_query_dict({"sslcert": "/path/to/crt"})
postgresql://user:***@host/dbname?sslcert=%2Fpath%2Fto%2Fcrt
```

要升级直接突变这些字段的代码，一个**向后和向前兼容的方法**是使用鸭子类型，如下所示：

```py
def set_url_drivername(some_url, some_drivername):
    # check for 1.4
    if hasattr(some_url, "set"):
        return some_url.set(drivername=some_drivername)
    else:
        # SQLAlchemy 1.3 or earlier, mutate in place
        some_url.drivername = some_drivername
        return some_url

def set_ssl_cert(some_url, ssl_cert):
    # check for 1.4
    if hasattr(some_url, "update_query_dict"):
        return some_url.update_query_dict({"sslcert": ssl_cert})
    else:
        # SQLAlchemy 1.3 or earlier, mutate in place
        some_url.query["sslcert"] = ssl_cert
        return some_url
```

查询字符串保留其现有格式，作为字符串到字符串的字典，使用字符串序列表示多个参数。例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url(
...     "postgresql://user:pass@host/dbname?alt_host=host1&alt_host=host2&sslcert=%2Fpath%2Fto%2Fcrt"
... )
>>> url.query
immutabledict({'alt_host': ('host1', 'host2'), 'sslcert': '/path/to/crt'})
```

要处理`URL.query`属性的内容，使所有值都归一化为序列，请使用`URL.normalized_query`属性：

```py
>>> url.normalized_query
immutabledict({'alt_host': ('host1', 'host2'), 'sslcert': ('/path/to/crt',)})
```

查询字符串可以通过`URL.update_query_dict()`、`URL.update_query_pairs()`、`URL.update_query_string()`等方法进行追加：

```py
>>> url.update_query_dict({"alt_host": "host3"}, append=True)
postgresql://user:***@host/dbname?alt_host=host1&alt_host=host2&alt_host=host3&sslcert=%2Fpath%2Fto%2Fcrt
```

另请参阅

`URL`

#### 对 CreateEnginePlugin 的更改

`CreateEnginePlugin` 也受到这一变化的影响，因为自定义插件的文档指出应该使用`dict.pop()`方法从 URL 对象中删除已使用的参数。现在应该使用`CreateEnginePlugin.update_url()` 方法来实现。向后兼容的方法如下：

```py
from sqlalchemy.engine import CreateEnginePlugin

class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        # check for 1.4 style
        if hasattr(CreateEnginePlugin, "update_url"):
            self.my_argument_one = url.query["my_argument_one"]
            self.my_argument_two = url.query["my_argument_two"]
        else:
            # legacy
            self.my_argument_one = url.query.pop("my_argument_one")
            self.my_argument_two = url.query.pop("my_argument_two")

        self.my_argument_three = kwargs.pop("my_argument_three", None)

    def update_url(self, url):
        # this method runs in 1.4 only and should be used to consume
        # plugin-specific arguments
        return url.difference_update_query(["my_argument_one", "my_argument_two"])
```

查看`CreateEnginePlugin`的文档字符串，了解如何使用该类的完整详细信息。

[#5526](https://www.sqlalchemy.org/trac/ticket/5526)  ### select(), case() 现在接受位置表达式

正如本文档中的其他地方所示，`select()` 构造现在将接受“columns clause”参数作为位置参数，而不需要将它们作为列表传递：

```py
# new way, supports 2.0
stmt = select(table.c.col1, table.c.col2, ...)
```

在将参数作为位置参数发送时，不允许其他关键字参数。在 SQLAlchemy 2.0 中，上述调用风格将是唯一支持的调用风格。

在 1.4 版本期间，先前的调用风格仍将继续运行，将列或其他表达式的列表作为列表传递：

```py
# old way, still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...])
```

上述传统调用风格还接受自那时起已从大多数叙述文档中删除的旧关键字参数。这些关键字参数的存在是为什么首先将 columns clause 作为列表传递的原因：

```py
# very much the old way, but still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...], whereclause=table.c.col1 == 5)
```

两种风格之间的区别在于第一个位置参数是否为列表。不幸的是，仍然可能存在一些使用情况看起来像以下这样，其中“whereclause”的关键字被省略：

```py
# very much the old way, but still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...], table.c.col1 == 5)
```

作为这一变化的一部分，`Select` 构造还获得了 2.0 风格的“future” API，其中包括更新的`Select.join()`方法以及诸如`Select.filter_by()`和`Select.join_from()`等方法。

在相关更改中，`case()` 构造也已经修改为接受其 WHEN 子句的列表作为位置参数，旧调用风格也有类似的弃用轨迹：

```py
stmt = select(users_table).where(
    case(
        (users_table.c.name == "wendy", "W"),
        (users_table.c.name == "jack", "J"),
        else_="E",
    )
)
```

对于 SQLAlchemy 构造函数接受`*args`与接受值列表的约定，例如`ColumnOperators.in_()`这样的构造函数，**位置参数用于结构规范，列表用于数据规范**。

另请参阅

select()不再接受各种构造函数参数，列按位置传递

在“遗留”模式中创建的 select()构造函数；关键字参数等。

[#5284](https://www.sqlalchemy.org/trac/ticket/5284)  ### 所有 IN 表达式都会动态生成列表中每个值的参数（例如，扩展参数）

“扩展 IN”功能首次在 晚扩展的 IN 参数集允许带有缓存语句的 IN 表达式 中引入，已经成熟到足以清楚地优于以前的渲染 IN 表达式的方法。随着该方法被改进以处理空值列表，它现在是 Core / ORM 用于渲染 IN 参数列表的唯一手段。

SQLAlchemy 自首次发布以来一直存在的先前方法是，当将值列表传递给`ColumnOperators.in_()`方法时，该列表将在语句构造时扩展为一系列单独的`BindParameter`对象。这种方法的局限性在于无法根据参数字典在语句执行时变化参数列表，这意味着无法独立缓存字符串 SQL 语句及其参数，也不能完全使用参数字典来处理通常包含 IN 表达式的语句。

为了服务于 Baked Queries 描述的“烘焙查询”功能，需要一个可缓存版本的 IN，这就引入了“扩展 IN”功能。与现有行为相反，现有行为是在语句构造时将参数列表展开为单独的`BindParameter`对象，该功能使用一个存储一次性值列表的`BindParameter`；当由`Engine`执行语句时，它会根据传递给`Connection.execute()`调用的参数，并根据以前执行时可能已经检索到的现有 SQL 字符串，使用正则表达式对其进行修改，以适应当前参数集。这允许相同的`Compiled`对象，该对象存储渲染的字符串语句，根据修改 IN 表达式的传递给多次调用的参数集，同时仍然保持将单个标量参数传递给 DBAPI 的行为。虽然某些 DBAPI 直接支持此功能，但通常不可用；“扩展 IN”功能现在为所有后端一致地支持行为。

1.4 的主要重点是在 Core 和 ORM 中允许真正的语句缓存，而不需要“烘焙”系统的笨拙性，而且由于“扩展 IN”功能代表了构建表达式的更简单方法，所以现在在传递值列表给 IN 表达式时自动调用它：

```py
stmt = select(A.id, A.data).where(A.id.in_([1, 2, 3]))
```

预执行字符串表示如下：

```py
>>> print(stmt)
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  ([POSTCOMPILE_id_1]) 
```

要直接渲染值，请像以前一样使用`literal_binds`：

```py
>>> print(stmt.compile(compile_kwargs={"literal_binds": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (1,  2,  3) 
```

添加了一个新标志，“render_postcompile”，作为帮助器，允许将当前绑定的值渲染为将要传递给数据库的样子：

```py
>>> print(stmt.compile(compile_kwargs={"render_postcompile": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (:id_1_1,  :id_1_2,  :id_1_3) 
```

引擎日志输出还显示了最终的渲染语句：

```py
INFO  sqlalchemy.engine.base.Engine  SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (?,  ?,  ?)
INFO  sqlalchemy.engine.base.Engine  (1,  2,  3)
```

作为这一变化的一部分，“空 IN”表达式的行为，其中列表参数为空，现在已经标准化为使用 IN 运算符针对所谓的“空集合”。由于没有空集合的标准 SQL 语法，因此使用返回零行的 SELECT，针对每个后端进行特定方式的定制，以便数据库将其视为空集合；此功能首次在版本 1.3 中引入，并在 扩展 IN 功能现在支持空列表 中进行了描述。在版本 1.2 中引入的 `create_engine.empty_in_strategy` 参数，作为迁移以前 IN 系统处理方式的手段，现已被弃用，此标志不再起作用；如 IN / NOT IN 运算符的空集合行为现在可配置；默认表达式简化 中所述，此标志允许方言在原始系统比较列与自身的情况下切换，这种情况被证明是一个巨大的性能问题，以及比较“1 != 1” 以产生“false”表达式的新系统。1.3 引入的行为现在在所有情况下都更为正确，比两种方法都更为正确，因为仍然使用 IN 运算符，并且不具有原始系统的性能问题。

此外，“扩展”参数系统已经泛化，以便还可以服务于其他特定于方言的用例，其中参数无法被 DBAPI 或后端数据库容纳；有关详细信息，请参见 Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数。

另请参见

Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数

扩展 IN 功能现在支持空列表

`BindParameter`

[#4645](https://www.sqlalchemy.org/trac/ticket/4645)  ### 内置 FROM 代码检查将警告任何 SELECT 语句中可能存在的笛卡尔积。

由于核心表达语言以及 ORM 建立在“隐式 FROMs”模型上，如果查询的任何部分引用了特定的 FROM 子句，那么该子句会自动添加，一个常见问题是 SELECT 语句的情况，无论是顶层语句还是嵌套子查询，包含了未与查询中的其他 FROM 元素连接的 FROM 元素，导致结果集中出现所谓的“笛卡尔积”，即每个未连接的 FROM 元素之间的所有可能行的组合。在关系数据库中，这几乎总是一个不良结果，因为它会产生一个充满重复、不相关数据的巨大结果集。

SQLAlchemy，尽管具有许多出色的功能，但特别容易出现这种问题，因为 SELECT 语句会自动从其他子句中看到的任何表中添加元素到其 FROM 子句中。一个典型的情况如下，其中两个表被 JOIN 在一起，然而在 WHERE 子句中可能无意中与这两个表不匹配的额外条目将创建一个额外的 FROM 条目：

```py
address_alias = aliased(Address)

q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
)
```

上面的查询从`User`和`address_alias`的 JOIN 中选择，后者是`Address`实体的别名。然而，`Address`实体在 WHERE 子句中直接使用，因此上述将导致 SQL：

```py
SELECT
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  addresses,  users  JOIN  addresses  AS  addresses_1  ON  users.id  =  addresses_1.user_id
WHERE  addresses.email_address  =  :email_address_1
```

在上面的 SQL 中，我们可以看到 SQLAlchemy 开发人员所谓的“可怕的逗号”，因为我们在 FROM 子句中看到“FROM addresses, users JOIN addresses”，这是笛卡尔积的经典迹象；查询正在使用 JOIN 来将 FROM 子句连接在一起，但是因为其中一个没有连接，它使用了逗号。上面的查询将返回一个完整的行集，将“user”和“addresses”表在“id / user_id”列上连接在一起，然后将所有这些行直接应用到“addresses”表中的每一行的笛卡尔积中。也就是说，如果有十个用户行和 100 个地址行，则上面的查询将返回其预期的结果行，可能为 100，因为所有地址行都将被选择，再乘以 100，因此总结果大小将为 10000 行。

“table1, table2 JOIN table3”模式在 SQLAlchemy ORM 中也经常出现，这要归因于 ORM 功能的微妙错误应用，特别是与连接式急加载或连接式表继承相关的功能，以及由于 SQLAlchemy ORM 中的错误而导致的问题。类似的问题也适用于使用“隐式连接”的 SELECT 语句，其中不使用 JOIN 关键字，而是通过 WHERE 子句将每个 FROM 元素与另一个元素链接起来。

多年来，维基上有一篇关于应用图算法到查询执行时的`select()`构造的配方，并检查查询的结构以寻找这些未链接的 FROM 子句，解析 WHERE 子句和所有 JOIN 子句以确定 FROM 元素如何相互连接，并确保所有 FROM 元素在单个图中连接。这个配方现已被调整为成为`SQLCompiler`的一部分，现在如果检测到此条件，它现在可选择发出警告。该警告使用`create_engine.enable_from_linting`标志启用，并且默认启用。linter 的计算开销非常低，而且它只发生在语句编译期间，这意味着对于缓存的 SQL 语句，它只会发生一次。

使用此功能，我们上面的 ORM 查询将发出警告：

```py
>>> q.all()
SAWarning: SELECT statement has a cartesian product between FROM
element(s) "addresses_1", "users" and FROM element "addresses".
Apply join condition(s) between each element to resolve.
```

linter 功能不仅适用于通过 JOIN 子句连接在一起的表，还适用于通过 WHERE 子句如上，我们可以添加一个 WHERE 子句来将新的`Address`实体与之前的`address_alias`实体链接起来，这将消除警告：

```py
q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
    .filter(Address.id == address_alias.id)
)  # resolve cartesian products,
# will no longer warn
```

笛卡尔积警告认为两个 FROM 子句之间的**任何**链接都是一个解决方案，即使最终结果集仍然是低效的，因为 linter 仅用于检测完全意外的 FROM 子句的常见情况。如果 FROM 子句在其他地方被明确引用并链接到其他 FROM 子句，则不会发出警告：

```py
q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
    .filter(Address.id > address_alias.id)
)  # will generate a lot of rows,
# but no warning
```

完整的笛卡尔积也是允许的，如果明确说明；例如，如果我们想要`User`和`Address`的笛卡尔积，我们可以在`true()`上进行 JOIN，以便每一行都与其他每一行匹配；以下查询将返回所有行并且不会产生警告：

```py
from sqlalchemy import true

# intentional cartesian product
q = session.query(User).join(Address, true())  # intentional cartesian product
```

默认情况下，只有在语句由`Connection`编译执行时才会生成警告；调用`ClauseElement.compile()`方法不会发出警告，除非提供了 linting 标志：

```py
>>> from sqlalchemy.sql import FROM_LINTING
>>> print(q.statement.compile(linting=FROM_LINTING))
SAWarning: SELECT statement has a cartesian product between FROM element(s) "addresses" and FROM element "users".  Apply join condition(s) between each element to resolve.
SELECT  users.id,  users.name,  users.fullname,  users.nickname
FROM  addresses,  users  JOIN  addresses  AS  addresses_1  ON  users.id  =  addresses_1.user_id
WHERE  addresses.email_address  =  :email_address_1 
```

[#4737](https://www.sqlalchemy.org/trac/ticket/4737)  ### 新 Result 对象

SQLAlchemy 2.0 的一个主要目标是统一 ORM 和 Core 之间如何处理“结果”的方式。为实现这一目标，版本 1.4 引入了自 SQLAlchemy 开始就存在的`ResultProxy`和`RowProxy`对象的新版本。

新对象的文档位于`Result`和`Row`，不仅用于核心结果集，还用于 ORM 中的 2.0 风格结果。

此结果对象与`ResultProxy`完全兼容，并包括许多新功能，现在对核心和 ORM 结果均应用，包括诸如：

`Result.one()` - 返回确切的单行，或引发异常：

```py
with engine.connect() as conn:
    row = conn.execute(table.select().where(table.c.id == 5)).one()
```

`Result.one_or_none()` - 相同，但对于没有行也返回 None

`Result.all()` - 返回所有行

`Result.partitions()` - 按块获取行：

```py
with engine.connect() as conn:
    result = conn.execute(
        table.select().order_by(table.c.id),
        execution_options={"stream_results": True},
    )
    for chunk in result.partitions(500):
        # process up to 500 records
        ...
```

`Result.columns()` - 允许对行进行切片和重新组织：

```py
with engine.connect() as conn:
    # requests x, y, z
    result = conn.execute(select(table.c.x, table.c.y, table.c.z))

    # iterate rows as y, x
    for y, x in result.columns("y", "x"):
        print("Y: %s X: %s" % (y, x))
```

`Result.scalars()` - 返回标量对象的列表，默认从第一列开始，但也可以选择：

```py
result = session.execute(select(User).order_by(User.id))
for user_obj in result.scalars():
    ...
```

`Result.mappings()` - 而不是命名元组行，返回字典：

```py
with engine.connect() as conn:
    result = conn.execute(select(table.c.x, table.c.y, table.c.z))

    for map_ in result.mappings():
        print("Y: %(y)s X: %(x)s" % map_)
```

在使用核心时，由`Connection.execute()`返回的对象是`CursorResult`的实例，其继续具有与`ResultProxy`相同的 API 功能，关于插入的主键、默认值、行数等。对于 ORM，将返回`Result`的子类，执行核心行到 ORM 行的转换，然后允许进行所有相同的操作。

另请参见

ORM 查询与核心选择统一 - 在 2.0 迁移文档中

[#5087](https://www.sqlalchemy.org/trac/ticket/5087)

[#4395](https://www.sqlalchemy.org/trac/ticket/4395)

[#4959](https://www.sqlalchemy.org/trac/ticket/4959)  ### RowProxy 不再是“代理”；现在称为 Row，并且行为类似于增强的命名元组

`RowProxy` 类，代表 Core 结果集中的单个数据库结果行，现在被称为 `Row`，不再是一个“代理”对象；这意味着当返回 `Row` 对象时，该行是一个简单的元组，其中包含数据的最终形式，已经通过与数据类型相关的结果行处理函数处理过（例如将数据库中的日期字符串转换为 `datetime` 对象，将 JSON 字符串转换为 Python 的 `json.loads()` 结果等）。

这样做的直接理由是为了使该行更像一个 Python 命名元组，而不是一个映射，其中元组中的值是元组上的 `__contains__` 运算符的主题，而不是键。由于 `Row` 表现得像一个命名元组，因此它适合用作 ORM 的 `KeyedTuple` 对象的替代，从而导致最终的 API 中，ORM 和 Core 提供的结果集行为相同。统一 ORM 和 Core 中的主要模式是 SQLAlchemy 2.0 的主要目标，而发布 1.4 旨在具有大多数或所有底层架构模式，以支持这一过程。Query 返回的`KeyedTuple`对象被 Row 替换 中的注释描述了 ORM 对 `Row` 类的使用。

对于发布 1.4 版本，`Row` 类提供了一个额外的子类 `LegacyRow`，它被 Core 使用，并提供了 `RowProxy` 的向后兼容版本，同时对那些将被移动的 API 功能和行为发出弃用警告。ORM `Query` 现在直接使用 `Row` 作为 `KeyedTuple` 的替代品。

`LegacyRow` 类是一个过渡类，其中 `__contains__` 方法仍然针对键进行测试，而不是值，当操作成功时会发出弃用警告。此外，先前 `RowProxy` 上的所有其他类似映射的方法也已弃用，包括 `LegacyRow.keys()`、`LegacyRow.items()` 等。对于从 `Row` 对象获得类似映射的行为，包括支持这些方法以及面向键的 `__contains__` 运算符，未来的 API 将是首先访问一个特殊属性 `Row._mapping`，然后该属性将为该行提供完整的映射接口，而不是元组接口。

#### 理念：表现得更像一个命名元组而不是映射

命名元组和映射之间在布尔运算方面的区别可以总结如下。给定伪代码中的“命名元组”为：

```py
row = (id: 5,  name: 'some name')
```

最大的不兼容差异是`__contains__`的行为：

```py
"id" in row  # True for a mapping, False for a named tuple
"some name" in row  # False for a mapping, True for a named tuple
```

在 1.4 版本中，当核心结果集返回一个`LegacyRow`时，上述`"id" in row`比较将继续成功，但会发出弃用警告。要将“in”运算符用作映射，请使用`Row._mapping`属性：

```py
"id" in row._mapping
```

SQLAlchemy 2.0 的结果对象将具有`.mappings()`修饰符，以便可以直接接收这些映射：

```py
# using sqlalchemy.future package
for row in result.mappings():
    row["id"]
```

#### 代理行为消失，对于现代用法也是不必要的

对`Row`的重构使其行为类似于元组，需要所有数据值一开始就完全可用。这是与`RowProxy`的内部行为变化不同，`RowProxy`中的结果行处理函数将在访问行的元素时被调用，而不是在首次获取行时被调用。这意味着例如从 SQLite 检索日期时间值时，以前在`RowProxy`对象中的行数据看起来像是：

```py
row_proxy = (1, "2019-12-31 19:56:58.272106")
```

然后通过`__getitem__`访问时，`datetime.strptime()`函数将即时用于将上述字符串日期转换为`datetime`对象。通过新架构，当元组返回时，`datetime()`对象已经存在于其中，`datetime.strptime()`函数只被提前调用了一次：

```py
row = (1, datetime.datetime(2019, 12, 31, 19, 56, 58, 272106))
```

SQLAlchemy 中的`RowProxy`和`Row`对象是大部分 SQLAlchemy 的 C 扩展代码发生的地方。这些代码已经经过高度重构，以有效地提供新的行为，并且整体性能已经得到改善，因为`Row`的设计现在相当简单。

之前行为背后的理念假设了一个结果行可能有几十甚至几百列存在的使用模型，其中大多数列不会被访问，并且其中大多数列需要一些结果值处理函数。通过仅在需要时调用处理函数，目标是不需要大量的结果处理函数，从而提高性能。

有许多原因导致上述假设不成立：

1.  调用绝大多数行处理函数是为了将字节字符串解码为 Python Unicode 字符串，在 Python 2 下。这是因为 Python Unicode 开始被使用并且在 Python 3 存在之前。一旦引入了 Python 3，在几年内，所有 Python DBAPIs 都开始正确地支持直接传递 Python Unicode 对象，在 Python 2 和 Python 3 下都是如此，在前一种情况下是作为选项，在后一种情况下是唯一的前进方式。最终，在大多数情况下，它也成为了 Python 2 的默认选项。SQLAlchemy 的 Python 2 支持仍然支持一些 DBAPIs，比如 cx_Oracle，但现在是在 DBAPI 级别执行而不是作为标准 SQLAlchemy 结果行处理函数。

1.  上述字符串转换，在使用时，通过 C 扩展被制作得非常高效，以至于即使在 1.4 版中，SQLAlchemy 的字节到 Unicode 编解码挂钩被插入到 cx_Oracle 中，观察到它比 cx_Oracle 自己的挂钩更高效；这意味着在任何情况下将所有字符串转换为行的开销都不像最初那样显着。

1.  在大多数其他情况下不使用行处理函数；例外情况包括 SQLite 的日期时间支持，某些后端的 JSON 支持，一些数字处理程序例如字符串到 `Decimal` 的转换。在 `Decimal` 的情况下，Python 3 也标准化了高性能的 `cdecimal` 实现，而在 Python 2 中则继续使用性能远远不及的纯 Python 版本。

1.  在实际使用案例中，很少会出现只需要少数列的情况在 SQLAlchemy 的早期，来自其他语言的数据库代码形式“row = fetch(‘SELECT * FROM table’)”很常见；然而，观察到的野外代码通常使用了需要的特定列的表达式语言。

另请参阅

查询返回的“KeyedTuple”对象已被“Row”替换

ORM 会话.execute() 在所有情况下都使用“future”风格的结果集

[#4710](https://www.sqlalchemy.org/trac/ticket/4710)  ### SELECT 对象和衍生的 FROM 子句允许重复的列和列标签

此更改允许 `select()` 构造现在允许重复的列标签以及重复的列对象本身，以便结果元组以相同的方式组织和排序，即所选列的方式。ORM `Query` 已经按照这种方式工作，因此此更改允许更大的跨兼容性，这是 2.0 过渡的一个关键目标：

```py
>>> from sqlalchemy import column, select
>>> c1, c2, c3, c4 = column("c1"), column("c2"), column("c3"), column("c4")
>>> stmt = select(c1, c2, c3.label("c2"), c2, c4)
>>> print(stmt)
SELECT  c1,  c2,  c3  AS  c2,  c2,  c4 
```

为了支持这一变化，`SelectBase`使用的`ColumnCollection`以及用于派生 FROM 子句的列集合，如子查询，也支持重复列；这包括新的`SelectBase.selected_columns`属性，已弃用的`SelectBase.c`属性，以及在诸如`Subquery`和`Alias`等构造中看到的`FromClause.c`属性：

```py
>>> list(stmt.selected_columns)
[
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcca20; c1>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcc9e8; c2>,
 <sqlalchemy.sql.elements.Label object at 0x7fa540b3e2e8>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcc9e8; c2>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540897048; c4>
]

>>> print(stmt.subquery().select())
SELECT  anon_1.c1,  anon_1.c2,  anon_1.c2,  anon_1.c2,  anon_1.c4
FROM  (SELECT  c1,  c2,  c3  AS  c2,  c2,  c4)  AS  anon_1 
```

`ColumnCollection`还允许通过整数索引访问，以支持当字符串“键”不明确时：

```py
>>> stmt.selected_columns[2]
<sqlalchemy.sql.elements.Label object at 0x7fa540b3e2e8>
```

为了适应`ColumnCollection`在诸如`Table`和`PrimaryKeyConstraint`等对象中的使用，保留了旧的“去重”行为，这对于这些对象更为关键，它被保存在一个新的类`DedupeColumnCollection`中。

此更改包括删除了熟悉的警告`"Column %r on table %r being replaced by %r, which has the same key.  Consider use_labels for select() statements."`；`Select.apply_labels()`仍然可用，并且仍然被 ORM 用于所有 SELECT 操作，但它不意味着列对象的去重，尽管它意味着隐式生成的标签的去重：

```py
>>> from sqlalchemy import table
>>> user = table("user", column("id"), column("name"))
>>> stmt = select(user.c.id, user.c.name, user.c.id).apply_labels()
>>> print(stmt)
SELECT "user".id AS user_id, "user".name AS user_name, "user".id AS id_1
FROM "user"
```

最后，该更改使得更容易创建 UNION 和其他`_selectable.CompoundSelect`对象，通过确保 SELECT 语句中的列数和位置与给定的相同，例如：

```py
>>> s1 = select(user, user.c.id)
>>> s2 = select(c1, c2, c3)
>>> from sqlalchemy import union
>>> u = union(s1, s2)
>>> print(u)
SELECT  "user".id,  "user".name,  "user".id
FROM  "user"  UNION  SELECT  c1,  c2,  c3 
```

[#4753](https://www.sqlalchemy.org/trac/ticket/4753)  ### 改进了使用 CAST 或类似方法对简单列表达式进行列标记

有用户指出，当针对命名列使用类似 CAST 的函数时，PostgreSQL 数据库具有方便的行为，即结果列名与内部表达式相同：

```py
test=> SELECT CAST(data AS VARCHAR) FROM foo;

data
------
 5
(1 row)
```

这使得可以对表列应用 CAST 而不会在结果行中丢失列名（上述使用名称`"data"`）。与 MySQL/MariaDB 等数据库��比，以及大多数其他数据库，其中列名取自完整的 SQL 表达式，不太具有可移植性：

```py
MariaDB [test]> SELECT CAST(data AS CHAR) FROM foo;
+--------------------+
| CAST(data AS CHAR) |
+--------------------+
| 5                  |
+--------------------+
1 row in set (0.003 sec)
```

在 SQLAlchemy Core 表达式中，我们从不处理像上面那样的原始生成名称，因为 SQLAlchemy 对这些表达式应用自动标记，这些表达式直到现在都是所谓的 “匿名” 表达式：

```py
>>> print(select(cast(foo.c.data, String)))
SELECT  CAST(foo.data  AS  VARCHAR)  AS  anon_1  #  old  behavior
FROM  foo 
```

这些匿名表达式是必需的，因为 SQLAlchemy 的 `ResultProxy` 大量使用结果列名称来匹配数据类型，例如 `String` 数据类型曾经具有结果行处理行为，以正确的列匹配起来，因此最重要的是这些名称必须易于以数据库无关的方式确定，并且在所有情况下都是唯一的。在 SQLAlchemy 1.0 中作为 [#918](https://www.sqlalchemy.org/trac/ticket/918) 的一部分，对于大多数核心 SELECT 构造，不再需要在结果行中使用命名列（特别是 PEP-249 游标的 `cursor.description` 元素），在 1.4 版本中，系统总体上变得更加适应具有重复列或标签名称的 SELECT 语句，例如在 SELECT 对象和派生 FROM 子句允许重复列和列标签 中。所以我们现在模仿 PostgreSQL 对单个列的简单修改的合理行为，尤其是与 CAST 相关的行为：

```py
>>> print(select(cast(foo.c.data, String)))
SELECT  CAST(foo.data  AS  VARCHAR)  AS  data
FROM  foo 
```

对于没有名称的表达式，使用先前的逻辑来生成通常的“匿名”标签：

```py
>>> print(select(cast("hi there," + foo.c.data, String)))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  anon_1
FROM  foo 
```

对于 `Label` 的 `cast()`，尽管必须省略标签表达式，因为这些表达式不会在 CAST 内部呈现，但仍然会使用给定的名称：

```py
>>> print(select(cast(("hi there," + foo.c.data).label("hello_data"), String)))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  hello_data
FROM  foo 
```

当然，一直以来都是这样，`Label` 可以应用于外部的表达式，直接应用 “AS <name>” 标签：

```py
>>> print(select(cast(("hi there," + foo.c.data), String).label("hello_data")))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  hello_data
FROM  foo 
```

[#4449](https://www.sqlalchemy.org/trac/ticket/4449)  ### 新的用于 LIMIT/OFFSET 的 “后编译” 绑定参数在 Oracle、SQL Server 中使用

1.4 系列的一个主要目标是确保所有核心 SQL 构造都是完全可缓存的，这意味着特定的 `Compiled` 结构将产生相同的 SQL 字符串，而不管使用它的任何 SQL 参数，其中特别包括用于指定 LIMIT 和 OFFSET 值的参数，通常用于分页和 “top N” 类型的结果。

虽然 SQLAlchemy 多年来一直使用绑定参数进行 LIMIT/OFFSET 方案，但仍然存在一些离群值，其中不允许使用这些参数，包括 SQL Server 的 “TOP N” 语句，例如：

```py
SELECT  TOP  5  mytable.id,  mytable.data  FROM  mytable
```

以及在 Oracle 中，如果向 `create_engine()` 传递了 `optimize_limits=True` 参数，SQLAlchemy 将使用 FIRST_ROWS() 提示，这不允许它们，但也有报道称使用绑定参数与 ROWNUM 比较会产生较慢的查询计划：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS(5) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id,  mytable.data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  :param_1
)  anon_1  WHERE  ora_rn  >  :param_2
```

为了让所有语句在编译级别无条件可缓存，添加了一种新形式的绑定参数，称为“后编译”参数，它利用了与“扩展 IN 参数”相同的机制。这是一个 `bindparam()`，其行为与任何其他绑定参数完全相同，只是参数值在发送到 DBAPI `cursor.execute()` 方法之前会被直接渲染到 SQL 字符串中。新参数在 SQL Server 和 Oracle 方言内部使用，以便驱动程序接收到直接渲染的值，但 SQLAlchemy 的其余部分仍然可以将其视为绑定参数。使用 `str(statement.compile(dialect=<dialect>))` 对上述两个语句进行字符串化后现在看起来像：

```py
SELECT  TOP  [POSTCOMPILE_param_1]  mytable.id,  mytable.data  FROM  mytable
```

和：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS([POSTCOMPILE__ora_frow_1]) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id,  mytable.data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  [POSTCOMPILE_param_1]
)  anon_1  WHERE  ora_rn  >  [POSTCOMPILE_param_2]
```

当使用“扩展 IN”时，也会看到 `[POSTCOMPILE_<param>]` 格式。

查看 SQL 日志输出时，将看到语句的最终形式：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS(5) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id  AS  id,  mytable.data  AS  data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  8
)  anon_1  WHERE  ora_rn  >  3
```

“后编译参数”功能通过 `bindparam.literal_execute` 参数作为公共 API 公开，但目前不打算供一般使用。字面值是使用底层数据类型的 `TypeEngine.literal_processor()` 渲染的，在 SQLAlchemy 中具有**极其有限**的范围，仅支持整数和简单字符串值。

[#4808](https://www.sqlalchemy.org/trac/ticket/4808)  ### 基于子事务，现在可以根据连接级事务是否处于非活动状态

现在，`Connection` 包括了一个行为，即由于内部事务的回滚，`Transaction` 可以变为非活动状态，但是 `Transaction` 在自身被回滚之前不会清除。

这本质上是一种新的错误条件，如果内部“子”事务已回滚，则不允许在 `Connection` 上继续执行语句。该行为与 ORM `Session` 的行为非常相似，如果已启动外部事务，则需要回滚以清除无效事务；此行为在 “由于刷新期间的前一个异常，此会话的事务已回滚。”（或类似内容） 中有描述。

虽然 `Connection` 的行为模式比 `Session` 更宽松，但由于它有助于确定子事务何时回滚了 DBAPI 事务，但外部代码并不知道此事并尝试继续进行，实际上是在新事务上运行操作，因此进行了更改。在 将会话加入外部事务（例如用于测试套件） 中描述的“测试套件”模式是这种情况的普遍发生地点。

Core 和 ORM 的“子事务”功能本身已被弃用，并且在 2.0 版本中将不再存在。因此，这种新的错误条件本身是临时的，因为一旦删除子事务，它就不再适用。

为了使用不包括子事务的 2.0 样式行为，请在 `create_engine()` 上使用 `create_engine.future` 参数。

错误消息在错误页面中描述为 此连接处于非活动事务中。 请在继续之前完全回滚（）。### 枚举和布尔数据类型不再默认为“创建约束”

`Enum.create_constraint` 和 `Boolean.create_constraint` 参数现在默认为 False，表示当创建这两种数据类型的所谓“非本机”版本时，默认不会生成 CHECK 约束。这些 CHECK 约束提出了应该选择的模式管理维护复杂性，而不是默认打开。

要确保为这些类型发出 CREATE CONSTRAINT，请将这些标志设置为`True`：

```py
class Spam(Base):
    __tablename__ = "spam"
    id = Column(Integer, primary_key=True)
    boolean = Column(Boolean(create_constraint=True))
    enum = Column(Enum("a", "b", "c", create_constraint=True))
```

[#5367](https://www.sqlalchemy.org/trac/ticket/5367)

## 新功能 - ORM

### 列的 Raiseload

“raiseload”功能会在访问未加载属性时引发`InvalidRequestError`，现在可以通过`defer.raiseload`参数来为基于列的属性提供支持。这与关系加载中使用的`raiseload()`选项的工作方式相同：

```py
book = session.query(Book).options(defer(Book.summary, raiseload=True)).first()

# would raise an exception
book.summary
```

要在映射上配置列级 raiseload，可以使用`deferred.raiseload`参数来为`deferred()`。然后可以在查询时使用`undefer()`选项来急切加载属性：

```py
class Book(Base):
    __tablename__ = "book"

    book_id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    summary = deferred(Column(String(2000)), raiseload=True)
    excerpt = deferred(Column(Text), raiseload=True)

book_w_excerpt = session.query(Book).options(undefer(Book.excerpt)).first()
```

最初考虑扩展现有的为`relationship()`属性工作的`raiseload()`选项，以支持基于列的属性。然而，这将破坏`raiseload()`的“通配符”行为，该行为被记录为允许阻止所有关系加载：

```py
session.query(Order).options(joinedload(Order.items), raiseload("*"))
```

如果我们扩展了`raiseload()`以适应列，通配符也将阻止列加载，从而导致向后不兼容的更改；此外，不清楚`raiseload()`是否同时涵盖列表达式和关系，如何实现上述仅阻止关系加载的效果，而不添加新的 API。因此，为了保持简单，列的选项仍然在`defer()`上：

> `raiseload()` - 查询选项，用于关系加载时引发异常
> 
> `defer.raiseload` - 查询选项，用于列表达式加载时引发异常

作为此更改的一部分，“deferred”与属性过期的行为已更改。以前，当对象被标记为过期，然后通过访问其中一个过期属性来取消过期时，映射为“deferred”的属性也会加载。现在已更改为映射中延迟的属性永远不会“取消过期”，只有在作为延迟加载器的一部分访问时才会加载。

一个未映射为“deferred”的属性，但在查询时通过`defer()`选项延迟，当对象或属性过期时将被重置；也就是说，延迟选项被移除。这与以前的行为相同。

另请参阅

使用 raiseload 防止延迟列加载

[#4826](https://www.sqlalchemy.org/trac/ticket/4826)  ### ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases

psycopg2 方言特性“execute_values”现在默认为 INSERT 语句添加 RETURNING，在 Core 中同时支持“executemany” + “RETURNING”，现在默认情况下使用 psycopg2 的 `execute_values()` 扩展为 psycopg2 方言启用。ORM 刷新过程现在利用此功能，以便在不丢失能够将 INSERT 语句批处理在一起的性能优势的同时实现新生成的主键值和服务器默认值的检索。此外，psycopg2 的 `execute_values()` 扩展本身通过将一个 INSERT 语句重写为包含许多“VALUES”表达式的单个语句而不是重复调用相同语句，提供了五倍的性能改进，因为 psycopg2 缺乏预先准备语句的能力，这通常是为了使这种方法具有高性能而预期的。

SQLAlchemy 在其示例中包含一个性能套件，我们可以比较“batch_inserts”运行程序在 1.3 和 1.4 中生成的时间，显示大多数批量插入的速度提升了 3 倍至 5 倍：

```py
# 1.3
$ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
test_flush_no_pk : (100000 iterations); total time 14.051527 sec
test_bulk_save_return_pks : (100000 iterations); total time 15.002470 sec
test_flush_pk_given : (100000 iterations); total time 7.863680 sec
test_bulk_save : (100000 iterations); total time 6.780378 sec
test_bulk_insert_mappings :  (100000 iterations); total time 5.363070 sec
test_core_insert : (100000 iterations); total time 5.362647 sec

# 1.4 with enhancement
$ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
test_flush_no_pk : (100000 iterations); total time 3.820807 sec
test_bulk_save_return_pks : (100000 iterations); total time 3.176378 sec
test_flush_pk_given : (100000 iterations); total time 4.037789 sec
test_bulk_save : (100000 iterations); total time 2.604446 sec
test_bulk_insert_mappings : (100000 iterations); total time 1.204897 sec
test_core_insert : (100000 iterations); total time 0.958976 sec
```

注意，`execute_values()` 扩展会修改在 psycopg2 层中由 SQLAlchemy 记录的 INSERT 语句**之后**。因此，在 SQL 记录中，可以看到参数集被批处理在一起，但多个“values”的连接在应用程序端不可见：

```py
2020-06-27 19:08:18,166 INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (%(data)s) RETURNING a.id
2020-06-27 19:08:18,166 INFO sqlalchemy.engine.Engine [generated in 0.00698s] ({'data': 'data 1'}, {'data': 'data 2'}, {'data': 'data 3'}, {'data': 'data 4'}, {'data': 'data 5'}, {'data': 'data 6'}, {'data': 'data 7'}, {'data': 'data 8'}  ... displaying 10 of 4999 total bound parameter sets ...  {'data': 'data 4998'}, {'data': 'data 4999'})
2020-06-27 19:08:18,254 INFO sqlalchemy.engine.Engine COMMIT
```

可以通过在 PostgreSQL 端启用语句记录来查看最终的 INSERT 语句：

```py
2020-06-27 19:08:18.169 EDT [26960] LOG:  statement: INSERT INTO a (data)
VALUES ('data 1'),('data 2'),('data 3'),('data 4'),('data 5'),('data 6'),('data
7'),('data 8'),('data 9'),('data 10'),('data 11'),('data 12'),
... ('data 999'),('data 1000') RETURNING a.id

2020-06-27 19:08:18.175 EDT
[26960] LOG:  statement: INSERT INTO a (data) VALUES ('data 1001'),('data
1002'),('data 1003'),('data 1004'),('data 1005 '),('data 1006'),('data
1007'),('data 1008'),('data 1009'),('data 1010'),('data 1011'), ...
```

该功能默认将行分组为每组 1000 行，可以使用文档中记录的 `executemany_values_page_size` 参数来影响。

[#5263](https://www.sqlalchemy.org/trac/ticket/5263)  ### ORM 批量更新和删除在可用时使用 RETURNING 作为“fetch”策略

使用“fetch”策略的 ORM 批量更新或删除：

```py
sess.query(User).filter(User.age > 29).update(
    {"age": User.age - 10}, synchronize_session="fetch"
)
```

现在如果后端数据库支持，将使用 RETURNING；目前包括 PostgreSQL 和 SQL Server（Oracle 方言不支持返回多行）：

```py
UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s RETURNING users.id
[generated in 0.00060s] {'age_int_1': 10, 'age_int_2': 29}
Col ('id',)
Row (2,)
Row (4,)
```

对于不支持返回多行的后端，仍然使用先前的方法在事先发出主键的 SELECT：

```py
SELECT users.id FROM users WHERE users.age_int > %(age_int_1)s
[generated in 0.00043s] {'age_int_1': 29}
Col ('id',)
Row (2,)
Row (4,)
UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s
[generated in 0.00102s] {'age_int_1': 10, 'age_int_2': 29}
```

这种变化的一个复杂挑战之一是支持水平分片扩展等情况，其中单个批量更新或删除可能在一些支持 RETURNING 的后端之间复用，而另一些则不支持。新的 1.4 执行架构支持这种情况，以便“fetch”策略可以保持不变，优雅地降级到使用 SELECT，而不是必须添加一个不具备后端通用性的新“returning”策略。

作为这一变化的一部分，“fetch”策略也变得更加高效，它不再使与匹配行对应的对象过期，对于可以在 Python 中求值的用于 SET 子句的 Python 表达式；相反，这些直接分配到对象上，就像“evaluate”策略一样。只有对于无法求值的 SQL 表达式，它才会退回到使属性过期。对于无法求值的值，“evaluate”策略也已经增强为退回到“expire”。

## 行为变化 - ORM

### 由 Query 返回的“KeyedTuple”对象被 Row 取代

如在 RowProxy is no longer a “proxy”; is now called Row and behaves like an enhanced named tuple 中所讨论的，核心`RowProxy`对象现在被一个名为`Row`的类所取代。基本的`Row`对象现在更像一个命名元组，因此现在被用作由`Query`对象返回的类似元组的结果的基础，而不是以前的“KeyedTuple”类。

其原因是到 SQLAlchemy 2.0，Core 和 ORM SELECT 语句将使用与命名元组相似的相同`Row`对象返回结果行。从`Row`中可以通过`Row._mapping`属性获取类似字典的功能。在此期间，Core 结果集将使用维护先前字典/元组混合行为的`Row`子类`LegacyRow`以确保向后兼容性，而`Row`类将直接用于`Query`对象返回的 ORM 元组结果。

为了让`Row`的大多数功能在 ORM 中可用，已经付出了努力，这意味着可以通过字符串名称以及实体/列来访问：

```py
row = s.query(User, Address).join(User.addresses).first()

row._mapping[User]  # same as row[0]
row._mapping[Address]  # same as row[1]
row._mapping["User"]  # same as row[0]
row._mapping["Address"]  # same as row[1]

u1 = aliased(User)
row = s.query(u1).only_return_tuples(True).first()
row._mapping[u1]  # same as row[0]

row = s.query(User.id, Address.email_address).join(User.addresses).first()

row._mapping[User.id]  # same as row[0]
row._mapping["id"]  # same as row[0]
row._mapping[users.c.id]  # same as row[0]
```

另见

RowProxy 不再是“代理”；现在称为 Row，并且行为类似于增强的命名元组

[#4710](https://www.sqlalchemy.org/trac/ticket/4710).  ### 会话功能的新“autobegin”行为

以前，在默认模式为`autocommit=False`的情况下，`Session`会在构造时立即内部开始一个`SessionTransaction`对象，并且在每次调用`Session.rollback()`或`Session.commit()`后会创建一个新的。

新行为是这个`SessionTransaction`对象现在只在需要时创建，当调用`Session.add()`或`Session.execute()`等方法时。但现在也可以显式调用`Session.begin()`来开始事务，即使在`autocommit=False`模式下，这与未来风格的`_base.Connection`的行为相匹配。

这表明的行为变化是：

+   `Session` 现在可以处于没有启动事务的状态，即使在 `autocommit=False` 模式下也是如此。以前，这种状态只在“自动提交”模式下可用。

+   在这种状态下，`Session.commit()` 和 `Session.rollback()` 方法都不起作用。依赖这些方法来使所有对象过期的代码应明确使用 `Session.begin()` 或 `Session.expire_all()` 来适应其用例。

+   当 `Session` 被创建时，或者在 `Session.rollback()` 或 `Session.commit()` 完成后，`SessionEvents.after_transaction_create()` 事件钩子不会立即被触发。

+   `Session.close()` 方法也不意味着隐式开始一个新的 `SessionTransaction`。

另请参阅

自动开始

#### 理由

`Session` 对象的默认行为 `autocommit=False` 历来意味着始终有一个与 `Session` 相关的 `SessionTransaction` 对象，通过 `Session.transaction` 属性关联。当给定的 `SessionTransaction` 完成时，由于提交、回滚或关闭，它会立即被新的替换。`SessionTransaction` 本身并不意味着使用任何连接相关资源，因此这种长期存在的行为具有特定的优雅之处，即 `Session.transaction` 的状态始终可预测为非 None。

但是，作为大大减少引用循环的[#5056](https://www.sqlalchemy.org/trac/ticket/5056)倡议的一部分，这种假设意味着调用`Session.close()`会导致一个仍然存在引用循环且更难清理的`Session`对象，更不用说构造`SessionTransaction`对象的一点小开销了，这意味着会为一个例如调用了`Session.commit()`然后再调用`Session.close()`的`Session`创建不必要的开销。

因此，决定`Session.close()`在内部状态`self.transaction`，现在内部称为`self._transaction`，留为空，并且只在需要时创建一个新的`SessionTransaction`。为了一致性和代码覆盖率，此行为还扩展到了所有期望“autobegin”的点，不仅仅是在调用`Session.close()`时。

特别是，这对于订阅`SessionEvents.after_transaction_create()`事件钩子的应用程序造成了行为上的改变；之前，当构造`Session`时，此事件会被触发，以及对关闭上一个事务的大多数操作，并且会触发`SessionEvents.after_transaction_end()`。新的行为是，当`Session`尚未创建新的`SessionTransaction`对象且映射对象通过诸如`Session.add()`和`Session.delete()`等方法与`Session`关联时，当调用`Session.transaction`属性时，当`Session.flush()`方法有任务需要完成等情况下，将会按需触发`SessionEvents.after_transaction_create()`。

此外，依赖于`Session.commit()`或`Session.rollback()`方法无条件使所有对象过期的代码不能再这样做了。当没有发生变化时需要使所有对象过期的代码应该针对此情况调用`Session.expire_all()`。

除了更改`SessionEvents.after_transaction_create()`事件发出的时间以及`Session.commit()`或`Session.rollback()`的无操作性质外，此更改不应对`Session`对象的行为产生其他用户可见的影响；`Session`在调用`Session.close()`后仍然保持可用于新操作的行为，并且`Session`与`Engine`和数据库本身的交互顺序也应保持不受影响，因为这些操作已经以按需方式运行。

[#5074](https://www.sqlalchemy.org/trac/ticket/5074)  ### 只读视图关系不同步回引

在 1.3.14 中的[#5149](https://www.sqlalchemy.org/trac/ticket/5149)中，SQLAlchemy 开始在目标关系上同时使用`relationship.backref`或`relationship.back_populates`关键字时发出警告，同时使用`relationship.viewonly`标志。这是因为“只读”关系实际上不会保存对其所做的更改，这可能导致一些误导行为发生。然而，在[#5237](https://www.sqlalchemy.org/trac/ticket/5237)中，我们试图优化这种行为，因为在只读关系上设置回引是有合法用例的，包括回填属性有时由关系懒加载器用于确定在另一个方向上不需要额外的急加载，以及回填可以用于映射器内省和`backref()`可以是设置双向关系的便捷方式。

那时的解决方案是使从反向引用发生的“变化”成为可选的事情，使用 `relationship.sync_backref` 标志。在 1.4 版本中，对于也设置了 `relationship.viewonly` 的关系目标，默认情况下 `relationship.sync_backref` 的值为 False。这表示对具有 viewonly 的关系所做的任何更改都不会以任何方式影响另一侧或 `Session` 的状态：

```py
class User(Base):
    # ...

    addresses = relationship(Address, backref=backref("user", viewonly=True))

class Address(Base): ...

u1 = session.query(User).filter_by(name="x").first()

a1 = Address()
a1.user = u1
```

上面，`a1` 对象将**不会**被添加到 `u1.addresses` 集合中，也不会将 `a1` 对象添加到会话中。之前，这两件事情都是正确的。当 `relationship.viewonly` 为 `False` 时，不再发出需要将 `relationship.sync_backref` 设置为 `False` 的警告，因为这现在是默认行为。

[#5237](https://www.sqlalchemy.org/trac/ticket/5237)  ### 在 2.0 版本中将删除对 cascade_backrefs 行为的弃用

SQLAlchemy 长期以来一直有一个根据反向引用赋值将对象级联到 `Session` 的行为。给定下面已经在 `Session` 中的 `User`，将其分配给 `Address` 对象的 `Address.user` 属性，假设已经建立了双向关系，这意味着在那一点上 `Address` 也会被放入 `Session` 中：

```py
u1 = User()
session.add(u1)

a1 = Address()
a1.user = u1  # <--- adds "a1" to the Session
```

上述行为是 backref 行为的意外副作用，因为由于`a1.user`意味着`u1.addresses.append(a1)`，`a1`会被级联到`Session`中。这在 1.4 版本中仍然是默认行为。在某个时候，添加了一个新标志`relationship.cascade_backrefs`来禁用上述行为，以及`backref.cascade_backrefs`来在通过`relationship.backref`指定关系时设置此行为，因为这可能会令人惊讶，也会妨碍一些操作，其中对象会过早地放入`Session`中并提前刷新。

在 2.0 版本中，默认行为将是“cascade_backrefs”为 False，并且此外不会有“True”行为，因为这通常不是一种理想的行为。当启用 2.0 版本的弃用警告时，当“backref 级联”实际发生时，将发出警告。要获得新行为，要么在任何目标关系上将`relationship.cascade_backrefs`和`backref.cascade_backrefs`设置为`False`，就像在 1.3 版本和更早版本中已经支持的那样，要么使用`Session.future`标志进入 2.0 风格模式：

```py
Session = sessionmaker(engine, future=True)

with Session() as session:
    u1 = User()
    session.add(u1)

    a1 = Address()
    a1.user = u1  # <--- will not add "a1" to the Session
```

[#5150](https://www.sqlalchemy.org/trac/ticket/5150)  ### Eager loaders emit during unexpire operations

长期以来一直期望的行为是，当访问一个过期对象时，配置的急切加载器将运行，以便在对象被刷新或以其他方式取消过期时急切加载过期对象上的关系。现在已经添加了这种行为，因此 joinedloaders 将像往常一样添加内联 JOIN，而 selectin/subquery loaders 将在对象被取消过期或对象被刷新时为给定关系运行“immediateload”操作：

```py
>>> a1 = session.query(A).options(joinedload(A.bs)).first()
>>> a1.data = "new data"
>>> session.commit()
```

上面，`A`对象使用了一个`joinedload()`选项来关联它，以便急切加载`bs`集合。在`session.commit()`之后，对象的状态被标记为过期。在访问`.data`列属性时，对象将被刷新，现在这将包括 joinedload 操作：

```py
>>> a1.data
SELECT  a.id  AS  a_id,  a.data  AS  a_data,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  a  LEFT  OUTER  JOIN  b  AS  b_1  ON  a.id  =  b_1.a_id
WHERE  a.id  =  ? 
```

该行为适用于直接应用于`relationship()`的加载策略，以及与`Query.options()`一起使用的选项，前提是对象最初是由该查询加载的。

对于“secondary”急切加载器“selectinload”和“subqueryload”，这些加载器的 SQL 策略并不是为了在单个对象上急切加载属性而必要；因此，在刷新场景中，它们将调用“immediateload”策略，这类似于由“lazyload”发出的查询，作为额外的查询发出：

```py
>>> a1 = session.query(A).options(selectinload(A.bs)).first()
>>> a1.data = "new data"
>>> session.commit()
>>> a1.data
SELECT  a.id  AS  a_id,  a.data  AS  a_data
FROM  a
WHERE  a.id  =  ?
(1,)
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  ?  =  b.a_id
(1,) 
```

请注意，加载器选项不适用于以不同方式引入到`Session`中的对象。也就是说，如果`a1`对象只是在这个`Session`中持久化，或者在急切选项应用之前用不同的查询加载了该对象，则该对象不具有与之关联的急切加载选项。这并不是一个新概念，但是寻找刷新行为上的 eagerload 的用户可能会发现这更加明显。

[#1763](https://www.sqlalchemy.org/trac/ticket/1763)  ### 列加载器，如`deferred()`，`with_expression()`，仅在最外层的完整实体查询中指示时才生效

注意

这个变更说明在本文档的早期版本中不存在，但对于所有 SQLAlchemy 1.4 版本都是相关的。

1.3 版本及之前版本从未支持的行为，但仍然会产生特定效果，是重新利用列加载器选项，如`defer()`和`with_expression()` 在子查询中，以控制每个子查询的列子句中的 SQL 表达式。一个典型的例子是构建 UNION 查询，例如：

```py
q1 = session.query(User).options(with_expression(User.expr, literal("u1")))
q2 = session.query(User).options(with_expression(User.expr, literal("u2")))

q1.union_all(q2).all()
```

在 1.3 版本中，`with_expression()`选项会对 UNION 的每个元素生效，例如：

```py
SELECT  anon_1.anon_2  AS  anon_1_anon_2,  anon_1.user_account_id  AS  anon_1_user_account_id,
anon_1.user_account_name  AS  anon_1_user_account_name
FROM  (
  SELECT  ?  AS  anon_2,  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  ?  AS  anon_3,  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_1
('u1',  'u2')
```

SQLAlchemy 1.4 对加载器选项的概念变得更加严格，因此仅应用于**查询的最外层部分**，即用于填充实际要返回的 ORM 实体的 SELECT；在 1.4 中上面的查询将产生：

```py
SELECT  ?  AS  anon_1,  anon_2.user_account_id  AS  anon_2_user_account_id,
anon_2.user_account_name  AS  anon_2_user_account_name
FROM  (
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_2
('u1',)
```

换句话说，`Query`的选项是从 UNION 的第一个元素中获取的，因为所有加载器选项只能在最顶层。第二个查询的选项被忽略了。

#### 理由

此行为现在更加接近于其他种类的加载选项，如在所有 SQLAlchemy 版本中，1.3 及更早版本已经复制到查询的最顶层的关系加载器选项，如 `joinedload()`，在 UNION 情况下已经复制到了查询的顶层，并且只从 UNION 的第一个元素中取出选项，丢弃查询的其他部分上的任何选项。

上面展示的隐式复制和选择性忽略选项的行为，是一种遗留行为，仅限于 `Query` 的一部分，是一个特殊的例子，说明了 `Query` 及其应用 `Query.union_all()` 的方式存在不足之处，因为不清楚如何将单个 SELECT 转换为自身和另一个查询的 UNION，并且不清楚如何将加载选项应用于该新语句。

SQLAlchemy 1.4 的行为可展示为通常优于 1.3 版本的情况，用于更常见情况的使用 `defer()`。以下查询：

```py
q1 = session.query(User).options(defer(User.name))
q2 = session.query(User).options(defer(User.name))

q1.union_all(q2).all()
```

在 1.3 版本中会笨拙地向内部查询添加 NULL，然后选择它：

```py
SELECT  anon_1.anon_2  AS  anon_1_anon_2,  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
  UNION  ALL
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
)  AS  anon_1
```

如果所有查询没有设置相同的选项，上述情况将由于无法形成正确的 UNION 而引发错误。

而在 1.4 中，该选项仅应用于顶层，省略了对 `User.name` 的获取，并且避免了这种复杂性：

```py
SELECT  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_1
```

#### 正确的方法

使用 2.0 风格 查询时，目前不会发出警告，然而嵌套的 `with_expression()` 选项一直被忽略，因为它们不适用于正在加载的实体，并且不会被隐式复制到任何地方。下面的查询对 `with_expression()` 调用不产生任何输出：

```py
s1 = select(User).options(with_expression(User.expr, literal("u1")))
s2 = select(User).options(with_expression(User.expr, literal("u2")))

stmt = union_all(s1, s2)

session.scalars(select(User).from_statement(stmt)).all()
```

生成 SQL：

```py
SELECT  user_account.id,  user_account.name
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name
FROM  user_account
```

要正确应用 `with_expression()` 到 `User` 实体，应该将其应用于查询的最外层，使用每个 SELECT 的列子句中的普通 SQL 表达式：

```py
s1 = select(User, literal("u1").label("some_literal"))
s2 = select(User, literal("u2").label("some_literal"))

stmt = union_all(s1, s2)

session.scalars(
    select(User)
    .from_statement(stmt)
    .options(with_expression(User.expr, stmt.selected_columns.some_literal))
).all()
```

将产生预期的 SQL：

```py
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
```

`User` 对象本身将在其内容中包含此表达式，在 `User.expr` 下面。### 在临时对象上访问未初始化的集合属性不再改变 __dict__

SQLAlchemy 一直以来的行为是，在新创建的对象上访问映射属性会返回一个隐式生成的值，而不是引发`AttributeError`，例如对于标量属性是`None`，对于保存列表的关系是`[]`：

```py
>>> u1 = User()
>>> u1.name
None
>>> u1.addresses
[]
```

上述行为的原因最初是为了使 ORM 对象更易于使用。由于 ORM 对象在刚创建时代表一个空行而没有任何状态，因此直观地认为其未访问的属性会解析为标量的`None`（或 SQL NULL），对于关系则是空集合。特别是，这使得一种极其常见的模式成为可能，即能够在不手动创建和分配空集合的情况下改变新集合：

```py
>>> u1 = User()
>>> u1.addresses.append(Address())  # no need to assign u1.addresses = []
```

直到 SQLAlchemy 的 1.0 版本，这种初始化系统对标量属性和集合的行为都是将`None`或空集合*填充*到对象的状态中，例如`__dict__`。这意味着以下两个操作是等效的：

```py
>>> u1 = User()
>>> u1.name = None  # explicit assignment

>>> u2 = User()
>>> u2.name  # implicit assignment just by accessing it
None
```

在上述情况下，`u1`和`u2`都会在`name`属性的值中填充`None`。由于这是一个 SQL NULL，ORM 会跳过将这些值包含在 INSERT 中，以便 SQL 级别的默认值生效，否则值会默认为数据库端的 NULL。

作为关于没有预先存在值的属性的属性事件和其他操作的更改的一部分，在 1.0 版本中，这种行为被调整，以便`None`值不再填充到`__dict__`中，只是返回。除了消除 getter 操作的变异副作用外，这种变化还使得可以通过实际分配`None`来将具有服务器默认值的列设置为 NULL，现在可以区分出只是读取它。

然而，这种变化并没有考虑到集合，其中返回一个未分配的空集合意味着这个可变集合每次都会不同，并且也无法正确地适应变异操作（例如追加、添加等）。虽然这种行为通常不会妨碍任何人，但最终在[#4519](https://www.sqlalchemy.org/trac/ticket/4519)中识别出了一个边缘情况，即当对象合并到会话中时，这个空集合可能会有害：

```py
>>> u1 = User(id=1)  # create an empty User to merge with id=1 in the database
>>> merged1 = session.merge(
...     u1
... )  # value of merged1.addresses is unchanged from that of the DB

>>> u2 = User(id=2)  # create an empty User to merge with id=2 in the database
>>> u2.addresses
[]
>>> merged2 = session.merge(u2)  # value of merged2.addresses has been emptied in the DB
```

在上述情况下，`merged1`上的`.addresses`集合将包含已经存在于数据库中的所有`Address()`对象。而`merged2`不会；因为它有一个隐式分配的空列表，`.addresses`集合将被擦除。这是一个例子，说明这种变异的副作用实际上可以改变数据库本身。

虽然曾考虑过属性系统是否应该开始使用严格的“纯 Python”行为，在所有情况下对非存在属性的非持久对象引发`AttributeError`，并要求所有集合都明确分配，但这样的改变可能对多年来依赖于这种行为的大量应用程序来说过于极端，导致复杂的发布/向后兼容性问题，以及恢复旧行为的解决方法可能会变得普遍，从而使整个改变在任何情况下都变得无效。

然后的改变是保持默认的生成行为，但最终使标量的非变异行为对集合也成为现实，通过在集合系统中添加额外的机制。当访问空属性时，新的集合被创建并与状态关联，但直到实际变异才添加到`__dict__`中：

```py
>>> u1 = User()
>>> l1 = u1.addresses  # new list is created, associated with the state
>>> assert u1.addresses is l1  # you get the same list each time you access it
>>> assert (
...     "addresses" not in u1.__dict__
... )  # but it won't go into __dict__ until it's mutated
>>> from sqlalchemy import inspect
>>> inspect(u1).attrs.addresses.history
History(added=None, unchanged=None, deleted=None)
```

当列表被更改时，它就成为要持久化到数据库的跟踪更改的一部分：

```py
>>> l1.append(Address())
>>> assert "addresses" in u1.__dict__
>>> inspect(u1).attrs.addresses.history
History(added=[<__main__.Address object at 0x7f49b725eda0>], unchanged=[], deleted=[])
```

这个改变预计对现有应用程序几乎没有任何影响，除了观察到一些应用程序可能依赖于对该集合的隐式分配，例如根据其`__dict__`中的某些值来断言对象包含某些值：

```py
>>> u1 = User()
>>> u1.addresses
[]
# this will now fail, would pass before
>>> assert {k: v for k, v in u1.__dict__.items() if not k.startswith("_")} == {
...     "addresses": []
... }
```

或者确保集合不需要延迟加载才能继续进行，下面（尽管有些尴尬）的代码现在也会失败：

```py
>>> u1 = User()
>>> u1.addresses
[]
>>> s.add(u1)
>>> s.flush()
>>> s.close()
>>> u1.addresses  # <-- will fail, .addresses is not loaded and object is detached
```

依赖于集合的隐式变异行为的应用程序需要更改，以便明确地分配所需的集合：

```py
>>> u1.addresses = []
```

[#4519](https://www.sqlalchemy.org/trac/ticket/4519)  ### “新实例与现有标识冲突”错误现在是一个警告

SQLAlchemy 一直有逻辑来检测当一个要插入到`Session`中的对象具有与已经存在的对象相同的主键时：

```py
class Product(Base):
    __tablename__ = "product"

    id = Column(Integer, primary_key=True)

session = Session(engine)

# add Product with primary key 1
session.add(Product(id=1))
session.flush()

# add another Product with same primary key
session.add(Product(id=1))
s.commit()  # <-- will raise FlushError
```

改变是`FlushError`被修改为仅仅是一个警告：

```py
sqlalchemy/orm/persistence.py:408: SAWarning: New instance <Product at 0x7f1ff65e0ba8> with identity key (<class '__main__.Product'>, (1,), None) conflicts with persistent instance <Product at 0x7f1ff60a4550>
```

随后，条件将尝试将行插入到数据库中，这将引发`IntegrityError`，这是与`Session`中已存在的主键标识不同的错误：

```py
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) UNIQUE constraint failed: product.id
```

这样做的理由是允许使用`IntegrityError`来捕获重复项的代码能够正常运行，而不受`Session`的现有状态的影响，通常使用保存点来实现：

```py
# add another Product with same primary key
try:
    with session.begin_nested():
        session.add(Product(id=1))
except exc.IntegrityError:
    print("row already exists")
```

早期上述逻辑并不完全可行，因为如果具有现有标识的 `Product` 对象已经在 `Session` 中，代码还必须捕获 `FlushError`，此外，该错误还未针对完整性问题的特定条件进行过滤。通过更改，上述块的行为与发出警告的异常一致。

由于涉及的逻辑处理主键，所有数据库在插入时出现主键冲突时都会发出完整性错误。不会引发错误的情况是极不寻常的，即定义了一个在映射的可选择项上定义了比实际配置的数据库模式更严格的主键的映射，例如在表的连接或在定义附加列作为复合主键的一部分时，这些列实际上在数据库模式中没有约束。然而，这些情况也更一致地工作，即使现有的标识仍然存在于数据库中，插入理论上也会继续进行。警告也可以使用 Python 警告过滤器配置为引发异常。

[#4662](https://www.sqlalchemy.org/trac/ticket/4662)  ### 持久性相关的级联操作在 viewonly=True 时不允许

当使用 `relationship.viewonly` 标志将 `relationship()` 设置为 `viewonly=True` 时，表示此关系仅应用于从数据库加载数据，不应进行变异或涉及持久性操作。为确保此约定成功运行，关系不再能指定在“仅查看”方面毫无意义的 `relationship.cascade` 设置。

这里的主要目标是“删除，删除孤儿”级联，即使 viewonly 为 True，通过 1.3 仍会影响持久性，这是一个错误；即使 viewonly 为 True，如果删除父对象或分离对象，对象仍会级联这两个操作到相关对象。而不是修改级联操作以检查 viewonly，这两者的配置简单地被禁止：

```py
class User(Base):
    # ...

    # this is now an error
    addresses = relationship("Address", viewonly=True, cascade="all, delete-orphan")
```

上述将引发：

```py
sqlalchemy.exc.ArgumentError: Cascade settings
"delete, delete-orphan, merge, save-update" apply to persistence
operations and should not be combined with a viewonly=True relationship.
```

应用程序存在此问题的情况下，从 SQLAlchemy 1.3.12 开始应发出警告，对于上述错误，解决方法是移除仅用于查看的关系的级联设置。

[#4993](https://www.sqlalchemy.org/trac/ticket/4993) [#4994](https://www.sqlalchemy.org/trac/ticket/4994)  ### 使用自定义查询查询继承映射时更严格的行为

这个更改适用于查询已完成的 SELECT 子查询以选择的情况下，一个连接或单个表继承子类实体。如果给定的子查询返回的行不对应于请求的多态标识或标识，将引发错误。以前，在连接表继承下，这种情况会悄悄通过，返回一个无效的子类，而在单表继承下，`Query`会添加额外的条件来限制结果，这可能会不恰当地干扰查询的意图。

鉴于`Employee`，`Engineer(Employee)`，`Manager(Employee)`的示例映射，在 1.3 系列中，如果我们针对连接继承映射发出以下查询：

```py
s = Session(e)

s.add_all([Engineer(), Manager()])

s.commit()

print(s.query(Manager).select_entity_from(s.query(Employee).subquery()).all())
```

子查询同时选择`Engineer`和`Manager`行，即使外部查询针对`Manager`，我们也会得到一个非`Manager`对象：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
2020-01-29 18:04:13,524 INFO sqlalchemy.engine.base.Engine ()
[<__main__.Engineer object at 0x7f7f5b9a9810>, <__main__.Manager object at 0x7f7f5b9a9750>]
```

新的行为是这种情况引发错误：

```py
sqlalchemy.exc.InvalidRequestError: Row with identity key
(<class '__main__.Employee'>, (1,), None) can't be loaded into an object;
the polymorphic discriminator column '%(140205120401296 anon)s.type'
refers to mapped class Engineer->engineer, which is not a sub-mapper of
the requested mapped class Manager->manager
```

仅当该实体的主键列为非 NULL 时，才会引发上述错误。如果一行中没有给定实体的主键，则不会尝试构造实体。

在单一继承映射的情况下，行为的变化稍微更为复杂；如果上述的`Engineer`和`Manager`被映射为单表继承，在 1.3 中将发出以下查询，并且只返回一个`Manager`对象：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
WHERE anon_1.type IN (?)
2020-01-29 18:08:32,975 INFO sqlalchemy.engine.base.Engine ('manager',)
[<__main__.Manager object at 0x7ff1b0200d50>]
```

`Query`向子查询添加了“单表继承”条件，对最初设置的意图进行了评论。这种行为是在版本 1.0 中添加的，在[#3891](https://www.sqlalchemy.org/trac/ticket/3891)中，它在“连接”和“单”表继承之间创建了行为不一致，并且还修改了给定查询的意图，可能意图返回额外的行，其中对应于继承实体的列为 NULL，这是一个有效的用例。现在的行为等同于连接表继承的行为，其中假定子查询返回正确的行，如果遇到意外的多态标识，则会引发错误：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
2020-01-29 18:13:10,554 INFO sqlalchemy.engine.base.Engine ()
Traceback (most recent call last):
# ...
sqlalchemy.exc.InvalidRequestError: Row with identity key
(<class '__main__.Employee'>, (1,), None) can't be loaded into an object;
the polymorphic discriminator column '%(140700085268432 anon)s.type'
refers to mapped class Engineer->employee, which is not a sub-mapper of
the requested mapped class Manager->employee
```

对于上述情况的正确调整，在 1.3 上运行的是调整给定子查询以根据鉴别器列正确过滤行：

```py
print(
    s.query(Manager)
    .select_entity_from(
        s.query(Employee).filter(Employee.discriminator == "manager").subquery()
    )
    .all()
)
```

```py
SELECT  anon_1.type  AS  anon_1_type,  anon_1.id  AS  anon_1_id
FROM  (SELECT  employee.type  AS  type,  employee.id  AS  id
FROM  employee
WHERE  employee.type  =  ?)  AS  anon_1
2020-01-29  18:14:49,770  INFO  sqlalchemy.engine.base.Engine  ('manager',)
[<__main__.Manager  object  at  0x7f70e13fca90>]
```

[#5122](https://www.sqlalchemy.org/trac/ticket/5122)

## 方言更改

### pg8000 的最低版本是 1.16.6，仅支持 Python 3

对 pg8000 方言的支持得到了显着改进，得益于项目的维护者。

由于 API 变化，pg8000 方言现在需要版本 1.16.6 或更高版本。自 1.13 系列起，pg8000 系列已不再支持 Python 2。需要 pg8000 的 Python 2 用户应确保其要求固定在 `SQLAlchemy<1.4`。

[#5451](https://www.sqlalchemy.org/trac/ticket/5451)

### PostgreSQL psycopg2 方言需要版本 2.7 或更高版本的 psycopg2。

psycopg2 方言依赖于过去几年中发布的许多 psycopg2 特性。为了简化方言，现在最低所需版本是 2017 年 3 月发布的版本 2.7。

### psycopg2 方言不再限制绑定参数名称

SQLAlchemy 1.3 无法适应在 psycopg2 方言下包含百分号或括号的绑定参数名称。这反过来意味着包含这些字符的列名也是有问题的，因为 INSERT 和其他 DML 语句将生成与该列匹配的参数名称，这将导致失败。解决方法是利用 `Column.key` 参数，以便生成参数的备用名称，或者在 `create_engine()` 级别更改方言的参数样式。从 SQLAlchemy 1.4.0beta3 开始，所有命名限制都已移除，并且在所有情况下参数都被完全转义，因此这些解决方法不再需要。

[#5941](https://www.sqlalchemy.org/trac/ticket/5941)

[#5653](https://www.sqlalchemy.org/trac/ticket/5653)  ### psycopg2 方言默认使用“execute_values”来进行 INSERT 语句的 RETURNING 操作

在使用 Core 和 ORM 时，对于 PostgreSQL 的重大性能增强的前半部分，psycopg2 方言现在默认使用 `psycopg2.extras.execute_values()` 来编译 INSERT 语句，并在此模式下实现了 RETURNING 支持。这个变化的另一半是 ORM 批量插入现在在大多数情况下使用 RETURNING 的批量语句，它允许 ORM 利用 executemany（即批量 INSERT 语句的批处理）从而使得使用 psycopg2 的 ORM 批量插入速度提高了 400% 取决于具体情况。

此扩展方法允许在单个语句中插入多行，使用语句的扩展 VALUES 子句。虽然 SQLAlchemy 的 `insert()` 构造已经通过 `Insert.values()` 方法支持此语法，但是扩展方法允许在执行语句时动态构建 VALUES 子句，当语句执行为“executemany”执行时，即当将参数字典列表传递给 `Connection.execute()` 时会发生这种情况。它还发生在缓存边界之外，因此在渲染 VALUES 之前可以缓存 INSERT 语句。

在 Performance 示例套件中使用 `bulk_inserts.py` 脚本进行 `execute_values()` 方法的快速测试显示了约**五倍的性能提升**：

```py
$ python -m examples.performance bulk_inserts --test test_core_insert --num 100000 --dburl postgresql://scott:tiger@localhost/test

# 1.3
test_core_insert : A single Core INSERT construct inserting mappings in bulk. (100000 iterations); total time 5.229326 sec

# 1.4
test_core_insert : A single Core INSERT construct inserting mappings in bulk. (100000 iterations); total time 0.944007 sec
```

在版本 1.2 中添加了对“批处理”扩展的支持 Support for Batch Mode / Fast Execution Helpers，并在版本 1.3 中增强了对 `execute_values` 扩展的支持 [#4623](https://www.sqlalchemy.org/trac/ticket/4623)。在版本 1.4 中，对于 INSERT 语句现在默认打开了 `execute_values` 扩展；UPDATE 和 DELETE 的“批处理”扩展仍然默认关闭。

`execute_values` 扩展函数还支持将由 RETURNING 生成的行作为聚合列表返回。如果给定的 `insert()` 构造请求返回通过 `Insert.returning()` 方法或类似用于返回生成的默认值的方法生成的行，那么 psycopg2 方言现在将检索此列表；然后将行安装在结果中，以便像直接来自游标一样检索它们。这使得像 ORM 这样的工具在所有情况下都可以使用批量插入，预计将提供显著的性能改进。

psycopg2 方言的 `executemany_mode` 功能已经进行了以下更改：

+   添加了一个新模式 `"values_only"`。此模式使用非常高效的 `psycopg2.extras.execute_values()` 扩展方法来运行编译的 INSERT 语句，但不使用 `execute_batch()` 来运行 UPDATE 和 DELETE 语句。此新模式现在是 psycopg2 方言的默认设置。

+   现有的 `"values"` 模式现已更名为 `"values_plus_batch"`。此模式将对 INSERT 语句使用 `execute_values`，并对 UPDATE 和 DELETE 语句使用 `execute_batch`。该模式默认未启用，因为它会导致使用 `executemany()` 执行的 UPDATE 和 DELETE 语句的 `cursor.rowcount` 的正常功能受到影响。

+   对于 INSERT 语句，启用了 RETURNING 支持，针对 `"values_only"` 和 `"values"`。Psycopg2 方言将使用 fetch=True 标志从 psycopg2 中接收行，并将它们安装到结果集中，就好像它们直接来自游标一样（实际上，它们确实是，不过 psycopg2 的扩展函数已将多个批次聚合为一个列表）。

+   `execute_values` 的默认 “page_size” 设置已从 100 增加到 1000。对于 `execute_batch` 函数，默认值仍为 100。这些参数可以像以前一样进行修改。

+   版本 1.2 中的 `use_batch_mode` 标志已移除；行为仍可通过版本 1.3 中添加的 `executemany_mode` 标志进行控制。

+   核心引擎和方言已增强以支持 executemany 加返回模式，目前仅适用于 psycopg2，通过提供新的 `CursorResult.inserted_primary_key_rows` 和 `CursorResult.returned_default_rows` 访问器。

另请参阅

Psycopg2 快速执行助手

[#5401](https://www.sqlalchemy.org/trac/ticket/5401) ### 从 SQLite 方言中删除了 “连接重写” 逻辑；更新了导入

放弃对右嵌套连接重写的支持，以支持 2013 年发布的旧 SQLite 版本 3.7.16 之前的版本。不期望任何现代 Python 版本依赖于此限制。

该行为首次引入于版本 0.9，并作为允许右嵌套连接的较大更改的一部分，如 Many JOIN and LEFT OUTER JOIN expressions will no longer be wrapped in (SELECT * FROM ..) AS ANON_1 中所述。然而，由于其复杂性，SQLite 的解决方法在 2013-2014 年期间产生了许多回归。2016 年，方言被修改，使连接重写逻辑仅在 SQLite 版本低于 3.7.16 时发生，使用二分法确定了 SQLite 修复了此结构支持的位置之后，并且未报告进一步的问题（尽管在内部发现了一些错误）。现在预计，几乎没有 Python 2.7 或 3.5 及以上版本（支持的 Python 版本）的构建包含低于 3.7.17 的 SQLite 版本，并且该行为仅在更复杂的 ORM 连接方案中才是必需的。如果安装的 SQLite 版本旧于 3.7.16，则现在会发出警告。

在相关更改中，SQLite 的模块导入不再尝试在 Python 3 上导入“pysqlite2”驱动程序，因为此驱动程序在 Python 3 上不存在；对于旧的 pysqlite2 版本的非常古老警告也被删除。

[#4895](https://www.sqlalchemy.org/trac/ticket/4895)  ### 为 MariaDB 10.3 添加了序列支持

截至 MariaDB 10.3，MariaDB 数据库支持序列。SQLAlchemy 的 MySQL 方言现在实现了对此数据库的`Sequence`对象的支持，这意味着对于在相同方式下的`Table`或`MetaData`集合中存在的`Sequence`，将发出“CREATE SEQUENCE” DDL。就像对于后端如 PostgreSQL、Oracle 一样，当方言的服务器版本检查确认数据库是 MariaDB 10.3 或更高版本时。此外，当以这些方式使用时，`Sequence`将作为列默认值和主键生成对象。

由于此更改将影响 DDL 和 INSERT 语句的假设，对于当前部署在 MariaDB 10.3 上的应用程序，该应用程序还明确使用其表定义中的`Sequence`构造，重要的是要注意`Sequence`支持一个标志`Sequence.optional`，用于限制`Sequence`生效的情况。当在表的整数主键列上使用“optional”时，`Sequence`上的“optional”将生效：

```py
Table(
    "some_table",
    metadata,
    Column(
        "id", Integer, Sequence("some_seq", start=1, optional=True), primary_key=True
    ),
)
```

上述`Sequence`仅在目标数据库不支持任何其他生成整数主键值的方式时用于 DDL 和 INSERT 语句。也就是说，上述 Oracle 数据库将使用序列，但是 PostgreSQL 和 MariaDB 10.3 数据库将不会。对于将升级到 SQLAlchemy 1.4 的现有应用程序可能很重要，因为如果试图使用未创建的序列，则 INSERT 语句将失败。

另请参阅

定义序列

[#4976](https://www.sqlalchemy.org/trac/ticket/4976)  ### 添加了与 SQL Server 不同的序列支持

`Sequence` 构造现在与 Microsoft SQL Server 完全兼容。当应用于 `Column` 时，表的 DDL 将不再包含 IDENTITY 关键字，而是依赖于“CREATE SEQUENCE”以确保序列存在，然后将用于表上的 INSERT 语句。

在版本 1.3 之前，`Sequence` 用于控制 SQL Server 中的 IDENTITY 列的参数；这种用法在 1.3 版本中发出了弃用警告，并在 1.4 版本中现已移除。对于控制 IDENTITY 列的参数，应使用 `mssql_identity_start` 和 `mssql_identity_increment` 参数；请参阅下面链接的 MSSQL 方言文档。

另请参阅

自增行为 / IDENTITY 列

[#4235](https://www.sqlalchemy.org/trac/ticket/4235)

[#4633](https://www.sqlalchemy.org/trac/ticket/4633)

## 主要的 API 更改和特性 - 通用

### Python 3.6 是最低 Python 3 版本；仍支持 Python 2.7

由于 Python 3.5 在 2020 年 9 月到达 EOL，SQLAlchemy 1.4 现在将版本 3.6 作为最低 Python 3 版本。仍支持 Python 2.7，但是 SQLAlchemy 1.4 系列将是最后一个支持 Python 2 的系列。  ### ORM 查询在内部与选择、更新、删除统一；2.0 风格的执行可用

对于 SQLAlchemy 版本 2.0 和本质上的 1.4 来说，最大的概念性改变是核心中的 `Select` 构造和 ORM 中的 `Query` 对象之间的巨大分离已被移除，以及在它们之间的 `Query.update()` 和 `Query.delete()` 方法与 `Update` 和 `Delete` 的关系。

关于`Select`和`Query`，这两个对象在许多版本中具有类似的、大部分重叠的 API，甚至有一些能够在两者之间切换的能力，但在使用模式和行为上仍然有很大的不同。这一历史背景是，`Query`对象是为了克服`Select`对象的缺点而引入的，后者曾经是 ORM 对象查询的核心，只是它们必须以`Table`元数据的形式进行查询。然而，`Query`只有一个简单的接口来加载对象，只有在许多主要版本的发布过程中，它最终才获得了大部分`Select`对象的灵活性，这导致这两个对象变得非常相似，但仍然在很大程度上不兼容。

在版本 1.4 中，所有核心和 ORM SELECT 语句都直接从`Select`对象呈现；当使用`Query`对象时，在语句调用时，它会将其状态复制到一个`Select`对象中，然后使用 2.0 风格执行。未来，`Query`对象将仅成为传统，应用程序将被鼓励转向 2.0 风格执行，允许核心构造自由地针对 ORM 实体使用：

```py
with Session(engine, future=True) as sess:
    stmt = (
        select(User)
        .where(User.name == "sandy")
        .join(User.addresses)
        .where(Address.email_address.like("%gmail%"))
    )

    result = sess.execute(stmt)

    for user in result.scalars():
        print(user)
```

以上示例的注意事项：

+   `Session`和`sessionmaker`对象现在具有完整的上下文管理器（即`with:`语句）功能；请参阅打开和关闭会话的修订文档以获取示例。

+   在 1.4 系列中，所有 2.0 风格 的 ORM 调用都使用一个包含 `Session` 的 `Session.future` 标志设置为 `True` 的标志；此标志表示 `Session` 应具有 2.0 风格的行为，其中包括可以从 `execute` 调用 ORM 查询以及一些事务特性的更改。在 2.0 版本中，此标志将始终为 `True`。

+   `select()` 构造不再需要在列子句周围加括号；有关此改进，请参见 select(), case() 现在接受位置表达式。

+   `select()` / `Select` 对象具有一个 `Select.join()` 方法，其行为类似于 `Query`，甚至可以容纳 ORM 关系属性（而不会破坏 Core 和 ORM 之间的分离！）- 有关此内容，请参见 select().join() 和 outerjoin() 将 JOIN 条件添加到当前查询，而不是创建子查询。

+   与 ORM 实体一起工作并且预计返回 ORM 结果的语句是使用 `Session.execute()` 调用的。请参见 Querying 以获取入门指南。另请参阅 ORM Session.execute() 在所有情况下使用“future”风格结果集 中的以下注意事项。

+   返回一个 `Result` 对象，而不是一个普通列表，这本身是以前的 `ResultProxy` 对象的一个更复杂的版本；此对象现在用于 Core 和 ORM 结果。有关此信息，请参见 New Result object，RowProxy 不再是“代理”；现在称为 Row 并且行为类似于增强的命名元组，以及 Query 返回的“KeyedTuple”对象被 Row 替换。

在 SQLAlchemy 的文档中，将会有许多关于 1.x 风格和 2.0 风格执行的引用。这是为了区分两种查询风格，并尝试向前记录新的调用风格。在 SQLAlchemy 2.0 中，虽然`Query`对象可能仍然作为传统构造保留，但在大多数文档中将不再出现。

对“批量更新和删除”进行了类似的调整，以便核心`update()`和`delete()`可用于批量操作。像下面这样的批量更新：

```py
session.query(User).filter(User.name == "sandy").update(
    {"password": "foobar"}, synchronize_session="fetch"
)
```

现在可以通过 2.0 风格来实现（实际上上述内容在内部以这种方式运行）如下所示：

```py
with Session(engine, future=True) as sess:
    stmt = (
        update(User)
        .where(User.name == "sandy")
        .values(password="foobar")
        .execution_options(synchronize_session="fetch")
    )

    sess.execute(stmt)
```

请注意使用`Executable.execution_options()`方法传递 ORM 相关选项。现在“执行选项”的使用在核心和 ORM 中更加普遍，许多来自`Query`的 ORM 相关方法现在被实现为执行选项（查看`Query.execution_options()`以获取一些示例）。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南

[#5159](https://www.sqlalchemy.org/trac/ticket/5159)  ### ORM `Session.execute()` 在所有情况下都使用“future”风格的`Result`集

如 RowProxy 不再是“代理”；现在称为 Row 并且行为类似增强的命名元组中所述，当与设置为`True`的`create_engine.future`参数的`Engine`一起使用时，`Result`和`Row`对象现在具有“命名元组”行为。这些特定的“命名元组”行现在包括一项行为变更，即 Python 包含表达式使用`in`，例如：

```py
>>> engine = create_engine("...", future=True)
>>> conn = engine.connect()
>>> row = conn.execute.first()
>>> "name" in row
True
```

上述包含测试将使用**值包含**，而不是**键包含**；`row`需要具有“name”的**值**才能返回`True`。

在 SQLAlchemy 1.4 中，当`create_engine.future`参数设置为`False`时，将返回传统风格的`LegacyRow`对象，其具有之前 SQLAlchemy 版本的部分命名元组行为，其中包含性检查继续使用键包含；如果行中有名为“name”的**列**，则`"name" in row`将返回 True，而不是一个值。

在使用`Session.execute()`时，完整的命名元组样式被**无条件地**启用，这意味着`"name" in row`将使用**值包含**作为测试，而**不是**键包含。这是为了适应`Session.execute()`现在返回一个`Result`，该结果还适应 ORM 结果，其中甚至像由`Query.all()`返回的传统 ORM 结果行也使用值包含。

这是从 SQLAlchemy 1.3 到 1.4 的行为变更。要继续接收键包含集合，请使用`Result.mappings()`方法接收返回行为字典的`MappingResult`：

```py
for dict_row in session.execute(text("select id from table")).mappings():
    assert "id" in dict_row
```  ### 透明 SQL 编译缓存添加到 Core、ORM 中的所有 DQL、DML 语句

这是单个 SQLAlchemy 版本中最广泛涵盖的更改之一，经过数月的重新组织和重构，从 Core 的基础到 ORM，现在允许大多数涉及从用户构造的语句生成 SQL 字符串和相关语句元数据的 Python 计算被缓存在内存中，因此对于相同的语句构造的后续调用将使用 35-60%更少的 CPU 资源。

这种缓存不仅限于构建 SQL 字符串，还包括构建将 SQL 结构链接到结果集的结果获取结构，以及在 ORM 中包括适应 ORM 启用的属性加载器、关系急加载器和其他选项，以及每次 ORM 查询试图运行并从结果集构建 ORM 对象时必须构建的对象构造例程。

为了介绍该功能的一般思想，给出了来自性能套件的代码，将调用一个非常简单的查询“n”次，其中 n 的默认值为 10000。查询仅返回一行，因为我们要减少的开销是**许多小查询**的开销。对于返回许多行的查询，优化并不那么显著：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    result = session.query(Customer).filter(Customer.id == id_).one()
```

在 Dell XPS13 运行 Linux 的 SQLAlchemy 1.3 版本中，此示例完成如下：

```py
test_orm_query : (10000 iterations); total time 3.440652 sec
```

在 1.4 版本中，上述代码不经修改即可完成：

```py
test_orm_query : (10000 iterations); total time 2.367934 sec
```

这个第一个测试表明，当使用缓存时，常规的 ORM 查询可以在很多次迭代中以**30% 更快**的速度运行。

功能的第二个变体是可选使用 Python lambdas 来延迟查询本身的构建。这是一种更复杂的方法变体，类似于版本 1.0.0 中引入的“烘焙查询”扩展。 “lambda” 功能可以以非常类似于烘焙查询的方式使用，除了它可以以临时方式用于任何 SQL 结构之外。它还包括扫描每次 lambda 调用的功能，以查找每次调用都会更改的绑定文字值，以及对其他结构的更改，例如每次查询不同的实体或列，同时仍然不必每次都运行实际代码。

使用此 API 如下所示：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    stmt = lambda_stmt(lambda: future_select(Customer))
    stmt += lambda s: s.where(Customer.id == id_)
    session.execute(stmt).scalar_one()
```

上述代码完成：

```py
test_orm_query_newstyle_w_lambdas : (10000 iterations); total time 1.247092 sec
```

此测试表明，使用较新的“select()”风格的 ORM 查询，与完全“烘焙”样式调用结合使用，后者缓存了整个构建过程，可以在很多次迭代中以**60% 更快**的速度运行，并且性能与被本地缓存系统取代的烘焙查询系统相当。

新系统利用现有的 `Connection.execution_options.compiled_cache` 执行选项，并直接向 `Engine` 添加缓存，该缓存使用 `Engine.query_cache_size` 参数进行配置。

API 和行为变化的重大部分是为了支持这一新功能而进行的。

另请参阅

SQL 编译缓存

[#4639](https://www.sqlalchemy.org/trac/ticket/4639) [#5380](https://www.sqlalchemy.org/trac/ticket/5380) [#4645](https://www.sqlalchemy.org/trac/ticket/4645) [#4808](https://www.sqlalchemy.org/trac/ticket/4808) [#5004](https://www.sqlalchemy.org/trac/ticket/5004)  ### 声明式现在与 ORM 集成，并带有新功能

大约十年左右的时间后，`sqlalchemy.ext.declarative` 包现在已集成到 `sqlalchemy.orm` 命名空间中，除了声明式“扩展”类仍然保持为声明式扩展之外。

`sqlalchemy.orm` 中新增的新类包括：

+   `registry` - 一个新的类，取代了“声明基类”的角色，作为映射类的注册表，可以通过字符串名称在`relationship()`调用中引用，并且不受任何特定类被映射的风格的影响。

+   `declarative_base()` - 这是在声明系统中一直在使用的相同声明基类，只是现在在内部引用了一个`registry`对象，并由`registry.generate_base()`方法实现，可以直接从`registry`中调用。`declarative_base()`函数会自动生成这个注册表，因此不会影响现有代码。`sqlalchemy.ext.declarative.declarative_base`名称仍然存在，在启用 2.0 弃用模式时会发出 2.0 弃用警告。

+   `declared_attr()` - 现在是`sqlalchemy.orm`的一部分的相同“声明属性”函数调用。`sqlalchemy.ext.declarative.declared_attr`名称仍然存在，在启用 2.0 弃用模式时会发出 2.0 弃用警告。

+   其他移入`sqlalchemy.orm`的名称包括`has_inherited_table()`，`synonym_for()`，`DeclarativeMeta`，`as_declarative()`。

另外，`instrument_declarative()`函数已被弃用，被`registry.map_declaratively()`取代。`ConcreteBase`、`AbstractConcreteBase`和`DeferredReflection`类仍然作为声明性扩展包中的扩展。

映射样式现在已经组织起来，它们都从`registry`对象扩展，并分为以下类别：

+   声明性映射

    +   使用`declarative_base()` 带有元类的基类

        +   使用`mapped_column()`的声明性表

        +   命令式表（又名“混合表”）

    +   使用`registry.mapped()` 声明性装饰器

        +   声明性表

        +   命令式表（混合）

            +   将 ORM 映射应用于现有数据类（传统数据类用法）

+   命令式（又名“经典”映射）

    +   使用`registry.map_imperatively()`

        +   使用命令式映射映射预先存在的数据类

现有的经典映射函数`sqlalchemy.orm.mapper()`仍然存在，但直接调用`sqlalchemy.orm.mapper()`已被弃用；新的`registry.map_imperatively()`方法现在通过`sqlalchemy.orm.registry()`路由请求，以便与其他声明性映射明确集成。

新方法与第三方类仪器系统互操作，这些系统必须在映射过程之前对类进行操作，允许声明性映射通过装饰器而不是声明性基类工作，以便像[dataclasses](https://docs.python.org/3/library/dataclasses.html)和[attrs](https://pypi.org/project/attrs/)这样的包可以与声明性映射一起使用，除了与经典映射一起使用。

声明性文档现已完全整合到 ORM 映射器配置文档中，并包括所有样式的映射示例，组织在一个地方。请查看新组织文档的开始部分 ORM 映射类概述。

另请参阅

ORM 映射类概述

Python Dataclasses, attrs 支持声明性，命令式映射

[#5508](https://www.sqlalchemy.org/trac/ticket/5508)  ### Python Dataclasses, attrs 支持声明性，命令式映射

除了在声明性现在与新功能整合到 ORM 中中引入的新声明性装饰器样式外，`Mapper`现在明确意识到 Python 的`dataclasses`模块，并将识别以这种方式配置的属性，并继续映射它们，而不像以前那样跳过它们。对于`attrs`模块，`attrs`已经从类中删除了自己的属性，因此已经与 SQLAlchemy 经典映射兼容。通过添加`registry.mapped()`装饰器，两个属性系统现在可以与声明性映射互操作。

另请参阅

将 ORM 映射应用于现有数据类（传统数据类用法）

使用命令式映射映射预先存在的数据类

[#5027](https://www.sqlalchemy.org/trac/ticket/5027)  ### Core 和 ORM 的异步 IO 支持

SQLAlchemy 现在支持使用全新的`asyncio`前端接口来支持 Python 的数据库驱动程序，用于`Connection`的 Core 使用以及用于 ORM 使用的`Session`，使用`AsyncConnection`和`AsyncSession`对象。

注意

初始的 SQLAlchemy 1.4 版本应该考虑新的 asyncio 功能是**alpha 级别**的。这是一种全新的东西，使用了一些以前不熟悉的编程技术。

初始支持的数据库 API 是用于 PostgreSQL 的 asyncpg asyncio 驱动程序。

SQLAlchemy 的内部特性完全集成了[greenlet](https://greenlet.readthedocs.io/en/latest/)库，以便将 SQLAlchemy 内部的执行流程适应于将 asyncio 的`await`关键字从数据库驱动器传播到端用户 API，该 API 具有`async`方法。使用这种方法，asyncpg 驱动程序在 SQLAlchemy 自己的测试套件中完全可操作，并与大多数 psycopg2 特性兼容。这种方法经过了 greenlet 项目的开发人员的审查和改进，对此 SQLAlchemy 表示感激。

用户接口的`async` API 本身集中于像`AsyncEngine.connect()`和`AsyncConnection.execute()`这样的 IO 导向方法。新的 Core 构造严格支持 2.0 样式的使用方式；这意味着所有语句必须在给定连接对象的情况下调用，即在这种情况下为`AsyncConnection`。

在 ORM 中，支持 2.0 样式的查询执行，使用`select()`构造与`AsyncSession.execute()`结合；传统的`Query`对象本身不受`AsyncSession`类支持。

ORM 功能，如延迟加载相关属性以及过期属性的非过期化，在传统的 asyncio 编程模型中是被禁止的，因为它们表示将隐式运行的 IO 操作在 Python 的`getattr()`操作的范围内。为了克服这一问题，**传统** asyncio 应用程序应该适度利用 eager loading 技术，并放弃使用诸如 expire on commit 之类的特性，以便不需要这些加载。

对于选择与传统**决裂**的 asyncio 应用程序开发人员，新的 API 提供了一个**严格可选的功能**，使希望利用此类 ORM 功能的应用程序可以选择将与数据库相关的代码组织到函数中，然后使用 `AsyncSession.run_sync()` 方法在 greenlets 中运行。请参阅 Asyncio Integration 中的 `greenlet_orm.py` 示例以进行演示。

还提供了对异步游标的支持，使用新方法 `AsyncConnection.stream()` 和 `AsyncSession.stream()`，支持一个新的 `AsyncResult` 对象，该对象本身提供了常见方法的可等待版本，如 `AsyncResult.all()` 和 `AsyncResult.fetchmany()`。核心和 ORM 都与传统 SQLAlchemy 中使用“服务器端游标”的功能集成。

另请参见

异步 I/O (asyncio)

Asyncio Integration

[#3414](https://www.sqlalchemy.org/trac/ticket/3414)  ### 许多核心和 ORM 语句对象现在在编译阶段执行大部分构建和验证工作

1.4 系列的一个重要举措是接近核心 SQL 语句和 ORM 查询的模型，以实现高效、可缓存的语句创建和编译模型，其中编译步骤将被缓存，基于创建的语句对象生成的缓存键，该对象本身是为每次使用新创建的。为实现这一目标，特别是在构建语句时发生的大部分 Python 计算，特别是 ORM `Query` 和 `select()` 构造在用于调用 ORM 查询时，正在移至语句的编译阶段，该阶段仅在调用语句后发生，并且仅在语句的编译形式尚未被缓存时才会发生。

从最终用户的角度来看，这意味着基于传递给对象的参数可能引发的某些错误消息将不再立即引发，而是仅在首次调用语句时发生。这些条件始终是结构性的，而不是数据驱动的，因此不会由于缓存语句而错过此类条件的风险。

属于此类别的错误条件包括：

+   当构造`_selectable.CompoundSelect`（例如 UNION，EXCEPT 等）并且传递的 SELECT 语句列数不同时，现在会引发`CompileError`；以前，在语句构造时会立即引发`ArgumentError`。

+   当调用`Query.join()`时可能出现的各种错误条件将在语句编译时进行评估，而不是在首次调用方法时。

可能会发生变化的其他事情涉及直接`Query`对象：

+   当调用`Query.statement`访问器时，行为可能会有所不同。返回的`Select`对象现在是与`Query`中存在的相同状态的直接副本，而不执行任何 ORM 特定的编译（这意味着速度大大提高）。然而，`Select`将不会像 1.3 中那样具有相同的内部状态，包括如果在`Query`中没有明确声明，则明确拼写出 FROM 子句等内容。这意味着依赖于操作此`Select`语句的代码，例如调用`Select.with_only_columns()`等方法，可能需要适应 FROM 子句。

另请参阅

透明 SQL 编译缓存添加到 Core，ORM 中的所有 DQL，DML 语句  ### 修复了内部导入约定，使代码检查工具可以正常工作

SQLAlchemy 长期以来一直使用参数注入装饰器来帮助解决相互依赖的模块导入，就像这样：

```py
@util.dependency_for("sqlalchemy.sql.dml")
def insert(self, dml, *args, **kw): ...
```

上述函数将被重写，不再在外部具有`dml`参数。这会让代码检查工具看到函数缺少参数而感到困惑。已经内部实现了一种新方法，使函数签名不再被修改，模块对象在函数内部获取。

[#4656](https://www.sqlalchemy.org/trac/ticket/4656)

[#4689](https://www.sqlalchemy.org/trac/ticket/4689)  ### 支持 SQL 正则表达式操作符

期待已久的功能，为数据库正则表达式操作符提供了基本支持，以补充`ColumnOperators.like()`和`ColumnOperators.match()`操作套件。新功能包括实现类似正则表达式匹配的`ColumnOperators.regexp_match()`函数，以及实现正则表达式字符串替换的`ColumnOperators.regexp_replace()`函数。

支持的后端包括 SQLite、PostgreSQL、MySQL / MariaDB 和 Oracle。SQLite 后端仅支持“regexp_match”而不支持“regexp_replace”。

正则表达式语法和标志**不是通用于所有后端**。未来的功能将允许一次指定多个正则表达式语法，以便在不同后端之间动态切换。

对于 SQLite，Python 的`re.search()`函数已被确定为实现，无需额外参数。

另请参阅

`ColumnOperators.regexp_match()`

`ColumnOperators.regexp_replace()`

正则表达式支持 - SQLite 实现注意事项

[#1390](https://www.sqlalchemy.org/trac/ticket/1390)  ### SQLAlchemy 2.0 弃用模式

1.4 版本的主要目标之一是提供一个“过渡”版本，以便应用程序可以逐渐迁移到 SQLAlchemy 2.0。为此，1.4 版本的一个主要特性是“2.0 弃用模式”，这是一系列针对每个可检测到的 API 模式发出的弃用警告，在版本 2.0 中将以不同方式工作。所有警告都使用`RemovedIn20Warning`类。由于这些警告影响包括`select()`和`Engine`构造在内的基础模式，即使是简单的应用程序也可能生成大量警告，直到适当的 API 更改完成。因此，默认情况下警告模式是关闭的，直到开发人员启用环境变量`SQLALCHEMY_WARN_20=1`。

要了解如何完整使用 2.0 弃用模式，请参阅迁移到 2.0 第二步 - 打开 RemovedIn20Warnings。

另请参见

SQLAlchemy 2.0 - 主要迁移指南

迁移到 2.0 第二步 - 打开 RemovedIn20Warnings  ### Python 3.6 是最低 Python 3 版本；Python 2.7 仍受支持

由于 Python 3.5 在 2020 年 9 月已达到生命周期终点，SQLAlchemy 1.4 现在将版本 3.6 作为最低 Python 3 版本。Python 2.7 仍然受支持，但 SQLAlchemy 1.4 系列将是最后一个支持 Python 2 的系列。

### ORM 查询在内部与 select、update、delete 统一；2.0 风格的执行可用

对于版本 2.0 的 SQLAlchemy 最大的概念性变化，实际上也是在 1.4 版本中，是 Core 中的`Select`构造和 ORM 中的`Query`对象之间的巨大分离已被移除，以及`Query.update()`和`Query.delete()`方法与`Update`和`Delete`的关系。

关于`Select`和`Query`，这两个对象在许多版本中具有类似的、大部分重叠的 API，甚至可以在两者之间切换，但在使用模式和行为上仍然有很大的不同。这背后的历史背景是，`Query`对象是为了克服`Select`对象的缺点而引入的，后者曾经是 ORM 对象查询的核心，但只能根据`Table`元数据进行查询。然而，`Query`只有一个简单的接口来加载对象，直到经过多个重大版本的发布，它最终才获得了大部分`Select`对象的灵活性，这导致这两个对象变得非常相似，但仍然在很大程度上不兼容。

在 1.4 版本中，所有 Core 和 ORM SELECT 语句都直接从`Select`对象渲染；当使用`Query`对象时，在语句调用时，它会将其状态复制到一个`Select`对象中，然后使用 2.0 风格执行内部调用。未来，`Query`对象将仅作为传统遗留，应用程序将被鼓励转向 2.0 风格执行，允许 Core 构造自由地针对 ORM 实体使用：

```py
with Session(engine, future=True) as sess:
    stmt = (
        select(User)
        .where(User.name == "sandy")
        .join(User.addresses)
        .where(Address.email_address.like("%gmail%"))
    )

    result = sess.execute(stmt)

    for user in result.scalars():
        print(user)
```

关于上面的示例需要注意的事项：

+   `Session`和`sessionmaker`对象现在具有完整的上下文管理器（即`with:`语句）功能；请参阅打开和关闭会话的修订文档以获取示例。

+   在 1.4 系列中，所有 2.0 风格的 ORM 调用都使用了一个包含`Session`的标志设置为`True`的标志；这个标志表示`Session`应该具有 2.0 风格的行为，其中包括 ORM 查询可以从`execute`中调用，以及一些事务特性的变化。在 2.0 版本中，这个标志将始终为`True`。

+   `select()`构造不再需要在列子句周围加括号；请参考 select(), case()现在接受位置表达式以了解此改进的背景。

+   `select()` / `Select`对象具有一个`Select.join()`方法，其行为类似于`Query`的方法，甚至可以容纳 ORM 关系属性（而不会破坏 Core 和 ORM 之间的分离！）- 请参考 select().join()和 outerjoin()向当前查询添加 JOIN 条件，而不是创建子查询以了解此背景。

+   与 ORM 实体一起工作并预计返回 ORM 结果的语句是使用`Session.execute()`来调用的。查看查询以获取入门指南。另请参阅 ORM Session.execute()在所有情况下使用“future”风格结果集中的以下注意事项。

+   返回一个`Result`对象，而不是一个普通列表，这本身是以前`ResultProxy`对象的一个更复杂的版本；这个对象现在被用于 Core 和 ORM 结果。查看新的 Result 对象，RowProxy 不再是一个“代理”；现在被称为 Row 并且行为类似于增强的命名元组，以及 Query 返回的“KeyedTuple”对象被 Row 替换以获取更多信息。

在 SQLAlchemy 的文档中，将会有许多关于 1.x 风格和 2.0 风格执行的引用。这是为了区分两种查询风格，并尝试向前文档化新的调用风格。在 SQLAlchemy 2.0 中，虽然`Query`对象可能仍然是一个遗留构造，但它将不再在大多数文档中出现。

对“批量更新和删除”进行了类似的调整，以便 Core `update()`和`delete()`可以用于批量操作。类似以下的批量更新：

```py
session.query(User).filter(User.name == "sandy").update(
    {"password": "foobar"}, synchronize_session="fetch"
)
```

现在可以以 2.0 风格实现（实际上上述在内部以这种方式运行）如下：

```py
with Session(engine, future=True) as sess:
    stmt = (
        update(User)
        .where(User.name == "sandy")
        .values(password="foobar")
        .execution_options(synchronize_session="fetch")
    )

    sess.execute(stmt)
```

请注意使用`Executable.execution_options()`方法传递 ORM 相关选项。现在“执行选项”的使用在 Core 和 ORM 中更加普遍，许多来自`Query`的 ORM 相关方法现在被实现为执行选项（查看`Query.execution_options()`以获取一些示例）。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南

[#5159](https://www.sqlalchemy.org/trac/ticket/5159)

### ORM `Session.execute()`在所有情况下都使用“future”风格的`Result`集

如在 RowProxy 不再是“代理”；现在称为 Row 并且行为类似增强命名元组中所述，当与设置了`create_engine.future`参数为`True`的`Engine`一起使用时，`Result`和`Row`对象现在具有“命名元组”行为。特别是这些“命名元组”行现在包括一个行为变化，即使用`in`的 Python 包含表达式，例如：

```py
>>> engine = create_engine("...", future=True)
>>> conn = engine.connect()
>>> row = conn.execute.first()
>>> "name" in row
True
```

上述包含测试将使用**值包含**，而不是**键包含**；`row`需要具有“name”的**值**才能返回`True`。

在 SQLAlchemy 1.4 版本中，当`create_engine.future`参数设置为`False`时，将返回传统风格的`LegacyRow`对象，其具有之前 SQLAlchemy 版本的部分命名元组行为，其中包含检查仍然使用键包含性；如果行中有名为“name”的**列**，则`"name" in row`将返回 True，而不是值。

在使用`Session.execute()`时，完整的命名元组样式被**无条件地**启用，这意味着`"name" in row`将使用**值包含性**作为测试，而**不是**键包含性。这是为了适应`Session.execute()`现在返回一个`Result`，该结果还适用于 ORM 结果，即使是由`Query.all()`返回的传统 ORM 结果行也使用值包含性。

这是从 SQLAlchemy 1.3 到 1.4 的行为变化。要继续接收键包含集合，请使用`Result.mappings()`方法来接收返回行为字典的`MappingResult`：

```py
for dict_row in session.execute(text("select id from table")).mappings():
    assert "id" in dict_row
```

### 透明 SQL 编译缓存添加到 Core、ORM 中的所有 DQL、DML 语句

这是 SQLAlchemy 版本中最广泛的变化之一，经过数月的重新组织和重构，从 Core 的基础一直到 ORM，现在允许大部分涉及从用户构建的语句生成 SQL 字符串和相关语句元数据的 Python 计算在内存中被缓存，因此对于相同的语句构造的后续调用将使用 35-60%更少的 CPU 资源。

这种缓存不仅限于构建 SQL 字符串，还包括构建将 SQL 构造与结果集链接起来的结果获取结构，在 ORM 中还包括适应 ORM 启用的属性加载器、关系急加载器和其他选项，以及每次 ORM 查询试图运行并从结果集构建 ORM 对象时必须构建的对象构造例程。

为了介绍该功能的一般概念，给出来自性能套件的代码如下，它将调用一个非常简单的查询“n”次，n 的默认值为 10000。该查询仅返回一行，因为我们希望减少的开销是**许多小查询**的开销。对于返回许多行的查询，优化并不那么显著：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    result = session.query(Customer).filter(Customer.id == id_).one()
```

在运行 Linux 的 Dell XPS13 上的 SQLAlchemy 1.3 版本中，此示例完成如下：

```py
test_orm_query : (10000 iterations); total time 3.440652 sec
```

在 1.4 中，上面的代码没有修改完成了：

```py
test_orm_query : (10000 iterations); total time 2.367934 sec
```

这个第一个测试表明，在使用缓存时，常规的 ORM 查询可以在许多迭代中运行速度快**30%**。

该功能的第二个变体是可选使用 Python lambda 来推迟查询本身的构建。这是“Baked Query”扩展所使用的方法的更复杂变体，该扩展是在 1.0.0 版本中引入的。 “lambda”功能可以以与烘焙查询非常相似的方式使用，只是它以一种临时方式可用于任何 SQL 构造。它还包括扫描每次调用 lambda 以查找在每次调用时更改的绑定文字值的能力，以及对其他构造的更改，例如每次查询来自不同实体或列，同时仍然不必每次运行实际代码。

使用这个 API 看起来如下：

```py
session = Session(bind=engine)
for id_ in random.sample(ids, n):
    stmt = lambda_stmt(lambda: future_select(Customer))
    stmt += lambda s: s.where(Customer.id == id_)
    session.execute(stmt).scalar_one()
```

上面的代码完成了：

```py
test_orm_query_newstyle_w_lambdas : (10000 iterations); total time 1.247092 sec
```

这个测试表明，使用较新的“select()”风格的 ORM 查询，结合完全“烘焙”风格的调用，可以在许多迭代中运行速度快**60%**，并且性能与现在被本地缓存系统取代的烘焙查询系统大致相同。

新系统利用现有的`Connection.execution_options.compiled_cache`执行选项，并直接向`Engine`添加缓存，该缓存使用`Engine.query_cache_size`参数进行配置。

1.4 版本中的 API 和行为变化的一个重要部分是为了支持这一新功能。

另请参阅

SQL 编译缓存

[#4639](https://www.sqlalchemy.org/trac/ticket/4639) [#5380](https://www.sqlalchemy.org/trac/ticket/5380) [#4645](https://www.sqlalchemy.org/trac/ticket/4645) [#4808](https://www.sqlalchemy.org/trac/ticket/4808) [#5004](https://www.sqlalchemy.org/trac/ticket/5004)

### 声明式现在已经与 ORM 集成，并具有新功能

大约十年后，`sqlalchemy.ext.declarative`包现在已经集成到`sqlalchemy.orm`命名空间中，除了保留为声明式扩展的声明式“extension”类。

添加到`sqlalchemy.orm`的新类包括：

+   `registry` - 一个新的类，取代了“声明基类”的角色，作为映射类的注册表，可以通过字符串名称在`relationship()`调用中引用，并且不受任何特定类被映射的风格的影响。

+   `declarative_base()` - 这是在声明系统中一直在使用的相同声明基类，只是现在在内部引用了一个`registry`对象，并由`registry.generate_base()`方法实现，可以直接从`registry`调用。`declarative_base()`函数会自动生成这个注册表，因此不会影响现有代码。`sqlalchemy.ext.declarative.declarative_base`名称仍然存在，在启用 2.0 弃用模式时会发出 2.0 弃用警告。

+   `declared_attr()` - 现在是`sqlalchemy.orm`的一部分的相同“声明属性”函数调用。`sqlalchemy.ext.declarative.declared_attr`名称仍然存在，在启用 2.0 弃用模式时会发出 2.0 弃用警告。

+   其他移至`sqlalchemy.orm`的名称包括`has_inherited_table()`，`synonym_for()`，`DeclarativeMeta`，`as_declarative()`。

另外，`instrument_declarative()`函数已被弃用，被`registry.map_declaratively()`取代。`ConcreteBase`、`AbstractConcreteBase`和`DeferredReflection`类仍然作为声明式扩展包中的扩展。

映射样式现在已经组织起来，它们都从`registry`对象扩展，并分为以下类别：

+   声明式映射

    +   使用`declarative_base()`基类与元类

        +   具有 mapped_column()的声明式表

        +   命令式表（又称“混合表”）

    +   使用`registry.mapped()`声明式装饰器

        +   声明式表

        +   命令式表（混合）

            +   将 ORM 映射应用于现有数据类（传统数据类用法）

+   命令式（又称“经典”映射）

    +   使用`registry.map_imperatively()`

        +   使用命令式映射映射预先存在的数据类

现有的经典映射函数`sqlalchemy.orm.mapper()`仍然存在，但不建议直接调用`sqlalchemy.orm.mapper()`；新的`registry.map_imperatively()`方法现在通过`sqlalchemy.orm.registry()`路由请求，以便与其他声明式映射无歧义地集成。

新方法与第三方类仪器系统互操作，这些系统必须在映射过程之前对类进行操作，允许通过装饰器而不是声明性基类工作的声明性映射，以便像 [dataclasses](https://docs.python.org/3/library/dataclasses.html) 和 [attrs](https://pypi.org/project/attrs/) 这样的包可以与声明性映射一起使用，除了与经典映射一起使用。

声明式文档现已完全整合到 ORM 映射器配置文档中，并包括所有样式映射的示例，组织在一个地方。请查看重新组织文档的开始部分 ORM 映射类概述。

另请参阅

ORM 映射类概述

Python Dataclasses, attrs 支持声明式、命令式映射

[#5508](https://www.sqlalchemy.org/trac/ticket/5508)

### Python Dataclasses, attrs 支持声明式、命令式映射

除了在 声明式现在与新功能一起集成到 ORM 中 中引入的新声明式装饰器样式外，`Mapper` 现在明确了解 Python `dataclasses` 模块，并将识别以这种方式配置的属性，并继续映射它们，而不像以前那样跳过它们。对于 `attrs` 模块，`attrs` 已经从类中删除了自己的属性，因此已经与 SQLAlchemy 经典映射兼容。通过 `registry.mapped()` 装饰器的添加，两个属性系统现在也可以与声明性映射互操作。

另请参阅

将 ORM 映射应用于现有数据类（传统数据类用法）

使用命令式映射映射预先存在的数据类

[#5027](https://www.sqlalchemy.org/trac/ticket/5027)

### Core 和 ORM 的异步 IO 支持

SQLAlchemy 现在支持使用全新的 asyncio 前端接口来支持 Python `asyncio` 兼容的数据库驱动程序，用于 Core 使用的 `Connection` 和用于 ORM 使用的 `Session`，使用 `AsyncConnection` 和 `AsyncSession` 对象。

注意

新的 asyncio 功能在 SQLAlchemy 1.4 的初始版本中应被视为**alpha 级别**。这是一些以前不熟悉的编程技术的全新内容。

初始数据库 API 支持的是 asyncpg PostgreSQL 的 asyncio 驱动程序。

SQLAlchemy 的内部特性完全集成，通过使用 [greenlet](https://greenlet.readthedocs.io/en/latest/) 库来调整在 SQLAlchemy 内部的执行流程，以将 asyncio 的 `await` 关键字从数据库驱动程序传播到最终用户 API，该 API 具有 `async` 方法。使用这种方法，asyncpg 驱动程序在 SQLAlchemy 自己的测试套件中完全可操作，并与大多数 psycopg2 特性兼容。这种方法经过了 greenlet 项目的开发人员的审查和改进，对此 SQLAlchemy 表示感谢。

用户面向的 `async` API 本身主要围绕 IO 导向的方法，如 `AsyncEngine.connect()` 和 `AsyncConnection.execute()`。新的 Core 结构严格只支持 2.0 风格 的用法；这意味着所有语句必须在给定连接对象的情况下调用，本例中为 `AsyncConnection`。

在 ORM 中，支持 2.0 风格 的查询执行，使用 `select()` 结构与 `AsyncSession.execute()` 结合使用；传统的 `Query` 对象本身不受 `AsyncSession` 类支持。

传统的 asyncio 编程模型不允许诸如延迟加载相关属性以及过期属性的非 ORM 特性，因为它们表示 IO 操作，这些操作会在 Python 的 `getattr()` 操作的范围内隐式运行。为了克服这一点，**传统** asyncio 应用程序应该审慎使用 急加载 技术，并放弃使用诸如 提交时过期 等特性，以避免需要这样的加载。

对于选择**打破**传统的 asyncio 应用程序开发人员，新的 API 提供了一个**严格可选的功能**，使希望使用此类 ORM 功能的应用程序可以选择将与数据库相关的代码组织到函数中，然后可以使用`AsyncSession.run_sync()`方法在 greenlets 中运行。请参阅 Asyncio 集成中的`greenlet_orm.py`示例进行演示。

支持异步游标的方法也提供了新的方法`AsyncConnection.stream()`和`AsyncSession.stream()`，支持一个新的`AsyncResult`对象，该对象本身提供了常见方法的可等待版本，如`AsyncResult.all()`和`AsyncResult.fetchmany()`。Core 和 ORM 都与这一特性集成，对应于传统 SQLAlchemy 中“服务器端游标”的使用。

另请参阅

异步 I/O（asyncio）

Asyncio 集成

[#3414](https://www.sqlalchemy.org/trac/ticket/3414)

### 许多 Core 和 ORM 语句对象现在在编译阶段执行大部分构建和验证工作

1.4 系列中的一个重要举措是处理 Core SQL 语句以及 ORM 查询的模型，以允许有效的、可缓存的语句创建和编译模型，其中编译步骤将被缓存，基于由创建的语句对象生成的缓存键，该对象本身为每次使用新创建。为实现这一目标，构建语句中发生的大部分 Python 计算，特别是 ORM `Query`以及在调用 ORM 查询时使用的`select()` 构造时的计算，正在被移至语句的编译阶段，该阶段仅在语句被调用后发生，并且仅在语句的编译形式尚未被缓存时才会发生。

从最终用户的角度来看，这意味着基于传递给对象的参数可能引发的某些错误消息将不再立即引发，而是仅在首次调用语句时发生。这些条件始终是结构性的，而不是数据驱动的，因此不会因为缓存语句而错过这种条件。

属于此类别的错误条件包括：

+   当构建`_selectable.CompoundSelect`（例如 UNION、EXCEPT 等）时，传递的 SELECT 语句列数不相同时，现在会引发`CompileError`；之前，在语句构建时会立即引发`ArgumentError`。

+   在调用`Query.join()`时可能出现的各种错误条件将在语句编译时进行评估，而不是在首次调用方法时。

其他可能发生变化的事情涉及直接的`Query`对象：

+   在调用`Query.statement`访问器时，行为可能会有所不同。现在返回的`Select`对象是与`Query`中存在的相同状态的直接副本，而不执行任何 ORM 特定的编译（这意味着速度大大提高）。但是，`Select`将不具有与 1.3 版本中相同的内部状态，包括如果在`Query`中未明确声明，则明确拼写出 FROM 子句。这意味着依赖于操作此`Select`语句的代码，例如调用`Select.with_only_columns()`等方法可能需要适应 FROM 子句。

另请参阅

透明 SQL 编译缓存添加到 Core、ORM 中的所有 DQL、DML 语句

### 修复了内部导入约定，使代码检查工具可以正常工作

SQLAlchemy 长期以来一直使用参数注入装饰器来帮助解决相互依赖的模块导入，就像这样：

```py
@util.dependency_for("sqlalchemy.sql.dml")
def insert(self, dml, *args, **kw): ...
```

上述函数将被重写，不再在外部具有 `dml` 参数。这会让代码检测工具看到函数缺少参数而感到困惑。内部已经实现了一种新方法，使函数签名不再被修改，而是在函数内部获取模块对象。

[#4656](https://www.sqlalchemy.org/trac/ticket/4656)

[#4689](https://www.sqlalchemy.org/trac/ticket/4689)

### 支持 SQL 正则表达式运算符

期待已久的功能是为数据库正则表达式运算符添加基本支持，以补充 `ColumnOperators.like()` 和 `ColumnOperators.match()` 套件的操作。新功能包括 `ColumnOperators.regexp_match()` 实现了类似的正则表达式匹配函数，以及 `ColumnOperators.regexp_replace()` 实现了正则表达式字符串替换函数。

支持的后端包括 SQLite、PostgreSQL、MySQL / MariaDB 和 Oracle。SQLite 后端仅支持“regexp_match”而不支持“regexp_replace”。

正则表达式语法和标志**不是通用于所有后端**的。未来的功能将允许一次指定多个正则表达式语法，以便在不同后端之间动态切换。

对于 SQLite，Python 的 `re.search()` 函数在没有额外参数的情况下被确定为实现。

另请参阅

`ColumnOperators.regexp_match()`

`ColumnOperators.regexp_replace()`

正则表达式支持 - SQLite 实现说明

[#1390](https://www.sqlalchemy.org/trac/ticket/1390)

### SQLAlchemy 2.0 弃用模式

1.4 版本的主要目标之一是提供一个“过渡”版本，以便应用程序可以逐渐迁移到 SQLAlchemy 2.0。为此，1.4 版本的一个主要特性是“2.0 弃用模式”，它是一系列弃用警告，针对每个在 2.0 版本中会有不同工作方式的可检测 API 模式发出。所有警告都使用`RemovedIn20Warning`类。由于这些警告影响到基础模式，包括`select()`和`Engine`构造，即使是简单的应用程序也可能生成大量警告，直到适当的 API 更改完成。因此，默认情况下警告模式被关闭，直到开发人员启用环境变量`SQLALCHEMY_WARN_20=1`。

要了解如何使用 2.0 弃用模式的完整步骤，请参阅 迁移至 2.0 第二步 - 打开 RemovedIn20Warnings。

另请参阅

SQLAlchemy 2.0 - 主要迁移指南

迁移至 2.0 第二步 - 打开 RemovedIn20Warnings

## API 和行为变化 - 核心

### SELECT 语句不再隐式地被视为 FROM 子句

这个变化是多年来 SQLAlchemy 中的一个较大的概念性变化之一，但希望最终用户的影响相对较小，因为这个变化更符合像 MySQL 和 PostgreSQL 这样的数据库在任何情况下所需的情况。

最直接显着的影响是，一个`select()`现在不能直接嵌套在另一个`select()`中，而不明确地先将内部`select()`转换为子查询。这在历史上是通过使用`SelectBase.alias()`方法来执行的，但现在更明确地适合使用新方法`SelectBase.subquery()`；这两种方法都是做同样的事情。现在返回的对象是`Subquery`，它与`Alias`对象非常相似，并共享一个公共基类`AliasedReturnsRows`。

也就是说，现在会引发以下错误：

```py
stmt1 = select(user.c.id, user.c.name)
stmt2 = select(addresses, stmt1).select_from(addresses.join(stmt1))
```

引发：

```py
sqlalchemy.exc.ArgumentError: Column expression or FROM clause expected,
got <...Select object ...>. To create a FROM clause from a <class
'sqlalchemy.sql.selectable.Select'> object, use the .subquery() method.
```

正确的调用形式应该是（同时注意到 不再需要为 select() 使用括号）：

```py
sq1 = select(user.c.id, user.c.name).subquery()
stmt2 = select(addresses, sq1).select_from(addresses.join(sq1))
```

注意到上面提到的 `SelectBase.subquery()` 方法本质上等同于使用 `SelectBase.alias()` 方法。

这种更改的基本原理是：

+   为了支持将 `Select` 与 `Query` 统一起来，`Select` 对象需要具有实际将 JOIN 条件添加到现有 FROM 子句的 `Select.join()` 和 `Select.outerjoin()` 方法，正如用户一直期望它所做的那样。之前的行为是，必须与 `FromClause` 的行为相一致，它会生成一个未命名的子查询，然后再与之 JOIN，这是一个完全无用的功能，只会混淆那些不幸尝试此操作的用户。这一变更在 select().join() 和 outerjoin() 将 JOIN 条件添加到当前查询，而不是创建子查询 中进行了讨论。

+   将一个 SELECT 包含在另一个 SELECT 的 FROM 子句中，而不先创建别名或子查询的行为会导致创建一个未命名的子查询。虽然标准 SQL 支持此语法，但实际上大多数数据库都会拒绝它。例如，MySQL 和 PostgreSQL 都直接拒绝使用未命名的子查询：

    ```py
    #  MySQL  /  MariaDB:

    MariaDB  [(none)]>  select  *  from  (select  1);
    ERROR  1248  (42000):  Every  derived  table  must  have  its  own  alias

    #  PostgreSQL:

    test=>  select  *  from  (select  1);
    ERROR:  subquery  in  FROM  must  have  an  alias
    LINE  1:  select  *  from  (select  1);
      ^
    HINT:  For  example,  FROM  (SELECT  ...)  [AS]  foo.
    ```

    SQLite 这样的数据库接受它们，但通常情况下，从这样的子查询中生成的名称太模糊而无法使用：

    ```py
    sqlite>  CREATE  TABLE  a(id  integer);
    sqlite>  CREATE  TABLE  b(id  integer);
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  ON  a.id=id;
    Error:  ambiguous  column  name:  id
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  ON  a.id=b.id;
    Error:  no  such  column:  b.id

    #  use  a  name
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  AS  anon_1  ON  a.id=anon_1.id;
    ```

由于 `SelectBase` 对象不再是 `FromClause` 对象，因此像 `.c` 属性以及 `.select()` 这样的方法现在已经被弃用，因为它们意味着隐式生成子查询。`.join()` 和 `.outerjoin()` 方法现在被重新用于在现有查询中附加 JOIN 条件，方式与 `Query.join()` 类似，这正是用户一直期望这些方法做的事情。

新的属性 `SelectBase.selected_columns` 替代了 `.c` 属性。该属性解析为一个列集合，大多数人希望 `.c` 能做到的事情（但实际上不行），即引用 SELECT 语句中列子句中的列。一个常见的初学者错误是以下代码：

```py
stmt = select(users)
stmt = stmt.where(stmt.c.name == "foo")
```

上述代码看起来直观，会生成“SELECT * FROM users WHERE name='foo'”，但是经验丰富的 SQLAlchemy 用户会认识到，它实际上生成了一个无用的子查询，类似于“SELECT * FROM (SELECT * FROM users) WHERE name='foo'”。

但新的 `SelectBase.selected_columns` 属性确实适用于上述用例，因为在上述情况下，它直接链接到 `users.c` 集合中存在的列：

```py
stmt = select(users)
stmt = stmt.where(stmt.selected_columns.name == "foo")
```

[#4617](https://www.sqlalchemy.org/trac/ticket/4617)  ### select().join() 和 outerjoin() 将 JOIN 条件添加到当前查询，而不是创建子查询

为了实现统一`Query`和`Select`的目标，特别是对于 2.0 风格使用`Select`，至关重要的是有一个工作的`Select.join()`方法，其行为类似于`Query.join()`方法，向现有 SELECT 的 FROM 子句添加额外条目，然后返回新的`Select`对象以进行进一步修改，而不是将对象包装在一个无名子查询中并从该子查询返回 JOIN，这种行为对用户来说一直是几乎无用和完全误导的。

为了实现这一点，首先实现了 A SELECT statement is no longer implicitly considered to be a FROM clause，这将`Select`从必须是`FromClause`中分离出来；这消除了`Select.join()`需要返回一个`Join`对象而不是包含新 JOIN 的新版本的`Select`对象的要求。

从那时起，由于`Select.join()`和`Select.outerjoin()`已经存在行为，最初的计划是这些方法将被弃用，并且新的“有用”版本的方法将在一个备用的“未来”`Select`对象上可用，作为一个单独的导入。

然而，经过一段时间与这个特定代码库一起工作后，决定让两种不同类型的`Select`对象漂浮，每个对象的行为几乎相同，只是一些方法的行为略有不同，这将比简单地改变这两种方法的行为更具误导性和不便，因为`Select.join()`和`Select.outerjoin()`的现有行为几乎从不被使用，只会造成混乱。

因为当前行为非常无用，而新行为将非常有用和重要，因此决定在这一领域做出**严格的行为更改**，而不是等待另一年并在此期间拥有更加尴尬的 API。 SQLAlchemy 开发人员并不轻易进行完全破坏性的更改，然而这是一个非常特殊的情况，以前几乎不太可能使用这些方法的实现；正如在 A SELECT statement is no longer implicitly considered to be a FROM clause 中所指出的，主要数据库如 MySQL 和 PostgreSQL 在任何情况下都不允许未命名的子查询，并且从语法角度来看，从未命名的子查询进行 JOIN 几乎不可能是有用的，因为在其中明确引用列非常困难。

使用新实现，`Select.join()`和`Select.outerjoin()`现在行为与`Query.join()`非常相似，通过与左实体匹配添加 JOIN 条件到现有语句中：

```py
stmt = select(user_table).join(
    addresses_table, user_table.c.id == addresses_table.c.user_id
)
```

产生：

```py
SELECT  user.id,  user.name  FROM  user  JOIN  address  ON  user.id=address.user_id
```

与`Join`一样，如果可行，ON 子句将自动确定：

```py
stmt = select(user_table).join(addresses_table)
```

当在语句中使用 ORM 实体时，这基本上是使用 2.0 风格调用构建 ORM 查询的方式。 ORM 实体将在语句内部分配一个“插件”，以便在将语句编译为 SQL 字符串时将发生与 ORM 相关的编译规则。 更直接地说，`Select.join()`方法可以适应 ORM 关系，而不会破坏 Core 和 ORM 内部之间的严格分离：

```py
stmt = select(User).join(User.addresses)
```

另一个新方法 `Select.join_from()` 也被添加，它允许更容易地一次性指定连接的左侧和右侧：

```py
stmt = select(Address.email_address, User.name).join_from(User, Address)
```

产生：

```py
SELECT  address.email_address,  user.name  FROM  user  JOIN  address  ON  user.id  ==  address.user_id
```  ### URL 对象现在是不可变的

`URL` 对象已经被正式规范化，现在它呈现为一个具有固定字段数量的不可变的 `namedtuple`。此外，由 `URL.query` 属性表示的字典也是一个不可变的映射。对 `URL` 对象的修改并不是一个正式支持或记录的用例，这导致了一些开放式用例，使得很难拦截不正确的用法，最常见的是修改 `URL.query` 字典以包含非字符串元素。这也导致了在一个基本数据对象中允许可变性的所有常见问题，即不希望的变化泄漏到不希望 URL 改变的代码中。最后，`namedtuple` 的设计灵感来自 Python 的 `urllib.parse.urlparse()`，它将解析后的对象作为一个命名元组返回。

直接更改 API 的决定是基于权衡不可能的弃用路径（这将涉及将 `URL.query` 字典更改为一个特殊字典，当调用任何标准库变异方法时会发出弃用警告，此外，当字典保存任何类型的元素列表时，列表也必须在变异时发出弃用警告）与项目已经在第一次变异 `URL` 对象的不太可能用例之间的比较，以及像 [#5341](https://www.sqlalchemy.org/trac/ticket/5341) 这样的小改变在任何情况下都会造成向后不兼容性。对 `URL` 对象进行变异的主要用例是在 `CreateEnginePlugin` 扩展点内解析插件参数，这本身是一个相当新的添加，根据 Github 代码搜索的结果，有两个仓库在使用它，但都没有实际变异 URL 对象。

`URL`对象现在提供了一个丰富的接口来检查和生成新的`URL`对象。用于创建`URL`对象的现有机制，即`make_url()`函数，保持不变：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
```

对于以编程方式构建的代码，如果参数作为关键字参数传递而不是精确的 7 元组，则可能已经使用`URL`构造函数或`__init__`方法的代码将收到弃用警告。现在可以通过`URL.create()`方法使用关键字样式的构造函数：

```py
>>> from sqlalchemy.engine import URL
>>> url = URL.create("postgresql", "user", "pass", host="host", database="dbname")
>>> str(url)
'postgresql://user:pass@host/dbname'
```

通常可以使用`URL.set()`方法更改字段，该方法返回一个应用更改的新`URL`对象：

```py
>>> mysql_url = url.set(drivername="mysql+pymysql")
>>> str(mysql_url)
'mysql+pymysql://user:pass@host/dbname'
```

要更改`URL.query`字典的内容，可以使用`URL.update_query_dict()`等方法：

```py
>>> url.update_query_dict({"sslcert": "/path/to/crt"})
postgresql://user:***@host/dbname?sslcert=%2Fpath%2Fto%2Fcrt
```

要升级直接更改这些字段的代码，一个**向后和向前兼容的方法**是使用鸭子类型，如以下样式：

```py
def set_url_drivername(some_url, some_drivername):
    # check for 1.4
    if hasattr(some_url, "set"):
        return some_url.set(drivername=some_drivername)
    else:
        # SQLAlchemy 1.3 or earlier, mutate in place
        some_url.drivername = some_drivername
        return some_url

def set_ssl_cert(some_url, ssl_cert):
    # check for 1.4
    if hasattr(some_url, "update_query_dict"):
        return some_url.update_query_dict({"sslcert": ssl_cert})
    else:
        # SQLAlchemy 1.3 or earlier, mutate in place
        some_url.query["sslcert"] = ssl_cert
        return some_url
```

查询字符串保留其现有格式，作为字符串到字符串的字典，使用字符串序列表示多个参数。例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url(
...     "postgresql://user:pass@host/dbname?alt_host=host1&alt_host=host2&sslcert=%2Fpath%2Fto%2Fcrt"
... )
>>> url.query
immutabledict({'alt_host': ('host1', 'host2'), 'sslcert': '/path/to/crt'})
```

要处理`URL.query`属性的内容，使所有值规范化为序列，可以使用`URL.normalized_query`属性：

```py
>>> url.normalized_query
immutabledict({'alt_host': ('host1', 'host2'), 'sslcert': ('/path/to/crt',)})
```

查询字符串可以通过`URL.update_query_dict()`、`URL.update_query_pairs()`、`URL.update_query_string()`等方法追加：

```py
>>> url.update_query_dict({"alt_host": "host3"}, append=True)
postgresql://user:***@host/dbname?alt_host=host1&alt_host=host2&alt_host=host3&sslcert=%2Fpath%2Fto%2Fcrt
```

另请参阅

`URL`

#### 更改 CreateEnginePlugin

`CreateEnginePlugin` 也受到了这一更改的影响，因为自定义插件的文档指出应该使用 `dict.pop()` 方法从 URL 对象中删除已使用的参数。现在应该使用 `CreateEnginePlugin.update_url()` 方法来实现。向后兼容的方法如下所示：

```py
from sqlalchemy.engine import CreateEnginePlugin

class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        # check for 1.4 style
        if hasattr(CreateEnginePlugin, "update_url"):
            self.my_argument_one = url.query["my_argument_one"]
            self.my_argument_two = url.query["my_argument_two"]
        else:
            # legacy
            self.my_argument_one = url.query.pop("my_argument_one")
            self.my_argument_two = url.query.pop("my_argument_two")

        self.my_argument_three = kwargs.pop("my_argument_three", None)

    def update_url(self, url):
        # this method runs in 1.4 only and should be used to consume
        # plugin-specific arguments
        return url.difference_update_query(["my_argument_one", "my_argument_two"])
```

有关此类的完整详细信息，请参阅`CreateEnginePlugin`中的文档字符串，了解此类的使用方法。

[#5526](https://www.sqlalchemy.org/trac/ticket/5526)  ### select(), case() 现在接受位置表达式

正如本文档中的其他地方所见，`select()` 构造现在将按位置接受“列子句”参数，而不需要将它们作为列表传递：

```py
# new way, supports 2.0
stmt = select(table.c.col1, table.c.col2, ...)
```

在按位置发送参数时，不允许使用其他关键字参数。在 SQLAlchemy 2.0 中，上述调用风格将是唯一支持的调用风格。

在 1.4 版本期间，先前的调用风格仍将继续运行，该调用风格将列或其他表达式的列表作为列表传递：

```py
# old way, still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...])
```

上述的旧调用风格也接受了从大多数叙述性文档中删除的旧关键字参数。正是这些关键字参数的存在导致了首次传递列子句作为列表的原因：

```py
# very much the old way, but still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...], whereclause=table.c.col1 == 5)
```

两种风格之间的区别是第一个位置参数是否为列表。不幸的是，仍然可能存在一些看起来像下面的用法，其中“whereclause”的关键字被省略了：

```py
# very much the old way, but still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...], table.c.col1 == 5)
```

作为此更改的一部分，`Select` 构造还获得了 2.0 风格的“未来” API，其中包括更新的 `Select.join()` 方法以及像 `Select.filter_by()` 和 `Select.join_from()` 这样的方法。

在相关更改中，`case()` 构造也已经修改为按位置接受其 WHEN 子句列表，旧的调用风格也有类似的弃用跟踪：

```py
stmt = select(users_table).where(
    case(
        (users_table.c.name == "wendy", "W"),
        (users_table.c.name == "jack", "J"),
        else_="E",
    )
)
```

对于接受`*args`与值列表的 SQLAlchemy 构造的约定，就像`ColumnOperators.in_()`这样的构造中的后者情况一样，**位置参数用于结构规范，列表用于数据规范**。

另请参阅

select() 不再接受不同的构造参数，列按位置传递

select() 构造以“legacy”模式创建；关键字参数等

[#5284](https://www.sqlalchemy.org/trac/ticket/5284)  ### 所有 IN 表达式会动态为列表中的每个值渲染参数（例如，扩展参数）

首次引入的“扩展 IN”功能，见延迟扩展的 IN 参数集允许具有缓存语句的 IN 表达式，已经足够成熟，明显优于以前的渲染 IN 表达式的方法。随着这种方法的改进以处理空值列表，它现在是 Core / ORM 渲染 IN 参数列表的唯一方法。

SQLAlchemy 自首次发布以来一直存在的先前方法是，当一系列值传递给`ColumnOperators.in_()`方法时，在语句构造时，列表将被扩展为一系列单独的`BindParameter`对象。这受到的限制是，在语句执行时无法根据参数字典变化参数列表，这意味着无法独立缓存字符串 SQL 语句及其参数，也无法完全用于包含 IN 表达式的语句。

为了支持 Baked Queries 中描述的“烘焙查询”功能，需要一个可缓存版本的 IN，这就带来了“扩展 IN”功能。与现有行为相反，在语句构造时将参数列表展开为单独的`BindParameter`对象，该功能使用一个存储值列表的单个`BindParameter`；当语句由`Engine`执行时，它会根据传递给`Connection.execute()`调用的参数，在执行过程中动态将其“展开”为基于参数传递的单个绑定参数位置，并且已从先前执行中检索到的现有 SQL 字符串将使用正则表达式进行修改，以适应当前参数集。这允许相同的`Compiled`对象，存储渲染的字符串语句，针对修改传递给 IN 表达式的列表内容的不同参数集多次调用，同时仍保持将单个标量参数传递给 DBAPI 的行为。虽然一些 DBAPI 直接支持此功能，但通常不可用；“扩展 IN”功能现在为所有后端一致支持此行为。

1.4 的一个主要重点是在 Core 和 ORM 中允许真正的语句缓存，而不需要“烘焙”系统的尴尬，而且由于“扩展 IN”功能代表了一种更简单的构建表达式的方法，因此当传递值列表给 IN 表达式时，它现在会自动调用：

```py
stmt = select(A.id, A.data).where(A.id.in_([1, 2, 3]))
```

预执行字符串表示为：

```py
>>> print(stmt)
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  ([POSTCOMPILE_id_1]) 
```

要直接渲染值，请像以前一样使用`literal_binds`：

```py
>>> print(stmt.compile(compile_kwargs={"literal_binds": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (1,  2,  3) 
```

新增了一个名为“render_postcompile”的标志，作为一个辅助工具，允许当前绑定的值被渲染，就像传递给数据库一样：

```py
>>> print(stmt.compile(compile_kwargs={"render_postcompile": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (:id_1_1,  :id_1_2,  :id_1_3) 
```

引擎日志输出也显示了最终渲染的语句：

```py
INFO  sqlalchemy.engine.base.Engine  SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (?,  ?,  ?)
INFO  sqlalchemy.engine.base.Engine  (1,  2,  3)
```

作为这一变化的一部分，“空 IN”表达式的行为，其中列表参数为空，现在标准化为使用针对所谓“空集”的 IN 运算符。由于没有标准的空集 SQL 语法，因此使用返回零行的 SELECT，针对每个后端以特定方式定制，以便数据库将其视为空集；此功能首次在版本 1.3 中引入，并在扩展 IN 功能现在支持空列表中描述。在版本 1.2 中引入的`create_engine.empty_in_strategy`参数，作为迁移以前 IN 系统处理方式的手段，现已弃用，此标志不再起作用；如 IN / NOT IN 运算符的空集合行为现在可配置；默认表达式简化中所述，此标志允许方言在原始系统比较列与自身的情况下切换，这种情况被证明是一个巨大的性能问题，以及比较“1 != 1” 以生成“false” 表达式的新系统。1.3 引入的行为现在在所有情况下都更加正确，因为仍然使用 IN 运算符，并且不具有原始系统的性能问题。

此外，“扩展”参数系统已经泛化，以便为其他特定方言的用例提供服务，其中参数无法由 DBAPI 或后端数据库容纳；详细信息请参见在 Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数。

另请参见

在 Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数

扩展 IN 功能现在支持空列表

`BindParameter`

[#4645](https://www.sqlalchemy.org/trac/ticket/4645)  ### 内置的 FROM linting 将警告任何 SELECT 语句中潜在的笛卡尔积

由于核心表达式语言以及 ORM 建立在“隐式 FROMs”模型上，如果查询的任何部分引用了它，那么特定的 FROM 子句将自动添加，一个常见问题是 SELECT 语句，无论是顶级语句还是嵌套子查询，包含了未与查询中的其他 FROM 元素连接的 FROM 元素，导致结果集中出现所谓的“笛卡尔积”，即每个未连接的 FROM 元素之间的所有可能组合的行。在关系数据库中，这几乎总是一个不希望的结果，因为它会产生一个充满重复、不相关数据的巨大结果集。

尽管 SQLAlchemy 具有许多出色的功能，但特别容易发生这种问题，因为 SELECT 语句将自动从其他子句中看到的任何表添加到其 FROM 子句中。典型情况如下，其中两个表被 JOIN 在一起，但是 WHERE 子句中可能无意中与这两个表不匹配的额外条目将创建一个额外的 FROM 条目：

```py
address_alias = aliased(Address)

q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
)
```

上述查询从`User`和`address_alias`的 JOIN 中进行选择，后者是`Address`实体的别名。然而，在 WHERE 子句中直接使用了`Address`实体，因此上述查询将导致以下 SQL：

```py
SELECT
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  addresses,  users  JOIN  addresses  AS  addresses_1  ON  users.id  =  addresses_1.user_id
WHERE  addresses.email_address  =  :email_address_1
```

在上述 SQL 中，我们可以看到 SQLAlchemy 开发人员所称的“可怕逗号”，因为我们在 FROM 子句中看到了“FROM addresses, users JOIN addresses”，这是笛卡尔积的经典迹象；查询正在使用 JOIN 来将 FROM 子句连接在一起，但是因为其中一个没有连接，所以使用了逗号。上述查询将返回一个完整的行集，将“user”和“addresses”表在“id / user_id”列上连接在一起，然后将所有这些行直接应用于“addresses”表中的每一行的笛卡尔积。也就是说，如果有十个用户行和 100 个地址行，上述查询将返回其预期的结果行，可能是 100，因为所有地址行都将被选择，再乘以 100，因此总结果大小将为 10000 行。

“table1, table2 JOIN table3”模式在 SQLAlchemy ORM 中也经常出现，这是由于 ORM 功能的微妙错误应用，特别是与连接式急加载或连接式表继承相关的功能，以及 SQLAlchemy ORM 系统中的错误造成的。类似的问题也适用于使用“隐式连接”的 SELECT 语句，其中不使用 JOIN 关键字，而是通过 WHERE 子句将每个 FROM 元素与另一个元素链接起来。

多年来，Wiki 上有一个配方，它在查询执行时将图算法应用于`select()`构造，并检查查询的结构以查找这些未链接的 FROM 子句，通过 WHERE 子句和所有 JOIN 子句解析以确定 FROM 元素如何链接在一起，并确保所有 FROM 元素在单个图中连接在一起。这个配方现在已经被调整为成为`SQLCompiler`的一部分，现在如果检测到这种条件，它现在可以选择性地为语句发出警告。使用`create_engine.enable_from_linting`标志启用警告，默认情况下启用。linter 的计算开销非常低，而且它只在语句编译期间发生，这意味着对于缓存的 SQL 语句，它只会发生一次。

使用此功能，我们上面的 ORM 查询将发出警告：

```py
>>> q.all()
SAWarning: SELECT statement has a cartesian product between FROM
element(s) "addresses_1", "users" and FROM element "addresses".
Apply join condition(s) between each element to resolve.
```

linter 功能不仅适用于通过 JOIN 子句链接在一起的表，还适用于通过 WHERE 子句链接在一起的表。上面，我们可以添加一个 WHERE 子句，将新的`Address`实体与先前的`address_alias`实体链接起来，这将消除警告：

```py
q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
    .filter(Address.id == address_alias.id)
)  # resolve cartesian products,
# will no longer warn
```

笛卡尔积警告考虑**任何**两个 FROM 子句之间的任何链接都是一个解析，即使最终结果集仍然是低效的，因为 linter 仅旨在检测完全意外的 FROM 子句的常见情况。如果 FROM 子句在其他地方明确引用并链接到其他 FROM 子句，则不会发出警告：

```py
q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
    .filter(Address.id > address_alias.id)
)  # will generate a lot of rows,
# but no warning
```

如果明确声明，也允许完整的笛卡尔积；例如，如果我们想要`User`和`Address`的笛卡尔积，我们可以在`true()`上进行 JOIN，以便每一行都与其他每一行匹配；以下查询将返回所有行并不会产生警告：

```py
from sqlalchemy import true

# intentional cartesian product
q = session.query(User).join(Address, true())  # intentional cartesian product
```

默认情况下，只有当语句由`Connection`编译以执行时才会生成警告；调用`ClauseElement.compile()`方法不会发出警告，除非提供了 linting 标志：

```py
>>> from sqlalchemy.sql import FROM_LINTING
>>> print(q.statement.compile(linting=FROM_LINTING))
SAWarning: SELECT statement has a cartesian product between FROM element(s) "addresses" and FROM element "users".  Apply join condition(s) between each element to resolve.
SELECT  users.id,  users.name,  users.fullname,  users.nickname
FROM  addresses,  users  JOIN  addresses  AS  addresses_1  ON  users.id  =  addresses_1.user_id
WHERE  addresses.email_address  =  :email_address_1 
```

[#4737](https://www.sqlalchemy.org/trac/ticket/4737)  ### 新的 Result 对象

SQLAlchemy 2.0 的一个主要目标是统一 ORM 和 Core 之间如何处理“结果”。为实现这一目标，版本 1.4 引入了自 SQLAlchemy 成立以来一直存在的`ResultProxy`和`RowProxy`对象的新版本。

这些新对象在 `Result` 和 `Row` 中有文档记录，并且不仅用于 Core 结果集，还用于 ORM 中的 2.0 风格 结果。

这个结果对象与 `ResultProxy` 完全兼容，并包括许多新功能，现在这些功能同样适用于 Core 和 ORM 结果，包括方法：

`Result.one()` - 返回确切的单行，否则引发异常：

```py
with engine.connect() as conn:
    row = conn.execute(table.select().where(table.c.id == 5)).one()
```

`Result.one_or_none()` - 相同，但对于没有行的情况也返回 None

`Result.all()` - 返回所有行

`Result.partitions()` - 按块获取行：

```py
with engine.connect() as conn:
    result = conn.execute(
        table.select().order_by(table.c.id),
        execution_options={"stream_results": True},
    )
    for chunk in result.partitions(500):
        # process up to 500 records
        ...
```

`Result.columns()` - 允许对行进行切片和重新组织：

```py
with engine.connect() as conn:
    # requests x, y, z
    result = conn.execute(select(table.c.x, table.c.y, table.c.z))

    # iterate rows as y, x
    for y, x in result.columns("y", "x"):
        print("Y: %s X: %s" % (y, x))
```

`Result.scalars()` - 返回标量对象的列表，默认情况下是从第一列开始，但也可以选择其他列：

```py
result = session.execute(select(User).order_by(User.id))
for user_obj in result.scalars():
    ...
```

`Result.mappings()` - 返回字典而不是命名元组行：

```py
with engine.connect() as conn:
    result = conn.execute(select(table.c.x, table.c.y, table.c.z))

    for map_ in result.mappings():
        print("Y: %(y)s X: %(x)s" % map_)
```

当使用 Core 时，`Connection.execute()` 返回的对象是 `CursorResult` 的一个实例，它继续具有与 `ResultProxy` 相同的 API 功能，包括插入的主键、默认值、行数等。对于 ORM，将返回一个 `Result` 的子类，它执行将 Core 行转换为 ORM 行的操作，然后允许进行所有相同的操作。

另请参阅

ORM 查询与 Core Select 统一 - 在 2.0 迁移文档中

[#5087](https://www.sqlalchemy.org/trac/ticket/5087)

[#4395](https://www.sqlalchemy.org/trac/ticket/4395)

[#4959](https://www.sqlalchemy.org/trac/ticket/4959)  ### RowProxy 不再是“代理”；现在称为 Row，并且像增强型命名元组一样运行

`RowProxy`类，代表 Core 结果集中的单个数据库结果行，现在被称为`Row`，不再是一个“代理”对象；这意味着当返回`Row`对象时，该行是一个简单的元组，其中包含数据的最终形式，已经通过与数据类型相关的结果行处理函数处理过（例如将数据库中的日期字符串转换为`datetime`对象，将 JSON 字符串转换为 Python 的`json.loads()`结果等）。

这样做的直接原因是为了使行更像 Python 的命名元组，而不是映射，元组中的值是元组上的`__contains__`运算符的主题，而不是键。当`Row`像命名元组一样工作时，它就适合用作 ORM 的`KeyedTuple`对象的替代品，从而导致最终 API 中 ORM 和 Core 提供的结果集行为相同。统一 ORM 和 Core 中的主要模式是 SQLAlchemy 2.0 的主要目标，版本 1.4 旨在在支持此过程的基础架构模式中放置大多数或所有的基础架构模式。Query 返回的`KeyedTuple`对象被 Row 替换中的注释描述了 ORM 对`Row`类的使用。

对于版本 1.4，`Row`类提供了一个额外的子类`LegacyRow`，它被 Core 使用，并提供了`RowProxy`的向后兼容版本，同时对那些将被移动的 API 功能和行为发出弃用警告。ORM `Query`现在直接使用`Row`作为`KeyedTuple`的替代品。

`LegacyRow`类是一个过渡类，其中`__contains__`方法仍然针对键进行测试，而不是值，当操作成功时发出弃用警告。此外，先前的`RowProxy`上的所有其他类似映射的方法也已弃用，包括`LegacyRow.keys()`，`LegacyRow.items()`等。对于从`Row`对象获得类似映射的行为，包括支持这些方法以及面向键的`__contains__`运算符，未来的 API 将首先访问一个特殊属性`Row._mapping`，然后提供完整的映射接口给行，而不是元组接口。

#### 理由：为了更像一个命名元组而不是一个映射

就布尔运算符而言，命名元组和映射之间的差异可以总结为：给定伪代码中的“命名元组”：

```py
row = (id: 5,  name: 'some name')
```

最大的跨不兼容差异是 `__contains__` 的行为：

```py
"id" in row  # True for a mapping, False for a named tuple
"some name" in row  # False for a mapping, True for a named tuple
```

在 1.4 中，当核心结果集返回一个 `LegacyRow` 时，上述 `"id" in row` 比较将继续成功，但是会发出弃用警告。要将“in”运算符用作映射，请使用 `Row._mapping` 属性：

```py
"id" in row._mapping
```

SQLAlchemy 2.0 的结果对象将具有 `.mappings()` 修改器，以便可以直接接收这些映射：

```py
# using sqlalchemy.future package
for row in result.mappings():
    row["id"]
```

#### 代理行为消失了，在现代用法中也是不必要的

将 `Row` 重构为行为类似于元组需要所有数据值一开始就完全可用。这是从 `RowProxy` 的内部行为更改而来，其中结果行处理函数将在访问行元素时调用，而不是在首次提取行时调用。这意味着例如从 SQLite 检索 datetime 值时，`RowProxy` 对象中的行数据先前看起来是这样的：

```py
row_proxy = (1, "2019-12-31 19:56:58.272106")
```

然后，在通过 `__getitem__` 访问时，`datetime.strptime()` 函数将在使用时动态将上述字符串日期转换为 `datetime` 对象。有了新的架构，`datetime()` 对象在返回元组时就已经存在，`datetime.strptime()` 函数只需在一开始调用一次：

```py
row = (1, datetime.datetime(2019, 12, 31, 19, 56, 58, 272106))
```

SQLAlchemy 中的 `RowProxy` 和 `Row` 对象是 SQLAlchemy 大部分 C 扩展代码的位置。该代码已经进行了大量重构，以高效的方式提供新行为，并且整体性能已经得到了提高，因为 `Row` 的设计现在相对简单。

先前行为的基本原理假设一个使用模型，其中一个结果行可能具有几十或几百列，其中大多数列不会被访问，并且其中大多数列需要一些结果值处理函数。通过仅在需要时调用处理函数，目标是不需要大量的结果处理函数，从而增加性能。

有许多原因说明上述假设不成立：

1.  大多数调用的行处理函数是将字节串解码为 Python Unicode 字符串，这是在 Python 2 下开始使用 Python Unicode 并在 Python 3 出现之前的情况。一旦引入了 Python 3，在几年内，所有 Python DBAPI 都承担了支持直接传递 Python Unicode 对象的正确角色，在 Python 2 和 Python 3 下，前者是一个选项，后者是唯一的前进方式。最终，在大多数情况下，它也成为 Python 2 的默认选项。SQLAlchemy 的 Python 2 支持仍然允许为一些 DBAPI（如 cx_Oracle）执行显式的字符串到 Unicode 转换，但现在是在 DBAPI 级别执行，而不是作为标准的 SQLAlchemy 结果行处理函数。

1.  上述字符串转换在使用时通过 C 扩展被设计得非常高效，以至于即使在 1.4 版本中，SQLAlchemy 的字节到 Unicode 编解码器挂接到了 cx_Oracle 中，据观察，它比 cx_Oracle 自己的挂接更高效；这意味着在任何情况下，将所有字符串转换为 Unicode 字符串的开销不再像最初那样显著。

1.  大多数情况下不使用行处理函数；例外情况包括 SQLite 的日期时间支持，某些后端的 JSON 支持，一些数值处理程序，如字符串转换为`Decimal`。在`Decimal`的情况下，Python 3 也标准化了高性能的`cdecimal`实现，而在 Python 2 中并非如此，Python 2 仍然使用性能较低的纯 Python 版本。

1.  在实际用例中，很少有需要获取完整行而只需要少数列的情况。在 SQLAlchemy 的早期，来自其他语言的数据库代码形式“row = fetch(‘SELECT * FROM table’)”很常见；然而，使用 SQLAlchemy 的表达式语言，实际观察到的代码通常使用所需的特定列。

另请参见

Query 返回的“KeyedTuple”对象被 Row 替换

ORM Session.execute() uses “future” style Result sets in all cases

[#4710](https://www.sqlalchemy.org/trac/ticket/4710)  ### SELECT 对象和派生的 FROM 子句允许重复列和列标签

此更改允许`select()`构造现在允许重复的列标签以及重复的列对象本身，以便结果元组以与选择列相同的方式组织和排序。ORM `Query`已经按照这种方式工作，因此这个更改允许两者之间更大的交叉兼容性，这是 2.0 过渡的一个关键目标：

```py
>>> from sqlalchemy import column, select
>>> c1, c2, c3, c4 = column("c1"), column("c2"), column("c3"), column("c4")
>>> stmt = select(c1, c2, c3.label("c2"), c2, c4)
>>> print(stmt)
SELECT  c1,  c2,  c3  AS  c2,  c2,  c4 
```

为了支持这一变化，`SelectBase`使用的`ColumnCollection`以及派生的 FROM 子句（如子查询）也支持重复列；这包括新的`SelectBase.selected_columns`属性，已弃用的`SelectBase.c`属性，以及在诸如`Subquery`和`Alias`等构造中看到的`FromClause.c`属性：

```py
>>> list(stmt.selected_columns)
[
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcca20; c1>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcc9e8; c2>,
 <sqlalchemy.sql.elements.Label object at 0x7fa540b3e2e8>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcc9e8; c2>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540897048; c4>
]

>>> print(stmt.subquery().select())
SELECT  anon_1.c1,  anon_1.c2,  anon_1.c2,  anon_1.c2,  anon_1.c4
FROM  (SELECT  c1,  c2,  c3  AS  c2,  c2,  c4)  AS  anon_1 
```

`ColumnCollection`还允许通过整数索引访问，以支持当字符串“键”不明确时：

```py
>>> stmt.selected_columns[2]
<sqlalchemy.sql.elements.Label object at 0x7fa540b3e2e8>
```

为了适应`Table`和`PrimaryKeyConstraint`等对象中对`ColumnCollection`的使用，保留了更适用于这些对象的旧的“去重”行为，这一行为在新的`DedupeColumnCollection`类中得以保留。

这一变化包括移除了熟悉的警告`“表%r 上的列%r 被%r 替换，具有相同的键。考虑为 select()语句使用 use_labels。”`；`Select.apply_labels()`仍然可用，并且仍然被 ORM 用于所有 SELECT 操作，但它不意味着对列对象进行去重，尽管它确实意味着对隐式生成的标签进行去重：

```py
>>> from sqlalchemy import table
>>> user = table("user", column("id"), column("name"))
>>> stmt = select(user.c.id, user.c.name, user.c.id).apply_labels()
>>> print(stmt)
SELECT "user".id AS user_id, "user".name AS user_name, "user".id AS id_1
FROM "user"
```

最后，这一变化使得更容易创建 UNION 和其他`_selectable.CompoundSelect`对象，通过确保 SELECT 语句中的列数和位置与给定的相符，例如：

```py
>>> s1 = select(user, user.c.id)
>>> s2 = select(c1, c2, c3)
>>> from sqlalchemy import union
>>> u = union(s1, s2)
>>> print(u)
SELECT  "user".id,  "user".name,  "user".id
FROM  "user"  UNION  SELECT  c1,  c2,  c3 
```

[#4753](https://www.sqlalchemy.org/trac/ticket/4753)  ### 使用 CAST 或类似方法改进简单列表达式的列标签

一位用户指出，PostgreSQL 数据库在使用诸如 CAST 之类的函数针对命名列时具有方便的行为，即结果列名与内部表达式同名：

```py
test=> SELECT CAST(data AS VARCHAR) FROM foo;

data
------
 5
(1 row)
```

这使得可以对表列应用 CAST 而不会丢失结果行中的列名（上面使用名称``"data"``）。与 MySQL/MariaDB 等大多数其他数据库不同，这些数据库的列名取自完整的 SQL 表达式，不太具有可移植性：

```py
MariaDB [test]> SELECT CAST(data AS CHAR) FROM foo;
+--------------------+
| CAST(data AS CHAR) |
+--------------------+
| 5                  |
+--------------------+
1 row in set (0.003 sec)
```

在 SQLAlchemy Core 表达式中，我们从不处理像上面那样的原始生成名称，因为 SQLAlchemy 对这些表达式应用自动标记，这些表达式直到现在始终是所谓的“匿名”表达式：

```py
>>> print(select(cast(foo.c.data, String)))
SELECT  CAST(foo.data  AS  VARCHAR)  AS  anon_1  #  old  behavior
FROM  foo 
```

这些匿名表达式是必要的，因为 SQLAlchemy 的`ResultProxy`大量使用结果列名称来匹配数据类型，例如`String`数据类型曾经具有结果行处理行为，以正确匹配列，因此最重要的是这些名称必须易于以与数据库无关的方式确定，并且在所有情况下都是唯一的。在 SQLAlchemy 1.0 中作为[#918](https://www.sqlalchemy.org/trac/ticket/918)的一部分，对于大多数 Core SELECT 构造，不再需要依赖结果行中的命名列（特别是 PEP-249 游标的`cursor.description`元素）；在 1.4 版本中，整个系统对于具有重复列或标签名称的 SELECT 语句变得更加舒适，例如在 SELECT 对象和派生 FROM 子句允许重复列和列标签中。因此，我们现在模拟 PostgreSQL 对于对单个列进行简单修改的合理行为，最显著的是使用 CAST：

```py
>>> print(select(cast(foo.c.data, String)))
SELECT  CAST(foo.data  AS  VARCHAR)  AS  data
FROM  foo 
```

对于没有名称的表达式进行 CAST，先前的逻辑用于生成通常的“匿名”标签：

```py
>>> print(select(cast("hi there," + foo.c.data, String)))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  anon_1
FROM  foo 
```

对于针对`Label`的`cast()`，尽管必须省略标签表达式，因为这些表达式不会在 CAST 内部呈现，但仍将使用给定的名称：

```py
>>> print(select(cast(("hi there," + foo.c.data).label("hello_data"), String)))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  hello_data
FROM  foo 
```

当然，正如往常一样，`Label` 可以应用于外部表达式，直接应用“AS <name>”标签：

```py
>>> print(select(cast(("hi there," + foo.c.data), String).label("hello_data")))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  hello_data
FROM  foo 
```

[#4449](https://www.sqlalchemy.org/trac/ticket/4449)  ### 用于 Oracle、SQL Server 中 LIMIT/OFFSET 的新“编译后”绑定参数

1.4 系列的一个主要目标是确保所有 Core SQL 构造完全可缓存，这意味着特定的`Compiled`结构将生成相同的 SQL 字符串，而不管与之一起使用的任何 SQL 参数，其中特别包括用于指定 LIMIT 和 OFFSET 值的参数，通常用于分页和“top N”样式的结果。

虽然 SQLAlchemy 多年来一直使用绑定参数来实现 LIMIT/OFFSET 方案，但仍有一些特例，其中不允许使用这些参数，包括 SQL Server 的“TOP N”语句，例如：

```py
SELECT  TOP  5  mytable.id,  mytable.data  FROM  mytable
```

以及在 Oracle 中，如果向`create_engine()`传递了`optimize_limits=True`参数并使用 Oracle URL，那么 FIRST_ROWS()提示（SQLAlchemy 将使用该提示）将不允许它们，但是使用绑定参数与 ROWNUM 比较已被报告为产生较慢的查询计划：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS(5) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id,  mytable.data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  :param_1
)  anon_1  WHERE  ora_rn  >  :param_2
```

为了使所有语句在编译级别无条件可缓存，添加了一种新形式的绑定参数，称为“后编译”参数，它利用了与“扩展 IN 参数”相同的机制。这是一个`bindparam()`，其行为与任何其他绑定参数完全相同，只是参数值将在发送到 DBAPI `cursor.execute()`方法之前被直接呈现到 SQL 字符串中。新参数在 SQL Server 和 Oracle 方言内部使用，以便驱动程序接收到字面呈现的值，但 SQLAlchemy 的其余部分仍然可以将其视为绑定参数。现在，使用`str(statement.compile(dialect=<dialect>))`将上述两个语句字符串化后看起来如下：

```py
SELECT  TOP  [POSTCOMPILE_param_1]  mytable.id,  mytable.data  FROM  mytable
```

和：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS([POSTCOMPILE__ora_frow_1]) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id,  mytable.data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  [POSTCOMPILE_param_1]
)  anon_1  WHERE  ora_rn  >  [POSTCOMPILE_param_2]
```

当使用“扩展 IN”时，也会看到`[POSTCOMPILE_<param>]`格式。

查看 SQL 日志输出时，将看到语句的最终形式：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS(5) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id  AS  id,  mytable.data  AS  data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  8
)  anon_1  WHERE  ora_rn  >  3
```

“后编译参数”功能通过`bindparam.literal_execute`参数作为公共 API 公开，但目前不打算供一般使用。在 SQLAlchemy 中，字面值使用底层数据类型的`TypeEngine.literal_processor()`进行呈现，其范围**极其有限**，仅支持整数和简单字符串值。

[#4808](https://www.sqlalchemy.org/trac/ticket/4808)  ### 连接级事务现在可以基于子事务处于非活动状态

一个`Connection`现在包括了一个行为，即由于内部事务的回滚而使得`Transaction`变为非活动状态，但是`Transaction`直到自身被回滚之前都不会清除。

这本质上是一个新的错误条件，如果内部“子”事务已被回滚，则将禁止语句执行继续在`Connection`上进行。该行为与 ORM `Session`的行为非常相似，如果已开始外部事务，则需要将其回滚以清除无效事务；此行为在“由于刷新期间的先前异常，此会话的事务已被回滚。”（或类似）中有描述。

虽然`Connection`的行为模式比`Session`更宽松，但由于它有助于识别子事务已回滚 DBAPI 事务，但外部代码不知道这一点并尝试继续进行，实际上在新事务上运行操作，因此进行了此更改。在将会话加入外部事务（例如用于测试套件）中描述的“测试工具”模式是发生这种情况的常见地方。

Core 和 ORM 的“子事务”功能本身已被弃用，并将不再出现在 2.0 版本中。因此，这种新的错误条件本身是临时的，一旦子事务被移除，它将不再适用。

为了与不包括子事务的 2.0 风格行为一起工作，请在`create_engine()`上使用`create_engine.future`参数。

错误消息在错误页面中描述为此连接处于非活动事务状态。请在继续之前完全回滚()。  ### 枚举和布尔数据类型不再默认为“创建约束”

`Enum.create_constraint`和`Boolean.create_constraint`参数现在默认为 False，表示当创建这两种数据类型的所谓“非本地”版本时，默认情况下不会生成 CHECK 约束。这些 CHECK 约束会带来应该选择的模式管理维护复杂性，而不是默认打开。

要确保为这些类型发出 CREATE CONSTRAINT，请将这些标志设置为`True`：

```py
class Spam(Base):
    __tablename__ = "spam"
    id = Column(Integer, primary_key=True)
    boolean = Column(Boolean(create_constraint=True))
    enum = Column(Enum("a", "b", "c", create_constraint=True))
```

[#5367](https://www.sqlalchemy.org/trac/ticket/5367)  ### SELECT 语句不再被隐式视为 FROM 子句

这个变化是 SQLAlchemy 多年来的一个较大的概念性变化之一，但希望最终用户的影响相对较小，因为这个变化更接近于 MySQL 和 PostgreSQL 等数据库所要求的情况。

最直观的显著影响是，一个 `select()` 不能再直接嵌套在另一个 `select()` 中，而不是先将内部的 `select()` 明确地转换为子查询。这在历史上是通过使用 `SelectBase.alias()` 方法来实现的，但更明确地使用新方法 `SelectBase.subquery()` 更适合；这两种方法做的事情是一样的。现在返回的对象是 `Subquery`，它与 `Alias` 对象非常相似，并共享一个公共基类 `AliasedReturnsRows`。

换句话说，现在会引发：

```py
stmt1 = select(user.c.id, user.c.name)
stmt2 = select(addresses, stmt1).select_from(addresses.join(stmt1))
```

提出：

```py
sqlalchemy.exc.ArgumentError: Column expression or FROM clause expected,
got <...Select object ...>. To create a FROM clause from a <class
'sqlalchemy.sql.selectable.Select'> object, use the .subquery() method.
```

正确的调用形式应为（同时注意不再需要对 select() 使用括号）：

```py
sq1 = select(user.c.id, user.c.name).subquery()
stmt2 = select(addresses, sq1).select_from(addresses.join(sq1))
```

注意到上面的 `SelectBase.subquery()` 方法本质上等同于使用 `SelectBase.alias()` 方法。

这种变更的原理如下：

+   为了支持将`Select`与`Query`统一起来，`Select`对象需要具有实际添加 JOIN 条件到现有 FROM 子句的`Select.join()`和`Select.outerjoin()`方法，正如用户一直期望它们做的那样。先前的行为，需要与`FromClause`一致，即生成一个无名称的子查询，然后 JOIN 到它，这是一个完全没有用的功能，只会让那些不幸尝试的用户感到困惑。这一变化在 select().join() and outerjoin() add JOIN criteria to the current query, rather than creating a subquery 中讨论。

+   在另一个 SELECT 的 FROM 子句中包含一个 SELECT 的行为，而不先创建别名或子查询，会导致创建一个无名称的子查询。虽然标准 SQL 支持这种语法，但实际上大多数数据库都会拒绝。例如，MySQL 和 PostgreSQL 都明确拒绝使用无名称子查询：

    ```py
    #  MySQL  /  MariaDB:

    MariaDB  [(none)]>  select  *  from  (select  1);
    ERROR  1248  (42000):  Every  derived  table  must  have  its  own  alias

    #  PostgreSQL:

    test=>  select  *  from  (select  1);
    ERROR:  subquery  in  FROM  must  have  an  alias
    LINE  1:  select  *  from  (select  1);
      ^
    HINT:  For  example,  FROM  (SELECT  ...)  [AS]  foo.
    ```

    像 SQLite 这样的数据库接受它们，但通常情况下，从这样的子查询中产生的名称太模糊，无法使用：

    ```py
    sqlite>  CREATE  TABLE  a(id  integer);
    sqlite>  CREATE  TABLE  b(id  integer);
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  ON  a.id=id;
    Error:  ambiguous  column  name:  id
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  ON  a.id=b.id;
    Error:  no  such  column:  b.id

    #  use  a  name
    sqlite>  SELECT  *  FROM  a  JOIN  (SELECT  *  FROM  b)  AS  anon_1  ON  a.id=anon_1.id;
    ```

由于`SelectBase`对象不再是`FromClause`对象，因此像`.c`属性以及`.select()`等方法现在已被弃用，因为它们暗示隐式生成子查询。`.join()`和`.outerjoin()`方法现在被重新用于在现有查询中追加 JOIN 条件，类似于`Query.join()`的方式，这正是用户一直期望这些方法做的事情。

在`.c`属性的位置，新增了一个属性`SelectBase.selected_columns`。这个属性解析为一个列集合，大多数人希望`.c`能够做到的（但实际上不能），即引用 SELECT 语句中列子句中的列。一个常见的初学者错误是以下代码：

```py
stmt = select(users)
stmt = stmt.where(stmt.c.name == "foo")
```

上述代码看起来直观，似乎会生成“SELECT * FROM users WHERE name=’foo’”，然而，经验丰富的 SQLAlchemy 用户会意识到，实际上它生成了一个无用的子查询，类似于“SELECT * FROM (SELECT * FROM users) WHERE name=’foo’”。

新的`SelectBase.selected_columns`属性确实**适用于**上述用例，因为在上述情况下，它直接链接到`users.c`集合中存在的列：

```py
stmt = select(users)
stmt = stmt.where(stmt.selected_columns.name == "foo")
```

[#4617](https://www.sqlalchemy.org/trac/ticket/4617)

### select().join()和 outerjoin()将 JOIN 条件添加到当前查询，而不是创建子查询

为了实现统一`Query`和`Select`的目标，特别是对于 2.0 风格使用`Select`，至关重要的是有一个工作的`Select.join()`方法，其行为类似于`Query.join()`方法，向现有 SELECT 的 FROM 子句添加额外条目，然后返回新的`Select`对象以进行进一步修改，而不是将对象包装在一个无名子查询中，并从该子查询返回 JOIN，这种行为对用户来说一直是几乎无用和完全误导的。

为了使这成为可能，首先实现了 A SELECT statement is no longer implicitly considered to be a FROM clause，这将`Select`从必须是`FromClause`的要求中分离出来；这消除了`Select.join()`需要返回一个`Join`对象而不是包含新 JOIN 的 FROM 子句的新版本`Select`对象的要求。

从那时起，由于 `Select.join()` 和 `Select.outerjoin()` 有着现有的行为，最初的计划是这些方法将被弃用，而新的“有用”版本的方法将在一个备用的“未来” `Select` 对象上作为一个单独的导入可用。

然而，经过一段时间的与这个特定的代码库一起工作后，决定有两种不同类型的 `Select` 对象在周围漂浮，每个对象的行为都相似，除了某些细微的方法行为上的差异，这比起简单地对这两种方法的行为进行硬性更改更加误导和不便，因为 `Select.join()` 和 `Select.outerjoin()` 的现有行为基本上是不会被使用的，只会造成混淆。

所以决定在这一个领域做出一个**硬性行为更改**，而不是等待另一年并在此期间产生更加尴尬的 API。SQLAlchemy 开发人员不轻易做出像这样完全破坏性的更改，然而这是一个非常特殊的情况，以前的这些方法的实现几乎肯定不会被使用；如 A SELECT statement is no longer implicitly considered to be a FROM clause 中所述，主要数据库如 MySQL 和 PostgreSQL 在任何情况下都不允许未命名的子查询，从语法上来说，从未命名的子查询中进行 JOIN 几乎是不可能有用的，因为在其中明确引用列非常困难。

有了新的实现，`Select.join()` 和 `Select.outerjoin()` 现在的行为与 `Query.join()` 非常相似，通过将 JOIN 条件与左实体进行匹配，将 JOIN 条件添加到现有语句中：

```py
stmt = select(user_table).join(
    addresses_table, user_table.c.id == addresses_table.c.user_id
)
```

产生的结果是：

```py
SELECT  user.id,  user.name  FROM  user  JOIN  address  ON  user.id=address.user_id
```

就像`Join`一样，如果可行的话，ON 子句会被自动确定：

```py
stmt = select(user_table).join(addresses_table)
```

当在语句中使用 ORM 实体时，这基本上是使用 2.0 风格 调用来构建 ORM 查询的方式。ORM 实体将在语句内部分配一个“插件”，这样当语句被编译成 SQL 字符串时，与 ORM 相关的编译规则将会发生作用。更直接地说，`Select.join()` 方法可以适应 ORM 关系，而不会破坏核心和 ORM 内部之间的严格分隔：

```py
stmt = select(User).join(User.addresses)
```

另外，还添加了另一个新方法 `Select.join_from()`，它允许一次更轻松地指定连接的左侧和右侧：

```py
stmt = select(Address.email_address, User.name).join_from(User, Address)
```

生成：

```py
SELECT  address.email_address,  user.name  FROM  user  JOIN  address  ON  user.id  ==  address.user_id
```

### URL 对象现在是不可变的

`URL` 对象已经被规范化，现在它呈现为一个具有固定数量字段的`namedtuple`，这些字段是不可变的。此外，由 `URL.query` 属性表示的字典也是一个不可变映射。对 `URL` 对象的修改不是正式支持或记录的用例，这导致了一些开放性用例，使得很难拦截不正确的用法，最常见的是修改 `URL.query` 字典以包含非字符串元素。这也导致了允许在基本数据对象中进行可变性的所有常见问题，即不期望 URL 改变的代码中泄露了不需要的变异。最后，`namedtuple` 的设计受到了 Python 的 `urllib.parse.urlparse()` 的启发，它将解析后的对象返回为命名元组。

直接更改 API 的决定基于对废弃路径的不可行性的权衡（这将涉及将`URL.query`字典更改为一个特殊字典，当调用任何标准库变异方法时会发出废弃警告，此外，当字典将保存任何元素列表时，列表也将在变异时发出废弃警告），而不是项目已经在第一次更改`URL`对象的不太可能使用情况，以及像[#5341](https://www.sqlalchemy.org/trac/ticket/5341)这样的小变化在任何情况下都会造成向后不兼容性。对于更改`URL`对象的主要情况是在`CreateEnginePlugin`扩展点内解析插件参数，这本身是一个相当新的添加，根据 Github 代码搜索的结果，有两个仓库在使用，但实际上都没有更改 URL 对象。

`URL`对象现在提供了一个丰富的接口来检查和生成新的`URL`对象。创建`URL`对象的现有机制，即`make_url()`函数，保持不变：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url("postgresql+psycopg2://user:pass@host/dbname")
```

对于以编程方式构建的代码，如果参数作为关键字参数传递而不是精确的 7 元组，则可能一直在使用`URL`构造函数或`__init__`方法的代码将收到废弃警告。现在可以通过`URL.create()`方法使用关键字样式的构造函数：

```py
>>> from sqlalchemy.engine import URL
>>> url = URL.create("postgresql", "user", "pass", host="host", database="dbname")
>>> str(url)
'postgresql://user:pass@host/dbname'
```

使用`URL.set()`方法通常可以修改字段，该方法返回一个应用更改的新`URL`对象：

```py
>>> mysql_url = url.set(drivername="mysql+pymysql")
>>> str(mysql_url)
'mysql+pymysql://user:pass@host/dbname'
```

要更改`URL.query`字典的内容，可以使用`URL.update_query_dict()`等方法：

```py
>>> url.update_query_dict({"sslcert": "/path/to/crt"})
postgresql://user:***@host/dbname?sslcert=%2Fpath%2Fto%2Fcrt
```

要升级直接更改这些字段的代码，一个**向后和向前兼容的方法**是使用鸭子类型，如以下样式：

```py
def set_url_drivername(some_url, some_drivername):
    # check for 1.4
    if hasattr(some_url, "set"):
        return some_url.set(drivername=some_drivername)
    else:
        # SQLAlchemy 1.3 or earlier, mutate in place
        some_url.drivername = some_drivername
        return some_url

def set_ssl_cert(some_url, ssl_cert):
    # check for 1.4
    if hasattr(some_url, "update_query_dict"):
        return some_url.update_query_dict({"sslcert": ssl_cert})
    else:
        # SQLAlchemy 1.3 or earlier, mutate in place
        some_url.query["sslcert"] = ssl_cert
        return some_url
```

查询字符串保留其现有格式，作为字符串到字符串的字典，使用字符串序列表示多个参数。例如：

```py
>>> from sqlalchemy.engine import make_url
>>> url = make_url(
...     "postgresql://user:pass@host/dbname?alt_host=host1&alt_host=host2&sslcert=%2Fpath%2Fto%2Fcrt"
... )
>>> url.query
immutabledict({'alt_host': ('host1', 'host2'), 'sslcert': '/path/to/crt'})
```

要处理`URL.query`属性的内容，使所有值规范化为序列，请使用`URL.normalized_query`属性：

```py
>>> url.normalized_query
immutabledict({'alt_host': ('host1', 'host2'), 'sslcert': ('/path/to/crt',)})
```

查询字符串可以通过诸如`URL.update_query_dict()`，`URL.update_query_pairs()`，`URL.update_query_string()`等方法进行追加：

```py
>>> url.update_query_dict({"alt_host": "host3"}, append=True)
postgresql://user:***@host/dbname?alt_host=host1&alt_host=host2&alt_host=host3&sslcert=%2Fpath%2Fto%2Fcrt
```

另请参阅

`URL`

#### 对 CreateEnginePlugin 的更改

此更改还会影响`CreateEnginePlugin`，因为自定义插件的文档指出应使用`dict.pop()`方法从 URL 对象中删除已使用的参数。现在应该使用`CreateEnginePlugin.update_url()`方法来实现。向后兼容的方法如下：

```py
from sqlalchemy.engine import CreateEnginePlugin

class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        # check for 1.4 style
        if hasattr(CreateEnginePlugin, "update_url"):
            self.my_argument_one = url.query["my_argument_one"]
            self.my_argument_two = url.query["my_argument_two"]
        else:
            # legacy
            self.my_argument_one = url.query.pop("my_argument_one")
            self.my_argument_two = url.query.pop("my_argument_two")

        self.my_argument_three = kwargs.pop("my_argument_three", None)

    def update_url(self, url):
        # this method runs in 1.4 only and should be used to consume
        # plugin-specific arguments
        return url.difference_update_query(["my_argument_one", "my_argument_two"])
```

查看`CreateEnginePlugin`处的文档字符串，了解此类的完整使用详情。

[#5526](https://www.sqlalchemy.org/trac/ticket/5526)

#### 对 CreateEnginePlugin 的更改

此更改还会影响`CreateEnginePlugin`，因为自定义插件的文档指出应使用`dict.pop()`方法从 URL 对象中删除已使用的参数。现在应该使用`CreateEnginePlugin.update_url()`方法来实现。向后兼容的方法如下：

```py
from sqlalchemy.engine import CreateEnginePlugin

class MyPlugin(CreateEnginePlugin):
    def __init__(self, url, kwargs):
        # check for 1.4 style
        if hasattr(CreateEnginePlugin, "update_url"):
            self.my_argument_one = url.query["my_argument_one"]
            self.my_argument_two = url.query["my_argument_two"]
        else:
            # legacy
            self.my_argument_one = url.query.pop("my_argument_one")
            self.my_argument_two = url.query.pop("my_argument_two")

        self.my_argument_three = kwargs.pop("my_argument_three", None)

    def update_url(self, url):
        # this method runs in 1.4 only and should be used to consume
        # plugin-specific arguments
        return url.difference_update_query(["my_argument_one", "my_argument_two"])
```

查看`CreateEnginePlugin`处的文档字符串，了解此类的完整使用详情。

[#5526](https://www.sqlalchemy.org/trac/ticket/5526)

### select()，case()现在接受位置表达式

正如在本文档的其他地方可能看到的那样，`select()` 构造现在将按位置接受“列子句”参数，而不需要将它们作为列表传递：

```py
# new way, supports 2.0
stmt = select(table.c.col1, table.c.col2, ...)
```

在按位置发送参数时，不允许其他关键字参数。在 SQLAlchemy 2.0 中，上述调用风格将是唯一支持的调用风格。

在 1.4 版本期间，先前的调用风格仍将继续运行，将列或其他表达式的列表作为列表传递：

```py
# old way, still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...])
```

上述传统调用风格还接受了自那时起已从大多数叙述文档中删除的旧关键字参数。这些关键字参数的存在是为什么列子句首先作为列表传递的原因：

```py
# very much the old way, but still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...], whereclause=table.c.col1 == 5)
```

两种风格之间的区别是第一个位置参数是否为列表。不幸的是，仍然可能有一些使用看起来像以下内容的用法，其中“whereclause”的关键字被省略：

```py
# very much the old way, but still works in 1.4
stmt = select([table.c.col1, table.c.col2, ...], table.c.col1 == 5)
```

作为这一更改的一部分，`Select` 构造还获得了 2.0 风格的“未来” API，其中包括更新的 `Select.join()` 方法以及诸如 `Select.filter_by()` 和 `Select.join_from()` 等方法。

在相关更改中，`case()` 构造也已经修改为按位置接受其 WHEN 子句列表，对于旧的调用风格也有类似的弃用跟踪：

```py
stmt = select(users_table).where(
    case(
        (users_table.c.name == "wendy", "W"),
        (users_table.c.name == "jack", "J"),
        else_="E",
    )
)
```

对于接受`*args`与值列表的 SQLAlchemy 构造的约定，就像`ColumnOperators.in_()`这样的构造中的后者情况一样，**位置参数用于结构规范，列表用于数据规范**。

另请参阅

select() 不再接受不同的构造参数，列是按位置传递的

在“传统”模式下创建的 select() 构造；关键字参数等

[#5284](https://www.sqlalchemy.org/trac/ticket/5284)

### 所有 IN 表达式都会即时为列表中的每个值呈现参数（例如，扩展参数）

首次引入的“扩展 IN”功能已经足够成熟，以至于它显然优于以前的 IN 表达式渲染方法。随着该方法被改进以处理空值列表，它现在是 Core / ORM 渲染 IN 参数列表的唯一方法。

之前在 SQLAlchemy 中一直存在的方法是，在`ColumnOperators.in_()`方法中传递值列表时，列表会在语句构建时扩展为一系列单独的`BindParameter`对象。这种方法的局限性在于，无法根据参数字典在语句执行时变化参数列表，这意味着字符串 SQL 语句不能独立于其参数缓存，并且参数字典不能完全用于包含 IN 表达式的语句。

为了支持 Baked Queries 中描述的“烘焙查询”功能，需要一个可缓存版本的 IN，这就引入了“扩展 IN”功能。与现有行为相比，在语句构造时将参数列表展开为单独的`BindParameter`对象，该功能使用一个存储所有值列表的单个`BindParameter`；当由`Engine`执行语句时，它会根据传递给`Connection.execute()`调用的参数，在执行过程中动态将其“展开”为基于当前参数集合的单个绑定参数位置，并且可能已从先前执行中检索到的现有 SQL 字符串会使用正则表达式修改以适应当前参数集合。这允许相同的`Compiled`对象，存储呈现的字符串语句，多次针对修改 IN 表达式传递的不同参数集合调用，同时仍保持将单个标量参数传递给 DBAPI 的行为。虽然一些 DBAPI 直接支持此功能，但通常不可用；“扩展 IN”功能现在为所有后端一致支持此行为。

1.4 版的一个主要重点是在 Core 和 ORM 中实现真正的语句缓存，而不需要“烘焙”系统的尴尬，由于“扩展 IN”功能代表了构建表达式的更简单方法，因此现在在将值列表传递给 IN 表达式时会自动调用它：

```py
stmt = select(A.id, A.data).where(A.id.in_([1, 2, 3]))
```

预执行字符串表示为：

```py
>>> print(stmt)
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  ([POSTCOMPILE_id_1]) 
```

要直接呈现值，请像以前一样使用`literal_binds`：

```py
>>> print(stmt.compile(compile_kwargs={"literal_binds": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (1,  2,  3) 
```

添加了一个新标志“render_postcompile”作为辅助工具，允许当前绑定值被呈现为传递给数据库的形式：

```py
>>> print(stmt.compile(compile_kwargs={"render_postcompile": True}))
SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (:id_1_1,  :id_1_2,  :id_1_3) 
```

引擎日志输出显示最终呈现的语句如下：

```py
INFO  sqlalchemy.engine.base.Engine  SELECT  a.id,  a.data
FROM  a
WHERE  a.id  IN  (?,  ?,  ?)
INFO  sqlalchemy.engine.base.Engine  (1,  2,  3)
```

作为这一变化的一部分，“空 IN”表达式的行为，其中列表参数为空，现在标准化为使用针对所谓“空集”的 IN 运算符。由于没有标准的空集 SQL 语法，使用返回零行的 SELECT，针对每个后端进行特定方式的定制，以便数据库将其视为空集；此功能首次在版本 1.3 中引入，并在 扩展 IN 功能现在支持空列表 中描述。在版本 1.2 中引入的 `create_engine.empty_in_strategy` 参数，作为迁移以前 IN 系统处理方式的手段，现已被弃用，此标志不再起作用；如 IN / NOT IN 运算符的空集合行为现在可配置；默认表达式简化 中所述，此标志允许方言在原始系统比较列与自身，这证明是一个巨大的性能问题，以及新系统比较“1 != 1” 以产生“false”表达式之间切换。1.3 引入的行为现在在所有情况下都更为正确，因为仍然使用 IN 运算符，并且不具有原始系统的性能问题。

此外，“扩展”参数系统已经泛化，以便还服务于其他特定方言的用例，其中参数无法被 DBAPI 或后端数据库容纳；有关详细信息，请参阅 Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数。

另请参阅

Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数

扩展 IN 功能现在支持空列表

`BindParameter`

[#4645](https://www.sqlalchemy.org/trac/ticket/4645)

### 内置 FROM 代码检查将警告任何 SELECT 语句中潜在的笛卡尔积

由于核心表达语言以及 ORM 建立在“隐式 FROMs”模型上，如果查询的任何部分引用了特定的 FROM 子句，那么该子句将自动添加，一个常见问题是 SELECT 语句的情况，无论是顶层语句还是嵌入的子查询，包含了未与查询中的其他 FROM 元素连接的 FROM 元素，导致结果集中出现所谓的“笛卡尔积”，即每个未连接的 FROM 元素之间的所有可能组合的行。在关系数据库中，这几乎总是一个不良结果，因为它会产生一个充满重复、不相关数据的巨大结果集。

SQLAlchemy，尽管具有许多出色的功能，但特别容易发生这种问题，因为 SELECT 语句将自动从其他子句中看到的任何表添加到其 FROM 子句中。典型情况如下，其中两个表被 JOIN 在一起，然而 WHERE 子句中可能无意中与这两个表不匹配的额外条目将创建一个额外的 FROM 条目：

```py
address_alias = aliased(Address)

q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
)
```

上述查询从`User`和`address_alias`的 JOIN 中选择，后者是`Address`实体的别名。然而，`Address`实体直接在 WHERE 子句中使用，因此上述查询将导致以下 SQL：

```py
SELECT
  users.id  AS  users_id,  users.name  AS  users_name,
  users.fullname  AS  users_fullname,
  users.nickname  AS  users_nickname
FROM  addresses,  users  JOIN  addresses  AS  addresses_1  ON  users.id  =  addresses_1.user_id
WHERE  addresses.email_address  =  :email_address_1
```

在上述 SQL 中，我们可以看到 SQLAlchemy 开发人员所称的“可怕的逗号”，因为我们在 FROM 子句中看到“FROM addresses, users JOIN addresses”，这是笛卡尔积的典型迹象；查询正在使用 JOIN 来将 FROM 子句连接在一起，但是因为其中一个没有连接，它使用了逗号。上述查询将返回一个完整的行集，将“user”和“addresses”表在“id / user_id”列上连接在一起，然后将所有这些行直接应用于“addresses”表中的每一行的笛卡尔积。也就是说，如果有十个用户行和 100 个地址行，上述查询将返回其预期的结果行，可能为 100，因为所有地址行都将被选择，再乘以 100，因此总结果大小将为 10000 行。

“table1, table2 JOIN table3”模式在 SQLAlchemy ORM 中也经常出现，这要归因于对 ORM 功能的微妙错误应用，特别是与连接式急加载或连接式表继承相关的功能，以及由于这些系统中的 SQLAlchemy ORM 错误而导致的结果。类似的问题也适用于使用“隐式连接”的 SELECT 语句，其中不使用 JOIN 关键字，而是通过 WHERE 子句将每个 FROM 元素与另一个元素链接起来。

多年来，Wiki 上有一个配方应用图算法于查询执行时间的 `select()` 构造，并检查查询的结构以寻找这些未连接的 FROM 子句，通过 WHERE 子句和所有 JOIN 子句解析来确定 FROM 元素如何连接在一起，并确保所有 FROM 元素在单个图中连接在一起。现在，这个配方已经被改编为 `SQLCompiler` 的一部分，它现在可以选择性地为语句发出警告，如果检测到这种条件。通过使用 `create_engine.enable_from_linting` 标志启用警告，默认情况下启用警告。检查程序的计算开销非常低，而且它只在语句编译期间发生，这意味着对于缓存的 SQL 语句，它只发生一次。

使用此功能，我们上面的 ORM 查询将发出警告：

```py
>>> q.all()
SAWarning: SELECT statement has a cartesian product between FROM
element(s) "addresses_1", "users" and FROM element "addresses".
Apply join condition(s) between each element to resolve.
```

检查程序功能不仅适用于通过 JOIN 子句连接在一起的表，还适用于通过 WHERE 子句上述，我们可以添加一个 WHERE 子句来将新的 `Address` 实体与以前的 `address_alias` 实体链接起来，这将消除警告：

```py
q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
    .filter(Address.id == address_alias.id)
)  # resolve cartesian products,
# will no longer warn
```

笛卡尔积警告将 **任何** 类型的两个 FROM 子句之间的任何链接都视为解析，即使最终结果集仍然是浪费的，因为检查程序旨在仅检测完全意外的 FROM 子句的常见情况。如果 FROM 子句在其他地方明确引用并链接到其他 FROM，就不会发出警告：

```py
q = (
    session.query(User)
    .join(address_alias, User.addresses)
    .filter(Address.email_address == "foo")
    .filter(Address.id > address_alias.id)
)  # will generate a lot of rows,
# but no warning
```

如果明确指定，也可以允许完全的笛卡尔积；例如，如果我们想要 `User` 和 `Address` 的笛卡尔积，我们可以在 `true()` 上进行 JOIN，以便每一行都与其他行匹配；以下查询将返回所有行并且不会产生警告：

```py
from sqlalchemy import true

# intentional cartesian product
q = session.query(User).join(Address, true())  # intentional cartesian product
```

默认情况下，只有当语句由 `Connection` 编译执行时才会生成警告；调用 `ClauseElement.compile()` 方法不会发出警告，除非提供了检查标志：

```py
>>> from sqlalchemy.sql import FROM_LINTING
>>> print(q.statement.compile(linting=FROM_LINTING))
SAWarning: SELECT statement has a cartesian product between FROM element(s) "addresses" and FROM element "users".  Apply join condition(s) between each element to resolve.
SELECT  users.id,  users.name,  users.fullname,  users.nickname
FROM  addresses,  users  JOIN  addresses  AS  addresses_1  ON  users.id  =  addresses_1.user_id
WHERE  addresses.email_address  =  :email_address_1 
```

[#4737](https://www.sqlalchemy.org/trac/ticket/4737)

### 新的 Result 对象

SQLAlchemy 2.0 的一个主要目标是统一 ORM 和 Core 之间如何处理 “结果”。为实现此目标，版本 1.4 引入了自从 SQLAlchemy 开始就存在的 `ResultProxy` 和 `RowProxy` 对象的新版本。

新对象在`Result`和`Row`中有文档记录，并且不仅用于核心结果集，还用于 ORM 中的 2.0 风格结果。

此结果对象与`ResultProxy`完全兼容，并包含许多新功能，现在同样适用于核心和 ORM 结果，包括方法如：

`Result.one()` - 返回确切的单行，或引发异常：

```py
with engine.connect() as conn:
    row = conn.execute(table.select().where(table.c.id == 5)).one()
```

`Result.one_or_none()` - 相同，但对于没有行的情况也返回 None

`Result.all()` - 返回所有行

`Result.partitions()` - 按块获取行：

```py
with engine.connect() as conn:
    result = conn.execute(
        table.select().order_by(table.c.id),
        execution_options={"stream_results": True},
    )
    for chunk in result.partitions(500):
        # process up to 500 records
        ...
```

`Result.columns()` - 允许对行进行切片和重新组织：

```py
with engine.connect() as conn:
    # requests x, y, z
    result = conn.execute(select(table.c.x, table.c.y, table.c.z))

    # iterate rows as y, x
    for y, x in result.columns("y", "x"):
        print("Y: %s X: %s" % (y, x))
```

`Result.scalars()` - 返回标量对象的列表，默认情况下从第一列开始，但也可以选择：

```py
result = session.execute(select(User).order_by(User.id))
for user_obj in result.scalars():
    ...
```

`Result.mappings()` - 返回字典而不是命名元组行：

```py
with engine.connect() as conn:
    result = conn.execute(select(table.c.x, table.c.y, table.c.z))

    for map_ in result.mappings():
        print("Y: %(y)s X: %(x)s" % map_)
```

在使用核心时，由`Connection.execute()`返回的对象是`CursorResult`的一个实例，它继续具有与`ResultProxy`相同的 API 功能，关于插入的主键、默认值、行数等。对于 ORM，将返回一个执行将核心行转换为 ORM 行的`Result`子类，然后允许进行所有相同的操作。

另请参见

ORM 查询与核心选择统一 - 在 2.0 迁移文档中

[#5087](https://www.sqlalchemy.org/trac/ticket/5087)

[#4395](https://www.sqlalchemy.org/trac/ticket/4395)

[#4959](https://www.sqlalchemy.org/trac/ticket/4959)

### RowProxy 不再是“代理”；现在称为 Row，并且行为类似于增强的命名元组

`RowProxy`类，表示 Core 结果集中的单个数据库结果行，现在称为`Row`，不再是“代理”对象；这意味着当返回`Row`对象时，行是一个简单的元组，其中包含数据的最终形式，已经通过与数据类型相关联的结果行处理函数处理过（示例包括将数据库中的日期字符串转换为`datetime`对象、将 JSON 字符串转换为 Python 的`json.loads()`结果等）。

该操作的直接理由是，使行更像 Python 中的命名元组，而不是映射，其中元组中的值是元组上的`__contains__`运算符的主题，而不是键。使用`Row`作为命名元组，然后适用于替换 ORM 的`KeyedTuple`对象，从而导致最终的 API，其中 ORM 和 Core 提供的结果集行为相同。统一 ORM 和 Core 中的主要模式是 SQLAlchemy 2.0 的主要目标，而 1.4 版本旨在具有大部分或全部基础架构模式，以支持此过程。 查询返回的`KeyedTuple`对象由 Row 替换中的说明描述了 ORM 对`Row`类的使用。

对于 1.4 版本，`Row`类提供了一个额外的子类`LegacyRow`，它由 Core 使用，并提供了`RowProxy`的向后兼容版本，同时对那些将被移动的 API 功能和行为发出弃用警告。ORM `Query`现在直接使用`Row`作为`KeyedTuple`的替代品。

`LegacyRow`类是一个过渡类，其中`__contains__`方法仍然针对键而不是值进行测试，当操作成功时发出弃用警告。此外，以前的`RowProxy`上的所有其他类似映射的方法也已弃用，包括`LegacyRow.keys()`、`LegacyRow.items()`等。为了从`Row`对象获取类似映射的行为，包括对这些方法的支持以及面向键的`__contains__`操作，未来的 API 将首先访问特殊属性`Row._mapping`，然后提供完整的映射接口以访问行，而不是元组接口。

#### 理念：表现得更像一个命名元组而不是一个映射

就布尔运算符而言，命名元组和映射之间的区别可以总结如下。假设伪代码中有一个“命名元组”：

```py
row = (id: 5,  name: 'some name')
```

最大的不兼容差异是`__contains__`的行为：

```py
"id" in row  # True for a mapping, False for a named tuple
"some name" in row  # False for a mapping, True for a named tuple
```

在 1.4 版本中，当核心结果集返回一个`LegacyRow`时，上述`"id" in row`比较将继续成功，但会发出弃用警告。要将“in”运算符用作映射，请使用`Row._mapping`属性：

```py
"id" in row._mapping
```

SQLAlchemy 2.0 的结果对象将具有`.mappings()`修饰符，以便可以直接接收这些映射：

```py
# using sqlalchemy.future package
for row in result.mappings():
    row["id"]
```

#### 代理行为消失，也在现代用法中是不必要的

将`Row`重构为像元组一样的行需要所有数据值一开始就完全可用。这是从`RowProxy`的内部行为变化，其中结果行处理函数将在访问行的元素时调用，而不是在首次获取行时调用。这意味着例如从 SQLite 检索日期时间值时，以前在`RowProxy`对象中的行数据看起来像：

```py
row_proxy = (1, "2019-12-31 19:56:58.272106")
```

然后通过`__getitem__`访问时，将实时使用`datetime.strptime()`函数将上述字符串日期转换为`datetime`对象。通过新的架构，当返回元组时，`datetime()`对象已经存在于其中，`datetime.strptime()`函数只被调用了一次：

```py
row = (1, datetime.datetime(2019, 12, 31, 19, 56, 58, 272106))
```

SQLAlchemy 中的`RowProxy`和`Row`对象是大部分 SQLAlchemy C 扩展代码的位置。这段代码已经进行了高度重构，以有效地提供新的行为，并且由于`Row`的设计现在相当简单，因此整体性能得到了改善。

之前行为背后的理念假定了一个使用模型，其中一个结果行可能有几十个或几百个列，其中大多数列不会被访问，并且其中大多数列需要一些结果值处理函数。通过仅在需要时调用处理函数，目标是不需要大量的结果处理函数，从而提高性能。

有许多原因导致上述假设不成立：

1.  调用的绝大多数行处理函数是将字节串解码为 Python Unicode 字符串在 Python 2 下。这是因为 Python Unicode 开始被使用，而 Python 3 之前存在。一旦引入了 Python 3，在几年内，所有 Python DBAPI 都承担了支持直接传递 Python Unicode 对象的正确角色，在 Python 2 和 Python 3 下，前者是一个选项，在后者是唯一的前进方式。最终，在大多数情况下，它也成为了 Python 2 的默认值。SQLAlchemy 的 Python 2 支持仍然允许对一些 DBAPI（如 cx_Oracle）进行显式的字符串转换为 Unicode，但现在是在 DBAPI 级别执行，而不是作为标准的 SQLAlchemy 结果行处理函数。

1.  上述字符串转换在使用时通过 C 扩展被设计得非常高效，以至于即使在 1.4 版本中，SQLAlchemy 的字节到 Unicode 编解码钩子被插入到 cx_Oracle 中，据观察，它比 cx_Oracle 自己的钩子更高效；这意味着在任何情况下，转换所有���符串的开销不再像最初那样显著。

1.  行处理函数在大多数其他情况下不被使用；例外情况包括 SQLite 的日期时间支持，一些后端的 JSON 支持，一些数值处理程序，如字符串到`Decimal`。在`Decimal`的情况下，Python 3 也标准化了高性能的`cdecimal`实现，而在 Python 2 中则继续使用性能较低的纯 Python 版本。

1.  在真实世界的用例中，很少有需要获取完整行而只需要几列的情况。在 SQLAlchemy 的早期，来自其他语言的数据库代码形式“row = fetch(‘SELECT * FROM table’)”很常见；然而，使用 SQLAlchemy 的表达式语言，实际观察到的代码通常只使用所需的特定列。

另请参见

Query 返回的“KeyedTuple”对象被 Row 替换

ORM Session.execute() 在所有情况下使用“future”风格的结果集

[#4710](https://www.sqlalchemy.org/trac/ticket/4710)

#### 理由：为了更像一个命名元组而不是一个映射

命名元组和映射之间在布尔运算方面的区别可以总结如下。给定一个伪代码中的“命名元组”：

```py
row = (id: 5,  name: 'some name')
```

最大的跨不兼容差异是`__contains__`的行为：

```py
"id" in row  # True for a mapping, False for a named tuple
"some name" in row  # False for a mapping, True for a named tuple
```

在 1.4 版本中，当核心结果集返回一个`LegacyRow`时，上述的`"id" in row`比较仍然会成功，但会发出一个弃用警告。要将“in”运算符用作映射，请使用`Row._mapping`属性：

```py
"id" in row._mapping
```

SQLAlchemy 2.0 的结果对象将具有`.mappings()`修饰符，以便可以直接接收这些映射：

```py
# using sqlalchemy.future package
for row in result.mappings():
    row["id"]
```

#### 代理行为消失，对于现代用法也是不必要的

对`Row`进行重构，使其像元组一样行为需要所有数据值在一开始就完全可用。这是与`RowProxy`的内部行为变化相对应的，其中结果行处理函数会在访问行的元素时被调用，而不是在首次获取行时。这意味着例如从 SQLite 检索 datetime 值时，`RowProxy`对象中的数据以前看起来是这样的：

```py
row_proxy = (1, "2019-12-31 19:56:58.272106")
```

然后通过`__getitem__`访问时，将动态使用`datetime.strptime()`函数将上述字符串日期转换为`datetime`对象。通过新的架构，当元组返回时，`datetime()`对象已经存在于其中，`datetime.strptime()`函数仅在一开始被调用了一次：

```py
row = (1, datetime.datetime(2019, 12, 31, 19, 56, 58, 272106))
```

在 SQLAlchemy 中，`RowProxy`和`Row`对象是大部分 SQLAlchemy C 扩展代码的主要位置。此代码已进行了大量重构，以有效地提供新的行为，并且由于`Row`的设计现在明显更简单，因此总体性能已得到提高。

先前行为的背后逻辑假定了一个使用模型，在这个模型中，结果行可能会有几十个甚至几百个列存在，其中大多数列不会被访问，并且其中大多数列需要某种结果值处理函数。通过仅在需要时调用处理函数，目标是不需要太多的结果处理函数，从而提高性能。

有许多原因导致上述假设不成立：

1.  调用的绝大多数行处理函数是将字节字符串解码为 Python Unicode 字符串。这是在 Python Unicode 开始使用并且 Python 3 存在之前的情况。一旦 Python 3 被引入，几年后，所有 Python DBAPI 都承担起直接支持交付 Python Unicode 对象的正确角色，在 Python 2 和 Python 3 下，前者是一个选项，而后者则是唯一的前进方式。最终，在大多数情况下，它也成为了 Python 2 的默认选项。SQLAlchemy 的 Python 2 支持仍然可以为某些 DBAPI（如 cx_Oracle）执行显式的字符串到 Unicode 转换，但现在是在 DBAPI 级别执行，而不是作为标准的 SQLAlchemy 结果行处理函数。

1.  上述字符串转换在使用时，通过 C 扩展实现了极高的性能，以至于甚至在 1.4 版本中，SQLAlchemy 的字节到 Unicode 编解码器钩子被插入了 cx_Oracle 中，在这种情况下，它被观察到比 cx_Oracle 自己的钩子更具性能；这意味着在任何情况下，转换行中的所有字符串的开销都不像最初那样显著。

1.  大多数情况下不使用行处理函数；例外情况包括 SQLite 的日期时间支持，一些后端的 JSON 支持，一些数值处理程序，如字符串转换为`Decimal`。在`Decimal`的情况下，Python 3 还规范化了高性能的`cdecimal`实现，而在 Python 2 中并非如此，它继续使用性能远低于`cdecimal`的纯 Python 版本。

1.  在真实世界的使用案例中，很少会出现仅需要几列的完整行抓取。在 SQLAlchemy 的早期，来自其他语言的数据库代码形式“row = fetch(‘SELECT * FROM table’)”很常见；然而，使用 SQLAlchemy 的表达式语言，观察到的实际代码通常只使用所需的特定列。

另请参见

查询返回的“KeyedTuple”对象被“Row”替换

ORM Session.execute() 在所有情况下使用“future”风格的结果集

[#4710](https://www.sqlalchemy.org/trac/ticket/4710)

### SELECT 对象和派生的 FROM 子句允许重复的列和列标签

此更改允许`select()`构造现在允许重复的列标签以及重复的列对象本身，以便结果元组按照选择列的相同方式组织和排序。ORM `Query`已经以这种方式工作，因此此更改允许更大程度上的跨两者的兼容性，这是 2.0 转换的关键目标之一：

```py
>>> from sqlalchemy import column, select
>>> c1, c2, c3, c4 = column("c1"), column("c2"), column("c3"), column("c4")
>>> stmt = select(c1, c2, c3.label("c2"), c2, c4)
>>> print(stmt)
SELECT  c1,  c2,  c3  AS  c2,  c2,  c4 
```

为了支持这一更改，`ColumnCollection` 由 `SelectBase` 使用，以及用于派生 FROM 子句的情况，例如子查询，也支持重复列；这包括新的 `SelectBase.selected_columns` 属性，已弃用的 `SelectBase.c` 属性，以及在构造中看到的 `FromClause.c` 属性，例如 `Subquery` 和 `Alias`：

```py
>>> list(stmt.selected_columns)
[
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcca20; c1>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcc9e8; c2>,
 <sqlalchemy.sql.elements.Label object at 0x7fa540b3e2e8>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540bcc9e8; c2>,
 <sqlalchemy.sql.elements.ColumnClause at 0x7fa540897048; c4>
]

>>> print(stmt.subquery().select())
SELECT  anon_1.c1,  anon_1.c2,  anon_1.c2,  anon_1.c2,  anon_1.c4
FROM  (SELECT  c1,  c2,  c3  AS  c2,  c2,  c4)  AS  anon_1 
```

`ColumnCollection` 还允许通过整数索引访问，以支持字符串 “key” 不明确时的情况：

```py
>>> stmt.selected_columns[2]
<sqlalchemy.sql.elements.Label object at 0x7fa540b3e2e8>
```

为了适应 `ColumnCollection` 在对象（例如 `Table` 和 `PrimaryKeyConstraint`）中的使用，保留了更适用于这些对象的旧的 “去重” 行为，并将其放在一个新的类 `DedupeColumnCollection` 中。

此更改包括将熟悉的警告 `"Column %r on table %r being replaced by %r, which has the same key.  Consider use_labels for select() statements."` **删除**；`Select.apply_labels()` 仍然可用，并且仍然由 ORM 用于所有 SELECT 操作，但它并不意味着列对象的重复，尽管它暗示了隐式生成的标签的重复：

```py
>>> from sqlalchemy import table
>>> user = table("user", column("id"), column("name"))
>>> stmt = select(user.c.id, user.c.name, user.c.id).apply_labels()
>>> print(stmt)
SELECT "user".id AS user_id, "user".name AS user_name, "user".id AS id_1
FROM "user"
```

最后，这个改变使得创建 UNION 和其他 `_selectable.CompoundSelect` 对象变得更加容易，通过确保 SELECT 语句中的列的数量和位置与所给定的相同，例如在以下用例中：

```py
>>> s1 = select(user, user.c.id)
>>> s2 = select(c1, c2, c3)
>>> from sqlalchemy import union
>>> u = union(s1, s2)
>>> print(u)
SELECT  "user".id,  "user".name,  "user".id
FROM  "user"  UNION  SELECT  c1,  c2,  c3 
```

[#4753](https://www.sqlalchemy.org/trac/ticket/4753)

### 使用 CAST 或类似方法改进简单列表达式的列标签

有用户指出，PostgreSQL 数据库在使用 CAST 等函数对命名列进行操作时具有方便的行为，因为结果列名与内部表达式相同：

```py
test=> SELECT CAST(data AS VARCHAR) FROM foo;

data
------
 5
(1 row)
```

这允许将 CAST 应用于表列，同时不丢失结果行中列名（上述使用名称 `"data"`）。与 MySQL/MariaDB 等大多数数据库相比，此处的列名取自完整的 SQL 表达式，而且不太可移植：

```py
MariaDB [test]> SELECT CAST(data AS CHAR) FROM foo;
+--------------------+
| CAST(data AS CHAR) |
+--------------------+
| 5                  |
+--------------------+
1 row in set (0.003 sec)
```

在 SQLAlchemy Core 表达式中，我们从不处理像上面那样的原始生成名称，因为 SQLAlchemy 对这些表达式应用自动标记，直到现在始终是所谓的“匿名”表达式：

```py
>>> print(select(cast(foo.c.data, String)))
SELECT  CAST(foo.data  AS  VARCHAR)  AS  anon_1  #  old  behavior
FROM  foo 
```

这些匿名表达式是必需的，因为 SQLAlchemy 的`ResultProxy`大量使用结果列名称来匹配数据类型，例如`String`数据类型曾经具有结果行处理行为，以正确匹配列，因此最重要的是名称必须易于以与数据库无关的方式确定，并且在所有情况下都是唯一的。在 SQLAlchemy 1.0 中作为[#918](https://www.sqlalchemy.org/trac/ticket/918)的一部分，对于大多数 Core SELECT 构造，不再需要依赖结果行中的命名列（特别是 PEP-249 游标的`cursor.description`元素）；在 1.4 版本中，整个系统对于具有重复列或标签名称的 SELECT 语句变得更加舒适，例如在 SELECT 对象和派生 FROM 子句允许重复列和列标签中。因此，我们现在模拟 PostgreSQL 对于对单个列进行简单修改的合理行为，最显著的是使用 CAST：

```py
>>> print(select(cast(foo.c.data, String)))
SELECT  CAST(foo.data  AS  VARCHAR)  AS  data
FROM  foo 
```

对于没有名称的表达式，CAST 使用先前的逻辑生成通常的“匿名”标签：

```py
>>> print(select(cast("hi there," + foo.c.data, String)))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  anon_1
FROM  foo 
```

对于`Label`的`cast()`，尽管必须省略标签表达式，因为这些在 CAST 内部不会呈现，但仍将使用给定的名称：

```py
>>> print(select(cast(("hi there," + foo.c.data).label("hello_data"), String)))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  hello_data
FROM  foo 
```

当然，一直以来，`Label`可以应用于表达式的外部，直接应用“AS <name>”标签：

```py
>>> print(select(cast(("hi there," + foo.c.data), String).label("hello_data")))
SELECT  CAST(:data_1  +  foo.data  AS  VARCHAR)  AS  hello_data
FROM  foo 
```

[#4449](https://www.sqlalchemy.org/trac/ticket/4449)

### 用于 Oracle、SQL Server 中 LIMIT/OFFSET 的新“后编译”绑定参数

1.4 系列的一个主要目标是确保所有 Core SQL 构造都是完全可缓存的，这意味着特定的`Compiled`结构将生成相同的 SQL 字符串，而不管与之一起使用的任何 SQL 参数，其中特别包括用于指定 LIMIT 和 OFFSET 值的参数，通常用于分页和“top N”样式的结果。

虽然 SQLAlchemy 多年来一直使用绑定参数来处理 LIMIT/OFFSET 方案，但仍然存在一些离群值，其中不允许使用这些参数，包括 SQL Server 的“TOP N”语句，例如：

```py
SELECT  TOP  5  mytable.id,  mytable.data  FROM  mytable
```

以及在 Oracle 中，如果将 `optimize_limits=True` 参数传递给 `create_engine()` 时，SQLAlchemy 将使用 FIRST_ROWS() 提示，不允许它们，但还有一种使用带有 ROWNUM 比较的绑定参数被报告为生成较慢的查询计划：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS(5) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id,  mytable.data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  :param_1
)  anon_1  WHERE  ora_rn  >  :param_2
```

为了使所有语句在编译级别无条件可缓存，添加了一种新形式的绑定参数，称为“后编译”参数，它使用与“扩展 IN 参数”相同的机制。这是一个`bindparam()`，其行为与任何其他绑定参数完全相同，只是参数值在发送到 DBAPI `cursor.execute()` 方法之前会被字面渲染到 SQL 字符串中。新参数在 SQL Server 和 Oracle 方言内部使用，以便驱动程序接收字面渲染值，但 SQLAlchemy 的其余部分仍然可以将其视为绑定参数。现在，使用 `str(statement.compile(dialect=<dialect>))` 将上述两个语句字符串化后，看起来如下：

```py
SELECT  TOP  [POSTCOMPILE_param_1]  mytable.id,  mytable.data  FROM  mytable
```

和：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS([POSTCOMPILE__ora_frow_1]) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id,  mytable.data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  [POSTCOMPILE_param_1]
)  anon_1  WHERE  ora_rn  >  [POSTCOMPILE_param_2]
```

`[POSTCOMPILE_<param>]` 格式也是在使用“扩展 IN”时所见的格式。

在查看 SQL 记录输出时，将看到语句的最终形式：

```py
SELECT  anon_1.id,  anon_1.data  FROM  (
  SELECT  /*+ FIRST_ROWS(5) */
  anon_2.id  AS  id,
  anon_2.data  AS  data,
  ROWNUM  AS  ora_rn  FROM  (
  SELECT  mytable.id  AS  id,  mytable.data  AS  data  FROM  mytable
  )  anon_2
  WHERE  ROWNUM  <=  8
)  anon_1  WHERE  ora_rn  >  3
```

“后编译参数”功能通过`bindparam.literal_execute`参数作为公共 API 公开，但目前不打算供一般使用。字面值是使用底层数据类型的`TypeEngine.literal_processor()`渲染的，在 SQLAlchemy 中的范围**极其有限**，仅支持整数和简单字符串值。

[#4808](https://www.sqlalchemy.org/trac/ticket/4808)

### 基于子事务，现在可以使连接级事务不活动

现在，`Connection` 包括了一个行为，即由于内部事务的回滚，`Transaction` 可以变为不活动，但是直到它自己被回滚之前，`Transaction` 不会清除。

这本质上是一个新的错误条件，如果内部“子”事务已被回滚，则会禁止在`Connection`上继续执行语句。该行为与 ORM `Session`的行为非常相似，如果已开始外部事务，则需要将其回滚以清除无效事务；此行为在“由于刷新期间的先前异常，此会话的事务已被回滚。”（或类似）中有描述。

虽然`Connection`的行为模式比`Session`更宽松，但由于它有助于识别子事务已回滚 DBAPI 事务，但外部代码并不知晓并尝试继续进行，实际上在新事务上运行操作，因此进行了此更改。在将会话加入外部事务（例如用于测试套件）中描述的“测试工具”模式是发生这种情况的常见地方。

Core 和 ORM 的“子事务”功能本身已被弃用，并将不再出现在 2.0 版本中。因此，这种新的错误条件本身是临时的，一旦子事务被移除，它将不再适用。

为了使用不包括子事务的 2.0 风格行为，可以在`create_engine()`上使用`create_engine.future`参数。

错误消息在错误页面中描述为此连接处于非活动事务状态。请在继续之前完全回滚()。

### 枚举和布尔数据类型不再默认为“创建约束”

`Enum.create_constraint`和`Boolean.create_constraint`参数现在默认为 False，表示当创建这两种数据类型的所谓“非本地”版本时，默认情况下不会生成 CHECK 约束。这些 CHECK 约束会带来应该选择的模式管理维护复杂性，而不是默认打开。

要确保为这些类型发出 CREATE CONSTRAINT，请将这些标志设置为`True`：

```py
class Spam(Base):
    __tablename__ = "spam"
    id = Column(Integer, primary_key=True)
    boolean = Column(Boolean(create_constraint=True))
    enum = Column(Enum("a", "b", "c", create_constraint=True))
```

[#5367](https://www.sqlalchemy.org/trac/ticket/5367)

## 新功能 - ORM

### 为列提升加载

“raiseload”功能现在可用于使用`defer.raiseload`参数的`defer()`的基于列的属性，当访问未加载的属性时引发`InvalidRequestError`。这与关系加载使用的`raiseload()`选项的工作方式相同：

```py
book = session.query(Book).options(defer(Book.summary, raiseload=True)).first()

# would raise an exception
book.summary
```

要在映射上配置列级 raiseload，可以使用`deferred.raiseload`参数的`deferred()`。然后可以在查询时使用`undefer()`选项来急切加载属性：

```py
class Book(Base):
    __tablename__ = "book"

    book_id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    summary = deferred(Column(String(2000)), raiseload=True)
    excerpt = deferred(Column(Text), raiseload=True)

book_w_excerpt = session.query(Book).options(undefer(Book.excerpt)).first()
```

最初认为现有的`raiseload()`选项适用于`relationship()`属性，应扩展以支持基于列的属性。然而，这将破坏`raiseload()`的“通配符”行为，该行为被记录为允许阻止所有关系加载：

```py
session.query(Order).options(joinedload(Order.items), raiseload("*"))
```

如果我们扩展了`raiseload()`以适应列，通配符也将阻止列加载，从而导致向后不兼容的更改；此外，不清楚`raiseload()`是否同时涵盖列表达式和关系，如何实现上述仅阻止关系加载的效果，而不添加新的 API。因此，为了保持简单，列的选项仍然在`defer()`上：

> `raiseload()` - 查询选项，用于关系加载时引发异常
> 
> `defer.raiseload` - 查询选项，用于列表达式加载时引发异常

作为这一变化的一部分，“deferred”与属性过期的行为已经改变。以前，当对象被标记为过期，然后通过访问其中一个过期属性来取消过期时，映射为“deferred”的属性也会加载。现在已更改为映射中延迟的属性永远不会“过期”，只有在作为延迟加载器的一部分访问时才会加载。

一个未映射为“deferred”的属性，但在查询时通过`defer()`选项进行了延迟，当对象或属性过期时将被重置；也就是说，延迟选项被移除。这与以前的行为相同。

另请参阅

使用 raiseload 来防止延迟列加载

[#4826](https://www.sqlalchemy.org/trac/ticket/4826)  ### 使用 psycopg2 进行 ORM 批量插入现在在大多数情况下批量处理带有 RETURNING 的语句

psycopg2 方言特性的变化“默认情况下为 INSERT 语句添加 RETURNING”在 Core 中添加了对“executemany” + “RETURNING”同时支持的功能，现在默认情况下通过 psycopg2 的`execute_values()`扩展为 psycopg2 方言启用。ORM 刷新过程现在利用此功能，以便在不丢失能够将 INSERT 语句批量处理在一起的性能优势的同时，实现新生成的主键值和服务器默认值的检索。此外，psycopg2 的`execute_values()`扩展本身通过将一个 INSERT 语句重写为包含许多“VALUES”表达式的单个语句，而不是重复调用相同的语句，提供了五倍的性能改进，因为 psycopg2 缺乏预先准备语句的能力，这通常被期望为这种方法提供性能。

SQLAlchemy 在其示例中包含一个性能套件，在这里我们可以比较“batch_inserts”运行程序在 1.3 和 1.4 版本中生成的时间，显示大多数批量插入的速度提升了 3 倍至 5 倍：

```py
# 1.3
$ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
test_flush_no_pk : (100000 iterations); total time 14.051527 sec
test_bulk_save_return_pks : (100000 iterations); total time 15.002470 sec
test_flush_pk_given : (100000 iterations); total time 7.863680 sec
test_bulk_save : (100000 iterations); total time 6.780378 sec
test_bulk_insert_mappings :  (100000 iterations); total time 5.363070 sec
test_core_insert : (100000 iterations); total time 5.362647 sec

# 1.4 with enhancement
$ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
test_flush_no_pk : (100000 iterations); total time 3.820807 sec
test_bulk_save_return_pks : (100000 iterations); total time 3.176378 sec
test_flush_pk_given : (100000 iterations); total time 4.037789 sec
test_bulk_save : (100000 iterations); total time 2.604446 sec
test_bulk_insert_mappings : (100000 iterations); total time 1.204897 sec
test_core_insert : (100000 iterations); total time 0.958976 sec
```

请注意，`execute_values()`扩展会在 SQLAlchemy 记录后修改 psycopg2 层的 INSERT 语句。因此，在 SQL 记录中，可以看到参数集被批处理在一起，但多个“values”的连接在应用程序端不可见：

```py
2020-06-27 19:08:18,166 INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (%(data)s) RETURNING a.id
2020-06-27 19:08:18,166 INFO sqlalchemy.engine.Engine [generated in 0.00698s] ({'data': 'data 1'}, {'data': 'data 2'}, {'data': 'data 3'}, {'data': 'data 4'}, {'data': 'data 5'}, {'data': 'data 6'}, {'data': 'data 7'}, {'data': 'data 8'}  ... displaying 10 of 4999 total bound parameter sets ...  {'data': 'data 4998'}, {'data': 'data 4999'})
2020-06-27 19:08:18,254 INFO sqlalchemy.engine.Engine COMMIT
```

可以通过在 PostgreSQL 端启用语句记录来查看最终的 INSERT 语句：

```py
2020-06-27 19:08:18.169 EDT [26960] LOG:  statement: INSERT INTO a (data)
VALUES ('data 1'),('data 2'),('data 3'),('data 4'),('data 5'),('data 6'),('data
7'),('data 8'),('data 9'),('data 10'),('data 11'),('data 12'),
... ('data 999'),('data 1000') RETURNING a.id

2020-06-27 19:08:18.175 EDT
[26960] LOG:  statement: INSERT INTO a (data) VALUES ('data 1001'),('data
1002'),('data 1003'),('data 1004'),('data 1005 '),('data 1006'),('data
1007'),('data 1008'),('data 1009'),('data 1010'),('data 1011'), ...
```

该功能默认将行分组为 1000 个一组，可以使用文档中记录的`executemany_values_page_size`参数来影响：

[#5263](https://www.sqlalchemy.org/trac/ticket/5263)  ### ORM 批量更新和删除在可用时使用 RETURNING 进行“fetch”策略

使用“fetch”策略的 ORM 批量更新或删除：

```py
sess.query(User).filter(User.age > 29).update(
    {"age": User.age - 10}, synchronize_session="fetch"
)
```

如果后端数据库支持，现在将使用 RETURNING；目前包括 PostgreSQL 和 SQL Server（Oracle 方言不支持返回多行）：

```py
UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s RETURNING users.id
[generated in 0.00060s] {'age_int_1': 10, 'age_int_2': 29}
Col ('id',)
Row (2,)
Row (4,)
```

对于不支持返回多行的后端，仍然使用先前的方法在主键之前发出 SELECT：

```py
SELECT users.id FROM users WHERE users.age_int > %(age_int_1)s
[generated in 0.00043s] {'age_int_1': 29}
Col ('id',)
Row (2,)
Row (4,)
UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s
[generated in 0.00102s] {'age_int_1': 10, 'age_int_2': 29}
```

这一变化的一个复杂挑战之一是支持诸如水平分片扩展之类的情况，其中单个批量更新或删除可能在一些支持 RETURNING 的后端之间多路复用，而另一些则不支持。新的 1.4 执行架构支持这种情况，以便“fetch”策略可以保持不变，优雅地降级到使用 SELECT，而不是必须添加一个新的不具备后端通用性的“returning”策略。

作为这一变化的一部分，“fetch”策略也变得更加高效，不再使匹配行的对象过期，对于可以在 Python 中评估的 SET 子句中使用的 Python 表达式；这些直接分配到对象上，方式与“evaluate”策略相同。只有对于无法评估的 SQL 表达式，它才会退回到使属性过期。对于无法评估的值，“evaluate”策略也已增强为退回到“expire”。  ### 列的 Raiseload

现在，使用`defer.raiseload`参数的`defer()`，可以为基于列的属性提供“raiseload”功能，当访问未加载的属性时引发`InvalidRequestError`，这与关系加载使用的`raiseload()`选项的方式相同：

```py
book = session.query(Book).options(defer(Book.summary, raiseload=True)).first()

# would raise an exception
book.summary
```

要在映射上配置列级 raiseload，可以使用`deferred.raiseload`参数的`deferred()`。然后可以在查询时使用`undefer()`选项来急切加载属性：

```py
class Book(Base):
    __tablename__ = "book"

    book_id = Column(Integer, primary_key=True)
    title = Column(String(200), nullable=False)
    summary = deferred(Column(String(2000)), raiseload=True)
    excerpt = deferred(Column(Text), raiseload=True)

book_w_excerpt = session.query(Book).options(undefer(Book.excerpt)).first()
```

最初考虑扩展现有的`raiseload()`选项，以支持面向列的属性。然而，这将破坏`raiseload()`的“通配符”行为，该行为被记录为允许阻止所有关系加载：

```py
session.query(Order).options(joinedload(Order.items), raiseload("*"))
```

如果我们扩展了`raiseload()`以适应列，通配符也将阻止列的加载，因此这将是一个不兼容的更改；此外，不清楚`raiseload()`是否同时涵盖列表达式和关系，如何实现上述仅阻止关系加载的效果，而不添加新的 API。因此，为了保持简单，列的选项仍然在`defer()`上：

> `raiseload()` - 查询选项，用于为关系加载时引发异常
> 
> `defer.raiseload` - 查询选项，用于为列表达式加载时引发异常

作为这一变化的一部分，“延迟”与属性过期的行为已经改变。以前，当对象被标记为过期，然后通过访问其中一个过期属性来取消过期时，映射为“deferred”的属性也会加载。现在已经更改为映射为延迟的属性永远不会“取消过期”，它只会在作为延迟加载器的一部分访问时加载。

一个未映射为“deferred”的属性，但在查询时通过`defer()`选项进行了延迟，当对象或属性过期时将被重置；也就是说，延迟选项被移除。这与以前的行为相同。

另请参见

使用 raiseload 阻止延迟列加载

[#4826](https://www.sqlalchemy.org/trac/ticket/4826)

### 使用 psycopg2 进行 ORM 批量插入现在在大多数情况下批量执行带有 RETURNING 的语句

psycopg2 方言特性“execute_values”默认使用 RETURNING 进行 INSERT 语句的更改在 Core 中添加了对“executemany” + “RETURNING”同时支持的功能，现在默认情况下为 psycopg2 方言启用了使用 psycopg2 的`execute_values()`扩展。ORM 刷新过程现在利用了这一特性，以便在不失去能够批量将 INSERT 语句一起执行的性能优势的同时，实现新生成的主键值和服务器默认值的检索。此外，psycopg2 的`execute_values()`扩展本身比 psycopg2 的默认“executemany”实现提供了五倍的性能改进，通过将 INSERT 语句重写为在一个语句中包含多个“VALUES”表达式，而不是重复调用相同的语句，因为 psycopg2 缺乏预先准备语句的能力，这通常被期望为这种方法提供性能。

SQLAlchemy 在其示例中包含了一个性能套件，在这里我们可以将“batch_inserts”运行程序生成的时间与 1.3 和 1.4 进行比较，对大多数批量插入的变体显示出 3 倍至 5 倍的加速：

```py
# 1.3
$ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
test_flush_no_pk : (100000 iterations); total time 14.051527 sec
test_bulk_save_return_pks : (100000 iterations); total time 15.002470 sec
test_flush_pk_given : (100000 iterations); total time 7.863680 sec
test_bulk_save : (100000 iterations); total time 6.780378 sec
test_bulk_insert_mappings :  (100000 iterations); total time 5.363070 sec
test_core_insert : (100000 iterations); total time 5.362647 sec

# 1.4 with enhancement
$ python -m examples.performance bulk_inserts --dburl postgresql://scott:tiger@localhost/test
test_flush_no_pk : (100000 iterations); total time 3.820807 sec
test_bulk_save_return_pks : (100000 iterations); total time 3.176378 sec
test_flush_pk_given : (100000 iterations); total time 4.037789 sec
test_bulk_save : (100000 iterations); total time 2.604446 sec
test_bulk_insert_mappings : (100000 iterations); total time 1.204897 sec
test_core_insert : (100000 iterations); total time 0.958976 sec
```

注意，`execute_values()`扩展会修改 SQLAlchemy 记录的 psycopg2 层中的 INSERT 语句**之后**。因此，在 SQL 记录中，可以看到参数集被批量处理在一起，但多个“values”的连接在应用程序端不可见：

```py
2020-06-27 19:08:18,166 INFO sqlalchemy.engine.Engine INSERT INTO a (data) VALUES (%(data)s) RETURNING a.id
2020-06-27 19:08:18,166 INFO sqlalchemy.engine.Engine [generated in 0.00698s] ({'data': 'data 1'}, {'data': 'data 2'}, {'data': 'data 3'}, {'data': 'data 4'}, {'data': 'data 5'}, {'data': 'data 6'}, {'data': 'data 7'}, {'data': 'data 8'}  ... displaying 10 of 4999 total bound parameter sets ...  {'data': 'data 4998'}, {'data': 'data 4999'})
2020-06-27 19:08:18,254 INFO sqlalchemy.engine.Engine COMMIT
```

可以通过在 PostgreSQL 端启用语句记录来查看最终的 INSERT 语句：

```py
2020-06-27 19:08:18.169 EDT [26960] LOG:  statement: INSERT INTO a (data)
VALUES ('data 1'),('data 2'),('data 3'),('data 4'),('data 5'),('data 6'),('data
7'),('data 8'),('data 9'),('data 10'),('data 11'),('data 12'),
... ('data 999'),('data 1000') RETURNING a.id

2020-06-27 19:08:18.175 EDT
[26960] LOG:  statement: INSERT INTO a (data) VALUES ('data 1001'),('data
1002'),('data 1003'),('data 1004'),('data 1005 '),('data 1006'),('data
1007'),('data 1008'),('data 1009'),('data 1010'),('data 1011'), ...
```

该功能默认将行分组为每组 1000 行，可以使用`executemany_values_page_size`参数进行影响，该参数在 Psycopg2 快速执行助手中有文档记录。

[#5263](https://www.sqlalchemy.org/trac/ticket/5263)

### ORM 批量更新和删除在可用时使用 RETURNING 进行“fetch”策略

使用“fetch”策略的 ORM 批量更新或删除：

```py
sess.query(User).filter(User.age > 29).update(
    {"age": User.age - 10}, synchronize_session="fetch"
)
```

如果后端数据库支持，现在将使用 RETURNING；目前包括 PostgreSQL 和 SQL Server（Oracle 方言不支持多行 RETURNING）：

```py
UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s RETURNING users.id
[generated in 0.00060s] {'age_int_1': 10, 'age_int_2': 29}
Col ('id',)
Row (2,)
Row (4,)
```

对于不支持多行 RETURNING 的后端，仍然使用之前的在主键之前发出 SELECT 的方法：

```py
SELECT users.id FROM users WHERE users.age_int > %(age_int_1)s
[generated in 0.00043s] {'age_int_1': 29}
Col ('id',)
Row (2,)
Row (4,)
UPDATE users SET age_int=(users.age_int - %(age_int_1)s) WHERE users.age_int > %(age_int_2)s
[generated in 0.00102s] {'age_int_1': 10, 'age_int_2': 29}
```

这一变化的一个复杂挑战之一是支持水平分片扩展等情况，其中单个批量更新或删除可能在一些支持 RETURNING 的后端中进行多路复用，而一些不支持。新的 1.4 执行架构支持这种情况，以便“fetch”策略可以保持完整，优雅地降级到使用 SELECT，而不是必须添加一个新的不具备后端通用性的“returning”策略。

作为这一变化的一部分，“fetch”策略也变得更加高效，不再使匹配行的对象过期，对于在 SET 子句中使用的可以在 Python 中评估的 Python 表达式；这些表达式会直接分配到对象上，就像“evaluate”策略一样。 只有对于无法评估的 SQL 表达式，才会退回到使属性过期。 “evaluate”策略也已经增强，对于无法评估的值会退回到“expire”。

## 行为变化 - ORM

### 查询返回的“KeyedTuple”对象已被`Row`替换

如在 RowProxy 不再是“代理”；现在称为 Row 并且行为类似于增强的命名元组中讨论的，Core 的`RowProxy`对象现在被一个名为`Row`的类替换。 基本的`Row`对象现在更像一个命名元组，因此现在用作由`Query`对象返回的类似元组的结果的基础，而不是以前的“KeyedTuple”类。

这样做的理由是，到达 SQLAlchemy 2.0，Core 和 ORM SELECT 语句将使用行为类似于命名元组的相同`Row`对象返回结果行。 通过`Row`的`Row._mapping`属性可以获得类似字典的功能。 在此期间，Core 结果集将使用一个`Row`子类`LegacyRow`，该子类保留了以前的字典/元组混合行为，以确保向后兼容性，而`Row`类将直接用于由`Query`对象返回的 ORM 元组结果。

已经努力使得`Row`的大部分功能集在 ORM 中可用，这意味着可以通过字符串名称以及实体/列进行访问：

```py
row = s.query(User, Address).join(User.addresses).first()

row._mapping[User]  # same as row[0]
row._mapping[Address]  # same as row[1]
row._mapping["User"]  # same as row[0]
row._mapping["Address"]  # same as row[1]

u1 = aliased(User)
row = s.query(u1).only_return_tuples(True).first()
row._mapping[u1]  # same as row[0]

row = s.query(User.id, Address.email_address).join(User.addresses).first()

row._mapping[User.id]  # same as row[0]
row._mapping["id"]  # same as row[0]
row._mapping[users.c.id]  # same as row[0]
```

另请参见

RowProxy 不再是“代理”；现在称为 Row，并且行为类似于增强的命名元组

[#4710](https://www.sqlalchemy.org/trac/ticket/4710).  ### 会话功能新的“autobegin”行为

以前，在其默认模式 `autocommit=False` 下，`Session` 在构造时会立即开始一个 `SessionTransaction` 对象，并且在每次调用 `Session.rollback()` 或 `Session.commit()` 后还会创建一个新的对象。

新行为是，这个 `SessionTransaction` 对象现在只在需要时才创建，当调用 `Session.add()` 或 `Session.execute()` 等方法时。但现在也可以显式调用 `Session.begin()` 来开始事务，即使在 `autocommit=False` 模式下，从而与未来风格的 `_base.Connection` 的行为相匹配。

这表明的行为变化是：

+   现在，即使在 `autocommit=False` 模式下，`Session` 也可以处于未开始事务的状态。以前，这种状态仅在“自动提交”模式下可用。

+   在这种状态下，`Session.commit()` 和 `Session.rollback()` 方法不起作用。依赖这些方法来使所有对象过期的代码应明确使用 `Session.begin()` 或 `Session.expire_all()` 来适应其用例。

+   当 `Session` 创建时，或在 `Session.rollback()` 或 `Session.commit()` 完成后，不会立即触发 `SessionEvents.after_transaction_create()` 事件钩子。

+   `Session.close()`方法也不意味着隐式开始新的`SessionTransaction`。

另请参阅

自动开始

#### 理由

`Session` 对象的默认行为`autocommit=False`在历史上意味着始终存在一个与`Session`相关联的`SessionTransaction`对象，通过`Session.transaction`属性关联。当给定的`SessionTransaction`完成时，由于提交、回滚或关闭，它会立即被新的替代。`SessionTransaction`本身并不意味着使用任何连接相关资源，因此这种长期存在的行为具有特定的优雅之处，即`Session.transaction`的状态始终可预测为非 None。

然而，作为[#5056](https://www.sqlalchemy.org/trac/ticket/5056)倡议的一部分，大大减少引用循环，这一假设意味着调用`Session.close()`会导致一个仍然存在引用循环且更昂贵的`Session`对象需要清理，更不用说构造`SessionTransaction`对象的一点开销，这意味着对于例如调用`Session.commit()`然后`Session.close()`的`Session`会创建不必要的开销。

因此，决定`Session.close()`应该将`self.transaction`的内部状态，现在在内部称为`self._transaction`，保留为 None，并且只有在需要时才会创建新的`SessionTransaction`。为了一致性和代码覆盖率，此行为还扩展到了所有“自动开始”预期的点，而不仅仅是在调用`Session.close()`时。

特别是，这对订阅`SessionEvents.after_transaction_create()`事件钩子的应用程序造成了行为上的变化；以前，当首次构建`Session`时，此事件将被触发，以及对关闭先前事务的大多数操作，并将触发`SessionEvents.after_transaction_end()`。新行为是，当`Session`尚未创建新的`SessionTransaction`对象并且映射对象通过`Session.add()`和`Session.delete()`等方法与`Session`关联时，当调用`Session.transaction`属性时，当`Session.flush()`方法有任务要完成时等，`SessionEvents.after_transaction_create()`会按需触发。

此外，依赖于`Session.commit()`或`Session.rollback()`方法无条件使所有对象过期的代码将不再能够这样做。当没有发生任何更改时需要使所有对象过期的代码应该针对此情况调用`Session.expire_all()`。

除了更改`SessionEvents.after_transaction_create()`事件发出的时间以及`Session.commit()`或`Session.rollback()`的无操作性质外，此更改不应对`Session`对象的行为产生其他用户可见的影响；`Session`在调用`Session.close()`后仍然保持可用于新操作的行为，并且`Session`与`Engine`和数据库本身的交互顺序也应保持不受影响，因为这些操作已经以按需方式运行。

[#5074](https://www.sqlalchemy.org/trac/ticket/5074)  ### 只读视图关系不同步回引

在 1.3.14 中的[#5149](https://www.sqlalchemy.org/trac/ticket/5149)中，SQLAlchemy 开始在目标关系上同时使用`relationship.backref`或`relationship.back_populates`关键字时发出警告，同时使用`relationship.viewonly`标志。这是因为“只读”关系实际上不会保存对其所做的更改，这可能导致一些误导行为发生。然而，在[#5237](https://www.sqlalchemy.org/trac/ticket/5237)中，我们试图优化这种行为，因为在只读关系上设置回引是有合法用例的，包括回填属性有时由关系懒加载器用于确定在另一个方向上不需要额外的急加载，以及回填可以用于映射器内省和`backref()`可以是设置双向关系的便捷方式。

当时的解决方案是使从反向引用发生的“变化”成为可选的事情，使用 `relationship.sync_backref` 标志。在 1.4 版本中，对于也设置了 `relationship.viewonly` 的关系目标，默认情况下 `relationship.sync_backref` 的值为 False。这表明对于具有 viewonly 的关系所做的任何更改都不会影响另一侧或 `Session` 的状态：

```py
class User(Base):
    # ...

    addresses = relationship(Address, backref=backref("user", viewonly=True))

class Address(Base): ...

u1 = session.query(User).filter_by(name="x").first()

a1 = Address()
a1.user = u1
```

上面，`a1` 对象将**不会**被添加到 `u1.addresses` 集合中，也不会将 `a1` 对象添加到会话中。之前，这两件事情都是正确的。当 `relationship.viewonly` 为 `False` 时，不再发出应将 `relationship.sync_backref` 设置为 `False` 的警告，因为这现在是默认行为。

[#5237](https://www.sqlalchemy.org/trac/ticket/5237)  ### 在 2.0 版本中将删除的 cascade_backrefs 行为

SQLAlchemy 长期以来一直有一个根据反向引用分配将对象级联到 `Session` 的行为。假设 `User` 已经在 `Session` 中，将其分配给 `Address` 对象的 `Address.user` 属性，假设已经建立了双向关系，这意味着此时 `Address` 也会被放入 `Session` 中：

```py
u1 = User()
session.add(u1)

a1 = Address()
a1.user = u1  # <--- adds "a1" to the Session
```

上述行为是反向引用行为的意外副作用，因为由于 `a1.user` 暗示着 `u1.addresses.append(a1)`，所以 `a1` 将被级联到 `Session` 中。这仍然是整个 1.4 版本的默认行为。在某些时候，新增了一个标志 `relationship.cascade_backrefs` 来禁用上述行为，以及 `backref.cascade_backrefs` 来设置此标志，当关系由 `relationship.backref` 指定时，因为这可能会令人惊讶，并且还会妨碍某些操作，其中对象会过早地放入 `Session` 中并过早地刷新。

在 2.0 版本中，默认行为将是“cascade_backrefs”为 False，另外也不会有“True”行为，因为这通常不是一种理想的行为。当启用 2.0 版本的弃用警告时，当“反向引用级联”实际发生时，将发出警告。要获取新行为，可以在任何目标关系上将 `relationship.cascade_backrefs` 和 `backref.cascade_backrefs` 设置为 `False`，就像在 1.3 版本和更早版本中已经支持的那样，或者可以使用 `Session.future` 标志进入 2.0 样式 模式：

```py
Session = sessionmaker(engine, future=True)

with Session() as session:
    u1 = User()
    session.add(u1)

    a1 = Address()
    a1.user = u1  # <--- will not add "a1" to the Session
```

[#5150](https://www.sqlalchemy.org/trac/ticket/5150)  ### 急切加载器在取消过期操作期间发出

长期以来，人们一直期待的行为是，当访问过期对象时，配置的急切加载器将运行，以便在刷新或其他情况下未过期时急切加载过期对象上的关系。现已添加了此行为，以便 joinedloaders 将像往常一样添加内联 JOIN，并且当未过期对象被刷新或对象被刷新时，selectin/subquery 加载器将为给定关系运行“immediateload”操作：

```py
>>> a1 = session.query(A).options(joinedload(A.bs)).first()
>>> a1.data = "new data"
>>> session.commit()
```

上述 `A` 对象是使用与其关联的 `joinedload()` 选项加载的，以便急切加载 `bs` 集合。在 `session.commit()` 后，对象的状态已过期。在访问 `.data` 列属性时，对象将被刷新，现在这将包括 joinedload 操作：

```py
>>> a1.data
SELECT  a.id  AS  a_id,  a.data  AS  a_data,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  a  LEFT  OUTER  JOIN  b  AS  b_1  ON  a.id  =  b_1.a_id
WHERE  a.id  =  ? 
```

该行为适用于直接应用于`relationship()`的加载策略，以及与`Query.options()`一起使用的选项，前提是对象最初是由该查询加载的。

对于“次要”急加载器“selectinload”和“subqueryload”，这些加载器的 SQL 策略在单个对象上急加载属性时并不是必要的；因此，在刷新场景中，它们将调用“immediateload”策略，类似于“lazyload”发出的查询，作为额外的查询：

```py
>>> a1 = session.query(A).options(selectinload(A.bs)).first()
>>> a1.data = "new data"
>>> session.commit()
>>> a1.data
SELECT  a.id  AS  a_id,  a.data  AS  a_data
FROM  a
WHERE  a.id  =  ?
(1,)
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  ?  =  b.a_id
(1,) 
```

请注意，加载器选项不适用于以不同方式引入到`Session`中的对象。也就是说，如果`a1`对象只是在此`Session`中持久化，或者在应用急加载选项之前使用不同的查询加载了该对象，则该对象不具有与之关联的急加载选项。这并不是一个新概念，但是寻找刷新行为上的急加载的用户可能会发现这更加明显。

[#1763](https://www.sqlalchemy.org/trac/ticket/1763)  ### 列加载器如`deferred()`、`with_expression()`仅在外部、完整实体查询中指示时才生效

注意

本更改说明在本文档的早期版本中不存在，但对于所有 SQLAlchemy 1.4 版本都是相关的。

1.3 版本和之前版本从未支持的行为，但仍然会产生特定效果，即重新利用列加载器选项，如`defer()`和`with_expression()`在子查询中，以控制每个子查询的列子句中的 SQL 表达式。一个典型的例子是构造 UNION 查询，例如：

```py
q1 = session.query(User).options(with_expression(User.expr, literal("u1")))
q2 = session.query(User).options(with_expression(User.expr, literal("u2")))

q1.union_all(q2).all()
```

在 1.3 版本中，`with_expression()`选项会对 UNION 的每个元素生效，例如：

```py
SELECT  anon_1.anon_2  AS  anon_1_anon_2,  anon_1.user_account_id  AS  anon_1_user_account_id,
anon_1.user_account_name  AS  anon_1_user_account_name
FROM  (
  SELECT  ?  AS  anon_2,  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  ?  AS  anon_3,  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_1
('u1',  'u2')
```

SQLAlchemy 1.4 对加载器选项的概念变得更加严格，因此仅应用于查询的**最外层部分**，即用于填充实际要返回的 ORM 实体的 SELECT；在 1.4 中，上述查询将产生：

```py
SELECT  ?  AS  anon_1,  anon_2.user_account_id  AS  anon_2_user_account_id,
anon_2.user_account_name  AS  anon_2_user_account_name
FROM  (
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_2
('u1',)
```

即，`Query`的选项是从 UNION 的第一个元素中获取的，因为所有加载器选项只能在最顶层。第二个查询的选项被忽略了。

#### 理由

这种行为现在更加接近于其他种类的加载器选项，例如所有 SQLAlchemy 版本中的关系加载器选项，如`joinedload()`，其中在 UNION 情况下已经复制到查询的最顶层，并且仅从 UNION 的第一个元素中获取，舍弃查询的其他部分的任何选项。

这种隐式复制和选择性忽略选项的行为，如上所示，是相当任意的，是`Query`的遗留行为，也是`Query`以及其应用`Query.union_all()`方式存在不足的一个特定示例，因为如何将单个 SELECT 转换为其自身和另一个查询的 UNION 以及如何应用加载器选项到新语句上都是模糊的。

对于使用`defer()`的更常见情况，可以演示 SQLAlchemy 1.4 的行为通常优于 1.3。以下查询：

```py
q1 = session.query(User).options(defer(User.name))
q2 = session.query(User).options(defer(User.name))

q1.union_all(q2).all()
```

在 1.3 中，会笨拙地向内部查询添加 NULL，然后选择它：

```py
SELECT  anon_1.anon_2  AS  anon_1_anon_2,  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
  UNION  ALL
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
)  AS  anon_1
```

如果所有查询都没有设置相同的选项，上述场景将由于无法形成正确的 UNION 而引发错误。

而在 1.4 中，该选项仅应用于顶层，省略了对`User.name`的提取，并且避免了这种复杂性：

```py
SELECT  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_1
```

#### 正确的方法

使用 2.0-style 查询时，目前不会发出警告，但是嵌套的`with_expression()`选项一致被忽略，因为它们不适用于正在加载的实体，并且不会被隐式复制到任何地方。以下查询不会为`with_expression()`调用产生输出：

```py
s1 = select(User).options(with_expression(User.expr, literal("u1")))
s2 = select(User).options(with_expression(User.expr, literal("u2")))

stmt = union_all(s1, s2)

session.scalars(select(User).from_statement(stmt)).all()
```

产生 SQL：

```py
SELECT  user_account.id,  user_account.name
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name
FROM  user_account
```

要将`with_expression()`正确应用于`User`实体，应将其应用于查询的最外层级，使用每个 SELECT 的列子句中的普通 SQL 表达式：

```py
s1 = select(User, literal("u1").label("some_literal"))
s2 = select(User, literal("u2").label("some_literal"))

stmt = union_all(s1, s2)

session.scalars(
    select(User)
    .from_statement(stmt)
    .options(with_expression(User.expr, stmt.selected_columns.some_literal))
).all()
```

这将产生预期的 SQL：

```py
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
```

`User`对象本身将在其内容中包含此表达式，在`User.expr`下方。

SQLAlchemy 一直以来的行为是，在新创建的对象上访问映射属性会返回一个隐式生成的值，而不是引发`AttributeError`，例如标量属性为`None`或列表关系为`[]`：

```py
>>> u1 = User()
>>> u1.name
None
>>> u1.addresses
[]
```

上述行为的理由最初是为了使 ORM 对象更易于使用。由于 ORM 对象在首次创建时代表一个空行而没有任何状态，因此直观地认为其未访问的属性会解析为标量的`None`（或 SQL NULL），对于关系则解析为空集合。特别是，这使得一种极其常见的模式成为可能，即能够在不手动创建和分配空集合的情况下改变新集合：

```py
>>> u1 = User()
>>> u1.addresses.append(Address())  # no need to assign u1.addresses = []
```

直到 SQLAlchemy 的 1.0 版本，对于标量属性以及集合的初始化系统的行为是，`None`或空集合会被*填充*到对象的状态中，例如`__dict__`。这意味着以下两个操作是等效的：

```py
>>> u1 = User()
>>> u1.name = None  # explicit assignment

>>> u2 = User()
>>> u2.name  # implicit assignment just by accessing it
None
```

在上述情况下，`u1`和`u2`都会在`name`属性的值中填充`None`。由于这是一个 SQL NULL，ORM 会跳过将这些值包含在 INSERT 中，以便 SQL 级别的默认值生效，如果有的话，否则该值默认为数据库端的 NULL。

在 1.0 版本中作为关于没有预先存在值的属性的属性事件和其他操作的更改的一部分，这种行为被改进，以便`None`值不再填充到`__dict__`中，只是返回。除了消除 getter 操作的变异副作用外，这种改变还使得可以通过实际分配`None`来将具有服务器默认值的列设置为 NULL 值，这现在与仅仅读取它有所区别。

然而，这种变化并没有考虑到集合，其中返回一个未分配的空集合意味着这个可变集合每次都会不同，并且也无法正确地适应变异操作（例如追加，添加等）调用它。虽然这种行为通常不会妨碍任何人，但最终在[#4519](https://www.sqlalchemy.org/trac/ticket/4519)中识别出了一个边缘情况，其中这个空集合可能是有害的，即当对象合并到会话中时：

```py
>>> u1 = User(id=1)  # create an empty User to merge with id=1 in the database
>>> merged1 = session.merge(
...     u1
... )  # value of merged1.addresses is unchanged from that of the DB

>>> u2 = User(id=2)  # create an empty User to merge with id=2 in the database
>>> u2.addresses
[]
>>> merged2 = session.merge(u2)  # value of merged2.addresses has been emptied in the DB
```

在上述情况下，`merged1`上的`.addresses`集合将包含已经在数据库中的所有`Address()`对象。`merged2`不会；因为它有一个隐式分配的空列表，`.addresses`集合将被擦除。这是一个例子，说明这种变异副作用实际上可以改变数据库本身。

虽然考虑过属性系统是否应开始使用严格的“纯 Python”行为，在所有情况下为非存在属性的非持久对象引发`AttributeError`，并要求所有集合都必须显式分配，但这样的更改可能对多年来依赖此行为的大量应用程序来说过于极端，导致复杂的发布/向后兼容性问题，以及恢复旧行为的解决方法可能会变得普遍，从而使整个更改在任何情况下都变得无效。

然后，更改是保持默认的生成行为，但最终使标量的非变异行为对集合也成为现实，通过在集合系统中添加额外的机制。访问空属性时，将创建新集合并与状态关联，但直到实际发生变异才添加到`__dict__`中：

```py
>>> u1 = User()
>>> l1 = u1.addresses  # new list is created, associated with the state
>>> assert u1.addresses is l1  # you get the same list each time you access it
>>> assert (
...     "addresses" not in u1.__dict__
... )  # but it won't go into __dict__ until it's mutated
>>> from sqlalchemy import inspect
>>> inspect(u1).attrs.addresses.history
History(added=None, unchanged=None, deleted=None)
```

当列表发生更改时，它将成为要持久化到数据库的已跟踪更改的一部分：

```py
>>> l1.append(Address())
>>> assert "addresses" in u1.__dict__
>>> inspect(u1).attrs.addresses.history
History(added=[<__main__.Address object at 0x7f49b725eda0>], unchanged=[], deleted=[])
```

预计这种更改几乎不会对现有应用程序产生任何影响，除了观察到一些应用程序可能依赖于对该集合的隐式分配，例如根据其`__dict__`断定对象包含某些值：

```py
>>> u1 = User()
>>> u1.addresses
[]
# this will now fail, would pass before
>>> assert {k: v for k, v in u1.__dict__.items() if not k.startswith("_")} == {
...     "addresses": []
... }
```

或者确保集合不需要进行延迟加载以继续进行，下面（尽管有些笨拙）的代码现在也会失败：

```py
>>> u1 = User()
>>> u1.addresses
[]
>>> s.add(u1)
>>> s.flush()
>>> s.close()
>>> u1.addresses  # <-- will fail, .addresses is not loaded and object is detached
```

依赖于集合的隐式变异行为的应用程序将需要更改，以便显式分配所需的集合：

```py
>>> u1.addresses = []
```

[#4519](https://www.sqlalchemy.org/trac/ticket/4519)  ### “新实例与现有标识冲突”错误现在是一个警告

SQLAlchemy 一直具有逻辑来检测要插入的`Session`中的对象是否具有与已存在对象相同的主键：

```py
class Product(Base):
    __tablename__ = "product"

    id = Column(Integer, primary_key=True)

session = Session(engine)

# add Product with primary key 1
session.add(Product(id=1))
session.flush()

# add another Product with same primary key
session.add(Product(id=1))
s.commit()  # <-- will raise FlushError
```

更改是将`FlushError`更改为仅作为警告：

```py
sqlalchemy/orm/persistence.py:408: SAWarning: New instance <Product at 0x7f1ff65e0ba8> with identity key (<class '__main__.Product'>, (1,), None) conflicts with persistent instance <Product at 0x7f1ff60a4550>
```

在此之后，该条件将尝试将行插入数据库，这将引发`IntegrityError`，这是与`Session`中尚不存在主键标识时引发的相同错误：

```py
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) UNIQUE constraint failed: product.id
```

为了允许使用`IntegrityError`的代码捕获重复项而无需考虑`Session`的现有状态，通常使用保存点来实现：

```py
# add another Product with same primary key
try:
    with session.begin_nested():
        session.add(Product(id=1))
except exc.IntegrityError:
    print("row already exists")
```

以前的逻辑并不完全可行，因为如果具有现有标识的`Product`对象已经在`Session`中，代码还必须捕获`FlushError`，此外，该错误也没有针对完整性问题的特定条件进行过滤。通过更改，上述块的行为与发出警告的异常一致。

由于涉及主键的逻辑，所有数据库在插入时发生主键冲突时都会发出完整性错误。不会引发错误的情况是极为罕见的，即定义了一个在映射的可选择项上定义了一个比实际配置的数据库模式更严格的主键的映射，例如在映射到表的连接或在定义附加列作为复合主键的一部分时，这些列实际上在数据库模式中没有约束。然而，这些情况也更一致地工作，即使现有标识仍然存在于数据库中，插入理论上也会继续进行。警告也可以使用 Python 警告过滤器配置为引发异常。

[#4662](https://www.sqlalchemy.org/trac/ticket/4662)  ### 持久性相关级联操作不允许与 viewonly=True 

当使用`relationship.viewonly`标志将`relationship()`设置为`viewonly=True`时，表示此关系仅用于从数据库加载数据，不应进行变异或参与持久化操作。为了确保此约定成功运行，关系不能再指定在“viewonly”方面毫无意义的`relationship.cascade`设置。

这里的主要目标是“delete, delete-orphan”级联，即使`viewonly`为 True，通过 1.3 仍会影响持久性，这是一个错误；即使`viewonly`为 True，如果删除父对象或分离对象，对象仍会级联这两个操作到相关对象。而不是修改级联操作以检查`viewonly`，这两者的配置被简单地禁止在一起：

```py
class User(Base):
    # ...

    # this is now an error
    addresses = relationship("Address", viewonly=True, cascade="all, delete-orphan")
```

上述将引发：

```py
sqlalchemy.exc.ArgumentError: Cascade settings
"delete, delete-orphan, merge, save-update" apply to persistence
operations and should not be combined with a viewonly=True relationship.
```

从 SQLAlchemy 1.3.12 开始，存在此问题的应用程序应发出警告，对于上述错误，解决方案是删除视图关系的级联设置。

[#4993](https://www.sqlalchemy.org/trac/ticket/4993) [#4994](https://www.sqlalchemy.org/trac/ticket/4994)  ### 使用自定义查询查询继承映射时更严格的行为

此更改适用于查询已完成的 SELECT 子查询以选择的连接或单表继承子类实体的情况。如果给定的子查询返回的行不对应于请求的多态标识或标识，则会引发错误。在以前的情况下，这种条件在连接表继承下会悄悄通过，返回一个无效的子类，在单表继承下，`Query`会添加额外的条件来限制结果，这可能会不当地干扰查询的意图。

给定`Employee`，`Engineer(Employee)`，`Manager(Employee)`的映射示例，在 1.3 系列中，如果我们针对连接继承映射发出以下查询：

```py
s = Session(e)

s.add_all([Engineer(), Manager()])

s.commit()

print(s.query(Manager).select_entity_from(s.query(Employee).subquery()).all())
```

子查询选择了`Engineer`和`Manager`行，即使外部查询针对`Manager`，我们也会得到一个非`Manager`对象：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
2020-01-29 18:04:13,524 INFO sqlalchemy.engine.base.Engine ()
[<__main__.Engineer object at 0x7f7f5b9a9810>, <__main__.Manager object at 0x7f7f5b9a9750>]
```

新的行为是这种情况会引发错误：

```py
sqlalchemy.exc.InvalidRequestError: Row with identity key
(<class '__main__.Employee'>, (1,), None) can't be loaded into an object;
the polymorphic discriminator column '%(140205120401296 anon)s.type'
refers to mapped class Engineer->engineer, which is not a sub-mapper of
the requested mapped class Manager->manager
```

仅当该实体的主键列为非 NULL 时，才会引发上述错误。如果一行中没有给定实体的主键，则不会尝试构造实体。

在单继承映射的情况下，行为的变化稍微更加复杂；如果上面的`Engineer`和`Manager`被映射为单表继承，那么在 1.3 版本中将会发出以下查询，并且只返回一个`Manager`对象：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
WHERE anon_1.type IN (?)
2020-01-29 18:08:32,975 INFO sqlalchemy.engine.base.Engine ('manager',)
[<__main__.Manager object at 0x7ff1b0200d50>]
```

`Query`向子查询添加了“单表继承”条件，对其最初设置的意图进行了评论。这种行为是在 1.0 版本中添加的[#3891](https://www.sqlalchemy.org/trac/ticket/3891)，在“连接”和“单”表继承之间创建了行为不一致，并且修改了给定查询的意图，可能意图返回额外的行，其中对应于继承实体的列为 NULL，这是一个有效的用例。现在的行为等同于连接表继承的行为，假定子查询返回正确的行，如果遇到意外的多态标识，则会引发错误：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
2020-01-29 18:13:10,554 INFO sqlalchemy.engine.base.Engine ()
Traceback (most recent call last):
# ...
sqlalchemy.exc.InvalidRequestError: Row with identity key
(<class '__main__.Employee'>, (1,), None) can't be loaded into an object;
the polymorphic discriminator column '%(140700085268432 anon)s.type'
refers to mapped class Engineer->employee, which is not a sub-mapper of
the requested mapped class Manager->employee
```

对于上述情况的正确调整是在 1.3 上运行的，调整给定的子查询以根据鉴别器列正确过滤行：

```py
print(
    s.query(Manager)
    .select_entity_from(
        s.query(Employee).filter(Employee.discriminator == "manager").subquery()
    )
    .all()
)
```

```py
SELECT  anon_1.type  AS  anon_1_type,  anon_1.id  AS  anon_1_id
FROM  (SELECT  employee.type  AS  type,  employee.id  AS  id
FROM  employee
WHERE  employee.type  =  ?)  AS  anon_1
2020-01-29  18:14:49,770  INFO  sqlalchemy.engine.base.Engine  ('manager',)
[<__main__.Manager  object  at  0x7f70e13fca90>]
```

[#5122](https://www.sqlalchemy.org/trac/ticket/5122)  ### 查询返回的“KeyedTuple”对象被“Row”替换

如在 RowProxy 不再是“代理”；现在被称为 Row，并且表现得像一个增强的命名元组中所讨论的，Core `RowProxy` 对象现在被一个名为`Row`的类所取代。基本的`Row`对象现在更像一个命名元组，因此它现在被用作由`Query`对象返回的类似元组的结果的基础，而不是以前的“KeyedTuple”类。

这样做的理由是到 SQLAlchemy 2.0，Core 和 ORM SELECT 语句将使用行为类似命名元组的相同`Row`对象返回结果行。通过`Row`的`Row._mapping`属性可以获得类似字典的功能。在此期间，Core 结果集将使用一个`Row`子类 `LegacyRow`，它保持了以前的字典/元组混合行为以实现向后兼容性，而`Row`类将直接用于由`Query`对象返回的 ORM 元组结果。

已经努力使`Row`的大部分功能集在 ORM 中可用，这意味着可以通过字符串名称以及实体/列进行访问：

```py
row = s.query(User, Address).join(User.addresses).first()

row._mapping[User]  # same as row[0]
row._mapping[Address]  # same as row[1]
row._mapping["User"]  # same as row[0]
row._mapping["Address"]  # same as row[1]

u1 = aliased(User)
row = s.query(u1).only_return_tuples(True).first()
row._mapping[u1]  # same as row[0]

row = s.query(User.id, Address.email_address).join(User.addresses).first()

row._mapping[User.id]  # same as row[0]
row._mapping["id"]  # same as row[0]
row._mapping[users.c.id]  # same as row[0]
```

另请参阅

RowProxy 不再是“代理”；现在被称为 Row，并且表现得像一个增强的命名元组

[#4710](https://www.sqlalchemy.org/trac/ticket/4710).

### Session 新的“autobegin”行为特性

以前，在其默认模式`autocommit=False`下，`Session`会在构造时立即内部开始一个`SessionTransaction`对象，并且在每次调用`Session.rollback()`或`Session.commit()`后还会创建一个新的对象。

新的行为是，只有在调用诸如`Session.add()`或`Session.execute()`等方法时，才会按需创建此`SessionTransaction`对象。然而，现在也可以显式调用`Session.begin()`来开始事务，即使在`autocommit=False`模式下，从而与未来风格的`_base.Connection`的行为相匹配。

这些变化所指示的行为变化是：

+   现在，`Session`可以处于未开始任何事务的状态，即使在`autocommit=False`模式下。以前，这种状态仅在“自动提交”模式下可用。

+   在这种状态下，`Session.commit()`和`Session.rollback()`方法是无操作的。依赖这些方法使所有对象过期的代码应明确使用`Session.begin()`或`Session.expire_all()`来适应其用例。

+   当`Session`被创建时，或在`Session.rollback()`或`Session.commit()`完成后，`SessionEvents.after_transaction_create()`事件钩子不会立即触发。

+   `Session.close()`方法也不意味着隐式开始新的`SessionTransaction`。

另请参阅

自动开始

#### 理由

`Session`对象的默认行为`autocommit=False`历史上意味着始终有一个与`Session`相关联的`SessionTransaction`对象，通过`Session.transaction`属性关联。当给定的`SessionTransaction`完成时，由于提交、回滚或关闭，它会立即被新的替换。`SessionTransaction`本身并不意味着使用任何连接相关资源，因此这种长期存在的行为具有特定的优雅之处，即`Session.transaction`的状态始终可预测为非 None。

但是，作为减少引用循环的倡议的一部分[#5056](https://www.sqlalchemy.org/trac/ticket/5056)，这意味着调用`Session.close()`会导致一个仍然存在引用循环且更昂贵的`Session`对象，清理起来更加费力，更不用说构造`SessionTransaction`对象时会有一些额外开销，这意味着对于一个例如调用了`Session.commit()`然后又调用`Session.close()`的`Session`会产生不必要的开销。

因此，决定`Session.close()`应该将`self.transaction`的内部状态，现在在内部称为`self._transaction`，保留为 None，并且只有在需要时才创建新的`SessionTransaction`。为了保持一致性和代码覆盖率，这种行为也扩展到了所有“autobegin”预期的点，而不仅仅是调用`Session.close()`时。

特别是，这对订阅`SessionEvents.after_transaction_create()`事件钩子的应用程序造成了行为上的改变；以前，当首次构建`Session`时，此事件将被触发，以及对关闭先前事务的大多数操作，并将触发`SessionEvents.after_transaction_end()`。新行为是，当`Session`尚未创建新的`SessionTransaction`对象并且映射对象通过`Session.add()`和`Session.delete()`等方法与`Session`关联时，当调用`Session.transaction`属性时，当`Session.flush()`方法有任务要完成时等，`SessionEvents.after_transaction_create()`会按需触发。

此外，依赖于`Session.commit()`或`Session.rollback()`方法无条件使所有对象过期的代码将不再能够这样做。当没有发生任何更改时需要使所有对象过期的代码应该在这种情况下调用`Session.expire_all()`。

除了`SessionEvents.after_transaction_create()`事件发出的时间变化以及`Session.commit()`或`Session.rollback()`的无操作性质之外，这种变化对`Session`对象的行为应该没有其他用户可见的影响；`Session`在调用`Session.close()`后仍然保持可用于新操作的行为，并且`Session`与`Engine`以及数据库本身的交互顺序也应该保持不受影响，因为这些操作已经以按需方式运行。

[#5074](https://www.sqlalchemy.org/trac/ticket/5074)

#### 理由

`Session` 对象的默认行为`autocommit=False`在历史上意味着始终存在一个与`Session`相关联的`SessionTransaction`对象，通过`Session.transaction`属性关联。当给定的`SessionTransaction`完成时，由于提交、回滚或关闭，它会立即被新的替代。`SessionTransaction`本身并不意味着使用任何连接相关资源，因此这种长期存在的行为具有特定的优雅之处，即`Session.transaction`的状态始终可预测为非 None。

但是，作为在[#5056](https://www.sqlalchemy.org/trac/ticket/5056)中极大减少引用循环的倡议的一部分，这意味着调用`Session.close()`会导致一个仍然存在引用循环且更昂贵清理的`Session`对象，更不用说构造`SessionTransaction`对象时会有一些小的开销，这意味着对于一个例如调用了`Session.commit()`然后调用`Session.close()`的`Session`会产生不必要的开销。

因此，决定让`Session.close()`将`self.transaction`的内部状态，现在在内部称为`self._transaction`，保留为 None，并且只在需要时创建一个新的`SessionTransaction`。为了一致性和代码覆盖率，这种行为也扩展到所有“自动开始”预期的点，不仅仅是在调用`Session.close()`时。

特别是，这会对订阅 `SessionEvents.after_transaction_create()` 事件钩子的应用程序造成行为上的改变；以前，此事件将在首次构造 `Session` 时被触发，以及对关闭上一个事务的大多数动作进行操作，并会发出 `SessionEvents.after_transaction_end()`。新行为是，当 `Session` 尚未创建新的 `SessionTransaction` 对象，并且映射对象通过 `Session.add()` 和 `Session.delete()` 等方法与 `Session` 关联时，以及调用 `Session.transaction` 属性时，当 `Session.flush()` 方法有任务需要完成等情况下，将按需触发 `SessionEvents.after_transaction_create()`。

此外，依赖于`Session.commit()` 或 `Session.rollback()` 方法来无条件使所有对象过期的代码将不再能够这样做。当没有发生变化时需要使所有对象过期的代码应该调用`Session.expire_all()`。

除了`SessionEvents.after_transaction_create()`事件的触发时间发生变化以及`Session.commit()`或`Session.rollback()`的无操作性质外，这一变化不应对`Session`对象的行为产生其他用户可见的影响；`Session`在调用`Session.close()`后仍然保持可用于新操作的行为，并且`Session`与`Engine`以及数据库本身的交互顺序也应保持不受影响，因为这些操作已经以按需方式运行。

[#5074](https://www.sqlalchemy.org/trac/ticket/5074)

### 只读视图关系不同步反向引用

在 1.3.14 中的[#5149](https://www.sqlalchemy.org/trac/ticket/5149)中，SQLAlchemy 开始在目标关系上同时使用`relationship.backref`或`relationship.back_populates`关键字时发出警告，与`relationship.viewonly`标志一起使用。这是因为“只读”关系实际上不会持久保存对其所做的更改，这可能导致一些误导性行为发生。然而，在[#5237](https://www.sqlalchemy.org/trac/ticket/5237)中，我们试图优化这种行为，因为在只读关系上设置反向引用是有合法用例的，包括反向填充属性有时被关系懒加载器用来确定在另一个方向上不需要额外的急加载，以及反向填充可以用于映射器内省，`backref()`也可以是设置双向关系的便捷方式。

那时的解决方案是使从反向引用发生的“变化”成为可选的事情，使用`relationship.sync_backref`标志。在 1.4 中，对于还设置了`relationship.viewonly`的关系目标，默认情况下`relationship.sync_backref`的值为 False。这表示对于具有 viewonly 的关系所做的任何更改都不会影响另一侧或`Session`的状态：

```py
class User(Base):
    # ...

    addresses = relationship(Address, backref=backref("user", viewonly=True))

class Address(Base): ...

u1 = session.query(User).filter_by(name="x").first()

a1 = Address()
a1.user = u1
```

上面，`a1`对象**不会**被添加到`u1.addresses`集合中，也不会将`a1`对象添加到会话中。以前，这两件事情都是正确的。当`relationship.viewonly`为`False`时，不再发出警告，即`relationship.sync_backref`应设置为`False`，因为这现在是默认行为。

[#5237](https://www.sqlalchemy.org/trac/ticket/5237)

### 在 2.0 版本中将删除 cascade_backrefs 行为

SQLAlchemy 长期以来一直有一个行为，根据反向引用赋值将对象级联到`Session`中。给定下面的`User`已经在`Session`中，将其分配给`Address`对象的`Address.user`属性，假设已建立双向关系，这意味着在那一点上`Address`也会被放入`Session`中：

```py
u1 = User()
session.add(u1)

a1 = Address()
a1.user = u1  # <--- adds "a1" to the Session
```

上述行为是反向引用行为的一个意外副作用，因为`a1.user`意味着`u1.addresses.append(a1)`，`a1`会被级联到`Session`中。这在 1.4 版本中仍然是默认行为。在某个时候，添加了一个新标志`relationship.cascade_backrefs`来禁用上述行为，以及`backref.cascade_backrefs`来在通过`relationship.backref`指定关系时设置此行为，因为这可能会令人惊讶，也会妨碍一些操作，其中对象会过早地放置在`Session`中并提前刷新。

在 2.0 版本中，默认行为将是“cascade_backrefs”为 False，并且另外不会有“True”行为，因为这通常不是一种理想的行为。当启用 2.0 版本的弃用警告时，当“backref cascade”实际发生时将发出警告。要获得新行为，可以在任何目标关系上将`relationship.cascade_backrefs`和`backref.cascade_backrefs`设置为`False`，就像在 1.3 版本和更早版本中已经支持的那样，或者使用`Session.future`标志进入 2.0 风格模式：

```py
Session = sessionmaker(engine, future=True)

with Session() as session:
    u1 = User()
    session.add(u1)

    a1 = Address()
    a1.user = u1  # <--- will not add "a1" to the Session
```

[#5150](https://www.sqlalchemy.org/trac/ticket/5150)

### 在取消过期操作期间急切加载器发出

长期以来一直寻求的行为是，当访问一个过期对象时，配置的急切加载器将运行，以便在对象被刷新或其他情况下取消过期时急切加载过期对象上的关系。现在已经添加了这种行为，因此 joinedloaders 将像往常一样添加内联 JOIN，而 selectin/subquery loaders 将在过期对象被取消过期或对象被刷新时运行“immediateload”操作：

```py
>>> a1 = session.query(A).options(joinedload(A.bs)).first()
>>> a1.data = "new data"
>>> session.commit()
```

在上面的例子中，`A`对象使用了`joinedload()`选项加载，以便急切加载`bs`集合。在`session.commit()`后，对象的状态会过期。访问`.data`列属性时，对象会被刷新，现在这将包括 joinedload 操作：

```py
>>> a1.data
SELECT  a.id  AS  a_id,  a.data  AS  a_data,  b_1.id  AS  b_1_id,  b_1.a_id  AS  b_1_a_id
FROM  a  LEFT  OUTER  JOIN  b  AS  b_1  ON  a.id  =  b_1.a_id
WHERE  a.id  =  ? 
```

该行为适用于直接应用于`relationship()`的加载策略，以及与`Query.options()`一起使用的选项，前提是对象最初是由该查询��载的。

对于“secondary”急加载器“selectinload”和“subqueryload”，这些加载器的 SQL 策略并不是必要的，以便在单个对象上急加载属性；因此它们将在刷新场景中调用“immediateload”策略，这类似于“lazyload”发出的查询，作为额外的查询：

```py
>>> a1 = session.query(A).options(selectinload(A.bs)).first()
>>> a1.data = "new data"
>>> session.commit()
>>> a1.data
SELECT  a.id  AS  a_id,  a.data  AS  a_data
FROM  a
WHERE  a.id  =  ?
(1,)
SELECT  b.id  AS  b_id,  b.a_id  AS  b_a_id
FROM  b
WHERE  ?  =  b.a_id
(1,) 
```

请注意，加载器选项不适用于以不同方式引入到`Session`中的对象。也就是说，如果`a1`对象只是在这个`Session`中被持久化，或者在应用急加载选项之前用不同的查询加载了该对象，那么该对象就没有与之关联的急加载选项。这并不是一个新概念，但是寻找刷新行为上的急加载的用户可能会发现这更加明显。

[#1763](https://www.sqlalchemy.org/trac/ticket/1763)

### 列加载器如`deferred()`、`with_expression()` 只在最外层、完整的实体查询中指定时才生效

注意

这个变更说明在此文档的早期版本中并不存在，但对于所有 SQLAlchemy 1.4 版本都是相关的。

一个在 1.3 版本和之前版本中从未支持过的行为，但仍然会产生特定效果的是重新利用列加载器选项，比如`defer()`和`with_expression()` 在子查询中，以控制哪些 SQL 表达式将出现在每个子查询的列子句中。一个典型的例子是构造 UNION 查询，例如：

```py
q1 = session.query(User).options(with_expression(User.expr, literal("u1")))
q2 = session.query(User).options(with_expression(User.expr, literal("u2")))

q1.union_all(q2).all()
```

在 1.3 版本中，`with_expression()` 选项会对 UNION 的每个元素生效，例如：

```py
SELECT  anon_1.anon_2  AS  anon_1_anon_2,  anon_1.user_account_id  AS  anon_1_user_account_id,
anon_1.user_account_name  AS  anon_1_user_account_name
FROM  (
  SELECT  ?  AS  anon_2,  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  ?  AS  anon_3,  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_1
('u1',  'u2')
```

SQLAlchemy 1.4 对加载器选项的概念变得更加严格，因此仅应用于**查询的最外层部分**，即用于填充实际要返回的 ORM 实体的 SELECT；在 1.4 中上述查询将产生：

```py
SELECT  ?  AS  anon_1,  anon_2.user_account_id  AS  anon_2_user_account_id,
anon_2.user_account_name  AS  anon_2_user_account_name
FROM  (
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_2
('u1',)
```

也就是说，`Query` 的选项是从 UNION 的第一个元素中获取的，因为所有加载器选项只能在最顶层。第二个查询的选项被忽略。

#### 理由

这种行为现在更接近于其他类型的加载器选项，比如在所有 SQLAlchemy 版本中，1.3 版本及更早版本中的关系加载器选项`joinedload()`，在 UNION 情况下已经被复制到查询的最顶层，并且仅从 UNION 的第一个元素中获取，丢弃查询其他部分的任何选项。

上面演示的这种隐式复制和选择性忽略选项的行为，是一个仅在`Query`中存在的遗留行为，也是一个特定的例子，说明了`Query`及其应用`Query.union_all()`的方式存在缺陷，因为如何将单个 SELECT 转换为自身和另一个查询的 UNION 以及如何应用加载器选项到该新语句是模棱两可的。

SQLAlchemy 1.4 的行为可以被证明在更常见的使用`defer()`的情况下，比 1.3 更为优越。以下查询：

```py
q1 = session.query(User).options(defer(User.name))
q2 = session.query(User).options(defer(User.name))

q1.union_all(q2).all()
```

在 1.3 版本中会尴尬地向内部查询中添加 NULL，然后再 SELECT 它：

```py
SELECT  anon_1.anon_2  AS  anon_1_anon_2,  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
  UNION  ALL
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
)  AS  anon_1
```

如果所有查询没有设置相同的选项，上述情况将由于无法形成正确的 UNION 而引发错误。

而在 1.4 版本中，该选项仅应用于顶层，省略了对 `User.name` 的提取，避免了这种复杂性：

```py
SELECT  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_1
```

#### 正确的方法

使用 2.0 风格查询，目前不会发出警告，但是嵌套的`with_expression()`选项始终被忽略，因为它们不适用于正在加载的实体，并且不会被隐式复制到任何地方。下面的查询对`with_expression()`调用不会产生任何输出：

```py
s1 = select(User).options(with_expression(User.expr, literal("u1")))
s2 = select(User).options(with_expression(User.expr, literal("u2")))

stmt = union_all(s1, s2)

session.scalars(select(User).from_statement(stmt)).all()
```

生成 SQL：

```py
SELECT  user_account.id,  user_account.name
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name
FROM  user_account
```

要正确应用`with_expression()`到 `User` 实体，应该将其应用于查询的最外层，使用普通的 SQL 表达式在每个 SELECT 的列子句中：

```py
s1 = select(User, literal("u1").label("some_literal"))
s2 = select(User, literal("u2").label("some_literal"))

stmt = union_all(s1, s2)

session.scalars(
    select(User)
    .from_statement(stmt)
    .options(with_expression(User.expr, stmt.selected_columns.some_literal))
).all()
```

这将产生预期的 SQL：

```py
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
```

`User` 对象本身将在 `User.expr` 下的内容中包含此表达式。

#### 理由

这种行为现在更接近于其他种类的加载器选项，比如关系加载器选项，比如`joinedload()`在所有 SQLAlchemy 版本中，包括 1.3 和更早的版本，这在 UNION 情况下已经复制到查询的最顶层，并且只从 UNION 的第一个元素中获取，丢弃查询其他部分的任何选项。

上面演示的这种隐式复制和选择性忽略选项的行为是一种遗留行为，仅属于`Query`的一部分，并且是一个特殊的例子，展示了`Query`及其应用`Query.union_all()`的方式存在缺陷，因为不清楚如何将单个 SELECT 转换为自身和另一个查询的 UNION，以及如何应用加载器选项到新语句。

对于更常见的情况，使用`defer()`，演示了 SQLAlchemy 1.4 的行为通常优于 1.3。以下查询：

```py
q1 = session.query(User).options(defer(User.name))
q2 = session.query(User).options(defer(User.name))

q1.union_all(q2).all()
```

在 1.3 中会尴尬地向内部查询添加 NULL，然后 SELECT 它：

```py
SELECT  anon_1.anon_2  AS  anon_1_anon_2,  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
  UNION  ALL
  SELECT  NULL  AS  anon_2,  user_account.id  AS  user_account_id
  FROM  user_account
)  AS  anon_1
```

如果所有查询没有设置相同的选项，上述情况将由于无法形成正确的 UNION 而引发错误。

而在 1.4 中，该选项仅应用于顶层，省略了对`User.name`的提取，避免了这种复杂性：

```py
SELECT  anon_1.user_account_id  AS  anon_1_user_account_id
FROM  (
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
  UNION  ALL
  SELECT  user_account.id  AS  user_account_id,  user_account.name  AS  user_account_name
  FROM  user_account
)  AS  anon_1
```

#### 正确的方法

使用 2.0 风格查询，目前不会发出警告，但是嵌套的`with_expression()`选项始终被忽略，因为它们不适用于正在加载的实体，并且不会被隐式复制到任何地方。下面的查询对`with_expression()`调用不会产生任何输出：

```py
s1 = select(User).options(with_expression(User.expr, literal("u1")))
s2 = select(User).options(with_expression(User.expr, literal("u2")))

stmt = union_all(s1, s2)

session.scalars(select(User).from_statement(stmt)).all()
```

生成 SQL：

```py
SELECT  user_account.id,  user_account.name
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name
FROM  user_account
```

要正确应用`with_expression()`到`User`实体，应该应用到查询的最外层，使用普通的 SQL 表达式放在每个 SELECT 的 columns 子句中：

```py
s1 = select(User, literal("u1").label("some_literal"))
s2 = select(User, literal("u2").label("some_literal"))

stmt = union_all(s1, s2)

session.scalars(
    select(User)
    .from_statement(stmt)
    .options(with_expression(User.expr, stmt.selected_columns.some_literal))
).all()
```

这将产生预期的 SQL：

```py
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
UNION  ALL
SELECT  user_account.id,  user_account.name,  ?  AS  some_literal
FROM  user_account
```

`User`对象本身将包含这个表达式在它们的内容中`User.expr`下面。

### 在瞬态对象上访问未初始化的集合属性不再改变 __dict__

对于新创建的对象访问映射属性始终返回隐式生成的值，而不是引发`AttributeError`，例如标量属性返回`None`或列表关系返回`[]`：

```py
>>> u1 = User()
>>> u1.name
None
>>> u1.addresses
[]
```

以上行为的理由最初是为了使 ORM 对象更易于使用。由于 ORM 对象在首次创建时表示一个空行而没有任何状态，因此直观地，其未访问的属性应该解析为标量的`None`（或 SQL NULL），并对关系解析为空集合。特别是，这使得一种极其常见的模式成为可能，即能够在不手动创建和分配空集合的情况下对新集合进行变异：

```py
>>> u1 = User()
>>> u1.addresses.append(Address())  # no need to assign u1.addresses = []
```

直到 SQLAlchemy 的 1.0 版本，这种初始化系统对标量属性以及集合的行为都是将`None`或空集合*填充*到对象的状态中，例如`__dict__`。这意味着以下两个操作是等效的：

```py
>>> u1 = User()
>>> u1.name = None  # explicit assignment

>>> u2 = User()
>>> u2.name  # implicit assignment just by accessing it
None
```

在上述情况下，`u1`和`u2`的`name`属性的值都将填充为`None`。由于这是一个 SQL NULL，ORM 将跳过将这些值包含在 INSERT 中，以便发生 SQL 级别的默认值，如果有的话，否则值将在数据库端默认为 NULL。

在版本 1.0 中作为关于没有预先存在值的属性的属性事件和其他操作的更改的一部分，这种行为被调整，以便`None`值不再填充到`__dict__`中，只是返回。除了消除获取器操作的变异副作用外，这种变化还使得可以将具有服务器默认值的列设置为 NULL 值，方法是实际分配`None`，这现在与仅仅读取它有所区别。

但是这种变化并没有考虑到集合，其中返回一个未分配的空集合意味着这个可变集合每次都会不同，也无法正确地适应对其进行的变异操作（例如追加、添加等）。虽然这种行为通常不会影响任何人，但最终在[#4519](https://www.sqlalchemy.org/trac/ticket/4519)中识别出了一个边缘情况，即当对象合并到会话中时，这个空集合可能会有害：

```py
>>> u1 = User(id=1)  # create an empty User to merge with id=1 in the database
>>> merged1 = session.merge(
...     u1
... )  # value of merged1.addresses is unchanged from that of the DB

>>> u2 = User(id=2)  # create an empty User to merge with id=2 in the database
>>> u2.addresses
[]
>>> merged2 = session.merge(u2)  # value of merged2.addresses has been emptied in the DB
```

在上述情况下，`merged1`上的`.addresses`集合将包含数据库中已经存在的所有`Address()`对象。`merged2`不会；因为它有一个隐式分配的空列表，`.addresses`集合将被擦除。这是一个实际上可以改变数据库本身的变异副作用的示例。

虽然考虑过属性系统是否应开始使用严格的“纯 Python”行为，在所有情况下对非存在属性的非持久对象引发`AttributeError`，并要求所有集合都必须显式分配，但这样的改变可能对多年来依赖于这种行为的大量应用程序来说过于极端，导致复杂的发布/向后兼容性问题，以及恢复旧行为的解决方法可能会变得普遍，从而使整个改变失效。

改变的是保持默认的生成行为，但最终使标量的非变异行为对集合也成为现实，通过在集合系统中添加额外的机制。当访问空属性时，新集合将被创建并与状态关联，但直到实际发生变异才会被添加到`__dict__`中：

```py
>>> u1 = User()
>>> l1 = u1.addresses  # new list is created, associated with the state
>>> assert u1.addresses is l1  # you get the same list each time you access it
>>> assert (
...     "addresses" not in u1.__dict__
... )  # but it won't go into __dict__ until it's mutated
>>> from sqlalchemy import inspect
>>> inspect(u1).attrs.addresses.history
History(added=None, unchanged=None, deleted=None)
```

当列表发生变化时，它将成为要持久化到数据库的跟踪更改的一部分：

```py
>>> l1.append(Address())
>>> assert "addresses" in u1.__dict__
>>> inspect(u1).attrs.addresses.history
History(added=[<__main__.Address object at 0x7f49b725eda0>], unchanged=[], deleted=[])
```

这种改变预计对现有应用程序几乎没有任何影响，除了观察到一些应用程序可能依赖于对该集合的隐式赋值，例如根据其`__dict__`来断定对象包含某些值：

```py
>>> u1 = User()
>>> u1.addresses
[]
# this will now fail, would pass before
>>> assert {k: v for k, v in u1.__dict__.items() if not k.startswith("_")} == {
...     "addresses": []
... }
```

或确保集合不需要延迟加载才能继续，现在下面这段（尽管有些尴尬）代码也将失败：

```py
>>> u1 = User()
>>> u1.addresses
[]
>>> s.add(u1)
>>> s.flush()
>>> s.close()
>>> u1.addresses  # <-- will fail, .addresses is not loaded and object is detached
```

依赖于集合的隐式变异行为的应用程序需要更改，以便显式地分配所需的集合：

```py
>>> u1.addresses = []
```

[#4519](https://www.sqlalchemy.org/trac/ticket/4519)

### “新实例与现有标识冲突”错误现在是一个警告

SQLAlchemy 一直有逻辑来检测要插入`Session`中的对象是否具有与已经存在的对象相同的主键：

```py
class Product(Base):
    __tablename__ = "product"

    id = Column(Integer, primary_key=True)

session = Session(engine)

# add Product with primary key 1
session.add(Product(id=1))
session.flush()

# add another Product with same primary key
session.add(Product(id=1))
s.commit()  # <-- will raise FlushError
```

改变是`FlushError`被修改为仅作为警告：

```py
sqlalchemy/orm/persistence.py:408: SAWarning: New instance <Product at 0x7f1ff65e0ba8> with identity key (<class '__main__.Product'>, (1,), None) conflicts with persistent instance <Product at 0x7f1ff60a4550>
```

随后，该条件将尝试将行插入数据库，这将引发`IntegrityError`，这是如果主键标识在`Session`中尚不存在时将引发的相同错误：

```py
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) UNIQUE constraint failed: product.id
```

其理念是允许使用`IntegrityError`来捕获重复项的代码能够正常运行，而不受`Session`的现有状态的影响，通常使用保存点来实现：

```py
# add another Product with same primary key
try:
    with session.begin_nested():
        session.add(Product(id=1))
except exc.IntegrityError:
    print("row already exists")
```

上述逻辑在早期并不完全可行，因为在`Session`中已经存在具有现有标识的`Product`对象的情况下，代码还必须捕获`FlushError`，而这种情况又没有针对完整性问题的特定条件进行过滤。通过这次更改，上述代码块的行为与警告也会被发出的例外情况一致。

由于涉及主键的逻辑会导致所有数据库在插入时出现主键冲突时发出完整性错误。不会引发错误的情况是极为罕见的，即在映射定义了比实际配置在数据库模式中更严格的主键的情况下，例如在映射到表的连接或在定义附加列作为复合主键的一部分时，这些列实际上在数据库模式中并没有约束。然而，这些情况也更一致地工作，即使现有标识仍然存在于数据库中，插入理论上也会继续进行。警告也可以通过 Python 警告过滤器配置为引发异常。

[#4662](https://www.sqlalchemy.org/trac/ticket/4662)

### 持久化相关级联操作在 viewonly=True 时不允许

当使用`relationship.viewonly`标志将`relationship()`设置为`viewonly=True`时，表示此关系应仅用于从数据库加载数据，并且不应进行变异或参与持久化操作。为了确保此契约成功运行，关系不能再指定在“viewonly”方面毫无意义的`relationship.cascade`设置。

这里的主要目标是“delete, delete-orphan”级联，即使 viewonly 为 True，通过 1.3 仍会影响持久性，这是一个错误；即使 viewonly 为 True，如果父对象被删除或对象被分离，对象仍会将这两个操作级联到相关对象。而不是修改级联操作以检查 viewonly，这两者的配置被简单地禁止在一起：

```py
class User(Base):
    # ...

    # this is now an error
    addresses = relationship("Address", viewonly=True, cascade="all, delete-orphan")
```

上述将引发：

```py
sqlalchemy.exc.ArgumentError: Cascade settings
"delete, delete-orphan, merge, save-update" apply to persistence
operations and should not be combined with a viewonly=True relationship.
```

作为 SQLAlchemy 1.3.12 的一部分，存在此问题的应用程序应该发出警告，对于上述错误，解决方案是删除视图关系的级联设置。

[#4993](https://www.sqlalchemy.org/trac/ticket/4993) [#4994](https://www.sqlalchemy.org/trac/ticket/4994)

### 使用自定义查询查询继承映射时更严格的行为

此更改适用于查询已完成的 SELECT 子查询以选择的连接或单表继承子类实体的情况。如果给定的子查询返回与请求的多态标识或标识不对应的行，则会引发错误。以前，在连接表继承下，此条件会悄悄通过，返回一个无效的子类，并且在单表继承下，`Query`会添加额外的条件来限制结果，这可能会不当地干扰查询的意图。

鉴于`Employee`，`Engineer(Employee)`，`Manager(Employee)`的示例映射，在 1.3 系列中，如果我们针对连接继承映射发出以下查询：

```py
s = Session(e)

s.add_all([Engineer(), Manager()])

s.commit()

print(s.query(Manager).select_entity_from(s.query(Employee).subquery()).all())
```

子查询选择了`Engineer`和`Manager`行，即使外部查询针对`Manager`，我们也会得到一个非`Manager`对象：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
2020-01-29 18:04:13,524 INFO sqlalchemy.engine.base.Engine ()
[<__main__.Engineer object at 0x7f7f5b9a9810>, <__main__.Manager object at 0x7f7f5b9a9750>]
```

新行为是这种情况会引发错误：

```py
sqlalchemy.exc.InvalidRequestError: Row with identity key
(<class '__main__.Employee'>, (1,), None) can't be loaded into an object;
the polymorphic discriminator column '%(140205120401296 anon)s.type'
refers to mapped class Engineer->engineer, which is not a sub-mapper of
the requested mapped class Manager->manager
```

仅当该实体的主键列为非 NULL 时才会引发上述错误。如果行中没有给定实体的主键，则不会尝试构造实体。

在单继承映射的情况下，行为的变化稍微更为复杂；如果上述的`Engineer`和`Manager`被映射为单表继承，在 1.3 中，将发出以下查询，并且只返回一个`Manager`对象：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
WHERE anon_1.type IN (?)
2020-01-29 18:08:32,975 INFO sqlalchemy.engine.base.Engine ('manager',)
[<__main__.Manager object at 0x7ff1b0200d50>]
```

`Query`向子查询添加了“单表继承”条件，对最初设置的意图进行了评论。此行为是在版本 1.0 中添加的[#3891](https://www.sqlalchemy.org/trac/ticket/3891)，并在“连接”和“单”表继承之间创建了行为不一致，并且修改了给定查询的意图，可能意图返回列对应于继承实体的空值的其他行，这是一个有效的用例。该行为现在等同于连接表继承的行为，其中假定子查询返回正确的行，如果遇到意外的多态标识，则会引发错误：

```py
SELECT anon_1.type AS anon_1_type, anon_1.id AS anon_1_id
FROM (SELECT employee.type AS type, employee.id AS id
FROM employee) AS anon_1
2020-01-29 18:13:10,554 INFO sqlalchemy.engine.base.Engine ()
Traceback (most recent call last):
# ...
sqlalchemy.exc.InvalidRequestError: Row with identity key
(<class '__main__.Employee'>, (1,), None) can't be loaded into an object;
the polymorphic discriminator column '%(140700085268432 anon)s.type'
refers to mapped class Engineer->employee, which is not a sub-mapper of
the requested mapped class Manager->employee
```

如上所述的情况下的正确调整是调整给定的子查询，以正确根据鉴别器列过滤行：

```py
print(
    s.query(Manager)
    .select_entity_from(
        s.query(Employee).filter(Employee.discriminator == "manager").subquery()
    )
    .all()
)
```

```py
SELECT  anon_1.type  AS  anon_1_type,  anon_1.id  AS  anon_1_id
FROM  (SELECT  employee.type  AS  type,  employee.id  AS  id
FROM  employee
WHERE  employee.type  =  ?)  AS  anon_1
2020-01-29  18:14:49,770  INFO  sqlalchemy.engine.base.Engine  ('manager',)
[<__main__.Manager  object  at  0x7f70e13fca90>]
```

[#5122](https://www.sqlalchemy.org/trac/ticket/5122)

## 方言更改

### pg8000 的最低版本为 1.16.6，仅支持 Python 3

支持 pg8000 方言已经得到显着改进，得益于该项目的维护者。

由于 API 更改，pg8000 方言现在需要版本 1.16.6 或更高版本。从 1.13 系列开始，pg8000 系列已经放弃了 Python 2 支持。需要 pg8000 的 Python 2 用户应确保他们的要求被固定在 `SQLAlchemy<1.4`。

[#5451](https://www.sqlalchemy.org/trac/ticket/5451)

### PostgreSQL psycopg2 方言需要 psycopg2 版本 2.7 或更高版本。

psycopg2 方言依赖于过去几年中发布的许多 psycopg2 特性。为了简化方言，现在要求的最低版本是 2017 年 3 月发布的版本 2.7。

### psycopg2 方言不再对绑定参数名称有限制

SQLAlchemy 1.3 无法适应在 psycopg2 方言下包含百分号或括号的绑定参数名称。这反过来意味着包含这些字符的列名也有问题，因为 INSERT 和其他 DML 语句会生成与列名匹配的参数名称，这将导致失败。解决方法是利用 `Column.key` 参数，以便生成用于生成参数的替代名称，或者必须在 `create_engine()` 级别更改方言的参数样式。从 SQLAlchemy 1.4.0beta3 开始，所有命名限制都已被移除，并且在所有情况下参数都被完全转义，因此这些解决方法不再必要。

[#5941](https://www.sqlalchemy.org/trac/ticket/5941)

[#5653](https://www.sqlalchemy.org/trac/ticket/5653)  ### psycopg2 方言默认使用“execute_values”和 RETURNING 来进行 INSERT 语句

使用 Core 和 ORM 时，PostgreSQL 的一个重要性能增强的前半部分，psycopg2 方言现在默认使用 `psycopg2.extras.execute_values()` 来编译 INSERT 语句，并在此模式下实现了 RETURNING 支持。这一变化的另一半是 ORM 批量插入现在在大多数情况下使用带有 RETURNING 的 psycopg2 批量语句，这使得 ORM 能够利用 executemany（即批量插入语句）的 RETURNING，因此使用 psycopg2 进行的 ORM 批量插入速度提高了 400%，具体取决于具体情况。

此扩展方法允许在单个语句中插入多行，使用语句的扩展 VALUES 子句。虽然 SQLAlchemy 的`insert()`构造已经通过`Insert.values()`方法支持此语法，但扩展方法允许在执行语句时动态构建 VALUES 子句，这是当将参数字典列表传递给`Connection.execute()`时发生的“executemany”执行。它还发生在缓存边界之外，以便在渲染 VALUES 之前可以缓存 INSERT 语句。

在性能示例套件中使用`bulk_inserts.py`脚本快速测试`execute_values()`方法，显示出约**五倍的性能提升**：

```py
$ python -m examples.performance bulk_inserts --test test_core_insert --num 100000 --dburl postgresql://scott:tiger@localhost/test

# 1.3
test_core_insert : A single Core INSERT construct inserting mappings in bulk. (100000 iterations); total time 5.229326 sec

# 1.4
test_core_insert : A single Core INSERT construct inserting mappings in bulk. (100000 iterations); total time 0.944007 sec
```

“batch”扩展的支持是在版本 1.2 中添加的，在支持批处理模式/快速执行助手，并在 1.3 中增强以支持`execute_values`扩展在[#4623](https://www.sqlalchemy.org/trac/ticket/4623)中。在 1.4 中，`execute_values`扩展现在默认为 INSERT 语句打开；UPDATE 和 DELETE 的“batch”扩展默认关闭。

此外，`execute_values`扩展函数支持将由 RETURNING 生成的行作为聚合列表返回。如果给定的`insert()`构造请求通过`Insert.returning()`方法或类似用于返回生成默认值的方法来返回，psycopg2 方言现在将检索此列表；然后将这些行安装在结果中，以便它们被检索为直接来自游标。这允许 ORM 等工具在所有情况下使用批量插入，预计将提供显著的性能改进。

psycopg2 方言的`executemany_mode`功能已经进行了以下更改：

+   添加了一个新模式`"values_only"`。此模式使用非常高效的`psycopg2.extras.execute_values()`扩展方法来运行使用 executemany()的编译 INSERT 语句，但不使用`execute_batch()`来运行 UPDATE 和 DELETE 语句。这种新模式现在是 psycopg2 方言的默认设置。

+   现有的`"values"`模式现在被命名为`"values_plus_batch"`。此模式将使用`execute_values`进行 INSERT 语句，使用`execute_batch`进行 UPDATE 和 DELETE 语句。该模式默认未启用，因为它会禁用使用`executemany()`执行 UPDATE 和 DELETE 语句时的`cursor.rowcount`的正确功能。

+   对于 INSERT 语句，启用了`"values_only"`和`"values"`的 RETURNING 支持。psycopg2 方言将使用 fetch=True 标志从 psycopg2 接收行，并将它们安装到结果集中，就好像它们直接来自游标（尽管最终确实是这样，但是 psycopg2 的扩展函数已经将多个批次聚合成一个列表）。

+   `execute_values`的默认“page_size”设置从 100 增加到 1000。`execute_batch`函数的默认值仍为 100。这些参数可以像以前一样进行修改。

+   1.2 版本功能中的`use_batch_mode`标志已被移除；行为仍可通过 1.3 中添加的`executemany_mode`标志进行控制。

+   核心引擎和方言已经增强，以支持`executemany`加上返回模式，目前仅适用于 psycopg2，通过提供新的`CursorResult.inserted_primary_key_rows`和`CursorResult.returned_default_rows`访问器。

另请参见

Psycopg2 快速执行助手

[#5401](https://www.sqlalchemy.org/trac/ticket/5401)  ### 从 SQLite 方言中删除了“连接重写”逻辑；更新了导入

放弃了支持右嵌套连接重写，以支持 2013 年发布的旧 SQLite 版本低于 3.7.16。不希望任何现代 Python 版本依赖于此限制。

该行为首次在 0.9 版本中引入，并作为更大变化的一部分，允许右嵌套连接，如[migration_09.html#feature-joins-09](https://www.sqlalchemy.org/trac/ticket/5401)所述。然而，由于其复杂性，SQLite 的解决方法在 2013-2014 年间产生了许多回归问题。2016 年，方言被修改，以便连接重写逻辑仅在 SQLite 版本低于 3.7.16 时发生，通过二分法确定 SQLite 修复了对此结构的支持的位置，并且没有进一步的问题报告该行为（尽管在内部发现了一些错误）。现在预计，几乎没有 Python 2.7 或 3.5 及以上版本（支持的 Python 版本）的构建包含 SQLite 版本低于 3.7.17，该行为仅在更复杂的 ORM 连接场景中才是必要的。如果安装的 SQLite 版本旧于 3.7.16，则现在会发出警告。

在相关更改中，SQLite 的模块导入不再尝试在 Python 3 上导入“pysqlite2”驱动程序，因为该驱动程序在 Python 3 上不存在；对于旧的 pysqlite2 版本的非常古老的警告也被删除。

[#4895](https://www.sqlalchemy.org/trac/ticket/4895)  ### 为 MariaDB 10.3 添加了序列支持

MariaDB 数据库截至 10.3 版本支持序列。SQLAlchemy 的 MySQL 方言现在在该数据库上实现了对`Sequence`对象的支持，这意味着对于在相同方式下的`Table`或`MetaData`集合中存在的`Sequence`将发出“CREATE SEQUENCE” DDL，就像对于后端如 PostgreSQL、Oracle 等一样，当方言的服务器版本检查确认数据库是 MariaDB 10.3 或更高版本时。此外，当以这些方式使用时，`Sequence`将作为列默认值和主键生成对象。

由于此更改将影响当前部署在 MariaDB 10.3 上的应用程序的 DDL 和 INSERT 语句的行为假设，同时也会显式使用`Sequence`构造在其表定义中，因此重要的是要注意`Sequence`支持一个标志`Sequence.optional`，用于限制`Sequence`生效的情况。当在表的整数主键列中使用“optional”时，`Sequence`：

```py
Table(
    "some_table",
    metadata,
    Column(
        "id", Integer, Sequence("some_seq", start=1, optional=True), primary_key=True
    ),
)
```

上述`Sequence`仅在目标数据库不支持其他生成整数主键值的方式时用于 DDL 和 INSERT 语句。也就是说，上述 Oracle 数据库将使用该序列，但 PostgreSQL 和 MariaDB 10.3 数据库不会。对于正在升级到 SQLAlchemy 1.4 的现有应用程序而言，这可能很重要，因为如果尝试使用未创建的序列进行 INSERT 语句，则会失败。

另请参阅

定义序列

[#4976](https://www.sqlalchemy.org/trac/ticket/4976)  ### 添加了对 SQL Server 的与 IDENTITY 不同的 Sequence 支持

`Sequence`构造现在与 Microsoft SQL Server 完全兼容。当应用于`Column`时，表的 DDL 将不再包含 IDENTITY 关键字，而是依赖于“CREATE SEQUENCE”来确保存在一个序列，然后将用于表上的 INSERT 语句。

在版本 1.3 之前，`Sequence`用于控制 SQL Server 中 IDENTITY 列的参数；这种用法在 1.3 期间发出了弃用警告，并在 1.4 中被移除。要控制 IDENTITY 列的参数，应使用`mssql_identity_start`和`mssql_identity_increment`参数；请参阅下面链接的 MSSQL 方言文档。

另请参见

自增行为/IDENTITY 列

[#4235](https://www.sqlalchemy.org/trac/ticket/4235)

[#4633](https://www.sqlalchemy.org/trac/ticket/4633)

### pg8000 的最低版本为 1.16.6，仅支持 Python 3

pg8000 方言的支持得到了显著改善，得益于项目的维护者的帮助。

由于 API 更改，pg8000 方言现在要求版本为 1.16.6 或更高。从 1.13 系列开始，pg8000 系列已经放弃了对 Python 2 的支持。需要 pg8000 的 Python 2 用户应确保他们的要求被固定在`SQLAlchemy<1.4`。

[#5451](https://www.sqlalchemy.org/trac/ticket/5451)

### 要求使用版本为 2.7 或更高的 psycopg2 以支持 PostgreSQL psycopg2 方言

psycopg2 方言依赖于过去几年中发布的许多 psycopg2 功能。为了简化方言，现在要求的最低版本是 2017 年 3 月发布的版本 2.7。

### psycopg2 方言不再对绑定参数名称有限制

SQLAlchemy 1.3 无法容纳包含百分号或括号的绑定参数名称，这意味着包含这些字符的列名也会有问题，因为 INSERT 和其他 DML 语句会生成与列名匹配的参数名称，这将导致失败。解决方法是利用`Column.key`参数，以便使用替代名称来生成参数，或者在`create_engine()`级别更改方言的参数样式。从 SQLAlchemy 1.4.0beta3 开始，所有命名限制都已被移除，并且在所有情况下参数都被完全转义，因此这些解决方法不再必要。

[#5941](https://www.sqlalchemy.org/trac/ticket/5941)

[#5653](https://www.sqlalchemy.org/trac/ticket/5653)

### psycopg2 方言默认使用“execute_values”与 RETURNING 来处理 INSERT 语句

在使用 Core 和 ORM 时，PostgreSQL 的一个重要性能增强的前半部分，psycopg2 方言现在默认使用`psycopg2.extras.execute_values()`来编译 INSERT 语句，并且还在此模式下实现了 RETURNING 支持。这一变化的另一半是 ORM Batch inserts with psycopg2 now batch statements with RETURNING in most cases，这允许 ORM 利用 RETURNING 与 executemany（即批量插入 INSERT 语句）以便 ORM 批量插入与 psycopg2 在具体情况下快 400%。

这个扩展方法允许在单个语句中插入多行，使用扩展的 VALUES 子句。虽然 SQLAlchemy 的`insert()`构造已经通过`Insert.values()`方法支持这种语法，但是扩展方法允许在执行语句时动态构建 VALUES 子句，这是在通过将参数字典列表传递给`Connection.execute()`时发生的“executemany”执行。它还发生在缓存边界之外，以便在渲染 VALUES 之前可以缓存 INSERT 语句。

在 Performance 示例套件中使用`bulk_inserts.py`脚本快速测试`execute_values()`方法，发现大约**五倍的性能提升**：

```py
$ python -m examples.performance bulk_inserts --test test_core_insert --num 100000 --dburl postgresql://scott:tiger@localhost/test

# 1.3
test_core_insert : A single Core INSERT construct inserting mappings in bulk. (100000 iterations); total time 5.229326 sec

# 1.4
test_core_insert : A single Core INSERT construct inserting mappings in bulk. (100000 iterations); total time 0.944007 sec
```

在版本 1.2 中添加了对“batch”扩展的支持 Support for Batch Mode / Fast Execution Helpers，并在 1.3 中增强以包括对`execute_values`扩展的支持[#4623](https://www.sqlalchemy.org/trac/ticket/4623)。在 1.4 中，`execute_values`扩展现在默认为 INSERT 语句打开；UPDATE 和 DELETE 的“batch”扩展默认关闭。

此外，`execute_values`扩展函数支持将由`RETURNING`生成的行作为聚合列表返回。如果给定的`insert()`构造请求通过`Insert.returning()`方法或类似方法返回生成的默认值，则`psycopg2`方言现在将检索此列表；然后将这些行安装在结果中，以便像直接来自游标一样检索它们。这允许 ORM 等工具在所有情况下使用批量插入，预计将提供显著的性能改进。

`psycopg2`方言的`executemany_mode`功能已经进行了以下更改：

+   添加了一个新模式`"values_only"`。此模式使用非常高效的`psycopg2.extras.execute_values()`扩展方法来运行使用`executemany()`的编译 INSERT 语句，但不使用`execute_batch()`来运行 UPDATE 和 DELETE 语句。这个新模式现在是`psycopg2`方言的默认设置。

+   现有的`"values"`模式现在被命名为`"values_plus_batch"`。此模式将使用`execute_values`进行 INSERT 语句，使用`execute_batch`进行 UPDATE 和 DELETE 语句。该模式默认情况下未启用，因为它会禁用使用`executemany()`执行 UPDATE 和 DELETE 语句时的`cursor.rowcount`的正确功能。

+   对于 INSERT 语句，`"values_only"`和`"values"`启用了 RETURNING 支持。`psycopg2`方言将使用`fetch=True`标志从`psycopg2`接收行，并将它们安装到结果集中，就好像它们直接来自游标一样（尽管它们最终确实来自游标，但`psycopg2`的扩展函数已将多个批次聚合成一个列表）。

+   `execute_values`的默认“page_size”设置已从 100 增加到 1000。`execute_batch`函数的默认值仍为 100。这些参数可以像以前一样进行修改。

+   1.2 版本功能的`use_batch_mode`标志已被移除；行为仍可通过 1.3 版本中添加的`executemany_mode`标志进行控制。

+   核心引擎和方言已经增强以支持`executemany`加返回模式，目前仅适用于`psycopg2`，通过提供新的`CursorResult.inserted_primary_key_rows`和`CursorResult.returned_default_rows`访问器。

另请参见

Psycopg2 快速执行助手

[#5401](https://www.sqlalchemy.org/trac/ticket/5401)

### 从 SQLite 方言中删除了“join rewriting”逻辑；更新了导入

放弃了支持右嵌套连接重写以支持 2013 年发布的旧 SQLite 版本 3.7.16 之前的版本。不希望任何现代 Python 版本依赖于此限制。

该行为首次在 0.9 版本中引入，并作为允许右嵌套连接的较大更改的一部分，如许多 JOIN 和 LEFT OUTER JOIN 表达式将不再包装在(SELECT * FROM ..) AS ANON_1 中所述。然而，由于其复杂性，SQLite 的解决方法在 2013-2014 年期间产生了许多回归。2016 年，方言被修改，以便仅在 SQLite 版本低于 3.7.16 的情况下进行连接重写逻辑，通过二分法确定 SQLite 修复了对此构造的支持的位置，并且没有进一步的问题报告该行为（尽管在内部发现了一些错误）。现在预计，几乎没有 Python 2.7 或 3.5 及以上版本（支持的 Python 版本）的构建会包含 SQLite 版本低于 3.7.17，该行为仅在更复杂的 ORM 连接场景中才是必要的。如果安装的 SQLite 版本旧于 3.7.16，则现在会发出警告。

在相关更改中，SQLite 的模块导入不再尝试在 Python 3 上导入“pysqlite2”驱动程序，因为该驱动程序在 Python 3 上不存在；对于旧的 pysqlite2 版本的非常古老警告也被删除。

[#4895](https://www.sqlalchemy.org/trac/ticket/4895)

### 为 MariaDB 10.3 添加了 Sequence 支持

截至 10.3 版本，MariaDB 数据库支持序列。SQLAlchemy 的 MySQL 方言现在实现了对该数据库的`Sequence`对象的支持，这意味着当方言的服务器版本检查确认数据库是 MariaDB 10.3 或更高版本时，将为`Table`或`MetaData`集合中存在的`Sequence`发出“CREATE SEQUENCE” DDL，就像对于后端如 PostgreSQL、Oracle 等一样。此外，当以这些方式使用时，`Sequence`将作为列默认值和主键生成对象。

由于这一变化将影响 DDL 的假设以及针对 MariaDB 10.3 的当前部署应用程序的 INSERT 语句的行为，该应用程序也恰好在其表定义中明确使用`Sequence`构造，因此重要的是要注意`Sequence`支持一个标志`Sequence.optional`，用于限制`Sequence`生效的情况。当在表的整数主键列上使用“optional”时：

```py
Table(
    "some_table",
    metadata,
    Column(
        "id", Integer, Sequence("some_seq", start=1, optional=True), primary_key=True
    ),
)
```

上述`Sequence`仅在目标数据库不支持其他生成整数主键值的方式时用于 DDL 和 INSERT 语句。也就是说，上述 Oracle 数据库将使用该序列，但 PostgreSQL 和 MariaDB 10.3 数据库不会。这对于正在升级到 SQLAlchemy 1.4 的现有应用程序可能很重要，因为如果插入语句试图使用未创建的序列，则会失败。

另请参见

定义序列

[#4976](https://www.sqlalchemy.org/trac/ticket/4976)

### 添加了对 SQL Server 的与 IDENTITY 不同的 Sequence 支持

`Sequence`构造现在已经完全与 Microsoft SQL Server 兼容。当应用于`Column`时，表的 DDL 将不再包含 IDENTITY 关键字，而是依赖于“CREATE SEQUENCE”来确保存在一个序列，然后将用于表的 INSERT 语句。

在版本 1.3 之前，`Sequence`用于控制 SQL Server 中的 IDENTITY 列的参数；这种用法在 1.3 中发出了弃用警告，并在 1.4 中已被移除。对于控制 IDENTITY 列的参数，应使用`mssql_identity_start`和`mssql_identity_increment`参数；请参阅下面链接的 MSSQL 方言文档。

另请参见

自增行为 / IDENTITY 列

[#4235](https://www.sqlalchemy.org/trac/ticket/4235)

[#4633](https://www.sqlalchemy.org/trac/ticket/4633)
