# Oracle

> [`docs.sqlalchemy.org/en/20/dialects/oracle.html`](https://docs.sqlalchemy.org/en/20/dialects/oracle.html)

支持 Oracle 数据库。

下表总结了数据库发布版本的当前支持级别。

**支持的 Oracle 版本**

| 支持类型 | 版本 |
| --- | --- |
| CI 完全测试通过 | 18c |
| 正常支持 | 11+ |
| 尽力而为 | 9+ |

## DBAPI 支持

下列方言/DBAPI 选项可用。请参考各个 DBAPI 部分获取连接信息。

+   cx-Oracle

+   python-oracledb

## 自增行为

包含整数主键的 SQLAlchemy Table 对象通常被假定具有“自动递增”行为，这意味着它们可以在插入时生成自己的主键值。在 Oracle 中，有两种可用的选项，即使用 IDENTITY 列（仅限 Oracle 12 及以上版本）或将 SEQUENCE 与列关联。

### 指定 GENERATED AS IDENTITY（Oracle 12 及以上）

从版本 12 开始，Oracle 可以使用 `Identity` 来指定自增行为：

```py
t = Table('mytable', metadata,
    Column('id', Integer, Identity(start=3), primary_key=True),
    Column(...), ...
)
```

上述 `Table` 对象的 CREATE TABLE 如下：

```py
CREATE  TABLE  mytable  (
  id  INTEGER  GENERATED  BY  DEFAULT  AS  IDENTITY  (START  WITH  3),
  ...,
  PRIMARY  KEY  (id)
)
```

`Identity` 对象支持许多选项来控制列的“自动递增”行为，例如起始值、递增值等。除了标准选项外，Oracle 还支持将 `Identity.always` 设置为 `None` 以使用默认生成模式，在 DDL 中呈现 GENERATED AS IDENTITY。它还支持将 `Identity.on_null` 设置为 `True`，以指定在与“BY DEFAULT”标识列一起使用时的 ON NULL。

### 使用 SEQUENCE（所有 Oracle 版本）

旧版 Oracle 没有“自动递增”功能，SQLAlchemy 依赖序列来生成这些值。对于旧版 Oracle，*必须始终明确指定序列以启用自动递增*。这与大多数文档示例不一致，后者假定使用支持自动递增的数据库。要指定序列，请使用传递给 Column 构造函数的 sqlalchemy.schema.Sequence 对象：

```py
t = Table('mytable', metadata,
      Column('id', Integer, Sequence('id_seq', start=1), primary_key=True),
      Column(...), ...
)
```

使用表反射时也需要此步骤，即 autoload_with=engine：

```py
t = Table('mytable', metadata,
      Column('id', Integer, Sequence('id_seq', start=1), primary_key=True),
      autoload_with=engine
)
```

从版本 1.4 起更改：在 `Column` 中添加了 `Identity` 构造，用于指定自增列的选项。

## 事务隔离级别 / 自动提交

Oracle 数据库支持“READ COMMITTED”和“SERIALIZABLE”隔离模式。cx_Oracle 方言还支持 AUTOCOMMIT 隔离级别。

若要使用每个连接的执行选项进行设置：

```py
connection = engine.connect()
connection = connection.execution_options(
    isolation_level="AUTOCOMMIT"
)
```

对于 `READ COMMITTED` 和 `SERIALIZABLE`，Oracle 方言使用 `ALTER SESSION` 在会话级别设置级别，在连接返回到连接池时会恢复到默认设置。

`isolation_level` 的有效值包括：

+   `READ COMMITTED`

+   `AUTOCOMMIT`

+   `SERIALIZABLE`

注意

Oracle 方言实现的 `Connection.get_isolation_level()` 方法必要地使用 Oracle LOCAL_TRANSACTION_ID 函数启动事务；否则通常无法读取任何级别。

此外，如果由于权限或其他原因导致 `v$transaction` 视图不可用，`Connection.get_isolation_level()` 方法将引发异常，这在 Oracle 安装中是常见的。

当方言首次连接到数据库时，cx_Oracle 方言尝试调用 `Connection.get_isolation_level()` 方法以获取“默认”隔离级别。这个默认级别是必要的，以便在使用 `Connection.execution_options()` 方法临时修改连接后，可以将级别重置为连接。在常见事件中，`Connection.get_isolation_level()` 方法由于 `v$transaction` 不可读以及任何其他与数据库相关的故障而引发异常时，级别被假定为“READ COMMITTED”。对于这种初始首次连接条件，不会发出警告，因为预计这是 Oracle 数据库的常见限制。

版本 1.3.16 中新增了对 cx_oracle 方言的 AUTOCOMMIT 支持，以及默认隔离级别的概念

版本 1.3.21 中新增了对 SERIALIZABLE 的支持，以及隔离级别的实时读取。

版本 1.3.22 中的更改：在默认隔离级别由于 v$transaction 视图的权限而无法读取的情况下（这在 Oracle 安装中很常见），默认隔离级别被硬编码为“READ COMMITTED”，这是 1.3.21 之前的行为。

请参阅

设置事务隔离级别，包括 DBAPI 自动提交

## 标识符大小写

在 Oracle 中，数据字典使用大写文本表示所有不区分大小写的标识符名称。另一方面，SQLAlchemy 将所有小写标识符名称视为不区分大小写。Oracle 方言在模式级通信（如表和索引的反射）期间将所有不区分大小写的标识符转换为这两种格式之一。在 SQLAlchemy 方面使用大写名称表示区分大小写的标识符，SQLAlchemy 将引用该名称 - 这将导致与从 Oracle 收到的数据字典数据不匹配，因此除非标识符名称真正被创建为区分大小写（即使用带引号的名称），否则在 SQLAlchemy 方面应使用所有小写名称。

## 最大标识符长度

截至 Oracle Server 版本 12.2，Oracle 已更改了默认的最大标识符长度。在此版本之前，长度为 30，在 12.2 及更高版本中，现在为 128。这一变化影响了 SQLAlchemy 在生成的 SQL 标签名称以及约束名称的区域，特别是在使用描述在 配置约束命名约定 中的约束命名约定特性时。

为了辅助这一变化和其他变化，Oracle 包括了“兼容性”版本的概念，这是一个与实际服务器版本无关的版本号，用于帮助迁移 Oracle 数据库，并可以在 Oracle 服务器内部配置。这个兼容性版本是使用查询 `SELECT value FROM v$parameter WHERE name = 'compatible';` 检索的。当 SQLAlchemy Oracle 方言被要求确定默认的最大标识符长度时，它将在第一次连接时尝试使用此查询，以确定服务器的有效兼容性版本，该版本确定了服务器允许的最大标识符长度。如果表不可用，则使用服务器版本信息。

从 SQLAlchemy 1.4 开始，Oracle 方言的默认最大标识符长度为 128 个字符。在第一次连接时，检测到兼容性版本，如果小于 Oracle 版本 12.2，则将最大标识符长度更改为 30 个字符。在所有情况下，设置 `create_engine.max_identifier_length` 参数将绕过此更改，并且将使用给定的值：

```py
engine = create_engine(
    "oracle+cx_oracle://scott:tiger@oracle122",
    max_identifier_length=30)
```

最大标识符长度在生成 SELECT 语句中的匿名化 SQL 标签时起作用，但更重要的是在根据命名约定生成约束名称时起作用。正是这一领域促使 SQLAlchemy 谨慎地更改了这个默认值。例如，以下命名约定根据标识符长度产生了两个非常不同的约束名称：

```py
from sqlalchemy import Column
from sqlalchemy import Index
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import Table
from sqlalchemy.dialects import oracle
from sqlalchemy.schema import CreateIndex

m = MetaData(naming_convention={"ix": "ix_%(column_0N_name)s"})

t = Table(
    "t",
    m,
    Column("some_column_name_1", Integer),
    Column("some_column_name_2", Integer),
    Column("some_column_name_3", Integer),
)

ix = Index(
    None,
    t.c.some_column_name_1,
    t.c.some_column_name_2,
    t.c.some_column_name_3,
)

oracle_dialect = oracle.dialect(max_identifier_length=30)
print(CreateIndex(ix).compile(dialect=oracle_dialect))
```

使用标识符长度为 30 时，上述 CREATE INDEX 看起来像：

```py
CREATE INDEX ix_some_column_name_1s_70cd ON t
(some_column_name_1, some_column_name_2, some_column_name_3)
```

但是，当长度为 128 时，它变成：

```py
CREATE INDEX ix_some_column_name_1some_column_name_2some_column_name_3 ON t
(some_column_name_1, some_column_name_2, some_column_name_3)
```

在 Oracle 服务器版本 12.2 或更高版本上运行 SQLAlchemy 之前版本的应用程序因此可能受到以下情景的影响：希望对以较短长度生成的名称进行“DROP CONSTRAINT”的数据库迁移。当更改标识符长度而不首先调整索引或约束的名称时，此迁移将失败。因此，强烈建议这些应用程序使用 `create_engine.max_identifier_length` 来控制生成截断名称，并在更改此值时完全审查和测试所有数据库迁移，以确保已减轻此更改的影响。

从版本 1.4 开始：Oracle 的默认 max_identifier_length 是 128 个字符，如果检测到旧版 Oracle 服务器（兼容性版本 < 12.2），则在首次连接时调整为 30。

## LIMIT/OFFSET/FETCH 支持

像 `Select.limit()` 和 `Select.offset()` 这样的方法使用 `FETCH FIRST N ROW / OFFSET N ROWS` 语法，假设 Oracle 12c 或以上版本，并假设 SELECT 语句不嵌套在 UNION 这样的复合语句中。使用 `Select.fetch()` 方法也可以直接使用此语法。

从版本 2.0 开始：Oracle 方言现在对所有 `Select.limit()` 和 `Select.offset()` 的用法，包括 ORM 和旧版 `Query`，都使用 `FETCH FIRST N ROW / OFFSET N ROWS`。要强制使用窗口函数来保留旧版行为，请将 `enable_offset_fetch=False` 方言参数传递给 `create_engine()`。

通过在任何 Oracle 版本上传递 `enable_offset_fetch=False` 给 `create_engine()`，可以禁用 `FETCH FIRST / OFFSET` 的使用，这将强制使用使用窗口函数的“传统”模式。在使用 Oracle 12c 之前的版本时，也会自动选择此模式。

在使用传统模式或将带有限制/偏移的 `Select` 语句嵌入到复合语句中时，将使用基于窗口函数的 LIMIT / OFFSET 的模拟方法，涉及使用 `ROW_NUMBER` 创建子查询，这种方法容易出现性能问题以及对于复杂语句的 SQL 构建问题。但是，所有 Oracle 版本都支持此方法。请参阅下面的说明。

### 关于 LIMIT / OFFSET 模拟（当无法使用 fetch() 方法时）

如果在 Oracle 12c 之前的版本中使用 `Select.limit()` 和 `Select.offset()`，或使用 ORM 中的 `Query.limit()` 和 `Query.offset()` 方法，以下注意事项适用：

+   SQLAlchemy 目前使用 ROWNUM 来实现 LIMIT/OFFSET；确切的方法取自 [`blogs.oracle.com/oraclemagazine/on-rownum-and-limiting-results`](https://blogs.oracle.com/oraclemagazine/on-rownum-and-limiting-results)。

+   默认情况下不使用“FIRST_ROWS()”优化关键字。要启用此优化指令的使用，请在 `create_engine()` 中指定 `optimize_limits=True`。

    1.4 版更改：Oracle 方言使用“编译后”方案呈现限制/偏移整数值，直接在传递语句给游标执行之前呈现整数。`use_binds_for_limits` 标志不再起作用。

    另请参阅

    Oracle 中用于 LIMIT/OFFSET 的新“编译后”绑定参数，以及 SQL Server。

## RETURNING 支持

Oracle 数据库完全支持对使用单个绑定参数集合调用的 INSERT、UPDATE 和 DELETE 语句进行 RETURNING（即`cursor.execute()`风格语句；SQLAlchemy 通常不支持在 executemany 语句中使用 RETURNING）。也可以返回多行。

2.0 版更改：Oracle 后端具有与其他后端相同的 RETURNING 完全支持。

## ON UPDATE CASCADE

Oracle 没有本地的 ON UPDATE CASCADE 功能。可以在 [`asktom.oracle.com/tkyte/update_cascade/index.html`](https://asktom.oracle.com/tkyte/update_cascade/index.html) 上找到基于触发器的解决方案。

在使用 SQLAlchemy ORM 时，ORM 有限的能力可以手动发出级联更新 - 使用“deferrable=True, initially=’deferred’”关键字参数指定 ForeignKey 对象，并在每个 relationship() 上指定 “passive_updates=False”。

## Oracle 8 兼容性

警告

SQLAlchemy 2.0 的 Oracle 8 兼容性状态未知。

当检测到 Oracle 8 时，方言会自动配置为以下行为：

+   使用 `use_ansi` 标志设置为 False。这会将所有 JOIN 词组转换为 WHERE 子句，并且在左外连接的情况下使用 Oracle 的 (+) 运算符。

+   当使用 `Unicode` 时，NVARCHAR2 和 NCLOB 数据类型不再生成 DDL - 而是生成 VARCHAR2 和 CLOB。这是因为即使这些类型可用，它们在 Oracle 8 上似乎无法正常工作。`NVARCHAR` 和 `NCLOB` 类型将始终生成 NVARCHAR2 和 NCLOB。

## 同义词/DBLINK 反射

在使用反射与表对象时，方言可以选择性地搜索由同义词指示的表，可以是在本地或远程模式或通过 DBLINK 访问，通过将标志 `oracle_resolve_synonyms=True` 作为关键字参数传递给 `Table` 构造函数：

```py
some_table = Table('some_table', autoload_with=some_engine,
                            oracle_resolve_synonyms=True)
```

当设置此标志时，给定的名称（例如上面的 `some_table`）将不仅在 `ALL_TABLES` 视图中搜索，还将在 `ALL_SYNONYMS` 视图中搜索，以查看此名称是否实际上是另一个名称的同义词。如果找到了同义词并且引用了 DBLINK，则 Oracle 方言知道如何使用 DBLINK 语法定位表的信息（例如 `@dblink`）。

`oracle_resolve_synonyms` 在接受反射参数的任何地方都被接受，包括诸如 `MetaData.reflect()` 和 `Inspector.get_columns()` 之类的方法。

如果不使用同义词，则应禁用此标志。

## 约束反射

Oracle 方言可以返回有关表上的外键、唯一约束和 CHECK 约束以及索引的信息。

可以使用`Inspector.get_foreign_keys()`、`Inspector.get_unique_constraints()`、`Inspector.get_check_constraints()` 和`Inspector.get_indexes()`获取关于这些约束的原始信息。

自版本 1.2 更改：Oracle 方言现在可以反映唯一约束和检查约束。

在`Table`级别使用反射时，`Table`也将包括这些约束。

请注意以下注意事项：

+   当使用`Inspector.get_check_constraints()`方法时，Oracle 为指定“NOT NULL”的列构建一个特殊的“IS NOT NULL”约束。 默认情况下，此约束**不**会被返回；要包括“IS NOT NULL”约束，传递标志`include_all=True`：

    ```py
    from sqlalchemy import create_engine, inspect

    engine = create_engine("oracle+cx_oracle://s:t@dsn")
    inspector = inspect(engine)
    all_check_constraints = inspector.get_check_constraints(
        "some_table", include_all=True)
    ```

+   在大多数情况下，当反映一个`Table`时，唯一约束将**不**作为`UniqueConstraint`对象可用，因为在大多数情况下，Oracle 使用唯一索引来镜像唯一约束（例外情况似乎是当两个或更多个唯一约束表示相同的列时）；`Table`将使用设置了`unique=True`标志的`Index`来代替这些。

+   Oracle 为表的主键创建一个隐式索引；此索引被**排除**在所有索引结果之外。

+   对于索引反映的列列表，不会包括以 SYS_NC 开头的列名。

## 具有 SYSTEM/SYSAUX 表空间的表名称

`Inspector.get_table_names()` 和 `Inspector.get_temp_table_names()` 方法分别返回当前引擎的表名列表。这些方法也是在操作中进行反射的一部分，例如`MetaData.reflect()`。默认情况下，这些操作将从操作中排除`SYSTEM`和`SYSAUX`表空间。为了更改此设置，可以在引擎级别使用`exclude_tablespaces`参数更改默认的排除表空间列表：

```py
# exclude SYSAUX and SOME_TABLESPACE, but not SYSTEM
e = create_engine(
  "oracle+cx_oracle://scott:tiger@xe",
  exclude_tablespaces=["SYSAUX", "SOME_TABLESPACE"])
```

## 日期时间兼容性

Oracle 没有名为`DATETIME`的数据类型，它只有`DATE`，实际上可以存储日期和时间值。因此，Oracle 方言提供了一个`DATE`类型，它是`DateTime`的子类。此类型没有特殊行为，仅作为此类型的“标记”存在；此外，当反映数据库列并且类型报告为`DATE`时，将使用支持时间的`DATE`类型。

## Oracle 表选项

在与`Table`结构一起使用 Oracle 时，CREATE TABLE 语句支持以下选项：

+   `ON COMMIT`：

    ```py
    Table(
        "some_table", metadata, ...,
        prefixes=['GLOBAL TEMPORARY'], oracle_on_commit='PRESERVE ROWS')
    ```

+   `COMPRESS`：

    ```py
     Table('mytable', metadata, Column('data', String(32)),
         oracle_compress=True)

     Table('mytable', metadata, Column('data', String(32)),
         oracle_compress=6)

    The ``oracle_compress`` parameter accepts either an integer compression
    level, or ``True`` to use the default compression level.
    ```  ## Oracle 特定索引选项

### 位图索引

你可以指定`oracle_bitmap`参数来创建位图索引，而不是 B 树索引：

```py
Index('my_index', my_table.c.data, oracle_bitmap=True)
```

位图索引不能是唯一的，也不能被压缩。SQLAlchemy 不会检查这些限制，只有数据库会。

### 索引压缩

Oracle 对包含大量重复值的索引有更有效的存储模式。使用`oracle_compress`参数来打开键压缩：

```py
Index('my_index', my_table.c.data, oracle_compress=True)

Index('my_index', my_table.c.data1, my_table.c.data2, unique=True,
       oracle_compress=1)
```

`oracle_compress`参数可以接受一个整数，指定要压缩的前缀列数，或者`True`以使用默认值（非唯一索引的所有列，唯一索引除最后一列外的所有列）。

## Oracle 数据类型

与所有 SQLAlchemy 方言一样，所有已知与 Oracle 有效的大写类型都可以从顶层方言导入，无论它们来自`sqlalchemy.types`还是来自本地方言：

```py
from sqlalchemy.dialects.oracle import (
    BFILE,
    BLOB,
    CHAR,
    CLOB,
    DATE,
    DOUBLE_PRECISION,
    FLOAT,
    INTERVAL,
    LONG,
    NCLOB,
    NCHAR,
    NUMBER,
    NVARCHAR,
    NVARCHAR2,
    RAW,
    TIMESTAMP,
    VARCHAR,
    VARCHAR2,
)
```

1.2.19 版中新增：将`NCHAR`添加到由 Oracle 方言导出的数据类型列表中。

特定于 Oracle 的类型，或具有特定于 Oracle 的构造参数的类型如下：

| 对象名称 | 描述 |
| --- | --- |
| BFILE |  |
| BINARY_DOUBLE |  |
| BINARY_FLOAT |  |
| DATE | 提供 oracle DATE 类型。 |
| FLOAT | Oracle FLOAT。 |
| INTERVAL |  |
| LONG |  |
| NCLOB |  |
| NUMBER |  |
| NVARCHAR2 | `NVARCHAR` 的别名 |
| RAW |  |
| ROWID | Oracle ROWID 类型。 |
| TIMESTAMP | Oracle 实现的`TIMESTAMP`，支持额外的 Oracle 特定模式 |

```py
class sqlalchemy.dialects.oracle.BFILE
```

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.oracle.BFILE` (`sqlalchemy.types.LargeBinary`)

```py
method __init__(length: int | None = None)
```

*从* `LargeBinary` 的 `sqlalchemy.types.LargeBinary.__init__` *方法继承*

构造一个 LargeBinary 类型。

参数:

**length** – 可选，用于 DDL 语句中的列长度，适用于接受长度的二进制类型，例如 MySQL 的 BLOB 类型。

```py
class sqlalchemy.dialects.oracle.BINARY_DOUBLE
```

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.oracle.BINARY_DOUBLE` (`sqlalchemy.types.Double`)

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*从* `Float` 的 `sqlalchemy.types.Float.__init__` *方法继承*

构造一个 Float。

参数:

+   `precision` –

    用于 DDL `CREATE TABLE` 中的数值精度。后端**应该**尝试确保此精度指示通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受 `Float.precision` 参数，因为 Oracle 不支持将浮点精度指定为小数位数。相反，请使用 Oracle 特定的 `FLOAT` 数据类型，并指定 `FLOAT.binary_precision` 参数。这是 SQLAlchemy 版本 2.0 中的新功能。

    要创建一个与数据库无关的 `Float`，并为 Oracle 单独指定二进制精度，请使用 `TypeEngine.with_variant()` 如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与 `Numeric` 相同的标志，但默认为 `False`。请注意，将此标志设置为 `True` 会导致浮点数转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 十进制数时使用的默认精度。由于十进制不准确性，浮点值通常会更长，而大多数浮点数据库类型没有“精度”概念，因此默认情况下，浮点类型在转换时会查找前十位小数点。指定此值将覆盖该长度。请注意，MySQL 浮点类型包括“精度”，如果未另行指定，则将使用“精度”作为 decimal_return_scale 的默认值。

```py
class sqlalchemy.dialects.oracle.BINARY_FLOAT
```

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.oracle.BINARY_FLOAT` (`sqlalchemy.types.Float`)

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` *的* `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` –

    用于在 DDL `CREATE TABLE` 中使用的数字精度。后端**应该**尽量确保此精度表示通用 `Float` 数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时，不接受`Float.precision`参数，因为 Oracle 不支持将浮点精度指定为小数位数。相反，使用特定于 Oracle 的`FLOAT`数据类型，并指定`FLOAT.binary_precision`参数。这是 SQLAlchemy 版本 2.0 中的新功能。

    要创建一个与数据库无关的`Float`，分别为 Oracle 指定二进制精度，可以使用`TypeEngine.with_variant()`，如下所示：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与`Numeric`相同的标志，但默认值为`False`。请注意，将此标志设置为`True`会导致浮点转换。

+   `decimal_return_scale` – 在将浮点数转换为 Python 十进制数时使用的默认精度。由于十进制的不准确性，浮点值通常会更长，并且大多数浮点数据库类型都没有“精度”的概念，因此，默认情况下，浮点类型在转换时会寻找前十位小数点。指定此值将覆盖该长度。请注意，如果未另行指定，包括“精度”的 MySQL 浮点类型将使用“精度”作为 decimal_return_scale 的默认值。

```py
class sqlalchemy.dialects.oracle.DATE
```

提供 Oracle DATE 类型。

此类型没有特殊的 Python 行为，除了它是`DateTime`的子类；这是为了适应 Oracle `DATE` 类型支持时间值的事实。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.oracle.DATE`（`sqlalchemy.dialects.oracle.types._OracleDateLiteralRender`，`sqlalchemy.types.DateTime`）

```py
method __init__(timezone: bool = False)
```

*继承自* `DateTime`*的* `sqlalchemy.types.DateTime.__init__` *方法*

构造一个新的`DateTime`。

参数：

**时区** – 布尔值。表示日期时间类型是否应在**仅基本日期/时间保存类型**上启用时区支持（如果可用）。建议在使用此标志时直接使用`TIMESTAMP`数据类型，因为一些数据库包括与时区可用的 TIMESTAMP 数据类型不同的独立通用日期/时间保存类型，例如 Oracle。

```py
class sqlalchemy.dialects.oracle.FLOAT
```

Oracle FLOAT。

这与`FLOAT`相同，不同之处在于接受特定于 Oracle 的`FLOAT.binary_precision`参数，并且不接受`Float.precision`参数。

Oracle FLOAT 类型以“二进制精度”表示精度，默认为 126。对于 REAL 类型，该值为 63。此参数不清晰地映射到特定数量的小数位数，但大致相当于所需小数位数除以 0.3103。

2.0 版中的新功能。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.oracle.FLOAT` (`sqlalchemy.types.FLOAT`)

```py
method __init__(binary_precision=None, asdecimal=False, decimal_return_scale=None)
```

构造一个 FLOAT

参数:

+   `binary_precision` – 要在 DDL 中呈现的 Oracle 二进制精度值。这可以使用公式“十进制精度= 0.30103 * 二进制精度”来近似到十进制字符的数量。Oracle 用于 FLOAT / DOUBLE PRECISION 的默认值为 126。

+   `asdecimal` – 参见`Float.asdecimal`

+   `decimal_return_scale` – 参见`Float.decimal_return_scale`

```py
class sqlalchemy.dialects.oracle.INTERVAL
```

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.oracle.INTERVAL` (`sqlalchemy.types.NativeForEmulated`, `sqlalchemy.types._AbstractInterval`)

```py
method __init__(day_precision=None, second_precision=None)
```

构造一个 INTERVAL。

请注意，当前仅支持 DAY TO SECOND 间隔。这是由于可用 DBAPI 中缺少对 YEAR TO MONTH 间隔的支持。

参数:

+   `day_precision` – 日期精度值。这是要存储的日字段的位数。默认为“2”。

+   `second_precision` – 秒精度值。这是要存储的分数秒字段的位数。默认为“6”。

```py
class sqlalchemy.dialects.oracle.NCLOB
```

**成员**

__init__()

**类签名**

`sqlalchemy.dialects.oracle.NCLOB`类（`sqlalchemy.types.Text`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数:

+   `length` – 可选的，在 DDL 和 CAST 表达式中使用的列的长度。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含了没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时将引发异常。该值被解释为字节还是字符取决于数据库。

+   `collation` –

    可选的，在 DDL 和 CAST 表达式中使用的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应使用`Unicode`或`UnicodeText`数据类型来存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
attribute sqlalchemy.dialects.oracle.NVARCHAR2
```

`NVARCHAR`的别名

```py
class sqlalchemy.dialects.oracle.NUMBER
```

**类签名**

`sqlalchemy.dialects.oracle.NUMBER`类（`sqlalchemy.types.Numeric`，`sqlalchemy.types.Integer`）

```py
class sqlalchemy.dialects.oracle.LONG
```

**成员**

__init__()

**类签名**

`sqlalchemy.dialects.oracle.LONG`类（`sqlalchemy.types.Text`）

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` *的* `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数:

+   `length` – 可选的，在 DDL 和 CAST 表达式中使用的列的长度。如果不会发出`CREATE TABLE`，可以安全地省略。某些数据库可能需要在 DDL 中使用长度，并且如果包含了没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时将引发异常。该值被解释为字节还是字符取决于数据库。

+   `collation` –

    可选的，在 DDL 和 CAST 表达式中使用的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用`Unicode`或`UnicodeText`数据类型来表示预期存储非 ASCII 数据的`Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.oracle.RAW
```

**类签名**

类`sqlalchemy.dialects.oracle.RAW` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.oracle.ROWID
```

Oracle ROWID 类型。

在 cast() 或类似情况下使用时，生成 ROWID。

**类签名**

类`sqlalchemy.dialects.oracle.ROWID` (`sqlalchemy.types.TypeEngine`)

```py
class sqlalchemy.dialects.oracle.TIMESTAMP
```

Oracle 实现的 `TIMESTAMP`，支持额外的 Oracle 特定模式

2.0 版本中的新功能。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.oracle.TIMESTAMP` (`sqlalchemy.types.TIMESTAMP`)

```py
method __init__(timezone: bool = False, local_timezone: bool = False)
```

构造一个新的`TIMESTAMP`。

参数：

+   `timezone` – 布尔值。表示 TIMESTAMP 类型应该使用 Oracle 的 `TIMESTAMP WITH TIME ZONE` 数据类型。

+   `local_timezone` – 布尔值。表示 TIMESTAMP 类型应该使用 Oracle 的 `TIMESTAMP WITH LOCAL TIME ZONE` 数据类型。

## cx_Oracle

通过 cx-Oracle 驱动程序支持 Oracle 数据库。

### DBAPI

cx-Oracle 的文档和下载信息（如果适用）可在此处获得：[`oracle.github.io/python-cx_Oracle/`](https://oracle.github.io/python-cx_Oracle/)

### 连接

连接字符串：

```py
oracle+cx_oracle://user:pass@hostname:port[/dbname][?service_name=<service>[&key=value&key=value...]]
```

### DSN vs. 主机名连接

cx_Oracle 提供了几种指示目标数据库的方法。方言从一系列不同的 URL 形式转换而来。

#### 使用 Easy Connect 语法的主机名连接

给定目标 Oracle 数据库的主机名、端口和服务名称，例如来自 Oracle 的 [Easy Connect 语法](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#easy-connect-syntax-for-connection-strings)，然后在 SQLAlchemy 中使用 `service_name` 查询字符串参数进行连接：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:port/?service_name=myservice&encoding=UTF-8&nencoding=UTF-8")
```

不支持[完整的 Easy Connect 语法](https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-B0437826-43C1-49EC-A94D-B650B6A4A6EE)。而是使用 `tnsnames.ora` 文件，并使用 DSN 进行连接。

#### 带有 tnsnames.ora 或 Oracle Cloud 的连接

或者，如果没有提供端口、数据库名称或 `service_name`，方言将使用 Oracle DSN “连接字符串”。这将 URL 的“主机名”部分视为数据源名称。例如，如果 `tnsnames.ora` 文件包含如下的[网络服务名称](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#net-service-names-for-connection-strings) `myalias`：

```py
myalias =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = mymachine.example.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orclpdb1)
    )
  )
```

当 `myalias` 是 URL 的主机名部分，而没有指定端口、数据库名称或 `service_name` 时，cx_Oracle 方言将连接到此数据库服务：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@myalias/?encoding=UTF-8&nencoding=UTF-8")
```

Oracle Cloud 的用户应该使用这种语法，并按照 cx_Oracle 文档 [连接到 Autonomous 数据库](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#connecting-to-autononmous-databases) 中所示配置云钱包。

#### SID 连接

要使用 Oracle 的过时 SID 连接语法，SID 可以像下面这样通过 URL 的“数据库名称”部分传递：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:1521/dbname?encoding=UTF-8&nencoding=UTF-8")
```

上面，传递给 cx_Oracle 的 DSN 是由 `cx_Oracle.makedsn()` 创建的，如下所示：

```py
>>> import cx_Oracle
>>> cx_Oracle.makedsn("hostname", 1521, sid="dbname")
'(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=hostname)(PORT=1521))(CONNECT_DATA=(SID=dbname)))'
```

### 传递 cx_Oracle 连接参数

通常可以通过 URL 查询字符串传递其他连接参数；特定符号如 `cx_Oracle.SYSDBA` 将被拦截并转换为正确的符号：

```py
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn?encoding=UTF-8&nencoding=UTF-8&mode=SYSDBA&events=true")
```

在版本 1.3 中更改：cx_oracle 方言现在接受 URL 字符串本身中的所有参数名称，以传递给 cx_Oracle DBAPI。与之前的情况一样，但没有正确记录，`create_engine.connect_args` 参数也接受所有 cx_Oracle DBAPI 连接参数。

要直接传递参数到 `.connect()` 而不使用查询字符串，可以使用 `create_engine.connect_args` 字典。可以传递任何 cx_Oracle 参数值和/或常量，例如：

```py
import cx_Oracle
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn",
    connect_args={
        "encoding": "UTF-8",
        "nencoding": "UTF-8",
        "mode": cx_Oracle.SYSDBA,
        "events": True
    }
)
```

请注意，在 cx_Oracle 8.0 中，`encoding` 和 `nencoding` 的默认值已更改为 “UTF-8”，因此在使用该版本或更高版本时可以省略这些参数。

### 在驱动程序之外由 SQLAlchemy cx_Oracle 方言使用的选项

还有一些选项是由 SQLAlchemy cx_oracle 方言自身使用的。这些选项始终直接传递给 `create_engine()` ，例如：

```py
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn", coerce_to_decimal=False)
```

cx_oracle 方言接受的参数如下：

+   `arraysize` - 在游标上设置 cx_oracle.arraysize 的值；默认为 `None`，表示应该使用驱动程序的默认值（通常值为 100）。此设置控制在获取行时缓冲多少行，并且在修改时可以对性能产生重大影响。该设置用于 `cx_Oracle` 以及 `oracledb`。

    在版本 2.0.26 中更改：- 将默认值从 50 更改为 None，以使用驱动程序本身的默认值。

+   `auto_convert_lobs` - 默认为 True；详见 LOB 数据类型。

+   `coerce_to_decimal` - 详见精度数值。

+   `encoding_errors` - 详见编码错误。

### 使用 cx_Oracle 会话池

cx_Oracle 库提供了自己的连接池实现，可以替代 SQLAlchemy 的池功能。可以通过使用`create_engine.creator`参数提供一个返回新连接的函数，并将`create_engine.pool_class`设置为`NullPool`来禁用 SQLAlchemy 的池化功能来实现：

```py
import cx_Oracle
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

pool = cx_Oracle.SessionPool(
    user="scott", password="tiger", dsn="orclpdb",
    min=2, max=5, increment=1, threaded=True,
    encoding="UTF-8", nencoding="UTF-8"
)

engine = create_engine("oracle+cx_oracle://", creator=pool.acquire, poolclass=NullPool)
```

上述引擎可以像平常一样使用，其中 cx_Oracle 的池处理连接池：

```py
with engine.connect() as conn:
    print(conn.scalar("select 1 FROM dual"))
```

除了为多用户应用程序提供可扩展的解决方案之外，cx_Oracle 会话池还支持一些 Oracle 功能，如 DRCP 和[应用程序连续性](https://cx-oracle.readthedocs.io/en/latest/user_guide/ha.html#application-continuity-ac)。

### 使用 Oracle 数据库 Resident 连接池（DRCP）

在使用 Oracle 的[DRCP](https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-015CA8C1-2386-4626-855D-CC546DDC1086)时，最佳实践是在从 SessionPool 获取连接时传递连接类和“purity”。请参阅[cx_Oracle DRCP 文档](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#database-resident-connection-pooling-drcp)。

可以通过包装`pool.acquire()`来实现：

```py
import cx_Oracle
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

pool = cx_Oracle.SessionPool(
    user="scott", password="tiger", dsn="orclpdb",
    min=2, max=5, increment=1, threaded=True,
    encoding="UTF-8", nencoding="UTF-8"
)

def creator():
    return pool.acquire(cclass="MYCLASS", purity=cx_Oracle.ATTR_PURITY_SELF)

engine = create_engine("oracle+cx_oracle://", creator=creator, poolclass=NullPool)
```

上述引擎可以像平常一样使用，其中 cx_Oracle 处理会话池，Oracle 数据库另外使用 DRCP：

```py
with engine.connect() as conn:
    print(conn.scalar("select 1 FROM dual"))
```

### Unicode

对于 Python 3 下的所有 DBAPI，所有字符串都是本质上的 Unicode 字符串。然而，在所有情况下，驱动程序都需要显式的编码配置。

#### 确保正确的客户端编码

几乎所有与 Oracle 相关的软件建立客户端编码的长期接受标准是通过[NLS_LANG](https://www.oracle.com/database/technologies/faq-nls-lang.html)环境变量。cx_Oracle 像大多数其他 Oracle 驱动程序一样将使用此环境变量作为其编码配置的来源。此变量的格式是特殊的；典型的值可能是`AMERICAN_AMERICA.AL32UTF8`。

cx_Oracle 驱动程序还支持一种编程方式，即直接将`encoding`和`nencoding`参数传递给其`.connect()`函数。这些可以在 URL 中如下所示：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@orclpdb/?encoding=UTF-8&nencoding=UTF-8")
```

关于`encoding`和`nencoding`参数的含义，请参考[字符集和国家语言支持（NLS）](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization)。

另见

[字符集和国家语言支持（NLS）](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization) - 在 cx_Oracle 文档中。

#### Unicode 特定的列数据类型

核心表达式语言通过使用`Unicode` 和 `UnicodeText` 数据类型处理 Unicode 数据。这些类型默认对应于 VARCHAR2 和 CLOB Oracle 数据类型。在使用这些数据类型处理 Unicode 数据时，预期 Oracle 数据库配置为具有 Unicode 意识的字符集，并且`NLS_LANG`环境变量设置正确，以便 VARCHAR2 和 CLOB 数据类型可以容纳数据。

如果 Oracle 数据库未配置为 Unicode 字符集，则有两种选择：显式使用`NCHAR`和`NCLOB`数据类型，或者在调用`create_engine()`时传递标志`use_nchar_for_unicode=True`，这将导致 SQLAlchemy 方言使用 NCHAR/NCLOB 代替 VARCHAR/CLOB 用于`Unicode` / `UnicodeText` 数据类型。

从版本 1.3 开始更改：`Unicode` 和 `UnicodeText` 数据类型现在对应于 `VARCHAR2` 和 `CLOB` Oracle 数据类型，除非在调用`create_engine()`时传递了`use_nchar_for_unicode=True`。

#### 编码错误

对于 Oracle 数据库中存在编码错误的情况，方言接受一个`encoding_errors`参数，该参数将传递给 Unicode 解码函数，以影响如何处理解码错误。该值最终由 Python [decode](https://docs.python.org/3/library/stdtypes.html#bytes.decode) 函数消耗，并通过 cx_Oracle 的`encodingErrors`参数（由`Cursor.var()`消耗）以及 SQLAlchemy 自己的解码函数传递，因为 cx_Oracle 方言在不同情况下都使用这两者。

新版本 1.3.11 中引入。### 使用 setinputsizes 对 cx_Oracle 数据绑定性能进行精细控制

cx_Oracle DBAPI 深度且根本上依赖于使用 DBAPI 的 `setinputsizes()` 调用。此调用的目的是为了为作为参数传递的 Python 值绑定到 SQL 语句的数据类型。虽然几乎没有其他 DBAPI 分配任何用途给 `setinputsizes()` 调用，但 cx_Oracle DBAPI 在与 Oracle 客户端接口的交互中大量依赖它，在某些情况下，SQLAlchemy 无法准确知道数据应该如何绑定，因为一些设置可能会导致截然不同的性能特征，同时也会改变类型强制转换行为。

cx_Oracle 方言的用户**强烈建议**阅读 cx_Oracle 的内置数据类型符号列表，网址为 [`cx-oracle.readthedocs.io/en/latest/api_manual/module.html#database-types`](https://cx-oracle.readthedocs.io/en/latest/api_manual/module.html#database-types)。请注意，在某些情况下，使用这些类型与不使用这些类型相比，可能会导致显著的性能下降，特别是在指定 `cx_Oracle.CLOB` 时。

在 SQLAlchemy 方面，`DialectEvents.do_setinputsizes()` 事件可用于运行时可见性（例如日志记录）设置 setinputsizes 步骤以及在每个语句基础上完全控制 `setinputsizes()` 的使用。

版本 1.2.9 中的新功能：添加了 `DialectEvents.setinputsizes()`

#### 示例 1 - 记录所有 setinputsizes 调用

以下示例说明了如何在将其转换为原始 `setinputsizes()` 参数字典之前从 SQLAlchemy 视角记录中间值。字典的键是具有 `.key` 和 `.type` 属性的 `BindParameter` 对象：

```py
from sqlalchemy import create_engine, event

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _log_setinputsizes(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in inputsizes.items():
            log.info(
                "Bound parameter name: %s SQLAlchemy type: %r "
                "DBAPI object: %s",
                bindparam.key, bindparam.type, dbapitype)
```

#### 示例 2 - 删除所有到 CLOB 的绑定

cx_Oracle 中的 `CLOB` 数据类型会产生显著的性能开销，但在 SQLAlchemy 1.2 系列中默认设置为 `Text` 类型。可以通过以下方式修改此设置：

```py
from sqlalchemy import create_engine, event
from cx_Oracle import CLOB

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _remove_clob(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in list(inputsizes.items()):
        if dbapitype is CLOB:
            del inputsizes[bindparam]
```  ### RETURNING 支持

cx_Oracle 方言使用 OUT 参数实现 RETURNING。该方言完全支持 RETURNING。  ### LOB 数据类型

LOB 数据类型是指诸如 CLOB、NCLOB 和 BLOB 之类的“大对象”数据类型。cx_Oracle 和 oracledb 的现代版本经过优化，可以将这些数据类型作为单个缓冲区传递。因此，SQLAlchemy 默认使用这些新型处理程序。

要禁用新型类型处理程序的使用，并将 LOB 对象作为具有 `read()` 方法的经典缓冲对象传递，可以向 `create_engine()` 传递参数 `auto_convert_lobs=False`，这仅在引擎范围内生效。

### 不支持两阶段事务

由于驱动程序支持不佳，cx_Oracle 不支持两阶段事务。从 cx_Oracle 6.0b1 开始，两阶段事务的接口已更改为更直接地通过到底层 OCI 层的传递，自动化程度较低。支持此系统的附加逻辑未在 SQLAlchemy 中实现。

### 精确数字

SQLAlchemy 的数字类型可以处理接收和返回 Python `Decimal` 对象或浮点对象的值。当使用 `Numeric` 对象或其子类如 `Float`，`DOUBLE_PRECISION` 等时，`Numeric.asdecimal` 标志确定返回时值是否应强制转换为 `Decimal`，或返回为浮点对象。在 Oracle 下更加复杂，Oracle 的 `NUMBER` 类型如果“scale”为零，也可以表示整数值，因此 Oracle 特定的 `NUMBER` 类型也考虑到了这一点。

cx_Oracle 方言广泛使用连接和游标级别的“outputtypehandler”可调用对象，以根据请求强制转换数值。这些可调用对象特定于使用的 `Numeric` 的具体类型，以及如果不存在 SQLAlchemy 类型对象。观察到的情况是，Oracle 可能发送关于返回的数字类型的不完整或模糊信息，例如查询中数字类型被埋在多层子查询中。类型处理程序在所有情况下都尽力做出正确决定，对于所有那些驱动程序可以做出最佳决定的情况，都会推迟到底层的 cx_Oracle DBAPI。

当不存在类型对象时，例如执行普通 SQL 字符串时，存在默认的“outputtypehandler”，通常会返回指定精度和标度的数值，作为 Python `Decimal` 对象。为了出于性能原因禁用此强制转换为十进制数的操作，请将标志 `coerce_to_decimal=False` 传递给 `create_engine()`:

```py
engine = create_engine("oracle+cx_oracle://dsn", coerce_to_decimal=False)
```

`coerce_to_decimal` 标志仅影响与 `Numeric` SQLAlchemy 类型（或其子类）无关的普通字符串 SQL 语句的结果。

从版本 1.2 开始更改：cx_Oracle 的数字处理系统已经重新设计，以利用更新的 cx_Oracle 功能以及更好地集成 outputtypehandlers。  ## python-oracledb

通过 python-oracledb 驱动程序支持 Oracle 数据库。

### DBAPI

python-oracledb 的文档和下载信息（如果适用）可在此处获取：[`oracle.github.io/python-oracledb/`](https://oracle.github.io/python-oracledb/)

### 连接

连接字符串：

```py
oracle+oracledb://user:pass@hostname:port[/dbname][?service_name=<service>[&key=value&key=value...]]
```

python-oracledb 是由 Oracle 发布的用于取代 cx_Oracle 驱动程序的驱动程序。它与 cx_Oracle 完全兼容，并且具有“thin”客户端模式（不需要依赖项）和“thick”模式（与 cx_Oracle 一样使用 Oracle 客户端接口）。

另请参阅

cx_Oracle - cx_Oracle 的所有说明也适用于 oracledb 驱动程序。

SQLAlchemy `oracledb` 方言在同一方言名称下提供了同步和异步实现。根据引擎的创建方式选择适当的版本：

+   使用 `oracle+oracledb://...` 调用 `create_engine()` 将自动选择同步版本，例如：

    ```py
    from sqlalchemy import create_engine
    sync_engine = create_engine("oracle+oracledb://scott:tiger@localhost/?service_name=XEPDB1")
    ```

+   使用 `oracle+oracledb://...` 调用 `create_async_engine()` 将自动选择异步版本，例如：

    ```py
    from sqlalchemy.ext.asyncio import create_async_engine
    asyncio_engine = create_async_engine("oracle+oracledb://scott:tiger@localhost/?service_name=XEPDB1")
    ```

也可以明确指定方言的 asyncio 版本，使用 `oracledb_async` 后缀，如：

```py
from sqlalchemy.ext.asyncio import create_async_engine
asyncio_engine = create_async_engine("oracle+oracledb_async://scott:tiger@localhost/?service_name=XEPDB1")
```

2.0.25 版本中的新功能：增加了对 oracledb 的异步版本的支持。

### Thick 模式支持

默认情况下，`python-oracledb` 以 thin 模式启动，不需要在系统中安装 Oracle 客户端库。`python-oracledb` 驱动程序还支持“thick”模式，行为类似于 `cx_oracle`，需要安装 Oracle 客户端接口（OCI）。

要启用此模式，用户可以手动调用 `oracledb.init_oracle_client`，或通过将参数 `thick_mode=True` 传递给 `create_engine()` 来启用。要向 `init_oracle_client` 传递自定义参数，如 `lib_dir` 路径，可以将字典传递给此参数，如下所示：

```py
engine = sa.create_engine("oracle+oracledb://...", thick_mode={
    "lib_dir": "/path/to/oracle/client/lib", "driver_name": "my-app"
})
```

另请参阅

[`python-oracledb.readthedocs.io/en/latest/api_manual/module.html#oracledb.init_oracle_client`](https://python-oracledb.readthedocs.io/en/latest/api_manual/module.html#oracledb.init_oracle_client)

2.0.0 版本中的新功能：增加了对 oracledb 驱动程序的支持。

对 Oracle 数据库的支持。

以下表格总结了当前数据库发布版本的支持级别。

**支持的 Oracle 版本**

| 支持类型 | 版本 |
| --- | --- |
| 在 CI 中进行完整测试 | 18c |
| 普通支持 | 11+ |
| 尽力而为 | 9+ |

## DBAPI 支持

提供以下方言/DBAPI 选项。请参阅各自的 DBAPI 部分获取连接信息。

+   cx-Oracle

+   python-oracledb

## 自动增量行为

包括整数主键的 SQLAlchemy Table 对象通常被假定具有“自动增量”行为，意味着它们可以在插入时生成自己的主键值。在 Oracle 中，有两个可用选项，即使用 IDENTITY 列（仅限 Oracle 12 及以上版本）或将序列与列相关联。

### 指定 GENERATED AS IDENTITY（Oracle 12 及以上）

从版本 12 开始，Oracle 可以使用 `Identity` 指定自动增量行为：

```py
t = Table('mytable', metadata,
    Column('id', Integer, Identity(start=3), primary_key=True),
    Column(...), ...
)
```

上述 `Table` 对象的 CREATE TABLE 如下：

```py
CREATE  TABLE  mytable  (
  id  INTEGER  GENERATED  BY  DEFAULT  AS  IDENTITY  (START  WITH  3),
  ...,
  PRIMARY  KEY  (id)
)
```

`Identity` 对象支持许多选项来控制列的“自动增量”行为，如起始值、增量值等。除了标准选项外，Oracle 还支持将 `Identity.always` 设置为 `None`，以使用默认生成模式，在 DDL 中呈现 GENERATED AS IDENTITY。它还支持将 `Identity.on_null` 设置为 `True`，以指定在“BY DEFAULT”身份列上与“ON NULL”一起使用。

### 使用 SEQUENCE（所有 Oracle 版本）

早期版本的 Oracle 没有“autoincrement”功能，SQLAlchemy 依赖序列来生成这些值。对于旧版的 Oracle，*必须始终明确指定序列以启用自动增量*。这与大多数文档示例不同，后者假设使用的是具有自动增量功能的数据库。要指定序列，请使用传递给列构造函数的 sqlalchemy.schema.Sequence 对象：

```py
t = Table('mytable', metadata,
      Column('id', Integer, Sequence('id_seq', start=1), primary_key=True),
      Column(...), ...
)
```

在使用表反射时，即 autoload_with=engine，也需要执行此步骤：

```py
t = Table('mytable', metadata,
      Column('id', Integer, Sequence('id_seq', start=1), primary_key=True),
      autoload_with=engine
)
```

版本 1.4 中的更改：在 `Column` 中添加了 `Identity` 构造函数，用于指定自动增量列的选项。

### 指定 GENERATED AS IDENTITY（Oracle 12 及以上）

从版本 12 开始，Oracle 可以使用 `Identity` 指定自动增量行为：

```py
t = Table('mytable', metadata,
    Column('id', Integer, Identity(start=3), primary_key=True),
    Column(...), ...
)
```

上述 `Table` 对象的 CREATE TABLE 如下：

```py
CREATE  TABLE  mytable  (
  id  INTEGER  GENERATED  BY  DEFAULT  AS  IDENTITY  (START  WITH  3),
  ...,
  PRIMARY  KEY  (id)
)
```

`Identity` 对象支持许多选项来控制列的“自动增量”行为，例如起始值、增量值等。除了标准选项外，Oracle 还支持将 `Identity.always` 设置为 `None`，以使用默认生成模式，将 GENERATED AS IDENTITY 渲染到 DDL 中。它还支持将 `Identity.on_null` 设置为 `True`，以指定与 “BY DEFAULT” 身份列一起使用 ON NULL。

### 使用 SEQUENCE（所有 Oracle 版本）

旧版本的 Oracle 没有“自动增量”功能，SQLAlchemy 依赖序列来生成这些值。在旧的 Oracle 版本中，*必须始终明确指定序列以启用自动增量*。这与大多数文档示例不同，后者假定使用支持自动增量的数据库。要指定序列，请使用传递给 Column 结构的 sqlalchemy.schema.Sequence 对象：

```py
t = Table('mytable', metadata,
      Column('id', Integer, Sequence('id_seq', start=1), primary_key=True),
      Column(...), ...
)
```

当使用表反射时也需要执行此步骤，即 autoload_with=engine：

```py
t = Table('mytable', metadata,
      Column('id', Integer, Sequence('id_seq', start=1), primary_key=True),
      autoload_with=engine
)
```

从 1.4 版本开始更改：在 `Column` 中添加 `Identity` 结构以指定自动增量列的选项。

## 事务隔离级别 / 自动提交

Oracle 数据库支持“READ COMMITTED”和“SERIALIZABLE”隔离模式。 cx_Oracle 方言也支持 AUTOCOMMIT 隔离级别。

设置使用每个连接的执行选项：

```py
connection = engine.connect()
connection = connection.execution_options(
    isolation_level="AUTOCOMMIT"
)
```

对于 `READ COMMITTED` 和 `SERIALIZABLE`，Oracle 方言使用 `ALTER SESSION` 在会话级别设置级别，当连接返回到连接池时，它将恢复为其默认设置。

`isolation_level` 的有效值包括：

+   `READ COMMITTED`

+   `AUTOCOMMIT`

+   `SERIALIZABLE`

注意

由 Oracle 方言实现的 `Connection.get_isolation_level()` 方法的实现必然使用 Oracle LOCAL_TRANSACTION_ID 函数启动事务；否则，通常不可读取级别。

此外，如果由于权限或其他原因导致 `v$transaction` 视图不可用，`Connection.get_isolation_level()` 方法将引发异常，在 Oracle 安装中这是常见的情况。

当 cx_Oracle 方言在其首次连接到数据库时，会尝试调用`Connection.get_isolation_level()`方法，以获取“默认”隔离级别。这个默认级别是必需的，以便在使用`Connection.execution_options()`方法临时修改连接后，可以重置级别。在常见情况下，如果`Connection.get_isolation_level()`方法由于`v$transaction`不可读以及任何其他与数据库相关的故障而引发异常，则假定级别为“READ COMMITTED”。对于这种初始第一次连接条件，不会发出警告，因为预计这是 Oracle 数据库上的常见限制。

自版本 1.3.16 新增：为 cx_oracle 方言添加了对 AUTOCOMMIT 的支持以及默认隔离级别的概念

自版本 1.3.21 新增：增加了对 SERIALIZABLE 的支持以及隔离级别的实时读取。

从版本 1.3.22 起更改：如果由于在 Oracle 安装中常见的 v$transaction 视图上的权限问题而无法读取默认隔离级别，则默认隔离级别硬编码为“READ COMMITTED”，这是 1.3.21 之前的行为。

另请参阅

设置事务隔离级别，包括 DBAPI 自动提交

## 标识符大小写

在 Oracle 中，数据字典使用大写文本表示所有不区分大小写的标识符名称。另一方面，SQLAlchemy 认为所有小写标识符名称都是不区分大小写的。Oracle 方言在模式级别通信期间（例如反射表和索引）将所有不区分大小写的标识符转换为这两种格式。在 SQLAlchemy 一侧使用大写名称表示区分大小写的标识符，并且 SQLAlchemy 会对名称加引号 - 这将导致与从 Oracle 接收到的数据字典数据不匹配，因此除非标识符名称真的已创建为区分大小写的（即使用带引号的名称），否则在 SQLAlchemy 一侧应使用所有小写名称。

## 最大标识符长度

Oracle 在 Oracle Server 版本 12.2 之后更改了默认的最大标识符长度。在此版本之前，长度为 30，在 12.2 及更高版本中，现在为 128。此更改影响 SQLAlchemy 在生成的 SQL 标签名称以及生成约束名称方面的操作，特别是在使用配置约束命名约定中描述的约束命名约定功能的情况下。

为了帮助进行此更改和其他更改，Oracle 包括“兼容性”版本的概念，这是一个与实际服务器版本无关的版本号，以帮助迁移 Oracle 数据库，并且可以在 Oracle 服务器内部配置。此兼容性版本通过查询`SELECT value FROM v$parameter WHERE name = 'compatible';`检索。当 SQLAlchemy Oracle 方言被要求确定默认最大标识符长度时，将尝试在首次连接时使用此查询以确定服务器的有效兼容性版本，该版本确定服务器的最大允许标识符长度。如果表不可用，则使用服务器版本信息。

从 SQLAlchemy 1.4 开始，Oracle 方言的默认最大标识符长度为 128 个字符。首次连接时，检测兼容性版本，如果低于 Oracle 版本 12.2，则将最大标识符长度更改为 30 个字符。在所有情况下，设置`create_engine.max_identifier_length`参数将绕过此更改，并且给定的值将如实使用：

```py
engine = create_engine(
    "oracle+cx_oracle://scott:tiger@oracle122",
    max_identifier_length=30)
```

最大标识符长度在生成 SELECT 语句中的匿名化 SQL 标签时起作用，但更重要的是在根据命名约定生成约束名称时起作用。正是这个领域促使 SQLAlchemy 保守地更改此默认值。例如，以下命名约定基于标识符长度产生两个非常不同的约束名称：

```py
from sqlalchemy import Column
from sqlalchemy import Index
from sqlalchemy import Integer
from sqlalchemy import MetaData
from sqlalchemy import Table
from sqlalchemy.dialects import oracle
from sqlalchemy.schema import CreateIndex

m = MetaData(naming_convention={"ix": "ix_%(column_0N_name)s"})

t = Table(
    "t",
    m,
    Column("some_column_name_1", Integer),
    Column("some_column_name_2", Integer),
    Column("some_column_name_3", Integer),
)

ix = Index(
    None,
    t.c.some_column_name_1,
    t.c.some_column_name_2,
    t.c.some_column_name_3,
)

oracle_dialect = oracle.dialect(max_identifier_length=30)
print(CreateIndex(ix).compile(dialect=oracle_dialect))
```

使用 30 个标识符长度，上述 CREATE INDEX 如下所示：

```py
CREATE INDEX ix_some_column_name_1s_70cd ON t
(some_column_name_1, some_column_name_2, some_column_name_3)
```

然而，长度为 128 时，变为：

```py
CREATE INDEX ix_some_column_name_1some_column_name_2some_column_name_3 ON t
(some_column_name_1, some_column_name_2, some_column_name_3)
```

在 Oracle 服务器版本 12.2 或更高版本上运行 SQLAlchemy 1.4 之前的版本的应用程序因此受到数据库迁移的影响，希望在较短长度生成的名称上“DROP CONSTRAINT”。当更改标识符长度而未先调整索引或约束的名称时，此迁移将失败。强烈建议这些应用程序使用`create_engine.max_identifier_length`以控制生成截断名称，并在更改此值时在分段环境中全面审查和测试所有数据库迁移，以确保已减轻此更改的影响。

自版本 1.4 起更改：Oracle 的默认 max_identifier_length 为 128 个字符，如果检测到旧版本的 Oracle 服务器（兼容性版本<12.2），则在首次连接时调整为 30 个字符。

## LIMIT/OFFSET/FETCH 支持

像 `Select.limit()` 和 `Select.offset()` 这样的方法使用 `FETCH FIRST N ROW / OFFSET N ROWS` 语法，假设是 Oracle 12c 或更高版本，并且假设 SELECT 语句没有嵌入在像 UNION 这样的复合语句中。通过使用 `Select.fetch()` 方法也可以直接使用此语法。

从 2.0 版本开始更改：Oracle 方言现在对所有包括 ORM 和传统 `Query` 内部在内的 `Select.limit()` 和 `Select.offset()` 使用中都使用 `FETCH FIRST N ROW / OFFSET N ROWS`。要强制使用基于窗口函数的传统行为，请在 `create_engine()` 中指定 `enable_offset_fetch=False` 方言参数。

通过向 `create_engine()` 传递 `enable_offset_fetch=False`，可以在任何 Oracle 版本上禁用 `FETCH FIRST / OFFSET` 的使用，这将强制使用“传统”模式，即使用窗口函数。当使用 Oracle 版本 12c 之前的版本时，也会自动选择此模式。

在使用传统模式或者将带有 limit/offset 的 `Select` 语句嵌入到复合语句中时，会使用基于窗口函数的 LIMIT / OFFSET 的模拟方法，这涉及使用 `ROW_NUMBER` 创建子查询，容易出现性能问题以及对复杂语句的 SQL 构造问题。然而，这种方法受到所有 Oracle 版本的支持。请参阅下面的注意事项。

### LIMIT / OFFSET 模拟的注意事项（当无法使用 fetch() 方法时）

如果在 Oracle 版本 12c 之前使用 `Select.limit()` 和 `Select.offset()` 方法，或者在 ORM 中使用 `Query.limit()` 和 `Query.offset()` 方法，则需要注意以下内容：

+   SQLAlchemy 目前使用 ROWNUM 来实现 LIMIT/OFFSET；确切的方法取自[`blogs.oracle.com/oraclemagazine/on-rownum-and-limiting-results`](https://blogs.oracle.com/oraclemagazine/on-rownum-and-limiting-results)。

+   “FIRST_ROWS()”优化关键字默认情况下不使用。要启用此优化指令的使用，请在`create_engine()`中指定`optimize_limits=True`。

    自版本 1.4 起：Oracle 方言使用“编译后”方案呈现限制/偏移整数值，直接在将语句传递给游标执行之前呈现整数。`use_binds_for_limits`标志不再起作用。

    另请参阅

    Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数。

### LIMIT / OFFSET 仿真注意事项（当无法使用 fetch()方法时）

如果在 Oracle 12c 之前的版本中使用`Select.limit()`和`Select.offset()`，或者在 ORM 中使用`Query.limit()`和`Query.offset()`方法，则适用以下注意事项：

+   SQLAlchemy 目前使用 ROWNUM 来实现 LIMIT/OFFSET；确切的方法取自[`blogs.oracle.com/oraclemagazine/on-rownum-and-limiting-results`](https://blogs.oracle.com/oraclemagazine/on-rownum-and-limiting-results)。

+   “FIRST_ROWS()”优化关键字默认情况下不使用。要启用此优化指令的使用，请在`create_engine()`中指定`optimize_limits=True`。

    自版本 1.4 起：Oracle 方言使用“编译后”方案呈现限制/偏移整数值，直接在将语句传递给游标执行之前呈现整数。`use_binds_for_limits`标志不再起作用。

    另请参阅

    Oracle、SQL Server 中用于 LIMIT/OFFSET 的新“编译后”绑定参数。

## RETURNING 支持

Oracle 数据库完全支持对使用单个绑定参数集合（即`cursor.execute()`风格语句；SQLAlchemy 通常不支持 executemany 语句）调用的 INSERT、UPDATE 和 DELETE 语句的 RETURNING。也可以返回多行。

自版本 2.0 起：Oracle 后端完全支持与其他后端相同的 RETURNING 功能。

## ON UPDATE CASCADE

Oracle 没有本机的 ON UPDATE CASCADE 功能。一个基于触发器的解决方案可在 [`asktom.oracle.com/tkyte/update_cascade/index.html`](https://asktom.oracle.com/tkyte/update_cascade/index.html) 找到。

当使用 SQLAlchemy ORM 时，ORM 有限的手动发出级联更新的能力 - 使用“deferrable=True, initially='deferred'”关键字参数指定 ForeignKey 对象，并在每个 relationship() 中指定“passive_updates=False”。

## Oracle 8 兼容性

警告

对于 SQLAlchemy 2.0，Oracle 8 兼容性的状态尚不清楚。

当检测到 Oracle 8 时，方言内部会配置为以下行为：

+   `use_ansi` 标志设置为 False。这会将所有 JOIN 短语转换为 WHERE 子句，并且在 LEFT OUTER JOIN 的情况下使用 Oracle 的 (+) 运算符。

+   当使用 `Unicode` 时，不再生成 NVARCHAR2 和 NCLOB 数据类型的 DDL - 而是生成 VARCHAR2 和 CLOB。这是因为即使这些类型在 Oracle 8 上是可用的，但在 Oracle 8 上似乎无法正确工作。`NVARCHAR` 和 `NCLOB` 类型将始终生成 NVARCHAR2 和 NCLOB。

## 同义词/DBLINK 反射

当使用 Table 对象进行反射时，方言可以选择搜索由同义词指示的表，无论是在本地还是远程模式，还是通过 DBLINK 访问，只需将标志 `oracle_resolve_synonyms=True` 作为关键字参数传递给 `Table` 构造函数：

```py
some_table = Table('some_table', autoload_with=some_engine,
                            oracle_resolve_synonyms=True)
```

当设置了此标志时，将会在 `ALL_TABLES` 视图中搜索给定的名称（例如上面的 `some_table`），而且还会在 `ALL_SYNONYMS` 视图中搜索，以查看该名称是否实际上是另一个名称的同义词。如果找到同义词并且它指向一个 DBLINK，Oracle 方言会使用 DBLINK 语法来定位表的信息（例如 `@dblink`）。

`oracle_resolve_synonyms` 被接受在任何接受反射参数的地方，包括 `MetaData.reflect()` 和 `Inspector.get_columns()` 等方法。

如果不使用同义词，应将此标志保持禁用。

## 约束反射

Oracle 方言可以返回有关表的外键、唯一约束、CHECK 约束以及索引的信息。

可以使用`Inspector.get_foreign_keys()`、`Inspector.get_unique_constraints()`、`Inspector.get_check_constraints()`和`Inspector.get_indexes()`获取关于这些约束的原始信息。

在 1.2 版本中更改：Oracle 方言现在可以反映唯一约束和检查约束。

在`Table`级别使用反射时，`Table`还将包括这些约束条件。

注意以下注意事项：

+   使用`Inspector.get_check_constraints()`方法时，Oracle 为指定“NOT NULL”的列构建一个特殊的“IS NOT NULL”约束条件。默认情况下，此约束条件**不会**被返回；要包括“IS NOT NULL”约束条件，请传递标志`include_all=True`：

    ```py
    from sqlalchemy import create_engine, inspect

    engine = create_engine("oracle+cx_oracle://s:t@dsn")
    inspector = inspect(engine)
    all_check_constraints = inspector.get_check_constraints(
        "some_table", include_all=True)
    ```

+   在大多数情况下，当反映`Table`时，唯一约束将**不可用**作为`UniqueConstraint`对象，因为 Oracle 在大多数情况下使用唯一索引来反映唯一约束（例外情况似乎是当两个或多个唯一约束表示相同列时）；相反，`Table`将使用带有`unique=True`标志的`Index`来表示这些约束。

+   Oracle 为表的主键创建一个隐式索引；此索引**不包含**在所有索引结果中。

+   反映索引的列列表不会包括以 SYS_NC 开头的列名。

## 具有 SYSTEM/SYSAUX 表空间的表名称

`Inspector.get_table_names()`和`Inspector.get_temp_table_names()`方法分别返回当前引擎的表名列表。这些方法也是在操作中发生的反射的一部分，比如`MetaData.reflect()`。默认情况下，这些操作会排除`SYSTEM`和`SYSAUX`表空间。要更改这一点，可以在引擎级别使用`exclude_tablespaces`参数更改默认排除的表空间列表：

```py
# exclude SYSAUX and SOME_TABLESPACE, but not SYSTEM
e = create_engine(
  "oracle+cx_oracle://scott:tiger@xe",
  exclude_tablespaces=["SYSAUX", "SOME_TABLESPACE"])
```

## 日期时间兼容性

Oracle 没有名为`DATETIME`的数据类型，它只有`DATE`，实际上可以存储日期和时间值。因此，Oracle 方言提供了一个类型`DATE`，它是`DateTime`的子类。这种类型没有特殊行为，只是作为这种类型的“标记”存在；此外，当反射数据库列并且类型报告为`DATE`时，将使用支持时间的`DATE`类型。

## Oracle 表选项

CREATE TABLE 短语与 Oracle 一起支持以下选项，与`Table`构造一起使用：

+   `ON COMMIT`：

    ```py
    Table(
        "some_table", metadata, ...,
        prefixes=['GLOBAL TEMPORARY'], oracle_on_commit='PRESERVE ROWS')
    ```

+   `COMPRESS`：

    ```py
     Table('mytable', metadata, Column('data', String(32)),
         oracle_compress=True)

     Table('mytable', metadata, Column('data', String(32)),
         oracle_compress=6)

    The ``oracle_compress`` parameter accepts either an integer compression
    level, or ``True`` to use the default compression level.
    ```

## Oracle 特定索引选项

### 位图索引

您可以指定`oracle_bitmap`参数来创建位图索引，而不是 B 树索引：

```py
Index('my_index', my_table.c.data, oracle_bitmap=True)
```

位图索引不能是唯一的，也不能被压缩。SQLAlchemy 不会检查这些限制，只有数据库会检查。

### 索引压缩

Oracle 有一种更高效的存储模式，适用于包含大量重复值的索引。使用`oracle_compress`参数来启用键压缩：

```py
Index('my_index', my_table.c.data, oracle_compress=True)

Index('my_index', my_table.c.data1, my_table.c.data2, unique=True,
       oracle_compress=1)
```

`oracle_compress`参数接受一个整数，指定要压缩的前缀列数，或者`True`来使用默认值（对于非唯一索引，使用所有列，对于唯一索引，使用除最后一列之外的所有列）。

### 位图索引

您可以指定`oracle_bitmap`参数来创建位图索引，而不是 B 树索引：

```py
Index('my_index', my_table.c.data, oracle_bitmap=True)
```

位图索引不能是唯一的，也不能被压缩。SQLAlchemy 不会检查这些限制，只有数据库会检查。

### 索引压缩

Oracle 有一种更高效的存储模式，适用于包含大量重复值的索引。使用`oracle_compress`参数来启用键压缩：

```py
Index('my_index', my_table.c.data, oracle_compress=True)

Index('my_index', my_table.c.data1, my_table.c.data2, unique=True,
       oracle_compress=1)
```

`oracle_compress` 参数接受一个整数，指定要压缩的前缀列数，或者接受 `True` 来使用默认值（对于非唯一索引是所有列，对于唯一索引是除最后一列外的所有列）。

## Oracle 数据类型

与所有 SQLAlchemy 方言一样，所有已知在 Oracle 中有效的大写类型都可以从顶级方言导入，无论它们是从 `sqlalchemy.types` 还是从本地方言派生的：

```py
from sqlalchemy.dialects.oracle import (
    BFILE,
    BLOB,
    CHAR,
    CLOB,
    DATE,
    DOUBLE_PRECISION,
    FLOAT,
    INTERVAL,
    LONG,
    NCLOB,
    NCHAR,
    NUMBER,
    NVARCHAR,
    NVARCHAR2,
    RAW,
    TIMESTAMP,
    VARCHAR,
    VARCHAR2,
)
```

1.2.19 版新增：将 `NCHAR` 添加到 Oracle 方言导出的数据类型列表中。

以下是特定于 Oracle 或具有 Oracle 特定构造参数的类型：

| 对象名称 | 描述 |
| --- | --- |
| BFILE |  |
| BINARY_DOUBLE |  |
| BINARY_FLOAT |  |
| DATE | 提供 Oracle DATE 类型。 |
| FLOAT | Oracle FLOAT 类型。 |
| INTERVAL |  |
| LONG |  |
| NCLOB |  |
| NUMBER |  |
| NVARCHAR2 | `NVARCHAR` 的别名 |
| RAW |  |
| ROWID | Oracle ROWID 类型。 |
| TIMESTAMP | `TIMESTAMP` 的 Oracle 实现，支持额外的 Oracle 特定模式。 |

```py
class sqlalchemy.dialects.oracle.BFILE
```

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.oracle.BFILE` (`sqlalchemy.types.LargeBinary`)

```py
method __init__(length: int | None = None)
```

*继承自* `LargeBinary` *的* `sqlalchemy.types.LargeBinary.__init__` *方法*

构造一个 LargeBinary 类型。

参数：

**length** – 可选，用于 DDL 语句中的列长度，适用于接受长度的二进制类型，如 MySQL 的 BLOB 类型。

```py
class sqlalchemy.dialects.oracle.BINARY_DOUBLE
```

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.oracle.BINARY_DOUBLE` (`sqlalchemy.types.Double`)

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*继承自* `Float` *的* `sqlalchemy.types.Float.__init__` *方法*

构造一个 Float。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE`中的数值精度。后端**应该**尽量确保此精度指示了通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时不接受`Float.precision`参数，因为 Oracle 不支持将浮点精度指定为小数位数。而是使用 Oracle 特定的`FLOAT`数据类型，并指定`FLOAT.binary_precision`参数。这是 SQLAlchemy 的 2.0 版本中的新功能。

    要创建一个数据库通用的`Float`，并分别为 Oracle 指定二进制精度，请使用`TypeEngine.with_variant()`如下：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与`Numeric`相同的标志，但默认值为`False`。请注意，将此标志设置为`True`会导致浮点转换。

+   `decimal_return_scale` – 将浮点数转换为 Python 十进制时使用的默认标度。由于十进制不准确，浮点值通常会更长，并且大多数浮点数据库类型都没有“标度”的概念，因此默认情况下，浮点类型在转换时会查找前十位小数点。指定此值将覆盖该长度。请注意，MySQL 浮点类型（包括“标度”）将使用“标度”作为 decimal_return_scale 的默认值，如果未另行指定。

```py
class sqlalchemy.dialects.oracle.BINARY_FLOAT
```

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.oracle.BINARY_FLOAT`（`sqlalchemy.types.Float`）

```py
method __init__(precision: int | None = None, asdecimal: bool = False, decimal_return_scale: int | None = None)
```

*从* `Float` *的* `sqlalchemy.types.Float.__init__` *方法继承而来*

构造一个 Float。

参数：

+   `precision` –

    用于 DDL `CREATE TABLE`中的数值精度。后端**应该**尽量确保此精度指示了通用`Float`数据类型的数字位数。

    注意

    对于 Oracle 后端，在渲染 DDL 时不接受 `Float.precision` *参数，因为 Oracle 不支持指定浮点精度为小数位数。* *相反，使用特定于 Oracle 的 `FLOAT` 数据类型，并指定 `FLOAT.binary_precision` *参数。* *这是 SQLAlchemy 的 2.0 新功能。*

    要创建一个数据库通用的 `Float`，为 Oracle 分别指定二进制精度，请使用 `TypeEngine.with_variant()` *如下所示*：

    ```py
    from sqlalchemy import Column
    from sqlalchemy import Float
    from sqlalchemy.dialects import oracle

    Column(
        "float_data",
        Float(5).with_variant(oracle.FLOAT(binary_precision=16), "oracle")
    )
    ```

+   `asdecimal` – 与 `Numeric` *相同的标志，但默认为* `False`。请注意，将此标志设置为 `True` 将导致浮点数转换。

+   `decimal_return_scale` – 在从浮点数转换为 Python 十进制数时要使用的默认比例。由于十进制不精确，浮点数值通常会更长，并且大多数浮点数数据库类型没有“比例”的概念，因此默认情况下，浮点类型在转换时会寻找前十位小数位数。指定此值将覆盖该长度。请注意，如果没有另行指定，具有“比例”的 MySQL 浮点类型将使用“比例”作为 `decimal_return_scale` 的默认值。

```py
class sqlalchemy.dialects.oracle.DATE
```

提供 Oracle DATE 类型。

此类型在 Python 中没有特殊的行为，除了它是 `DateTime` *的子类*；*这是为了适应 Oracle `DATE` 类型支持时间值的事实。*

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.oracle.DATE` (`sqlalchemy.dialects.oracle.types._OracleDateLiteralRender`, `sqlalchemy.types.DateTime`)

```py
method __init__(timezone: bool = False)
```

*继承自* `DateTime` *方法的* `sqlalchemy.types.DateTime.__init__` *方法*

构造一个新的 `DateTime`。

参数：

**时区** – 布尔值。指示日期时间类型应在仅在基本日期/时间保持类型上可用时启用时区支持。建议在使用此标志时直接使用`TIMESTAMP`数据类型，因为某些数据库包括与时区支持 TIMESTAMP 数据类型不同的通用日期/时间保持类型，如 Oracle。

```py
class sqlalchemy.dialects.oracle.FLOAT
```

Oracle FLOAT。

这与`FLOAT`相同，不同之处在于接受一个 Oracle 特定的`FLOAT.binary_precision`参数，并且不接受`Float.precision`参数。

Oracle FLOAT 类型以“二进制精度”表示精度，默认为 126。对于 REAL 类型，该值为 63。该参数不清晰地映射到特定数量的小数位，但大致相当于所需小数位数除以 0.3103。

新版本 2.0 中新增。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.oracle.FLOAT` (`sqlalchemy.types.FLOAT`)

```py
method __init__(binary_precision=None, asdecimal=False, decimal_return_scale=None)
```

构造一个浮点数

参数：

+   `binary_precision` – 要在 DDL 中呈现的 Oracle 二进制精度值。这可以使用公式“十进制精度 = 0.30103 * 二进制精度”来近似为十进制字符的数量。Oracle 用于 FLOAT / DOUBLE PRECISION 的默认值为 126。

+   `asdecimal` – 参见`Float.asdecimal`

+   `decimal_return_scale` – 参见`Float.decimal_return_scale`

```py
class sqlalchemy.dialects.oracle.INTERVAL
```

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.oracle.INTERVAL` (`sqlalchemy.types.NativeForEmulated`, `sqlalchemy.types._AbstractInterval`)

```py
method __init__(day_precision=None, second_precision=None)
```

构造一个间隔。

请注意，目前仅支持“DAY TO SECOND”间隔。这是由于可用的 DBAPI 不支持“YEAR TO MONTH”间隔。

参数：

+   `day_precision` – 天精度值。这是用于存储天字段的数字位数。默认为“2”

+   `second_precision` – 秒精度值。这是用于存储小数秒字段的数字位数。默认为“6”。

```py
class sqlalchemy.dialects.oracle.NCLOB
```

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.oracle.NCLOB` (`sqlalchemy.types.Text`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` 的 `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中使用`length`，如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是特定于数据库的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，`Column` 预计存储非 ASCII 数据的列应使用`Unicode`或`UnicodeText`数据类型。这些数据类型将确保在数据库上使用正确的类型。

```py
attribute sqlalchemy.dialects.oracle.NVARCHAR2
```

别名为 `NVARCHAR`

```py
class sqlalchemy.dialects.oracle.NUMBER
```

**类签名**

class `sqlalchemy.dialects.oracle.NUMBER` (`sqlalchemy.types.Numeric`, `sqlalchemy.types.Integer`)

```py
class sqlalchemy.dialects.oracle.LONG
```

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.oracle.LONG` (`sqlalchemy.types.Text`)

```py
method __init__(length: int | None = None, collation: str | None = None)
```

*继承自* `String` 的 `sqlalchemy.types.String.__init__` *方法*

创建一个持有字符串的类型。

参数：

+   `length` – 可选，用于 DDL 和 CAST 表达式中的列长度。如果不会发出`CREATE TABLE`，则可以安全地省略。某些数据库可能需要在 DDL 中���用`length`，如果包含没有长度的`VARCHAR`，则在发出`CREATE TABLE` DDL 时会引发异常。值是以字节还是字符解释是特定于数据库的。

+   `collation` –

    可选，用于 DDL 和 CAST 表达式中的列级排序。使用 SQLite、MySQL 和 PostgreSQL 支持的 COLLATE 关键字进行呈现。例如：

    ```py
    >>> from sqlalchemy import cast, select, String
    >>> print(select(cast('some string', String(collation='utf8'))))
    SELECT  CAST(:param_1  AS  VARCHAR  COLLATE  utf8)  AS  anon_1 
    ```

    注意

    在大多数情况下，应该使用 `Unicode` 或 `UnicodeText` 数据类型来存储非 ASCII 数据的 `Column`。这些数据类型将确保在数据库上使用正确的类型。

```py
class sqlalchemy.dialects.oracle.RAW
```

**类签名**

类 `sqlalchemy.dialects.oracle.RAW` （`sqlalchemy.types._Binary`）

```py
class sqlalchemy.dialects.oracle.ROWID
```

Oracle ROWID 类型。

当在 cast() 或类似情况下使用时，生成 ROWID。

**类签名**

类 `sqlalchemy.dialects.oracle.ROWID` （`sqlalchemy.types.TypeEngine`）

```py
class sqlalchemy.dialects.oracle.TIMESTAMP
```

Oracle 实现的 `TIMESTAMP`，支持附加的 Oracle 特定模式

2.0 版本中的新增功能。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.oracle.TIMESTAMP` （`sqlalchemy.types.TIMESTAMP`）

```py
method __init__(timezone: bool = False, local_timezone: bool = False)
```

构造一个新的 `TIMESTAMP`。

参数：

+   `timezone` – 布尔值。指示 TIMESTAMP 类型应该使用 Oracle 的 `TIMESTAMP WITH TIME ZONE` 数据类型。

+   `local_timezone` – 布尔值。指示 TIMESTAMP 类型应该使用 Oracle 的 `TIMESTAMP WITH LOCAL TIME ZONE` 数据类型。

## cx_Oracle

通过 cx-Oracle 驱动程序支持 Oracle 数据库。

### DBAPI

cx-Oracle 的文档和下载信息（如果适用）可在此处找到：[`oracle.github.io/python-cx_Oracle/`](https://oracle.github.io/python-cx_Oracle/)

### 连接

连接字符串：

```py
oracle+cx_oracle://user:pass@hostname:port[/dbname][?service_name=<service>[&key=value&key=value...]]
```

### DSN vs. 主机名连接

cx_Oracle 提供了几种指示目标数据库的方法。方言从一系列不同的 URL 形式转换而来。

#### 使用 Easy Connect 语法的主机名连接

给定目标 Oracle 数据库的主机名、端口和服务名称，例如来自 Oracle 的 [Easy Connect 语法](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#easy-connect-syntax-for-connection-strings)，然后使用 SQLAlchemy 中的 `service_name` 查询字符串参数进行连接：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:port/?service_name=myservice&encoding=UTF-8&nencoding=UTF-8")
```

完整的 Easy Connect 语法不受支持。请使用 `tnsnames.ora` 文件，并使用 DSN 进行连接。

#### 与 tnsnames.ora 或 Oracle Cloud 的连接

或者，如果未提供端口、数据库名称或 `service_name`，则方言将使用 Oracle DSN “连接字符串”。这将“主机名”部分的 URL 作为数据源名称。例如，如果 `tnsnames.ora` 文件包含以下内容的 [Net Service Name](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#net-service-names-for-connection-strings)：

```py
myalias =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = mymachine.example.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orclpdb1)
    )
  )
```

当 `myalias` 是 URL 的主机名部分时，cx_Oracle 方言将连接到此数据库服务，而不指定端口、数据库名称或 `service_name`：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@myalias/?encoding=UTF-8&nencoding=UTF-8")
```

Oracle Cloud 的用户应该使用此语法，并按照 cx_Oracle 文档中所示配置云钱包 [连接到自主数据库](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#connecting-to-autononmous-databases)。

#### SID 连接

要使用 Oracle 的过时 SID 连接语法，SID 可以在 URL 的“数据库名称”部分中传递，如下所示：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:1521/dbname?encoding=UTF-8&nencoding=UTF-8")
```

上面，传递给 cx_Oracle 的 DSN 是由 `cx_Oracle.makedsn()` 创建的，如下所示：

```py
>>> import cx_Oracle
>>> cx_Oracle.makedsn("hostname", 1521, sid="dbname")
'(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=hostname)(PORT=1521))(CONNECT_DATA=(SID=dbname)))'
```

### 传递 cx_Oracle 连接参数

通常可以通过 URL 查询字符串传递其他连接参数；特定符号如 `cx_Oracle.SYSDBA` 将被拦截并转换为正确的符号：

```py
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn?encoding=UTF-8&nencoding=UTF-8&mode=SYSDBA&events=true")
```

从版本 1.3 开始：cx_oracle 方言现在接受 URL 字符串本身中的所有参数名称，以传递给 cx_Oracle DBAPI。与之前的情况一样，但没有正确记录，`create_engine.connect_args` 参数也接受 cx_Oracle DBAPI 的所有连接参数。

要直接传递参数给 `.connect()` 而不使用查询字符串，可以使用 `create_engine.connect_args` 字典。可以传递任何 cx_Oracle 参数值和/或常量，例如：

```py
import cx_Oracle
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn",
    connect_args={
        "encoding": "UTF-8",
        "nencoding": "UTF-8",
        "mode": cx_Oracle.SYSDBA,
        "events": True
    }
)
```

请注意，在 cx_Oracle 8.0 中，`encoding` 和 `nencoding` 的默认值已更改为“UTF-8”，因此在使用该版本或更高版本时，可以省略这些参数。

### 在驱动程序之外由 SQLAlchemy cx_Oracle 方言消耗的选项

还有一些选项是由 SQLAlchemy cx_oracle 方言本身消耗的。这些选项总是直接传递给 `create_engine()` ，例如：

```py
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn", coerce_to_decimal=False)
```

cx_oracle 方言接受的参数如下：

+   `arraysize` - 在游标上设置 cx_oracle.arraysize 值；默认为 `None`，表示应使用驱动程序的默认值（通常该值为 100）。此设置控制在获取行时缓冲多少行，并且在修改时可能对性能产生重大影响。该设置用于 `cx_Oracle` 以及 `oracledb`。

    从版本 2.0.26 起更改：- 将默认值从 50 更改为 None，以使用驱动程序本身的默认值。

+   `auto_convert_lobs` - 默认为 True；详见 LOB 数据类型。

+   `coerce_to_decimal` - 详见精确数值。

+   `encoding_errors` - 详见编码错误。

### 使用 cx_Oracle SessionPool

cx_Oracle 库提供了自己的连接池实现，可以用来替代 SQLAlchemy 的连接池功能。这可以通过使用`create_engine.creator`参数提供一个返回新连接的函数，以及设置`create_engine.pool_class`为`NullPool`来禁用 SQLAlchemy 的连接池来实现：

```py
import cx_Oracle
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

pool = cx_Oracle.SessionPool(
    user="scott", password="tiger", dsn="orclpdb",
    min=2, max=5, increment=1, threaded=True,
    encoding="UTF-8", nencoding="UTF-8"
)

engine = create_engine("oracle+cx_oracle://", creator=pool.acquire, poolclass=NullPool)
```

上述引擎可以正常使用，其中 cx_Oracle 的池处理连接池：

```py
with engine.connect() as conn:
    print(conn.scalar("select 1 FROM dual"))
```

除了为多用户应用程序提供可扩展的解决方案外，cx_Oracle 会话池还支持一些 Oracle 功能，如 DRCP 和[应用连续性](https://cx-oracle.readthedocs.io/en/latest/user_guide/ha.html#application-continuity-ac)。

### 使用 Oracle 数据库 Resident Connection Pooling（DRCP）

在使用 Oracle 的[DRCP](https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-015CA8C1-2386-4626-855D-CC546DDC1086)时，最佳实践是在从 SessionPool 获取连接时传递连接类和“纯度”。请参阅[cx_Oracle DRCP 文档](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#database-resident-connection-pooling-drcp)。

这可以通过包装`pool.acquire()`来实现：

```py
import cx_Oracle
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

pool = cx_Oracle.SessionPool(
    user="scott", password="tiger", dsn="orclpdb",
    min=2, max=5, increment=1, threaded=True,
    encoding="UTF-8", nencoding="UTF-8"
)

def creator():
    return pool.acquire(cclass="MYCLASS", purity=cx_Oracle.ATTR_PURITY_SELF)

engine = create_engine("oracle+cx_oracle://", creator=creator, poolclass=NullPool)
```

上述引擎可以正常使用，其中 cx_Oracle 处理会话池，Oracle 数据库另外使用 DRCP：

```py
with engine.connect() as conn:
    print(conn.scalar("select 1 FROM dual"))
```

### Unicode

对于 Python 3 下的所有 DBAPI，所有字符串都是本质上的 Unicode 字符串。然而，在所有情况下，驱动程序都需要明确的编码配置。

#### 确保正确的客户端编码

几乎所有与 Oracle 相关的软件建立客户端编码的长期接受标准是通过[NLS_LANG](https://www.oracle.com/database/technologies/faq-nls-lang.html)环境变量。cx_Oracle 像大多数其他 Oracle 驱动程序一样将使用此环境变量作为其编码配置的来源。此变量的格式是特殊的；典型值可能是`AMERICAN_AMERICA.AL32UTF8`。

cx_Oracle 驱动程序还支持一种编程替代方案，即直接将`encoding`和`nencoding`参数传递给其`.connect()`函数。这些可以在 URL 中如下所示：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@orclpdb/?encoding=UTF-8&nencoding=UTF-8")
```

有关`encoding`和`nencoding`参数的含义，请参阅[字符集和国家语言支持（NLS）](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization)。

另请参阅

在 cx_Oracle 文档中 [字符集和国家语言支持 (NLS)](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization)。

#### Unicode 特定的列数据类型

核心表达语言通过使用 `Unicode` 和 `UnicodeText` 数据类型处理 Unicode 数据。这些类型默认对应于 VARCHAR2 和 CLOB Oracle 数据类型。当使用这些数据类型处理 Unicode 数据时，期望 Oracle 数据库配置有 Unicode-aware 字符集，并且 `NLS_LANG` 环境变量已适当设置，以便 VARCHAR2 和 CLOB 数据类型能够容纳数据。

如果 Oracle 数据库未配置 Unicode 字符集，则有两种选择：显式使用 `NCHAR` 和 `NCLOB` 数据类型，或者在调用 `create_engine()` 时传递标志 `use_nchar_for_unicode=True`，这将导致 SQLAlchemy 方言使用 NCHAR/NCLOB 替代 VARCHAR/CLOB 用于 `Unicode` / `UnicodeText` 数据类型。

1.3 版本更改：`Unicode` 和 `UnicodeText` 数据类型现在默认对应于 `VARCHAR2` 和 `CLOB` Oracle 数据类型，除非在调用 `create_engine()` 时传递了 `use_nchar_for_unicode=True`。

#### 编码错误

对于 Oracle 数据库中存在损坏编码的情况，方言接受一个参数 `encoding_errors`，该参数将传递给 Unicode 解码函数，以影响如何处理解码错误。该值最终由 Python [decode](https://docs.python.org/3/library/stdtypes.html#bytes.decode) 函数消耗，并且通过 cx_Oracle 的 `encodingErrors` 参数和 SQLAlchemy 自己的解码函数传递，因为在不同情况下 cx_Oracle 方言都会使用它们。

新版本 1.3.11 的新增功能：### 通过 setinputsizes 实现对 cx_Oracle 数据绑定性能的精细控制

cx_Oracle DBAPI 对 DBAPI `setinputsizes()` 调用具有深刻且基本的依赖关系。此调用的目的是为了为作为参数传递的 Python 值绑定到 SQL 语句的数据类型。虽然几乎没有其他 DBAPI 对`setinputsizes()`调用分配任何用途，但 cx_Oracle DBAPI 在与 Oracle 客户端接口的交互中严重依赖它，在某些情况下，SQLAlchemy 不可能知道数据应该如何绑定，因为某些设置可能导致完全不同的性能特征，同时还改变了类型强制转换行为。

使用 cx_Oracle 方言的用户**强烈建议**阅读 cx_Oracle 内置数据类型符号列表，网址为[`cx-oracle.readthedocs.io/en/latest/api_manual/module.html#database-types`](https://cx-oracle.readthedocs.io/en/latest/api_manual/module.html#database-types)。请注意，在某些情况下，使用这些类型可能会导致显著的性能下降，尤其是在指定`cx_Oracle.CLOB`时。

在 SQLAlchemy 方面，可以使用`DialectEvents.do_setinputsizes()`事件来实现运行时可见性（例如日志记录）和完全控制每个语句上如何使用`setinputsizes()`。

版本 1.2.9 中新增：添加了`DialectEvents.setinputsizes()`

#### 示例 1 - 记录所有 setinputsizes 调用

下面的示例说明了如何在将其转换为原始`setinputsizes()`参数字典之前，从 SQLAlchemy 视角记录中间值。字典的键是具有`.key`和`.type`属性的`BindParameter`对象：

```py
from sqlalchemy import create_engine, event

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _log_setinputsizes(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in inputsizes.items():
            log.info(
                "Bound parameter name: %s SQLAlchemy type: %r "
                "DBAPI object: %s",
                bindparam.key, bindparam.type, dbapitype)
```

#### 示例 2 - 删除所有与 CLOB 的绑定

在 cx_Oracle 中，`CLOB` 数据类型会导致显著的性能开销，但在 SQLAlchemy 1.2 系列中默认为`Text`类型。可以按以下方式修改此设置：

```py
from sqlalchemy import create_engine, event
from cx_Oracle import CLOB

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _remove_clob(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in list(inputsizes.items()):
        if dbapitype is CLOB:
            del inputsizes[bindparam]
```### RETURNING 支持

cx_Oracle 方言使用 OUT 参数实现 RETURNING。该方言完全支持 RETURNING。### LOB 数据类型

LOB 数据类型指的是诸如 CLOB、NCLOB 和 BLOB 等“大对象”数据类型。cx_Oracle 和 oracledb 的现代版本经过优化，使得这些数据类型能够作为单个缓冲区传递。因此，默认情况下，SQLAlchemy 使用这些较新的类型处理程序。

要禁用较新的类型处理程序，并将 LOB 对象作为具有`read()`方法的经典缓冲对象传递，请将参数`auto_convert_lobs=False`传递给`create_engine()`，该参数仅对整个引擎生效。

### 不支持两阶段事务

由于驱动程序支持不佳，cx_Oracle 不支持两阶段事务。 从 cx_Oracle 6.0b1 开始，两阶段事务的接口已更改为更直接地通过底层 OCI 层进行传递，并减少了自动化。 支持此系统的附加逻辑未在 SQLAlchemy 中实现。

### 精确数字

SQLAlchemy 的数字类型可以将值作为 Python `Decimal` 对象或 float 对象接收和返回。 当使用 `Numeric` 对象或其子类（如 `Float`，`DOUBLE_PRECISION` 等）时， `Numeric.asdecimal` 标志决定是否应在返回时将值强制转换为 `Decimal`，或以 float 对象返回。 在 Oracle 下情况更加复杂，如果“scale”为零，Oracle 的 `NUMBER` 类型还可以表示整数值，因此 Oracle 特定的 `NUMBER` 类型也考虑了这一点。

cx_Oracle 方言广泛使用连接和游标级别的“outputtypehandler”可调用来根据需要强制转换数值。 这些可调用是针对正在使用的具体 `Numeric` 的特定风味的，以及如果不存在 SQLAlchemy 类型化对象。 已观察到的情况包括 Oracle 可能发送有关返回的数字类型的不完整或模糊信息的情况，例如查询，其中数字类型被嵌套在多个子查询的多个级别下。 类型处理程序在所有情况下都尽力做出正确的决定，在所有情况下都将决策委托给底层的 cx_Oracle DBAPI，以便在驱动程序可以做出最佳决策的所有情况下进行。

当不存在类型化对象时，例如在执行纯 SQL 字符串时，存在默认的“outputtypehandler”，该处理程序通常将指定精度和比例的数字值作为 Python `Decimal` 对象返回。 为了性能原因禁用此转换为十进制数的操作，请将标志 `coerce_to_decimal=False` 传递给 `create_engine()`：

```py
engine = create_engine("oracle+cx_oracle://dsn", coerce_to_decimal=False)
```

`coerce_to_decimal` 标志仅影响与 `Numeric` SQLAlchemy 类型（或其子类）无关联的纯字符串 SQL 语句的结果。

自 1.2 版本起进行了更改：cx_Oracle 的数字处理系统已经重写，以利用较新的 cx_Oracle 特性以及更好地集成输出类型处理程序。

### DBAPI

[cx-Oracle 的文档和下载信息](https://oracle.github.io/python-cx_Oracle/)（如适用）可在此处获取。

### 连接

连接字符串：

```py
oracle+cx_oracle://user:pass@hostname:port[/dbname][?service_name=<service>[&key=value&key=value...]]
```

### DSN vs. 主机名连接

cx_Oracle 提供了几种指示目标数据库的方法。方言将一系列不同的 URL 形式进行转换。

#### 使用简易连接语法连接主机名

给定目标 Oracle 数据库的主机名、端口和服务名，例如来自 Oracle 的[简易连接语法](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#easy-connect-syntax-for-connection-strings)，然后在 SQLAlchemy 中使用`service_name`查询字符串参数进行连接：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:port/?service_name=myservice&encoding=UTF-8&nencoding=UTF-8")
```

不支持[完整的简易连接语法](https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-B0437826-43C1-49EC-A94D-B650B6A4A6EE)。而是使用`tnsnames.ora`文件，并使用 DSN 进行连接。

#### 使用 tnsnames.ora 或 Oracle Cloud 进行连接

或者，如果未提供端口、数据库名称或`service_name`，则方言将使用 Oracle DSN “连接字符串”。这将 URL 的“主机名”部分作为数据源名称。例如，如果`tnsnames.ora`文件包含如下的[网络服务名称](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#net-service-names-for-connection-strings)`myalias`：  

```py
myalias =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = mymachine.example.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orclpdb1)
    )
  )
```

当`myalias`是 URL 的主机名部分时，cx_Oracle 方言将连接到此数据库服务，而不指定端口、数据库名称或`service_name`：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@myalias/?encoding=UTF-8&nencoding=UTF-8")
```

Oracle Cloud 的用户应使用此语法，并按照 cx_Oracle 文档[连接到 Autonomous 数据库](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#connecting-to-autononmous-databases)中所示配置云钱包。

#### SID 连接

要使用 Oracle 的过时 SID 连接语法，可以如下传递 SID 在 URL 的“数据库名称”部分：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:1521/dbname?encoding=UTF-8&nencoding=UTF-8")
```

在上述代码中，传递给 cx_Oracle 的 DSN 由`cx_Oracle.makedsn()`创建，如下所示：

```py
>>> import cx_Oracle
>>> cx_Oracle.makedsn("hostname", 1521, sid="dbname")
'(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=hostname)(PORT=1521))(CONNECT_DATA=(SID=dbname)))'
```

#### 使用简易连接语法连接主机名

给定目标 Oracle 数据库的主机名、端口和服务名，例如来自 Oracle 的[简易连接语法](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#easy-connect-syntax-for-connection-strings)，然后在 SQLAlchemy 中使用`service_name`查询字符串参数进行连接：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:port/?service_name=myservice&encoding=UTF-8&nencoding=UTF-8")
```

不支持[完整的简易连接语法](https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-B0437826-43C1-49EC-A94D-B650B6A4A6EE)。而是使用`tnsnames.ora`文件，并使用 DSN 进行连接。

#### 使用 tnsnames.ora 或 Oracle Cloud 进行连接

或者，如果没有提供端口、数据库名称或`service_name`，则方言将使用 Oracle DSN “连接字符串”。这将 URL 的“主机名”部分作为数据源名称。例如，如果`tnsnames.ora`文件包含如下的[网络服务名](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#net-service-names-for-connection-strings)`myalias`：

```py
myalias =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = mymachine.example.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orclpdb1)
    )
  )
```

当`myalias`是 URL 的主机名部分时，cx_Oracle 方言将连接到此数据库服务，而不指定端口、数据库名称或`service_name`：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@myalias/?encoding=UTF-8&nencoding=UTF-8")
```

Oracle Cloud 的用户应使用此语法，并按照 cx_Oracle 文档中显示的方式配置云钱包[连接到自主数据库](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#connecting-to-autononmous-databases)。

#### SID 连接

要使用 Oracle 的过时 SID 连接语法，SID 可以在 URL 的“数据库名称”部分中传递，如下所示：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@hostname:1521/dbname?encoding=UTF-8&nencoding=UTF-8")
```

上面，传递给 cx_Oracle 的 DSN 是通过`cx_Oracle.makedsn()`创建的，如下所示：

```py
>>> import cx_Oracle
>>> cx_Oracle.makedsn("hostname", 1521, sid="dbname")
'(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=hostname)(PORT=1521))(CONNECT_DATA=(SID=dbname)))'
```

### 传递 cx_Oracle 连接参数

通常可以通过 URL 查询字符串传递其他连接参数；像`cx_Oracle.SYSDBA`这样的特殊符号将被拦截并转换为正确的符号：

```py
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn?encoding=UTF-8&nencoding=UTF-8&mode=SYSDBA&events=true")
```

从版本 1.3 开始：cx_oracle 方言现在接受 URL 字符串中的所有参数名称，以传递给 cx_Oracle DBAPI。与早期情况相同但没有正确记录的是，`create_engine.connect_args`参数也接受所有 cx_Oracle DBAPI 连接参数。

要直接传递参数给`.connect()`而不使用查询字符串，请使用`create_engine.connect_args`字典。可以传递任何 cx_Oracle 参数值和/或常量，例如：

```py
import cx_Oracle
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn",
    connect_args={
        "encoding": "UTF-8",
        "nencoding": "UTF-8",
        "mode": cx_Oracle.SYSDBA,
        "events": True
    }
)
```

请注意，在 cx_Oracle 8.0 中，`encoding`和`nencoding`的默认值已更改为“UTF-8”，因此在使用该版本或更高版本时可以省略这些参数。

### SQLAlchemy cx_Oracle 方言在驱动程序之外消耗的选项

还有一些选项是由 SQLAlchemy cx_oracle 方言本身消耗的。这些选项始终直接传递给`create_engine()`，例如：

```py
e = create_engine(
    "oracle+cx_oracle://user:pass@dsn", coerce_to_decimal=False)
```

cx_oracle 方言接受的参数如下：

+   `arraysize` - 设置光标上的 cx_oracle.arraysize 值；默认为`None`，表示应使用驱动程序的默认值（通常值为 100）。此设置控制在提取行时缓冲多少行，并且在修改时可能会对性能产生显着影响。该设置用于`cx_Oracle`以及`oracledb`。

    改变版本 2.0.26：- 将默认值从 50 更改为 None，以使用驱动程序本身的默认值。

+   `auto_convert_lobs` - 默认为 True；请参阅 LOB 数据类型。

+   `coerce_to_decimal` - 详情请参阅精确数字。

+   `encoding_errors` - 详情请参阅编码错误。

### 使用 cx_Oracle SessionPool

cx_Oracle 库提供了自己的连接池实现，可以代替 SQLAlchemy 的池功能。这可以通过使用 `create_engine.creator` 参数提供一个返回新连接的函数，以及将 `create_engine.pool_class` 设置为 `NullPool` 来实现禁用 SQLAlchemy 的池：

```py
import cx_Oracle
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

pool = cx_Oracle.SessionPool(
    user="scott", password="tiger", dsn="orclpdb",
    min=2, max=5, increment=1, threaded=True,
    encoding="UTF-8", nencoding="UTF-8"
)

engine = create_engine("oracle+cx_oracle://", creator=pool.acquire, poolclass=NullPool)
```

然后可以正常使用上述引擎，其中 cx_Oracle 的池处理连接池：

```py
with engine.connect() as conn:
    print(conn.scalar("select 1 FROM dual"))
```

除了为多用户应用程序提供可扩展的解决方案外，cx_Oracle 会话池还支持一些 Oracle 功能，例如 DRCP 和[应用程序连续性](https://cx-oracle.readthedocs.io/en/latest/user_guide/ha.html#application-continuity-ac)。

### 使用 Oracle 数据库常驻连接池（DRCP）

当使用 Oracle 的[DRCP](https://www.oracle.com/pls/topic/lookup?ctx=dblatest&id=GUID-015CA8C1-2386-4626-855D-CC546DDC1086)时，最佳实践是在从 SessionPool 获取连接时传递连接类和“纯度”。参考 [cx_Oracle DRCP 文档](https://cx-oracle.readthedocs.io/en/latest/user_guide/connection_handling.html#database-resident-connection-pooling-drcp)。

这可以通过包装 `pool.acquire()` 来实现：

```py
import cx_Oracle
from sqlalchemy import create_engine
from sqlalchemy.pool import NullPool

pool = cx_Oracle.SessionPool(
    user="scott", password="tiger", dsn="orclpdb",
    min=2, max=5, increment=1, threaded=True,
    encoding="UTF-8", nencoding="UTF-8"
)

def creator():
    return pool.acquire(cclass="MYCLASS", purity=cx_Oracle.ATTR_PURITY_SELF)

engine = create_engine("oracle+cx_oracle://", creator=creator, poolclass=NullPool)
```

然后可以正常使用上述引擎，其中 cx_Oracle 处理会话池，Oracle 数据库另外使用 DRCP：

```py
with engine.connect() as conn:
    print(conn.scalar("select 1 FROM dual"))
```

### Unicode

对于 Python 3 下的所有 DBAPI，所有字符串本质上都是 Unicode 字符串。然而，在所有情况下，驱动程序都需要明确的编码配置。

#### 确保正确的客户端编码

几乎所有与 Oracle 相关的软件的建立客户端编码的长期接受标准是通过[NLS_LANG](https://www.oracle.com/database/technologies/faq-nls-lang.html)环境变量。像大多数其他 Oracle 驱动程序一样，cx_Oracle 将使用此环境变量作为其编码配置的源。该变量的格式是特殊的；典型值可能是 `AMERICAN_AMERICA.AL32UTF8`。

cx_Oracle 驱动程序还支持一种编程替代方法，即直接将 `encoding` 和 `nencoding` 参数传递给其 `.connect()` 函数。这些可以在 URL 中存在如下：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@orclpdb/?encoding=UTF-8&nencoding=UTF-8")
```

关于 `encoding` 和 `nencoding` 参数的含义，请参阅[字符集和国家语言支持（NLS）](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization)。

另请参阅

[字符集和国家语言支持 (NLS)](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization) - 在 cx_Oracle 文档中。

#### Unicode 特定的列数据类型

核心表达式语言通过使用 `Unicode` 和 `UnicodeText` 数据类型处理 Unicode 数据。这些类型默认对应于 VARCHAR2 和 CLOB Oracle 数据类型。当使用这些数据类型处理 Unicode 数据时，期望 Oracle 数据库配置了 Unicode-aware 字符集，并且 `NLS_LANG` 环境变量被适当设置，以便 VARCHAR2 和 CLOB 数据类型可以容纳数据。

如果 Oracle 数据库未配置为 Unicode 字符集，则两个选项是显式使用 `NCHAR` 和 `NCLOB` 数据类型，或者在调用 `create_engine()` 时传递 `use_nchar_for_unicode=True` 标志，这将导致 SQLAlchemy 方言对 `Unicode` / `UnicodeText` 数据类型使用 NCHAR/NCLOB 而不是 VARCHAR/CLOB。  

版本 1.3 中的变更：`Unicode` 和 `UnicodeText` 数据类型现在对应于 `VARCHAR2` 和 `CLOB` Oracle 数据类型，除非在调用 `create_engine()` 时传递了 `use_nchar_for_unicode=True` 参数给方言。

#### 编码错误

对于 Oracle 数据库中数据存在破损编码的特殊情况，方言接受一个名为 `encoding_errors` 的参数，该参数将传递给 Unicode 解码函数，以影响如何处理解码错误。该值最终由 Python 的 [decode](https://docs.python.org/3/library/stdtypes.html#bytes.decode) 函数消耗，并且通过 cx_Oracle 的 `encodingErrors` 参数（由 `Cursor.var()` 消耗）以及 SQLAlchemy 自己的解码函数传递，因为在不同情况下 cx_Oracle 方言都会使用它们。

新版本 1.3.11 中新增。

#### 确保正确的客户端编码

几乎所有与 Oracle 相关的软件建立客户端编码的长期接受标准是通过 [NLS_LANG](https://www.oracle.com/database/technologies/faq-nls-lang.html) 环境变量。cx_Oracle 像大多数其他 Oracle 驱动程序一样将使用此环境变量作为其编码配置的来源。该变量的格式是特殊的；典型值可能是 `AMERICAN_AMERICA.AL32UTF8`。

cx_Oracle 驱动程序还支持一种编程方式，即直接将 `encoding` 和 `nencoding` 参数传递给其 `.connect()` 函数。可以在 URL 中以以下方式存在：

```py
engine = create_engine("oracle+cx_oracle://scott:tiger@orclpdb/?encoding=UTF-8&nencoding=UTF-8")
```

关于 `encoding` 和 `nencoding` 参数的含义，请参阅[字符集和国家语言支持（NLS）](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization)。

参见

[字符集和国家语言支持（NLS）](https://cx-oracle.readthedocs.io/en/latest/user_guide/globalization.html#globalization) - 在 cx_Oracle 文档中。

#### Unicode 特定列数据类型

核心表达式语言通过使用 `Unicode` 和 `UnicodeText` 数据类型处理 Unicode 数据。这些类型默认对应于 VARCHAR2 和 CLOB Oracle 数据类型。当使用这些数据类型处理 Unicode 数据时，预期 Oracle 数据库已配置为使用 Unicode 意识字符集，并且 `NLS_LANG` 环境变量已适当设置，以便 VARCHAR2 和 CLOB 数据类型可以容纳数据。

如果 Oracle 数据库未配置为 Unicode 字符集，则两个选项是显式使用 `NCHAR` 和 `NCLOB` 数据类型，或者在调用 `create_engine()` 时传递标志 `use_nchar_for_unicode=True` 给 SQLAlchemy 方言，这将导致 SQLAlchemy 方言在 Unicode / UnicodeText 数据类型上使用 NCHAR/NCLOB 而不是 VARCHAR/CLOB。

从版本 1.3 开始更改：`Unicode` 和 `UnicodeText` 数据类型现在对应于 `VARCHAR2` 和 `CLOB` Oracle 数据类型，除非在调用 `create_engine()` 时传递了 `use_nchar_for_unicode=True`。

#### 编码错误

对于 Oracle 数据库中存在损坏编码的特殊情况，该方言接受一个名为 `encoding_errors` 的参数，该参数将传递给 Unicode 解码函数，以影响如何处理解码错误。该值最终由 Python 的 [decode](https://docs.python.org/3/library/stdtypes.html#bytes.decode) 函数消耗，并且通过 cx_Oracle 的 `encodingErrors` 参数传递给 `Cursor.var()`，以及通过 SQLAlchemy 自己的解码函数传递，因为在不同情况下 cx_Oracle 方言都会使用两者。

自版本 1.3.11 起新增。

### 使用 setinputsizes 对 cx_Oracle 数据绑定性能进行精细控制

cx_Oracle DBAPI 对 DBAPI `setinputsizes()` 调用具有深层且根本的依赖性。此调用的目的是为通过参数传递的 Python 值绑定到 SQL 语句的数据类型建立起来。虽然几乎没有其他 DBAPI 将任何用途分配给 `setinputsizes()` 调用，但是 cx_Oracle DBAPI 在与 Oracle 客户端接口的交互中大量依赖它，并且在某些情况下，SQLAlchemy 无法确切地知道数据应该如何绑定，因为某些设置可能会导致性能特性发生深刻不同，同时改变类型强制转换行为。

强烈建议 cx_Oracle 方言的用户阅读 cx_Oracle 内置数据类型符号的列表，网址为 [`cx-oracle.readthedocs.io/en/latest/api_manual/module.html#database-types`](https://cx-oracle.readthedocs.io/en/latest/api_manual/module.html#database-types)。请注意，在某些情况下，使用这些类型与不使用这些类型相比，性能可能会显著下降，特别是在指定 `cx_Oracle.CLOB` 时。

在 SQLAlchemy 方面，`DialectEvents.do_setinputsizes()` 事件可用于在运行时（例如记录）可见 setinputsizes 步骤，以及完全控制每个语句如何使用 `setinputsizes()`。

自版本 1.2.9 起新增：增加了 `DialectEvents.setinputsizes()`

#### 示例 1 - 记录所有 setinputsizes 调用

以下示例说明了如何在转换为原始 `setinputsizes()` 参数字典之前从 SQLAlchemy 视角记录中间值。字典的键是具有 `.key` 和 `.type` 属性的 `BindParameter` 对象：

```py
from sqlalchemy import create_engine, event

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _log_setinputsizes(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in inputsizes.items():
            log.info(
                "Bound parameter name: %s SQLAlchemy type: %r "
                "DBAPI object: %s",
                bindparam.key, bindparam.type, dbapitype)
```

#### 示例 2 - 移除所有对 CLOB 的绑定

在 cx_Oracle 中，`CLOB` 数据类型会导致显着的性能开销，但是在 SQLAlchemy 1.2 系列中，默认为 `Text` 类型设置了该类型。可以按照以下方式修改此设置：

```py
from sqlalchemy import create_engine, event
from cx_Oracle import CLOB

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _remove_clob(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in list(inputsizes.items()):
        if dbapitype is CLOB:
            del inputsizes[bindparam]
```

#### 示例 1 - 记录所有 setinputsizes 调用

以下示例说明了如何在 SQLAlchemy 视角下记录中间值，然后再将它们转换为原始`setinputsizes()`参数字典。字典的键是具有`.key`和`.type`属性的`BindParameter`对象：

```py
from sqlalchemy import create_engine, event

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _log_setinputsizes(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in inputsizes.items():
            log.info(
                "Bound parameter name: %s SQLAlchemy type: %r "
                "DBAPI object: %s",
                bindparam.key, bindparam.type, dbapitype)
```

#### 示例 2 - 删除所有对 CLOB 的绑定

在 cx_Oracle 中，`CLOB` 数据类型会产生显着的性能开销，但在 SQLAlchemy 1.2 系列中默认设置为`Text`类型。可以按以下方式修改此设置：

```py
from sqlalchemy import create_engine, event
from cx_Oracle import CLOB

engine = create_engine("oracle+cx_oracle://scott:tiger@host/xe")

@event.listens_for(engine, "do_setinputsizes")
def _remove_clob(inputsizes, cursor, statement, parameters, context):
    for bindparam, dbapitype in list(inputsizes.items()):
        if dbapitype is CLOB:
            del inputsizes[bindparam]
```

### RETURNING 支持

cx_Oracle 方言使用 OUT 参数实现 RETURNING。该方言完全支持 RETURNING。

### LOB 数据类型

LOB 数据类型指的是诸如 CLOB、NCLOB 和 BLOB 等“大对象”数据类型。现代版本的 cx_Oracle 和 oracledb 都经过优化，以便将这些数据类型作为单个缓冲区传递。因此，默认情况下 SQLAlchemy 使用这些较新的类型处理程序。

要禁用较新类型处理程序的使用，并将 LOB 对象作为具有`read()`方法的经典缓冲对象传递，可以将参数`auto_convert_lobs=False`传递给`create_engine()`，这仅在整个引擎范围内生效。

### 不支持两阶段事务

由于 cx_Oracle 的驱动程序支持不佳，cx_Oracle 不支持两阶段事务。从 cx_Oracle 6.0b1 开始，用于两阶段事务的接口已更改为更直接地通过到底层 OCI 层的传递，自动化程度较低。支持此系统的附加逻辑未在 SQLAlchemy 中实现。

### 精确数值

SQLAlchemy 的数值类型可以处理接收和返回 Python `Decimal` 对象或浮点对象的值。当使用 `Numeric` 对象或其子类如 `Float`、`DOUBLE_PRECISION` 等时，`Numeric.asdecimal` 标志确定返回时值是否应强制转换为 `Decimal`，或作为浮点对象返回。在 Oracle 下更加复杂的是，如果“scale”为零，Oracle 的 `NUMBER` 类型也可以表示整数值，因此 Oracle 特定的 `NUMBER` 类型也考虑到了这一点。

cx_Oracle 方言广泛使用连接和游标级别的“outputtypehandler”可调用对象，以按请求强制转换数值。这些可调用对象特定于正在使用的特定`Numeric`的类型，以及如果没有 SQLAlchemy 类型化对象存在。已经观察到 Oracle 可能会发送关于返回的数值类型不完整或模糊的信息的情况，例如查询，其中数值类型被埋在多级子查询下。类型处理程序尽最大努力在所有情况下做出正确的决定，在所有情况下都推迟到底层 cx_Oracle DBAPI，以便在驱动程序可以做出最佳决定的所有这些情况下。

当没有类型化对象时，例如执行纯 SQL 字符串时，存在一个默认的“outputtypehandler”，通常返回指定精度和比例的数值，其类型为 Python 的`Decimal`对象。为了出于性能考虑禁用对十进制数的强制转换，请在`create_engine()`中传递标志`coerce_to_decimal=False`：

```py
engine = create_engine("oracle+cx_oracle://dsn", coerce_to_decimal=False)
```

`coerce_to_decimal`标志仅影响不与`Numeric` SQLAlchemy 类型（或其子类）相关联的纯字符串 SQL 语句的结果。

从 1.2 版本开始更改：cx_Oracle 的数值处理系统已经重新设计，以利用较新的 cx_Oracle 功能以及更好地集成 outputtypehandlers。

## python-oracledb

通过 python-oracledb 驱动程序支持 Oracle 数据库。

### DBAPI

有关 python-oracledb 的文档和下载信息（如果适用），请访问：[`oracle.github.io/python-oracledb/`](https://oracle.github.io/python-oracledb/)

### 连接

连接字符串：

```py
oracle+oracledb://user:pass@hostname:port[/dbname][?service_name=<service>[&key=value&key=value...]]
```

python-oracledb 是由 Oracle 发布的，旨在取代 cx_Oracle 驱动程序。它与 cx_Oracle 完全兼容，具有不需要任何依赖项的“轻客户端”模式，以及使用与 cx_Oracle 相同的方式使用 Oracle Client Interface 的“厚客户端”模式。

另请参阅

cx_Oracle - cx_Oracle 的所有注意事项也适用于 oracledb 驱动程序。

SQLAlchemy 的`oracledb`方言提供了同名的同步和异步实现。根据引擎的创建方式选择合适的版本：

+   使用`oracle+oracledb://...`调用`create_engine()`将自动选择同步版本，例如：

    ```py
    from sqlalchemy import create_engine
    sync_engine = create_engine("oracle+oracledb://scott:tiger@localhost/?service_name=XEPDB1")
    ```

+   使用`oracle+oracledb://...`调用`create_async_engine()`将自动选择异步版本，例如：

    ```py
    from sqlalchemy.ext.asyncio import create_async_engine
    asyncio_engine = create_async_engine("oracle+oracledb://scott:tiger@localhost/?service_name=XEPDB1")
    ```

可以明确指定方言的异步版本，例如使用`oracledb_async`后缀：

```py
from sqlalchemy.ext.asyncio import create_async_engine
asyncio_engine = create_async_engine("oracle+oracledb_async://scott:tiger@localhost/?service_name=XEPDB1")
```

新版本 2.0.25 中新增对 oracledb 的异步版本的支持。

### Thick mode 支持

默认情况下，`python-oracledb` 以 thin 模式启动，不需要在系统中安装 Oracle 客户端库。`python-oracledb` 驱动程序还支持一种“thick”模式，其行为类似于`cx_oracle`，并且要求安装 Oracle 客户端接口（OCI）。

要启用此模式，用户可以手动调用`oracledb.init_oracle_client`，也可以通过将参数`thick_mode=True`传递给`create_engine()`来实现。要将自定义参数传递给`init_oracle_client`，如`lib_dir`路径，则可以将字典传递给此参数，如下所示：

```py
engine = sa.create_engine("oracle+oracledb://...", thick_mode={
    "lib_dir": "/path/to/oracle/client/lib", "driver_name": "my-app"
})
```

另请参见

[`python-oracledb.readthedocs.io/en/latest/api_manual/module.html#oracledb.init_oracle_client`](https://python-oracledb.readthedocs.io/en/latest/api_manual/module.html#oracledb.init_oracle_client)

新版本 2.0.0 中新增对 oracledb 驱动程序的支持。

### DBAPI

python-oracledb 的文档和下载信息（如适用）可在此处找到：[`oracle.github.io/python-oracledb/`](https://oracle.github.io/python-oracledb/)

### 连接

连接字符串：

```py
oracle+oracledb://user:pass@hostname:port[/dbname][?service_name=<service>[&key=value&key=value...]]
```

### Thick mode 支持

默认情况下，`python-oracledb` 以 thin 模式启动，不需要在系统中安装 Oracle 客户端库。`python-oracledb` 驱动程序还支持一种“thick”模式，其行为类似于`cx_oracle`，并且要求安装 Oracle 客户端接口（OCI）。

要启用此模式，用户可以手动调用`oracledb.init_oracle_client`，也可以通过将参数`thick_mode=True`传递给`create_engine()`来实现。要将自定义参数传递给`init_oracle_client`，如`lib_dir`路径，则可以将字典传递给此参数，如下所示：

```py
engine = sa.create_engine("oracle+oracledb://...", thick_mode={
    "lib_dir": "/path/to/oracle/client/lib", "driver_name": "my-app"
})
```

另请参见

[`python-oracledb.readthedocs.io/en/latest/api_manual/module.html#oracledb.init_oracle_client`](https://python-oracledb.readthedocs.io/en/latest/api_manual/module.html#oracledb.init_oracle_client)

新版本 2.0.0 中新增对 oracledb 驱动程序的支持。
