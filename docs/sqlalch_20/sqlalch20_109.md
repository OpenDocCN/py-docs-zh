# MySQL 和 MariaDB

> 原文：[`docs.sqlalchemy.org/en/20/dialects/mysql.html`](https://docs.sqlalchemy.org/en/20/dialects/mysql.html)

支持 MySQL / MariaDB 数据库。

以下表总结了数据库发布版本的当前支持级别。

**支持的 MySQL / MariaDB 版本**

| 支持类型 | 版本 |
| --- | --- |
| CI 中完全测试 | 5.6, 5.7, 8.0 / 10.8, 10.9 |
| 正常支持 | 5.6+ / 10+ |
| 尽力而为 | 5.0.2+ / 5.0.2+ |

## DBAPI 支持

提供以下方言/DBAPI 选项。请参考各自的 DBAPI 部分以获取连接信息。

+   mysqlclient（MySQL-Python 的维护分支）

+   PyMySQL

+   MariaDB Connector/Python

+   MySQL Connector/Python

+   asyncmy

+   aiomysql

+   CyMySQL

+   PyODBC

## 支持的版本和功能

SQLAlchemy 支持从版本 5.0.2 开始的 MySQL，以及所有现代版本的 MariaDB。有关任何给定服务器版本支持的功能的详细信息，请参阅官方 MySQL 文档。

从版本 1.4 开始更改：最低支持的 MySQL 版本现在是 5.0.2。

### MariaDB 支持

MySQL 的 MariaDB 变体保留了与 MySQL 协议的基本兼容性，但这两个产品的发展仍在分歧。在 SQLAlchemy 领域，这两个数据库有一小部分语法和行为上的差异，SQLAlchemy 会自动适应。要连接到 MariaDB 数据库，不需要对数据库 URL 进行任何更改：

```py
engine = create_engine("mysql+pymysql://user:pass@some_mariadb/dbname?charset=utf8mb4")
```

在首次连接时，SQLAlchemy 方言采用服务器版本检测方案，确定后端数据库是否报告为 MariaDB。根据此标志，方言可以在其行为必须不同的领域做出不同选择。

### 仅 MariaDB 模式

该方言还支持一个**可选**的“仅 MariaDB”连接模式，这对于应用程序使用 MariaDB 特定功能且与 MySQL 数据库不兼容的情况可能很有用。要使用此操作模式，请将上述 URL 中的“mysql”标记替换为“mariadb”：

```py
engine = create_engine("mariadb+pymysql://user:pass@some_mariadb/dbname?charset=utf8mb4")
```

上述引擎在首次连接时，如果服务器版本检测检测到后端数据库不是 MariaDB，则会引发错误。

当使用以`"mariadb"`作为方言名称的引擎时，**所有包含“mysql”名称的 mysql 特定选项现在都以`"mariadb"`命名**。这意味着选项如`mysql_engine`应该命名为`mariadb_engine`，等等。对于同时使用“mysql”和`"mariadb"`方言的应用程序，可以同时使用“mysql”和`"mariadb"`选项：

```py
my_table = Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("textdata", String(50)),
    mariadb_engine="InnoDB",
    mysql_engine="InnoDB",
)

Index(
    "textdata_ix",
    my_table.c.textdata,
    mysql_prefix="FULLTEXT",
    mariadb_prefix="FULLTEXT",
)
```

当上述结构被反映时，将发生类似的行为，即当数据库 URL 基于“mariadb”名称时，“mariadb”前缀将存在于选项名称中。

版本 1.4 中的新功能：添加了支持“MariaDB-only mode”的“mariadb”方言名称，用于 MySQL 方言。  ## 连接超时和断开连接

MySQL / MariaDB 具有自动连接关闭行为，对于空闲时间超过固定时间的连接，默认为 8 小时。为了避免出现此问题，使用`create_engine.pool_recycle`选项，该选项确保如果连接在池中存在了固定秒数，则该连接将被丢弃并替换为新连接：

```py
engine = create_engine('mysql+mysqldb://...', pool_recycle=3600)
```

为了更全面地检测连接池中的断开连接，包括适应服务器重启和网络问题，可以采用预先 ping 的方法。请参阅处理断开连接了解当前的方法。

另请参阅

处理断开连接 - 关于处理超时连接以及数据库重新启动的几种技术的背景。  ## 包括存储引擎的 CREATE TABLE 参数

MySQL 和 MariaDB 的 CREATE TABLE 语法都包括各种特殊选项，包括`ENGINE`、`CHARSET`、`MAX_ROWS`、`ROW_FORMAT`、`INSERT_METHOD`等等。为了适应这些参数的渲染，指定形式`mysql_argument_name="value"`。例如，要指定一个具有`ENGINE`为`InnoDB`、`CHARSET`为`utf8mb4`和`KEY_BLOCK_SIZE`为`1024`的表：

```py
Table('mytable', metadata,
      Column('data', String(32)),
      mysql_engine='InnoDB',
      mysql_charset='utf8mb4',
      mysql_key_block_size="1024"
     )
```

当支持仅 MariaDB 模式时，还必须包含相同的“mariadb”前缀下的键。这些值当然可以独立变化，以便在 MySQL 和 MariaDB 上保持不同的设置：

```py
# support both "mysql" and "mariadb-only" engine URLs

Table('mytable', metadata,
      Column('data', String(32)),

      mysql_engine='InnoDB',
      mariadb_engine='InnoDB',

      mysql_charset='utf8mb4',
      mariadb_charset='utf8',

      mysql_key_block_size="1024"
      mariadb_key_block_size="1024"

     )
```

MySQL / MariaDB 方言通常将任何指定为`mysql_keyword_name`的关键字转换为`CREATE TABLE`语句中的`KEYWORD_NAME`。其中一些名称将以空格而不是下划线呈现；为了支持此，MySQL 方言具有对这些特定名称的意识，其中包括`DATA DIRECTORY`（例如`mysql_data_directory`）、`CHARACTER SET`（例如`mysql_character_set`）和`INDEX DIRECTORY`（例如`mysql_index_directory`）。

最常见的参数是 `mysql_engine`，它指的是表的存储引擎。从历史上看，MySQL 服务器安装会将此值默认为 `MyISAM`，尽管较新版本可能将默认值设置为 `InnoDB`。`InnoDB` 引擎通常更受欢迎，因为它支持事务和外键。

在使用 `MyISAM` 存储引擎创建的 MySQL / MariaDB 数据库中创建的 `Table` 实际上是非事务性的，这意味着对该表的任何 INSERT/UPDATE/DELETE 语句都将被调用为自动提交。它也不支持外键约束；虽然 `CREATE TABLE` 语句接受外键选项，但在使用 `MyISAM` 存储引擎时，这些参数将被丢弃。反映这样的表也不会产生外键约束信息。

对于完全原子事务以及对外键约束的支持，所有参与的 `CREATE TABLE` 语句必须指定事务引擎，在绝大多数情况下是 `InnoDB`。

## 大小写敏感性和表反射

MySQL 和 MariaDB 都对大小写敏感的标识符名称提供不一致的支持，其支持基于底层操作系统的具体细节。但是，已经观察到无论存在何种大小写敏感性行为，外键声明中的表名称总是以全小写形式从数据库接收，这使得准确反映使用混合大小写标识符名称的相互关联表的架构成为不可能。

因此，强烈建议在 SQLAlchemy 中以及在 MySQL / MariaDB 数据库本身中将表名声明为全小写，特别是如果要使用数据库反射功能的话。

## 事务隔离级别

所有 MySQL / MariaDB 方言都支持通过方言特定参数 `create_engine.isolation_level`（由 `create_engine()` 接受）以及作为传递给 `Connection.execution_options()` 的参数的 `Connection.execution_options.isolation_level` 参数来设置事务隔离级别。此功能通过为每个新连接发出命令 `SET SESSION TRANSACTION ISOLATION LEVEL <level>` 来工作。对于特殊的 AUTOCOMMIT 隔离级别，使用了 DBAPI 特定的技术。

使用 `create_engine()` 设置隔离级别：

```py
engine = create_engine(
                "mysql+mysqldb://scott:tiger@localhost/test",
                isolation_level="READ UNCOMMITTED"
            )
```

通过每个连接执行选项进行设置：

```py
connection = engine.connect()
connection = connection.execution_options(
    isolation_level="READ COMMITTED"
)
```

`isolation_level`的有效值包括：

+   `READ COMMITTED`

+   `READ UNCOMMITTED`

+   `REPEATABLE READ`

+   `SERIALIZABLE`

+   `AUTOCOMMIT`

特殊的`AUTOCOMMIT`值利用特定 DBAPI 提供的各种“autocommit”属性，并且目前受到 MySQLdb、MySQL-Client、MySQL-Connector Python 和 PyMySQL 的支持。使用它，数据库连接将返回`SELECT @@autocommit;`的值为 true。

还有更多隔离级别配置选项，例如与主`Engine`关联的“子引擎”对象，每个对象应用不同的隔离级别设置。有关详情，请参阅设置事务隔离级别，包括 DBAPI 自动提交。

另请参见

设置事务隔离级别，包括 DBAPI 自动提交

## AUTO_INCREMENT 行为

在创建表时，SQLAlchemy 将自动在第一个未标记为外键的`Integer`主键列上设置`AUTO_INCREMENT`：

```py
>>> t = Table('mytable', metadata,
...   Column('mytable_id', Integer, primary_key=True)
... )
>>> t.create()
CREATE TABLE mytable (
 id INTEGER NOT NULL AUTO_INCREMENT,
 PRIMARY KEY (id)
)
```

您可以通过将`False`传递给`Column.autoincrement`参数的`Column`来禁用此行为。此标志也可用于在某些存储引擎中为多列键的次要列启用自动增量：

```py
Table('mytable', metadata,
      Column('gid', Integer, primary_key=True, autoincrement=False),
      Column('id', Integer, primary_key=True)
     )
```

## 服务器端游标

mysqlclient、PyMySQL、mariadbconnector 方言支持服务器端游标，并且可能也适用于其他方言。这可以通过使用“buffered=True/False”标志（如果可用）或通过在内部使用类似于`MySQLdb.cursors.SSCursor`或`pymysql.cursors.SSCursor`的类来实现。

服务器端游标可以通过使用`Connection.execution_options.stream_results`连接执行选项来启用每个语句的。

```py
with engine.connect() as conn:
    result = conn.execution_options(stream_results=True).execute(text("select * from table"))
```

请注意，某些类型的 SQL 语句可能不支持使用服务器端游标；通常，只应该使用返回行的 SQL 语句与此选项一起使用。

自版本 1.4 起弃用：dialect-level server_side_cursors 标志已弃用，并将在将来的版本中删除。请使用`Connection.stream_results`执行选项来支持无缓冲游标。

另请参见

使用服务器端游标（也称为流式结果）  ## Unicode

### 字符集选择

大多数 MySQL / MariaDB DBAPI 都提供了为连接设置客户端字符集的选项。通常可以使用 URL 中的`charset`参数来实现，例如：

```py
e = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4")
```

这个字符集是连接的**客户端字符集**。一些 MySQL DBAPI 会将其默认为诸如`latin1`之类的值，而一些则会使用`my.cnf`文件中的`default-character-set`设置。应该查阅所使用的 DBAPI 的文档以获取具体行为。

用于 Unicode 的编码传统上一直是`'utf8'`。然而，从 MySQL 版本 5.5.3 和 MariaDB 5.5 开始，引入了一个新的 MySQL 特定编码`'utf8mb4'`，而且自 MySQL 8.0 起，如果在任何服务器端指令中指定了纯`utf8`，服务器会发出警告，并用`utf8mb3`替换。之所以使用这种新编码的原因是因为 MySQL 的传统 utf-8 编码只支持三字节的代码点而不是四字节。因此，在与包含超过三字节大小的代码点的 MySQL 或 MariaDB 数据库通信时，如果数据库和客户端 DBAPI 都支持，首选使用这种新的字符集，如下所示：

```py
e = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4")
```

所有现代的 DBAPI 都应支持`utf8mb4`字符集。

为了对使用了传统`utf8`的模式使用`utf8mb4`编码，可能需要对 MySQL/MariaDB 模式和/或服务器配置进行更改。

另请参阅

[utf8mb4 字符集](https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html) - 在 MySQL 文档中

### 处理二进制数据警告和 Unicode

在写作本文时，MySQL 版本 5.6、5.7 和以后版本（不包括 MariaDB）现在在尝试将二进制数据传递到数据库时会发出警告，同时也设置了字符集编码，但是二进制数据本身对于该编码来说不合法：

```py
default.py:509: Warning: (1300, "Invalid utf8mb4 character string:
'F9876A'")
  cursor.execute(statement, parameters)
```

此警告是由于 MySQL 客户端库尝试将二进制字符串解释为 Unicode 对象，即使使用了诸如`LargeBinary`这样的数据类型也是如此。要解决此问题，SQL 语句在任何呈现如下的非 NULL 值之前都需要存在一个二进制的“字符集介绍”：

```py
INSERT INTO table (data) VALUES (_binary %s)
```

这些字符集介绍由 DBAPI 驱动程序提供，假设使用了 mysqlclient 或 PyMySQL（两者都是推荐的）。将查询字符串参数`binary_prefix=true`添加到 URL 中以修复此警告：

```py
# mysqlclient
engine = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/test?charset=utf8mb4&binary_prefix=true")

# PyMySQL
engine = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4&binary_prefix=true")
```

`binary_prefix`标志可能受到其他 MySQL 驱动程序的支持与否的影响。

SQLAlchemy 本身无法可靠地渲染这个`_binary`前缀，因为它不适用于 NULL 值，而绑定参数时 NULL 值是有效的。由于 MySQL 驱动程序将参数直接渲染到 SQL 字符串中，这是传递此附加关键字的最有效位置。

另请参阅

[字符集介绍](https://dev.mysql.com/doc/refman/5.7/en/charset-introducer.html) - 在 MySQL 网站上

## ANSI 引用风格

MySQL / MariaDB 具有两种标识符“引用风格”，一种使用反引号，另一种使用引号，例如 ``some_identifier`` vs. `"some_identifier"`。所有 MySQL 方言在首次使用特定 `Engine` 建立连接时，通过检查 sql_mode 的值来检测使用的版本。此引用风格在呈现表和列名称以及反映现有数据库结构时起作用。检测完全是自动的，不需要任何特殊配置来使用任一引用风格。

## 更改 sql_mode

MySQL 支持在多个 [服务器 SQL 模式](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html)下运行，对于服务器和客户端都是如此。要为给定应用程序更改 `sql_mode`，开发人员可以利用 SQLAlchemy 的事件系统。

在以下示例中，事件系统用于在 `first_connect` 和 `connect` 事件上设置 `sql_mode`：

```py
from sqlalchemy import create_engine, event

eng = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo='debug')

# `insert=True` will ensure this is the very first listener to run
@event.listens_for(eng, "connect", insert=True)
def connect(dbapi_connection, connection_record):
    cursor = dbapi_connection.cursor()
    cursor.execute("SET sql_mode = 'STRICT_ALL_TABLES'")

conn = eng.connect()
```

在上面示例中，当特定的 DBAPI 连接首次为给定的连接池创建时，“connect”事件将在连接可供连接池使用之前在连接上调用“SET”语句。此外，因为函数被注册为 `insert=True`，它将被添加到已注册函数的内部列表之前。

## MySQL / MariaDB SQL 扩展

许多 MySQL / MariaDB 的 SQL 扩展都通过 SQLAlchemy 的通用函数和操作符支持：

```py
table.select(table.c.password==func.md5('plaintext'))
table.select(table.c.username.op('regexp')('^[a-d]'))
```

当然，任何有效的 SQL 语句也可以作为字符串执行。

目前可以直接支持一些有限的 MySQL / MariaDB 对 SQL 的扩展。

+   INSERT..ON DUPLICATE KEY UPDATE：参见 INSERT…ON DUPLICATE KEY UPDATE (Upsert)

+   SELECT 命令，使用`Select.prefix_with()`和`Query.prefix_with()`：

    ```py
    select(...).prefix_with(['HIGH_PRIORITY', 'SQL_SMALL_RESULT'])
    ```

+   带有 LIMIT 的 UPDATE：

    ```py
    update(..., mysql_limit=10, mariadb_limit=10)
    ```

+   优化器提示，使用`Select.prefix_with()`和`Query.prefix_with()`：

    ```py
    select(...).prefix_with("/*+ NO_RANGE_OPTIMIZATION(t4 PRIMARY) */")
    ```

+   索引提示，使用`Select.with_hint()`和`Query.with_hint()`：

    ```py
    select(...).with_hint(some_table, "USE INDEX xyz")
    ```

+   MATCH 运算符支持：

    ```py
    from sqlalchemy.dialects.mysql import match
    select(...).where(match(col1, col2, against="some expr").in_boolean_mode())

    .. seealso::

        :class:`_mysql.match`
    ```

## INSERT/DELETE…RETURNING

MariaDB 方言支持 10.5+的`INSERT..RETURNING`和`DELETE..RETURNING`（10.0+）语法。在某些情况下，`INSERT..RETURNING`可以自动使用，以获取新生成的标识符，而不是使用`cursor.lastrowid`的传统方法，但是对于简单的单语句情况，目前仍更喜欢使用`cursor.lastrowid`，因为它的性能更好。

要在每个语句的基础上使用显式的`RETURNING`子句，请使用 _UpdateBase.returning()方法：

```py
# INSERT..RETURNING
result = connection.execute(
    table.insert().
    values(name='foo').
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

版本 2.0 中的新功能：增加了 MariaDB RETURNING 支持

## 插入…在重复键更新时（Upsert）

MySQL / MariaDB 允许通过 INSERT 语句的 ON DUPLICATE KEY UPDATE 子句将行“upserts”（更新或插入）到表中。只有在该行不匹配表中现有的主键或唯一键时，候选行才会被插入；否则，将执行更新。该语句允许分开指定要插入的值与要更新的值。

SQLAlchemy 通过 MySQL 特定的`insert()`函数提供`ON DUPLICATE KEY UPDATE`支持，该函数提供了生成方法`Insert.on_duplicate_key_update()`：

```py
>>> from sqlalchemy.dialects.mysql import insert

>>> insert_stmt = insert(my_table).values(
...     id='some_existing_id',
...     data='inserted value')

>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     data=insert_stmt.inserted.data,
...     status='U'
... )
>>> print(on_duplicate_key_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (%s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  VALUES(data),  status  =  %s 
```

不同于 PostgreSQL 的“ON CONFLICT”短语，"ON DUPLICATE KEY UPDATE"短语将始终匹配任何主键或唯一键，并且如果有匹配，将始终执行更新；它没有选项可以引发错误或跳过执行更新。

`ON DUPLICATE KEY UPDATE`用于对已存在的行执行更新，使用新值的任何组合以及提议插入的值。这些值通常使用传递给`Insert.on_duplicate_key_update()`的关键字参数指定为列键值（通常是列的名称，除非它指定了`Column.key`）作为键，字面值或 SQL 表达式作为值：

```py
>>> insert_stmt = insert(my_table).values(
...          id='some_existing_id',
...          data='inserted value')

>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     data="some data",
...     updated_at=func.current_timestamp(),
... )

>>> print(on_duplicate_key_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (%s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  %s,  updated_at  =  CURRENT_TIMESTAMP 
```

与`UpdateBase.values()`类似的方式，也接受其他参数形式，包括一个单一的字典：

```py
>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     {"data": "some data", "updated_at": func.current_timestamp()},
... )
```

以及一个 2-tuple 列表，它将自动提供类似于参数有序更新描述的参数有序 UPDATE 语句的方式。与`Update`对象不同，不需要特殊标志来指定意图，因为在此上下文中的参数形式是清楚的：

```py
>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     [
...         ("data", "some data"),
...         ("updated_at", func.current_timestamp()),
...     ]
... )

>>> print(on_duplicate_key_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (%s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  %s,  updated_at  =  CURRENT_TIMESTAMP 
```

版本 1.3 中的更改：支持 MySQL ON DUPLICATE KEY UPDATE 内的参数有序 UPDATE 子句

警告

`Insert.on_duplicate_key_update()` 方法 **不** 考虑 Python 端的默认 UPDATE 值或生成函数，例如使用 `Column.onupdate` 指定的值。这些值不会用于 ON DUPLICATE KEY 样式的 UPDATE，除非在参数中手动明确指定。

为了引用提议的插入行，`Insert.inserted` 特殊别名可作为 `Insert` 对象上的属性；此对象是一个 `ColumnCollection`，包含目标表的所有列：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh')

>>> do_update_stmt = stmt.on_duplicate_key_update(
...     data="updated value",
...     author=stmt.inserted.author
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (%s,  %s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  %s,  author  =  VALUES(author) 
```

渲染时，“inserted” 命名空间将生成表达式 `VALUES(<columnname>)`。

版本 1.2 中新增了对 MySQL ON DUPLICATE KEY UPDATE 子句的支持。

## rowcount 支持

SQLAlchemy 将 DBAPI `cursor.rowcount` 属性标准化为“UPDATE 或 DELETE 语句匹配的行数”的通常定义。这与大多数 MySQL DBAPI 驱动程序的默认设置相矛盾，后者是“实际修改/删除的行数”。因此，SQLAlchemy MySQL 方言在连接时始终添加 `constants.CLIENT.FOUND_ROWS` 标志，或者在目标方言上等效的标志。这个设置目前是硬编码的。

另见

`CursorResult.rowcount`

## MySQL / MariaDB 特定索引选项

可用于 MySQL 和 MariaDB 的 `Index` 构造的特定扩展。

### 索引长度

MySQL 和 MariaDB 都提供了一个选项，可以创建一定长度的索引条目，其中“长度”是指每个值中的字符数或字节数，这些值将成为索引的一部分。SQLAlchemy 通过 `mysql_length` 和/或 `mariadb_length` 参数提供了这个功能：

```py
Index('my_index', my_table.c.data, mysql_length=10, mariadb_length=10)

Index('a_b_idx', my_table.c.a, my_table.c.b, mysql_length={'a': 4,
                                                           'b': 9})

Index('a_b_idx', my_table.c.a, my_table.c.b, mariadb_length={'a': 4,
                                                           'b': 9})
```

对于非二进制字符串类型，前缀长度以字符表示，对于二进制字符串类型，以字节表示。传递给关键字参数的值 *必须* 是整数（因此对索引的所有列都指定相同的前缀长度值）或字典，在字典中，键是列名，值是相应列的前缀长度值。MySQL 和 MariaDB 仅允许对索引的列指定长度，如果它是 CHAR、VARCHAR、TEXT、BINARY、VARBINARY 和 BLOB 类型的列。

### 索引前缀

MySQL 存储引擎允许在创建索引时指定索引前缀。SQLAlchemy 通过`Index`的`mysql_prefix`参数提供了这个功能：

```py
Index('my_index', my_table.c.data, mysql_prefix='FULLTEXT')
```

传递给关键字参数的值将简单地传递给底层的 CREATE INDEX，因此它*必须*是您的 MySQL 存储引擎的有效索引前缀。

另请参阅

[CREATE INDEX](https://dev.mysql.com/doc/refman/5.0/en/create-index.html) - MySQL 文档

### 索引类型

一些 MySQL 存储引擎允许在创建索引或主键约束时指定索引类型。SQLAlchemy 通过`Index`的`mysql_using`参数提供了这个功能：

```py
Index('my_index', my_table.c.data, mysql_using='hash', mariadb_using='hash')
```

以及`PrimaryKeyConstraint`上的`mysql_using`参数：

```py
PrimaryKeyConstraint("data", mysql_using='hash', mariadb_using='hash')
```

传递给关键字参数的值将简单地传递给底层的 CREATE INDEX 或 PRIMARY KEY 子句，因此它*必须*是您的 MySQL 存储引擎的有效索引类型。

更多信息请查看：

[`dev.mysql.com/doc/refman/5.0/en/create-index.html`](https://dev.mysql.com/doc/refman/5.0/en/create-index.html)

[`dev.mysql.com/doc/refman/5.0/en/create-table.html`](https://dev.mysql.com/doc/refman/5.0/en/create-table.html)

### 索引解析器

MySQL 中的 CREATE FULLTEXT INDEX 也支持“WITH PARSER”选项。可以使用关键字参数`mysql_with_parser`来实现：

```py
Index(
    'my_index', my_table.c.data,
    mysql_prefix='FULLTEXT', mysql_with_parser="ngram",
    mariadb_prefix='FULLTEXT', mariadb_with_parser="ngram",
)
```

1.3 版本中的新功能  ## MySQL / MariaDB 外键

MySQL 和 MariaDB 关于外键的行为有一些重要的注意事项。

### 避免使用的外键参数

MySQL 和 MariaDB 都不支持外键参数“DEFERRABLE”、“INITIALLY”或“MATCH”。在`ForeignKeyConstraint`或`ForeignKey`中使用`deferrable`或`initially`关键字参数将导致这些关键字在 DDL 表达式中被渲染，然后在 MySQL 或 MariaDB 上引发错误。为了在外键上使用这些关键字，同时在 MySQL / MariaDB 后端上忽略它们，可以使用自定义编译规则：

```py
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.schema import ForeignKeyConstraint

@compiles(ForeignKeyConstraint, "mysql", "mariadb")
def process(element, compiler, **kw):
    element.deferrable = element.initially = None
    return compiler.visit_foreign_key_constraint(element, **kw)
```

“MATCH”关键字实际上更加隐蔽，而且在与 MySQL 或 MariaDB 后端一起使用时，SQLAlchemy 明确禁止使用。这个参数在 MySQL / MariaDB 中被静默忽略，但另外的效果是 ON UPDATE 和 ON DELETE 选项也被后端忽略。因此，不应该在 MySQL / MariaDB 后端使用 MATCH；与 DEFERRABLE 和 INITIALLY 一样，可以使用自定义编译规则来在 DDL 定义时纠正 ForeignKeyConstraint。

### 外键约束的反射

并非所有 MySQL / MariaDB 存储引擎都支持外键。在使用非常常见的 `MyISAM` MySQL 存储引擎时，通过表反射加载的信息将不包括外键。对于这些表，您可以在反射时提供一个 `ForeignKeyConstraint`：

```py
Table('mytable', metadata,
      ForeignKeyConstraint(['other_id'], ['othertable.other_id']),
      autoload_with=engine
     )
```

另请参阅

包括存储引擎的 CREATE TABLE 参数  ## MySQL / MariaDB 唯一约束和反射

SQLAlchemy 支持带有标志 `unique=True` 的 `Index` 构造，表示唯一索引，以及表示唯一约束的 `UniqueConstraint` 构造。在创建这些约束时，MySQL / MariaDB 支持这两种对象/语法。但是，MySQL / MariaDB 没有一个独立于唯一索引的唯一约束构造；也就是说，在 MySQL / MariaDB 上，“UNIQUE” 约束等同于创建一个“UNIQUE INDEX”。

在反射这些构造时，`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法都会在 MySQL / MariaDB 中为唯一索引返回一个条目。然而，在使用 `Table(..., autoload_with=engine)` 执行完整表反射时，`UniqueConstraint` 构造在任何情况下都不是完全反映的 `Table` 构造的一部分；这个构造总是由 `Table.indexes` 集合中存在 `unique=True` 设置的 `Index` 表示。

## TIMESTAMP / DATETIME 问题

### 为 MySQL / MariaDB 的 explicit_defaults_for_timestamp 启用 ON UPDATE CURRENT TIMESTAMP 渲染

MySQL / MariaDB 在历史上将 `TIMESTAMP` 数据类型的 DDL 扩展为短语“TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP”，其中包含非标准 SQL，当发生 UPDATE 时自动使用当前时间戳更新列，消除了在需要服务器端更新更改的情况下使用触发器的常规需求。

MySQL 5.6 引入了一个新的标志 [explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp)，它禁用了上述行为，在 MySQL 8 中，此标志默认为 true，这意味着为了获得 MySQL 的“更新时间戳”，而不改变此标志，上述 DDL 必须显式地呈现。此外，相同的 DDL 对于 `DATETIME` 数据类型也是有效的。

SQLAlchemy 的 MySQL 方言尚未提供生成 MySQL 的“ON UPDATE CURRENT_TIMESTAMP”子句的选项，注意这不是通用的“ON UPDATE”，因为标准 SQL 中没有这样的语法。SQLAlchemy 的 `Column.server_onupdate` 参数目前与此特殊的 MySQL 行为无关。

要生成此 DDL，请使用 `Column.server_default` 参数，并传递一个包含 ON UPDATE 子句的文本子句：

```py
from sqlalchemy import Table, MetaData, Column, Integer, String, TIMESTAMP
from sqlalchemy import text

metadata = MetaData()

mytable = Table(
    "mytable",
    metadata,
    Column('id', Integer, primary_key=True),
    Column('data', String(50)),
    Column(
        'last_updated',
        TIMESTAMP,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    )
)
```

相同的说明适用于使用 `DateTime` 和 `DATETIME` 数据类型：

```py
from sqlalchemy import DateTime

mytable = Table(
    "mytable",
    metadata,
    Column('id', Integer, primary_key=True),
    Column('data', String(50)),
    Column(
        'last_updated',
        DateTime,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    )
)
```

即使 `Column.server_onupdate` 功能不生成此 DDL，但仍然有可能希望向 ORM 发出信号，表示应该获取此更新的值。此语法如下所示：

```py
from sqlalchemy.schema import FetchedValue

class MyClass(Base):
    __tablename__ = 'mytable'

    id = Column(Integer, primary_key=True)
    data = Column(String(50))
    last_updated = Column(
        TIMESTAMP,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP"),
        server_onupdate=FetchedValue()
    )
```  ### TIMESTAMP 列和 NULL

MySQL 历史上要求指定 TIMESTAMP 数据类型的列隐式包括默认值 CURRENT_TIMESTAMP，即使没有明确说明，并且另外将列设置为 NOT NULL，这与所有其他数据类型相反的行为：

```py
mysql> CREATE TABLE ts_test (
    -> a INTEGER,
    -> b INTEGER NOT NULL,
    -> c TIMESTAMP,
    -> d TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -> e TIMESTAMP NULL);
Query OK, 0 rows affected (0.03 sec)

mysql> SHOW CREATE TABLE ts_test;
+---------+-----------------------------------------------------
| Table   | Create Table
+---------+-----------------------------------------------------
| ts_test | CREATE TABLE `ts_test` (
  `a` int(11) DEFAULT NULL,
  `b` int(11) NOT NULL,
  `c` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `d` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `e` timestamp NULL DEFAULT NULL
) ENGINE=MyISAM DEFAULT CHARSET=latin1
```

以上，我们看到一个 INTEGER 列默认为 NULL，除非指定为 NOT NULL。但是当列的类型为 TIMESTAMP 时，会生成一个隐式的默认值 CURRENT_TIMESTAMP，这也会强制使列成为 NOT NULL，即使我们没有这样指定。

MySQL 的这种行为可以通过 MySQL 方面的 [explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp) 配置标志在 MySQL 5.6 中引入。启用此服务器设置后，TIMESTAMP 列在 MySQL 方面的默认值和可空性方面的行为类似于任何其他数据类型。

然而，为了适应大多数未指定此新标志的 MySQL 数据库，SQLAlchemy 在任何未指定 `nullable=False` 的 TIMESTAMP 列中都显式地发出“NULL”指示符。为了适应指定了 `explicit_defaults_for_timestamp` 的新数据库，SQLAlchemy 还为指定了 `nullable=False` 的 TIMESTAMP 列发出 NOT NULL。以下示例说明了：

```py
from sqlalchemy import MetaData, Integer, Table, Column, text
from sqlalchemy.dialects.mysql import TIMESTAMP

m = MetaData()
t = Table('ts_test', m,
        Column('a', Integer),
        Column('b', Integer, nullable=False),
        Column('c', TIMESTAMP),
        Column('d', TIMESTAMP, nullable=False)
    )

from sqlalchemy import create_engine
e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo=True)
m.create_all(e)
```

输出：

```py
CREATE TABLE ts_test (
    a INTEGER,
    b INTEGER NOT NULL,
    c TIMESTAMP NULL,
    d TIMESTAMP NOT NULL
)
```

## MySQL SQL 构造

| 对象名称 | 描述 |
| --- | --- |
| match | 生成一个 `MATCH (X, Y) AGAINST ('TEXT')` 子句。 |

```py
class sqlalchemy.dialects.mysql.match
```

生成一个 `MATCH (X, Y) AGAINST ('TEXT')` 子句。

例如：

```py
from sqlalchemy import desc
from sqlalchemy.dialects.mysql import match

match_expr = match(
    users_table.c.firstname,
    users_table.c.lastname,
    against="Firstname Lastname",
)

stmt = (
    select(users_table)
    .where(match_expr.in_boolean_mode())
    .order_by(desc(match_expr))
)
```

将生成类似于 SQL 的代码：

```py
SELECT id, firstname, lastname
FROM user
WHERE MATCH(firstname, lastname) AGAINST (:param_1 IN BOOLEAN MODE)
ORDER BY MATCH(firstname, lastname) AGAINST (:param_2) DESC
```

`match()` 函数是所有 SQL 表达式可用的 `ColumnElement.match()` 方法的独立版本，与使用 `ColumnElement.match()` 时一样，但允许传递多个列。

参数：

+   `cols` – 要匹配的列表达式

+   `against` – 要比较的表达式

+   `in_boolean_mode` – 布尔值，将“布尔模式”设置为 true

+   `in_natural_language_mode` – 布尔值，将“自然语言”设置为 true

+   `with_query_expansion` – 布尔值，将“查询扩展”设置为 true

从版本 1.4.19 开始。

另请参阅

`ColumnElement.match()`

**成员**

in_boolean_mode(), in_natural_language_mode(), inherit_cache, with_query_expansion()

**类签名**

类 `sqlalchemy.dialects.mysql.match` (`sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.BinaryExpression`)。

```py
method in_boolean_mode() → Self
```

对 MATCH 表达式应用“IN BOOLEAN MODE”修饰符。

返回：

一个新的 `match` 实例，应用了修改。

```py
method in_natural_language_mode() → Self
```

对 MATCH 表达式应用“IN NATURAL LANGUAGE MODE”修饰符。

返回：

一个新的 `match` 实例，应用了修改。

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应该使用其直接超类使用的缓存密钥生成方案。

此属性默认为 `None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为 `False`，但还会发出警告。

如果 SQL 与对象对应的属性不基于该类本身的属性而变化，并且不是基于其超类，则可以在特定类上设置此标志为`True`。

另请参阅

启用自定义构造的缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method with_query_expansion() → Self
```

对 MATCH 表达式应用 “WITH QUERY EXPANSION” 修饰符。

返回：

一个具有应用修改的新 `match` 实例。

## MySQL 数据类型

与所有 SQLAlchemy 方言一样，已知与 MySQL 兼容的所有大写类型都可以从顶级方言导入：

```py
from sqlalchemy.dialects.mysql import (
    BIGINT,
    BINARY,
    BIT,
    BLOB,
    BOOLEAN,
    CHAR,
    DATE,
    DATETIME,
    DECIMAL,
    DECIMAL,
    DOUBLE,
    ENUM,
    FLOAT,
    INTEGER,
    LONGBLOB,
    LONGTEXT,
    MEDIUMBLOB,
    MEDIUMINT,
    MEDIUMTEXT,
    NCHAR,
    NUMERIC,
    NVARCHAR,
    REAL,
    SET,
    SMALLINT,
    TEXT,
    TIME,
    TIMESTAMP,
    TINYBLOB,
    TINYINT,
    TINYTEXT,
    VARBINARY,
    VARCHAR,
    YEAR,
)
```

特定于 MySQL 或具有 MySQL 特定构造参数的类型如下：

| 对象名称 | 描述 |
| --- | --- |
| BIGINT | MySQL BIGINTEGER 类型。 |
| BIT | MySQL BIT 类型。 |
| CHAR | MySQL CHAR 类型，用于固定长度的字符数据。 |
| DATETIME | MySQL DATETIME 类型。 |
| DECIMAL | MySQL DECIMAL 类型。 |
| ENUM | MySQL ENUM 类型。 |
| FLOAT | MySQL FLOAT 类型。 |
| INTEGER | MySQL INTEGER 类型。 |
| JSON | MySQL JSON 类型。 |
| LONGBLOB | MySQL LONGBLOB 类型，用于存储最多 2³² 字节的二进制数据。 |
| LONGTEXT | MySQL LONGTEXT 类型，用于存储编码长度达到 2³² 字节的字符数据。 |
| MEDIUMBLOB | MySQL MEDIUMBLOB 类型，用于存储最多 2²⁴ 字节的二进制数据。 |
| MEDIUMINT | MySQL MEDIUMINTEGER 类型。 |
| MEDIUMTEXT | MySQL MEDIUMTEXT 类型，用于存储编码长度达到 2²⁴ 字节的字符数据。 |
| NCHAR | MySQL NCHAR 类型。 |
| NUMERIC | MySQL NUMERIC 类型。 |
| NVARCHAR | MySQL NVARCHAR 类型。 |
| REAL | MySQL REAL 类型。 |
| SET | MySQL SET 类型。 |
| SMALLINT | MySQL SMALLINTEGER 类型。 |
| TIME | MySQL TIME 类型。 |
| TIMESTAMP | MySQL 的 TIMESTAMP 类型。 |
| TINYBLOB | MySQL 的 TINYBLOB 类型，用于最多 2⁸ 字节的二进制数据。 |
| TINYINT | MySQL 的 TINYINT 类型。 |
| TINYTEXT | MySQL 的 TINYTEXT 类型，用于最多 2⁸ 字节的字符存储。 |
| VARCHAR | MySQL 的 VARCHAR 类型，用于可变长度的字符数据。 |
| YEAR | MySQL 的 YEAR 类型，用于存储 1901-2155 年的单字节。 |

```py
class sqlalchemy.dialects.mysql.BIGINT
```

MySQL 的 BIGINTEGER 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.BIGINT`（`sqlalchemy.dialects.mysql.types._IntegerType`，`sqlalchemy.types.BIGINT`）

```py
method __init__(display_width=None, **kw)
```

构造一个 BIGINTEGER。

参数：

+   `display_width` – 可选项，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选项。

+   `zerofill` – 可选项。如果为 true，则值将作为左填充零的字符串存储。请注意，这不会影响底层数据库 API 返回的值，它们仍然是数字。

```py
class sqlalchemy.dialects.mysql.BINARY
```

SQL 的 BINARY 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.BINARY`（`sqlalchemy.types._Binary`）

```py
class sqlalchemy.dialects.mysql.BIT
```

MySQL 的 BIT 类型。

此类型适用于 MySQL 5.0.3 或更高版本的 MyISAM，以及 5.0.5 或更高版本的 MyISAM，MEMORY，InnoDB 和 BDB。对于较旧的版本，请使用 MSTinyInteger() 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.BIT`（`sqlalchemy.types.TypeEngine`）

```py
method __init__(length=None)
```

构造一个 BIT。

参数：

**length** – 可选项，位数。

```py
class sqlalchemy.dialects.mysql.BLOB
```

SQL 的 BLOB 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.BLOB`（`sqlalchemy.types.LargeBinary`）

```py
method __init__(length: int | None = None)
```

*继承自* `LargeBinary` *的* `sqlalchemy.types.LargeBinary.__init__` *方法*

构造一个 LargeBinary 类型。

参数：

**length** – 可选项，在 DDL 语句中用于列的长度，对于那些接受长度的二进制类型，比如 MySQL 的 BLOB 类型。

```py
class sqlalchemy.dialects.mysql.BOOLEAN
```

SQL 的 BOOLEAN 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.BOOLEAN`（`sqlalchemy.types.Boolean`）

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

*继承自* `Boolean` *的* `sqlalchemy.types.Boolean.__init__` *方法*

构造一个布尔值。

参数：

+   `create_constraint` –

    默认为 False。如果布尔值生成为 int/smallint，则还在表上创建一个 CHECK 约束，以确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有显式名称，以支持模式管理问题。这可以通过设置 `Boolean.name` 参数或设置适当的命名约定来实现；有关背景信息，请参阅配置约束命名约定。

    从版本 1.4 开始更改：- 此标志现在默认为 False，意味着对非本地枚举类型不生成 CHECK 约束。

+   `name` – 如果生成 CHECK 约束，则指定约束的名称。

```py
class sqlalchemy.dialects.mysql.CHAR
```

MySQL CHAR 类型，用于固定长度字符数据。

**成员**

__init__() 

**类签名**

类 `sqlalchemy.dialects.mysql.CHAR` (`sqlalchemy.dialects.mysql.types._StringType`, `sqlalchemy.types.CHAR`)

```py
method __init__(length=None, **kwargs)
```

构造一个 CHAR。

参数：

+   `length` – 最大数据长度，以字符为单位。

+   `binary` – 可选项，使用国家字符集的默认二进制排序。这不影响存储的数据类型，对于二进制数据，请使用 BINARY 类型。

+   `collation` – 可选项，请求特定的排序规则。必须与国家字符集兼容。

```py
class sqlalchemy.dialects.mysql.DATE
```

SQL DATE 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.DATE` (`sqlalchemy.types.Date`)

```py
class sqlalchemy.dialects.mysql.DATETIME
```

MySQL DATETIME 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.DATETIME` (`sqlalchemy.types.DATETIME`)

```py
method __init__(timezone=False, fsp=None)
```

构造一个 MySQL DATETIME 类型。

参数：

+   `timezone` – MySQL 方言不使用。

+   `fsp` –

    小数秒精度值。MySQL 5.6.4 支持存储小数秒；在为 DATETIME 类型生成 DDL 时将使用此参数。

    注意

    对于小数秒的 DBAPI 驱动程序支持可能有限；当前支持包括 MySQL Connector/Python。

```py
class sqlalchemy.dialects.mysql.DECIMAL
```

MySQL DECIMAL 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.DECIMAL` (`sqlalchemy.dialects.mysql.types._NumericType`, `sqlalchemy.types.DECIMAL`)

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构造 DECIMAL。

参数：

+   `precision` – 此数字中的总位数。如果 scale 和 precision 都为 None，则值存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选的。

+   `zerofill` – 可选的。 如果为真，则值将作为左填充零的字符串存储。 请注意，这不影响底层数据库 API 返回的值，后者仍然是数字。

```py
class sqlalchemy.dialects.mysql.DOUBLE
```

MySQL DOUBLE 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.DOUBLE` (`sqlalchemy.dialects.mysql.types._FloatType`, `sqlalchemy.types.DOUBLE`)

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构造一个 DOUBLE。

注意

`DOUBLE` 类型默认将浮点数转换为 Decimal，使用默认为 10 位的截断。 指定 `scale=n` 或 `decimal_return_scale=n` 以更改此比例，或指定 `asdecimal=False` 以直接将值返回为 Python 浮点数。

参数：

+   `precision` – 此数字中的总位数。 如果比例和精度都是无，则值将存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选的。

+   `zerofill` – 可选的。 如果为真，则值将作为左填充零的字符串存储。 请注意，这不影响底层数据库 API 返回的值，后者仍然是数字。

```py
class sqlalchemy.dialects.mysql.ENUM
```

MySQL ENUM 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.ENUM` (`sqlalchemy.types.NativeForEmulated`, `sqlalchemy.types.Enum`, `sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(*enums, **kw)
```

构造一个 ENUM。

例如：

```py
Column('myenum', ENUM("foo", "bar", "baz"))
```

参数：

+   `enums` –

    此 ENUM 的有效值范围。 在枚举中的值不带引号，生成模式时将被转义并用单引号括起来。 此对象还可以是符合 PEP-435 的枚举类型。

+   `strict` –

    此标志不起作用。

    版本中更改：MySQL ENUM 类型以及基本 Enum 类型现在验证所有 Python 数据值。

+   `charset` – 可选的，用于此字符串值的列级字符集。 优先于 ‘ascii’ 或 ‘unicode’ 简写。

+   `collation` – 可选的，用于此字符串值的列级排序。 优先于 ‘binary’ 简写。

+   `ascii` – 默认为 False：`latin1` 字符集的简写，生成模式中的 ASCII。

+   `unicode` – 默认为 False：`ucs2` 字符集的简写，生成模式中的 UNICODE。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制排序类型。 在模式中生成 BINARY。 这不影响存储的数据类型，只影响字符数据的排序。

```py
class sqlalchemy.dialects.mysql.FLOAT
```

MySQL FLOAT 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.FLOAT`（`sqlalchemy.dialects.mysql.types._FloatType`，`sqlalchemy.types.FLOAT`)。

```py
method __init__(precision=None, scale=None, asdecimal=False, **kw)
```

构造一个 FLOAT。

参数：

+   `precision` – 此数字中的总位数。如果 scale 和 precision 都为 None，则将值存储到服务器允许的限制。

+   `scale` – 小数点后的数字位数。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将作为左填充零的字符串存储。请注意，这不会影响底层数据库 API 返回的值，这些值仍然是数值型的。

```py
class sqlalchemy.dialects.mysql.INTEGER
```

MySQL 的 INTEGER 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.INTEGER`（`sqlalchemy.dialects.mysql.types._IntegerType`，`sqlalchemy.types.INTEGER`）。

```py
method __init__(display_width=None, **kw)
```

构造一个 INTEGER。

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将作为左填充零的字符串存储。请注意，这不会影响底层数据库 API 返回的值，这些值仍然是数值型的。

```py
class sqlalchemy.dialects.mysql.JSON
```

MySQL 的 JSON 类型。

从 5.7 版本开始，MySQL 支持 JSON。从 10.2 版本开始，MariaDB 支持 JSON（作为 LONGTEXT 的别名）。

`JSON`在针对 MySQL 或 MariaDB 后端使用基本的`JSON`数据类型时会自动使用。

另请参阅

`JSON` - 通用跨平台 JSON 数据类型的主要文档。

`JSON`类型支持将 JSON 值持久化，以及通过适应操作在数据库级别呈现`JSON_EXTRACT`函数所提供的核心索引操作。

**类签名**

类`sqlalchemy.dialects.mysql.JSON`（`sqlalchemy.types.JSON`）。

```py
class sqlalchemy.dialects.mysql.LONGBLOB
```

MySQL 的 LONGBLOB 类型，用于二进制数据长达 2³² 字节。

**类签名**

类`sqlalchemy.dialects.mysql.LONGBLOB`（`sqlalchemy.types._Binary`）。

```py
class sqlalchemy.dialects.mysql.LONGTEXT
```

MySQL 的 LONGTEXT 类型，用于存储编码长达 2³² 字节的字符。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.mysql.LONGTEXT` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(**kwargs)
```

构建一个 LONGTEXT。

参数：

+   `charset` – 可选，该字符串值的列级字符集。优先于 'ascii' 或 'unicode' 简写。

+   `collation` – 可选，该字符串值的列级排序规则。优先于 'binary' 简写。

+   `ascii` – 默认为 False：`latin1` 字符集的简写，在模式中生成 ASCII。

+   `unicode` – 默认为 False：`ucs2` 字符集的简写，在模式中生成 UNICODE。

+   `national` – 可选。如果为真，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列字符集匹配的二进制排序规则类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的排序规则。

```py
class sqlalchemy.dialects.mysql.MEDIUMBLOB
```

MySQL MEDIUMBLOB 类型，用于最多 2²⁴ 字节的二进制数据。

**类签名**

class `sqlalchemy.dialects.mysql.MEDIUMBLOB` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.MEDIUMINT
```

MySQL MEDIUMINTEGER 类型。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.mysql.MEDIUMINT` (`sqlalchemy.dialects.mysql.types._IntegerType`)

```py
method __init__(display_width=None, **kw)
```

构建一个 MEDIUMINTEGER

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为真，则将值存储为左填充的带零字符串。请注意，这不会影响底层数据库 API 返回的值，这些值仍然是数字。

```py
class sqlalchemy.dialects.mysql.MEDIUMTEXT
```

MySQL MEDIUMTEXT 类型，用于最多编码 2²⁴ 字节的字符存储。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.mysql.MEDIUMTEXT` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(**kwargs)
```

构建一个 MEDIUMTEXT。

参数：

+   `charset` – 可选，该字符串值的列级字符集。优先于 'ascii' 或 'unicode' 简写。

+   `collation` – 可选，该字符串值的列级排序规则。优先于 'binary' 简写。

+   `ascii` – 默认为 False：`latin1` 字符集的简写，在模式中生成 ASCII。

+   `unicode` – 默认为 False：`ucs2` 字符集的简写，在模式中生成 UNICODE。

+   `national` – 可选。如果为真，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列字符集匹配的二进制排序规则类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的排序规则。

```py
class sqlalchemy.dialects.mysql.NCHAR
```

MySQL NCHAR 类型。

对于服务器配置的国家字符集中的固定长度字符数据。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.NCHAR` (`sqlalchemy.dialects.mysql.types._StringType`, `sqlalchemy.types.NCHAR`)

```py
method __init__(length=None, **kwargs)
```

构造一个 NCHAR。

参数：

+   `length` – 最大数据长度，以字符为单位。

+   `binary` – 可选的，使用默认的二进制排序规则进行国家字符集。这不影响存储的数据类型，对于二进制数据，请使用 BINARY 类型。

+   `collation` – 可选的，请求特定的排序规则。必须与国家字符集兼容。

```py
class sqlalchemy.dialects.mysql.NUMERIC
```

MySQL NUMERIC 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.NUMERIC` (`sqlalchemy.dialects.mysql.types._NumericType`, `sqlalchemy.types.NUMERIC`)

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构造一个 NUMERIC。

参数：

+   `precision` – 此数字中的总位数。如果比例和精度都为 None，则将值存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选的。

+   `zerofill` – 可选的。如果为真，则值将存储为左边用零填充的字符串。请注意，这不会影响底层数据库 API 返回的值，该值仍为数字。

```py
class sqlalchemy.dialects.mysql.NVARCHAR
```

MySQL NVARCHAR 类型。

对于服务器配置的国家字符集中的可变长度字符数据。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.NVARCHAR` (`sqlalchemy.dialects.mysql.types._StringType`, `sqlalchemy.types.NVARCHAR`)

```py
method __init__(length=None, **kwargs)
```

构造一个 NVARCHAR。

参数：

+   `length` – 最大数据长度，以字符为单位。

+   `binary` – 可选的，使用默认的二进制排序规则进行国家字符集。这不影响存储的数据类型，对于二进制数据，请使用 BINARY 类型。

+   `collation` – 可选的，请求特定的排序规则。必须与国家字符集兼容。

```py
class sqlalchemy.dialects.mysql.REAL
```

MySQL REAL 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.REAL` (`sqlalchemy.dialects.mysql.types._FloatType`, `sqlalchemy.types.REAL`)

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构造一个 REAL。

注意

默认情况下，`REAL`类型从浮点数转换为 Decimal，使用默认为 10 位的截断。指定`scale=n`或`decimal_return_scale=n`以更改此比例，或者`asdecimal=False`以直接将值返回为 Python 浮点数。

参数：

+   `precision` – 此数字中的总位数。如果比例和精度都为 None，则值将存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将存储为左侧填充零的字符串。请注意，这不会影响底层数据库 API 返回的值，它们仍然是数值。

```py
class sqlalchemy.dialects.mysql.SET
```

MySQL SET 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.SET` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(*values, **kw)
```

构造一个 SET。

例如：

```py
Column('myset', SET("foo", "bar", "baz"))
```

在这种情况下，潜在值的列表是必需的，因为此集合将用于为表生成 DDL，或者如果设置了`SET.retrieve_as_bitwise`标志。

参数：

+   `values` – 此 SET 的有效值范围。这些值不带引号，生成模式时将被转义并用单引号括起。

+   `convert_unicode` – 与`String.convert_unicode`相同的标志。

+   `collation` – 与`String.collation`相同

+   `charset` – 与`VARCHAR.charset`相同。

+   `ascii` – 与`VARCHAR.ascii`相同。

+   `unicode` – 与`VARCHAR.unicode`相同。

+   `binary` – 与`VARCHAR.binary`相同。

+   `retrieve_as_bitwise` –

    如果为 True，则集合类型的数据将使用整数值持久化和选择，其中集合被强制转换为持久化的位掩码。MySQL 允许这种模式，它的优势在于能够明确存储值，例如空字符串`''`。数据类型将在 SELECT 语句中显示为表达式`col + 0`，以便将值强制转换为结果集中的整数值。如果希望持久化可以存储空字符串`''`作为值的集合，则需要此标志。

    警告

    在使用`SET.retrieve_as_bitwise`时，必须确保集合值的列表与 MySQL 数据库中的**完全相同的顺序**。

```py
class sqlalchemy.dialects.mysql.SMALLINT
```

MySQL SMALLINTEGER 类型。

**成员**

__init__()的初始化方法。

**类签名**

类`sqlalchemy.dialects.mysql.SMALLINT`（`sqlalchemy.dialects.mysql.types._IntegerType`，`sqlalchemy.types.SMALLINT`）

```py
method __init__(display_width=None, **kw)
```

构造一个 SMALLINTEGER。

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，值将以左侧填充零的字符串形式存储。请注意，这不会影响底层数据库 API 返回的值，其仍然是数值。

```py
class sqlalchemy.dialects.mysql.TEXT
```

MySQL TEXT 类型，用于编码最多 2¹⁶ 字节的字符存储。

**类签名**

类`sqlalchemy.dialects.mysql.TEXT`（`sqlalchemy.dialects.mysql.types._StringType`，`sqlalchemy.types.TEXT`）

```py
method __init__(length=None, **kw)
```

构造一个 TEXT。

参数：

+   `length` – 可选，如果提供，服务器可以通过替换足以存储`length`字节字符的最小 TEXT 类型来优化存储。

+   `charset` – 可选，用于此字符串值的列级字符集。优先于‘ascii’或‘unicode’简写。

+   `collation` – 可选，用于此字符串值的列级排序。优先于‘binary’简写。

+   `ascii` – 默认为 False：`latin1`字符集的简写，生成模式中的 ASCII。

+   `unicode` – 默认为 False：`ucs2`字符集的简写，生成模式中的 UNICODE。

+   `national` – 可选。如果为 true，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制排序类型。在模式中生成 BINARY。这不会影响存储的数据类型，只会影响字符数据的排序。

```py
class sqlalchemy.dialects.mysql.TIME
```

MySQL TIME 类型。

**成员**

__init__()的初始化方法。

**类签名**

类`sqlalchemy.dialects.mysql.TIME`（`sqlalchemy.types.TIME`）

```py
method __init__(timezone=False, fsp=None)
```

构造一个 MySQL TIME 类型。

参数：

+   `timezone` – MySQL 方言不使用。

+   `fsp` –

    分数秒精度值。MySQL 5.6 支持存储分数秒；在为 TIME 类型发出 DDL 时将使用此参数。

    注意

    DBAPI 驱动程序对分数秒的支持可能有限；当前支持包括 MySQL Connector/Python。

```py
class sqlalchemy.dialects.mysql.TIMESTAMP
```

MySQL TIMESTAMP 类型。

**成员**

__init__()的初始化方法。

**类签名**

类 `sqlalchemy.dialects.mysql.TIMESTAMP` (`sqlalchemy.types.TIMESTAMP`)

```py
method __init__(timezone=False, fsp=None)
```

构造一个 MySQL TIMESTAMP 类型。

参数：

+   `timezone` – MySQL 方言不使用。

+   `fsp` –

    小数秒精度值。MySQL 5.6.4 支持存储小数秒；在为 TIMESTAMP 类型发出 DDL 时将使用此参数。

    注意

    DBAPI 驱动程序对小数秒的支持可能有限；当前支持包括 MySQL Connector/Python。

```py
class sqlalchemy.dialects.mysql.TINYBLOB
```

MySQL TINYBLOB 类型，用于最多 2⁸ 字节的二进制数据。

**类签名**

类 `sqlalchemy.dialects.mysql.TINYBLOB` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.TINYINT
```

MySQL TINYINT 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.TINYINT` (`sqlalchemy.dialects.mysql.types._IntegerType`)

```py
method __init__(display_width=None, **kw)
```

构造一个 TINYINT。

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将作为左填充零的字符串存储。请注意，这不影响底层数据库 API 返回的值，其仍然是数值。

```py
class sqlalchemy.dialects.mysql.TINYTEXT
```

MySQL TINYTEXT 类型，用于编码最多 2⁸ 字节的字符存储。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.TINYTEXT` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(**kwargs)
```

构造一个 TINYTEXT。

参数：

+   `charset` – 可选，此字符串值的列级字符集。优先于 ‘ascii’ 或 ‘unicode’ 简写。

+   `collation` – 可选，此字符串值的列级排序规则。优先于 ‘binary’ 简写。

+   `ascii` – 默认为 False：`latin1` 字符集的简写，在模式中生成 ASCII。

+   `unicode` – 默认为 False：`ucs2` 字符集的简写，在模式中生成 UNICODE。

+   `national` – 可选。如果为 true，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制排序类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的排序。

```py
class sqlalchemy.dialects.mysql.VARBINARY
```

SQL VARBINARY 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.VARBINARY` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.VARCHAR
```

MySQL VARCHAR 类型，用于可变长度字符数据。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.VARCHAR`（`sqlalchemy.dialects.mysql.types._StringType`，`sqlalchemy.types.VARCHAR`）

```py
method __init__(length=None, **kwargs)
```

构造一个 VARCHAR。

参数：

+   `charset` – 可选，用于此字符串值的列级字符集。优先于‘ascii’或‘unicode’简写。

+   `collation` – 可选，用于此字符串值的列级排序。优先于‘binary’简写。

+   `ascii` – 默认为 False：`latin1`字符集的简写，模式中生成 ASCII。

+   `unicode` – 默认为 False：`ucs2`字符集的简写，模式中生成 UNICODE。

+   `national` – 可选。如果为 true，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制排序类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的排序。

```py
class sqlalchemy.dialects.mysql.YEAR
```

MySQL YEAR 类型，用于存储 1901-2155 年的单字节。

**类签名**

类`sqlalchemy.dialects.mysql.YEAR`（`sqlalchemy.types.TypeEngine`）

## MySQL DML 构造

| 对象名称 | 描述 |
| --- | --- |
| insert(table) | 构造一个 MySQL/MariaDB 特定变体`Insert`构造。 |
| Insert | MySQL 特定的 INSERT 实现。 |

```py
function sqlalchemy.dialects.mysql.insert(table: _DMLTableArgument) → Insert
```

构造一个 MySQL/MariaDB 特定变体`Insert`构造。

`sqlalchemy.dialects.mysql.insert()`函数创建一个`sqlalchemy.dialects.mysql.Insert`。这个类基于方言不可知的`Insert`构造，可以使用 SQLAlchemy Core 中的`insert()`函数构造。

`Insert`构造包括额外的方法`Insert.on_duplicate_key_update()`。

```py
class sqlalchemy.dialects.mysql.Insert
```

MySQL 特定的 INSERT 实现。

添加了针对 MySQL 特定语法的方法，例如 ON DUPLICATE KEY UPDATE。

使用 `sqlalchemy.dialects.mysql.insert()` 函数创建 `Insert` 对象。

版本 1.2 中的新功能。

**成员**

inherit_cache, inserted, on_duplicate_key_update()

**类签名**

class `sqlalchemy.dialects.mysql.Insert` (`sqlalchemy.sql.expression.Insert`)

```py
attribute inherit_cache: bool | None = False
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存密钥生成方案。

该属性默认为 `None`，表示构造尚未考虑是否适合参与缓存; 这在功能上等同于将值设置为 `False`，除了还会发出警告。

如果与对象对应的 SQL 不基于此类的本地属性而更改，并且不基于其超类，则可以在特定类上将此标志设置为 `True`。

另请参见

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
attribute inserted
```

为 ON DUPLICATE KEY UPDATE 语句提供“inserted”命名空间

MySQL 的 ON DUPLICATE KEY UPDATE 子句允许引用将要插入的行，通过一个称为 `VALUES()` 的特殊函数。 此属性提供了此行中的所有列，以便它们可在 ON DUPLICATE KEY UPDATE 子句中的 `VALUES()` 函数内部引用。 属性被命名为 `.inserted`，以避免与现有的 `Insert.values()` 方法冲突。

提示

`Insert.inserted` 属性是 `ColumnCollection` 的实例，其提供了与 访问表和列中描述的 `Table.c` 集合相同的接口。使用此集合，普通名称可以像属性一样访问（例如 `stmt.inserted.some_column`），但特殊名称和字典方法名称应使用索引访问，例如 `stmt.inserted["column name"]` 或 `stmt.inserted["values"]`。有关更多示例，请参阅 `ColumnCollection` 的文档字符串。

请参阅

插入…在重复键更新（Upsert）时 - 使用 `Insert.inserted` 的示例

```py
method on_duplicate_key_update(*args: Mapping[Any, Any] | List[Tuple[str, Any]] | ColumnCollection[Any, Any], **kw: Any) → Self
```

指定 ON DUPLICATE KEY UPDATE 子句。

参数：

****kw** – 与 UPDATE 值关联的列键。这些值可以是任何 SQL 表达式或支持的字面 Python 值。

警告

此字典**不**考虑 Python 指定的默认 UPDATE 值或生成函数，例如使用 `Column.onupdate` 指定的值。这些值不会被用于 ON DUPLICATE KEY UPDATE 类型的 UPDATE，除非在此手动指定值。

参数：

***args** –

作为传递键/值参数的替代方案，可以将字典或 2 元组列表作为单个位置参数传递。

传递单个字典等效于关键字参数形式：

```py
insert().on_duplicate_key_update({"name": "some name"})
```

传递 2 元组列表表示 UPDATE 子句中的参数分配应按发送的顺序排序，类似于参数有序更新中总体描述的 `Update` 构造：

```py
insert().on_duplicate_key_update(
    [("name", "some name"), ("value", "some value")])
```

版本 1.3 中的更改：参数可以指定为字典或 2 元组列表；后一种形式提供了参数排序。

版本 1.2 中的新功能。

请参阅

插入…在重复键更新（Upsert）时

## mysqlclient（MySQL-Python 的分支）

通过 mysqlclient（MySQL-Python 的维护分支）驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

mysqlclient（MySQL-Python 的维护分支）的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/mysqlclient/`](https://pypi.org/project/mysqlclient/)

### 连接

连接字符串：

```py
mysql+mysqldb://<user>:<password>@<host>[:<port>]/<dbname>
```

### 驱动程序状态

mysqlclient DBAPI 是不再维护的 [MySQL-Python](https://sourceforge.net/projects/mysql-python) DBAPI 的一个维护的分支。[mysqlclient](https://github.com/PyMySQL/mysqlclient-python) 支持 Python 2 和 Python 3，并且非常稳定。

### Unicode

有关当前关于 Unicode 处理的建议，请参阅 Unicode。### SSL 连接

mysqlclient 和 PyMySQL DBAPIs 接受一个额外的字典，其键为“ssl”，可以使用 `create_engine.connect_args` 字典来指定：

```py
engine = create_engine(
    "mysql+mysqldb://scott:tiger@192.168.0.134/test",
    connect_args={
        "ssl": {
            "ca": "/home/gord/client-ssl/ca.pem",
            "cert": "/home/gord/client-ssl/client-cert.pem",
            "key": "/home/gord/client-ssl/client-key.pem"
        }
    }
)
```

为了方便起见，以下键也可以内联指定在 URL 中，它们将被自动解释为“ssl”字典： “ssl_ca”、“ssl_cert”、“ssl_key”、“ssl_capath”、“ssl_cipher”、“ssl_check_hostname”。示例如下：

```py
connection_uri = (
    "mysql+mysqldb://scott:tiger@192.168.0.134/test"
    "?ssl_ca=/home/gord/client-ssl/ca.pem"
    "&ssl_cert=/home/gord/client-ssl/client-cert.pem"
    "&ssl_key=/home/gord/client-ssl/client-key.pem"
)
```

另请参阅

在 PyMySQL 方言中的 SSL 连接

### 使用 MySQLdb 与 Google Cloud SQL

Google Cloud SQL 现在建议使用 MySQLdb 方言。使用如下 URL 进行连接：

```py
mysql+mysqldb://root@/<dbname>?unix_socket=/cloudsql/<projectid>:<instancename>
```

### 服务器端游标

mysqldb 方言支持服务器端游标。请参阅 服务器端游标。## PyMySQL

通过 PyMySQL 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

PyMySQL 的文档和下载信息（如果适用）可在以下链接获取：[`pymysql.readthedocs.io/`](https://pymysql.readthedocs.io/)

### 连接

连接字符串：

```py
mysql+pymysql://<username>:<password>@<host>/<dbname>[?<options>]
```

### Unicode

有关当前关于 Unicode 处理的建议，请参阅 Unicode。

### SSL 连接

PyMySQL DBAPI 接受与 MySQLdb 相同的 SSL 参数，描述如 SSL 连接。请参阅该部分以获取其他示例。

如果服务器使用自动生成的自签名证书或与主机名不匹配（从客户端看），则在 PyMySQL 中也可能需要指示 `ssl_check_hostname=false`：

```py
connection_uri = (
    "mysql+pymysql://scott:tiger@192.168.0.134/test"
    "?ssl_ca=/home/gord/client-ssl/ca.pem"
    "&ssl_cert=/home/gord/client-ssl/client-cert.pem"
    "&ssl_key=/home/gord/client-ssl/client-key.pem"
    "&ssl_check_hostname=false"
)
```

### MySQL-Python 兼容性

pymysql DBAPI 是 MySQL-python（MySQLdb）驱动程序的纯 Python 移植版本，目标是 100% 的兼容性。大多数针对 MySQL-python 的行为说明也适用于 pymysql 驱动程序。## MariaDB-Connector

通过 MariaDB Connector/Python 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

MariaDB Connector/Python 的文档和下载信息（如果适用）可在以下链接获取：[`pypi.org/project/mariadb/`](https://pypi.org/project/mariadb/)

### 连接

连接字符串：

```py
mariadb+mariadbconnector://<user>:<password>@<host>[:<port>]/<dbname>
```

### 驱动程序状态

MariaDB Connector/Python 允许 Python 程序使用与 Python DB API 2.0（PEP-249）兼容的 API 访问 MariaDB 和 MySQL 数据库。它是用 C 编写的，使用 MariaDB Connector/C 客户端库进行客户端服务器通信。

注意 `mariadb://` 连接 URI 的默认驱动程序仍然是 `mysqldb`。要使用此驱动程序，需要使用 `mariadb+mariadbconnector://`。## MySQL-Connector

通过 MySQL Connector/Python 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

文档和 MySQL Connector/Python 的下载信息（如果适用）可在此处获取：[`pypi.org/project/mysql-connector-python/`](https://pypi.org/project/mysql-connector-python/)

### 连接

连接字符串：

```py
mysql+mysqlconnector://<user>:<password>@<host>[:<port>]/<dbname>
```

注意

自 MySQL Connector/Python 发布以来，DBAPI 存在许多问题，其中一些可能仍未解决，并且 mysqlconnector 方言 **未经过 SQLAlchemy 的持续集成测试**。推荐的 MySQL 方言是 mysqlclient 和 PyMySQL。 ## asyncmy

通过 asyncmy 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

文档和 asyncmy 的下载信息（如果适用）可在此处获取：[`github.com/long2ice/asyncmy`](https://github.com/long2ice/asyncmy)

### 连接

连接字符串：

```py
mysql+asyncmy://user:password@host:port/dbname[?key=value&key=value...]
```

使用特殊的 asyncio 中介层，asyncmy 方言可用作 SQLAlchemy asyncio 扩展包的后端。

此方言通常仅应与`create_async_engine()`引擎创建函数一起使用：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("mysql+asyncmy://user:pass@hostname/dbname?charset=utf8mb4")
```  ## aiomysql

通过 aiomysql 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

文档和 aiomysql 的下载信息（如果适用）可在此处获取：[`github.com/aio-libs/aiomysql`](https://github.com/aio-libs/aiomysql)

### 连接

连接字符串：

```py
mysql+aiomysql://user:password@host:port/dbname[?key=value&key=value...]
```

aiomysql 方言是 SQLAlchemy 的第二个 Python asyncio 方言。

使用特殊的 asyncio 中介层，aiomysql 方言可用作 SQLAlchemy asyncio 扩展包的后端。

此方言通常仅应与`create_async_engine()`引擎创建函数一起使用：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("mysql+aiomysql://user:pass@hostname/dbname?charset=utf8mb4")
```  ## cymysql

通过 CyMySQL 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

文档和 CyMySQL 的下载信息（如果适用）可在此处获取：[`github.com/nakagami/CyMySQL`](https://github.com/nakagami/CyMySQL)

### 连接

连接字符串：

```py
mysql+cymysql://<username>:<password>@<host>/<dbname>[?<options>]
```

注意

CyMySQL 方言 **未经过 SQLAlchemy 的持续集成测试**，可能存在未解决的问题。推荐的 MySQL 方言是 mysqlclient 和 PyMySQL。 ## pyodbc

通过 PyODBC 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

文档和 PyODBC 的下载信息（如果适用）可在此处获取：[`pypi.org/project/pyodbc/`](https://pypi.org/project/pyodbc/)

### 连接

连接字符串：

```py
mysql+pyodbc://<username>:<password>@<dsnname>
```

注意

MySQL 的 PyODBC 方言**未经过 SQLAlchemy 的持续集成测试**。推荐使用的 MySQL 方言是 mysqlclient 和 PyMySQL。但是，如果您想使用 mysql+pyodbc 方言并需要完全支持`utf8mb4`字符（包括表情符号等辅助字符），请确保使用当前版本的 MySQL Connector/ODBC 并在 DSN 或连接字符串中指定“ANSI”（**不是**“Unicode”）驱动程序版本。

通过精确的 pyodbc 连接字符串传递：

```py
import urllib
connection_string = (
    'DRIVER=MySQL ODBC 8.0 ANSI Driver;'
    'SERVER=localhost;'
    'PORT=3307;'
    'DATABASE=mydb;'
    'UID=root;'
    'PWD=(whatever);'
    'charset=utf8mb4;'
)
params = urllib.parse.quote_plus(connection_string)
connection_uri = "mysql+pyodbc:///?odbc_connect=%s" % params
```

支持 MySQL / MariaDB 数据库。

以下表总结了数据库发布版本的当前支持水平。

**支持的 MySQL / MariaDB 版本**

| 支持类型 | 版本 |
| --- | --- |
| 持续集成完全测试 | 5.6, 5.7, 8.0 / 10.8, 10.9 |
| 正常支持 | 5.6+ / 10+ |
| 尽力而为 | 5.0.2+ / 5.0.2+ |

## DBAPI 支持

提供以下方言/DBAPI 选项。有关连接信息，请参考各个 DBAPI 部分。

+   mysqlclient（MySQL-Python 的维护分支）

+   PyMySQL

+   MariaDB Connector/Python

+   MySQL Connector/Python

+   asyncmy

+   aiomysql

+   CyMySQL

+   PyODBC

## 支持的版本和功能

SQLAlchemy 支持从版本 5.0.2 开始的 MySQL，直至现代版本，以及所有现代版本的 MariaDB。有关任何给定服务器版本支持的功能的详细信息，请参阅官方 MySQL 文档。

从版本 1.4 开始更改：支持的最低 MySQL 版本现在是 5.0.2。

### MariaDB 支持

MariaDB 变种的 MySQL 保留了与 MySQL 协议的基本兼容性，但这两个产品的发展仍在分歧。在 SQLAlchemy 领域，这两个数据库有一些语法和行为上的差异，SQLAlchemy 会自动适应。要连接到 MariaDB 数据库，不需要对数据库 URL 进行任何更改：

```py
engine = create_engine("mysql+pymysql://user:pass@some_mariadb/dbname?charset=utf8mb4")
```

在首次连接时，SQLAlchemy 方言采用服务器版本检测方案，确定后端数据库是否报告为 MariaDB。根据此标志，方言可以在必须有不同行为的领域做出不同选择。

### 仅限 MariaDB 模式

该方言还支持**可选**的“仅限 MariaDB”连接模式，这对于应用程序使用 MariaDB 特定功能且与 MySQL 数据库不兼容的情况可能很有用。要使用此操作模式，请将上述 URL 中的“mysql”标记替换为“mariadb”：

```py
engine = create_engine("mariadb+pymysql://user:pass@some_mariadb/dbname?charset=utf8mb4")
```

在第一次连接时，上述引擎会在服务器版本检测检测到后端数据库不是 MariaDB 时引发错误。

当使用以 `"mariadb"` 为方言名称的引擎时，**所有包含名称 “mysql” 的 MySQL 特定选项现在都以 `"mariadb"` 命名**。这意味着像 `mysql_engine` 这样的选项应该命名为 `mariadb_engine`，等等。对于同时使用“mysql”和`"mariadb"`方言 URL 的应用程序，可以同时使用“mysql”和`"mariadb"`选项：

```py
my_table = Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("textdata", String(50)),
    mariadb_engine="InnoDB",
    mysql_engine="InnoDB",
)

Index(
    "textdata_ix",
    my_table.c.textdata,
    mysql_prefix="FULLTEXT",
    mariadb_prefix="FULLTEXT",
)
```

当上述结构反映时，会出现类似的行为，即当数据库 URL 基于“mariadb”名称时，选项名称中将包含“mariadb”前缀。

新版本中新增了“mariadb”方言名称，支持 MySQL 方言的“仅 MariaDB 模式”。

### MariaDB 支持

MySQL 的 MariaDB 变体保留了与 MySQL 协议的基本兼容性，但这两个产品的开发仍在分歧。在 SQLAlchemy 的领域内，这两个数据库有一些语法和行为上的小差异，SQLAlchemy 会自动适应。连接到 MariaDB 数据库时，不需要对数据库 URL 进行任何更改：

```py
engine = create_engine("mysql+pymysql://user:pass@some_mariadb/dbname?charset=utf8mb4")
```

在第一次连接时，SQLAlchemy 方言采用了一种服务器版本检测方案，以确定后端数据库是否报告为 MariaDB。根据此标志，方言可以在必须具有不同行为的领域中做出不同选择。

### 仅 MariaDB 模式

该方言还支持一种 **可选的** “仅 MariaDB” 连接模式，这在应用程序使用 MariaDB 特定功能且与 MySQL 数据库不兼容的情况下可能很有用。要使用此操作模式，请将上述 URL 中的 “mysql” 令牌替换为 “mariadb”：

```py
engine = create_engine("mariadb+pymysql://user:pass@some_mariadb/dbname?charset=utf8mb4")
```

在第一次连接时，上述引擎会在服务器版本检测检测到后端数据库不是 MariaDB 时引发错误。

当使用以 `"mariadb"` 为方言名称的引擎时，**所有包含名称 “mysql” 的 MySQL 特定选项现在都以 `"mariadb"` 命名**。这意味着像 `mysql_engine` 这样的选项应该命名为 `mariadb_engine`，等等。对于同时使用“mysql”和`"mariadb"`方言 URL 的应用程序，可以同时使用“mysql”和`"mariadb"`选项：

```py
my_table = Table(
    "mytable",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("textdata", String(50)),
    mariadb_engine="InnoDB",
    mysql_engine="InnoDB",
)

Index(
    "textdata_ix",
    my_table.c.textdata,
    mysql_prefix="FULLTEXT",
    mariadb_prefix="FULLTEXT",
)
```

当上述结构反映时，会出现类似的行为，即当数据库 URL 基于“mariadb”名称时，选项名称中将包含“mariadb”前缀。

新版本中新增了“mariadb”方言名称，支持 MySQL 方言的“仅 MariaDB 模式”。

## 连接超时和断开连接

MySQL / MariaDB 具有自动关闭连接行为，对于空闲一段固定时间的连接，默认为八小时。要避免出现此问题，可以使用`create_engine.pool_recycle` 选项，该选项确保如果连接在池中存在了固定数量的秒数，则将其丢弃并替换为新连接：

```py
engine = create_engine('mysql+mysqldb://...', pool_recycle=3600)
```

对于更全面的池化连接断开检测，包括适应服务器重启和网络问题，可以采用预先 ping 的方法。有关当前方法，请参阅处理断开连接。

另请参阅

处理断开连接 - 关于处理超时连接以及数据库重启的几种技术的背景。

## 包括存储引擎在内的 CREATE TABLE 参数

MySQL 和 MariaDB 的 CREATE TABLE 语法都包含许多特殊选项，包括`ENGINE`、`CHARSET`、`MAX_ROWS`、`ROW_FORMAT`、`INSERT_METHOD`等等。为了适应这些参数的渲染，需要指定形式`mysql_argument_name="value"`。例如，要指定一个具有`ENGINE`为`InnoDB`、`CHARSET`为`utf8mb4`和`KEY_BLOCK_SIZE`为`1024`的表：

```py
Table('mytable', metadata,
      Column('data', String(32)),
      mysql_engine='InnoDB',
      mysql_charset='utf8mb4',
      mysql_key_block_size="1024"
     )
```

在支持 仅限 MariaDB 模式 时，还必须包含对“mariadb”前缀的类似键。当然，值可以独立变化，以便可以维护 MySQL 与 MariaDB 上的不同设置：

```py
# support both "mysql" and "mariadb-only" engine URLs

Table('mytable', metadata,
      Column('data', String(32)),

      mysql_engine='InnoDB',
      mariadb_engine='InnoDB',

      mysql_charset='utf8mb4',
      mariadb_charset='utf8',

      mysql_key_block_size="1024"
      mariadb_key_block_size="1024"

     )
```

MySQL / MariaDB 方言通常会将指定为`mysql_keyword_name`的任何关键字转换为`CREATE TABLE`语句中的`KEYWORD_NAME`。其中少数名称将以空格而不是下划线呈现；为支持此功能，MySQL 方言具有对这些特定名称的认知，其中包括`DATA DIRECTORY`（例如`mysql_data_directory`）、`CHARACTER SET`（例如`mysql_character_set`）和`INDEX DIRECTORY`（例如`mysql_index_directory`）。

最常见的参数是`mysql_engine`，它指的是表格的存储引擎。历史上，MySQL 服务器安装通常默认将此值设置为`MyISAM`，尽管较新的版本可能默认为`InnoDB`。`InnoDB` 引擎通常更受欢迎，因为它支持事务和外键。

在 MySQL / MariaDB 数据库中创建的具有`MyISAM`存储引擎的`Table`将基本上是非事务性的，这意味着任何涉及此表的 INSERT/UPDATE/DELETE 语句都将被调用为自动提交。它也不支持外键约束；虽然`CREATE TABLE`语句接受外键选项，但在使用`MyISAM`存储引擎时，这些参数将被丢弃。反映这样一张表也不会产生外键约束信息。

为了完全原子性的事务以及支持外键约束，所有参与的`CREATE TABLE`语句必须指定一个事务性引擎，在绝大多数情况下是`InnoDB`。

## 大小写敏感和表反射

MySQL 和 MariaDB 都不一致地支持区分大小写的标识符名称，其支持基于底层操作系统的具体细节。然而，已经观察到，无论存在何种大小写敏感性行为，外键声明中的表名 *始终* 以全部小写的形式从数据库接收到，这使得无法准确反映使用混合大小写标识符名称的相互关联表的模式。

因此，强烈建议在 SQLAlchemy 中以及在 MySQL / MariaDB 数据库本身中将表名声明为全部小写，特别是如果要使用数据库反射功能的话。

## 事务隔离级别

所有 MySQL / MariaDB 方言都支持通过方言特定参数 `create_engine.isolation_level`（由 `create_engine()` 接受）以及作为传递给 `Connection.execution_options()` 的参数的 `Connection.execution_options.isolation_level` 参数来设置事务隔离级别。此功能通过为每个新连接发出命令 `SET SESSION TRANSACTION ISOLATION LEVEL <level>` 来工作。对于特殊的 AUTOCOMMIT 隔离级别，使用了特定于 DBAPI 的技术。

使用 `create_engine()` 设置隔离级别：

```py
engine = create_engine(
                "mysql+mysqldb://scott:tiger@localhost/test",
                isolation_level="READ UNCOMMITTED"
            )
```

使用每个连接的执行选项进行设置： 

```py
connection = engine.connect()
connection = connection.execution_options(
    isolation_level="READ COMMITTED"
)
```

`isolation_level`的有效值包括：

+   `READ COMMITTED`

+   `READ UNCOMMITTED`

+   `REPEATABLE READ`

+   `SERIALIZABLE`

+   `AUTOCOMMIT`

特殊值 `AUTOCOMMIT` 利用了特定 DBAPI 提供的各种“自动提交”属性，目前由 MySQLdb、MySQL-Client、MySQL-Connector Python 和 PyMySQL 支持。使用它，数据库连接将返回 `SELECT @@autocommit;` 的值为真。

还有更多隔离级别配置选项，例如与主 `Engine` 关联的“子引擎”对象，每个对象应用不同的隔离级别设置。请参阅 设置事务隔离级别，包括 DBAPI 自动提交 中的讨论。

另请参阅

设置事务隔离级别，包括 DBAPI 自动提交

## AUTO_INCREMENT 行为

在创建表时，SQLAlchemy 将自动在第一个未标记为外键的`Integer`主键列上设置`AUTO_INCREMENT`：

```py
>>> t = Table('mytable', metadata,
...   Column('mytable_id', Integer, primary_key=True)
... )
>>> t.create()
CREATE TABLE mytable (
 id INTEGER NOT NULL AUTO_INCREMENT,
 PRIMARY KEY (id)
)
```

通过将`False`传递给`Column.autoincrement`参数，您可以禁用此行为。此标志还可用于在某些存储引擎中启用多列键中的辅助列的自动增量：

```py
Table('mytable', metadata,
      Column('gid', Integer, primary_key=True, autoincrement=False),
      Column('id', Integer, primary_key=True)
     )
```

## 服务器端游标

服务器端游标支持适用于 mysqlclient、PyMySQL、mariadbconnector 方言，也可能适用于其他方言。如果可用，可以使用“buffered=True/False”标志，也可以在内部使用诸如`MySQLdb.cursors.SSCursor`或`pymysql.cursors.SSCursor`这样的类。

服务器端游标通过使用`Connection.execution_options.stream_results`连接执行选项基于语句来启用：

```py
with engine.connect() as conn:
    result = conn.execution_options(stream_results=True).execute(text("select * from table"))
```

请注意，某些类型的 SQL 语句可能不支持服务器端游标；通常，只应该使用返回行的 SQL 语句来使用此选项。

自版本 1.4 弃用：dialect-level server_side_cursors 标志已弃用，并将在未来版本中删除。请使用`Connection.stream_results`执行选项来支持无缓冲游标。

另请参阅

使用服务器端游标（也称为流式结果）

## Unicode

### 字符集选择

大多数 MySQL / MariaDB DBAPI 都提供了设置连接的客户端字符集的选项。这通常使用 URL 中的`charset`参数传递，例如：

```py
e = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4")
```

此字符集是连接的**客户端字符集**。某些 MySQL DBAPI 将默认将此设置为诸如`latin1`之类的值，而某些将使用`my.cnf`文件中的`default-character-set`设置。应该查阅正在使用的 DBAPI 的文档以获取特定的行为。

对于 Unicode 使用的编码传统上是`'utf8'`。然而，对于 MySQL 版本 5.5.3 和 MariaDB 5.5 以后的版本，引入了一个新的 MySQL 特定编码`'utf8mb4'`，而且从 MySQL 8.0 开始，如果在任何服务器端指令中指定了普通的`utf8`，服务器将发出警告，并替换为`utf8mb3`。引入这种新编码的原因是因为 MySQL 的传统 utf-8 编码仅支持最多三个字节的码点，而不是四个。因此，在与包含超过三个字节大小的码点的 MySQL 或 MariaDB 数据库通信时，如果数据库和客户端 DBAPI 都支持，优先使用这种新的字符集，如下所示：

```py
e = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4")
```

所有现代 DBAPI 应该支持`utf8mb4`字符集。

要在使用传统`utf8`创建的模式中使用`utf8mb4`编码，可能需要对 MySQL/MariaDB 模式和/或服务器配置进行更改。

另请参阅

[utf8mb4 字符集](https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html) - MySQL 文档中

### 处理二进制数据警告和 Unicode

MySQL 版本 5.6、5.7 和以后（在本文写作时不包括 MariaDB）现在在尝试将二进制数据传递到数据库时发出警告，而在二进制数据本身不适用于该编码时，也放置了字符集编码：

```py
default.py:509: Warning: (1300, "Invalid utf8mb4 character string:
'F9876A'")
  cursor.execute(statement, parameters)
```

此警告是因为 MySQL 客户端库试图将二进制字符串解释为 unicode 对象，即使使用了诸如`LargeBinary`之类的数据类型。为了解决此问题，SQL 语句需要在任何呈现如下的非 NULL 值之前存在一个二进制“字符集引导”：

```py
INSERT INTO table (data) VALUES (_binary %s)
```

这些字符集引导由 DBAPI 驱动程序提供，假设使用的是 mysqlclient 或 PyMySQL（两者都建议使用）。将查询字符串参数`binary_prefix=true`添加到 URL 中以修复此警告：

```py
# mysqlclient
engine = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/test?charset=utf8mb4&binary_prefix=true")

# PyMySQL
engine = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4&binary_prefix=true")
```

`binary_prefix`标志可能会或可能不会被其他 MySQL 驱动程序支持。

由于 MySQL 驱动程序直接将参数呈现到 SQL 字符串中，因此无法可靠地呈现此`_binary`前缀，因为它不会与 NULL 值一起使用，而 NULL 值可以作为绑定参数发送。由于 MySQL 驱动程序直接将参数呈现到 SQL 字符串中，因此这个附加关键字被传递的地方效率最高。

另请参阅

[字符集引导](https://dev.mysql.com/doc/refman/5.7/en/charset-introducer.html) - MySQL 网站上

### 字符集选择

大多数 MySQL / MariaDB DBAPI 都提供了为连接设置客户端字符集的选项。这通常使用 URL 中的`charset`参数传递，例如：

```py
e = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4")
```

这个字符集是**客户端字符集**用于连接。某些 MySQL DBAPI 将其默认为诸如`latin1`之类的值，有些将使用`my.cnf`文件中的`default-character-set`设置。应该咨询使用的 DBAPI 的文档以获取具体行为。

用于 Unicode 的编码传统上是`'utf8'`。然而，对于 MySQL 版本 5.5.3 和 MariaDB 5.5 以及更高版本，引入了一个新的 MySQL 特定编码`'utf8mb4'`，并且从 MySQL 8.0 开始，如果在任何服务器端指令中指定了普通的`utf8`，服务器将发出警告，并替换为`utf8mb3`。引入这种新编码的原因是因为 MySQL 的传统 utf-8 编码只支持最多三个字节的代码点，而不是四个。因此，当与包含超过三个字节大小的代码点的 MySQL 或 MariaDB 数据库通信时，如果数据库和客户端 DBAPI 都支持，首选使用这种新的字符集，如下所示：

```py
e = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4")
```

所有现代 DBAPI 应该支持`utf8mb4`字符集。

为了在使用了传统`utf8`创建的模式中使用`utf8mb4`编码，可能需要对 MySQL/MariaDB 模式和/或服务器配置进行更改。

另请参阅

[utf8mb4 字符集](https://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html) - 在 MySQL 文档中

### 处理二进制数据警告和 Unicode

MySQL 版本 5.6、5.7 及更高版本（在撰写本文时不包括 MariaDB）现在在尝试将二进制数据传递给数据库时发出警告，同时还存在字符集编码，当二进制数据本身对该编码无效时：

```py
default.py:509: Warning: (1300, "Invalid utf8mb4 character string:
'F9876A'")
  cursor.execute(statement, parameters)
```

此警告是由于 MySQL 客户端库试图将二进制字符串解释为 Unicode 对象，即使使用了诸如`LargeBinary`之类的数据类型。为了解决这个问题，SQL 语句在任何呈现如下的非 NULL 值之前需要存在一个二进制“字符集介绍”：

```py
INSERT INTO table (data) VALUES (_binary %s)
```

这些字符集介绍由 DBAPI 驱动程序提供，假设使用 mysqlclient 或 PyMySQL（两者都推荐）。在 URL 中添加查询字符串参数`binary_prefix=true`以修复此警告：

```py
# mysqlclient
engine = create_engine(
    "mysql+mysqldb://scott:tiger@localhost/test?charset=utf8mb4&binary_prefix=true")

# PyMySQL
engine = create_engine(
    "mysql+pymysql://scott:tiger@localhost/test?charset=utf8mb4&binary_prefix=true")
```

`binary_prefix`标志可能会或可能不会被其他 MySQL 驱动程序支持。

SQLAlchemy 本身无法可靠地呈现这个`_binary`前缀，因为它不适用于 NULL 值，而 NULL 值是可以作为绑定参数发送的。由于 MySQL 驱动程序直接将参数呈现到 SQL 字符串中，这是传递此附加关键字的最有效位置。

另请参阅

[字符集介绍](https://dev.mysql.com/doc/refman/5.7/en/charset-introducer.html) - 在 MySQL 网站上

## ANSI 引用风格

MySQL / MariaDB 有两种不同的标识符“引号样式”，一种使用反引号，另一种使用引号，例如``some_identifier`` vs. `"some_identifier"`。 所有 MySQL 方言通过检查在与特定`Engine`建立连接时的 sql_mode 的值来检测正在使用的版本。 当与特定池的给定 DBAPI 连接首次创建连接时，此引号样式用于渲染表和列名称以及反映现有数据库结构。 检测完全自动，不需要特殊配置来使用任何引号样式。

## 更改 sql_mode

MySQL 支持在服务器和客户端上运行多种[服务器 SQL 模式](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html)。 要更改给定应用程序的`sql_mode`，开发人员可以利用 SQLAlchemy 的事件系统。

在下面的示例中，事件系统用于在`first_connect`和`connect`事件上设置`sql_mode`：

```py
from sqlalchemy import create_engine, event

eng = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo='debug')

# `insert=True` will ensure this is the very first listener to run
@event.listens_for(eng, "connect", insert=True)
def connect(dbapi_connection, connection_record):
    cursor = dbapi_connection.cursor()
    cursor.execute("SET sql_mode = 'STRICT_ALL_TABLES'")

conn = eng.connect()
```

在上面说明的示例中，“connect”事件将在特定 DBAPI 连接首次为给定的池创建连接时在连接池将连接提供给连接池之前在连接上调用“SET”语句。 此外，因为函数已注册为`insert=True`，它将被添加到注册函数的内部列表的开头。

## MySQL / MariaDB SQL 扩展

许多 MySQL / MariaDB SQL 扩展都通过 SQLAlchemy 的通用函数和操作符支持：

```py
table.select(table.c.password==func.md5('plaintext'))
table.select(table.c.username.op('regexp')('^[a-d]'))
```

当然，任何有效的 SQL 语句也可以作为字符串执行。

目前有一些有限的直接支持 MySQL / MariaDB SQL 扩展到 SQL 的方法。

+   INSERT..ON DUPLICATE KEY UPDATE：参见 INSERT…ON DUPLICATE KEY UPDATE（Upsert）

+   SELECT pragma，请使用`Select.prefix_with()`和`Query.prefix_with()`：

    ```py
    select(...).prefix_with(['HIGH_PRIORITY', 'SQL_SMALL_RESULT'])
    ```

+   使用 LIMIT 的 UPDATE：

    ```py
    update(..., mysql_limit=10, mariadb_limit=10)
    ```

+   优化器提示，请使用`Select.prefix_with()`和`Query.prefix_with()`：

    ```py
    select(...).prefix_with("/*+ NO_RANGE_OPTIMIZATION(t4 PRIMARY) */")
    ```

+   索引提示，请使用`Select.with_hint()`和`Query.with_hint()`：

    ```py
    select(...).with_hint(some_table, "USE INDEX xyz")
    ```

+   MATCH 操作符支持：

    ```py
    from sqlalchemy.dialects.mysql import match
    select(...).where(match(col1, col2, against="some expr").in_boolean_mode())

    .. seealso::

        :class:`_mysql.match`
    ```

## INSERT/DELETE…RETURNING

MariaDB 方言支持 10.5+ 的`INSERT..RETURNING`和`DELETE..RETURNING`（10.0+）语法。`INSERT..RETURNING`可能会在某些情况下自动使用，以获取新生成的标识符，而不是使用`cursor.lastrowid`的传统方法，但是目前在简单的单语句情况下仍然更喜欢使用`cursor.lastrowid`，因为其性能更好。

要指定显式的`RETURNING`子句，请在每个语句上使用`_UpdateBase.returning()`方法：

```py
# INSERT..RETURNING
result = connection.execute(
    table.insert().
    values(name='foo').
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

2.0 版中的新功能：添加了对 MariaDB RETURNING 的支持

## INSERT…ON DUPLICATE KEY UPDATE（更新插入）

MySQL / MariaDB 允许通过`INSERT`语句的`ON DUPLICATE KEY UPDATE`子句将行“upsert”（更新或插入）到表中。只有候选行与表中现有的主键或唯一键不匹配时，才会插入候选行；否则，将执行更新。该语句允许单独指定要插入的值与要更新的值。

SQLAlchemy 通过 MySQL 特定的`insert()`函数提供`ON DUPLICATE KEY UPDATE`支持，该函数提供了生成方法`Insert.on_duplicate_key_update()`：

```py
>>> from sqlalchemy.dialects.mysql import insert

>>> insert_stmt = insert(my_table).values(
...     id='some_existing_id',
...     data='inserted value')

>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     data=insert_stmt.inserted.data,
...     status='U'
... )
>>> print(on_duplicate_key_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (%s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  VALUES(data),  status  =  %s 
```

与 PostgreSQL 的“ON CONFLICT”短语不同，“ON DUPLICATE KEY UPDATE”短语将始终匹配任何主键或唯一键，并且始终在匹配时执行 UPDATE；它没有选项可以引发错误或跳过执行 UPDATE。

`ON DUPLICATE KEY UPDATE`用于对已经存在的行执行更新，使用新值的任何组合以及建议插入的值。这些值通常使用关键字参数传递给`Insert.on_duplicate_key_update()`给定列键值（通常是列的名称，除非它指定`Column.key`）作为键和文字或 SQL 表达式作为值：

```py
>>> insert_stmt = insert(my_table).values(
...          id='some_existing_id',
...          data='inserted value')

>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     data="some data",
...     updated_at=func.current_timestamp(),
... )

>>> print(on_duplicate_key_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (%s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  %s,  updated_at  =  CURRENT_TIMESTAMP 
```

与`UpdateBase.values()`类似，还接受其他参数形式，包括单个字典：

```py
>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     {"data": "some data", "updated_at": func.current_timestamp()},
... )
```

以及一个 2 元组的列表，它将自动提供类似于参数顺序更新中描述的方法的参数排序 UPDATE 语句。与`Update`对象不同，不需要指定特殊标志来指定意图，因为此上下文中的参数形式是清晰明了的：

```py
>>> on_duplicate_key_stmt = insert_stmt.on_duplicate_key_update(
...     [
...         ("data", "some data"),
...         ("updated_at", func.current_timestamp()),
...     ]
... )

>>> print(on_duplicate_key_stmt)
INSERT  INTO  my_table  (id,  data)  VALUES  (%s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  %s,  updated_at  =  CURRENT_TIMESTAMP 
```

在 1.3 版中更改：支持 MySQL ON DUPLICATE KEY UPDATE 中的参数顺序 UPDATE 子句

警告

`Insert.on_duplicate_key_update()` 方法 **不** 考虑 Python 端的默认 UPDATE 值或生成函数，例如，那些使用 `Column.onupdate` 指定的值。这些值不会在 ON DUPLICATE KEY 样式的 UPDATE 中生效，除非它们在参数中手动指定。

为了引用所提出的插入行，特殊别名 `Insert.inserted` 可作为 `Insert` 对象的属性使用；这个对象是一个包含目标表所有列的 `ColumnCollection`：

```py
>>> stmt = insert(my_table).values(
...     id='some_id',
...     data='inserted value',
...     author='jlh')

>>> do_update_stmt = stmt.on_duplicate_key_update(
...     data="updated value",
...     author=stmt.inserted.author
... )

>>> print(do_update_stmt)
INSERT  INTO  my_table  (id,  data,  author)  VALUES  (%s,  %s,  %s)
ON  DUPLICATE  KEY  UPDATE  data  =  %s,  author  =  VALUES(author) 
```

渲染时，“inserted”命名空间将产生表达式 `VALUES(<columnname>)`。

新版本 1.2 中：增加了对 MySQL ON DUPLICATE KEY UPDATE 子句的支持

## 行数支持

SQLAlchemy 将 DBAPI `cursor.rowcount` 属性标准化为“UPDATE 或 DELETE 语句匹配的行数”的常规定义。这与大多数 MySQL DBAPI 驱动程序的默认设置相矛盾，后者是“实际修改/删除的行数”。因此，SQLAlchemy MySQL 方言总是在连接时添加 `constants.CLIENT.FOUND_ROWS` 标志，或者等效于目标方言的标志。这个设置目前是硬编码的。

另请参见

`CursorResult.rowcount`

## MySQL / MariaDB 特定的索引选项

可用的 MySQL 和 MariaDB 特定的 `Index` 构造扩展。

### 索引长度

MySQL 和 MariaDB 都提供了创建具有特定长度的索引条目的选项，这里的“长度”指的是每个值中将成为索引一部分的字符或字节的数量。SQLAlchemy 通过 `mysql_length` 和/或 `mariadb_length` 参数提供了这个特性：

```py
Index('my_index', my_table.c.data, mysql_length=10, mariadb_length=10)

Index('a_b_idx', my_table.c.a, my_table.c.b, mysql_length={'a': 4,
                                                           'b': 9})

Index('a_b_idx', my_table.c.a, my_table.c.b, mariadb_length={'a': 4,
                                                           'b': 9})
```

前缀长度以字符形式给出，用于非二进制字符串类型，以字节形式给出，用于二进制字符串类型。传递给关键字参数的值 *必须* 是一个整数（因此，为索引的所有列指定相同的前缀长度值），或者是一个字典，其中键是列名，值是相应列的前缀长度值。MySQL 和 MariaDB 仅允许索引列的长度为 CHAR、VARCHAR、TEXT、BINARY、VARBINARY 和 BLOB。

### 索引前缀

MySQL 存储引擎允许在创建索引时指定索引前缀。SQLAlchemy 通过 `Index` 的 `mysql_prefix` 参数提供了这个功能：

```py
Index('my_index', my_table.c.data, mysql_prefix='FULLTEXT')
```

传递给关键字参数的值将简单地传递给底层的 CREATE INDEX，因此它 *必须* 是你的 MySQL 存储引擎的有效索引前缀。

另请参阅

[创建索引](https://dev.mysql.com/doc/refman/5.0/en/create-index.html) - MySQL 文档

### 索引类型

一些 MySQL 存储引擎允许在创建索引或主键约束时指定索引类型。SQLAlchemy 通过 `Index` 的 `mysql_using` 参数提供了这个功能：

```py
Index('my_index', my_table.c.data, mysql_using='hash', mariadb_using='hash')
```

以及 `PrimaryKeyConstraint` 上的 `mysql_using` 参数：

```py
PrimaryKeyConstraint("data", mysql_using='hash', mariadb_using='hash')
```

传递给关键字参数的值将简单地传递给底层的 CREATE INDEX 或 PRIMARY KEY 子句，因此它 *必须* 是你的 MySQL 存储引擎的有效索引类型。

更多信息请参阅：

[`dev.mysql.com/doc/refman/5.0/en/create-index.html`](https://dev.mysql.com/doc/refman/5.0/en/create-index.html)

[`dev.mysql.com/doc/refman/5.0/en/create-table.html`](https://dev.mysql.com/doc/refman/5.0/en/create-table.html)

### 索引解析器

在 MySQL 中，CREATE FULLTEXT INDEX 还支持 “WITH PARSER” 选项。这可以使用关键字参数 `mysql_with_parser` 实现：

```py
Index(
    'my_index', my_table.c.data,
    mysql_prefix='FULLTEXT', mysql_with_parser="ngram",
    mariadb_prefix='FULLTEXT', mariadb_with_parser="ngram",
)
```

版本 1.3 中的新增内容。

### 索引长度

MySQL 和 MariaDB 都提供了创建带有一定长度的索引条目的选项，其中“长度”指的是将成为索引一部分的每个值中的字符或字节数。SQLAlchemy 通过 `mysql_length` 和/或 `mariadb_length` 参数提供了这个功能：

```py
Index('my_index', my_table.c.data, mysql_length=10, mariadb_length=10)

Index('a_b_idx', my_table.c.a, my_table.c.b, mysql_length={'a': 4,
                                                           'b': 9})

Index('a_b_idx', my_table.c.a, my_table.c.b, mariadb_length={'a': 4,
                                                           'b': 9})
```

非二进制字符串类型的字符给出前缀长度，二进制字符串类型的字节给出前缀长度。传递给关键字参数的值 *必须* 是整数（因此为所有索引列指定相同的前缀长度值）或字典，其中键是列名，值是对应列的前缀长度值。MySQL 和 MariaDB 只允许索引列的长度为 CHAR、VARCHAR、TEXT、BINARY、VARBINARY 和 BLOB 类型时指定长度。

### 索引前缀

MySQL 存储引擎允许在创建索引时指定索引前缀。SQLAlchemy 通过 `Index` 的 `mysql_prefix` 参数提供了这个功能：

```py
Index('my_index', my_table.c.data, mysql_prefix='FULLTEXT')
```

传递给关键字参数的值将简单地传递给底层的 CREATE INDEX，因此它 *必须* 是你的 MySQL 存储引擎的有效索引前缀。

另请参阅

[创建索引](https://dev.mysql.com/doc/refman/5.0/en/create-index.html) - MySQL 文档

### 索引类型

一些 MySQL 存储引擎允许您在创建索引或主键约束时指定索引类型。SQLAlchemy 通过`Index`上的`mysql_using`参数提供了此功能：

```py
Index('my_index', my_table.c.data, mysql_using='hash', mariadb_using='hash')
```

以及`PrimaryKeyConstraint`上的`mysql_using`参数：

```py
PrimaryKeyConstraint("data", mysql_using='hash', mariadb_using='hash')
```

传递给关键字参数的值将简单地传递给底层的 CREATE INDEX 或 PRIMARY KEY 子句，因此它*必须*是您的 MySQL 存储引擎的有效索引类型。

更多信息请参见：

[`dev.mysql.com/doc/refman/5.0/en/create-index.html`](https://dev.mysql.com/doc/refman/5.0/en/create-index.html)

[`dev.mysql.com/doc/refman/5.0/en/create-table.html`](https://dev.mysql.com/doc/refman/5.0/en/create-table.html)

### 索引解析器

MySQL 中的 CREATE FULLTEXT INDEX 也支持“WITH PARSER”选项。可以使用关键字参数`mysql_with_parser`：

```py
Index(
    'my_index', my_table.c.data,
    mysql_prefix='FULLTEXT', mysql_with_parser="ngram",
    mariadb_prefix='FULLTEXT', mariadb_with_parser="ngram",
)
```

1.3 版本中的新功能。

## MySQL / MariaDB 外键

MySQL 和 MariaDB 在外键方面的行为有一些重要的注意事项。

### 需要避免的外键参数

MySQL 和 MariaDB 都不支持外键参数“DEFERRABLE”、“INITIALLY”或“MATCH”。在`ForeignKeyConstraint`或`ForeignKey`上使用`deferrable`或`initially`关键字参数将导致这些关键字在 DDL 表达式中呈现，然后在 MySQL 或 MariaDB 上引发错误。为了在 MySQL / MariaDB 后端忽略这些关键字的情况下在外键上使用这些关键字，使用自定义编译规则：

```py
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.schema import ForeignKeyConstraint

@compiles(ForeignKeyConstraint, "mysql", "mariadb")
def process(element, compiler, **kw):
    element.deferrable = element.initially = None
    return compiler.visit_foreign_key_constraint(element, **kw)
```

"MATCH" 关键字实际上更加隐匿，且在与 MySQL 或 MariaDB 后端一起明确禁止。此参数被 MySQL / MariaDB 默默忽略，但此外还导致 ON UPDATE 和 ON DELETE 选项也被后端忽略。因此，永远不应该在 MySQL / MariaDB 后端使用 MATCH；与 DEFERRABLE 和 INITIALLY 一样，可以使用自定义编译规则在 DDL 定义时纠正 ForeignKeyConstraint。

### 外键约束的反射

并非所有 MySQL / MariaDB 存储引擎都支持外键。当使用非常常见的`MyISAM` MySQL 存储引擎时，表格反射加载的信息将不包括外键。对于这些表格，您可以在反射时提供`ForeignKeyConstraint`：

```py
Table('mytable', metadata,
      ForeignKeyConstraint(['other_id'], ['othertable.other_id']),
      autoload_with=engine
     )
```

另请参阅

创建表格参数，包括存储引擎

### 需要避免的外键参数

MySQL 和 MariaDB 都不支持外键参数“DEFERRABLE”、“INITIALLY”或“MATCH”。在 `ForeignKeyConstraint` 或 `ForeignKey` 中使用 `deferrable` 或 `initially` 关键字参数会使这些关键字在 DDL 表达式中呈现，然后在 MySQL 或 MariaDB 上引发错误。为了在外键上使用这些关键字，同时忽略 MySQL / MariaDB 后端上的它们，可以使用自定义编译规则：

```py
from sqlalchemy.ext.compiler import compiles
from sqlalchemy.schema import ForeignKeyConstraint

@compiles(ForeignKeyConstraint, "mysql", "mariadb")
def process(element, compiler, **kw):
    element.deferrable = element.initially = None
    return compiler.visit_foreign_key_constraint(element, **kw)
```

“MATCH” 关键字实际上更加阴险，并且在与 MySQL 或 MariaDB 后端一起使用时，SQLAlchemy 明确禁止使用此参数。这个参数被 MySQL / MariaDB 默默地忽略，但此外，还会导致后端也忽略 ON UPDATE 和 ON DELETE 选项。因此，不应该在 MySQL / MariaDB 后端使用 MATCH；与 DEFERRABLE 和 INITIALLY 一样，可以使用自定义编译规则来在 DDL 定义时纠正 ForeignKeyConstraint。

### 外键约束的反射

并非所有的 MySQL / MariaDB 存储引擎都支持外键。在使用非常常见的 `MyISAM` MySQL 存储引擎时，通过表反射加载的信息将不包括外键。对于这些表，可以在反射时提供 `ForeignKeyConstraint`：

```py
Table('mytable', metadata,
      ForeignKeyConstraint(['other_id'], ['othertable.other_id']),
      autoload_with=engine
     )
```

另请参阅

包括存储引擎的 CREATE TABLE 参数

## MySQL / MariaDB 唯一约束和反射

SQLAlchemy 支持带有标志 `unique=True` 的 `Index` 构造，表示唯一索引，以及表示唯一约束的 `UniqueConstraint` 构造。当发出 DDL 以创建这些约束时，MySQL / MariaDB 支持这两种对象/语法。然而，MySQL / MariaDB 没有与唯一索引分离的唯一约束构造；也就是说，在 MySQL / MariaDB 上，“UNIQUE” 约束等效于创建一个 “UNIQUE INDEX”。

当反射这些结构时，`Inspector.get_indexes()` 和 `Inspector.get_unique_constraints()` 方法都会在 MySQL / MariaDB 中为唯一索引返回一个条目。然而，当使用 `Table(..., autoload_with=engine)` 进行完整表反射时，`UniqueConstraint` 结构在任何情况下都不是完全反映的 `Table` 结构的一部分；这个结构总是由在 `Table.indexes` 集合中存在 `unique=True` 设置的 `Index` 表示。

## TIMESTAMP / DATETIME 问题

### 为 MySQL / MariaDB 的 explicit_defaults_for_timestamp 渲染 ON UPDATE CURRENT TIMESTAMP

MySQL / MariaDB 在历史上将 `TIMESTAMP` 数据类型的 DDL 扩展为短语 “TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP”，其中包含非标准 SQL，当发生 UPDATE 时自动更新列为当前时间戳，消除了在需要服务器端更新更改时通常需��使用触发器的情况。

MySQL 5.6 引入了一个新标志 [explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp)，它禁用了上述行为，在 MySQL 8 中，该标志默认为 true，这意味着为了获得一个 MySQL 的 “on update timestamp” 而不改变这个标志，上述 DDL 必须显式地渲染。此外，相同的 DDL 也适用于 `DATETIME` 数据类型。

SQLAlchemy 的 MySQL 方言目前还没有选项来生成 MySQL 的 “ON UPDATE CURRENT_TIMESTAMP” 子句，需要注意这不是一个通用的 “ON UPDATE”，因为标准 SQL 中没有这样的语法。SQLAlchemy 的 `Column.server_onupdate` 参数目前与这种特殊的 MySQL 行为无关。

要生成这个 DDL，请使用 `Column.server_default` 参数，并传递一个包含 ON UPDATE 子句的文本子句：

```py
from sqlalchemy import Table, MetaData, Column, Integer, String, TIMESTAMP
from sqlalchemy import text

metadata = MetaData()

mytable = Table(
    "mytable",
    metadata,
    Column('id', Integer, primary_key=True),
    Column('data', String(50)),
    Column(
        'last_updated',
        TIMESTAMP,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    )
)
```

使用`DateTime`和`DATETIME`数据类型也适用相同的说明：

```py
from sqlalchemy import DateTime

mytable = Table(
    "mytable",
    metadata,
    Column('id', Integer, primary_key=True),
    Column('data', String(50)),
    Column(
        'last_updated',
        DateTime,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    )
)
```

即使`Column.server_onupdate`功能不生成此 DDL，但仍然希望向 ORM 发出信号，表明应该获取此更新值。此语法如下所示：

```py
from sqlalchemy.schema import FetchedValue

class MyClass(Base):
    __tablename__ = 'mytable'

    id = Column(Integer, primary_key=True)
    data = Column(String(50))
    last_updated = Column(
        TIMESTAMP,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP"),
        server_onupdate=FetchedValue()
    )
```  ### TIMESTAMP 列和 NULL

MySQL 历史上强制要求指定 TIMESTAMP 数据类型的列隐式包含 CURRENT_TIMESTAMP 的默认值，即使没有明确说明，还将列设置为 NOT NULL，这与所有其他数据类型的行为相反：

```py
mysql> CREATE TABLE ts_test (
    -> a INTEGER,
    -> b INTEGER NOT NULL,
    -> c TIMESTAMP,
    -> d TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -> e TIMESTAMP NULL);
Query OK, 0 rows affected (0.03 sec)

mysql> SHOW CREATE TABLE ts_test;
+---------+-----------------------------------------------------
| Table   | Create Table
+---------+-----------------------------------------------------
| ts_test | CREATE TABLE `ts_test` (
  `a` int(11) DEFAULT NULL,
  `b` int(11) NOT NULL,
  `c` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `d` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `e` timestamp NULL DEFAULT NULL
) ENGINE=MyISAM DEFAULT CHARSET=latin1
```

如上所示，INTEGER 列默认为 NULL，除非指定为 NOT NULL。但是当列的类型为 TIMESTAMP 时，将生成一个 CURRENT_TIMESTAMP 的隐式默认值，这也会强制列为 NOT NULL，即使我们没有这样指定。

可以通过 MySQL 的 explicit_defaults_for_timestamp 配置标志在 MySQL 端更改此 MySQL 的行为，该标志在 MySQL 5.6 中引入。启用此服务器设置后，TIMESTAMP 列在 MySQL 端与默认值和可空性方面的行为与任何其他数据类型相同。

但是，为了适应大多数不指定此新标志的 MySQL 数据库，SQLAlchemy 会在不指定`nullable=False`的任何 TIMESTAMP 列中显式发出“NULL”说明符。为了适应指定了`explicit_defaults_for_timestamp`的较新数据库，SQLAlchemy 还会为指定了`nullable=False`的 TIMESTAMP 列发出 NOT NULL。以下示例说明了这一点：

```py
from sqlalchemy import MetaData, Integer, Table, Column, text
from sqlalchemy.dialects.mysql import TIMESTAMP

m = MetaData()
t = Table('ts_test', m,
        Column('a', Integer),
        Column('b', Integer, nullable=False),
        Column('c', TIMESTAMP),
        Column('d', TIMESTAMP, nullable=False)
    )

from sqlalchemy import create_engine
e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo=True)
m.create_all(e)
```

输出：

```py
CREATE TABLE ts_test (
    a INTEGER,
    b INTEGER NOT NULL,
    c TIMESTAMP NULL,
    d TIMESTAMP NOT NULL
)
```  ### 为 MySQL / MariaDB 的 explicit_defaults_for_timestamp 渲染 ON UPDATE CURRENT TIMESTAMP

MySQL / MariaDB 历史上扩展了 DDL，将`TIMESTAMP`数据类型扩展为“TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP”，其中包含非标准的 SQL，当发生 UPDATE 时自动更新列为当前时间戳，从而消除了在需要服务器端更新更改的情况下使用触发器的常规需求。

MySQL 5.6 引入了一个新标志[explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp)，禁用了上述行为，在 MySQL 8 中，此标志默认为 true，这意味着为了获得 MySQL 的“on update timestamp”而不更改此标志，必须显式呈现上述 DDL。此外，对于 DATETIME 数据类型，相同的 DDL 也是有效的。

SQLAlchemy 的 MySQL 方言目前还没有选项来生成 MySQL 的“ON UPDATE CURRENT_TIMESTAMP”子句，需要注意的是这不是一个通用的“ON UPDATE”，因为标准 SQL 中没有这样的语法。SQLAlchemy 的`Column.server_onupdate`参数目前与这种特殊的 MySQL 行为无关。

要生成这个 DDL，请使用`Column.server_default`参数，并传递一个包含 ON UPDATE 子句的文本子句：

```py
from sqlalchemy import Table, MetaData, Column, Integer, String, TIMESTAMP
from sqlalchemy import text

metadata = MetaData()

mytable = Table(
    "mytable",
    metadata,
    Column('id', Integer, primary_key=True),
    Column('data', String(50)),
    Column(
        'last_updated',
        TIMESTAMP,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    )
)
```

同样的指令适用于使用`DateTime`和`DATETIME`数据类型的情况：

```py
from sqlalchemy import DateTime

mytable = Table(
    "mytable",
    metadata,
    Column('id', Integer, primary_key=True),
    Column('data', String(50)),
    Column(
        'last_updated',
        DateTime,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP")
    )
)
```

即使`Column.server_onupdate`特性不生成这个 DDL，仍然有必要向 ORM 发出信号，表明应该获取这个更新后的值。语法如下所示：

```py
from sqlalchemy.schema import FetchedValue

class MyClass(Base):
    __tablename__ = 'mytable'

    id = Column(Integer, primary_key=True)
    data = Column(String(50))
    last_updated = Column(
        TIMESTAMP,
        server_default=text("CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP"),
        server_onupdate=FetchedValue()
    )
```

### TIMESTAMP 列和 NULL

MySQL 在历史上规定，指定 TIMESTAMP 数据类型的列隐含地包含了 CURRENT_TIMESTAMP 的默认值，即使没有明确说明，并且还将该列设置为 NOT NULL，与所有其他数据类型相反的行为：

```py
mysql> CREATE TABLE ts_test (
    -> a INTEGER,
    -> b INTEGER NOT NULL,
    -> c TIMESTAMP,
    -> d TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    -> e TIMESTAMP NULL);
Query OK, 0 rows affected (0.03 sec)

mysql> SHOW CREATE TABLE ts_test;
+---------+-----------------------------------------------------
| Table   | Create Table
+---------+-----------------------------------------------------
| ts_test | CREATE TABLE `ts_test` (
  `a` int(11) DEFAULT NULL,
  `b` int(11) NOT NULL,
  `c` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `d` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `e` timestamp NULL DEFAULT NULL
) ENGINE=MyISAM DEFAULT CHARSET=latin1
```

在上面的例子中，我们看到一个 INTEGER 列默认为 NULL，除非指定为 NOT NULL。但是当列的类型为 TIMESTAMP 时，会生成一个隐含的默认值 CURRENT_TIMESTAMP，这也会强制将列设置为 NOT NULL，即使我们没有明确指定。

MySQL 的这种行为可以通过 MySQL 端使用 MySQL 5.6 引入的[explicit_defaults_for_timestamp](https://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_explicit_defaults_for_timestamp)配置标志来更改。启用此服务器设置后，TIMESTAMP 列在 MySQL 端的默认值和可空性方面的行为与任何其他数据类型相同。

然而，为了适应大多数不指定此新标志的 MySQL 数据库，SQLAlchemy 对于不指定 `nullable=False` 的任何 TIMESTAMP 列都显式地发出“NULL”指定符。为了适应指定了 `nullable=False` 的 TIMESTAMP 列的新数据库，SQLAlchemy 还为这些列发出 NOT NULL。以下示例说明了这一点：

```py
from sqlalchemy import MetaData, Integer, Table, Column, text
from sqlalchemy.dialects.mysql import TIMESTAMP

m = MetaData()
t = Table('ts_test', m,
        Column('a', Integer),
        Column('b', Integer, nullable=False),
        Column('c', TIMESTAMP),
        Column('d', TIMESTAMP, nullable=False)
    )

from sqlalchemy import create_engine
e = create_engine("mysql+mysqldb://scott:tiger@localhost/test", echo=True)
m.create_all(e)
```

输出：

```py
CREATE TABLE ts_test (
    a INTEGER,
    b INTEGER NOT NULL,
    c TIMESTAMP NULL,
    d TIMESTAMP NOT NULL
)
```

## MySQL SQL 构造

| 对象名称 | 描述 |
| --- | --- |
| match | 生成 `MATCH (X, Y) AGAINST ('TEXT')` 子句。 |

```py
class sqlalchemy.dialects.mysql.match
```

生成 `MATCH (X, Y) AGAINST ('TEXT')` 子句。

例如：

```py
from sqlalchemy import desc
from sqlalchemy.dialects.mysql import match

match_expr = match(
    users_table.c.firstname,
    users_table.c.lastname,
    against="Firstname Lastname",
)

stmt = (
    select(users_table)
    .where(match_expr.in_boolean_mode())
    .order_by(desc(match_expr))
)
```

将产生类似于以下的 SQL：

```py
SELECT id, firstname, lastname
FROM user
WHERE MATCH(firstname, lastname) AGAINST (:param_1 IN BOOLEAN MODE)
ORDER BY MATCH(firstname, lastname) AGAINST (:param_2) DESC
```

`match()` 函数是所有 SQL 表达式上都可用的 `ColumnElement.match()` 方法的独立版本，与使用 `ColumnElement.match()` 时相同，但允许传递多个列

参数：

+   `cols` – 要匹配的列表达式

+   `against` – 要比较的表达式

+   `in_boolean_mode` – 布尔值，将“布尔模式”设置为真

+   `in_natural_language_mode` – 布尔值，将“自然语言”设置为真

+   `with_query_expansion` – 布尔值，将“查询扩展”设置为真

版本 1.4.19 中新增。

另请参阅

`ColumnElement.match()`

**成员**

in_boolean_mode(), in_natural_language_mode(), inherit_cache, with_query_expansion()

**类签名**

类 `sqlalchemy.dialects.mysql.match` (`sqlalchemy.sql.expression.Generative`, `sqlalchemy.sql.expression.BinaryExpression`)

```py
method in_boolean_mode() → Self
```

将“IN BOOLEAN MODE”修饰符应用于 MATCH 表达式。

返回：

带有修改的新 `match` 实例。

```py
method in_natural_language_mode() → Self
```

将“IN NATURAL LANGUAGE MODE”修饰符应用于 MATCH 表达式。

返回：

带有修改的新 `match` 实例。

```py
attribute inherit_cache: bool | None = True
```

指示此 `HasCacheKey` 实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为 `None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等效于将值设置为 `False`，只是还会发出警告。

如果 SQL 与对象对应的类没有基于该类本地属性而不是其超类发生变化，则可以将此标志设置为 `True`。

另请参阅

为自定义构造启用缓存支持 - 设置第三方或用户定义的 SQL 构造的 `HasCacheKey.inherit_cache` 属性的一般指南。

```py
method with_query_expansion() → Self
```

对 MATCH 表达式应用 “WITH QUERY EXPANSION” 修饰符。

返回：

应用了修改的 `match` 实例。

## MySQL 数据类型

与所有 SQLAlchemy 方言一样，已知与 MySQL 有效的所有大写类型都可以从顶级方言导入：

```py
from sqlalchemy.dialects.mysql import (
    BIGINT,
    BINARY,
    BIT,
    BLOB,
    BOOLEAN,
    CHAR,
    DATE,
    DATETIME,
    DECIMAL,
    DECIMAL,
    DOUBLE,
    ENUM,
    FLOAT,
    INTEGER,
    LONGBLOB,
    LONGTEXT,
    MEDIUMBLOB,
    MEDIUMINT,
    MEDIUMTEXT,
    NCHAR,
    NUMERIC,
    NVARCHAR,
    REAL,
    SET,
    SMALLINT,
    TEXT,
    TIME,
    TIMESTAMP,
    TINYBLOB,
    TINYINT,
    TINYTEXT,
    VARBINARY,
    VARCHAR,
    YEAR,
)
```

MySQL 特有的类型，或具有特定于 MySQL 的构造参数的类型如下：

| 对象名称 | 描述 |
| --- | --- |
| BIGINT | MySQL BIGINTEGER 类型。 |
| BIT | MySQL BIT 类型。 |
| CHAR | MySQL CHAR 类型，用于固定长度的字符数据。 |
| DATETIME | MySQL DATETIME 类型。 |
| DECIMAL | MySQL DECIMAL 类型。 |
| ENUM | MySQL ENUM 类型。 |
| FLOAT | MySQL FLOAT 类型。 |
| INTEGER | MySQL INTEGER 类型。 |
| JSON | MySQL JSON 类型。 |
| LONGBLOB | MySQL LONGBLOB 类型，用于最多 2³² 字节的二进制数据。 |
| LONGTEXT | MySQL LONGTEXT 类型，用于最多编码为 2³² 字节的字符存储。 |
| MEDIUMBLOB | MySQL MEDIUMBLOB 类型，用于最多 2²⁴ 字节的二进制数据。 |
| MEDIUMINT | MySQL MEDIUMINTEGER 类型。 |
| MEDIUMTEXT | MySQL MEDIUMTEXT 类型，用于最多编码为 2²⁴ 字节的字符存储。 |
| NCHAR | MySQL NCHAR 类型。 |
| NUMERIC | MySQL NUMERIC 类型。 |
| NVARCHAR | MySQL NVARCHAR 类型。 |
| REAL | MySQL REAL 类型。 |
| SET | MySQL SET 类型。 |
| SMALLINT | MySQL SMALLINTEGER 类型。 |
| TIME | MySQL TIME 类型。 |
| TIMESTAMP | MySQL TIMESTAMP 类型。 |
| TINYBLOB | MySQL TINYBLOB 类型，用于存储最多 2⁸ 字节的二进制数据。 |
| TINYINT | MySQL TINYINT 类型。 |
| TINYTEXT | MySQL TINYTEXT 类型，用于存储编码最多 2⁸ 字节的字符数据。 |
| VARCHAR | MySQL VARCHAR 类型，用于存储可变长度的字符数据。 |
| YEAR | MySQL YEAR 类型，用于单字节存储 1901 年至 2155 年之间的年份。 |

```py
class sqlalchemy.dialects.mysql.BIGINT
```

MySQL BIGINTEGER 类型。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.mysql.BIGINT` (`sqlalchemy.dialects.mysql.types._IntegerType`, `sqlalchemy.types.BIGINT`)

```py
method __init__(display_width=None, **kw)
```

构造一个 BIGINTEGER。

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将以左填充零的字符串形式存储。注意，这不会影响底层数据库 API 返回的值，其仍然是数值。

```py
class sqlalchemy.dialects.mysql.BINARY
```

SQL BINARY 类型。

**类签名**

class `sqlalchemy.dialects.mysql.BINARY` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.BIT
```

MySQL BIT 类型。

此类型适用于 MySQL 5.0.3 或更高版本的 MyISAM，并且适用于 5.0.5 或更高版本的 MyISAM、MEMORY、InnoDB 和 BDB。对于较旧版本，请使用 MSTinyInteger() 类型。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.mysql.BIT` (`sqlalchemy.types.TypeEngine`)

```py
method __init__(length=None)
```

构造一个 BIT。

参数：

**length** – 可选，位数。

```py
class sqlalchemy.dialects.mysql.BLOB
```

SQL BLOB 类型。

**类签名**

class `sqlalchemy.dialects.mysql.BLOB` (`sqlalchemy.types.LargeBinary`)

```py
method __init__(length: int | None = None)
```

*继承自* `LargeBinary` 的 `sqlalchemy.types.LargeBinary.__init__` *方法*

构造一个 LargeBinary 类型。

参数：

**length** – 可选，用于 DDL 语句中的列长度，适用于那些接受长度的二进制类型，比如 MySQL BLOB 类型。

```py
class sqlalchemy.dialects.mysql.BOOLEAN
```

SQL BOOLEAN 类型。

**类签名**

class `sqlalchemy.dialects.mysql.BOOLEAN` (`sqlalchemy.types.Boolean`)

```py
method __init__(create_constraint: bool = False, name: str | None = None, _create_events: bool = True, _adapted_from: SchemaType | None = None)
```

*继承自* `Boolean` 的 `sqlalchemy.types.Boolean.__init__` *方法*

构造一个 Boolean。

参数：

+   `create_constraint` –

    默认为 False。如果将布尔值生成为 int/smallint，则还在表上创建一个 CHECK 约束，以确保值为 1 或 0。

    注意

    强烈建议 CHECK 约束具有明确的名称，以支持模式管理问题。这可以通过设置 `Boolean.name` 参数或设置适当的命名约定来建立；有关背景信息，请参阅 配置约束命名约定。

    从版本 1.4 开始更改：- 此标志现在默认为 False，表示非本地枚举类型不生成 CHECK 约束。

+   `name` – 如果生成 CHECK 约束，请指定约束的名称。

```py
class sqlalchemy.dialects.mysql.CHAR
```

MySQL CHAR 类型，用于固定长度的字符数据。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.CHAR` (`sqlalchemy.dialects.mysql.types._StringType`, `sqlalchemy.types.CHAR`)

```py
method __init__(length=None, **kwargs)
```

构建一个 CHAR。

参数：

+   `length` – 最大数据长度，以字符为单位。

+   `binary` – 可选项，使用国家字符集的默认二进制排序规则。这不影响存储的数据类型，对于二进制数据，请使用 BINARY 类型。

+   `collation` – 可选项，请求特定排序规则。必须与国家字符集兼容。

```py
class sqlalchemy.dialects.mysql.DATE
```

SQL DATE 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.DATE` (`sqlalchemy.types.Date`)

```py
class sqlalchemy.dialects.mysql.DATETIME
```

MySQL DATETIME 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.DATETIME` (`sqlalchemy.types.DATETIME`)

```py
method __init__(timezone=False, fsp=None)
```

构建一个 MySQL DATETIME 类型。

参数：

+   `timezone` – MySQL 方言不使用。

+   `fsp` –

    小数秒精度值。MySQL 5.6.4 支持小数秒的存储；在发出 DATETIME 类型的 DDL 时将使用此参数。

    注意

    DBAPI 驱动程序对小数秒的支持可能有限；当前支持包括 MySQL Connector/Python。

```py
class sqlalchemy.dialects.mysql.DECIMAL
```

MySQL DECIMAL 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.DECIMAL` (`sqlalchemy.dialects.mysql.types._NumericType`, `sqlalchemy.types.DECIMAL`)

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构建一个 DECIMAL。

参数：

+   `precision` – 此数字中的总位数。如果比例和精度都为 None，则值将存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选项。如果为真，则值将以左侧填充零的字符串形式存储。请注意，这不会影响底层数据库 API 返回的值，其仍然是数值。

```py
class sqlalchemy.dialects.mysql.DOUBLE
```

MySQL DOUBLE 类型。

**类签名**

class `sqlalchemy.dialects.mysql.DOUBLE` (`sqlalchemy.dialects.mysql.types._FloatType`, `sqlalchemy.types.DOUBLE`)

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构造一个 DOUBLE。

注意

默认情况下，`DOUBLE` 类型将从浮点数转换为 Decimal，使用默认的截断为 10 位数。指定 `scale=n` 或 `decimal_return_scale=n` 以更改此比例，或者 `asdecimal=False` 以直接返回 Python 浮点数值。

参数：

+   `precision` – 此数字中的总位数。如果 scale 和 precision 都为 None，则值将存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选项。如果为真，则值将以左侧填充零的字符串形式存储。请注意，这不会影响底层数据库 API 返回的值，其仍然是数值。

```py
class sqlalchemy.dialects.mysql.ENUM
```

MySQL ENUM 类型。

**成员**

__init__()

**类签名**

class `sqlalchemy.dialects.mysql.ENUM` (`sqlalchemy.types.NativeForEmulated`, `sqlalchemy.types.Enum`, `sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(*enums, **kw)
```

构造一个 ENUM。

例如：

```py
Column('myenum', ENUM("foo", "bar", "baz"))
```

参数：

+   `enums` –

    此 ENUM 的有效值范围。在枚举中的值不带引号，生成模式时将被转义并用单引号括起。此对象也可以是符合 PEP-435 的枚举类型。

+   `strict` –

    此标志无效。

    在版本更改：MySQL ENUM 类型以及基本 Enum 类型现在验证所有 Python 数据值。

+   `charset` – 可选项，用于此字符串值的列级字符集。优先于‘ascii’或‘unicode’简写。

+   `collation` – 可选项，用于此字符串值的列级排序。优先于‘binary’简写。

+   `ascii` – 默认为 False：`latin1` 字符集的简写，生成模式中的 ASCII。

+   `unicode` – 默认为 False：`ucs2` 字符集的简写，生成模式中的 UNICODE。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制排序类型。在模式中生成 BINARY。这不会影响存储的数据类型，只会影响字符数据的排序。

```py
class sqlalchemy.dialects.mysql.FLOAT
```

MySQL FLOAT 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.FLOAT` (`sqlalchemy.dialects.mysql.types._FloatType`, `sqlalchemy.types.FLOAT`)

```py
method __init__(precision=None, scale=None, asdecimal=False, **kw)
```

构造一个浮点数。

参数:

+   `precision` – 此数字中的总位数。如果 scale 和 precision 都为 None，则将值存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将存储为左填充的带有零的字符串。请注意，这不会影响底层数据库 API 返回的值，其仍然为数值。

```py
class sqlalchemy.dialects.mysql.INTEGER
```

MySQL INTEGER 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.INTEGER` (`sqlalchemy.dialects.mysql.types._IntegerType`, `sqlalchemy.types.INTEGER`)

```py
method __init__(display_width=None, **kw)
```

构造一个整数。

参数:

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将存储为左填充的带有零的字符串。请注意，这不会影响底层数据库 API 返回的值，其仍然为数值。

```py
class sqlalchemy.dialects.mysql.JSON
```

MySQL JSON 类型。

MySQL 从版本 5.7 开始支持 JSON。MariaDB 从版本 10.2 开始支持 JSON（作为 LONGTEXT 的别名）。

当基本的 `JSON` 数据类型与 MySQL 或 MariaDB 后端一起使用时，`JSON` 会自动使用。

另请参阅

`JSON` - 用于通用跨平台 JSON 数据类型的主要文档。

`JSON` 类型支持 JSON 值的持久性以及通过调整操作以在数据库级别呈现 `JSON_EXTRACT` 函数所提供的核心索引操作，从而适应基本的 `JSON` 数据类型。

**类签名**

类 `sqlalchemy.dialects.mysql.JSON` （`sqlalchemy.types.JSON`）

```py
class sqlalchemy.dialects.mysql.LONGBLOB
```

MySQL LONGBLOB 类型，用于二进制数据最多 2³² 字节。

**类签名**

类 `sqlalchemy.dialects.mysql.LONGBLOB` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.LONGTEXT
```

MySQL LONGTEXT 类型，用于字符存储编码最多 2³² 字节。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.LONGTEXT` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(**kwargs)
```

构造一个 LONGTEXT。

参数：

+   `charset` – 可选，此字符串值的列级字符集。优先于‘ascii’或‘unicode’简写。

+   `collation` – 可选，此字符串值的列级校对。优先于‘binary’简写。

+   `ascii` – 默认为 False：`latin1`字符集的简写，生成模式中的 ASCII。

+   `unicode` – 默认为 False：`ucs2`字符集的简写，生成模式中的 UNICODE。

+   `national` – 可选。如果为 true，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制校对类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的校对。

```py
class sqlalchemy.dialects.mysql.MEDIUMBLOB
```

MySQL MEDIUMBLOB 类型，用于二进制数据达到 2²⁴ 字节。

**类签名**

类 `sqlalchemy.dialects.mysql.MEDIUMBLOB` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.MEDIUMINT
```

MySQL MEDIUMINTEGER 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.MEDIUMINT` (`sqlalchemy.dialects.mysql.types._IntegerType`)

```py
method __init__(display_width=None, **kw)
```

构造一个 MEDIUMINTEGER

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将以左边填充零的字符串形式存储。请注意，这不影响底层数据库 API 返回的值，其仍然是数值。

```py
class sqlalchemy.dialects.mysql.MEDIUMTEXT
```

MySQL MEDIUMTEXT 类型，用于存储编码达到 2²⁴ 字节的字符。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.MEDIUMTEXT` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(**kwargs)
```

构造一个 MEDIUMTEXT。

参数：

+   `charset` – 可选，此字符串值的列级字符集。优先于‘ascii’或‘unicode’简写。

+   `collation` – 可选，此字符串值的列级校对。优先于‘binary’简写。

+   `ascii` – 默认为 False：`latin1`字符集的简写，生成模式中的 ASCII。

+   `unicode` – 默认为 False：`ucs2`字符集的简写，生成模式中的 UNICODE。

+   `national` – 可选。如果为 true，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制校对类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的校对。

```py
class sqlalchemy.dialects.mysql.NCHAR
```

MySQL NCHAR 类型。

用于服务器配置的国家字符集中的固定长度字符数据。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.NCHAR`（`sqlalchemy.dialects.mysql.types._StringType`，`sqlalchemy.types.NCHAR`）

```py
method __init__(length=None, **kwargs)
```

构造一个 NCHAR。

参数：

+   `length` – 最大数据长度，以字符为单位。

+   `binary` – 可选，使用国家字符集的默认二进制排序规则。这不会影响存储的数据类型，对于二进制数据，请使用 BINARY 类型。

+   `collation` – 可选，请求特定��排序规则。必须与国家字符集兼容。

```py
class sqlalchemy.dialects.mysql.NUMERIC
```

MySQL NUMERIC 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.NUMERIC`（`sqlalchemy.dialects.mysql.types._NumericType`，`sqlalchemy.types.NUMERIC`）

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构造一个 NUMERIC。

参数：

+   `precision` – 此数字中的总位数。如果比例和精度都为 None，则值将存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将以左侧填充零的字符串形式存储。请注意，这不会影响底层数据库 API 返回的值，其仍然是数值型的。

```py
class sqlalchemy.dialects.mysql.NVARCHAR
```

MySQL NVARCHAR 类型。

用于服务器配置的国家字符集中的可变长度字符数据。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.NVARCHAR`（`sqlalchemy.dialects.mysql.types._StringType`，`sqlalchemy.types.NVARCHAR`）

```py
method __init__(length=None, **kwargs)
```

构造一个 NVARCHAR。

参数：

+   `length` – 最大数据长度，以字符为单位。

+   `binary` – 可选，使用国家字符集的默认二进制排序规则。这不会影响存储的数据类型，对于二进制数据，请使用 BINARY 类型。

+   `collation` – 可选，请求特定的排序规则。必须与国家字符集兼容。

```py
class sqlalchemy.dialects.mysql.REAL
```

MySQL REAL 类型。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.REAL`（`sqlalchemy.dialects.mysql.types._FloatType`，`sqlalchemy.types.REAL`）

```py
method __init__(precision=None, scale=None, asdecimal=True, **kw)
```

构造一个 REAL。

注意

默认情况下，`REAL` 类型将从浮点数转换为 Decimal，使用默认为 10 位的截断。要更改此标度，请指定 `scale=n` 或 `decimal_return_scale=n`，或者指定 `asdecimal=False` 以直接将值返回为 Python 浮点数。

参数：

+   `precision` – 此数字中的总位数。如果 scale 和 precision 都为 None，则值将存储到服务器允许的限制。

+   `scale` – 小数点后的位数。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为真，则值将作为用零左填充的字符串存储。请注意，这不影响底层数据库 API 返回的值，这些值仍然是数字。

```py
class sqlalchemy.dialects.mysql.SET
```

MySQL SET 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.SET` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(*values, **kw)
```

构造一个 SET。

例如：

```py
Column('myset', SET("foo", "bar", "baz"))
```

在此 set 将用于为表生成 DDL 或者如果 `SET.retrieve_as_bitwise` 标志设置为 True，则必须提供潜在值的列表。

参数：

+   `values` – 此 SET 的有效值范围。这些值不加引号，生成模式时会被转义并用单引号括起来。

+   `convert_unicode` – 与 `String.convert_unicode` 相同的标志。

+   `collation` – 与 `String.collation` 相同。

+   `charset` – 与 `VARCHAR.charset` 相同。

+   `ascii` – 与 `VARCHAR.ascii` 相同。

+   `unicode` – 与 `VARCHAR.unicode` 相同。

+   `binary` – 与 `VARCHAR.binary` 相同。

+   `retrieve_as_bitwise` –

    如果为 True，set 类型的数据将使用整数值进行持久化和选择，其中一个 set 被强制转换为位掩码进行持久化。MySQL 允许此模式，它的优点是能够明确地存储值，如空字符串 `''`。在 SELECT 语句中，数据类型将显示为表达式 `col + 0`，以便值被强制转换为整数值在结果集中返回。如果希望持久化一个可以存储空字符串 `''` 作为值的 set，则需要此标志。

    警告

    在使用 `SET.retrieve_as_bitwise` 时，重要的是确保集合值的列表与 MySQL 数据库中存在的**完全相同的顺序**。

```py
class sqlalchemy.dialects.mysql.SMALLINT
```

MySQL SMALLINTEGER 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.SMALLINT` (`sqlalchemy.dialects.mysql.types._IntegerType`, `sqlalchemy.types.SMALLINT`)

```py
method __init__(display_width=None, **kw)
```

构造一个 SMALLINTEGER。

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为真，则值将作为用零填充的字符串存储。请注意，这不影响底层数据库 API 返回的值，它们仍然是数字。

```py
class sqlalchemy.dialects.mysql.TEXT
```

MySQL TEXT 类型，用于存储编码为最多 2¹⁶ 字节的字符。

**类签名**

类 `sqlalchemy.dialects.mysql.TEXT` (`sqlalchemy.dialects.mysql.types._StringType`, `sqlalchemy.types.TEXT`)

```py
method __init__(length=None, **kw)
```

构造一个 TEXT。

参数：

+   `length` – 可选，如果提供了，服务器可以通过用足够存储 `length` 字节字符的最小 TEXT 类型替换来优化存储。

+   `charset` – 可选，此字符串值的列级字符集。优先于 ‘ascii’ 或 ‘unicode’ 简写。

+   `collation` – 可选，此字符串值的列级排序。优先于 ‘binary’ 简写。

+   `ascii` – 默认为 False：`latin1` 字符集的简写，生成模式中的 ASCII。

+   `unicode` – 默认为 False：`ucs2` 字符集的简写，生成模式中的 UNICODE。

+   `national` – 可选。如果为真，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制排序类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的排序。

```py
class sqlalchemy.dialects.mysql.TIME
```

MySQL TIME 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.TIME` (`sqlalchemy.types.TIME`)

```py
method __init__(timezone=False, fsp=None)
```

构造一个 MySQL TIME 类型。

参数：

+   `timezone` – MySQL 方言不使用。

+   `fsp` –

    分数秒精度值。MySQL 5.6 支持存储分数秒；在发出 TIME 类型的 DDL 时将使用此参数。

    注意

    DBAPI 驱动程序对于分数秒的支持可能有限；当前支持包括 MySQL Connector/Python。

```py
class sqlalchemy.dialects.mysql.TIMESTAMP
```

MySQL TIMESTAMP 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.TIMESTAMP` (`sqlalchemy.types.TIMESTAMP`)

```py
method __init__(timezone=False, fsp=None)
```

构造一个 MySQL TIMESTAMP 类型。

参数：

+   `timezone` – MySQL 方言不使用。

+   `fsp` –

    分数秒精度值。MySQL 5.6.4 支持分数秒的存储；在为 TIMESTAMP 类型发出 DDL 时将使用此参数。

    注意

    DBAPI 驱动程序对分数秒的支持可能有限；当前支持包括 MySQL Connector/Python。

```py
class sqlalchemy.dialects.mysql.TINYBLOB
```

MySQL TINYBLOB 类型，用于存储最多 2⁸ 字节的二进制数据。

**类签名**

类 `sqlalchemy.dialects.mysql.TINYBLOB` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.TINYINT
```

MySQL TINYINT 类型。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.TINYINT` (`sqlalchemy.dialects.mysql.types._IntegerType`)

```py
method __init__(display_width=None, **kw)
```

构造一个 TINYINT。

参数：

+   `display_width` – 可选，此数字的最大显示宽度。

+   `unsigned` – 一个布尔值，可选。

+   `zerofill` – 可选。如果为 true，则值将存储为左填充零的字符串。注意，这不会影响底层数据库 API 返回的值，这些值仍然是数字。

```py
class sqlalchemy.dialects.mysql.TINYTEXT
```

MySQL TINYTEXT 类型，用于存储编码为 2⁸ 字节的字符。

**成员**

__init__()

**类签名**

类 `sqlalchemy.dialects.mysql.TINYTEXT` (`sqlalchemy.dialects.mysql.types._StringType`)

```py
method __init__(**kwargs)
```

构造一个 TINYTEXT。

参数：

+   `charset` – 可选，此字符串值的列级字符集。优先于 ‘ascii’ 或 ‘unicode’ 简写。

+   `collation` – 可选，此字符串值的列级排序。优先于 ‘binary’ 简写。

+   `ascii` – 默认为 False：`latin1` 字符集的简写，在模式中生成 ASCII。

+   `unicode` – 默认为 False：`ucs2` 字符集的简写，在模式中生成 UNICODE。

+   `national` – 可选。如果为 true，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写，选择与列的字符集匹配的二进制排序类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的排序。

```py
class sqlalchemy.dialects.mysql.VARBINARY
```

SQL VARBINARY 类型。

**类签名**

类 `sqlalchemy.dialects.mysql.VARBINARY` (`sqlalchemy.types._Binary`)

```py
class sqlalchemy.dialects.mysql.VARCHAR
```

MySQL VARCHAR 类型，用于可变长度字符数据。

**成员**

__init__()

**类签名**

类`sqlalchemy.dialects.mysql.VARCHAR` (`sqlalchemy.dialects.mysql.types._StringType`, `sqlalchemy.types.VARCHAR`)

```py
method __init__(length=None, **kwargs)
```

构造一个 VARCHAR。

参数：

+   `charset` – 可选，此字符串值的列级字符集。优先于‘ascii’或‘unicode’的简写形式。

+   `collation` – 可选，此字符串值的列级排序规则。优先于‘binary’的简写形式。

+   `ascii` – 默认为 False：`latin1`字符集的简写形式，在模式中生成 ASCII。

+   `unicode` – 默认为 False：`ucs2`字符集的简写形式，在模式中生成 UNICODE。

+   `national` – 可选。如果为 true，则使用服务器配置的国家字符集。

+   `binary` – 默认为 False：简写形式，选择与列的字符集匹配的二进制排序类型。在模式中生成 BINARY。这不影响存储的数据类型，只影响字符数据的排序规则。

```py
class sqlalchemy.dialects.mysql.YEAR
```

MySQL YEAR 类型，用于存储 1901-2155 年的单字节。

**类签名**

类`sqlalchemy.dialects.mysql.YEAR` (`sqlalchemy.types.TypeEngine`)

## MySQL DML 构造

| 对象名称 | 描述 |
| --- | --- |
| 插入(表) | 构造一个 MySQL/MariaDB 特定变体的`Insert`构造。 |
| 插入 | INSERT 的 MySQL 特定实现。 |

```py
function sqlalchemy.dialects.mysql.insert(table: _DMLTableArgument) → Insert
```

构造一个 MySQL/MariaDB 特定变体的`Insert`构造。

`sqlalchemy.dialects.mysql.insert()`函数创建一个`sqlalchemy.dialects.mysql.Insert`。这个类基于方言不可知的`Insert`构造，可以使用 SQLAlchemy Core 中的`insert()`函数构造。

`Insert`构造包括额外的方法`Insert.on_duplicate_key_update()`。

```py
class sqlalchemy.dialects.mysql.Insert
```

INSERT 的 MySQL 特定实现。

添加用于 MySQL 特定语法的方法，如 ON DUPLICATE KEY UPDATE。

`Insert`对象是使用`sqlalchemy.dialects.mysql.insert()`函数创建的。

版本 1.2 中的新功能。

**成员**

inherit_cache, inserted, on_duplicate_key_update()

**类签名**

类`sqlalchemy.dialects.mysql.Insert` (`sqlalchemy.sql.expression.Insert`)

```py
attribute inherit_cache: bool | None = False
```

指示此`HasCacheKey`实例是否应使用其直接超类使用的缓存键生成方案。

该属性默认为`None`，表示构造尚未考虑其是否适合参与缓存；这在功能上等同于将值设置为`False`，只是还会发出警告。

如果与该对象对应的 SQL 不基于此类的本地属性而是其超类，则可以在特定类上将此标志设置为`True`。

另请参阅

为自定义构造启用缓存支持 - 为第三方或用户定义的 SQL 构造设置`HasCacheKey.inherit_cache`属性的一般指南。

```py
attribute inserted
```

为 ON DUPLICATE KEY UPDATE 语句提供“inserted”命名空间

MySQL 的 ON DUPLICATE KEY UPDATE 子句允许引用将要插入的行，通过一个名为`VALUES()`的特殊函数。此属性提供了此行中的所有列可引用，以便它们在 ON DUPLICATE KEY UPDATE 子句中的`VALUES()`函数内呈现。该属性命名为`.inserted`，以避免与现有的`Insert.values()`方法发生冲突。

提示

`Insert.inserted` 属性是 `ColumnCollection` 的实例，提供了与 访问表和列 中描述的 `Table.c` 集合相同的接口。使用此集合，可以像属性一样访问普通名称（例如 `stmt.inserted.some_column`），但应该使用索引访问特殊名称和字典方法名称，例如 `stmt.inserted["column name"]` 或 `stmt.inserted["values"]`。有关更多示例，请参阅 `ColumnCollection` 的文档字符串。

另请参阅

INSERT…ON DUPLICATE KEY UPDATE（插入或更新） - 使用 `Insert.inserted` 的示例

```py
method on_duplicate_key_update(*args: Mapping[Any, Any] | List[Tuple[str, Any]] | ColumnCollection[Any, Any], **kw: Any) → Self
```

指定 ON DUPLICATE KEY UPDATE 子句。

参数：

****kw** – 与 UPDATE 值关联的列键。值可以是任何 SQL 表达式或支持的字面 Python 值。

警告

此字典**不会**考虑 Python 指定的默认 UPDATE 值或生成函数，例如那些使用 `Column.onupdate` 指定的值。除非在此处手动指定值，否则这些值将不会被用于 ON DUPLICATE KEY UPDATE 风格的 UPDATE。

参数：

***args** –

作为传递键/值参数的替代方法，可以将字典或 2 元组的列表作为单个位置参数传递。

传递单个字典相当于关键字参数形式：

```py
insert().on_duplicate_key_update({"name": "some name"})
```

传递 2 元组的列表表示 UPDATE 子句中的参数分配应按发送顺序排序，类似于整体描述的 `Update` 构造中的 参数排序更新：

```py
insert().on_duplicate_key_update(
    [("name", "some name"), ("value", "some value")])
```

自 1.3 版更改：参数可以指定为字典或 2 元组的列表；后一种形式提供了参数的排序。

自 1.2 版新功能。

另请参阅

INSERT…ON DUPLICATE KEY UPDATE（插入或更新）

## mysqlclient（MySQL-Python 的分支）

通过 mysqlclient（MySQL-Python 的维护分支）驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

mysqlclient（MySQL-Python 的维护分支）的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/mysqlclient/`](https://pypi.org/project/mysqlclient/)

### 连接中

连接字符串：

```py
mysql+mysqldb://<user>:<password>@<host>[:<port>]/<dbname>
```

### 驱动程序状态

mysqlclient DBAPI 是[MySQL-Python](https://sourceforge.net/projects/mysql-python) DBAPI 的维护分支，后者已不再维护。[mysqlclient](https://github.com/PyMySQL/mysqlclient-python)支持 Python 2 和 Python 3，并且非常稳定。

### Unicode

请参阅 Unicode 以获取有关 Unicode 处理的当前建议。### SSL Connections

mysqlclient 和 PyMySQL DBAPI 接受一个额外的字典，键为“ssl”，可以使用`create_engine.connect_args`字典指定：

```py
engine = create_engine(
    "mysql+mysqldb://scott:tiger@192.168.0.134/test",
    connect_args={
        "ssl": {
            "ca": "/home/gord/client-ssl/ca.pem",
            "cert": "/home/gord/client-ssl/client-cert.pem",
            "key": "/home/gord/client-ssl/client-key.pem"
        }
    }
)
```

为方便起见，以下键也可以内联在 URL 中指定，它们将自动解释为“ssl”字典中： “ssl_ca”，“ssl_cert”，“ssl_key”，“ssl_capath”，“ssl_cipher”，“ssl_check_hostname”。示例如下：

```py
connection_uri = (
    "mysql+mysqldb://scott:tiger@192.168.0.134/test"
    "?ssl_ca=/home/gord/client-ssl/ca.pem"
    "&ssl_cert=/home/gord/client-ssl/client-cert.pem"
    "&ssl_key=/home/gord/client-ssl/client-key.pem"
)
```

另请参阅

SSL Connections 在 PyMySQL 方言中

### 使用 MySQLdb 与 Google Cloud SQL

Google Cloud SQL 现在建议使用 MySQLdb 方言。使用以下 URL 进行连接：

```py
mysql+mysqldb://root@/<dbname>?unix_socket=/cloudsql/<projectid>:<instancename>
```

### 服务器端游标

mysqldb 方言支持服务器端游标。请参阅 Server Side Cursors。

### DBAPI

mysqlclient（MySQL-Python 的维护分支）的文档和下载信息（如果适用）可在以下链接找到：[`pypi.org/project/mysqlclient/`](https://pypi.org/project/mysqlclient/)

### 连接中

连接字符串：

```py
mysql+mysqldb://<user>:<password>@<host>[:<port>]/<dbname>
```

### 驱动程序状态

mysqlclient DBAPI 是[MySQL-Python](https://sourceforge.net/projects/mysql-python) DBAPI 的维护分支，后者已不再维护。[mysqlclient](https://github.com/PyMySQL/mysqlclient-python)支持 Python 2 和 Python 3，并且非常稳定。

### Unicode

请参阅 Unicode 以获取有关 Unicode 处理的当前建议。

### SSL Connections

mysqlclient 和 PyMySQL DBAPI 接受一个额外的字典，键为“ssl”，可以使用`create_engine.connect_args`字典指定：

```py
engine = create_engine(
    "mysql+mysqldb://scott:tiger@192.168.0.134/test",
    connect_args={
        "ssl": {
            "ca": "/home/gord/client-ssl/ca.pem",
            "cert": "/home/gord/client-ssl/client-cert.pem",
            "key": "/home/gord/client-ssl/client-key.pem"
        }
    }
)
```

为方便起见，以下键也可以内联在 URL 中指定，它们将自动解释为“ssl”字典中： “ssl_ca”，“ssl_cert”，“ssl_key”，“ssl_capath”，“ssl_cipher”，“ssl_check_hostname”。示例如下：

```py
connection_uri = (
    "mysql+mysqldb://scott:tiger@192.168.0.134/test"
    "?ssl_ca=/home/gord/client-ssl/ca.pem"
    "&ssl_cert=/home/gord/client-ssl/client-cert.pem"
    "&ssl_key=/home/gord/client-ssl/client-key.pem"
)
```

另请参阅

SSL Connections 在 PyMySQL 方言中

### 使用 MySQLdb 与 Google Cloud SQL

Google Cloud SQL 现在建议使用 MySQLdb 方言。使用以下 URL 进行连接：

```py
mysql+mysqldb://root@/<dbname>?unix_socket=/cloudsql/<projectid>:<instancename>
```

### 服务器端游标

mysqldb 方言支持服务器端游标。请参阅 Server Side Cursors。

## PyMySQL

通过 PyMySQL 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

PyMySQL 的文档和下载信息（如果适用）可在以下链接找到：[`pymysql.readthedocs.io/`](https://pymysql.readthedocs.io/)

### 连接中

连接字符串：

```py
mysql+pymysql://<username>:<password>@<host>/<dbname>[?<options>]
```

### Unicode

请参阅 Unicode 以获取有关 Unicode 处理的当前建议。

### SSL 连接

PyMySQL DBAPI 接受与 MySQLdb 相同���SSL 参数，详见 SSL 连接。请参阅该部分以获取其他示例。

如果服务器使用自动生成的自签名证书或与主机名不匹配（从客户端看），还可能需要在 PyMySQL 中指定`ssl_check_hostname=false`：

```py
connection_uri = (
    "mysql+pymysql://scott:tiger@192.168.0.134/test"
    "?ssl_ca=/home/gord/client-ssl/ca.pem"
    "&ssl_cert=/home/gord/client-ssl/client-cert.pem"
    "&ssl_key=/home/gord/client-ssl/client-key.pem"
    "&ssl_check_hostname=false"
)
```

### MySQL-Python 兼容性

pymysql DBAPI 是 MySQL-python（MySQLdb）驱动程序的纯 Python 移植版本，目标是 100%兼容。对于 MySQL-python 的大多数行为注意事项也适用于 pymysql 驱动程序。

### DBAPI

PyMySQL 的文档和下载信息（如果适用）可在此处找到：[`pymysql.readthedocs.io/`](https://pymysql.readthedocs.io/)

### 连接

连接字符串:

```py
mysql+pymysql://<username>:<password>@<host>/<dbname>[?<options>]
```

### Unicode

有关当前有关 Unicode 处理的建议，请参阅 Unicode。

### SSL 连接

PyMySQL DBAPI 接受与 MySQLdb 相同的 SSL 参数，详见 SSL 连接。请参阅该部分以获取其他示例。

如果服务器使用自动生成的自签名证书或与主机名不匹配（从客户端看），还可能需要在 PyMySQL 中指定`ssl_check_hostname=false`：

```py
connection_uri = (
    "mysql+pymysql://scott:tiger@192.168.0.134/test"
    "?ssl_ca=/home/gord/client-ssl/ca.pem"
    "&ssl_cert=/home/gord/client-ssl/client-cert.pem"
    "&ssl_key=/home/gord/client-ssl/client-key.pem"
    "&ssl_check_hostname=false"
)
```

### MySQL-Python 兼容性

pymysql DBAPI 是 MySQL-python（MySQLdb）驱动程序的纯 Python 移植版本，目标是 100%兼容。对于 MySQL-python 的大多数行为注意事项也适用于 pymysql 驱动程序。

## MariaDB-连接器

通过 MariaDB Connector/Python 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

MariaDB Connector/Python 的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/mariadb/`](https://pypi.org/project/mariadb/)

### 连接

连接字符串:

```py
mariadb+mariadbconnector://<user>:<password>@<host>[:<port>]/<dbname>
```

### 驱动程序状态

MariaDB Connector/Python 使 Python 程序能够使用符合 Python DB API 2.0（PEP-249）的 API 访问 MariaDB 和 MySQL 数据库。它是用 C 编写的，并使用 MariaDB Connector/C 客户端库进行客户端服务器通信。

请注意，`mariadb://`连接 URI 的默认驱动程序仍然是`mysqldb`。要使用此驱动程序，需要`mariadb+mariadbconnector://`。

### DBAPI

MariaDB Connector/Python 的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/mariadb/`](https://pypi.org/project/mariadb/)

### 连接

连接字符串:

```py
mariadb+mariadbconnector://<user>:<password>@<host>[:<port>]/<dbname>
```

### 驱动程序状态

MariaDB Connector/Python 使 Python 程序能够使用符合 Python DB API 2.0（PEP-249）的 API 访问 MariaDB 和 MySQL 数据库。它是用 C 编写的，并使用 MariaDB Connector/C 客户端库进行客户端服务器通信。

请注意，`mariadb://`连接 URI 的默认驱动程序仍然是`mysqldb`。要使用此驱动程序，需要`mariadb+mariadbconnector://`。

## MySQL-连接器

通过 MySQL Connector/Python 驱动程序支持 MySQL/MariaDB 数据库。

### DBAPI

MySQL Connector/Python 的文档和下载信息（如果适用）可在此处获取：[`pypi.org/project/mysql-connector-python/`](https://pypi.org/project/mysql-connector-python/)

### 连接

连接字符串：

```py
mysql+mysqlconnector://<user>:<password>@<host>[:<port>]/<dbname>
```

注意

自发布以来，MySQL Connector/Python DBAPI 存在许多问题，其中一些可能仍未解决，而 mysqlconnector 方言**未作为 SQLAlchemy 持续集成的一部分进行测试**。推荐的 MySQL 方言是 mysqlclient 和 PyMySQL。

### DBAPI

MySQL Connector/Python 的文档和下载信息（如果适用）可在此处获取：[`pypi.org/project/mysql-connector-python/`](https://pypi.org/project/mysql-connector-python/)

### 连接

连接字符串：

```py
mysql+mysqlconnector://<user>:<password>@<host>[:<port>]/<dbname>
```

## asyncmy

通过 asyncmy 驱动程序支持 MySQL/MariaDB 数据库。

### DBAPI

异步 MySQL 的文档和下载信息（如果适用）可在此处获取：[`github.com/long2ice/asyncmy`](https://github.com/long2ice/asyncmy)

### 连接

连接字符串：

```py
mysql+asyncmy://user:password@host:port/dbname[?key=value&key=value...]
```

使用特殊的 asyncio 中介层，asyncmy 方言可作为 SQLAlchemy asyncio 扩展包的后端使用。

此方言通常只应与`create_async_engine()`引擎创建函数一起使用：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("mysql+asyncmy://user:pass@hostname/dbname?charset=utf8mb4")
```

### DBAPI

异步 MySQL 的文档和下载信息（如果适用）可在此处获取：[`github.com/long2ice/asyncmy`](https://github.com/long2ice/asyncmy)

### 连接

连接字符串：

```py
mysql+asyncmy://user:password@host:port/dbname[?key=value&key=value...]
```

## aiomysql

通过 aiomysql 驱动程序支持 MySQL/MariaDB 数据库。

### DBAPI

aiomysql 的文档和下载信息（如果适用）可在此处获取：[`github.com/aio-libs/aiomysql`](https://github.com/aio-libs/aiomysql)

### 连接

连接字符串：

```py
mysql+aiomysql://user:password@host:port/dbname[?key=value&key=value...]
```

aiomysql 方言是 SQLAlchemy 的第二个 Python asyncio 方言。

使用特殊的 asyncio 中介层，aiomysql 方言可作为 SQLAlchemy asyncio 扩展包的后端使用。

此方言通常只应与`create_async_engine()`引擎创建函数一起使用：

```py
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("mysql+aiomysql://user:pass@hostname/dbname?charset=utf8mb4")
```

### DBAPI

aiomysql 的文档和下载信息（如果适用）可在此处获取：[`github.com/aio-libs/aiomysql`](https://github.com/aio-libs/aiomysql)

### 连接

连接字符串：

```py
mysql+aiomysql://user:password@host:port/dbname[?key=value&key=value...]
```

## cymysql

通过 CyMySQL 驱动程序支持 MySQL/MariaDB 数据库。

### DBAPI

CyMySQL 的文档和下载信息（如果适用）可在此处获取：[`github.com/nakagami/CyMySQL`](https://github.com/nakagami/CyMySQL)

### 连接

连接字符串：

```py
mysql+cymysql://<username>:<password>@<host>/<dbname>[?<options>]
```

注意

CyMySQL 方言**不在 SQLAlchemy 的持续集成测试范围内**，可能存在未解决的问题。推荐使用的 MySQL 方言是 mysqlclient 和 PyMySQL。

### DBAPI

CyMySQL 的文档和下载信息（如果适用）可在此处找到：[`github.com/nakagami/CyMySQL`](https://github.com/nakagami/CyMySQL)

### 连接

连接字符串：

```py
mysql+cymysql://<username>:<password>@<host>/<dbname>[?<options>]
```

## pyodbc

通过 PyODBC 驱动程序支持 MySQL / MariaDB 数据库。

### DBAPI

PyODBC 的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/pyodbc/`](https://pypi.org/project/pyodbc/)

### 连接

连接字符串：

```py
mysql+pyodbc://<username>:<password>@<dsnname>
```

注意

PyODBC 对于 MySQL 方言**不在 SQLAlchemy 的持续集成测试范围内**。推荐使用的 MySQL 方言是 mysqlclient 和 PyMySQL。但是，如果您想使用 mysql+pyodbc 方言并且需要对`utf8mb4`字符（包括表情符号等辅助字符）进行完全支持，请确保使用当前版本的 MySQL Connector/ODBC 并在 DSN 或连接字符串中指定“ANSI”（而不是“Unicode”）版本的驱动程序。

通过精确的 pyodbc 连接字符串进行传递：

```py
import urllib
connection_string = (
    'DRIVER=MySQL ODBC 8.0 ANSI Driver;'
    'SERVER=localhost;'
    'PORT=3307;'
    'DATABASE=mydb;'
    'UID=root;'
    'PWD=(whatever);'
    'charset=utf8mb4;'
)
params = urllib.parse.quote_plus(connection_string)
connection_uri = "mysql+pyodbc:///?odbc_connect=%s" % params
```

### DBAPI

PyODBC 的文档和下载信息（如果适用）可在此处找到：[`pypi.org/project/pyodbc/`](https://pypi.org/project/pyodbc/)

### 连接

连接字符串：

```py
mysql+pyodbc://<username>:<password>@<dsnname>
```
