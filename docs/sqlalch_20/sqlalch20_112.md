# Microsoft SQL Server

> 原文：[`docs.sqlalchemy.org/en/20/dialects/mssql.html`](https://docs.sqlalchemy.org/en/20/dialects/mssql.html)

对 Microsoft SQL Server 数据库的支持。

下表总结了当前数据库发布版本的支持水平。

**支持的 Microsoft SQL Server 版本**

| 支持类型 | 版本 |
| --- | --- |
| CI 全面测试 | 2017 |
| 正常支持 | 2012+ |
| 尽力而为 | 2005+ |

## DBAPI 支持

可用的方言/DBAPI 选项如下。请参考各个 DBAPI 部分获取连接信息。

+   PyODBC

+   pymssql

+   aioodbc

## 外部方言

除了上述具有原生 SQLAlchemy 支持的 DBAPI 层之外，还有适用于 SQL Server 的与第三方方言兼容的其他 DBAPI 层。请参阅 Dialects 页面上的“外部方言”列表。  ## 自动递增行为 / IDENTITY 列

SQL Server 使用 `IDENTITY` 结构提供所谓的“自动递增”行为，可以放置在表中的任何单个整数列上。SQLAlchemy 将 `IDENTITY` 视为整数主键列的默认“autoincrement”行为的一部分，该行为在 `Column.autoincrement` 中描述。这意味着默认情况下，`Table` 中的第一个整数主键列将被视为标识列 - 除非它与 `Sequence` 关联 - 并将生成 DDL 如下：

```py
from sqlalchemy import Table, MetaData, Column, Integer

m = MetaData()
t = Table('t', m,
        Column('id', Integer, primary_key=True),
        Column('x', Integer))
m.create_all(engine)
```

上述示例将生成 DDL 如下：

```py
CREATE  TABLE  t  (
  id  INTEGER  NOT  NULL  IDENTITY,
  x  INTEGER  NULL,
  PRIMARY  KEY  (id)
)
```

对于不希望使用默认生成的 `IDENTITY` 的情况，请在第一个整数主键列上将 `Column.autoincrement` 标志设置为 `False`：

```py
m = MetaData()
t = Table('t', m,
        Column('id', Integer, primary_key=True, autoincrement=False),
        Column('x', Integer))
m.create_all(engine)
```

要将 `IDENTITY` 关键字添加到非主键列，请在所需的 `Column` 对象上将 `Column.autoincrement` 标志设置为 `True`，并确保在任何整数主键列上将 `Column.autoincrement` 设置为 `False`：

```py
m = MetaData()
t = Table('t', m,
        Column('id', Integer, primary_key=True, autoincrement=False),
        Column('x', Integer, autoincrement=True))
m.create_all(engine)
```

自 1.4 版更改：在 `Column` 中添加了 `Identity` 结构，以指定 IDENTITY 的起始和增量参数。这些取代了使用 `Sequence` 对象来指定这些值。

自 1.4 版弃用：`Column` 的 `mssql_identity_start` 和 `mssql_identity_increment` 参数已被弃用，应该用 `Identity` 对象替换。同时指定两种配置 IDENTITY 的方式将导致编译错误。这些选项也不再作为 `Inspector.get_columns()` 中 `dialect_options` 键的一部分返回。请使用 `identity` 键中的信息。

自 1.3 版弃用：使用 `Sequence` 指定 IDENTITY 特性已被弃用，并将在未来版本中删除。请使用 `Identity` 对象参数 `Identity.start` 和 `Identity.increment`。

自 1.4 版更改：移除了使用 `Sequence` 对象修改 IDENTITY 特性的能力。`Sequence` 对象现在仅操作真正的 T-SQL SEQUENCE 类型。

注意

表中只能有一个 IDENTITY 列。当使用 `autoincrement=True` 启用 IDENTITY 关键字时，SQLAlchemy 不会防止多个列同时指定该选项。SQL Server 数据库将拒绝 `CREATE TABLE` 语句。

注意

尝试为标记为 IDENTITY 的列提供值的 INSERT 语句将被 SQL Server 拒绝。为了接受该值，必须启用会话级选项“SET IDENTITY_INSERT”。当使用核心 `Insert` 构造时，SQLAlchemy SQL Server 方言将在执行指定 IDENTITY 列的值时自动执行此操作；如果执行为该语句的调用启用了“IDENTITY_INSERT”选项。然而，这种情况性能不高，不应依赖于正常使用。如果表实际上不需要其整数主键列的 IDENTITY 行为，则在创建表时应禁用该关键字，确保设置 `autoincrement=False`。

### 控制“Start”和“Increment”

通过传递给 `Identity` 对象的 `Identity.start` 和 `Identity.increment` 参数，可以对 `IDENTITY` 生成器的“start”和“increment”值进行具体控制：

```py
from sqlalchemy import Table, Integer, Column, Identity

test = Table(
    'test', metadata,
    Column(
        'id',
        Integer,
        primary_key=True,
        Identity(start=100, increment=10)
    ),
    Column('name', String(20))
)
```

上述 `Table` 对象的 CREATE TABLE 将是：

```py
CREATE  TABLE  test  (
  id  INTEGER  NOT  NULL  IDENTITY(100,10)  PRIMARY  KEY,
  name  VARCHAR(20)  NULL,
  )
```

注意

`Identity` 对象支持许多其他参数，除了 `start` 和 `increment` 之外。这些参数不受 SQL Server 支持，在生成 CREATE TABLE ddl 时将被忽略。

从版本 1.3.19 更改：`Identity` 对象现在用于影响 SQL Server 下的 `Column` 的 `IDENTITY` 生成器。以前使用 `Sequence` 对象。由于 SQL Server 现在支持真正的序列作为一个单独的构造，`Sequence` 将从 SQLAlchemy 版本 1.4 开始以正常方式运行。

### 使用非整数数值类型的 IDENTITY

SQL Server 也允许将 `IDENTITY` 用于 `NUMERIC` 列。为了在 SQLAlchemy 中顺利实现这种模式，列的主要数据类型应保持为 `Integer`，但是可以使用 `TypeEngine.with_variant()` 指定在 SQL Server 数据库中部署的底层实现类型为 `Numeric`：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class TestTable(Base):
    __tablename__ = "test"
    id = Column(
        Integer().with_variant(Numeric(10, 0), "mssql"),
        primary_key=True,
        autoincrement=True,
    )
    name = Column(String)
```

在上述示例中，`Integer().with_variant()`提供了清晰的使用信息，准确描述了代码的意图。`autoincrement`仅适用于`Integer`的一般限制是在元数据级别而不是每个方言级别建立的。

使用上述模式时，从行插入返回的主键标识符（也是将分配给诸如上面的`TestTable`之类的 ORM 对象的值）在使用 SQL Server 时将是`Decimal()`的实例，而不是`int`。通过向`Numeric`类型的`Numeric.asdecimal`传递 False 来更改上述`Numeric(10, 0)`的返回类型以返回浮点数。为了将上述`Numeric(10, 0)`的返回类型规范化为返回 Python int（Python 3 中还支持“长”整数值），请使用`TypeDecorator`如下所示：

```py
from sqlalchemy import TypeDecorator

class NumericAsInteger(TypeDecorator):
  '''normalize floating point return values into ints'''

    impl = Numeric(10, 0, asdecimal=False)
    cache_ok = True

    def process_result_value(self, value, dialect):
        if value is not None:
            value = int(value)
        return value

class TestTable(Base):
    __tablename__ = "test"
    id = Column(
        Integer().with_variant(NumericAsInteger, "mssql"),
        primary_key=True,
        autoincrement=True,
    )
    name = Column(String)
```

### INSERT 行为

在 INSERT 时处理 `IDENTITY` 列涉及两个关键技术。最常见的是能够获取给定 `IDENTITY` 列的“最后插入值”，这是 SQLAlchemy 在许多情况下隐式执行的过程，最重要的是在 ORM 中。

获取此值的过程有几种变体：

+   在绝大多数情况下，RETURNING 与 SQL Server 上的 INSERT 语句一起使用，以获取新生成的主键值：

    ```py
    INSERT  INTO  t  (x)  OUTPUT  inserted.id  VALUES  (?)
    ```

    从 SQLAlchemy 2.0 开始，默认还使用 “插入多个值”行为适用于 INSERT 语句 功能来优化多行 INSERT 语句；对于 SQL Server，该功能适用于 RETURNING 和非 RETURNING INSERT 语句。

    在版本 2.0.10 中更改：由于与行排序问题有关，SQLAlchemy 版本 2.0.9 的 SQL Server 的 “插入多个值”行为适用于 INSERT 语句 功能暂时被禁用。从 2.0.10 开始，该功能已重新启用，并针对工作单元对 RETURNING 的要求进行了特殊处理，以进行排序。

+   当 RETURNING 不可用或已通过`implicit_returning=False`禁用时，将使用`scope_identity()`函数或`@@identity`变量；后端的行为各不相同：

    +   使用 PyODBC 时，短语`; select scope_identity()`将附加到 INSERT 语句的末尾；将获取第二个结果集以接收该值。给定表格为：

        ```py
        t = Table(
            't',
            metadata,
            Column('id', Integer, primary_key=True),
            Column('x', Integer),
            implicit_returning=False
        )
        ```

        INSERT 将如下所示：

        ```py
        INSERT  INTO  t  (x)  VALUES  (?);  select  scope_identity()
        ```

    +   其他方言如 pymssql 在 INSERT 语句之后调用 `SELECT scope_identity() AS lastrowid`。如果将标志 `use_scope_identity=False` 传递给 `create_engine()`，则会使用语句 `SELECT @@identity AS lastrowid`。

包含 `IDENTITY` 列的表将禁止明确引用标识列的 INSERT 语句。当使用核心 `insert()` 构造（而不是纯字符串 SQL）创建的 INSERT 构造引用标识列时，SQLAlchemy 方言将检测到，并且在此情况下将在执行 INSERT 语句之前发出 `SET IDENTITY_INSERT ON`，并在执行后发出 `SET IDENTITY_INSERT OFF`。例如：

```py
m = MetaData()
t = Table('t', m, Column('id', Integer, primary_key=True),
                Column('x', Integer))
m.create_all(engine)

with engine.begin() as conn:
    conn.execute(t.insert(), {'id': 1, 'x':1}, {'id':2, 'x':2})
```

上述列将使用 IDENTITY 创建，但我们发出的 INSERT 语句指定了显式值。在回声输出中，我们可以看到 SQLAlchemy 如何处理这种情况：

```py
CREATE  TABLE  t  (
  id  INTEGER  NOT  NULL  IDENTITY(1,1),
  x  INTEGER  NULL,
  PRIMARY  KEY  (id)
)

COMMIT
SET  IDENTITY_INSERT  t  ON
INSERT  INTO  t  (id,  x)  VALUES  (?,  ?)
((1,  1),  (2,  2))
SET  IDENTITY_INSERT  t  OFF
COMMIT
```

这是适用于测试和大量插入场景的辅助用例。

## SEQUENCE 支持

`Sequence` 对象创建“真正”的序列，即 `CREATE SEQUENCE`：

```py
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> from sqlalchemy.dialects import mssql
>>> print(CreateSequence(Sequence("my_seq", start=1)).compile(dialect=mssql.dialect()))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

对于整数主键生成，通常应优先选择 SQL Server 的 `IDENTITY` 构造而不是序列。

提示

T-SQL 的默认起始值为 `-2**63`，而不是大多数其他 SQL 数据库中的 1。如果这是预期的默认值，则用户应明确设置 `Sequence.start` 为 1：

```py
seq = Sequence("my_sequence", start=1)
```

新版 1.4 中新增 SQL Server 对 `Sequence` 的支持

从版本 2.0 起更改：SQL Server 方言不再为 `CREATE SEQUENCE` 隐式呈现“START WITH 1”，这是在版本 1.4 中首次实现的行为。

## VARCHAR / NVARCHAR 上的 MAX

SQL Server 支持特殊字符串“MAX”在 `VARCHAR` 和 `NVARCHAR` 数据类型中，表示“最大可能长度”。当前的方言将此处理为基本类型中的长度“None”，而不是提供这些类型的方言特定版本，因此指定基本类型如 `VARCHAR(None)` 可以在不同的后端上假定“无长度”的行为而不使用方言特定的类型。

要构建具有 MAX 长度的 SQL Server VARCHAR 或 NVARCHAR，请使用 None：

```py
my_table = Table(
    'my_table', metadata,
    Column('my_data', VARCHAR(None)),
    Column('my_n_data', NVARCHAR(None))
)
```

## 校对支持

字符排序由字符串参数“collation”指定的基本字符串类型支持：

```py
from sqlalchemy import VARCHAR
Column('login', VARCHAR(32, collation='Latin1_General_CI_AS'))
```

当这样的列与 `Table` 关联时，此列的 CREATE TABLE 语句将产生：

```py
login VARCHAR(32) COLLATE Latin1_General_CI_AS NULL
```

## LIMIT/OFFSET 支持

MSSQL 自 SQL Server 2012 起已添加了对 LIMIT / OFFSET 的支持，通过“OFFSET n ROWS”和“FETCH NEXT n ROWS”子句。如果检测到 SQL Server 2012 或更高版本，SQLAlchemy 将自动支持这些语法。

1.4 版本更改：增加了对 SQL Server “OFFSET n ROWS” 和 “FETCH NEXT n ROWS” 语法的支持。

对于仅指定 LIMIT 而不带 OFFSET 的语句，所有版本的 SQL Server 都支持 TOP 关键字。当没有 OFFSET 子句时，此语法用于所有 SQL Server 版本。例如：

```py
select(some_table).limit(5)
```

将类似于渲染：

```py
SELECT TOP 5 col1, col2.. FROM table
```

对于早于 SQL Server 2012 的 SQL Server 版本，使用 LIMIT 和 OFFSET 或仅 OFFSET 的语句将使用 `ROW_NUMBER()` 窗口函数进行渲染。例如：

```py
select(some_table).order_by(some_table.c.col3).limit(5).offset(10)
```

将类似于渲染：

```py
SELECT anon_1.col1, anon_1.col2 FROM (SELECT col1, col2,
ROW_NUMBER() OVER (ORDER BY col3) AS
mssql_rn FROM table WHERE t.x = :x_1) AS
anon_1 WHERE mssql_rn > :param_1 AND mssql_rn <= :param_2 + :param_1
```

注意，当使用 LIMIT 和/或 OFFSET 时，无论是使用较旧还是较新的 SQL Server 语法，语句都必须有 ORDER BY，否则会引发 `CompileError`。

## DDL 注释支持

注释支持包括对 `Table.comment` 和 `Column.comment` 等属性的 DDL 渲染，以及反映这些注释的能力，假设使用的 SQL Server 版本支持。如果在首次连接时检测到不支持的版本，如 Azure Synapse（基于 `fn_listextendedproperty` SQL 函数的存在），则禁用注释支持，包括渲染和表注释反射，因为这两个功能都依赖于并非所有后端类型都可用的 SQL Server 存储过程和函数。

要强制启用或禁用注释支持，绕过自动检测，在 `create_engine()` 中设置参数 `supports_comments`：

```py
e = create_engine("mssql+pyodbc://u:p@dsn", supports_comments=False)
```

版本 2.0 新增了对 SQL Server 方言的表和列注释的支持，包括 DDL 生成和反射。 ## 事务隔离级别

所有 SQL Server 方言都支持通过方言特定参数`create_engine.isolation_level`（由`create_engine()`接受）以及作为传递给`Connection.execution_options()`的参数的`Connection.execution_options.isolation_level`来设置事务隔离级别。此功能通过为每个新连接发出`SET TRANSACTION ISOLATION LEVEL <level>`命令来工作。

使用 `create_engine()` 设置隔离级别：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@ms_2008",
    isolation_level="REPEATABLE READ"
)
```

使用每个连接的执行选项来设置：

```py
connection = engine.connect()
connection = connection.execution_options(
    isolation_level="READ COMMITTED"
)
```

`isolation_level` 的有效值包括：

+   `AUTOCOMMIT` - 仅适用于 pyodbc / pymssql

+   `READ COMMITTED`

+   `READ UNCOMMITTED`

+   `REPEATABLE READ`

+   `SERIALIZABLE`

+   `SNAPSHOT` - 适用于 SQL Server 特定的隔离级别

隔离级别配置还有更多选项，比如与主`Engine`相关联的“子引擎”对象，每个对象都应用不同的隔离级别设置。有关详情，请参阅设置事务隔离级别，包括 DBAPI 自动提交中的讨论。

另请参阅

设置事务隔离级别，包括 DBAPI 自动提交  ## 临时表 / 资源重置以用于连接池

SQLAlchemy `Engine` 对象使用的 `QueuePool` 连接池实现包括在连接返回池时调用 DBAPI 的`.rollback()`方法的 重置行为，虽然此回滚会清除上一个事务使用的即时状态，但它不包括更广泛的会话级状态，包括临时表以及其他服务器状态，如预编译的语句句柄和语句缓存。一个名为 `sp_reset_connection` 的未记录的 SQL Server 过程已知可解决此问题，它将重置在连接上建立的大部分会话状态，包括临时表。

要将 `sp_reset_connection` 安装为执行返回时的重置手段，可以使用 `PoolEvents.reset()` 事件挂钩，如下面的示例所示。`create_engine.pool_reset_on_return` 参数设置为 `None`，以便自定义方案可以完全替换默认行为。自定义挂钩实现在任何情况下都调用 `.rollback()`，因为通常重要的是 DBAPI 自身对提交/回滚的跟踪将与事务的状态保持一致：

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

从版本 2.0.0b3 中更改：为 `PoolEvents.reset()` 事件添加了额外的状态参数，并确保该事件对所有“重置”事件都会被调用，以便作为自定义“重置”处理程序的适当位置。以前使用 `PoolEvents.checkin()` 处理程序的方案仍然可用。

另请参阅

返回时重置 - 在 连接池 文档中

## 可空性

MSSQL 支持三种列可空性级别。默认的可空性允许空值，并在 CREATE TABLE 构造中明确指定：

```py
name VARCHAR(20) NULL
```

如果指定了 `nullable=None`，则不进行任何规定。换句话说，将使用数据库配置的默认值。这将导致：

```py
name VARCHAR(20)
```

如果 `nullable` 是 `True` 或 `False`，则列将分别为 `NULL` 或 `NOT NULL`。

## 日期/时间处理

DATE 和 TIME 是受支持的。必要时，绑定参数将转换为 datetime.datetime() 对象，大多数 MSSQL 驱动程序都需要这样做，并且如果需要的话，结果将从字符串中进行处理。 DATE 和 TIME 类型对于 MSSQL 2005 及以前的版本不可用 - 如果检测到低于 2008 的服务器版本，则将为这些类型发出 DATETIME 的 DDL。

## 大型文本/二进制类型弃用

根据 [SQL Server 2012/2014 文档](https://technet.microsoft.com/en-us/library/ms187993.aspx)，`NTEXT`、`TEXT` 和 `IMAGE` 数据类型将在将来的版本中从 SQL Server 中删除。SQLAlchemy 通常将这些类型关联到 `UnicodeText`、`TextClause` 和 `LargeBinary` 数据类型。

为了适应这种变化，为该方言添加了一个新标志 `deprecate_large_types`，如果用户没有另外设置，则将基于使用的服务器版本自动设置。此标志的行为如下：

+   当此标志为`True`时，当用于渲染 DDL 时，`UnicodeText`、`TextClause`和`LargeBinary`数据类型将分别呈现类型`NVARCHAR(max)`、`VARCHAR(max)`和`VARBINARY(max)`。这是从添加此标志开始的新行为。

+   当此标志为`False`时，当用于渲染 DDL 时，`UnicodeText`、`TextClause`和`LargeBinary`数据类型将分别呈现类型`NTEXT`、`TEXT`和`IMAGE`。这是这些类型的长期行为。

+   在建立数据库连接之前，标志始于值`None`。如果使用方言渲染 DDL 而没有设置标志，则其被解释为`False`。

+   在首次连接时，方言会检测是否使用了 SQL Server 版本 2012 或更高版本；如果标志仍然为`None`，则基于是否检测到 2012 或更高版本，将其设置为`True`或`False`。

+   当方言创建时，可以将标志设置为`True`或`False`，通常通过`create_engine()`来实现：

    ```py
    eng = create_engine("mssql+pymssql://user:pass@host/db",
                    deprecate_large_types=True)
    ```

+   在所有 SQLAlchemy 版本中，始终可以使用大写类型对象完全控制“旧”或“新”类型的渲染：`NVARCHAR`、`VARCHAR`、`VARBINARY`、`TEXT`、`NTEXT`、`IMAGE`将始终保持不变，并且始终输出确切的类型。  ## 多部分模式名称

SQL Server 模式有时需要多部分来表示其“模式”限定符，即将数据库名称和所有者名称作为单独的标记，例如`mydatabase.dbo.some_table`。可以使用`Table.schema`参数一次设置这些多部分名称。

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="mydatabase.dbo"
)
```

在执行诸如表或组件反射之类的操作时，包含点的模式参数将被拆分为单独的“数据库”和“所有者”组件，以便正确查询 SQL Server 信息模式表，因为这两个值是分开存储的。此外，在为 DDL 或 SQL 呈现模式名称时，这两个组件将被分别引用以用于区分大小写的名称和其他特殊字符。给定如下参数：

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="MyDataBase.dbo"
)
```

上述模式将呈现为`[MyDataBase].dbo`，并且在反射中，将使用“dbo”作为所有者和“MyDataBase”作为数据库名称进行反射。

要控制模式名称如何被拆分为数据库/所有者，请在名称中指定括号（在 SQL Server 中是引用字符）。下面，“所有者”将被视为`MyDataBase.dbo`，而“数据库”将为 None：

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="[MyDataBase.dbo]"
)
```

要单独指定带有特殊字符或嵌入点的数据库和所有者名称，请使用两组括号：

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="[MyDataBase.Period].[MyOwner.Dot]"
)
```

自版本 1.2 更改：SQL Server 方言现在将括号视为标识符分隔符，将模式拆分为单独的数据库和所有者标记，以允许名称本身中的点。  ## 传统模式模式

非常旧版本的 MSSQL 方言引入了这样的行为，即在 SELECT 语句中使用模式限定的表时，将自动为其设置别名；给定一个表：

```py
account_table = Table(
    'account', metadata,
    Column('id', Integer, primary_key=True),
    Column('info', String(100)),
    schema="customer_schema"
)
```

此传统呈现模式将假定“customer_schema.account”不会被 SQL 语句的所有部分接受，如下所示：

```py
>>> eng = create_engine("mssql+pymssql://mydsn", legacy_schema_aliasing=True)
>>> print(account_table.select().compile(eng))
SELECT  account_1.id,  account_1.info
FROM  customer_schema.account  AS  account_1 
```

此行为模式现在默认关闭，因为似乎没有任何作用；但是，如果传统应用程序依赖于它，则可以使用`legacy_schema_aliasing`参数来`create_engine()`，如上所示。

自版本 1.4 弃用：`legacy_schema_aliasing`标志现已弃用，并将在将来的版本中删除。  ## 聚集索引支持

MSSQL 方言支持通过`mssql_clustered`选项生成聚集索引（和主键）。此选项适用于`Index`、`UniqueConstraint`和`PrimaryKeyConstraint`。对于索引，此选项可以与`mssql_columnstore`结合使用以创建聚集列存储索引。

要生成聚集索引：

```py
Index("my_index", table.c.x, mssql_clustered=True)
```

将索引呈现为`CREATE CLUSTERED INDEX my_index ON table (x)`。

要生成聚集主键，请使用：

```py
Table('my_table', metadata,
      Column('x', ...),
      Column('y', ...),
      PrimaryKeyConstraint("x", "y", mssql_clustered=True))
```

这将例如呈现表为：

```py
CREATE TABLE my_table (x INTEGER NOT NULL, y INTEGER NOT NULL,
                       PRIMARY KEY CLUSTERED (x, y))
```

类似地，我们可以使用以下方式生成��集唯一约束：

```py
Table('my_table', metadata,
      Column('x', ...),
      Column('y', ...),
      PrimaryKeyConstraint("x"),
      UniqueConstraint("y", mssql_clustered=True),
      )
```

要显式请求非聚集主键（例如，当需要单独的聚集索引时），请使用：

```py
Table('my_table', metadata,
      Column('x', ...),
      Column('y', ...),
      PrimaryKeyConstraint("x", "y", mssql_clustered=False))
```

这将例如呈现表为：

```py
CREATE TABLE my_table (x INTEGER NOT NULL, y INTEGER NOT NULL,
                       PRIMARY KEY NONCLUSTERED (x, y))
```

## 列存储索引支持

MSSQL 方言通过 `mssql_columnstore` 选项支持列存储索引。此选项适用于 `Index`。它可以与 `mssql_clustered` 选项结合使用以创建聚集列存储索引。

生成列存储索引：

```py
Index("my_index", table.c.x, mssql_columnstore=True)
```

渲染索引为 `CREATE COLUMNSTORE INDEX my_index ON table (x)`。

要生成聚集列存储索引，请不提供列：

```py
idx = Index("my_index", mssql_clustered=True, mssql_columnstore=True)
# required to associate the index with the table
table.append_constraint(idx)
```

上述将索引渲染为 `CREATE CLUSTERED COLUMNSTORE INDEX my_index ON table`。

版本 2.0.18 中新增。

## MSSQL 特定的索引选项

除了聚集外，MSSQL 方言还支持其他特殊选项用于 `Index`。

### 包括

`mssql_include` 选项为给定的字符串名称渲染 INCLUDE(colname)：

```py
Index("my_index", table.c.x, mssql_include=['y'])
```

将索引渲染为 `CREATE INDEX my_index ON table (x) INCLUDE (y)`

### 过滤索引

`mssql_where` 选项为给定的字符串名称渲染 WHERE(condition)：

```py
Index("my_index", table.c.x, mssql_where=table.c.x > 10)
```

将索引渲染为 `CREATE INDEX my_index ON table (x) WHERE x > 10`。

版本 1.3.4 中新增。

### 索引排序

索引排序可通过功能表达式实现，例如：

```py
Index("my_index", table.c.x.desc())
```

将索引渲染为 `CREATE INDEX my_index ON table (x DESC)`

另请参阅

功能索引

## 兼容性级别

MSSQL 支持在数据库级别设置兼容性级别的概念。例如，可以在运行在 SQL2005 数据库服务器上的数据库上运行与 SQL2000 兼容的数据库。`server_version_info` 将始终返回数据库服务器版本信息（在本例中为 SQL2005），而不是兼容性级别信息。因此，如果在向后兼容模式下运行，SQLAlchemy 可能会尝试使用数据库服务器无法解析的 T-SQL 语句。

## 触发器

SQLAlchemy 默认使用 OUTPUT INSERTED 来获取通过 IDENTITY 列或其他服务器端默认生成的新主键值。MS-SQL 不允许在具有触发器的表上使用 OUTPUT INSERTED。要在每个具有触发器的 `Table` 上禁用 OUTPUT INSERTED 的使用，为其指定 `implicit_returning=False`：

```py
Table('mytable', metadata,
    Column('id', Integer, primary_key=True),
    # ...,
    implicit_returning=False
)
```

声明形式：

```py
class MyClass(Base):
    # ...
    __table_args__ = {'implicit_returning':False}
```  ## 行数支持 / ORM 版本控制

SQL Server 驱动程序可能有限的能力返回从 UPDATE 或 DELETE 语句中更新的行数。

截至目前，PyODBC 驱动程序无法在使用 OUTPUT INSERTED 时返回行数。因此，之前的 SQLAlchemy 版本对于依赖于准确行数以将版本号与匹配行匹配的功能（如“ORM 版本控制”功能）存在限制。

SQLAlchemy 2.0 现在针对这些特定用例基于返回的行数手动检索“rowcount”；因此，虽然驱动程序仍然具有此限制，但 ORM 版本功能不再受其影响。从 SQLAlchemy 2.0.5 开始，ORM 版本控制已完全重新启用 pyodbc 驱动程序。

在版本 2.0.5 中更改：为 pyodbc 驱动程序恢复了 ORM 版本控制支持。之前，在 ORM 刷新期间会发出警告，说明不支持版本控制。

## 启用快照隔离

SQL Server 具有默认的事务隔离模式，它锁定整个表，并导致即使是轻度并发的应用程序也具有长时间的持有锁定和频繁的死锁。推荐为整个数据库启用快照隔离以支持现代的并发级别。这通过在 SQL 提示符下执行以下 ALTER DATABASE 命令来完成：

```py
ALTER DATABASE MyDatabase SET ALLOW_SNAPSHOT_ISOLATION ON

ALTER DATABASE MyDatabase SET READ_COMMITTED_SNAPSHOT ON
```

关于 SQL Server 快照隔离的背景信息，请参阅 [`msdn.microsoft.com/en-us/library/ms175095.aspx`](https://msdn.microsoft.com/en-us/library/ms175095.aspx)。

## SQL Server SQL 构造

| 对象名称 | 描述 |
| --- | --- |
| try_cast(expression, type_) | 为支持的后端生成一个 `TRY_CAST` 表达式；这是一个 `CAST`，对于不可转换的转换返回 NULL。 |

```py
function sqlalchemy.dialects.mssql.try_cast(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → TryCast[_T]
```

为支持它的后端生成一个 `TRY_CAST` 表达式；这是一个 `CAST`，对于不可转换的转换返回 NULL。

在 SQLAlchemy 中，此结构仅由 SQL Server 方言支持，并且如果在其他包含的后端上使用，将引发 `CompileError`。但是，第三方后端也可能支持此结构。

提示

由于 `try_cast()` 起源于 SQL Server 方言，因此可以从 `sqlalchemy.` 以及 `sqlalchemy.dialects.mssql` 导入。

`try_cast()` 返回一个 `TryCast` 实例，并且通常行为类似于 `Cast` 结构；在 SQL 层面，`CAST` 和 `TRY_CAST` 的区别在于 `TRY_CAST` 对于不可转换的表达式，如将字符串 `"hi"` 转换为整数值，将返回 NULL。

例如：

```py
from sqlalchemy import select, try_cast, Numeric

stmt = select(
    try_cast(product_table.c.unit_price, Numeric(10, 4))
)
```

上述内容在 Microsoft SQL Server 上呈现为：

```py
SELECT TRY_CAST (product_table.unit_price AS NUMERIC(10, 4))
FROM product_table
```

新版本 2.0.14 中：`try_cast()` 已从 SQL Server 方言广义化为一个可能由其他方言支持的通用结构。

## SQL Server 数据类型

与所有 SQLAlchemy 方言一样，所有已知与 SQL Server 有效的大写类型都可以从顶级方言导入，无论它们是来自`sqlalchemy.types`还是来自本地方言：

```py
from sqlalchemy.dialects.mssql import (
    BIGINT,
    BINARY,
    BIT,
    CHAR,
    DATE,
    DATETIME,
    DATETIME2,
    DATETIMEOFFSET,
    DECIMAL,
    DOUBLE_PRECISION,
    FLOAT,
    IMAGE,
    INTEGER,
    JSON,
    MONEY,
    NCHAR,
    NTEXT,
    NUMERIC,
    NVARCHAR,
    REAL,
    SMALLDATETIME,
    SMALLINT,
    SMALLMONEY,
    SQL_VARIANT,
    TEXT,
    TIME,
    TIMESTAMP,
    TINYINT,
    UNIQUEIDENTIFIER,
    VARBINARY,
    VARCHAR,
)
```

以下是特定于 SQL Server 或具有 SQL Server 特定构造参数的类型：

| 对象名称 | 描述 |
| --- | --- |
| BIT | MSSQL BIT 类型。 |
| DATETIME2 |  |
| DATETIMEOFFSET |  |
| DOUBLE_PRECISION | SQL Server DOUBLE PRECISION 数据类型。 |
| IMAGE |  |
| JSON | MSSQL JSON 类型。 |
| MONEY |  |
| NTEXT | MSSQL NTEXT 类型，用于最多 2³⁰ 个字符的可变长度 Unicode 文本。 |
| REAL | SQL Server REAL 数据类型。 |
| ROWVERSION | 实现 SQL Server ROWVERSION 类型。 |
| SMALLDATETIME |  |
| SMALLMONEY |  |
| SQL_VARIANT |  |
| TIME |  |
| TIMESTAMP | 实现 SQL Server TIMESTAMP 类型。 |
| TINYINT |  |
| UNIQUEIDENTIFIER |  |
| XML | MSSQL XML 类型。 |

```py
class sqlalchemy.dialects.mssql.BIT
```

MSSQL BIT 类型。

pyodbc 和 pymssql 都将 BIT 列的值作为 Python <class ‘bool’>返回，因此只需子类化 Boolean。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mssql.BIT` (`sqlalchemy.types.Boolean`)

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

*继承自* `sqlalchemy.types.Boolean.__init__` *方法的* `Boolean`

构造一个布尔值。

参数：

+   `create_constraint` –

    默认为 False。如果布尔值生成为 int/smallint，则还会在表上创建一个 CHECK 约束，确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有显式名称，以支持模式管理问题。这可以通过设置`Boolean.name`参数或设置适当的命名约定来实现；有关背景信息，请参阅配置约束命名约定。

    从版本 1.4 开始更改：- 此标志现在默认为 False，表示对于非本地枚举类型不会生成 CHECK 约束。

+   `name` – 如果生成 CHECK 约束，请指定约束的名称。

```py
class sqlalchemy.dialects.mssql.CHAR
```

SQL CHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.CHAR` (`sqlalchemy.types.String`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `sqlalchemy.types.String.__init__` *方法的* `String`

创建一个持有字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列的长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用`length`，如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是特定于数据库的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode` 或 `UnicodeText` 数据类型应该用于预期存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.DATETIME2
```

**类签名**

类 `sqlalchemy.dialects.mssql.DATETIME2` (`sqlalchemy.dialects.mssql.base._DateTimeBase`, `sqlalchemy.types.DateTime`)

```py
class sqlalchemy.dialects.mssql.DATETIMEOFFSET
```

**类签名**

类 `sqlalchemy.dialects.mssql.DATETIMEOFFSET` (`sqlalchemy.dialects.mssql.base._DateTimeBase`, `sqlalchemy.types.DateTime`)

```py
class sqlalchemy.dialects.mssql.DOUBLE_PRECISION
```

SQL Server DOUBLE PRECISION 数据类型。

新版本 2.0.11 中新增。

**类签名**

类 `sqlalchemy.dialects.mssql.DOUBLE_PRECISION` (`sqlalchemy.types.DOUBLE_PRECISION`)

```py
class sqlalchemy.dialects.mssql.IMAGE
```

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.IMAGE` (`sqlalchemy.types.LargeBinary`)

```py
method __init__(length: int | None = None)
```

*继承自* `sqlalchemy.types.LargeBinary.__init__` *方法的* `LargeBinary`

构造一个 LargeBinary 类型。

参数：

**length** – 可选，用于 DDL 语句的列长度，用于那些接受长度的二进制类型，例如 MySQL BLOB 类型。

```py
class sqlalchemy.dialects.mssql.JSON
```

MSSQL JSON 类型。

MSSQL 支持 JSON 格式的数据，自 SQL Server 2016 起。

DDL 级别的 `JSON` 数据类型将以 `NVARCHAR(max)` 形式表示数据类型，但也提供了 JSON 级别的比较函数以及 Python 强制转换行为。

自动使用 `JSON` 每当基础 `JSON` 数据类型针对 SQL Server 后端使用时。

另请参阅

`JSON` - 通用跨平台 JSON 数据类型的主要文档。

`JSON` 类型支持将 JSON 值持久化存储，以及通过将操作适配到数据库级别的 `JSON_VALUE` 或 `JSON_QUERY` 函数来提供的核心索引操作，以支持 `JSON` 数据类型。

SQL Server `JSON` 类型在查询 JSON 对象元素时必然使用 `JSON_QUERY` 和 `JSON_VALUE` 函数。这两个函数有一个主要限制，即它们根据要返回的对象类型是**互斥的**。`JSON_QUERY` 函数**仅**返回 JSON 字典或列表，但不返回单个字符串、数值或布尔值元素；`JSON_VALUE` 函数**仅**返回单个字符串、数值或布尔值元素。**如果它们没有针对正确的预期值使用，这两个函数都会返回 NULL 或引发错误**。

为了处理这个尴尬的要求，索引访问规则如下：

1.  当从一个 JSON 中提取一个子元素，该 JSON 本身是一个 JSON 字典或列表时，应使用 `Comparator.as_json()` 访问器：

    ```py
    stmt = select(
        data_table.c.data["some key"].as_json()
    ).where(
        data_table.c.data["some key"].as_json() == {"sub": "structure"}
    )
    ```

1.  从 JSON 中提取平面布尔值、字符串、整数或浮点数的子元素时，使用以下适当的方法之一： `Comparator.as_boolean()`, `Comparator.as_string()`, `Comparator.as_integer()`, `Comparator.as_float()`:

    ```py
    stmt = select(
        data_table.c.data["some key"].as_string()
    ).where(
        data_table.c.data["some key"].as_string() == "some string"
    )
    ```

版本 1.4 中的新功能。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.JSON` (`sqlalchemy.types.JSON`)

```py
method __init__(none_as_null: bool = False)
```

*继承自* `JSON` *方法* `sqlalchemy.types.JSON.__init__`

构造一个 `JSON` 类型。

参数：

**none_as_null=False** –

如果为 True，则将值`None`持久化为 SQL NULL 值，而不是`null`的 JSON 编码。注意，当此标志为 False 时，`null()` 构造仍然可以用于持久化 NULL 值，可以直接将其作为参数值传递，该值由 `JSON` 类型特别解释为 SQL NULL：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"data": null()})
```

注意

`JSON.none_as_null` 不适用于传递给 `Column.default` 和 `Column.server_default` 的值；这些参数的传递值为 `None` 意味着“没有默认值”。

此外，当用于 SQL 比较表达式时，Python 值 `None` 仍然指代 SQL null，而不是 JSON NULL。`JSON.none_as_null` 标志明确指的是在 INSERT 或 UPDATE 语句中值的**持久化**。 `JSON.NULL` 值应用于希望与 JSON null 进行比较的 SQL 表达式。

另请参阅

`JSON.NULL`

```py
class sqlalchemy.dialects.mssql.MONEY
```

**类签名**

类 `sqlalchemy.dialects.mssql.MONEY` (`sqlalchemy.types.TypeEngine`)

```py
class sqlalchemy.dialects.mssql.NCHAR
```

SQL NCHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.NCHAR` (`sqlalchemy.types.Unicode`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `sqlalchemy.types.String.__init__` *方法*

创建一个保存字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是特定于数据库的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.NTEXT
```

MSSQL NTEXT 类型，用于最多 2³⁰ 个字符的可变长度 Unicode 文本。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.NTEXT` (`sqlalchemy.types.UnicodeText`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `sqlalchemy.types.String.__init__` *方法*

创建一个保存字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是特定于数据库的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行��现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.NVARCHAR
```

SQL NVARCHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.NVARCHAR` (`sqlalchemy.types.Unicode`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个字符串持有类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且当包含没有长度的 `VARCHAR` 时，将在发出 `CREATE TABLE` DDL 时引发异常。该值被解释为字节还是字符是特定于数据库的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.REAL
```

SQL Server REAL 数据类型。

**类签名**

类`sqlalchemy.dialects.mssql.REAL` (`sqlalchemy.types.REAL`)

```py
class sqlalchemy.dialects.mssql.ROWVERSION
```

实现 SQL Server ROWVERSION 类型。

ROWVERSION 数据类型是 TIMESTAMP 数据类型的 SQL Server 同义词，但当前 SQL Server 文档建议将 ROWVERSION 用于未来的新数据类型。

ROWVERSION 数据类型不会从数据库中反映出来，返回的数据类型将是 `TIMESTAMP`。

这是一种只读数据类型，不支持插入值。

版本 1.2 中的新功能。

另请参阅

`TIMESTAMP`

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.ROWVERSION` （`sqlalchemy.dialects.mssql.base.TIMESTAMP`）

```py
method __init__(convert_int=False)
```

*从* `TIMESTAMP` *的* `sqlalchemy.dialects.mssql.base.TIMESTAMP.__init__` *方法继承*

构造一个 TIMESTAMP 或 ROWVERSION 类型。

参数：

**convert_int** – 如果为 True，则在读取时将二进制整数值转换为整数。

新版本 1.2。

```py
class sqlalchemy.dialects.mssql.SMALLDATETIME
```

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.SMALLDATETIME` （`sqlalchemy.dialects.mssql.base._DateTimeBase`，`sqlalchemy.types.DateTime`）

```py
method __init__(timezone: bool = False)
```

*从* `DateTime` *的* `sqlalchemy.types.DateTime.__init__` *方法继承*

构造一个新的`DateTime`。

参数：

**时区** – 布尔值。指示日期时间类型是否应在仅在**基础日期/时间持有类型上可用时启用时区支持**。建议在使用此标志时直接使用`TIMESTAMP`数据类型，因为一些数据库包括与时区功能的 TIMESTAMP 数据类型不同的单独的通用日期/时间持有类型，如 Oracle。

```py
class sqlalchemy.dialects.mssql.SMALLMONEY
```

**类签名**

类 `sqlalchemy.dialects.mssql.SMALLMONEY` （`sqlalchemy.types.TypeEngine`）

```py
class sqlalchemy.dialects.mssql.SQL_VARIANT
```

**类签名**

类 `sqlalchemy.dialects.mssql.SQL_VARIANT` （`sqlalchemy.types.TypeEngine`）

```py
class sqlalchemy.dialects.mssql.TEXT
```

SQL TEXT 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.TEXT` （`sqlalchemy.types.Text`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个字符串持有类型。

参数：

+   `length` – 可选项，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含一个没有长度的`VARCHAR`，则在发出`CREATE TABLE`DDL 时会引发异常。值是作为字节还是字符解释的是特定于数据库的。

+   `collation` –

    可选项，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库中使用正确的类型。

```py
class sqlalchemy.dialects.mssql.TIME
```

**类签名**

类`sqlalchemy.dialects.mssql.TIME`（`sqlalchemy.types.TIME`）

```py
class sqlalchemy.dialects.mssql.TIMESTAMP
```

实现 SQL Server TIMESTAMP 类型。

请注意，这与 SQL 标准 TIMESTAMP 类型完全不同，SQL Server 不支持该类型。它是一个只读数据类型，不支持插入值。

新功能在版本 1.2 中引入。

另请参阅

`ROWVERSION`

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mssql.TIMESTAMP`（`sqlalchemy.types._Binary`）

```py
method __init__(convert_int=False)
```

构造 TIMESTAMP 或 ROWVERSION 类型。

参数：

**convert_int** – 如果为 True，则二进制整数值将在读取时转换为整数。

新功能在版本 1.2 中引入。

```py
class sqlalchemy.dialects.mssql.TINYINT
```

**类签名**

类`sqlalchemy.dialects.mssql.TINYINT`（`sqlalchemy.types.Integer`）

```py
class sqlalchemy.dialects.mssql.UNIQUEIDENTIFIER
```

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mssql.UNIQUEIDENTIFIER`（`sqlalchemy.types.Uuid`）

```py
method __init__(as_uuid: bool = True)
```

构造 `UNIQUEIDENTIFIER` 类型。

参数：

**as_uuid=True** –

如果为 True，则值将被解释为 Python uuid 对象，通过 DBAPI 转换为/从字符串。

```py
class sqlalchemy.dialects.mssql.VARBINARY
```

MSSQL VARBINARY 类型。

该类型为核心的`VARBINARY`类型添加了额外的功能，包括“deprecate_large_types”模式，其中会渲染`VARBINARY(max)`或 IMAGE，以及 SQL Server 的`FILESTREAM`选项。

另请参见

大型文本/二进制类型弃用

**类签名**

类 `sqlalchemy.dialects.mssql.VARBINARY` (`sqlalchemy.types.VARBINARY`, `sqlalchemy.types.LargeBinary`)

```py
method __init__(length=None, filestream=False)
```

构造一个 VARBINARY 类型。

参数：

+   `length` – 可选参数，在 DDL 语句中用于列长度，用于那些接受长度参数的二进制类型，比如 MySQL 的 BLOB 类型。

+   `filestream=False` –

    如果为 True，在表定义中会渲染`FILESTREAM`关键字。在这种情况下，`length`必须为`None`或者`'max'`。

    新于 1.4.31 版本。

```py
class sqlalchemy.dialects.mssql.VARCHAR
```

SQL VARCHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.VARCHAR` (`sqlalchemy.types.String`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个保存字符串的类型。

参数：

+   `length` – 可选参数，在 DDL 和 CAST 表达式中用于列长度。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要 DDL 中使用长度，并且如果包括了没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释的，这取决于数据库。

+   `collation` –

    可选参数，在 DDL 和 CAST 表达式中用于列级别的排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode`或者`UnicodeText`数据类型应该用于预期存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.XML
```

MSSQL XML 类型。

这是一个占位符类型，用于反射目的，不包括任何 Python 端数据类型支持。它也不支持额外的参数，比如“CONTENT”、“DOCUMENT”、“xml_schema_collection”。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.XML` (`sqlalchemy.types.Text`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选项，用于 DDL 和 CAST 表达式中列的长度。如果不会发出`CREATE TABLE`语句，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包括了没有长度的`VARCHAR`，则在发出`CREATE TABLE`DDL 时会引发异常。值是按字节还是按字符解释，取决于数据库。

+   `collation` –

    可选项，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

## PyODBC

通过 PyODBC 驱动程序支持 Microsoft SQL Server 数据库。

### DBAPI

PyODBC 的文档和下载信息（如果适用）可在以下网址获取：[`pypi.org/project/pyodbc/`](https://pypi.org/project/pyodbc/)

### 连接

连接字符串：

```py
mssql+pyodbc://<username>:<password>@<dsnname>
```

### 连接到 PyODBC

此处的 URL 将被翻译为 PyODBC 连接字符串，详见[ConnectionStrings](https://code.google.com/p/pyodbc/wiki/ConnectionStrings)。

#### DSN 连接

ODBC 中的 DSN 连接意味着在客户端机器上配置了预先存在的 ODBC 数据源。然后，应用程序指定此数据源的名称，其中包括诸如正在使用的特定 ODBC 驱动程序以及数据库的网络地址等细节。假设在客户端上配置了数据源，则基本的基于 DSN 的连接如下所示：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@some_dsn")
```

将上述内容传递给 PyODBC 的连接字符串如下：

```py
DSN=some_dsn;UID=scott;PWD=tiger
```

如果省略了用户名和密码，则 DSN 表单还将向 ODBC 字符串添加`Trusted_Connection=yes`指令。

#### 主机名连接

PyODBC 也支持基于主机名的连接。这通常比 DSN 更容易使用，并且具有另一个优势，即可以在 URL 中本地指定要连接到的特定数据库名称，而不是将其固定为数据源配置的一部分。

在使用主机名连接时，还必须在 URL 的查询参数中指定驱动程序名称。由于这些名称通常包含空格，因此必须对名称进行 URL 编码，这意味着使用加号代替空格：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=ODBC+Driver+17+for+SQL+Server")
```

`driver`关键字对于 pyodbc 方言非常重要，必须以小写形式指定。

查询字符串中传递的任何其他名称都将通过 pyodbc 连接字符串传递，例如 `authentication`、`TrustServerCertificate` 等。 多个关键字参数必须用与号（`&`）分隔；这些参数在生成 pyodbc 连接字符串时将被转换为分号：

```py
e = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?"
    "driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes"
    "&authentication=ActiveDirectoryIntegrated"
)
```

可以使用 `URL` 构造相等的 URL：

```py
from sqlalchemy.engine import URL
connection_url = URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssql2017",
    port=1433,
    database="test",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "TrustServerCertificate": "yes",
        "authentication": "ActiveDirectoryIntegrated",
    },
)
```

#### 通过确切的 Pyodbc 字符串传递

PyODBC 连接字符串也可以直接以 pyodbc 的格式发送，如[PyODBC 文档](https://github.com/mkleehammer/pyodbc/wiki/Connecting-to-databases)中所述，使用参数 `odbc_connect`。 `URL` 对象可以帮助简化此过程：

```py
from sqlalchemy.engine import URL
connection_string = "DRIVER={SQL Server Native Client 10.0};SERVER=dagger;DATABASE=test;UID=user;PWD=password"
connection_url = URL.create("mssql+pyodbc", query={"odbc_connect": connection_string})

engine = create_engine(connection_url)
```

#### 使用访问令牌连接到数据库

一些数据库服务器仅允许使用访问令牌进行登录。 例如，SQL Server 允许使用 Azure Active Directory 令牌连接到数据库。 这需要使用 `azure-identity` 库创建凭据对象。 关于身份验证步骤的更多信息可以在 [Microsoft 文档](https://docs.microsoft.com/en-us/azure/developer/python/azure-sdk-authenticate?tabs=bash)中找到。

获得引擎后，每次请求连接都需要将凭据发送到 `pyodbc.connect`。 一种方法是在引擎上设置事件侦听器，该事件侦听器将凭据令牌添加到方言的连接调用中。 关于这一点的更多讨论可以在 生成动态身份验证令牌中找到。 尤其对于 SQL Server，这将作为由 Microsoft 描述的 ODBC 连接属性传递[的数据结构](https://docs.microsoft.com/en-us/sql/connect/odbc/using-azure-active-directory#authenticating-with-an-access-token)。

下面的代码片段将创建一个引擎，该引擎使用 Azure 凭据连接到 Azure SQL 数据库：

```py
import struct
from sqlalchemy import create_engine, event
from sqlalchemy.engine.url import URL
from azure import identity

SQL_COPT_SS_ACCESS_TOKEN = 1256  # Connection option for access tokens, as defined in msodbcsql.h
TOKEN_URL = "https://database.windows.net/"  # The token URL for any Azure SQL database

connection_string = "mssql+pyodbc://@my-server.database.windows.net/myDb?driver=ODBC+Driver+17+for+SQL+Server"

engine = create_engine(connection_string)

azure_credentials = identity.DefaultAzureCredential()

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    # remove the "Trusted_Connection" parameter that SQLAlchemy adds
    cargs[0] = cargs[0].replace(";Trusted_Connection=Yes", "")

    # create token credential
    raw_token = azure_credentials.get_token(TOKEN_URL).token.encode("utf-16-le")
    token_struct = struct.pack(f"<I{len(raw_token)}s", len(raw_token), raw_token)

    # apply it to keyword arguments
    cparams["attrs_before"] = {SQL_COPT_SS_ACCESS_TOKEN: token_struct}
```

提示

当没有用户名或密码时，SQLAlchemy pyodbc 方言当前会添加 `Trusted_Connection` 令牌。 根据 Microsoft 的[用于 Azure 访问令牌的文档](https://docs.microsoft.com/en-us/sql/connect/odbc/using-azure-active-directory#authenticating-with-an-access-token)，当使用访问令牌时，连接字符串不得包含 `UID`、`PWD`、`Authentication` 或 `Trusted_Connection` 参数，需要将其删除。 #### 在 Azure Synapse Analytics 上避免事务相关的异常

Azure Synapse Analytics 在事务处理方面与普通 SQL Server 有显着差异；在某些情况下，Synapse 事务中的错误可能导致服务器端任意终止，从而导致 DBAPI 的 `.rollback()` 方法 (以及 `.commit()`) 失败。该问题阻止了允许 `.rollback()` 在没有事务存在时静默通过的常规 DBAPI 合同，因为驱动程序不期望出现此条件。此故障的症状是，在某些操作失败后尝试发出 `.rollback()` 后，异常消息类似于‘No corresponding transaction found. (111214)’。

可以通过向 SQL Server 方言传递 `ignore_no_transaction_on_rollback=True` 参数来处理此特定情况，方法是通过`create_engine()` 函数如下所示：

```py
engine = create_engine(connection_url, ignore_no_transaction_on_rollback=True)
```

使用上述参数，方言将捕获在 `connection.rollback()` 期间引发的 `ProgrammingError` 异常，并在错误消息中包含代码 `111214` 时发出警告，但不会引发异常。

版本 1.4.40 中的新功能：添加了 `ignore_no_transaction_on_rollback=True` 参数。

#### 为 Azure SQL 数据仓库 (DW) 连接启用自动提交

Azure SQL 数据仓库不支持事务，这可能会导致 SQLAlchemy 的“autobegin”（以及隐式提交/回滚）行为出现问题。我们可以通过在 pyodbc 和 engine 级别启用自动提交来避免这些问题：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="dw.azure.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 17 for SQL Server",
        "autocommit": "True",
    },
)

engine = create_engine(connection_url).execution_options(
    isolation_level="AUTOCOMMIT"
)
```

#### 避免将大字符串参数发送为 TEXT/NTEXT

出于历史原因，默认情况下，Microsoft 的 SQL Server ODBC 驱动程序将长字符串参数（大于 4000 个 SBCS 字符或 2000 个 Unicode 字符）发送为 TEXT/NTEXT 值。多年来，TEXT 和 NTEXT 已经被弃用，并且开始在新版本的 SQL_Server/Azure 中引起兼容性问题。例如，参见[此问题](https://github.com/mkleehammer/pyodbc/issues/835)。

从 ODBC 驱动程序 18 开始，我们可以通过 `LongAsMax=Yes` 连接字符串参数覆盖传统行为，并将长字符串作为 varchar(max)/nvarchar(max) 传递：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssqlserver.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "LongAsMax": "Yes",
    },
)
```

### Pyodbc 连接池 / 连接关闭行为

PyODBC 默认使用内部[连接池](https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module#pooling)，这意味着连接的生命周期比在 SQLAlchemy 本身中更长。由于 SQLAlchemy 有自己的连接池行为，通常最好禁用此行为。此行为只能在创建任何连接之前在 PyODBC 模块级别**全局**禁用：

```py
import pyodbc

pyodbc.pooling = False

# don't use the engine before pooling is set to False
engine = create_engine("mssql+pyodbc://user:pass@dsn")
```

如果将此变量保留在默认值 `True`，**应用程序将继续保持活动数据库连接**，即使 SQLAlchemy 引擎本身完全丢弃连接或引擎被处理掉。

另请参阅

[连接池](https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module#pooling) - 在 PyODBC 文档中。

### 驱动程序 / Unicode 支持

PyODBC 最适合与微软 ODBC 驱动程序一起使用，特别是在 Python 2 和 Python 3 上都支持 Unicode 的领域。

不建议在 Linux 或 OSX 上使用 FreeTDS ODBC 驱动程序与 PyODBC 一起使用；在这个领域，包括在微软为 Linux 和 OSX 提供 ODBC 驱动程序之前，历史上存在许多与 Unicode 相关的问题。现在微软为所有平台提供驱动程序，对于 PyODBC 支持，建议使用这些驱动程序。FreeTDS 仍然适用于非 ODBC 驱动程序，例如 pymssql，在那里它的工作非常出色。

### 行计数支持

至于 Pyodbc 与 SQLAlchemy ORM 的“版本化行”功能之前的限制，在 SQLAlchemy 2.0.5 版中已经解决。请参阅 Rowcount Support / ORM Versioning 中的说明。

### 快速执行多次模式

PyODBC 驱动程序包括对执行 DBAPI `executemany()` 调用时大大减少往返次数的“快速执行多次”模式的支持，当使用微软 ODBC 驱动程序时，对于**内存中适合的有限大小批次**。通过在 DBAPI 游标上设置属性 `.fast_executemany` 来启用此功能，当要使用 executemany 调用时。SQLAlchemy PyODBC SQL Server 方言通过将 `fast_executemany` 参数传递给 `create_engine()` 来支持此参数，仅当使用**微软 ODBC 驱动程序时**：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
    fast_executemany=True)
```

从版本 2.0.9 开始更改：- `fast_executemany` 参数现在具有其预期的效果，这使得 PyODBC 功能在执行具有多个参数集的所有 INSERT 语句时生效，不包括 RETURNING。之前，SQLAlchemy 2.0 的 insertmanyvalues 功能通常会导致即使指定了，也大多数情况下不使用`fast_executemany`。

1.3 版中的新功能。

另请参阅

[快速执行多次](https://github.com/mkleehammer/pyodbc/wiki/Features-beyond-the-DB-API#fast_executemany) - 在 github ### 设置输入大小支持

从版本 2.0 开始，pyodbc `cursor.setinputsizes()` 方法用于所有语句执行，除了当 fast_executemany=True 时，不支持`cursor.executemany()`调用（假设 insertmanyvalues 已启用，“fastexecutemany”不管怎样都不会对 INSERT 语句产生影响）。

通过将 `use_setinputsizes=False` 传递给 `create_engine()` 可以禁用`cursor.setinputsizes()`的使用。

当 `use_setinputsizes` 保持默认值 `True` 时，可以通过使用 `DialectEvents.do_setinputsizes()` 钩子来自定义传递给 `cursor.setinputsizes()` 的每种类型符号。请参阅该方法以获取用法示例。

从 2.0 版本开始更改：mssql+pyodbc 方言现在默认为在所有语句执行中使用`use_setinputsizes=True`，但 fast_executemany=True 时除外，快速执行多次`cursor.executemany()` 调用。该行为可以通过将 `use_setinputsizes=False` 传递给 `create_engine()` 来关闭。 ## pymssql

通过 pymssql 驱动程序支持 Microsoft SQL Server 数据库。

### 连接

连接字符串：

```py
mssql+pymssql://<username>:<password>@<freetds_name>/?charset=utf8
```

pymssql 是一个提供围绕 [FreeTDS](https://www.freetds.org/) 的 Python DBAPI 接口的 Python 模块。

从 2.0.5 版本开始更改：pymssql 已恢复到 SQLAlchemy 的持续集成测试  ## aioodbc

通过 aioodbc 驱动程序支持 Microsoft SQL Server 数据库。

### DBAPI

aioodbc 的文档和下载信息（如果适用）可在此处获取：[`pypi.org/project/aioodbc/`](https://pypi.org/project/aioodbc/)

### 连接

连接字符串：

```py
mssql+aioodbc://<username>:<password>@<dsnname>
```

以 asyncio 样式支持 SQL Server 数据库，使用 aioodbc 驱动程序，它本身是 pyodbc 的线程包装器。

从 2.0.23 版本开始新增：添加了在 pyodbc 和通用 aio* 方言架构之上构建的 mssql+aioodbc 方言。

使用特殊的 asyncio 中介层，aioodbc 方言可用作 SQLAlchemy asyncio 扩展包的后端。

该驱动程序的大多数行为和注意事项与在 SQL Server 上使用的 pyodbc 方言相同；有关一般背景，请参阅 PyODBC。

该方言通常仅应与 `create_async_engine()` 引擎创建函数一起使用；否则，连接样式与在 pyodbc 部分文档中记录的相同：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine(
    "mssql+aioodbc://scott:tiger@mssql2017:1433/test?"
    "driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes"
)
```

支持 Microsoft SQL Server 数据库。

以下表格总结了数据库发布版本的当前支持级别。

**支持的 Microsoft SQL Server 版本**

| 支持类型 | 版本 |
| --- | --- |
| 在 CI 中进行全面测试 | 2017 |
| 普通支持 | 2012+ |
| 尽力而为 | 2005+ |

## DBAPI 支持

提供以下方言/DBAPI 选项。请参阅各个 DBAPI 部分以获取连接信息。

+   PyODBC

+   pymssql

+   aioodbc

## 外部方言

除了具有本地 SQLAlchemy 支持的上述 DBAPI 层之外，还有用于其他与 SQL Server 兼容的 DBAPI 层的第三方方言。请参阅 方言 页面上的“外部方言”列表。

## 自动递增行为 / IDENTITY 列

SQL Server 使用`IDENTITY`构造提供所谓的“自动增量”行为，该构造可以放置在表中的任何单个整数列上。SQLAlchemy 将`IDENTITY`考虑在其整数主键列的默认“autoincrement”行为中，该行为在`Column.autoincrement`中描述。这意味着默认情况下，`Table`中的第一个整数主键列将被视为标识列 - 除非它与`Sequence`相关联 - 并且将生成 DDL 如下：

```py
from sqlalchemy import Table, MetaData, Column, Integer

m = MetaData()
t = Table('t', m,
        Column('id', Integer, primary_key=True),
        Column('x', Integer))
m.create_all(engine)
```

上述示例将生成 DDL 如下：

```py
CREATE  TABLE  t  (
  id  INTEGER  NOT  NULL  IDENTITY,
  x  INTEGER  NULL,
  PRIMARY  KEY  (id)
)
```

对于不希望使用此默认生成的`IDENTITY`的情况，在第一个整数主键列上指定`Column.autoincrement`标志为`False`：

```py
m = MetaData()
t = Table('t', m,
        Column('id', Integer, primary_key=True, autoincrement=False),
        Column('x', Integer))
m.create_all(engine)
```

要将`IDENTITY`关键字添加到非主键列，请在所需的`Column`对象上指定`Column.autoincrement`标志为`True`，并确保在任何整数主键列上将`Column.autoincrement`设置为`False`：

```py
m = MetaData()
t = Table('t', m,
        Column('id', Integer, primary_key=True, autoincrement=False),
        Column('x', Integer, autoincrement=True))
m.create_all(engine)
```

自版本 1.4 更改：在`Column`中添加了`Identity`构造，用于指定`IDENTITY`的起始值和增量参数。这些参数取代了使用`Sequence`对象来指定这些值。

自版本 1.4 弃用：`Column`的`mssql_identity_start`和`mssql_identity_increment`参数已弃用，应该用`Identity`对象替换。指定两种配置`IDENTITY`的方式将导致编译错误。这些选项也不再作为`Inspector.get_columns()`中`dialect_options`键的一部分返回。请改为使用`identity`键中的信息。

自版本 1.3 起弃用：使用`Sequence`指定 IDENTITY 特性已被弃用，并将在将来的版本中删除。请使用`Identity`对象参数`Identity.start`和`Identity.increment`。

从版本 1.4 开始更改：移除了使用`Sequence`对象修改 IDENTITY 特性的能力。现在，`Sequence`对象仅操作真正的 T-SQL SEQUENCE 类型。

注意

表上只能有一个 IDENTITY 列。当使用`autoincrement=True`启用 IDENTITY 关键字时，SQLAlchemy 不会阻止多个列同时指定该选项。相反，SQL Server 数据库将拒绝`CREATE TABLE`语句。

注意

尝试为标记为 IDENTITY 的列提供值的 INSERT 语句将被 SQL Server 拒绝。为了接受该值，必须启用会话级选项“SET IDENTITY_INSERT”。当使用核心`Insert`构造时，SQLAlchemy SQL Server 方言将在执行指定 IDENTITY 列的值时自动执行此操作；如果执行为 IDENTITY 列指定了一个值，则“IDENTITY_INSERT”选项将在该语句调用的范围内启用。然而，这种情况的性能不高，不应该依赖于常规使用。如果表实际上不需要 IDENTITY 行为在其整数主键列中，创建表时应禁用该关键字，方法是确保`autoincrement=False`被设置。

### 控制“开始”和“增量”

通过将参数`Identity.start`和`Identity.increment`传递给`Identity`对象提供了对“开始”和“增量”值的特定控制：

```py
from sqlalchemy import Table, Integer, Column, Identity

test = Table(
    'test', metadata,
    Column(
        'id',
        Integer,
        primary_key=True,
        Identity(start=100, increment=10)
    ),
    Column('name', String(20))
)
```

上述`Table`对象的 CREATE TABLE 将是：

```py
CREATE  TABLE  test  (
  id  INTEGER  NOT  NULL  IDENTITY(100,10)  PRIMARY  KEY,
  name  VARCHAR(20)  NULL,
  )
```

注意

`Identity`对象支持许多其他参数，除了`start`和`increment`之外。这些参数不受 SQL Server 支持，在生成 CREATE TABLE ddl 时将被忽略。

版本 1.3.19 中的更改：在 SQL Server 下，现在使用 `Identity` 对象来影响 `Column` 的 `IDENTITY` 生成器。之前使用的是 `Sequence` 对象。由于 SQL Server 现在支持将实际序列作为一个独立的构造，因此 `Sequence` 将从 SQLAlchemy 版本 1.4 开始以正常方式运作。

### 使用非整数数值类型的 IDENTITY

SQL Server 还允许将 `IDENTITY` 用于 `NUMERIC` 列。为了在 SQLAlchemy 中平滑实现这种模式，在列的主要数据类型应保持为 `Integer`，但是可以使用 `TypeEngine.with_variant()` 来指定部署到 SQL Server 数据库的底层实现类型为 `Numeric`：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class TestTable(Base):
    __tablename__ = "test"
    id = Column(
        Integer().with_variant(Numeric(10, 0), "mssql"),
        primary_key=True,
        autoincrement=True,
    )
    name = Column(String)
```

在上面的示例中，`Integer().with_variant()` 提供了明确的使用信息，准确描述了代码的意图。将 `autoincrement` 仅适用于 `Integer` 的一般限制建立在元数据级别而不是每个方言级别。

使用上述模式时，从插入行返回的主键标识符（也是将分配给类似于上面的 `TestTable` 的 ORM 对象的值）将是 `Decimal()` 的实例，而不是使用 SQL Server 时的 `int`。通过将 False 传递给 `Numeric.asdecimal`，可以将 `Numeric` 类型的数值返回类型更改为浮点数。要将上述 `Numeric(10, 0)` 的返回类型规范化为返回 Python 整数（在 Python 3 中也支持“长”整数值），请使用 `TypeDecorator` 如下所示：

```py
from sqlalchemy import TypeDecorator

class NumericAsInteger(TypeDecorator):
  '''normalize floating point return values into ints'''

    impl = Numeric(10, 0, asdecimal=False)
    cache_ok = True

    def process_result_value(self, value, dialect):
        if value is not None:
            value = int(value)
        return value

class TestTable(Base):
    __tablename__ = "test"
    id = Column(
        Integer().with_variant(NumericAsInteger, "mssql"),
        primary_key=True,
        autoincrement=True,
    )
    name = Column(String)
```

### 插入行为

在 INSERT 时处理 `IDENTITY` 列涉及两个关键技术。最常见的是能够获取给定 `IDENTITY` 列的“最后插入值”，SQLAlchemy 在许多情况下都会隐式执行这个过程，最重要的是在 ORM 中。

获取此值的过程有几种变体：

+   在绝大多数情况下，在 SQL Server 上与 INSERT 语句一起使用 RETURNING 以获取新生成的主键值：

    ```py
    INSERT  INTO  t  (x)  OUTPUT  inserted.id  VALUES  (?)
    ```

    从 SQLAlchemy 2.0 开始，默认还使用 INSERT 语句的“插入多个值”行为功能来优化多行 INSERT 语句；对于 SQL Server，该功能适用于 RETURNING 和非 RETURNING INSERT 语句。

    从版本 2.0.10 开始更改：由于行排序问题，SQLAlchemy 版本 2.0.9 暂时禁用了 SQL Server 的 INSERT 语句的“插入多个值”行为功能。从 2.0.10 开始，该功能已重新启用，并对工作单元对 RETURNING 的排序要求进行了特殊处理。

+   当`RETURNING`不可用或通过`implicit_returning=False`禁用时，将使用`scope_identity()`函数或`@@identity`变量；后端的行为各不相同：

    +   使用 PyODBC 时，短语`; select scope_identity()`将被附加到插入语句的末尾；为了接收值，将获取第二个结果集。给定一个表如下：

        ```py
        t = Table(
            't',
            metadata,
            Column('id', Integer, primary_key=True),
            Column('x', Integer),
            implicit_returning=False
        )
        ```

        插入操作看起来像是：

        ```py
        INSERT  INTO  t  (x)  VALUES  (?);  select  scope_identity()
        ```

    +   其他方言，如 pymssql，在 INSERT 语句后调用`SELECT scope_identity() AS lastrowid`。如果将标志`use_scope_identity=False`传递给`create_engine()`，则将改为使用语句`SELECT @@identity AS lastrowid`。

包含`IDENTITY`列的表将禁止明确引用标识列的插入语句。SQLAlchemy 方言将检测到当使用核心`insert()`构造创建的 INSERT 构造引用标识列时（而不是普通的字符串 SQL），在这种情况下，将在插入语句执行之前发出`SET IDENTITY_INSERT ON`，并在执行后发出`SET IDENTITY_INSERT OFF`。给定此示例：

```py
m = MetaData()
t = Table('t', m, Column('id', Integer, primary_key=True),
                Column('x', Integer))
m.create_all(engine)

with engine.begin() as conn:
    conn.execute(t.insert(), {'id': 1, 'x':1}, {'id':2, 'x':2})
```

上述列将使用 IDENTITY 创建，但我们发出的 INSERT 语句指定了显式值。在回显输出中，我们可以看到 SQLAlchemy 如何处理这个问题：

```py
CREATE  TABLE  t  (
  id  INTEGER  NOT  NULL  IDENTITY(1,1),
  x  INTEGER  NULL,
  PRIMARY  KEY  (id)
)

COMMIT
SET  IDENTITY_INSERT  t  ON
INSERT  INTO  t  (id,  x)  VALUES  (?,  ?)
((1,  1),  (2,  2))
SET  IDENTITY_INSERT  t  OFF
COMMIT
```

这是一个适用于测试和批量插入场景的辅助用例。

### 控制“开始”和“增量”

使用传递给`Identity`对象的`Identity.start`和`Identity.increment`参数提供对`IDENTITY`生成器的“开始”和“增量”值的特定控制：

```py
from sqlalchemy import Table, Integer, Column, Identity

test = Table(
    'test', metadata,
    Column(
        'id',
        Integer,
        primary_key=True,
        Identity(start=100, increment=10)
    ),
    Column('name', String(20))
)
```

上述`Table`对象的 CREATE TABLE 将是：

```py
CREATE  TABLE  test  (
  id  INTEGER  NOT  NULL  IDENTITY(100,10)  PRIMARY  KEY,
  name  VARCHAR(20)  NULL,
  )
```

注意

`Identity`对象除了`start`和`increment`之外还支持许多其他参数。这些参数在 SQL Server 中不受支持，在生成 CREATE TABLE ddl 时将被忽略。

从版本 1.3.19 开始更改：`Identity`对象现在用于影响 SQL Server 下的`Column`的`IDENTITY`生成器。以前，使用的是`Sequence`对象。由于 SQL Server 现在支持真实的序列作为单独的构造，因此从 SQLAlchemy 版本 1.4 开始，`Sequence`将以正常的方式运行。

### 使用非整数数值类型的 IDENTITY

SQL Server 还允许将`IDENTITY`与`NUMERIC`列一起使用。要在 SQLAlchemy 中顺利实现此模式，列的主要数据类型应保持为`Integer`，但是可以使用`TypeEngine.with_variant()`指定部署到 SQL Server 数据库的底层实现类型为`Numeric`：

```py
from sqlalchemy import Column
from sqlalchemy import Integer
from sqlalchemy import Numeric
from sqlalchemy import String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class TestTable(Base):
    __tablename__ = "test"
    id = Column(
        Integer().with_variant(Numeric(10, 0), "mssql"),
        primary_key=True,
        autoincrement=True,
    )
    name = Column(String)
```

在上面的示例中，`Integer().with_variant()`提供了清晰的使用信息，准确描述了代码的意图。`autoincrement`仅适用于`Integer`的一般限制是在元数据级别而不是在每个方言级别上建立的。

当使用上述模式时，从插入行返回的主键标识符，也就是将被分配给诸如上述`TestTable`的 ORM 对象的值，当使用 SQL Server 时将是`Decimal()`的实例，而不是`int`。 `Numeric`类型的数值返回类型可以通过将 False 传递给`Numeric.asdecimal`来更改为返回浮点数。要将上述`Numeric(10, 0)`的返回类型规范化为返回 Python 整数（在 Python 3 中也支持“long”整数值），请使用`TypeDecorator`如下所示：

```py
from sqlalchemy import TypeDecorator

class NumericAsInteger(TypeDecorator):
  '''normalize floating point return values into ints'''

    impl = Numeric(10, 0, asdecimal=False)
    cache_ok = True

    def process_result_value(self, value, dialect):
        if value is not None:
            value = int(value)
        return value

class TestTable(Base):
    __tablename__ = "test"
    id = Column(
        Integer().with_variant(NumericAsInteger, "mssql"),
        primary_key=True,
        autoincrement=True,
    )
    name = Column(String)
```

### 插入行为

在 INSERT 时处理`IDENTITY`列涉及两种关键技术。最常见的是能够获取给定`IDENTITY`列的“最后插入的值”，这是 SQLAlchemy 在许多情况下隐式执行的过程，最重要的是在 ORM 中。

获取此值的过程有几种变体：

+   在绝大多数情况下，RETURNING 与 SQL Server 上的 INSERT 语句一起使用，以获取新生成的主键值：

    ```py
    INSERT  INTO  t  (x)  OUTPUT  inserted.id  VALUES  (?)
    ```

    从 SQLAlchemy 2.0 开始，默认还使用“Insert Many Values” Behavior for INSERT statements 功能来优化多行 INSERT 语句；对于 SQL Server，该功能适用于 RETURNING 和非 RETURNING INSERT 语句。

    从版本 2.0.10 开始更改：由于行排序问题，SQLAlchemy 版本 2.0.9 暂时禁用了 SQL Server 的“Insert Many Values” Behavior for INSERT statements 功能。从 2.0.10 开始，该功能重新启用，并针对工作单元对 RETURNING 的排序要求进行特殊处理。

+   当 RETURNING 不可用或通过`implicit_returning=False`禁用时，将使用`scope_identity()`函数或`@@identity`变量；后端的行为各不相同：

    +   使用 PyODBC 时，短语`; select scope_identity()`将附加到 INSERT 语句的末尾；为了接收值，将获取第二个结果集。假设有一个表：

        ```py
        t = Table(
            't',
            metadata,
            Column('id', Integer, primary_key=True),
            Column('x', Integer),
            implicit_returning=False
        )
        ```

        一个 INSERT 看起来像：

        ```py
        INSERT  INTO  t  (x)  VALUES  (?);  select  scope_identity()
        ```

    +   其他方言，如 pymssql，在 INSERT 语句后将调用`SELECT scope_identity() AS lastrowid`。如果将标志`use_scope_identity=False`传递给`create_engine()`，则将使用语句`SELECT @@identity AS lastrowid`。

包含`IDENTITY`列的表将禁止引用显式标识列的 INSERT 语句。当 SQLAlchemy 方言检测到使用核心`insert()`构造（而不是纯字符串 SQL）创建的 INSERT 构造引用标识列时，在这种情况下，将在继续插入语句之前发出`SET IDENTITY_INSERT ON`，并在执行后继续发出`SET IDENTITY_INSERT OFF`。给出这个例子：

```py
m = MetaData()
t = Table('t', m, Column('id', Integer, primary_key=True),
                Column('x', Integer))
m.create_all(engine)

with engine.begin() as conn:
    conn.execute(t.insert(), {'id': 1, 'x':1}, {'id':2, 'x':2})
```

上述列将使用 IDENTITY 创建，但我们发出的 INSERT 语句指定了显式值。在回显输出中，我们可以看到 SQLAlchemy 如何处理这个问题：

```py
CREATE  TABLE  t  (
  id  INTEGER  NOT  NULL  IDENTITY(1,1),
  x  INTEGER  NULL,
  PRIMARY  KEY  (id)
)

COMMIT
SET  IDENTITY_INSERT  t  ON
INSERT  INTO  t  (id,  x)  VALUES  (?,  ?)
((1,  1),  (2,  2))
SET  IDENTITY_INSERT  t  OFF
COMMIT
```

这是一个适用于测试和批量插入场景的辅助用例。

## SEQUENCE 支持

`Sequence`对象创建“真实”序列，即`CREATE SEQUENCE`：

```py
>>> from sqlalchemy import Sequence
>>> from sqlalchemy.schema import CreateSequence
>>> from sqlalchemy.dialects import mssql
>>> print(CreateSequence(Sequence("my_seq", start=1)).compile(dialect=mssql.dialect()))
CREATE  SEQUENCE  my_seq  START  WITH  1 
```

对于整数主键生成，通常应优先选择 SQL Server 的`IDENTITY`构造而不是序列。

提示

T-SQL 的默认起始值为`-2**63`，而不是大多数其他 SQL 数据库中的 1。如果预期默认值是 1，则用户应明确设置`Sequence.start`：

```py
seq = Sequence("my_sequence", start=1)
```

从版本 1.4 开始：为`Sequence`添加了对 SQL Server 的支持

在 2.0 版本中更改：SQL Server 方言将不再隐式呈现“START WITH 1”用于`CREATE SEQUENCE`，这是在 1.4 版本中首次实现的行为。

## VARCHAR / NVARCHAR 上的 MAX

SQL Server 支持特殊字符串“MAX”在`VARCHAR`和`NVARCHAR`数据类型中，以指示“可能的最大长度”。方言当前将此处理为基本类型中长度为“None”，而不是提供这些类型的特定于方言的版本，因此可以假定指定为`VARCHAR(None)`之类的基本类型在不使用特定于方言的类型的情况下，在多个后端上表现出“无长度”的行为。

要构建具有 MAX 长度的 SQL Server VARCHAR 或 NVARCHAR，请使用 None：

```py
my_table = Table(
    'my_table', metadata,
    Column('my_data', VARCHAR(None)),
    Column('my_n_data', NVARCHAR(None))
)
```

## 字符串排序支持

基本字符串类型支持字符排序，由字符串参数“collation”指定：

```py
from sqlalchemy import VARCHAR
Column('login', VARCHAR(32, collation='Latin1_General_CI_AS'))
```

当此列与`Table`关联时，该列的 CREATE TABLE 语句将产生：

```py
login VARCHAR(32) COLLATE Latin1_General_CI_AS NULL
```

## LIMIT/OFFSET 支持

MSSQL 从 SQL Server 2012 开始增加了对 LIMIT / OFFSET 的支持，通过“OFFSET n ROWS”和“FETCH NEXT n ROWS”子句。如果检测到 SQL Server 2012 或更高版本，则 SQLAlchemy 会自动支持这些语法。

1.4 版本中更改：增加了对 SQL Server“OFFSET n ROWS”和“FETCH NEXT n ROWS”语法的支持。

对于仅指定 LIMIT 而不指定 OFFSET 的语句，所有版本的 SQL Server 都支持 TOP 关键字。当没有 OFFSET 子句时，此语法用于所有 SQL Server 版本。例如这样的语句：

```py
select(some_table).limit(5)
```

将类似于以下内容呈现：

```py
SELECT TOP 5 col1, col2.. FROM table
```

对于 SQL Server 2012 之前的版本，使用 LIMIT 和 OFFSET 或仅使用 OFFSET 的语句将使用`ROW_NUMBER()`窗口函数呈现。例如这样的语句：

```py
select(some_table).order_by(some_table.c.col3).limit(5).offset(10)
```

将类似于以下内容呈现：

```py
SELECT anon_1.col1, anon_1.col2 FROM (SELECT col1, col2,
ROW_NUMBER() OVER (ORDER BY col3) AS
mssql_rn FROM table WHERE t.x = :x_1) AS
anon_1 WHERE mssql_rn > :param_1 AND mssql_rn <= :param_2 + :param_1
```

请注意，无论是使用旧版还是新版 SQL Server 语法，使用 LIMIT 和/或 OFFSET 时，语句必须也有 ORDER BY，否则会引发`CompileError`。

## DDL 注释支持

支持注释，包括对`Table.comment`和`Column.comment`等属性的 DDL 呈现，以及反映这些注释的能力，假定正在使用受支持的 SQL Server 版本。如果在首次连接时检测到不受支持的版本（例如 Azure Synapse）（基于`fn_listextendedproperty` SQL 函数的存在），则会禁用注释支持，包括呈现和表注释反射，因为这两个功能依赖于并非所有后端类型都可用的 SQL Server 存储过程和函数。

要强制开启或关闭注释支持，绕过自动检测，请在 `create_engine()` 中设置参数 `supports_comments`：

```py
e = create_engine("mssql+pyodbc://u:p@dsn", supports_comments=False)
```

新版本 2.0 中：增加了对 SQL Server 方言的表和列注释的支持，包括 DDL 生成和反射。

## 事务隔离级别

所有 SQL Server 方言都支持通过方言特定参数`create_engine.isolation_level`（由`create_engine()` 接受）以及传递给 `Connection.execution_options()` 的`Connection.execution_options.isolation_level` 参数来设置事务隔离级别。该功能通过为每个新连接发出命令 `SET TRANSACTION ISOLATION LEVEL <level>` 来实现。

使用 `create_engine()` 设置隔离级别：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@ms_2008",
    isolation_level="REPEATABLE READ"
)
```

使用每个连接的执行选项来设置：

```py
connection = engine.connect()
connection = connection.execution_options(
    isolation_level="READ COMMITTED"
)
```

`isolation_level` 的有效值包括：

+   `AUTOCOMMIT` - pyodbc / pymssql 特有

+   `READ COMMITTED`

+   `READ UNCOMMITTED`

+   `REPEATABLE READ`

+   `SERIALIZABLE`

+   `SNAPSHOT` - SQL Server 特有

还有更多关于隔离级别配置的选项，例如与主`Engine`关联的“子引擎”对象，每个对象都应用不同的隔离级别设置。请参阅设置事务隔离级别，包括 DBAPI 自动提交的讨论以获取更多背景信息。

另请参阅

设置事务隔离级别，包括 DBAPI 自动提交

## 连接池的临时表 / 资源重置

SQLAlchemy `Engine` 对象使用的 `QueuePool` 连接池实现包含 返回时重置 行为，当连接返回到池中时将调用 DBAPI 的`.rollback()` 方法。虽然此回滚会清除前一个事务使用的即时状态，但它不涵盖更广泛范围的会话级状态，包括临时表以及其他服务器状态，如准备好的语句句柄和语句缓存。一个名为`sp_reset_connection`的未记录的 SQL Server 程序被认为是此问题的解决方法，它将重置建立在连接上的大部分会话状态，包括临时表。

要将`sp_reset_connection`安装为执行返回时重置的方法，可以使用 `PoolEvents.reset()` 事件钩子，如下例所示。将 `create_engine.pool_reset_on_return` 参数设置为`None`，以便自定义方案可以完全替换默认行为。自定义钩子实现在任何情况下调用`.rollback()`，因为通常重要的是 DBAPI 自身的提交/回滚跟踪将保持与事务状态一致：

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

在 2.0.0b3 版中更改：为 `PoolEvents.reset()` 事件添加了额外的状态参数，并且还确保对所有“重置”事件进行调用，因此它适用于自定义“重置”处理程序的地方。先前使用 `PoolEvents.checkin()` 处理程序的方案仍然可用。

另请参阅

返回时重置 - 在 连接池 文档中

## 可空性

MSSQL 支持三个级别的列可空性。默认的可空性允许空值，并且在 CREATE TABLE 构造中是显式的：

```py
name VARCHAR(20) NULL
```

如果指定了`nullable=None`，则不做任何规定。换句话说，将使用数据库配置的默认值。这将呈现为：

```py
name VARCHAR(20)
```

如果`nullable`为`True`或`False`，则列将分别为`NULL`或`NOT NULL`。

## 日期/时间处理

支持 DATE 和 TIME。根据大多数 MSSQL 驱动程序的要求，绑定参数将转换为 datetime.datetime() 对象，并且如果需要的话，结果将从字符串中处理。对于 MSSQL 2005 及之前版本，不可用 DATE 和 TIME 类型 - 如果检测到低于 2008 的服务器版本，则将为这些类型发出 DDL 作为 DATETIME。

## 大文本/二进制类型弃用

根据 [SQL Server 2012/2014 文档](https://technet.microsoft.com/en-us/library/ms187993.aspx)，`NTEXT`、`TEXT` 和 `IMAGE` 数据类型将在将来的发布中从 SQL Server 中删除。SQLAlchemy 通常将这些类型关联到 `UnicodeText`、`TextClause` 和 `LargeBinary` 数据类型。

为了适应这一变化，方言新增了一个名为 `deprecate_large_types` 的新标志，该标志将根据正在使用的服务器版本的检测自动设置，如果用户未设置其他值的话。该标志的行为如下：

+   当此标志为 `True` 时，`UnicodeText`、`TextClause` 和 `LargeBinary` 数据类型在用于渲染 DDL 时，将分别呈现类型 `NVARCHAR(max)`、`VARCHAR(max)` 和 `VARBINARY(max)`。这是此标志添加后的新行为。

+   当此标志为 `False` 时，`UnicodeText`、`TextClause` 和 `LargeBinary` 数据类型在用于渲染 DDL 时，将分别呈现类型 `NTEXT`、`TEXT` 和 `IMAGE`。这是这些类型的长期行为。

+   标志在建立数据库连接之前以值 `None` 开始。如果方言在未设置标志的情况下用于渲染 DDL，则其解释方式与 `False` 相同。

+   在第一次连接时，方言会检测是否正在使用 SQL Server 2012 或更高版本；如果标志仍处于 `None`，则根据是否检测到 2012 或更高版本来设置为 `True` 或 `False`。

+   当创建方言时，可以将标志设置为 `True` 或 `False`，通常通过 `create_engine()` 完成：

    ```py
    eng = create_engine("mssql+pymssql://user:pass@host/db",
                    deprecate_large_types=True)
    ```

+   Complete control over whether the “old” or “new” types are rendered is available in all SQLAlchemy versions by using the UPPERCASE type objects instead: `NVARCHAR`, `VARCHAR`, `VARBINARY`, `TEXT`, `NTEXT`, `IMAGE` will always remain fixed and always output exactly that type.

## Multipart Schema Names

SQL Server schemas sometimes require multiple parts to their “schema” qualifier, that is, including the database name and owner name as separate tokens, such as `mydatabase.dbo.some_table`. These multipart names can be set at once using the `Table.schema` argument of `Table`:

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="mydatabase.dbo"
)
```

When performing operations such as table or component reflection, a schema argument that contains a dot will be split into separate “database” and “owner” components in order to correctly query the SQL Server information schema tables, as these two values are stored separately. Additionally, when rendering the schema name for DDL or SQL, the two components will be quoted separately for case sensitive names and other special characters. Given an argument as below:

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="MyDataBase.dbo"
)
```

The above schema would be rendered as `[MyDataBase].dbo`, and also in reflection, would be reflected using “dbo” as the owner and “MyDataBase” as the database name.

To control how the schema name is broken into database / owner, specify brackets (which in SQL Server are quoting characters) in the name. Below, the “owner” will be considered as `MyDataBase.dbo` and the “database” will be None:

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="[MyDataBase.dbo]"
)
```

To individually specify both database and owner name with special characters or embedded dots, use two sets of brackets:

```py
Table(
    "some_table", metadata,
    Column("q", String(50)),
    schema="[MyDataBase.Period].[MyOwner.Dot]"
)
```

Changed in version 1.2: the SQL Server dialect now treats brackets as identifier delimiters splitting the schema into separate database and owner tokens, to allow dots within either name itself.

## Legacy Schema Mode

Very old versions of the MSSQL dialect introduced the behavior such that a schema-qualified table would be auto-aliased when used in a SELECT statement; given a table:

```py
account_table = Table(
    'account', metadata,
    Column('id', Integer, primary_key=True),
    Column('info', String(100)),
    schema="customer_schema"
)
```

this legacy mode of rendering would assume that “customer_schema.account” would not be accepted by all parts of the SQL statement, as illustrated below:

```py
>>> eng = create_engine("mssql+pymssql://mydsn", legacy_schema_aliasing=True)
>>> print(account_table.select().compile(eng))
SELECT  account_1.id,  account_1.info
FROM  customer_schema.account  AS  account_1 
```

此行为模式现在默认关闭，因为似乎没有任何作用；但是，如果传统应用程序依赖于它，则可以使用`create_engine()`中的`legacy_schema_aliasing`参数来使用，如上所示。

自版本 1.4 起弃用：`legacy_schema_aliasing`标志现已弃用，并将在将来的版本中删除。

## 聚集索引支持

MSSQL 方言支持通过`mssql_clustered`选项生成聚集索引（和主键）。此选项适用于`Index`、`UniqueConstraint`和`PrimaryKeyConstraint`。对于索引，此选项可以与`mssql_columnstore`结合使用以创建聚集列存储索引。

生成一个聚集索引：

```py
Index("my_index", table.c.x, mssql_clustered=True)
```

将索引渲染为`CREATE CLUSTERED INDEX my_index ON table (x)`。

要生成一个聚集主键，请使用：

```py
Table('my_table', metadata,
      Column('x', ...),
      Column('y', ...),
      PrimaryKeyConstraint("x", "y", mssql_clustered=True))
```

例如，将表渲染为：

```py
CREATE TABLE my_table (x INTEGER NOT NULL, y INTEGER NOT NULL,
                       PRIMARY KEY CLUSTERED (x, y))
```

类似地，我们可以使用以下方法生成一个聚类唯一约束：

```py
Table('my_table', metadata,
      Column('x', ...),
      Column('y', ...),
      PrimaryKeyConstraint("x"),
      UniqueConstraint("y", mssql_clustered=True),
      )
```

要明确请求非聚集主键（例如，当需要单独的聚集索引时），请使用：

```py
Table('my_table', metadata,
      Column('x', ...),
      Column('y', ...),
      PrimaryKeyConstraint("x", "y", mssql_clustered=False))
```

例如，将表渲染为：

```py
CREATE TABLE my_table (x INTEGER NOT NULL, y INTEGER NOT NULL,
                       PRIMARY KEY NONCLUSTERED (x, y))
```

## 列存储索引支持

MSSQL 方言通过`mssql_columnstore`选项支持列存储索引。此选项适用于`Index`。它可以与`mssql_clustered`选项结合使用以创建聚集列存储索引。

要生成列存储索引：

```py
Index("my_index", table.c.x, mssql_columnstore=True)
```

将索引渲染为`CREATE COLUMNSTORE INDEX my_index ON table (x)`。

要生成一个聚集列存储索引，请不提供列：

```py
idx = Index("my_index", mssql_clustered=True, mssql_columnstore=True)
# required to associate the index with the table
table.append_constraint(idx)
```

上述将索引渲染为`CREATE CLUSTERED COLUMNSTORE INDEX my_index ON table`。

版本 2.0.18 中的新功能。

## MSSQL 特定的索引选项

除了聚类外，MSSQL 方言还支持其他特殊选项用于`Index`。

### 包括

`mssql_include`选项为给定的字符串名称渲染 INCLUDE(colname)：

```py
Index("my_index", table.c.x, mssql_include=['y'])
```

将索引渲染为`CREATE INDEX my_index ON table (x) INCLUDE (y)`

### 过滤索引

`mssql_where`选项为给定的字符串名称渲染 WHERE(condition)：

```py
Index("my_index", table.c.x, mssql_where=table.c.x > 10)
```

将索引渲染为`CREATE INDEX my_index ON table (x) WHERE x > 10`。

版本 1.3.4 中的新功能。

### 索引排序

索引排序可通过功能表达式获得，例如：

```py
Index("my_index", table.c.x.desc())
```

将索引渲染为`CREATE INDEX my_index ON table (x DESC)`

另请参阅

功能性索引

### 包括

`mssql_include`选项为给定的字符串名称渲染 INCLUDE(colname)：

```py
Index("my_index", table.c.x, mssql_include=['y'])
```

渲染索引为`CREATE INDEX my_index ON table (x) INCLUDE (y)`。

### 过滤索引

`mssql_where` 选项为给定的字符串名称渲染 WHERE(condition)：

```py
Index("my_index", table.c.x, mssql_where=table.c.x > 10)
```

渲染索引为`CREATE INDEX my_index ON table (x) WHERE x > 10`。

在 1.3.4 版中新增。

### 索引排序

可以通过函数表达式实现索引排序，例如：

```py
Index("my_index", table.c.x.desc())
```

渲染索引为`CREATE INDEX my_index ON table (x DESC)`。

另请参阅

功能索引

## 兼容性级别

MSSQL 支持在数据库级别设置兼容性级别的概念。这允许例如，在运行于 SQL2005 数据库服务器上时运行与 SQL2000 兼容的数据库。`server_version_info` 将始终返回数据库服务器版本信息（在此情况下为 SQL2005），而不是兼容性级别信息。因此，如果在向后兼容模式下运行，则 SQLAlchemy 可能会尝试使用数据库服务器无法解析的 T-SQL 语句。

## 触发器

SQLAlchemy 默认使用 OUTPUT INSERTED 获取通过 IDENTITY 列或其他服务器端默认值生成的新主键值。MS-SQL 不允许在具有触发器的表上使用 OUTPUT INSERTED。要在每个具有触发器的 `Table` 上禁用 OUTPUT INSERTED 的使用，请为其指定 `implicit_returning=False`：

```py
Table('mytable', metadata,
    Column('id', Integer, primary_key=True),
    # ...,
    implicit_returning=False
)
```

声明形式：

```py
class MyClass(Base):
    # ...
    __table_args__ = {'implicit_returning':False}
```

## 行数支持 / ORM 版本控制

SQL Server 驱动程序可能有限的能力来返回更新或删除语句所影响的行数。

截至本文撰写时，PyODBC 驱动程序无法在使用 OUTPUT INSERTED 时返回行数。因此，SQLAlchemy 的先前版本在功能上存在限制，例如依赖准确的行数来匹配版本号与匹配行的“ORM 版本控制”功能。

SQLAlchemy 2.0 现在根据返回的 RETURNING 中到达的行数手动检索这些特定用例的“行数”；因此，虽然驱动程序仍具有此限制，但 ORM 版本控制功能不再受其影响。截至 SQLAlchemy 2.0.5，已完全重新启用了 pyodbc 驱动程序的 ORM 版本控制功能。

在 2.0.5 版更改：对于 pyodbc 驱动程序，已恢复 ORM 版本控制支持。先前，ORM 刷新期间会发出警告，说明不支持版本控制。

## 启用快照隔离

SQL Server 具有默认的事务隔离模式，锁定整个表，并导致即使是稍微并发的应用程序也具有长时间持有的锁定和频繁的死锁。为了支持现代级别的并发性，建议为整个数据库启用快照隔离。这通过在 SQL 提示符下执行以下 ALTER DATABASE 命令来完成： 

```py
ALTER DATABASE MyDatabase SET ALLOW_SNAPSHOT_ISOLATION ON

ALTER DATABASE MyDatabase SET READ_COMMITTED_SNAPSHOT ON
```

关于 SQL Server 快照隔离的背景信息，请访问 [`msdn.microsoft.com/en-us/library/ms175095.aspx`](https://msdn.microsoft.com/en-us/library/ms175095.aspx)。

## SQL Server SQL 构造

| 对象名称 | 描述 |
| --- | --- |
| try_cast(expression, type_) | 为支持的后端生成一个 `TRY_CAST` 表达式；这是一个返回 NULL 的 `CAST`，用于不可转换的转换。 |

```py
function sqlalchemy.dialects.mssql.try_cast(expression: _ColumnExpressionOrLiteralArgument[Any], type_: _TypeEngineArgument[_T]) → TryCast[_T]
```

为支持的后端生成一个 `TRY_CAST` 表达式；这是一个返回 NULL 的 `CAST`，用于不可转换的转换。

在 SQLAlchemy 中，此构造仅受 SQL Server 方言支持，并且如果在其他包含的后端上使用，则会引发 `CompileError`。但是，第三方后端也可能支持此构造。

提示

由于 `try_cast()` 来源于 SQL Server 方言，因此它既可以从 `sqlalchemy.` 导入，也可以从 `sqlalchemy.dialects.mssql` 导入。

`try_cast()` 返回 `TryCast` 的实例，并且通常表现得与 `Cast` 构造类似；在 SQL 层面，`CAST` 和 `TRY_CAST` 之间的区别在于 `TRY_CAST` 对于不可转换的表达式（例如，尝试将字符串 `"hi"` 转换为整数值）返回 NULL。

例如：

```py
from sqlalchemy import select, try_cast, Numeric

stmt = select(
    try_cast(product_table.c.unit_price, Numeric(10, 4))
)
```

上述内容在 Microsoft SQL Server 上呈现为：

```py
SELECT TRY_CAST (product_table.unit_price AS NUMERIC(10, 4))
FROM product_table
```

从版本 2.0.14 开始：`try_cast()`已从 SQL Server 方言泛化为一个通用构造，可能由其他方言支持。

## SQL Server 数据类型

与所有 SQLAlchemy 方言一样，所有已知在 SQL Server 中有效的大写类型都可以从顶级方言导入，无论其来源是`sqlalchemy.types` 还是来自本地方言：

```py
from sqlalchemy.dialects.mssql import (
    BIGINT,
    BINARY,
    BIT,
    CHAR,
    DATE,
    DATETIME,
    DATETIME2,
    DATETIMEOFFSET,
    DECIMAL,
    DOUBLE_PRECISION,
    FLOAT,
    IMAGE,
    INTEGER,
    JSON,
    MONEY,
    NCHAR,
    NTEXT,
    NUMERIC,
    NVARCHAR,
    REAL,
    SMALLDATETIME,
    SMALLINT,
    SMALLMONEY,
    SQL_VARIANT,
    TEXT,
    TIME,
    TIMESTAMP,
    TINYINT,
    UNIQUEIDENTIFIER,
    VARBINARY,
    VARCHAR,
)
```

特定于 SQL Server 或具有 SQL Server 特定构造参数的类型如下：

| 对象名称 | 描述 |
| --- | --- |
| BIT | MSSQL BIT 类型。 |
| DATETIME2 |  |
| DATETIMEOFFSET |  |
| DOUBLE_PRECISION | SQL Server DOUBLE PRECISION 数据类型。 |
| IMAGE |  |
| JSON | MSSQL JSON 类型。 |
| MONEY |  |
| NTEXT | MSSQL NTEXT 类型，用于最多 2³⁰ 个字符的变长 unicode 文本。 |
| REAL | SQL Server REAL 数据类型。 |
| ROWVERSION | 实现 SQL Server ROWVERSION 类型。 |
| SMALLDATETIME |  |
| SMALLMONEY |  |
| SQL_VARIANT |  |
| TIME |  |
| TIMESTAMP | 实现 SQL Server TIMESTAMP 类型。 |
| TINYINT |  |
| UNIQUEIDENTIFIER |  |
| XML | MSSQL XML 类型。 |

```py
class sqlalchemy.dialects.mssql.BIT
```

MSSQL BIT 类型。

pyodbc 和 pymssql 都将 BIT 列的值作为 Python <class ‘bool’> 返回，因此只需对 Boolean 进行子类化。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.BIT` (`sqlalchemy.types.Boolean`)

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

*从* `Boolean` *的* `sqlalchemy.types.Boolean.__init__` *方法继承*

构造一个布尔值。

参数：

+   `create_constraint` - 

    默认为 False。如果布尔值生成为 int/smallint，还会在表上创建 CHECK 约束，以确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有明确的名称，以支持模式管理方面的考虑。这可以通过设置 `Boolean.name` 参数或设置适当的命名约定来实现；请参阅 配置约束命名约定 了解背景信息。

    从版本 1.4 开始更改：- 此标志现在默认为 False，意味着对于非本地枚举类型不生成 CHECK 约束。

+   `name` - 如果生成 CHECK 约束，请指定约束的名称。

```py
class sqlalchemy.dialects.mssql.CHAR
```

SQL CHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.CHAR` (`sqlalchemy.types.String`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个保存字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能需要用于 DDL 的 `length`，如果包含了没有长度的 `VARCHAR`，则会在发出 `CREATE TABLE` DDL 时引发异常。值是作为字节还是字符解释是特定于数据库的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式的列级别排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该为预计存储非 ASCII 数据的 `Column` 使用 `Unicode` 或 `UnicodeText` 数据类型。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.DATETIME2
```

**类签名**

类 `sqlalchemy.dialects.mssql.DATETIME2`（`sqlalchemy.dialects.mssql.base._DateTimeBase`，`sqlalchemy.types.DateTime`）

```py
class sqlalchemy.dialects.mssql.DATETIMEOFFSET
```

**类签名**

类 `sqlalchemy.dialects.mssql.DATETIMEOFFSET`（`sqlalchemy.dialects.mssql.base._DateTimeBase`，`sqlalchemy.types.DateTime`）

```py
class sqlalchemy.dialects.mssql.DOUBLE_PRECISION
```

SQL Server 的 DOUBLE PRECISION 数据类型。

新版 2.0.11 中新增。

**类签名**

类 `sqlalchemy.dialects.mssql.DOUBLE_PRECISION`（`sqlalchemy.types.DOUBLE_PRECISION`）

```py
class sqlalchemy.dialects.mssql.IMAGE
```

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.IMAGE`（`sqlalchemy.types.LargeBinary`）

```py
method __init__(length: int | None = None)
```

*继承自* `LargeBinary` *的* `sqlalchemy.types.LargeBinary.__init__` *方法*

构造一个 LargeBinary 类型。

参数：

**length** – 可选，用于 DDL 语句中的列长度，对于那些接受长度的二进制类型，如 MySQL 的 BLOB 类型。

```py
class sqlalchemy.dialects.mssql.JSON
```

MSSQL JSON 类型。

MSSQL 支持 JSON 格式的数据，从 SQL Server 2016 开始。

在 DDL 级别上，`JSON` 数据类型将表示为 `NVARCHAR(max)`，但还提供了 JSON 级别的比较函数以及 Python 强制行为。

每当基本的 `JSON` 数据类型用于 SQL Server 后端时，都会自动使用 `JSON`。

另请参阅

`JSON` - 通用跨平台 JSON 数据类型的主要文档。

`JSON` 类型支持将 JSON 值持久化，同时通过调整操作以在数据库级别渲染 `JSON_VALUE` 或 `JSON_QUERY` 函数来提供 `JSON` 数据类型提供的核心索引操作。

SQL Server `JSON` 类型在查询 JSON 对象的元素时必然使用 `JSON_QUERY` 和 `JSON_VALUE` 函数。 这两个函数有一个主要限制，即它们基于要返回的对象类型是 **互斥的**。 `JSON_QUERY` 函数**仅**返回 JSON 字典或列表，而不是单个字符串、数字或布尔元素；`JSON_VALUE` 函数**仅**返回单个字符串、数字或布尔元素。 **这两个函数都会在不使用预期正确的值时返回 NULL 或引发错误**。

为了处理这个尴尬的要求，索引访问规则如下：

1.  当从 JSON 中提取的子元素本身是 JSON 字典或列表时，应使用 `Comparator.as_json()` 访问器：

    ```py
    stmt = select(
        data_table.c.data["some key"].as_json()
    ).where(
        data_table.c.data["some key"].as_json() == {"sub": "structure"}
    )
    ```

1.  当从 JSON 中提取为普通布尔值、字符串、整数或浮点数的子元素时，请使用以下适当的方法之一：`Comparator.as_boolean()`、`Comparator.as_string()`、`Comparator.as_integer()`、`Comparator.as_float()`:

    ```py
    stmt = select(
        data_table.c.data["some key"].as_string()
    ).where(
        data_table.c.data["some key"].as_string() == "some string"
    )
    ```

版本 1.4 中的新功能。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.JSON`（`sqlalchemy.types.JSON`）

```py
method __init__(none_as_null: bool = False)
```

*继承自* `JSON` 的 `sqlalchemy.types.JSON.__init__` *方法*

构造一个 `JSON` 类型。

参数：

**none_as_null=False** –

如果为 True，则将值 `None` 持久化为 SQL NULL 值，而不是 `null` 的 JSON 编码。请注意，当此标志为 False 时，`null()` 构造仍然可以用于持久化 NULL 值，可以直接作为参数值传递，由 `JSON` 类型特殊解释为 SQL NULL：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"data": null()})
```

注意

`JSON.none_as_null` 不适用于传递给 `Column.default` 和 `Column.server_default` 的值；这些参数的值为 `None` 表示“没有默认值”。

此外，在 SQL 比较表达式中使用时，Python 值 `None` 仍然指的是 SQL 空值，而不是 JSON 的 NULL。`JSON.none_as_null` 标志明确指示了值在 INSERT 或 UPDATE 语句中的**持久性**。`JSON.NULL` 值应该用于希望与 JSON null 进行比较的 SQL 表达式。

另请参阅

`JSON.NULL`

```py
class sqlalchemy.dialects.mssql.MONEY
```

**类签名**

类 `sqlalchemy.dialects.mssql.MONEY` (`sqlalchemy.types.TypeEngine`)

```py
class sqlalchemy.dialects.mssql.NCHAR
```

SQL NCHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.NCHAR` (`sqlalchemy.types.Unicode`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` 的 `sqlalchemy.types.String.__init__` *方法*

创建一个保存字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能要求在 DDL 中使用长度，并且如果包含没有长度的 `VARCHAR`，则在发出 `CREATE TABLE` DDL 时会引发异常。值是按字节还是按字符解释是数据库特定的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级别排序。使用由 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.NTEXT
```

MSSQL NTEXT 类型，用于最多 2³⁰ 个字符的可变长度 Unicode 文本。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.NTEXT` (`sqlalchemy.types.UnicodeText`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数:

+   `length` – 可选的，用于 DDL 和 CAST 表达式中的列的长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用`length`，如果包含一个没有长度的`VARCHAR`，则会在发出`CREATE TABLE` DDL 时引发异常。值是以字节还是字符解释是数据库特定的。

+   `collation` –

    可选的，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.NVARCHAR
```

SQL NVARCHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.NVARCHAR` (`sqlalchemy.types.Unicode`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数:

+   `length` – 可选的，用于 DDL 和 CAST 表达式中的列的长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用`length`，如果包含一个没有长度的`VARCHAR`，则会在发出`CREATE TABLE` DDL 时引发异常。值是以字节还是字符解释是与数据库相关的。

+   `collation` –

    可选的，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行渲染。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用 `Column` 期望存储非 ASCII 数据的 `Unicode` 或 `UnicodeText` 数据类型。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.REAL
```

SQL Server REAL 数据类型。

**类签名**

类 `sqlalchemy.dialects.mssql.REAL` (`sqlalchemy.types.REAL`)。

```py
class sqlalchemy.dialects.mssql.ROWVERSION
```

实现 SQL Server ROWVERSION 类型。

ROWVERSION 数据类型是 SQL Server TIMESTAMP 数据类型的同义词，但当前的 SQL Server 文档建议将 ROWVERSION 用于未来新的数据类型。

ROWVERSION 数据类型 **不会** 作为自身反映（例如自省）从数据库中返回；返回的数据类型将是 `TIMESTAMP`。

这是一个只读数据类型，不支持插入值。

新版本 1.2 中的新增功能。

另请参阅

`TIMESTAMP`

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.ROWVERSION` (`sqlalchemy.dialects.mssql.base.TIMESTAMP`)。

```py
method __init__(convert_int=False)
```

*继承自* `TIMESTAMP` *的* `sqlalchemy.dialects.mssql.base.TIMESTAMP.__init__` *方法*。

构造一个 TIMESTAMP 或 ROWVERSION 类型。

参数：

**convert_int** – 如果为 True，则在读取时将二进制整数值转换为整数。

新版本 1.2 中的新增功能。

```py
class sqlalchemy.dialects.mssql.SMALLDATETIME
```

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.SMALLDATETIME` (`sqlalchemy.dialects.mssql.base._DateTimeBase`, `sqlalchemy.types.DateTime`)。

```py
method __init__(timezone: bool = False)
```

*继承自* `DateTime` *的* `sqlalchemy.types.DateTime.__init__` *方法*。

构造一个新的 `DateTime`。

参数：

**timezone** – 布尔值。指示日期/时间类型是否应启用时区支持，仅当**基本日期/时间持有类型可用**时。建议在使用此标志时直接使用 `TIMESTAMP` 数据类型，因为某些数据库包含与支持时区的 TIMESTAMP 数据类型不同的单独的通用日期/时间持有类型，例如 Oracle。

```py
class sqlalchemy.dialects.mssql.SMALLMONEY
```

**类签名**

类`sqlalchemy.dialects.mssql.SMALLMONEY`（`sqlalchemy.types.TypeEngine`）

```py
class sqlalchemy.dialects.mssql.SQL_VARIANT
```

**类签名**

类`sqlalchemy.dialects.mssql.SQL_VARIANT`（`sqlalchemy.types.TypeEngine`）

```py
class sqlalchemy.dialects.mssql.TEXT
```

SQL TEXT 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.TEXT`（`sqlalchemy.types.Text`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个持有字符串的类型。

参数：

+   `length` – 可选项，用于 DDL 和 CAST 表达式中的列长度。如果不会发出 `CREATE TABLE`，则可以安全地省略。某些数据库可能要求在 DDL 中使用长度，并且如果包括没有长度的 `VARCHAR`，则在发出 `CREATE TABLE` DDL 时会引发异常。该值是以字节还是字符解释是数据库特定的。

+   `collation` –

    可选项，用于 DDL 和 CAST 表达式中的列级排序。在 SQLite、MySQL 和 PostgreSQL 中使用 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Unicode` 或 `UnicodeText` 数据类型应用于预期存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.TIME
```

**类签名**

类`sqlalchemy.dialects.mssql.TIME`（`sqlalchemy.types.TIME`）

```py
class sqlalchemy.dialects.mssql.TIMESTAMP
```

实现 SQL Server TIMESTAMP 类型。

注意，这与 SQL 标准的 TIMESTAMP 类型**完全不同**，该类型不受 SQL Server 支持。它是一个只读数据类型，不支持插入值。

版本 1.2 中的新功能。

另请参阅

`ROWVERSION`

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.TIMESTAMP` (`sqlalchemy.types._Binary`)

```py
method __init__(convert_int=False)
```

构造 TIMESTAMP 或 ROWVERSION 类型。

参数：

**convert_int** – 如果为 True，则二进制整数值将在读取时转换为整数。

新功能，版本 1.2。

```py
class sqlalchemy.dialects.mssql.TINYINT
```

**类签名**

类 `sqlalchemy.dialects.mssql.TINYINT` (`sqlalchemy.types.Integer`)

```py
class sqlalchemy.dialects.mssql.UNIQUEIDENTIFIER
```

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.UNIQUEIDENTIFIER` (`sqlalchemy.types.Uuid`)

```py
method __init__(as_uuid: bool = True)
```

构造一个 `UNIQUEIDENTIFIER` 类型。

参数：

**as_uuid=True** –

如果为 True，则将值解释为 Python uuid 对象，通过 DBAPI 转换为/从字符串。

```py
class sqlalchemy.dialects.mssql.VARBINARY
```

MSSQL VARBINARY 类型。

此类型为核心 `VARBINARY` 类型添加了其他功能，包括“弃用大型类型”模式，在此模式下将呈现 `VARBINARY(max)` 或 IMAGE，以及 SQL Server `FILESTREAM` 选项。

另请参阅

大型文本/二进制类型弃用

**类签名**

类 `sqlalchemy.dialects.mssql.VARBINARY` (`sqlalchemy.types.VARBINARY`, `sqlalchemy.types.LargeBinary`)

```py
method __init__(length=None, filestream=False)
```

构造一个 VARBINARY 类型。

参数：

+   `length` – 可选，用于 DDL 语句中的列的长度，用于那些接受长度的二进制类型，例如 MySQL BLOB 类型。

+   `filestream=False` –

    如果为 True，在表定义中渲染 `FILESTREAM` 关键字。在这种情况下，`length` 必须为 `None` 或 `'max'`。

    新功能，版本 1.4.31。

```py
class sqlalchemy.dialects.mssql.VARCHAR
```

SQL VARCHAR 类型。

**类签名**

类 `sqlalchemy.dialects.mssql.VARCHAR` (`sqlalchemy.types.String`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*从* `String` *的* `sqlalchemy.types.String.__init__` *方法继承*

创建一个字符串持有类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。该值是以字节还是字符解释的取决于数据库。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.mssql.XML
```

MSSQL XML 类型。

这是一个用于反射目的的占位符类型，不包括任何 Python 端数据类型支持。它也不支持额外的参数，如“CONTENT”、“DOCUMENT”、“xml_schema_collection”。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mssql.XML` (`sqlalchemy.types.Text`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` 的 `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。该值是以字节还是字符解释的取决于数据库。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

## PyODBC

通过 PyODBC 驱动程序支持 Microsoft SQL Server 数据库。

### DBAPI

PyODBC 的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/pyodbc/`](https://pypi.org/project/pyodbc/)

### 连接

连接字符串：

```py
mssql+pyodbc://<username>:<password>@<dsnname>
```

### 连接到 PyODBC

此处的 URL 将被翻译为 PyODBC 连接字符串，详细信息请参阅 [ConnectionStrings](https://code.google.com/p/pyodbc/wiki/ConnectionStrings)。

#### DSN 连接

ODBC 中的 DSN 连接意味着在客户端机器上配置了预先存在的 ODBC 数据源。然后，应用程序指定此数据源的名称，其中包括诸如正在使用的特定 ODBC 驱动程序以及数据库的网络地址等详细信息。假设客户端已配置了数据源，则基本的基于 DSN 的连接如下所示：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@some_dsn")
```

以上内容将以下连接字符串传递给 PyODBC：

```py
DSN=some_dsn;UID=scott;PWD=tiger
```

如果省略了用户名和密码，则 DSN 表单还将向 ODBC 字符串添加 `Trusted_Connection=yes` 指令。

#### 主机名连接

主机名连接也受到了 pyodbc 的支持。这通常比 DSN 更容易使用，并且具有另一个优势，即可以在 URL 中本地指定要连接的特定数据库名称，而不是作为数据源配置的一部分固定下来。

在使用主机名连接时，驱动程序名称也必须在 URL 的查询参数中指定。由于这些名称通常带有空格，因此名称必须进行 URL 编码，这意味着使用加号代替空格：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=ODBC+Driver+17+for+SQL+Server")
```

`driver` 关键字对于 pyodbc 方言很重要，必须以小写形式指定。

在查询字符串中传递的任何其他名称都会在 pyodbc 连接字符串中传递，例如`authentication`、`TrustServerCertificate`等。多个关键字参数必须用与号(`&`)分隔；这些在生成内部 pyodbc 连接字符串时将被翻译为分号：

```py
e = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?"
    "driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes"
    "&authentication=ActiveDirectoryIntegrated"
)
```

可以使用 `URL` 构造等效的 URL：

```py
from sqlalchemy.engine import URL
connection_url = URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssql2017",
    port=1433,
    database="test",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "TrustServerCertificate": "yes",
        "authentication": "ActiveDirectoryIntegrated",
    },
)
```

#### 通过精确的 Pyodbc 字符串

一个 PyODBC 连接字符串也可以直接以 pyodbc 的格式发送，如 [PyODBC 文档](https://github.com/mkleehammer/pyodbc/wiki/Connecting-to-databases) 中所述，使用参数 `odbc_connect`。一个 `URL` 对象可以帮助简化此过程：

```py
from sqlalchemy.engine import URL
connection_string = "DRIVER={SQL Server Native Client 10.0};SERVER=dagger;DATABASE=test;UID=user;PWD=password"
connection_url = URL.create("mssql+pyodbc", query={"odbc_connect": connection_string})

engine = create_engine(connection_url)
```

#### 使用访问令牌连接数据库

某些数据库服务器只设置为仅接受访问令牌进行登录。例如，SQL Server 允许使用 Azure Active Directory 令牌连接到数据库。这需要使用 `azure-identity` 库创建凭据对象。有关身份验证步骤的更多信息，请参阅 [Microsoft 文档](https://docs.microsoft.com/en-us/azure/developer/python/azure-sdk-authenticate?tabs=bash)。

获得引擎后，每次请求连接时都需要将凭证发送到`pyodbc.connect`。一种方法是在引擎上设置事件监听器，该监听器将凭证令牌添加到方言的连接调用中。更详细地讨论了这一点，可以参考生成动态认证令牌。特别是对于 SQL Server，这是作为一个 ODBC 连接属性传递的，具有由微软描述的数据结构。

以下代码片段将创建一个连接到 Azure SQL 数据库的引擎，使用 Azure 凭据连接：

```py
import struct
from sqlalchemy import create_engine, event
from sqlalchemy.engine.url import URL
from azure import identity

SQL_COPT_SS_ACCESS_TOKEN = 1256  # Connection option for access tokens, as defined in msodbcsql.h
TOKEN_URL = "https://database.windows.net/"  # The token URL for any Azure SQL database

connection_string = "mssql+pyodbc://@my-server.database.windows.net/myDb?driver=ODBC+Driver+17+for+SQL+Server"

engine = create_engine(connection_string)

azure_credentials = identity.DefaultAzureCredential()

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    # remove the "Trusted_Connection" parameter that SQLAlchemy adds
    cargs[0] = cargs[0].replace(";Trusted_Connection=Yes", "")

    # create token credential
    raw_token = azure_credentials.get_token(TOKEN_URL).token.encode("utf-16-le")
    token_struct = struct.pack(f"<I{len(raw_token)}s", len(raw_token), raw_token)

    # apply it to keyword arguments
    cparams["attrs_before"] = {SQL_COPT_SS_ACCESS_TOKEN: token_struct}
```

提示

当没有用户名或密码时，SQLAlchemy pyodbc 方言当前会添加`Trusted_Connection`令牌。根据 Microsoft 的[使用 Azure 访问令牌](https://docs.microsoft.com/en-us/sql/connect/odbc/using-azure-active-directory#authenticating-with-an-access-token)文档，这需要删除，该文档指出，使用访问令牌时的连接字符串不得包含`UID`、`PWD`、`Authentication`或`Trusted_Connection`参数。#### 避免在 Azure Synapse Analytics 上出现与事务相关的异常

Azure Synapse Analytics 在其事务处理方面与普通的 SQL Server 有显着的差异；在某些情况下，Synapse 事务内的错误可能导致服务器端任意终止，然后导致 DBAPI 的`.rollback()`方法（以及`.commit()`）失败。这个问题阻止了通常的 DBAPI 合同允许`.rollback()`在没有事务存在时悄悄通过，因为驱动程序不期望出现这种情况。这种失败的症状是在尝试在某个操作失败后发出`.rollback()`时出现的异常，消息类似于“找不到相应的事务。 (111214)”。

通过以下方式向 SQL Server 方言的`create_engine()`函数传递`ignore_no_transaction_on_rollback=True`参数，可以处理此特定情况：

```py
engine = create_engine(connection_url, ignore_no_transaction_on_rollback=True)
```

使用上述参数，方言将捕获在`connection.rollback()`期间引发的`ProgrammingError`异常，并在错误消息中包含代码`111214`时发出警告，但不会引发异常。

在版本 1.4.40 中新增了`ignore_no_transaction_on_rollback=True`参数。

#### 为 Azure SQL Data Warehouse (DW) 连接启用自动提交

Azure SQL Data Warehouse 不支持事务，这可能会导致 SQLAlchemy 的“自动开始”（以及隐式提交/回滚）行为出现问题。我们可以通过在 pyodbc 和引擎级别启用自动提交来避免这些问题：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="dw.azure.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 17 for SQL Server",
        "autocommit": "True",
    },
)

engine = create_engine(connection_url).execution_options(
    isolation_level="AUTOCOMMIT"
)
```

#### 避免将大型字符串参数作为 TEXT/NTEXT 发送

出于历史原因，默认情况下，Microsoft 的 SQL Server ODBC 驱动程序将长字符串参数（大于 4000 个 SBCS 字符或 2000 个 Unicode 字符）发送为 TEXT/NTEXT 值。多年来，TEXT 和 NTEXT 已经被弃用，并且开始在新版本的 SQL_Server/Azure 中引起兼容性问题。例如，请参阅[此问题](https://github.com/mkleehammer/pyodbc/issues/835)。

从 ODBC Driver 18 for SQL Server 开始，我们可以通过使用`LongAsMax=Yes`连接字符串参数覆盖传统行为，并将长字符串作为 varchar(max)/nvarchar(max) 传递：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssqlserver.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "LongAsMax": "Yes",
    },
)
```

### Pyodbc 连接池 / 连接关闭行为

PyODBC 默认使用内部[连接池](https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module#pooling)，这意味着连接的生命周期将比在 SQLAlchemy 中更长。由于 SQLAlchemy 有自己的连接池行为，通常最好禁用此行为。此行为只能在创建任何连接之前在 PyODBC 模块级别全局禁用，**之前**：

```py
import pyodbc

pyodbc.pooling = False

# don't use the engine before pooling is set to False
engine = create_engine("mssql+pyodbc://user:pass@dsn")
```

如果将此变量保留在其默认值`True`，**应用程序将继续保持活动数据库连接**，即使 SQLAlchemy 引擎本身完全丢弃连接或引擎被处理。

另请参阅

[连接池](https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module#pooling) - 在 PyODBC 文档中。

### 驱动程序 / Unicode 支持

PyODBC 最适合与 Microsoft ODBC 驱动程序一起使用，特别是在 Python 2 和 Python 3 上的 Unicode 支持方面。

在 Linux 或 OSX 上使用 FreeTDS ODBC 驱动程序与 PyODBC **不**推荐；在这个领域，包括在 Microsoft 为 Linux 和 OSX 提供 ODBC 驱动程序之前，历史上存在许多与 Unicode 相关的问题。现在 Microsoft 为所有平台提供驱动程序，对于 PyODBC 支持，建议使用这些驱动程序。FreeTDS 对于非 ODBC 驱动程序（如 pymssql）仍然很重要，在那里它运行得非常好。

### 行数支持

截至 SQLAlchemy 2.0.5 版本，已解决了 Pyodbc 与 SQLAlchemy ORM 的“版本化行”功能的先前限制。请参阅行数支持 / ORM 版本控制处的说明。

### 快速执行多个模式

PyODBC 驱动程序包括对“快速执行多个”执行模式的支持，当使用 Microsoft ODBC 驱动程序时，对于**适合内存的有限大小批次**的 DBAPI `executemany()` 调用，可以大大减少往返次数。该功能通过在要使用 executemany 调用时在 DBAPI 游标上设置属性`.fast_executemany` 来启用。当仅使用**Microsoft ODBC 驱动程序**时，SQLAlchemy PyODBC SQL Server 方言支持通过将 `fast_executemany` 参数传递给 `create_engine()` 来支持此参数：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
    fast_executemany=True)
```

从版本 2.0.9 开始更改：- `fast_executemany`参数现在对于执行多个参数集的所有 INSERT 语句具有其预期效果，不包括 RETURNING。以前，SQLAlchemy 2.0 的 insertmanyvalues 功能会导致在大多数情况下即使指定了`fast_executemany`也不会使用。

版本 1.3 中的新功能。

另请参阅

[fast executemany](https://github.com/mkleehammer/pyodbc/wiki/Features-beyond-the-DB-API#fast_executemany) - 在 github 上  ### Setinputsizes 支持

从版本 2.0 开始，pyodbc 的`cursor.setinputsizes()`方法用于所有语句执行，除了在`fast_executemany=True`时不支持`cursor.executemany()`调用（假设 insertmanyvalues 已启用，“fastexecutemany”在任何情况下都不会对 INSERT 语句生效）。

通过向`create_engine()`传递`use_setinputsizes=False`可以禁用`cursor.setinputsizes()`的使用。

当`use_setinputsizes`保持默认值`True`时，传递给`cursor.setinputsizes()`的特定每种类型符号可以使用`DialectEvents.do_setinputsizes()`钩子进行程序化定制。请参阅该方法以获取用法示例。

从版本 2.0 开始更改：mssql+pyodbc 方言现在默认为所有语句执行使用`use_setinputsizes=True`，除了在`fast_executemany=True`时的 cursor.executemany()调用。可以通过向`create_engine()`传递`use_setinputsizes=False`来关闭此行为。

### DBAPI

PyODBC 的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/pyodbc/`](https://pypi.org/project/pyodbc/)

### 连接

连接字符串：

```py
mssql+pyodbc://<username>:<password>@<dsnname>
```

### 连接到 PyODBC

此处的 URL 将被翻译为 PyODBC 连接字符串，详细信息请参阅[ConnectionStrings](https://code.google.com/p/pyodbc/wiki/ConnectionStrings)。

#### DSN 连接

ODBC 中的 DSN 连接意味着客户端机器上配置了预先存在的 ODBC 数据源。然后，应用程序指定此数据源的名称，其中包括诸如正在使用的特定 ODBC 驱动程序以及数据库的网络地址等详细信息。假设客户端上配置了数据源，基本的基于 DSN 的连接如下所示：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@some_dsn")
```

以上内容将向 PyODBC 传递以下连接字符串：

```py
DSN=some_dsn;UID=scott;PWD=tiger
```

如果省略了用户名和密码，DSN 表单还将向 ODBC 字符串添加`Trusted_Connection=yes`指令。

#### 主机名连接

pyodbc 也支持基于主机名的连接。这通常比使用 DSN 更容易，并且具有以下额外的优势：可以在 URL 中本地指定要连接的特定数据库名称，而不是将其作为数据源配置的固定部分。

使用主机名连接时，还必须在 URL 的查询参数中指定驱动程序名称。由于这些名称通常包含空格，因此名称必须进行 URL 编码，这意味着使用加号代替空格：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=ODBC+Driver+17+for+SQL+Server")
```

`driver` 关键字对于 pyodbc 方言是重要的，并且必须以小写形式指定。

在查询字符串中传递的任何其他名称都将通过 pyodbc 连接字符串传递，例如 `authentication`、`TrustServerCertificate` 等。多个关键字参数必须用和号 (`&`) 分隔；在内部生成 pyodbc 连接字符串时，这些将被翻译为分号：

```py
e = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?"
    "driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes"
    "&authentication=ActiveDirectoryIntegrated"
)
```

可以使用 `URL` 构造等效的 URL：

```py
from sqlalchemy.engine import URL
connection_url = URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssql2017",
    port=1433,
    database="test",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "TrustServerCertificate": "yes",
        "authentication": "ActiveDirectoryIntegrated",
    },
)
```

#### 通过准确的 Pyodbc 字符串传递

也可以直接以 pyodbc 的格式发送 PyODBC 连接字符串，如 [PyODBC 文档](https://github.com/mkleehammer/pyodbc/wiki/Connecting-to-databases) 中所指定，使用参数 `odbc_connect`。`URL` 对象可以帮助简化这一过程：

```py
from sqlalchemy.engine import URL
connection_string = "DRIVER={SQL Server Native Client 10.0};SERVER=dagger;DATABASE=test;UID=user;PWD=password"
connection_url = URL.create("mssql+pyodbc", query={"odbc_connect": connection_string})

engine = create_engine(connection_url)
```

#### 使用访问令牌连接到数据库

一些数据库服务器仅设置为仅接受访问令牌进行登录。例如，SQL Server 允许使用 Azure Active Directory 令牌连接到数据库。这需要使用 `azure-identity` 库创建凭据对象。有关身份验证步骤的更多信息，请参阅 [Microsoft 文档](https://docs.microsoft.com/en-us/azure/developer/python/azure-sdk-authenticate?tabs=bash)。

获得引擎后，每次请求连接时都需要将凭据发送给 `pyodbc.connect`。一种方法是在引擎上设置事件侦听器，以将凭据令牌添加到方言的连接调用中。关于这一点，可以在 生成动态认证令牌 中进行更一般的讨论。特别对于 SQL Server，这是作为 ODBC 连接属性传递的，具有由 Microsoft 描述的数据结构。

以下代码片段将创建一个使用 Azure 凭据连接到 Azure SQL 数据库的引擎：

```py
import struct
from sqlalchemy import create_engine, event
from sqlalchemy.engine.url import URL
from azure import identity

SQL_COPT_SS_ACCESS_TOKEN = 1256  # Connection option for access tokens, as defined in msodbcsql.h
TOKEN_URL = "https://database.windows.net/"  # The token URL for any Azure SQL database

connection_string = "mssql+pyodbc://@my-server.database.windows.net/myDb?driver=ODBC+Driver+17+for+SQL+Server"

engine = create_engine(connection_string)

azure_credentials = identity.DefaultAzureCredential()

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    # remove the "Trusted_Connection" parameter that SQLAlchemy adds
    cargs[0] = cargs[0].replace(";Trusted_Connection=Yes", "")

    # create token credential
    raw_token = azure_credentials.get_token(TOKEN_URL).token.encode("utf-16-le")
    token_struct = struct.pack(f"<I{len(raw_token)}s", len(raw_token), raw_token)

    # apply it to keyword arguments
    cparams["attrs_before"] = {SQL_COPT_SS_ACCESS_TOKEN: token_struct}
```

提示

当没有用户名或密码时，SQLAlchemy pyodbc 方言当前会添加`Trusted_Connection`令牌。根据微软的[文档](https://docs.microsoft.com/en-us/sql/connect/odbc/using-azure-active-directory#authenticating-with-an-access-token)，使用访问令牌时连接字符串不能包含`UID`、`PWD`、`Authentication`或`Trusted_Connection`参数，这需要删除。#### 避免 Azure Synapse Analytics 上的与事务相关的异常

Azure Synapse Analytics 在处理事务方面与普通的 SQL Server 有显着的不同；在某些情况下，Synapse 事务内的错误可能导致服务器端任意终止，这将导致 DBAPI 的`.rollback()`方法（以及`.commit()`）失败。该问题阻止了通常的 DBAPI 合同，即允许`.rollback()`在没有事务存在时静默通过，因为驱动程序不期望出现这种情况。此失败的症状是，在某种操作失败后尝试发出`.rollback()`时出现类似于‘No corresponding transaction found. (111214)’的异常消息。

可以通过以下方式通过`create_engine()`函数将`ignore_no_transaction_on_rollback=True`传递给 SQL Server 方言来处理此特定情况：

```py
engine = create_engine(connection_url, ignore_no_transaction_on_rollback=True)
```

使用上述参数，方言将捕获在`connection.rollback()`期间引发的`ProgrammingError`异常，并在错误消息包含代码`111214`时发出警告，但不会引发异常。

新版本 1.4.40 中添加了`ignore_no_transaction_on_rollback=True`参数。

#### 为 Azure SQL Data Warehouse（DW）连接启用自动提交

Azure SQL Data Warehouse 不支持事务，这可能会导致 SQLAlchemy 的“autobegin”（和隐式提交/回滚）行为出现问题。我们可以通过在 pyodbc 和引擎级别启用自动提交来避免这些问题：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="dw.azure.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 17 for SQL Server",
        "autocommit": "True",
    },
)

engine = create_engine(connection_url).execution_options(
    isolation_level="AUTOCOMMIT"
)
```

#### 避免将大型字符串参数发送为 TEXT/NTEXT

出于历史原因，默认情况下，Microsoft 的 SQL Server ODBC 驱动程序将长字符串参数（大于 4000 个 SBCS 字符或 2000 个 Unicode 字符）发送为 TEXT/NTEXT 值。多年来，TEXT 和 NTEXT 已经过时，并且开始与 SQL_Server/Azure 的新版本引起兼容性问题。例如，请参阅[此问题](https://github.com/mkleehammer/pyodbc/issues/835)。

从 ODBC 驱动程序 18 开始，我们可以通过`LongAsMax=Yes`连接字符串参数覆盖传统行为，并将长字符串传递为 varchar(max)/nvarchar(max)：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssqlserver.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "LongAsMax": "Yes",
    },
)
```

#### DSN 连接

ODBC 中的 DSN 连接意味着客户端计算机上配置了预定义的 ODBC 数据源。然后，应用程序指定此数据源的名称，其中包括诸如正在使用的特定 ODBC 驱动程序以及数据库的网络地址等详细信息。假设客户端上配置了数据源，基本的基于 DSN 的连接如下所示：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@some_dsn")
```

以上内容将传递以下连接字符串给 PyODBC：

```py
DSN=some_dsn;UID=scott;PWD=tiger
```

如果省略了用户名和密码，则 DSN 表单还将在 ODBC 字符串中添加 `Trusted_Connection=yes` 指令。

#### 主机名连接

基于主机名的连接也受 pyodbc 支持。这些通常比 DSN 更容易使用，并且具有其他优点，即可以在 URL 中本地指定要连接的特定数据库名称，而不是作为数据源配置的一部分固定下来。

当使用主机名连接时，驱动程序名称也必须在 URL 的查询参数中指定。由于这些名称通常包含空格，因此名称必须进行 URL 编码，这意味着用加号代替空格：

```py
engine = create_engine("mssql+pyodbc://scott:tiger@myhost:port/databasename?driver=ODBC+Driver+17+for+SQL+Server")
```

`driver` 关键字对于 pyodbc 方言至关重要，必须以小写指定。

查询字符串中传递的任何其他名称都将通过 pyodbc 连接字符串传递，例如`authentication`、`TrustServerCertificate`等。多个关键字参数必须用 ampersand (`&`) 分隔；在生成内部 pyodbc 连接字符串时，这些将被翻译为分号：

```py
e = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?"
    "driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes"
    "&authentication=ActiveDirectoryIntegrated"
)
```

可以使用 `URL` 构造等效的 URL：

```py
from sqlalchemy.engine import URL
connection_url = URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssql2017",
    port=1433,
    database="test",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "TrustServerCertificate": "yes",
        "authentication": "ActiveDirectoryIntegrated",
    },
)
```

#### 经过准确的 Pyodbc 字符串

PyODBC 连接字符串也可以直接以 pyodbc 格式发送，如 [PyODBC 文档](https://github.com/mkleehammer/pyodbc/wiki/Connecting-to-databases) 中所述，使用参数 `odbc_connect`。`URL` 对象可以帮助简化此过程：

```py
from sqlalchemy.engine import URL
connection_string = "DRIVER={SQL Server Native Client 10.0};SERVER=dagger;DATABASE=test;UID=user;PWD=password"
connection_url = URL.create("mssql+pyodbc", query={"odbc_connect": connection_string})

engine = create_engine(connection_url)
```

#### 使用访问令牌连接数据库

某些数据库服务器设置为仅接受访问令牌进行登录。例如，SQL Server 允许使用 Azure Active Directory 令牌连接到数据库。这需要使用 `azure-identity` 库创建凭据对象。有关身份验证步骤的更多信息，请参阅 [微软文档](https://docs.microsoft.com/zh-cn/azure/developer/python/azure-sdk-authenticate?tabs=bash)。

获得引擎后，每次请求连接都需要将凭据发送到 `pyodbc.connect`。一种方法是在引擎上设置一个事件侦听器，该事件侦听器将凭据令牌添加到方言的连接调用中。这在 生成动态认证令牌 中更一般地讨论过。特别对于 SQL Server，这是作为 ODBC 连接属性传递的，其数据结构由 Microsoft 描述。

以下代码片段将创建一个引擎，该引擎使用 Azure 凭据连接到 Azure SQL 数据库：

```py
import struct
from sqlalchemy import create_engine, event
from sqlalchemy.engine.url import URL
from azure import identity

SQL_COPT_SS_ACCESS_TOKEN = 1256  # Connection option for access tokens, as defined in msodbcsql.h
TOKEN_URL = "https://database.windows.net/"  # The token URL for any Azure SQL database

connection_string = "mssql+pyodbc://@my-server.database.windows.net/myDb?driver=ODBC+Driver+17+for+SQL+Server"

engine = create_engine(connection_string)

azure_credentials = identity.DefaultAzureCredential()

@event.listens_for(engine, "do_connect")
def provide_token(dialect, conn_rec, cargs, cparams):
    # remove the "Trusted_Connection" parameter that SQLAlchemy adds
    cargs[0] = cargs[0].replace(";Trusted_Connection=Yes", "")

    # create token credential
    raw_token = azure_credentials.get_token(TOKEN_URL).token.encode("utf-16-le")
    token_struct = struct.pack(f"<I{len(raw_token)}s", len(raw_token), raw_token)

    # apply it to keyword arguments
    cparams["attrs_before"] = {SQL_COPT_SS_ACCESS_TOKEN: token_struct}
```

提示

当没有用户名或密码时，SQLAlchemy pyodbc 方言当前会添加 `Trusted_Connection` 令牌。根据 Microsoft 的 [Azure 访问令牌文档](https://docs.microsoft.com/en-us/sql/connect/odbc/using-azure-active-directory#authenticating-with-an-access-token)，当使用访问令牌时，连接字符串不得包含 `UID`、`PWD`、`Authentication` 或 `Trusted_Connection` 参数，因此需要删除此参数。

#### 避免在 Azure Synapse Analytics 上出现与事务相关的异常

Azure Synapse Analytics 在事务处理方面与普通 SQL Server 有显着差异；在某些情况下，Synapse 事务中的错误可能导致服务器端任意终止，这会导致 DBAPI 的 `.rollback()` 方法（以及 `.commit()`）失败。该问题阻止了通常的 DBAPI 契约，即允许 `.rollback()` 在没有事务存在时悄无声息地传递，因为驱动程序不期望出现这种情况。此失败的症状是在尝试在某种操作失败后发出 `.rollback()` 时出现类似于 'No corresponding transaction found. (111214)' 的消息的异常。

通过以下方式将 `ignore_no_transaction_on_rollback=True` 传递给 SQL Server 方言，可以处理这种特殊情况，通过 `create_engine()` 函数：

```py
engine = create_engine(connection_url, ignore_no_transaction_on_rollback=True)
```

使用上述参数，方言将捕获 `connection.rollback()` 期间引发的 `ProgrammingError` 异常，并在错误消息中包含代码 `111214` 时发出警告，但不会引发异常。

新版本 1.4.40 中新增了 `ignore_no_transaction_on_rollback=True` 参数。

#### 为 Azure SQL 数据仓库 (DW) 连接启用自动提交

Azure SQL 数据仓库不支持事务，这可能会导致 SQLAlchemy 的 “自动开始” (以及隐式提交/回滚) 行为出现问题。我们可以通过在 pyodbc 和引擎级别启用自动提交来避免这些问题：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="dw.azure.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 17 for SQL Server",
        "autocommit": "True",
    },
)

engine = create_engine(connection_url).execution_options(
    isolation_level="AUTOCOMMIT"
)
```

#### 避免将大型字符串参数发送为 TEXT/NTEXT

出于历史原因，默认情况下，Microsoft 的 ODBC 驱动程序会将长字符串参数（大于 4000 个 SBCS 字符或 2000 个 Unicode 字符）发送为 TEXT/NTEXT 值。多年来，TEXT 和 NTEXT 已经被弃用，并且开始在新版本的 SQL_Server/Azure 中引起兼容性问题。例如，请参阅[此问题](https://github.com/mkleehammer/pyodbc/issues/835)。

从 ODBC Driver 18 for SQL Server 开始，我们可以通过`LongAsMax=Yes`连接字符串参数覆盖传递长字符串作为 varchar(max)/nvarchar(max)的传统行为：

```py
connection_url = sa.engine.URL.create(
    "mssql+pyodbc",
    username="scott",
    password="tiger",
    host="mssqlserver.example.com",
    database="mydb",
    query={
        "driver": "ODBC Driver 18 for SQL Server",
        "LongAsMax": "Yes",
    },
)
```

### Pyodbc 连接池/连接关闭行为

PyODBC 默认使用内部[连接池](https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module#pooling)，这意味着连接的生命周期将比在 SQLAlchemy 内部更长。由于 SQLAlchemy 具有自己的连接池行为，通常最好禁用此行为。此行为只能在创建任何连接之前在 PyODBC 模块级别全局禁用：

```py
import pyodbc

pyodbc.pooling = False

# don't use the engine before pooling is set to False
engine = create_engine("mssql+pyodbc://user:pass@dsn")
```

如果将此变量保留在其默认值`True`，**应用程序将继续保持活动数据库连接**，即使 SQLAlchemy 引擎本身完全丢弃连接或引擎被处理。

另请参见

[连接池](https://github.com/mkleehammer/pyodbc/wiki/The-pyodbc-Module#pooling) - 在 PyODBC 文档中。

### 驱动程序/Unicode 支持

PyODBC 最适合与 Microsoft ODBC 驱动程序一起使用，特别是在 Python 2 和 Python 3 的 Unicode 支持领域。

在 Linux 或 OSX 上使用 FreeTDS ODBC 驱动与 PyODBC **不**推荐；在这个领域历史上存在许多与 Unicode 相关的问题，包括在 Microsoft 为 Linux 和 OSX 提供 ODBC 驱动之前。现在 Microsoft 为所有平台提供驱动程序，对于 PyODBC 支持，这些是推荐的。FreeTDS 仍然适用于非 ODBC 驱动程序，如 pymssql，在这里它运行得非常好。

### 行数支持

截至 SQLAlchemy 2.0.5 版本，已解决了 Pyodbc 与 SQLAlchemy ORM 的“版本化行”功能的先前限制。请参阅 Rowcount Support / ORM Versioning 中的说明。

### 快速 Executemany 模式

PyODBC 驱动程序包括对“快速 executemany”执行模式的支持，当使用 Microsoft ODBC 驱动程序时，对于适合内存的**有限大小批次**的 DBAPI `executemany()`调用大大减少了往返次数。当要使用 executemany 调用时，在 DBAPI 游标上设置属性`.fast_executemany`即可启用此功能。SQLAlchemy PyODBC SQL Server 方言通过将`fast_executemany`参数传递给`create_engine()`来支持此参数，仅使用**Microsoft ODBC 驱动程序**：

```py
engine = create_engine(
    "mssql+pyodbc://scott:tiger@mssql2017:1433/test?driver=ODBC+Driver+17+for+SQL+Server",
    fast_executemany=True)
```

2.0.9 版本更改：- `fast_executemany` 参数现在对具有多个参数集的所有 INSERT 语句产生了预期的效果，这些参数集不包括 RETURNING。在以前的情况下，即使指定了，SQLAlchemy 2.0 的 insertmanyvalues 特性也会导致在大多数情况下不使用 `fast_executemany`。

新功能版本 1.3。

另请参阅

[快速执行多次](https://github.com/mkleehammer/pyodbc/wiki/Features-beyond-the-DB-API#fast_executemany) - 在 github 上

### Setinputsizes 支持

从版本 2.0 开始，pyodbc 的 `cursor.setinputsizes()` 方法用于所有语句执行，除了 `cursor.executemany()` 调用 fast_executemany=True 的情况下不支持（假设保持 insertmanyvalues 已启用，“fastexecutemany” 将不会对任何情况下的 INSERT 语句产生作用）。

通过将 `use_setinputsizes=False` 传递给 `create_engine()` 可以禁用 `cursor.setinputsizes()` 的使用。

当 `use_setinputsizes` 保持默认值 `True` 时，传递给 `cursor.setinputsizes()` 的具体每种类型的符号可以使用 `DialectEvents.do_setinputsizes()` 钩子进行程序化定制。查看该方法以获取用法示例。

2.0 版本更改：mssql+pyodbc 方言现在默认为所有语句执行使用 `use_setinputsizes=True`，除了 cursor.executemany() 调用 fast_executemany=True 时的情况。可以通过将 `use_setinputsizes=False` 传递给 `create_engine()` 来关闭此行为。

## pymssql

通过 pymssql 驱动程序支持 Microsoft SQL Server 数据库。

### 连接

连接字符串：

```py
mssql+pymssql://<username>:<password>@<freetds_name>/?charset=utf8
```

pymssql 是一个提供围绕 [FreeTDS](https://www.freetds.org/) 的 Python DBAPI 接口的 Python 模块。

2.0.5 版本更改：pymssql 已恢复到 SQLAlchemy 的持续集成测试中。

### 连接

连接字符串：

```py
mssql+pymssql://<username>:<password>@<freetds_name>/?charset=utf8
```

## aioodbc

通过 aioodbc 驱动程序支持 Microsoft SQL Server 数据库。

### DBAPI

aioodbc 的文档和下载信息（如适用）可在此处获取：[`pypi.org/project/aioodbc/`](https://pypi.org/project/aioodbc/)

### 连接

连接字符串：

```py
mssql+aioodbc://<username>:<password>@<dsnname>
```

通过 aioodbc 驱动程序以 asyncio 风格支持 SQL Server 数据库，该驱动程序本身是围绕 pyodbc 的线程包装器。

新功能版本 2.0.23：增加了 mssql+aioodbc 方言，它是基于 pyodbc 和通用 aio* 方言架构构建的。

使用特殊的 asyncio 中介层，aioodbc 方言可作为 SQLAlchemy asyncio 扩展包的后端使用。

此驱动程序的大多数行为和注意事项与在 SQL Server 上使用的 pyodbc 方言相同；有关一般背景，请参阅 PyODBC。

此方言通常仅应与`create_async_engine()`引擎创建函数一起使用；否则连接样式与 pyodbc 部分中记录的相同：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine(
    "mssql+aioodbc://scott:tiger@mssql2017:1433/test?"
    "driver=ODBC+Driver+18+for+SQL+Server&TrustServerCertificate=yes"
)
```

### DBAPI

文档和下载信息（如果适用）可在以下网址找到：[`pypi.org/project/aioodbc/`](https://pypi.org/project/aioodbc/)

### 连接中

连接字符串：

```py
mssql+aioodbc://<username>:<password>@<dsnname>
```
