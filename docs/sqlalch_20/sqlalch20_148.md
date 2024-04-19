# SQLAlchemy 0.7 中的新功能？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_07.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_07.html)

关于本文档

本文描述了 SQLAlchemy 版本 0.6（最后发布于 2012 年 5 月 5 日）和 SQLAlchemy 版本 0.7（截至 2012 年 10 月正在进行维护发布）之间的更改。

文档日期：2011 年 7 月 27 日

## 介绍

本指南介绍了 SQLAlchemy 版本 0.7 中的新功能，还记录了影响用户将其应用程序从 SQLAlchemy 0.6 系列迁移到 0.7 的更改。

尽可能地，更改是以不破坏为 0.6 构建的应用程序的兼容性的方式进行的。必然不向后兼容的更改非常少，除了可变属性默认值的更改之外，应该只影响极小部分应用程序 - 许多更改涉及非公共 API 和一些用户可能一直在尝试使用的未记录的黑客。

还有第二个更小的非向后兼容更改类别也有文档记录。这类更改涉及那些至少自 0.5 版本以来已被弃用并自弃用以来一直引发警告的功能和行为。这些更改只会影响仍在使用 0.4 或早期 0.5 风格 API 的应用程序。随着项目的成熟，我们在 0.x 级别发布中有越来越少这类更改，这是由于我们的 API 具有越来越少的功能，这些功能不太适合它们原本要解决的用例。

一系列现有功能在 SQLAlchemy 0.7 中已被取代。术语“取代”和“弃用”之间没有太大区别，只是前者更弱地暗示了旧功能可能会被移除。在 0.7 中，像`synonym`和`comparable_property`这样的功能，以及所有的`Extension`和其他事件类都已被取代。但这些“被取代”的功能已被重新实现，使得它们的实现大部分存在于核心 ORM 代码之外，因此它们的持续“挂在那里”并不影响 SQLAlchemy 进一步简化和完善其内部结构的能力，我们预计它们将在可预见的未来保留在 API 中。

## 新功能

### 新事件系统

SQLAlchemy 早期使用了`MapperExtension`类，该类提供了对映射器持久性周期的钩子。随着 SQLAlchemy 迅速变得更加组件化，将映射器推入更专注的配置角色，许多更多的“扩展”，“监听器”和“代理”类出现，以以一种临时方式解决各种活动拦截用例。部分原因是由活动的分歧驱动的；`ConnectionProxy`对象希望提供一个重写语句和参数的系统；`AttributeExtension`提供了一个替换传入值的系统，而`DDL`对象具有可以根据方言敏感的可调用函数进行切换的事件。

0.7 版本重新实现了几乎所有这些插件点，采用了一种新的、统一的方法，保留了不同系统的所有功能，提供了更多的灵活性和更少的样板代码，性能更好，并且消除了需要为每个事件子系统学习根本不同的 API 的必要性。之前存在的类 `MapperExtension`、`SessionExtension`、`AttributeExtension`、`ConnectionProxy`、`PoolListener` 以及 `DDLElement.execute_at` 方法已被弃用，现在根据新系统实现 - 这些 API 仍然完全可用，并预计将在可预见的未来保持不变。

新方法使用命名事件和用户定义的可调用对象将活动与事件关联起来。API 的外观和感觉受到了 JQuery、Blinker 和 Hibernate 等多样化来源的驱动，并且在与数十位用户进行的 Twitter 会议期间进行了多次修改，这似乎比邮件列表对此类问题的回应率要高得多。

它还具有一个开放式的目标规范系统，允许将事件与 API 类关联，例如所有 `Session` 或 `Engine` 对象，以及与 API 类的特定实例关联，例如特定的 `Pool` 或 `Mapper`，以及与用户定义的类（映射的类）或特定子类的实例的特定属性等相关对象。各个监听器子系统可以对传入的用户定义监听器函数应用包装器，修改它们的调用方式 - 映射器事件可以接收被操作对象的实例，或者其底层的 `InstanceState` 对象。属性事件可以选择是否有责任返回一个新值。

几个系统现在基于新的事件 API 进行构建，包括新的“可变属性” API 以及复合属性。对事件的更大强调还导致了一些新事件的引入，包括属性过期和刷新操作，pickle 加载/转储操作，完成的映射器构建操作。

另请参阅

事件

[#1902](https://www.sqlalchemy.org/trac/ticket/1902)

### Hybrid Attributes，实现/取代了 synonym()、comparable_property()

“派生属性”示例现在已成为官方扩展。`synonym()` 的典型用例是为映射列提供描述符访问；`comparable_property()` 的用例是能够从任何描述符返回 `PropComparator`。实际上，“派生”的方法更易于使用，更具可扩展性，用几十行纯 Python 实现，几乎不需要导入，甚至不需要 ORM 核心知道它。该功能现在被称为“Hybrid Attributes”扩展。

`synonym()`和`comparable_property()`仍然是 ORM 的一部分，尽管它们的实现已经移出，建立在类似于混合扩展的方法上，因此核心 ORM 映射器/查询/属性模块在其他方面并不真正意识到它们。

另请参见

混合属性

[#1903](https://www.sqlalchemy.org/trac/ticket/1903)

### 速度增强

与所有主要 SQLA 版本一样，通过内部进行广泛遍历以减少开销和调用次数，进一步减少了常见情况下所需的工作量。此版本的亮点包括：

+   刷新过程现在将 INSERT 语句捆绑成批次提供给`cursor.executemany()`，对于主键已经存在的行。特别是这通常适用于连接表继承配置中的“子”表，这意味着对于大量连接表对象的大量插入，可以将对`cursor.execute`的调用次数减半，从而允许本地 DBAPI 优化为那些传递给`cursor.executemany()`的语句（如重用准备好的语句）。

+   当访问已加载的相关对象的多对一引用时调用的代码路径已经大大简化。直接检查标识映射，无需首先生成新的`Query`对象，这在上下文中访问成千上万个内存中的多对一时是昂贵的。对于大多数延迟属性加载，也不再使用每次调用构造的“加载器”对象。

+   重新编写组合使得在映射器内部访问刷新时映射属性的代码路径更短。

+   新的内联属性访问函数取代了以前在“保存-更新”和其他级联操作需要在属性关联的所有数据成员范围内级联时使用“历史”时的用法。这减少了为这个速度关键操作生成新的`History`对象的开销。

+   `ExecutionContext`的内部，即对语句执行的对象，已经被内联和简化。

+   为每个语句执行生成的类型的`bind_processor()`和`result_processor()`可调用现在被缓存（小心处理，以避免临时类型和方言的内存泄漏）为该类型的生命周期，进一步减少每个语句的调用开销。

+   特定语句的`Compiled`实例的“绑定处理器”集合也被缓存在`Compiled`对象上，进一步利用刷新过程使用的“编译缓存”来重用相同的编译形式的 INSERT、UPDATE、DELETE 语句。

包括一个示例基准脚本在内的减少调用次数的演示可在[`techspot.zzzeek.org/2010/12/12/a-tale-of-three`](https://techspot.zzzeek.org/2010/12/12/a-tale-of-three)- profiles/中找到。

### 重写组合

“复合”特性已被重写，与`synonym()`和`comparable_property()`一样，使用了基于描述符和事件的轻量级实现，而不是构建到 ORM 内部。这使得从映射器/工作单元内部删除了一些延迟，并简化了复合的工作原理。复合属性现在不再隐藏其建立在其上的基础列，这些列现在保持为常规属性。复合对象还可以充当`relationship()`以及`Column()`属性的代理。

复合的主要向后不兼容变更是，它们不再使用`mutable=True`系统来检测原地突变。请使用[Mutation Tracking](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展来建立对现有复合用法的原位更改事件。

另请参阅

复合列类型

突变追踪

[#2008](https://www.sqlalchemy.org/trac/ticket/2008) [#2024](https://www.sqlalchemy.org/trac/ticket/2024)

### 更简洁的查询.join(target, onclause)形式

向具有显式 onclause 的目标发出`query.join()`的默认方法现在是：

```py
query.join(SomeClass, SomeClass.id == ParentClass.some_id)
```

在 0.6 版本中，此用法被认为是错误的，因为`join()`接受多个参数，对应于多个 JOIN 子句 - 两个参数形式需要在元组中以消除单参数和双参数连接目标之间的歧义。在 0.6 的中间，我们添加了对此特定调用样式的检测和错误消息，因为它是如此常见。在 0.7 中，由于我们无论如何都在检测确切的模式，并且由于为了没有理由而必须键入元组而极端烦人，因此非元组方法现在成为“正常”方法。这种“多个 JOIN”用例与单个 JOIN 用例相比极为罕见，而且这些天多次连接更清楚地表示为多次调用`join()`。

元组形式将保留以确保向后兼容性。

请注意，所有其他形式的`query.join()`保持不变：

```py
query.join(MyClass.somerelation)
query.join("somerelation")
query.join(MyTarget)
# ... etc
```

[使用联接查询](https://www.sqlalchemy.org/docs/07/orm/tutorial.html#querying-with-joins)

[#1923](https://www.sqlalchemy.org/trac/ticket/1923)

### 突变事件扩展，取代了“mutable=True”

一个新的扩展，突变追踪，提供了一种机制，通过该机制，用户定义的数据类型可以向拥有的父对象提供更改事件。该扩展包括了一种用于标量数据库值的方法，例如由`PickleType`、`postgresql.ARRAY`或其他自定义`MutableType`类管理的值，以及一种用于 ORM“复合”对象的方法，这些对象使用`composite()`进行配置。

另请参阅

突变追踪

### NULLS FIRST / NULLS LAST 运算符

这些作为 `asc()` 和 `desc()` 操作符的扩展实现，称为 `nullsfirst()` 和 `nullslast()`。

另见

`nullsfirst()`

`nullslast()`

[#723](https://www.sqlalchemy.org/trac/ticket/723)

### select.distinct()、query.distinct() 接受 *args 用于 PostgreSQL 的 DISTINCT ON

通过向 `select()` 的 `distinct` 关键字参数传递表达式列表，现在当使用 PostgreSQL 后端时，`select()` 和 `Query` 的 `distinct()` 方法接受位置参数，这些参数将被渲染为 DISTINCT ON。

[distinct()](https://www.sqlalchemy.org/docs/07/core/expression_api.html#sqlalchemy.sql.expression.Select.distinct)

[Query.distinct()](https://www.sqlalchemy.org/docs/07/orm/query.html#sqlalchemy.orm.query.Query.distinct)

[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

### `Index()` 可以嵌入到 `Table`、`__table_args__` 内。

Index() 构造可以与 Table 定义一起内联创建，使用字符串作为列名，作为在 Table 外部创建索引的替代方法。即：

```py
Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50), nullable=False),
    Index("idx_name", "name"),
)
```

这里的主要理由是为了声明性 `__table_args__` 的利益，特别是与混入类一起使用时：

```py
class HasNameMixin(object):
    name = Column("name", String(50), nullable=False)

    @declared_attr
    def __table_args__(cls):
        return (Index("name"), {})

class User(HasNameMixin, Base):
    __tablename__ = "user"
    id = Column("id", Integer, primary_key=True)
```

[Indexes](https://www.sqlalchemy.org/docs/07/core/schema.html#indexes)

### 窗口函数 SQL 结构

“窗口函数”在语句生成结果集时向其提供信息。这允许针对诸如“行号”、“排名”等各种条件进行判断。已知至少由 PostgreSQL、SQL Server 和 Oracle 支持。可能还有其他数据库也支持。

最好的窗口函数介绍在 PostgreSQL 网站上，自从版本 8.4 开始就支持窗口函数：

[`www.postgresql.org/docs/current/static/tutorial-window.html`](https://www.postgresql.org/docs/current/static/tutorial-window.html)

SQLAlchemy 提供了一个简单的构造，通常通过现有函数子句调用，使用 `over()` 方法，该方法接受 `order_by` 和 `partition_by` 关键字参数。下面我们复制了 PG 教程中的第一个示例：

```py
from sqlalchemy.sql import table, column, select, func

empsalary = table("empsalary", column("depname"), column("empno"), column("salary"))

s = select(
    [
        empsalary,
        func.avg(empsalary.c.salary)
        .over(partition_by=empsalary.c.depname)
        .label("avg"),
    ]
)

print(s)
```

SQL:

```py
SELECT  empsalary.depname,  empsalary.empno,  empsalary.salary,
avg(empsalary.salary)  OVER  (PARTITION  BY  empsalary.depname)  AS  avg
FROM  empsalary
```

[sqlalchemy.sql.expression.over](https://www.sqlalchemy.org/docs/07/core/expression_api.html#sqlalchemy.sql.expression.over)

[#1844](https://www.sqlalchemy.org/trac/ticket/1844)

### Connection 上的 execution_options() 接受 “isolation_level” 参数

这为单个 `Connection` 设置了事务隔离级别，直到该 `Connection` 关闭并且其底层的 DBAPI 资源返回到连接池，此时隔离级别将重置回默认值。默认的隔离级别是通过 `create_engine()` 的 `isolation_level` 参数设置的。

目前只有 PostgreSQL 和 SQLite 后端支持事务隔离。

[execution_options()](https://www.sqlalchemy.org/docs/07/core/connections.html#sqlalchemy.engine.base.Connection.execution_options)

[#2001](https://www.sqlalchemy.org/trac/ticket/2001)

### `TypeDecorator`与整数主键列一起工作

可以使用扩展`Integer`行为的`TypeDecorator`与主键列一起使用。`Column`的“autoincrement”功能现在将识别底层数据库列仍然是整数，以便继续使 lastrowid 机制正常工作。`TypeDecorator`本身的结果值处理器将应用于新生成的主键，包括通过 DBAPI `cursor.lastrowid` 访问器接收的主键。

[#2005](https://www.sqlalchemy.org/trac/ticket/2005) [#2006](https://www.sqlalchemy.org/trac/ticket/2006)

### `TypeDecorator`存在于“sqlalchemy”导入空间中

不再需要从`sqlalchemy.types`导入，现在在`sqlalchemy`中有镜像。

### 新方言

已添加方言：

+   用于 Drizzle 数据库的 MySQLdb 驱动程序：

    [Drizzle](https://www.sqlalchemy.org/docs/07/dialects/drizzle.html)

+   支持 pymysql DBAPI：

    [pymsql Notes](https://www.sqlalchemy.org/docs/07/dialects/mysql.html#module-sqlalchemy.dialects.mysql.pymysql)

+   psycopg2 现在支持 Python 3

## 行为变化（向后兼容）

### 默认情况下构建 C 扩展

这是从 0.7b4 开始的。如果检测到 cPython 2.xx，则会构建扩展。如果构建失败，例如在 Windows 安装中，会捕获该条件并继续非 C 安装。如果使用 Python 3 或 PyPy，则不会构建 C 扩展。

### 简化了 Query.count()，几乎总是有效

`Query.count()`中非常古老的猜测现在已经现代化，使用`.from_self()`。也就是说，`query.count()`现在等效于：

```py
query.from_self(func.count(literal_column("1"))).scalar()
```

以前，内部逻辑尝试重写查询本身的列子句，并在检测到“子查询”条件时，例如可能在其中包含聚合的基于列的查询，或者具有 DISTINCT 的查询时，会经历一个复杂的过程来重写列子句。这种逻辑在复杂条件下失败，特别是涉及联接表继承的条件，并且长期以来已经被更全面的`.from_self()`调用所取代。

`query.count()`现在始终生成以下形式的 SQL：

```py
SELECT  count(1)  AS  count_1  FROM  (
  SELECT  user.id  AS  user_id,  user.name  AS  user_name  from  user
)  AS  anon_1
```

也就是说，原始查询完全保留在子查询中，不再猜测如何应用 count。

[#2093](https://www.sqlalchemy.org/trac/ticket/2093)

#### 发出非子查询形式的 count()

MySQL 用户已经报告说 MyISAM 引擎不出所料地完全崩溃了。请注意，对于优化不能处理简单子查询的数据库的简单`count()`，应该使用`func.count()`：

```py
from sqlalchemy import func

session.query(func.count(MyClass.id)).scalar()
```

或者对于`count(*)`：

```py
from sqlalchemy import func, literal_column

session.query(func.count(literal_column("*"))).select_from(MyClass).scalar()
```

### LIMIT/OFFSET 子句现在使用绑定参数

LIMIT 和 OFFSET 子句，或其后端等效项（即 TOP、ROW NUMBER OVER 等），对于所有支持的后端（除了 Sybase），使用绑定参数进行实际值，这允许更好的查询优化器性能，因为具有不同 LIMIT/OFFSET 的多个语句的文本字符串现在是相同的。

[#805](https://www.sqlalchemy.org/trac/ticket/805)

### 日志增强

Vinay Sajip 提供了一个补丁，使我们的日志系统中不再需要嵌入在引擎和池日志语句中的“十六进制字符串”以使 `echo` 标志能够正常工作。使用过滤日志对象的新系统使我们能够保持当前行为，即 `echo` 仅适用于各个引擎，而无需为这些引擎添加额外的标识字符串。

[#1926](https://www.sqlalchemy.org/trac/ticket/1926)

### 简化的 polymorphic_on 赋值

当在继承场景中使用时，`polymorphic_on` 列映射属性的填充现在发生在对象构造时，即其 `__init__` 方法被调用时，使用 init 事件。然后，该属性的行为与任何其他列映射属性相同。以前，特殊逻辑会在刷新期间触发以填充此列，这会阻止任何用户代码修改其行为。新方法在三个方面改进了这一点：1. 多态标识现在在对象构造时立即存在；2. 多态标识可以被用户代码更改，而不会与任何其他列映射属性的行为有任何区别；3. 刷新期间映射器的内部简化，不再需要对此列进行特殊检查。

[#1895](https://www.sqlalchemy.org/trac/ticket/1895)

### 在多个路径（即“all()”）上进行 contains_eager() 链

`contains_eager()` 修改器现在会把自己链接到一个更长的路径上，而不需要释放独立的`contains_eager()`调用。而不是：

```py
session.query(A).options(contains_eager(A.b), contains_eager(A.b, B.c))
```

你可以说：

```py
session.query(A).options(contains_eager(A.b, B.c))
```

[#2032](https://www.sqlalchemy.org/trac/ticket/2032)

### 允许刷新没有父级的孤立对象

我们一直有一个长期存在的行为，即在刷新期间检查所谓的“孤立对象”，即与指定“delete-orphan”级联的 `relationship()` 关联的对象，已经被新增到会话中进行插入，并且没有建立父关系。多年前添加了此检查以适应一些测试用例，这些测试用例测试了孤立行为的一致性。在现代 SQLA 中，此检查在 Python 端不再需要。通过使外键引用对象的父行 NOT NULL，数据库会以与 SQLA 允许大多数其他操作相同的方式建立数据一致性。如果对象的父外键可为空，则可以插入行。当对象与特定父对象一起持久化，然后与该父对象解除关联时，会触发“孤立”行为，导致为其发出 DELETE 语句。

[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

### 在刷新时生成警告，当集合成员、标量引用不在刷新中时

当父对象上标记为 “脏” 的加载的 `relationship()` 引用的相关对象不在当前 `Session` 中时，现在会发出警告。

当对象添加到 `Session` 中或首次与父对象关联时，`save-update` 级联生效，因此对象及其相关内容通常都存在于同一个 `Session` 中。然而，如果对于特定的 `relationship()` 禁用了 `save-update` 级联，则不会发生这种行为，刷新过程也不会尝试纠正它，而是保持与配置的级联行为一致。以前，在刷新时检测到这样的对象时，它们会被静默跳过。新的行为是发出警告，目的是提醒可能是意外行为来源的情况。

[#1973](https://www.sqlalchemy.org/trac/ticket/1973)

### 安装程序不再安装 Nose 插件

自从我们转向 nose 以来，我们使用了一个通过 setuptools 安装的插件，这样 `nosetests` 脚本会自动运行 SQLA 的插件代码，这对于我们的测试来说是必要的，以便具有完整的环境。在 0.6 中间，我们意识到这里的导入模式意味着 Nose 的 “coverage” 插件会中断，因为 “coverage” 要求在导入要覆盖的任何模块之前启动它；因此，在 0.6 中间，我们通过添加一个单独的 `sqlalchemy-nose` 包来解决这个问题，使情况变得更糟。

在 0.7 中，我们放弃了尝试让 `nosetests` 自动工作，因为 SQLAlchemy 模块会为所有使用 `nosetests` 的用法产生大量的 nose 配置选项，不仅仅是 SQLAlchemy 单元测试本身，而且额外的 `sqlalchemy-nose` 安装是一个更糟糕的想法，在 Python 环境中产生了一个额外的包。在 0.7 中，`sqla_nose.py` 脚本现在是使用 nose 运行测试的唯一方法。

[#1949](https://www.sqlalchemy.org/trac/ticket/1949)

### 非 `Table` 派生的构造可以被映射

一个根本不针对任何 `Table` 的构造，比如一个函数，可以被映射。

```py
from sqlalchemy import select, func
from sqlalchemy.orm import mapper

class Subset(object):
    pass

selectable = select(["x", "y", "z"]).select_from(func.some_db_function()).alias()
mapper(Subset, selectable, primary_key=[selectable.c.x])
```

[#1876](https://www.sqlalchemy.org/trac/ticket/1876)

### aliased() 接受 `FromClause` 元素

这是一个方便的辅助程序，当传递一个普通的 `FromClause`，比如一个 `select`、`Table` 或 `join` 到 `orm.aliased()` 构造时，它会通过到达该 from 构造的 `.alias()` 方法，而不是构造一个 ORM 级别的 `AliasedClass`。

[#2018](https://www.sqlalchemy.org/trac/ticket/2018)

### Session.connection()，Session.execute() 接受 ‘bind’

这是为了允许 execute/connection 操作明确参与引擎的开放事务。它还允许自定义的 `Session` 子类实现自己的 `get_bind()` 方法和参数，以便在 `execute()` 和 `connection()` 方法中同样使用这些自定义参数。

[Session.connection](https://www.sqlalchemy.org/docs/07/orm/session.html#sqlalchemy.orm.session.Session.connection) [Session.execute](https://www.sqlalchemy.org/docs/07/orm/session.html#sqlalchemy.orm.session.Session.execute)

[#1996](https://www.sqlalchemy.org/trac/ticket/1996)

### 独立的绑定参数在列子句中自动标记。

存在于 select 的“columns clause”中的绑定参数现在像其他“匿名”子句一样自动标记，这样在获取行时它们的“类型”就有意义，就像结果行处理器一样。

### SQLite - 相对文件路径通过 os.path.abspath() 进行标准化

这样，更改当前目录的脚本将继续定位到后续建立的 SQLite 连接的相同位置。

[#2036](https://www.sqlalchemy.org/trac/ticket/2036)

### MS-SQL - `String`/`Unicode`/`VARCHAR`/`NVARCHAR`/`VARBINARY` 在未指定长度时发出“max”

在 MS-SQL 后端，String/Unicode 类型及其对应的 VARCHAR/NVARCHAR，以及 VARBINARY ([#1833](https://www.sqlalchemy.org/trac/ticket/1833)) 在未指定长度时发出“max”作为长度。这使其更兼容于 PostgreSQL 的 VARCHAR 类型，当未指定长度时同样是无界限的。SQL Server 在未指定长度时默认这些类型的长度为‘1’。

## 行为变更（不兼容后向）

再次注意，除了默认的可变性更改外，大多数这些更改都是*极其微小*的，不会影响大多数用户。

### `PickleType` 和 ARRAY 的可变性默认关闭

此更改涉及 ORM 在映射具有 `PickleType` 或 `postgresql.ARRAY` 数据类型的列时的默认行为。`mutable` 标志现在默认设置为 `False`。如果现有应用程序使用这些类型并依赖于就地变异的检测，则必须使用 `mutable=True` 构造类型对象以恢复 0.6 版本的行为：

```py
Table(
    "mytable",
    metadata,
    # ....
    Column("pickled_data", PickleType(mutable=True)),
)
```

`mutable=True` 标志正在逐步淘汰，取而代之的是新的[Mutation Tracking](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html) 扩展。该扩展提供了一种机制，通过该机制，用户定义的数据类型可以向拥有的父级或父级提供更改事件。

以前使用`mutable=True`的方法不提供更改事件 - 相反，ORM 必须在每次调用`flush()`时扫描会话中存在的所有可变值，并将它们与它们的原始值进行比较，这是一个非常耗时的事件。这是 SQLAlchemy 非常早期的遗留问题，当时`flush()`不是自动的，历史跟踪系统也不像现在这样复杂。

现有应用程序使用`PickleType`，`postgresql.ARRAY`或其他`MutableType`子类，并需要原地变异检测的应用程序应该迁移到新的变异跟踪系统，因为`mutable=True`可能会在未来被弃用。

[#1980](https://www.sqlalchemy.org/trac/ticket/1980)

### `composite()`的可变性检测需要变异跟踪扩展

所谓的“复合”映射属性，使用在[复合列类型](https://www.sqlalchemy.org/docs/07/orm/mapper_config.html#composite-column-types)中描述的技术配置的那些，已经重新实现，以使 ORM 内部不再意识到它们（导致关键部分中的代码路径更短更高效）。虽然复合类型通常应被视为不可变值对象，但从未强制执行。对于使用具有可变性的复合的应用程序，[变异跟踪](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展提供了一个基类，该基类建立了一个机制，使用户定义的复合类型能够向每个对象的拥有父对象或父对象发送更改事件消息。

使用复合类型并依赖于这些对象的原地变异检测的应用程序应该迁移到“变异跟踪”扩展，或者更改复合类型的使用，以便不再需要原地更改（即将它们视为不可变值对象）。

### SQLite - SQLite 方言现在对基于文件的数据库使用`NullPool`

这个改变是**99.999%向后兼容**，除非您在连接池连接之间使用临时表。

基于文件的 SQLite 连接速度非常快，使用`NullPool`意味着每次调用`Engine.connect`都会创建一个新的 pysqlite 连接。

以前，使用`SingletonThreadPool`，这意味着在一个线程中对某个引擎的所有连接将是相同的连接。新方法更直观，特别是在使用多个连接时。

当使用`:memory:`数据库时，`SingletonThreadPool`仍然是默认引擎。

请注意，这个改变**破坏了跨会话提交使用的临时表**，这是由于 SQLite 处理临时表的方式。如果需要超出一个连接池连接范围的临时表，请参阅[`www.sqlalchemy.org/docs/dialects/sqlite.html#using`](https://www.sqlalchemy.org/docs/dialects/sqlite.html#using)- temporary-tables-with-sqlite 中的说明。

[#1921](https://www.sqlalchemy.org/trac/ticket/1921)

### `Session.merge()`为具有版本控制的映射器检查版本 id

`Session.merge()`将会检查传入状态的版本 id 与数据库中的版本 id 是否匹配，假设映射使用了版本 id，并且传入状态已经分配了一个版本 id，如果它们不匹配，则会引发`StaleDataError`。这是正确的行为，因为如果传入状态包含一个过期的版本 id，则应该假设该状态已过期。

如果将数据合并到一个有版本控制的状态中，则版本 id 属性可以不定义，并且不会进行版本检查。

通过检查 Hibernate 的做法已确认了这一点 - `merge()`和版本控制功能最初都是从 Hibernate 适配而来的。

[#2027](https://www.sqlalchemy.org/trac/ticket/2027)

### 查询中改进的元组标签名称

这种改进可能对依赖于旧行为的应用程序稍微具有向后不兼容性。

给定两个映射类`Foo`和`Bar`，每个类都有一个名为`spam`的列：

```py
qa = session.query(Foo.spam)
qb = session.query(Bar.spam)

qu = qa.union(qb)
```

由`qu`产生的单个列的名称将是`spam`。之前由于`union`组合的方式，它可能是`foo_spam`之类的东西，这与非联合查询的情况下的`spam`名称不一致。

[#1942](https://www.sqlalchemy.org/trac/ticket/1942)

### 映射的列属性首先引用最具体的列

这是一个行为变更，涉及到当一个映射的列属性引用多个列时，特别是在处理一个具有与超类相同名称的属性的联接表子类的属性时。

使用声明式的情况是这样的：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)

class Child(Parent):
    __tablename__ = "child"
    id = Column(Integer, ForeignKey("parent.id"), primary_key=True)
```

在上面的例子中，属性`Child.id`同时引用了`child.id`列和`parent.id`列 - 这是由于属性的名称。如果在类上以不同的方式命名它，比如`Child.child_id`，那么它将明确地映射到`child.id`，而`Child.id`将是与`Parent.id`相同的属性。

当`id`属性被设置为引用`parent.id`和`child.id`时，它们会被存储在一个有序列表中。例如`Child.id`这样的表达式在渲染时只会引用其中*一个*列。直到 0.6 版本，这个列会是`parent.id`。在 0.7 版本中，它是不那么令人惊讶的`child.id`。

这种行为的传统与 ORM 的行为和限制相关，这些限制实际上已经不适用了；一切所需的只是颠倒顺序。

这种方法的一个主要优势是现在更容易构造引用本地列的`primaryjoin`表达式：

```py
class Child(Parent):
    __tablename__ = "child"
    id = Column(Integer, ForeignKey("parent.id"), primary_key=True)
    some_related = relationship(
        "SomeRelated", primaryjoin="Child.id==SomeRelated.child_id"
    )

class SomeRelated(Base):
    __tablename__ = "some_related"
    id = Column(Integer, primary_key=True)
    child_id = Column(Integer, ForeignKey("child.id"))
```

在 0.7 版本之前，`Child.id`表达式会引用`Parent.id`，并且需要将`child.id`映射到一个不同的属性上。

这也意味着像这样的查询的行为发生了变化：

```py
session.query(Parent).filter(Child.id > 7)
```

在 0.6 版本中，这将呈现为：

```py
SELECT  parent.id  AS  parent_id
FROM  parent
WHERE  parent.id  >  :id_1
```

在 0.7 版本中，您会得到：

```py
SELECT  parent.id  AS  parent_id
FROM  parent,  child
WHERE  child.id  >  :id_1
```

你会注意到这是一个笛卡尔积 - 这种行为现在等同于`Child`中的任何其他局部属性。`with_polymorphic()` 方法或类似的显式连接基础 `Table` 对象的策略，用于对所有带有`Child`条件的 `Parent` 对象进行查询，方式与 0.5 和 0.6 相同：

```py
print(s.query(Parent).with_polymorphic([Child]).filter(Child.id > 7))
```

在 0.6 和 0.7 版本都是这样呈现的：

```py
SELECT  parent.id  AS  parent_id,  child.id  AS  child_id
FROM  parent  LEFT  OUTER  JOIN  child  ON  parent.id  =  child.id
WHERE  child.id  >  :id_1
```

这种更改的另一个效果是，跨两个表的连接继承加载将从子表的值填充，而不是从父表的值填充。一个不寻常的情况是，使用`with_polymorphic="*"`对“Parent”进行查询会对“parent”发出查询，并且左外连接到“child”。行位于“Parent”中，看到多态标识对应于“Child”，但是假设“child”中的实际行已被*删除*。由于这种损坏，行会带有所有对应于“child”的列设置为 NULL 的值 - 这是现在被填充的值，而不是父表中的值。

[#1892](https://www.sqlalchemy.org/trac/ticket/1892)

### 将两个或更多同名列映射到连接时需要明确声明

这与之前的变更[#1892](https://www.sqlalchemy.org/trac/ticket/1892)有些相关。在映射到连接时，同名列必须显式地链接到映射属性，即如[将类映射到多个表](http://www.sqlalchemy.org/docs/07/orm/mapper_config.html#mapping-a-class-against-multiple-tables)中描述的那样。

给定两个表 `foo` 和 `bar`，每个表都有一个主键列 `id`，现在会产生一个错误：

```py
foobar = foo.join(bar, foo.c.id == bar.c.foo_id)
mapper(FooBar, foobar)
```

这是因为 `mapper()` 拒绝猜测 `FooBar.id` 的主要表示列是 `foo.c.id` 还是 `bar.c.id`？属性必须是明确的：

```py
foobar = foo.join(bar, foo.c.id == bar.c.foo_id)
mapper(FooBar, foobar, properties={"id": [foo.c.id, bar.c.id]})
```

[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

### 映射器要求多态性的列在映射的可选择项中存在

在 0.6 中是一个警告，现在在 0.7 中是一个错误。给定用于 `polymorphic_on` 的列必须在映射的可选择项中。这是为了防止一些偶发的用户错误，例如：

```py
mapper(SomeClass, sometable, polymorphic_on=some_lookup_table.c.id)
```

在这种情况下，`polymorphic_on` 需要在`sometable`列上，也许是`sometable.c.some_lookup_id`。有时还会出现一些“多态联合”场景，类似的错误有时也会发生。

这样的配置错误一直都是“错误”的，并且上述映射不按照指定的方式工作 - 列将被忽略。然而，在极少数情况下，这可能是向后不兼容的，因为应用程序可能一直在无意中依赖于这种行为。

[#1875](https://www.sqlalchemy.org/trac/ticket/1875)

### `DDL()` 构造现在会转义百分号

以前，在 `DDL()` 字符串中的百分号必须进行转义，即 `%%` 取决于 DBAPI，对于那些接受 `pyformat` 或 `format` 绑定的 DBAPI（例如 psycopg2，mysql-python），这与自动执行此操作的 `text()` 构造不一致。现在，`DDL()` 与 `text()` 一样进行相同的转义。

[#1897](https://www.sqlalchemy.org/trac/ticket/1897)

### `Table.c` / `MetaData.tables` 稍微精炼了一下，不允许直接变异。

另一个领域，一些用户在进行某种方式的尝试时实际上并不按预期工作，但仍然留下了极小的机会，即某些应用程序依赖于这种行为，`.c` 属性在 `Table` 上返回的构造和 `MetaData` 上的 `.tables` 属性明确是不可变的。构造的“可变”版本现在是私有的。向 `.c` 添加列涉及使用 `Table` 的 `append_column()` 方法，这确保了事物以适当的方式与父 `Table` 关联；同样，`MetaData.tables` 与存储在此字典中的 `Table` 对象有一个合同，以及一些新的簿记，跟踪所有模式名称的 `set()`，只有通过使用公共 `Table` 构造函数以及 `Table.tometadata()` 才能满足。

当然，`ColumnCollection` 和 `dict` 集合可能会在某一天实现对其所有变异方法的事件，以便在直接变异集合时发生适当的簿记，但在有人有动力实现所有这些以及数十个新单元测试之前，缩小这些集合的变异路径将确保没有应用程序试图依赖当前不支持的用法。

[#1893](https://www.sqlalchemy.org/trac/ticket/1893) [#1917](https://www.sqlalchemy.org/trac/ticket/1917)

### `server_default` 对所有 `inserted_primary_key` 值始终返回 `None`。

当 `server_default` 出现在整数主键列上时确立了一致性。SQLA 不会预取这些值，也不会在 `cursor.lastrowid`（DBAPI）中返回它们。确保所有后端在 `result.inserted_primary_key` 中一致地返回 `None` - 一些后端可能之前返回过一个值。在主键列上使用 `server_default` 是极不寻常的。如果使用特殊函数或 SQL 表达式生成主键默认值，则应将其确定为 Python 端的“default” 而不是 `server_default`。

对于这种情况的反射，具有 `server_default` 的 int PK 列的反射将“autoincrement” 标志设置为 `False`，除非是 PG SERIAL 列，我们检测到一个序列默认值。

[#2020](https://www.sqlalchemy.org/trac/ticket/2020) [#2021](https://www.sqlalchemy.org/trac/ticket/2021)

### `sys.modules` 中的 `sqlalchemy.exceptions` 别名被移除。

几年来，我们一直将字符串 `sqlalchemy.exceptions` 添加到 `sys.modules` 中，以便像“`import sqlalchemy.exceptions`”这样的语句能够正常工作。核心异常模块的名称现在已经是 `exc` 很长时间了，因此建议导入此模块的方式是：

```py
from sqlalchemy import exc
```

对于可能已经说过 `from sqlalchemy import exceptions` 的应用程序，`exceptions` 名称仍然存在于“`sqlalchemy`”中，但他们也应该开始使用 `exc` 名称。

### 查询时间配方更改

虽然不是 SQLAlchemy 本身的一部分，但值得一提的是，将 `ConnectionProxy` 重构为新的事件系统意味着不再适用于“Timing all Queries”配方。请调整查询计时器以使用 `before_cursor_execute()` 和 `after_cursor_execute()` 事件，在更新后的配方 UsageRecipes/Profiling 中有示例。

## 弃用的 API

### 类型上的默认构造函数不会接受参数

核心类型模块中的简单类型如 `Integer`、`Date` 等不接受参数。从 0.7b4/0.7.0 开始，接受/忽略 catchall `\*args, \**kwargs` 的默认构造函数已经恢复，但会发出弃用警告。

如果在核心类型如 `Integer` 中使用参数，可能是你打算使用特定于方言的类型，比如 `sqlalchemy.dialects.mysql.INTEGER`，它接受一个“display_width”参数。

### compile_mappers() 重命名为 configure_mappers()，简化配置内部

这个系统从最初是一个小型的、实现在单个映射器本地的东西，命名不当，逐渐演变成了更像是一个全局“注册表”级别的功能，命名不当，因此我们通过将实现移出 `Mapper` 并将其重命名为 `configure_mappers()` 来修复这两个问题。当然，通常情况下应用程序不需要调用 `configure_mappers()`，因为这个过程是根据需要的，一旦通过属性或查询访问需要映射时就会发生。

[#1966](https://www.sqlalchemy.org/trac/ticket/1966)

### 核心监听器/代理被事件监听器取代

`PoolListener`、`ConnectionProxy`、`DDLElement.execute_at` 被分别替代为 `event.listen()`，使用 `PoolEvents`、`EngineEvents`、`DDLEvents` 分发目标。

### ORM 扩展被事件监听��取代

`MapperExtension`、`AttributeExtension`、`SessionExtension` 被分别替代为 `event.listen()`，使用 `MapperEvents`/`InstanceEvents`、`AttributeEvents`、`SessionEvents` 分发目标。

### 在 MySQL 中，将字符串发送到 `select()` 的 ‘distinct’ 应该通过前缀来完成

这个晦涩的特性允许在 MySQL 后端中使用这种模式：

```py
select([mytable], distinct="ALL", prefixes=["HIGH_PRIORITY"])
```

对于非标准或不寻常的前缀，应该使用 `prefixes` 关键字或 `prefix_with()` 方法：

```py
select([mytable]).prefix_with("HIGH_PRIORITY", "ALL")
```

### `useexisting` 被 `extend_existing` 和 `keep_existing` 取代

Table 上的 `useexisting` 标志已被新的一对标志 `keep_existing` 和 `extend_existing` 取代。`extend_existing` 等同于 `useexisting` - 返回现有的 Table，并添加额外的构造元素。使用 `keep_existing`，返回现有的 Table，但不添加额外的构造元素 - 这些元素仅在新创建 Table 时应用。

## 不兼容的后向 API 更改

### 传递给 `bindparam()` 的可调用对象不会被评估 - 影响 Beaker 示例

[#1950](https://www.sqlalchemy.org/trac/ticket/1950)

请注意，这影响了 Beaker 缓存示例，其中 `_params_from_query()` 函数的工作需要进行轻微调整。如果您正在使用 Beaker 示例中的代码，则应用此更改。

### types.type_map 现在是私有的，types._type_map

我们注意到一些用户在 `sqlalchemy.types` 内部利用这个字典作为将 Python 类型与 SQL 类型关联的快捷方式。我们无法保证这个字典的内容或格式，并且将 Python 类型一对一关联的业务有一些灰色地带，最好由各个应用程序自行决定，因此我们已经强调了这个属性。

[#1870](https://www.sqlalchemy.org/trac/ticket/1870)

### 将独立 `alias()` 函数的 `alias` 关键字参数重命名为 `name`

这样关键字参数 `name` 与所有 `FromClause` 对象上的 `alias()` 方法以及 `Query.subquery()` 上的 `name` 参数匹配。

只有使用独立的 `alias()` 函数，而不是方法绑定函数，并且使用显式关键字名称 `alias` 而不是位置上的别名名称的代码需要在这里进行修改。

### 非公共 `Pool` 方法已加下划线

所有 `Pool` 及其子类的所有不打算公开使用的方法都已改名为下划线。它们以前没有这样命名是一个错误。

现在下划线或已移除的池化方法：

`Pool.create_connection()` -> `Pool._create_connection()`

`Pool.do_get()` -> `Pool._do_get()`

`Pool.do_return_conn()` -> `Pool._do_return_conn()`

`Pool.do_return_invalid()` -> 已移除，未被使用

`Pool.return_conn()` -> `Pool._return_conn()`

`Pool.get()` -> `Pool._get()`, 公共 API 是 `Pool.connect()`

`SingletonThreadPool.cleanup()` -> `_cleanup()`

`SingletonThreadPool.dispose_local()` -> 已移除，使用 `conn.invalidate()`

[#1982](https://www.sqlalchemy.org/trac/ticket/1982)

## 以前弃用，现在已移除

### Query.join(), Query.outerjoin(), eagerload(), eagerload_all(), 其他不再允许将属性列表作为参数

从 0.5 开始，将属性或属性名称列表传递给 `Query.join`, `eagerload()` 等已被弃用：

```py
# old way, deprecated since 0.5
session.query(Houses).join([Houses.rooms, Room.closets])
session.query(Houses).options(eagerload_all([Houses.rooms, Room.closets]))
```

这些方法在 0.5 系列中都接受 *args：

```py
# current way, in place since 0.5
session.query(Houses).join(Houses.rooms, Room.closets)
session.query(Houses).options(eagerload_all(Houses.rooms, Room.closets))
```

### `ScopedSession.mapper` 已移除

这一功能提供了一个映射器扩展，将基于类的功能与特定的`ScopedSession`链接起来，特别是提供了这样的行为，即新对象实例将自动与该会话关联。 该功能被教程和框架过度使用，导致用户混淆，因为其隐式行为，并在 0.5.5 中被弃用。 复制其功能的技术在[wiki:UsageRecipes/SessionAwareMapper]中。

## 介绍

本指南介绍了 SQLAlchemy 版本 0.7 中的新功能，并记录了影响用户将其应用程序从 SQLAlchemy 0.6 系列迁移到 0.7 的更改。

尽可能地，更改是以不破坏为 0.6 构建的应用程序的兼容性的方式进行的。 必然不向后兼容的更改非常少，除了一个，即对可变属性默认值的更改，应该影响极小部分应用程序 - 许多更改涉及非公共 API 和未记录的一些用户可能一直在尝试使用的黑客技巧。

还有第二个更小的一类不向后兼容的更改也有文档记录。 这类更改涉及那些自 0.5 版本以来已被弃用并自弃用以来一直引发警告的功能和行为。 这些更改只会影响仍在使用 0.4 或早期 0.5 样式 API 的应用程序。 随着项目的成熟，我们在 0.x 级别发布中有越来越少这类更改，这是由于我们的 API 具有越来越少的功能，这些功能对于它们旨在解决的用例来说不太理想。 

在 SQLAlchemy 0.7 中，一系列现有功能已被取代。 “取代”和“弃用”之间没有太大区别，只是前者对旧功能的暗示要被删除的可能性较小。 在 0.7 中，像`synonym`和`comparable_property`以及所有`Extension`和其他事件类等功能已被取代。 但是这些“取代”功能已被重新实现，使得它们的实现主要存在于核心 ORM 代码之外，因此它们的持续“挂在”不会影响 SQLAlchemy 进一步简化和完善其内部的能力，并且我们预计它们将在可预见的未来保留在 API 中。

## 新功能

### 新事件系统

SQLAlchemy 早期开始使用 `MapperExtension` 类，该类提供了对映射器持久性周期的钩子。随着 SQLAlchemy 快速变得更加组件化，将映射器推入更专注的配置角色，许多更多的“扩展”、“监听器”和“代理”类出现，以以一种临时的方式解决各种活动拦截用例。其中一部分是由活动的分歧驱动的；`ConnectionProxy` 对象希望提供一个重写语句和参数的系统；`AttributeExtension` 提供了一个替换传入值的系统，而 `DDL` 对象具有可以切换到方言敏感可调用对象的事件。

0.7 重新实现了几乎所有这些插件点，采用了一种新的、统一的方法，保留了不同系统的所有功能，提供了更多的灵活性和更少的样板代码，性能更好，并消除了需要为每个事件子系统学习根本不同的 API 的必要性。现有的类 `MapperExtension`、`SessionExtension`、`AttributeExtension`、`ConnectionProxy`、`PoolListener` 以及 `DDLElement.execute_at` 方法已被弃用，现在根据新系统实现 - 这些 API 仍然完全可用，并预计将在可预见的未来保持不变。

新方法使用命名事件和用户定义的可调用对象将活动与事件关联起来。API 的外观和感觉受到了 JQuery、Blinker 和 Hibernate 等多样化来源的驱动，并且在与数十位用户进行的 Twitter 会议期间进行了多次修改，这似乎比邮件列表对这类问题的回应率要高得多。

它还具有一个开放式的目标规范系统，允许将事件与 API 类关联，例如所有 `Session` 或 `Engine` 对象，以及与 API 类的特定实例关联，例如特定的 `Pool` 或 `Mapper`，以及与用户定义的类相关的对象，例如映射的类，或者特定子类的实例上的某个属性。个别监听器子系统可以将包装器应用于传入的用户定义的监听器函数，从而修改它们的调用方式 - 映射器事件可以接收被操作对象的实例，或者其底层的 `InstanceState` 对象。属性事件可以选择是否要负责返回一个新值。

几个系统现在基于新的事件 API 构建，包括新的“可变属性” API 以及复合属性。对事件的更大强调还导致引入了一些新事件，包括属性过期和刷新操作，pickle 加载/转储操作，完成的映射器构造操作。

另请参见

事件

[#1902](https://www.sqlalchemy.org/trac/ticket/1902)

### 混合属性，实现/取代了 synonym()、comparable_property()

“派生属性”示例现在已经转变为官方扩展。`synonym()`的典型用例是为映射列提供描述符访问；`comparable_property()`的用例是能够从任何描述符返回`PropComparator`。实际上，“派生”的方法更易于使用，更具可扩展性，用几十行纯 Python 实现几乎不需要导入，甚至不需要 ORM 核心意识到它。该功能现在被称为“混合属性”扩展。

`synonym()`和`comparable_property()`仍然是 ORM 的一部分，尽管它们的实现已经移出，建立在类似于混合扩展的方法之上，以便核心 ORM 映射器/查询/属性模块在其他情况下并不真正意识到它们。

另请参阅

混合属性

[#1903](https://www.sqlalchemy.org/trac/ticket/1903)

### 速度增强

与所有主要 SQLA 版本一样，通过内部进行广泛的遍历以减少开销和调用次数，进一步减少了在常见情况下所需的工作量。此版本的亮点包括：

+   刷新过程现在将 INSERT 语句捆绑成批次供`cursor.executemany()`执行，对于主键已经存在的行。特别是在连接表继承配置中通常适用于“子”表，这意味着对于大量连接表对象的大批量插入，可以将对`cursor.execute`的调用次数减半，从而允许本地 DBAPI 优化对传递给`cursor.executemany()`的语句进行（例如重用准备好的语句）。

+   当访问已加载的相关对象的多对一引用时，调用的代码路径已经大大简化。直接检查标识映射，无需首先生成新的`Query`对象，这在访问成千上万个内存中的多对一时是昂贵的。对于大多数延迟属性加载，也不再使用每次构造的“加载器”对象。

+   重新编写组合使得在映射器内部访问映射属性时可以走更短的代码路径。

+   新的内联属性访问函数取代了以前在“保存-更新”和其他级联操作需要在属性关联的所有数据成员范围内级联时使用“history”的做法。这减少了为这个速度关键操作生成新的`History`对象的开销。

+   `ExecutionContext`的内部，即对语句执行的对象，已经被内联和简化。

+   由类型为每个语句执行生成的`bind_processor()`和`result_processor()`可调用现在被缓存（谨慎地，以避免临时类型和方言的内存泄漏）为该类型的寿命，进一步减少每个语句调用的开销。

+   特定`Compiled`语句实例的“绑定处理器”集合也缓存在`Compiled`对象上，进一步利用了由刷新过程使用的“编译缓存”来重复使用相同的 INSERT、UPDATE、DELETE 语句的编译形式。

包括一个示例基准脚本的调用次数减少演示在[`techspot.zzzeek.org/2010/12/12/a-tale-of-three`](https://techspot.zzzeek.org/2010/12/12/a-tale-of-three)- profiles/

### 复合已重写

“composite”功能已被重写，类似于`synonym()`和`comparable_property()`，使用了基于描述符和事件的轻量级实现，而不是构建到 ORM 内部。这样做可以从映射器/工作单元内部删除一些延迟，并简化复合属性的工作方式。复合属性现在不再隐藏其构建在其上的基础列，这些列现在保持为常规属性。复合还可以充当`relationship()`以及`Column()`属性的代理。

复合的主要不兼容性变化是它们不再使用`mutable=True`系统来检测原地突变。请使用[Mutation Tracking](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展来建立对现有复合使用的原地更改事件。

另请参阅

复合列类型

Mutation Tracking

[#2008](https://www.sqlalchemy.org/trac/ticket/2008) [#2024](https://www.sqlalchemy.org/trac/ticket/2024)

### 更简洁的查询.join(target, onclause)形式

向具有显式 onclause 的目标发出`query.join()`的默认方法现在是：

```py
query.join(SomeClass, SomeClass.id == ParentClass.some_id)
```

在 0.6 版本中，这种用法被认为是错误的，因为`join()`接受多个参数对应于多个 JOIN 子句 - 两个参数形式需要在元组中以消除单参数和双参数连接目标之间的歧义。在 0.6 中间，我们为这种特定的调用风格添加了检测和错误消息，因为它是如此常见。在 0.7 中，由于我们无论如何都在检测确切的模式，并且因为不得不无缘无故地输入一个元组是极其恼人的，非元组方法现在成为“正常”做法。与单个连接情况相比，“多个 JOIN”用例极为罕见，而如今多个连接更清晰地表示为多次调用`join()`。

元组形式将保留以确保向后兼容性。

请注意，所有其他形式的`query.join()`保持不变：

```py
query.join(MyClass.somerelation)
query.join("somerelation")
query.join(MyTarget)
# ... etc
```

[使用连接查询](https://www.sqlalchemy.org/docs/07/orm/tutorial.html#querying-with-joins)

[#1923](https://www.sqlalchemy.org/trac/ticket/1923)

### 突变事件扩展，取代“mutable=True”

一个新的扩展，Mutation Tracking，提供了一种机制，用户定义的数据类型可以向拥有的父级或父级提供更改事件。该扩展包括一种用于标量数据库值的方法，例如由`PickleType`、`postgresql.ARRAY`或其他自定义`MutableType`类管理的值，以及一种用于 ORM “组合”配置的方法，这些配置使用`composite()`。

另请参阅

Mutation Tracking

### NULLS FIRST / NULLS LAST 操作符

这些操作符作为`asc()`和`desc()`操作符的扩展实现，称为`nullsfirst()`和`nullslast()`。

另请参阅

`nullsfirst()`

`nullslast()`

[#723](https://www.sqlalchemy.org/trac/ticket/723)

### select.distinct()、query.distinct()接受 PostgreSQL DISTINCT ON 的*args

通过将表达式列表传递给`select()`的`distinct`关键字参数，现在`select()`和`Query`的`distinct()`方法接受位置参数，当使用 PostgreSQL 后端时，这些参数将被渲染为 DISTINCT ON。

[distinct()](https://www.sqlalchemy.org/docs/07/core/expression_api.html#sqlalchemy.sql.expression.Select.distinct)

[Query.distinct()](https://www.sqlalchemy.org/docs/07/orm/query.html#sqlalchemy.orm.query.Query.distinct)

[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

### `Index()`可以内联放置在`Table`、`__table_args__`中

Index() 构造可以与 Table 定义内联创建，使用字符串作为列名，作为在 Table 外创建索引的替代方法。即：

```py
Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50), nullable=False),
    Index("idx_name", "name"),
)
```

这里的主要原因是为了声明式`__table_args__`的好处，特别是在与 mixins 一起使用时：

```py
class HasNameMixin(object):
    name = Column("name", String(50), nullable=False)

    @declared_attr
    def __table_args__(cls):
        return (Index("name"), {})

class User(HasNameMixin, Base):
    __tablename__ = "user"
    id = Column("id", Integer, primary_key=True)
```

[Indexes](https://www.sqlalchemy.org/docs/07/core/schema.html#indexes)

### 窗口函数 SQL 构造

“窗口函数”向语句提供有关生成的结果集的信息。这允许根据诸如“行号”、“排名”等各种条件进行筛选。它们至少被 PostgreSQL、SQL Server 和 Oracle 支持，可能还有其他数据库。

关于窗口函数的最佳介绍在 PostgreSQL 的网站上，自从 8.4 版本以来就支持窗口函数：

[`www.postgresql.org/docs/current/static/tutorial-window.html`](https://www.postgresql.org/docs/current/static/tutorial-window.html)

SQLAlchemy 提供了一个简单的构造，通常通过现有的函数子句调用，使用`over()`方法，接受`order_by`和`partition_by`关键字参数。下面我们复制了 PG 教程中的第一个示例：

```py
from sqlalchemy.sql import table, column, select, func

empsalary = table("empsalary", column("depname"), column("empno"), column("salary"))

s = select(
    [
        empsalary,
        func.avg(empsalary.c.salary)
        .over(partition_by=empsalary.c.depname)
        .label("avg"),
    ]
)

print(s)
```

SQL:

```py
SELECT  empsalary.depname,  empsalary.empno,  empsalary.salary,
avg(empsalary.salary)  OVER  (PARTITION  BY  empsalary.depname)  AS  avg
FROM  empsalary
```

[sqlalchemy.sql.expression.over](https://www.sqlalchemy.org/docs/07/core/expression_api.html#sqlalchemy.sql.expression.over)

[#1844](https://www.sqlalchemy.org/trac/ticket/1844)

### Connection 上的 execution_options()接受“isolation_level”参数

这将为单个`Connection`设置事务隔离级别，直到该`Connection`关闭并其底层 DBAPI 资源返回到连接池，此时隔离级别将重置为默认值。默认的隔离级别是使用`create_engine()`的`isolation_level`参数设置的。

事务隔离支持目前仅由 PostgreSQL 和 SQLite 后端支持。

[execution_options()](https://www.sqlalchemy.org/docs/07/core/connections.html#sqlalchemy.engine.base.Connection.execution_options)

[#2001](https://www.sqlalchemy.org/trac/ticket/2001)

### `TypeDecorator`与整数主键列一起使用

一个`TypeDecorator`，它扩展了`Integer`的行为，可以与主键列一起使用。`Column`的“autoincrement”特性现在将识别到底层数据库列仍然是整数，以便`lastrowid`机制继续正常工作。`TypeDecorator`本身将其结果值处理器应用于新生成的主键，包括通过 DBAPI `cursor.lastrowid`访问器接收到的主键。

[#2005](https://www.sqlalchemy.org/trac/ticket/2005) [#2006](https://www.sqlalchemy.org/trac/ticket/2006)

### `TypeDecorator`存在于“sqlalchemy”导入空间中

不再需要从`sqlalchemy.types`导入此内容，现在在`sqlalchemy`中有镜像。

### 新方言

已添加方言：

+   一个用于 Drizzle 数据库的 MySQLdb 驱动程序：

    [Drizzle](https://www.sqlalchemy.org/docs/07/dialects/drizzle.html)

+   支持 pymysql DBAPI：

    [pymsql Notes](https://www.sqlalchemy.org/docs/07/dialects/mysql.html#module-sqlalchemy.dialects.mysql.pymysql)

+   psycopg2 现在与 Python 3 兼容

### 新事件系统

SQLAlchemy 早期开始使用`MapperExtension`类，该类提供了对映射器持久化周期的钩子。随着 SQLAlchemy 迅速变得更加组件化，将映射器推入更专注的配置角色，许多更多的“extension”、“listener”和“proxy”类出现，以解决各种活动拦截用例。部分原因是由活动的分歧驱动的；`ConnectionProxy`对象希望提供一个重写语句和参数的系统；`AttributeExtension`提供了一个替换传入值的系统，而`DDL`对象具有可以切换为方言敏感可调用的事件。

0.7 版本使用了一种新的、统一的方法重新实现了几乎所有这些插件点，保留了不同系统的所有功能，提供了更多的灵活性和更少的样板代码，性能更好，并且消除了需要为每个事件子系统学习根本不同的 API 的必要性。预先存在的类`MapperExtension`、`SessionExtension`、`AttributeExtension`、`ConnectionProxy`、`PoolListener`以及`DDLElement.execute_at`方法已被弃用，现在根据新系统实现 - 这些 API 仍然完全可用，并且预计将在可预见的未来保持不变。

新方法使用命名事件和用户定义的可调用对象将活动与事件关联起来。API 的外观和感觉受到了 JQuery、Blinder 和 Hibernate 等多样化来源的驱动，并且在与数十名用户进行的会议期间多次进行了修改，这些会议在 Twitter 上的响应率似乎比邮件列表更高。

它还具有一种开放式的目标规范系统，允许将事件与 API 类关联，例如所有的`Session`或`Engine`对象，以及特定 API 类的实例，例如特定的`Pool`或`Mapper`，以及相关对象，如映射的用户定义类，或者特定子类的映射父类实例的特定属性。各个监听器子系统可以对传入的用户定义监听器函数应用包装器，修改它们的调用方式 - 映射事件可以接收被操作对象的实例，或者其底层的`InstanceState`对象。属性事件可以选择是否要负责返回一个新值。

几个系统现在建立在新的事件 API 之上，包括新的“可变属性”API 以及复合属性。对事件的更大强调还导致引入了一些新事件，包括属性过期和刷新操作，pickle 加载/转储操作，完成的映射器构造操作。

另请参阅

事件

[#1902](https://www.sqlalchemy.org/trac/ticket/1902)

### 混合属性，实现/取代了 synonym()、comparable_property()

“派生属性”示例现在已成为官方扩展。`synonym()`的典型用例是为映射列提供描述符访问；`comparable_property()`的用例是能够从任何描述符返回`PropComparator`。实际上，“派生”的方法更容易使用，更具可扩展性，用几十行纯 Python 实现，几乎不需要导入，甚至不需要 ORM 核心知道它。该功能现在被称为“混合属性”扩展。

`synonym()`和`comparable_property()`仍然是 ORM 的一部分，尽管它们的实现已经移出，建立在类似于混合扩展的方法上，因此核心 ORM 映射器/查询/属性模块在其他情况下并不真正意识到它们。

另请参见

混合属性

[#1903](https://www.sqlalchemy.org/trac/ticket/1903)

### 速度增强

与所有主要 SQLA 版本一样，通过内部进行广泛的遍历以减少开销和调用次数，进一步减少了常见情况下所需的工作量。此版本的亮点包括：

+   刷新过程现在将 INSERT 语句捆绑成批次提供给`cursor.executemany()`，对于主键已经存在的行。特别是这通常适用于连接表继承配置中的“子”表，这意味着对于大量连接表对象的批量插入，可以将`cursor.execute`的调用次数减少一半，从而允许针对那些传递给`cursor.executemany()`的语句进行本地 DBAPI 优化（例如重用准备好的语句）。

+   当访问已加载的相关对象的多对一引用时，调用的代码路径已经大大简化。直接检查标识映射，无需首先生成一个新的`Query`对象，这在访问成千上万个内存中的多对一时是昂贵的。对于大多数延迟属性加载，也不再使用每次构造的“加载器”对象。

+   重写复合体允许在映射器内部访问在刷新中与映射属性相关的属性时，使用更短的代码路径。

+   新的内联属性访问函数取代了以前在“save-update”和其他级联操作需要在属性的所有数据成员范围内级联时使用“history”的用法。这减少了为这个速度关键操作生成新的`History`对象的开销。

+   `ExecutionContext`的内部，对应于语句执行的对象，已经内联并简化。

+   为每个语句执行生成的类型的`bind_processor()`和`result_processor()`可调用现在被缓存（小心翼翼地，以避免对临时类型和方言造成内存泄漏），在该类型的生命周期内，进一步减少每个语句调用的开销。

+   特定`Compiled`实例的“绑定处理器”集合也被缓存在`Compiled`对象上，进一步利用刷新过程使用的“编译缓存”，以重用相同的 INSERT、UPDATE、DELETE 语句的编译形式。

一个减少调用次数的演示，包括一个示例基准脚本，位于[`techspot.zzzeek.org/2010/12/12/a-tale-of-three`](https://techspot.zzzeek.org/2010/12/12/a-tale-of-three)- profiles/。

### 复合体重写

“复合”功能已经被重写，就像`synonym()`和`comparable_property()`一样，使用基于描述符和事件的轻量级实现，而不是构建到 ORM 内部。这允许从映射器/工作单元内部删除一些延迟，并简化复合的工作方式。复合属性现在不再隐藏其构建在其上的基础列，这些列现在保持为常规属性。复合还可以充当`relationship()`以及`Column()`属性的代理。

复合的主要不兼容变更是，它们不再使用`mutable=True`系统来检测原地变异。请使用[Mutation Tracking](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展来建立对现有复合使用的原地更改事件。

另请参见

复合列类型

变异跟踪

[#2008](https://www.sqlalchemy.org/trac/ticket/2008) [#2024](https://www.sqlalchemy.org/trac/ticket/2024)

### 更简洁的查询.join(target, onclause)形式

向具有显式 onclause 的目标发出`query.join()`的默认方法现在是：

```py
query.join(SomeClass, SomeClass.id == ParentClass.some_id)
```

在 0.6 版本中，这种用法被认为是错误的，因为`join()`接受多个参数对应于多个 JOIN 子句 - 两个参数形式需要在元组中以消除单参数和双参数连接目标之间的歧义。在 0.6 的中间，我们添加了检测和针对这种特定调用风格的错误消息，因为这种情况非常普遍。在 0.7 中，由于我们无论如何都在检测确切的模式，并且由于不得不无缘无故地输入元组非常恼人，非元组方法现在成为“正常”做法。与单个连接情况相比，“多个 JOIN”用例极为罕见，而如今多个连接更清晰地表示为多次调用`join()`。

元组形式将保留以确保向后兼容性。

请注意，所有其他形式的`query.join()`保持不变：

```py
query.join(MyClass.somerelation)
query.join("somerelation")
query.join(MyTarget)
# ... etc
```

[使用连接查询](https://www.sqlalchemy.org/docs/07/orm/tutorial.html#querying-with-joins)

[#1923](https://www.sqlalchemy.org/trac/ticket/1923)

### 变异事件扩展，取代“mutable=True”

一个新的扩展，变异跟踪，提供了一种机制，通过该机制，用户定义的数据类型可以向拥有的父级或父级提供更改事件。该扩展包括一种用于标量数据库值的方法，例如由`PickleType`管理的值，`postgresql.ARRAY`或其他自定义`MutableType`类，以及一种用于 ORM“复合”配置的方法，这些配置使用`composite()`。

另请参见

变异跟踪

### NULLS FIRST / NULLS LAST 操作符

这些被实现为`asc()`和`desc()`运算符的扩展，称为`nullsfirst()`和`nullslast()`。

另请参阅

`nullsfirst()`

`nullslast()`

[#723](https://www.sqlalchemy.org/trac/ticket/723)

### select.distinct()，query.distinct()接受*args 用于 PostgreSQL DISTINCT ON

通过将表达式列表传递给`select()`的`distinct`关键字参数，现在`select()`和`Query`的`distinct()`方法接受位置参数，当使用 PostgreSQL 后端时，这些参数将被渲染为 DISTINCT ON。

[distinct()](https://www.sqlalchemy.org/docs/07/core/expression_api.html#sqlalchemy.sql.expression.Select.distinct)

[Query.distinct()](https://www.sqlalchemy.org/docs/07/orm/query.html#sqlalchemy.orm.query.Query.distinct)

[#1069](https://www.sqlalchemy.org/trac/ticket/1069)

### `Index()`可以内联放置在`Table`、`__table_args__`内

Index()构造可以与 Table 定义内联创建，使用字符串作为列名，作为在 Table 之外创建索引的替代方法。即：

```py
Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("name", String(50), nullable=False),
    Index("idx_name", "name"),
)
```

这里的主要原因是为了声明性`__table_args__`的好处，特别是在与混合使用时：

```py
class HasNameMixin(object):
    name = Column("name", String(50), nullable=False)

    @declared_attr
    def __table_args__(cls):
        return (Index("name"), {})

class User(HasNameMixin, Base):
    __tablename__ = "user"
    id = Column("id", Integer, primary_key=True)
```

[Indexes](https://www.sqlalchemy.org/docs/07/core/schema.html#indexes)

### 窗口函数 SQL 构造

“窗口函数”为语句提供了有关生成的结果集的信息。这允许根据诸如“行号”、“排名”等各种条件进行查询。它们至少被已知支持的 PostgreSQL、SQL Server 和 Oracle 支持，可能还有其他数据库。

关于窗口函数的最佳介绍在 PostgreSQL 的网站上，窗口函数自 8.4 版本起就得到支持：

[`www.postgresql.org/docs/current/static/tutorial-window.html`](https://www.postgresql.org/docs/current/static/tutorial-window.html)

SQLAlchemy 提供了一个简单的构造，通常通过现有的函数子句调用，使用`over()`方法，接受`order_by`和`partition_by`关键字参数。下面我们复制了 PG 教程中的第一个示例：

```py
from sqlalchemy.sql import table, column, select, func

empsalary = table("empsalary", column("depname"), column("empno"), column("salary"))

s = select(
    [
        empsalary,
        func.avg(empsalary.c.salary)
        .over(partition_by=empsalary.c.depname)
        .label("avg"),
    ]
)

print(s)
```

SQL：

```py
SELECT  empsalary.depname,  empsalary.empno,  empsalary.salary,
avg(empsalary.salary)  OVER  (PARTITION  BY  empsalary.depname)  AS  avg
FROM  empsalary
```

[sqlalchemy.sql.expression.over](https://www.sqlalchemy.org/docs/07/core/expression_api.html#sqlalchemy.sql.expression.over)

[#1844](https://www.sqlalchemy.org/trac/ticket/1844)

### Connection 上的 execution_options()接受“isolation_level”参数

这为单个`Connection`设置了事务隔离级别，直到该`Connection`关闭并其底层 DBAPI 资源返回到连接池，此时隔离级别将重置为默认值。默认的隔离级别是使用`create_engine()`的`isolation_level`参数设置的。

事务隔离支持目前仅由 PostgreSQL 和 SQLite 后端支持。

[execution_options()](https://www.sqlalchemy.org/docs/07/core/connections.html#sqlalchemy.engine.base.Connection.execution_options)

[#2001](https://www.sqlalchemy.org/trac/ticket/2001)

### `TypeDecorator` 与整数主键列一起使用

可以使用扩展`Integer`行为的`TypeDecorator`与主键列一起使用。`Column`的“autoincrement”特性现在将识别到底层数据库列仍然是整数，以便`lastrowid`机制继续正常工作。`TypeDecorator`本身的结果值处理器将应用于新生成的主键，包括通过 DBAPI `cursor.lastrowid`访问器接收到的主键。

[#2005](https://www.sqlalchemy.org/trac/ticket/2005) [#2006](https://www.sqlalchemy.org/trac/ticket/2006)

### `TypeDecorator` 存在于“sqlalchemy”导入空间中

不再需要从`sqlalchemy.types`导入，现在在`sqlalchemy`中有镜像。

### 新方言

已添加方言：

+   用于 Drizzle 数据库的 MySQLdb 驱动程序：

    [Drizzle](https://www.sqlalchemy.org/docs/07/dialects/drizzle.html)

+   支持 pymysql DBAPI：

    [pymsql Notes](https://www.sqlalchemy.org/docs/07/dialects/mysql.html#module-sqlalchemy.dialects.mysql.pymysql)

+   psycopg2 现在与 Python 3 兼容

## 行为变更（向后兼容）

### 默认情况下构建 C 扩展

这是从 0.7b4 开始的。如果检测到 cPython 2.xx，则会构建扩展。如果构建失败，例如在 Windows 安装中，会捕获该条件并继续非 C 安装。如果使用 Python 3 或 PyPy，则不会构建 C 扩展。

### 简化的 Query.count()，几乎总是有效

`Query.count()`内部的非常古老的猜测现在已经被现代化，使用`.from_self()`。也就是说，`query.count()`现在等效于：

```py
query.from_self(func.count(literal_column("1"))).scalar()
```

以前，内部逻辑尝试重写查询本身的列子句，并在检测到“子查询”条件时，例如可能在其中具有聚合的基于列的查询，或具有 DISTINCT 的查询时，会经历一个繁琐的过程来重写列子句。这种逻辑在复杂条件下失败，特别是涉及联接表继承的条件，并且长期以来已经被更全面的`.from_self()`调用所淘汰。

`query.count()`生成的 SQL 现在总是形式为：

```py
SELECT  count(1)  AS  count_1  FROM  (
  SELECT  user.id  AS  user_id,  user.name  AS  user_name  from  user
)  AS  anon_1
```

换句话说，原始查询完全保留在子查询中，不再需要猜测如何应用计数。

[#2093](https://www.sqlalchemy.org/trac/ticket/2093)

#### 发出非子查询形式的 count()

MySQL 用户已经报告说，MyISAM 引擎在这个简单的更改中完全崩溃，这并不奇怪。请注意，对于优化不能处理简单子查询的数据库的简单`count()`，应该使用`func.count()`：

```py
from sqlalchemy import func

session.query(func.count(MyClass.id)).scalar()
```

或者对于`count(*)`：

```py
from sqlalchemy import func, literal_column

session.query(func.count(literal_column("*"))).select_from(MyClass).scalar()
```

### LIMIT/OFFSET 子句现在使用绑定参数

LIMIT 和 OFFSET 子句，或其后端等效项（即 TOP，ROW NUMBER OVER 等），对于支持它的所有后端使用绑定参数进行实际值，（除了 Sybase 之外的大多数后端）。这样做可以提高查询优化器的性能，因为具有不同 LIMIT/OFFSET 的多个语句的文本字符串现在是相同的。

[#805](https://www.sqlalchemy.org/trac/ticket/805)

### 日志增强

Vinay Sajip 提供了一个补丁，使我们的日志系统中不再需要在引擎和池的日志语句中嵌入“十六进制字符串”以使`echo`标志正常工作。使用过滤日志对象的新系统使我们能够保持`echo`仅适用于各个引擎而无需额外的标识字符串。

[#1926](https://www.sqlalchemy.org/trac/ticket/1926)

### 简化的多态 _on 赋值

在继承场景中使用时，`polymorphic_on`列映射属性的填充现在发生在对象构造时，即调用其`__init__`方法时，使用 init 事件。然后，该属性的行为与任何其他列映射属性相同。以前，特殊逻辑会在刷新期间触发以填充此列，这会阻止任何用户代码修改其行为。新方法在三个方面改进了这一点：1.多态标识现在在对象构造时立即存在；2.用户代码可以更改多态标识而不会与任何其他列映射属性有任何不同的行为；3.在刷新期间，映射器的内部简化，不再需要对此列进行特殊检查。

[#1895](https://www.sqlalchemy.org/trac/ticket/1895)

### 跨多个路径（即“all()”）的 contains_eager()链

`contains_eager()`修改器现在会把自己链接到一个更长的路径上，而不需要释放独立的`contains_eager()`调用。而不是：

```py
session.query(A).options(contains_eager(A.b), contains_eager(A.b, B.c))
```

你可以说：

```py
session.query(A).options(contains_eager(A.b, B.c))
```

[#2032](https://www.sqlalchemy.org/trac/ticket/2032)

### 允许刷新没有父级的孤儿

我们一直有一个长期存在的行为，即在刷新时检查所谓的“孤儿”，即与指定“delete-orphan”级联的`relationship()`相关联的对象，已经被新添加到会话中进行 INSERT 操作，但尚未建立父关系。多年前添加了此检查以适应一些测试用例，这些测试用例测试了孤儿行为的一致性。在现代 SQLA 中，这种检查在 Python 端不再需要。通过使对象的外键引用对象的父行为 NOT NULL，数据库会以 SQLA 允许大多数其他操作执行的方式确保数据一致性，从而实现“孤儿检查”的等效行为。如果对象的父外键是可为空的，则可以插入行。当对象与特定父对象一起持久化，然后与该父对象解除关联时，会触发“孤儿”行为，导致为其发出 DELETE 语句。

[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

### 在收集成员，不是刷新的标量引用时生成的警告

当通过父对象上标记为“脏”的加载`relationship()`引用的相关对象在当前`Session`中不存在时，现在会发出警告。

当对象被添加到`Session`时，或者当对象首次与父对象关联时，`save-update`级联生效，以便对象及其所有相关内容通常都存在于同一个`Session`中。但是，如果对于特定的`relationship()`禁用了`save-update`级联，则此行为不会发生，并且刷新过程不会尝试纠正它，而是保持一致到配置的级联行为。以前，在刷新期间检测到这样的对象时，它们会被静默跳过。新行为是发出警告，目的是提醒一个经常是意外行为来源的情况。

[#1973](https://www.sqlalchemy.org/trac/ticket/1973)

### 设置不再安装 Nose 插件

自从我们转向 nose 以来，我们使用了一个通过 setuptools 安装的插件，这样`nosetests`脚本会自动运行 SQLA 的插件代码，这对于我们的测试来说是必要的，以便有一个完整的环境。在 0.6 的中间，我们意识到这里的导入模式意味着 Nose 的“coverage”插件会中断，因为“coverage”要求在导入要覆盖的任何模块之前启动它；所以在 0.6 的中间，我们通过添加一个单独的`sqlalchemy-nose`包来克服这一情况，使情况变得更糟。

在 0.7 中，我们已经放弃了尝试让`nosetests`自动工作，因为 SQLAlchemy 模块会为所有`nosetests`的用法产生大量的 nose 配置选项，而不仅仅是 SQLAlchemy 单元测试本身，而且额外的`sqlalchemy-nose`安装甚至更糟，会在 Python 环境中产生一个额外的包。在 0.7 中，`sqla_nose.py`脚本现在是使用 nose 运行测试的唯一方法。

[#1949](https://www.sqlalchemy.org/trac/ticket/1949)

### 非`Table`派生的构造可以被映射

一个根本不针对任何`Table`的构造，比如一个函数，可以被映射。

```py
from sqlalchemy import select, func
from sqlalchemy.orm import mapper

class Subset(object):
    pass

selectable = select(["x", "y", "z"]).select_from(func.some_db_function()).alias()
mapper(Subset, selectable, primary_key=[selectable.c.x])
```

[#1876](https://www.sqlalchemy.org/trac/ticket/1876)

### aliased()接受`FromClause`元素

这是一个方便的辅助工具，以便在传递一个普通的`FromClause`，比如一个`select`，`Table`或`join`到`orm.aliased()`构造时，它会通过到该 from 构造的`.alias()`方法，而不是构造一个 ORM 级别的`AliasedClass`。

[#2018](https://www.sqlalchemy.org/trac/ticket/2018)

### Session.connection()，Session.execute()接受‘bind’

这是为了允许执行/连接操作明确参与引擎的打开事务。它还允许`Session`的自定义子类实现自己的`get_bind()`方法和参数，以便在`execute()`和`connection()`方法中等效地使用这些自定义参数。

[Session.connection](https://www.sqlalchemy.org/docs/07/orm/session.html#sqlalchemy.orm.session.Session.connection) [Session.execute](https://www.sqlalchemy.org/docs/07/orm/session.html#sqlalchemy.orm.session.Session.execute)

[#1996](https://www.sqlalchemy.org/trac/ticket/1996)

### 列子句中的独立绑定参数自动标记。

在选择的“列子句”中存在的绑定参数现在像其他“匿名”子句一样自动标记，这样在获取行时它们的“类型”就有意义，就像结果行处理器一样。

### SQLite - 相对文件路径通过 os.path.abspath()进行标准化

这样，更改当前目录的脚本将继续以后续建立的 SQLite 连接的相同位置为目标。

[#2036](https://www.sqlalchemy.org/trac/ticket/2036)

### MS-SQL - `String`/`Unicode`/`VARCHAR`/`NVARCHAR`/`VARBINARY`在未指定长度时发出“max”

在 MS-SQL 后端，String/Unicode 类型及其对应的 VARCHAR/NVARCHAR 类型，以及 VARBINARY ([#1833](https://www.sqlalchemy.org/trac/ticket/1833))在未指定长度时发出“max”作为长度。这使其与 PostgreSQL 的 VARCHAR 类型更兼容，当未指定长度时同样是无界的。SQL Server 在未指定长度时将这些类型的长度默认为‘1’。

### 默认构建 C 扩展

这是从 0.7b4 开始的。如果检测到 cPython 2.xx，则将构建扩展。如果构建失败，例如在 Windows 安装中，将捕获该条件并继续非 C 安装。如果使用 Python 3 或 PyPy，则不会构建 C 扩展。

### Query.count()简化，几乎总是有效

`Query.count()`内部的非常古老的猜测现已现代化为使用`.from_self()`。也就是说，`query.count()`现在等效于：

```py
query.from_self(func.count(literal_column("1"))).scalar()
```

以前，内部逻辑尝试重写查询本身的列子句，并在检测到“子查询”条件时，例如可能在其中具有聚合函数的基于列的查询，或具有 DISTINCT 的查询，将经历一个复杂的过程来重写列子句。这种逻辑在复杂条件下失败，特别是涉及联接表继承的条件，并且已经被更全面的`.from_self()`调用长时间废弃。

`query.count()`发出的 SQL 现在始终是以下形式：

```py
SELECT  count(1)  AS  count_1  FROM  (
  SELECT  user.id  AS  user_id,  user.name  AS  user_name  from  user
)  AS  anon_1
```

即原始查询完全保留在子查询中，不再猜测应如何应用计数。

[#2093](https://www.sqlalchemy.org/trac/ticket/2093)

#### 发出非子查询形式的 count()

MySQL 用户已经报告说，MyISAM 引擎不出所料地完全崩溃了这个简单的更改。请注意，对于优化无法处理简单子查询的数据库的简单`count()`，应使用`func.count()`：

```py
from sqlalchemy import func

session.query(func.count(MyClass.id)).scalar()
```

或对于`count(*)`：

```py
from sqlalchemy import func, literal_column

session.query(func.count(literal_column("*"))).select_from(MyClass).scalar()
```

#### 发出非子查询形式的 count()

MySQL 用户已经报告说，MyISAM 引擎不出所料地完全崩溃了这个简单的更改。请注意，对于优化无法处理简单子查询的数据库的简单`count()`，应使用`func.count()`：

```py
from sqlalchemy import func

session.query(func.count(MyClass.id)).scalar()
```

或对于`count(*)`：

```py
from sqlalchemy import func, literal_column

session.query(func.count(literal_column("*"))).select_from(MyClass).scalar()
```

### LIMIT/OFFSET 子句现在使用绑定参数

LIMIT 和 OFFSET 子句，或其后端等效项（即 TOP，ROW NUMBER OVER 等），对实际值使用绑定参数，对于支持它的所有后端（除了 Sybase）。这允许更好的查询优化器性能，因为具有不同 LIMIT/OFFSET 的多个语句的文本字符串现在是相同的。

[#805](https://www.sqlalchemy.org/trac/ticket/805)

### 日志增强

Vinay Sajip 提供了一个补丁，使我们的日志系统中不再需要“十六进制字符串”嵌入到引擎和池的日志语句中，以使`echo`标志能够正确工作。使用过滤日志对象的新系统使我们能够保持我们当前的`echo`行为，即`echo`局限于各个引擎，而无需额外的标识字符串局限于这些引擎。

[#1926](https://www.sqlalchemy.org/trac/ticket/1926)

### 简化的多态 _on 赋值

当在继承场景中使用时，`polymorphic_on`列映射属性的人口现在在对象构造时发生，即调用其`__init__`方法，使用 init 事件。然后，该属性的行为与任何其他列映射属性相同。以前，特殊逻辑会在刷新时触发以填充此列，这会阻止任何用户代码修改其行为。新方法在三个方面改进了这一点：1. 多态标识现在在对象构造时立即存在；2. 多态标识可以被用户代码更改，而不会与任何其他列映射属性的行为有任何区别；3. 刷新期间映射器的内部简化，不再需要对此列进行特殊检查。

[#1895](https://www.sqlalchemy.org/trac/ticket/1895)

### 包含跨多个路径（即“all()”）的 contains_eager()链

`contains_eager()`修改器现在会把自己链接到一个更长的路径上，而不需要释放独立的`contains_eager()`调用。而不是：

```py
session.query(A).options(contains_eager(A.b), contains_eager(A.b, B.c))
```

你可以说：

```py
session.query(A).options(contains_eager(A.b, B.c))
```

[#2032](https://www.sqlalchemy.org/trac/ticket/2032)

### 允许刷新没有父级的孤立对象

我们长期以来一直有一个行为，即在执行 flush 操作时检查所谓的“孤立对象”，即，与一个指定了“delete-orphan”级联的 `relationship()` 相关联的对象已经被新增到了会话中以进行 INSERT 操作，但尚未建立父关系。多年前，为了满足一些测试用例对孤立对象行为的一致性进行测试，添加了此检查。在现代 SQLA 中，不再需要在 Python 端进行此检查。通过将对象的外键引用设置为对象的父行的 NOT NULL，数据库会在确立数据一致性方面发挥作用，SQLA 允许大多数其他操作以相同的方式完成。如果对象的父外键可为空，则可以插入行。当对象被持久化到特定父对象，并且然后与该父对象取消关联时，“孤立”行为会运行，导致为其发出 DELETE 语句。

[#1912](https://www.sqlalchemy.org/trac/ticket/1912)

### 在刷新时生成的警告，集合成员，不是刷新的一部分的标量引用

当通过父对象上标记为“脏”的加载的 `relationship()` 引用的相关对象在当前 `Session` 中不存在时，会发出警告。

当对象被添加到 `Session` 中，或者当对象首次与父对象关联时，`save-update` 级联会生效，以便对象及其相关内容通常都存在于同一个 `Session` 中。但是，如果对特定的 `relationship()` 禁用了 `save-update` 级联，则此行为不会发生，并且刷新过程不会尝试进行纠正，而是保持一致性以配置的级联行为。以前，在刷新期间检测到此类对象时，它们会被静默跳过。新行为是发出警告，以便提醒通常是意外行为的根源。

[#1973](https://www.sqlalchemy.org/trac/ticket/1973)

### 设置不再安装 Nose 插件。

自从我们使用 nose 以来，我们已经使用通过 setuptools 安装的插件，以便 `nosetests` 脚本会自动运行 SQLA 的插件代码，这对于我们的测试来说是必要的，以获得完整的环境。在 0.6 版本中间，我们意识到这里的导入模式意味着 Nose 的“覆盖率”插件会中断，因为“覆盖率”要求在导入要覆盖的任何模块之前启动；因此，在 0.6 版本中间，我们通过添加一个单独的 `sqlalchemy-nose` 包到构建中来克服这个问题，使情况变得更糟。

在 0.7 中，我们放弃了尝试自动使`nosetests`工作，因为 SQLAlchemy 模块会为所有`nosetests`的用法产生大量的 nose 配置选项，而不仅仅是 SQLAlchemy 单元测试本身，而且额外的`sqlalchemy-nose`安装是一个更糟糕的想法，会在 Python 环境中产生一个额外的包。在 0.7 中，`sqla_nose.py`脚本现在是使用 nose 运行测试的唯一方法。

[#1949](https://www.sqlalchemy.org/trac/ticket/1949)

### 非`Table`派生的构造可以被映射

一个根本不针对任何`Table`的构造，比如一个函数，可以被映射。

```py
from sqlalchemy import select, func
from sqlalchemy.orm import mapper

class Subset(object):
    pass

selectable = select(["x", "y", "z"]).select_from(func.some_db_function()).alias()
mapper(Subset, selectable, primary_key=[selectable.c.x])
```

[#1876](https://www.sqlalchemy.org/trac/ticket/1876)

### aliased()接受`FromClause`元素

这是一个方便的辅助工具，以便在传递一个普通的`FromClause`，比如一个`select`，`Table`或`join`到`orm.aliased()`构造时，它会通过到该 from 构造的`.alias()`方法，而不是构造一个 ORM 级别的`AliasedClass`。

[#2018](https://www.sqlalchemy.org/trac/ticket/2018)

### Session.connection()，Session.execute()接受‘bind’

这是为了允许 execute/connection 操作明确参与引擎的打开事务。它还允许`Session`的自定义子类实现自己的`get_bind()`方法和参数，以便在`execute()`和`connection()`方法中同等使用这些自定义参数。

[Session.connection](https://www.sqlalchemy.org/docs/07/orm/session.html#sqlalchemy.orm.session.Session.connection) [Session.execute](https://www.sqlalchemy.org/docs/07/orm/session.html#sqlalchemy.orm.session.Session.execute)

[#1996](https://www.sqlalchemy.org/trac/ticket/1996)

### 列子句中的独立绑定参数自动标记。

在 select 的“columns clause”中存在的绑定参数现在会像其他“匿名”子句一样自动标记，这样在获取行时它们的“类型”就会有意义，就像结果行处理器一样。

### SQLite - 相对文件路径通过 os.path.abspath()进行标准化

这样，一个改变当前目录的脚本将继续以后续建立的 SQLite 连接的相同位置为目标。

[#2036](https://www.sqlalchemy.org/trac/ticket/2036)

### MS-SQL - `String`/`Unicode`/`VARCHAR`/`NVARCHAR`/`VARBINARY`在没有长度时发出“max”

在 MS-SQL 后端，String/Unicode 类型及其对应的 VARCHAR/ NVARCHAR 类型，以及 VARBINARY ([#1833](https://www.sqlalchemy.org/trac/ticket/1833))在没有指定长度时发出“max”作为长度。这使其更兼容于 PostgreSQL 的 VARCHAR 类型，当没有指定长度时也是无限制的。SQL Server 在没有指定长度时将这些类型的长度默认为‘1’。

## 行为变更（不兼容）

再次注意，除了默认的可变性更改外，大多数这些更改都是*非常微小*的，不会影响大多数用户。

### `PickleType`和 ARRAY 的可变性默认关闭

这个改变涉及 ORM 映射具有`PickleType`或`postgresql.ARRAY`数据类型的列时的默认行为。`mutable`标志现在默认设置为`False`。如果现有应用程序使用这些类型并依赖于检测就地变异，那么类型对象必须使用`mutable=True`构造以恢复 0.6 版本的行为：

```py
Table(
    "mytable",
    metadata,
    # ....
    Column("pickled_data", PickleType(mutable=True)),
)
```

`mutable=True`标志正在逐步淘汰，而新的[Mutation Tracking](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展则更受青睐。该扩展提供了一种机制，使用户定义的数据类型能够向拥有父对象发送更改事件。

先前使用`mutable=True`的方法不提供更改事件 - 相反，ORM 必须在每次调用`flush()`时扫描会话中存在的所有可变值，并将它们与原始值进行比较以检测更改，这是一个非常耗时的事件。这是 SQLAlchemy 非常早期的遗留问题，当时`flush()`不是自动的，历史跟踪系统也不像现在这样复杂。

现有应用程序使用`PickleType`、`postgresql.ARRAY`或其他`MutableType`子类，并且需要就地变异检测的应用程序，应该迁移到新的变异跟踪系统，因为`mutable=True`可能会在未来被弃用。

[#1980](https://www.sqlalchemy.org/trac/ticket/1980)

### `composite()`的可变性检测需要使用 Mutation Tracking Extension。

所谓的“复合”映射属性，即使用[Composite Column Types](https://www.sqlalchemy.org/docs/07/orm/mapper_config.html#composite-column-types)描述的技术配置的属性，已经重新实现，ORM 内部不再意识到它们（导致关键部分的代码路径更短更高效）。虽然复合类型通常被视为不可变的值对象，但从未强制执行。对于使用具有可变性的复合对象的应用程序，[Mutation Tracking](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展提供了一个基类，该基类建立了一个机制，使用户定义的复合类型能够向每个对象的拥有父对象发送更改事件消息。

使用复合类型并依赖于这些对象的就地变异检测的应用程序应该迁移到“变异跟踪”扩展，或者更改复合类型的使用方式，使得不再需要就地更改（即将其视为不可变的值对象）。

### SQLite - SQLite 方言现在对基于文件的数据库使用`NullPool`。

这个改变是**99.999%向后兼容**的，除非你在连接池连接之间使用临时表。

基于文件的 SQLite 连接速度非常快，使用`NullPool`意味着每次调用`Engine.connect`都会创建一个新的 pysqlite 连接。

以前使用的是`SingletonThreadPool`，这意味着线程中对某个引擎的所有连接都是相同的连接。新方法更直观，特别是在使用多个连接时。

当使用`:memory:`数据库时，`SingletonThreadPool`仍然是默认引擎。

请注意，这个改变**破坏了跨 Session 提交使用的临时表**，这是由于 SQLite 处理临时表的方式造成的。如果需要超出一个连接池范围的临时表，请参阅[`www.sqlalchemy.org/docs/dialects/sqlite.html#using`](https://www.sqlalchemy.org/docs/dialects/sqlite.html#using)- temporary-tables-with-sqlite 中的说明。

[#1921](https://www.sqlalchemy.org/trac/ticket/1921)

### `Session.merge()`检查版本化映射器的版本 id

Session.merge()将检查传入状态的版本 id 与数据库的版本 id 是否匹配，假设映射使用版本 id 并且传入状态已分配版本 id，并且如果它们不匹配，则引发 StaleDataError。这是正确的行为，因为如果传入状态包含过时的版本 id，则应假定状态是过时的。

如果将数据合并到版本化状态中，则可以将版本 id 属性未定义，并且不会进行版本检查。

通过检查 Hibernate 的操作来确认了这个检查 - `merge()`和版本化功能最初都是从 Hibernate 中适应过来的。

[#2027](https://www.sqlalchemy.org/trac/ticket/2027)

### 查询中的元组标签名称改进

这个改进对于依赖于旧行为的应用程序可能会有一点向后不兼容。

给定两个映射类`Foo`和`Bar`，每个类都有一个列`spam`：

```py
qa = session.query(Foo.spam)
qb = session.query(Bar.spam)

qu = qa.union(qb)
```

由`qu`产生的单个列的名称将是`spam`。以前会是类似`foo_spam`的东西，这是由于`union`组合事物的方式，这与非联合查询的情况下的名称`spam`不一致。

[#1942](https://www.sqlalchemy.org/trac/ticket/1942)

### 映射列属性首先引用最具体的列

这是在映射列属性引用多个列时涉及的行为更改，特别是在处理具有与超类属性相同名称的连接表子类上的属性时。

使用声明性，情景是这样的：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)

class Child(Parent):
    __tablename__ = "child"
    id = Column(Integer, ForeignKey("parent.id"), primary_key=True)
```

上面，属性`Child.id`同时指代`child.id`列和`parent.id` - 这是由于属性的名称。如果在类上命名不同，比如`Child.child_id`，那么它将明确映射到`child.id`，而`Child.id`将与`Parent.id`相同。

当`id`属性被设置为引用`parent.id`和`child.id`时，它们被存储在一个有序列表中。这样，诸如`Child.id`的表达式在呈现时只引用*其中一个*列。直到 0.6 版本，这一列将是`parent.id`。在 0.7 版本中，它是更少令人惊讶的`child.id`。

这种行为的遗留与 ORM 的行为和限制有关，这些限制实际上已经不再适用；所需要的只是颠倒顺序。

这种方法的一个主要优势是现在更容易构造引用本地列的`primaryjoin`表达式：

```py
class Child(Parent):
    __tablename__ = "child"
    id = Column(Integer, ForeignKey("parent.id"), primary_key=True)
    some_related = relationship(
        "SomeRelated", primaryjoin="Child.id==SomeRelated.child_id"
    )

class SomeRelated(Base):
    __tablename__ = "some_related"
    id = Column(Integer, primary_key=True)
    child_id = Column(Integer, ForeignKey("child.id"))
```

在 0.7 之前，`Child.id`表达式将引用`Parent.id`，并且需要将`child.id`映射到一个不同的属性。

这也意味着这样一个查询的行为发生了变化：

```py
session.query(Parent).filter(Child.id > 7)
```

在 0.6 中，这将产生：

```py
SELECT  parent.id  AS  parent_id
FROM  parent
WHERE  parent.id  >  :id_1
```

在 0.7 版中，你会得到：

```py
SELECT  parent.id  AS  parent_id
FROM  parent,  child
WHERE  child.id  >  :id_1
```

你会注意到这是一个笛卡尔积 - 这种行为现在等同于任何其他本地于`Child`的属性。`with_polymorphic()`方法，或者类似的显式连接底层`Table`对象的策略，用于以与 0.5 和 0.6 相同的方式渲染针对所有`Parent`对象的查询，并针对`Child`进行条件限制：

```py
print(s.query(Parent).with_polymorphic([Child]).filter(Child.id > 7))
```

在 0.6 和 0.7 上都会产生：

```py
SELECT  parent.id  AS  parent_id,  child.id  AS  child_id
FROM  parent  LEFT  OUTER  JOIN  child  ON  parent.id  =  child.id
WHERE  child.id  >  :id_1
```

这种变化的另一个影响是，跨两个表的连接继承加载将从子表的值填充，而不是父表的值。一个不寻常的情况是，使用`with_polymorphic="*"`对“Parent”进行查询，会发出一个针对“parent”的查询，带有一个左外连接到“child”。行位于“Parent”，看到多态标识对应于“Child”，但假设“child”中的实际行已经*被删除*。由于这种损坏，行将带有所有对应于“child”的列设置为 NULL 的列 - 这现在是被填充的值，而不是父表中的值。

[#1892](https://www.sqlalchemy.org/trac/ticket/1892)

### 映射到具有两个或更多同名列的连接需要明确声明。

这与先前在[#1892](https://www.sqlalchemy.org/trac/ticket/1892)中的更改有些相关。在映射到连接时，同名列必须明确链接到映射的属性，即如[在多个表上映射一个类](http://www.sqlalchemy.org/docs/07/orm/mapper_config.html#mapping-a-class-against-multiple-tables)中所述。

给定两个表`foo`和`bar`，每个表都有一个主键列`id`，现在以下内容会产生错误：

```py
foobar = foo.join(bar, foo.c.id == bar.c.foo_id)
mapper(FooBar, foobar)
```

这是因为`mapper()`拒绝猜测哪一列是`FooBar.id`的主要表示 - 是`foo.c.id`还是`bar.c.id`？属性必须是明确的：

```py
foobar = foo.join(bar, foo.c.id == bar.c.foo_id)
mapper(FooBar, foobar, properties={"id": [foo.c.id, bar.c.id]})
```

[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

### Mapper 要求在映射的可选择项中存在多态列`polymorphic_on`。

这是 0.6 版的警告，现在在 0.7 版中已经成为错误。给定`polymorphic_on`的列必须在映射的可选择项中。这是为了防止一些偶发的用户错误，例如：

```py
mapper(SomeClass, sometable, polymorphic_on=some_lookup_table.c.id)
```

在这种情况下，`polymorphic_on`需要在`sometable`列上，例如在这种情况下可能是`sometable.c.some_lookup_id`。还有一些“多态联合”场景，类似的错误有时会发生。

这样的配置错误一直是“错误的”，上述映射不像规定的那样工作 - 该列将被忽略。然而，如果某个应用程序不知情地依赖于此行为，则这可能是潜在的向后不兼容的情况。

[#1875](https://www.sqlalchemy.org/trac/ticket/1875)

### `DDL()`构造现在转义百分号

以前，在`DDL()`字符串中的百分号必须转义，即 `%%` 取决于 DBAPI，对于那些接受`pyformat`或`format`绑定（即 psycopg2，mysql-python），这与`text()`构造函数自动执行的操作不一致。现在对于`DDL()`与`text()`，进行了相同的转义。

[#1897](https://www.sqlalchemy.org/trac/ticket/1897)

### `Table.c` / `MetaData.tables` 稍微改进，不允许直接突变

另一个区域是，一些用户以一种实际上不按预期工作的方式进行尝试，但仍然留下了极小的可能性，即某些应用程序正在依赖于此行为，`Table`的`.c`属性返回的构造和`MetaData`的`.tables`属性是明确不可变的。构造的“可变”版本现在是私有的。向`.c`添加列涉及使用`Table`的`append_column()`方法，该方法确保事物以适当的方式与父`Table`关联; 同样，`MetaData.tables`与此字典中存储的`Table`对象有一个协议，以及一些新的簿记，即跟踪所有模式名称的`set()`，仅使用公共`Table`构造函数以及`Table.tometadata()`才能满足。

当然，`ColumnCollection`和`dict`这些属性所查询的集合可能会在所有突变方法上实现事件，以便在直接突变集合时发生适当的簿记，但是在有人有动力实现所有这些以及数十个新单元测试之前，缩小这些集合突变路径将确保没有应用程序试图依赖于当前不受支持的用法。

[#1893](https://www.sqlalchemy.org/trac/ticket/1893) [#1917](https://www.sqlalchemy.org/trac/ticket/1917)

### server_default 对所有 inserted_primary_key 值一致地返回 None

当 Integer PK 列上存在 server_default 时确保一致性。SQLA 不预先获取这些，它们也不会在 cursor.lastrowid（DBAPI）中返回。确保所有后端一致地对这些值在 result.inserted_primary_key 中返回 None - 一些后端以前可能返回了一个值。在主键列上使用 server_default 是极不常见的。如果使用特殊函数或 SQL 表达式生成主键默认值，则应将其建立为 Python 端的“默认”而不是 server_default。

对于这种情况的反射，使用具有 server_default 的 int PK 列的反射将“autoincrement”标志设置为 False，除非在检测到序列默认值的 PG SERIAL 列的情况下。

[#2020](https://www.sqlalchemy.org/trac/ticket/2020) [#2021](https://www.sqlalchemy.org/trac/ticket/2021)

### `sqlalchemy.exceptions`在 sys.modules 中的别名已被移除

几年来，我们已经将字符串`sqlalchemy.exceptions`添加到`sys.modules`中，以便像“`import sqlalchemy.exceptions`”这样的语句可以工作。核心异常模块的名称现在已经是`exc`很长时间了，因此该模块的推荐导入方式是：

```py
from sqlalchemy import exc
```

对于可能已经说过`from sqlalchemy import exceptions`的应用程序，“`sqlalchemy`”中仍然存在`exceptions`名称，但他们也应该开始使用`exc`名称。

### 查询时间配方更改

虽然不是 SQLAlchemy 本身的一部分，但值得一提的是，将`ConnectionProxy`重构为新的事件系统意味着不再适用于“Timing all Queries”配方。请调整查询计时器以使用`before_cursor_execute()`和`after_cursor_execute()`事件，在更新的配方 UsageRecipes/Profiling 中有示例。

### `PickleType`和 ARRAY 的可变性默认关闭

此更改涉及 ORM 在映射具有`PickleType`或`postgresql.ARRAY`数据类型的列时的默认行为。`mutable`标志现在默认设置为`False`。如果现有应用程序使用这些类型并依赖于原地变异的检测，则必须使用`mutable=True`构造类型对象以恢复 0.6 行为：

```py
Table(
    "mytable",
    metadata,
    # ....
    Column("pickled_data", PickleType(mutable=True)),
)
```

`mutable=True`标志正在逐步淘汰，取而代之的是新的[Mutation Tracking](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展。该扩展提供了一种机制，使用户定义的数据类型可以向拥有的父级或父级提供更改事件。

先前使用`mutable=True`的方法不提供更改事件 - 相反，ORM 必须在每次调用`flush()`时扫描会话中存在的所有可变值，并将它们与原始值进行比较以检测更改，这是一个非常耗时的事件。这是 SQLAlchemy 非常早期的遗留问题，当时`flush()`不是自动的，历史跟踪系统也没有现在这么复杂。

使用`PickleType`、`postgresql.ARRAY`或其他`MutableType`子类，并且需要原地变异检测的现有应用程序，应该迁移到新的变异跟踪系统，因为`mutable=True`可能会在未来被弃用。

[#1980](https://www.sqlalchemy.org/trac/ticket/1980)

### `composite()`的可变性检测需要 Mutation Tracking 扩展

所谓“复合”映射属性，即使用[复合列类型](https://www.sqlalchemy.org/docs/07/orm/mapper_config.html#composite-column-types)描述的技术配置的属性，已经重新实现，使得 ORM 内部不再意识到它们（导致关键部分的代码路径更短更高效）。虽然复合类型通常被视为不可变的值对象，但从未强制执行。对于使用具有可变性的复合类型的应用程序，[变异跟踪](https://www.sqlalchemy.org/docs/07/orm/extensions/mutable.html)扩展提供了一个基类，建立了一个机制，使用户定义的复合类型能够向每个对象的拥有父对象发送更改事件消息。

使用复合类型并依赖于这些对象的原地变异检测的应用程序应该迁移到“变异跟踪”扩展，或者更改复合类型的使用方式，使得不再需要原地更改（即将其视为不可变的值对象）。

### SQLite - SQLite 方言现在对于基于文件的数据库使用`NullPool`

这个改变**99.999%向后兼容**，除非您正在跨连接池连接使用临时表。

基于文件的 SQLite 连接非常快速，并且使用`NullPool`意味着每次调用`Engine.connect`都会创建一个新的 pysqlite 连接。

以前使用的是`SingletonThreadPool`，这意味着线程中对某个引擎的所有连接都是相同的连接。新方法的目的是更直观，特别是在使用多个连接时。

当使用`:memory:`数据库时，`SingletonThreadPool`仍然是默认引擎。

请注意，这个改变**破坏了跨 Session 提交使用的临时表**，这是由于 SQLite 处理临时表的方式。如果需要超出一个连接池连接范围的临时表，请参阅[`www.sqlalchemy.org/docs/dialects/sqlite.html#using`](https://www.sqlalchemy.org/docs/dialects/sqlite.html#using)- temporary-tables-with-sqlite 中的说明。

[#1921](https://www.sqlalchemy.org/trac/ticket/1921)

### `Session.merge()`检查带版本的映射器的版本 ID

`Session.merge()`将检查传入状态的版本 ID 与数据库的版本 ID 是否匹配，假设映射使用版本 ID 并且传入状态已分配版本 ID，并且如果它们不匹配，则引发 StaleDataError。这是正确的行为，即如果传入状态包含过时的版本 ID，则应假定状态是过时的。

如果将数据合并到带版本的状态中，则可以将版本 ID 属性留空，不会进行版本检查。

通过检查 Hibernate 的操作确认了这一变化 - `merge()`和版本控制功能最初都是从 Hibernate 中适应过来的。

[#2027](https://www.sqlalchemy.org/trac/ticket/2027)

### 查询改进中的元组标签名称

这种改进对于依赖旧行为的应用程序可能会有一点点向后不兼容。

给定两个映射类`Foo`和`Bar`，每个类都有一个名为`spam`的列：

```py
qa = session.query(Foo.spam)
qb = session.query(Bar.spam)

qu = qa.union(qb)
```

由`qu`生成的单列的名称将是`spam`。以前会是类似于`foo_spam`的东西，这是由于`union`的组合方式导致的，这与非联合查询中的名称`spam`不一致。

[#1942](https://www.sqlalchemy.org/trac/ticket/1942)

### 映射列属性首先引用最具体的列

这是在映射列属性引用多个列时涉及的行为变化，特别是在处理具有与超类属性相同名称的连接表子类上的属性时。

使用声明性，情景如下：

```py
class Parent(Base):
    __tablename__ = "parent"
    id = Column(Integer, primary_key=True)

class Child(Parent):
    __tablename__ = "child"
    id = Column(Integer, ForeignKey("parent.id"), primary_key=True)
```

上面，属性`Child.id`同时指代`child.id`列和`parent.id` - 这是由于属性的名称。如果在类上命名不同，比如`Child.child_id`，那么它将明确映射到`child.id`，而`Child.id`将成为与`Parent.id`相同的属性。

当`id`属性被设置为引用`parent.id`和`child.id`时，它们被存储在一个有序列表中。这样，诸如`Child.id`的表达式在呈现时只会引用其中的*一个*列。直到 0.6 版本，这一列将是`parent.id`。在 0.7 版本中，它是更少令人惊讶的`child.id`。

这种行为的遗留涉及到 ORM 的行为和限制，这些限制实际上已经不再适用；只需要改变顺序即可。

这种方法的一个主要优势是现在更容易构建引用本地列的`primaryjoin`表达式：

```py
class Child(Parent):
    __tablename__ = "child"
    id = Column(Integer, ForeignKey("parent.id"), primary_key=True)
    some_related = relationship(
        "SomeRelated", primaryjoin="Child.id==SomeRelated.child_id"
    )

class SomeRelated(Base):
    __tablename__ = "some_related"
    id = Column(Integer, primary_key=True)
    child_id = Column(Integer, ForeignKey("child.id"))
```

在 0.7 之前，`Child.id`表达式将引用`Parent.id`，并且需要将`child.id`映射到一个不同的属性。

这也意味着这样一个查询的行为发生了变化：

```py
session.query(Parent).filter(Child.id > 7)
```

在 0.6 版本中，会呈现如下：

```py
SELECT  parent.id  AS  parent_id
FROM  parent
WHERE  parent.id  >  :id_1
```

在 0.7 中，你会得到：

```py
SELECT  parent.id  AS  parent_id
FROM  parent,  child
WHERE  child.id  >  :id_1
```

你会注意到这是一个笛卡尔积 - 这种行为现在等同于任何本地于`Child`的其他属性。`with_polymorphic()`方法，或者类似的显式连接基础`Table`对象的策略，用于对所有具有针对`Child`的条件的`Parent`对象进行查询，方式与 0.5 和 0.6 相同：

```py
print(s.query(Parent).with_polymorphic([Child]).filter(Child.id > 7))
```

在 0.6 和 0.7 版本中都会呈现：

```py
SELECT  parent.id  AS  parent_id,  child.id  AS  child_id
FROM  parent  LEFT  OUTER  JOIN  child  ON  parent.id  =  child.id
WHERE  child.id  >  :id_1
```

这种变化的另一个影响是，跨两个表的连接继承加载将从子表的值填充，而不是父表的值。一个不寻常的情况是，使用`with_polymorphic="*"`对“Parent”进行查询会发出一个针对“parent”的查询，并与“child”进行左外连接。行位于“Parent”，看到多态标识对应于“Child”，但假设“child”中的实际行已经*删除*。由于这种损坏，行中所有与“child”对应的列都设置为 NULL - 这现在是被填充的值，而不是父表中的值。

[#1892](https://www.sqlalchemy.org/trac/ticket/1892)

### 映射到具有两个或更多同名列的连接需要明确声明

这与[#1892](https://www.sqlalchemy.org/trac/ticket/1892)中的先前更改有些相关。在映射到连接时，同名列必须明确链接到映射属性，即如[映射一个类到多个表](http://www.sqlalchemy.org/docs/07/orm/mapper_config.html#mapping-a-class-against-multiple-tables)中所述。

给定两个表`foo`和`bar`，每个表都有一个主键列`id`，现在执行以下操作会产生错误：

```py
foobar = foo.join(bar, foo.c.id == bar.c.foo_id)
mapper(FooBar, foobar)
```

这是因为`mapper()`拒绝猜测哪一列是`FooBar.id`的主要表示 - 是`foo.c.id`还是`bar.c.id`？属性必须明确：

```py
foobar = foo.join(bar, foo.c.id == bar.c.foo_id)
mapper(FooBar, foobar, properties={"id": [foo.c.id, bar.c.id]})
```

[#1896](https://www.sqlalchemy.org/trac/ticket/1896)

### 映射器要求在映射的可选择项中存在多态列

这在 0.6 中是一个警告，现在在 0.7 中是一个错误。为`polymorphic_on`指定的列必须在映射的可选择项中。这是为了防止一些偶尔发生的用户错误，例如：

```py
mapper(SomeClass, sometable, polymorphic_on=some_lookup_table.c.id)
```

在上面的例子中，`polymorphic_on`需要在`sometable`列上，此时可能是`sometable.c.some_lookup_id`。有时也会发生一些类似的“多态联合”场景中的错误。

这种配置错误一直是“错误的”，上述映射不按规定工作 - 列将被忽略。然而，在极少数情况下，如果应用程序无意中依赖于这种行为，则可能会产生潜在的向后不兼容性。

[#1875](https://www.sqlalchemy.org/trac/ticket/1875)

### `DDL()`构造现在转义百分号

以前，在`DDL()`字符串中的百分号必须进行转义，即根据 DBAPI，对于那些接受`pyformat`或`format`绑定（例如 psycopg2，mysql-python）的 DBAPI，这是不一致的，与`text()`构造自动执行的操作不同。现在，`DDL()`与`text()`一样进行相同的转义。

[#1897](https://www.sqlalchemy.org/trac/ticket/1897)

### `Table.c` / `MetaData.tables`稍作调整，不允许直接变异

另一个领域，一些用户在进行尝试时，并不按预期工作，但仍然存在极小的可能性，即某些应用程序依赖于这种行为，`Table`的`.c`属性和`MetaData`的`.tables`属性返回的构造明确是不可变的。构造的“可变”版本现在是私有的。向`.c`添加列涉及使用`Table`的`append_column()`方法，这确保了事物以适当的方式与父`Table`关联；同样，`MetaData.tables`与存储在此字典中的`Table`对象有合同，还有一点新的簿记，即跟踪所有模式名称的`set()`，只有使用公共`Table`构造函数以及`Table.tometadata()`才能满足。

当然，`ColumnCollection` 和 `dict` 集合在这些属性上进行咨询的情况下，可能会有一天实现所有变异方法的事件，从而在直接变异集合时发生适当的簿记，但在有人有动力实现所有这些以及数十个新单元测试之前，缩小这些集合的变异路径将确保没有应用程序试图依赖当前不受支持的用法。

[#1893](https://www.sqlalchemy.org/trac/ticket/1893) [#1917](https://www.sqlalchemy.org/trac/ticket/1917)

### server_default 对所有 inserted_primary_key 值一致地返回 None。

当在 Integer PK 列上存在 server_default 时，已确立一致性。 SQLA 不会预取这些值，它们也不会在 cursor.lastrowid（DBAPI）中返回。 确保所有后端一致地在 result.inserted_primary_key 中为这些值返回 None-一些后端以前可能返回了一个值。 在主键列上使用 server_default 是极不寻常的。 如果使用特殊函数或 SQL 表达式生成主键默认值，则应将其建立为 Python 端的“默认”而不是 server_default。

关于此情况的反射，具有服务器默认值的 int PK 列的反射将 “autoincrement” 标志设置为 False，但在检测到 PG SERIAL 列的序列默认值的情况下除外。

[#2020](https://www.sqlalchemy.org/trac/ticket/2020) [#2021](https://www.sqlalchemy.org/trac/ticket/2021)

### `sqlalchemy.exceptions` 在 sys.modules 中的别名已被移除。

几年来，我们已经将字符串 `sqlalchemy.exceptions` 添加到 `sys.modules` 中，以便像 “`import sqlalchemy.exceptions`” 这样的语句可以工作。 核心异常模块的名称已经很久以来是 `exc`，因此建议导入此模块的方式是：

```py
from sqlalchemy import exc
```

`exceptions` 名称仍然存在于“`sqlalchemy`”中，供可能已经使用 `from sqlalchemy import exceptions` 的应用程序使用，但他们也应该开始使用 `exc` 名称。

### 查询时间配方更改

虽然不是 SQLAlchemy 本身的一部分，但值得一提的是，将 `ConnectionProxy` 重构为新的事件系统意味着它不再适用于“Timing all Queries”配方。 请调整查询计时器以使用 `before_cursor_execute()` 和 `after_cursor_execute()` 事件，在更新的配方 UsageRecipes/Profiling 中有示例。

## 已弃用的 API

### 类型的默认构造函数不会接受参数。

核心类型模块中的诸如 `Integer`、`Date` 等简单类型不接受参数。 从 0.7b4/0.7.0 开始，接受/忽略 catchall `\*args, \**kwargs` 的默认构造函数已经恢复，但会发出弃用警告。

如果正在使用诸如 `Integer` 等核心类型的参数，可能是您打算使用特定于方言的类型，例如 `sqlalchemy.dialects.mysql.INTEGER`，该类型接受“display_width”参数，例如。

### `compile_mappers()`重命名为`configure_mappers()`，简化配置内部。

这个系统从一个小型、局部实现在单个映射器上，并且命名不当的东西慢慢演变成了更像是全局“注册表”级别函数且命名不当的东西，所以我们通过将实现移出`Mapper`并将其重命名为`configure_mappers()`来修复了这两个问题。当然，一般情况下应用程序通常不需要调用`configure_mappers()`，因为这个过程是根据需要的，一旦通过属性或查询访问需要映射时就会发生。

[#1966](https://www.sqlalchemy.org/trac/ticket/1966)

### 核心监听器/代理被事件监听器取代。

`PoolListener`、`ConnectionProxy`、`DDLElement.execute_at`被`event.listen()`取代，分别使用`PoolEvents`、`EngineEvents`、`DDLEvents`作为分派目标。

### ORM 扩展被事件监听器取代。

`MapperExtension`、`AttributeExtension`、`SessionExtension`被`event.listen()`取代，分别使用`MapperEvents`/`InstanceEvents`、`AttributeEvents`、`SessionEvents`作为分派目标。

### 在 MySQL 中，将字符串发送给 `select()` 的 `distinct` 应通过前缀进行。

这个晦涩的功能允许使用 MySQL 后端的这种模式：

```py
select([mytable], distinct="ALL", prefixes=["HIGH_PRIORITY"])
```

对于非标准或不寻常的前缀，应使用`prefixes`关键字或`prefix_with()`方法：

```py
select([mytable]).prefix_with("HIGH_PRIORITY", "ALL")
```

### `useexisting`被`extend_existing`和`keep_existing`取代。

对表的`useexisting`标志已被新的一对标志`keep_existing`和`extend_existing`取代。`extend_existing`等同于`useexisting` - 返回现有表，并添加额外的构造元素。使用`keep_existing`，返回现有表，但不添加额外的构造元素 - 这些元素仅在表新建时应用。

### 类的默认构造函数不接受参数。

核心类型模块中的简单类型，如`Integer`、`Date`等，不接受参数。接受/忽略通用参数 `\*args, \**kwargs` 的默认构造函数在 0.7b4/0.7.0 版本中已恢复，但会发出弃用警告。

如果在核心类型如`Integer`中使用参数，可能是您打算使用特定于方言的类型，例如`sqlalchemy.dialects.mysql.INTEGER`，例如它接受“display_width”参数。

### `compile_mappers()`重命名为`configure_mappers()`，简化配置内部。

这个系统从一个小型、局部实现在单个映射器上，并且命名不当的东西慢慢演变成了更像是全局“注册表”级别函数且命名不当的东西，所以我们通过将实现移出`Mapper`并将其重命名为`configure_mappers()`来修复了这两个问题。当然，一般情况下应用程序通常不需要调用`configure_mappers()`，因为这个过程是根据需要的，一旦通过属性或查询访问需要映射时就会发生。

[#1966](https://www.sqlalchemy.org/trac/ticket/1966)

### 核心监听器/代理被事件监听器取代

`PoolListener`、`ConnectionProxy`、`DDLElement.execute_at` 被 `event.listen()` 取代，分别使用 `PoolEvents`、`EngineEvents`、`DDLEvents` 分发目标。

### ORM 扩展被事件监听器取代

`MapperExtension`、`AttributeExtension`、`SessionExtension` 被 `event.listen()` 取代，分别使用 `MapperEvents`/`InstanceEvents`、`AttributeEvents`、`SessionEvents`、分发目标。

### 为了在 MySQL 中向 `select()` 中的 ‘distinct’ 发送字符串，应该通过前缀来完成

这个隐晦的特性允许在 MySQL 后端中使用这种模式：

```py
select([mytable], distinct="ALL", prefixes=["HIGH_PRIORITY"])
```

应该使用 `prefixes` 关键字或 `prefix_with()` 方法来处理非标准或不寻常的前缀：

```py
select([mytable]).prefix_with("HIGH_PRIORITY", "ALL")
```

### `useexisting` 被 `extend_existing` 和 `keep_existing` 取代

Table 上的 `useexisting` 标志已被一对新标志 `keep_existing` 和 `extend_existing` 取代。`extend_existing` 等同于 `useexisting` - 返回现有的 Table，并添加额外的构造元素。使用 `keep_existing`，返回现有的 Table，但不添加额外的构造元素 - 这些元素仅在创建新 Table 时应用。

## 向后不兼容的 API 更改

### 传递给 `bindparam()` 的可调用对象不会被评估 - 影响 Beaker 示例

[#1950](https://www.sqlalchemy.org/trac/ticket/1950)

请注意，这会影响 Beaker 缓存示例，其中 `_params_from_query()` 函数的工作方式需要进行轻微调整。如果您正在使用 Beaker 示例中的代码，则应用此更改。

### types.type_map 现在是私有的，types._type_map

我们注意到一些用户在 `sqlalchemy.types` 中利用这个字典作为将 Python 类型与 SQL 类型关联的快捷方式。我们无法保证这个字典的内容或格式，此外，将 Python 类型一对一关联的业务有一些灰色地带，最好由各个应用程序自行决定，因此我们强调了这个属性。

[#1870](https://www.sqlalchemy.org/trac/ticket/1870)

### 将独立 `alias()` 函数的 `alias` 关键字参数重命名为 `name`

这样关键字参数 `name` 就与所有 `FromClause` 对象上的 `alias()` 方法以及 `Query.subquery()` 上的 `name` 参数匹配了。

只有使用独立的 `alias()` 函数的代码，而不是绑定方法函数，并且使用显式关键字名称 `alias` 而不是位置上的别名名称传递的代码需要在这里进行修改。

### 非公共 `Pool` 方法已被下划线标记

所有不打算公开使用的 `Pool` 及其子类方法都已重命名为下划线。之前没有以这种方式命名是一个错误。

现在已经弃用或删除了池化方法：

`Pool.create_connection()` -> `Pool._create_connection()`

`Pool.do_get()` -> `Pool._do_get()`

`Pool.do_return_conn()` -> `Pool._do_return_conn()`

`Pool.do_return_invalid()` -> 已移除，未被使用

`Pool.return_conn()` -> `Pool._return_conn()`

`Pool.get()` -> `Pool._get()`, 公共 API 是 `Pool.connect()`

`SingletonThreadPool.cleanup()` -> `_cleanup()`

`SingletonThreadPool.dispose_local()` -> 已移除，使用 `conn.invalidate()`

[#1982](https://www.sqlalchemy.org/trac/ticket/1982)

### 传递给 `bindparam()` 的可调用对象不会被评估 - 影响 Beaker 示例

[#1950](https://www.sqlalchemy.org/trac/ticket/1950)

请注意，这会影响 Beaker 缓存示例，其中 `_params_from_query()` 函数的工作需要进行轻微调整。如果你正在使用 Beaker 示例中的代码，应用这一变更。

### types.type_map 现在是私有的，types._type_map

我们注意到一些用户在 `sqlalchemy.types` 中利用这个字典作为将 Python 类型与 SQL 类型关联的快捷方式。我们无法保证这个字典的内容或格式，此外，将 Python 类型与 SQL 类型一对一关联的业务有一些灰色地带，最好由各个应用程序自行决定，因此我们已经将这个属性标记为下划线。

[#1870](https://www.sqlalchemy.org/trac/ticket/1870)

### 将独立 `alias()` 函数的 `alias` 关键字参数重命名为 `name`

这样做是为了使关键字参数 `name` 与所有 `FromClause` 对象上的 `alias()` 方法以及 `Query.subquery()` 上的 `name` 参数匹配。

只有使用独立 `alias()` 函数的代码，而不是绑定方法函数，并且使用显式关键字名称 `alias` 而不是位置参数传递别名的代码需要在这里进行修改。

### 非公共 `Pool` 方法已被标记下划线

所有 `Pool` 及其子类的不打算公开使用的方法都已重命名为下划线。它们之前没有以这种方式命名是一个 bug。

池化方法现在已经强调或移除：

`Pool.create_connection()` -> `Pool._create_connection()`

`Pool.do_get()` -> `Pool._do_get()`

`Pool.do_return_conn()` -> `Pool._do_return_conn()`

`Pool.do_return_invalid()` -> 已移除，未被使用

`Pool.return_conn()` -> `Pool._return_conn()`

`Pool.get()` -> `Pool._get()`, 公共 API 是 `Pool.connect()`

`SingletonThreadPool.cleanup()` -> `_cleanup()`

`SingletonThreadPool.dispose_local()` -> 已移除，使用 `conn.invalidate()`

[#1982](https://www.sqlalchemy.org/trac/ticket/1982)

## 之前已弃用，现在已移除

### Query.join()、Query.outerjoin()、eagerload()、eagerload_all() 等不再允许作为参数传递属性列表。

自 0.5 版本以来，将属性或属性名称列表传递给 `Query.join`、`eagerload()` 和类似方法已被弃用：

```py
# old way, deprecated since 0.5
session.query(Houses).join([Houses.rooms, Room.closets])
session.query(Houses).options(eagerload_all([Houses.rooms, Room.closets]))
```

这些方法自 0.5 系列以来都接受 *args：

```py
# current way, in place since 0.5
session.query(Houses).join(Houses.rooms, Room.closets)
session.query(Houses).options(eagerload_all(Houses.rooms, Room.closets))
```

### `ScopedSession.mapper` 已被移除

此功能提供了一个映射器扩展，将基于类的功能与特定的`ScopedSession`关联起来，特别是提供了新对象实例自动与该会话关联的行为。该功能被教程和框架过度使用，由于其隐式行为而导致用户混乱，并在 0.5.5 中被弃用。复制其功能的技术位于[wiki:UsageRecipes/SessionAwareMapper]。

### `Query.join()`、`Query.outerjoin()`、`eagerload()`、`eagerload_all()`等不再接受属性列表作为参数。

自 0.5 版本以来，向`Query.join`、`eagerload()`等传递属性列表或属性名称已被弃用。

```py
# old way, deprecated since 0.5
session.query(Houses).join([Houses.rooms, Room.closets])
session.query(Houses).options(eagerload_all([Houses.rooms, Room.closets]))
```

0.5 系列以来，这些方法都接受`*args`。

```py
# current way, in place since 0.5
session.query(Houses).join(Houses.rooms, Room.closets)
session.query(Houses).options(eagerload_all(Houses.rooms, Room.closets))
```

### `ScopedSession.mapper`已移除。

此功能提供了一个映射器扩展，将基于类的功能与特定的`ScopedSession`关联起来，特别是提供了新对象实例自动与该会话关联的行为。该功能被教程和框架过度使用，由于其隐式行为而导致用户混乱，并在 0.5.5 中被弃用。复制其功能的技术位于[wiki:UsageRecipes/SessionAwareMapper]。
