# SQLAlchemy 0.6 中的新功能是什么？

> 原文：[`docs.sqlalchemy.org/en/20/changelog/migration_06.html`](https://docs.sqlalchemy.org/en/20/changelog/migration_06.html)

关于本文档

本文档描述了 SQLAlchemy 版本 0.5（上次发布于 2010 年 1 月 16 日）与 SQLAlchemy 版本 0.6（上次发布于 2012 年 5 月 5 日）之间的变化。

文档日期：2010 年 6 月 6 日

本指南记录了影响用户将其应用程序从 SQLAlchemy 0.5 系列迁移到 0.6 版本的 API 更改。请注意，SQLAlchemy 0.6 移除了一些在 0.5 系列期间已弃用的行为，并且还弃用了更多与 0.5 版本特定的行为。

## 平台支持

+   cPython 版本 2.4 及以上，遍及 2.xx 系列

+   Jython 2.5.1 - 使用 Jython 随附的 zxJDBC DBAPI。

+   cPython 3.x - 有关如何为 python3 构建的信息，请参见 [source:sqlalchemy/trunk/README.py3k]。

## 新的方言系统

方言模块现在被分解为单个数据库后端范围内的不同子组件。方言实现现在位于 `sqlalchemy.dialects` 包中。`sqlalchemy.databases` 包仍然存在作为占位符，以提供一定程度的向后兼容性，用于简单的导入。

对于每个受支持的数据库，在 `sqlalchemy.dialects` 中都存在一个子包，其中包含几个文件。每个包包含一个名为 `base.py` 的模块，该模块定义了该数据库使用的特定 SQL 方言。它还包含一个或多个“driver”模块，每个模块对应一个特定的 DBAPI - 这些文件的名称与 DBAPI 本身相对应，例如 `pysqlite`、`cx_oracle` 或 `pyodbc`。SQLAlchemy 方言使用的类首先在 `base.py` 模块中声明，定义数据库定义的所有行为特征。这些包括能力映射，例如“支持序列”，“支持返回”等，类型定义和 SQL 编译规则。然后，每个“driver”模块根据需要提供那些类的子类，这些子类覆盖默认行为，以适应该 DBAPI 的附加功能、行为和怪癖。对于支持多个后端的 DBAPI（如 pyodbc、zxJDBC、mxODBC），方言模块将使用来自 `sqlalchemy.connectors` 包的混合物，这些混合物提供了跨所有后端的该 DBAPI 的功能，最常见的是处理连接参数。这意味着使用 pyodbc、zxJDBC 或 mxODBC（当实现时）进行连接在受支持的后端上是非常一致的。

`create_engine()` 使用的 URL 格式已经增强，以处理特定后端的任意数量的 DBAPI，使用的方案受到 JDBC 的启发。以前的格式仍然有效，并且将选择“默认”DBAPI 实现，例如下面的 PostgreSQL URL 将使用 psycopg2：

```
create_engine("postgresql://scott:tiger@localhost/test")
```

但是，要指定特定的 DBAPI 后端，比如 pg8000，请将其添加到 URL 的“protocol”部分，使用加号“+”：

```
create_engine("postgresql+pg8000://scott:tiger@localhost/test")
```

重要的方言链接：

+   连接参数的文档：[`www.sqlalchemy.org/docs/06/dbengine.html#create`](https://www.sqlalchemy.org/docs/06/dbengine.html#create)- engine-url-arguments。

+   个别方言的参考文档：[`ww`](https://ww) w.sqlalchemy.org/docs/06/reference/dialects/index.html

+   DatabaseNotes 上的技巧和窍门。

关于方言的其他注意事项：

+   SQLAlchemy 0.6 中类型系统发生了巨大变化。这对所有方言都有影响，包括命名约定、行为和实现。请参阅下面关于“类型”的部分。

+   `ResultProxy`对象现在在某些情况下提供了 2 倍的速度改进，这要归功于一些重构。

+   `RowProxy`，即单个结果行对象，现在可以直接进行 pickle。

+   setuptools 入口点现在用于定位外部方言的名称是`sqlalchemy.dialects`。针对 0.4 或 0.5 编写的外部方言需要修改以适应 0.6，在任何情况下这个改变并不增加任何额外的困难。

+   方言现在在初始连接时接收一个 initialize()事件来确定连接属性。

+   编译器生成的函数和操作符现在使用（几乎）常规的调度函数形式“visit_<opname>”和“visit_<funcname>_fn”来提供定制处理。这取代了在编译器子类中复制“functions”和“operators”字典的需要，改为使用直接的访问者方法，同时也允许编译器子类完全控制渲染，因为完整的 _Function 或 _BinaryExpression 对象被传递进来。

### 方言导入

方言的导入结构已经改变。每个方言现在通过`sqlalchemy.dialects.<name>`导出其基本的“dialect”类以及该方言支持的完整一组 SQL 类型。例如，要导入一组 PG 类型：

```
from sqlalchemy.dialects.postgresql import (
    INTEGER,
    BIGINT,
    SMALLINT,
    VARCHAR,
    MACADDR,
    DATE,
    BYTEA,
)
```

在上面，`INTEGER`实际上是来自`sqlalchemy.types`的普通`INTEGER`类型，但 PG 方言使其以与那些特定于 PG 的类型相同的方式可用，例如`BYTEA`和`MACADDR`。

## 表达式语言变化

### 一个重要的表达式语言陷阱

表达式语言有一个相当重要的行为变化，可能会影响一些应用程序。Python 布尔表达式的布尔值，即`==`、`!=`等，现在在比较两个子句对象时会准确评估。

正如我们所知，将`ClauseElement`与任何其他对象进行比较会返回另一个`ClauseElement`：

```
>>> from sqlalchemy.sql import column
>>> column("foo") == 5
<sqlalchemy.sql.expression._BinaryExpression object at 0x1252490>
```

这样 Python 表达式在转换为字符串时会产生 SQL 表达式：

```
>>> str(column("foo") == 5)
'foo = :foo_1'
```

但是如果我们这样说会发生什么？

```
>>> if column("foo") == 5:
...     print("yes")
```

在之前的 SQLAlchemy 版本中，返回的`_BinaryExpression`是一个普通的 Python 对象，其求值为`True`。现在它的求值取决于实际的`ClauseElement`是否应该具有与被比较的相同哈希值。意思是：

```
>>> bool(column("foo") == 5)
False
>>> bool(column("foo") == column("foo"))
False
>>> c = column("foo")
>>> bool(c == c)
True
>>>
```

这意味着如下代码：

```
if expression:
    print("the expression is:", expression)
```

如果 `expression` 是二进制子句，则不会被评估。由于上述模式永远不应该被使用，因此基本的 `ClauseElement` 现在在布尔上下文中调用时会引发异常：

```
>>> bool(c)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  ...
  raise TypeError("Boolean value of this clause is not defined")
TypeError: Boolean value of this clause is not defined
```

希望检查 `ClauseElement` 表达式是否存在的代码应该这样说：

```
if expression is not None:
    print("the expression is:", expression)
```

请记住，**这也适用于 Table 和 Column 对象**。

更改的理由有两个：

+   形如 `if c1 == c2:  <do something>` 的比较现在可以这样写了。

+   现在支持在其他平台（即 Jython）上正确对 `ClauseElement` 对象进行哈希处理。直到这一点，SQLAlchemy 在这方面严重依赖 cPython 的特定行为（并且仍然偶尔出现问题）。

### 更严格的“executemany”行为

SQLAlchemy 中的“executemany”对应于调用 `execute()`，传递一系列绑定参数集：

```
connection.execute(table.insert(), {"data": "row1"}, {"data": "row2"}, {"data": "row3"})
```

当 `Connection` 对象发送给定的 `insert()` 构造进行编译时，它传递给编译器传递的第一组绑定中存在的键名，以确定语句的 VALUES 子句的构造。熟悉这种构造的用户会知道，剩余字典中存在的额外键没有任何影响。现在的不同之处在于，所有后续字典都需要至少包含第一个字典中存在的*每个*键。这意味着像这样的调用不再起作用：

```
connection.execute(
    table.insert(),
    {"timestamp": today, "data": "row1"},
    {"timestamp": today, "data": "row2"},
    {"data": "row3"},
)
```

因为第三行没有指定`timestamp`列。之前的 SQLAlchemy 版本会简单地为这些缺失的列插入 NULL。然而，如果上面示例中的 `timestamp` 列包含 Python 端的默认值或函数，则*不会*被使用。这是因为“executemany”操作针对大量参数集的最大性能进行了优化，并且不会尝试评估那些缺失键的 Python 端默认值。由于默认值通常被实现为嵌入在 INSERT 语句中的 SQL 表达式，或者是服务器端表达式，再次根据 INSERT 字符串的结构触发，这些默认值无法根据每个参数集有条件地触发，因此 Python 端默认值与 SQL/服务器端默认值的行为不同是不一致的。（基于 SQL 表达式的默认值从 0.5 系列开始嵌入在内部，再次为了最小化大量参数集的影响）。

因此，SQLAlchemy 0.6 通过禁止任何后续参数集留下任何字段空白来建立可预测的一致性。这样，Python 端默认值和函数就不会再默默失败，而且它们的行为与 SQL 和服务器端默认值保持一致。

### UNION 和其他“复合”结构都有一致的括号配对。

为了帮助 SQLite 而设计的规则已被移除，即在另一个复合元素内的第一个复合元素（例如，在`except_()`内部的`union()`）不会被括号括起来。这是不一致的，并且在 PostgreSQL 上产生错误的结果，因为它对 INTERSECTION 有优先规则，这通常会让人感到惊讶。在与 SQLite 使用复杂复合时，现在需要将第一个元素转换为子查询（这也在 PG 上兼容）。在 SQL 表达式教程的最后有一个新的示例[[`www.sqlalchemy.org/docs/06/sqlexpression.html`](https://www.sqlalchemy.org/docs/06/sqlexpression.html) #unions-and-other-set-operations]。更多背景信息请参见[#1665](https://www.sqlalchemy.org/trac/ticket/1665)和 r6690。

## 用于结果提取的 C 扩展

`ResultProxy`及其相关元素，包括大多数常见的“行处理”函数，如 Unicode 转换、数值/布尔转换和日期解析，已经被重新实现为可选的 C 扩展，以提高性能。这标志着 SQLAlchemy 走向“黑暗面”的开始，我们希望通过在 C 中重新实现关键部分来持续改进性能。可以通过指定`--with-cextensions`来构建这些扩展，即`python setup.py --with- cextensions install`。

这些扩展对使用直接`ResultProxy`访问的结果提取影响最为显著，即由`engine.execute()`、`connection.execute()`或`session.execute()`返回的结果。在 ORM `Query`对象返回的结果中，结果提取不占很高的开销比例，因此 ORM 性能改善较为适度，主要体现在提取大型结果集方面。性能改进高度依赖于使用的 dbapi 以及访问每行列的语法（例如，`row['name']`比`row.name`快得多）。当前的扩展对插入/更新/删除的速度没有影响，也不会改善 SQL 执行的延迟，也就是说，一个大部分时间用于执行许多语句且结果集非常小的应用程序不会看到太多改进。

与扩展无关，0.6 版本的性能比 0.5 版本有所提高。使用 SQLite 连接和提取 50000 行的快速概述，主要使用直接的 SQLite 访问、`ResultProxy`和简单的映射 ORM 对象：

```
sqlite select/native: 0.260s

0.6 / C extension

sqlalchemy.sql select: 0.360s
sqlalchemy.orm fetch: 2.500s

0.6 / Pure Python

sqlalchemy.sql select: 0.600s
sqlalchemy.orm fetch: 3.000s

0.5 / Pure Python

sqlalchemy.sql select: 0.790s
sqlalchemy.orm fetch: 4.030s
```

在上述情况下，ORM 由于 Python 中的性能增强，比 0.5 版本快 33%。使用 C 扩展，我们又获得了 20%的提升。然而，与不使用 C 扩展相比，`ResultProxy`的提取速度提高了 67%。其他测试报告显示，在某些场景中，如发生大量字符串转换的情况下，速度提高了多达 200%。

## 新的模式功能

`sqlalchemy.schema`包得到了一些长期需要的关注。最显著的变化是新扩展的 DDL 系统。在 SQLAlchemy 中，自版本 0.5 以来，可以创建自定义的 DDL 字符串并将其与表或元数据对象关联：

```
from sqlalchemy.schema import DDL

DDL("CREATE TRIGGER users_trigger ...").execute_at("after-create", metadata)
```

现在，完整的 DDL 构造都在同一系统下可用，包括用于 CREATE TABLE、ADD CONSTRAINT 等的构造：

```
from sqlalchemy.schema import Constraint, AddConstraint

AddContraint(CheckConstraint("value > 5")).execute_at("after-create", mytable)
```

此外，所有 DDL 对象现在都是常规的`ClauseElement`对象，就像任何其他 SQLAlchemy 表达式对象一样：

```
from sqlalchemy.schema import CreateTable

create = CreateTable(mytable)

# dumps the CREATE TABLE as a string
print(create)

# executes the CREATE TABLE statement
engine.execute(create)
```

并且使用`sqlalchemy.ext.compiler`扩展，你可以自己制作：

```
from sqlalchemy.schema import DDLElement
from sqlalchemy.ext.compiler import compiles

class AlterColumn(DDLElement):
    def __init__(self, column, cmd):
        self.column = column
        self.cmd = cmd

@compiles(AlterColumn)
def visit_alter_column(element, compiler, **kw):
    return "ALTER TABLE %s ALTER COLUMN %s  %s ..." % (
        element.column.table.name,
        element.column.name,
        element.cmd,
    )

engine.execute(AlterColumn(table.c.mycolumn, "SET DEFAULT 'test'"))
```

### 废弃/移除的模式元素

模式包也经过了大幅简化。在整个 0.5 版本中被废弃的许多选项和方法已被移除。其他鲜为人知的访问器和方法也已被移除。

+   “owner”关键字参数从`Table`中移除。使用“schema”表示要预置到表名之前的任何命名空间。

+   废弃的`MetaData.connect()`和`ThreadLocalMetaData.connect()`已被移除 - 将“bind”属性发送到绑定元数据。

+   废弃的 metadata.table_iterator()方法已移除（使用 sorted_tables）

+   从`DefaultGenerator`和子类中移除了“metadata”参数，但在`Sequence`上仍然本地存在，`Sequence`是 DDL 中的一个独立构造。

+   废弃的`PassiveDefault` - 使用`DefaultClause`。

+   从`Index`和`Constraint`对象中移除了公共可变性：

    +   `ForeignKeyConstraint.append_element()`

    +   `Index.append_column()`

    +   `UniqueConstraint.append_column()`

    +   `PrimaryKeyConstraint.add()`

    +   `PrimaryKeyConstraint.remove()`

这些应该以声明方式构建（即一次构建）。

+   其他已移除的内容：

    +   `Table.key`（不知道这是干什么的）

    +   `Column.bind`（通过 column.table.bind 获取）

    +   `Column.metadata`（通过 column.table.metadata 获取）

    +   `Column.sequence`（使用 column.default）

### 其他行为变化

+   `UniqueConstraint`，`Index`，`PrimaryKeyConstraint`都接受列名或列对象的列表作为参数。

+   `ForeignKey`上的`use_alter`标志现在是一个快捷选项，用于可以使用`DDL()`事件系统手动构建的操作。这次重构的一个副作用是，具有`use_alter=True`的`ForeignKeyConstraint`对象将不会在 SQLite 上发出，因为 SQLite 不支持外键的 ALTER。这对 SQLite 的行为没有影响，因为 SQLite 实际上不遵守外键约束。

+   `Table.primary_key`不可分配 - 使用`table.append_constraint(PrimaryKeyConstraint(...))`

+   具有`ForeignKey`但没有类型的`Column`定义，例如`Column(name, ForeignKey(sometable.c.somecol))`曾经可以获得引用列的类型。现在对于该自动类型推断的支持是部分的，可能不适用于所有情况。

## 日志打开了

通过在引擎、池或映射器创建后进行一些额外的方法调用，您可以设置 INFO 和 DEBUG 的日志级别，日志记录将开始。现在，isEnabledFor(INFO)方法将每个 Connection 调用一次，isEnabledFor(DEBUG)将每个 ResultProxy 调用一次，如果已在父连接上启用。池日志发送到 log.info()和 log.debug()，无需检查 - 请注意，池的检出/归还通常是每个事务一次。

## 反射/检查器 API

反射系统，允许通过 Table('sometable', metadata, autoload=True)反射表列的系统已经开放为自己的细粒度 API，允许直接检查数据库元素，如表、列、约束、索引等。这个 API 将返回值表达为简单的字符串列表、字典和 TypeEngine 对象。现在，autoload=True 的内部构建在这个系统之上，将原始数据库信息转换为 sqlalchemy.schema 构造集中化，各个方言的契约大大简化，大大减少了不同后端之间的错误和不一致性。

要使用检查器：

```
from sqlalchemy.engine.reflection import Inspector

insp = Inspector.from_engine(my_engine)

print(insp.get_schema_names())
```

在某些情况下，from_engine()方法将提供一个特定于后端的检查器，具有额外的功能，例如 PostgreSQL 提供了一个 get_table_oid()方法：

```
my_engine = create_engine("postgresql://...")
pg_insp = Inspector.from_engine(my_engine)

print(pg_insp.get_table_oid("my_table"))
```

## RETURNING 支持

insert()、update()和 delete()构造现在支持一个 returning()方法，对应于 SQL RETURNING 子句，支持 PostgreSQL、Oracle、MS-SQL 和 Firebird。目前不支持其他后端。

给定一个与 select()构造相同方式的列表达式列表，这些列的值将作为常规结果集返回：

```
result = connection.execute(
    table.insert().values(data="some data").returning(table.c.id, table.c.timestamp)
)
row = result.first()
print("ID:", row["id"], "Timestamp:", row["timestamp"])
```

四个支持后端的 RETURNING 实现差异很大，Oracle 需要复杂使用 OUT 参数，这些参数被重新路由到一个“模拟”结果集中，而 MS-SQL 使用笨拙的 SQL 语法。RETURNING 的使用受到限制：

+   它不适用于任何“executemany()”风格的执行。这是所有支持的 DBAPI 的限制。

+   一些后端，比如 Oracle，只支持返回单行的 RETURNING - 这包括 UPDATE 和 DELETE 语句，这意味着 update()或 delete()构造必须仅匹配单行，否则会引发错误（由 Oracle 而不是 SQLAlchemy 引发）。

当单行 INSERT 语句需要获取新生成的主键值时，SQLAlchemy 也会自动使用 RETURNING，当没有通过显式的`returning()`调用另行指定时。这意味着在需要主键值的插入语句中不再需要“SELECT nextval(sequence)”预执行。说实话，隐式 RETURNING 功能确实比旧的“select nextval()”系统产生更多的方法开销，后者使用快速而简单的 cursor.execute()来获取序列值，并且在 Oracle 的情况下需要额外绑定输出参数。因此，如果方法/协议开销比额外的数据库往返更昂贵，可以通过在`create_engine()`中指定`implicit_returning=False`来禁用该功能。

## 类型系统更改

### 新架构

类型系统在幕后完全重建，以实现两个目标：

+   将绑定参数和结果行值的处理分开，通常是 DBAPI 的要求，与类型本身的 SQL 规范分开，这是与总体方言重构一致的，将数据库 SQL 行为与 DBAPI 分开。

+   为从`TypeEngine`对象生成 DDL 和基于列反射构造`TypeEngine`对象建立明确一致的契约。

这些变化的亮点包括：

+   方言中类型的构建已经彻底改变。方言现在专门将公开可用的类型定义为大写名称，并使用下划线标识符（即私有）来定义内部实现类型。用于在 SQL 和 DDL 中表达类型的系统已移至编译器系统。这样做的效果是大多数方言中的类型对象要少得多。有关此架构的详细文档供方言作者参考在[source:/lib/sqlalchemy/dialects/type_migration_guidelines.txt]中。

+   现在类型的反射返回 types.py 中的确切大写类型，或者如果类型不是标准 SQL 类型，则返回方言本身中的大写类型。这意味着反射现在返回有关反射类型的更准确信息。

+   用户定义的类型，如果要提供`get_col_spec()`方法，现在应该继承`UserDefinedType`。

+   所有类型类上的`result_processor()`方法现在接受额外的参数`coltype`。这是附加到 cursor.description 的 DBAPI 类型对象，应在适用时使用，以便更好地决定应返回什么类型的结果处理可调用函数。理想情况下，结果处理函数永远不应该使用`isinstance()`，因为在这个级别上这是一个昂贵的调用。

### 本地 Unicode 模式

随着更多的 DBAPI 支持直接返回 Python Unicode 对象，基本方言现在在第一次连接时执行检查，以确定 DBAPI 是否为基本的 VARCHAR 值的基本选择返回 Python Unicode 对象。如果是这样，`String`类型和所有子类（即`Text`，`Unicode`等）将在接收到结果行时跳过“unicode”检查/转换步骤。这为大型结果集提供了显著的性能提升。目前已知“unicode 模式”可以与以下一起使用：

+   sqlite3 / pysqlite

+   psycopg2 - SQLA 0.6 现在默认在每个 psycopg2 连接对象上使用“UNICODE”类型扩展

+   pg8000

+   cx_oracle（我们使用输出处理器 - 很好的功能！）

其他类型可能会根据需要禁用 Unicode 处理，例如在与 MS-SQL 一起使用时的`NVARCHAR`类型。

特别是，如果迁移基于以前返回非 Unicode 字符串的 DBAPI 的应用程序，则“本地 Unicode”模式具有明显不同的默认行为 - 声明为`String`或`VARCHAR`的列现在默认返回 Unicode，而以前会返回字符串。这可能会破坏期望非 Unicode 字符串的代码。可以通过向`create_engine()`传递`use_native_unicode=False`来禁用 psycopg2 的“本地 Unicode”模式。

一个更通用的解决方案是针对明确不想要 Unicode 对象的字符串列使用`TypeDecorator`，将 Unicode 转换回 utf-8，或者其他所需的格式：

```
class UTF8Encoded(TypeDecorator):
  """Unicode type which coerces to utf-8."""

    impl = sa.VARCHAR

    def process_result_value(self, value, dialect):
        if isinstance(value, unicode):
            value = value.encode("utf-8")
        return value
```

请注意，`assert_unicode`标志现已弃用。SQLAlchemy 允许 DBAPI 和后端数据库在可用时处理 Unicode 参数，并且不会通过检查传入类型增加操作开销；现代系统如 sqlite 和 PostgreSQL 会在其端引发编码错误，如果传递了无效数据。在 SQLAlchemy 确实需要将绑定参数从 Python Unicode 强制转换为编码字符串时，或者显式使用 Unicode 类型时，如果对象是字节串，则会发出警告。可以使用 Python 警告过滤器文档中记录的警告来抑制或将其转换为异常：[`docs.python.org/library/warnings.html`](https://docs.python.org/library/warnings.html)

### 通用枚举类型

现在在 `types` 模块中有一个 `Enum`。这是一个字符串类型，给定一组“标签”，限制了给这些标签赋予的可能值。默认情况下，该类型生成一个`VARCHAR`，使用最大标签的大小，并在 CREATE TABLE 语句中对表应用 CHECK 约束。当使用 MySQL 时，默认情况下，该类型使用 MySQL 的 ENUM 类型；当使用 PostgreSQL 时，该类型将使用 `CREATE TYPE <mytype> AS ENUM` 生成用户定义类型。为了使用 PostgreSQL 创建类型，必须在构造函数中指定 `name` 参数。该类型还接受一个 `native_enum=False` 选项，该选项将为所有数据库发出 VARCHAR/CHECK 策略。请注意，当前 PostgreSQL ENUM 类型不能与 pg8000 或 zxjdbc 一起使用。

### 反射返回方言特定类型

反射现在从数据库返回尽可能最具体的类型。也就是说，如果您使用 `String` 创建一个表，然后反射它，那么反射的列可能是 `VARCHAR`。对于支持更特定形式类型的方言，您将得到相应的类型。因此，在 Oracle 上，`Text` 类型将返回为 `oracle.CLOB`，`LargeBinary` 可能是 `mysql.MEDIUMBLOB` 等等。这里的明显优势是反射尽可能地保留了数据库要说的信息。

一些处理表元数据的应用程序可能希望比较反映的表和/或非反射的表上的类型。`TypeEngine` 上有一个半私有访问器叫做 `_type_affinity`，以及一个相关的比较助手 `_compare_type_affinity`。此访问器返回类型对应的“通用” `types` 类：

```
>>> String(50)._compare_type_affinity(postgresql.VARCHAR(50))
True
>>> Integer()._compare_type_affinity(mysql.REAL)
False
```

### 杂项 API 更改

通常的“通用”类型仍然是正在使用的一般系统，即 `String`、`Float`、`DateTime`。在那里有一些变化：

+   类型不再猜测默认参数。特别是，`Numeric`、`Float`，以及 NUMERIC、FLOAT、DECIMAL 的子类不生成长度或比例，除非指定。这也包括有争议的 `String` 和 `VARCHAR` 类型（尽管 MySQL 方言在要求不带长度渲染 VARCHAR 时会预先引发）。不假设默认值，如果它们在 CREATE TABLE 语句中使用，则在底层数据库不允许这些类型的非长度版本时会引发错误。

+   `Binary` 类型已更名为 `LargeBinary`，用于 BLOB/BYTEA/类似类型。对于 `BINARY` 和 `VARBINARY`，它们直接存在于 `types.BINARY`、`types.VARBINARY`，以及 MySQL 和 MS-SQL 方言中。

+   当 mutable=True 时，`PickleType` 现在使用 == 比较值，除非为该类型指定了带有比较函数的“comparator”参数。如果您要 pickle 自定义对象，应该实现一个 `__eq__()` 方法，以便基于值的比较准确。

+   Numeric 和 Float 的默认“precision”和“scale”参数已被移除，并且现在默认为 None。NUMERIC 和 FLOAT 默认情况下将不带有数值参数呈现，除非提供了这些值。

+   SQLite 上的 DATE、TIME 和 DATETIME 类型现在可以接受可选的“storage_format”和“regexp”参数。“storage_format”可用于使用自定义字符串格式存储这些类型。“regexp”允许使用自定义正则表达式来匹配数据库中的字符串值。

+   不再支持 SQLite 上 `Time` 和 `DateTime` 类型的 `__legacy_microseconds__`。你应该使用新的 “storage_format” 参数。

+   SQLite 上的 `DateTime` 类型现在默认使用更严格的正则表达式来匹配数据库中的字符串。如果你使用存储在传统格式中的数据，请使用新的 “regexp” 参数。

## ORM 变更

从 0.5 升级到 0.6 的 ORM 应用应该几乎不需要更改，因为 ORM 的行为基本保持不变。有一些默认参数和名称更改，以及一些加载行为已经改进。

### 新工作单元

工作单元的内部，主要是 `topological.py` 和 `unitofwork.py`，已经完全重写并且大大简化。这对使用没有影响，因为所有现有的行为在 flush 过程中都被完全保持了下来（或者至少在我们的测试套件和少数重度测试的生产环境中被保持了下来）。flush() 的性能现在减少了 20-30% 的方法调用，并且应该使用更少的内存。现在，源代码的意图和流程应该相当容易理解，而且 flush 的架构在这一点上相当开放，为潜在的新领域创造了空间。flush 过程不再依赖递归，因此可以刷新任意大小和复杂度的 flush 计划。此外，mapper 的 “save” 过程，发出 INSERT 和 UPDATE 语句，现在缓存了两个语句的 “compiled” 形式，因此在非常大的 flush 过程中进一步大幅减少了调用次数。

与早期版本 0.6 或 0.5 相比，在 flush 与 flush 之间观察到的任何行为变化都应该尽快向我们报告 - 我们将确保不会丢失任何功能。

### `query.update()` 和 `query.delete()` 的变更

+   query.update() 上的 ‘expire’ 选项已更名为 ‘fetch’，因此与 query.delete() 的匹配项相匹配

+   `query.update()` 和 `query.delete()` 的 synchronize 策略都默认为 ‘evaluate’。

+   ‘synchronize’ 策略对 update() 和 delete() 抛出错误时会触发错误。在失败时没有隐式回退到“fetch”。评估的失败基于条件的结构，因此成功/失败是基于代码结构确定性的。

### `relation()` 现在正式命名为 `relationship()`

这是为了解决长期存在的问题，“relation”在关系代数术语中意味着“表或派生表”。`relation()`名称，少打字，将在可预见的未来继续存在，因此这个改变应该完全没有痛苦。

### 子查询的急切加载

添加了一种新的急切加载方式，称为“subquery”加载。这是一种在第一个 SQL 查询之后立即发出第二个 SQL 查询的加载方式，为第一个查询中的所有父级加载完整集合，使用 INNER JOIN 向上连接到父级。子查询加载类似于当前的连接急切加载，使用`subqueryload()`、`subqueryload_all()`选项，以及设置在`relationship()`上的`lazy='subquery'`。子查询加载通常比较高效，用于加载许多较大的集合，因为它无条件地使用 INNER JOIN，而且也不会重新加载父行。

### `eagerload()`, `eagerload_all()`现在是`joinedload()`, `joinedload_all()`

为了为新的子查询加载功能腾出空间，现有的`eagerload()`/`eagerload_all()` options are now superseded by `joinedload()` and `joinedload_all()`. The old names will hang around for the foreseeable future just like `relation()`。

### `lazy=False|None|True|'dynamic'`现在接受`lazy='noload'|'joined'|'subquery'|'select'|'dynamic'`

在继续开放加载器策略的主题上，`relationship()`上的标准关键字`lazy`选项现在是，用于延迟加载的`select`（通过属性访问时发出的 SELECT），用于急切连接加载的`joined`，用于急切子查询加载的`subquery`，不应出现任何负载的`noload`，以及用于“动态”关系的`dynamic`。旧的`True`, `False`, `None`参数仍然被接受，行为与以前完全相同。

### 在关系、joinedload 上设置 innerjoin=True

现在可以指示连接急切加载的标量和集合使用 INNER JOIN 而不是 OUTER JOIN。在 PostgreSQL 上观察到这可以在某些查询上提供 300-600%的速度提升。为任何在 NOT NULLable 外键上的多对一设置此标志，以及对于任何保证存在相关项目的集合。

在映射器级别：

```
mapper(Child, child)
mapper(
    Parent,
    parent,
    properties={"child": relationship(Child, lazy="joined", innerjoin=True)},
)
```

在查询时间级别：

```
session.query(Parent).options(joinedload(Parent.child, innerjoin=True)).all()
```

在`relationship()`级别设置`innerjoin=True`标志也将对任何不覆盖该值的`joinedload()`选项生效。

### 多对一增强

+   多对一关系现在在更少的情况下会触发延迟加载，包括在大多数情况下不会在替换新值时获取“旧”值。

+   多对一关系到一个连接表子类现在使用 get()进行简单加载（称为“use_get”条件），即`Related`->``Sub(Base)``，无需重新定义基表的 primaryjoin 条件。[ticket:1186]

+   使用声明性列指定外键，即`ForeignKey(MyRelatedClass.id)`不会导致“use_get”条件发生变化 [ticket:1492]

+   relationship()，joinedload()和 joinedload_all()现在具有一个名为“innerjoin”的选项。指定`True`或`False`来控制是否构建内连接或外连接的预加载连接。默认始终为`False`。映射器选项将覆盖在 relationship()上指定的任何设置。通常应该为多对一、非空外键关系设置���以允许改进的连接性能。[ticket:1544]

+   当存在 LIMIT/OFFSET 时，连接式预加载的行为会将主查询包装在子查询中，现在对所有预加载都是多对一连接的情况做了一个例外。在这些情况下，预加载连接直接针对父表进行，同时包括限制/偏移，而不需要额外的子查询开销，因为多对一连接不会向结果添加行。

    例如，在 0.5 中，这个查询：

    ```
    session.query(Address).options(eagerload(Address.user)).limit(10)
    ```

    会生成类似于以下的 SQL：

    ```
    SELECT  *  FROM
      (SELECT  *  FROM  addresses  LIMIT  10)  AS  anon_1
      LEFT  OUTER  JOIN  users  AS  users_1  ON  users_1.id  =  anon_1.addresses_user_id
    ```

    这是因为任何预加载的存在都暗示着其中一些或全部可能与多行集合相关联，这将需要将任何类似于 LIMIT 这样的行数敏感修饰符包装在子查询中。

    在 0.6 中，该逻辑更加敏感，可以检测到所有预加载是否都表示多对一关系，如果是这种情况，预加载连接不会影响行数：

    ```
    SELECT  *  FROM  addresses  LEFT  OUTER  JOIN  users  AS  users_1  ON  users_1.id  =  addresses.user_id  LIMIT  10
    ```

### 具有联接表继承的可变主键

在子表具有外键指向父表主键的联接表继承配置现在可以在像 PostgreSQL 这样支持级联的数据库上更新。`mapper()`现在有一个选项`passive_updates=True`，表示此外键将自动更新。如果在不支持级联的数据库上，如 SQLite 或 MySQL/MyISAM 上，将此标志设置为`False`。未来的功能增强将尝试根据使用的方言/表格样式自动配置此标志。

### Beaker 缓存

Beaker 集成的一个有前途的新示例在`examples/beaker_caching`中。这是一个简单的示例，它在`Query`的结果生成引擎中应用了一个 Beaker 缓存。缓存参数通过`query.options()`提供，并允许完全控制缓存内容。SQLAlchemy 0.6 对`Session.merge()`方法进行了改进，以支持这种和类似的示例，并在大多数情况下提供了显著改进的性能。

### 其他更改

+   当选择多个列/实体时，`Query`返回的“行元组”对象现在也是可序列化的，并且性能更高。

+   `query.join()`已经重新设计，以提供更一致的行为和更灵活的功能（包括[ticket:1537]）

+   `query.select_from()`接受多个子句，以在 FROM 子句中生成多个逗号分隔的条目。在从多个 join()子句中选择时很有用。

+   `Session.merge()`上的“dont_load=True”标志已被弃用，现在是“load=False”。

+   添加了“make_transient()”辅助函数，将持久化/分离实例转换为瞬态实例（即删除实例键并从任何会话中移除）。[ticket:1052]

+   mapper() 上的 allow_null_pks 标志已被废弃，并已重命名为 allow_partial_pks。默认情况下已打开此标志。这意味着对于任何主键列中有非空值的行将被视为标识。通常只有在映射到外连接时才需要此情景。当设置为 False 时，具有 NULL 值的 PK 不会被视为主键 - 特别是这意味着结果行将返回为 None（或不会填充到集合中），并且在 0.6 中还表示 session.merge() 不会为此类 PK 值发出数据库的往返。【票号：1680】

+   “backref”的机制已完全合并到更精细的 “back_populates” 系统中，并完全在 `RelationProperty` 的 `_generate_backref()` 方法中进行。这使得 `RelationProperty` 的初始化过程更简单，并允许更容易地将设置（如 `RelationProperty` 的子类）传播到反向引用中。内部的 `BackRef()` 已经消失，`backref()` 返回一个普通元组，被 `RelationProperty` 理解。

+   `ResultProxy` 的 keys 属性现在是一个方法，因此对它的引用（`result.keys`）必须更改为方法调用（`result.keys()`）。

+   `ResultProxy.last_inserted_ids` 现在已废弃，使用 `ResultProxy.inserted_primary_key` 替代。

### 废弃/移除的 ORM 元素

在 0.5 版本中废弃并引发废弃警告的大多数元素已被移除（有几个例外）。所有标记为 “待废弃” 的元素现在已被废弃，并在使用时引发警告。

+   sessionmaker() 和其他地方上的 ‘transactional’ 标志已移除。使用 ‘autocommit=True’ 表示 ‘transactional=False’。

+   mapper() 上的 ‘polymorphic_fetch’ 参数已移除。可以使用 ‘with_polymorphic’ 选项来控制加载。

+   mapper() 上的 ‘select_table’ 参数已移除。使用 ‘with_polymorphic=(“*”, <some selectable>)’ 实现此功能。

+   synonym() 上的 ‘proxy’ 参数已移除。在 0.5 版本中此标志没有任何作用，因为 “代理生成” 行为现在是自动的。

+   将单个元素列表传递给 joinedload()、joinedload_all()、contains_eager()、lazyload()、defer() 和 undefer() 而不是多个位置 *args 已被废弃。

+   将单个元素列表传递给 query.order_by()、query.group_by()、query.join() 或 query.outerjoin() 而不是多个位置 *args 已被废弃。

+   `query.iterate_instances()` 被移除了。使用 `query.instances()`。

+   `Query.query_from_parent()` 被移除了。使用 sqlalchemy.orm.with_parent() 函数生成 “parent” 子句，或者使用 `query.with_parent()`。

+   `query._from_self()` 被移除，使用 `query.from_self()` 代替。

+   composite() 方法的 “comparator” 参数被移除了。使用 “comparator_factory”。

+   `RelationProperty._get_join()` 已移除。

+   Session 上的 ‘echo_uow’ 标志已移除。在 “sqlalchemy.orm.unitofwork” 名称上使用日志记录。

+   `session.clear()` 已移除。使用 `session.expunge_all()`。

+   `session.save()`，`session.update()`，`session.save_or_update()` 已移除。使用 `session.add()` 和 `session.add_all()`。

+   在 `session.flush()` 上的 “objects” 标志仍然被弃用。

+   在 `session.merge()` 上的 “dont_load=True” 标志已弃用，建议使用 “load=False”。

+   `ScopedSession.mapper` 仍然被弃用。请参阅[用法](https://www.sqlalchemy.org/trac/wiki/Usag)配方在 Recipes/SessionAwareMapper

+   将 `InstanceState`（内部 SQLAlchemy 状态对象）传递给 `attributes.init_collection()` 或 `attributes.get_history()` 已弃用。 这些函数是公共 API，并且通常希望是常规映射对象实例。

+   `declarative_base()` 的 “engine” 参数已移除。使用 “bind” 关键字参数。

## 扩展

### SQLSoup

SQLSoup 已现代化并更新以反映常见的 0.5/0.6 功能，包括明确定义的会话集成。请阅读新文档[[`www.sqlalc`](https://www.sqlalc) hemy.org/docs/06/reference/ext/sqlsoup.html]。

### 声明

`DeclarativeMeta`（`declarative_base` 的默认元类）之前允许子类修改 `dict_` 来添加类属性（例如列）。 这种方式不再有效，`DeclarativeMeta` 构造函数现在忽略 `dict_`。相反，类属性应直接赋值，例如 `cls.id=Column(...)`，或者应该使用 [MixIn 类](https://www.sqlalchemy.org/docs/reference/ext/declarative.html#mix-in-classes) 方法而不是元类方法。

## 平台支持

+   cPython 版本从 2.4 开始，在 2.xx 系列中

+   Jython 2.5.1 - 使用 Jython 自带的 zxJDBC DBAPI。

+   cPython 3.x - 参见[源码:sqlalchemy/trunk/README.py3k] 了解如何构建 Python3 版本。

## 新方言系统

方言模块现在被分解为单个数据库后端范围内的不同子组件。 方言实现现在在 `sqlalchemy.dialects` 包中。 `sqlalchemy.databases` 包仍然存在，作为一个占位符，为简单导入提供一定程度的向后兼容性。

对于每个支持的数据库，在`sqlalchemy.dialects`中都存在一个子包，其中包含几个文件。每个包都包含一个名为`base.py`的模块，该模块定义了该数据库使用的特定 SQL 方言。它还包含一个或多个“driver”模块，每个模块对应于特定的 DBAPI - 这些文件的命名与 DBAPI 本身相对应，例如`pysqlite`、`cx_oracle`或`pyodbc`。SQLAlchemy 方言使用的类首先在`base.py`模块中声明，定义了数据库定义的所有行为特征。这些包括功能映射，例如“支持序列”，“支持返回”等，类型定义和 SQL 编译规则。每个“driver”模块依次提供所需的那些类的子类，这些子类覆盖默认行为以适应该 DBAPI 的附加功能、行为和怪癖。对于支持多个后端的 DBAPI（pyodbc、zxJDBC、mxODBC），方言模块将使用`sqlalchemy.connectors`包中的 mixin，这些 mixin 提供了在所有后端上通用的功能，最常见的是处理连接参数。这意味着使用 pyodbc、zxJDBC 或 mxODBC（一旦实现）进行连接在支持的后端上是非常一致的。

`create_engine()`使用的 URL 格式已经改进，以处理特定后端的任意数量的 DBAPI，使用了受 JDBC 启发的方案。以前的格式仍然有效，并且将选择一个“默认”的 DBAPI 实现，例如下面将使用 psycopg2 的 PostgreSQL URL：

```
create_engine("postgresql://scott:tiger@localhost/test")
```

但是，要指定特定的 DBAPI 后端，例如 pg8000，请在 URL 的“protocol”部分使用加号“+”：

```
create_engine("postgresql+pg8000://scott:tiger@localhost/test")
```

重要的方言链接：

+   连接参数的文档：[`www.sqlalchemy.org/docs/06/dbengine.html#create`](https://www.sqlalchemy.org/docs/06/dbengine.html#create)- engine-url-arguments。

+   各个方言的参考文档：[`ww`](https://ww) w.sqlalchemy.org/docs/06/reference/dialects/index.html

+   DatabaseNotes 中的技巧和窍门。

关于方言的其他注意事项：

+   SQLAlchemy 0.6 中类型系统发生了巨大变化。这对所有方言的命名约定、行为和实现都产生了影响。请参见下面关于“类型”的部分。

+   `ResultProxy`对象现在在某些情况下提供了 2 倍的速度改进，这要归功于一些重构。

+   `RowProxy`，即单个结果行对象，现在可以直接进行 pickle。

+   用于定位外部方言的 setuptools entrypoint 现在称为`sqlalchemy.dialects`。针对 0.4 或 0.5 编写的外部方言需要修改以适应 0.6，在任何情况下，因此这一变化并不会增加任何额外的困难。

+   方言现在在初始连接时会接收一个 initialize()事件，以确定连接属性。

+   编译器生成的函数和操作符现在使用（几乎）常规的分发函数形式“visit_<opname>”和“visit_<funcname>_fn”来提供定制处理。这取代了在编译器子类中复制“functions”和“operators”字典的需要，改为使用直接的访问者方法，并且还允许编译器子类完全控制渲染，因为完整的 _Function 或 _BinaryExpression 对象被传递进来。

### 方���导入

方言的导入结构已经改变。每个方言现在通过 `sqlalchemy.dialects.<name>` 导出其基本的“dialect”类以及该方言支持的完整一组 SQL 类型。例如，要导入一组 PG 类型：

```
from sqlalchemy.dialects.postgresql import (
    INTEGER,
    BIGINT,
    SMALLINT,
    VARCHAR,
    MACADDR,
    DATE,
    BYTEA,
)
```

上面，`INTEGER` 实际上是 `sqlalchemy.types` 中的普通 `INTEGER` 类型，但 PG 方言使其以与那些特定于 PG 的类型相同的方式可用，比如 `BYTEA` 和 `MACADDR`。

### 方言导入

方言的导入结构已经改变。每个方言现在通过 `sqlalchemy.dialects.<name>` 导出其基本的“dialect”类以及该方言支持的完整一组 SQL 类型。例如，要导入一组 PG 类型：

```
from sqlalchemy.dialects.postgresql import (
    INTEGER,
    BIGINT,
    SMALLINT,
    VARCHAR,
    MACADDR,
    DATE,
    BYTEA,
)
```

上面，`INTEGER` 实际上是 `sqlalchemy.types` 中的普通 `INTEGER` 类型，但 PG 方言使其以与那些特定于 PG 的类型相同的方式可用，比如 `BYTEA` 和 `MACADDR`。

## 表达式语言变化

### 一个重要的表达式语言陷阱

表达式语言有一个相当重要的行为变化，可能会影响一些应用程序。Python 布尔表达式的布尔值，即 `==`、`!=` 等，现在在与被比较的两个子句对象相关时会准确求值。

正如我们所知，将 `ClauseElement` 与任何其他对象进行比较会返回另一个 `ClauseElement`：

```
>>> from sqlalchemy.sql import column
>>> column("foo") == 5
<sqlalchemy.sql.expression._BinaryExpression object at 0x1252490>
```

这样当 Python 表达式转换为字符串时会产生 SQL 表达式：

```
>>> str(column("foo") == 5)
'foo = :foo_1'
```

但如果我们这样说会发生什么？

```
>>> if column("foo") == 5:
...     print("yes")
```

在之前的 SQLAlchemy 版本中，返回的 `_BinaryExpression` 是一个普通的 Python 对象，其求值为 `True`。现在它的求值取决于实际的 `ClauseElement` 是否应该具有与被比较的哈希值相同的值。意思是：

```
>>> bool(column("foo") == 5)
False
>>> bool(column("foo") == column("foo"))
False
>>> c = column("foo")
>>> bool(c == c)
True
>>>
```

这意味着像下面这样的代码：

```
if expression:
    print("the expression is:", expression)
```

如果 `expression` 是一个二进制子句，则不会求值。由于上述模式永远不应该被使用，基本的 `ClauseElement` 现在在布尔上下文中调用时会引发异常：

```
>>> bool(c)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  ...
  raise TypeError("Boolean value of this clause is not defined")
TypeError: Boolean value of this clause is not defined
```

想要检查是否存在 `ClauseElement` 表达式的代码应该改为：

```
if expression is not None:
    print("the expression is:", expression)
```

请记住，**这也适用于 Table 和 Column 对象**。

更改的理由有两个：

+   形如 `if c1 == c2:  <do something>` 的比较现在可以这样写

+   现在正确哈希 `ClauseElement` 对象的支持也适用于其他平台，比如 Jython。直到这一点，SQLAlchemy 在这方面严重依赖 cPython 的特定行为（并且仍然偶尔出现问题）。

### 更严格的 “executemany” 行为

在 SQLAlchemy 中，“executemany” 对应于调用 `execute()`，传递一系列绑定参数集：

```
connection.execute(table.insert(), {"data": "row1"}, {"data": "row2"}, {"data": "row3"})
```

当 `Connection` 对象将给定的 `insert()` 构造发送到编译时，它会传递给编译器在第一组传递的绑定中存在的键名，以确定语句的 VALUES 子句的构造。熟悉这种构造的用户会知道剩余字典中存在的额外键没有任何影响。现在不同的是，所有后续字典都需要至少包含第一个字典中存在的*每个*键。这意味着像这样的调用不再起作用：

```
connection.execute(
    table.insert(),
    {"timestamp": today, "data": "row1"},
    {"timestamp": today, "data": "row2"},
    {"data": "row3"},
)
```

因为第三行没有指定 `timestamp` 列。之前的 SQLAlchemy 版本会简单地为这些缺失的列插入 NULL。然而，在上面的示例中，如果 `timestamp` 列包含 Python 端默认值或函数，则*不*会被使用。这是因为 “executemany” 操作被优化为在大量参数集上实现最大性能，并且不会尝试评估那些缺失键的 Python 端默认值。因为默认值通常被实现为嵌入在 INSERT 语句中的 SQL 表达式，或者是服务器端表达式，再次根据 INSERT 字符串的结构触发，这些默认值不能根据每个参数集有条件地触发，让 Python 端默认值与 SQL/服务器端默认值的行为不一致将是不一致的。 （从 0.5 系列开始，基于 SQL 表达式的默认值被嵌入到行内，以最小化大量参数集的影响）。

因此，SQLAlchemy 0.6 通过禁止任何后续参数集留下任何字段空白来建立可预测的一致性。这样，Python 端默认值和函数不再默默失败，此外，它们允许保持与 SQL 和服务器端默认值一致的行为。

### UNION 和其他“复合”结构一致地加括号。

为了帮助 SQLite 而设计的规则已被移除，即在另一个复合元素内的第一个复合元素（例如，在 `except_()` 中的 `union()`）不会被括号括起来。这是不一致的，并且在 PostgreSQL 上产生错误的结果，因为它有关于 INTERSECTION 的优先规则，通常会让人感到惊讶。在使用 SQLite 的复杂组合时，现在需要将第一个元素转换为子查询（这也与 PG 兼容）。在[[`www.sqlalchemy.org/docs/06/sqlexpression.html`](https://www.sqlalchemy.org/docs/06/sqlexpression.html) #unions-and-other-set-operations]的 SQL 表达式教程的末尾有一个新的示例。查看[#1665](https://www.sqlalchemy.org/trac/ticket/1665)和 r6690 以获取更多背景信息。

### 一个重要的表达语言陷阱

表达语言中有一个相当重要的行为变化，可能会影响一些应用程序。Python 布尔表达式的布尔值，即`==`，`!=`等，现在在比较两个子句对象时会准确评估。

我们知道，将`ClauseElement`与任何其他对象进行比较会返回另一个`ClauseElement`：

```
>>> from sqlalchemy.sql import column
>>> column("foo") == 5
<sqlalchemy.sql.expression._BinaryExpression object at 0x1252490>
```

这样 Python 表达式在转换为字符串时会产生 SQL 表达式：

```
>>> str(column("foo") == 5)
'foo = :foo_1'
```

但如果我们这样说会发生什么呢？

```
>>> if column("foo") == 5:
...     print("yes")
```

在以前的 SQLAlchemy 版本中，返回的`_BinaryExpression`是一个普通的 Python 对象，其求值为`True`。现在它的求值取决于实际的`ClauseElement`是否应该具有与被比较的相同哈希值。意思是：

```
>>> bool(column("foo") == 5)
False
>>> bool(column("foo") == column("foo"))
False
>>> c = column("foo")
>>> bool(c == c)
True
>>>
```

这意味着像下面这样的代码：

```
if expression:
    print("the expression is:", expression)
```

如果`expression`是一个二元子句，将不会评估。由于上述模式不应该被使用，基本的`ClauseElement`现在在布尔上下文中调用时会引发异常：

```
>>> bool(c)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  ...
  raise TypeError("Boolean value of this clause is not defined")
TypeError: Boolean value of this clause is not defined
```

想要检查`ClauseElement`表达式是否存在的代码应该改为：

```
if expression is not None:
    print("the expression is:", expression)
```

请记住，**这也适用于 Table 和 Column 对象**。

更改的原因有两个：

+   现在实际上可以编写形式为`if c1 == c2: <do something>`的比较。

+   对`ClauseElement`对象进行正确哈希的支持现在在其他平台上也能正常工作，即 Jython。直到这一点，SQLAlchemy 在这方面严重依赖 cPython 的特定行为（并且偶尔还会出现问题）。

### 更严格的“executemany”行为

在 SQLAlchemy 中，“executemany”对应于调用`execute()`，传递一系列绑定参数集合：

```
connection.execute(table.insert(), {"data": "row1"}, {"data": "row2"}, {"data": "row3"})
```

当`Connection`对象发送给定的`insert()`构造进行编译时，它会传递给编译器在第一组传递的绑定中存在的键名，以确定语句的 VALUES 子句的构造。熟悉这种构造的用户将知道，剩余字典中存在的额外键不会产生任何影响。现在不同的是，所有后续字典都需要至少包含第一个字典中存在的*每个*键。这意味着像这样的调用不再起作用：

```
connection.execute(
    table.insert(),
    {"timestamp": today, "data": "row1"},
    {"timestamp": today, "data": "row2"},
    {"data": "row3"},
)
```

因为第三行未指定`timestamp`列。之前的 SQLAlchemy 版本会简单地为这些缺失的列插入 NULL。然而，在上面的示例中，如果`timestamp`列包含 Python 端默认值或函数，则*不会*被使用。这是因为“executemany”操作针对大量参数集进行了优化，不会尝试评估这些缺失键的 Python 端默认值。因为默认值通常被实现为嵌入在 INSERT 语句中的 SQL 表达式，或者是服务器端表达式，再次根据 INSERT 字符串的结构触发，这显然不能根据每个参数集有条件地触发，让 Python 端默认值与 SQL/服务器端默认值的行为不一致是不合理的（从 0.5 系列开始，基于 SQL 表达式的默认值被嵌入到内联，以最小化大量参数集的影响）。

SQLAlchemy 0.6 因此通过禁止任何后续参数集留空字段来确立可预测的一致性。这样，Python 端默认值和函数不再默默失败，而且它们的行为与 SQL 和服务器端默认值保持一致。

### UNION 和其他“复合”结构一致地加括号

为了帮助 SQLite 而设计的规则已被移除，即另一个复合元素内的第一个复合元素（例如，在`except_()`内部的`union()`）不会被括号括起来。这是不一致的，并且在 PostgreSQL 上产生错误的结果，因为它有关于 INTERSECTION 的优先规则，通常会让人感到惊讶。在与 SQLite 一起使用复杂的复合时，现在需要将第一个元素转换为子查询（这也在 PG 上兼容）。在[[`www.sqlalchemy.org/docs/06/sqlexpression.html`](https://www.sqlalchemy.org/docs/06/sqlexpression.html) #unions-and-other-set-operations]的 SQL 表达式教程的末尾有一个新示例。查看[#1665](https://www.sqlalchemy.org/trac/ticket/1665)和 r6690 以获取更多背景信息。

## 用于结果获取的 C 扩展

`ResultProxy`和相关元素，包括大多数常见的“行处理”函数，如 Unicode 转换、数值/布尔转换和日期解析，已被重新实现为可选的 C 扩展，以提高性能。这标志着 SQLAlchemy 走向“黑暗面”的开始，我们希望通过在 C 中重新实现关键部分来继续改进性能。可以通过指定`--with-cextensions`来构建这些扩展，即`python setup.py --with- cextensions install`。

扩展对使用直接`ResultProxy`访问的结果获取具有最显著影响，即由`engine.execute()`、`connection.execute()`或`session.execute()`返回的结果。在 ORM `Query`对象返回的结果中，结果获取不是开销的高比例，因此 ORM 性能改善较为适度，主要在获取大型结果集的领域。性能改进高度依赖于使用的 dbapi 以及访问每行列的语法（例如`row['name']`比`row.name`快得多）。当前的扩展对插入/更新/删除的速度没有影响，也不会提高 SQL 执行的延迟，也就是说，一个大部分时间用于执行许多具有非常小结果集的语句的应用程序不会看到太多改进。

与扩展无关，0.6 版本的性能比 0.5 版本有所提高。使用 SQLite 连接和获取 50,000 行的快速概述，主要使用直接 SQLite 访问、`ResultProxy`和简单映射的 ORM 对象：

```
sqlite select/native: 0.260s

0.6 / C extension

sqlalchemy.sql select: 0.360s
sqlalchemy.orm fetch: 2.500s

0.6 / Pure Python

sqlalchemy.sql select: 0.600s
sqlalchemy.orm fetch: 3.000s

0.5 / Pure Python

sqlalchemy.sql select: 0.790s
sqlalchemy.orm fetch: 4.030s
```

在上述例子中，ORM 比 0.5 版本快 33%获取行，这归功于 Python 内部性能的提升。使用 C 扩展我们可以再获得 20%的提升。然而，`ResultProxy`使用 C 扩展比不使用提升了 67%。其他测试报告显示在某些情况下，例如发生大量字符串转换的情况下，速度提高了高达 200%。

## 新的模式功能

`sqlalchemy.schema`包得到了一些长期需要的关注。最显著的变化是新扩展的 DDL 系统。在 SQLAlchemy 中，自 0.5 版本以来，可以创建自定义 DDL 字符串并将其与表或元数据对象关联：

```
from sqlalchemy.schema import DDL

DDL("CREATE TRIGGER users_trigger ...").execute_at("after-create", metadata)
```

现在完整的 DDL 构造都在同一系统下可用，包括用于 CREATE TABLE、ADD CONSTRAINT 等的构造：

```
from sqlalchemy.schema import Constraint, AddConstraint

AddContraint(CheckConstraint("value > 5")).execute_at("after-create", mytable)
```

此外，所有 DDL 对象现在都是常规的`ClauseElement`对象，就像任何其他 SQLAlchemy 表达式对象一样：

```
from sqlalchemy.schema import CreateTable

create = CreateTable(mytable)

# dumps the CREATE TABLE as a string
print(create)

# executes the CREATE TABLE statement
engine.execute(create)
```

并且使用`sqlalchemy.ext.compiler`扩展，您可以制作自己的：

```
from sqlalchemy.schema import DDLElement
from sqlalchemy.ext.compiler import compiles

class AlterColumn(DDLElement):
    def __init__(self, column, cmd):
        self.column = column
        self.cmd = cmd

@compiles(AlterColumn)
def visit_alter_column(element, compiler, **kw):
    return "ALTER TABLE %s ALTER COLUMN %s  %s ..." % (
        element.column.table.name,
        element.column.name,
        element.cmd,
    )

engine.execute(AlterColumn(table.c.mycolumn, "SET DEFAULT 'test'"))
```

### 废弃/移除的模式元素

schema 包也得到了极大简化。许多在 0.5 版本中被废弃的选项和方法已被移除。其他鲜为人知的访问器和方法也已被移除。

+   “owner”关键字参数已从`Table`中移除。使用“schema”表示要预先添加到表名的任何命名空间。

+   废弃的`MetaData.connect()`和`ThreadLocalMetaData.connect()`已被移除 - 将“bind”属性发送到绑定元数据。

+   废弃的 metadata.table_iterator()方法已被移除（使用 sorted_tables）。

+   从`DefaultGenerator`和子类中移除了“metadata”参数，但仍然在`Sequence`上本地存在，这是 DDL��的一个独立构造。

+   废弃的`PassiveDefault` - 使用`DefaultClause`。

+   从`Index`和`Constraint`对象中移除了公共可变性：

    +   `ForeignKeyConstraint.append_element()`

    +   `Index.append_column()`

    +   `UniqueConstraint.append_column()`

    +   `PrimaryKeyConstraint.add()`

    +   `PrimaryKeyConstraint.remove()`

这些应该以声明性的方式构造（即一次性构造）。

+   其他已移除的内容：

    +   `Table.key`（不知道用于什么）

    +   `Column.bind`（通过列的`table.bind`获取）

    +   `Column.metadata`（通过列的`table.metadata`获取）

    +   `Column.sequence`（使用列的默认值`column.default`）

### 其他行为变化

+   `UniqueConstraint`、`Index`、`PrimaryKeyConstraint` 都接受列名或列对象的列表作为参数。

+   `ForeignKey` 上的 `use_alter` 标志现在是一个快捷选项，用于可以手动构造使用 `DDL()` 事件系统的操作。此重构的一个副作用是，具有 `use_alter=True` 的 `ForeignKeyConstraint` 对象将 *不会* 在 SQLite 上发出，因为 SQLite 不支持外键的 ALTER。这对 SQLite 的行为没有影响，因为 SQLite 实际上不遵守外键约束。

+   `Table.primary_key` 不可分配 - 使用 `table.append_constraint(PrimaryKeyConstraint(...))`

+   带有 `ForeignKey` 但没有类型定义的 `Column` 定义，例如 `Column(name, ForeignKey(sometable.c.somecol))` 曾用于获取引用列的类型。现在，对于该自动类型推断的支持是部分的，可能并不适用于所有情况。

### 废弃/移除的模式元素

模式包也已经大大简化。在 0.5 版本中已弃用的许多选项和方法已被移除。其他不太常用的访问器和方法也已被移除。

+   从 `Table` 中移除了“owner”关键字参数。使用“schema”表示要预先添加到表名的任何命名空间。

+   废弃的 `MetaData.connect()` 和 `ThreadLocalMetaData.connect()` 已被移除 - 发送“bind”属性以绑定元数据。

+   已移除的废弃的 `metadata.table_iterator()` 方法（使用 `sorted_tables`）

+   从 `DefaultGenerator` 和子类中移除了“metadata”参数，但在 `Sequence` 中仍然局部存在，`Sequence` 是 DDL 中的一个独立构造。

+   废弃的 `PassiveDefault` - 使用 `DefaultClause`。

+   从 `Index` 和 `Constraint` 对象中移除了公共可变性：

    +   `ForeignKeyConstraint.append_element()`

    +   `Index.append_column()`

    +   `UniqueConstraint.append_column()`

    +   `PrimaryKeyConstraint.add()`

    +   `PrimaryKeyConstraint.remove()`

这些应该以声明性的方式构造（即一次性构造）。

+   其他已移除的内容：

    +   `Table.key`（不知道用于什么）

    +   `Column.bind`（通过列的`table.bind`获取）

    +   `Column.metadata`（通过列的`table.metadata`获取）

    +   `Column.sequence`（使用列的默认值`column.default`）

### 其他行为变化

+   `UniqueConstraint`、`Index`、`PrimaryKeyConstraint` 都接受列名或列对象的列表作为参数。

+   `ForeignKey` 上的 `use_alter` 标志现在是手动构造使用 `DDL()` 事件系统的操作的快捷选项。这个重构的副作用是，带有 `use_alter=True` 的 `ForeignKeyConstraint` 对象将不会在 SQLite 上发出，因为 SQLite 不支持外键的 ALTER。这对 SQLite 的行为没有影响，因为 SQLite 实际上不遵守 FOREIGN KEY 约束。

+   `Table.primary_key` 不可分配 - 使用 `table.append_constraint(PrimaryKeyConstraint(...))`

+   `Column` 定义中有一个 `ForeignKey` 而没有类型，例如 `Column(name, ForeignKey(sometable.c.somecol))` 用于获取引用列的类型。现在对于这种自动类型推断的支持是部分的，并且可能不适用于所有情况。

## 日志开放

通过多次额外的方法调用，你可以在创建引擎、池或映射器后设置 INFO 和 DEBUG 的日志级别，日志将开始记录。`isEnabledFor(INFO)` 方法现在每个 `Connection` 调用一次，如果已在父连接上启用，则每个 `ResultProxy` 调用一次 `isEnabledFor(DEBUG)`。池日志发送到 `log.info()` 和 `log.debug()`，没有检查 - 请注意，池的检出/归还通常是每个事务一次。

## 反射/检查器 API

反射系统，允许通过 `Table('sometable', metadata, autoload=True)` 反射表列已被开放到其自己的细粒度 API 中，该 API 允许直接检查数据库元素，如表、列、约束、索引等等。此 API 将返回值表示为简单的字符串、字典和 `TypeEngine` 对象列表。现在 `autoload=True` 的内部构建在此系统之上，将原始数据库信息转换为 `sqlalchemy.schema` 构造的过程集中化，并且各个方言的契约大大简化，极大地减少了不同后端之间的错误和不一致性。

使用检查器：

```
from sqlalchemy.engine.reflection import Inspector

insp = Inspector.from_engine(my_engine)

print(insp.get_schema_names())
```

`from_engine()` 方法在某些情况下将提供一个具有额外功能的特定于后端的检查器，例如 PostgreSQL 提供一个 `get_table_oid()` 方法：

```
my_engine = create_engine("postgresql://...")
pg_insp = Inspector.from_engine(my_engine)

print(pg_insp.get_table_oid("my_table"))
```

## RETURNING 支持

`insert()`、`update()` 和 `delete()` 构造现在支持一个 `returning()` 方法，该方法对应于 PostgreSQL、Oracle、MS-SQL 和 Firebird 支持的 SQL RETURNING 子句。目前其他后端不支持。

给定一个与 `select()` 构造方式相同的列表达式列表，这些列的值将作为常规结果集返回：

```
result = connection.execute(
    table.insert().values(data="some data").returning(table.c.id, table.c.timestamp)
)
row = result.first()
print("ID:", row["id"], "Timestamp:", row["timestamp"])
```

在四个支持的后端中，RETURNING 的实现差异很大，在 Oracle 的情况下，需要复杂地使用 OUT 参数，这些参数被重新路由到一个“模拟”结果集中，在 MS-SQL 的情况下使用笨拙的 SQL 语法。RETURNING 的使用受到限制：

+   它不适用于任何“executemany()”风格的执行。这是所有支持的 DBAPI 的限制。

+   某些后端，如 Oracle，仅支持返回单行的 RETURNING - 这包括 UPDATE 和 DELETE 语句，意味着 update()或 delete()构造必须仅匹配单行，否则会引发错误（由 Oracle 而不是 SQLAlchemy 引发）。

当单行 INSERT 语句需要获取新生成的主键值时，SQLAlchemy 也会自动使用 RETURNING，当其他地方没有通过显式的`returning()`调用指定时。这意味着对于需要主键值的插入语句，不再需要“SELECT nextval(sequence)”预执行。说实话，隐式的 RETURNING 特性确实比旧的“select nextval()”系统多产生了更多的方法开销，后者使用了一个快速而肮脏的 cursor.execute()来获取序列值，并且在 Oracle 的情况下需要额外绑定 out 参数。因此，如果方法/协议开销比额外的数据库往返开销更昂贵，则可以通过向`create_engine()`指定`implicit_returning=False`来禁用该特性。

## 类型系统更改

### 新架构

在幕后，类型系统已经完全重构，以实现两个目标：

+   将绑定参数和结果行值的处理分开，通常是 DBAPI 的要求，与类型本身的 SQL 规范分开，这是数据库的要求。这与将数据库 SQL 行为与 DBAPI 分离的总体方言重构一致。

+   为从`TypeEngine`对象生成 DDL 和基于列反射构造`TypeEngine`对象建立清晰一致的合同。

这些变化的亮点包括：

+   方言中类型的构造已彻底改写。方言现在将公开可用的类型定义为仅大写名称，并使用下划线标识符（即私有）进行内部实现类型。类型在 SQL 和 DDL 中的表达方式已移至编译器系统。这样做的效果是大多数方言中几乎没有类型对象。关于此架构的详细文档可供方言作者使用，在[source:/lib/sqlalchemy/dialects/type_migration_guidelines.txt]中。

+   现在类型的反射将返回 types.py 中的确切大写类型，或者如果该类型不是标准 SQL 类型，则在方言本身中返回大写类型。这意味着反射现在返回更准确的反射类型信息。

+   用户定义的类型，其子类为`TypeEngine`且希望提供`get_col_spec()`，现在应该将其子类化为`UserDefinedType`。

+   所有类型类上的`result_processor()`方法现在接受一个额外的参数`coltype`。这是附加到 cursor.description 的 DBAPI 类型对象，并且在适用时应该使用它来做出更好的决定，以确定应返回什么类型的结果处理可调用对象。理想情况下，结果处理函数永远不应该使用`isinstance()`，因为这是一个在这个级别上昂贵的调用。

### 本地 Unicode 模式

随着更多的 DBAPI 支持直接返回 Python unicode 对象，基本方言现在在建立第一个连接时执行检查，以确定 DBAPI 是否为基本 VARCHAR 值的基本选择返回 Python unicode 对象。如果是这样，`String`类型及其所有子类（即`Text`，`Unicode`等）在接收到结果行时将跳过“unicode”检查/转换步骤。对于大型结果集，这将大幅提高性能。目前已知“unicode 模式”可以与以下内容配合使用：

+   sqlite3 / pysqlite

+   psycopg2 - SQLA 0.6 现在在每个 psycopg2 连接对象上默认使用“UNICODE”类型扩展

+   pg8000

+   cx_oracle（我们使用输出处理器 - 很好的功能！）

其他类型可以根据需要选择禁用 unicode 处理，例如与 MS-SQL 一起使用时的`NVARCHAR`类型。

特别是，如果基于以前返回非 unicode 字符串的 DBAPI 的应用程序，则“本地 unicode”模式具有明显不同的默认行为 - 声明为`String`或`VARCHAR`的列现在默认返回 unicode，而以前则返回字符串。这可能会破坏期望非 unicode 字符串的代码。可以通过将`use_native_unicode=False`传递给`create_engine()`来禁用 psycopg2 的“本地 unicode”模式。

对于明确不希望使用 unicode 对象的字符串列，更一般的解决方案是使用`TypeDecorator`将 unicode 转换回 utf-8，或者任何所需的格式：

```
class UTF8Encoded(TypeDecorator):
  """Unicode type which coerces to utf-8."""

    impl = sa.VARCHAR

    def process_result_value(self, value, dialect):
        if isinstance(value, unicode):
            value = value.encode("utf-8")
        return value
```

请注意，`assert_unicode`标志现已弃用。SQLAlchemy 允许 DBAPI 和正在使用的后端数据库在可用时处理 Unicode 参数，并且通过检查传入类型来增加操作开销;像 sqlite 和 PostgreSQL 这样的现代系统将在其端口上引发编码错误，如果传递的数据无效。在 SQLAlchemy 确实需要将绑定参数从 Python Unicode 强制转换为编码字符串时，或者当显式使用 Unicode 类型时，如果对象是字节串，则会发出警告。可以使用 Python 警告过滤器抑制或将此警告转换为异常，该过滤器的文档在：[`docs.python.org/library/warnings.html`](https://docs.python.org/library/warnings.html)

### 通用枚举类型

现在我们在 `types` 模块中有一个 `Enum`。这是一个字符串类型，给定一组“标签”，限制给这些标签的可能值。默认情况下，此类型生成一个 `VARCHAR`，其大小为最大标签的大小，并在 CREATE TABLE 语句中对表施加 CHECK 约束。当使用 MySQL 时，默认情况下该类型使用 MySQL 的 ENUM 类型；当使用 PostgreSQL 时，该类型将使用 `CREATE TYPE <mytype> AS ENUM` 生成用户定义类型。为了在 PostgreSQL 中创建该类型，必须在构造函数中指定 `name` 参数。该类型还接受一个 `native_enum=False` 选项，它将为所有数据库使用 VARCHAR/CHECK 策略。请注意，PostgreSQL 的 ENUM 类型目前无法与 pg8000 或 zxjdbc 一起使用。

### 反射返回方言特定类型

反射现在从数据库返回最具体的类型。也就是说，如果你使用 `String` 创建一个表，然后将其反射回来，反射的列可能是 `VARCHAR`。对于支持更具体类型形式的方言，你会得到相应的类型。因此，在 Oracle 上，`Text` 类型会返回 `oracle.CLOB`，`LargeBinary` 可能是 `mysql.MEDIUMBLOB` 等等。这里的明显优势在于反射尽可能保留来自数据库的信息。

一些处理表元数据的应用程序可能希望在反射表和/或非反射表之间比较类型。`TypeEngine` 上有一个半私有访问器叫做 `_type_affinity`，以及一个相关的比较辅助函数 `_compare_type_affinity`。该访问器返回与类型对应的“通用” `types` 类：

```
>>> String(50)._compare_type_affinity(postgresql.VARCHAR(50))
True
>>> Integer()._compare_type_affinity(mysql.REAL)
False
```

### 杂项 API 变更

通常的“通用”类型仍然是通用系统中使用的一般类型，即 `String`、`Float`、`DateTime`。这里有一些变化：

+   类型不再猜测默认参数。特别是 `Numeric`、`Float`，以及 NUMERIC、FLOAT、DECIMAL 的子类，除非指定，否则不会生成任何长度或比例。这也包括有争议的 `String` 和 `VARCHAR` 类型（尽管 MySQL 方言在要求不带长度的 VARCHAR 时会预先引发错误）。不假设任何默认值，如果它们在 CREATE TABLE 语句中使用，并且底层数据库不允许这些类型的非长度版本，则会引发错误。

+   `Binary` 类型已经改名为 `LargeBinary`，用于 BLOB/BYTEA/类似类型。对于 `BINARY` 和 `VARBINARY`，直接使用 `types.BINARY`、`types.VARBINARY`，以及在 MySQL 和 MS-SQL 方言中。

+   当 `PickleType` 的 `mutable=True` 时，现在使用 `==` 进行值的比较，除非指定了带有比较函数的 “comparator” 参数给该类型。如果您要 pickle 一个自定义对象，应该实现一个 `__eq__()` 方法，以确保基于值的比较准确。

+   Numeric 和 Float 的默认 “precision” 和 “scale” 参数已移除，现在默认为 None。NUMERIC 和 FLOAT 将默认不带数字参数呈现，除非提供这些值。

+   SQLite 上的 DATE、TIME 和 DATETIME 类型现在可以使用可选的 “storage_format” 和 “regexp” 参数。“storage_format” 可用于使用自定义字符串格式存储这些类型。“regexp” 允许使用自定义正则表达式来匹配来自数据库的字符串值。

+   `__legacy_microseconds__` 在 SQLite 的 `Time` 和 `DateTime` 类型上不再受支持。您应该使用新的 “storage_format” 参数代替。

+   SQLite 上的 `DateTime` 类型现在默认使用更严格的正则表达式来匹配来自数据库的字符串。如果您使用存储在遗留格式中的数据，请使用新的 “regexp” 参数。

### 新架构

类型系统已在幕后完全重做，以实现两个目标：

+   将绑定参数和结果行值的处理分开，通常是 DBAPI 的要求，与类型本身的 SQL 规范分开，这是数据库的要求。这与将数据库 SQL 行为与 DBAPI 分开的整体方言重构保持一致。

+   为从 `TypeEngine` 对象生成 DDL 和基于列反射构造 `TypeEngine` 对象建立清晰一致的合同。

这些变更的亮点包括：

+   方言中类型的构造已完全重构。方言现在专门使用大写名称定义公开可用的类型，并使用下划线标识符（即私有）定义内部实现类型。用于在 SQL 和 DDL 中表达类型的系统已移至编译器系统。这意味着大多数方言中的类型对象大大减少。有关此架构的详细文档，供方言作者参考在 [source:/lib/sqlalchemy/dialects/type_migration_guidelines.txt]。

+   现在，类型的反射返回 types.py 中的确切大写类型，或者如果类型不是标准 SQL 类型，则返回方言本身的大写类型。这意味着反射现在返回有关反射类型的更准确信息。

+   子类化 `TypeEngine` 并希望提供 `get_col_spec()` 的用户定义类型现在应该子类化 `UserDefinedType`。

+   所有类型类上的 `result_processor()` 方法现在接受附加参数 `coltype`。这是附加到 cursor.description 的 DBAPI 类型对象，并且应该在适用时使用，以便更好地决定返回何种类型的结果处理可调用函数。理想情况下，结果处理函数永远不应该使用 `isinstance()`，因为这是在此级别的一个昂贵的调用。

### 本地 Unicode 模式

随着越来越多的 DBAPI 支持直接返回 Python Unicode 对象，基本方言现在在第一次连接时执行检查，以确定 DBAPI 是否为 VARCHAR 值的基本选择返回 Python Unicode 对象。如果是这样，`String` 类型和所有子类（即 `Text`，`Unicode` 等）在接收到结果行时将跳过“unicode”检查/转换步骤。这为大型结果集提供了显著的性能提升。目前“unicode 模式”已知可与以下一起使用：

+   sqlite3 / pysqlite

+   psycopg2 - SQLA 0.6 现在默认在每个 psycopg2 连接对象上使用“UNICODE” 类型扩展

+   pg8000

+   cx_oracle（我们使用输出处理器 - 很好的功能！）

其他类型可能会根据需要禁用 Unicode 处理，例如在与 MS-SQL 一起使用时的 `NVARCHAR` 类型。

特别是，如果迁移基于以前返回非 Unicode 字符串的 DBAPI 的应用程序，则“本机 Unicode” 模式具有明显不同的默认行为 - 声明为 `String` 或 `VARCHAR` 的列现在默认返回 Unicode，而以前会返回字符串。这可能会破坏期望非 Unicode 字符串的代码。可以通过向 `create_engine()` 传递 `use_native_unicode=False` 来禁用 psycopg2 的“本机 Unicode” 模式。

对于明确不希望使用 Unicode 对象的字符串列的更一般解决方案是使用一个 `TypeDecorator`，将 Unicode 转换回 utf-8，或者其他所需的格式：

```
class UTF8Encoded(TypeDecorator):
  """Unicode type which coerces to utf-8."""

    impl = sa.VARCHAR

    def process_result_value(self, value, dialect):
        if isinstance(value, unicode):
            value = value.encode("utf-8")
        return value
```

请注意，`assert_unicode` 标志现已弃用。SQLAlchemy 允许 DBAPI 和后端数据库在可用时处理 Unicode 参数，并且不通过检查传入类型增加操作开销；现代系统如 sqlite 和 PostgreSQL 将在其端引发编码错误，如果传递了无效数据。在 SQLAlchemy 需要将绑定参数从 Python Unicode 强制转换为编码字符串时，或者显式使用 Unicode 类型时，如果对象是字节字符串，则会发出警告。可以使用 Python 警告过滤器文档中记录的警告过滤器将此警告抑制或转换为异常：[`docs.python.org/library/warnings.html`](https://docs.python.org/library/warnings.html)

### 通用枚举类型

现在在 `types` 模块中有一个 `Enum`。这是一个字符串类型，给定一组“标签”，这些标签限制了给定给这些标签的可能值。默认情况下，此类型生成一个使用最大标签大小的 `VARCHAR`，并在 CREATE TABLE 语句中对表应用 CHECK 约束。在使用 MySQL 时，默认情况下，该类型使用 MySQL 的 ENUM 类型，而在使用 PostgreSQL 时，该类型将生成一个使用 `CREATE TYPE <mytype> AS ENUM` 的用户定义类型。为了在 PostgreSQL 中创建类型，必须在构造函数中指定 `name` 参数。该类型还接受一个 `native_enum=False` 选项，该选项将为所有数据库发出 VARCHAR/CHECK 策略。请注意，PostgreSQL ENUM 类型目前无法与 pg8000 或 zxjdbc 一起使用。

### 反射返回方言特定类型

反射现在从数据库返回尽可能具体的类型。也就是说，如果使用 `String` 创建表，然后反射它，反射的列可能是 `VARCHAR`。对于支持更具体形式的类型的方言，您将得到该类型。因此，在 Oracle 上，`Text` 类型将返回为 `oracle.CLOB`，在 MySQL 上，`LargeBinary` 可能是 `mysql.MEDIUMBLOB` 等。这里的明显优势是反射尽可能保留数据库所说的信息。

一些处理表元数据的应用程序可能希望比较反射表和/或非反射表上的类型。`TypeEngine` 上有一个半私有访问器叫做 `_type_affinity`，以及一个相关的比较助手 `_compare_type_affinity`。此访问器返回类型对应的“通用” `types` 类：

```
>>> String(50)._compare_type_affinity(postgresql.VARCHAR(50))
True
>>> Integer()._compare_type_affinity(mysql.REAL)
False
```

### 杂项 API 更改

通常的“通用”类型仍然是使用的一般系统，即 `String`、`Float`、`DateTime`。在那里有一些变化：

+   类型不再对默认参数进行任何猜测。特别是，`Numeric`、`Float`，以及子类 NUMERIC、FLOAT、DECIMAL 不会生成任何长度或精度，除非指定。这也包括有争议的 `String` 和 `VARCHAR` 类型（尽管 MySQL 方言在要求渲染没有长度的 VARCHAR 时会预先引发错误）。不会假设任何默认值，如果它们在 CREATE TABLE 语句中使用，如果底层数据库不允许这些类型的无长度版本，则会引发错误。

+   `Binary` 类型已更名为 `LargeBinary`，用于 BLOB/BYTEA/类似类型。对于 `BINARY` 和 `VARBINARY`，它们直接存在于 `types.BINARY`、`types.VARBINARY`，以及 MySQL 和 MS-SQL 方言中。

+   当 `PickleType` 的 mutable=True 时，现在使用 == 进行值比较，除非为该类型指定了带有比较函数的 “comparator” 参数。如果要对自定义对象进行 pickle，应实现一个 `__eq__()` 方法，以确保基于值的比较准确。

+   Numeric 和 Float 的默认“precision” 和 “scale” 参数已被移除，现在默认为 None。NUMERIC 和 FLOAT 现在默认不带任何数字参数呈现，除非提供这些值。

+   SQLite 上的 DATE、TIME 和 DATETIME 类型现在可以使用可选的 “storage_format” 和 “regexp” 参数。“storage_format” 可以用于使用自定义字符串格式存储这些类型。“regexp” 允许使用自定义正则表达式来匹配数据库中的字符串值。

+   在 SQLite 的 `Time` 和 `DateTime` 类型上不再支持 `__legacy_microseconds__`。您应该使用新的“storage_format”参数。

+   SQLite 上的 `DateTime` 类型现在默认使用更严格的正则表达式来匹配来自数据库的字符串。如果使用存储在传统格式中的数据，则使用新的“regexp”参数。

## ORM 更改

将 ORM 应用程序从 0.5 升级到 0.6 应该几乎不需要任何更改，因为 ORM 的行为几乎保持不变。有一些默认参数和名称更改，以及一些加载行为已经得到改进。

### 新的工作单元

工作单元的内部，主要是 `topological.py` 和 `unitofwork.py`，已完全重写并大大简化。这不应对使用产生任何影响，因为所有现有的刷新行为都已完全保持不变（或者至少在我们的测试套件和少数经过大量测试的生产环境中被使用）。刷新() 的性能现在使用 20-30% 更少的方法调用，并且还应该使用更少的内存。源代码的意图和流程现在应该相当容易理解，刷新的架构在这一点上相当开放，为潜在的新领域提供了空间。刷新过程不再依赖递归，因此可以刷新任意大小和复杂度的刷新计划。此外，映射器的“保存”过程，发出 INSERT 和 UPDATE 语句，现在缓存了这两个语句的“编译”形式，因此在非常大的刷新中进一步大幅减少了调用次数。

与 0.6 或 0.5 早期版本相比，刷新的任何行为变化都应尽快向我们报告 - 我们将确保不会丢失任何功能。

### 对 `query.update()` 和 `query.delete()` 的更改

+   查询.update() 上的 ‘expire’ 选项已更名为 ‘fetch’，与 query.delete() 的匹配方式相同。

+   `query.update()` 和 `query.delete()` 的同步策略默认为 ‘evaluate’。

+   update() 和 delete() 的 ‘synchronize’ 策略在失败时会引发错误。没有隐式回退到 “fetch”。评估的失败基于条件的结构，因此成功/失败是基于代码结构的确定性的。

### `relation()` 现在正式更名为 `relationship()`

这是为了解决长期存在的问题，“relation”在关系代数术语中意味着“表或派生表”。`relation()`名称，输入较少，将会持续存在可预见的未来，因此此更改应完全无痛。

### 子查询急切加载

添加了一种称为“子查询”加载的新型急切加载。这是一种在第一个 SQL 查询之后立即发出第二个 SQL 查询的加载，该查询为第一个查询中的所有父项加载完整集合，使用 INNER JOIN 向上连接到父项。子查询加载类似于当前的连接急切加载，使用`subqueryload()`、`subqueryload_all()`选项，以及设置在`relationship()`上的`lazy='subquery'`。子查询加载通常对加载许多较大的集合更有效，因为它无条件地使用 INNER JOIN，并且还不会重新加载父行。

### `eagerload()`, `eagerload_all()`现在是`joinedload()`, `joinedload_all()`

为了为新的子查询加载功能腾出空间，现有的`eagerload()`/`eagerload_all()` options are now superseded by `joinedload()` and `joinedload_all()`. The old names will hang around for the foreseeable future just like `relation()`将会改变。

### `lazy=False|None|True|'dynamic'`现在接受`lazy='noload'|'joined'|'subquery'|'select'|'dynamic'`

继续开放加载器策略的主题，`relationship()`上的标准关键字`lazy`选项现在是，用于延迟加载的`select`（通过属性访问时发出的 SELECT），用于急切连接加载的`joined`，用于急切子查询加载的`subquery`，不应出现任何负载的`noload`，以及用于“动态”关系的`dynamic`。旧的`True`, `False`, `None`参数仍然被接受，行为与以前完全相同。

### 在关系、连接加载上的`innerjoin=True`

现在可以指示连接急切加载的标量和集合使用 INNER JOIN 而不是 OUTER JOIN。在 PostgreSQL 上，观察到这可以在某些查询中提供 300-600%的加速。为任何在 NOT NULLable 外键上的多对一关系设置此标志，类似地，为任何保证存在相关项的集合设置此标志。

在映射器级别：

```
mapper(Child, child)
mapper(
    Parent,
    parent,
    properties={"child": relationship(Child, lazy="joined", innerjoin=True)},
)
```

在查询时级别：

```
session.query(Parent).options(joinedload(Parent.child, innerjoin=True)).all()
```

在`relationship()`级别使用`innerjoin=True`标志也将影响任何不覆盖该值的`joinedload()`选项。

### 多对一增强

+   多对一关系现在在更少的情况下会触发惰性加载，包括在大多数情况下当新值替换旧值时不会获取“旧”值。

+   与连接表子类的多对一关系现在使用`get()`进行简单加载（称为“use_get”条件），即`Related`->``Sub(Base)``，无需重新定义基表的主连接条件。[ticket:1186]

+   使用声明性列指定外键，即`ForeignKey(MyRelatedClass.id)`不会破坏“use_get”条件的发生。[ticket:1492]

+   relationship()、joinedload() 和 joinedload_all() 现在具有一个名为“innerjoin”的选项。指定 `True` 或 `False` 来控制急切连接是构造为 INNER 还是 OUTER 连接。默认始终为 `False`。映射器选项将覆盖 relationship() 上指定的任何设置。通常应该为一对多、非空外键关系设置此选项，以允许改进的连接性能。[ticket:1544]

+   联接急切加载的行为，当存在 LIMIT/OFFSET 时，使主查询包装在子查询中的情况现在除了所有急切加载都是一对多连接时有一个例外。在这些情况下，急切连接直接针对父表，同时限制/偏移量没有子查询的额外开销，因为一对多连接不会将行添加到结果中。

    例如，在 0.5 版本中这个查询：

    ```
    session.query(Address).options(eagerload(Address.user)).limit(10)
    ```

    将生成如下 SQL 语句：

    ```
    SELECT  *  FROM
      (SELECT  *  FROM  addresses  LIMIT  10)  AS  anon_1
      LEFT  OUTER  JOIN  users  AS  users_1  ON  users_1.id  =  anon_1.addresses_user_id
    ```

    这是因为任何急切的加载程序的存在都表明它们中的一部分或全部可能与多行集合相关，这将需要将任何种类的行数敏感修改器，如 LIMIT，包装在子查询中。

    在 0.6 版本中，该逻辑更加敏感，并且可以检测到所有急切加载是否表示一对多关系，在这种情况下，急切连接不会影响行数：

    ```
    SELECT  *  FROM  addresses  LEFT  OUTER  JOIN  users  AS  users_1  ON  users_1.id  =  addresses.user_id  LIMIT  10
    ```

### 具有联接表继承的可变主键

在具有子表主键外键到父表主键的联接表继承配置上，现在可以在类似于 PostgreSQL 的具有级联功能的数据库上更新子表。`mapper()` 现在有一个选项 `passive_updates=True`，表示此外键将自动更新。如果在不支持级联的数据库上，如 SQLite 或 MySQL/MyISAM，则将此标志设置为 `False`。将来的功能增强将尝试根据正在使用的方言/表样式来自动配置此标志。

### Beaker 缓存

Beaker 集成的一个有前途的新示例在 `examples/beaker_caching` 中。这是一个简单的示例，它在 `Query` 的结果生成引擎中应用了 Beaker 缓存。缓存参数通过 `query.options()` 提供，并允许完全控制缓存内容。SQLAlchemy 0.6 对 `Session.merge()` 方法进行了改进，以支持此类示例，并在大多数情况下提供了显著改进的性能。

### 其他更改

+   当选择多列/实体时，`Query` 返回的“行元组”对象现在可以进行序列化，性能更高。

+   `query.join()` 已重新设计以提供更一致的行为和更灵活的功能（包括 [ticket:1537]）

+   `query.select_from()` 接受多个子句，以在 FROM 子句中生成多个逗号分隔的条目。在从多个 join() 子句中选择时非常有用。

+   `Session.merge()` 上的“dont_load=True”标志已弃用，现在为“load=False”。

+   添加了“make_transient()”助手函数，它将一个持久化/分离的实例转换为瞬态实例（即删除实例键并从任何会话中删除）。[ticket:1052]

+   在 mapper()上的 allow_null_pks 标志已弃用，并已更名为 allow_partial_pks。它默认为“on”。这意味着对于任何主键列具有非空值的行都将被视为标识。这种情况的需要通常仅在映射到外连接时发生。当设置为 False 时，具有 NULL 的 PK 将不被视为主键 - 特别是这意味着结果行将返回为 None（或不填入集合中），并且新的 0.6 版本还表示 session.merge()不会为此类 PK 值向数据库发出往返传输。【票号：1680】

+   “backref”的机制已完全合并到更精细的“back_populates”系统中，并完全在`RelationProperty`的`_generate_backref()`方法中进行。这使得`RelationProperty`的初始化过程更简单，并允许更轻松地传播设置（例如从`RelationProperty`的子类）。内部的`BackRef()`已经消失，`backref()`返回一个纯元组，`RelationProperty`理解这个元组。

+   `ResultProxy`的 keys 属性现在是一个方法，因此对它的引用（`result.keys`）必须改为方法调用（`result.keys()`）

+   `ResultProxy.last_inserted_ids`现已弃用，请改用`ResultProxy.inserted_primary_key`。

### 已弃用/移除的 ORM 元素

大多数在 0.5 版本中已弃用并引发弃用警告的元素已移除（有几个例外）。所有标记为“待弃用”的元素现在已弃用，并将在使用时引发警告。

+   ‘transactional’标志在 sessionmaker()和其他函数中已移除。使用‘autocommit=True’表示‘transactional=False’。

+   在 mapper()上的‘polymorphic_fetch’参数已移除。加载可以使用‘with_polymorphic’选项来控制。

+   在 mapper()上的‘select_table’参数已移除。为了实现此功能，请使用‘with_polymorphic=(“*”, <some selectable>)’。

+   在 synonym()上的‘proxy’参数已移除。此标志在 0.5 版本中未起作用，因为“proxy generation”行为现在是自动的。

+   对 joinedload()、joinedload_all()、contains_eager()、lazyload()、defer()和 undefer()传递单个元素列表而不是多个位置参数的做法已弃用。

+   对 query.order_by()、query.group_by()、query.join()或 query.outerjoin()传递单个元素列表而不是多个位置参数的做法已弃用。

+   `query.iterate_instances()`已移除。使用`query.instances()`。

+   `Query.query_from_parent()`已移除。使用 sqlalchemy.orm.with_parent()函数生成一个“parent”子句，或者使用`query.with_parent()`。

+   `query._from_self()`已移除，请改用`query.from_self()`。

+   对 composite()的“comparator”参数已移除。使用“comparator_factory”。

+   `RelationProperty._get_join()`已移除。

+   Session 上的‘echo_uow’标志已移除。在“sqlalchemy.orm.unitofwork”名称上使用日志记录。

+   `session.clear()` 已移除。请使用 `session.expunge_all()`。

+   `session.save()`、`session.update()` 和 `session.save_or_update()` 已移除。请使用 `session.add()` 和 `session.add_all()`。

+   `session.flush()` 中的 “objects” 标志仍然被弃用。

+   `session.merge()` 中的 “dont_load=True” 标志已弃用，改为使用 “load=False”。

+   `ScopedSession.mapper` 仍然被弃用。请参阅[`www.sqlalchemy.org/trac/wiki/Usag`](https://www.sqlalchemy.org/trac/wiki/Usag) eRecipes/SessionAwareMapper 上的使用配方。

+   将 `InstanceState`（内部 SQLAlchemy 状态对象）传递给 `attributes.init_collection()` 或 `attributes.get_history()` 已被弃用。这些函数是公共 API，通常期望一个常规映射对象实例。

+   `declarative_base()` 中的 ‘engine’ 参数已被移除。请使用 ‘bind’ 关键字参数。

### 新的工作单元

工作单元的内部，主要是 `topological.py` 和 `unitofwork.py`，已完全重写并大大简化。这对使用没有影响，因为所有现有的刷新行为都被完全保留了（或者至少在我们的测试套件和少数大量测试的生产环境中被保留了）。刷新（flush）的性能现在使用的方法调用减少了 20-30%，而且还应该使用更少的内存。源代码的意图和流程现在应该相当容易跟踪，并且刷新的架构在这一点上相当开放，为潜在的新技术领域提供了空间。刷新过程不再依赖于递归，因此可以刷新任意大小和复杂度的刷新计划。此外，映射器的“保存”过程，发出 INSERT 和 UPDATE 语句，现在缓存了这两个语句的“编译”形式，以便在非常大的刷新中进一步大幅减少调用次数。

请尽快向我们报告在刷新与 0.6 或 0.5 早期版本之间观察到的任何行为变化——我们将确保不会丢失任何功能。

### 对 `query.update()` 和 `query.delete()` 的更改

+   `query.update()` 中的‘expire’选项已更名为‘fetch’，与 `query.delete()` 的命名一致。

+   `query.update()` 和 `query.delete()` 在同步策略上都默认为 ‘evaluate’。

+   对 `update()` 和 `delete()` 的 ‘synchronize’ 策略在失败时会引发错误。没有隐式回退到“fetch”。评估的失败是基于条件结构的，因此基于代码结构，成功/失败是可以确定的。

### `relation()` 正式更名为 `relationship()`

这是为了解决“relation”在关系代数中表示“表或派生表”的长期问题。`relation()` 这个名字，打字更少，将在可预见的将来继续存在，所以这个变化应该完全没有痛苦。

### 子查询的贪婪加载

添加了一种名为“subquery”加载的新类型的急切加载。这是一种加载，它在第一个加载完整集合的 SQL 查询之后立即发出第二个 SQL 查询，通过 INNER JOIN 连接到第一个查询中的所有父级。子查询加载类似于当前的连接预加载，使用`subqueryload()`、`subqueryload_all()`选项，以及设置在`relationship()`上的`lazy='subquery'`。子查询加载通常更有效地加载许多较大的集合，因为它无条件地使用 INNER JOIN，而且也不会重新加载父行。

### `eagerload()`, `eagerload_all()`现在是`joinedload()`, `joinedload_all()`

为了为新的子查询加载功能腾出空间，现有的`eagerload()`/`eagerload_all()` options are now superseded by `joinedload()` and `joinedload_all()`. The old names will hang around for the foreseeable future just like `relation()`。

### `lazy=False|None|True|'dynamic'`现在接受`lazy='noload'|'joined'|'subquery'|'select'|'dynamic'`

在加载器策略开放的主题上继续，`relationship()`上的标准关键字`lazy`选项现在是，用于延迟加载的`select`（通过属性访问时发出的 SELECT），用于急切连接加载的`joined`，用于急切子查询加载的`subquery`，不应出现任何负载的`noload`，以及用于“动态”关系的`dynamic`。旧的`True`, `False`, `None`参数仍然被接受，行为与以前完全相同。

### 在关系、joinedload 上设置 innerjoin=True

现在可以指示使用 INNER JOIN 而不是 OUTER JOIN 来连接预加载的标量和集合。在 PostgreSQL 上，这被观察到可以为某些查询提供 300-600% 的速度提升。为任何在 NOT NULLable 外键上的多对一设置此标志，以及对于任何保证存在相关项目的集合。

在映射器级别：

```
mapper(Child, child)
mapper(
    Parent,
    parent,
    properties={"child": relationship(Child, lazy="joined", innerjoin=True)},
)
```

在查询时间级别：

```
session.query(Parent).options(joinedload(Parent.child, innerjoin=True)).all()
```

在 `relationship()` 级别的 `innerjoin=True` 标志也将对任何不覆盖该值的 `joinedload()` 选项产生影响。

### 对许多对一的增强

+   许多对一关系现在在更少的情况下会触发延迟加载，包括在大多数情况下不会在替换新值时获取“旧”值。

+   对于连接表子类的多对一关系现在使用 get() 进行简单加载（称为“use_get”条件），即 `Related`->``Sub(Base)``, 无需重新定义基表的主连接条件。[ticket:1186]

+   指定具有声明列的外键，即 `ForeignKey(MyRelatedClass.id)` 不会阻止“use_get”条件的发生 [ticket:1492]

+   relationship()、joinedload() 和 joinedload_all() 现在具有一个名为“innerjoin”的选项。指定 `True` 或 `False` 来控制是否构建一个 INNER 或 OUTER 连接的急切连接。默认始终为 `False`。映射器选项将覆盖在 relationship() 上指定的任何设置。通常应为多对一、非空外键关系设置以允许改进的连接性能。[ticket:1544]

+   联接急切加载的行为，即当 LIMIT/OFFSET 存在时，主查询被包装在子查询中，现在对所有急切加载都是多对一联接的情况做了一个例外。在这些情况下，急切连接直接针对父表进行，同时限制/偏移量没有额外的子查询开销，因为多对一连接不会向结果添加行。

    例如，在 0.5 版本中，这个查询：

    ```
    session.query(Address).options(eagerload(Address.user)).limit(10)
    ```

    会生成类似于以下的 SQL：

    ```
    SELECT  *  FROM
      (SELECT  *  FROM  addresses  LIMIT  10)  AS  anon_1
      LEFT  OUTER  JOIN  users  AS  users_1  ON  users_1.id  =  anon_1.addresses_user_id
    ```

    这是因为任何急切加载器的存在都表明它们中的一些或全部可能与多行集合相关联，这将需要将任何种类的行计数敏感修饰符（如 LIMIT）包装在子查询中。

    在 0.6 版本中，该逻辑更加敏感，可以检测到所有急切加载器是否代表多对一关系，如果是这种情况，则急切连接不会影响行数：

    ```
    SELECT  *  FROM  addresses  LEFT  OUTER  JOIN  users  AS  users_1  ON  users_1.id  =  addresses.user_id  LIMIT  10
    ```

### 使用联接表继承的可变主键

在子表具有外键到父表主键的联接表继承配置中，现在可以在类似 PostgreSQL 这样支持级联的数据库上进行更新。`mapper()`现在有一个选项`passive_updates=True`，表示此外键将自动更新。如果在不支持级联的数据库上，如 SQLite 或 MySQL/MyISAM 上，将此标志设置为`False`。未来的功能增强将尝试根据使用的方言/表样式自动配置此标志。

### Beaker 缓存

Beaker 集成的一个有前途的新例子在`examples/beaker_caching`中。这是一个简单的配方，将 Beaker 缓存应用于`Query`的结果生成引擎中。缓存参数通过`query.options()`提供，并允许完全控制缓存的内容。SQLAlchemy 0.6 对`Session.merge()`方法进行了改进，以支持这种和类似的配方，并在大多数情况下提供了显著改进的性能。

### 其他变化

+   当选择多列/实体时，`Query`返回的“行元组”对象现在也是可序列化的，并且性能更高。

+   `query.join()`已经重新设计，以提供更一致的行为和更灵活性（包括[ticket:1537]）

+   `query.select_from()`接受多个子句，以在 FROM 子句中产生多个逗号分隔的条目。在从多个 join()子句中选择时很有用。

+   `Session.merge()`上的“dont_load=True”标志已被弃用，现在是“load=False”。

+   添加了“make_transient()”辅助函数，将持久/分离实例转换为瞬态实例（即删除实例键并从任何会话中移除。）[ticket:1052]

+   `mapper()`上的`allow_null_pks`标志已被弃用，并已重命名为`allow_partial_pks`。默认情况下已打开。这意味着对于任何主键列中有非空值的行将被视为标识。这种情况通常只在映射到外连接时发生。当设置为 False 时，具有 NULL 值的 PK 将不被视为主键 - 特别是这意味着结果行将返回为 None（或不会填充到集合中），并且在 0.6 版本中还表示`session.merge()`不会为此类 PK 值发出往返数据库的请求。[ticket:1680]

+   “backref”的机制已完全合并到更精细的“back_populates”系统中，并完全在`RelationProperty`的`_generate_backref()`方法中进行。这使得`RelationProperty`的初始化过程更简单，并允许更容易地传播设置（例如从`RelationProperty`的子类）到反向引用。内部的`BackRef()`已经消失，`backref()`返回一个被`RelationProperty`理解的普通元组。

+   `ResultProxy`的`keys`属性现在是一个方法，因此对它的引用（`result.keys`）必须更改为方法调用（`result.keys()`）。

+   `ResultProxy.last_inserted_ids`现已弃用，改用`ResultProxy.inserted_primary_key`。

### 弃用/移除的 ORM 元素

在 0.5 版本中被弃用并引发弃用警告的大多数元素已被移除（有少数例外）。所有标记为“即将弃用”的元素现在已被弃用，并在使用时会引发警告。

+   `sessionmaker()`和其他地方的‘transactional’标志已被移除。使用‘autocommit=True’来表示‘transactional=False’。

+   `mapper()`上的‘polymorphic_fetch’参数已被移除。可以使用‘with_polymorphic’选项来控制加载。

+   `mapper()`上的‘select_table’参数已被移除。使用‘with_polymorphic=(“*”, <some selectable>)’来实现此功能。

+   `synonym()`上的‘proxy’参数已被移除。在 0.5 版本中，此标志没有任何作用，因为“代理生成”行为现在是自动的。

+   将元素的单个列表传递给`joinedload()`、`joinedload_all()`、`contains_eager()`、`lazyload()`、`defer()`和`undefer()`，而不是多个位置*args，已被弃用。

+   将元素的单个列表传递给`query.order_by()`、`query.group_by()`、`query.join()`或`query.outerjoin()`，而不是多个位置*args，已被弃用。

+   移除了`query.iterate_instances()`。使用`query.instances()`。

+   移除了`Query.query_from_parent()`。使用`sqlalchemy.orm.with_parent()`函数生成“parent”子句，或者使用`query.with_parent()`。

+   移除了`query._from_self()`，请改用`query.from_self()`。

+   `composite()`的“comparator”参数已被移除。使用“comparator_factory”。

+   移除了`RelationProperty._get_join()`。

+   `Session`上的‘echo_uow’标志已被移除。在“sqlalchemy.orm.unitofwork”名称上使用日志记录。

+   `session.clear()` 被移除。使用 `session.expunge_all()`。

+   `session.save()`、`session.update()`、`session.save_or_update()` 被移除。使用 `session.add()` 和 `session.add_all()`。

+   在 `session.flush()` 上的 “objects” 标志仍然被弃用。

+   在 `session.merge()` 上的 “dont_load=True” 标志已被弃用，改用 “load=False”。

+   `ScopedSession.mapper` 仍然被弃用。参见使用方法的示例：[`www.sqlalchemy.org/trac/wiki/Usag`](https://www.sqlalchemy.org/trac/wiki/Usag) eRecipes/SessionAwareMapper

+   在 `attributes.init_collection()` 或 `attributes.get_history()` 中传递 `InstanceState`（内部 SQLAlchemy 状态对象）已被弃用。这些函数是公共 API，通常期望普通映射对象实例。

+   `declarative_base()` 上的 ‘engine’ 参数已被移除。使用 ‘bind’ 关键字参数。

## 扩展

### SQLSoup

SQLSoup 已经现代化并更新以反映常见的 0.5/0.6 功能，包括明确定义的会话集成。请阅读新文档：[[`www.sqlalc`](https://www.sqlalc) hemy.org/docs/06/reference/ext/sqlsoup.html]。

### Declarative

`DeclarativeMeta`（`declarative_base` 的默认元类）以前允许子类修改 `dict_` 来添加类属性（例如列）。这种方式已不再起作用，`DeclarativeMeta` 构造函数现在忽略 `dict_`。相反，类属性应直接赋值，例如 `cls.id=Column(...)`，或者应该使用 [MixIn 类](https://www.sqlalchemy.org/docs/reference/ext/declarative.html#mix-in-classes) 方法而不是元类方法。

### SQLSoup

SQLSoup 已经现代化并更新以反映常见的 0.5/0.6 功能，包括明确定义的会话集成。请阅读新文档：[[`www.sqlalc`](https://www.sqlalc) hemy.org/docs/06/reference/ext/sqlsoup.html]。

### Declarative

`DeclarativeMeta`（`declarative_base` 的默认元类）以前允许子类修改 `dict_` 来添加类属性（例如列）。这种方式已不再起作用，`DeclarativeMeta` 构造函数现在忽略 `dict_`。相反，类属性应直接赋值，例如 `cls.id=Column(...)`，或者应该使用 [MixIn 类](https://www.sqlalchemy.org/docs/reference/ext/declarative.html#mix-in-classes) 方法而不是元类方法。
