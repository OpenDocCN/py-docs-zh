# SQLite

> 原文：[`docs.sqlalchemy.org/en/20/dialects/sqlite.html`](https://docs.sqlalchemy.org/en/20/dialects/sqlite.html)

对 SQLite 数据库的支持。

以下表格总结了数据库发布版本的当前支持水平。

**支持的 SQLite 版本**

| 支持类型 | 版本 |
| --- | --- |
| CI 中完全测试过 | 3.36.0 |
| 普通支持 | 3.12+ |
| 尽力而为 | 3.7.16+ |

## DBAPI 支持

可用以下方言/DBAPI 选项。有关连接信息，请参阅各个 DBAPI 部分。

+   pysqlite

+   aiosqlite

+   pysqlcipher

## 日期和时间类型

SQLite 没有内置的 DATE、TIME 或 DATETIME 类型，而 pysqlite 也没有提供将值在 Python datetime 对象和 SQLite 支持的格式之间转换的开箱即用功能。当使用 SQLite 时，SQLAlchemy 自己的 `DateTime` 和相关类型提供日期格式化和解析功能。实现类是 `DATETIME`、`DATE` 和 `TIME`。这些类型将日期和时间表示为 ISO 格式的字符串，也很好地支持排序。对于这些函数，不依赖于典型的“libc”内部，因此完全支持历史日期。

### 确保文本亲和性

这些类型的 DDL 渲染是标准的 `DATE`、`TIME` 和 `DATETIME` 指示符。然而，这些类型也可以应用自定义存储格式。当检测到存储格式不包含字母字符时，这些类型的 DDL 被渲染为 `DATE_CHAR`、`TIME_CHAR` 和 `DATETIME_CHAR`，以便列继续具有文本亲和性。

另请参阅

[类型亲和性](https://www.sqlite.org/datatype3.html#affinity) - SQLite 文档中的说明  ## SQLite 自增行为

SQLite 的自动增量背景资料位于：[`sqlite.org/autoinc.html`](https://sqlite.org/autoinc.html)

关键概念：

+   SQLite 对于任何非复合主键列都有一个隐式的“自动增量”功能，只要使用“INTEGER PRIMARY KEY”类型 + 主键明确创建该列即可。

+   SQLite 还有一个显式的“AUTOINCREMENT”关键字，它与隐式自增功能**不**等同；不推荐一般使用此关键字。除非使用了特殊的 SQLite 特定指令（见下文），否则 SQLAlchemy 不会渲染此关键字。但仍然要求列的类型命名为“INTEGER”。

### 使用 AUTOINCREMENT 关键字

要在渲染 DDL 时特别呈现主键列上的 AUTOINCREMENT 关键字，将标志`sqlite_autoincrement=True`添加到 Table 构造中：

```py
Table('sometable', metadata,
        Column('id', Integer, primary_key=True),
        sqlite_autoincrement=True)
```

### 允许自动增量行为的 SQLAlchemy 类型不仅限于 Integer/INTEGER

SQLite 的类型模型基于命名约定。除其他外，这意味着任何包含子字符串`"INT"`的类型名称将被确定为“整数亲和性”。一个名为`"BIGINT"`、`"SPECIAL_INT"`甚至`"XYZINTQPR"`的类型，SQLite 都会认为是“整数”亲和性。然而，**SQLite 的自动增量功能，无论是隐式还是显式启用，都要求列类型的名称正好是字符串`"INTEGER"`**。因此，如果应用程序对主键使用类似`BigInteger`的类型，在 SQLite 中，当发出初始`CREATE TABLE`语句时，此类型需要呈现为名称`"INTEGER"`，以便使自动增量行为可用。

实现此目的的一种方法是仅在 SQLite 上使用`Integer`，并使用`TypeEngine.with_variant()`：

```py
table = Table(
    "my_table", metadata,
    Column("id", BigInteger().with_variant(Integer, "sqlite"), primary_key=True)
)
```

另一种方法是使用`BigInteger`的子类，在针对 SQLite 编译时覆盖其 DDL 名称为`INTEGER`：

```py
from sqlalchemy import BigInteger
from sqlalchemy.ext.compiler import compiles

class SLBigInteger(BigInteger):
    pass

@compiles(SLBigInteger, 'sqlite')
def bi_c(element, compiler, **kw):
    return "INTEGER"

@compiles(SLBigInteger)
def bi_c(element, compiler, **kw):
    return compiler.visit_BIGINT(element, **kw)

table = Table(
    "my_table", metadata,
    Column("id", SLBigInteger(), primary_key=True)
)
```

另请参阅

`TypeEngine.with_variant()`

自定义 SQL 构造和编译扩展

[SQLite 版本 3 中的数据类型](https://sqlite.org/datatype3.html)  ## 数据库锁定行为 / 并发性

SQLite 不适用于高并发写入。数据库本身作为文件，在事务中的写操作期间完全被锁定，这意味着在此期间仅有一个“连接”（实际上是一个文件句柄）对数据库具有独占访问权限 - 在此期间所有其他“连接”将被阻塞。

Python DBAPI 规范还要求连接模型始终处于事务中；没有`connection.begin()`方法，只有`connection.commit()`和`connection.rollback()`，在其上立即开始新事务。这似乎意味着 SQLite 驱动理论上只允许在任何时候对特定数据库文件进行单个文件句柄的操作；然而，SQLite 本身以及 pysqlite 驱动中有几个因素显著放宽了这一限制。

但是，无论使用何种锁定模式，一旦启动事务并且至少发出了 DML（例如 INSERT、UPDATE、DELETE），SQLite 将始终锁定数据库文件，并且这将至少在其他事务试图发出 DML 时阻止其他事务。默认情况下，此阻塞的时间非常短，然后会超时并显示错误。

当与 SQLAlchemy ORM 结合使用时，此行为变得更加关键。SQLAlchemy 的 `Session` 对象默认在事务中运行，并且使用其自动刷新模式，可能会在任何 SELECT 语句之前发出 DML。这可能会导致 SQLite 数据库比预期更快地锁定。可以在某种程度上操纵 SQLite 和 pysqlite 驱动程序的锁定模式，但应注意，要在 SQLite 中实现高度的写并发是一场失败的战斗。

有关 SQLite 按设计缺乏写并发的更多信息，请参阅页面底部的 [在其他关系数据库管理系统可能更适合的情况下 - 高并发](https://www.sqlite.org/whentouse.html)。

以下各小节介绍了受 SQLite 文件型架构影响的区域，并在使用 pysqlite 驱动程序时通常需要解决方法才能正常工作。## 事务隔离级别 / 自动提交

SQLite 以非标准方式支持“事务隔离”，沿着两个轴。一个是 [PRAGMA read_uncommitted](https://www.sqlite.org/pragma.html#pragma_read_uncommitted) 指令。此设置可以在 SQLite 的默认模式 `SERIALIZABLE` 隔离和通常称为 `READ UNCOMMITTED` 的 “脏读” 隔离模式之间切换。

SQLAlchemy 使用 `create_engine.isolation_level` 参数的 PRAGMA 语句绑定到此。当与 SQLite 结合使用时，此参数的有效值是 `"SERIALIZABLE"` 和 `"READ UNCOMMITTED"`，分别对应值 0 和 1。SQLite 默认为 `SERIALIZABLE`，但其行为受 pysqlite 驱动程序的默认行为影响。

当使用 pysqlite 驱动程序时，还可以使用 `"AUTOCOMMIT"` 隔离级别，这将通过 DBAPI 连接上的 `.isolation_level` 属性来更改 pysqlite 连接，并在设置的持续时间内将其设置为 None。

新版本 1.3.16 中：在使用 pysqlite / sqlite3 SQLite 驱动程序时添加了对 SQLite AUTOCOMMIT 隔离级别的支持。

影响 SQLite 事务性锁定的另一个轴是使用的 `BEGIN` 语句的性质。三种变体是“deferred”、“immediate” 和 “exclusive”，如 [BEGIN TRANSACTION](https://sqlite.org/lang_transaction.html) 中所述。直接的 `BEGIN` 语句使用“deferred”模式，在第一次读取或写入操作之前不会锁定数据库文件，并且在第一次写入操作之前会保持对其他事务的读取访问打开。但是，关键要注意的是 pysqlite 驱动程序通过*甚至不发出 BEGIN*来干扰此行为。

警告

SQLite 的事务范围受到 pysqlite 驱动程序中未解决的问题的影响，该驱动程序将 BEGIN 语句推迟到比通常更大的程度。请参阅 Serializable isolation / Savepoints / Transactional DDL 或 Serializable isolation / Savepoints / Transactional DDL (asyncio version) 部分，了解解决此行为的技术。

另请参见

设置事务隔离级别，包括 DBAPI 自动提交

## INSERT/UPDATE/DELETE…RETURNING

SQLite 方言支持 SQLite 3.35 的 `INSERT|UPDATE|DELETE..RETURNING` 语法。在某些情况下，`INSERT..RETURNING` 可以自动使用，以在生成新标识符时替代传统方法使用 `cursor.lastrowid`，但是在简单的单语句情况下，目前仍更倾向于使用 `cursor.lastrowid`，因为其性能更好。

要指定显式的 `RETURNING` 子句，请在每个语句基础上使用 `_UpdateBase.returning()` 方法：

```py
# INSERT..RETURNING
result = connection.execute(
    table.insert().
    values(name='foo').
    returning(table.c.col1, table.c.col2)
)
print(result.all())

# UPDATE..RETURNING
result = connection.execute(
    table.update().
    where(table.c.name=='foo').
    values(name='bar').
    returning(table.c.col1, table.c.col2)
)
print(result.all())

# DELETE..RETURNING
result = connection.execute(
    table.delete().
    where(table.c.name=='foo').
    returning(table.c.col1, table.c.col2)
)
print(result.all())
```

版本 2.0 中的新功能：添加对 SQLite RETURNING 的支持

## SAVEPOINT 支持

SQLite 支持 SAVEPOINT，仅在事务开始后才起作用。SQLAlchemy 的 SAVEPOINT 支持可使用 Core 级别的 `Connection.begin_nested()` 方法和 ORM 级别的 `Session.begin_nested()` 方法。但是，除非采取解决方法，否则在 pysqlite 中根本无法使用 SAVEPOINT。

警告

SQLite 的 SAVEPOINT 功能受到 pysqlite 和 aiosqlite 驱动程序中未解决的问题的影响，这些驱动程序将 BEGIN 语句推迟到比通常更大的程度。请参阅 Serializable isolation / Savepoints / Transactional DDL 和 Serializable isolation / Savepoints / Transactional DDL (asyncio version) 部分，了解解决此行为的技术。

## 事务性 DDL

SQLite 数据库也支持事务性 DDL。在这种情况下，pysqlite 驱动程序不仅未能启动事务，还在检测到 DDL 时结束了任何现有事务，因此需要解决方法。

警告

SQLite 的事务 DDL 受到 pysqlite 驱动程序中未解决的问题的影响，该驱动程序在遇到 DDL 时未发出 BEGIN 并且还强制执行 COMMIT 以取消任何事务。请参阅 Serializable isolation / Savepoints / Transactional DDL 部分以了解解决此行为的技巧。

## 外键支持

SQLite 在发出 CREATE 语句创建表时支持 FOREIGN KEY 语法，但默认情况下这些约束对表的操作没有任何影响。

在 SQLite 上进行约束检查有三个前提条件：

+   必须使用至少版本 3.6.19 的 SQLite。

+   SQLite 库必须编译为 *不包含* SQLITE_OMIT_FOREIGN_KEY 或 SQLITE_OMIT_TRIGGER 符号的状态。

+   必须在所有连接上发出 `PRAGMA foreign_keys = ON` 语句，包括对 `MetaData.create_all()` 的初始调用。

SQLAlchemy 允许通过事件的使用自动发出 `PRAGMA` 语句以用于新连接：

```py
from sqlalchemy.engine import Engine
from sqlalchemy import event

@event.listens_for(Engine, "connect")
def set_sqlite_pragma(dbapi_connection, connection_record):
    cursor = dbapi_connection.cursor()
    cursor.execute("PRAGMA foreign_keys=ON")
    cursor.close()
```

警告

当启用 SQLite 外键时，**不可能** 发出包含相互依赖外键约束的表的 CREATE 或 DROP 语句；要为这些表发出 DDL，需要使用 ALTER TABLE 分别创建或删除这些约束，而 SQLite 不支持此操作。

另请参阅

[SQLite 外键支持](https://www.sqlite.org/foreignkeys.html) - SQLite 网站上的链接。

Events - SQLAlchemy 事件 API。

通过 ALTER 创建/删除外键约束 - 关于 SQLAlchemy 处理的更多信息

相互依赖的外键约束。## 用于约束的 ON CONFLICT 支持

另请参阅

本节描述了 SQLite 中在 CREATE TABLE 语句内部发生的 “ON CONFLICT” 的 DDL 版本。有关作用于 INSERT 语句的 “ON CONFLICT”，请参阅 INSERT…ON CONFLICT (Upsert)。

SQLite 支持一个名为 ON CONFLICT 的非标准 DDL 子句，可应用于主键、唯一、检查和非空约束。在 DDL 中，它要么在“CONSTRAINT”子句中呈现，要么在目标约束的位置取决于列定义本身。要在 DDL 中呈现此子句，可以使用扩展参数`sqlite_on_conflict`并在`PrimaryKeyConstraint`、`UniqueConstraint`、`CheckConstraint`对象中指定字符串冲突解析算法。在`Column`对象中，有单独的参数`sqlite_on_conflict_not_null`、`sqlite_on_conflict_primary_key`、`sqlite_on_conflict_unique`，它们分别对应于可以从`Column`对象指示的三种相关约束类型。

另请参见

[ON CONFLICT](https://www.sqlite.org/lang_conflict.html) - SQLite 文档中的内容

版本 1.3 中的新功能。

`sqlite_on_conflict`参数接受一个字符串参数，该参数只是要选择的解析名称，在 SQLite 中可以是 ROLLBACK、ABORT、FAIL、IGNORE 和 REPLACE 中的一个。例如，要添加指定 IGNORE 算法的唯一约束：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', Integer),
    UniqueConstraint('id', 'data', sqlite_on_conflict='IGNORE')
)
```

以上呈现了 CREATE TABLE DDL 如下：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    data INTEGER,
    PRIMARY KEY (id),
    UNIQUE (id, data) ON CONFLICT IGNORE
)
```

当使用`Column.unique`标志将唯一约束添加到单个列时，也可以将`sqlite_on_conflict_unique`参数添加到`Column`中，该参数将添加到 DDL 中的唯一约束中：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', Integer, unique=True,
           sqlite_on_conflict_unique='IGNORE')
)
```

渲染：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    data INTEGER,
    PRIMARY KEY (id),
    UNIQUE (data) ON CONFLICT IGNORE
)
```

要应用 FAIL 算法以满足非空约束，使用`sqlite_on_conflict_not_null`：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', Integer, nullable=False,
           sqlite_on_conflict_not_null='FAIL')
)
```

这将呈现列内联的 ON CONFLICT 短语：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    data INTEGER NOT NULL ON CONFLICT FAIL,
    PRIMARY KEY (id)
)
```

类似地，对于内联主键，请使用`sqlite_on_conflict_primary_key`：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True,
           sqlite_on_conflict_primary_key='FAIL')
)
```

SQLAlchemy 单独呈现主键约束，因此冲突解析算法应用于约束本身：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    PRIMARY KEY (id) ON CONFLICT FAIL
)
```  ## INSERT…ON CONFLICT（Upsert）

另请参见

本节描述了 SQLite 中“ON CONFLICT”的 DML 版本，它出现在 INSERT 语句中。有关应用于 CREATE TABLE 语句的“ON CONFLICT”，请参见 ON CONFLICT 支持约束。

从版本 3.24.0 开始，SQLite 支持通过 `INSERT` 语句的 `ON CONFLICT` 子句将行“upsert”（更新或插入）到表中。只有候选行不违反任何唯一约束或主键约束时，才会插入候选行。在唯一约束违反的情况下，可以发生二次操作，可以是“DO UPDATE”，表示目标行中的数据应该更新，也可以是“DO NOTHING”，表示要默默跳过此行。

冲突是使用现有唯一约束和索引的列确定的。这些约束通过说明组成索引的列和条件来识别。

SQLAlchemy 通过 SQLite 特定的 `insert()` 函数提供了 `ON CONFLICT` 支持，该函数提供了生成方法 `Insert.on_conflict_do_update()` 和 `Insert.on_conflict_do_nothing()`：

```py
>>> from sqlalchemy.dialects.sqlite import insert

>>> insert_stmt = insert(my_table).values(
...     id='some_existing_id',
...     data='inserted value')

>>> do_update_stmt = insert_stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value')
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?
>>> do_nothing_stmt = insert_stmt.on_conflict_do_nothing(
...     index_elements=['id']
... )

>>> print(do_nothing_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)
ON  CONFLICT  (id)  DO  NOTHING 
```

版本 1.4 中的新功能。

另请参阅

[Upsert](https://sqlite.org/lang_UPSERT.html) - SQLite 文档中的内容。

### 指定目标

这两种方法都使用列推断提供冲突的“目标”：

+   `Insert.on_conflict_do_update.index_elements` 参数指定一个序列，其中包含字符串列名、`Column` 对象和/或 SQL 表达式元素，用于标识唯一索引或唯一约束。

+   当使用 `Insert.on_conflict_do_update.index_elements` 推断索引时，也可以通过指定 `Insert.on_conflict_do_update.index_where` 参数来推断部分索引：

    ```py
    >>> stmt = insert(my_table).values(user_email='a@b.com', data='inserted data')

    >>> do_update_stmt = stmt.on_conflict_do_update(
    ...     index_elements=[my_table.c.user_email],
    ...     index_where=my_table.c.user_email.like('%@gmail.com'),
    ...     set_=dict(data=stmt.excluded.data)
    ...     )

    >>> print(do_update_stmt)
    INSERT  INTO  my_table  (data,  user_email)  VALUES  (?,  ?)
    ON  CONFLICT  (user_email)
    WHERE  user_email  LIKE  '%@gmail.com'
    DO  UPDATE  SET  data  =  excluded.data 
    ```

### SET 子句

`ON CONFLICT...DO UPDATE` 用于执行已存在行的更新操作，使用新值以及建议插入的值的任意组合。这些值使用 `Insert.on_conflict_do_update.set_` 参数指定。该参数接受一个字典，其中包含更新的直接值：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')

>>> do_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value')
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ? 
```

警告

`Insert.on_conflict_do_update()` 方法**不会**考虑 Python 端的默认 UPDATE 值或生成函数，例如使用 `Column.onupdate` 指定的值。这些值不会在 ON CONFLICT 类型的 UPDATE 中执行，除非它们在 `Insert.on_conflict_do_update.set_` 字典中手动指定。

### 使用排除的 INSERT 值进行更新

要引用提议的插入行，`Insert.excluded` 这个特殊别名可作为 `Insert` 对象的属性使用；这个对象在列上创建一个“excluded.” 前缀，通知 DO UPDATE 使用将要插入的值来更新行，如果约束没有失败的话：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh'
... )

>>> do_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value', author=stmt.excluded.author)
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (?,  ?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?,  author  =  excluded.author 
```

### 额外的 WHERE 条件

`Insert.on_conflict_do_update()` 方法还接受使用 `Insert.on_conflict_do_update.where` 参数的 WHERE 子句，这将限制接收 UPDATE 的行：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh'
... )

>>> on_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value', author=stmt.excluded.author),
...     where=(my_table.c.status == 2)
... )
>>> print(on_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (?,  ?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?,  author  =  excluded.author
WHERE  my_table.status  =  ? 
```

### 使用 DO NOTHING 跳过行

`ON CONFLICT` 可用于完全跳过插入行，如果与唯一约束发生冲突；下面使用 `Insert.on_conflict_do_nothing()` 方法进行说明：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')
>>> stmt = stmt.on_conflict_do_nothing(index_elements=['id'])
>>> print(stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)  ON  CONFLICT  (id)  DO  NOTHING 
```

如果使用 `DO NOTHING` 而没有指定任何列或约束，它将跳过任何唯一性冲突导致的 INSERT：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')
>>> stmt = stmt.on_conflict_do_nothing()
>>> print(stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)  ON  CONFLICT  DO  NOTHING 
```  ## 类型反射

SQLite 类型与大多数其他数据库后端的类型不同，因为类型的字符串名称通常不是一对一对应的“类型”。相反，SQLite 将每列的类型行为链接到五种所谓的“类型亲和性”之一，基于类型的字符串匹配模式。

SQLAlchemy 的反射过程，在检查类型时，使用一个简单的查找表将返回的关键字链接到提供的 SQLAlchemy 类型。这个查找表存在于 SQLite 方言中，就像所有其他方言一样。然而，当特定类型名称未在查找映射中找到时，SQLite 方言有一个不同的“回退”程序；它实际上实现了位于 [`www.sqlite.org/datatype3.html`](https://www.sqlite.org/datatype3.html) 第 2.1 节的 SQLite “类型亲和性”方案。

提供的类型映射将直接从以下类型的精确字符串名称匹配中进行关联：

如果类型名称包含字符串`BLOB`，则返回`BIGINT`、`BLOB`、`BOOLEAN`、`BOOLEAN`、`CHAR`、`DATE`、`DATETIME`、`FLOAT`、`DECIMAL`、`FLOAT`、`INTEGER`、`INTEGER`、`NUMERIC`、`REAL`、`SMALLINT`、`TEXT`、`TIME`、`TIMESTAMP`、`VARCHAR`、`NVARCHAR`、`NCHAR`

当类型名称不匹配上述类型之一时，将使用“类型亲和性”查找代替：

+   如果类型名称包含字符串`INT`，则返回`INTEGER`

+   如果类型名称包含字符串`CHAR`、`CLOB`或`TEXT`，则返回`TEXT`

+   如果类型名称包含字符串`BLOB`，则返回`NullType`

+   如果类型名称包含字符串`REAL`、`FLOA`或`DOUB`，则返回`REAL`

+   否则，将使用`NUMERIC`类型。## 部分索引

可以使用 DDL 系统使用参数`sqlite_where`来指定部分索引，例如使用 WHERE 子句的索引：

```py
tbl = Table('testtbl', m, Column('data', Integer))
idx = Index('test_idx1', tbl.c.data,
            sqlite_where=and_(tbl.c.data > 5, tbl.c.data < 10))
```

索引将在创建时呈现为：

```py
CREATE INDEX test_idx1 ON testtbl (data)
WHERE data > 5 AND data < 10
```  ## 点列名

使用明确包含句点的表格或列名**不推荐**。虽然这通常对关系数据库来说是个坏主意，因为句点是一个语法上重要的字符，但直到 SQLite 版本**3.10.0**之前的 SQLite 驱动程序存在一个 bug，需要 SQLAlchemy 在结果集中过滤掉这些句点。

这个 bug 完全不是 SQLAlchemy 的问题，可以这样说明：

```py
import sqlite3

assert sqlite3.sqlite_version_info < (3, 10, 0), "bug is fixed in this version"

conn = sqlite3.connect(":memory:")
cursor = conn.cursor()

cursor.execute("create table x (a integer, b integer)")
cursor.execute("insert into x (a, b) values (1, 1)")
cursor.execute("insert into x (a, b) values (2, 2)")

cursor.execute("select x.a, x.b from x")
assert [c[0] for c in cursor.description] == ['a', 'b']

cursor.execute('''
 select x.a, x.b from x where a=1
 union
 select x.a, x.b from x where a=2
''')
assert [c[0] for c in cursor.description] == ['a', 'b'], \
    [c[0] for c in cursor.description]
```

第二个断言失败：

```py
Traceback (most recent call last):
  File "test.py", line 19, in <module>
    [c[0] for c in cursor.description]
AssertionError: ['x.a', 'x.b']
```

在上述情况下，驱动程序错误地报告包括表名在内的列名，这与没有 UNION 时完全不一致。

SQLAlchemy 依赖于列名在匹配原始语句时的可预测性，因此 SQLAlchemy 方言别无选择，只能过滤掉这些内容：

```py
from sqlalchemy import create_engine

eng = create_engine("sqlite://")
conn = eng.connect()

conn.exec_driver_sql("create table x (a integer, b integer)")
conn.exec_driver_sql("insert into x (a, b) values (1, 1)")
conn.exec_driver_sql("insert into x (a, b) values (2, 2)")

result = conn.exec_driver_sql("select x.a, x.b from x")
assert result.keys() == ["a", "b"]

result = conn.exec_driver_sql('''
 select x.a, x.b from x where a=1
 union
 select x.a, x.b from x where a=2
''')
assert result.keys() == ["a", "b"]
```

请注意，即使 SQLAlchemy 过滤掉了句点，*这两个名称仍然可寻址*：

```py
>>> row = result.first()
>>> row["a"]
1
>>> row["x.a"]
1
>>> row["b"]
1
>>> row["x.b"]
1
```

因此，SQLAlchemy 应用的解决方法仅影响公共 API 中的`CursorResult.keys()`和`Row.keys()`，在应用被迫使用包含句点的列名，并且需要`CursorResult.keys()`和`Row.keys()`返回这些带点的名称时，可以提供`sqlite_raw_colnames`执行选项，或者基于每个`Connection`的基础上：

```py
result = conn.execution_options(sqlite_raw_colnames=True).exec_driver_sql('''
 select x.a, x.b from x where a=1
 union
 select x.a, x.b from x where a=2
''')
assert result.keys() == ["x.a", "x.b"]
```

或者基于每个`Engine`的基础上：

```py
engine = create_engine("sqlite://", execution_options={"sqlite_raw_colnames": True})
```

在使用基于每个`Engine`的执行选项时，请注意**使用 UNION 的 Core 和 ORM 查询可能无法正常工作**。

## 特定于 SQLite 的表选项

一个 CREATE TABLE 的选项直接由 SQLite 方言支持，与`Table`构造一起使用：

+   `WITHOUT ROWID`：

    ```py
    Table("some_table", metadata, ..., sqlite_with_rowid=False)
    ```

另请参见

[SQLite CREATE TABLE options](https://www.sqlite.org/lang_createtable.html)

## 反射内部模式表

返回表列表的反射方法将省略所谓的“SQLite 内部模式对象”名称，这些名称被 SQLite 视为任何以`sqlite_`为前缀的对象名称。这种对象的一个例子是在使用`AUTOINCREMENT`列参数时生成的`sqlite_sequence`表。为了返回这些对象，可以将参数`sqlite_include_internal=True`传递给诸如`MetaData.reflect()`或`Inspector.get_table_names()`等方法。

新增于版本 2.0：添加了 `sqlite_include_internal=True` 参数。以前，这些表不会被 SQLAlchemy 反射方法所忽略。

注

`sqlite_include_internal` 参数不是指与 `sqlite_master` 等模式中存在的“系统”表相关的内容。

另请参见

[SQLite 内部模式对象](https://www.sqlite.org/fileformat2.html#intschema) - 在 SQLite 文档中。

## SQLite 数据类型

与所有 SQLAlchemy 方言一样，所有已知与 SQLite 兼容的大写类型都可以从顶级方言导入，无论它们是来自 `sqlalchemy.types` 还是本地方言：

```py
from sqlalchemy.dialects.sqlite import (
    BLOB,
    BOOLEAN,
    CHAR,
    DATE,
    DATETIME,
    DECIMAL,
    FLOAT,
    INTEGER,
    NUMERIC,
    JSON,
    SMALLINT,
    TEXT,
    TIME,
    TIMESTAMP,
    VARCHAR,
)
```

| 对象名称 | 描述 |
| --- | --- |
| 日期 | 使用字符串在 SQLite 中表示 Python 日期对象。 |
| 日期时间 | 使用字符串在 SQLite 中表示 Python 日期时间对象。 |
| JSON | SQLite JSON 类型。 |
| 时间 | 使用字符串在 SQLite 中表示 Python 时间对象。 |

```py
class sqlalchemy.dialects.sqlite.DATETIME
```

使用字符串在 SQLite 中表示 Python 日期时间对象。

默认的字符串存储格式为：

```py
"%(year)04d-%(month)02d-%(day)02d  %(hour)02d:%(minute)02d:%(second)02d.%(microsecond)06d"
```

例如：

```py
2021-03-15 12:05:57.105542
```

默认情况下，传入的存储格式将使用 Python 的 `datetime.fromisoformat()` 函数解析。

从版本 2.0 开始更改：默认日期时间字符串解析使用 `datetime.fromisoformat()`。

可以使用 `storage_format` 和 `regexp` 参数在一定程度上定制存储格式，例如：

```py
import re
from sqlalchemy.dialects.sqlite import DATETIME

dt = DATETIME(storage_format="%(year)04d/%(month)02d/%(day)02d "
                             "%(hour)02d:%(minute)02d:%(second)02d",
              regexp=r"(\d+)/(\d+)/(\d+) (\d+)-(\d+)-(\d+)"
)
```

参数：

+   `storage_format` – 格式字符串，将应用于具有键年、月、日、小时、分钟、秒和微秒的字典。

+   `regexp` – 将应用于传入结果行的正则表达式，替换使用 `datetime.fromisoformat()` 解析传入字符串的方法。如果正则表达式包含命名组，则将生成的匹配字典作为关键字参数应用于 Python 的 datetime() 构造函数。否则，如果使用了位置组，则通过 `*map(int, match_obj.groups(0))` 调用 datetime() 构造函数以使用位置参数。

**类签名**

类`sqlalchemy.dialects.sqlite.DATETIME`（`sqlalchemy.dialects.sqlite.base._DateTimeMixin`，`sqlalchemy.types.DateTime`)

```py
class sqlalchemy.dialects.sqlite.DATE
```

使用字符串在 SQLite 中表示 Python 日期对象。

默认的字符串存储格式为：

```py
"%(year)04d-%(month)02d-%(day)02d"
```

例如：

```py
2011-03-15
```

默认情况下，传入的存储格式将使用 Python 的 `date.fromisoformat()` 函数解析。

从版本 2.0 开始更改：默认日期字符串解析使用 `date.fromisoformat()`。

可以使用 `storage_format` 和 `regexp` 参数在一定程度上定制存储格式，例如：

```py
import re
from sqlalchemy.dialects.sqlite import DATE

d = DATE(
        storage_format="%(month)02d/%(day)02d/%(year)04d",
        regexp=re.compile("(?P<month>\d+)/(?P<day>\d+)/(?P<year>\d+)")
    )
```

参数：

+   `storage_format` – 格式字符串，将应用于具有键年、月和日的字典。

+   `regexp` – 将应用于传入结果行的正则表达式，以替换使用 `date.fromisoformat()` 来解析传入字符串。如果正则表达式包含命名组，则生成的匹配字典将作为关键字参数应用于 Python 的 date() 构造函数。否则，如果使用位置组，则通过 `*map(int, match_obj.groups(0))` 调用 date() 构造函数来传递位置参数。

**类签名**

class `sqlalchemy.dialects.sqlite.DATE` (`sqlalchemy.dialects.sqlite.base._DateTimeMixin`, `sqlalchemy.types.Date`)

```py
class sqlalchemy.dialects.sqlite.JSON
```

SQLite JSON 类型。

SQLite 从版本 3.9 开始支持 JSON，通过其 [JSON1](https://www.sqlite.org/json1.html) 扩展。请注意，[JSON1](https://www.sqlite.org/json1.html) 是一个[可加载扩展](https://www.sqlite.org/loadext.html)，因此可能不可用，或者可能需要运行时加载。

在 SQLite 后端中使用基本 `JSON` 数据类型时，`JSON` 会自动使用。

另请参阅

`JSON` - 通用跨平台 JSON 数据类型的主文档。

`JSON` 类型支持将 JSON 值持久化，以及通过在数据库级别包装 `JSON_EXTRACT` 函数并渲染为 `JSON_QUOTE` 函数来提供核心索引操作的 `JSON` 数据类型，以适应这些操作。提取的值都被引用，以确保结果始终是 JSON 字符串值。

版本 1.3 中的新功能。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.sqlite.JSON` (`sqlalchemy.types.JSON`)

```py
method __init__(none_as_null: bool = False)
```

*继承自* `JSON` 的 `sqlalchemy.types.JSON.__init__` *方法*

构造一个 `JSON` 类型。

参数：

**none_as_null=False** –

如果为 True，则将值 `None` 持久化为 SQL NULL 值，而不是 `null` 的 JSON 编码。注意，当此标志为 False 时，仍然可以使用 `null()` 构造来持久化 NULL 值，该值可以直接作为参数值传递，由 `JSON` 类型特殊解释为 SQL NULL：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"data": null()})
```

注意

`JSON.none_as_null` **不**适用于传递给 `Column.default` 和 `Column.server_default` 的值；这些参数的传递值为 `None` 意味着“没有默认值”。

此外，在 SQL 比较表达式中使用时，Python 值 `None` 仍然表示 SQL 空值，而不是 JSON NULL。`JSON.none_as_null` 标志显式指定了值在 INSERT 或 UPDATE 语句中的**持久性**。应该使用 `JSON.NULL` 值来表示希望与 JSON 空值进行比较的 SQL 表达式。

另请参阅

`JSON.NULL`

```py
class sqlalchemy.dialects.sqlite.TIME
```

使用字符串在 SQLite 中表示 Python 时间对象。

默认字符串存储格式为：

```py
"%(hour)02d:%(minute)02d:%(second)02d.%(microsecond)06d"
```

例如：

```py
12:05:57.10558
```

默认情况下，传入的存储格式使用 Python 的 `time.fromisoformat()` 函数解析。

自 2.0 版本更改：默认时间字符串解析现在使用 `time.fromisoformat()`。

存储格式可以在一定程度上使用 `storage_format` 和 `regexp` 参数进行自定义，例如：

```py
import re
from sqlalchemy.dialects.sqlite import TIME

t = TIME(storage_format="%(hour)02d-%(minute)02d-"
                        "%(second)02d-%(microsecond)06d",
         regexp=re.compile("(\d+)-(\d+)-(\d+)-(?:-(\d+))?")
)
```

参数：

+   `storage_format` – 将应用于包含小时、分钟、秒和微秒键的字典的格式字符串。

+   `regexp` – 将应用于传入结果行的正则表达式，取代使用 `datetime.fromisoformat()` 解析传入字符串。如果正则表达式包含命名组，则结果匹配字典将作为关键字参数应用于 Python 的 time() 构造函数。否则，如果使用了位置组，则通过 `*map(int, match_obj.groups(0))` 将调用 time() 构造函数以传递位置参数。

**类签名**

class `sqlalchemy.dialects.sqlite.TIME` (`sqlalchemy.dialects.sqlite.base._DateTimeMixin`, `sqlalchemy.types.Time`)

## SQLite DML Constructs

| 对象名称 | 描述 |
| --- | --- |
| insert(table) | 构造一个特定于 SQLite 的变体 `Insert` 构造。 |
| Insert | SQLite 的 INSERT 的特定实现。 |

```py
function sqlalchemy.dialects.sqlite.insert(table: _DMLTableArgument) → Insert
```

构造一个特定于 SQLite 的变体 `Insert` 构造。

`sqlalchemy.dialects.sqlite.insert()` 函数创建 `sqlalchemy.dialects.sqlite.Insert`。该类基于方言无关的 `Insert` 结构，可以使用 SQLAlchemy Core 中的 `insert()` 函数构造。

`Insert` 结构包括额外的方法 `Insert.on_conflict_do_update()`、`Insert.on_conflict_do_nothing()`。

```py
class sqlalchemy.dialects.sqlite.Insert
```

SQLite 特定的 INSERT 实现。

添加了针对 SQLite 特定语法的方法，如 ON CONFLICT。

`Insert` 对象是通过 `sqlalchemy.dialects.sqlite.insert()` 函数创建的。

在 1.4 版中新增。

另请参阅

INSERT…ON CONFLICT（插入或替换）

**成员**

excluded、inherit_cache、on_conflict_do_nothing()、on_conflict_do_update()

**类签名**

class `sqlalchemy.dialects.sqlite.Insert`（`sqlalchemy.sql.expression.Insert`）

```py
attribute excluded
```

为 ON CONFLICT 语句提供 `excluded` 命名空间。

SQLite 的 ON CONFLICT 子句允许引用将要插入的行，称为 `excluded`。此属性提供了对此行中的所有列的引用。

提示

`Insert.excluded` 属性是 `ColumnCollection` 的一个实例，它提供与访问表和列描述的 `Table.c` 集合相同的接口。通过这个集合，普通名称可以像属性一样访问（例如 `stmt.excluded.some_column`），但特殊名称和字典方法名称应使用索引访问，例如 `stmt.excluded["column name"]` 或 `stmt.excluded["values"]`。有关更多示例，请参阅 `ColumnCollection` 的文档字符串。

```py
attribute inherit_cache: bool | None = False
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存密钥生成方案。

此属性默认为 `None`，表示构造尚未考虑是否适合参与缓存；这在功能上相当于将值设置为 `False`，但还会发出警告。

如果与此类本地属性（而不是其超类）无关，则可以在特定类上设置此标志为 `True`，则与对象对应的 SQL 不会根据这个类的属性而改变。

另请参阅

为自定义结构启用缓存支持 - 设置`HasCacheKey.inherit_cache` 属性的通用指南，用于第三方或用户定义的 SQL 结构。

```py
method on_conflict_do_nothing(index_elements: _OnConflictIndexElementsT = None, index_where: _OnConflictIndexWhereT = None) → Self
```

指定了 ON CONFLICT 子句的 DO NOTHING 操作。

参数：

+   `index_elements` – 由字符串列名、`Column` 对象或其他列表达式对象组成的序列，将用于推断目标索引或唯一约束。

+   `index_where` – 用于推断条件目标索引的额外 WHERE 条件。

```py
method on_conflict_do_update(index_elements: _OnConflictIndexElementsT = None, index_where: _OnConflictIndexWhereT = None, set_: _OnConflictSetT = None, where: _OnConflictWhereT = None) → Self
```

指定了 ON CONFLICT 子句的 DO UPDATE SET 操作。

参数：

+   `index_elements` – 由字符串列名、`Column` 对象或其他列表达式对象组成的序列，将用于推断目标索引或唯一约束。

+   `index_where` – 用于推断条件目标索引的额外 WHERE 条件。

+   `set_` –

    一个字典或其他映射对象，其中键是目标表中的列名称，或者是 `Column` 对象或其他 ORM 映射的列，匹配目标表的列，值是表达式或文字，指定要采取的 `SET` 操作。

    从版本 1.4 开始：`Insert.on_conflict_do_update.set_` 参数支持目标 `Table` 中的 `Column` 对象作为键。

    警告

    此字典**不**考虑 Python 指定的默认 UPDATE 值或生成函数，例如使用 `Column.onupdate` 指定的值。除非在 `Insert.on_conflict_do_update.set_` 字典中手动指定，否则这些值将不会用于 ON CONFLICT 类型的 UPDATE。

+   `where` – 可选参数。如果存在，则可以是一个文字 SQL 字符串或一个可接受的 `WHERE` 子句表达式，用于限制受 `DO UPDATE SET` 影响的行。不满足 `WHERE` 条件的行将不会更新（对于这些行实际上是 `DO NOTHING`）。

## Pysqlite

通过 pysqlite 驱动程序支持 SQLite 数据库。

请注意，`pysqlite` 与 Python 发行版中包含的 `sqlite3` 模块是相同的驱动程序。

### DBAPI

pysqlite 的文档和下载信息（如果适用）可在此处找到：[`docs.python.org/library/sqlite3.html`](https://docs.python.org/library/sqlite3.html)

### 连接

连接字符串：

```py
sqlite+pysqlite:///file_path
```

### 驱动程序

在所有现代 Python 版本上，`sqlite3` Python DBAPI 都是标准的；对于 cPython 和 Pypy，不需要额外安装。

### 连接字符串

SQLite 数据库的文件规范被视为 URL 的 “数据库” 部分。请注意，SQLAlchemy URL 的格式为：

```py
driver://user:pass@host/database
```

这意味着要使用的实际文件名从第三个斜杠的**右边**开始。因此，连接到相对文件路径看起来像：

```py
# relative path
e = create_engine('sqlite:///path/to/database.db')
```

绝对路径，以斜杠开头表示，意味着您需要**四个**斜杠：

```py
# absolute path
e = create_engine('sqlite:////path/to/database.db')
```

要使用 Windows 路径，可以使用常规的驱动器规范和反斜杠。可能需要双反斜杠：

```py
# absolute path on Windows
e = create_engine('sqlite:///C:\\path\\to\\database.db')
```

要使用 sqlite `:memory:` 数据库，请将其指定为使用 `sqlite://:memory:` 的文件名。如果没有文件路径，指定只有 `sqlite://` 而没有其他内容：

```py
# in-memory database
e = create_engine('sqlite://:memory:')
# also in-memory database
e2 = create_engine('sqlite://')
```

#### URI 连接

现代版本的 SQLite 支持使用[驱动级 URI](https://www.sqlite.org/uri.html)进行连接的另一种系统，其优势在于可以传递附加的驱动级参数，包括诸如“只读”之类的选项。Python 的 sqlite3 驱动在现代 Python 3 版本下支持此模式。SQLAlchemy 的 pysqlite 驱动通过在 URL 查询字符串中指定“uri=true”来支持此使用模式。SQLite 级别的“URI”被保留为 SQLAlchemy URL 的“database”部分（即在斜杠后面）：

```py
e = create_engine("sqlite:///file:path/to/database?mode=ro&uri=true")
```

注意

“uri=true”参数必须出现在 URL 的**查询字符串**中。如果仅出现在`create_engine.connect_args`参数字典中，则目前不会按预期工作。

该逻辑通过分离属于 Python sqlite3 驱动程序和属于 SQLite URI 的参数来协调 SQLAlchemy 查询字符串和 SQLite 查询字符串的同时存在。这是通过使用已知被 Python 驱动程序的固定参数列表来实现的。例如，要包含指示 Python sqlite3“timeout”和“check_same_thread”参数以及 SQLite“mode”和“nolock”参数的 URL，它们都可以一起传递到查询字符串中：

```py
e = create_engine(
    "sqlite:///file:path/to/database?"
    "check_same_thread=true&timeout=10&mode=ro&nolock=1&uri=true"
)
```

如上，pysqlite / sqlite3 DBAPI 将传递参数为：

```py
sqlite3.connect(
    "file:path/to/database?mode=ro&nolock=1",
    check_same_thread=True, timeout=10, uri=True
)
```

关于将来添加到 Python 或本机驱动程序的参数。 SQLite URI 方案中添加的新参数名称应该会自动适应此方案。可以通过在`create_engine.connect_args`字典中指定它们来适应 Python 驱动程序端添加的新参数名称，直到 SQLAlchemy 添加了方言支持为止。对于本机 SQLite 驱动程序添加的新参数名称与现有的已知 Python 驱动程序参数之一（例如“timeout”）重叠的不太可能的情况，SQLAlchemy 的方言将需要调整 URL 方案以继续支持此参数。

对于所有 SQLAlchemy 方言，可以通过`create_engine()`的`create_engine.creator`参数绕过整个“URL”过程，该参数允许创建直接创建 Python sqlite3 驱动级连接的自定义可调用函数。

新版本中新增。

另请参见

[统一资源标识符](https://www.sqlite.org/uri.html) - SQLite 文档中的正则表达式支持  ### 正则表达式支持

新版本中新增。

使用 Python 的 [re.search](https://docs.python.org/3/library/re.html#re.search) 函数提供了对 `ColumnOperators.regexp_match()` 运算符的支持。SQLite 本身不包括可用的正则表达式运算符；相反，它包括一个未实现的占位符运算符 `REGEXP`，调用必须提供的用户定义函数。

SQLAlchemy 的实现使用 pysqlite 的 [create_function](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function) 钩子，如下所示：

```py
def regexp(a, b):
    return re.search(a, b) is not None

sqlite_connection.create_function(
    "regexp", 2, regexp,
)
```

目前不支持将正则表达式标志作为单独参数，因为这些标志不受 SQLite 的 REGEXP 运算符支持，但可以在正则表达式字符串内联包含。详见[Python 正则表达式](https://docs.python.org/3/library/re.html#re.search)。

另请参见

[Python 正则表达式](https://docs.python.org/3/library/re.html#re.search)：Python 正则表达式语法的文档。

### 与 sqlite3 “本地”日期和日期时间类型兼容

pysqlite 驱动程序包括 sqlite3.PARSE_DECLTYPES 和 sqlite3.PARSE_COLNAMES 选项，这些选项的效果是任何明确转换为“date”或“timestamp”的列或表达式将转换为 Python 日期或日期时间对象。pysqlite 方言提供的日期和日期时间类型目前与这些选项不兼容，因为它们呈现 ISO 日期/日期时间，包括微秒，而 pysqlite 的驱动程序不包括。此外，SQLAlchemy 目前不会自动呈现“cast”语法，以使自由函数“current_timestamp”和“current_date”返回原生的 datetime/date 类型。不幸的是，pysqlite 不提供 `cursor.description` 中的标准 DBAPI 类型，使得 SQLAlchemy 无法在不进行昂贵的每行类型检查的情况下动态检测这些类型。

请注意，不推荐使用 pysqlite 的解析选项，也不应该使用 SQLAlchemy，如果配置了 "native_datetime=True" 在 create_engine() 上，可以强制使用 PARSE_DECLTYPES。

```py
engine = create_engine('sqlite://',
    connect_args={'detect_types':
        sqlite3.PARSE_DECLTYPES|sqlite3.PARSE_COLNAMES},
    native_datetime=True
)
```

启用此标志后，DATE 和 TIMESTAMP 类型（但请注意 - 不是 DATETIME 或 TIME 类型...还困惑吗？）将不执行任何绑定参数或结果处理。执行 “func.current_date()” 将返回一个字符串。在 SQLAlchemy 中，“func.current_timestamp()” 被注册为返回 DATETIME 类型，因此此函数仍接收 SQLAlchemy 级别的结果处理。

### 线程/池行为

默认情况下，`sqlite3` DBAPI 禁止在创建它的线程之外的线程中使用特定连接。随着 SQLite 的成熟，它在多线程下的行为已经改进，甚至包括选项，使得内存数据库可以在多个线程中使用。

线程禁止被称为“检查同一线程”，可以使用`sqlite3`参数`check_same_thread`进行控制，该参数将禁用或启用此检查。在使用基于文件的数据库时，SQLAlchemy 的默认行为是自动将`check_same_thread`设置为`False`，以确立与默认池类`QueuePool`的兼容性。

SQLAlchemy 的 `pysqlite` DBAPI 根据请求的 SQLite 数据库的类型以不同的方式建立连接池：

+   当指定`:memory:` SQLite 数据库时，默认情况下方言将使用`SingletonThreadPool`。此池在每个线程中维护单个连接，因此当前线程内对引擎的所有访问都使用相同的`:memory:`数据库 - 其他线程将访问不同的`:memory:`数据库。`check_same_thread`参数默认为`True`。

+   当指定基于文件的数据库时，方言将使用`QueuePool`作为连接的源。同时，默认情况下将`check_same_thread`标志设置为`False`，除非被覆盖。

    自 2.0 版本更改：SQLite 文件数据库引擎现在默认使用`QueuePool`。以前使用的是`NullPool`。可以通过`create_engine.poolclass`参数指定使用`NullPool`类。

#### 禁用文件数据库的连接池

可以通过为`poolclass()`参数指定`NullPool`实现来禁用基于文件的数据库的连接池：

```py
from sqlalchemy import NullPool
engine = create_engine("sqlite:///myfile.db", poolclass=NullPool)
```

当使用`NullPool`实现时，由于`QueuePool`未实现连接重用，因此对于重复检出，`NullPool`实现会产生极小的性能开销。然而，如果应用程序遇到文件被锁定的问题，仍然可能有利于使用此类。

#### 在多个线程中使用内存数据库

要在多线程场景中使用 `:memory:` 数据库，必须在线程之间共享同一个连接对象，因为数据库仅存在于该连接的范围内。 `StaticPool` 实现将全局维护单个连接，并且可以将 `check_same_thread` 标志传递给 Pysqlite 为 `False`。

```py
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite://',
                    connect_args={'check_same_thread':False},
                    poolclass=StaticPool)
```

请注意，要在多个线程中使用 `:memory:` 数据库，需要使用最近版本的 SQLite。

#### 使用 SQLite 临时表

由于 SQLite 处理临时表的方式，如果希望在基于文件的 SQLite 数据库中跨多个连接池检出使用临时表（例如在使用 ORM `Session` 时，临时表应在 `Session.commit()` 或 `Session.rollback()` 调用后继续存在），则必须使用维护单个连接的池。如果范围仅在当前线程内，则使用 `SingletonThreadPool`，如果此情况需要范围在多个线程内，则使用 `StaticPool`：

```py
# maintain the same connection per thread
from sqlalchemy.pool import SingletonThreadPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=SingletonThreadPool)

# maintain the same connection across all threads
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=StaticPool)
```

请注意，`SingletonThreadPool` 应配置为要使用的线程数；超出该数量的连接将以不确定的方式关闭。

### 处理混合字符串/二进制列

SQLite 数据库是弱类型的，因此当使用二进制值（在 Python 中表示为 `b'some string'`）时，可能发生以下情况，即特定的 SQLite 数据库可以在不同行中返回数据值，其中某些值将由 Pysqlite 驱动程序返回为 `b''` 值，而其他值将作为 Python 字符串返回，例如 `''` 值。如果始终一致使用 SQLAlchemy 的 `LargeBinary` 数据类型，则不知道是否会发生此情况；但是如果特定的 SQLite 数据库具有使用 Pysqlite 驱动程序直接插入的数据，或者在使用后更改为 `LargeBinary` 的 SQLAlchemy `String` 类型时，该表将无法一致地读取，因为 SQLAlchemy 的 `LargeBinary` 数据类型不处理字符串，因此无法“编码”字符串格式的值。

要处理具有相同列中的混合字符串/二进制数据的 SQLite 表，请使用一个将逐个检查每行的自定义类型：

```py
from sqlalchemy import String
from sqlalchemy import TypeDecorator

class MixedBinary(TypeDecorator):
    impl = String
    cache_ok = True

    def process_result_value(self, value, dialect):
        if isinstance(value, str):
            value = bytes(value, 'utf-8')
        elif value is not None:
            value = bytes(value)

        return value
```

然后在通常会使用`LargeBinary`的地方使用上述`MixedBinary`数据类型。

### 可序列化隔离/保存点/事务 DDL

在数据库锁定行为/并发性部分，我们提到 pysqlite 驱动程序的一系列问题，这些问题阻止 SQLite 的几个功能正常工作。 pysqlite DBAPI 驱动程序有几个长期存在的错误，影响其事务行为的正确性。在其默认操作模式下，SQLite 的功能，如可序列化隔离、事务 DDL 和 SAVEPOINT 支持是不起作用的，为了使用这些功能，必须采取解决方法。

问题实质上是驱动程序试图猜测用户意图，未能启动事务，有时会过早结束事务，以减少 SQLite 数据库的文件锁定行为，尽管 SQLite 本身对只读活动使用“共享”锁。

SQLAlchemy 选择默认情况下不更改此行为，因为这是 pysqlite 驱动程序的长期预期行为；如果 pysqlite 驱动程序尝试修复这些问��，那将更多地推动 SQLAlchemy 的默认值。

好消息是，通过几个事件，我们可以完全实现事务支持，通过完全禁用 pysqlite 的功能并自己发出 BEGIN。这是通过使用两个事件监听器实现的：

```py
from sqlalchemy import create_engine, event

engine = create_engine("sqlite:///myfile.db")

@event.listens_for(engine, "connect")
def do_connect(dbapi_connection, connection_record):
    # disable pysqlite's emitting of the BEGIN statement entirely.
    # also stops it from emitting COMMIT before any DDL.
    dbapi_connection.isolation_level = None

@event.listens_for(engine, "begin")
def do_begin(conn):
    # emit our own BEGIN
    conn.exec_driver_sql("BEGIN")
```

警告

在使用上述配方时，建议不要在 SQLite 驱动程序上使用`Connection.execution_options.isolation_level`设置`Connection`和`create_engine()`，因为此函数必然也会改变“.isolation_level”设置。

在上面，我们拦截一个新的 pysqlite 连接并禁用任何事务集成。然后，在 SQLAlchemy 知道事务范围将开始的时候，我们自己发出`"BEGIN"`。

当我们控制`"BEGIN"`时，我们还可以直接控制 SQLite 的锁定模式，通过将所需的锁定模式添加到我们的`"BEGIN"`中引入的[开始事务](https://sqlite.org/lang_transaction.html)：

```py
@event.listens_for(engine, "begin")
def do_begin(conn):
    conn.exec_driver_sql("BEGIN EXCLUSIVE")
```

另请参阅

[开始事务](https://sqlite.org/lang_transaction.html) - 在 SQLite 网站上

[sqlite3 SELECT 不会 BEGIN 事务](https://bugs.python.org/issue9924) - 在 Python 错误跟踪器上

[sqlite3 模块中断事务并可能损坏数据](https://bugs.python.org/issue10740) - 在 Python 错误跟踪器上  ### 用户定义的函数

pysqlite 支持一个 [create_function()](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function) 方法，允许我们在 Python 中创建自己的用户定义的函数 (UDFs)，并直接在 SQLite 查询中使用它们。这些函数已与特定的 DBAPI 连接注册。

SQLAlchemy 使用基于文件的 SQLite 数据库的连接池，因此我们需要确保在创建连接时将 UDF 附加到连接。这通过事件监听器实现：

```py
from sqlalchemy import create_engine
from sqlalchemy import event
from sqlalchemy import text

def udf():
    return "udf-ok"

engine = create_engine("sqlite:///./db_file")

@event.listens_for(engine, "connect")
def connect(conn, rec):
    conn.create_function("udf", 0, udf)

for i in range(5):
    with engine.connect() as conn:
        print(conn.scalar(text("SELECT UDF()")))
```  ## Aiosqlite

通过 aiosqlite 驱动程序支持 SQLite 数据库。

### DBAPI

aiosqlite 的文档和下载信息（如果适用）可在此处获得：[`pypi.org/project/aiosqlite/`](https://pypi.org/project/aiosqlite/)

### 连接

连接字符串：

```py
sqlite+aiosqlite:///file_path
```

aiosqlite 方言提供了对运行在 pysqlite 之上的 SQLAlchemy asyncio 接口的支持。

aiosqlite 是对 pysqlite 的封装，每个连接使用一个后台线程。它实际上不使用非阻塞 IO，因为 SQLite 数据库不是基于套接字的。但是它提供了一个可用于测试和原型设计的工作 asyncio 接口。

使用特殊的 asyncio 中介层，aiosqlite 方言可作为 SQLAlchemy asyncio 扩展包的后端使用。

通常应使用 `create_async_engine()` 引擎创建函数创建此方言：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("sqlite+aiosqlite:///filename")
```

URL 通过所有参数传递给 `pysqlite` 驱动程序，因此所有连接参数与 Pysqlite 的相同。

### 用户定义的函数

aiosqlite 扩展了 pysqlite 以支持异步，因此我们可以在 Python 中创建自定义用户定义的函数 (UDFs)，并直接在 SQLite 查询中使用它们，如此处所述：用户定义的函数。### Serializable isolation / Savepoints / Transactional DDL (asyncio 版本)

类似于 pysqlite，aiosqlite 不支持 SAVEPOINT 功能。

解决方案类似于 Serializable isolation / Savepoints / Transactional DDL。这是通过 async 中的事件监听器实现的：

```py
from sqlalchemy import create_engine, event
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine("sqlite+aiosqlite:///myfile.db")

@event.listens_for(engine.sync_engine, "connect")
def do_connect(dbapi_connection, connection_record):
    # disable aiosqlite's emitting of the BEGIN statement entirely.
    # also stops it from emitting COMMIT before any DDL.
    dbapi_connection.isolation_level = None

@event.listens_for(engine.sync_engine, "begin")
def do_begin(conn):
    # emit our own BEGIN
    conn.exec_driver_sql("BEGIN")
```

警告

在使用以上方案时，建议不要在 SQLite 驱动程序上使用 `Connection.execution_options.isolation_level` 设置 `Connection` 和 `create_engine()` ，因为该函数必然也会改变 “.isolation_level” 设置。## Pysqlcipher

通过 pysqlcipher 驱动程序支持 SQLite 数据库。

为支持利用 [SQLCipher](https://www.zetetic.net/sqlcipher) 后端的 DBAPI 提供方言。

### 连接中

连接字符串：

```py
sqlite+pysqlcipher://:passphrase@/file_path[?kdf_iter=<iter>]
```

### 驱动程序

当前的方言选择逻辑是：

+   如果 `create_engine.module` 参数提供了一个 DBAPI 模块，则使用该模块。

+   否则对于 Python 3，选择[`pypi.org/project/sqlcipher3/`](https://pypi.org/project/sqlcipher3/)

+   如果不可用，回退到[`pypi.org/project/pysqlcipher3/`](https://pypi.org/project/pysqlcipher3/)

+   对于 Python 2，使用[`pypi.org/project/pysqlcipher/`](https://pypi.org/project/pysqlcipher/)。

警告

截止到目前为止，`pysqlcipher3` 和 `pysqlcipher` DBAPI 驱动程序不再维护；`sqlcipher3` 驱动程序似乎是当前的。为了未来的兼容性，可以使用任何兼容 pysqlcipher 的 DBAPI 如下所示：

```py
import sqlcipher_compatible_driver

from sqlalchemy import create_engine

e = create_engine(
    "sqlite+pysqlcipher://:password@/dbname.db",
    module=sqlcipher_compatible_driver
)
```

这些驱动程序利用了 SQLCipher 引擎。该系统基本上引入了新的 PRAGMA 命令到 SQLite，这允许设置密码和其他加密参数，从而允许加密数据库文件。

### 连接字符串

连接字符串的格式在每个方面与 `pysqlite` 驱动程序的格式相同，除了现在接受“密码”字段，该字段应包含一个密码：

```py
e = create_engine('sqlite+pysqlcipher://:testing@/foo.db')
```

对于绝对文件路径，应该使用两个前导斜杠作为数据库名：

```py
e = create_engine('sqlite+pysqlcipher://:testing@//path/to/foo.db')
```

可以通过查询字符串传递一系列由 SQLCipher 支持的附加加密相关的 PRAGMA，如 [`www.zetetic.net/sqlcipher/sqlcipher-api/`](https://www.zetetic.net/sqlcipher/sqlcipher-api/) 中所述，并且将导致每个新连接调用该 PRAGMA。目前，支持 `cipher`、`kdf_iter`、`cipher_page_size` 和 `cipher_use_hmac`：

```py
e = create_engine('sqlite+pysqlcipher://:testing@/foo.db?cipher=aes-256-cfb&kdf_iter=64000')
```

警告

先前版本的 sqlalchemy 没有考虑到在 url 字符串中传递的与加密相关的 PRAGMA，这些 PRAGMA 被悄悄地忽略了。如果加密选项不匹配，这可能会导致打开由之前的 sqlalchemy 版本保存的文件时出错。

### 池行为

驱动程序对 pysqlite 的默认池行为进行了更改，如线程/池行为中所述。观察到 pysqlcipher 驱动程序连接速度比 pysqlite 驱动程序慢得多，很可能是由于加密开销，因此该方言在这里默认使用`SingletonThreadPool`实现，而不是 pysqlite 使用的`NullPool`池。与往常一样，可以使用`create_engine.poolclass`参数完全配置池实现；`StaticPool`可能更适合单线程使用，或者可以使用`NullPool`来防止未加密的连接被长时间保持打开，但新连接的启动时间较慢。

支持 SQLite 数据库。

以下表总结了当前数据库发布版本的支持水平。

**支持的 SQLite 版本**

| 支持类型 | 版本 |
| --- | --- |
| 在 CI 中完全测试 | 3.36.0 |
| 常规支持 | 3.12+ |
| 尽力而为 | 3.7.16+ |

## DBAPI 支持

可用以下方言/DBAPI 选项。请参考各个 DBAPI 部分以获取连接信息。

+   pysqlite

+   aiosqlite

+   pysqlcipher

## 日期和时间类型

SQLite 没有内置的 DATE、TIME 或 DATETIME 类型，而 pysqlite 也没有提供将值在 Python datetime 对象和 SQLite 支持的格式之间转换的开箱即用功能。当使用 SQLite 时，SQLAlchemy 自己的`DateTime`和相关类型提供日期格式化和解析功能。实现类是`DATETIME`、`DATE`和`TIME`。这些类型将日期和时间表示为 ISO 格式的字符串，这也很好地支持排序。这些函数不依赖于典型的“libc”内部，因此完全支持历史日期。

### 确保文本亲和性

这些类型的 DDL 呈现是标准的 `DATE`、`TIME` 和 `DATETIME` 指示符。然而，这些类型也可以应用自定义的存储格式。当检测到存储格式不包含任何字母字符时，这些类型的 DDL 将呈现为 `DATE_CHAR`、`TIME_CHAR` 和 `DATETIME_CHAR`，以便列继续具有文本亲和性。

参见

[类型亲和性](https://www.sqlite.org/datatype3.html#affinity) - SQLite 文档中的内容

### 确保文本亲和性

这些类型的 DDL 呈现是标准的 `DATE`、`TIME` 和 `DATETIME` 指示符。然而，这些类型也可以应用自定义的存储格式。当检测到存储格式不包含任何字母字符时，这些类型的 DDL 将呈现为 `DATE_CHAR`、`TIME_CHAR` 和 `DATETIME_CHAR`，以便列继续具有文本亲和性。

参见

[类型亲和性](https://www.sqlite.org/datatype3.html#affinity) - SQLite 文档中的内容

## SQLite 自动增量行为

关于 SQLite 的自动增量的背景信息请参阅：[`sqlite.org/autoinc.html`](https://sqlite.org/autoinc.html)

关键概念：

+   SQLite 具有隐式的“自动增量”功能，适用于任何使用“INTEGER PRIMARY KEY”来明确创建的非复合主键列。

+   SQLite 还具有显式的 “AUTOINCREMENT” 关键字，这与隐式自动增量功能 **不** 等同；不建议一般使用这个关键字。SQLAlchemy 不会呈现此关键字，除非使用特殊的特定于 SQLite 的指令（见下文）。但是，它仍然要求列的类型被命名为 “INTEGER”。

### 使用 AUTOINCREMENT 关键字

要在渲染 DDL 时在主键列上具体呈现 AUTOINCREMENT 关键字，请将 `sqlite_autoincrement=True` 标志添加到 Table 构造函数中：

```py
Table('sometable', metadata,
        Column('id', Integer, primary_key=True),
        sqlite_autoincrement=True)
```

### 允许除 Integer/INTEGER 之外的 SQLAlchemy 类型具有自动增量行为

SQLite 的类型模型基于命名约定。这意味着包含子字符串 `"INT"` 的任何类型名称都将被确定为“整数亲和性”。一个名为 `"BIGINT"`、`"SPECIAL_INT"` 或甚至 `"XYZINTQPR"` 的类型都将被 SQLite 视为“整数”亲和性。然而，**无论隐式还是显式启用了 SQLite 的自动增量功能，列类型的名称都必须正好是字符串 `"INTEGER"`**。因此，如果应用程序使用 `BigInteger` 作为主键的类型，在 SQLite 上，当在发出初始的 `CREATE TABLE` 语句时，这个类型将需要被渲染为名称 `"INTEGER"`，以便自动增量行为可用。

实现此目标的一种方法是仅在 SQLite 上使用 `TypeEngine.with_variant()` 使用 `Integer`:

```py
table = Table(
    "my_table", metadata,
    Column("id", BigInteger().with_variant(Integer, "sqlite"), primary_key=True)
)
```

另一种方法是使用 `BigInteger` 的子类，当编译针对 SQLite 时，重写其 DDL 名称为 `INTEGER`：

```py
from sqlalchemy import BigInteger
from sqlalchemy.ext.compiler import compiles

class SLBigInteger(BigInteger):
    pass

@compiles(SLBigInteger, 'sqlite')
def bi_c(element, compiler, **kw):
    return "INTEGER"

@compiles(SLBigInteger)
def bi_c(element, compiler, **kw):
    return compiler.visit_BIGINT(element, **kw)

table = Table(
    "my_table", metadata,
    Column("id", SLBigInteger(), primary_key=True)
)
```

另请参阅

`TypeEngine.with_variant()`

自定义 SQL 构造和编译扩展

[SQLite 版本 3 中的数据类型](https://sqlite.org/datatype3.html)

### 使用 AUTOINCREMENT 关键字

要在渲染 DDL 时特别呈现主键列上的 AUTOINCREMENT 关键字，请向 Table 构造添加标志 `sqlite_autoincrement=True`：

```py
Table('sometable', metadata,
        Column('id', Integer, primary_key=True),
        sqlite_autoincrement=True)
```

### 允许除 Integer/INTEGER 外的 SQLAlchemy 类型具有自增行为

SQLite 的类型模型基于命名约定。除其他外，这意味着包含子字符串 `"INT"` 的任何类型名称都将被确定为“整数亲和性”。类型名称为 `"BIGINT"`、`"SPECIAL_INT"` 甚至 `"XYZINTQPR"` 的类型，SQLite 都会将其视为“整数”亲和性。然而，**无论是隐式还是显式启用的 SQLite 自增特性，都要求列的类型名称正好是字符串`"INTEGER"`**。因此，如果应用程序使用类似 `BigInteger` 的类型作为主键，在 SQLite 上，此类型在发出初始的 `CREATE TABLE` 语句时需要呈现为名称 `"INTEGER"`，以便自增行为可用。

实现此目标的一种方法是仅在 SQLite 上使用 `TypeEngine.with_variant()` 使用 `Integer`:

```py
table = Table(
    "my_table", metadata,
    Column("id", BigInteger().with_variant(Integer, "sqlite"), primary_key=True)
)
```

另一种方法是使用 `BigInteger` 的子类，当编译针对 SQLite 时，重写其 DDL 名称为 `INTEGER`：

```py
from sqlalchemy import BigInteger
from sqlalchemy.ext.compiler import compiles

class SLBigInteger(BigInteger):
    pass

@compiles(SLBigInteger, 'sqlite')
def bi_c(element, compiler, **kw):
    return "INTEGER"

@compiles(SLBigInteger)
def bi_c(element, compiler, **kw):
    return compiler.visit_BIGINT(element, **kw)

table = Table(
    "my_table", metadata,
    Column("id", SLBigInteger(), primary_key=True)
)
```

另请参阅

`TypeEngine.with_variant()`

自定义 SQL 构造和编译扩展

[SQLite 版本 3 中的数据类型](https://sqlite.org/datatype3.html)

## 数据库锁定行为 / 并发

SQLite 并不适用于高度写并发性。数据库本身，作为一个文件，在事务内的写操作期间完全被锁定，这意味着在此期间仅有一个“连接”（实际上是一个文件句柄）对数据库具有独占访问权限 - 在此期间所有其他“连接”都将被阻塞。

Python DBAPI 规范还要求一个始终处于事务中的连接模型；没有 `connection.begin()` 方法，只有 `connection.commit()` 和 `connection.rollback()`，在这之后立即开始一个新事务。这似乎暗示着 SQLite 驱动理论上只允许在任何时候对特定数据库文件进行单个文件句柄的访问；然而，SQLite 本身以及 pysqlite 驱动程序中有几个因素大大放宽了这个限制。

然而，无论使用什么锁定模式，一旦启动事务并且已经发出了 DML（例如 INSERT、UPDATE、DELETE），SQLite 都会锁定数据库文件，这将至少在其他事务也试图发出 DML 的时候阻塞其他事务。默认情况下，在此阻塞的时间长度非常短，超时后会出现错误。

当与 SQLAlchemy ORM 结合使用时，这种行为变得更加关键。SQLAlchemy 的 `Session` 对象默认在事务内运行，并且使用其自动刷新模型，可能会在任何 SELECT 语句之前发出 DML。这可能导致 SQLite 数据库比预期更快地锁定。SQLite 和 pysqlite 驱动程序的锁定模式可以在一定程度上被操纵，但应注意，要想在 SQLite 中实现高度的写并发性是一场失败的战斗。

欲了解 SQLite 的设计缺乏写并发性的更多信息，请参阅[在哪些情况下另一个 RDBMS 可能更适合使用 - 高并发性](https://www.sqlite.org/whentouse.html)，页面底部。

以下子节介绍了受 SQLite 的基于文件的架构影响的领域，此外，通常在使用 pysqlite 驱动程序时需要一些解决方法。

## 事务隔离级别 / 自动提交

SQLite 以一种非标准的方式支持“事务隔离”，沿着两个轴线。一个是[PRAGMA read_uncommitted](https://www.sqlite.org/pragma.html#pragma_read_uncommitted) 指令。这个设置可以基本上在 SQLite 的默认模式 `SERIALIZABLE` 隔离和一个通常称为 `READ UNCOMMITTED` 的“脏读”隔离模式之间切换。

SQLAlchemy 使用`create_engine.isolation_level`参数来连接到此 PRAGMA 语句`create_engine()`。在与 SQLite 一起使用此参数的有效值是`"SERIALIZABLE"`和`"READ UNCOMMITTED"`，分别对应于 0 和 1 的值。SQLite 默认为`SERIALIZABLE`，但其行为受到 pysqlite 驱动程序的默认行为的影响。

使用 pysqlite 驱动程序时，还可以使用`"AUTOCOMMIT"`隔离级别，该级别将通过 DBAPI 连接的`.isolation_level`属性更改 pysqlite 连接，并将其设置为 None 以进行设置的持续时间。

版本 1.3.16 中的新功能：在使用 pysqlite/sqlite3 SQLite 驱动程序时增加了对 SQLite AUTOCOMMIT 隔离级别的支持。

SQLite 的事务锁定受影响的另一个轴是通过使用的`BEGIN`语句的性质。这三种类型是“延迟”、“立即”和“独占”，如[开始事务](https://sqlite.org/lang_transaction.html)所述。直接的`BEGIN`语句使用“延迟”模式，在第一次读取或写入操作之前不锁定数据库文件，并且读取访问在第一次写入操作之前仍然对其他事务开放。但需要再次强调的是，pysqlite 驱动器通过*甚至不发出 BEGIN*直到第一次写入操作来干扰此行为。

警告

SQLite 的事务范围受到 pysqlite 驱动程序中未解决的问题的影响，该问题将 BEGIN 语句推迟到比通常可行的更大程度。有关解决此行为的技术，请参阅部分可序列化隔离/保存点/事务 DDL 或可序列化隔离/保存点/事务 DDL（asyncio 版本）。

请参阅

设置事务隔离级别，包括 DBAPI 自动提交

## INSERT/UPDATE/DELETE…RETURNING

SQLite 方言支持 SQLite 3.35 的`INSERT|UPDATE|DELETE..RETURNING`语法。在某些情况下，`INSERT..RETURNING`可能会自动使用，以获取新生成的标识符，而不是传统方法中使用`cursor.lastrowid`，但目前仍然推荐对于简单的单语句情况使用`cursor.lastrowid`，因为其性能更好。

要指定显式的`RETURNING`子句，请在每个语句上使用`_UpdateBase.returning()`方法：

```py
# INSERT..RETURNING
result = connection.execute(
    table.insert().
    values(name='foo').
    returning(table.c.col1, table.c.col2)
)
print(result.all())

# UPDATE..RETURNING
result = connection.execute(
    table.update().
    where(table.c.name=='foo').
    values(name='bar').
    returning(table.c.col1, table.c.col2)
)
print(result.all())

# DELETE..RETURNING
result = connection.execute(
    table.delete().
    where(table.c.name=='foo').
    returning(table.c.col1, table.c.col2)
)
print(result.all())
```

版本 2.0 中的新功能：增加了对 SQLite RETURNING 的支持

## 保存点支持

SQLite 支持 SAVEPOINT，仅在启动事务后才能运行。SQLAlchemy 的 SAVEPOINT 支持可在 Core 级别使用 `Connection.begin_nested()` 方法，在 ORM 级别使用 `Session.begin_nested()`。但是，除非采取解决方法，否则 SAVEPOINT 在 pysqlite 中将无法工作。

警告

pysqlite 和 aiosqlite 驱动存在未解决的问题，这些问题将 BEGIN 语句推迟到一个更大程度上比通常可行的程度。有关绕过此行为的技术，请参见 Serializable isolation / Savepoints / Transactional DDL 和 Serializable isolation / Savepoints / Transactional DDL (asyncio version) 部分。

## 事务性 DDL

SQLite 数据库还支持事务性 DDL。在这种情况下，pysqlite 驱动不仅在检测到 DDL 时无法启动事务，还会结束任何现有事务，因此需要采取解决方法。

警告

pysqlite 驱动中存在未解决的问题影响了 SQLite 的事务性 DDL，当遇到 DDL 时，该驱动器未发出 BEGIN 并且还强制执行 COMMIT 来取消任何事务。有关绕过此行为的技术，请参见 Serializable isolation / Savepoints / Transactional DDL 部分。

## 外键支持

当发出用于表的 CREATE 语句时，SQLite 支持 FOREIGN KEY 语法，但是默认情况下，这些约束对表的操作没有任何影响。

在 SQLite 上进行约束检查有三个先决条件：

+   必须使用至少版本 3.6.19 的 SQLite

+   必须在编译 SQLite 库时 *没有* 启用 SQLITE_OMIT_FOREIGN_KEY 或 SQLITE_OMIT_TRIGGER 符号。

+   必须在所有连接上发出 `PRAGMA foreign_keys = ON` 语句，包括对 `MetaData.create_all()` 的初始调用。

SQLAlchemy 允许通过事件的使用自动发出 `PRAGMA` 语句以进行新连接：

```py
from sqlalchemy.engine import Engine
from sqlalchemy import event

@event.listens_for(Engine, "connect")
def set_sqlite_pragma(dbapi_connection, connection_record):
    cursor = dbapi_connection.cursor()
    cursor.execute("PRAGMA foreign_keys=ON")
    cursor.close()
```

警告

当启用 SQLite 外键时，**不可能**对包含相互依赖的外键约束的表发出 CREATE 或 DROP 语句；要发出这些表的 DDL，需要单独使用 ALTER TABLE 创建或删除这些约束，而 SQLite 不支持这一点。

另请参阅

[SQLite 外键支持](https://www.sqlite.org/foreignkeys.html) - 在 SQLite 网站上。

事件 - SQLAlchemy 事件 API。

通过 ALTER 创建/删除外键约束 - 有关 SQLAlchemy 处理的更多信息

互相依赖的外键约束。

## 对约束的 ON CONFLICT 支持

另请参见

本节描述了 SQLite 中“ON CONFLICT”的 DDL 版本，该版本出现在 CREATE TABLE 语句中。有关应用于 INSERT 语句的“ON CONFLICT”，请参见 INSERT…ON CONFLICT (Upsert)。

SQLite 支持一个名为 ON CONFLICT 的非标准 DDL 子句，可应用于主键、唯一、检查和非空约束。在 DDL 中，它要么在“CONSTRAINT”子句中呈现，要么在目标约束的位置取决于列定义本身。要在 DDL 中呈现此子句，可以在`PrimaryKeyConstraint`、`UniqueConstraint`、`CheckConstraint`对象中指定扩展参数`sqlite_on_conflict`，并在`Column`对象中，有单独的参数`sqlite_on_conflict_not_null`、`sqlite_on_conflict_primary_key`、`sqlite_on_conflict_unique`，分别对应于可以从`Column`对象指示的三种相关约束类型。

另请参见

[冲突时执行](https://www.sqlite.org/lang_conflict.html) - 在 SQLite 文档中

版本 1.3 中的新功能。

`sqlite_on_conflict`参数接受一个字符串参数，该参数只是要选择的解决方案名称，在 SQLite 上可以是 ROLLBACK、ABORT、FAIL、IGNORE 和 REPLACE 中的一个。例如，要添加一个指定 IGNORE 算法的唯一约束：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', Integer),
    UniqueConstraint('id', 'data', sqlite_on_conflict='IGNORE')
)
```

以上将 CREATE TABLE DDL 呈现为：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    data INTEGER,
    PRIMARY KEY (id),
    UNIQUE (id, data) ON CONFLICT IGNORE
)
```

当使用`Column.unique`标志向单个列添加唯一约束时，也可以向`Column`添加`sqlite_on_conflict_unique`参数，该参数将添加到 DDL 中的唯一约束中：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', Integer, unique=True,
           sqlite_on_conflict_unique='IGNORE')
)
```

渲染：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    data INTEGER,
    PRIMARY KEY (id),
    UNIQUE (data) ON CONFLICT IGNORE
)
```

要应用 FAIL 算法以满足 NOT NULL 约束，使用`sqlite_on_conflict_not_null`：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True),
    Column('data', Integer, nullable=False,
           sqlite_on_conflict_not_null='FAIL')
)
```

这将使列内联 ON CONFLICT 短语：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    data INTEGER NOT NULL ON CONFLICT FAIL,
    PRIMARY KEY (id)
)
```

同样，对于内联主键，使用`sqlite_on_conflict_primary_key`：

```py
some_table = Table(
    'some_table', metadata,
    Column('id', Integer, primary_key=True,
           sqlite_on_conflict_primary_key='FAIL')
)
```

SQLAlchemy 将主键约束单独呈现，因此冲突解决算法应用于约束本身：

```py
CREATE TABLE some_table (
    id INTEGER NOT NULL,
    PRIMARY KEY (id) ON CONFLICT FAIL
)
```

## 插入…冲突时执行（Upsert）

另请参见

本节描述了 SQLite 的“ON CONFLICT”的 DML 版本，它发生在 INSERT 语句中。有关应用于 CREATE TABLE 语句的“ON CONFLICT”，请参见约束的 ON CONFLICT 支持。

从版本 3.24.0 开始，SQLite 支持通过 `INSERT` 语句的 `ON CONFLICT` 子句进行行的“upserts”（更新或插入）到表中。仅当候选行不违反任何唯一或主键约束时才会插入该行。在唯一约束违反的情况下，可以发生次要操作，可以是“DO UPDATE”，表示应更新目标行中的数据，或者是“DO NOTHING”，表示默默地跳过此行。

冲突是使用现有唯一约束和索引的列确定的。这些约束通过说明组成索引的列和条件来确定。

SQLAlchemy 通过 SQLite 特定的 `insert()` 函数提供 `ON CONFLICT` 支持，该函数提供生成方法 `Insert.on_conflict_do_update()` 和 `Insert.on_conflict_do_nothing()`：

```py
>>> from sqlalchemy.dialects.sqlite import insert

>>> insert_stmt = insert(my_table).values(
...     id='some_existing_id',
...     data='inserted value')

>>> do_update_stmt = insert_stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value')
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?
>>> do_nothing_stmt = insert_stmt.on_conflict_do_nothing(
...     index_elements=['id']
... )

>>> print(do_nothing_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)
ON  CONFLICT  (id)  DO  NOTHING 
```

新版本 1.4 中新增。

另请参阅

[Upsert](https://sqlite.org/lang_UPSERT.html) - SQLite 文档中的内容。

### 指定目标

两种方法都使用列推断冲突的“目标”：

+   `Insert.on_conflict_do_update.index_elements` 参数指定包含字符串列名称、`Column` 对象和/或 SQL 表达式元素的序列，这些元素将标识唯一索引或唯一约束。

+   当使用 `Insert.on_conflict_do_update.index_elements` 来推断索引时，还可以通过指定 `Insert.on_conflict_do_update.index_where` 参数推断出部分索引：

    ```py
    >>> stmt = insert(my_table).values(user_email='a@b.com', data='inserted data')

    >>> do_update_stmt = stmt.on_conflict_do_update(
    ...     index_elements=[my_table.c.user_email],
    ...     index_where=my_table.c.user_email.like('%@gmail.com'),
    ...     set_=dict(data=stmt.excluded.data)
    ...     )

    >>> print(do_update_stmt)
    INSERT  INTO  my_table  (data,  user_email)  VALUES  (?,  ?)
    ON  CONFLICT  (user_email)
    WHERE  user_email  LIKE  '%@gmail.com'
    DO  UPDATE  SET  data  =  excluded.data 
    ```

### SET 子句

使用 `ON CONFLICT...DO UPDATE` 来执行已经存在行的更新，使用任何组合的新值以及来自所提议插入的值。这些值使用 `Insert.on_conflict_do_update.set_` 参数指定。该参数接受一个包含直接 UPDATE 值的字典：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')

>>> do_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value')
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ? 
```

警告

`Insert.on_conflict_do_update()` 方法 **不** 考虑 Python 端的默认 UPDATE 值或生成函数，例如，使用 `Column.onupdate` 指定的那些。除非在 `Insert.on_conflict_do_update.set_` 字典中手动指定，否则这些值不会在 ON CONFLICT 类型的 UPDATE 中使用。

### 使用被排除的 INSERT 值进行更新

为了引用所提议的插入行，特殊别名 `Insert.excluded` 可以作为 `Insert` 对象的属性使用；该对象在列上创建了一个 “excluded.” 前缀，它通知 DO UPDATE 使用将插入的值更新行，如果约束没有失败的话将会插入的值：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh'
... )

>>> do_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value', author=stmt.excluded.author)
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (?,  ?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?,  author  =  excluded.author 
```

### 附加的 WHERE 条件

`Insert.on_conflict_do_update()` 方法还接受使用 `Insert.on_conflict_do_update.where` 参数的 WHERE 子句，这将限制接收 UPDATE 的行：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh'
... )

>>> on_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value', author=stmt.excluded.author),
...     where=(my_table.c.status == 2)
... )
>>> print(on_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (?,  ?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?,  author  =  excluded.author
WHERE  my_table.status  =  ? 
```

### 使用 DO NOTHING 跳过行

`ON CONFLICT` 可以用来完全跳过插入行，如果任何与唯一约束发生冲突的话；下面通过使用 `Insert.on_conflict_do_nothing()` 方法进行了说明：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')
>>> stmt = stmt.on_conflict_do_nothing(index_elements=['id'])
>>> print(stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)  ON  CONFLICT  (id)  DO  NOTHING 
```

如果使用 `DO NOTHING` 而没有指定任何列或约束，则会跳过发生的任何唯一性冲突的 INSERT：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')
>>> stmt = stmt.on_conflict_do_nothing()
>>> print(stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)  ON  CONFLICT  DO  NOTHING 
```

### 指定目标

两种方法都使用列推断提供冲突的 “目标”：

+   `Insert.on_conflict_do_update.index_elements` 参数指定一个序列，包含字符串列名、`Column` 对象和/或 SQL 表达式元素，用于标识唯一索引或唯一约束。

+   当使用 `Insert.on_conflict_do_update.index_elements` 推断索引时，还可以通过指定 `Insert.on_conflict_do_update.index_where` 参数来推断部分索引。

    ```py
    >>> stmt = insert(my_table).values(user_email='a@b.com', data='inserted data')

    >>> do_update_stmt = stmt.on_conflict_do_update(
    ...     index_elements=[my_table.c.user_email],
    ...     index_where=my_table.c.user_email.like('%@gmail.com'),
    ...     set_=dict(data=stmt.excluded.data)
    ...     )

    >>> print(do_update_stmt)
    INSERT  INTO  my_table  (data,  user_email)  VALUES  (?,  ?)
    ON  CONFLICT  (user_email)
    WHERE  user_email  LIKE  '%@gmail.com'
    DO  UPDATE  SET  data  =  excluded.data 
    ```

### SET 子句

`ON CONFLICT...DO UPDATE` 用于对已存在的行进行更新，可以使用新值与插入提议中的任意组合值。这些值使用 `Insert.on_conflict_do_update.set_` 参数指定。此参数接受一个字典，其中包含 UPDATE 的直接值：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')

>>> do_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value')
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ? 
```

警告

`Insert.on_conflict_do_update()` 方法**不会**考虑 Python 端默认的 UPDATE 值或生成函数，例如使用 `Column.onupdate` 指定的值。除非这些值在 `Insert.on_conflict_do_update.set_` 字典中手动指定，否则这些值不会用于 ON CONFLICT 类型的 UPDATE。

### 使用插入的排除值进行更新

为了引用插入提议的行，特殊别名 `Insert.excluded` 可作为 `Insert` 对象的属性使用；此对象在列上创建一个“excluded.”前缀，该前缀告知 DO UPDATE 使用将在约束失败时插入的值更新行：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh'
... )

>>> do_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value', author=stmt.excluded.author)
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (?,  ?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?,  author  =  excluded.author 
```

### 附加的 WHERE 条件

`Insert.on_conflict_do_update()` 方法还接受使用 `Insert.on_conflict_do_update.where` 参数的 WHERE 子句，这将限制那些接收 UPDATE 的行：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh'
... )

>>> on_update_stmt = stmt.on_conflict_do_update(
...     index_elements=['id'],
...     set_=dict(data='updated value', author=stmt.excluded.author),
...     where=(my_table.c.status == 2)
... )
>>> print(on_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (?,  ?,  ?)
ON  CONFLICT  (id)  DO  UPDATE  SET  data  =  ?,  author  =  excluded.author
WHERE  my_table.status  =  ? 
```

### 使用 DO NOTHING 跳过行

`ON CONFLICT` 可以用于完全跳过插入行，如果发生与唯一约束的冲突；以下是使用 `Insert.on_conflict_do_nothing()` 方法进行说明：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')
>>> stmt = stmt.on_conflict_do_nothing(index_elements=['id'])
>>> print(stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)  ON  CONFLICT  (id)  DO  NOTHING 
```

如果 `DO NOTHING` 在不指定任何列或约束的情况下使用，则会跳过发生的任何唯一违规的 INSERT：

```py
>>> stmt = insert(my_table).values(id='some_id', data='inserted value')
>>> stmt = stmt.on_conflict_do_nothing()
>>> print(stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (?,  ?)  ON  CONFLICT  DO  NOTHING 
```

## 类型反射

SQLite 类型与大多数其他数据库后端不同，在于类型的字符串名称通常不是一对一地对应于一个“类型”。相反，SQLite 将每列的类型行为链接到五种所谓的“类型亲和性”之一，基于类型的字符串匹配模式。

当 SQLAlchemy 的反射过程检查类型时，它使用一个简单的查找表将返回的关键字链接到提供的 SQLAlchemy 类型。这个查找表存在于 SQLite 方言中，就像存在于所有其他方言中一样。然而，当某个特定类型名称未在查找映射中找到时，SQLite 方言有一个不同的“回退”例程；它实现了位于 [`www.sqlite.org/datatype3.html`](https://www.sqlite.org/datatype3.html) 第 2.1 节的 SQLite “类型亲和性”方案。

提供的类型映射将直接从以下类型的精确字符串名称匹配进行关联：

`BIGINT`、`BLOB`、`BOOLEAN`、`BOOLEAN`、`CHAR`、`DATE`、`DATETIME`、`FLOAT`、`DECIMAL`、`FLOAT`、`INTEGER`、`INTEGER`、`NUMERIC`、`REAL`、`SMALLINT`、`TEXT`、`TIME`、`TIMESTAMP`、`VARCHAR`、`NVARCHAR`、`NCHAR`

当类型名称与上述类型之一不匹配时，将使用“类型亲和性”查找：

+   如果类型名称包含字符串`INT`，则返回`INTEGER`类型。

+   如果类型名称包含字符串`CHAR`、`CLOB`或`TEXT`，则返回`TEXT`类型。

+   如果类型名称包含字符串`BLOB`，则返回`NullType`类型。

+   如果类型名称包含字符串`REAL`、`FLOA`或`DOUB`，则返回`REAL`类型。

+   否则，使用`NUMERIC`类型。

## 部分索引

可以使用 DDL 系统指定带有 WHERE 子句的部分索引，例如使用参数`sqlite_where`：

```py
tbl = Table('testtbl', m, Column('data', Integer))
idx = Index('test_idx1', tbl.c.data,
            sqlite_where=and_(tbl.c.data > 5, tbl.c.data < 10))
```

在创建时，索引将被渲染为：

```py
CREATE INDEX test_idx1 ON testtbl (data)
WHERE data > 5 AND data < 10
```

## 点分列名

不推荐使用显式带有句点的表名或列名。虽然这对于关系数据库来说通常是个坏主意，因为句点是一个语法上重要的字符，但 SQLite 驱动在 SQLite 版本 **3.10.0** 之前存在一个 bug，要求 SQLAlchemy 在结果集中滤掉这些句点。

这个 bug 完全不在 SQLAlchemy 的范围之内，可以用以下方式加以说明：

```py
import sqlite3

assert sqlite3.sqlite_version_info < (3, 10, 0), "bug is fixed in this version"

conn = sqlite3.connect(":memory:")
cursor = conn.cursor()

cursor.execute("create table x (a integer, b integer)")
cursor.execute("insert into x (a, b) values (1, 1)")
cursor.execute("insert into x (a, b) values (2, 2)")

cursor.execute("select x.a, x.b from x")
assert [c[0] for c in cursor.description] == ['a', 'b']

cursor.execute('''
 select x.a, x.b from x where a=1
 union
 select x.a, x.b from x where a=2
''')
assert [c[0] for c in cursor.description] == ['a', 'b'], \
    [c[0] for c in cursor.description]
```

第二个断言失败：

```py
Traceback (most recent call last):
  File "test.py", line 19, in <module>
    [c[0] for c in cursor.description]
AssertionError: ['x.a', 'x.b']
```

在上述情况下，驱动程序错误地报告包括表名在内的列名，这与 UNION 不在时完全不一致。

SQLAlchemy 依赖于列名在匹配原始语句时的可预测性，因此 SQLAlchemy 方言别无选择，只能将这些列名滤除：

```py
from sqlalchemy import create_engine

eng = create_engine("sqlite://")
conn = eng.connect()

conn.exec_driver_sql("create table x (a integer, b integer)")
conn.exec_driver_sql("insert into x (a, b) values (1, 1)")
conn.exec_driver_sql("insert into x (a, b) values (2, 2)")

result = conn.exec_driver_sql("select x.a, x.b from x")
assert result.keys() == ["a", "b"]

result = conn.exec_driver_sql('''
 select x.a, x.b from x where a=1
 union
 select x.a, x.b from x where a=2
''')
assert result.keys() == ["a", "b"]
```

请注意，尽管 SQLAlchemy 过滤掉了句点，*这两个名称仍然是可寻址的*：

```py
>>> row = result.first()
>>> row["a"]
1
>>> row["x.a"]
1
>>> row["b"]
1
>>> row["x.b"]
1
```

因此，SQLAlchemy 应用的解决方法只会影响公共 API 中的 `CursorResult.keys()` 和 `Row.keys()`。在强制使用包含句点的列名，并且需要 `CursorResult.keys()` 和 `Row.keys()` 返回这些带点的名称不经修改的非常特殊情况下，可以在每个 `Connection` 上提供 `sqlite_raw_colnames` 执行选项：

```py
result = conn.execution_options(sqlite_raw_colnames=True).exec_driver_sql('''
 select x.a, x.b from x where a=1
 union
 select x.a, x.b from x where a=2
''')
assert result.keys() == ["x.a", "x.b"]
```

或者在每个 `Engine` 上：

```py
engine = create_engine("sqlite://", execution_options={"sqlite_raw_colnames": True})
```

使用每个 `Engine` 的执行选项时，请注意**使用 UNION 的 Core 和 ORM 查询可能无法正常工作**。

## SQLite 特定的表选项

`CREATE TABLE` 的一种选项直接由 SQLite 方言支持，与 `Table` 结构配合使用：

+   `WITHOUT ROWID`：

    ```py
    Table("some_table", metadata, ..., sqlite_with_rowid=False)
    ```

另请参阅

[SQLite CREATE TABLE 选项](https://www.sqlite.org/lang_createtable.html)

## 反映内部模式表

返回表列表的反射方法将省略所谓的“SQLite 内部模式对象”名称，这些对象被 SQLite 视为任何以 `sqlite_` 为前缀的对象名称。这种对象的示例是在使用 `AUTOINCREMENT` 列参数时生成的 `sqlite_sequence` 表。为了返回这些对象，可以向诸如 `MetaData.reflect()` 或 `Inspector.get_table_names()` 这样的方法传递参数 `sqlite_include_internal=True`。

新版本 2.0 中新增了 `sqlite_include_internal=True` 参数。以前，这些表不被 SQLAlchemy 反射方法忽略。

注意

`sqlite_include_internal` 参数不引用存在于 `sqlite_master` 等模式中的 “系统” 表。

另请参阅

[SQLite 内部模式对象](https://www.sqlite.org/fileformat2.html#intschema) - SQLite 文档中。

## SQLite 数据类型

与所有 SQLAlchemy 方言一样，已知与 SQLite 兼容的所有大写类型都可以从顶级方言导入，无论它们来自 `sqlalchemy.types` 还是来自本地方言：

```py
from sqlalchemy.dialects.sqlite import (
    BLOB,
    BOOLEAN,
    CHAR,
    DATE,
    DATETIME,
    DECIMAL,
    FLOAT,
    INTEGER,
    NUMERIC,
    JSON,
    SMALLINT,
    TEXT,
    TIME,
    TIMESTAMP,
    VARCHAR,
)
```

| 对象名称 | 描述 |
| --- | --- |
| DATE | 使用字符串在 SQLite 中表示 Python date 对象。 |
| DATETIME | 使用字符串在 SQLite 中表示 Python datetime 对象。 |
| JSON | SQLite JSON 类型。 |
| TIME | 使用字符串在 SQLite 中表示 Python time 对象。 |

```py
class sqlalchemy.dialects.sqlite.DATETIME
```

使用字符串在 SQLite 中表示 Python datetime 对象。

默认字符串存储格式为：

```py
"%(year)04d-%(month)02d-%(day)02d  %(hour)02d:%(minute)02d:%(second)02d.%(microsecond)06d"
```

例如：

```py
2021-03-15 12:05:57.105542
```

默认情况下，输入的存储格式使用 Python `datetime.fromisoformat()` 函数进行解析。

新版本 2.0 中已更改：默认日期时间字符串解析使用 `datetime.fromisoformat()`。

可以使用 `storage_format` 和 `regexp` 参数在一定程度上自定义存储格式，例如：

```py
import re
from sqlalchemy.dialects.sqlite import DATETIME

dt = DATETIME(storage_format="%(year)04d/%(month)02d/%(day)02d "
                             "%(hour)02d:%(minute)02d:%(second)02d",
              regexp=r"(\d+)/(\d+)/(\d+) (\d+)-(\d+)-(\d+)"
)
```

参数：

+   `storage_format` - 应用于带有年、月、日、小时、分钟、秒和微秒键的字典的格式字符串。

+   `regexp` - 应用于输入结果行的正则表达式，用于替换使用 `datetime.fromisoformat()` 解析输入字符串。如果正则表达式包含命名组，则结果匹配字典将作为关键字参数应用于 Python datetime() 构造函数。否则，如果使用位置组，则通过 `*map(int, match_obj.groups(0))` 调用 datetime() 构造函数进行位置参数传递。

**类签名**

类 `sqlalchemy.dialects.sqlite.DATETIME` (`sqlalchemy.dialects.sqlite.base._DateTimeMixin`, `sqlalchemy.types.DateTime`)

```py
class sqlalchemy.dialects.sqlite.DATE
```

使用字符串在 SQLite 中表示 Python 日期对象。

默认字符串存储格式为：

```py
"%(year)04d-%(month)02d-%(day)02d"
```

例如：

```py
2011-03-15
```

默认情况下，输入的存储格式使用 Python `date.fromisoformat()` 函数进行解析。

新版本 2.0 中已更改：默认日期字符串解析使用 `date.fromisoformat()`。

可以使用 `storage_format` 和 `regexp` 参数在一定程度上自定义存储格式，例如：

```py
import re
from sqlalchemy.dialects.sqlite import DATE

d = DATE(
        storage_format="%(month)02d/%(day)02d/%(year)04d",
        regexp=re.compile("(?P<month>\d+)/(?P<day>\d+)/(?P<year>\d+)")
    )
```

参数：

+   `storage_format` - 应用于带有年、月和日键的字典的格式字符串。

+   `regexp` – 将应用于传入结果行的正则表达式，替换使用 `date.fromisoformat()` 来解析传入字符串。如果 regexp 包含命名组，则生成的匹配字典将作为关键字参数应用于 Python date() 构造函数。否则，如果使用位置组，则通过 `*map(int, match_obj.groups(0))` 将调用 date() 构造函数以位置参数形式。

**类签名**

类 `sqlalchemy.dialects.sqlite.DATE` (`sqlalchemy.dialects.sqlite.base._DateTimeMixin`, `sqlalchemy.types.Date`)

```py
class sqlalchemy.dialects.sqlite.JSON
```

SQLite JSON 类型。

SQLite 从版本 3.9 开始支持 JSON，通过其 [JSON1](https://www.sqlite.org/json1.html) 扩展。请注意，[JSON1](https://www.sqlite.org/json1.html) 是一个[可加载扩展](https://www.sqlite.org/loadext.html)，因此可能不可用，或者可能需要运行时加载。

当基本 `JSON` 数据类型用于 SQLite 后端时，`JSON` 会自动使用。

另请参阅

`JSON` - 通用跨平台 JSON 数据类型的主要文档。

`JSON` 类型支持将 JSON 值持久化，同时通过在数据库级别将 `JSON_EXTRACT` 函数包装在 `JSON_QUOTE` 函数中来提供 `JSON` 数据类型提供的核心索引操作。提取的值被引用以确保结果始终为 JSON 字符串值。

版本 1.3 中的新内容。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.sqlite.JSON` (`sqlalchemy.types.JSON`)

```py
method __init__(none_as_null: bool = False)
```

*继承自* `JSON` 的 `sqlalchemy.types.JSON.__init__` *方法*

构造一个 `JSON` 类型。

参数：

**none_as_null=False** –

如果为 True，则将值 `None` 持久化为 SQL NULL 值，而不是 `null` 的 JSON 编码。请注意，当此标志为 False 时，仍然可以使用 `null()` 构造来持久化 NULL 值，该构造可以直接作为参数值传递，由 `JSON` 类型特殊解释为 SQL NULL：

```py
from sqlalchemy import null
conn.execute(table.insert(), {"data": null()})
```

注意

`JSON.none_as_null` 不适用于传递给 `Column.default` 和 `Column.server_default` 的值；这些参数传递的`None`值表示“无默认值”。

此外，当在 SQL 比较表达式中使用时，Python 值 `None` 仍然表示 SQL null，而不是 JSON NULL。 `JSON.none_as_null` 标志明确指的是值在 INSERT 或 UPDATE 语句中的**持久性**。应使用 `JSON.NULL` 值进行希望与 JSON null 进行比较的 SQL 表达式。

另请参阅

`JSON.NULL`

```py
class sqlalchemy.dialects.sqlite.TIME
```

以字符串形式在 SQLite 中表示 Python 时间对象。

默认的字符串存储格式是：

```py
"%(hour)02d:%(minute)02d:%(second)02d.%(microsecond)06d"
```

例如：

```py
12:05:57.10558
```

默认情况下，传入的存储格式是使用 Python `time.fromisoformat()`函数解析的。

在 2.0 版本中更改：默认时间字符串解析使用 `time.fromisoformat()`。

存储格式可以在一定程度上使用 `storage_format` 和 `regexp` 参数进行自定义，例如：

```py
import re
from sqlalchemy.dialects.sqlite import TIME

t = TIME(storage_format="%(hour)02d-%(minute)02d-"
                        "%(second)02d-%(microsecond)06d",
         regexp=re.compile("(\d+)-(\d+)-(\d+)-(?:-(\d+))?")
)
```

参数：

+   `storage_format` – 将应用于带有小时、分钟、秒和微秒键的字典的格式字符串。

+   `regexp` – 将应用于传入结果行的正则表达式，替换使用 `datetime.fromisoformat()` 解析传入字符串的用法。如果正则表达式包含命名分组，则生成的匹配字典将作为关键字参数应用于 Python 的 `time()` 构造函数。否则，如果使用了位置分组，则通过 `*map(int, match_obj.groups(0))` 将调用时间()构造函数以位置参数的方式。

**类签名**

类 `sqlalchemy.dialects.sqlite.TIME` (`sqlalchemy.dialects.sqlite.base._DateTimeMixin`, `sqlalchemy.types.Time`)

## SQLite DML Constructs

| 对象名称 | 描述 |
| --- | --- |
| insert(table) | 构造 SQLite 特定的变体 `Insert` 构造。 |
| Insert | INSERT 的 SQLite 特定实现。 |

```py
function sqlalchemy.dialects.sqlite.insert(table: _DMLTableArgument) → Insert
```

构造 SQLite 特定的变体 `Insert` 构造。

`sqlalchemy.dialects.sqlite.insert()` 函数创建一个 `sqlalchemy.dialects.sqlite.Insert`。此类基于方言不可知的 `Insert` 构造，可以使用 SQLAlchemy Core 中的 `insert()` 函数构造。

`Insert` 构造包括其他方法 `Insert.on_conflict_do_update()`, `Insert.on_conflict_do_nothing()`。

```py
class sqlalchemy.dialects.sqlite.Insert
```

SQLite 特定的 INSERT 实现。

添加了针对 SQLite 特定语法的方法，例如 ON CONFLICT。

使用 `sqlalchemy.dialects.sqlite.insert()` 函数创建 `Insert` 对象。

新版本 1.4 中新增。

另请参见

INSERT…ON CONFLICT (Upsert)

**成员**

excluded, inherit_cache, on_conflict_do_nothing(), on_conflict_do_update()

**类签名**

类 `sqlalchemy.dialects.sqlite.Insert` (`sqlalchemy.sql.expression.Insert`)

```py
attribute excluded
```

对于 ON CONFLICT 语句提供了 `excluded` 命名空间

SQLite 的 ON CONFLICT 子句允许引用将要插入的行，称为 `excluded`。此属性提供了此行中的所有列以供引用。

提示

`Insert.excluded` 属性是 `ColumnCollection` 的一个实例，提供了与 访问表和列 中描述的 `Table.c` 集合相同的接口。通过该集合，普通名称可以像属性一样访问（例如 `stmt.excluded.some_column`），但特殊名称和字典方法名称应使用索引访问，例如 `stmt.excluded["column name"]` 或 `stmt.excluded["values"]`。有关更多示例，请参阅 `ColumnCollection` 的文档字符串。

```py
attribute inherit_cache: bool | None = False
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为 `None`，表示结构尚未考虑是否适合参与缓存；这在功能上等同于将值设置为 `False`，除了还会发出警告。

如果与此类本地属性而不是其超类有关的属性不会改变与对象相对应的 SQL，则可以将此标志设置为 `True`。

See also

为自定义结构启用缓存支持 - 设置`HasCacheKey.inherit_cache` 属性的一般指南，用于第三方或用户定义的 SQL 构造。

```py
method on_conflict_do_nothing(index_elements: _OnConflictIndexElementsT = None, index_where: _OnConflictIndexWhereT = None) → Self
```

指定了 ON CONFLICT 子句的 DO NOTHING 操作。

参数：

+   `index_elements` – 由字符串列名、`Column` 对象或其他列表达式对象组成的序列，将用于推断目标索引或唯一约束。

+   `index_where` – 可用于推断条件目标索引的附加 WHERE 准则。

```py
method on_conflict_do_update(index_elements: _OnConflictIndexElementsT = None, index_where: _OnConflictIndexWhereT = None, set_: _OnConflictSetT = None, where: _OnConflictWhereT = None) → Self
```

指定了 ON CONFLICT 子句的 DO UPDATE SET 操作。

参数：

+   `index_elements` – 由字符串列名、`Column` 对象或其他列表达式对象组成的序列，将用于推断目标索引或唯一约束。

+   `index_where` – 可用于推断条件目标索引的附加 WHERE 准则。

+   `set_` –

    一个字典或其他映射对象，其中键可以是目标表中的列名，或者是 `Column` 对象或其他 ORM 映射的列，与目标表匹配，以及表达式或字面值作为值，指定要执行的 `SET` 操作。

    从版本 1.4 开始：`Insert.on_conflict_do_update.set_` 参数支持来自目标 `Table` 的 `Column` 对象作为键。

    警告

    这个字典**不**考虑 Python 指定的默认 UPDATE 值或生成函数，例如使用 `Column.onupdate` 指定的那些。这些值不会对 ON CONFLICT 风格的 UPDATE 生效，除非它们在 `Insert.on_conflict_do_update.set_` 字典中手动指定。

+   `where` – 可选参数。如果存在，则可以是一个 SQL 字符串字面量或 `WHERE` 子句的可接受表达式，该子句限制了由 `DO UPDATE SET` 受影响的行。不符合 `WHERE` 条件的行将不会更新（对于这些行实际上是 `DO NOTHING`）。

## Pysqlite

通过 pysqlite 驱动程序支持 SQLite 数据库。

请注意，`pysqlite` 与 Python 发行版中包含的 `sqlite3` 模块是相同的驱动程序。

### DBAPI

pysqlite 的文档和下载信息（如果适用）可在此处获取：[`docs.python.org/library/sqlite3.html`](https://docs.python.org/library/sqlite3.html)

### 连接

连接字符串：

```py
sqlite+pysqlite:///file_path
```

### 驱动程序

`sqlite3` Python DBAPI 是所有现代 Python 版本的标准；对于 cPython 和 Pypy，不需要额外安装。

### 连接字符串

对于 SQLite 数据库的文件规范被视为 URL 的“数据库”部分。请注意，SQLAlchemy URL 的格式是：

```py
driver://user:pass@host/database
```

文件名的实际使用要从第三个斜杠的**右边**开始的字符开始。因此，连接到相对文件路径看起来像是：

```py
# relative path
e = create_engine('sqlite:///path/to/database.db')
```

绝对路径，以斜杠开头表示，意味着你需要**四个**斜杠：

```py
# absolute path
e = create_engine('sqlite:////path/to/database.db')
```

要使用 Windows 路径，可以使用常规的驱动器规范和反斜杠。可能需要双反斜杠：

```py
# absolute path on Windows
e = create_engine('sqlite:///C:\\path\\to\\database.db')
```

要使用 sqlite `:memory:` 数据库，请将其指定为使用 `sqlite://:memory:` 的文件名。如果没有路径名，指定 `sqlite://` 并什么都不写，也是默认情况：

```py
# in-memory database
e = create_engine('sqlite://:memory:')
# also in-memory database
e2 = create_engine('sqlite://')
```

#### URI 连接

现代版本的 SQLite 支持使用[驱动程序级 URI](https://www.sqlite.org/uri.html)进行连接的替代系统，其优势在于可以传递额外的驱动程序级参数，包括“只读”等选项。Python sqlite3 驱动程序在现代 Python 3 版本下支持此模式。SQLAlchemy pysqlite 驱动程序通过在 URL 查询字符串中指定“uri=true”来支持此使用模式。SQLite 级别的“URI”保留为 SQLAlchemy URL 的“数据库”部分（即在斜杠后面）：

```py
e = create_engine("sqlite:///file:path/to/database?mode=ro&uri=true")
```

注意

“uri=true”参数必须出现在 URL 的**查询字符串**中。如果它仅出现在`create_engine.connect_args`参数字典中，则目前不会按预期工作。

该逻辑通过分离属于 Python sqlite3 驱动程序的参数和属于 SQLite URI 的参数来协调 SQLAlchemy 的查询字符串和 SQLite 的查询字符串的同时存在。这是通过使用已知被 Python 驱动程序接受的一组固定参数来实现的。例如，要包含指示 Python sqlite3“timeout”和“check_same_thread”参数以及 SQLite“mode”和“nolock”参数的 URL，它们可以一起传递在查询字符串中：

```py
e = create_engine(
    "sqlite:///file:path/to/database?"
    "check_same_thread=true&timeout=10&mode=ro&nolock=1&uri=true"
)
```

在上面，pysqlite / sqlite3 DBAPI 将传递参数如下：

```py
sqlite3.connect(
    "file:path/to/database?mode=ro&nolock=1",
    check_same_thread=True, timeout=10, uri=True
)
```

关于将来添加到 Python 或本机驱动程序的参数。应该自动适应此方案的 SQLite URI 方案中添加的新参数名称。添加到 Python 驱动程序端的新参数名称可以通过在`create_engine.connect_args`字典中指定它们来适应，直到 SQLAlchemy 添加方言支持。对于较不可能的情况，即本机 SQLite 驱动程序添加了一个与现有已知 Python 驱动程序参数之一重叠的新参数名称（例如“timeout”），SQLAlchemy 的方言将需要调整 URL 方案以继续支持这一点。

对于所有 SQLAlchemy 方言，始终可以通过使用`create_engine()`中的`create_engine.creator`参数来绕过整个“URL”过程，该参数允许创建一个直接创建 Python sqlite3 驱动程序级连接的自定义可调用对象。

版本 1.3.9 中的新功能。

另请参阅

[统一资源标识符](https://www.sqlite.org/uri.html) - SQLite 文档中的内容  ### 正则表达式支持

版本 1.4 中的新功能。

支持使用 Python 的[re.search](https://docs.python.org/3/library/re.html#re.search)函数提供`ColumnOperators.regexp_match()`操作符。SQLite 本身不包括有效的正则表达式操作符；相反，它包括一个未实现的占位符操作符`REGEXP`，调用必须提供的用户定义函数。

SQLAlchemy 的实现利用 pysqlite 的[create_function](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function)钩子如下：

```py
def regexp(a, b):
    return re.search(a, b) is not None

sqlite_connection.create_function(
    "regexp", 2, regexp,
)
```

目前不支持将正则表达式标志作为单独参数，因为 SQLite 的 REGEXP 操作符不支持这些标志，但可以在正则表达式字符串内联包含这些标志。有关详细信息，请参阅[Python 正则表达式](https://docs.python.org/3/library/re.html#re.search)。

另请参见

[Python 正则表达式](https://docs.python.org/3/library/re.html#re.search)：Python 正则表达式语法的文档。

### 与 sqlite3“本地”日期和日期时间类型兼容

pysqlite 驱动程序包括 sqlite3.PARSE_DECLTYPES 和 sqlite3.PARSE_COLNAMES 选项，其效果是任何显式转换为“date”或“timestamp”的列或表达式将转换为 Python 日期或日期时间对象。pysqlite 方言提供的日期和日期时间类型目前与这些选项不兼容，因为它们呈现包括微秒的 ISO 日期/日期时间，而 pysqlite 的驱动程序不包括。此外，SQLAlchemy 目前不会自动呈现“cast”语法，以便使独立函数“current_timestamp”和“current_date”返回本地的日期时间/日期类型。不幸的是，pysqlite 不会在`cursor.description`中提供标准的 DBAPI 类型，使得 SQLAlchemy 无法在不进行昂贵的每行类型检查的情况下动态检测这些类型。

请记住，不建议使用 pysqlite 的解析选项，也不应该与 SQLAlchemy 一起使用，如果在 create_engine()上配置“native_datetime=True”，则可以强制使用 PARSE_DECLTYPES：

```py
engine = create_engine('sqlite://',
    connect_args={'detect_types':
        sqlite3.PARSE_DECLTYPES|sqlite3.PARSE_COLNAMES},
    native_datetime=True
)
```

启用此标志后，DATE 和 TIMESTAMP 类型（但请注意 - 不包括 DATETIME 或 TIME 类型...搞糊涂了吗？）将不执行任何绑定参数或结果处理。执行“func.current_date()”将返回一个字符串。“func.current_timestamp()”在 SQLAlchemy 中注册为返回 DATETIME 类型，因此此函数仍然接收 SQLAlchemy 级别的结果处理。

### 线程/池行为

默认情况下，`sqlite3` DBAPI 禁止在创建它的线程之外的线程中使用特定连接。随着 SQLite 的成熟，它在多线程下的行为已经改进，甚至包括选项，允许在多个线程中使用仅内存数据库。

线程禁止被称为“检查同一线程”，可以使用`sqlite3`参数`check_same_thread`来控制，该参数将禁用或启用此检查。 SQLAlchemy 在这里的默认行为是，当使用基于文件的数据库时，自动将`check_same_thread`设置为`False`，以与默认的池类`QueuePool`建立兼容性。

SQLAlchemy 的`pysqlite` DBAPI 基于请求的 SQLite 数据库类型以不同的方式建立连接池：

+   当指定了一个`:memory:`的 SQLite 数据库时，默认情况下方言将使用`SingletonThreadPool`。该池每个线程维护一个单一连接，因此当前线程内对引擎的所有访问都使用相同的`:memory:`数据库，而其他线程将访问不同的`:memory:`数据库。`check_same_thread`参数的默认值为`True`。

+   当指定文件型数据库时，方言将同时使用`QueuePool`作为连接源。同时，默认情况下将`check_same_thread`标志设置为`False`，除非被覆盖。

    从版本 2.0 开始更改：SQLite 文件数据库引擎现在默认使用`QueuePool`。先前使用的是`NullPool`。可以通过在`create_engine.poolclass`参数中指定`NullPool`类来使用`NullPool`类。

#### 为文件数据库禁用连接池

可以通过为`poolclass()`参数指定`NullPool`实现来禁用文件型数据库的连接池：

```py
from sqlalchemy import NullPool
engine = create_engine("sqlite:///myfile.db", poolclass=NullPool)
```

已观察到由于`QueuePool`没有连接重用，因此`NullPool`实现在重复检出时产生极小的性能开销。然而，如果应用程序出现文件被锁定的问题，仍然可能有利于使用此类。

#### 在多个线程中使用内存数据库

要在多线程情况下使用 `:memory:` 数据库，必须共享相同的连接对象，因为数据库仅存在于该连接的范围内。 `StaticPool` 实现将在全局维护单个连接，并且可以将 `check_same_thread` 标志传递给 Pysqlite 为 `False`：

```py
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite://',
                    connect_args={'check_same_thread':False},
                    poolclass=StaticPool)
```

注意在多线程中使用 `:memory:` 数据库需要 SQLite 的最新版本。

#### 使用临时表与 SQLite

由于 SQLite 处理临时表的方式，如果希望在基于文件的 SQLite 数据库中跨多个连接池检出时使用临时表，例如在使用 ORM `Session` 时，临时表应在 `Session.commit()` 或 `Session.rollback()` 调用后继续存在，必须使用维护单个连接的连接池。如果作用域仅在当前线程中需要，请使用 `SingletonThreadPool`，或者如果需要在多个线程中使用，则使用 `StaticPool`：

```py
# maintain the same connection per thread
from sqlalchemy.pool import SingletonThreadPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=SingletonThreadPool)

# maintain the same connection across all threads
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=StaticPool)
```

请注意，`SingletonThreadPool` 应配置为要使用的线程数；超出该数量时，连接将以不确定的方式关闭。

### 处理混合字符串 / 二进制列

SQLite 数据库是弱类型的，因此在使用二进制值时（在 Python 中表示为 `b'some string'`），可能会出现特定的 SQLite 数据库，其中一些行的数据值将由 Pysqlite 驱动程序返回为 `b''` 值，而其他行将作为 Python 字符串返回，例如 `''` 值。如果一致使用 SQLAlchemy `LargeBinary` 数据类型，则不会发生此情况，但是如果特定的 SQLite 数据库具有使用 Pysqlite 驱动程序直接插入的数据，或者当使用 SQLAlchemy `String` 类型时后来更改为 `LargeBinary`，表将无法一致可读，因为 SQLAlchemy 的 `LargeBinary` 数据类型不处理字符串，因此无法对处于字符串格式的值进行“编码”。

要处理具有混合字符串/二进制数据的 SQLite 表中的情况，请使用一个自定义类型，将逐行检查每一行：

```py
from sqlalchemy import String
from sqlalchemy import TypeDecorator

class MixedBinary(TypeDecorator):
    impl = String
    cache_ok = True

    def process_result_value(self, value, dialect):
        if isinstance(value, str):
            value = bytes(value, 'utf-8')
        elif value is not None:
            value = bytes(value)

        return value
```

然后在通常会使用 `LargeBinary` 的地方使用上述 `MixedBinary` 数据类型。

### 可序列化隔离 / 保存点 / 事务 DDL

在 数据库锁定行为 / 并发性 部分中，我们提到 pysqlite 驱动程序的一系列问题，这些问题会导致 SQLite 的几个功能无法正常工作。 pysqlite DBAPI 驱动程序有几个长期存在的错误，影响其事务行为的正确性。 在其默认操作模式下，SQLite 的功能，如 SERIALIZABLE 隔离、事务 DDL 和 SAVEPOINT 支持是不起作用的，为了使用这些功能，必须采取解决方法。

问题本质上是驱动程序试图猜测用户的意图，未能启动事务，有时会过早结束事务，以减少 SQLite 数据库的文件锁定行为，尽管 SQLite 本身对只读活动使用“共享”锁。

SQLAlchemy 默认选择不更改此行为，因为这是 pysqlite 驱动程序长期期望的行为；如果 pysqlite 驱动程序尝试修复这些问题，那将更多地影响到 SQLAlchemy 的默认设置。

好消息是，通过几个事件，我们可以完全实现事务支持，通过完全禁用 pysqlite 的功能并自己发出 BEGIN 来实现。 这是通过使用两个事件监听器实现的：

```py
from sqlalchemy import create_engine, event

engine = create_engine("sqlite:///myfile.db")

@event.listens_for(engine, "connect")
def do_connect(dbapi_connection, connection_record):
    # disable pysqlite's emitting of the BEGIN statement entirely.
    # also stops it from emitting COMMIT before any DDL.
    dbapi_connection.isolation_level = None

@event.listens_for(engine, "begin")
def do_begin(conn):
    # emit our own BEGIN
    conn.exec_driver_sql("BEGIN")
```

警告

在使用上述方法时，建议不要在 SQLite 驱动程序上使用 `Connection.execution_options.isolation_level` 设置，因为这个函数必然也会改变“.isolation_level”设置。

在上面，我们拦截一个新的 pysqlite 连接并禁用任何事务集成。然后，在 SQLAlchemy 知道事务范围即将开始的时候，我们自己发出 `"BEGIN"`。

当我们控制 `"BEGIN"` 时，我们也可以直接控制 SQLite 的锁定模式，通过在我们的 `"BEGIN"` 中添加所需的锁定模式：

```py
@event.listens_for(engine, "begin")
def do_begin(conn):
    conn.exec_driver_sql("BEGIN EXCLUSIVE")
```

另请参阅

[BEGIN TRANSACTION](https://sqlite.org/lang_transaction.html) - 在 SQLite 网站上

[sqlite3 SELECT 不会开始事务](https://bugs.python.org/issue9924) - 在 Python 缺陷跟踪器上

[sqlite3 模块破坏事务并可能损坏数据](https://bugs.python.org/issue10740) - 在 Python 缺陷跟踪器上  ### 用户定义函数

pysqlite 支持一个[create_function()](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function)方法，允许我们在 Python 中创建自己的用户定义函数（UDF）并直接在 SQLite 查询中使用它们。这些函数与特定的 DBAPI 连接相关联。

SQLAlchemy 使用基于文件的 SQLite 数据库的连接池，因此我们需要确保在创建连接时将 UDF 附加到连接上。通过事件监听器实现：

```py
from sqlalchemy import create_engine
from sqlalchemy import event
from sqlalchemy import text

def udf():
    return "udf-ok"

engine = create_engine("sqlite:///./db_file")

@event.listens_for(engine, "connect")
def connect(conn, rec):
    conn.create_function("udf", 0, udf)

for i in range(5):
    with engine.connect() as conn:
        print(conn.scalar(text("SELECT UDF()")))
```

### 数据库 API

pysqlite 的文档和下载信息（如果适用）可在此处获取：[`docs.python.org/library/sqlite3.html`](https://docs.python.org/library/sqlite3.html)

### 连接

连接字符串：

```py
sqlite+pysqlite:///file_path
```

### 驱动程序

在所有现代 Python 版本上，`sqlite3` Python 数据库 API 是标准的；对于 cPython 和 Pypy，不需要额外安装。

### 连接字符串

SQLite 数据库的文件规范被视为 URL 的“数据库”部分。请注意，SQLAlchemy URL 的格式是：

```py
driver://user:pass@host/database
```

这意味着要使用的实际文件名从第三个斜杠的**右侧**字符开始。因此，连接到相对文件路径看起来像：

```py
# relative path
e = create_engine('sqlite:///path/to/database.db')
```

绝对路径，以斜杠开头，意味着你需要**四个**斜杠：

```py
# absolute path
e = create_engine('sqlite:////path/to/database.db')
```

要使用 Windows 路径，可以使用常规的驱动器规范和反斜杠。可能需要双反斜杠：

```py
# absolute path on Windows
e = create_engine('sqlite:///C:\\path\\to\\database.db')
```

要使用 sqlite 的`:memory:`数据库，请将其指定为文件名，使用`sqlite://:memory:`。如果没有文件路径，则它也是默认值，只需指定`sqlite://`而不加其他内容：

```py
# in-memory database
e = create_engine('sqlite://:memory:')
# also in-memory database
e2 = create_engine('sqlite://')
```

#### URI 连接

现代版本的 SQLite 支持使用[驱动程序级 URI](https://www.sqlite.org/uri.html)进行连接的替代系统，其优点是可以传递额外的驱动程序级参数，包括“只读”选项。 Python sqlite3 驱动程序在现代 Python 3 版本下支持此模式。 SQLAlchemy pysqlite 驱动程序通过在 URL 查询字符串中指定“uri=true”来支持此使用模式。 SQLite 级别的“URI”保留为 SQLAlchemy URL 的“数据库”部分（即，跟在斜杠后面）：

```py
e = create_engine("sqlite:///file:path/to/database?mode=ro&uri=true")
```

注意

“uri=true”参数必须出现在 URL 的**查询字符串**中。如果它只出现在`create_engine.connect_args`参数字典中，则当前不会按预期工作。

该逻辑通过分离属于 Python sqlite3 驱动程序与属于 SQLite URI 的参数，来调和 SQLAlchemy 查询字符串和 SQLite 查询字符串的同时出现。这通过使用一个已知被驱动程序的 Python 部分接受的固定参数列表来实现。例如，要包含指示 Python sqlite3“timeout”和“check_same_thread”参数以及 SQLite“mode”和“nolock”参数的 URL，它们可以一起传递到查询字符串中：

```py
e = create_engine(
    "sqlite:///file:path/to/database?"
    "check_same_thread=true&timeout=10&mode=ro&nolock=1&uri=true"
)
```

上面，pysqlite / sqlite3 DBAPI 将被传递参数如下：

```py
sqlite3.connect(
    "file:path/to/database?mode=ro&nolock=1",
    check_same_thread=True, timeout=10, uri=True
)
```

关于将来添加到 Python 或本机驱动程序的新参数。添加到 SQLite URI 方案的新参数名称应该自动适应此方案。添加到 Python 驱动程序端的新参数名称可以通过在`create_engine.connect_args`字典中指定它们来适应，直到 SQLAlchemy 添加了方言支持。对于本机 SQLite 驱动程序添加一个与现有已知 Python 驱动程序参数（例如“timeout”）重叠的新参数名称的可能性较小，SQLAlchemy 的方言将需要调整 URL 方案以继续支持此参数。

对于所有 SQLAlchemy 方言，始终可以通过使用`create_engine.creator`参数绕过整个“URL”过程，在`create_engine()`中直接通过使用一个自定义可调用对象来创建 Python sqlite3 驱动程序级别的连接。

版本 1.3.9 中的新功能。

另请参见

[统一资源标识符](https://www.sqlite.org/uri.html) - 在 SQLite 文档中  #### URI 连接

现代版本的 SQLite 支持使用[驱动级 URI](https://www.sqlite.org/uri.html)进行连接的替代系统，其优势在于可以传递额外的驱动级参数，包括“只读”等选项。Python sqlite3 驱动程序在现代 Python 3 版本下支持此模式。SQLAlchemy pysqlite 驱动程序通过在 URL 查询字符串中指定“uri=true”来支持此使用模式。SQLite 级别的“URI”保留为 SQLAlchemy URL 的“database”部分（即，在斜杠后面）：

```py
e = create_engine("sqlite:///file:path/to/database?mode=ro&uri=true")
```

注意

“uri=true” 参数必须出现在 URL 的**查询字符串**中。如果它只存在于`create_engine.connect_args`参数字典中，它目前不会按预期工作。

逻辑通过将属于 Python sqlite3 驱动程序的参数与属于 SQLite URI 的参数分开，来协调 SQLAlchemy 的查询字符串和 SQLite 的查询字符串的同时存在。这是通过使用已知被 Python 驱动程序接受的一组固定参数来实现的。例如，要包含指示 Python sqlite3“timeout”和“check_same_thread”参数以及 SQLite“mode”和“nolock”参数的 URL，它们可以一起传递在查询字符串中：

```py
e = create_engine(
    "sqlite:///file:path/to/database?"
    "check_same_thread=true&timeout=10&mode=ro&nolock=1&uri=true"
)
```

上面，pysqlite / sqlite3 DBAPI 将被传递参数如下：

```py
sqlite3.connect(
    "file:path/to/database?mode=ro&nolock=1",
    check_same_thread=True, timeout=10, uri=True
)
```

关于将来添加到 Python 或本机驱动程序的参数。新增加到 SQLite URI 方案的参数名应该由该方案自动适应。新增加到 Python 驱动程序端的参数名可以通过在 `create_engine.connect_args` 字典中指定它们来容纳，直到 SQLAlchemy 添加了方言支持。对于较不可能的情况，即本机 SQLite 驱动程序添加了与现有已知 Python 驱动程序参数（例如“timeout”）重叠的新参数名，SQLAlchemy 的方言需要调整 URL 方案以继续支持此参数。

与 SQLAlchemy 方言的所有情况一样，整个“URL”过程都可以通过 `create_engine()` 中的 `create_engine.creator` 参数绕过，该参数允许自定义可调用项，直接创建 Python sqlite3 驱动程序级连接。

1.3.9 版的新内容。

另请参阅

[统一资源标识符](https://www.sqlite.org/uri.html) - SQLite 文档中

### 正则表达式支持

1.4 版中的新内容。

支持使用 Python 的 [re.search](https://docs.python.org/3/library/re.html#re.search) 函数提供 `ColumnOperators.regexp_match()` 操作符。SQLite 本身不包括工作正则表达式运算符；相反，它包括一个未实现的占位符操作符 `REGEXP`，该操作符调用必须提供的用户定义函数。

SQLAlchemy 的实现使用 pysqlite [create_function](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function) 钩子，如下所示：

```py
def regexp(a, b):
    return re.search(a, b) is not None

sqlite_connection.create_function(
    "regexp", 2, regexp,
)
```

目前不支持将正则表达式标志作为单独参数，因为这些标志不受 SQLite 的 REGEXP 操作符支持，但可以内联在正则表达式字符串中。有关详情，请参阅 [Python 正则表达式](https://docs.python.org/3/library/re.html#re.search)。

另请参阅

[Python 正则表达式](https://docs.python.org/3/library/re.html#re.search)：Python 正则表达式语法的文档。

### 兼容性与 sqlite3 的“本地”日期和日期时间类型

pysqlite 驱动程序包括 sqlite3.PARSE_DECLTYPES 和 sqlite3.PARSE_COLNAMES 选项，其效果是任何明确转换为“date”或“timestamp”的列或表达式将被转换为 Python 的日期或日期时间对象。pysqlite 方言提供的日期和日期时间类型目前与这些选项不兼容，因为它们呈现的 ISO 日期/日期时间包括微秒，而 pysqlite 的驱动程序没有。此外，SQLAlchemy 目前不会自动渲染“cast”语法，该语法要求独立的函数“current_timestamp”和“current_date”以本地返回 datetime/date 类型。不幸的是，pysqlite 不会在 `cursor.description` 中提供标准的 DBAPI 类型，因此 SQLAlchemy 无法在不执行昂贵的每行类型检查的情况下即时检测到这些类型。

特别注意，pysqlite 的解析选项不建议使用，也不应该在与 SQLAlchemy 一起使用时需要使用，如果在 create_engine() 上配置了 “native_datetime=True”，则可以强制使用 PARSE_DECLTYPES 选项：

```py
engine = create_engine('sqlite://',
    connect_args={'detect_types':
        sqlite3.PARSE_DECLTYPES|sqlite3.PARSE_COLNAMES},
    native_datetime=True
)
```

启用此标志后，DATE 和 TIMESTAMP 类型（但请注意 - 不是 DATETIME 或 TIME 类型…搞糊涂了吗？）将不执行任何绑定参数或结果处理。执行“func.current_date()”将返回一个字符串。“func.current_timestamp()”在 SQLAlchemy 中注册为返回 DATETIME 类型，因此此函数仍然接收 SQLAlchemy 级别的结果处理。

### 线程/池行为

默认情况下，`sqlite3` DBAPI 禁止在非创建它的线程中使用特定的连接。随着 SQLite 的成熟，它在多线程下的行为已经改进，甚至包括选项让内存数据库可以在多个线程中使用。

线程禁止被称为“检查同一线程”，可以使用 `sqlite3` 参数 `check_same_thread` 来控制，这将禁用或启用此检查。SQLAlchemy 在这里的默认行为是，当使用基于文件的数据库时，自动将 `check_same_thread` 设置为 `False`，以确保与默认的池类 `QueuePool` 兼容。

SQLAlchemy `pysqlite` DBAPI 根据所请求的 SQLite 数据库的类型不同而建立连接池：

+   当指定了一个 `:memory:` 的 SQLite 数据库时，默认情况下方言会使用 `SingletonThreadPool`。这个池每个线程维护一个连接，所以当前线程内的对引擎的所有访问都使用同一个 `:memory:` 数据库 - 其他线程将访问一个不同的 `:memory:` 数据库。`check_same_thread` 参数默认为 `True`。

+   当指定基于文件的数据库时，方言将使用`QueuePool`作为连接的源。同时，默认情况下将`check_same_thread`标志设置为 False，除非被覆盖。

    从版本 2.0 开始更改：SQLite 文件数据库引擎现在默认使用`QueuePool`。以前使用的是`NullPool`。可以通过`create_engine.poolclass`参数指定使用`NullPool`类。

#### 禁用文件数据库的连接池

通过为`poolclass()`参数指定`NullPool`实现，可以禁用基于文件的数据库的连接池：

```py
from sqlalchemy import NullPool
engine = create_engine("sqlite:///myfile.db", poolclass=NullPool)
```

据观察，`NullPool`实现由于`QueuePool`实现的连接不重用而导致极小的性能开销。但是，如果应用程序遇到文件被锁定的问题，仍然可能有益于使用此类。

#### 在多个线程中使用内存数据库

在多线程场景中使用`:memory:`数据库，必须共享相同的连接对象，因为数据库仅存在于该连接的范围内。`StaticPool`实现将全局维护一个单一连接，并且`check_same_thread`标志可以传递给 Pysqlite 作为 `False`：

```py
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite://',
                    connect_args={'check_same_thread':False},
                    poolclass=StaticPool)
```

请注意，在多线程中使用`:memory:`数据库需要最新版本的 SQLite。

#### 使用 SQLite 临时表

由于 SQLite 处理临时表的方式，如果希望在基于文件的 SQLite 数据库中跨多个连接池检出使用临时表，例如在使用 ORM `Session`时，临时表应在`Session.commit()`或`Session.rollback()`之后继续保留，必须使用维护单个连接的池。如果范围仅在当前线程内使用，则使用`SingletonThreadPool`，或者在此情况下需要在多个线程中使用范围，则使用`StaticPool`：

```py
# maintain the same connection per thread
from sqlalchemy.pool import SingletonThreadPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=SingletonThreadPool)

# maintain the same connection across all threads
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=StaticPool)
```

请注意，应该为`SingletonThreadPool`配置要使用的线程数；超出该数量，连接将以不确定的方式关闭。

#### 禁用文件数据库的连接池

可以通过为`poolclass()`参数指定`NullPool`实现来禁用基于文件的数据库的池化：

```py
from sqlalchemy import NullPool
engine = create_engine("sqlite:///myfile.db", poolclass=NullPool)
```

使用`NullPool`实现观察到，由于`QueuePool`没有实现连接重用，因此对于重复检出，它会产生极小的性能开销。然而，如果应用程序遇到文件被锁定的问题，仍然可能有利用这个类。

#### 在多线程中使用内存数据库

在多线程方案中使用`:memory:`数据库，相同的连接对象必须在线程之间共享，因为数据库仅存在于该连接的范围内。`StaticPool`实现将在全局维护一个单一连接，并且`check_same_thread`标志可以传递给 Pysqlite，设置为`False`：

```py
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite://',
                    connect_args={'check_same_thread':False},
                    poolclass=StaticPool)
```

请注意，在多个线程中使用`:memory:`数据库需要 SQLite 的最新版本。

#### 使用 SQLite 临时表

由于 SQLite 处理临时表的方式，如果希望在基于文件的 SQLite 数据库中跨多次从连接池检出时使用临时表，例如在使用 ORM `Session`时，在`Session.commit()`或`Session.rollback()`之后，临时表应继续保持，必须使用维护单个连接的池。如果范围仅在当前线程内需要，则使用`SingletonThreadPool`，如果在多个线程中需要范围，则使用`StaticPool`用于此案例：

```py
# maintain the same connection per thread
from sqlalchemy.pool import SingletonThreadPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=SingletonThreadPool)

# maintain the same connection across all threads
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite:///mydb.db',
                    poolclass=StaticPool)
```

请注意，`SingletonThreadPool`应配置为要使用的线程数；超出该数字后，连接将以不确定的方式关闭。

### 处理混合字符串/二进制列

SQLite 数据库是弱类型的，因此当使用二进制值时，可能出现一种情况，即在 Python 中表示为`b'some string'`的情况下，特定的 SQLite 数据库可能会在不同的行中具有不同的数据值，其中一些将被 Pysqlite 驱动器返回为`b''`值，而另一些将被返回为 Python 字符串，例如`''`值。如果一直使用 SQLAlchemy 的`LargeBinary`数据类型，则不会发生此情况，但是如果特定的 SQLite 数据库具有使用 Pysqlite 驱动器直接插入的数据，或者在使用后将其更改为`LargeBinary`的 SQLAlchemy `String`类型时，表将无法一致地读取，因为 SQLAlchemy 的`LargeBinary`数据类型不处理字符串，因此无法“编码”字符串格式的值。

要处理具有相同列中的混合字符串/二进制数据的 SQLite 表，请使用自定义类型逐个检查每一行：

```py
from sqlalchemy import String
from sqlalchemy import TypeDecorator

class MixedBinary(TypeDecorator):
    impl = String
    cache_ok = True

    def process_result_value(self, value, dialect):
        if isinstance(value, str):
            value = bytes(value, 'utf-8')
        elif value is not None:
            value = bytes(value)

        return value
```

然后在通常会使用`LargeBinary`的地方使用上述的`MixedBinary`数据类型。

### 可序列化隔离/保存点/事务 DDL

在 数据库锁定行为 / 并发性 部分中，我们提到了 pysqlite 驱动程序的各种问题，这些问题阻止了 SQLite 的几个功能正常工作。pysqlite DBAPI 驱动程序有一些长期存在的错误，这些错误影响了其事务行为的正确性。在其默认操作模式下，SQLite 功能（如 SERIALIZABLE 隔离、事务性 DDL 和 SAVEPOINT 支持）是不起作用的，为了使用这些功能，必须采取一些变通方法。

问题本质上是驱动程序试图猜测用户的意图，未能启动事务，并有时过早结束它们，以尽量减少 SQLite 数据库的文件锁定行为，尽管 SQLite 本身对只读活动使用“共享”锁。

SQLAlchemy 选择默认情况下不更改此行为，因为这是 pysqlite 驱动程序的长期期望行为；如果 pysqlite 驱动程序尝试修复这些问题，那将更多地驱动 SQLAlchemy 的默认设置。

好消息是，通过几个事件，我们可以完全实现事务支持，方法是完全禁用 pysqlite 的功能，并自行发出 BEGIN。这通过两个事件监听器实现：

```py
from sqlalchemy import create_engine, event

engine = create_engine("sqlite:///myfile.db")

@event.listens_for(engine, "connect")
def do_connect(dbapi_connection, connection_record):
    # disable pysqlite's emitting of the BEGIN statement entirely.
    # also stops it from emitting COMMIT before any DDL.
    dbapi_connection.isolation_level = None

@event.listens_for(engine, "begin")
def do_begin(conn):
    # emit our own BEGIN
    conn.exec_driver_sql("BEGIN")
```

警告

当使用上述方法时，建议不要在 SQLite 驱动程序上使用 `Connection.execution_options.isolation_level` 设置以及 `create_engine()`，因为这个函数必然会改变“.isolation_level”设置。

上面，我们拦截了一个新的 pysqlite 连接，并禁用了任何事务集成。然后，在 SQLAlchemy 知道事务范围即将开始的时候，我们自己发出了 `"BEGIN"`。

当我们控制 `"BEGIN"` 时，我们也可以直接控制 SQLite 的锁定模式，通过在我们的 `"BEGIN"` 中添加所需的锁定模式来引入 [BEGIN TRANSACTION](https://sqlite.org/lang_transaction.html) 中的锁定模式：

```py
@event.listens_for(engine, "begin")
def do_begin(conn):
    conn.exec_driver_sql("BEGIN EXCLUSIVE")
```

另请参阅

[BEGIN TRANSACTION](https://sqlite.org/lang_transaction.html) - SQLite 网站上的内容

[sqlite3 SELECT does not BEGIN a transaction](https://bugs.python.org/issue9924) - Python 缺陷跟踪器上的问题

[sqlite3 模块破坏事务并可能损坏数据](https://bugs.python.org/issue10740) - Python 缺陷跟踪器上的问题

### 用户定义的函数

pysqlite 支持一个 [create_function()](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function) 方法，允许我们在 Python 中创建自己的用户定义函数（UDFs），并直接在 SQLite 查询中使用它们。这些函数与特定的 DBAPI 连接相关联。

SQLAlchemy 在基于文件的 SQLite 数据库中使用连接池，因此我们需要确保在创建连接时将 UDF 附加到连接上。这可以通过事件侦听器完成：

```py
from sqlalchemy import create_engine
from sqlalchemy import event
from sqlalchemy import text

def udf():
    return "udf-ok"

engine = create_engine("sqlite:///./db_file")

@event.listens_for(engine, "connect")
def connect(conn, rec):
    conn.create_function("udf", 0, udf)

for i in range(5):
    with engine.connect() as conn:
        print(conn.scalar(text("SELECT UDF()")))
```

## Aiosqlite

通过 aiosqlite 驱动程序支持 SQLite 数据库。

### DBAPI

aiosqlite 的文档和下载信息 (如果适用) 可在此处找到: [`pypi.org/project/aiosqlite/`](https://pypi.org/project/aiosqlite/)

### 连接

连接字符串:

```py
sqlite+aiosqlite:///file_path
```

aiosqlite 方言提供了对在 pysqlite 上运行的 SQLAlchemy asyncio 接口的支持。

aiosqlite 是 pysqlite 的一个封装，它为每个连接使用一个后台线程。它实际上不使用非阻塞 IO，因为 SQLite 数据库不是基于套接字的。但是，它提供了一个有效的 asyncio 接口，对于测试和原型设计非常有用。

使用特殊的 asyncio 中介层，aiosqlite 方言可用作 SQLAlchemy asyncio 扩展包的后端。

这个方言通常应该仅与 `create_async_engine()` 引擎创建函数一起使用：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("sqlite+aiosqlite:///filename")
```

URL 通过所有参数传递给 `pysqlite` 驱动程序，因此所有连接参数与 Pysqlite 的参数相同。

### 用户定义函数

aiosqlite 扩展了 pysqlite 来支持异步，因此我们可以在 Python 中创建自己的用户定义函数 (UDFs)，并直接在 SQLite 查询中使用它们，如此处所述: 用户定义函数。### 可串行化隔离/保存点/事务 DDL (asyncio 版本)

与 pysqlite 类似，aiosqlite 不支持 SAVEPOINT 功能。

解决方案类似于 可串行化隔离/保存点/事务 DDL。这通过 async 中的事件侦听器实现：

```py
from sqlalchemy import create_engine, event
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine("sqlite+aiosqlite:///myfile.db")

@event.listens_for(engine.sync_engine, "connect")
def do_connect(dbapi_connection, connection_record):
    # disable aiosqlite's emitting of the BEGIN statement entirely.
    # also stops it from emitting COMMIT before any DDL.
    dbapi_connection.isolation_level = None

@event.listens_for(engine.sync_engine, "begin")
def do_begin(conn):
    # emit our own BEGIN
    conn.exec_driver_sql("BEGIN")
```

警告

使用上述方法时，建议不要在 SQLite 驱动程序上使用 `Connection.execution_options.isolation_level` 设置，以及不要在 `Connection` 和 `create_engine()` 上使用，因为这个函数必然也会改变“isolation_level”设置。

### DBAPI

aiosqlite 的文档和下载信息 (如果适用) 可在此处找到: [`pypi.org/project/aiosqlite/`](https://pypi.org/project/aiosqlite/)

### 连接

连接字符串:

```py
sqlite+aiosqlite:///file_path
```

### 用户定义函数

aiosqlite 扩展了 pysqlite 来支持异步，因此我们可以在 Python 中创建自己的用户定义函数 (UDFs)，并直接在 SQLite 查询中使用它们，如此处所述: 用户定义函数。

### Serializable isolation / Savepoints / Transactional DDL（asyncio 版本）

与 pysqlite 类似，aiosqlite 不支持 SAVEPOINT 功能。

解决方案类似于 Serializable isolation / Savepoints / Transactional DDL。这是通过异步事件监听器实现的：

```py
from sqlalchemy import create_engine, event
from sqlalchemy.ext.asyncio import create_async_engine

engine = create_async_engine("sqlite+aiosqlite:///myfile.db")

@event.listens_for(engine.sync_engine, "connect")
def do_connect(dbapi_connection, connection_record):
    # disable aiosqlite's emitting of the BEGIN statement entirely.
    # also stops it from emitting COMMIT before any DDL.
    dbapi_connection.isolation_level = None

@event.listens_for(engine.sync_engine, "begin")
def do_begin(conn):
    # emit our own BEGIN
    conn.exec_driver_sql("BEGIN")
```

警告

当使用上述配方时，建议不要在 SQLite 驱动上使用`Connection.execution_options.isolation_level`设置，并且不要在`Connection`和`create_engine()`中使用，因为这个函数必然会改变“.isolation_level”设置。

## Pysqlcipher

通过 pysqlcipher 驱动支持 SQLite 数据库。

支持使用 [SQLCipher](https://www.zetetic.net/sqlcipher) 后端的 DBAPI 的方言。

### 连接

连接字符串：

```py
sqlite+pysqlcipher://:passphrase@/file_path[?kdf_iter=<iter>]
```

### 驱动程序

当前的方言选择逻辑是：

+   如果`create_engine.module`参数提供了一个 DBAPI 模块，则使用该模块。

+   否则对于 Python 3，选择 [`pypi.org/project/sqlcipher3/`](https://pypi.org/project/sqlcipher3/)

+   如果不可用，则回退到 [`pypi.org/project/pysqlcipher3/`](https://pypi.org/project/pysqlcipher3/)

+   对于 Python 2，使用 [`pypi.org/project/pysqlcipher/`](https://pypi.org/project/pysqlcipher/)。

警告

`pysqlcipher3` 和 `pysqlcipher` DBAPI 驱动已不再维护；截至目前为止，`sqlcipher3` 驱动似乎是最新的。为了未来的兼容性，可以使用任何与 pysqlcipher 兼容的 DBAPI，如下所示：

```py
import sqlcipher_compatible_driver

from sqlalchemy import create_engine

e = create_engine(
    "sqlite+pysqlcipher://:password@/dbname.db",
    module=sqlcipher_compatible_driver
)
```

这些驱动程序使用了 SQLCipher 引擎。该系统基本上引入了新的 PRAGMA 命令到 SQLite，这些命令允许设置密码和其他加密参数，从而允许对数据库文件进行加密。

### 连接字符串

连接字符串的格式在各方面与`pysqlite`驱动程序完全相同，只是现在接受“password”字段，其中应包含一个密码：

```py
e = create_engine('sqlite+pysqlcipher://:testing@/foo.db')
```

对于绝对文件路径，数据库名称应使用两个前导斜杠：

```py
e = create_engine('sqlite+pysqlcipher://:testing@//path/to/foo.db')
```

可以在查询字符串中传递由 SQLCipher 文档记录的一些额外的与加密相关的 PRAGMA，这将导致每个新连接调用该 PRAGMA。目前支持 `cipher`、`kdf_iter`、`cipher_page_size` 和 `cipher_use_hmac`：

```py
e = create_engine('sqlite+pysqlcipher://:testing@/foo.db?cipher=aes-256-cfb&kdf_iter=64000')
```

警告

先前版本的 sqlalchemy 没有考虑到 url 字符串中传递的与加密相关的 pragma，这些 pragma 被静默忽略。如果加密选项不匹配，这可能导致打开先前 sqlalchemy 版本保存的文件时出错。

### 池行为

该驱动对 pysqlite 的默认池行为进行了更改，如 Threading/Pooling Behavior 所述。观察到 pysqlcipher 驱动在连接方面明显比 pysqlite 驱动慢得多，很可能是由于加密开销，因此此处的方言默认使用 `SingletonThreadPool` 实现，而不是 pysqlite 使用的 `NullPool` 池。与以往一样，池实现完全可通过 `create_engine.poolclass` 参数进行配置；`StaticPool` 可能更适合单线程使用，或者可以使用 `NullPool` 来防止未加密的连接被保持打开长时间，但新连接的启动时间会变慢。

### 连接

连接字符串：

```py
sqlite+pysqlcipher://:passphrase@/file_path[?kdf_iter=<iter>]
```

### 驱动程序

当前方言选择逻辑为：

+   如果 `create_engine.module` 参数提供了一个 DBAPI 模块，则使用该模块。

+   否则对于 Python 3，选择 [`pypi.org/project/sqlcipher3/`](https://pypi.org/project/sqlcipher3/)

+   如果不可用，则回退到 [`pypi.org/project/pysqlcipher3/`](https://pypi.org/project/pysqlcipher3/)

+   对于 Python 2，使用 [`pypi.org/project/pysqlcipher/`](https://pypi.org/project/pysqlcipher/)。

警告

`pysqlcipher3` 和 `pysqlcipher` DBAPI 驱动已经不再维护；截至目前，`sqlcipher3` 驱动似乎是最新的。为了未来的兼容性，可以使用任何与 pysqlcipher 兼容的 DBAPI，如下所示：

```py
import sqlcipher_compatible_driver

from sqlalchemy import create_engine

e = create_engine(
    "sqlite+pysqlcipher://:password@/dbname.db",
    module=sqlcipher_compatible_driver
)
```

这些驱动程序使用了 SQLCipher 引擎。该系统基本上向 SQLite 引入了新的 PRAGMA 命令，允许设置密码短语和其他加密参数，从而使数据库文件被加密。

### 连接字符串

连接字符串的格式与`pysqlite`驱动完全相同，只是现在接受了“password”字段，其中应该包含一个密码短语：

```py
e = create_engine('sqlite+pysqlcipher://:testing@/foo.db')
```

对于绝对文件路径，应该在数据库名称前使用两个斜杠：

```py
e = create_engine('sqlite+pysqlcipher://:testing@//path/to/foo.db')
```

可以在查询字符串中传递一组额外的与加密相关的 SQLCipher 所支持的 pragma，详见[`www.zetetic.net/sqlcipher/sqlcipher-api/`](https://www.zetetic.net/sqlcipher/sqlcipher-api/)，并且会导致每个新连接调用该 PRAGMA。目前支持的有：`cipher`、`kdf_iter`、`cipher_page_size` 和 `cipher_use_hmac`。

```py
e = create_engine('sqlite+pysqlcipher://:testing@/foo.db?cipher=aes-256-cfb&kdf_iter=64000')
```

警告

先前版本的 SQLAlchemy 并未考虑传递在 URL 字符串中的与加密相关的 pragma，这些 pragma 被默默忽略。如果加密选项不匹配，这可能导致在打开之前由先前的 SQLAlchemy 版本保存的文件时出现错误。

### 池行为

驱动程序对 pysqlite 的默认池行为进行了更改，详见线程/池行为。观察到 pysqlcipher 驱动程序在连接时比 pysqlite 驱动程序慢得多，很可能是由于加密开销，因此这里的方言默认使用 `SingletonThreadPool` 实现，而不是 pysqlite 使用的 `NullPool` 池。与往常一样，池实现完全可配置，使用 `create_engine.poolclass` 参数；`StaticPool` 可能更适合单线程使用，或者 `NullPool` 可以用于防止未加密的连接长时间保持打开，但会牺牲新连接的启动速度。
