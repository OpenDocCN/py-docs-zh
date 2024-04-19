# SQLAlchemy 2.0 - 主要迁移指南

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_20.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_20.html)

读者须知

SQLAlchemy 2.0 的迁移文档分为**两个**文档 - 一个详细说明了从 1.x 到 2.x 系列的主要 API 转变，另一个详细说明了相对于 SQLAlchemy 1.4 的新功能和行为：

+   SQLAlchemy 2.0 - 主要迁移指南 - 本文档，1.x 到 2.x API 转变

+   SQLAlchemy 2.0 有什么新功能？ - SQLAlchemy 2.0 的新功能和行为

已经将其 1.4 应用程序更新以遵循 SQLAlchemy 2.0 引擎和 ORM 约定的读者，可以转到 SQLAlchemy 2.0 有什么新功能？ 查看新功能和能力的概述。

关于本文档

本文档描述了 SQLAlchemy 版本 1.4 和 SQLAlchemy 版本 2.0 之间的变化。

SQLAlchemy 2.0 对于 Core 和 ORM 组件中许多关键的 SQLAlchemy 使用模式都进行了重大转变。此次发布的目标是对自 SQLAlchemy 早期开始以来的一些最基本的假设进行轻微调整，并提供一个新的简化使用模型，希望能够在 Core 和 ORM 组件之间更加一致和更加精简，并且功能更强大。Python 转向仅支持 Python 3，以及 Python 3 的逐渐类型化系统的出现，是这种转变的最初灵感来源，还有 Python 社区的变化，现在不仅包括了核心的数据库程序员，还有一个涵盖了许多不同学科的庞大的新社区的数据科学家和学生。

SQLAlchemy 从 Python 2.3 开始，那个时候没有上下文管理器，没有函数装饰器，Unicode 作为第二类特性，以及其他许多现在已经不为人知的缺点。SQLAlchemy 2.0 最大的变化是针对 SQLAlchemy 发展早期留下的残余假设以及由于增量引入关键 API 特性（如 `Query` 和声明）而导致的残余残留物。它还希望标准化一些已被证明非常有效的较新功能。

## 1.4->2.0 迁移路径

被认为是“SQLAlchemy 2.0”的最突出的架构特性和 API 变化实际上已经在 1.4 系列中完全可用，以提供从 1.x 到 2.x 系列的清晰升级路径，并为这些特性本身提供一个 beta 平台。这些变化包括：

+   新的 ORM 语句范例

+   核心和 ORM 中的 SQL 缓存

+   新的声明特性，ORM 集成

+   新的结果对象

+   select() / case() 接受位置表达式

+   Core 和 ORM 的 asyncio 支持

上述项目链接到在 SQLAlchemy 1.4 中介绍这些新范式的描述，位于 SQLAlchemy 1.4 中的新功能是什么？文档中。

对于 SQLAlchemy 2.0，所有被标记为 2.0 废弃的 API 特性和行为现在已经被最终确定；特别是，**不再存在的主要 API**包括：

+   元数据限制和无连接执行

+   在 Connection 上模拟的自动提交

+   Session.autocommit 参数/模式

+   对 select() 的列表 / 关键字参数

+   Python 2 支持

上述项目指的是在 2.0 版本中最突出的完全不兼容的更改，这些更改已在 2.0 发布中最终确定。应用程序适应这些更改以及其他更改的迁移路径首先是进入到 SQLAlchemy 1.4 系列中，其中“未来”API 可用于提供“2.0”工作方式，然后是进入到 2.0 系列中，其中上述不再使用的 API 和其他 API 被移除。

此迁移路径的完整步骤稍后在本文档的 1.x -> 2.x 迁移概览中。

## 1.x -> 2.x 迁移概览

SQLAlchemy 2.0 迁移在 SQLAlchemy 1.4 发布中呈现为一系列步骤，允许以渐进、迭代的方式将任何大小或复杂性的应用程序迁移到 SQLAlchemy 2.0。从 Python 2 到 Python 3 迁移中学到的经验启发了一个系统，该系统尽可能地不需要任何“破坏性”更改，或者不需要普遍进行或完全不进行的更改。

作为验证 2.0 架构的手段，同时允许完全迭代的过渡环境，2.0 新 API 和特性的整个范围都存在于 1.4 系列中，并且可用；这包括了一些重要的新功能领域，如 SQL 缓存系统、新的 ORM 语句执行模型、ORM 和 Core 的新事务范例、统一经典和声明性映射的新 ORM 声明系统、对 Python 数据类的支持，以及 Core 和 ORM 的 asyncio 支持。

实现 2.0 迁移的步骤在以下子章节中；总体上，一般策略是一旦一个应用程序在所有警告标志都打开的情况下在 1.4 上运行，并且不发出任何 2.0 废弃警告，则该应用程序现在**基本上**与 SQLAlchemy 2.0 兼容。**请注意，在运行针对 SQLAlchemy 2.0 的实际代码迁移的最后一步中，可能会有其他 API 和行为更改，这些更改在运行时可能会表现出不同的行为**。

### 第一个先决条件，第一步 - 一个可用的 1.3 应用程序

第一步是让现有应用程序升级到 1.4，在典型的非平凡应用程序的情况下，确保它在 SQLAlchemy 1.3 上运行时没有弃用警告。1.4 版确实有一些与之前版本中发出警告的条件相关的更改，包括一些在 1.3 版中引入的警告，特别是一些关于`relationship.viewonly`和`relationship.sync_backref`标志行为的更改。

为了获得最佳结果，应用程序应该能够在最新的 SQLAlchemy 1.3 发布版中运行，或通过所有测试，而不会出现 SQLAlchemy 的弃用警告；这些是针对`SADeprecationWarning`类发出的警告。

### 第一个先决条件，第二步 - 一个运行的 1.4 应用程序

应用程序一旦在 SQLAlchemy 1.3 上准备就绪，下一步就是让它在 SQLAlchemy 1.4 上运行。在绝大多数情况下，应用程序应该可以在从 SQLAlchemy 1.3 到 1.4 的过程中无问题地运行。然而，像任何 1.x 和 1.y 版本之间的情况一样，API 和行为可能会发生细微的或在某些情况下略微显著的变化，SQLAlchemy 项目总是在前几个月收到大量的回归报告。

1.x->1.y 发布过程通常会围绕一些边缘情况进行一些比较激进的更改，这些更改基于预计将很少或根本不会使用的用例。对于 1.4，被确定为属于这个领域的更改如下：

+   URL 对象现在是不可变的 - 这影响了对`URL`对象进行操作的代码，可能会影响到使用`CreateEnginePlugin`扩展点的代码。这是一个不常见的情况，但可能特别影响到一些使用特殊数据库提供逻辑的测试套件。通过对使用相对较新且鲜为人知的`CreateEnginePlugin`类的代码进行 Github 搜索，找到了两个不受此更改影响的项目。

+   SELECT 语句不再隐式视为 FROM 子句 - 这个变化可能会影响一些依赖于 `Select` 构造的行为的代码，它会创建通常令人困惑且无法工作的未命名子查询。这些子查询在大多数数据库中都会被拒绝，因为通常需要一个名称，除了 SQLite 外。然而，一些应用程序可能需要调整一些意外依赖于此的查询。

+   select().join() 和 outerjoin() 将 JOIN 条件添加到当前查询中，而不是创建子查询 - 有些相关的是，`Select` 类中的 `.join()` 和 `.outerjoin()` 方法隐式地创建了一个子查询，然后返回一个 `Join` 构造，这在大多数情况下是没有用的，会导致很多混乱。决定采用更加有用的 2.0 风格的连接构建方法，这些方法现在与 ORM `Query.join()` 方法的工作方式相同。

+   许多 Core 和 ORM 语句对象现在在编译阶段执行大部分构建和验证工作 - 一些与构建 `Query` 或 `Select` 相关的错误消息可能直到编译 / 执行阶段才会被发出，而不是在构建时。这可能会影响一些针对失败模式进行测试的测试套件。

要查看 SQLAlchemy 1.4 变化的完整概述，请参阅 SQLAlchemy 1.4 有什么新特性？ 文档。

### 迁移至 2.0 第一步 - 仅支持 Python 3（2.0 兼容性最低要求 Python 3.7）

SQLAlchemy 2.0 最初是受到 Python 2 的 EOL 是在 2020 年的事实启发的。SQLAlchemy 比其他主要项目花费更长的时间来放弃对 Python 2.7 的支持。然而，为了使用 SQLAlchemy 2.0，应用程序需要至少在 **Python 3.7** 上可运行。SQLAlchemy 1.4 在 Python 3 系列中支持 Python 3.6 或更新版本；在 1.4 系列中，应用程序可以继续在 Python 2.7 或至少 Python 3.6 上运行。然而，版本 2.0 从 Python 3.7 开始。

### 迁移至 2.0 第二步 - 打开 RemovedIn20Warnings

SQLAlchemy 1.4 版本提供了一个条件化的弃用警告系统，灵感来自于 Python 的“-3”标志，该标志会指示运行中应用程序中的传统模式。对于 SQLAlchemy 1.4，只有当环境变量 `SQLALCHEMY_WARN_20` 设置为 `true` 或 `1` 时，才会发出 `RemovedIn20Warning` 弃用类。

给出以下示例程序：

```py
from sqlalchemy import column
from sqlalchemy import create_engine
from sqlalchemy import select
from sqlalchemy import table

engine = create_engine("sqlite://")

engine.execute("CREATE TABLE foo (id integer)")
engine.execute("INSERT INTO foo (id) VALUES (1)")

foo = table("foo", column("id"))
result = engine.execute(select([foo.c.id]))

print(result.fetchall())
```

上述程序使用了许多用户已经识别为“传统”的模式，即使用了“无连接执行”API 的 `Engine.execute()` 方法。当我们针对 1.4 运行上述程序时，它返回一行：

```py
$ python test3.py
[(1,)]
```

要启用“2.0 弃用模式”，我们启用 `SQLALCHEMY_WARN_20=1` 变量，并确保选择了一个不会抑制任何警告的 [警告过滤器](https://docs.python.org/3/library/warnings.html#the-warnings-filter)：

```py
SQLALCHEMY_WARN_20=1 python -W always::DeprecationWarning test3.py
```

由于报告的警告位置并不总是正确的，如果没有完整的堆栈跟踪，定位有问题的代码可能会很困难。可以通过指定 `error` 警告过滤器将警告转换为异常，使用 Python 选项 `-W error::DeprecationWarning` 来实现这一点。

打开警告后，我们的程序现在有很多话要说：

```py
$ SQLALCHEMY_WARN_20=1 python2 -W always::DeprecationWarning test3.py
test3.py:9: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  engine.execute("CREATE TABLE foo (id integer)")
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0\.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return connection.execute(statement, *multiparams, **params)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0\.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  self._commit_impl(autocommit=True)
test3.py:10: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  engine.execute("INSERT INTO foo (id) VALUES (1)")
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0\.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return connection.execute(statement, *multiparams, **params)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0\.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  self._commit_impl(autocommit=True)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/sql/selectable.py:4271: RemovedIn20Warning: The legacy calling style of select() is deprecated and will be removed in SQLAlchemy 2.0\.  Please use the new calling style described at select(). (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return cls.create_legacy_select(*args, **kw)
test3.py:14: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  result = engine.execute(select([foo.c.id]))
[(1,)]
```

根据上述指导，我们可以将我们的程序迁移到使用 2.0 样式，作为奖励，我们的程序更加清晰：

```py
from sqlalchemy import column
from sqlalchemy import create_engine
from sqlalchemy import select
from sqlalchemy import table
from sqlalchemy import text

engine = create_engine("sqlite://")

# don't rely on autocommit for DML and DDL
with engine.begin() as connection:
    # use connection.execute(), not engine.execute()
    # use the text() construct to execute textual SQL
    connection.execute(text("CREATE TABLE foo (id integer)"))
    connection.execute(text("INSERT INTO foo (id) VALUES (1)"))

foo = table("foo", column("id"))

with engine.connect() as connection:
    # use connection.execute(), not engine.execute()
    # select() now accepts column / table expressions positionally
    result = connection.execute(select(foo.c.id))

    print(result.fetchall())
```

“2.0 弃用模式”的目标是，当使用“2.0 弃用模式”运行时没有 `RemovedIn20Warning` 警告的程序，然后准备好在 SQLAlchemy 2.0 中运行。

### 迁移到 2.0 第三步 - 解决所有的 RemovedIn20Warnings

可以迭代地开发代码来解决这些警告。在 SQLAlchemy 项目本身中，采取的方法如下：

1.  在测试套件中启用 `SQLALCHEMY_WARN_20=1` 环境变量，对于 SQLAlchemy 来说，这在 tox.ini 文件中。

1.  在测试套件的设置中，设置一系列警告过滤器，以选择特定子集的警告来引发异常，或者忽略（或记录）它们。一次只处理一个警告子组。下面，配置了一个警告过滤器，用于一个应用程序，其中需要更改 Core 级别的 `.execute()` 调用，以便所有测试都通过，但是所有其他 2.0 样式的警告将被抑制：

    ```py
    import warnings
    from sqlalchemy import exc

    # for warnings not included in regex-based filter below, just log
    warnings.filterwarnings("always", category=exc.RemovedIn20Warning)

    # for warnings related to execute() / scalar(), raise
    for msg in [
        r"The (?:Executable|Engine)\.(?:execute|scalar)\(\) function",
        r"The current statement is being autocommitted using implicit autocommit,",
        r"The connection.execute\(\) method in SQLAlchemy 2.0 will accept "
        "parameters as a single dictionary or a single sequence of "
        "dictionaries only.",
        r"The Connection.connect\(\) function/method is considered legacy",
        r".*DefaultGenerator.execute\(\)",
    ]:
        warnings.filterwarnings(
            "error",
            message=msg,
            category=exc.RemovedIn20Warning,
        )
    ```

1.  当应用程序中的每个警告子类被解决时，可以将被“always”过滤器捕获的新警告添加到要解决的“错误”列表中。

1.  一旦不再发出警告，就可以移除过滤器。

### 迁移到 2.0 第四步 - 在 Engine 上使用 `future` 标志

`Engine` 对象在 2.0 版本中具有更新的事务级 API。在 1.4 中，通过将标志 `future=True` 传递给 `create_engine()` 函数可以使用这个新 API。

当使用`create_engine.future`标志时，`Engine`和`Connection`对象完全支持 2.0 API，不再支持任何旧特性，包括`Connection.execute()`的新参数格式、去除“隐式自动提交”、字符串语句要求使用`text()`构造，除非使用`Connection.exec_driver_sql()`方法，以及从`Engine`进行无连接执行已被移除。

如果关于`Engine`和`Connection`使用的所有`RemovedIn20Warning`警告都已解决，则可以启用`create_engine.future`标志，并且不应引发任何错误。

新引擎的描述见`Engine`，它提供了一个新的`Connection`对象。除了上述更改之外，`Connection`对象具有`Connection.commit()`和`Connection.rollback()`方法，以支持新的“随时提交”操作模式：

```py
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2:///")

with engine.connect() as conn:
    conn.execute(text("insert into table (x) values (:some_x)"), {"some_x": 10})

    conn.commit()  # commit as you go
```

### 迁移到 2.0 第五步 - 在会话上使用`future`标志

`Session`对象还在版本 2.0 中提供了更新的事务/连接级 API。此 API 在 1.4 中可通过在`Session`或`sessionmaker`上使用`Session.future`标志获得。

`Session`对象支持“future”模式，并涉及以下更改：

1.  当解析用于连接的引擎时，`Session` 不再支持“绑定元数据”。这意味着必须将一个 `Engine` 对象传递给构造函数（这可以是传统或未来风格的对象）。

1.  不再支持 `Session.begin.subtransactions` 标志。

1.  `Session.commit()` 方法总是向数据库发出 COMMIT，而不是尝试协调“子事务”。

1.  `Session.rollback()` 方法总是一次性回滚整个事务堆栈，而不是尝试保持“子事务”不变。

在 1.4 版本中，`Session` 也支持更灵活的创建模式，这些模式现在与 `Connection` 对象使用的模式紧密匹配。重点包括 `Session` 可以用作上下文管理器：

```py
from sqlalchemy.orm import Session

with Session(engine) as session:
    session.add(MyObject())
    session.commit()
```

此外，`sessionmaker` 对象支持一个 `sessionmaker.begin()` 上下文管理器，它将创建一个 `Session` 并在一个块中开始/提交事务：

```py
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(engine)

with Session.begin() as session:
    session.add(MyObject())
```

请参阅 Session-level vs. Engine level transaction control 部分，了解 `Session` 创建模式与 `Connection` 创建模式的比较。

一旦应用程序通过所有测试/使用 `SQLALCHEMY_WARN_20=1` 运行，并将所有 `exc.RemovedIn20Warning` 出现设置为引发错误，**应用程序就准备好了！**。

接下来的部分将详细说明所有主要 API 修改所需进行的特定更改。

### 迁移到 2.0 第六步 - 为显式类型的 ORM 模型添加 `__allow_unmapped__`

SQLAlchemy 2.0 对 ORM 模型上的[**PEP 484**](https://peps.python.org/pep-0484/)类型注释进行了新的运行时解释支持。这些注释的要求是它们必须使用`Mapped`通用容器。不使用`Mapped`的注释，例如链接到`relationship()`等构造的，将在 Python 中引发错误，因为它们暗示了错误的配置。

使用 Mypy 插件的 SQLAlchemy 应用，其中明确注释不使用`Mapped`在其注释中的，会遇到这些错误，就像下面的示例中会发生的一样：

```py
Base = declarative_base()

class Foo(Base):
    __tablename__ = "foo"

    id: int = Column(Integer, primary_key=True)

    # will raise
    bars: List["Bar"] = relationship("Bar", back_populates="foo")

class Bar(Base):
    __tablename__ = "bar"

    id: int = Column(Integer, primary_key=True)
    foo_id = Column(ForeignKey("foo.id"))

    # will raise
    foo: Foo = relationship(Foo, back_populates="bars", cascade="all")
```

上面的`Foo.bars`和`Bar.foo`的`relationship()`声明在类构建时会引发错误，因为它们没有使用`Mapped`（相比之下，使用`Column`的注释在 2.0 中被忽略，因为这些能够被识别为传统配置风格）。为了允许所有不使用`Mapped`的注释通过而不引发错误，可以在类或任何子类上使用`__allow_unmapped__`属性，这将导致在这些情况下这些注释被新的 Declarative 系统完全忽略。

注意

`__allow_unmapped__`指令**仅**适用于 ORM 的*运行时*行为。它不会影响 Mypy 的行为，上述映射仍然要求安装 Mypy 插件。对于完全符合 2.0 风格的 ORM 模型，可以在 Migrating an Existing Mapping 中遵循迁移步骤，以便在 Mypy 下正确类型化*无需*插件。

下面的示例说明了将`__allow_unmapped__`应用于 Declarative `Base`类的应用，它将对所有从`Base`继承的类生效：

```py
# qualify the base with __allow_unmapped__.  Can also be
# applied to classes directly if preferred
class Base:
    __allow_unmapped__ = True

Base = declarative_base(cls=Base)

# existing mapping proceeds, Declarative will ignore any annotations
# which don't include ``Mapped[]``
class Foo(Base):
    __tablename__ = "foo"

    id: int = Column(Integer, primary_key=True)

    bars: List["Bar"] = relationship("Bar", back_populates="foo")

class Bar(Base):
    __tablename__ = "bar"

    id: int = Column(Integer, primary_key=True)
    foo_id = Column(ForeignKey("foo.id"))

    foo: Foo = relationship(Foo, back_populates="bars", cascade="all")
```

在 2.0.0beta3 版本中更改：- 改进了`__allow_unmapped__`属性支持，以允许不使用`Mapped`的 1.4 风格显式注释关系保持可用。### 迁移到 2.0 第七步 - 针对 SQLAlchemy 2.0 版本进行测试

如前所述，SQLAlchemy 2.0 有一些额外的 API 和行为变化，旨在向后兼容，但可能仍会引入一些不兼容性。因此，在整个移植过程完成后，最后一步是针对 SQLAlchemy 2.0 的最新版本进行测试，以纠正可能存在的任何剩余问题。

SQLAlchemy 2.0 中的新特性是什么？ 提供了 SQLAlchemy 2.0 的新特性和行为的概述，这些特性和行为超出了 1.4->2.0 API 变化的基本集合。

## 2.0 迁移 - 核心连接 / 事务

### 库级别的（但不是驱动级别的）“自动提交”已从核心和 ORM 中删除

**概要**

在 SQLAlchemy 1.x 中，以下语句将自动提交底层的 DBAPI 事务，但在 SQLAlchemy 2.0 中不会发生：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(some_table.insert().values(foo="bar"))
```

这个自动提交也不会发生：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(text("INSERT INTO table (foo) VALUES ('bar')"))
```

对于需要提交的自定义 DML 的常见解决方法，“自动提交”执行选项将被移除：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(text("EXEC my_procedural_thing()").execution_options(autocommit=True))
```

**迁移到 2.0**

与 1.x 风格 和 2.0 风格 执行兼容的方法是使用 `Connection.begin()` 方法，或者使用 `Engine.begin()` 上下文管理器：

```py
with engine.begin() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.execute(some_other_table.insert().values(bat="hoho"))

with engine.connect() as conn:
    with conn.begin():
        conn.execute(some_table.insert().values(foo="bar"))
        conn.execute(some_other_table.insert().values(bat="hoho"))

with engine.begin() as conn:
    conn.execute(text("EXEC my_procedural_thing()"))
```

当使用 `create_engine.future` 标志和 2.0 风格 时，“逐步提交”风格也可以使用，因为 `Connection` 特性具有 **autobegin** 行为，当在没有显式调用 `Connection.begin()` 的情况下首次调用语句时会发生：

```py
with engine.connect() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.execute(some_other_table.insert().values(bat="hoho"))

    conn.commit()
```

当启用 2.0 弃用模式 时，将会发出警告，指示发生了已弃用的“自动提交”功能的地方，表明应该注意明确事务的地方。

**讨论**

SQLAlchemy 的第一个版本与 Python DBAPI（[**PEP 249**](https://peps.python.org/pep-0249/)）的精神不合，因为它试图隐藏 [**PEP 249**](https://peps.python.org/pep-0249/) 强调的“隐式开始”和“显式提交”事务。十五年后，我们现在认识到这实际上是一个错误，因为 SQLAlchemy 的许多尝试“隐藏”事务存在的模式导致 API 更复杂，工作不一致，对那些对关系数据库和 ACID 事务一般都是新手的用户来说极其混乱。SQLAlchemy 2.0 将取消所有隐式提交事务的尝试，使用模式将始终要求用户以某种方式标记事务的“开始”和“结束”，就像在 Python 中读取或写入文件具有“开始”和“结束”一样。

对于纯文本语句的自动提交情况，实际上有一个正则表达式来解析每个语句以检测自动提交！毫不奇怪，这个正则表达式不断地无法适应各种类型的语句和暗示向数据库“写入”的存储过程，导致持续的混淆，因为一些语句在数据库中产生结果，而其他语句则没有。通过阻止用户意识到事务概念，我们在这一点上收到了很多错误报告，因为用户不理解无论某些层是否自动提交事务，数据库都会始终使用事务。

SQLAlchemy 2.0 将要求在每个级别上所有数据库操作都明确指明事务的使用方式。对于绝大多数核心用例，已经推荐的模式是：

```py
with engine.begin() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
```

对于“随时提交，或者回滚”的用法，这类似于如何通常使用 `Session`，即 `Engine` 创建时使用了 `create_engine.future` 标志后返回的“未来”版本 `Connection`，包含新的 `Connection.commit()` 和 `Connection.rollback()` 方法，这些方法会在首次调用语句时自动开始事务：

```py
# 1.4 / 2.0 code

from sqlalchemy import create_engine

engine = create_engine(..., future=True)

with engine.connect() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.commit()

    conn.execute(text("some other SQL"))
    conn.rollback()
```

在上面，`engine.connect()`方法将返回一个具有**自动开始**功能的`Connection`，这意味着在首次使用执行方法时会发出`begin()`事件（注意，Python DBAPI 中实际上没有“BEGIN”）。“自动开始”是 SQLAlchemy 1.4 中的一种新模式，它既由`Connection`特性，也由 ORM `Session`对象特性；自动开始允许在首次获取对象时显式调用`Connection.begin()`方法，以便为希望标记事务开始的模式提供方案，但如果不调用该方法，则在首次对对象进行工作时隐式发生。

“隐式”和“无连接”执行，“绑定元数据”已移除的移除与讨论“隐式”和“无连接”执行，移除“绑定元数据”密切相关。所有这些传统模式都是从 Python 在 SQLAlchemy 首次创建时没有上下文管理器或装饰器这一事实上建立起来的，因此没有便捷的惯用模式来标记资源的使用。

#### 驱动程序级别的自动提交仍然可用

真正的“自动提交”行为现在已经在大多数 DBAPI 实现中广泛可用，并且通过`Connection.execution_options.isolation_level`参数由 SQLAlchemy 支持，如设置事务隔离级别，包括 DBAPI 自动提交中所讨论的。真正的自动提交被视为一种“隔离级别”，因此当使用自动提交时，应用程序代码的结构不会发生变化；`Connection.begin()`上下文管理器以及诸如`Connection.commit()`之类的方法仍然可以使用，它们在数据库驱动程序级别只是无操作。

**概要**

将`Engine`与`MetaData`对象关联起来的能力已被移除，这样就可以使用一系列所谓的“无连接”执行模式：

```py
from sqlalchemy import MetaData

metadata_obj = MetaData(bind=engine)  # no longer supported

metadata_obj.create_all()  # requires Engine or Connection

metadata_obj.reflect()  # requires Engine or Connection

t = Table("t", metadata_obj, autoload=True)  # use autoload_with=engine

result = engine.execute(t.select())  # no longer supported

result = t.select().execute()  # no longer supported
```

**2.0 迁移**

对于架构级模式，需要明确使用`Engine`或`Connection`。`Engine`仍然可以直接用作`MetaData.create_all()`操作或 autoload 操作的连接源。对于执行语句，只有`Connection`对象具有`Connection.execute()`方法（除了 ORM 级别的`Session.execute()`方法）：

```py
from sqlalchemy import MetaData

metadata_obj = MetaData()

# engine level:

# create tables
metadata_obj.create_all(engine)

# reflect all tables
metadata_obj.reflect(engine)

# reflect individual table
t = Table("t", metadata_obj, autoload_with=engine)

# connection level:

with engine.connect() as connection:
    # create tables, requires explicit begin and/or commit:
    with connection.begin():
        metadata_obj.create_all(connection)

    # reflect all tables
    metadata_obj.reflect(connection)

    # reflect individual table
    t = Table("t", metadata_obj, autoload_with=connection)

    # execute SQL statements
    result = connection.execute(t.select())
```

**讨论**

核心文档已经在这里规范了所需的模式，因此大多数现代应用程序可能不需要在任何情况下进行太多更改，但是可能仍然有许多应用程序依赖于`engine.execute()`调用，这些调用将需要进行调整。

“无连接”执行指的是仍然相当流行的从`Engine`中调用`.execute()`的模式：

```py
result = engine.execute(some_statement)
```

上述操作隐式地获取了一个`Connection`对象，并在其上运行`.execute()`方法。虽然这似乎是一个简单的便利功能，但已经证明它引起了几个问题：

+   程序中出现了大量的`engine.execute()`调用，这已经变得普遍，过度使用了原本应该很少使用的功能，并导致了低效的非事务性应用程序。新用户对于`engine.execute()`和`connection.execute()`之间的区别感到困惑，这两种方法之间的微妙差别通常不被理解。

+   该功能依赖于“应用程序级自动提交”功能才有意义，该功能本身也正在被删除，因为它也是低效和误导性的。

+   为了处理结果集，`Engine.execute` 返回一个带有未消耗的游标结果的结果对象。这个游标结果必然仍然链接到保持打开事务的 DBAPI 连接，所有这些在结果集完全消耗了等待在游标中的行之后释放。这意味着 `Engine.execute` 实际上并没有在调用完成时关闭它声称正在管理的连接资源。SQLAlchemy 的“自动关闭”行为调整得足够好，以至于用户通常不会报告这个系统的任何负面影响，然而，这仍然是 SQLAlchemy 最早版本中遗留下来的一个过于隐式和低效的系统。

移除“无连接”执行，然后移除一个更加遗留的模式，“隐式、无连接”执行：

```py
result = some_statement.execute()
```

上述模式具有“无连接”执行的所有问题，而且它依赖于“绑定元数据”模式，SQLAlchemy 多年来一直试图减少使用这种模式。这是 SQLAlchemy 在 0.1 版本中首次宣传的使用模型，当 `Connection` 对象被引入并且后来 Python 上下文管理器提供了更好的在固定范围内使用资源的模式时，这种模式几乎立即过时。

随着隐式执行的移除，“绑定元数据”本身也不再在此系统中有用。在现代用法中，“绑定元数据”仍然在 `MetaData.create_all()` 调用以及 `Session` 对象中有些方便，然而，让这些函数显式接收一个 `Engine` 提供了更清晰的应用程序设计。

#### 多种选择变成了一个选择。

总的来说，上述执行模式是在 SQLAlchemy 的第一个 0.1 版本发布之前引入的。经过多年的减少这些模式的使用，“隐式、无连接”执行和“绑定元数据”不再像以前那样广泛使用，所以在 2.0 中，我们试图最终减少从“多种选择”中执行语句到 Core 的选择数量：

```py
# many choices

# bound metadata?
metadata_obj = MetaData(engine)

# or not?
metadata_obj = MetaData()

# execute from engine?
result = engine.execute(stmt)

# or execute the statement itself (but only if you did
# "bound metadata" above, which means you can't get rid of "bound" if any
# part of your program uses this form)
result = stmt.execute()

# execute from connection, but it autocommits?
conn = engine.connect()
conn.execute(stmt)

# execute from connection, but autocommit isn't working, so use the special
# option?
conn.execution_options(autocommit=True).execute(stmt)

# or on the statement ?!
conn.execute(stmt.execution_options(autocommit=True))

# or execute from connection, and we use explicit transaction?
with conn.begin():
    conn.execute(stmt)
```

对于“一个选择”，我们指的是“与显式事务的显式连接”；根据需要，仍然有一些方法来标记事务块。在写操作的情况下，“一个选择”是获取`Connection`，然后显式地标记事务：

```py
# one choice - work with explicit connection, explicit transaction
# (there remain a few variants on how to demarcate the transaction)

# "begin once" - one transaction only per checkout
with engine.begin() as conn:
    result = conn.execute(stmt)

# "commit as you go" - zero or more commits per checkout
with engine.connect() as conn:
    result = conn.execute(stmt)
    conn.commit()

# "commit as you go" but with a transaction block instead of autobegin
with engine.connect() as conn:
    with conn.begin():
        result = conn.execute(stmt)
```

### `execute()` 方法更加严格，执行选项更加突出。

**概要**

在 SQLAlchemy 2.0 中可能与`sqlalchemy.engine.Connection()` execute 方法一起使用的参数模式已经大大简化，删除了许多以前可用的参数模式。 1.4 系列中的新 API 在`sqlalchemy.engine.Connection()`中描述。下面的示例说明了需要修改的模式：

```py
connection = engine.connect()

# direct string SQL not supported; use text() or exec_driver_sql() method
result = connection.execute("select * from table")

# positional parameters no longer supported, only named
# unless using exec_driver_sql()
result = connection.execute(table.insert(), ("x", "y", "z"))

# **kwargs no longer accepted, pass a single dictionary
result = connection.execute(table.insert(), x=10, y=5)

# multiple *args no longer accepted, pass a list
result = connection.execute(
    table.insert(), {"x": 10, "y": 5}, {"x": 15, "y": 12}, {"x": 9, "y": 8}
)
```

**迁移到 2.0**

新的`Connection.execute()`方法现在接受 1.x `Connection.execute()`方法接受的参数样式的子集，因此以下代码在 1.x 和 2.0 之间是兼容的：

```py
connection = engine.connect()

from sqlalchemy import text

result = connection.execute(text("select * from table"))

# pass a single dictionary for single statement execution
result = connection.execute(table.insert(), {"x": 10, "y": 5})

# pass a list of dictionaries for executemany
result = connection.execute(
    table.insert(), [{"x": 10, "y": 5}, {"x": 15, "y": 12}, {"x": 9, "y": 8}]
)
```

**讨论**

删除了使用`*args`和`**kwargs`的方式，这样做既可以消除对方法传递了哪种类型参数的猜测的复杂性，又可以为其他选项留出空间，即现在可用于根据每个语句提供选项的`Connection.execute.execution_options` 字典。该方法也经过修改，以使其使用模式与`Session.execute()`方法相匹配，后者在 2.0 风格中是一个更加突出的 API。

删除直接字符串 SQL 是为了解决`Connection.execute()`和`Session.execute()`之间的不一致性，前者情况下字符串直接传递给驱动程序原始，而后者情况下首先将其转换为`text()`构造。通过仅允许`text()`，这也将接受的参数格式限制为“命名”而不是“位置”。最后，从安全角度来看，字符串 SQL 用例越来越受到审查，`text()`构造已经成为文本 SQL 领域的明确边界，需要注意不受信任的用户输入。

### 结果行的行为类似于命名元组

**简介**

版本 1.4 引入了一个全新的 Result 对象，该对象反过来返回 `Row` 对象，当使用“future”模式时，它们的行为类似于命名元组：

```py
engine = create_engine(..., future=True)  # using future mode

with engine.connect() as conn:
    result = conn.execute(text("select x, y from table"))

    row = result.first()  # suppose the row is (1, 2)

    "x" in row  # evaluates to False, in 1.x / future=False, this would be True

    1 in row  # evaluates to True, in 1.x / future=False, this would be False
```

**迁移到 2.0**

应用程序代码或测试套件，如果正在测试行中是否存在特定键，则需要测试 `row.keys()` 集合。但是，这是一个不太常见的用例，因为结果行通常由已经知道其中存在哪些列的代码使用。

**讨论**

已经作为 1.4 的一部分，之前从 `Query` 对象中选择行时使用的 `KeyedTuple` 类已被替换为 `Row` 类，该类是使用 `create_engine.future` 标志与 `Engine` 一起使用 Core 语句结果时返回的相同 `Row` 的基类（当未设置 `create_engine.future` 标志时，Core 结果集使用 `LegacyRow` 子类，该子类维护了 `__contains__()` 方法的向后兼容行为；ORM 专门直接使用 `Row` 类）。

这个 `Row` 的行为类似于命名元组，因为它作为序列，但也支持属性名访问，例如 `row.some_column`。然而，它还通过特殊属性 `row._mapping` 提供了之前的“映射”行为，该属性产生一个 Python 映射，使得可以使用基于键的访问，如 `row["some_column"]`。

为了一开始就接收到映射的结果，可以在结果上使用 `mappings()` 修改器：

```py
from sqlalchemy.future.orm import Session

session = Session(some_engine)

result = session.execute(stmt)
for row in result.mappings():
    print("the user is: %s" % row["User"])
```

ORM 中使用的 `Row` 类也支持通过实体或属性进行访问：

```py
from sqlalchemy.future import select

stmt = select(User, Address).join(User.addresses)

for row in session.execute(stmt).mappings():
    print("the user is: %s the address is: %s" % (row[User], row[Address]))
```

另请参阅

RowProxy 不再是“代理”; 现在称为 Row，并且行为类似于增强型命名元组

## 2.0 迁移 - Core 使用

### `select()` 不再接受不同的构造函数参数，列被按位置传递

**简介**

`select()` 构造以及相关方法 `FromClause.select()` 将不再接受用于构建 WHERE 子句、FROM 列表和 ORDER BY 等元素的关键字参数。列的列表现在可以按位置发送，而不是作为列表。此外，`case()` 构造现在按位置接受其 WHEN 标准，而不是作为列表：

```py
# select_from / order_by keywords no longer supported
stmt = select([1], select_from=table, order_by=table.c.id)

# whereclause parameter no longer supported
stmt = select([table.c.x], table.c.id == 5)

# whereclause parameter no longer supported
stmt = table.select(table.c.id == 5)

# list emits a deprecation warning
stmt = select([table.c.x, table.c.y])

# list emits a deprecation warning
case_clause = case(
    [(table.c.x == 5, "five"), (table.c.x == 7, "seven")],
    else_="neither five nor seven",
)
```

**迁移到 2.0**

只支持 `select()` 的“生成”样式。应按位置传递要从中选择的列 / 表的列表。SQLAlchemy 1.4 中的 `select()` 构造接受传统样式和新样式，使用自动检测方案，因此下面的代码与 1.4 和 2.0 兼容：

```py
# use generative methods
stmt = select(1).select_from(table).order_by(table.c.id)

# use generative methods
stmt = select(table).where(table.c.id == 5)

# use generative methods
stmt = table.select().where(table.c.id == 5)

# pass columns clause expressions positionally
stmt = select(table.c.x, table.c.y)

# case conditions passed positionally
case_clause = case(
    (table.c.x == 5, "five"), (table.c.x == 7, "seven"), else_="neither five nor seven"
)
```

**讨论**

多年来，SQLAlchemy 发展了一个约定，即 SQL 构造接受参数时可以作为列表或按位置参数传递。该约定规定了形成 SQL 语句结构的**结构**元素应**按位置**传递。相反，形成 SQL 语句参数化数据的**数据**元素应作为**列表**传递。多年来，`select()` 构造无法顺利参与此约定，因为“WHERE”子句的传递模式非常传统。SQLAlchemy 2.0 最终通过将 `select()` 构造更改为仅接受多年来一直是 Core 教程中唯一记录的样式的“生成”样式来解决了这个问题。

“结构”与“数据”元素的示例如下：

```py
# table columns for CREATE TABLE - structural
table = Table("table", metadata_obj, Column("x", Integer), Column("y", Integer))

# columns in a SELECT statement - structural
stmt = select(table.c.x, table.c.y)

# literal elements in an IN clause - data
stmt = stmt.where(table.c.y.in_([1, 2, 3]))
```

另请参阅

select()，case() 现在接受位置表达式

以“传统”模式创建的 select() 构造; 关键字参数等

### insert/update/delete DML 不再接受关键字构造函数参数

**简介**

与之前对 `select()` 的更改类似，除了表参数之外，`insert()`、`update()` 和 `delete()` 的构造函数参数基本上被移除：

```py
# no longer supported
stmt = insert(table, values={"x": 10, "y": 15}, inline=True)

# no longer supported
stmt = insert(table, values={"x": 10, "y": 15}, returning=[table.c.x])

# no longer supported
stmt = table.delete(table.c.x > 15)

# no longer supported
stmt = table.update(table.c.x < 15, preserve_parameter_order=True).values(
    [(table.c.y, 20), (table.c.x, table.c.y + 10)]
)
```

**迁移到 2.0**

以下示例说明了上述示例的生成方法使用：

```py
# use generative methods, **kwargs OK for values()
stmt = insert(table).values(x=10, y=15).inline()

# use generative methods, dictionary also still  OK for values()
stmt = insert(table).values({"x": 10, "y": 15}).returning(table.c.x)

# use generative methods
stmt = table.delete().where(table.c.x > 15)

# use generative methods, ordered_values() replaces preserve_parameter_order
stmt = (
    table.update()
    .where(
        table.c.x < 15,
    )
    .ordered_values((table.c.y, 20), (table.c.x, table.c.y + 10))
)
```

**讨论**

API 和内部结构正在以类似于 `select()` 构造的方式进行简化。

## 2.0 迁移 - ORM 配置

### Declarative 成为一流的 API

**简介**

`sqlalchemy.ext.declarative` 包大部分已经移动到 `sqlalchemy.orm` 包中，但有一些例外。`declarative_base()` 和 `declared_attr()` 函数没有任何行为上的变化。一个名为 `registry` 的新超级实现现在作为顶级 ORM 配置构造存在，它还提供了基于装饰器的声明性以及与声明式注册表集成的经典映射的新支持。

**迁移到 2.0**

更改导入：

```py
from sqlalchemy.ext import declarative_base, declared_attr
```

至：

```py
from sqlalchemy.orm import declarative_base, declared_attr
```

**讨论**

大约十年后，`sqlalchemy.ext.declarative` 包现在已经整合到 `sqlalchemy.orm` 命名空间中，除了保留为 Declarative 扩展的声明性 “extension” 类。更改在 1.4 迁移指南中进一步详细说明，详见 Declarative is now integrated into the ORM with new features。

另请参阅

ORM 映射类概述 - 用于 Declarative、经典映射、数据类、attrs 等的全新统一文档。

Declarative is now integrated into the ORM with new features

### 最初的 “mapper()” 函数现在是 Declarative 的核心元素，已更名

**简介**

`sqlalchemy.orm.mapper()` 独立函数在幕后移动，由高级 API 调用。这个函数的新版本是从 `registry` 对象获取的方法 `registry.map_imperatively()`。

**迁移到 2.0**

使用经典映射的代码应从以下导入和代码更改：

```py
from sqlalchemy.orm import mapper

mapper(SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)})
```

要从中央`registry`对象工作：

```py
from sqlalchemy.orm import registry

mapper_reg = registry()

mapper_reg.map_imperatively(
    SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)}
)
```

上述`registry`也是声明式映射的源头，经典映射现在可以访问此注册表，包括在`relationship()`上进行基于字符串的配置：

```py
from sqlalchemy.orm import registry

mapper_reg = registry()

Base = mapper_reg.generate_base()

class SomeRelatedClass(Base):
    __tablename__ = "related"

    # ...

mapper_reg.map_imperatively(
    SomeClass,
    some_table,
    properties={
        "related": relationship(
            "SomeRelatedClass",
            primaryjoin="SomeRelatedClass.related_id == SomeClass.id",
        )
    },
)
```

**讨论**

受到广泛需求，“经典映射”仍然存在，但其新形式基于`registry`对象，并可作为`registry.map_imperatively()`使用。

此外，“经典映射”的主要理由是保持`Table`设置与类别不同。声明式一直允许使用所谓的混合声明式来使用这种样式。然而，为了去除基类要求，已经添加了第一类装饰器形式。

又一个独立但相关的增强功能是，支持 Python 数据类已添加到声明装饰器和经典映射形式中。

另请参阅

ORM 映射类概述 - 用于声明式、经典映射、数据类、attrs 等的全新统一文档。

## 2.0 迁移 - ORM 使用

SQLAlchemy 2.0 中最显着的可见变化是使用`Session.execute()`与`select()`一起运行 ORM 查询，而不是使用`Session.query()`。正如在其他地方提到的，实际上没有计划真正删除`Session.query()` API 本身，因为现在它是通过内部使用新 API 来实现的，它将保留为遗留 API，并且两个 API 可以自由使用。

下表介绍了调用形式的一般变化，并链接到每个技术的文档。嵌入部分中的个别迁移注释位于表之后，并可能包含此处未概括的其他注释。

**主要 ORM 查询模式概述**

| 1.x 风格表单 | 2.0 风格表单 | 另请参阅 |
| --- | --- | --- |

|

```py
session.query(User).get(42)
```

|

```py
session.get(User, 42)
```

| ORM 查询 - get()方法移到 Session |
| --- |

|

```py
session.query(User).all()
```

|

```py
session.execute(
  select(User)
).scalars().all()

# or

session.scalars(
  select(User)
).all()
```

| 使用核心选择统一的 ORM 查询`Session.scalars()` `Result.scalars()` |
| --- |

|

```py
session.query(User).\
  filter_by(name="some user").\
  one()
```

|

```py
session.execute(
  select(User).
  filter_by(name="some user")
).scalar_one()
```

| 使用核心选择统一的 ORM 查询`Result.scalar_one()` |
| --- |

|

```py
session.query(User).\
  filter_by(name="some user").\
  first()
```

|

```py
session.scalars(
  select(User).
  filter_by(name="some user").
  limit(1)
).first()
```

| 使用核心选择统一的 ORM 查询`Result.first()` |
| --- |

|

```py
session.query(User).options(
  joinedload(User.addresses)
).all()
```

|

```py
session.scalars(
  select(User).
  options(
    joinedload(User.addresses)
  )
).unique().all()
```

| 默认情况下，ORM 行不是唯一的 |
| --- |

|

```py
session.query(User).\
  join(Address).\
  filter(
    Address.email == "e@sa.us"
  ).\
  all()
```

|

```py
session.execute(
  select(User).
  join(Address).
  where(
    Address.email == "e@sa.us"
  )
).scalars().all()
```

| 使用核心选择统一的 ORM 查询连接 |
| --- |

|

```py
session.query(User).\
  from_statement(
    text("select * from users")
  ).\
  all()
```

|

```py
session.scalars(
  select(User).
  from_statement(
    text("select * from users")
  )
).all()
```

| 从文本语句获取 ORM 结果 |
| --- |

|

```py
session.query(User).\
  join(User.addresses).\
  options(
    contains_eager(User.addresses)
  ).\
  populate_existing().all()
```

|

```py
session.execute(
  select(User)
  .join(User.addresses)
  .options(
    contains_eager(User.addresses)
  )
  .execution_options(
      populate_existing=True
  )
).scalars().all()
```

| ORM 执行选项填充现有 |
| --- |

|

```py
session.query(User).\
  filter(User.name == "foo").\
  update(
    {"fullname": "Foo Bar"},
    synchronize_session="evaluate"
  )
```

|

```py
session.execute(
  update(User)
  .where(User.name == "foo")
  .values(fullname="Foo Bar")
  .execution_options(
    synchronize_session="evaluate"
  )
)
```

| 启用 ORM 的 INSERT、UPDATE 和 DELETE 语句 |
| --- |

|

```py
session.query(User).count()
```

|

```py
session.scalar(
  select(func.count()).
  select_from(User)
)

# or

session.scalar(
  select(func.count(User.id))
)
```

| `Session.scalar()` |
| --- |

### ORM 查询与核心选择统一

**概要**

`Query`对象（以及`BakedQuery`和`ShardedQuery`扩展）成为长期的遗留对象，被直接使用`select()`构造与`Session.execute()`方法取代。从`Query`返回的对象，以列表形式返回对象或元组，或作为标量 ORM 对象统一地作为`Session.execute()`返回的`Result`对象，其接口与 Core 执行一致。

下面是旧代码示例：

```py
session = Session(engine)

# becomes legacy use case
user = session.query(User).filter_by(name="some user").one()

# becomes legacy use case
user = session.query(User).filter_by(name="some user").first()

# becomes legacy use case
user = session.query(User).get(5)

# becomes legacy use case
for user in (
    session.query(User).join(User.addresses).filter(Address.email == "some@email.com")
):
    ...

# becomes legacy use case
users = session.query(User).options(joinedload(User.addresses)).order_by(User.id).all()

# becomes legacy use case
users = session.query(User).from_statement(text("select * from users")).all()

# etc
```

**迁移到 2.0 版本**

由于 ORM 应用的绝大部分预计都将使用`Query`对象，并且`Query`接口的可用性不影响新接口，该对象将在 2.0 版本中保留，但不再是文档的一部分，也大部分不再受支持。现在，`select()`构造适用于核心和 ORM 用例，当通过`Session.execute()`方法调用时，将返回 ORM 导向的结果，即如果请求的是 ORM 对象。

`Select()`构造**添加了许多新方法**，以与`Query`兼容，包括`Select.filter()`、`Select.filter_by()`、重新设计的`Select.join()`和`Select.outerjoin()`方法、`Select.options()`等。其他更多的`Query`的补充方法，如`Query.populate_existing()`，通过执行选项实现。

返回结果以`Result`对象的形式表示，这是 SQLAlchemy `ResultProxy`对象的新版本，它也添加了许多新方法，以与`Query`兼容，包括`Result.one()`、`Result.all()`、`Result.first()`、`Result.one_or_none()`等。

但是，`Result` 对象需要一些不同的调用模式，首次返回时它将**始终返回元组**，并且**不会在内存中去重结果**。为了以 `Query` 的方式返回单个 ORM 对象，必须首先调用 `Result.scalars()` 修改器。为了返回唯一对象，就像在使用连接式急加载时所必需的那样，必须首先调用 `Result.unique()` 修改器。

所有新特性的文档`select()`，包括执行选项等，位于 ORM 查询指南。

下面是一些迁移到 `select()` 的示例：

```py
session = Session(engine)

user = session.execute(select(User).filter_by(name="some user")).scalar_one()

# for first(), no LIMIT is applied automatically; add limit(1) if LIMIT
# is desired on the query
user = (
    session.execute(select(User).filter_by(name="some user").limit(1)).scalars().first()
)

# get() moves to the Session directly
user = session.get(User, 5)

for user in session.execute(
    select(User).join(User.addresses).filter(Address.email == "some@email.case")
).scalars():
    ...

# when using joinedload() against collections, use unique() on the result
users = (
    session.execute(select(User).options(joinedload(User.addresses)).order_by(User.id))
    .unique()
    .all()
)

# select() has ORM-ish methods like from_statement() that only work
# if the statement is against ORM entities
users = (
    session.execute(select(User).from_statement(text("select * from users")))
    .scalars()
    .all()
)
```

**讨论**

SQLAlchemy 同时具有 `select()` 构造和一个单独的 `Query` 对象，它具有极其相似但基本上不兼容的接口，这可能是 SQLAlchemy 中最大的不一致性，这是由于随着时间的推移，小的增量添加累积成了两个主要的不同的 API。

在 SQLAlchemy 的最初版本中，根本不存在 `Query` 对象。最初的想法是 `Mapper` 构造本身将能够选择行，并且 `Table` 对象而不是类将用于在 Core 风格的方法中创建各种条件。`Query` 是在 SQLAlchemy 的历史中的某个时期，作为用户建议的一个新的、“可构建”的查询对象被接受的。在之前的 SQLAlchemy 中，像 `.where()` 方法这样的概念，`SelectResults` 称为 `.filter()`，是不存在的，而 `select()` 构造只使用了现在已经弃用的“一次全部”构造样式，该样式已在 select() no longer accepts varied constructor arguments, columns are passed positionally 中不再接受各种构造函数参数。

随着新方法的推出，该对象逐渐演变成为 `Query` 对象，随着新功能的添加，例如能够选择单个列、能够一次选择多个实体、能够从 `Query` 对象而不是从 `select` 对象构建子查询等。目标是 `Query` 应该具有 `select` 的所有功能，即它可以被组合以完全构建 SELECT 语句，不需要显式使用 `select()`。与此同时，`select()` 也发展出了像 `Select.where()` 和 `Select.order_by()` 这样的“生成”方法。

在现代 SQLAlchemy 中，已经实现了这个目标，这两个对象现在在功能上完全重叠。统一这些对象的主要挑战是 `select()` 对象需要保持**对 ORM 完全不可知**。为了实现这一点，大部分来自 `Query` 的逻辑已经移动到 SQL 编译阶段，其中 ORM 特定的编译器插件接收 `Select` 构造并按照 ORM 风格的查询解释其内容，然后传递给核心级别的编译器以创建 SQL 字符串。随着新的 SQL 编译缓存系统的出现，大部分这种 ORM 逻辑也被缓存了。

另请参阅

ORM 查询与 select、update、delete 内部统一；2.0 风格执行可用  ### ORM 查询 - get() 方法移到会话

**概要**

`Query.get()` 方法仍然保留出于遗留目的，但主要接口现在是 `Session.get()` 方法：

```py
# legacy usage
user_obj = session.query(User).get(5)
```

**迁移到 2.0**

在 1.4 / 2.0 中，`Session`对象新增了一个新的`Session.get()`方法：

```py
# 1.4 / 2.0 cross-compatible use
user_obj = session.get(User, 5)
```

**讨论**

`Query`对象在 2.0 中将成为传统对象，因为 ORM 查询现在可以使用`select()`对象。由于`Query.get()`方法与`Session`定义了一种特殊的交互，并且甚至不一定会发出查询，因此更适合将其作为`Session`的一部分，其中它类似于其他“身份”方法，例如`refresh`和`merge`。

SQLAlchemy 最初包含了 "get()" 来模仿 Hibernate 的 `Session.load()` 方法。就像经常发生的那样，我们稍微弄错了，因为这个方法实际上更多地与`Session`有关，而不是编写 SQL 查询。

**概要**

这指的是诸如`Query.join()`之类的模式，以及像`joinedload()`这样的查询选项，它们目前接受字符串属性名称或实际类属性的混合。字符串形式将在 2.0 中全部被移除：

```py
# string use removed
q = session.query(User).join("addresses")

# string use removed
q = session.query(User).options(joinedload("addresses"))

# string use removed
q = session.query(Address).filter(with_parent(u1, "addresses"))
```

**迁移到 2.0**

现代 SQLAlchemy 1.x 版本支持推荐的技术，即使用映射属性：

```py
# compatible with all modern SQLAlchemy versions

q = session.query(User).join(User.addresses)

q = session.query(User).options(joinedload(User.addresses))

q = session.query(Address).filter(with_parent(u1, User.addresses))
```

同样的技术适用于 2.0 风格的使用：

```py
# SQLAlchemy 1.4 / 2.0 cross compatible use

stmt = select(User).join(User.addresses)
result = session.execute(stmt)

stmt = select(User).options(joinedload(User.addresses))
result = session.execute(stmt)

stmt = select(Address).where(with_parent(u1, User.addresses))
result = session.execute(stmt)
```

**讨论**

字符串调用形式不明确，并且需要内部额外工作以确定适当的路径并检索正确的映射属性。通过直接传递 ORM 映射属性，不仅需要传递必要的信息，而且属性还是经过类型化的，并且更可能与 IDE 和 pep-484 集成兼容。

### ORM 查询 - 使用属性列表进行链接的链式形式，而不是单独调用，已移除

**概要**

关于 ORM 查询 - 加入 / 加载关系使用属性，而不是字符串的方式将被移除：

```py
# chaining removed
q = session.query(User).join("orders", "items", "keywords")
```

**迁移到 2.0**

使用单独的调用`Query.join()`进行 1.x /2.0 交叉兼容使用：

```py
q = session.query(User).join(User.orders).join(Order.items).join(Item.keywords)
```

对于 2.0 风格的用法，`Select`具有与`Select.join()`相同的行为，还具有一个新的`Select.join_from()`方法，允许显式左侧：

```py
# 1.4 / 2.0 cross compatible

stmt = select(User).join(User.orders).join(Order.items).join(Item.keywords)
result = session.execute(stmt)

# join_from can also be helpful
stmt = select(User).join_from(User, Order).join_from(Order, Item, Order.items)
result = session.execute(stmt)
```

**讨论**

移除属性链接的操作符符合简化方法调用接口的原则，比如`Select.join()`。

### ORM 查询 - join(…, aliased=True)，from_joinpoint 已移除

**概要**

在`Query.join()`上的`aliased=True`选项已被移除，`from_joinpoint`标志也被移除：

```py
# no longer supported
q = (
    session.query(Node)
    .join("children", aliased=True)
    .filter(Node.name == "some sub child")
    .join("children", from_joinpoint=True, aliased=True)
    .filter(Node.name == "some sub sub child")
)
```

**迁移到 2.0**

使用显式别名代替：

```py
n1 = aliased(Node)
n2 = aliased(Node)

q = (
    select(Node)
    .join(Node.children.of_type(n1))
    .where(n1.name == "some sub child")
    .join(n1.children.of_type(n2))
    .where(n2.name == "some sub child")
)
```

**讨论**

在广泛的代码搜索中，几乎没有人使用`Query.join()`上的`aliased=True`选项，这个特性的内部复杂性是**巨大**的，并且将在 2.0 版本中被移除。

大多数用户不熟悉这个标志，但它允许自动为连接的元素进行别名处理，然后将自动别名应用于过滤条件。最初的用例是帮助处理长链的自引用连接，就像上面显示的示例一样。然而，过滤条件的自动调整在内部非常复杂，几乎从不在实际应用中使用。这种模式还会导致问题，比如如果需要在链中的每个链接处添加过滤条件；那么模式必须使用`from_joinpoint`标志，而 SQLAlchemy 开发人员绝对找不到这个参数在实际应用中的任何使用情况。

`aliased=True`和`from_joinpoint`参数是在`Query`对象还没有良好的能力进行沿着关系属性连接时开发的，像`PropComparator.of_type()`这样的函数还不存在，而`aliased()`构造本身在早期也不存在。### 使用 DISTINCT 与额外列，但仅选择实体

**概要**

当使用 DISTINCT 时，`Query` 将自动在 ORDER BY 中添加列。以下查询将从所有用户列以及“address.email_address”中选择，但仅返回用户对象：

```py
# 1.xx code

result = (
    session.query(User)
    .join(User.addresses)
    .distinct()
    .order_by(Address.email_address)
    .all()
)
```

在 2.0 版本中，“email_address”列将不会自动添加到列子句中，上述查询将失败，因为当使用 DISTINCT 时，关系型数据库不允许您按“address.email_address”排序，如果它不在列子句中的话。

**升级到 2.0**

在 2.0 中，必须显式添加列。为了解决仅返回主实体对象而不是额外列的问题，使用 `Result.columns()` 方法：

```py
# 1.4 / 2.0 code

stmt = (
    select(User, Address.email_address)
    .join(User.addresses)
    .distinct()
    .order_by(Address.email_address)
)

result = session.execute(stmt).columns(User).all()
```

**讨论**

此案例是`Query`灵活性有限的一个示例，导致需要添加隐式的“神奇”行为的情况；“email_address”列隐式地添加到列子句中，然后额外的内部逻辑将从实际返回的结果中省略该列。

新方法简化了交互，并使正在进行的操作明确，同时仍然可以实现原始用例而不会带来不便。### 从查询本身作为子查询进行选择，例如“from_self()”

**概要**

`Query.from_self()` 方法将从 `Query` 中移除：

```py
# from_self is removed
q = (
    session.query(User, Address.email_address)
    .join(User.addresses)
    .from_self(User)
    .order_by(Address.email_address)
)
```

**升级到 2.0**

`aliased()` 构造可以用于对基于任意可选择的实体发出 ORM 查询。在版本 1.4 中，它已经增强，以便对同一个子查询多次进行不同实体的使用。可以在 1.x 样式 中与 `Query` 一起使用如下；请注意，由于最终查询想要查询关于 `User` 和 `Address` 实体的内容，因此会创建两个单独的 `aliased()` 构造：

```py
from sqlalchemy.orm import aliased

subq = session.query(User, Address.email_address).join(User.addresses).subquery()

ua = aliased(User, subq)

aa = aliased(Address, subq)

q = session.query(ua, aa).order_by(aa.email_address)
```

可以在 2.0 样式 中使用相同的形式：

```py
from sqlalchemy.orm import aliased

subq = select(User, Address.email_address).join(User.addresses).subquery()

ua = aliased(User, subq)

aa = aliased(Address, subq)

stmt = select(ua, aa).order_by(aa.email_address)

result = session.execute(stmt)
```

**讨论**

`Query.from_self()`方法是一个非常复杂的方法，很少被使用。该方法的目的是将一个`Query`转换为一个子查询，然后返回一个从该子查询中 SELECT 的新的`Query`。该方法的精彩之处在于返回的查询应用了 ORM 实体和列的自动转换，以便以子查询的形式在 SELECT 中声明，以及它允许修改要从中 SELECT 的实体和列。

因为`Query.from_self()`将大量的隐式转换打包到其生成的 SQL 中，虽然它确实允许某种类型的模式被执行得非常简洁，但该方法的实际应用很少，因为它不容易理解。

新方法利用了`aliased()`构造，使得 ORM 内部不需要猜测应该如何适应哪些实体和列；在上面的例子中，`ua`和`aa`对象，都是`AliasedClass`实例，为内部提供了一个明确的标记，表明子查询应该被引用以及正在考虑的查询组件的哪个实体列或关系。

SQLAlchemy 1.4 还具有改进的标签样式，不再需要使用包含表名以消除来自不同表的相同名称列的歧义的长标签。在上面的示例中，即使我们的`User`和`Address`实体具有重叠的列名，我们也可以一次从两个实体中选择，而无需指定任何特定的标签：

```py
# 1.4 / 2.0 code

subq = select(User, Address).join(User.addresses).subquery()

ua = aliased(User, subq)
aa = aliased(Address, subq)

stmt = select(ua, aa).order_by(aa.email_address)
result = session.execute(stmt)
```

上述查询将消除`User`和`Address`的`.id`列的歧义，其中`Address.id`被呈现和追踪为`id_1`：

```py
SELECT  anon_1.id  AS  anon_1_id,  anon_1.id_1  AS  anon_1_id_1,
  anon_1.user_id  AS  anon_1_user_id,
  anon_1.email_address  AS  anon_1_email_address
FROM  (
  SELECT  "user".id  AS  id,  address.id  AS  id_1,
  address.user_id  AS  user_id,  address.email_address  AS  email_address
  FROM  "user"  JOIN  address  ON  "user".id  =  address.user_id
)  AS  anon_1  ORDER  BY  anon_1.email_address
```

[#5221](https://www.sqlalchemy.org/trac/ticket/5221)

### 从替代选择中选择实体；Query.select_entity_from()

**概要**

`Query.select_entity_from()`方法将在 2.0 中被移除：

```py
subquery = session.query(User).filter(User.id == 5).subquery()

user = session.query(User).select_entity_from(subquery).first()
```

**迁移到 2.0**

如在查询本身作为子查询中选择，例如“from_self()”所述，`aliased()`对象提供了一个单一的地方，可以实现诸如“从子查询中选择实体”之类的操作。使用 1.x 风格：

```py
from sqlalchemy.orm import aliased

subquery = session.query(User).filter(User.name.like("%somename%")).subquery()

ua = aliased(User, subquery)

user = session.query(ua).order_by(ua.id).first()
```

使用 2.0 风格：

```py
from sqlalchemy.orm import aliased

subquery = select(User).where(User.name.like("%somename%")).subquery()

ua = aliased(User, subquery)

# note that LIMIT 1 is not automatically supplied, if needed
user = session.execute(select(ua).order_by(ua.id).limit(1)).scalars().first()
```

**讨论**

这里的观点基本与从查询本身选择为子查询，例如“from_self()”讨论的观点相同。`Query.select_from_entity()`方法是指示查询从替代可选择的 ORM 映射实体加载行的另一种方式，其中涉及 ORM 在稍后在查询中使用该实体时自动为该实体应用别名，例如在 WHERE 子句或 ORDER BY 中。这个极其复杂的功能很少被这种方式使用，就像`Query.from_self()`的情况一样，当使用显式`aliased()`对象时，无论是从用户的角度还是从 SQLAlchemy ORM 内部处理的角度来看，都更容易跟踪发生了什么。

### ORM 行默认情况下不唯一

**概要**

通过`session.execute(stmt)`返回的 ORM 行不再自动“唯一化”。这通常是一个受欢迎的变化，除非使用了“联接贪婪加载”加载器策略与集合：

```py
# In the legacy API, many rows each have the same User primary key, but
# only one User per primary key is returned
users = session.query(User).options(joinedload(User.addresses))

# In the new API, uniquing is available but not implicitly
# enabled
result = session.execute(select(User).options(joinedload(User.addresses)))

# this actually will raise an error to let the user know that
# uniquing should be applied
rows = result.all()
```

**迁移到 2.0**

在使用集合的联接加载时，需要调用`Result.unique()`方法。ORM 实际上会设置一个默认的行处理程序，如果未执行此操作，它将引发错误，以确保联接贪婪加载集合不返回重复行，同时保持明确性：

```py
# 1.4 / 2.0 code

stmt = select(User).options(joinedload(User.addresses))

# statement will raise if unique() is not used, due to joinedload()
# of a collection.  in all other cases, unique() is not needed.
# By stating unique() explicitly, confusion over discrepancies between
# number of objects/ rows returned vs. "SELECT COUNT(*)" is resolved
rows = session.execute(stmt).unique().all()
```

**讨论**

这里的情况有点不同寻常，因为 SQLAlchemy 要求调用一个它完全可以自动执行的方法。要求调用该方法的原因是确保开发者“选择”使用`Result.unique()`方法，这样他们在直接计算行数与实际结果集中的记录数不冲突时不会感到困惑，这已经是多年来用户困惑和错误报告的长期问题了。默认情况下不会在任何其他情况下进行唯一化，这将提高性能，并在自动唯一化导致混淆结果的情况下提高清晰度。

到目前为止，当使用联接贪婪加载集合时需要调用`Result.unique()`有些不方便，在现代 SQLAlchemy 中，`selectinload()`策略提供了一个面向集合的贪婪加载器，它在大多数方面都优于`joinedload()`，应优先使用。 ### “动态”关系加载器被“仅写”取代

**概要**

讨论 动态关系加载器 中讨论的 `lazy="dynamic"` 关系加载策略使用了在 2.0 中已经过时的 `Query` 对象。 “dynamic” 关系在没有解决方法的情况下无法直接兼容 asyncio，此外，它也不能实现其原始目的，即防止大型集合的迭代，因为它有几种隐式迭代的行为。

引入了一种新的加载策略，称为 `lazy="write_only"`，通过 `WriteOnlyCollection` 集合类提供了一个非常严格的“无隐式迭代”的 API，此外，还与 2.0 风格的语句执行集成，支持 asyncio 以及与新的 启用 ORM 的批量 DML 特性集成。

同时，在 2.0 版本中`lazy="dynamic"` 仍然**完全支持**；应用程序可以延迟将这种特定模式迁移到完全使用 2.0 版本的时候。

**迁移到 2.0**

新的“只写”功能仅在 SQLAlchemy 2.0 中可用，并不是 1.4 的一部分。同时，`lazy="dynamic"` 加载策略在 2.0 版本中仍然得到充分支持，甚至包括了新的 pep-484 和带注释的映射支持。

因此，从 “dynamic” 迁移到 2.0 的最佳策略是**等到应用程序完全运行在 2.0 上**，然后直接从 `AppenderQuery` 迁移到 `WriteOnlyCollection`，它是 “write_only” 策略使用的集合类型。

有一些技巧可以在 1.4 中以更“2.0”的风格使用 `lazy="dynamic"`。有两种方法可以实现基于特定关系的 2.0 风格的查询：

+   利用现有的 `lazy="dynamic"` 关系的 `Query.statement` 属性。我们可以立即像下面这样直接使用动态加载器的方法，比如 `Session.scalars()`：

    ```py
    class User(Base):
        __tablename__ = "user"

        posts = relationship(Post, lazy="dynamic")

    jack = session.get(User, 5)

    # filter Jack's blog posts
    posts = session.scalars(jack.posts.statement.where(Post.headline == "this is a post"))
    ```

+   使用 `with_parent()` 函数直接构造一个 `select()` 构造：

    ```py
    from sqlalchemy.orm import with_parent

    jack = session.get(User, 5)

    posts = session.scalars(
        select(Post)
        .where(with_parent(jack, User.posts))
        .where(Post.headline == "this is a post")
    )
    ```

**讨论**

最初的想法是`with_parent()`函数应该足以满足需求，然而继续利用关系本身的特殊属性仍然有吸引力，而且没有理由不能使 2.0 样式的构造在这里起作用。

新的“write_only”加载策略提供了一种新的集合类型，它不支持隐式迭代或项目访问。相反，通过调用其`.select()`方法来读取集合的内容，以帮助构造一个适当的 SELECT 语句。该集合还包括`.insert()`、`.update()`、`.delete()`方法，可用于对集合中的项目发出批量 DML 语句。与“dynamic”功能类似，还有`.add()`、`.add_all()`和`.remove()`方法，它们通过工作单元流程为单个成员排队以进行添加或移除。有关新功能的介绍如下 新的“仅写”关系策略取代了“动态”。

另请参阅

新的“仅写”关系策略取代了“动态”

仅写关系  ### 从会话中移除自动提交模式；添加了自动开始支持

**简介**

`Session`将不再支持“自动提交”模式，即这种模式：

```py
from sqlalchemy.orm import Session

sess = Session(engine, autocommit=True)

# no transaction begun, but emits SQL, won't be supported
obj = sess.query(Class).first()

# session flushes in a transaction that it begins and
# commits, won't be supported
sess.flush()
```

**迁移到 2.0**

在“自动提交”模式下使用`Session`的主要原因是使得`Session.begin()`方法可用，以便框架集成和事件钩子可以控制此事件发生的时间。在 1.4 中，`Session`现在具有自动开始行为来解决这个问题；现在可以调用`Session.begin()`方法了：

```py
from sqlalchemy.orm import Session

sess = Session(engine)

sess.begin()  # begin explicitly; if not called, will autobegin
# when database access is needed

sess.add(obj)

sess.commit()
```

**讨论**

“自动提交”模式是 SQLAlchemy 最初版本的另一个遗留问题。这个标志主要保留下来以支持允许显式使用`Session.begin()`，这个问题现在已经在 1.4 中解决了，以及允许使用“子事务”，这在 2.0 中也已经移除。### 会话“子事务”行为已移除

**简介**

“子事务”模式在 1.4 版本中已经不推荐使用，这种模式经常与自动提交模式一起使用。这种模式允许在事务已经开始时使用`Session.begin()`方法，导致产生一个称为“子事务”的结构，本质上是一个阻止`Session.commit()`方法实际提交的块。

**迁移到 2.0**

为了为使用这种模式的应用程序提供向后兼容性，可以使用以下上下文管理器或基于装饰器的类似实现：

```py
import contextlib

@contextlib.contextmanager
def transaction(session):
    if not session.in_transaction():
        with session.begin():
            yield
    else:
        yield
```

上述上下文管理器可以像“子事务”标志一样使用，例如以下示例：

```py
# method_a starts a transaction and calls method_b
def method_a(session):
    with transaction(session):
        method_b(session)

# method_b also starts a transaction, but when
# called from method_a participates in the ongoing
# transaction.
def method_b(session):
    with transaction(session):
        session.add(SomeObject("bat", "lala"))

Session = sessionmaker(engine)

# create a Session and call method_a
with Session() as session:
    method_a(session)
```

为了与首选的惯用模式进行比较，begin 块应该在最外层。这样就不需要单独的函数或方法关注事务划分的细节：

```py
def method_a(session):
    method_b(session)

def method_b(session):
    session.add(SomeObject("bat", "lala"))

Session = sessionmaker(engine)

# create a Session and call method_a
with Session() as session:
    with session.begin():
        method_a(session)
```

**讨论**

这种模式已经被证明在实际应用中令人困惑，最好是确保应用程序的最顶层数据库操作使用单个 begin/commit 对执行。

## 2.0 迁移 - ORM 扩展和配方更改

### Dogpile 缓存配方和水平分片使用新的 Session API

随着`Query`对象变得过时，之前依赖于`Query`对象子类化的这两个方法现在使用`SessionEvents.do_orm_execute()`钩子。请参阅重新执行语句部分以获取示例。

### 烘焙查询扩展被内置缓存所取代

烘焙查询扩展被内置缓存系统取代，不再被 ORM 内部使用。

请查看 SQL 编译缓存以获取新缓存系统的完整背景。

## 异步 IO 支持

SQLAlchemy 1.4 包括 Core 和 ORM 的异步 IO 支持。新 API 专门使用上述“future”模式。请参阅 Core 和 ORM 的异步 IO 支持以获取背景信息。

## 1.4->2.0 迁移路径

被认为是“SQLAlchemy 2.0”的最突出的架构特性和 API 更改实际上在 1.4 系列中已经完全可用，以提供从 1.x 到 2.x 系列的清晰升级路径，同时作为这些功能的 beta 平台。这些更改包括：

+   新的 ORM 语句范式

+   Core 和 ORM 中的 SQL 缓存

+   新的声明性特性，ORM 集成

+   新的 Result 对象

+   select() / case()接受位置表达式

+   Core 和 ORM 的 asyncio 支持

上述要点链接到在 SQLAlchemy 1.4 中介绍的这些新范例的描述中。在 SQLAlchemy 1.4 有什么新功能？文档中。

对于 SQLAlchemy 2.0，所有标记为 2.0 不推荐使用的 API 功能和行为现已最终确定；特别是**不再存在**的主要 API 包括：

+   绑定的 MetaData 和无连接执行

+   连接上的模拟自动提交

+   Session.autocommit 参数/模式

+   select()的列表/关键字参数

+   Python 2 支持

上述要点涉及 2.0 版本中最显著的完全不兼容的更改。应用程序适应这些更改以及其他更改的迁移路径首先被构建为转换到 SQLAlchemy 1.4 系列，其中“未来”API 可用以提供“2.0”工作方式，然后转换到 2.0 系列，其中上述不再使用的 API 以及其他 API 已被移除。

此迁移路径的完整步骤稍后在本文档的 1.x -> 2.x 迁移概述处介绍。

## 1.x -> 2.x 迁移概述

SQLAlchemy 2.0 过渡在 SQLAlchemy 1.4 发布中呈现为一系列步骤，允许任何规模或复杂度的应用程序使用渐进式、迭代式的过程迁移到 SQLAlchemy 2.0。从 Python 2 到 Python 3 的转换中吸取的教训启发了一个系统，尽可能地不需要任何“破坏性”更改，或者不需要普遍进行或根本不进行任何更改。

作为证明 2.0 架构的手段，同时也为全面迭代的过渡环境提供支持，2.0 全新 API 和功能的整体范围均包含在 1.4 系列中；其中包括主要的新功能领域，如 SQL 缓存系统、新的 ORM 语句执行模型、ORM 和 Core 的新事务范式、统一经典和声明性映射的新 ORM 声明性系统、对 Python 数据类的支持，以及 Core 和 ORM 的 asyncio 支持。

实现 2.0 迁移的步骤在以下子章节中；总体而言，一般策略是，一旦一个应用程序在 1.4 上运行，并且没有发出任何 2.0 弃用警告，它现在**基本上**与 SQLAlchemy 2.0 兼容。**请注意，当运行针对 SQLAlchemy 2.0 时，可能会有额外的 API 和行为变化，这些变化可能在迁移时表现不同；始终在实际 SQLAlchemy 2.0 版本上测试代码作为迁移的最后一步**。

### 第一个先决条件，第一步 - 一个可运行的 1.3 应用程序

第一步是将现有应用程序迁移到 1.4，在典型的非平凡应用程序的情况下，确保它在 SQLAlchemy 1.3 上运行且没有弃用警告。1.4 版本确实有一些与在先前版本中发出警告的条件相关的变化，包括一些在 1.3 中引入的警告，特别是对`relationship.viewonly`和`relationship.sync_backref`标志行为的一些变化。

为了获得最佳结果，应用程序应该能够在最新的 SQLAlchemy 1.3 版本上运行，或通过所有测试，而不会出现任何 SQLAlchemy 弃用警告；这些警告是针对`SADeprecationWarning`类发出的。

### 第一个先决条件，第二步 - 一个可运行的 1.4 应用程序

一旦应用程序在 SQLAlchemy 1.3 上运行良好，下一步是让它在 SQLAlchemy 1.4 上运行。在绝大多数情况下，应用程序应该可以从 SQLAlchemy 1.3 顺利过渡到 1.4。然而，在任何 1.x 和 1.y 版本之间，API 和行为都可能发生了微妙的变化，或者在某些情况下变化更加明显，SQLAlchemy 项目总是在最初几个月收到大量的回归报告。

1.x->1.y 版本的发布过程通常在边缘上有一些比较显著的变化，这些变化基于预期很少或根本不会使用的用例。对于 1.4 版本，被确定为属于这个领域的变化如下：

+   URL 对象现在是不可变的 - 这会影响那些会操作 `URL` 对象的代码，并可能影响使用 `CreateEnginePlugin` 扩展点的代码。这是一个不常见的情况，但可能会特别影响一些使用特殊数据库提供逻辑的测试套件。通过搜索使用相对较新且鲜为人知的 `CreateEnginePlugin` 类的代码，发现两个项目不受此更改影响。

+   SELECT 语句不再隐式视为 FROM 子句 - 这一变化可能会影响某些依赖于 `Select` 构造的行为的代码，其中它会创建通常令人困惑且无效的无名子查询。这些子查询在大多数数据库中都会被拒绝，因为通常需要一个名称，除了 SQLite 外。然而，一些应用程序可能需要调整一些意外依赖于此的查询。

+   select().join() 和 outerjoin() 现在向当前查询添加 JOIN 条件，而不是创建子查询 - 有些相关的，`Select` 类具有 `.join()` 和 `.outerjoin()` 方法，它们隐式创建一个子查询，然后返回一个 `Join` 构造，这再次几乎没有用处且会产生很多混淆。决定采用更有用的 2.0 风格的连接构建方法，这些方法现在与 ORM `Query.join()` 方法的工作方式相同。

+   许多 Core 和 ORM 语句对象现在在编译阶段执行大部分构造和验证工作 - 一些与构造 `Query` 或 `Select` 相关的错误消息可能直到编译/执行时才会被发出，而不是在构造时。这可能会影响一些测试套件，这些测试套件正在针对失败模式进行测试。

要查看 SQLAlchemy 1.4 变化的完整概述，请参阅 SQLAlchemy 1.4 有什么新特性？ 文档。

### 迁移到 2.0 第一步 - 仅支持 Python 3（Python 3.7 最低版本兼容 2.0）

SQLAlchemy 2.0 最初受到 Python 2 的 EOL 是在 2020 年的事实的启发。SQLAlchemy 比其他主要项目花费更长的时间来放弃对 Python 2.7 的支持。然而，为了使用 SQLAlchemy 2.0，应用程序需要至少在**Python 3.7**上运行。SQLAlchemy 1.4 支持 Python 3 系列中的 Python 3.6 或更新版本；在 1.4 系列中，应用程序可以继续在 Python 2.7 上运行，或者至少在 Python 3.6 上运行。但是，版本 2.0 从 Python 3.7 开始。

### 迁移到 2.0 步骤二 - 打开 RemovedIn20Warnings

SQLAlchemy 1.4 特性一个有条件的弃用警告系统，灵感来自 Python 中指示运行应用程序中的遗留模式的“-3”标志。对于 SQLAlchemy 1.4，只有当环境变量 `SQLALCHEMY_WARN_20` 被设置为 `true` 或 `1` 时，才会发出 `RemovedIn20Warning` 弃用类。

鉴于下面的示例程序：

```py
from sqlalchemy import column
from sqlalchemy import create_engine
from sqlalchemy import select
from sqlalchemy import table

engine = create_engine("sqlite://")

engine.execute("CREATE TABLE foo (id integer)")
engine.execute("INSERT INTO foo (id) VALUES (1)")

foo = table("foo", column("id"))
result = engine.execute(select([foo.c.id]))

print(result.fetchall())
```

上述程序使用了许多用户可能已经将其识别为“遗留”的模式，即使用 `Engine.execute()` 方法的“无连接执行”API 的使用。当我们针对 1.4 运行上述程序时，它返回一行：

```py
$ python test3.py
[(1,)]
```

要启用“2.0 弃用模式”，我们启用 `SQLALCHEMY_WARN_20=1` 变量，并确保选择了一个[警告过滤器](https://docs.python.org/3/library/warnings.html#the-warnings-filter)，该过滤器不会抑制任何警告：

```py
SQLALCHEMY_WARN_20=1 python -W always::DeprecationWarning test3.py
```

由于报告的警告位置不总是在正确的位置，没有完整的堆栈跟踪可能会很难找到有问题的代码。这可以通过将警告转换为异常来实现，通过指定 `error` 警告过滤器，使用 Python 选项 `-W error::DeprecationWarning`。 

打开警告后，我们的程序现在有很多话要说：

```py
$ SQLALCHEMY_WARN_20=1 python2 -W always::DeprecationWarning test3.py
test3.py:9: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  engine.execute("CREATE TABLE foo (id integer)")
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0\.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return connection.execute(statement, *multiparams, **params)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0\.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  self._commit_impl(autocommit=True)
test3.py:10: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  engine.execute("INSERT INTO foo (id) VALUES (1)")
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0\.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return connection.execute(statement, *multiparams, **params)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0\.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  self._commit_impl(autocommit=True)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/sql/selectable.py:4271: RemovedIn20Warning: The legacy calling style of select() is deprecated and will be removed in SQLAlchemy 2.0\.  Please use the new calling style described at select(). (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return cls.create_legacy_select(*args, **kw)
test3.py:14: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  result = engine.execute(select([foo.c.id]))
[(1,)]
```

通过上述指导，我们可以将我们的程序迁移到使用 2.0 样式，并且作为奖励，我们的程序变得更加清晰：

```py
from sqlalchemy import column
from sqlalchemy import create_engine
from sqlalchemy import select
from sqlalchemy import table
from sqlalchemy import text

engine = create_engine("sqlite://")

# don't rely on autocommit for DML and DDL
with engine.begin() as connection:
    # use connection.execute(), not engine.execute()
    # use the text() construct to execute textual SQL
    connection.execute(text("CREATE TABLE foo (id integer)"))
    connection.execute(text("INSERT INTO foo (id) VALUES (1)"))

foo = table("foo", column("id"))

with engine.connect() as connection:
    # use connection.execute(), not engine.execute()
    # select() now accepts column / table expressions positionally
    result = connection.execute(select(foo.c.id))

    print(result.fetchall())
```

“2.0 弃用模式”的目标是，一个在“2.0 弃用模式”下没有 `RemovedIn20Warning` 警告的程序，然后准备运行在 SQLAlchemy 2.0 中。

### 迁移到 2.0 步骤三 - 解决所有 RemovedIn20Warnings

代码可以迭代开发以解决这些警告。在 SQLAlchemy 项目本身中，采取的方法如下：

1.  在测试套件中启用 `SQLALCHEMY_WARN_20=1` 环境变量，对于 SQLAlchemy，这在 tox.ini 文件中

1.  在测试套件的设置中，设置一系列警告过滤器，以选择特定的警告子集来引发异常，或者被忽略（或记录）。逐个子组警告地进行工作。下面，为一个应用程序配置了一个警告过滤器，其中需要对核心级别的 `.execute()` 调用进行更改，以便所有测试都能通过，但是将所有其他 2.0 样式的警告都抑制：

    ```py
    import warnings
    from sqlalchemy import exc

    # for warnings not included in regex-based filter below, just log
    warnings.filterwarnings("always", category=exc.RemovedIn20Warning)

    # for warnings related to execute() / scalar(), raise
    for msg in [
        r"The (?:Executable|Engine)\.(?:execute|scalar)\(\) function",
        r"The current statement is being autocommitted using implicit autocommit,",
        r"The connection.execute\(\) method in SQLAlchemy 2.0 will accept "
        "parameters as a single dictionary or a single sequence of "
        "dictionaries only.",
        r"The Connection.connect\(\) function/method is considered legacy",
        r".*DefaultGenerator.execute\(\)",
    ]:
        warnings.filterwarnings(
            "error",
            message=msg,
            category=exc.RemovedIn20Warning,
        )
    ```

1.  当应用程序中解决了警告的每个子类别时，被“always”过滤器捕获的新警告可以添加到“错误”列表中以解决。

1.  一旦不再发出警告，过滤器可以被移除。

### 迁移到 2.0 步骤四 - 在引擎上使用`future`标志

`Engine`对象在 2.0 版本中具有更新的事务级 API。在 1.4 中，通过将标志`future=True`传递给`create_engine()`函数，可以使用此新 API。

当使用`create_engine.future`标志时，`Engine`和`Connection`对象完全支持 2.0 API，不再支持任何旧特性，包括`Connection.execute()`的新参数格式，删除了“隐式自动提交”，字符串语句需要使用`text()`构造，除非使用`Connection.exec_driver_sql()`方法，以及从`Engine`���行无连接执行。

如果关于`Engine`和`Connection`的所有`RemovedIn20Warning`警告都已解决，则可以启用`create_engine.future`标志，并且不应该引发任何错误。

新引擎在`Engine`中描述，它提供了一个新的`Connection`对象。除了上述更改外，`Connection`对象具有`Connection.commit()`和`Connection.rollback()`方法，以支持新的“随时提交”操作模式：

```py
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2:///")

with engine.connect() as conn:
    conn.execute(text("insert into table (x) values (:some_x)"), {"some_x": 10})

    conn.commit()  # commit as you go
```

### 迁移到 2.0 步骤五 - 在会话上使用`future`标志

在 2.0 版本中，`Session` 对象还具有更新的事务/连接级 API。在 1.4 中可通过在 `Session` 或 `sessionmaker` 上使用 `Session.future` 标志来使用此 API。

`Session` 对象支持“future”模式，并涉及以下更改：

1.  当解析用于连接的引擎时，`Session` 不再支持“绑定的元数据”。这意味着必须将一个 `Engine` 对象传递给构造函数（这可以是传统或未来风格的对象）。

1.  `Session.begin.subtransactions` 标志不再受支持。

1.  `Session.commit()` 方法始终向数据库发出 COMMIT，而不是尝试调和“子事务”。

1.  `Session.rollback()` 方法总是一次性回滚整个事务堆栈，而不是尝试保留“子事务”。

在 1.4 中，`Session` 还支持更灵活的创建模式，现在与 `Connection` 对象使用的模式紧密匹配。重点包括 `Session` 可以作为上下文管理器使用：

```py
from sqlalchemy.orm import Session

with Session(engine) as session:
    session.add(MyObject())
    session.commit()
```

此外，`sessionmaker` 对象支持一个 `sessionmaker.begin()` 上下文管理器，将在一个块中创建一个 `Session` 并开始/提交一个事务：

```py
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(engine)

with Session.begin() as session:
    session.add(MyObject())
```

参见 Session 级 vs. Engine 级事务控制 部分，比较 `Session` 创建模式与 `Connection` 创建模式的对比。

一旦应用程序通过了所有测试/使用 `SQLALCHEMY_WARN_20=1` 运行，并且所有 `exc.RemovedIn20Warning` 的出现都设置为引发错误，**应用程序就准备好了！**。

接下来的章节将详细介绍所有主要 API 修改的具体更改。

### 迁移到 2.0 第六步 - 在显式类型的 ORM 模型中添加 `__allow_unmapped__`

SQLAlchemy 2.0 新增了对 ORM 模型上 [**PEP 484**](https://peps.python.org/pep-0484/) 类型标注的运行时解释支持。这些注解的要求是它们必须使用 `Mapped` 泛型容器。那些不使用 `Mapped` 的注解，比如与 `relationship()` 等构造关联的注解，在 Python 中会引发错误，因为它们暗示了配置错误。

使用 Mypy 插件 的 SQLAlchemy 应用程序中，如果显式注解不使用 `Mapped`，则会出现这些错误，如下例所示：

```py
Base = declarative_base()

class Foo(Base):
    __tablename__ = "foo"

    id: int = Column(Integer, primary_key=True)

    # will raise
    bars: List["Bar"] = relationship("Bar", back_populates="foo")

class Bar(Base):
    __tablename__ = "bar"

    id: int = Column(Integer, primary_key=True)
    foo_id = Column(ForeignKey("foo.id"))

    # will raise
    foo: Foo = relationship(Foo, back_populates="bars", cascade="all")
```

在上述代码中，`Foo.bars` 和 `Bar.foo` 的 `relationship()` 声明会在类构造时引发错误，因为它们没有使用 `Mapped`（相比之下，使用 `Column` 的注解在 2.0 版本中会被忽略，因为这些注解能够被识别为传统的配置风格）。为了允许所有不使用 `Mapped` 的注解通过而不报错，可以在类或任何子类上使用 `__allow_unmapped__` 属性，这将导致在这些情况下完全忽略新 Declarative 系统中的注解。

注：

`__allow_unmapped__` 指令**仅**适用于 ORM 的*运行时*行为。它不会影响 Mypy 的行为，上述映射仍然需要安装 Mypy 插件。对于完全符合 2.0 样式的 ORM 模型，在不需要插件的情况下可以正确进行类型标注，请遵循 迁移现有映射 中的迁移步骤。

下面的示例说明了将 `__allow_unmapped__` 应用于 Declarative `Base` 类，在那里它将对所有从 `Base` 继承的类生效：

```py
# qualify the base with __allow_unmapped__.  Can also be
# applied to classes directly if preferred
class Base:
    __allow_unmapped__ = True

Base = declarative_base(cls=Base)

# existing mapping proceeds, Declarative will ignore any annotations
# which don't include ``Mapped[]``
class Foo(Base):
    __tablename__ = "foo"

    id: int = Column(Integer, primary_key=True)

    bars: List["Bar"] = relationship("Bar", back_populates="foo")

class Bar(Base):
    __tablename__ = "bar"

    id: int = Column(Integer, primary_key=True)
    foo_id = Column(ForeignKey("foo.id"))

    foo: Foo = relationship(Foo, back_populates="bars", cascade="all")
```

自 2.0.0beta3 版本起发生了变化：- 改进了`__allow_unmapped__`属性支持，使得能够保持对 1.4 风格的显式注释关系的支持，而不使用`Mapped` 也能够保持可用性。### 迁移到 2.0 第七步 - 对 SQLAlchemy 2.0 发行版进行测试

如前所述，SQLAlchemy 2.0 具有额外的 API 和行为更改，旨在向后兼容，但仍然可能引入一些不兼容性。 因此，在整个迁移过程完成后，最后一步是针对最新版本的 SQLAlchemy 2.0 进行测试，以纠正可能存在的任何剩余问题。

在 SQLAlchemy 2.0 的新特性是什么？ 指南中提供了对超出基本 1.4->2.0 API 更改范围的 SQLAlchemy 2.0 的新特性和行为的概述。

### 第一个先决条件，第一步 - 一个工作中的 1.3 应用程序

第一步是将现有的应用程序升级到 1.4，在典型的非平凡应用程序的情况下，确保它在 SQLAlchemy 1.3 上运行时没有弃用警告。 发布 1.4 确实有一些与在之前版本中发出警告的条件相关的更改，包括一些在 1.3 中引入的警告，特别是一些关于`relationship.viewonly`和`relationship.sync_backref` 标志行为的更改。

为了达到最佳效果，应用程序应该能够在最新的 SQLAlchemy 1.3 版本上运行，或通过所有测试，并且没有 SQLAlchemy 弃用警告； 这些是针对`SADeprecationWarning` 类发出的警告。

### 第一个先决条件，第二步 - 一个工作中的 1.4 应用程序

一旦应用程序在 SQLAlchemy 1.3 上运行良好，下一步是将其运行在 SQLAlchemy 1.4 上。 在绝大多数情况下，应用程序应该可以在从 SQLAlchemy 1.3 到 1.4 的过程中无问题地运行。 然而，不管是在任何 1.x 和 1.y 发行版之间，API 和行为都可能发生了微妙的或在某些情况下稍微不那么微妙的变化，而 SQLAlchemy 项目总是在前几个月收到大量的回归报告。

1.x->1.y 发行过程通常会有一些在边缘方面略微戏剧性的更改，这些更改是基于预期几乎不会或根本不会使用的用例。 对于 1.4，被确定为属于此领域的更改如下：

+   URL 对象现在是不可变的 - 这会影响那些会操作`URL`对象的代码，并可能影响使用`CreateEnginePlugin`扩展点的代码。这是一个不常见的情况，但可能会特别影响一些使用特殊数据库提供逻辑的测试套件。在 github 上搜索使用相对较新且鲜为人知的`CreateEnginePlugin`类的代码时，发现两个项目不受此更改影响。

+   SELECT 语句不再被隐式视为 FROM 子句 - 这个变化可能会影响一些某种程度上依赖于在`Select`构造中通常无法使用的行为的代码，其中它会创建通常令人困惑且无法工作的未命名子查询。这些子查询在大多数情况下会被大多数数据库拒绝，因为通常需要一个名称，除了 SQLite 之外，然而一些应用程序可能需要调整一些意外依赖于此的查询。

+   select().join()和 outerjoin()现在向当前查询添加 JOIN 条件，而不是创建子查询 - 有些相关的是，`Select`类具有`.join()`和`.outerjoin()`方法，这些方法隐式地创建了一个子查询，然后返回一个`Join`构造，这在大多数情况下是无用的并且会产生很多混乱。决定采用更加有用的 2.0 风格的连接构建方法，这些方法现在与 ORM `Query.join()`方法的工作方式相同。

+   许多核心和 ORM 语句对象现在在编译阶段执行大部分构建和验证工作 - 一些与`Query`或`Select`构建相关的错误消息可能要等到编译/执行阶段才会被发出，而不是在构建时。这可能会影响一些针对失败模式进行测试的测试套件。

要查看 SQLAlchemy 1.4 变更的完整概述，请参阅 SQLAlchemy 1.4 有什么新特性？文档。

### 迁移到 2.0 第一步 - 仅支持 Python 3（Python 3.7 最低版本以实现 2.0 兼容性）

SQLAlchemy 2.0 最初的灵感来自于 Python 2 的 EOL 是在 2020 年。SQLAlchemy 花费的时间比其他主要项目更长来放弃 Python 2.7 支持。但是，为了使用 SQLAlchemy 2.0，应用程序将需要至少运行在**Python 3.7**上。SQLAlchemy 1.4 在 Python 3 系列中支持 Python 3.6 或更新版本；在 1.4 系列中，应用程序可以继续在 Python 2.7 上运行或至少在 Python 3.6 上运行。然而，版本 2.0 从 Python 3.7 开始。

### 迁移到 2.0 第二步 - 打开 RemovedIn20Warnings

SQLAlchemy 1.4 具有受 Python“-3”标志启发的条件弃用警告系统，该标志将在运行中的应用程序中指示遗留模式。对于 SQLAlchemy 1.4，仅当环境变量`SQLALCHEMY_WARN_20`设置为`true`或`1`时，才会发出`RemovedIn20Warning`弃用类。

给定以下示例程序：

```py
from sqlalchemy import column
from sqlalchemy import create_engine
from sqlalchemy import select
from sqlalchemy import table

engine = create_engine("sqlite://")

engine.execute("CREATE TABLE foo (id integer)")
engine.execute("INSERT INTO foo (id) VALUES (1)")

foo = table("foo", column("id"))
result = engine.execute(select([foo.c.id]))

print(result.fetchall())
```

上述程序使用了许多用户已经将其识别为“遗留”的模式，即使用`Engine.execute()`方法的“无连接执行”API。当我们对上述程序运行 1.4 版本时，它返回一行：

```py
$ python test3.py
[(1,)]
```

要启用“2.0 弃用模式”，我们启用`SQLALCHEMY_WARN_20=1`变量，并确保选择了一个[警告过滤器](https://docs.python.org/3/library/warnings.html#the-warnings-filter)，不会抑制任何警告：

```py
SQLALCHEMY_WARN_20=1 python -W always::DeprecationWarning test3.py
```

由于报告的警告位置并不总是在正确的位置，找到有问题的代码可能会很困难，没有完整的堆栈跟踪。这可以通过将警告转换为异常来实现，方法是指定`error`警告过滤器，使用 Python 选项`-W error::DeprecationWarning`。

打开警告后，我们的程序现在有很多话要说：

```py
$ SQLALCHEMY_WARN_20=1 python2 -W always::DeprecationWarning test3.py
test3.py:9: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  engine.execute("CREATE TABLE foo (id integer)")
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0\.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return connection.execute(statement, *multiparams, **params)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0\.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  self._commit_impl(autocommit=True)
test3.py:10: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  engine.execute("INSERT INTO foo (id) VALUES (1)")
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0\.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return connection.execute(statement, *multiparams, **params)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0\.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  self._commit_impl(autocommit=True)
/home/classic/dev/sqlalchemy/lib/sqlalchemy/sql/selectable.py:4271: RemovedIn20Warning: The legacy calling style of select() is deprecated and will be removed in SQLAlchemy 2.0\.  Please use the new calling style described at select(). (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  return cls.create_legacy_select(*args, **kw)
test3.py:14: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0\. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
  result = engine.execute(select([foo.c.id]))
[(1,)]
```

根据上述指导，我们可以将我们的程序迁移到使用 2.0 风格，作为额外的奖励，我们的程序更加清晰：

```py
from sqlalchemy import column
from sqlalchemy import create_engine
from sqlalchemy import select
from sqlalchemy import table
from sqlalchemy import text

engine = create_engine("sqlite://")

# don't rely on autocommit for DML and DDL
with engine.begin() as connection:
    # use connection.execute(), not engine.execute()
    # use the text() construct to execute textual SQL
    connection.execute(text("CREATE TABLE foo (id integer)"))
    connection.execute(text("INSERT INTO foo (id) VALUES (1)"))

foo = table("foo", column("id"))

with engine.connect() as connection:
    # use connection.execute(), not engine.execute()
    # select() now accepts column / table expressions positionally
    result = connection.execute(select(foo.c.id))

    print(result.fetchall())
```

“2.0 弃用模式”的目标是，在打开“2.0 弃用模式”时没有`RemovedIn20Warning`警告的程序，然后准备在 SQLAlchemy 2.0 中运行。

### 迁移到 2.0 第三步 - 解决所有 RemovedIn20Warnings

代码可以迭代开发以解决这些警告。在 SQLAlchemy 项目本身中，采取的方法如下：

1.  在测试套件中启用`SQLALCHEMY_WARN_20=1`环境变量，对于 SQLAlchemy，这是在 tox.ini 文件中

1.  在测试套件的设置中，设置一系列警告过滤器，将选择特定的警告子集以引发异常，或者忽略（或记录）警告。一次只处理一个警告子组。下面，为应用程序配置了一个警告过滤器，其中需要对 Core 级别的`.execute()`调用进行更改，以便所有测试都通过，但是所有其他 2.0 风格的警告都将被抑制：

    ```py
    import warnings
    from sqlalchemy import exc

    # for warnings not included in regex-based filter below, just log
    warnings.filterwarnings("always", category=exc.RemovedIn20Warning)

    # for warnings related to execute() / scalar(), raise
    for msg in [
        r"The (?:Executable|Engine)\.(?:execute|scalar)\(\) function",
        r"The current statement is being autocommitted using implicit autocommit,",
        r"The connection.execute\(\) method in SQLAlchemy 2.0 will accept "
        "parameters as a single dictionary or a single sequence of "
        "dictionaries only.",
        r"The Connection.connect\(\) function/method is considered legacy",
        r".*DefaultGenerator.execute\(\)",
    ]:
        warnings.filterwarnings(
            "error",
            message=msg,
            category=exc.RemovedIn20Warning,
        )
    ```

1.  当应用程序中解决了每个警告的子类别时，被“always”过滤器捕获的新警告可以添加到“errors”列表中以解决。

1.  一旦不再发出任何警告，过滤器就可以被移除。

### 迁移至 2.0 第四步 - 在 Engine 上使用 `future` 标志

`Engine` 对象在 2.0 版本中具有更新的事务级 API。在 1.4 版本中，通过向 `create_engine()` 函数传递 `future=True` 标志即可使用这个新 API。

当使用 `create_engine.future` 标志时，`Engine` 和 `Connection` 对象完全支持 2.0 API，不再支持任何旧特性，包括 `Connection.execute()` 的新参数格式，移除了“隐式自动提交”，字符串语句需要使用 `text()` 构造，除非使用 `Connection.exec_driver_sql()` 方法，以及从 `Engine` 进行无连接执行的功能被移除。

如果关于使用 `Engine` 和 `Connection` 的所有 `RemovedIn20Warning` 警告都已解决，则可以启用 `create_engine.future` 标志，并且不应该引发任何错误。

新引擎被描述为 `Engine`，它提供了一个新的 `Connection` 对象。除了上述更改外，`Connection` 对象还具有 `Connection.commit()` 和 `Connection.rollback()` 方法，以支持新的“随时提交”操作模式：

```py
from sqlalchemy import create_engine

engine = create_engine("postgresql+psycopg2:///")

with engine.connect() as conn:
    conn.execute(text("insert into table (x) values (:some_x)"), {"some_x": 10})

    conn.commit()  # commit as you go
```

### 迁移至 2.0 第五步 - 在 Session 上使用 `future` 标志

`Session` 对象在 2.0 版本中还具有更新的事务/连接级 API。在 1.4 版本中，可以在 `Session` 或 `sessionmaker` 上使用 `Session.future` 标志来使用此 API。

`Session` 对象支持“future”模式，并涉及以下更改：

1.  当 `Session` 解析用于连接的引擎时，不再支持“bound metadata”。这意味着必须将一个 `Engine` 对象传递给构造函数（这可以是传统或未来风格对象）。

1.  不再支持 `Session.begin.subtransactions` 标志。

1.  `Session.commit()` 方法总是向数据库发出 COMMIT，而不��尝试协调“子事务”。

1.  `Session.rollback()` 方法总是一次性回滚整个事务堆栈，而不是尝试保持“子事务”。

在 1.4 版本中，`Session` 还支持更灵活的创建模式，这些模式现在与 `Connection` 对象使用的模式紧密匹配。亮点包括 `Session` 可以用作上下文管理器：

```py
from sqlalchemy.orm import Session

with Session(engine) as session:
    session.add(MyObject())
    session.commit()
```

此外，`sessionmaker` 对象支持 `sessionmaker.begin()` 上下文管理器，将在一个块中创建一个 `Session` 并开始/提交一个事务：

```py
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(engine)

with Session.begin() as session:
    session.add(MyObject())
```

参见 Session-level vs. Engine level transaction control 部分，比较了 `Session` 的创建模式与 `Connection` 的模式。

一旦应用程序通过所有测试/运行，并且所有 `SQLALCHEMY_WARN_20=1` 和所有 `exc.RemovedIn20Warning` 出现的地方都设置为引发错误，**应用程序就准备好了！**。

接下来的部分将详细说明所有主要 API 修改的具体更改。

### 迁移至 2.0 第六步 - 为显式类型化的 ORM 模型添加 `__allow_unmapped__`

SQLAlchemy 2.0 新增了对 ORM 模型上 [**PEP 484**](https://peps.python.org/pep-0484/) 类型注解的运行时解释支持。这些注解的要求是它们必须使用 `Mapped` 泛型容器。不使用 `Mapped` 的注解，比如链接到 `relationship()` 等构造的注解将在 Python 中引发错误，因为它们暗示了错误的配置。

使用 Mypy 插件 的 SQLAlchemy 应用程序，如果在注解中使用了不使用 `Mapped` 的显式注释，则会出现这些错误，就像下面的示例中所发生的一样：

```py
Base = declarative_base()

class Foo(Base):
    __tablename__ = "foo"

    id: int = Column(Integer, primary_key=True)

    # will raise
    bars: List["Bar"] = relationship("Bar", back_populates="foo")

class Bar(Base):
    __tablename__ = "bar"

    id: int = Column(Integer, primary_key=True)
    foo_id = Column(ForeignKey("foo.id"))

    # will raise
    foo: Foo = relationship(Foo, back_populates="bars", cascade="all")
```

上面，`Foo.bars` 和 `Bar.foo` 的 `relationship()` 声明将在类构建时引发错误，因为它们不使用 `Mapped`（相比之下，使用 `Column` 的注解在 2.0 中被忽略，因为这些能够被识别为传统配置样式）。为了让所有不使用 `Mapped` 的注解能够无错误通过，可以在类或任何子类上使用 `__allow_unmapped__` 属性，这将导致在这些情况下这些注解完全被新的声明式系统忽略。

注意

`__allow_unmapped__` 指令**仅**适用于 ORM 的*运行时*行为。它不会影响 Mypy 的行为，上述映射仍然要求安装 Mypy 插件。对于完全符合 2.0 样式的 ORM 模型，可以在不使用插件的情况下正确进行类型化，遵循 迁移现有映射 中的迁移步骤。

下面的示例说明了将 `__allow_unmapped__` 应用于声明式 `Base` 类的情况，它将对所有从 `Base` 继承的类生效：

```py
# qualify the base with __allow_unmapped__.  Can also be
# applied to classes directly if preferred
class Base:
    __allow_unmapped__ = True

Base = declarative_base(cls=Base)

# existing mapping proceeds, Declarative will ignore any annotations
# which don't include ``Mapped[]``
class Foo(Base):
    __tablename__ = "foo"

    id: int = Column(Integer, primary_key=True)

    bars: List["Bar"] = relationship("Bar", back_populates="foo")

class Bar(Base):
    __tablename__ = "bar"

    id: int = Column(Integer, primary_key=True)
    foo_id = Column(ForeignKey("foo.id"))

    foo: Foo = relationship(Foo, back_populates="bars", cascade="all")
```

2.0.0beta3 版本中的变更：- 改进了 `__allow_unmapped__` 属性支持，允许保持可用的 1.4 样式显式注释关系，而不使用 `Mapped`。

### 迁移到 2.0 第七步 - 测试针对 SQLAlchemy 2.0 版本

如前所述，SQLAlchemy 2.0 具有额外的 API 和行为变化，旨在向后兼容，但仍可能引入一些不兼容性。因此，在整体移植过程完成后，最后一步是针对最新版本的 SQLAlchemy 2.0 进行测试，以纠正可能存在的任何剩余问题。

在 SQLAlchemy 2.0 有什么新特性？ 的指南中提供了一个关于 SQLAlchemy 2.0 的新功能和行为的概述，这些新功能和行为超出了 1.4->2.0 API 变化的基本集。

## 2.0 迁移 - Core 连接 / 事务

### 库级别（但不是驱动程序级别）的“自动提交”从 Core 和 ORM 中移除

**概要**

在 SQLAlchemy 1.x 中，以下语句将自动提交底层的 DBAPI 事务，但在 SQLAlchemy 2.0 中不会发生：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(some_table.insert().values(foo="bar"))
```

这个自动提交也不会发生：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(text("INSERT INTO table (foo) VALUES ('bar')"))
```

自定义 DML 的常见解决方法是需要提交的“自动提交”执行选项，将被移除：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(text("EXEC my_procedural_thing()").execution_options(autocommit=True))
```

**迁移到 2.0**

与 1.x 风格 和 2.0 风格 执行兼容的方法是使用 `Connection.begin()` 方法，或者 `Engine.begin()` 上下文管理器：

```py
with engine.begin() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.execute(some_other_table.insert().values(bat="hoho"))

with engine.connect() as conn:
    with conn.begin():
        conn.execute(some_table.insert().values(foo="bar"))
        conn.execute(some_other_table.insert().values(bat="hoho"))

with engine.begin() as conn:
    conn.execute(text("EXEC my_procedural_thing()"))
```

当使用 2.0 风格 与 `create_engine.future` 标志时，也可以使用“边提交边进行”风格，因为 `Connection` 具有 **autobegin** 行为，当在没有显式调用 `Connection.begin()` 的情况下首次调用语句时会发生：

```py
with engine.connect() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.execute(some_other_table.insert().values(bat="hoho"))

    conn.commit()
```

当启用 2.0 弃用模式 时，当弃用的“自动提交”功能发生时，将发出警告，指示应注意显式事务的地方。

**讨论**

SQLAlchemy 的最初版本与 Python DBAPI（[**PEP 249**](https://peps.python.org/pep-0249/)）的精神相悖，因为它试图隐藏 [**PEP 249**](https://peps.python.org/pep-0249/) 强调的事务的“隐式开始”和“显式提交”。十五年后，我们现在认识到这基本上是一个错误，因为 SQLAlchemy 的许多试图“隐藏”事务存在的模式导致了更复杂的 API，其工作不一致，并且对于那些对关系数据库和 ACID 事务一般不熟悉的用户来说，特别是新用户，这种模式极其令人困惑。SQLAlchemy 2.0 将取消所有隐式提交事务的尝试，并且使用模式将始终要求用户以某种方式标示事务的“开始”和“结束”，就像在 Python 中读取或写入文件一样有“开始”和“结束”一样。

在纯文本语句的自动提交情况下，实际上有一个正则表达式来解析每个语句以检测自动提交！毫不奇怪，这个正则表达式在持续失败以适应各种隐含对数据库进行“写入”的语句和存储过程，导致持续混淆，因为有些语句在数据库中产生结果，而其他语句则没有。通过防止用户意识到事务概念，我们因为用户不理解数据库是否总是使用事务而产生了很多错误报告，无论某些层是否自动提交了它。

SQLAlchemy 2.0 将要求在每个级别的数据库操作中都明确指定事务的使用方式。对于绝大多数 Core 使用案例，已经推荐了以下模式：

```py
with engine.begin() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
```

对于“边做边提交，或者回滚”的用法，这与如今通常如何使用 `Session` 很相似，使用 `create_engine.future` 标志创建的 `Engine` 返回的“future”版本 `Connection` 中包含了新的 `Connection.commit()` 和 `Connection.rollback()` 方法，这些方法在首次调用语句时自动开始事务：

```py
# 1.4 / 2.0 code

from sqlalchemy import create_engine

engine = create_engine(..., future=True)

with engine.connect() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.commit()

    conn.execute(text("some other SQL"))
    conn.rollback()
```

上面的 `engine.connect()` 方法将返回一个 `Connection` 对象，其具有**自动开始**特性，意味着当首次使用 `execute` 方法时会触发 `begin()` 事件（注意 Python DBAPI 中实际上没有“BEGIN”）。“自动开始”是 SQLAlchemy 1.4 中的一个新模式，由 `Connection` 和 ORM `Session` 对象都支持；自动开始允许在首次获取对象时显式调用 `Connection.begin()` 方法，用于希望在事务开始时划定范围的方案，但如果未调用该方法，则在首次对对象执行操作时隐式发生事务开始。

删除“自动提交”与删除讨论过的 “隐式”和“无连接”执行，“绑定元数据”被移除密切相关。所有这些遗留模式都建立在 Python 在 SQLAlchemy 首次创建时没有上下文管理器或装饰器的事实上，因此没有方便的成语模式来标记资源的使用。

#### 驱动程序级别的自动提交仍然可用

现在，大多数 DBAPI 实现都广泛支持真正的“自动提交”行为，并且通过 `Connection.execution_options.isolation_level` 参数来支持 SQLAlchemy，如 设置事务隔离级别，包括 DBAPI 自动提交 中所讨论的。真正的自动提交被视为“隔离级别”，因此当使用自动提交时，应用程序代码的结构不会发生变化；`Connection.begin()` 上下文管理器以及像 `Connection.commit()` 这样的方法仍然可以使用，它们在数据库驱动程序级别时简单地成为无操作。 ### “隐式”和“无连接”执行，移除“绑定元数据”

**概要**

将 `Engine` 与 `MetaData` 对象关联的能力被移除，这样一来就无法使用一系列所谓的“无连接”执行模式：

```py
from sqlalchemy import MetaData

metadata_obj = MetaData(bind=engine)  # no longer supported

metadata_obj.create_all()  # requires Engine or Connection

metadata_obj.reflect()  # requires Engine or Connection

t = Table("t", metadata_obj, autoload=True)  # use autoload_with=engine

result = engine.execute(t.select())  # no longer supported

result = t.select().execute()  # no longer supported
```

**迁移到 2.0 版本**

对于模式级别的模式，需要显式使用`Engine`或`Connection`。`Engine`仍然可以直接用作`MetaData.create_all()`操作或自动加载操作的连接源。对于执行语句，只有`Connection`对象有一个`Connection.execute()`方法（除了 ORM 级别的`Session.execute()`方法）：

```py
from sqlalchemy import MetaData

metadata_obj = MetaData()

# engine level:

# create tables
metadata_obj.create_all(engine)

# reflect all tables
metadata_obj.reflect(engine)

# reflect individual table
t = Table("t", metadata_obj, autoload_with=engine)

# connection level:

with engine.connect() as connection:
    # create tables, requires explicit begin and/or commit:
    with connection.begin():
        metadata_obj.create_all(connection)

    # reflect all tables
    metadata_obj.reflect(connection)

    # reflect individual table
    t = Table("t", metadata_obj, autoload_with=connection)

    # execute SQL statements
    result = connection.execute(t.select())
```

**讨论**

核心文档已经对此处的期望模式进行了标准化，因此大多数现代应用程序很可能不需要做太多改变，然而仍然有许多依赖于`engine.execute()`调用的应用程序需要进行调整。

“无连接”执行是指从`Engine`中调用`.execute()`的仍然相当流行的模式：

```py
result = engine.execute(some_statement)
```

上述操作隐式地获取了一个`Connection`对象，并在其上运行了`.execute()`方法。虽然这看起来是一个简单的便利功能，但已经证明引发了几个问题：

+   具有大量`engine.execute()`调用的程序已经普遍存在，过度使用了本来应该很少使用的功能，并导致了效率低下的非事务性应用。新用户对`engine.execute()`和`connection.execute()`之间的区别感到困惑，而这两种方法之间的微妙差别通常不为人所理解。

+   此功能依赖于“应用程序级自动提交”功能才能有意义，而这个功能本身也正在被移除，因为它也是低效且具有误导性的。

+   为了处理结果集，`Engine.execute` 返回一个带有未消耗的游标结果的结果对象。这个游标结果必然仍然链接到保持打开的事务的 DBAPI 连接，所有这些在结果集完全消耗了在游标中等待的行后都会被释放。这意味着当调用完成时，`Engine.execute` 实际上并不关闭它声称正在管理的连接资源。SQLAlchemy 的“自动关闭”行为已经调优得足够好，以至于用户通常不会报告这个系统的任何负面影响，但是它仍然是 SQLAlchemy 最早版本中留下的一个过于隐式和低效的系统。

移除“无连接”执行然后导致移除更加传统的模式，即“隐式，无连接”执行：

```py
result = some_statement.execute()
```

上述模式存在“无连接”执行的所有问题，而且它依赖于“绑定元数据”模式，这是 SQLAlchemy 多年来试图减少强调的模式之一。这在 SQLAlchemy 的第一个广告使用模型中是 SQLAlchemy 的第一个广告使用模型，在版本 0.1 中立即变得过时，当`Connection`对象被引入后，后来的 Python 上下文管理器提供了更好的在固定范围内使用资源的模式。

随着隐式执行的移除，“绑定元数据”本身在该系统中也不再有作用。在现代使用中，“绑定元数据”在`MetaData.create_all()`调用以及`Session`对象中仍然有一定的方便之处，但是让这些函数明确接收`Engine`提供了更清晰的应用设计。

#### 多种选择变为一种选择

总的来说，上述执行模式是在 SQLAlchemy 的第一个 0.1 版本中引入的，甚至在`Connection`对象都不存在的情况下。经过多年的减少这些模式的强调，“隐式，无连接”执行和“绑定元数据”不再被广泛使用，所以在 2.0 版本中我们希望最终减少在核心中执行语句的选择数量：

```py
# many choices

# bound metadata?
metadata_obj = MetaData(engine)

# or not?
metadata_obj = MetaData()

# execute from engine?
result = engine.execute(stmt)

# or execute the statement itself (but only if you did
# "bound metadata" above, which means you can't get rid of "bound" if any
# part of your program uses this form)
result = stmt.execute()

# execute from connection, but it autocommits?
conn = engine.connect()
conn.execute(stmt)

# execute from connection, but autocommit isn't working, so use the special
# option?
conn.execution_options(autocommit=True).execute(stmt)

# or on the statement ?!
conn.execute(stmt.execution_options(autocommit=True))

# or execute from connection, and we use explicit transaction?
with conn.begin():
    conn.execute(stmt)
```

到“一个选择”，这里的“一个选择”指的是“显式连接和显式事务”；根据需要仍然有几种方式来标记事务块。这个“一个选择”是获得一个`Connection`，然后在操作是写操作的情况下明确标记事务：

```py
# one choice - work with explicit connection, explicit transaction
# (there remain a few variants on how to demarcate the transaction)

# "begin once" - one transaction only per checkout
with engine.begin() as conn:
    result = conn.execute(stmt)

# "commit as you go" - zero or more commits per checkout
with engine.connect() as conn:
    result = conn.execute(stmt)
    conn.commit()

# "commit as you go" but with a transaction block instead of autobegin
with engine.connect() as conn:
    with conn.begin():
        result = conn.execute(stmt)
```

### execute() 方法更加严格，执行选项更加突出。

**概要**

SQLAlchemy 2.0 版本中可用于`sqlalchemy.engine.Connection()`执行方法的参数模式大大简化，删除了许多以前可用的参数模式。 1.4 系列中的新 API 在`sqlalchemy.engine.Connection()`中描述。下面的示例说明了需要修改的模式：

```py
connection = engine.connect()

# direct string SQL not supported; use text() or exec_driver_sql() method
result = connection.execute("select * from table")

# positional parameters no longer supported, only named
# unless using exec_driver_sql()
result = connection.execute(table.insert(), ("x", "y", "z"))

# **kwargs no longer accepted, pass a single dictionary
result = connection.execute(table.insert(), x=10, y=5)

# multiple *args no longer accepted, pass a list
result = connection.execute(
    table.insert(), {"x": 10, "y": 5}, {"x": 15, "y": 12}, {"x": 9, "y": 8}
)
```

**迁移到 2.0 版本**

新的`Connection.execute()`方法现在接受一部分被 1.x 版本的`Connection.execute()`方法接受的参数样式，所以以下代码在 1.x 和 2.0 版本之间是兼容的：

```py
connection = engine.connect()

from sqlalchemy import text

result = connection.execute(text("select * from table"))

# pass a single dictionary for single statement execution
result = connection.execute(table.insert(), {"x": 10, "y": 5})

# pass a list of dictionaries for executemany
result = connection.execute(
    table.insert(), [{"x": 10, "y": 5}, {"x": 15, "y": 12}, {"x": 9, "y": 8}]
)
```

**讨论**

删除了对`*args`和`**kwargs`的使用，旨在消除猜测传递给方法的参数类型的复杂性，以及为其他选项腾出空间，即现在可用于在每个语句基础上提供选项的`Connection.execute.execution_options`字典。该方法也修改为其使用模式与`Session.execute()`方法相匹配，后者是 2.0 风格中更突出的 API。

删除直接字符串 SQL 是为了解决`Connection.execute()`和`Session.execute()`之间的不一致性，前者情况下将字符串原始传递给驱动程序，而后者情况下首先将其转换为`text()`构造。通过仅允许`text()`，这也限制了接受的参数格式为“命名”而不是“位置”。最后，字符串 SQL 使用案例越来越受到安全性方面的审查，而`text()`构造已成为明确的界限，将其纳入文本 SQL 领域需要注意不受信任用户输入的关注。

### 结果行表现得像命名元组

**简介**

版本 1.4 引入了一个全新的 Result 对象，该对象反过来返回 `Row` 对象，在“future”模式下行为类似命名元组：

```py
engine = create_engine(..., future=True)  # using future mode

with engine.connect() as conn:
    result = conn.execute(text("select x, y from table"))

    row = result.first()  # suppose the row is (1, 2)

    "x" in row  # evaluates to False, in 1.x / future=False, this would be True

    1 in row  # evaluates to True, in 1.x / future=False, this would be False
```

**迁移到 2.0**

应用代码或测试套件，如果测试某一行是否存在特定键，则需要测试 `row.keys()` 集合。然而，这是一个不寻常的用例，因为结果行通常由已经知道其中存在哪些列的代码使用。

**讨论**

在 1.4 中已经成为一部分的 `KeyedTuple` 类被替换为从 `Query` 对象中选择行时使用的 `Row` 类，该类是使用 `create_engine.future` 标志与 `Engine` 一起使用 Core 语句结果返回的相同 `Row` 的基类（当未设置 `create_engine.future` 标志时，Core 结果集使用 `LegacyRow` 子类，该子类保留了 `__contains__()` 方法的向后兼容行为；ORM 专门直接使用 `Row` 类）。

这个 `Row` 的行为类似于命名元组，因为它充当序列但也支持属性名称访问，例如 `row.some_column`。然而，它还通过特殊属性 `row._mapping` 提供了以前的“映射”行为，该属性产生一个 Python 映射，以便可以使用键访问，例如 `row["some_column"]`。

为了在前期接收到映射结果，可以使用结果的 `mappings()` 修饰符：

```py
from sqlalchemy.future.orm import Session

session = Session(some_engine)

result = session.execute(stmt)
for row in result.mappings():
    print("the user is: %s" % row["User"])
```

ORM 中使用的 `Row` 类还支持通过实体或属性访问：

```py
from sqlalchemy.future import select

stmt = select(User, Address).join(User.addresses)

for row in session.execute(stmt).mappings():
    print("the user is: %s the address is: %s" % (row[User], row[Address]))
```

另请参阅

RowProxy 不再是“代理”；现在被称为 Row 并且行为类似增强的命名元组  ### 库级别（但不是驱动程序级别）“自动提交”从 Core 和 ORM 中删除

**概要**

在 SQLAlchemy 1.x 中，以下语句将自动提交底层的 DBAPI 事务，但在 SQLAlchemy 2.0 中不会发生：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(some_table.insert().values(foo="bar"))
```

也不会自动提交：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(text("INSERT INTO table (foo) VALUES ('bar')"))
```

对于需要提交的自定义 DML 的常见解决方法，“自动提交”执行选项将被删除：

```py
conn = engine.connect()

# won't autocommit in 2.0
conn.execute(text("EXEC my_procedural_thing()").execution_options(autocommit=True))
```

**迁移到 2.0**

与 1.x 风格 和 2.0 风格 执行跨兼容的方法是使用 `Connection.begin()` 方法，或者 `Engine.begin()` 上下文管理器：

```py
with engine.begin() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.execute(some_other_table.insert().values(bat="hoho"))

with engine.connect() as conn:
    with conn.begin():
        conn.execute(some_table.insert().values(foo="bar"))
        conn.execute(some_other_table.insert().values(bat="hoho"))

with engine.begin() as conn:
    conn.execute(text("EXEC my_procedural_thing()"))
```

当使用 `create_engine.future` 标志的 2.0 风格 时，也可以使用“随时提交”风格，因为 `Connection` 具有 **自动开始** 行为，当第一次调用语句时，在没有显式调用 `Connection.begin()` 的情况下发生：

```py
with engine.connect() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.execute(some_other_table.insert().values(bat="hoho"))

    conn.commit()
```

当启用 2.0 弃用模式 时，将在发生已弃用的 “自动提交” 功能时发出警告，指示应明确注意事务的地方。

**讨论**

SQLAlchemy 的最初版本与 Python DBAPI（[**PEP 249**](https://peps.python.org/pep-0249/)）的精神相抵触，因为它试图隐藏 [**PEP 249**](https://peps.python.org/pep-0249/) 对事务的 “隐式开始” 和 “显式提交”的强调。十五年后，我们现在看到这实际上是一个错误，因为 SQLAlchemy 的许多尝试“隐藏”事务存在的模式使得 API 更加复杂，工作不一致，并且对于那些新手用户尤其是对关系数据库和 ACID 事务一般来说极其混乱的用户来说。SQLAlchemy 2.0 将放弃所有隐式提交事务的尝试，使用模式将始终要求用户以某种方式划分事务的 “开始” 和 “结束”，就像在 Python 中读取或写入文件一样具有 “开始” 和 “结束”。

在纯文本语句的自动提交案例中，实际上有一个正则表达式来解析每个语句以检测自动提交！毫不奇怪，这个正则表达式不断地无法适应各种暗示向数据库“写入”的语句和存储过程，导致持续混淆，因为一些语句在数据库中产生结果，而其他语句则没有。通过阻止用户意识到事务概念，我们在这一点上收到了很多错误报告，因为用户不明白数据库是否总是使用事务，无论某一层是否自动提交它。

SQLAlchemy 2.0 将要求在每个级别的数据库操作都要明确指定事务的使用方式。对于绝大多数 Core 使用情况，这已经是推荐的模式：

```py
with engine.begin() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
```

对于“边提交边回滚”使用方式，类似于如何今天通常使用`Session`，使用了`create_engine.future`标志创建的`Engine`返回的“未来”版本的`Connection`包含新的`Connection.commit()`和`Connection.rollback()`方法，这些方法在首次调用语句时会自动开始一个事务：

```py
# 1.4 / 2.0 code

from sqlalchemy import create_engine

engine = create_engine(..., future=True)

with engine.connect() as conn:
    conn.execute(some_table.insert().values(foo="bar"))
    conn.commit()

    conn.execute(text("some other SQL"))
    conn.rollback()
```

在上面，`engine.connect()`方法将返回一个具有**autobegin**功能的`Connection`，意味着在首次使用执行方法时会触发`begin()`事件（但请注意，Python DBAPI 中实际上没有“BEGIN”）。 “autobegin”是 SQLAlchemy 1.4 中的一个新模式，由`Connection`和 ORM `Session` 对象共同支持；autobegin 允许在首次获取对象时显式调用`Connection.begin()`方法，用于希望标记事务开始的方案，但如果未调用该方法，则在首次对对象进行操作时会隐式发生。

“自动提交”功能的移除与讨论中的“隐式”和“无连接”执行的移除密切相关，详见“隐式”和“无连接”执行，“绑定元数据”移除。所有这些传统模式都源自于 Python 在 SQLAlchemy 最初创建时没有上下文管理器或装饰器的事实，因此没有方便的惯用模式来标记资源的使用。

#### 驱动程序级别的自动提交仍然可用

大多数 DBAPI 实现现在广泛支持真正的“自动提交”行为，并且通过 SQLAlchemy 支持`Connection.execution_options.isolation_level`参数，如设置事务隔离级别，包括 DBAPI 自动提交中所讨论的那样。真正的自动提交被视为一种“隔离级别”，因此当使用自动提交时，应用程序代码的结构不会发生变化；`Connection.begin()`上下文管理器以及像`Connection.commit()`这样的方法仍然可以使用，它们在数据库驱动程序级别仅仅是空操作。

#### 驱动程序级自动提交仍然可用

大多数 DBAPI 实现现在广泛支持真正的“自动提交”行为，并且通过 SQLAlchemy 支持`Connection.execution_options.isolation_level`参数，如设置事务隔离级别，包括 DBAPI 自动提交中所讨论的那样。真正的自动提交被视为一种“隔离级别”，因此当使用自动提交时，应用程序代码的结构不会发生变化；`Connection.begin()`上下文管理器以及像`Connection.commit()`这样的方法仍然可以使用，它们在数据库驱动程序级别仅仅是空操作。

### “隐式”和“无连接”执行，“绑定元数据”已移除

**概要**

将`Engine`与`MetaData`对象关联的能力，从而使一系列所谓的“无连接”执行模式可用，已被移除：

```py
from sqlalchemy import MetaData

metadata_obj = MetaData(bind=engine)  # no longer supported

metadata_obj.create_all()  # requires Engine or Connection

metadata_obj.reflect()  # requires Engine or Connection

t = Table("t", metadata_obj, autoload=True)  # use autoload_with=engine

result = engine.execute(t.select())  # no longer supported

result = t.select().execute()  # no longer supported
```

**迁移到 2.0**

对于模式级别的模式，需要明确使用`Engine`或`Connection`。`Engine`仍然可以直接用作`MetaData.create_all()`操作或 autoload 操作的连接源。对于执行语句，只有`Connection`对象具有`Connection.execute()`方法（除了 ORM 级别的`Session.execute()`方法）：

```py
from sqlalchemy import MetaData

metadata_obj = MetaData()

# engine level:

# create tables
metadata_obj.create_all(engine)

# reflect all tables
metadata_obj.reflect(engine)

# reflect individual table
t = Table("t", metadata_obj, autoload_with=engine)

# connection level:

with engine.connect() as connection:
    # create tables, requires explicit begin and/or commit:
    with connection.begin():
        metadata_obj.create_all(connection)

    # reflect all tables
    metadata_obj.reflect(connection)

    # reflect individual table
    t = Table("t", metadata_obj, autoload_with=connection)

    # execute SQL statements
    result = connection.execute(t.select())
```

**讨论**

核心文档已经在这里标准化了所需的模式，因此大多数现代应用程序可能不需要在任何情况下进行太多更改，但是仍然有许多应用程序可能仍然依赖于`engine.execute()`调用，这些调用将需要进行调整。

“无连接”执行是指仍然相当流行的从`Engine`调用`.execute()`的模式：

```py
result = engine.execute(some_statement)
```

上述操作隐式获取了一个`Connection`对象，并在其上运行`.execute()`方法。虽然这看起来是一个简单的便利功能，但已经证明会引发几个问题：

+   频繁使用`engine.execute()`调用的程序已经变得普遍，过度使用了原本意图很少使用的功能，导致非事务型应用程序效率低下。新用户对`engine.execute()`和`connection.execute()`之间的区别感到困惑，这两种方法之间的微妙差别通常不为人所理解。

+   该功能依赖于“应用程序级别的自动提交”功能才能有意义，这个功能本身也正在被移除，因为它也是低效且误导性的。

+   为了处理结果集，`Engine.execute`返回一个带有未消耗游标结果的结果对象。这个游标结果必然仍然链接到仍然处于打开事务中的 DBAPI 连接，所有这些在结果集完全消耗了游标中等待的行后被释放。这意味着`Engine.execute`实际上并没有在调用完成时关闭它声称正在管理的连接资源。SQLAlchemy 的“自动关闭”行为调整得足够好，以至于用户通常不会报告这个系统带来的任何负面影响，然而它仍然是 SQLAlchemy 最早版本中剩余的一个过于隐式和低效的系统。

移除“无连接”执行然后导致了更加传统的模式的移除，即“隐式，无连接”执行：

```py
result = some_statement.execute()
```

上述模式具有“无连接”执行的所有问题，另外它依赖于“绑定元数据”模式，SQLAlchemy 多年来一直试图减少这种依赖。这是 SQLAlchemy 在 0.1 版本中宣传的第一个使用模型，当`Connection`对象被引入并且后来 Python 上下文管理器提供了更好的在固定范围内使用资源的模式时，它几乎立即变得过时。

隐式执行被移除后，“绑定元数据”本身在这个系统中也不再有目的。在现代用法中，“绑定元数据”在`MetaData.create_all()`调用以及与`Session`对象一起工作时仍然有些方便，然而让这些函数显式接收一个`Engine`提供了更清晰的应用设计。

#### 多种选择变为一种选择

总的来说，上述执行模式是在 SQLAlchemy 的第一个 0.1 版本发布之前引入的，甚至`Connection`对象都不存在。经过多年的减少这些模式的重要性，“隐式，无连接”执行和“绑定元数据”不再被广泛使用，因此在 2.0 版本中，我们希望最终减少在 Core 中执行语句的选择数量从“多种选择”：

```py
# many choices

# bound metadata?
metadata_obj = MetaData(engine)

# or not?
metadata_obj = MetaData()

# execute from engine?
result = engine.execute(stmt)

# or execute the statement itself (but only if you did
# "bound metadata" above, which means you can't get rid of "bound" if any
# part of your program uses this form)
result = stmt.execute()

# execute from connection, but it autocommits?
conn = engine.connect()
conn.execute(stmt)

# execute from connection, but autocommit isn't working, so use the special
# option?
conn.execution_options(autocommit=True).execute(stmt)

# or on the statement ?!
conn.execute(stmt.execution_options(autocommit=True))

# or execute from connection, and we use explicit transaction?
with conn.begin():
    conn.execute(stmt)
```

到“一种选择”，所谓“一种选择”是指“显式连接与显式事务”；根据需要仍然有一些方法来标记事务块。这“一种选择”是获取一个`Connection`，然后明确标记事务，在操作是写操作的情况下：

```py
# one choice - work with explicit connection, explicit transaction
# (there remain a few variants on how to demarcate the transaction)

# "begin once" - one transaction only per checkout
with engine.begin() as conn:
    result = conn.execute(stmt)

# "commit as you go" - zero or more commits per checkout
with engine.connect() as conn:
    result = conn.execute(stmt)
    conn.commit()

# "commit as you go" but with a transaction block instead of autobegin
with engine.connect() as conn:
    with conn.begin():
        result = conn.execute(stmt)
```

#### 多种选择变为一种选择

总的来说，上述执行模式是在 SQLAlchemy 的第一个 0.1 版本发布之前引入的，甚至在`Connection`对象不存在之前。经过多年的淡化这些模式，“隐式、无连接”执行和“绑定元数据”不再被广泛使用，因此在 2.0 中，我们试图最终减少 Core 中如何执行语句的选择数量从“多种选择”：

```py
# many choices

# bound metadata?
metadata_obj = MetaData(engine)

# or not?
metadata_obj = MetaData()

# execute from engine?
result = engine.execute(stmt)

# or execute the statement itself (but only if you did
# "bound metadata" above, which means you can't get rid of "bound" if any
# part of your program uses this form)
result = stmt.execute()

# execute from connection, but it autocommits?
conn = engine.connect()
conn.execute(stmt)

# execute from connection, but autocommit isn't working, so use the special
# option?
conn.execution_options(autocommit=True).execute(stmt)

# or on the statement ?!
conn.execute(stmt.execution_options(autocommit=True))

# or execute from connection, and we use explicit transaction?
with conn.begin():
    conn.execute(stmt)
```

对“一种选择”，“一种选择”指的是“显式连接与显式事务”; 根据需要，仍然有几种方法来标记事务块。 “一种选择”是获取`Connection`，然后在操作为写操作的情况下显式地标记事务：

```py
# one choice - work with explicit connection, explicit transaction
# (there remain a few variants on how to demarcate the transaction)

# "begin once" - one transaction only per checkout
with engine.begin() as conn:
    result = conn.execute(stmt)

# "commit as you go" - zero or more commits per checkout
with engine.connect() as conn:
    result = conn.execute(stmt)
    conn.commit()

# "commit as you go" but with a transaction block instead of autobegin
with engine.connect() as conn:
    with conn.begin():
        result = conn.execute(stmt)
```

### execute()方法更严格，执行选项更为突出

**概述**

在 SQLAlchemy 2.0 中，可以与`sqlalchemy.engine.Connection()`的执行方法一起使用的参数模式大大简化，删除了许多以前可用的参数模式。1.4 系列中的新 API 在`sqlalchemy.engine.Connection()`中描述。下面的示例说明了需要修改的模式：

```py
connection = engine.connect()

# direct string SQL not supported; use text() or exec_driver_sql() method
result = connection.execute("select * from table")

# positional parameters no longer supported, only named
# unless using exec_driver_sql()
result = connection.execute(table.insert(), ("x", "y", "z"))

# **kwargs no longer accepted, pass a single dictionary
result = connection.execute(table.insert(), x=10, y=5)

# multiple *args no longer accepted, pass a list
result = connection.execute(
    table.insert(), {"x": 10, "y": 5}, {"x": 15, "y": 12}, {"x": 9, "y": 8}
)
```

**迁移到 2.0**

新的`Connection.execute()`方法现在接受了 1.x 版本`Connection.execute()`方法所接受的参数样式的子集，因此以下代码在 1.x 和 2.0 之间是兼容的：

```py
connection = engine.connect()

from sqlalchemy import text

result = connection.execute(text("select * from table"))

# pass a single dictionary for single statement execution
result = connection.execute(table.insert(), {"x": 10, "y": 5})

# pass a list of dictionaries for executemany
result = connection.execute(
    table.insert(), [{"x": 10, "y": 5}, {"x": 15, "y": 12}, {"x": 9, "y": 8}]
)
```

**讨论**

使用`*args`和`**kwargs`已被移除，一方面是为了消除猜测传递给方法的参数类型的复杂性，另一方面是为了为其他选项腾出空间，即现在可以使用的`Connection.execute.execution_options`字典，以便按语句为单位提供选项。该方法还被修改，使其使用模式与`Session.execute()`方法相匹配，后者是 2.0 样式中更为突出的 API。

直接字符串 SQL 的移除是为了解决 `Connection.execute()` 和 `Session.execute()` 之间的不一致性，前者的情况下字符串被原始传递给驱动程序，而在后者的情况下首先将其转换为 `text()` 构造。通过仅允许 `text()` ，这也限制了接受的参数格式为“命名”而不是“位置”。最后，字符串 SQL 使用案例正在越来越多地受到来自安全角度的审查，并且 `text()` 构造已经成为了表示文本 SQL 领域的明确边界，必须注意不受信任的用户输入。

### 结果行的行为类似命名元组

**概要**

版本 1.4 引入了一个 全新的 Result 对象，该对象反过来返回 `Row` 对象，当使用“future”模式时，它们的行为类似命名元组：

```py
engine = create_engine(..., future=True)  # using future mode

with engine.connect() as conn:
    result = conn.execute(text("select x, y from table"))

    row = result.first()  # suppose the row is (1, 2)

    "x" in row  # evaluates to False, in 1.x / future=False, this would be True

    1 in row  # evaluates to True, in 1.x / future=False, this would be False
```

**迁移到 2.0 版本**

应用程序代码或测试套件，如果要测试行中是否存在特定键，则需要测试`row.keys()`集合。然而，这是一个不寻常的用例，因为结果行通常由已经知道其中存在哪些列的代码使用。

**讨论**

已经作为 1.4 版本的一部分，之前用于从`Query`对象中选择行时使用的`KeyedTuple`类已被替换为`Row`类，该类是与使用 `create_engine.future` 标志的 `Engine` 一起返回的相同 `Row` 的基类（当未设置 `create_engine.future` 标志时，Core 结果集使用 `LegacyRow` 子类，该子类保持了 `__contains__()` 方法的向后兼容行为；ORM 专门直接使用 `Row` 类）。

这个 `Row` 的行为类似于命名元组，它作为一个序列，同时也支持属性名称访问，例如 `row.some_column`。然而，它还通过特殊属性 `row._mapping` 提供了以前的“映射”行为，这样就可以使用键访问，例如 `row["some_column"]`。

为了立即接收映射结果，可以在结果上使用 `mappings()` 修饰符：

```py
from sqlalchemy.future.orm import Session

session = Session(some_engine)

result = session.execute(stmt)
for row in result.mappings():
    print("the user is: %s" % row["User"])
```

`Row` 类在 ORM 中的使用也支持通过实体或属性访问：

```py
from sqlalchemy.future import select

stmt = select(User, Address).join(User.addresses)

for row in session.execute(stmt).mappings():
    print("the user is: %s the address is: %s" % (row[User], row[Address]))
```

另请参阅

RowProxy 不再是“代理”；现在称为 Row 并且行为类似于增强的命名元组

## 2.0 迁移 - 核心用法

### select() 不再接受不同的构造参数，列现在按位置传递

**概要**

`select()` 构造以及相关方法 `FromClause.select()` 将不再接受关键字参数来构建诸如 WHERE 子句、FROM 列表和 ORDER BY 等元素。现在列的列表可以按位置发送，而不是作为列表。此外，`case()` 构造现在接受其 WHEN 条件按位置传递，而不是作为列表：

```py
# select_from / order_by keywords no longer supported
stmt = select([1], select_from=table, order_by=table.c.id)

# whereclause parameter no longer supported
stmt = select([table.c.x], table.c.id == 5)

# whereclause parameter no longer supported
stmt = table.select(table.c.id == 5)

# list emits a deprecation warning
stmt = select([table.c.x, table.c.y])

# list emits a deprecation warning
case_clause = case(
    [(table.c.x == 5, "five"), (table.c.x == 7, "seven")],
    else_="neither five nor seven",
)
```

**迁移到 2.0**

只支持“生成”风格的 `select()`。应该按位置传递要从中选择的列 / 表的列表。SQLAlchemy 1.4 中的 `select()` 构造接受传统风格和新风格，使用自动检测方案，因此下面的代码与 1.4 和 2.0 兼容：

```py
# use generative methods
stmt = select(1).select_from(table).order_by(table.c.id)

# use generative methods
stmt = select(table).where(table.c.id == 5)

# use generative methods
stmt = table.select().where(table.c.id == 5)

# pass columns clause expressions positionally
stmt = select(table.c.x, table.c.y)

# case conditions passed positionally
case_clause = case(
    (table.c.x == 5, "five"), (table.c.x == 7, "seven"), else_="neither five nor seven"
)
```

**讨论**

多年来，SQLAlchemy 已经发展了一个约定，即 SQL 构造可以接受列表或位置参数作为参数。该约定规定，形成 SQL 语句结构的“结构”元素应该按位置传递。相反，形成 SQL 语句参数化数据的“数据”元素应该作为列表传递。多年来，`select()` 构造无法顺利参与这个约定，因为“WHERE”子句的传递方式是按位置传递的。SQLAlchemy 2.0 最终通过将 `select()` 构造更改为仅接受多年来一直是核心教程中唯一记录的样式的“生成”样式来解决了这个问题。

“结构”与“数据”元素的示例如下：

```py
# table columns for CREATE TABLE - structural
table = Table("table", metadata_obj, Column("x", Integer), Column("y", Integer))

# columns in a SELECT statement - structural
stmt = select(table.c.x, table.c.y)

# literal elements in an IN clause - data
stmt = stmt.where(table.c.y.in_([1, 2, 3]))
```

另请参阅

select()，case() 现在接受位置表达式

在“传统”模式下创建的 select() 构造；关键字参数等

### 插入/更新/删除 DML 不再接受关键字构造参数

**概要**

与前面对 `select()` 的更改类似，除了表参数之外，`insert()`、`update()` 和 `delete()` 的构造参数基本上被移除了：

```py
# no longer supported
stmt = insert(table, values={"x": 10, "y": 15}, inline=True)

# no longer supported
stmt = insert(table, values={"x": 10, "y": 15}, returning=[table.c.x])

# no longer supported
stmt = table.delete(table.c.x > 15)

# no longer supported
stmt = table.update(table.c.x < 15, preserve_parameter_order=True).values(
    [(table.c.y, 20), (table.c.x, table.c.y + 10)]
)
```

**迁移到 2.0**

以下示例说明了上述示例的生成方法的使用：

```py
# use generative methods, **kwargs OK for values()
stmt = insert(table).values(x=10, y=15).inline()

# use generative methods, dictionary also still  OK for values()
stmt = insert(table).values({"x": 10, "y": 15}).returning(table.c.x)

# use generative methods
stmt = table.delete().where(table.c.x > 15)

# use generative methods, ordered_values() replaces preserve_parameter_order
stmt = (
    table.update()
    .where(
        table.c.x < 15,
    )
    .ordered_values((table.c.y, 20), (table.c.x, table.c.y + 10))
)
```

**讨论**

API 和内部正在简化 DML 构造，方式与 `select()` 构造相似。

### select() 不再接受各种构造参数，列按位置传递

**概要**

`select()` 构造以及相关方法 `FromClause.select()` 将不再接受关键字参数来构建 WHERE 子句、FROM 列表和 ORDER BY 等元素。现在列的列表可以按位置发送，而不是作为列表。此外，`case()` 构造现在接受其 WHEN 条件的位置传递，而不是作为列表：

```py
# select_from / order_by keywords no longer supported
stmt = select([1], select_from=table, order_by=table.c.id)

# whereclause parameter no longer supported
stmt = select([table.c.x], table.c.id == 5)

# whereclause parameter no longer supported
stmt = table.select(table.c.id == 5)

# list emits a deprecation warning
stmt = select([table.c.x, table.c.y])

# list emits a deprecation warning
case_clause = case(
    [(table.c.x == 5, "five"), (table.c.x == 7, "seven")],
    else_="neither five nor seven",
)
```

**迁移到 2.0**

只支持 `select()` 的 “生成” 样式。应该将要从中选择的列 / 表的列表位置传递。SQLAlchemy 1.4 中的 `select()` 构造接受遗留样式和使用自动检测方案的新样式，因此下面的代码与 1.4 和 2.0 兼容：

```py
# use generative methods
stmt = select(1).select_from(table).order_by(table.c.id)

# use generative methods
stmt = select(table).where(table.c.id == 5)

# use generative methods
stmt = table.select().where(table.c.id == 5)

# pass columns clause expressions positionally
stmt = select(table.c.x, table.c.y)

# case conditions passed positionally
case_clause = case(
    (table.c.x == 5, "five"), (table.c.x == 7, "seven"), else_="neither five nor seven"
)
```

**讨论**

SQLAlchemy 多年来一直开发了一种约定，用于接受参数作为列表或位置参数的 SQL 构造。这个约定规定，**结构**元素，即形成 SQL 语句结构的元素，应该以**位置**方式传递。相反，**数据**元素，即形成 SQL 语句参数化数据的元素，应该作为**列表**传递。多年来，`select()` 构造由于非常古老的调用模式而无法顺利参与这个约定，其中 “WHERE” 子句将以位置方式传递。SQLAlchemy 2.0 最终通过将 `select()` 构造更改为只接受多年来一直是核心教程中唯一记录的样式的 “生成” 样式来解决了这个问题。

“结构”与“数据”元素的示例如下：

```py
# table columns for CREATE TABLE - structural
table = Table("table", metadata_obj, Column("x", Integer), Column("y", Integer))

# columns in a SELECT statement - structural
stmt = select(table.c.x, table.c.y)

# literal elements in an IN clause - data
stmt = stmt.where(table.c.y.in_([1, 2, 3]))
```

另请参阅

select(), case() 现在接受位置表达式

`select()` 构造创建为“遗留”模式；关键字参数等

### insert/update/delete DML 不再接受关键字构造函数参数

**概要**

与之前对 `select()` 的更改类似，除表参数之外的 `insert()`、`update()` 和 `delete()` 的构造函数参数基本上被移除：

```py
# no longer supported
stmt = insert(table, values={"x": 10, "y": 15}, inline=True)

# no longer supported
stmt = insert(table, values={"x": 10, "y": 15}, returning=[table.c.x])

# no longer supported
stmt = table.delete(table.c.x > 15)

# no longer supported
stmt = table.update(table.c.x < 15, preserve_parameter_order=True).values(
    [(table.c.y, 20), (table.c.x, table.c.y + 10)]
)
```

**迁移到 2.0**

以下示例说明了用于上述示例的生成方法的使用：

```py
# use generative methods, **kwargs OK for values()
stmt = insert(table).values(x=10, y=15).inline()

# use generative methods, dictionary also still  OK for values()
stmt = insert(table).values({"x": 10, "y": 15}).returning(table.c.x)

# use generative methods
stmt = table.delete().where(table.c.x > 15)

# use generative methods, ordered_values() replaces preserve_parameter_order
stmt = (
    table.update()
    .where(
        table.c.x < 15,
    )
    .ordered_values((table.c.y, 20), (table.c.x, table.c.y + 10))
)
```

**讨论**

与 `select()` 构造一样，DML 构造的 API 和内部正在以类似的方式简化。

## 2.0 迁移 - ORM 配置

### 声明式成为一流的 API

**概要**

`sqlalchemy.ext.declarative`包大部分（有一些例外）已移至`sqlalchemy.orm`包。`declarative_base()`和`declared_attr()`函数存在，没有任何行为变化。一个名为`registry`的新超级实现现在作为顶级 ORM 配置构造，还提供基于装饰器的声明性和与声明性注册表集成的经典映射的新支持。

**迁移到 2.0 版本**

更改导入：

```py
from sqlalchemy.ext import declarative_base, declared_attr
```

至：

```py
from sqlalchemy.orm import declarative_base, declared_attr
```

**讨论**

大约十年后，`sqlalchemy.ext.declarative`包现在已集成到`sqlalchemy.orm`命名空间中，除了保留为声明性扩展的“extension”类。更多详细信息请参阅 1.4 迁移指南中的声明性现在与新功能集成到 ORM 中。 

另请参阅

ORM 映射类概述 - 用于声明性、经典映射、数据类、attrs 等的全新统一文档。

声明性现在与新功能集成到 ORM 中

### 原始的“mapper()”函数现在是声明性的核心元素，已重命名

**概要**

`sqlalchemy.orm.mapper()`独立函数在幕后移动，由更高级别的 API 调用。这个函数的新版本是从`registry`对象中获取的方法`registry.map_imperatively()`。

**迁移到 2.0 版本**

与经典映射一起工作的代码应更改导入和代码从：

```py
from sqlalchemy.orm import mapper

mapper(SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)})
```

从中心`registry`对象开始工作：

```py
from sqlalchemy.orm import registry

mapper_reg = registry()

mapper_reg.map_imperatively(
    SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)}
)
```

上述`registry`也是声明性映射的来源，经典映射现在可以访问此注册表，包括基于字符串的配置在`relationship()`上：

```py
from sqlalchemy.orm import registry

mapper_reg = registry()

Base = mapper_reg.generate_base()

class SomeRelatedClass(Base):
    __tablename__ = "related"

    # ...

mapper_reg.map_imperatively(
    SomeClass,
    some_table,
    properties={
        "related": relationship(
            "SomeRelatedClass",
            primaryjoin="SomeRelatedClass.related_id == SomeClass.id",
        )
    },
)
```

**讨论**

应要求，“经典映射”仍然存在，但其新形式基于`registry`对象，并且可通过`registry.map_imperatively()`使用。

此外，“经典映射”的主要理由是将 `Table` 设置与类分开。声明性始终允许使用所谓的 混合声明性 风格。但是，为了消除基类要求，已添加了一流的 装饰器 形式。

作为另一个单独但相关的增强，还支持 Python 数据类，并添加到声明性装饰器和经典映射形式中。

另见

ORM 映射类概述 - 所有新的统一文档，涵盖声明性、经典映射、数据类、attrs 等。

### 声明性成为一流 API

**概述**

`sqlalchemy.ext.declarative` 包大部分，除了一些例外，都已移至 `sqlalchemy.orm` 包中。`declarative_base()` 和 `declared_attr()` 函数存在，没有任何行为变化。一个名为 `registry` 的新超级实现现在作为顶级 ORM 配置构造存在，它还提供了基于装饰器的声明性和与声明性注册表集成的经典映射的新支持。

**迁移到 2.0**

更改导入：

```py
from sqlalchemy.ext import declarative_base, declared_attr
```

为：

```py
from sqlalchemy.orm import declarative_base, declared_attr
```

**讨论**

十多年来备受欢迎的 `sqlalchemy.ext.declarative` 包现已整合到 `sqlalchemy.orm` 命名空间中，但声明性“扩展”类除外，它们仍然作为声明性扩展。更多详细信息请参阅 1.4 迁移指南的 声明性现在已经与带有新特性的 ORM 整合。

另见

ORM 映射类概述 - 所有新的统一文档，涵盖声明性、经典映射、数据类、attrs 等。

声明性现在已经与带有新特性的 ORM 整合

### 最初的 “mapper()” 函数现在成为声明性的核心元素，重命名为

**概述**

`sqlalchemy.orm.mapper()` 独立函数在幕后移动，由更高级别的 API 调用。此函数的新版本是从 `registry` 对象中获取的方法 `registry.map_imperatively()`。

**迁移到 2.0**

与经典映射一起工作的代码应该从以下形式的导入和代码更改为：

```py
from sqlalchemy.orm import mapper

mapper(SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)})
```

从一个中心`registry`对象中工作：

```py
from sqlalchemy.orm import registry

mapper_reg = registry()

mapper_reg.map_imperatively(
    SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)}
)
```

上述`registry`也是声明式映射的来源，经典映射现在也可以访问此注册表，包括基于字符串的配置在`relationship()`上：

```py
from sqlalchemy.orm import registry

mapper_reg = registry()

Base = mapper_reg.generate_base()

class SomeRelatedClass(Base):
    __tablename__ = "related"

    # ...

mapper_reg.map_imperatively(
    SomeClass,
    some_table,
    properties={
        "related": relationship(
            "SomeRelatedClass",
            primaryjoin="SomeRelatedClass.related_id == SomeClass.id",
        )
    },
)
```

**讨论**

受欢迎的需求，“经典映射”仍然存在，但是它的新形式是基于`registry`对象，并且可作为`registry.map_imperatively()`使用。

此外，“经典映射”所使用的主要原理是将`Table`的设置与类别区分开来。声明式一直以来都允许使用所谓的混合声明式来采用这种风格。然而，为了去除基类的要求，首先增加了一种一流的装饰器形式。

作为另一个单独但相关的增强，还添加了对 Python 数据类的支持，可以同时用于声明式装饰器和经典映射形式。

另请参阅

ORM Mapped Class Overview - 为声明式、经典映射、数据类、attrs 等提供的全新统一文档。

## 2.0 迁移 - ORM 使用

SQLAlchemy 2.0 中最显著的可见变化是与`Session.execute()`结合使用`select()`来运行 ORM 查询，而不是使用`Session.query()`。如其他地方所述，实际上没有计划删除`Session.query()` API 本身，因为它现在是通过内部使用新 API 来实现的，它将作为遗留 API 保留，并且两个 API 都可以自由使用。

下表提供了一般调用形式的变化介绍，并链接到每个技术的文档。单个迁移说明在表格后面的嵌入部分中，可能包含未在此处概述的其他说明。

**主要 ORM 查询模式概述**

| 1.x 样式形式 | 2.0 样式形式 | 另请参阅 |
| --- | --- | --- |

|

```py
session.query(User).get(42)
```

|

```py
session.get(User, 42)
```

| ORM 查询 - get() 方法移至 Session |
| --- |

|

```py
session.query(User).all()
```

|

```py
session.execute(
  select(User)
).scalars().all()

# or

session.scalars(
  select(User)
).all()
```

| ORM 查询与 Core Select 统一`Session.scalars()` `Result.scalars()` |
| --- |

|

```py
session.query(User).\
  filter_by(name="some user").\
  one()
```

|

```py
session.execute(
  select(User).
  filter_by(name="some user")
).scalar_one()
```

| ORM 查询与 Core Select 统一`Result.scalar_one()` |
| --- |

|

```py
session.query(User).\
  filter_by(name="some user").\
  first()
```

|

```py
session.scalars(
  select(User).
  filter_by(name="some user").
  limit(1)
).first()
```

| ORM 查询与 Core Select 统一`Result.first()` |
| --- |

|

```py
session.query(User).options(
  joinedload(User.addresses)
).all()
```

|

```py
session.scalars(
  select(User).
  options(
    joinedload(User.addresses)
  )
).unique().all()
```

| ORM 行默认情况下不唯一化 |
| --- |

|

```py
session.query(User).\
  join(Address).\
  filter(
    Address.email == "e@sa.us"
  ).\
  all()
```

|

```py
session.execute(
  select(User).
  join(Address).
  where(
    Address.email == "e@sa.us"
  )
).scalars().all()
```

| ORM 查询与 Core Select 统一连接 |
| --- |

|

```py
session.query(User).\
  from_statement(
    text("select * from users")
  ).\
  all()
```

|

```py
session.scalars(
  select(User).
  from_statement(
    text("select * from users")
  )
).all()
```

| 从文本语句获取 ORM 结果 |
| --- |

|

```py
session.query(User).\
  join(User.addresses).\
  options(
    contains_eager(User.addresses)
  ).\
  populate_existing().all()
```

|

```py
session.execute(
  select(User)
  .join(User.addresses)
  .options(
    contains_eager(User.addresses)
  )
  .execution_options(
      populate_existing=True
  )
).scalars().all()
```

| ORM 执行选项填充现有 |
| --- |

|

```py
session.query(User).\
  filter(User.name == "foo").\
  update(
    {"fullname": "Foo Bar"},
    synchronize_session="evaluate"
  )
```

|

```py
session.execute(
  update(User)
  .where(User.name == "foo")
  .values(fullname="Foo Bar")
  .execution_options(
    synchronize_session="evaluate"
  )
)
```

| 启用 ORM 的 INSERT、UPDATE 和 DELETE 语句 |
| --- |

|

```py
session.query(User).count()
```

|

```py
session.scalar(
  select(func.count()).
  select_from(User)
)

# or

session.scalar(
  select(func.count(User.id))
)
```

| `Session.scalar()` |
| --- |

### ORM 查询与 Core Select 统一

**概要**

`Query` 对象（以及 `BakedQuery` 和 `ShardedQuery` 扩展）成为长期遗留对象，被直接使用 `select()` 构造与 `Session.execute()` 方法取代。从 `Query` 返回的对象形式为对象或元组的列表，或作为标量 ORM 对象返回的结果统一作为 `Result` 对象，其接口与 Core 执行一致。

以下是遗留代码示例：

```py
session = Session(engine)

# becomes legacy use case
user = session.query(User).filter_by(name="some user").one()

# becomes legacy use case
user = session.query(User).filter_by(name="some user").first()

# becomes legacy use case
user = session.query(User).get(5)

# becomes legacy use case
for user in (
    session.query(User).join(User.addresses).filter(Address.email == "some@email.com")
):
    ...

# becomes legacy use case
users = session.query(User).options(joinedload(User.addresses)).order_by(User.id).all()

# becomes legacy use case
users = session.query(User).from_statement(text("select * from users")).all()

# etc
```

**迁移至 2.0**

因为 ORM 应用程序的绝大部分预期会使用 `Query` 对象，并且 `Query` 接口的可用性不会影响新接口，该对象将在 2.0 版本中保留，但将不再是文档的一部分，也大多不会得到支持。 `select()` 构造现在适用于 Core 和 ORM 用例，当通过 `Session.execute()` 方法调用时，将返回 ORM 导向的结果，也就是说，如果需要 ORM 对象，那么就返回 ORM 对象。

`Select()` 构造 **添加了许多新方法**，以与 `Query` 兼容，包括 `Select.filter()`、`Select.filter_by()`、重新设计的 `Select.join()` 和 `Select.outerjoin()` 方法，`Select.options()` 等。 `Query` 的其他补充方法，例如 `Query.populate_existing()`，则通过执行选项来实现。

返回结果以 `Result` 对象形式呈现，这是 SQLAlchemy `ResultProxy` 对象的新版本，还添加了许多与 `Query` 兼容的新方法，包括 `Result.one()`、`Result.all()`、`Result.first()`、`Result.one_or_none()` 等。

然而，`Result`对象确实需要一些不同的调用模式，因为当首次返回时，它将**始终返回元组**，并且**不会在内存中去重结果**。为了像`Query`那样返回单个 ORM 对象，必须首先调用`Result.scalars()`修饰符。为了返回唯一的对象，当使用连接式急加载时是必要的，必须首先调用`Result.unique()`修饰符。

包括执行选项等在内的所有新特性的`select()`的文档在 ORM 查询指南中。

以下是迁移到`select()`的一些示例：

```py
session = Session(engine)

user = session.execute(select(User).filter_by(name="some user")).scalar_one()

# for first(), no LIMIT is applied automatically; add limit(1) if LIMIT
# is desired on the query
user = (
    session.execute(select(User).filter_by(name="some user").limit(1)).scalars().first()
)

# get() moves to the Session directly
user = session.get(User, 5)

for user in session.execute(
    select(User).join(User.addresses).filter(Address.email == "some@email.case")
).scalars():
    ...

# when using joinedload() against collections, use unique() on the result
users = (
    session.execute(select(User).options(joinedload(User.addresses)).order_by(User.id))
    .unique()
    .all()
)

# select() has ORM-ish methods like from_statement() that only work
# if the statement is against ORM entities
users = (
    session.execute(select(User).from_statement(text("select * from users")))
    .scalars()
    .all()
)
```

**讨论**

SQLAlchemy 既有`select()`构造，也有一个独立的`Query`对象，其接口非常相似，但基本上是不兼容的，这可能是 SQLAlchemy 中最大的不一致性，这是随着时间的小增量添加而产生的，导致了两个不同的主要 API。

在 SQLAlchemy 的最初版本中，`Query`对象根本不存在。最初的想法是`Mapper`构造本身将能够选择行，并且`Table`对象，而不是类，将用于以 Core 风格创建各种条件。`Query`在 SQLAlchemy 的历史中的一些月份/年作为一个名为`SelectResults`的新“可构建”查询对象的用户提案被接受。像`.where()`方法这样的概念，`SelectResults`称为`.filter()`，在 SQLAlchemy 中以前不存在，并且`select()`构造仅使用现在已弃用的“一次性”构造样式，该样式在 select()不再接受不同的构造参数，列按位置传递中。

随着新方法的推出，对象演变为 `Query` 对象，新增功能如能够选择单个列、能够一次选择多个实体、能够从 `Query` 对象而不是从 `select` 对象构建子查询。目标是使 `Query` 具有 `select` 的全部功能，可以完全组合构建 SELECT 语句，无需显式使用 `select()`。同时，`select()` 也发展出了“生成”方法，如 `Select.where()` 和 `Select.order_by()`。

在现代 SQLAlchemy 中，这一目标已经实现，这两个对象现在在功能上完全重叠。统一这些对象的主要挑战是，`select()` 对象需要保持**与 ORM 完全无关**。为了实现这一点，`Query` 中的绝大部分逻辑已经移至 SQL 编译阶段，其中 ORM 特定的编译器插件接收 `Select` 构造并根据 ORM 风格的查询内容解释其内容，然后传递给核心级别的编译器以创建 SQL 字符串。随着新的 SQL 编译缓存系统的出现，大部分这种 ORM 逻辑也被缓存。

另请参阅

ORM 查询现在与 select、update、delete 内部统一；2.0 风格执行可用  ### ORM 查询 - get() 方法移至 Session

**简介**

`Query.get()` 方法仍保留以供遗留目的，但主要接口现在是 `Session.get()` 方法：

```py
# legacy usage
user_obj = session.query(User).get(5)
```

**迁移至 2.0**

在 1.4 / 2.0 中，`Session`对象添加了一个新的`Session.get()`方法：

```py
# 1.4 / 2.0 cross-compatible use
user_obj = session.get(User, 5)
```

**讨论**

在 2.0 中，`Query`对象将成为遗留对象，因为现在可以使用`select()`对象进行 ORM 查询。由于`Query.get()`方法与`Session`定义了特殊的交互，并且甚至不一定会发出查询，因此更适合将其作为`Session`的一部分，其中它类似于其他“标识”方法，例如`refresh`和`merge`。

SQLAlchemy 最初包含“get()”以类似于 Hibernate 的`Session.load()`方法。就像经常发生的情况一样，我们稍微做错了，因为这种方法实际上更多地涉及`Session`而不是编写 SQL 查询。### ORM 查询 - 使用属性而不是字符串进行关系连接/加载

**概要**

这指的是诸如`Query.join()`之类的模式，以及查询选项，如`joinedload()`，它目前接受字符串属性名称或实际类属性的混合。字符串形式将在 2.0 中全部移除：

```py
# string use removed
q = session.query(User).join("addresses")

# string use removed
q = session.query(User).options(joinedload("addresses"))

# string use removed
q = session.query(Address).filter(with_parent(u1, "addresses"))
```

**升级到 2.0**

现代 SQLAlchemy 1.x 版本支持推荐的技术，即使用映射属性：

```py
# compatible with all modern SQLAlchemy versions

q = session.query(User).join(User.addresses)

q = session.query(User).options(joinedload(User.addresses))

q = session.query(Address).filter(with_parent(u1, User.addresses))
```

2.0 风格的使用也适用相同的技术：

```py
# SQLAlchemy 1.4 / 2.0 cross compatible use

stmt = select(User).join(User.addresses)
result = session.execute(stmt)

stmt = select(User).options(joinedload(User.addresses))
result = session.execute(stmt)

stmt = select(Address).where(with_parent(u1, User.addresses))
result = session.execute(stmt)
```

**讨论**

字符串调用形式不明确，并且需要内部进行额外的工作以确定适当的路径并检索正确的映射属性。通过直接传递 ORM 映射属性，不仅可以提前传递必要的信息，而且该属性还是经过类型化的，并且更具有与 IDE 和 pep-484 集成的潜在兼容性。

### ORM 查询 - 使用属性列表而不是单独调用进行链式调用已移除

**概要**

接受多个映射属性列表的“链接”形式和加载器选项将被移除：

```py
# chaining removed
q = session.query(User).join("orders", "items", "keywords")
```

**升级到 2.0**

对于 1.x / 2.0 跨版本兼容使用，使用单独的调用`Query.join()`：

```py
q = session.query(User).join(User.orders).join(Order.items).join(Item.keywords)
```

对于 2.0 风格 使用，`Select` 具有与 `Select.join()` 相同的行为，还具有一个新的 `Select.join_from()` 方法，允许显式左侧：

```py
# 1.4 / 2.0 cross compatible

stmt = select(User).join(User.orders).join(Order.items).join(Item.keywords)
result = session.execute(stmt)

# join_from can also be helpful
stmt = select(User).join_from(User, Order).join_from(Order, Item, Order.items)
result = session.execute(stmt)
```

**讨论**

移除属性的链接符合简化方法的调用接口，例如 `Select.join()`。

### ORM 查询 - join(…, aliased=True)，移除了 from_joinpoint

**简介**

`Query.join()` 上的 `aliased=True` 选项被移除，`from_joinpoint` 标志也被移除：

```py
# no longer supported
q = (
    session.query(Node)
    .join("children", aliased=True)
    .filter(Node.name == "some sub child")
    .join("children", from_joinpoint=True, aliased=True)
    .filter(Node.name == "some sub sub child")
)
```

**迁移到 2.0 版本**

使用显式别名代替：

```py
n1 = aliased(Node)
n2 = aliased(Node)

q = (
    select(Node)
    .join(Node.children.of_type(n1))
    .where(n1.name == "some sub child")
    .join(n1.children.of_type(n2))
    .where(n2.name == "some sub child")
)
```

**讨论**

`Query.join()` 上的 `aliased=True` 选项是另一个几乎从未被使用过的功能，根据广泛的代码搜索来查找实际使用这个功能的情况。`aliased=True` 标志需要的内部复杂性是**巨大**的，并且将在 2.0 版本中被移除。

大多数用户不熟悉这个标志，但它允许沿着连接自动对元素进行别名，然后将自动别名应用于过滤条件。最初的用例是帮助长链的自引用连接，就像上面显示的例子一样。然而，过滤条件的自动调整在内部是极其复杂的，并且在现实世界的应用中几乎从不使用。这种模式还会导致问题，例如，如果在链中的每个链接中需要添加过滤条件；那么模式必须使用 `from_joinpoint` 标志，而 SQLAlchemy 开发人员绝对找不到此参数在现实世界的应用中的任何使用情况。

`aliased=True` 和 `from_joinpoint` 参数是在`Query` 对象在关系属性上还没有很好的加入能力时开发的，像`PropComparator.of_type()` 这样的函数还不存在，而`aliased()` 构造本身在早期也不存在。### 使用 DISTINCT 与其他列，但仅选择实体

**简介**

当使用 distinct 时，`Query`将自动添加 ORDER BY 中的列。以下查询将从所有 User 列以及“address.email_address”中选择，但只返回 User 对象：

```py
# 1.xx code

result = (
    session.query(User)
    .join(User.addresses)
    .distinct()
    .order_by(Address.email_address)
    .all()
)
```

在 2.0 版本中，“email_address”列不会自动添加到列子句中，上述查询将失败，因为关系数据库在使用 DISTINCT 时不允许您按“address.email_address”排序，如果它也不在列子句中。

**迁移到 2.0**

在 2.0 中，必须显式添加列。要解决仅返回主实体对象而不是额外列的问题，请使用`Result.columns()`方法：

```py
# 1.4 / 2.0 code

stmt = (
    select(User, Address.email_address)
    .join(User.addresses)
    .distinct()
    .order_by(Address.email_address)
)

result = session.execute(stmt).columns(User).all()
```

**讨论**

这种情况是`Query`有限的灵活性的一个例子，导致需要添加隐式的“神奇”行为；“email_address”列被隐式添加到列子句中，然后额外的内部逻辑将从实际返回的结果中省略该列。

新方法简化了交互，并明确了正在发生的事情，同时仍然可以实现原始用例而不会造成不便。 ### 从查询本身选择作为子查询，例如“from_self()”

**概要**

`Query.from_self()`方法将从`Query`中移除：

```py
# from_self is removed
q = (
    session.query(User, Address.email_address)
    .join(User.addresses)
    .from_self(User)
    .order_by(Address.email_address)
)
```

**迁移到 2.0**

`aliased()`构造可以用于根据任意可选择的实体发出 ORM 查询。在 1.4 版本中已增强了对同一子查询多次使用不同实体的支持。这可以在 1.x 风格中与`Query`一起使用，如下所示；请注意，由于最终查询想要根据`User`和`Address`实体查询，因此创建了两个单独的`aliased()`构造：

```py
from sqlalchemy.orm import aliased

subq = session.query(User, Address.email_address).join(User.addresses).subquery()

ua = aliased(User, subq)

aa = aliased(Address, subq)

q = session.query(ua, aa).order_by(aa.email_address)
```

相同的形式可以在 2.0 风格中使用：

```py
from sqlalchemy.orm import aliased

subq = select(User, Address.email_address).join(User.addresses).subquery()

ua = aliased(User, subq)

aa = aliased(Address, subq)

stmt = select(ua, aa).order_by(aa.email_address)

result = session.execute(stmt)
```

**讨论**

`Query.from_self()` 方法是一个非常复杂的方法，很少被使用。该方法的目的是将 `Query` 转换为子查询，然后返回一个从该子查询中 SELECT 的新 `Query`。该方法的复杂之处在于返回的查询会自动将 ORM 实体和列转换为子查询中的 SELECT，同时允许修改要 SELECT 的实体和列。

因为 `Query.from_self()` 在生成的 SQL 中隐含了大量的转换，虽然它确实允许以非常简洁的方式执行某种模式，但这种方法在实际应用中很少见，因为它不容易理解。

新方法利用 `aliased()` 构造，使得 ORM 内部不需要猜测哪些实体和列应该以何种方式适应；在上面的示例中，`ua` 和 `aa` 对象都是 `AliasedClass` 实例，为内部提供了一个明确的标记，指示子查询应该被引用以及对于查询的给定组件考虑了哪个实体列或关系。

SQLAlchemy 1.4 还提供了一种改进的标记风格，不再需要使用包含表名以消除同名列的长标签。在上面的示例中，即使我们的 `User` 和 `Address` 实体具有重叠的列名，我们也可以同时从两个实体中选择而无需指定任何特定的标签：

```py
# 1.4 / 2.0 code

subq = select(User, Address).join(User.addresses).subquery()

ua = aliased(User, subq)
aa = aliased(Address, subq)

stmt = select(ua, aa).order_by(aa.email_address)
result = session.execute(stmt)
```

上述查询将澄清 `User` 和 `Address` 的 `.id` 列，其中 `Address.id` 被呈现和跟踪为 `id_1`：

```py
SELECT  anon_1.id  AS  anon_1_id,  anon_1.id_1  AS  anon_1_id_1,
  anon_1.user_id  AS  anon_1_user_id,
  anon_1.email_address  AS  anon_1_email_address
FROM  (
  SELECT  "user".id  AS  id,  address.id  AS  id_1,
  address.user_id  AS  user_id,  address.email_address  AS  email_address
  FROM  "user"  JOIN  address  ON  "user".id  =  address.user_id
)  AS  anon_1  ORDER  BY  anon_1.email_address
```

[#5221](https://www.sqlalchemy.org/trac/ticket/5221)

### 从其他可选择的实体中选择实体；`Query.select_entity_from()`

**概要**

`Query.select_entity_from()` 方法将在 2.0 中移除：

```py
subquery = session.query(User).filter(User.id == 5).subquery()

user = session.query(User).select_entity_from(subquery).first()
```

**迁移到 2.0**

正如在 从查询本身作为子查询进行选择，例如“from_self()” 中所描述的那样，`aliased()` 对象提供了一个单一的地方，可以实现诸如“从子查询选择实体”之类的操作。使用 1.x 风格：

```py
from sqlalchemy.orm import aliased

subquery = session.query(User).filter(User.name.like("%somename%")).subquery()

ua = aliased(User, subquery)

user = session.query(ua).order_by(ua.id).first()
```

使用 2.0 风格：

```py
from sqlalchemy.orm import aliased

subquery = select(User).where(User.name.like("%somename%")).subquery()

ua = aliased(User, subquery)

# note that LIMIT 1 is not automatically supplied, if needed
user = session.execute(select(ua).order_by(ua.id).limit(1)).scalars().first()
```

**讨论**

这里的要点基本与从查询本身选择子查询，例如“from_self()”讨论的内容相同。 `Query.select_from_entity()`方法是指示查询从替代可选择项加载特定 ORM 映射实体的行的另一种方法，这涉及在以后的查询中，例如在 WHERE 子句或 ORDER BY 中，ORM 将自动为该实体应用别名，如`Query.from_self()`的情况一样，当使用显式`aliased()`对象时，更容易跟踪发生的情况，无论从用户的角度还是从 SQLAlchemy ORM 内部处理的角度。

### 默认情况下 ORM 行不会唯一化

**简介**

`session.execute(stmt)`返回的 ORM 行不再自动“唯一”。这通常是一个受欢迎的变化，除非使用“联接预加载”加载器策略与集合一起使用时：

```py
# In the legacy API, many rows each have the same User primary key, but
# only one User per primary key is returned
users = session.query(User).options(joinedload(User.addresses))

# In the new API, uniquing is available but not implicitly
# enabled
result = session.execute(select(User).options(joinedload(User.addresses)))

# this actually will raise an error to let the user know that
# uniquing should be applied
rows = result.all()
```

**迁移到 2.0 版本**

当使用集合的联接加载时，必须调用`Result.unique()`方法。ORM 实际上将设置一个默认行处理程序，如果未执行此操作，将引发错误，以确保联接预加载集合不返回重复行，同时保持明确性：

```py
# 1.4 / 2.0 code

stmt = select(User).options(joinedload(User.addresses))

# statement will raise if unique() is not used, due to joinedload()
# of a collection.  in all other cases, unique() is not needed.
# By stating unique() explicitly, confusion over discrepancies between
# number of objects/ rows returned vs. "SELECT COUNT(*)" is resolved
rows = session.execute(stmt).unique().all()
```

**讨论**

这里的情况有点不同寻常，因为 SQLAlchemy 要求调用一个方法，而实际上完全有能力自动完成。需要调用该方法的原因是确保开发人员“选择”使用`Result.unique()`方法，这样他们在直接计算行数与实际结果集中记录数不一致时不会感到困惑，这已经是多年来用户困惑和错误报告的长期问题了。默认情况下不会在任何其他情况下进行唯一化，这将提高性能并提高自动唯一化导致混淆结果的情况的清晰度。

在现代 SQLAlchemy 中，当使用联接预加载集合时，必须调用`Result.unique()`方法时，在现代 SQLAlchemy 中，`selectinload()`策略提供了一个集合导向的预加载器，在大多数情况下优于`joinedload()`，应该优先使用。 ### “动态”关系加载器被“仅写入”取代

**简介**

讨论了`lazy="dynamic"`关系加载策略，详见动态关系加载器，它使用了在 2.0 中已经过时的`Query`对象。这种“dynamic”关系不直接兼容 asyncio，除非通过解决方法，此外，它也没有实现其原始目的，即防止大型集合的迭代，因为它有几种行为会隐式发生迭代。

引入了一种名为`lazy="write_only"`的新加载策略，通过`WriteOnlyCollection`集合类提供了一个非常严格的“无隐式迭代”API，并且还与 2.0 风格的语句执行集成，支持 asyncio 以及与新的 ORM 启用的批量 DML 功能集成。

与此同时，在 2.0 版本中仍然**完全支持**`lazy="dynamic"`；应用程序可以延迟迁移这种特定模式，直到完全使用 2.0 系列。

**迁移至 2.0**

新的“write only”功能仅在 SQLAlchemy 2.0 中可用，不是 1.4 的一部分。与此同时，`lazy="dynamic"`加载策略在 2.0 版本中仍然得到完全支持，甚至包括新的 pep-484 和带注释的映射支持。

因此，从“dynamic”迁移的最佳策略是**等到应用程序完全在 2.0 上运行**，然后直接从`AppenderQuery`迁移到`WriteOnlyCollection`，后者是“dynamic”策略使用的集合类型。

一些技术可用于在 1.4 版本下以更“2.0”风格使用`lazy="dynamic"`。有两种方法可以实现 2.0 风格的查询，即针对特定关系的查询：

+   利用现有`lazy="dynamic"`关系上的`Query.statement`属性。我们可以立即使用动态加载器进行方法，如`Session.scalars()`，如下所示：

    ```py
    class User(Base):
        __tablename__ = "user"

        posts = relationship(Post, lazy="dynamic")

    jack = session.get(User, 5)

    # filter Jack's blog posts
    posts = session.scalars(jack.posts.statement.where(Post.headline == "this is a post"))
    ```

+   使用`with_parent()`函数直接构造一个`select()`构造：

    ```py
    from sqlalchemy.orm import with_parent

    jack = session.get(User, 5)

    posts = session.scalars(
        select(Post)
        .where(with_parent(jack, User.posts))
        .where(Post.headline == "this is a post")
    )
    ```

**讨论**

最初的想法是 `with_parent()` 函数应该足够了，但是继续使用关系本身的特殊属性仍然具有吸引力，并且在这里也可以使用 2.0 风格的构造方法。

新的“write_only”加载器策略提供了一种新的集合类型，不支持隐式迭代或项目访问。而是通过调用其 `.select()` 方法来读取集合的内容，以帮助构建适当的 SELECT 语句。该集合还包括 `.insert()`、`.update()`、`.delete()` 方法，用于为集合中的项目发出批量 DML 语句。与“动态”功能类似，还有 `.add()`、`.add_all()` 和 `.remove()` 方法，它们使用工作单元过程将单个成员排队以添加或删除。对新功能的介绍可以查看 新的“仅写”关系策略取代了“动态”。

另请参阅

新的“仅写”关系策略取代了“动态”

仅写关系  ### Session 中删除了自动提交模式；添加了自动开始支持

**概要**

`Session` 将不再支持“自动提交”模式，也就是这种模式：

```py
from sqlalchemy.orm import Session

sess = Session(engine, autocommit=True)

# no transaction begun, but emits SQL, won't be supported
obj = sess.query(Class).first()

# session flushes in a transaction that it begins and
# commits, won't be supported
sess.flush()
```

**2.0 迁移**

使用 `Session` 在“自动提交”模式下的主要原因是为了使 `Session.begin()` 方法可用，以便框架集成和事件挂钩可以控制此事件的发生。在 1.4 中，`Session` 现在具有 自动开始行为，解决了这个问题；现在可以调用 `Session.begin()` 方法：

```py
from sqlalchemy.orm import Session

sess = Session(engine)

sess.begin()  # begin explicitly; if not called, will autobegin
# when database access is needed

sess.add(obj)

sess.commit()
```

**讨论**

“自动提交”模式是 SQLAlchemy 最初版本的遗留问题之一。这个标志主要用于允许显式使用 `Session.begin()`，在 1.4 中已经解决了这个问题，以及允许使用“子事务”，这在 2.0 中也已经移除。### Session “子事务”行为已移除

**概要**

使用自动提交模式时经常使用的“子事务”模式在 1.4 中也已经弃用。这种模式允许在已经开始事务时使用`Session.begin()`方法，导致产生一种称为“子事务”的结构，它本质上是一个阻止`Session.commit()`方法实际提交的块。

**迁移到 2.0**

为了为使用此模式的应用程序提供向后兼容性，可以使用以下上下文管理器或基于装饰器的类似实现：

```py
import contextlib

@contextlib.contextmanager
def transaction(session):
    if not session.in_transaction():
        with session.begin():
            yield
    else:
        yield
```

上述上下文管理器可以以与“子事务”标志相同的方式使用，例如以下示例：

```py
# method_a starts a transaction and calls method_b
def method_a(session):
    with transaction(session):
        method_b(session)

# method_b also starts a transaction, but when
# called from method_a participates in the ongoing
# transaction.
def method_b(session):
    with transaction(session):
        session.add(SomeObject("bat", "lala"))

Session = sessionmaker(engine)

# create a Session and call method_a
with Session() as session:
    method_a(session)
```

为了与首选的惯用模式进行比较，开始块应位于最外层。这样就不需要个别函数或方法关心事务界定的细节：

```py
def method_a(session):
    method_b(session)

def method_b(session):
    session.add(SomeObject("bat", "lala"))

Session = sessionmaker(engine)

# create a Session and call method_a
with Session() as session:
    with session.begin():
        method_a(session)
```

**讨论**

这种模式已被证明在实际应用程序中令人困惑，最好是确保应用程序的最顶层数据库操作使用单个 begin/commit 对执行。### ORM 查询与 Core 选择统一

**简介**

`Query`对象（以及`BakedQuery`和`ShardedQuery`扩展）成为长期遗留对象，取而代之的是直接使用`select()`构造与`Session.execute()`方法结合使用。从`Query`返回的列表对象或元组形式的结果，或作为标量 ORM 对象从`Session.execute()`统一作为`Result`对象返回，其接口与 Core 执行一致。

以下是遗留代码示例：

```py
session = Session(engine)

# becomes legacy use case
user = session.query(User).filter_by(name="some user").one()

# becomes legacy use case
user = session.query(User).filter_by(name="some user").first()

# becomes legacy use case
user = session.query(User).get(5)

# becomes legacy use case
for user in (
    session.query(User).join(User.addresses).filter(Address.email == "some@email.com")
):
    ...

# becomes legacy use case
users = session.query(User).options(joinedload(User.addresses)).order_by(User.id).all()

# becomes legacy use case
users = session.query(User).from_statement(text("select * from users")).all()

# etc
```

**迁移到 2.0**

由于绝大多数 ORM 应用程序预计将使用`Query`对象，并且`Query`接口的可用性不会影响新接口，该对象将在 2.0 版本中保留，但不再是文档的一部分，也大多数情况下不再受支持。 `select()`构造现在适用于 Core 和 ORM 用例，当通过`Session.execute()`方法调用时，将返回 ORM 导向的结果，即如果请求的是 ORM 对象，则返回 ORM 对象。

`Select()`构造**添加了许多新方法**，以与`Query`兼容，包括`Select.filter()`，`Select.filter_by()`，重新设计的`Select.join()`和`Select.outerjoin()`方法，`Select.options()`等。 `Query`的其他更多补充方法，如`Query.populate_existing()`是通过执行选项实现的。

返回结果以`Result`对象的形式呈现，这是 SQLAlchemy `ResultProxy`对象的新版本，还添加了许多与`Query`兼容的新方法，包括`Result.one()`，`Result.all()`，`Result.first()`，`Result.one_or_none()`等。

然而，`Result`对象需要一些不同的调用模式，因为当首次返回时，它将**始终返回元组**，并且**不会在内存中去重结果**。为了以`Query`的方式返回单个 ORM 对象，必须首先调用`Result.scalars()`修饰符。为了返回唯一的对象，就像在使用连接式贪婪加载时所必需的那样，必须首先调用`Result.unique()`修饰符。

包括执行选项等在内的`select()`的所有新功能的文档在 ORM 查询指南中。

以下是一些迁移到`select()`的示例：

```py
session = Session(engine)

user = session.execute(select(User).filter_by(name="some user")).scalar_one()

# for first(), no LIMIT is applied automatically; add limit(1) if LIMIT
# is desired on the query
user = (
    session.execute(select(User).filter_by(name="some user").limit(1)).scalars().first()
)

# get() moves to the Session directly
user = session.get(User, 5)

for user in session.execute(
    select(User).join(User.addresses).filter(Address.email == "some@email.case")
).scalars():
    ...

# when using joinedload() against collections, use unique() on the result
users = (
    session.execute(select(User).options(joinedload(User.addresses)).order_by(User.id))
    .unique()
    .all()
)

# select() has ORM-ish methods like from_statement() that only work
# if the statement is against ORM entities
users = (
    session.execute(select(User).from_statement(text("select * from users")))
    .scalars()
    .all()
)
```

**讨论**

SQLAlchemy 既有一个`select()`构造，也有一个具有极其相似但基本上不兼容的接口的单独的`Query`对象，这可能是 SQLAlchemy 中最大的不一致性，这是随着时间的小幅增加而产生的两个主要 API 的分歧。

在 SQLAlchemy 的最初版本中，根本不存在`Query`对象。最初的想法是`Mapper`构造本身将能够选择行，并且`Table`对象，而不是类，将用于以 Core 风格的方式创建各种条件。`Query`在 SQLAlchemy 的历史中的一些月份/年作为一个名为`SelectResults`的新的“可构建”查询对象的用户提案被接受。像`.where()`方法这样的概念，在`SelectResults`中称为`.filter()`，在 SQLAlchemy 之前不存在，并且`select()`构造仅使用现在已弃用的“一次性”构造样式，该样式不再接受各种构造函数参数，列是按位置传递的。

随着新方法的推出，该对象演变为`Query`对象，新增了诸如能够选择单个列、能够一次选择多个实体、能够从`Query`对象构建子查询而不是从`select`对象开始的新功能。目标是`Query`应该具有`select`的全部功能，可以组合构建完整的 SELECT 语句，而不需要显式使用`select()`。与此同时，`select()`也发展出了“生成”方法，如`Select.where()`和`Select.order_by()`。

在现代的 SQLAlchemy 中，这个目标已经实现，这两个对象现在在功能上完全重叠。统一这些对象的主要挑战是`select()`对象需要保持**对 ORM 完全不可知**。为了实现这一点，大部分来自`Query`的逻辑已经移动到 SQL 编译阶段，ORM 特定的编译器插件接收`Select`构造并根据 ORM 风格的查询解释其内容，然后传递给核心级别的编译器以创建 SQL 字符串。随着新的 SQL 编译缓存系统的出现，大部分这种 ORM 逻辑也被缓存。

另请参阅

ORM 查询在内部与 select、update、delete 统一；2.0 风格执行可用

### ORM 查询 - get() 方法移到 Session

**概要**

`Query.get()` 方法仍然保留以供遗留目的，但主要接口现在是`Session.get()` 方法：

```py
# legacy usage
user_obj = session.query(User).get(5)
```

**迁移到 2.0**

在 1.4 / 2.0 中，`Session`对象添加了一个新的`Session.get()`方法：

```py
# 1.4 / 2.0 cross-compatible use
user_obj = session.get(User, 5)
```

**讨论**

`Query`对象将在 2.0 中成为遗留对象，因为现在可以使用`select()`对象进行 ORM 查询。由于`Query.get()`方法定义了与`Session`的特殊交互，并且甚至不一定会发出查询，因此将其作为`Session`的一部分更为适合，其中类似于其他“identity”方法，例如`refresh`和`merge`。

SQLAlchemy 最初包含“get()”，以类似于 Hibernate `Session.load()`方法。像往常一样，我们稍微搞错了，因为这个方法实际上更多地与`Session`有关，而不是写 SQL 查询。

### ORM 查询 - 在关系上进行链接/加载使用属性，而不是字符串

**概要**

这涉及`Query.join()`等模式，以及查询选项，如`joinedload()`，它目前接受混合的字符串属性名称或实际类属性。字符串形式将在 2.0 中全部移除：

```py
# string use removed
q = session.query(User).join("addresses")

# string use removed
q = session.query(User).options(joinedload("addresses"))

# string use removed
q = session.query(Address).filter(with_parent(u1, "addresses"))
```

**迁移到 2.0**

现代 SQLAlchemy 1.x 版本支持推荐的技术，即使用映射属性：

```py
# compatible with all modern SQLAlchemy versions

q = session.query(User).join(User.addresses)

q = session.query(User).options(joinedload(User.addresses))

q = session.query(Address).filter(with_parent(u1, User.addresses))
```

相同的技术适用于 2.0 样式的使用：

```py
# SQLAlchemy 1.4 / 2.0 cross compatible use

stmt = select(User).join(User.addresses)
result = session.execute(stmt)

stmt = select(User).options(joinedload(User.addresses))
result = session.execute(stmt)

stmt = select(Address).where(with_parent(u1, User.addresses))
result = session.execute(stmt)
```

**讨论**

字符串调用形式不明确，并且需要内部执行额外的工作来确定适当的路径并检索正确的映射属性。通过直接传递 ORM 映射属性，不仅可以提前传递必要的信息，而且该属性还具有类型，并且更具有与 IDE 和 pep-484 集成的潜在兼容性。

### ORM 查询 - 使用属性列表进行链接的链式调用，而不是单个调用，已移除

**概要**

“链接”形式的连接和加载选项接受多个映射属性的列表将被移除：

```py
# chaining removed
q = session.query(User).join("orders", "items", "keywords")
```

**迁移到 2.0**

对于 1.x / 2.0 跨兼容使用，使用单独的调用`Query.join()`：

```py
q = session.query(User).join(User.orders).join(Order.items).join(Item.keywords)
```

对于 2.0 风格的用法，`Select`具有与`Select.join()`相同的行为，并且还具有一个新的`Select.join_from()`方法，允许显式左侧：

```py
# 1.4 / 2.0 cross compatible

stmt = select(User).join(User.orders).join(Order.items).join(Item.keywords)
result = session.execute(stmt)

# join_from can also be helpful
stmt = select(User).join_from(User, Order).join_from(Order, Item, Order.items)
result = session.execute(stmt)
```

**讨论**

移除属性的链接符合简化诸如`Select.join()`等方法的调用接口。

### ORM 查询 - join(…, aliased=True)，移除 from_joinpoint

**概要**

`Query.join()`上的`aliased=True`选项已被移除，`from_joinpoint`标志也被移除：

```py
# no longer supported
q = (
    session.query(Node)
    .join("children", aliased=True)
    .filter(Node.name == "some sub child")
    .join("children", from_joinpoint=True, aliased=True)
    .filter(Node.name == "some sub sub child")
)
```

**迁移至 2.0**

使用显式别名代替：

```py
n1 = aliased(Node)
n2 = aliased(Node)

q = (
    select(Node)
    .join(Node.children.of_type(n1))
    .where(n1.name == "some sub child")
    .join(n1.children.of_type(n2))
    .where(n2.name == "some sub child")
)
```

**讨论**

`Query.join()`上的`aliased=True`选项似乎几乎从未被使用，通过广泛的代码搜索来查找实际使用此功能的情况。`aliased=True`标志所需的内部复杂性是**巨大**的，并且将在 2.0 中消失。

大多数用户不熟悉这个标志，但它允许沿着连接自动为元素添加别名，然后将自动别名应用于过滤条件。最初的用例是帮助处理长链的自引用连接，就像上面显示的示例一样。然而，过滤条件的自动调整在内部非常复杂，几乎从不在现实世界的应用中使用。这种模式还会导致问题，例如如果需要在链中的每个链接处添加过滤条件；那么模式必须使用`from_joinpoint`标志，SQLAlchemy 开发人员绝对找不到这个参数在实际应用中的任何使用情况。

`aliased=True`和`from_joinpoint`参数是在`Query`对象在关联属性方面还没有良好能力时开发的，像`PropComparator.of_type()`这样的函数还不存在，而`aliased()`构造本身在早期也不存在。

### 使用 DISTINCT 与其他列，但仅选择实体

**概要**

当使用 DISTINCT 时，`Query` 将自动添加 ORDER BY 中的列。以下查询将从所有用户列以及“address.email_address” 中选择，但只返回用户对象：

```py
# 1.xx code

result = (
    session.query(User)
    .join(User.addresses)
    .distinct()
    .order_by(Address.email_address)
    .all()
)
```

在 2.0 版本中，“email_address” 列不会自动添加到列子句中，上述查询将失败，因为关系数据库在使用 DISTINCT 时不允许您按“address.email_address” 排序，如果它也不在列子句中。

**迁移到 2.0 版本**

在 2.0 版本中，必须显式添加列。要解决仅返回主实体对象而不是额外列的问题，请使用`Result.columns()` 方法：

```py
# 1.4 / 2.0 code

stmt = (
    select(User, Address.email_address)
    .join(User.addresses)
    .distinct()
    .order_by(Address.email_address)
)

result = session.execute(stmt).columns(User).all()
```

**讨论**

这种情况是`Query`的有限灵活性导致需要添加隐式“神奇”行为的一个例子；“email_address” 列会隐式添加到列子句中，然后额外的内部逻辑会从实际返回的结果中省略该列。

新方法简化了交互并明确了正在发生的事情，同时仍然可以实现原始用例而不会带来不便。

### 从查询本身作为子查询选择，例如“from_self()”

**概要**

`Query.from_self()` 方法将从`Query`中移除：

```py
# from_self is removed
q = (
    session.query(User, Address.email_address)
    .join(User.addresses)
    .from_self(User)
    .order_by(Address.email_address)
)
```

**迁移到 2.0 版本**

`aliased()` 构造可用于针对任意可选择的实体发出 ORM 查询。它在 1.4 版本中已经得到增强，以便顺利地多次针对相同子查询用于不同实体。这可以在 1.x 风格中与`Query`一起使用，注意由于最终查询想要查询`User`和`Address`实体，因此创建了两个单独的`aliased()` 构造：

```py
from sqlalchemy.orm import aliased

subq = session.query(User, Address.email_address).join(User.addresses).subquery()

ua = aliased(User, subq)

aa = aliased(Address, subq)

q = session.query(ua, aa).order_by(aa.email_address)
```

相同的形式可以在 2.0 风格中使用：

```py
from sqlalchemy.orm import aliased

subq = select(User, Address.email_address).join(User.addresses).subquery()

ua = aliased(User, subq)

aa = aliased(Address, subq)

stmt = select(ua, aa).order_by(aa.email_address)

result = session.execute(stmt)
```

**讨论**

`Query.from_self()`方法是一个非常复杂的方法，很少被使用。这个方法的目的是将`Query`转换为一个子查询，然后返回一个从该子查询中 SELECT 的新`Query`。这个方法的复杂之处在于返回的查询应用了 ORM 实体和列的自动翻译，以便以子查询的方式在 SELECT 中陈述，以及允许被 SELECT 的实体和列进行修改。

因为`Query.from_self()`方法在生成的 SQL 中隐含了大量的转换，虽然它确实允许某种类型的模式被非常简洁地执行，但是这种方法在实际应用中很少见，因为它不容易理解。

新方法利用了`aliased()`构造，因此 ORM 内部无需猜测应该如何调整哪些实体和列，以及以何种方式进行调整；在上面的示例中，`ua`和`aa`对象都是`AliasedClass`实例，为内部提供了一个明确的标记，指示子查询应该在何处引用以及正在考虑查询的给定组件的哪个实体列或关系。

SQLAlchemy 1.4 还提供了一种改进的标签样式，不再需要使用包含表名以消除不同表中具有相同名称的列的歧义的长标签。在上面的示例中，即使我们的`User`和`Address`实体具有重叠的列名称，我们也可以同时从两个实体中选择，而无需指定任何特定的标签：

```py
# 1.4 / 2.0 code

subq = select(User, Address).join(User.addresses).subquery()

ua = aliased(User, subq)
aa = aliased(Address, subq)

stmt = select(ua, aa).order_by(aa.email_address)
result = session.execute(stmt)
```

上述查询将区分`User`和`Address`的`.id`列，其中`Address.id`被渲染和跟踪为`id_1`：

```py
SELECT  anon_1.id  AS  anon_1_id,  anon_1.id_1  AS  anon_1_id_1,
  anon_1.user_id  AS  anon_1_user_id,
  anon_1.email_address  AS  anon_1_email_address
FROM  (
  SELECT  "user".id  AS  id,  address.id  AS  id_1,
  address.user_id  AS  user_id,  address.email_address  AS  email_address
  FROM  "user"  JOIN  address  ON  "user".id  =  address.user_id
)  AS  anon_1  ORDER  BY  anon_1.email_address
```

[#5221](https://www.sqlalchemy.org/trac/ticket/5221)

### 从备选可选项选择实体；Query.select_entity_from()

**概要**

`Query.select_entity_from()`方法将在 2.0 中被移除：

```py
subquery = session.query(User).filter(User.id == 5).subquery()

user = session.query(User).select_entity_from(subquery).first()
```

**迁移到 2.0**

就像在从查询本身选择子查询，例如“from_self()”中描述的情况一样，`aliased()`对象提供了一个可以实现“从子查询选择实体”等操作的单一位置。使用 1.x 风格：

```py
from sqlalchemy.orm import aliased

subquery = session.query(User).filter(User.name.like("%somename%")).subquery()

ua = aliased(User, subquery)

user = session.query(ua).order_by(ua.id).first()
```

使用 2.0 风格：

```py
from sqlalchemy.orm import aliased

subquery = select(User).where(User.name.like("%somename%")).subquery()

ua = aliased(User, subquery)

# note that LIMIT 1 is not automatically supplied, if needed
user = session.execute(select(ua).order_by(ua.id).limit(1)).scalars().first()
```

**讨论**

这里的要点基本与从查询本身选择子查询的情况相同，例如“from_self()”讨论的内容相同。 `Query.select_from_entity()` 方法是指示查询从备用可选择的 ORM 映射实体加载行的另一种方法，这涉及到 ORM 在后续查询中无论在何处使用该实体，例如在 WHERE 子句或 ORDER BY 中，都会对该实体进行自动别名。这个非常复杂的特性很少以这种方式使用，就像`Query.from_self()`的情况一样，使用显式的`aliased()`对象时更容易理解正在发生的情况，从用户的角度和 SQLAlchemy ORM 的内部处理方式来看都是如此。

### 默认情况下 ORM 行不唯一

**简介**

`session.execute(stmt)` 返回的 ORM 行不再自动“唯一”。这通常是一个受欢迎的变化，但在使用集合的“联接急加载”加载器的情况下可能会出现问题：

```py
# In the legacy API, many rows each have the same User primary key, but
# only one User per primary key is returned
users = session.query(User).options(joinedload(User.addresses))

# In the new API, uniquing is available but not implicitly
# enabled
result = session.execute(select(User).options(joinedload(User.addresses)))

# this actually will raise an error to let the user know that
# uniquing should be applied
rows = result.all()
```

**迁移到 2.0**

当使用集合的联接加载时，需要调用`Result.unique()`方法。ORM 实际上会设置一个默认的行处理程序，如果未执行此操作，它将引发错误，以确保联接急加载集合不会返回重复的行，同时保持显式性：

```py
# 1.4 / 2.0 code

stmt = select(User).options(joinedload(User.addresses))

# statement will raise if unique() is not used, due to joinedload()
# of a collection.  in all other cases, unique() is not needed.
# By stating unique() explicitly, confusion over discrepancies between
# number of objects/ rows returned vs. "SELECT COUNT(*)" is resolved
rows = session.execute(stmt).unique().all()
```

**讨论**

这里的情况有点不同寻常，因为 SQLAlchemy 要求调用一个它完全可以自动执行的方法。要求调用该方法的原因是确保开发人员“选择”使用`Result.unique()` 方法，这样当行数的直接计数与实际结果集中的记录计数不冲突时，他们不会感到困惑，这已经是多年来用户困惑和错误报告的长期问题。默认情况下，不在任何其他情况下唯一化将提高性能，并且在自动唯一化导致混淆的情况下将提高清晰度。

要调用`Result.unique()`对联接的急加载集合进行处理可能有些不方便，在现代的 SQLAlchemy 中，`selectinload()` 策略提供了一个针对集合的急加载器，在大多数情况下优于`joinedload()`，应该优先考虑使用。

### “动态”关系加载器被“只写”替代

**简介**

讨论过的 `lazy="dynamic"` 关系加载策略，利用了 2.0 版本中的遗留 `Query` 对象。 “动态”关系在没有解决方案的情况下不直接兼容 asyncio，此外，它也没有实现其原始目的，即防止大型集合的迭代，因为它有几种隐式发生迭代的行为。

引入了一种名为 `lazy="write_only"` 的新加载策略，通过 `WriteOnlyCollection` 集合类提供了一个非常严格的“无隐式迭代”API，并且与 2.0 风格的语句执行集成，支持 asyncio 以及与新的 ORM-enabled Bulk DML 功能集的直接集成。

与此同时，`lazy="dynamic"` 在 2.0 版本中仍然得到**全面支持**；应用程序可以延迟迁移这种特定模式，直到完全升级到 2.0 系列。

**迁移到 2.0**

新的“write only”功能仅在 SQLAlchemy 2.0 中可用，不是 1.4 的一部分。与此同时，`lazy="dynamic"` 加载策略在 2.0 版本中仍然得到全面支持，甚至包括新的 pep-484 和注释映射支持。

因此，从“dynamic”迁移的最佳策略是**等到应用程序完全运行在 2.0 上**，然后直接从 `AppenderQuery` 迁移到 `WriteOnlyCollection`，后者是“write_only”策略使用的集合类型。

有一些技术可用于在 1.4 版本下以更“2.0”风格使用 `lazy="dynamic"`。有两种方法可以实现特定关系的 2.0 风格查询：

+   利用现有 `lazy="dynamic"` 关系上的 `Query.statement` 属性。我们可以立即使用像 `Session.scalars()` 这样的方法与动态加载器一起使用，如下所示：

    ```py
    class User(Base):
        __tablename__ = "user"

        posts = relationship(Post, lazy="dynamic")

    jack = session.get(User, 5)

    # filter Jack's blog posts
    posts = session.scalars(jack.posts.statement.where(Post.headline == "this is a post"))
    ```

+   使用 `with_parent()` 函数直接构造一个 `select()` 构造：

    ```py
    from sqlalchemy.orm import with_parent

    jack = session.get(User, 5)

    posts = session.scalars(
        select(Post)
        .where(with_parent(jack, User.posts))
        .where(Post.headline == "this is a post")
    )
    ```

**讨论**

最初的想法是`with_parent()`函数应该足够了，但是继续利用关系本身上的特殊属性仍然吸引人，并且没有理由不能在这里使用 2.0 风格的构造。

新的“仅写”加载策略提供了一种不支持隐式迭代或项访问的新类型集合。相反，通过调用其`.select()`方法来读取集合的内容，以帮助构造适当的 SELECT 语句。该集合还包括`.insert()`、`.update()`、`.delete()`等方法，可用于对集合中的项目发出批量 DML 语句。类似于“动态”功能，该特性还包括`.add()`、`.add_all()`和`.remove()`方法，使用工作单元过程为单个成员排队进行添加或删除。有关该新功能的介绍，请参阅新的“仅写”关系策略取代“动态”。

请参见

新的“仅写”关系策略取代“动态”

仅写关系

### 从 Session 中移除了自动提交模式；添加了自动开始支持

**简介**

`Session`将不再支持“自动提交”模式，即以下模式：

```py
from sqlalchemy.orm import Session

sess = Session(engine, autocommit=True)

# no transaction begun, but emits SQL, won't be supported
obj = sess.query(Class).first()

# session flushes in a transaction that it begins and
# commits, won't be supported
sess.flush()
```

**迁移到 2.0**

使用“自动提交”模式的`Session`的主要原因是使`Session.begin()`方法可用，以便框架集成和事件钩子可以控制此事件何时发生。在 1.4 中，`Session`现在具有自动开始行为，解决了此问题；现在可以调用`Session.begin()`方法：

```py
from sqlalchemy.orm import Session

sess = Session(engine)

sess.begin()  # begin explicitly; if not called, will autobegin
# when database access is needed

sess.add(obj)

sess.commit()
```

**讨论**

“自动提交”模式是 SQLAlchemy 初始版本的另一个遗留功能。这个标志一直存在主要是为了支持允许显式使用`Session.begin()`，这在 1.4 版本中已经解决，以及允许使用“子事务”，这在 2.0 版本中也被移除了。

### 移除了会话“子事务”行为

**简介**

在 1.4 中，经常与自动提交模式一起使用的“子事务”模式已被弃用。这种模式允许在事务已经开始时使用 `Session.begin()` 方法，导致一个称为“子事务”的构造，其本质上是一个阻止 `Session.commit()` 方法实际提交的块。

**迁移至 2.0**

为了向使用这种模式的应用程序提供向后兼容性，可以使用以下上下文管理器或基于装饰器的类似实现：

```py
import contextlib

@contextlib.contextmanager
def transaction(session):
    if not session.in_transaction():
        with session.begin():
            yield
    else:
        yield
```

上述上下文管理器可以像“子事务”标志的工作方式一样使用，例如以下示例：

```py
# method_a starts a transaction and calls method_b
def method_a(session):
    with transaction(session):
        method_b(session)

# method_b also starts a transaction, but when
# called from method_a participates in the ongoing
# transaction.
def method_b(session):
    with transaction(session):
        session.add(SomeObject("bat", "lala"))

Session = sessionmaker(engine)

# create a Session and call method_a
with Session() as session:
    method_a(session)
```

为了与首选惯用模式进行比较，begin 块应位于最外层。这样就不需要单个函数或方法关注事务界定的细节：

```py
def method_a(session):
    method_b(session)

def method_b(session):
    session.add(SomeObject("bat", "lala"))

Session = sessionmaker(engine)

# create a Session and call method_a
with Session() as session:
    with session.begin():
        method_a(session)
```

**讨论**

已经证明这种模式在实际应用中令人困惑，并且最好是应用确保数据库操作的最高级别由单个 begin/commit 对完成。

## 2.0 迁移 - ORM 扩展和示例更改

### Dogpile 缓存示例和水平分片使用新的 Session API

当 `Query` 对象成为遗留对象时，先前依赖于 `Query` 对象子类化的这两个示例现在使用 `SessionEvents.do_orm_execute()` 钩子。请参阅 重新执行语句 一节进行示例。

### 烘焙查询扩展已被内置缓存系统取代

烘焙查询扩展已被内置缓存系统取代，并不再被 ORM 内部使用。

参见 SQL 编译缓存 获取有关新缓存系统的完整背景。

### Dogpile 缓存示例和水平分片使用新的 Session API

当 `Query` 对象成为遗留对象时，先前依赖于 `Query` 对象子类化的这两个示例现在使用 `SessionEvents.do_orm_execute()` 钩子。请参阅 重新执行语句 一节进行示例。

### 烘焙查询扩展已被内置缓存系统取代

烘焙查询扩展已被内置缓存系统取代，不再被 ORM 内部使用。

有关新缓存系统的完整背景，请参阅 SQL 编译缓存。

## 异步 IO 支持

SQLAlchemy 1.4 版本为核心和 ORM 都提供了 asyncio 支持。新的 API 专门使用上述的“未来”模式。请参阅核心和 ORM 的异步 IO 支持获取背景信息。
